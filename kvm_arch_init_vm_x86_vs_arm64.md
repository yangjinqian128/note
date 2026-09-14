# kvm_arch_init_vm / create VM 笔记（openEuler OLK-6.6）

> 基准：openEuler 内核 `/home/code/kernel`（OLK-6.6）；主线 `/home/code/atomgit/linux`（v7.2-12253-g3d44dd2ac2e1b）

---

# 第一章　openEuler 中 x86 vs arm64 的 kvm_arch_init_vm

## 1.1 公共入口

两架构共用同一入口，用户态通过 `ioctl(KVM_CREATE_VM)` 触发：

```
ioctl(fd, KVM_CREATE_VM, type)                      [用户态]
  └─ kvm_dev_ioctl_create_vm    virt/kvm/kvm_main.c:5698
       └─ kvm_create_vm          virt/kvm/kvm_main.c:1228
            ├─ ...（memslots、io bus 等架构无关初始化）
            └─ kvm_arch_init_vm   virt/kvm/kvm_main.c:1304   ← 架构分叉点
```

- `type` 从用户态原样传到 `kvm_arch_init_vm(kvm, type)`，但两架构对 type 的语义完全不同（见 1.4）。
- 失败处理：`kvm_create_vm` 跳 `out_err_no_arch_destroy_vm`（kvm_main.c:1363），该路径只调 `kvm_arch_free_vm`，**不调 `kvm_arch_destroy_vm`**。因此 `kvm_arch_init_vm` 内部错误路径必须自己回滚全部资源——x86 做到了，arm64 有一处漏洞（见 1.5）。

## 1.2 x86 调用栈（arch/x86/kvm/x86.c:12714）

```
kvm_arch_init_vm(kvm, type)
├─ kvm_is_vm_type_supported(type)                失败 → -EINVAL（所有分配之前，安全）
├─ kvm->arch_ext.vm_type = type
│    arch_ext.has_private_mem  = (type == KVM_X86_SW_PROTECTED_VM)   ← 软件保护 VM
│    arch_ext.pre_fault_allowed = (DEFAULT_VM || SW_PROTECTED_VM)
├─ kvm_page_track_init          mmu/page_track.c:147   脏页跟踪（dirty logging 基础设施）
├─ kvm_mmu_init_vm              mmu/mmu.c:6417         MMU 结构初始化（void）
├─ static_call(kvm_x86_vm_init)                        ← KVM_X86_OP(vm_init)（kvm-x86-ops.h:23）
│    ├─ vmx_vm_init             vmx/vmx.c:7600         Intel
│    └─ svm_vm_init             svm/svm.c（.vm_init 挂载点 svm.c:5215）  AMD
├─ INIT_HLIST_HEAD(mask_notifier_list) / INIT_LIST_HEAD(assigned_dev_head)
│    atomic_set(noncoherent_dma_count, 0)
├─ irq_sources_bitmap 预留两个位：KVM_USERSPACE_IRQ_SOURCE_ID、KVM_IRQFD_RESAMPLE_IRQ_SOURCE_ID
├─ TSC/kvmclock 体系：
│    tsc_write_lock / apic_map_lock / pvclock seqcount
│    kvmclock_offset = -get_kvmclock_base_ns() → pvclock_update_vm_gtod_copy(kvm)
│    default_tsc_khz = max_tsc_khz ?: tsc_khz
├─ guest_can_read_msr_platform_info = true;  enable_pmu = enable_pmu
├─ [CONFIG_HYPERV] hv_root_tdp_lock / hv_root_tdp = INVALID_PAGE
├─ INIT_DELAYED_WORK(kvmclock_update_work / kvmclock_sync_work)
├─ kvm_apicv_init               x86.c:10094 (static)   APICv/AVIC 能力初始化
├─ kvm_hv_init_vm               hyperv.c:2650
└─ kvm_xen_init_vm              xen.c:2124
   返回 0

错误路径（严格反向回滚）：
  out_uninit_mmu: kvm_mmu_uninit_vm → kvm_page_track_cleanup → out
```

