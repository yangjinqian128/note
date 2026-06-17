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

## 1. KVM 寄存器 ID 编码

ARM 的编码是**硬件指令编码**，只解决"读写寄存器"的问题。

KVM 的 ID 是软件 API 编码，需要解决"用户空间怎么通过一个 ioctl 操作 vcpu
所有状态"的问题。

为什么 KVM 要构造自己的一套寄存器命名空间 ？
  - 系统寄存器有 ARM 编码，可以直接复用
  - 但是通用寄存器、浮点寄存器没有任何统一编码

因此，KVM 如果要在接口统一寄存器的编码规范，就必须自己发明一套。

### 1.1 KVM 寄存器 ID 结构

```
[63:60]  架构标志 (KVM_REG_ARM64 = 0x6)
[59:52]  大小标志 (KVM_REG_SIZE_U64/U32/U128)
[31:16]  COPROC 类型 (寄存器组分类)
[15:0]   寄存器编码
```

### 1.2 架构类型和大小标志

定义在通用部分：`include/uapi/linux/kvm.h`
```
/*
 * Architecture specific registers are to be defined in arch headers and
 * ORed with the arch identifier.
 */
#define KVM_REG_PPC		0x1000000000000000ULL
#define KVM_REG_X86		0x2000000000000000ULL
#define KVM_REG_IA64		0x3000000000000000ULL
#define KVM_REG_ARM		0x4000000000000000ULL
#define KVM_REG_S390		0x5000000000000000ULL
#define KVM_REG_ARM64		0x6000000000000000ULL
#define KVM_REG_MIPS		0x7000000000000000ULL
#define KVM_REG_RISCV		0x8000000000000000ULL
#define KVM_REG_LOONGARCH	0x9000000000000000ULL

#define KVM_REG_SIZE_SHIFT	52
#define KVM_REG_SIZE_MASK	0x00f0000000000000ULL
#define KVM_REG_SIZE_U8		0x0000000000000000ULL
#define KVM_REG_SIZE_U16	0x0010000000000000ULL
#define KVM_REG_SIZE_U32	0x0020000000000000ULL
#define KVM_REG_SIZE_U64	0x0030000000000000ULL
#define KVM_REG_SIZE_U128	0x0040000000000000ULL
#define KVM_REG_SIZE_U256	0x0050000000000000ULL
#define KVM_REG_SIZE_U512	0x0060000000000000ULL
#define KVM_REG_SIZE_U1024	0x0070000000000000ULL
#define KVM_REG_SIZE_U2048	0x0080000000000000ULL
```

### 1.3 COPROC 类型分类

| 类型值 | 名称 | 说明 |
|--------|------|------|
| 0x0010 | KVM_REG_ARM_CORE | 核心寄存器 (x0-x30, sp, pc, pstate, fp_regs) |
| 0x0011 | KVM_REG_ARM_DEMUX | 多值寄存器 (如 CCSIDR) |
| 0x0013 | KVM_REG_ARM64_SYSREG | AArch64 系统寄存器 |
| 0x0014 | KVM_REG_ARM_FW | 固件伪寄存器 (PSCI_VERSION 等) |
| 0x0015 | KVM_REG_ARM64_SVE | SVE 寄存器 |

### 1.4 系统寄存器编码

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

### 1.5 QEMU 宏定义

```c
// target/arm/kvm.c
#define AARCH64_CORE_REG(x)   (KVM_REG_ARM64 | KVM_REG_SIZE_U64 | \
                 KVM_REG_ARM_CORE | KVM_REG_ARM_CORE_REG(x))

#define AARCH64_SIMD_CORE_REG(x)   (KVM_REG_ARM64 | KVM_REG_SIZE_U128 | \
                 KVM_REG_ARM_CORE | KVM_REG_ARM_CORE_REG(x))

#define AARCH64_SIMD_CTRL_REG(x)   (KVM_REG_ARM64 | KVM_REG_SIZE_U32 | \
                 KVM_REG_ARM_CORE | KVM_REG_ARM_CORE_REG(x))
```

### 1.6 示例

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

#### 2.1.1 pt_regs 通用寄存器

定义在 `arch/arm64/include/uapi/asm/kvm.h:51`

```c
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

这里命名 kvm_cpu_context 上下文是因为，host切到guest，kvm会保存host的寄存器信息，恢复
guest的寄存器信息；在guest切换会host时再恢复host的寄存器信息，保存guest的寄存器信息。
因此这些寄存器信息就是上下文信息。

```

sys_regs[] 的 idx 的 寄存器语义定义在：`arch/arm64/include/asm/kvm_host.h:338`
```
enum vcpu_sysreg {
	__INVALID_SYSREG__,   /* 0 is reserved as an invalid value */
	MPIDR_EL1,	/* MultiProcessor Affinity Register */
	CLIDR_EL1,	/* Cache Level ID Register */
	CSSELR_EL1,	/* Cache Size Selection Register */
	SCTLR_EL1,	/* System Control Register */
	ACTLR_EL1,	/* Auxiliary Control Register */

    ...

	CNTHV_CVAL_EL2,

	NR_SYS_REGS	/* Nothing after this line! */
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

#### 2.1.4 kvm_arch - ID 寄存器
```
struct kvm_arch {
    ...

