# KVM_DEV_ARM_VGIC_GRP_CPU_SYSREGS 通道解析（VGIC 设备属性 group）

VGIC v3 设备上访问 per-vCPU CPU interface（ICC_*）系统寄存器的通道。通过
`KVM_SET_DEVICE_ATTR` / `KVM_GET_DEVICE_ATTR` 作用在 `KVM_DEV_TYPE_ARM_VGIC_V3`
设备 fd 上，主要用于 live migration 时保存/恢复 GIC CPU interface 状态。

> **概念澄清**：`KVM_DEV_ARM_VGIC_GRP_CPU_SYSREGS`（= 6）**不是 ioctl**，而是
> `struct kvm_device_attr` 的 `group` 字段的取值之一，即"寄存器组选择器"。
> 真正的 ioctl 请求码是 `KVM_CREATE_DEVICE`（创建设备 fd，type =
> `KVM_DEV_TYPE_ARM_VGIC_V3`）以及 `KVM_SET_DEVICE_ATTR` (`_IOW(KVMIO, 0xe1)`)、
> `KVM_GET_DEVICE_ATTR` (0xe2)、`KVM_HAS_DEVICE_ATTR` (0xe3)，定义在
> include/uapi/linux/kvm.h:1689-1694，作用在 `KVM_CREATE_DEVICE` 返回的设备 fd 上。

## 目录

