# KVM 寄存器 ID 的 size 编码与 QEMU 寄存器宽度判定

**问题**：QEMU 通过 `KVM_GET_ONE_REG` / `KVM_SET_ONE_REG` 读写寄存器时，如何确定该
寄存器是 32 位还是 64 位？有没有专门的接口查询这个信息？

**结论**：没有单独的查询接口。宽度直接编码在寄存器 ID 的 bit[54:52]（`KVM_REG_SIZE`
字段）。`KVM_GET_REG_LIST` 返回的每个 reg id 都自带该字段，QEMU 用
`id & KVM_REG_SIZE_MASK` 解码出宽度，再按宽度调用 ONE_REG。

## 目录

1. [UAPI：size 编码在 reg id 里](#1-uapisize-编码在-reg-id-里)
2. [id 构造方：size 位由谁填](#2-id-构造方size-位由谁填)
3. [QEMU 侧：判定宽度的调用栈](#3-qemu-侧判定宽度的调用栈)
4. [内核侧：GET/SET_ONE_REG 处理栈](#4-内核侧getset_one_reg-处理栈)
5. [要点](#5-要点)

---

## 1. UAPI：size 编码在 reg id 里

`include/uapi/linux/kvm.h`：

```c
#define KVM_REG_SIZE_SHIFT	52
#define KVM_REG_SIZE_MASK	0x00f0000000000000ULL
#define KVM_REG_SIZE(id)	\
	(1U << (((id) & KVM_REG_SIZE_MASK) >> KVM_REG_SIZE_SHIFT))

#define KVM_REG_SIZE_U32	0x0020000000000000ULL
#define KVM_REG_SIZE_U64	0x0030000000000000ULL
```

bit[54:52] 的数值 n 表示寄存器宽度为 2^n 字节（U32=2，U64=3，最大 U2048）。
QEMU 侧的副本在 `linux-headers/linux/kvm.h:1104-1107`。

## 2. id 构造方：size 位由谁填

**内核构造 reg id 时就把 size 填好了**（KVM_GET_REG_LIST 返回给用户态）：

```c
/* arch/arm64/kvm/sys_regs.c:3920 — 系统寄存器固定 U64 */
static u64 sys_reg_to_index(const struct sys_reg_desc *reg)
{
	return (KVM_REG_ARM64 | KVM_REG_SIZE_U64 |
		KVM_REG_ARM64_SYSREG |
		(reg->Op0 << KVM_REG_ARM64_SYSREG_OP0_SHIFT) | ...);
}

/* arch/arm64/kvm/sys_regs.c:3906 — demux 寄存器(CCSIDR)是 U32 */
static int write_demux_regids(struct kvm_vcpu *vcpu, u64 __user *uindices)
{
	u64 val = KVM_REG_ARM64 | KVM_REG_SIZE_U32 | KVM_REG_ARM_DEMUX;
	...
}
```

AArch32 的 CP15 寄存器同理走 U32。

**QEMU 自己构造 id 时也显式带 size**（`linux-headers/asm-arm64/kvm.h:255`）：

```c
#define ARM64_SYS_REG(...) (__ARM64_SYS_REG(__VA_ARGS__) | KVM_REG_SIZE_U64)
```

## 3. QEMU 侧：判定宽度的调用栈

### 3.1 初始化：KVM_GET_REG_LIST

```
kvm_arch_init (target/arm/kvm.c)
  └─ kvm_arm_init_cpreg_list                      kvm.c:825
       ├─ ioctl(cs, KVM_GET_REG_LIST)             先 n=0 拿数量，再拿全量
       ├─ qsort 按 id 排序（cpreg_tuples 要求严格升序）
       └─ for each reg id:
            ├─ kvm_arm_reg_syncs_via_cpreg_list    kvm.c:804
            │    CORE/VFP 走别的同步通道，过滤掉
            └─ switch (reg & KVM_REG_SIZE_MASK) {  kvm.c:852  ← 宽度判定
                 case KVM_REG_SIZE_U32:
                 case KVM_REG_SIZE_U64:  break;
                 default: 报错 "Can't handle size..." 返回 -EINVAL
               }
            → 存入 cpreg_indexes[]（id）、cpreg_values[]（值）
```

### 3.2 读：write_kvmstate_to_list (kvm.c:920)

```
kvm_arch_get_registers
  └─ write_kvmstate_to_list
       for i in cpreg_array_len:
         regidx = cpreg_indexes[i]
         switch (regidx & KVM_REG_SIZE_MASK)        kvm.c:931  ← 再次解码
           case KVM_REG_SIZE_U32:
             kvm_get_one_reg(cs, regidx, &v32);     ← 局部 uint32_t
             cpreg_values[i] = v32;
           case KVM_REG_SIZE_U64:
             kvm_get_one_reg(cs, regidx, &cpreg_values[i]);  ← uint64_t 槽位
               └─ ioctl(vcpu_fd, KVM_GET_ONE_REG, &reg)
```

### 3.3 写：write_list_to_kvmstate (kvm.c:1003)

对称实现，`switch (regidx & KVM_REG_SIZE_MASK)` (kvm.c:1018)，U32 用局部变量、
U64 直接传槽位指针，调 `KVM_SET_ONE_REG`。

另外探针类函数里有断言保护（kvm.c:195-215 `read_sys_reg32/64`）：

```c
assert((id & KVM_REG_SIZE_MASK) == KVM_REG_SIZE_U64);
```

## 4. 内核侧：GET/SET_ONE_REG 处理栈

```
KVM_GET_ONE_REG ioctl
  └─ kvm_arm_sys_reg_get_reg              arch/arm64/kvm/sys_regs.c:3853
       ├─ demux_c15_get  (COPROC == KVM_REG_ARM_DEMUX 时)      :3773
       │    KVM_REG_SIZE(id) != 4 → -ENOENT；u32 读写 CCSIDR
       └─ kvm_sys_reg_get_user                                :3828
            ├─ id_to_sys_reg_desc  校验 COPROC == ARM64_SYSREG
            │    └─ get_reg_by_id → index_to_params            :3713
            │         switch (id & KVM_REG_SIZE_MASK)  ← 只收 U64，
            │         其余返回 false → 调用方 -ENOENT
            └─ put_user(val, uaddr)  固定按 u64 拷贝

KVM_SET_ONE_REG 对称：
  kvm_arm_sys_reg_set_reg → kvm_sys_reg_set_user               :3892 / :3864
```

内核侧对 size 位**不是只用不管，而是校验**：`index_to_params()` 对 sysreg 只接受
`KVM_REG_SIZE_U64`，size 位不匹配整个 id 视为无效 → `-ENOENT`。所以 size 位是
ABI 的一部分，不是纯信息性的。

## 5. 要点

- **完整链路**：内核在 `sys_reg_to_index`/`write_demux_regids` 里把宽度编码进 id →
  QEMU 用 `KVM_GET_REG_LIST` 拿到所有 id → 用 `id & KVM_REG_SIZE_MASK` 解出宽度 →
  按宽度调用 `KVM_GET/SET_ONE_REG`。不需要也不存在单独的查询接口。
- **size 位表示传输字节数**，不是寄存器的逻辑宽度。同一个逻辑寄存器的 32 位视图
  （AArch32 CP15 访问）与 64 位视图（AArch64 sysreg 访问）靠不同的 coproc 字段区分
  （`KVM_REG_ARM_CP15` vs `KVM_REG_ARM64_SYSREG`），即不同的 id。
- VGIC 的 device attr 通道（`KVM_DEV_ARM_VGIC_GRP_CPU_SYSREGS`）不适用上述判定，
  固定按 u64 传输，见 [[vgic_cpu_sysregs_ioctl]]。

## 关键文件

| 文件 | 内容 |
|---|---|
| include/uapi/linux/kvm.h | KVM_REG_SIZE_SHIFT/MASK、KVM_REG_SIZE(id) 宏 |
| arch/arm64/kvm/sys_regs.c | sys_reg_to_index (:3920)、index_to_params (:3713)、kvm_sys_reg_get/set_user (:3828/:3864) |
| qemu/target/arm/kvm.c | kvm_arm_init_cpreg_list (:825)、write_kvmstate_to_list (:920)、write_list_to_kvmstate (:1003) |
| qemu/linux-headers/asm-arm64/kvm.h | ARM64_SYS_REG 带 KVM_REG_SIZE_U64 (:255) |