	/*
	 * Emulated CPU ID registers per VM
	 * (Op0, Op1, CRn, CRm, Op2) of the ID registers to be saved in it
	 * is (3, 0, 0, crm, op2), where 1<=crm<8, 0<=op2<8.
	 *
	 * These emulated idregs are VM-wide, but accessed from the context of a vCPU.
	 * Atomic access to multiple idregs are guarded by kvm_arch.config_lock.
	 */
#define IDREG_IDX(id)		(((sys_reg_CRm(id) - 1) << 3) | sys_reg_Op2(id))
#define KVM_ARM_ID_REG_NUM	(IDREG_IDX(sys_reg(3, 0, 0, 7, 7)) + 1)
	u64 id_regs[KVM_ARM_ID_REG_NUM];

    ...
}
```
ID 寄存器描述的是这个虚拟机的 CPU 能力（比如支持哪些指令集扩展、有多少 cache    
层级），这是整台虚拟机的属性，所有 vcpu 必须呈现相同的值，所以放在 vm 粒度的
数据结构中。

---

#### 2.1.5 sys_reg_descs[] - 系统寄存器描述表

`sys_reg_desc` 是对一个系统寄存器的完整抽象（`arch/arm64/kvm/sys_regs.h:52`），
一个表项解决四个问题：寄存器地址是什么、guest 访问时怎么处理、初始值怎么设、
值存在 sys_regs[] 的哪个位置。

```c
struct sys_reg_desc {
    /* Sysreg string for debug */
    const char *name;

    // 32位 guest 访存该寄存器时的映射方式
    enum { AA32_DIRECT, AA32_LO, AA32_HI } aarch32_map;

    // 寄存器地址：ARM 架构定义的 Op0/Op1/CRn/CRm/Op2
    u8 Op0;
    u8 Op1;
    u8 CRn;
    u8 CRm;
    u8 Op2;

    // guest 执行 MRS/MSR 访问该寄存器时，trap 到 KVM 的处理函数
    // NULL = guest 可直接读写，不 trap
    bool (*access)(struct kvm_vcpu *, struct sys_reg_params *,
                   const struct sys_reg_desc *);

    // reset 回调：返回该寄存器的初始值
    // 对于 ID 寄存器，返回的是 host 值的 sanitized 版本
    u64 (*reset)(struct kvm_vcpu *, const struct sys_reg_desc *);

    // 寄存器在 vcpu->arch.ctxt.sys_regs[] 中的索引（enum vcpu_sysreg 值）
    int reg;

    // 对于非 ID 寄存器：寄存器的 reset 初始值
    // 对于 ID 寄存器：可写位的掩码
    u64 val;

    // 用户空间（QEMU）get/set 时的特殊处理函数，NULL 则走默认路径
    int (*get_user)(struct kvm_vcpu *, const struct sys_reg_desc *, u64 *val);
    int (*set_user)(struct kvm_vcpu *, const struct sys_reg_desc *, u64 val);
};
```

`SYS_DESC` 宏（`sys_regs.h:248`）展开为 Op0/Op1/CRn/CRm/Op2 和 name：
```c
#define SYS_DESC(reg)               \
    .name = #reg,                   \
    Op0(sys_reg_Op0(reg)), Op1(sys_reg_Op1(reg)), \
    CRn(sys_reg_CRn(reg)), CRm(sys_reg_CRm(reg)), \
    Op2(sys_reg_Op2(reg))
```

实际使用——`sys_reg_descs[]` 数组中每个表项都是 `{ SYS_DESC(xxx), access, reset, reg, val }`：

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

    // ID 寄存器: access_id_reg 限制只读，reset 返回 sanitized host 值
    { SYS_DESC(SYS_ID_AA64MMFR0_EL1), access_id_reg, .reset = reset_vcpu_ftr_id_reg },
    { SYS_DESC(SYS_ID_AA64MMFR1_EL1), access_id_reg, .reset = reset_vcpu_ftr_id_reg },

    // ...
};
```

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
│  └── kvm_reset_vcpu(vcpu)    <--- *重点*                          │
│      ├── kvm_pmu_vcpu_reset(vcpu)                              │
│      ├── kvm_vcpu_reset_sve(vcpu)                              │
│      ├── memset(vcpu_gp_regs(vcpu), 0, ..)  <--- 将核心寄存器置零（除了PSTATE）   │
│      ├── memset(&vcpu->arch.ctxt.fp_regs, 0, ..)                │
│      ├── kvm_reset_sys_regs(vcpu)    <--- 初始化系统寄存器       │
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

### 3.2 KVM 内核处理步骤

#### Step 4: ioctl 入口 → features 写入 vcpu → 触发寄存器 reset

`arch/arm64/kvm/arm.c:1851`

ioctl 入口接收 QEMU 传来的 `kvm_vcpu_init`，其中 `features` 位图决定了这个 vcpu
启用哪些寄存器能力（SVE、PTRAUTH、PMU 等），是后续所有寄存器初始化的前提。

```c
struct kvm_vcpu_init {
    __u32 target;       // 固定填 KVM_ARM_TARGET_GENERIC_V8，现代内核无差异化处理
    __u32 features[7];  // 位图：每个 bit 对应一种 vcpu 能力
};

// features bit 定义（arch/arm64/include/uapi/asm/kvm.h）
#define KVM_ARM_VCPU_POWER_OFF          0  // 启动即关机（非寄存器能力，单独摘出处理）
#define KVM_ARM_VCPU_EL1_32BIT          1  // guest 运行 AArch32
#define KVM_ARM_VCPU_PSCI_0_2           2  // PSCI v0.2
#define KVM_ARM_VCPU_PMU_V3             3  // 启用 PMU 寄存器组
#define KVM_ARM_VCPU_SVE                4  // 启用 SVE 寄存器（Z/P/FFR）
#define KVM_ARM_VCPU_PTRAUTH_ADDRESS    5  // 启用地址认证寄存器
#define KVM_ARM_VCPU_PTRAUTH_GENERIC    6  // 启用通用认证寄存器
```

