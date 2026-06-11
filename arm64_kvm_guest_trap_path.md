# ARM64 KVM Guest 系统寄存器访问 Trap 全路径

本文档详细追踪 guest 执行 `MRS/MSR` 访问系统寄存器时，从硬件 trap 到 KVM 执行
access 回调的完整路径。

## 目录

1. [整体概览](#1-整体概览)
2. [VM entry：KVM 写入 HCR_EL2](#2-vm-entrykvm-写入-hcrel2)
3. [Guest 执行 MRS → 硬件 trap → EL2](#3-guest-执行-mrs-硬件-trap-el2)
4. [汇编异常向量：保存 guest GPRs](#4-汇编异常向量保存-guest-gprs)
5. [两级 handler 分发](#5-两级-handler-分发)
6. [解码 ESR → emulate → access 回调](#6-解码-esr-emulate-access-回调)
7. [回到 guest](#7-回到-guest)
8. [关键文件](#8-关键文件)

---

## 1. 整体概览

以 guest 执行 `MRS x0, SCTLR_EL1` 为例：

```
guest:  MRS x0, SCTLR_EL1
          │
          ▼  硬件 trap → EL2
┌─────────────────────────────────────────────────┐
│ EL2 异常向量 (hyp-entry.S)                       │
│   el1_sync → el1_trap → __guest_exit             │
│   保存 X0~X29 到 vcpu->arch.ctxt (entry.S)       │
│   read_sysreg_el2(ESR) → vcpu->arch.fault.esr_el2│
│                                                  │
│   第一级: hyp_exit_handlers[0x18]                 │ ← EL2 内处理
│   → kvm_hyp_handle_sysreg（简单情况直接回 guest） │
│   → 未处理 → 退出到 host                          │
│                                                  │
│   第二级: arm_exit_handlers[0x18]                 │ ← host 内核处理
│   → kvm_handle_sys_reg                           │
│       → esr_sys64_to_params(esr)                  │
│       → emulate_sys_reg → find_reg(二分查找)      │
│       → perform_access → r->access()              │
│       → vcpu_set_reg(X0, regval)                  │
│       → kvm_incr_pc                               │
│                                                  │
│   __guest_enter → 恢复 GPRs → eret                │
└─────────────────────────────────────────────────┘
          │
          ▼
guest:  X0 = SCTLR_EL1 的值，继续执行下一条指令
```

总开销：一次完整 VM exit + 两级 handler 分发 + 二分查找 + access 回调 + VM entry。
硬件 trap/保存/恢复占大头（几百 cycle），KVM 软件处理只占小部分。

---

## 2. VM entry：KVM 写入 HCR_EL2

每次进入 guest 前，KVM 把 `vcpu->arch.hcr_el2` 写入物理寄存器，硬件从此按这些
trap 位监控 guest 的每条指令。

`arch/arm64/kvm/hyp/include/hyp/switch.h`：

```c
static inline void ___activate_traps(struct kvm_vcpu *vcpu)
{
    write_sysreg(vcpu->arch.hcr_el2, hcr_el2);
    ...
}
```

`vcpu->arch.hcr_el2` 的值由 `vcpu_reset_hcr`（`arch/arm64/include/asm/kvm_emulate.h:70`）设置：

```c
static inline void vcpu_reset_hcr(struct kvm_vcpu *vcpu)
{
    vcpu->arch.hcr_el2 = HCR_GUEST_FLAGS;
    if (has_vhe()) vcpu->arch.hcr_el2 |= HCR_E2H;
    ...
    if (!has_fwb) vcpu->arch.hcr_el2 |= HCR_TVM;  // TVM = trap SCTLR_EL1 等
    if (vcpu_el1_is_32bit(vcpu))
        vcpu->arch.hcr_el2 &= ~HCR_RW;
    ...
}
```

---

## 3. Guest 执行 MRS → 硬件 trap → EL2

当 guest 执行 `MRS x0, SCTLR_EL1`，CPU 硬件完成以下步骤：

```
1. 解码指令，发现访问系统寄存器 (Op0=3,Op1=0,CRn=1,CRm=0,Op2=0)
2. 查 HCR_EL2.TVM = 1 → 该 trap
3. 保存 guest 现场：
   PSTATE  → SPSR_EL2       (guest 的处理器状态)
   PC      → ELR_EL2         (MRS 指令的地址，用于返回)
4. 填写 ESR_EL2 (Exception Syndrome Register)：
   ┌──────┬──────────────────────────────────────┐
   │ EC   │ 0x18 = ESR_ELx_EC_SYS64              │  ← 异常类别
   │ ISS  │ Op0=3, Op1=0, CRn=1, CRm=0, Op2=0   │  ← 寄存器编码
   │      │ 方向 bit0=1 = 读(MRS)                │  ← 读写方向
   │      │ Rt=0 = 目标寄存器 X0                  │  ← 目标 GPR
   └──────┴──────────────────────────────────────┘
5. 切换到 EL2
6. 跳转到 VBAR_EL2 指向的异常向量表
```

**关键设计：硬件把目标寄存器的 Op0/Op1/CRn/CRm/Op2 编码进了 ESR_EL2 的 ISS 字段，
KVM 不需要查指令、不需要反汇编，直接从 ESR 解码就能知道 guest 访问了哪个寄存器。**

---

## 4. 汇编异常向量：保存 guest GPRs

`arch/arm64/kvm/hyp/hyp-entry.S:181` — EL2 异常向量表：

```asm
SYM_CODE_START(__kvm_hyp_vector)
    valid_vect  el1_sync       // 64-bit EL1 同步异常入口 ← guest trap 到这
    ...
```

`arch/arm64/kvm/hyp/hyp-entry.S:44`：

```asm
el1_sync:                           // Guest 同步异常入口
    mrs     x0, esr_el2             // 读 ESR_EL2
    ubfx    x0, x0, #ESR_ELx_EC_SHIFT, #ESR_ELx_EC_WIDTH
    cmp     x0, #ESR_ELx_EC_HVC64   // 是 HVC 调用吗？
    ccmp    x0, #ESR_ELx_EC_HVC32, #4, ne
    b.ne    el1_trap                // 不是 HVC → el1_trap

el1_trap:
    get_vcpu_ptr x1, x0
    mov     x0, #ARM_EXCEPTION_TRAP   // exit_code = ARM_EXCEPTION_TRAP
    b       __guest_exit
```

`arch/arm64/kvm/hyp/entry.S:108` — 保存 guest GPRs 到 vcpu 结构体：

```asm
__guest_exit:
    // x0: exit_code
    // x1: vcpu
    // x2-x29,lr: guest 寄存器（硬件在 trap 时已保存到这些影子寄存器）

    add     x1, x1, #VCPU_CONTEXT

    // X2~X17 存入 vcpu
    stp     x2, x3,   [x1, #CPU_XREG_OFFSET(2)]
    stp     x4, x5,   [x1, #CPU_XREG_OFFSET(4)]
    ...
    // X0~X1 在栈上，取回并保存
    ldp     x2, x3, [sp], #16
    stp     x2, x3,   [x1, #CPU_XREG_OFFSET(0)]

    save_callee_saved_regs x1     // X18~X29, lr
    ret                           // 返回 C 代码
```

执行完后，guest 所有 GPRs 已保存在 `vcpu->arch.ctxt.regs` 中。

---

## 5. 两级 handler 分发

KVM 对系统寄存器 trap 采用了**两级分发**策略。

### 5.1 第一级：Hyp 层（EL2 内处理）

`arch/arm64/kvm/hyp/include/hyp/switch.h:685`：

```c
static inline bool __fixup_guest_exit(struct kvm_vcpu *vcpu, u64 *exit_code, ...)
{
    // 从硬件 ESR_EL2 读到 vcpu 结构体
    if (ARM_EXCEPTION_CODE(*exit_code) != ARM_EXCEPTION_IRQ)
        vcpu->arch.fault.esr_el2 = read_sysreg_el2(SYS_ESR);

    // 先尝试在 EL2 直接处理
    if (kvm_hyp_handle_exit(vcpu, exit_code, handlers))
        goto guest;    // 处理完了，直接回 guest，不需要退出到 host

    return false;      // 处理不了，退出到 host 内核
}
```

`arch/arm64/kvm/hyp/vhe/switch.c:163`：

```c
static const exit_handler_fn hyp_exit_handlers[] = {
    [ESR_ELx_EC_SYS64]  = kvm_hyp_handle_sysreg,   // VGIC CPU IF、CNTPCT 等
    [ESR_ELx_EC_FP_ASIMD] = kvm_hyp_handle_fpsimd,
    [ESR_ELx_EC_IABT_LOW] = kvm_hyp_handle_iabt_low,
    [ESR_ELx_EC_DABT_LOW] = kvm_hyp_handle_dabt_low,
    ...
};
```

Hyp 层的 `kvm_hyp_handle_sysreg` 只处理少数可以在 EL2 直接完成、不需要 host 参与的
简单情况（如 CNTPCT 读取、VGIC CPU interface）。SCTLR_EL1 不在此列，返回 false。

**设计目的：** 对于高频、简单的 trap，在 EL2 处理完直接回 guest，省掉一次完整的
EL2→EL1 host→EL2 来回的上下文切换。

### 5.2 第二级：Host 内核层

`arch/arm64/kvm/arm.c:1253`：

```c
int kvm_arch_vcpu_ioctl_run(struct kvm_vcpu *vcpu)
{
    while (ret > 0) {
        ret = kvm_arm_vcpu_enter_exit(vcpu);  // 进入 guest，返回 exit_code
        ret = handle_exit(vcpu, ret);          // 在 host 内核处理
    }
}
```

`arch/arm64/kvm/handle_exit.c:372`：

```c
int handle_exit(struct kvm_vcpu *vcpu, int exception_index)
{
    switch (ARM_EXCEPTION_CODE(exception_index)) {
    case ARM_EXCEPTION_TRAP:
        return handle_trap_exceptions(vcpu);   // 系统寄存器 trap
    ...
    }
}
```

`arch/arm64/kvm/handle_exit.c:259` — host 层的分发表：

```c
static exit_handle_fn arm_exit_handlers[] = {
    [ESR_ELx_EC_SYS64]  = kvm_handle_sys_reg,      // ← MRS/MSR 最终在这里
    [ESR_ELx_EC_SVE]    = handle_sve,
    [ESR_ELx_EC_FP_ASIMD] = handle_no_fpsimd,
    ...
};
```

---

## 6. 解码 ESR → emulate → access 回调

`arch/arm64/kvm/sys_regs.c:3683`：

```c
int kvm_handle_sys_reg(struct kvm_vcpu *vcpu)
{
    struct sys_reg_params params;
    unsigned long esr = kvm_vcpu_get_esr(vcpu);   // 即 vcpu->arch.fault.esr_el2
    int Rt = kvm_vcpu_sys_get_rt(vcpu);            // 从 ESR 提取目标寄存器号

    trace_kvm_handle_sys_reg(esr);
    vcpu->stat.sys64_exit_stat++;

    if (__check_nv_sr_forward(vcpu))               // 嵌套虚拟化：可能转发给 L1
        return 1;

    params = esr_sys64_to_params(esr);             // 解码 ESR → 5 字段
    params.regval = vcpu_get_reg(vcpu, Rt);        // 读目标 GPR（MSR 场景需要）

    if (!emulate_sys_reg(vcpu, &params))
        return 1;

    if (!params.is_write)
        vcpu_set_reg(vcpu, Rt, params.regval);     // MRS：结果写回 GPR
    return 1;
}
```

`esr_sys64_to_params`（`arch/arm64/kvm/sys_regs.h:37`）直接从 ESR 的 ISS 字段
解码出寄存器五元组：

```c
#define esr_sys64_to_params(esr)                       \
    ((struct sys_reg_params){                          \
        .Op0 = ((esr) >> 20) & 3,                      \
        .Op1 = ((esr) >> 14) & 0x7,                    \
        .CRn = ((esr) >> 10) & 0xf,                    \
        .CRm = ((esr) >>  1) & 0xf,                    \
        .Op2 = ((esr) >> 17) & 0x7,                    \
        .is_write = !((esr) & 1) })   // bit0: 0=写(MSR), 1=读(MRS)
```

`emulate_sys_reg`（`sys_regs.c:3607`）在 `sys_reg_descs[]` 表中二分查找并执行回调：

```c
static bool emulate_sys_reg(struct kvm_vcpu *vcpu,
                           struct sys_reg_params *params)
{
    const struct sys_reg_desc *r;
    r = find_reg(params, sys_reg_descs, ARRAY_SIZE(sys_reg_descs));  // 二分查找

    if (likely(r)) {
        perform_access(vcpu, params, r);
        return true;
    }
    kvm_inject_undefined(vcpu);   // 找不到 → 注入未定义异常
    return false;
}
```

`find_reg`（`sys_regs.h:222`）用 `bsearch` 二分查找：

```c
static inline const struct sys_reg_desc *
find_reg(const struct sys_reg_params *p, const struct sys_reg_desc table[], unsigned int num)
{
    unsigned long pval = reg_to_encoding(p);  // 把 params 拼回完整编码
    return __inline_bsearch((void *)pval, table, num, sizeof(table[0]), match_sys_reg);
}
```

`perform_access`（`sys_regs.c:3276`）：

```c
static void perform_access(struct kvm_vcpu *vcpu, struct sys_reg_params *params,
                           const struct sys_reg_desc *r)
{
    trace_kvm_sys_access(*vcpu_pc(vcpu), params, r);

    if (sysreg_hidden(vcpu, r)) {              // 运行时检查：寄存器是否被隐藏
        kvm_inject_undefined(vcpu);
        return;
    }

    BUG_ON(!r->access);

    if (likely(r->access(vcpu, params, r)))    // 调用注册的 access 回调
        kvm_incr_pc(vcpu);                      // 成功 → PC += 4
}
```

---

## 7. 回到 guest

KVM 处理完毕后，`kvm_arch_vcpu_ioctl_run` 的循环继续，再次调用 `kvm_arm_vcpu_enter_exit`
→ `__guest_enter`：

```asm
__guest_enter:
    // 从 vcpu->arch.ctxt 恢复 guest GPRs 到硬件寄存器
    // 此时 X0 = SCTLR_EL1 的值（由 kvm_handle_sys_reg 写入）
    // PC  = ELR_EL2（指向 MRS 的下一条指令）
    eret   // 回到 guest
```

guest 继续执行，X0 拿到了 "SCTLR_EL1" 的值（实际上来自 sys_regs[]）。

---

## 8. 关键文件

| 文件 | 角色 |
|------|------|
| `arch/arm64/kvm/hyp/hyp-entry.S` | EL2 异常向量表 `__kvm_hyp_vector`，同步异常入口 `el1_sync` |
| `arch/arm64/kvm/hyp/entry.S` | `__guest_enter` / `__guest_exit`：汇编保存/恢复 GPRs |
| `arch/arm64/kvm/hyp/vhe/switch.c` | VHE: `fixup_guest_exit`，`hyp_exit_handlers[]` 一级分发表 |
| `arch/arm64/kvm/hyp/nvhe/switch.c` | nVHE: 同上 |
| `arch/arm64/kvm/hyp/include/hyp/switch.h` | `__fixup_guest_exit`，`___activate_traps`（写入 HCR_EL2） |
| `arch/arm64/kvm/handle_exit.c` | `handle_exit`，`arm_exit_handlers[]` 二级分发表 |
| `arch/arm64/kvm/sys_regs.c` | `kvm_handle_sys_reg`，`emulate_sys_reg`，`perform_access` |
| `arch/arm64/kvm/sys_regs.h` | `esr_sys64_to_params()` 解码 ESR，`find_reg()` 二分查找 |
| `arch/arm64/include/asm/kvm_emulate.h` | `kvm_vcpu_get_esr()`，`kvm_vcpu_sys_get_rt()` |
| `arch/arm64/include/asm/esr.h` | ESR_ELx_EC_SYS64 等异常类别定义 |
| `arch/arm64/kvm/arm.c` | `kvm_arch_vcpu_ioctl_run` 主循环 |