**风格**：偏向"CPU 虚拟化能力 + 时钟 + 厂商扩展"的组装——厂商差异（VMX/SVM）经 `static_call` 接入；中断（APICv）只做初始化，真正的 irqchip 在之后的 `KVM_CREATE_IRQCHIP` ioctl 才创建。

## 1.3 arm64 调用栈（arch/arm64/kvm/arm.c:326）

```
kvm_arch_init_vm(kvm, type)
├─ kvm_sched_affinity_vm_init   hisilicon/hisi_virt.c:644  ← openEuler DVMBM 调度亲和性
│    （kvm_dvmbm_support 开时：sched_lock + 分配 sched_cpus cpumask）
├─ mutex_init(config_lock)；[LOCKDEP] 声明 config_lock 在 kvm->lock 之内
├─ type 检查：type & ~(ARM_MASK | IPA_SIZE_MASK) → -EINVAL
├─ switch(type & ARM_MASK)：NORMAL 直通 / REALM 要求 kvm_rme_is_available 并置 is_realm / 其他 -EINVAL
├─ kvm_share_hyp(kvm, kvm+1)    mmu.c:529    与 hyp(EL2) 共享 struct kvm 所在页
├─ pkvm_init_host_vm            pkvm.c:223   pKVM 宿主 VM 记录
├─ zalloc_cpumask_var(supported_cpus) = cpu_possible_mask
├─ kvm_init_stage2_mmu          mmu.c:912    Stage-2 页表（IPA size 取自 type！）
├─ kvm_vgic_early_init          vgic/vgic-init.c:58
├─ kvm_timer_init_vm            arch_timer.c:1266
├─ max_vcpus 由宿主 GIC 型号决定
├─ [FEAT_NMI && !vgic_v3_cpuif_trap] pfr1_nmi = NMI_IMP
├─ kvm_arm_init_hypercalls      hypercalls.c:525
├─ [!realm] kvm_arm_timer_early_inject_vm_init   pvtimer_early.c:76
├─ bitmap_zero(vcpu_features)
└─ [realm] kvm_init_realm_vm    cca_base.c:65
   返回 0

错误路径（反向回滚）：
  err_free_cpumask: free supported_cpus
  err_unshare_kvm:  kvm_unshare_hyp
```

**风格**：偏向"系统级硬件资源"——GIC/timer/Stage-2 都是每 VM 一份的物理资源抽象，早期初始化就在 init_vm 完成；叠加 hyp 共享、pKVM、Realm 等安全世界步骤。

## 1.4 关键差异

| 维度 | x86 | arm64 |
|---|---|---|
| type 语义 | 纯 VM 类型枚举（DEFAULT / SW_PROTECTED） | 类型位(ARM_MASK) + **IPA size 位**(IPA_SIZE_MASK) 组合 |
| 厂商扩展机制 | `static_call(kvm_x86_vm_init)` 分发 VMX/SVM | 无厂商分发；有 Realm(RME) 分支 |
| MMU | `kvm_mmu_init_vm`（常规 EPT/NPT） | `kvm_init_stage2_mmu` + `kvm_share_hyp` + pKVM |
| 中断 | init_vm 只预留 irq source 位，irqchip 在后续 ioctl 创建 | init_vm 内即做 vGIC/timer 早期初始化 |
| 时钟 | TSC/pvclock/kvmclock 大量初始化 + 2 个 delayed work | 无（arch timer 子系统内部处理） |
| openEuler 特有 | arch_ext、assigned_dev_head、mask_notifier、hv/xen init | DVMBM 调度亲和性、Realm、NMI、IPA-in-type |

## 1.5 观察：arm64 错误路径的一处资源泄漏隐患