调用链：`kvm_arch_vcpu_ioctl_vcpu_init` → `kvm_vcpu_set_target` →
`__kvm_vcpu_set_target` → `kvm_reset_vcpu`

`__kvm_vcpu_set_target` 是 features 落地的地方（`arch/arm64/kvm/arm.c:1649`）：

```c
static int __kvm_vcpu_set_target(struct kvm_vcpu *vcpu,
                                 const struct kvm_vcpu_init *init)
{
    unsigned long features = init->features[0];
    struct kvm *kvm = vcpu->kvm;

    mutex_lock(&kvm->arch.config_lock);

    /*
     * 同一 VM 内所有 vcpu 的 features 必须完全一致。
     * 原因：features 决定寄存器集合（如 SVE/PTRAUTH），若不同 vcpu 的寄存器集合
     * 不同，guest OS 在 vcpu 迁移时会看到寄存器能力突变，行为不可预测。
     */
    if (test_bit(KVM_ARCH_FLAG_VCPU_FEATURES_CONFIGURED, &kvm->arch.flags) &&
        !bitmap_equal(kvm->arch.vcpu_features, &features, KVM_VCPU_MAX_FEATURES))
        goto out_unlock;

    // features 写入 vcpu->arch.features，后续 reset 函数据此决定初始化哪些寄存器
    bitmap_copy(vcpu->arch.features, &features, KVM_VCPU_MAX_FEATURES);

    ret = kvm_setup_vcpu(vcpu);   // 若启用 PMU 且 VM 尚未分配 PMU，则设默认 PMU
    if (ret)
        goto out_unlock;

    ret = kvm_reset_vcpu(vcpu);   // Step 5: 寄存器初始化主体
    ...

    // 记录 VM 级别已配置的 features，后续 vcpu 初始化以此为基准校验
    bitmap_copy(kvm->arch.vcpu_features, &features, KVM_VCPU_MAX_FEATURES);
    set_bit(KVM_ARCH_FLAG_VCPU_FEATURES_CONFIGURED, &kvm->arch.flags);
    vcpu_set_flag(vcpu, VCPU_INITIALIZED);

out_unlock:
    mutex_unlock(&kvm->arch.config_lock);
    return ret;
}
```

`kvm_reset_vcpu` 返回后，回到 `kvm_arch_vcpu_ioctl_vcpu_init` 还有两个寄存器相关操作
（`arch/arm64/kvm/arm.c:1741`）：

```c
    // 重置 HCR_EL2：控制哪些 guest 操作 trap 到 EL2（KVM）
    // 例如：是否 trap WFI、SMC、系统寄存器访问等
    vcpu_reset_hcr(vcpu);

    // 重置 CPTR_EL2：控制浮点/SVE/SME 的 trap 行为
    // 若 guest 启用了 SVE（features bit 4），此处会放开 SVE trap
    vcpu->arch.cptr_el2 = kvm_get_reset_cptr_el2(vcpu);
```

这两个寄存器不存在于 `kvm_cpu_context` 中，而是直接存在 `kvm_vcpu_arch` 里，
在每次 vcpu 进入 guest 前写入硬件。


#### Step 5: VCPU 重置 — 寄存器初始化主体

`arch/arm64/kvm/reset.c:237`

`kvm_reset_vcpu` 是寄存器初始化的核心。前面 Step 4 把 features 写入 vcpu 后，
这里根据 features 决定哪些寄存器组需要分配资源，然后将所有寄存器设为其架构定义的
reset 值。

精简后的核心逻辑（省略 PMU/Timer 等与寄存器无直接关系的部分）：