1. [UAPI：attr 编码](#1-uapiattr-编码)
2. [内核调用栈：SET/GET_DEVICE_ATTR](#2-内核调用栈setget_device_attr)
3. [内核调用栈：HAS_DEVICE_ATTR](#3-内核调用栈has_device_attr)
4. [寄存器处理语义](#4-寄存器处理语义)
5. [QEMU 调用栈](#5-qemu-调用栈)
6. [与 KVM_GET_ONE_REG 的关系](#6-与-kvm_get_one_reg-的关系)

---

## 1. UAPI：attr 编码

```c
/* arch/arm64/include/uapi/asm/kvm.h */
#define KVM_DEV_ARM_VGIC_GRP_CPU_SYSREGS 6                       /* :426 */

#define KVM_DEV_ARM_VGIC_V3_MPIDR_SHIFT  32                      /* :417 */
#define KVM_DEV_ARM_VGIC_V3_MPIDR_MASK   (0xffffffffULL << 32)
#define KVM_DEV_ARM_VGIC_SYSREG_INSTR_MASK (0xffff)              /* :422 */
```

```
kvm_device_attr {
  .group = KVM_DEV_ARM_VGIC_GRP_CPU_SYSREGS (6)
  .attr  = [63:32] MPIDR（选 vCPU） | [15:0] sysreg 编码 (op0,op1,crn,crm,op2)
  .addr  = 用户态 u64 缓冲区指针
}
```

低 16 位的位布局与 `KVM_REG_ARM64_SYSREG` 的 id 低 16 位**完全一致**（OP0=bit14-15、
OP1=11-13、CRn=7-10、CRm=3-6、Op2=0-2），因此内核可以无换算地把 attr 拼成标准
sysreg id。

覆盖的 15 个寄存器：ICC_PMR_EL1、ICC_BPR0_EL1、ICC_AP0R0-3_EL1、ICC_AP1R0-3_EL1、
ICC_BPR1_EL1、ICC_CTLR_EL1、ICC_SRE_EL1、ICC_IGRPEN0/1_EL1。

## 2. 内核调用栈：SET/GET_DEVICE_ATTR

```
ioctl(vgic_fd, KVM_SET_DEVICE_ATTR / KVM_GET_DEVICE_ATTR)  /* fd 来自 KVM_CREATE_DEVICE */
  └─ kvm_device_ioctl                                   virt/kvm/kvm_main.c:4950
       └─ kvm_device_ioctl_attr   copy_from_user(struct kvm_device_attr)  :4934
            └─ dev->ops->set_attr / get_attr
  └─ kvm_vgic_v3_set_attr / kvm_vgic_v3_get_attr    vgic/vgic-kvm-device.c:597/611
       switch (attr->group)  ← KVM_DEV_ARM_VGIC_GRP_CPU_SYSREGS 在此被消费
       └─ vgic_v3_attr_regs_access(dev, attr, is_write)  vgic-kvm-device.c:507
            ├─ vgic_v3_parse_attr                        vgic-kvm-device.c:473
            │    attr[63:32] → MPIDR → kvm_mpidr_to_vcpu 定位 vCPU
            │    （DIST_REGS 组例外：attr 不带 MPIDR，默认 vcpu0）
            ├─ CPU_SYSREGS → uaccess = false             跳过通用 u32 拷贝
            │    （本通道固定 u64，由 sysreg 处理代码自己 uaccess）
            ├─ 加锁：kvm->lock + lock_all_vcpus + arch.config_lock
            ├─ 检查 vgic_initialized()，未 init → -EBUSY
            └─ vgic_v3_cpu_sysregs_uaccess                vgic-sys-reg-v3.c:351
                 ├─ attr_to_id                              vgic-sys-reg-v3.c:333
                 │   低 16 位 → ARM64_SYS_REG(...) 完整 reg id
                 │   （含 KVM_REG_SIZE_U64，固定 64 位）
                 └─ kvm_sys_reg_set_user / get_user          sys_regs.c:3864/3828
                      ├─ id_to_sys_reg_desc → find_reg(gic_v3_icc_reg_descs)
                      └─ r->set_user / r->get_user 回调
```

**关键设计**：`vgic_v3_cpu_sysregs_uaccess` 伪造一个 `struct kvm_one_reg`
（id = attr_to_id(attr->attr)，addr = attr->addr），**直接复用 sys_regs.c 的
`kvm_sys_reg_get/set_user()`**——也就是 `KVM_GET_ONE_REG` 的核心函数，只是查找表从
通用 `sys_reg_descs` 换成私有表 `gic_v3_icc_reg_descs`。读写固定按一个 `u64`
拷贝（`get_user(val, uaddr)` / `put_user(val, uaddr)`），不存在 32/64 判定。

## 3. 内核调用栈：HAS_DEVICE_ATTR

```
ioctl KVM_HAS_DEVICE_ATTR
  └─ kvm_device_ioctl → kvm_device_ioctl_attr  virt/kvm/kvm_main.c:4950/:4934
       └─ kvm_vgic_v3_has_attr              vgic-kvm-device.c:625
       └─ vgic_v3_has_attr_regs        vgic/vgic-mmio-v3.c:1069
            case KVM_DEV_ARM_VGIC_GRP_CPU_SYSREGS:        :1097
              └─ vgic_v3_has_cpu_sysregs_attr  vgic-sys-reg-v3.c:342
                   └─ get_reg_by_id(attr_to_id(attr->attr),
                                    gic_v3_icc_reg_descs, ...)
```

## 4. 寄存器处理语义

`gic_v3_icc_reg_descs[]`（vgic-sys-reg-v3.c:300）每个条目有专属 get/set_user 回调：

| 寄存器 | 处理 |
|---|---|
| ICC_CTLR_EL1 | set：校验 PriBits/IDBits 不超过 host 能力、SEIS/A3V 必须与 host `ICH_VTR_EL2` 一致；cbpr/eoim 写入 `vmcr`（ICC_CTLR 布局，`vgic_set_vmcr()` 内部转 ICH_VMCR 布局） |
| ICC_PMR_EL1 | 直接映射 `vmcr.pmr` |
| ICC_BPR0_EL1 | 映射 `vmcr.bpr` |
| ICC_BPR1_EL1 | 映射 `vmcr.abpr`，仅 `!vmcr.cbpr` 时生效；get 在 cbpr 时返回 `min(bpr+1, 7)` |
| ICC_IGRPEN0/1_EL1 | 映射 `vmcr.grpen0/1` |
| ICC_AP0R<n>/AP1R<n>_EL1 | 直接读写 `vgic_v3_cpu_if.vgic_ap0r[idx]/ap1r[idx]`，idx = Op2 & 3，越界 `vgic_v3_max_apr_idx()` → -EINVAL |
| ICC_SRE_EL1 | set 校验 SRE 位必须为 1（只支持 v3 模式）；get 返回缓存 `vgic_sre` |

## 5. QEMU 调用栈

**编码宏**（hw/intc/arm_gicv3_kvm.c）：

```c
#define KVM_DEV_ARM_VGIC_SYSREG(op0, op1, crn, crm, op2)   /* :45 */
        (ARM64_SYS_REG_SHIFT_MASK(op0, OP0) | ...)
#define ICC_CTLR_EL1    KVM_DEV_ARM_VGIC_SYSREG(3, 0, 12, 12, 4)  /* :62 */

#define KVM_VGIC_ATTR(reg, typer)                          /* :84 */
        ((typer & KVM_DEV_ARM_VGIC_V3_MPIDR_MASK) | (reg))
```

**迁移路径**：

```
保存: vmstate pre_save → kvm_arm_gicv3_get   arm_gicv3_kvm.c:502
恢复: post_load        → kvm_arm_gicv3_put   arm_gicv3_kvm.c:317
  └─ kvm_gicc_access(s, ICC_xx_EL1, cpu, &val, write)      :103
       └─ kvm_device_access(s->dev_fd,
                            KVM_DEV_ARM_VGIC_GRP_CPU_SYSREGS,
                            KVM_VGIC_ATTR(reg, c->gicr_typer),
                            &val, write)     /* uint64_t */
            └─ ioctl KVM_SET/GET_DEVICE_ATTR
```

MPIDR 取自 `gicr_typer`（GICR_TYPER 的 Affinity 字段即 MPIDR 编码）。

另外 realize 阶段读一次初始 `ICC_CTLR_EL1` 存到 `kvm_reset_icc_ctlr_el1`
（arm_gicv3_kvm.c:944），供 CPU interface reset 时恢复。

## 6. 与 KVM_GET_ONE_REG 的关系

- **不同通道**：device attr（vgic 设备 fd，attr 里带 MPIDR 选 vCPU）vs
  vcpu one_reg（vcpu fd，id 里带 op0/op1/crn/crm/op2 与 size 位）。
- **宽度固定**：本通道 attr_to_id 构造的 id 固定带 `KVM_REG_SIZE_U64`，不存在
  `KVM_REG_SIZE_MASK` 的 32/64 判定，见 [[kvm_reg_id_size_encoding]]。
- **机制复用**：内核侧通过伪造 `kvm_one_reg` 复用 `kvm_sys_reg_get/set_user`，
  只是换私有查找表，避免为 VGIC 再实现一套 sysreg 查找/校验逻辑。
- **历史**：ICC_* 系统寄存器原先混在通用 `sys_reg_descs` 表里由 one_reg 通道
  处理，后随 VGIC CPU interface 状态管理收敛，独立成 vgic-sys-reg-v3.c +
  device attr 通道。

## 关键文件

| 文件 | 内容 |
|---|---|
| include/uapi/linux/kvm.h | kvm_device_attr 结构 (:1473)、KVM_SET/GET/HAS_DEVICE_ATTR (:1689-1694) |
| virt/kvm/kvm_main.c | kvm_device_ioctl (:4950)、kvm_device_ioctl_attr (:4934) |
| arch/arm64/include/uapi/asm/kvm.h | group 定义 (:426)、MPIDR/INSTR 掩码 (:417-422) |
| arch/arm64/kvm/vgic/vgic-kvm-device.c | set/get/has_attr (:597/:611/:625)、attr_regs_access (:507)、parse_attr (:473) |
| arch/arm64/kvm/vgic-sys-reg-v3.c | gic_v3_icc_reg_descs (:300)、attr_to_id (:333)、cpu_sysregs_uaccess (:351) |
| arch/arm64/kvm/vgic/vgic-mmio-v3.c | has_attr_regs (:1069) |
| arch/arm64/kvm/sys_regs.c | kvm_sys_reg_get/set_user (:3828/:3864)，被复用的核心 |
| qemu/hw/intc/arm_gicv3_kvm.c | 编码宏 (:45/:84)、kvm_gicc_access (:103)、get/put (:502/:317) |