`kvm_arch_init_vm`（arm.c）中，`kvm_sched_affinity_vm_init` 成功后，若**类型检查失败**（arm.c:344）或 **`kvm_share_hyp` 失败**会直接 return；而通用代码 `out_err_no_arch_destroy_vm` 不调 `kvm_arch_destroy_vm`，于是 `sched_cpus` cpumask 泄漏。仅发生在错误路径 + `kvm_dvmbm_support` 开启时。x86 无此问题（类型检查在所有分配之前）；mainline arm64 也无此问题（没有 `kvm_sched_affinity_vm_init` 这一步），属 openEuler 合入 DVMBM 特性时引入的微小不一致。

---

# 第二章　openEuler(OLK-6.6) vs 主线(v7.2)：x86 create VM 对比

## 2.1 通用层 kvm_create_vm（virt/kvm/kvm_main.c）

两条路径骨架一致（`kvm_arch_alloc_vm` → memslots/io bus → `kvm_arch_init_vm` → `hardware_enable_all` → mmu notifier → debugfs）。差异有两处：

| 步骤 | openEuler OLK-6.6 | 主线 v7.2 |
|---|---|---|
| `kvm_arch_post_init_vm` | 有：weak 定义 kvm_main.c:1188，kvm_create_vm 中调用（kvm_main.c:1334）；x86 实现即 `kvm_mmu_post_init_vm` | **无**：`kvm_mmu_post_init_vm` 挪到 `kvm_arch_vcpu_ioctl_run` 开头（x86.c:11963），首次运行 vCPU 时才做 MMU post-init |
| `kvm_create_shadow` | 有（kvm_main.c:1212）：分配 `kvm_shadow`（`ioeventfds_shadow` 链表 + `in_shadow` 标志），支撑 ioeventfd 批量/影子机制——`kvm_ioeventfd()` 支持 `KVM_IOEVENTFD_FLAG_BATCH_BEGIN/END`（eventfd.c:1238 起），begin 置 `in_shadow=true`，期间的 ioeventfd 挂到影子链表，end 时统一释放 | **无** |

主线把"与 vCPU 运行相关的初始化"从 create_vm 路径后移到首次 vcpu run，create_vm 更轻；openEuler 则保留了 VM 创建期完成全部初始化的老结构，并叠加自己的 shadow/批量机制。

## 2.2 x86 kvm_arch_init_vm 逐项对比

主线 v7.2 版本（arch/x86/kvm/x86.c:13276）：

```
kvm_arch_init_vm(kvm, type)
├─ kvm_is_vm_type_supported(type)                失败 → -EINVAL
├─ kvm->arch.vm_type = type                       ← 存 kvm_arch，非 arch_ext
│    kvm->arch.has_private_mem  = (type == KVM_X86_SW_PROTECTED_VM)
│    kvm->arch.pre_fault_allowed = (DEFAULT_VM || SW_PROTECTED_VM)
│    kvm->arch.disabled_quirks = kvm_caps.inapplicable_quirks & kvm_caps.supported_quirks  ← 新增
├─ kvm_page_track_init
├─ kvm_mmu_init_vm                                ← 现在返回 int，失败 → out_cleanup_page_track
├─ kvm_x86_call(vm_init)                          ← static_call 宏改名（宏定义 arch/x86/include/asm/kvm_host.h）
├─ atomic_set(noncoherent_dma_count, 0)           ← 无 mask_notifier_list / assigned_dev_head / irq 位预留
├─ TSC/kvmclock：tsc_write_lock、apic_map_lock、pvclock seqcount、kvmclock_offset、
│    ratelimit_state_init(kvmclock_update_rs, HZ, 10)   ← 取代两个 delayed work
│    pvclock_update_vm_gtod_copy
├─ default_tsc_khz；apic_bus_cycle_ns = APIC_BUS_CYCLE_NS_DEFAULT   ← 新增
├─ guest_can_read_msr_platform_info = true
│    enable_pmu = enable_pmu && !kvm->arch.has_protected_pmu        ← 新增 protected_pmu 保护
├─ [CONFIG_HYPERV] hv_root_tdp_lock
├─ kvm_apicv_init / kvm_hv_init_vm / kvm_xen_init_vm               ← 与 openEuler 相同
├─ ignore_msrs 不合法配置告警                                          ← 新增
└─ once_init(&kvm->arch.nx_once)                                     ← 新增
```