```c
int kvm_reset_vcpu(struct kvm_vcpu *vcpu)
{
    struct vcpu_reset_state reset_state;
    int ret;
    bool loaded;
    u32 pstate;

    spin_lock(&vcpu->arch.mp_state_lock);
    reset_state = vcpu->arch.reset_state;
    vcpu->arch.reset_state.reset = false;
    spin_unlock(&vcpu->arch.mp_state_lock);

    kvm_pmu_vcpu_reset(vcpu);

    // 若 vcpu 正 load 在物理 CPU 上（reboot 场景），先卸下来，
    // 这样后续 memset 只操作内存中的 kvm_cpu_context，不需要碰硬件
    preempt_disable();
    loaded = (vcpu->cpu != -1);
    if (loaded)
        kvm_arch_vcpu_put(vcpu);

    /* ===== 扩展寄存器：根据 features 按需分配 ===== */

    // SVE：首次 reset 时分配 sve_state（Z/P/FFR 寄存器），已分配则清零
    if (!kvm_arm_vcpu_sve_finalized(vcpu)) {
        if (test_bit(KVM_ARM_VCPU_SVE, vcpu->arch.features)) {
            ret = kvm_vcpu_enable_sve(vcpu);
            if (ret)
                goto out;
        }
    } else {
        kvm_vcpu_reset_sve(vcpu);
    }

    // PTRAUTH：确认 host 密钥可用
    if (test_bit(KVM_ARM_VCPU_PTRAUTH_ADDRESS, vcpu->arch.features) ||
        test_bit(KVM_ARM_VCPU_PTRAUTH_GENERIC, vcpu->arch.features)) {
        if (kvm_vcpu_enable_ptrauth(vcpu)) {
            ret = -EINVAL;
            goto out;
        }
    }

    /* ===== PSTATE：决定 guest 启动异常级别 ===== */

    if (vcpu_el1_is_32bit(vcpu))
        pstate = VCPU_RESET_PSTATE_SVC;   // AArch32: SVC 模式（等价 EL1）
    else if (vcpu_has_nv(vcpu))
        pstate = VCPU_RESET_PSTATE_EL2;   // 嵌套虚拟化: EL2
    else
        pstate = VCPU_RESET_PSTATE_EL1;   // 正常: EL1
    // PSTATE 定义：
    //   EL1 = PSR_MODE_EL1h | A | I | F | D
    //   EL2 = PSR_MODE_EL2h | A | I | F | D
    //   SVC = PSR_AA32_MODE_SVC | A | I | F
    //   A/I/F 初始置位 = 启动时屏蔽外部中断

    /* ===== 通用寄存器 & 浮点寄存器：统一清零 ===== */

    memset(vcpu_gp_regs(vcpu), 0, sizeof(*vcpu_gp_regs(vcpu)));
    // → X0-X30 = 0, SP = 0, PC = 0
    memset(&vcpu->arch.ctxt.fp_regs, 0, sizeof(vcpu->arch.ctxt.fp_regs));
    // → V0-V31 = 0, FPSR = 0, FPCR = 0
    vcpu->arch.ctxt.spsr_abt = 0;
    vcpu->arch.ctxt.spsr_und = 0;
    vcpu->arch.ctxt.spsr_irq = 0;
    vcpu->arch.ctxt.spsr_fiq = 0;
    vcpu_gp_regs(vcpu)->pstate = pstate;

    /* ===== 系统寄存器：逐个按描述表 reset ===== */

    kvm_reset_sys_regs(vcpu);   // → Step 6

    /* ===== PSCI 预置状态：覆盖 PC 和 X0 ===== */

    if (reset_state.reset) {
        if (vcpu_mode_is_32bit(vcpu) && (reset_state.pc & 1)) {
            reset_state.pc &= ~1UL;
            vcpu_set_thumb(vcpu);
        }
        if (reset_state.be)
            kvm_vcpu_set_be(vcpu);
        *vcpu_pc(vcpu) = reset_state.pc;          // 覆盖 PC
        vcpu_set_reg(vcpu, 0, reset_state.r0);    // 覆盖 X0
    }

    ret = kvm_timer_vcpu_reset(vcpu);

out:
    if (loaded)
        kvm_arch_vcpu_load(vcpu, smp_processor_id());
    preempt_enable();
    return ret;
}
```

**从寄存器视角总结：**

```
kvm_reset_vcpu 操作的数据结构          对应的寄存器
────────────────────────────────────────────────────────
vcpu->arch.sve_state                 Z0-Z31, P0-P15, FFR（SVE）
vcpu->arch.ctxt.regs                 X0-X30, SP, PC, PSTATE（通用）
vcpu->arch.ctxt.fp_regs              V0-V31, FPSR, FPCR（浮点）
vcpu->arch.ctxt.spsr_*               SPSR_ABT/UND/IRQ/FIQ
vcpu->arch.ctxt.sys_regs[]           SCTLR_EL1, TCR_EL1, TTBRx_EL1...
```

**关键设计点：** 通用/浮点寄存器统一 memset 清零即可，系统寄存器需要逐个按描述表
设定有意义初始值，而 SVE/PTRAUTH 则根据 features 按需分配 —
features 最终决定了"哪些寄存器真实存在"。

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

**三种寄存器的 reset 策略：**

| 类型 | 举例 | 存储位置 | 初始化时机 | reboot |
|------|------|---------|-----------|--------|
| VM 级 ID 寄存器 | ID_AA64MMFR0_EL1 | kvm->arch.id_regs[] | 首个 vcpu init | 跳过 |
| VCPU 级 ID 寄存器 | MPIDR_EL1 | ctxt.sys_regs[] | 该 vcpu 首次 init | 跳过 |
| 普通系统寄存器 | SCTLR_EL1, TCR_EL1 | ctxt.sys_regs[] | 每次 vcpu init | reset |

设计原因：ID 寄存器描述"这是个什么 CPU"，创建后就固定了，不能因 guest reboot 而变化。
普通寄存器描述"CPU 当前运行状态"，reboot 需要回到初始值。



### 3.3 数据结构初始化结果

经过上述流程，以下数据结构被初始化:

#### 3.3.1 kvm_cpu_context 初始化

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

#### 3.3.2 系统寄存器初始化

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
sys_regs[MPIDR_EL1]     = vcpu->mp_affinity;
sys_regs[ID_AA64MMFR0]  = sanitized host value;
sys_regs[ID_AA64MMFR1]  = sanitized host value;
// ...
```

#### 3.3.3 SVE 状态初始化 (如果启用)

```c
// SVE 寄存器状态
vcpu->arch.sve_state    = kzalloc(vcpu_sve_state_size(vcpu), ...);

