# ARM64 KVM-QEMU 寄存器虚拟化详解

本文档详细介绍 ARM64 架构下 KVM-QEMU 的寄存器虚拟化机制，包括数据结构、初始化流程和修改流程。

## 目录

1. [寄存器 ID 编码](#1-寄存器-id-编码)
2. [数据结构](#2-数据结构)
3. [寄存器初始化流程](#3-寄存器初始化流程)
4. [寄存器同步流程](#4-寄存器同步流程)
5. [寄存器修改流程](#5-寄存器修改流程)
6. [关键代码路径](#6-关键代码路径)

---

## 1. 寄存器 ID 编码

KVM 通过统一的 64 位 ID 来标识各类寄存器，这是用户空间 API 的核心。

### 1.1 寄存器 ID 结构

```
[63:60]  架构标志 (KVM_REG_ARM64 = 0x6)
[59:52]  大小标志 (KVM_REG_SIZE_U64/U32/U128)
[31:16]  COPROC 类型 (寄存器组分类)
[15:0]   寄存器编码
```

### 1.2 COPROC 类型分类

| 类型值 | 名称 | 说明 |
|--------|------|------|
| 0x0010 | KVM_REG_ARM_CORE | 核心寄存器 (x0-x30, sp, pc, pstate, fp_regs) |
| 0x0011 | KVM_REG_ARM_DEMUX | 多值寄存器 (如 CCSIDR) |
| 0x0013 | KVM_REG_ARM64_SYSREG | AArch64 系统寄存器 |
| 0x0014 | KVM_REG_ARM_FW | 固件伪寄存器 (PSCI_VERSION 等) |
| 0x0015 | KVM_REG_ARM64_SVE | SVE 寄存器 |

### 1.3 系统寄存器编码

系统寄存器 ID 格式（定义在 `Documentation/virt/kvm/api.rst`）：

```
0x6030 0000 0013 <op0:2> <op1:3> <crn:4> <crm:4> <op2:3>
```

位域布局：

| 字段 | 移位 | 说明 |
|------|------|------|
| Op0 | 14 | 操作码 0 (通常为 2 或 3) |
| Op1 | 11 | 操作码 1 |
| CRn | 7 | 寄存器号 |
| CRm | 3 | 寄存器修饰符 |
| Op2 | 0 | 操作码 2 |

### 1.4 QEMU 宏定义

```c
// target/arm/kvm.c
#define AARCH64_CORE_REG(x)   (KVM_REG_ARM64 | KVM_REG_SIZE_U64 | \
                 KVM_REG_ARM_CORE | KVM_REG_ARM_CORE_REG(x))

#define AARCH64_SIMD_CORE_REG(x)   (KVM_REG_ARM64 | KVM_REG_SIZE_U128 | \
                 KVM_REG_ARM_CORE | KVM_REG_ARM_CORE_REG(x))

#define AARCH64_SIMD_CTRL_REG(x)   (KVM_REG_ARM64 | KVM_REG_SIZE_U32 | \
                 KVM_REG_ARM_CORE | KVM_REG_ARM_CORE_REG(x))
```

### 1.5 示例

`ID_AA64MMFR4_EL1 (3, 0, 0, 7, 4)` 的 KVM 寄存器 ID：

```c
// 内核内部编码
sys_reg(3, 0, 0, 7, 4)        = 0x001800e0

// KVM 用户空间 API
ARM64_SYS_REG(3, 0, 0, 7, 4)  = 0x603000000000c038
  │                          = KVM_REG_ARM64 | KVM_REG_SIZE_U64 | KVM_REG_ARM64_SYSREG
  │                          | (3 << 14) | (0 << 11) | (0 << 7) | (7 << 3) | 4
```

---

## 2. 数据结构

### 2.1 内核数据结构

#### 2.1.1 kvm_regs - 核心寄存器

定义在 `arch/arm64/include/uapi/asm/kvm.h:51`

```c
struct kvm_regs {
    struct user_pt_regs regs;   /* sp = sp_el0 */
    __u64 sp_el1;
    __u64 elr_el1;
    __u64 spsr[KVM_NR_SPSR];
    struct user_fpsimd_state fp_regs;
};

struct user_pt_regs {
    __u64 regs[31];   /* x0-x30 */
    __u64 sp;         /* sp_el0 */
    __u64 pc;
    __u64 pstate;
};

struct user_fpsimd_state {
    __uint128_t vregs[32];  /* V0-V31 / D0-D31 / Q0-Q31 */
    __u32 fpsr;
    __u32 fpcr;
};
```

#### 2.1.2 kvm_cpu_context - 完整 CPU 上下文

定义在 `arch/arm64/include/asm/kvm_host.h:468`

```c
struct kvm_cpu_context {
    struct user_pt_regs regs;   /* sp = sp_el0 */
    
    u64 spsr_abt;               /* SPSR for ABT mode */
    u64 spsr_und;               /* SPSR for UND mode */
    u64 spsr_irq;               /* SPSR for IRQ mode */
    u64 spsr_fiq;               /* SPSR for FIQ mode */
    
    struct user_fpsimd_state fp_regs;
    
    u64 sys_regs[NR_SYS_REGS];  /* 系统寄存器数组 */
    
    struct kvm_vcpu *__hyp_running_vcpu;
};
```

#### 2.1.3 kvm_vcpu_arch - VCPU 架构状态

定义在 `arch/arm64/include/asm/kvm_host.h:552`

```c
struct kvm_vcpu_arch {
    struct kvm_cpu_context ctxt;    /* 主要寄存器上下文 */
    
    /* 浮点/SVE 状态 */
    void *sve_state;
    enum fp_type fp_type;
    unsigned int sve_max_vl;
    u64 svcr;
    
    /* Stage 2 页表状态 */
    struct kvm_s2_mmu *hw_mmu;
    
    /* Trap 寄存器 */
    u64 hcr_el2;
    u64 mdcr_el2;
    u64 cptr_el2;
    
    /* 调试寄存器 */
    struct kvm_guest_debug_arch *debug_ptr;
    struct kvm_guest_debug_arch vcpu_debug_state;
    struct kvm_guest_debug_arch external_debug_state;
    
    /* VGIC 状态 */
    struct vgic_cpu vgic_cpu;
    
    /* Timer 状态 */
    struct arch_timer_cpu timer_cpu;
    
    /* PMU 状态 */
    struct kvm_pmu pmu;
    
    /* 特性标志 */
    DECLARE_BITMAP(features, KVM_VCPU_MAX_FEATURES)
    
    /* ... 其他字段 */
};
```

### 2.2 QEMU 数据结构

#### 2.2.1 CPUARMState - CPU 状态

定义在 `target/arm/cpu.h:233`

```c
typedef struct CPUArchState {
    /* 32 位模式寄存器 */
    uint32_t regs[16];
    
    /* 64 位模式寄存器 */
    uint64_t xregs[32];  /* X0-X30, X31(SP) */
    uint64_t pc;
    uint32_t pstate;
    bool aarch64;        /* True if in AArch64 mode */
    bool thumb;
    
    /* PSTATE 相关 */
    uint32_t CF, VF, NF, ZF, QF;
    uint64_t daif;
    uint64_t svcr;       /* PSTATE.{SM,ZA} */
    
    /* Banked 寄存器 */
    uint64_t banked_spsr[8];
    uint64_t elr_el[4];  /* ELR_EL1-ELR_EL3 */
    uint64_t sp_el[4];   /* SP_EL0-SP_EL3 */
    
    /* 系统控制寄存器 (cp15) */
    struct {
        uint32_t c0_cpuid;
        uint64_t csselr_el[4];
        uint64_t sctlr_el[4];
        uint64_t ttbr0_el[4];
        uint64_t ttbr1_el[4];
        uint64_t tcr_el[4];
        uint64_t esr_el[4];
        uint64_t far_el[4];
        uint64_t mair_el[4];
        uint64_t vbar_el[4];
        /* ... 更多系统寄存器 */
    } cp15;
    
    /* 浮点/SIMD 寄存器 */
    ARMVectorReg vfp.zregs[32];    /* SVE Z 寄存器 */
    ARMPredicateReg vfp.pregs[17]; /* SVE P 寄存器 + FFR */
    uint32_t vfp.fpsr;
    uint32_t vfp.fpcr;
    
    /* SVE/SME 状态 */
#ifdef TARGET_AARCH64
    ARMPACKey apaci_key;  /* PAC 密钥 */
    ARMPACKey apad_key;
    ARMPACKey apib_key;
    ARMPACKey apid_key;
#endif
    
} CPUARMState;
```

#### 2.2.2 ARMCPU - ARM CPU 对象

定义在 `target/arm/cpu.h`

```c
struct ARMCPU {
    CPUState parent_obj;
    CPUARMState env;
    
    /* KVM 相关 */
    uint32_t kvm_target;
    uint32_t kvm_init_features[7];
    
    /* Coprocessor 寄存器列表 */
    GHashTable *cp_regs;
    uint64_t *cpreg_indexes;
    uint64_t *cpreg_values;
    int cpreg_array_len;
    
    /* MPIDR */
    uint64_t mp_affinity;
    
    /* PSCI 版本 */
    uint32_t psci_version;
    
    /* ... 其他字段 */
};
```

---

## 3. 寄存器初始化流程

初始化流程遵循 **QEMU → KVM ioctl → KVM 内核处理 → 数据结构初始化** 的时序关系。

### 3.1 完整初始化流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                      QEMU 用户空间                               │
│                                                                 │
│  Step 1: 创建 VCPU                                              │
│  └── cpu_create()                                               │
│      └── kvm_init_vcpu()                                        │
│          └── kvm_arch_init_vcpu(CPUState *cs)                  │
│                                                                 │
│  Step 2: 设置初始化特性                                         │
│  └── memset(cpu->kvm_init_features, 0, ...)                    │
│  ├── 设置 POWER_OFF、PSCI_0_2、EL1_32BIT                       │
│  ├── 设置 PMU_V3、SVE、PTRAUTH                                 │
│  └── 构造 struct kvm_vcpu_init                                 │
│                                                                 │
│  Step 3: 调用 KVM 初始化 ioctl                                  │
│  └── kvm_arm_vcpu_init(cpu)                                    │
│      └── kvm_vcpu_ioctl(cs, KVM_ARM_VCPU_INIT, &init)          │
│                                                                 │
│                  ↓ ioctl(KVM_ARM_VCPU_INIT)                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                      KVM 内核空间                               │
│                                                                 │
│  Step 4: ioctl 入口处理                                         │
│  └── kvm_arch_vcpu_ioctl(file, KVM_ARM_VCPU_INIT, arg)         │
│      └── kvm_arch_vcpu_ioctl_vcpu_init(vcpu, &init)           │
│                                                                 │
│  Step 5: VCPU 重置                                              │
│  └── kvm_reset_vcpu(vcpu)                                      │
│      ├── kvm_pmu_vcpu_reset(vcpu)                              │
│      ├── kvm_vcpu_reset_sve(vcpu)                              │
│      ├── memset(vcpu_gp_regs(vcpu), ...)                      │
│      ├── memset(&vcpu->arch.ctxt.fp_regs, ...)                │
│      ├── kvm_reset_sys_regs(vcpu)                              │
│      └── kvm_timer_vcpu_reset(vcpu)                            │
│                                                                 │
│  Step 6: 系统寄存器重置                                         │
│  └── kvm_reset_sys_regs(vcpu)                                  │
│      └── for (i = 0; i < ARRAY_SIZE(sys_reg_descs); i++)      │
│          ├── reset_vm_ftr_id_reg(vcpu, r)                     │
│          ├── reset_vcpu_ftr_id_reg(vcpu, r)                   │
│          └── r->reset(vcpu, r)                                │
│                                                                 │
│  数据结构初始化完成:                                            │
│  ├── vcpu->arch.ctxt.regs (X0-X30, SP, PC, PSTATE)            │
│  ├── vcpu->arch.ctxt.fp_regs (V0-V31, FPSR, FPCR)             │
│  ├── vcpu->arch.ctxt.sys_regs[NR_SYS_REGS]                    │
│  ├── vcpu->arch.ctxt.spsr_abt/und/irq/fiq                     │
│  └── vcpu->arch.sve_state                                      │
│                                                                 │
│                  ↓ ioctl 返回成功                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                      QEMU 用户空间                               │
│                                                                 │
│  Step 7: 后续初始化                                             │
│  └── kvm_arm_vcpu_finalize(cpu, KVM_ARM_VCPU_SVE)             │
│  └── kvm_get_one_reg(cs, KVM_REG_ARM_PSCI_VERSION, ...)       │
│  └── kvm_get_one_reg(cs, ARM64_SYS_REG(ARM_CPU_ID_MPIDR), ...)│
│  └── kvm_arm_init_cpreg_list(cpu)                              │
│      ├── ioctl(KVM_GET_REG_LIST)                               │
│      └── ioctl(KVM_GET_ONE_REG) 批量获取                       │
│                                                                 │
│  Step 8: 初始化 cpreg 列表                                      │
│  └── 分配 cpu->cpreg_indexes[]                                 │
│  ├── 分配 cpu->cpreg_values[]                                  │
│  └── 填充寄存器值                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 QEMU 侧初始化步骤

#### Step 1-2: 创建 VCPU 并设置初始化特性

`target/arm/kvm.c:1865`

```c
int kvm_arch_init_vcpu(CPUState *cs)
{
    ARMCPU *cpu = ARM_CPU(cs);
    CPUARMState *env = &cpu->env;
    
    // Step 2: 设置 KVM 初始化特性标志
    memset(cpu->kvm_init_features, 0, sizeof(cpu->kvm_init_features));
    
    // 设置 VCPU 特性:
    if (cs->start_powered_off)
        cpu->kvm_init_features[0] |= 1 << KVM_ARM_VCPU_POWER_OFF;
    
    if (kvm_check_extension(cs->kvm_state, KVM_CAP_ARM_PSCI_0_2)) {
        cpu->psci_version = QEMU_PSCI_VERSION_0_2;
        cpu->kvm_init_features[0] |= 1 << KVM_ARM_VCPU_PSCI_0_2;
    }
    
    if (!arm_feature(&cpu->env, ARM_FEATURE_AARCH64))
        cpu->kvm_init_features[0] |= 1 << KVM_ARM_VCPU_EL1_32BIT;
    
    if (cpu->has_pmu)
        cpu->kvm_init_features[0] |= 1 << KVM_ARM_VCPU_PMU_V3;
    
    if (cpu_isar_feature(aa64_sve, cpu))
        cpu->kvm_init_features[0] |= 1 << KVM_ARM_VCPU_SVE;
    
    if (cpu_isar_feature(aa64_pauth, cpu))
        cpu->kvm_init_features[0] |= (1 << KVM_ARM_VCPU_PTRAUTH_ADDRESS |
                                      1 << KVM_ARM_VCPU_PTRAUTH_GENERIC);
    
    // 构造 kvm_vcpu_init 结构:
    struct kvm_vcpu_init init;
    init.target = cpu->kvm_target;
    memcpy(init.features, cpu->kvm_init_features, sizeof(init.features));
    
    // Step 3: 调用 KVM ioctl
    ret = kvm_arm_vcpu_init(cpu);  // → ioctl(KVM_ARM_VCPU_INIT)
    if (ret)
        return ret;
    
    // ... 后续步骤见 3.4
}
```

```c
// target/arm/kvm.c:73
static int kvm_arm_vcpu_init(ARMCPU *cpu)
{
    struct kvm_vcpu_init init;
    
    init.target = cpu->kvm_target;
    memcpy(init.features, cpu->kvm_init_features, sizeof(init.features));
    
    // 发送 ioctl 到内核
    return kvm_vcpu_ioctl(CPU(cpu), KVM_ARM_VCPU_INIT, &init);
}
```

**此时 QEMU 设置的关键参数:**
- `init.target`: CPU 类型 (如 `KVM_ARM_TARGET_GENERIC_V8`)
- `init.features[0]`: 特性位图 (POWER_OFF, PSCI_0_2, EL1_32BIT, PMU_V3, SVE, PTRAUTH)

### 3.3 KVM 内核处理步骤

#### Step 4: ioctl 入口处理

`arch/arm64/kvm/arm.c`

```c
long kvm_arch_vcpu_ioctl(struct file *filp, unsigned int ioctl, ...)
{
    struct kvm_vcpu *vcpu = filp->private_data;
    
    switch (ioctl) {
    case KVM_ARM_VCPU_INIT: {
        struct kvm_vcpu_init init;
        
        // 从用户空间复制初始化参数
        if (copy_from_user(&init, argp, sizeof(init)))
            return -EFAULT;
        
        // 调用 VCPU 初始化
        r = kvm_arch_vcpu_ioctl_vcpu_init(vcpu, &init);
        break;
    }
    }
}
```

```c
static int kvm_arch_vcpu_ioctl_vcpu_init(struct kvm_vcpu *vcpu,
                                          struct kvm_vcpu_init *init)
{
    // 检查 CPU target 是否合法
    if (init->target >= KVM_ARM_NUM_TARGETS)
        return -EINVAL;
    
    // 设置 VCPU 特性
    bitmap_zero(vcpu->arch.features, KVM_VCPU_MAX_FEATURES);
    bitmap_copy(vcpu->arch.features, 
                (unsigned long *)init->features,
                KVM_VCPU_MAX_FEATURES);
    
    // Step 5: 调用 VCPU 重置
    return kvm_reset_vcpu(vcpu);
}
```

#### Step 5: VCPU 重置 - 核心寄存器初始化

`arch/arm64/kvm/reset.c:237`

```c
int kvm_reset_vcpu(struct kvm_vcpu *vcpu)
{
    struct vcpu_reset_state reset_state;
    u32 pstate;
    int ret;
    
    // 获取 pending reset 状态
    spin_lock(&vcpu->arch.mp_state_lock);
    reset_state = vcpu->arch.reset_state;
    vcpu->arch.reset_state.reset = false;
    spin_unlock(&vcpu->arch.mp_state_lock);
    
    // 5.1: 重置 PMU
    kvm_pmu_vcpu_reset(vcpu);
    
    // 5.2: 处理 SVE 状态分配
    if (!kvm_arm_vcpu_sve_finalized(vcpu)) {
        if (test_bit(KVM_ARM_VCPU_SVE, vcpu->arch.features)) {
            ret = kvm_vcpu_enable_sve(vcpu);
            if (ret)
                goto out;
        }
    } else {
        // SVE 已 finalized，清零寄存器
        kvm_vcpu_reset_sve(vcpu);
    }
    
    // 5.3: 处理 PTRAUTH
    if (test_bit(KVM_ARM_VCPU_PTRAUTH_ADDRESS, vcpu->arch.features) ||
        test_bit(KVM_ARM_VCPU_PTRAUTH_GENERIC, vcpu->arch.features)) {
        if (kvm_vcpu_enable_ptrauth(vcpu)) {
            ret = -EINVAL;
            goto out;
        }
    }
    
    // 5.4: 确定初始 PSTATE (CPU 模式)
    if (vcpu_el1_is_32bit(vcpu))
        pstate = VCPU_RESET_PSTATE_SVC;   // 32 位 SVC 模式
    else if (vcpu_has_nv(vcpu))
        pstate = VCPU_RESET_PSTATE_EL2;   // 嵌套虚拟化: EL2
    else
        pstate = VCPU_RESET_PSTATE_EL1;   // 64 位: EL1h
    
    // 5.5: 【关键】重置核心寄存器
    //      初始化 vcpu->arch.ctxt.regs
    memset(vcpu_gp_regs(vcpu), 0, sizeof(*vcpu_gp_regs(vcpu)));
    
    // 5.6: 【关键】重置浮点寄存器
    //      初始化 vcpu->arch.ctxt.fp_regs
    memset(&vcpu->arch.ctxt.fp_regs, 0, sizeof(vcpu->arch.ctxt.fp_regs));
    
    // 5.7: 重置 SPSR 寄存器
    vcpu->arch.ctxt.spsr_abt = 0;
    vcpu->arch.ctxt.spsr_und = 0;
    vcpu->arch.ctxt.spsr_irq = 0;
    vcpu->arch.ctxt.spsr_fiq = 0;
    
    // 设置初始 PSTATE
    vcpu_gp_regs(vcpu)->pstate = pstate;
    
    // 5.8: 【关键】重置系统寄存器
    //      初始化 vcpu->arch.ctxt.sys_regs[]
    kvm_reset_sys_regs(vcpu);
    
    // 5.9: 处理 PSCI 重置状态
    if (reset_state.reset) {
        unsigned long target_pc = reset_state.pc;
        
        // Thumb2 入口点处理
        if (vcpu_mode_is_32bit(vcpu) && (target_pc & 1)) {
            target_pc &= ~1UL;
            vcpu_set_thumb(vcpu);
        }
        
        // 设置 PC 和 R0
        *vcpu_pc(vcpu) = target_pc;
        vcpu_set_reg(vcpu, 0, reset_state.r0);
    }
    
    // 5.10: 重置 Timer
    ret = kvm_timer_vcpu_reset(vcpu);
    
out:
    return ret;
}
```

#### Step 6: 系统寄存器重置

`arch/arm64/kvm/sys_regs.c:3708`

```c
void kvm_reset_sys_regs(struct kvm_vcpu *vcpu)
{
    struct kvm *kvm = vcpu->kvm;
    unsigned long i;
    
    // 遍历系统寄存器描述表
    for (i = 0; i < ARRAY_SIZE(sys_reg_descs); i++) {
        const struct sys_reg_desc *r = &sys_reg_descs[i];
        
        // 跳过无 reset 函数的寄存器
        if (!r->reset)
            continue;
        
        // ID 寄存器特殊处理:
        if (is_vm_ftr_id_reg(reg_to_encoding(r)))
            // VM 级别 ID 寄存器 (如 ID_AA64MMFR*)
            reset_vm_ftr_id_reg(vcpu, r);
        else if (is_vcpu_ftr_id_reg(reg_to_encoding(r)))
            // VCPU 级别 ID 寄存器 (如 MPIDR)
            reset_vcpu_ftr_id_reg(vcpu, r);
        else
            // 普通 system register，调用 reset 回调
            // 例如: SCTLR_EL1, TCR_EL1, TTBR0_EL1...
            r->reset(vcpu, r);
    }
    
    // 标记 ID 寄存器已初始化
    set_bit(KVM_ARCH_FLAG_ID_REGS_INITIALIZED, &kvm->arch.flags);
}
```

**系统寄存器描述表示例:**

```c
// arch/arm64/kvm/sys_regs.c
static const struct sys_reg_desc sys_reg_descs[] = {
    // SCTLR_EL1: 系统控制寄存器
    { SYS_DESC(SYS_SCTLR_EL1), access_vm, NULL, reset_sysreg_el1, 
      SCTLR_EL1_RES0, SCTLR_EL1_EE | SCTLR_EL1_I | ... },
    
    // TCR_EL1: 转换控制寄存器
    { SYS_DESC(SYS_TCR_EL1), access_vm, NULL, reset_sysreg_el1, 
      TCR_EL1_RES0, 0 },
    
    // TTBR0_EL1: 转换表基址
    { SYS_DESC(SYS_TTBR0_EL1), access_vm, NULL, reset_sysreg_el1, 0, 0 },
    
    // ID 寄存器:
    { SYS_DESC(SYS_ID_AA64MMFR0_EL1), access_id_reg, .reset = reset_vcpu_ftr_id_reg },
    { SYS_DESC(SYS_ID_AA64MMFR1_EL1), access_id_reg, .reset = reset_vcpu_ftr_id_reg },
    
    // ...
};
```

### 3.4 数据结构初始化结果

经过上述流程，以下数据结构被初始化:

#### 3.4.1 kvm_cpu_context 初始化

```c
struct kvm_cpu_context ctxt;

// 通用寄存器 (全部清零，PSTATE 设置为 EL1h/EL2/SVC)
ctxt.regs.regs[31]      = {0};        // X0-X30
ctxt.regs.sp            = 0;          // SP_EL0
ctxt.regs.pc            = 0;          // PC
ctxt.regs.pstate        = pstate;     // PSTATE (EL1h 或 SVC)

// 浮点寄存器 (全部清零)
ctxt.fp_regs.vregs[32]  = {0};        // V0-V31
ctxt.fp_regs.fpsr       = 0;          // FPSR
ctxt.fp_regs.fpcr       = 0;          // FPCR

// SPSR (全部清零)
ctxt.spsr_abt           = 0;
ctxt.spsr_und           = 0;
ctxt.spsr_irq           = 0;
ctxt.spsr_fiq           = 0;
```

#### 3.4.2 系统寄存器初始化

```c
// sys_regs[] 数组，索引由寄存器编码决定
ctxt.sys_regs[NR_SYS_REGS];

// 典型寄存器初始值:
sys_regs[SCTLR_EL1]     = SCTLR_EL1_RES0 | SCTLR_EL1_EE | ...;
sys_regs[TCR_EL1]       = TCR_EL1_RES0;
sys_regs[TTBR0_EL1]     = 0;
sys_regs[TTBR1_EL1]     = 0;
sys_regs[SPSR_EL1]      = 0;
sys_regs[ELR_EL1]       = 0;
sys_regs[MPIDR_EL1]     = vcpu->mp_affinity;  // 来自硬件
sys_regs[ID_AA64MMFR0]  = sanitized host value;
sys_regs[ID_AA64MMFR1]  = sanitized host value;
// ...
```

#### 3.4.3 SVE 状态初始化 (如果启用)

```c
// SVE 寄存器状态
vcpu->arch.sve_state    = kzalloc(vcpu_sve_state_size(vcpu), ...);

// Z 寄存器 (2048-bit × 32)
// P 寄存器 (256-bit × 17, 包括 FFR)
// 全部清零
```

### 3.5 QEMU 后续初始化步骤

#### Step 7-8: 获取寄存器值并初始化 cpreg 列表

```c
int kvm_arch_init_vcpu(CPUState *cs)
{
    // ... Step 1-3 已完成
    
    // Step 7: SVE finalize (如果需要)
    if (cpu_isar_feature(aa64_sve, cpu)) {
        ret = kvm_arm_sve_set_vls(cpu);
        if (ret)
            return ret;
        ret = kvm_arm_vcpu_finalize(cpu, KVM_ARM_VCPU_SVE);
        if (ret)
            return ret;
    }
    
    // Step 7: 获取 PSCI 版本 (通过 KVM_GET_ONE_REG)
    if (!kvm_get_one_reg(cs, KVM_REG_ARM_PSCI_VERSION, &psciver)) {
        cpu->psci_version = psciver;
    }
    
    // Step 7: 获取 MPIDR (通过 KVM_GET_ONE_REG)
    ret = kvm_get_one_reg(cs, ARM64_SYS_REG(ARM_CPU_ID_MPIDR), &mpidr);
    if (ret)
        return ret;
    cpu->mp_affinity = mpidr & ARM64_AFFINITY_MASK;
    
    // Step 8: 【关键】初始化 cpreg 列表
    return kvm_arm_init_cpreg_list(cpu);
}
```

```c
int kvm_arm_init_cpreg_list(ARMCPU *cpu)
{
    CPUState *cs = CPU(cpu);
    struct kvm_reg_list reg_list;
    int i, ret;
    
    // 8.1: 从 KVM 获取寄存器列表
    //      ioctl(KVM_GET_REG_LIST)
    reg_list.n = 0;
    ret = kvm_vcpu_ioctl(cs, KVM_GET_REG_LIST, &reg_list);
    if (ret)
        return ret;
    
    // 重新分配空间并获取完整列表
    reg_list.reg = g_new(uint64_t, reg_list.n);
    ret = kvm_vcpu_ioctl(cs, KVM_GET_REG_LIST, &reg_list);
    
    // 8.2: 分配 QEMU cpreg 数组
    cpu->cpreg_indexes = g_new(uint64_t, reg_list.n);
    cpu->cpreg_values = g_new(uint64_t, reg_list.n);
    cpu->cpreg_array_len = reg_list.n;
    
    // 8.3: 复制寄存器 ID (indexes)
    memcpy(cpu->cpreg_indexes, reg_list.reg, reg_list.n * sizeof(uint64_t));
    
    // 8.4: 批量获取寄存器值 (通过 KVM_GET_ONE_REG)
    for (i = 0; i < cpu->cpreg_array_len; i++) {
        uint64_t regidx = cpu->cpreg_indexes[i];
        
        switch (regidx & KVM_REG_SIZE_MASK) {
        case KVM_REG_SIZE_U32:
            uint32_t v32;
            ret = kvm_get_one_reg(cs, regidx, &v32);
            if (!ret)
                cpu->cpreg_values[i] = v32;
            break;
        case KVM_REG_SIZE_U64:
            ret = kvm_get_one_reg(cs, regidx, cpu->cpreg_values + i);
            break;
        }
    }
    
    // 8.5: 将值写入 CPUARMState (同步到 env)
    write_list_to_cpustate(cpu);
    
    return 0;
}
```

### 3.6 初始化时序总结

| 步骤 | 位置 | 操作 | 数据结构 |
|------|------|------|----------|
| 1-2 | QEMU | 设置 kvm_init_features | `struct kvm_vcpu_init` |
| 3 | QEMU | ioctl(KVM_ARM_VCPU_INIT) | - |
| 4 | KVM | ioctl 入口处理 | `vcpu->arch.features` |
| 5 | KVM | kvm_reset_vcpu() | `vcpu->arch.ctxt` |
| 5.5 | KVM | 核心寄存器清零 | `ctxt.regs, fp_regs` |
| 5.6 | KVM | 浮点寄存器清零 | `fp_regs.vregs[32]` |
| 5.8 | KVM | 系统寄存器重置 | `ctxt.sys_regs[NR_SYS_REGS]` |
| 6 | KVM | 遍历 sys_reg_descs | 调用各 reset 函数 |
| 7 | QEMU | 获取 PSCI/MPIDR | `cpu->psci_version, mp_affinity` |
| 8 | QEMU | 初始化 cpreg 列表 | `cpreg_indexes[], cpreg_values[]` |

**关键点:**
- QEMU 先构造参数，通过 ioctl 传递给 KVM
- KVM 完成所有寄存器的实际初始化
- QEMU 再通过 KVM_GET_ONE_REG 获取初始化后的值
- cpreg 列表是 QEMU 维护的系统寄存器缓存
```

---

## 4. 寄存器同步流程

### 4.1 QEMU → KVM (kvm_arch_put_registers)

`target/arm/kvm.c:2052`

```c
int kvm_arch_put_registers(CPUState *cs, int level)
{
    ARMCPU *cpu = ARM_CPU(cs);
    CPUARMState *env = &cpu->env;
    
    // 1. 如果是 32 位模式，同步到 64 位
    if (!is_a64(env))
        aarch64_sync_32_to_64(env);
    
    // 2. 设置通用寄存器 X0-X30
    for (i = 0; i < 31; i++) {
        kvm_set_one_reg(cs, AARCH64_CORE_REG(regs.regs[i]), &env->xregs[i]);
    }
    
    // 3. 设置 SP
    aarch64_save_sp(env, 1);
    kvm_set_one_reg(cs, AARCH64_CORE_REG(regs.sp), &env->sp_el[0]);
    kvm_set_one_reg(cs, AARCH64_CORE_REG(sp_el1), &env->sp_el[1]);
    
    // 4. 设置 PSTATE
    val = is_a64(env) ? pstate_read(env) : cpsr_read(env);
    kvm_set_one_reg(cs, AARCH64_CORE_REG(regs.pstate), &val);
    
    // 5. 设置 PC
    kvm_set_one_reg(cs, AARCH64_CORE_REG(regs.pc), &env->pc);
    
    // 6. 设置 ELR_EL1
    kvm_set_one_reg(cs, AARCH64_CORE_REG(elr_el1), &env->elr_el[1]);
    
    // 7. 设置 SPSR
    for (i = 0; i < KVM_NR_SPSR; i++) {
        kvm_set_one_reg(cs, AARCH64_CORE_REG(spsr[i]), &env->banked_spsr[i + 1]);
    }
    
    // 8. 设置浮点寄存器
    if (cpu_isar_feature(aa64_sve, cpu))
        kvm_arch_put_sve(cs);
    else
        kvm_arch_put_fpsimd(cs);
    
    // 9. 设置 FPSR/FPCR
    fpr = vfp_get_fpsr(env);
    kvm_set_one_reg(cs, AARCH64_SIMD_CTRL_REG(fp_regs.fpsr), &fpr);
    fpr = vfp_get_fpcr(env);
    kvm_set_one_reg(cs, AARCH64_SIMD_CTRL_REG(fp_regs.fpcr), &fpr);
    
    // 10. 同步系统寄存器
    write_cpustate_to_list(cpu, true);
    write_list_to_kvmstate(cpu, level);
    
    return 0;
}
```

### 4.2 KVM → QEMU (kvm_arch_get_registers)

`target/arm/kvm.c:2236`

```c
int kvm_arch_get_registers(CPUState *cs)
{
    ARMCPU *cpu = ARM_CPU(cs);
    CPUARMState *env = &cpu->env;
    
    // 1. 获取通用寄存器 X0-X30
    for (i = 0; i < 31; i++) {
        kvm_get_one_reg(cs, AARCH64_CORE_REG(regs.regs[i]), &env->xregs[i]);
    }
    
    // 2. 获取 SP
    kvm_get_one_reg(cs, AARCH64_CORE_REG(regs.sp), &env->sp_el[0]);
    kvm_get_one_reg(cs, AARCH64_CORE_REG(sp_el1), &env->sp_el[1]);
    
    // 3. 获取 PSTATE
    kvm_get_one_reg(cs, AARCH64_CORE_REG(regs.pstate), &val);
    env->aarch64 = ((val & PSTATE_nRW) == 0);
    if (is_a64(env))
        pstate_write(env, val);
    else
        cpsr_write(env, val, 0xffffffff, CPSRWriteRaw);
    
    aarch64_restore_sp(env, 1);
    
    // 4. 获取 PC
    kvm_get_one_reg(cs, AARCH64_CORE_REG(regs.pc), &env->pc);
    
    // 5. 如果是 32 位模式，同步回 32 位寄存器
    if (!is_a64(env))
        aarch64_sync_64_to_32(env);
    
    // 6. 获取 ELR_EL1
    kvm_get_one_reg(cs, AARCH64_CORE_REG(elr_el1), &env->elr_el[1]);
    
    // 7. 获取 SPSR
    for (i = 0; i < KVM_NR_SPSR; i++) {
        kvm_get_one_reg(cs, AARCH64_CORE_REG(spsr[i]), &env->banked_spsr[i + 1]);
    }
    
    // 8. 获取浮点寄存器
    if (cpu_isar_feature(aa64_sve, cpu))
        kvm_arch_get_sve(cs);
    else
        kvm_arch_get_fpsimd(cs);
    
    // 9. 获取 FPSR/FPCR
    kvm_get_one_reg(cs, AARCH64_SIMD_CTRL_REG(fp_regs.fpsr), &fpr);
    vfp_set_fpsr(env, fpr);
    kvm_get_one_reg(cs, AARCH64_SIMD_CTRL_REG(fp_regs.fpcr), &fpr);
    vfp_set_fpcr(env, fpr);
    
    // 10. 同步系统寄存器
    write_kvmstate_to_list(cpu);
    write_list_to_cpustate(cpu);
    
    return 0;
}
```

### 4.3 系统寄存器同步机制

#### 4.3.1 cpreg 列表操作

```c
// 从 CPUARMState 同步到 cpreg 值列表
bool write_cpustate_to_list(ARMCPU *cpu, bool kvm_sync)
{
    for (i = 0; i < cpu->cpreg_array_len; i++) {
        uint32_t regidx = kvm_to_cpreg_id(cpu->cpreg_indexes[i]);
        const ARMCPRegInfo *ri = get_arm_cp_reginfo(cpu->cp_regs, regidx);
        
        if (ri && !(ri->type & ARM_CP_NO_RAW))
            cpu->cpreg_values[i] = read_raw_cp_reg(&cpu->env, ri);
    }
}

// 从 cpreg 值列表同步到 CPUARMState
bool write_list_to_cpustate(ARMCPU *cpu)
{
    for (i = 0; i < cpu->cpreg_array_len; i++) {
        uint32_t regidx = kvm_to_cpreg_id(cpu->cpreg_indexes[i]);
        const ARMCPRegInfo *ri = get_arm_cp_reginfo(cpu->cp_regs, regidx);
        
        if (ri && !(ri->type & ARM_CP_NO_RAW))
            write_raw_cp_reg(&cpu->env, ri, cpu->cpreg_values[i]);
    }
}

// 从 KVM 同步到 cpreg 值列表
bool write_kvmstate_to_list(ARMCPU *cpu)
{
    for (i = 0; i < cpu->cpreg_array_len; i++) {
        kvm_get_one_reg(cs, cpu->cpreg_indexes[i], cpu->cpreg_values + i);
    }
}

// 从 cpreg 值列表同步到 KVM
bool write_list_to_kvmstate(ARMCPU *cpu, int level)
{
    for (i = 0; i < cpu->cpreg_array_len; i++) {
        if (kvm_arm_cpreg_level(regidx) > level)
            continue;
        kvm_set_one_reg(cs, cpu->cpreg_indexes[i], cpu->cpreg_values + i);
    }
}
```

---

## 5. 寄存器修改流程

### 5.1 ioctl 入口

`arch/arm64/kvm/arm.c:1864`

```c
long kvm_arch_vcpu_ioctl(struct file *filp, unsigned int ioctl, ...)
{
    switch (ioctl) {
    case KVM_SET_ONE_REG:
    case KVM_GET_ONE_REG: {
        struct kvm_one_reg reg;
        
        if (copy_from_user(&reg, argp, sizeof(reg)))
            break;
        
        // 处理 pending reset
        if (kvm_check_request(KVM_REQ_VCPU_RESET, vcpu))
            kvm_reset_vcpu(vcpu);
        
        if (ioctl == KVM_SET_ONE_REG)
            r = kvm_arm_set_reg(vcpu, &reg);
        else
            r = kvm_arm_get_reg(vcpu, &reg);
        break;
    }
    }
}
```

### 5.2 寄存器分发

`arch/arm64/kvm/guest.c:875`

```c
int kvm_arm_set_reg(struct kvm_vcpu *vcpu, const struct kvm_one_reg *reg)
{
    switch (reg->id & KVM_REG_ARM_COPROC_MASK) {
    case KVM_REG_ARM_CORE:
        return set_core_reg(vcpu, reg);
    case KVM_REG_ARM_FW:
    case KVM_REG_ARM_FW_FEAT_BMAP:
        return kvm_arm_set_fw_reg(vcpu, reg);
    case KVM_REG_ARM64_SVE:
        return set_sve_reg(vcpu, reg);
    }
    
    if (is_timer_reg(reg->id))
        return set_timer_reg(vcpu, reg);
    
    return kvm_arm_sys_reg_set_reg(vcpu, reg);
}
```

### 5.3 核心寄存器设置

`arch/arm64/kvm/guest.c:272`

```c
static int set_core_reg(struct kvm_vcpu *vcpu, const struct kvm_one_reg *reg)
{
    void *addr;
    u64 off;
    
    // 1. 从 ID 计算偏移
    off = core_reg_offset_from_id(reg->id);
    
    // 2. 获取目标地址
    addr = core_reg_addr(vcpu, reg);
    if (!addr)
        return -EINVAL;
    
    // 3. 从用户空间复制值
    if (copy_from_user(valp, uaddr, KVM_REG_SIZE(reg->id)))
        return -EFAULT;
    
    // 4. PSTATE 特殊处理
    if (off == KVM_REG_ARM_CORE_REG(regs.pstate)) {
        // 验证模式是否合法
        u64 mode = (*(u64 *)valp) & PSR_AA32_MODE_MASK;
        // ... 模式验证逻辑
    }
    
    // 5. 写入寄存器
    memcpy(addr, valp, KVM_REG_SIZE(reg->id));
    
    return 0;
}
```

### 5.4 核心寄存器地址映射

`arch/arm64/kvm/guest.c:178`

```c
static void *core_reg_addr(struct kvm_vcpu *vcpu, const struct kvm_one_reg *reg)
{
    u64 off = core_reg_offset_from_id(reg->id);
    
    switch (off) {
    case KVM_REG_ARM_CORE_REG(regs.regs[0]) ...
         KVM_REG_ARM_CORE_REG(regs.regs[30]):
        off -= KVM_REG_ARM_CORE_REG(regs.regs[0]);
        off /= 2;
        return &vcpu->arch.ctxt.regs.regs[off];
    
    case KVM_REG_ARM_CORE_REG(regs.sp):
        return &vcpu->arch.ctxt.regs.sp;
    
    case KVM_REG_ARM_CORE_REG(regs.pc):
        return &vcpu->arch.ctxt.regs.pc;
    
    case KVM_REG_ARM_CORE_REG(regs.pstate):
        return &vcpu->arch.ctxt.regs.pstate;
    
    case KVM_REG_ARM_CORE_REG(sp_el1):
        return __ctxt_sys_reg(&vcpu->arch.ctxt, SP_EL1);
    
    case KVM_REG_ARM_CORE_REG(elr_el1):
        return __ctxt_sys_reg(&vcpu->arch.ctxt, ELR_EL1);
    
    case KVM_REG_ARM_CORE_REG(spsr[KVM_SPSR_EL1]):
        return __ctxt_sys_reg(&vcpu->arch.ctxt, SPSR_EL1);
    
    // ... 更多映射
    
    case KVM_REG_ARM_CORE_REG(fp_regs.vregs[0]) ...
         KVM_REG_ARM_CORE_REG(fp_regs.vregs[31]):
        off -= KVM_REG_ARM_CORE_REG(fp_regs.vregs[0]);
        off /= 4;
        return &vcpu->arch.ctxt.fp_regs.vregs[off];
    
    default:
        return NULL;
    }
}
```

### 5.5 系统寄存器设置

`arch/arm64/kvm/sys_regs.c:3912`

```c
int kvm_sys_reg_set_user(struct kvm_vcpu *vcpu, const struct kvm_one_reg *reg,
                         const struct sys_reg_desc table[], unsigned int num)
{
    u64 val;
    const struct sys_reg_desc *r;
    
    // 1. 从用户空间获取值
    if (get_user(val, uaddr))
        return -EFAULT;
    
    // 2. 找到寄存器描述符
    r = id_to_sys_reg_desc(vcpu, reg->id, table, num);
    if (!r || sysreg_hidden_user(vcpu, r))
        return -ENOENT;
    
    // 3. 检查是否忽略写入
    if (sysreg_user_write_ignore(vcpu, r))
        return 0;
    
    // 4. 写入寄存器
    if (r->set_user) {
        ret = (r->set_user)(vcpu, r, val);  // 特殊处理函数
    } else {
        __vcpu_sys_reg(vcpu, r->reg) = val;  // 直接写入
        ret = 0;
    }
    
    return ret;
}
```

### 5.6 QEMU 侧 ioctl 封装

`accel/kvm/kvm-all.c:3427`

```c
int kvm_set_one_reg(CPUState *cs, uint64_t id, void *source)
{
    struct kvm_one_reg reg;
    
    reg.id = id;
    reg.addr = (uintptr_t) source;
    
    return kvm_vcpu_ioctl(cs, KVM_SET_ONE_REG, &reg);
}

int kvm_get_one_reg(CPUState *cs, uint64_t id, void *target)
{
    struct kvm_one_reg reg;
    
    reg.id = id;
    reg.addr = (uintptr_t) target;
    
    return kvm_vcpu_ioctl(cs, KVM_GET_ONE_REG, &reg);
}
```

---

## 6. 关键代码路径

### 6.1 数据流向图

```
┌─────────────────────────────────────────────────────────────────────┐
│                          QEMU 用户空间                               │
│                                                                     │
│   CPUARMState                                                       │
│   ├── xregs[32]         (X0-X30, SP)                               │
│   ├── pc                                                            │
│   ├── pstate                                                        │
│   ├── sp_el[4]                                                      │
│   ├── elr_el[4]                                                     │
│   ├── banked_spsr[8]                                                │
│   ├── cp15.*                                                        │
│   └── vfp.zregs/pregs                                               │
│                                                                     │
│   ┌──────────────────┐      ┌─────────────────────┐               │
│   │ cpreg_indexes[]  │      │  cpreg_values[]     │               │
│   │  (寄存器 ID)     │ ←──→ │  (寄存器值)         │               │
│   └──────────────────┘      └─────────────────────┘               │
│                                                                     │
│   kvm_arch_put_registers()          kvm_arch_get_registers()       │
│   ├── AARCH64_CORE_REG()            ├── AARCH64_CORE_REG()         │
│   ├── AARCH64_SIMD_*_REG()          ├── AARCH64_SIMD_*_REG()       │
│   ├── write_cpustate_to_list()      ├── write_kvmstate_to_list()   │
│   └── write_list_to_kvmstate()      └── write_list_to_cpustate()   │
│                                                                     │
│   kvm_set_one_reg()                 kvm_get_one_reg()               │
│   └── ioctl(KVM_SET_ONE_REG)        └── ioctl(KVM_GET_ONE_REG)     │
│                                                                     │
└───────────────────────────────ioctl─────────────────────────────────┘
                                       ↓
┌─────────────────────────────────────────────────────────────────────┐
│                           KVM 内核空间                               │
│                                                                     │
│   kvm_arch_vcpu_ioctl()                                             │
│   └── kvm_arm_set_reg() / kvm_arm_get_reg()                        │
│       ├── set_core_reg() / get_core_reg()                          │
│       ├── kvm_arm_sys_reg_set_reg() / get_reg()                    │
│       └── set_timer_reg() / get_timer_reg()                        │
│                                                                     │
│   struct kvm_vcpu                                                   │
│   └── struct kvm_vcpu_arch                                          │
│       └── struct kvm_cpu_context ctxt                               │
│           ├── struct user_pt_regs regs                             │
│           │   ├── regs[31]  (X0-X30)                               │
│           │   ├── sp                                                │
│           │   ├── pc                                                │
│           │   └── pstate                                            │
│           ├── spsr_abt/und/irq/fiq                                 │
│           ├── struct user_fpsimd_state fp_regs                     │
│           │   ├── vregs[32]                                        │
│           │   ├── fpsr                                             │
│           │   └── fpcr                                             │
│           └── u64 sys_regs[NR_SYS_REGS]                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.2 同步时机

| 场景 | 调用函数 | 方向 | level |
|------|---------|------|-------|
| VM 启动前 | kvm_arch_put_registers() | QEMU→KVM | KVM_PUT_FULL_STATE |
| VM 运行前 (每次) | kvm_arch_put_registers() | QEMU→KVM | KVM_PUT_RUNTIME_STATE |
| VM 退出后 (每次) | kvm_arch_get_registers() | KVM→QEMU | - |
| VM 状态保存 | kvm_arch_get_registers() | KVM→QEMU | - |
| VM 状态恢复 | kvm_arch_put_registers() | QEMU→KVM | KVM_PUT_FULL_STATE |

### 6.3 关键文件路径

**内核:**
- `arch/arm64/include/uapi/asm/kvm.h` - 用户空间 API 定义
- `arch/arm64/include/asm/kvm_host.h` - 内核数据结构
- `arch/arm64/kvm/arm.c` - VCPU ioctl 处理
- `arch/arm64/kvm/guest.c` - 寄存器 get/set 实现
- `arch/arm64/kvm/sys_regs.c` - 系统寄存器处理
- `arch/arm64/kvm/reset.c` - VCPU 重置

**QEMU:**
- `target/arm/cpu.h` - CPUARMState 定义
- `target/arm/kvm.c` - KVM 寄存器同步
- `target/arm/kvm_arm.h` - KVM 相关声明
- `target/arm/helper.c` - cpreg 操作
- `accel/kvm/kvm-all.c` - ioctl 封装

---

## 总结

ARM64 KVM-QEMU 寄存器虚拟化的核心要点：

1. **统一的寄存器 ID 编码**：KVM 通过标准化的 64 位 ID 标识所有寄存器
2. **分层的数据结构**：内核使用 `kvm_vcpu_arch` → `kvm_cpu_context` 存储寄存器
3. **初始化流程**：`kvm_reset_vcpu()` 遍历寄存器描述表设置初始值
4. **同步机制**：QEMU 通过 cpreg 列表作为中间层同步系统寄存器
5. **ioctl 接口**：`KVM_SET_ONE_REG/KVM_GET_ONE_REG` 是用户空间修改寄存器的唯一入口