**差异清单**（openEuler → 主线 v7.2）：

1. **字段存放位置**：openEuler 把 `vm_type` / `has_private_mem` / `pre_fault_allowed` 放在 `kvm->arch_ext`（KABI 扩展结构，定义 arch/x86/include/asm/kvm_host.h:1286，注释说明挂在 struct kvm 尾部）；主线直接放 `kvm->arch`。openEuler 出于内核 ABI 兼容约束新增字段必须走 arch_ext 通道。
2. **KVM_X86_SW_PROTECTED_VM**：⚠ 两边都有（值都是 1，arch/x86/include/uapi/asm/kvm.h，openEuler 为 916 行、主线 971 行）——openEuler 是 6.6 上的 backport，**不是差异点**。
3. **disabled_quirks**（主线新增）：per-VM quirks 能力位（`kvm_caps.supported_quirks` / `inapplicable_quirks`），openEuler 无。
4. **kvm_mmu_init_vm 签名**：主线改为返回 int 并新增 `out_cleanup_page_track` 错误路径；openEuler 为 void。
5. **static_call 宏演进**：openEuler `static_call(kvm_x86_vm_init)`；主线 `kvm_x86_call(vm_init)`（宏定义在 arch/x86/include/asm/kvm_host.h）——同一机制，命名/封装演进。
6. **mask_notifier_list**：openEuler 挂在 `kvm->arch`（per-VM，init_vm 中 `INIT_HLIST_HEAD`，irq_comm.c 使用）；主线 v7.2 挂在 ioapic 上（`kvm->arch.vioapic->mask_notifier_list`，ioapic.c:296 注册）。
7. **assigned_dev_head**：openEuler 保留（legacy 设备分配链表）；主线已无。
8. **irq_sources_bitmap 机制**：openEuler 在 init_vm 中 `set_bit` 预留 userspace / irqfd-resampler 两个 source ID；主线 v7.2 **整个 bitmap 机制已删除**，保留 ID 仅作常量直接传给 `kvm_set_irq`（irqchip.c:70、eventfd.c:55）。
9. **kvmclock delayed work**：openEuler 有 `kvmclock_update_work` + `kvmclock_sync_work` 两个；主线两个都移除，改为 `kvmclock_update_rs` ratelimit（HZ, 10，`RATELIMIT_MSG_ON_RELEASE`）。
10. **apic_bus_cycle_ns**（主线新增）：`APIC_BUS_CYCLE_NS_DEFAULT`，LAPIC timer 换算用。
11. **protected PMU**（主线新增）：`enable_pmu && !has_protected_pmu`。
12. **ignore_msrs 告警 + nx_once**（主线新增）：启动时配置合法性检查、NX 相关一次性初始化。

## 2.3 小结

主线 v7.2 相对 openEuler OLK-6.6 的演进方向：

- **错误处理强化**：`kvm_mmu_init_vm` 带返回值、错误路径完整回滚；
- **按需初始化后移**：MMU post-init 从 create_vm 挪到首次 vcpu run，create_vm 变轻；
- **机制裁剪**：删除 irq_sources_bitmap、两个 kvmclock work、legacy assigned_dev；
- **结构收敛**：新字段直接进 `kvm_arch` 而非扩展结构。

openEuler 侧的特点是：KABI 约束下新字段走 `arch_ext`；并保留了主线已裁剪的机制（mask_notifier per-VM、assigned_dev、irq 位预留），以及自己的增量特性（`kvm_create_shadow` / ioeventfd batch、arch_ext、DVMBM 等）。