// Z 寄存器 (2048-bit × 32)
// P 寄存器 (256-bit × 17, 包括 FFR)
// 全部清零
```

### 3.4 初始化时序总结

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

---

## 4. 寄存器读写接口

KVM 通过 `KVM_SET_ONE_REG` / `KVM_GET_ONE_REG` 两个 ioctl 提供用户空间对 vcpu 所有
寄存器的读写。寄存器 ID 编码（见第 1 章）决定了该请求被分发到哪个处理函数。

### 4.1 ioctl 入口与 COPROC 分发

`arch/arm64/kvm/arm.c:1864`

读写共用一个 ioctl 入口，根据 `ioctl` 参数分叉到 get 或 set：

```c
long kvm_arch_vcpu_ioctl(struct file *filp, unsigned int ioctl, ...)
{
    switch (ioctl) {
    case KVM_SET_ONE_REG:
    case KVM_GET_ONE_REG: {
        struct kvm_one_reg reg;
        if (copy_from_user(&reg, argp, sizeof(reg)))
            break;

        // 处理 pending reset（PSCI 可能触发）
        if (kvm_check_request(KVM_REQ_VCPU_RESET, vcpu))
            kvm_reset_vcpu(vcpu);

        if (ioctl == KVM_SET_ONE_REG)
            r = kvm_arm_set_reg(vcpu, &reg);   // → 写路径
        else
            r = kvm_arm_get_reg(vcpu, &reg);   // → 读路径
        break;
    }
    }
}
```

`kvm_arm_get_reg` / `kvm_arm_set_reg` 位于 `arch/arm64/kvm/guest.c:826/875`，
根据寄存器 ID 中的 COPROC 字段分发到不同子处理函数：

```c
int kvm_arm_get_reg(struct kvm_vcpu *vcpu, const struct kvm_one_reg *reg)
{
    switch (reg->id & KVM_REG_ARM_COPROC_MASK) {
    case KVM_REG_ARM_CORE:    return get_core_reg(vcpu, reg);
    case KVM_REG_ARM_FW:                 // 固件伪寄存器
    case KVM_REG_ARM_FW_FEAT_BMAP:       return kvm_arm_get_fw_reg(vcpu, reg);
    case KVM_REG_ARM64_SVE:   return get_sve_reg(vcpu, reg);
    }
    if (is_timer_reg(reg->id))           return get_timer_reg(vcpu, reg);
    return kvm_arm_sys_reg_get_reg(vcpu, reg);  // 兜底：系统寄存器
}

int kvm_arm_set_reg(struct kvm_vcpu *vcpu, const struct kvm_one_reg *reg)
{
    // Realm 额外校验（略）

    switch (reg->id & KVM_REG_ARM_COPROC_MASK) {
    case KVM_REG_ARM_CORE:    return set_core_reg(vcpu, reg);
    case KVM_REG_ARM_FW:
    case KVM_REG_ARM_FW_FEAT_BMAP:       return kvm_arm_set_fw_reg(vcpu, reg);
    case KVM_REG_ARM64_SVE:   return set_sve_reg(vcpu, reg);
    }
    if (is_timer_reg(reg->id))           return set_timer_reg(vcpu, reg);
    return kvm_arm_sys_reg_set_reg(vcpu, reg);  // 兜底：系统寄存器
}
```

分发策略：先按 COPROC 匹配 CORE/FW/SVE，然后判断是否 Timer 寄存器，
其余全部归为系统寄存器。因为系统寄存器数量远超其他类型，所以"兜底"处理。

`kvm_one_reg` 结构体（`include/uapi/linux/kvm.h`）：
```c
struct kvm_one_reg {
    __u64 id;     // 寄存器 ID（按第 1 章的编码格式）
    __u64 addr;   // 用户空间缓冲区地址（存放/读取寄存器值）
};
```

---

### 4.2 写路径：KVM_SET_ONE_REG

#### 4.2.1 核心寄存器写入：set_core_reg

`arch/arm64/kvm/guest.c:272`

```c
static int set_core_reg(struct kvm_vcpu *vcpu, const struct kvm_one_reg *reg)
{
    __u32 __user *uaddr = (__u32 __user *)(unsigned long)reg->addr;
    __uint128_t tmp;
    void *valp = &tmp, *addr;
    u64 off;

    // 1. 用 ID 计算出在 kvm_regs 结构中的偏移
    off = core_reg_offset_from_id(reg->id);
    if (off >= nr_regs)
        return -ENOENT;

    // 2. 通过 core_reg_addr 将偏移映射到 vcpu 内存地址
    addr = core_reg_addr(vcpu, reg);
    if (!addr)
        return -EINVAL;

    // 3. 从用户空间 copy 值到内核临时变量
    if (copy_from_user(valp, uaddr, KVM_REG_SIZE(reg->id)))
        return -EFAULT;

    // 4. PSTATE 特殊校验：检查写入的模式值是否合法
    if (off == KVM_REG_ARM_CORE_REG(regs.pstate)) {
        u64 mode = (*(u64 *)valp) & PSR_AA32_MODE_MASK;
        switch (mode) {
        case PSR_AA32_MODE_USR:
            if (!kvm_supports_32bit_el0())
                return -EINVAL;
            break;
        case PSR_AA32_MODE_FIQ: ...  // 32-bit 模式逐一检查
        }
    }

    // 5. 将值从临时变量 memcpy 到目标地址
    memcpy(addr, valp, KVM_REG_SIZE(reg->id));
    return 0;
}
```

核心流程：**ID → 偏移 → 内核地址 → copy_from_user → 校验 → memcpy 写入**。

#### 4.2.2 系统寄存器写入：kvm_sys_reg_set_user

`arch/arm64/kvm/sys_regs.c:3861`

```c
int kvm_sys_reg_set_user(struct kvm_vcpu *vcpu, const struct kvm_one_reg *reg,
                         const struct sys_reg_desc table[], unsigned int num)
{
    u64 __user *uaddr = (u64 __user *)(unsigned long)reg->addr;
    const struct sys_reg_desc *r;
    u64 val;

    // 1. 从用户空间取原始值
    if (get_user(val, uaddr))
        return -EFAULT;

    // 2. 在 sys_reg_descs[] 表中二分查找对应描述符
    r = id_to_sys_reg_desc(vcpu, reg->id, table, num);
    if (!r || sysreg_hidden_user(vcpu, r))
        return -ENOENT;

    // 3. 是否忽略写入？（某些寄存器对用户空间只读）
    if (sysreg_user_write_ignore(vcpu, r))
        return 0;

    // 4. 有自定义 set_user → 走自定义（如 ID 寄存器有写掩码校验）
    if (r->set_user)
        ret = (r->set_user)(vcpu, r, val);
    else
        __vcpu_sys_reg(vcpu, r->reg) = val;  // 直接写入 sys_regs[]
    return ret;
}
```

核心流程：**ID → 二分查找 sys_reg_descs[] → 自定义 set_user 或直接写入 sys_regs[r->reg]**。

与 set_core_reg 最大的不同：核心寄存器靠偏移计算地址，系统寄存器靠描述符表查找。

---

### 4.3 读路径：KVM_GET_ONE_REG

#### 4.3.1 核心寄存器读取：get_core_reg

`arch/arm64/kvm/guest.c:243`

```c
static int get_core_reg(struct kvm_vcpu *vcpu, const struct kvm_one_reg *reg)
{
    __u32 __user *uaddr = (__u32 __user *)(unsigned long)reg->addr;
    int nr_regs = sizeof(struct kvm_regs) / sizeof(__u32);
    void *addr;
    u32 off;

    // 1. ID → 偏移
    off = core_reg_offset_from_id(reg->id);
    if (off >= nr_regs)
        return -ENOENT;

    // 2. 偏移 → 内核地址（与 set 共用 core_reg_addr）
    addr = core_reg_addr(vcpu, reg);
    if (!addr)
        return -EINVAL;

    // 3. 从内核地址 copy 到用户空间
    if (copy_to_user(uaddr, addr, KVM_REG_SIZE(reg->id)))
        return -EFAULT;

    return 0;
}
```

比 set 简单很多：不需要校验（是从内核往外读），不需要临时变量，直接 `copy_to_user`。

#### 4.3.2 系统寄存器读取：kvm_sys_reg_get_user

`arch/arm64/kvm/sys_regs.c:3825`

```c
int kvm_sys_reg_get_user(struct kvm_vcpu *vcpu, const struct kvm_one_reg *reg,
                         const struct sys_reg_desc table[], unsigned int num)
{
    u64 __user *uaddr = (u64 __user *)(unsigned long)reg->addr;
    const struct sys_reg_desc *r;
    u64 val;

    // 1. 二分查找描述符
    r = id_to_sys_reg_desc(vcpu, reg->id, table, num);
    if (!r || sysreg_hidden_user(vcpu, r))
        return -ENOENT;

    // 2. 读取：有自定义 get_user 则走自定义，否则直接读 sys_regs[]
    if (r->get_user)
        ret = (r->get_user)(vcpu, r, &val);
    else
        val = __vcpu_sys_reg(vcpu, r->reg);

    // 3. 返回给用户空间
    return put_user(val, uaddr);
}
```

---

### 4.4 辅助：核心寄存器 ID → 地址映射

读和写都依赖 `core_reg_addr`（`arch/arm64/kvm/guest.c:178`），它把寄存器 ID 映射到
`kvm_cpu_context` 中的真实内存地址：

```c
static void *core_reg_addr(struct kvm_vcpu *vcpu, const struct kvm_one_reg *reg)
{
    u64 off = core_reg_offset_from_id(reg->id);

    switch (off) {
    case KVM_REG_ARM_CORE_REG(regs.regs[0]) ...
         KVM_REG_ARM_CORE_REG(regs.regs[30]):
        off -= KVM_REG_ARM_CORE_REG(regs.regs[0]);
        off /= 2;
        return &vcpu->arch.ctxt.regs.regs[off];   // X0-X30

    case KVM_REG_ARM_CORE_REG(regs.sp):
        return &vcpu->arch.ctxt.regs.sp;

    case KVM_REG_ARM_CORE_REG(regs.pc):
        return &vcpu->arch.ctxt.regs.pc;

    case KVM_REG_ARM_CORE_REG(regs.pstate):
        return &vcpu->arch.ctxt.regs.pstate;

    case KVM_REG_ARM_CORE_REG(sp_el1):
        return __ctxt_sys_reg(&vcpu->arch.ctxt, SP_EL1);  // 跨界：指向 sys_regs[]

    case KVM_REG_ARM_CORE_REG(elr_el1):
        return __ctxt_sys_reg(&vcpu->arch.ctxt, ELR_EL1);

    case KVM_REG_ARM_CORE_REG(spsr[KVM_SPSR_EL1]):
        return __ctxt_sys_reg(&vcpu->arch.ctxt, SPSR_EL1);

    // 浮点寄存器：
    case KVM_REG_ARM_CORE_REG(fp_regs.vregs[0]) ...
         KVM_REG_ARM_CORE_REG(fp_regs.vregs[31]):
        off -= KVM_REG_ARM_CORE_REG(fp_regs.vregs[0]);
        off /= 4;
        return &vcpu->arch.ctxt.fp_regs.vregs[off];   // V0-V31

    default:
        return NULL;
    }
}
```

注意一个设计细节：**CORE 类型是个大杂烩**，虽然 ID 里的 COPROC 是 `KVM_REG_ARM_CORE`，
但实际映射到的地址可能跨多个子结构 —— `regs.regs[]`、`fp_regs.vregs[]`、甚至
`sys_regs[]`（SP_EL1、ELR_EL1、SPSR_EL1 都指向了 sys_regs）。

---

### 4.5 读写流程对比

```
SET_ONE_REG                          GET_ONE_REG
    │                                    │
    ├─ kvm_arm_set_reg                   ├─ kvm_arm_get_reg
    │   └─ COPROC 分发                   │   └─ COPROC 分发
    │       ├─ set_core_reg              │       ├─ get_core_reg
    │       │   ID→偏移→core_reg_addr    │       │   ID→偏移→core_reg_addr
    │       │   copy_from_user→memcpy     │       │   copy_to_user
    │       │                            │       │
    │       └─ kvm_sys_reg_set_user      │       └─ kvm_sys_reg_get_user
    │           ID→id_to_sys_reg_desc    │           ID→id_to_sys_reg_desc
    │           set_user 或直接赋值       │           get_user 或直接读取
    │           sys_regs[r->reg]=val     │           val=sys_regs[r->reg]
```

**核心区别：**
- 核心寄存器：靠固定偏移表映射地址，读写走 `copy_from/to_user`
- 系统寄存器：靠二分查找描述符表，可能有自定义 get_user/set_user 回调


---



---
## 5. Guest 读写寄存器的两种路径

前面第 4 章讲的是**QEMU 用户空间**如何通过 ioctl 读写寄存器。本章讨论 **guest 运行时**
如何访问寄存器——分为完全不 trap 和 trap 到 KVM 两种完全不同的机制。

### 5.1 不 trap：guest 直接在硬件读写

通用寄存器（X0-X30、SP、PC、PSTATE）和浮点寄存器（V0-V31、FPSR、FPCR）在 guest
运行时**直接映射到物理 CPU 的硬件寄存器**：

```
guest 代码:  add x0, x1, x2
                 ↓
物理 CPU:    硬件执行 ADD 指令，读取 X1、X2，写回 X0
                 ↓
             KVM 完全不参与，零开销
```

为什么可以不 trap？因为这些寄存器**没有副作用**，不会影响 guest 之外的任何状态。
KVM 只需要在 VM exit/entry 时做保存恢复：

```
VM exit 时:   硬件自动保存 guest GPRs → 内存
              KVM 不需要手动干预（硬件做了）

VM entry 时:  KVM 把 GPRs 从内存恢复到硬件寄存器
              然后 ERET 回 guest
```

---

### 5.2 trap：系统寄存器走 KVM 模拟

当 guest 执行 `MRS` 或 `MSR` 访问系统寄存器时，硬件检查 **HCR_EL2** 和 **CPTR_EL2**
决定是否 trap 到 EL2（KVM）：

```
guest 代码:  MRS x0, SCTLR_EL1
                 ↓
硬件检查 HCR_EL2 的 trap 位
  │
  ├── trap 位 = 0 → guest 直接读硬件 SCTLR_EL1（不经过 KVM，快）
  │
  └── trap 位 = 1 → trap 到 EL2 (KVM)
                      ↓
                   KVM 解码 ESR → 找出 Op0/Op1/CRn/CRm/Op2
                      ↓
                   在 sys_reg_descs[] 中二分查找
                      ↓
                   调用 access() 回调
                      ↓
                   从 sys_regs[] 读值 → 写入 guest X0
                      ↓
                   ERET 回 guest，PC+4
```

---

### 5.3 哪些 trap 由 HCR_EL2 控制？

`vcpu_reset_hcr` 在 vcpu init 时设置 trap 位（`arch/arm64/include/asm/kvm_emulate.h:70`）：

```c
static inline void vcpu_reset_hcr(struct kvm_vcpu *vcpu)
{
    vcpu->arch.hcr_el2 = HCR_GUEST_FLAGS;
    if (has_vhe()) vcpu->arch.hcr_el2 |= HCR_E2H;
    if (cpus_have_const_cap(ARM64_HAS_RAS_EXTN)) {
        vcpu->arch.hcr_el2 |= HCR_TEA;    // trap RAS error record
        vcpu->arch.hcr_el2 |= HCR_TERR;
    }
    if (cpus_have_const_cap(ARM64_HAS_STAGE2_FWB))
        vcpu->arch.hcr_el2 |= HCR_FWB;
    else
        vcpu->arch.hcr_el2 |= HCR_TVM;    // trap SCTLR_EL1 直到 MMU 打开
    // ... TID4/TID2 for CTR_EL0 trap
}
```

`HCR_GUEST_FLAGS` 包含的 trap 位（`arch/arm64/include/asm/kvm_arm.h:104`）：

```c
#define HCR_GUEST_FLAGS (                        HCR_TSC  | HCR_TSW | HCR_TWE | HCR_TWI | /* SMC/SWI/WFI/WFE  trap      */
    HCR_VM   |                                /* Stage 2 MMU 启用            */
    HCR_TACR | HCR_AMO | HCR_SWIO |          /* cache/TLB 维护            trap */
    HCR_TIDCP| HCR_TLOR |                     /* imp-def/lor               trap */
    HCR_TID3 | HCR_TID1 |                     /* ID 寄存器 group 3/1       trap */
    HCR_FMO  | HCR_IMO |                      /* 物理中断路由到 EL2          */
    HCR_PTW  |                                /* 设备内存 stage1 walk     trap */
    HCR_RW   |                                /* 64-bit EL1                  */
    HCR_BSU_IS | HCR_FB)                      /* barrier/TLB 广播            */
```

**每个位对应一类 trap。** 比如 `HCR_TID3`：group 3 的 ID 寄存器的 MRS/MSR 一律 trap。
`HCR_TVM`：SCTLR_EL1、TTBRx_EL1、TCR_EL1 等虚拟内存控制寄存器 trap 到 EL2。

---

### 5.4 trap 后怎么分发：kvm_handle_sys_reg

`arch/arm64/kvm/sys_regs.c:3683`

```c
int kvm_handle_sys_reg(struct kvm_vcpu *vcpu)
{
    struct sys_reg_params params;
    unsigned long esr = kvm_vcpu_get_esr(vcpu);  // 读取 ESR_EL2
    int Rt = kvm_vcpu_sys_get_rt(vcpu);           // 目标 GPR（MRS 写哪个 Xn）

    // 从 ESR 解码出 Op0/Op1/CRn/CRm/Op2 + 读写方向
    params = esr_sys64_to_params(esr);

    if (params.is_write)
        params.regval = vcpu_get_reg(vcpu, Rt); // MSR：取 Xn 值

    // 在 sys_reg_descs[] 中二分查找，调用 access 回调
    if (!emulate_sys_reg(vcpu, &params))
        return 1;

    if (!params.is_write)
        vcpu_set_reg(vcpu, Rt, params.regval);  // MRS：结果写回 Xn
    return 1;
}
```

**ESR_EL2（Exception Syndrome Register）** 是硬件在 trap 时自动填写的寄存器，记录了
trap 原因。`esr_sys64_to_params`（`sys_regs.h:37`）直接从中解码出五字段：

```c
#define esr_sys64_to_params(esr)                            ((struct sys_reg_params){                                   .Op0 = ((esr) >> 20) & 3,      /* 硬件已填好 Op0 */          .Op1 = ((esr) >> 14) & 0x7,    /* 硬件已填好 Op1 */          .CRn = ((esr) >> 10) & 0xf,                              .CRm = ((esr) >>  1) & 0xf,                              .Op2 = ((esr) >> 17) & 0x7,                              .is_write = !((esr) & 1) })   /* bit0=0 写, bit0=1 读 */
```

**这就是前面第 1 章说的"ARM 硬件编码可以直接复用"的具体体现——硬件把 Op0/Op1/CRn/CRm/Op2
编码进了 MRS/MSR 指令，trap 时又写入了 ESR，KVM 直接解码即可，不需要额外查表。**

---

### 5.5 二分查找 + access 回调

`emulate_sys_reg`（`sys_regs.c:3607`）：

```c
static bool emulate_sys_reg(struct kvm_vcpu *vcpu, struct sys_reg_params *params)
{
    // 二分查找 sys_reg_descs[]
    const struct sys_reg_desc *r;
    r = find_reg(params, sys_reg_descs, ARRAY_SIZE(sys_reg_descs));

    if (likely(r)) {
        perform_access(vcpu, params, r);   // 找到了，调用 access 回调
        return true;
    }
    kvm_inject_undefined(vcpu);            // 找不到 → 注入未定义异常
    return false;
}
```

`perform_access`（`sys_regs.c:3276`）：

```c
static void perform_access(struct kvm_vcpu *vcpu, struct sys_reg_params *params,
                           const struct sys_reg_desc *r)
{
    if (sysreg_hidden(vcpu, r)) {          // 运行时检查：该寄存器是否被隐藏
        kvm_inject_undefined(vcpu);
        return;
    }
    if (likely(r->access(vcpu, params, r)))
        kvm_incr_pc(vcpu);                  // access 成功 → PC+4，执行下一条
}
```

---

### 5.6 两种路径汇总

```
                        guest 读写寄存器
                              │
                    ┌─────────┴─────────┐
                    │                   │
                 通用/FP              系统寄存器
             (X0-X30, V0-V31)    (MRS/MSR 访问)
                    │                   │
                    ▼                   ▼
              硬件直接执行        硬件检查 HCR_EL2
                （零开销）               │
                               ┌───────┴───────┐
                               │               │
                            trap=0          trap=1
                         硬件直接读写     trap → EL2 (KVM)
                           （快）             │
                                       解码 ESR → 五字段
                                            │
                                      二分查找 sys_reg_descs[]
                                            │
                                       access() 读/写 sys_regs[]
                                            │
                                       ERET 回 guest
```

**性能量级：**
- 通用寄存器 / 不 trap 的系统寄存器：0 额外 cycle（硬件原生执行）
- trap 的系统寄存器：一次 VM exit + 处理 + VM entry，~几百到上千 cycle


ARM64 KVM-QEMU 寄存器虚拟化的核心要点：

1. **统一的寄存器 ID 编码**：KVM 通过标准化的 64 位 ID 标识所有寄存器
2. **分层的数据结构**：内核使用 `kvm_vcpu_arch` → `kvm_cpu_context` 存储寄存器
3. **初始化流程**：`kvm_reset_vcpu()` 遍历寄存器描述表设置初始值
4. **同步机制**：QEMU 通过 cpreg 列表作为中间层同步系统寄存器
5. **ioctl 接口**：`KVM_SET_ONE_REG/KVM_GET_ONE_REG` 是用户空间修改寄存器的唯一入口