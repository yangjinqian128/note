# 笔记：arm64 FEAT_NMI RFC v2 Patch Set 分析

> 链接: https://lore.kernel.org/linux-arm-kernel/20260727163453.7969-1-vladimir.murzin@arm.com/
> 作者: Vladimir Murzin (Arm)，主要贡献者 Ada Couprie Diaz (20)、Mark Brown (5)、Lorenzo Pieralisi (2)
> 版本: RFC v2，45 patches，基于 v7.2-rc5（2026-07-27 发布）
> RFC v1: 2026-07-09，https://lore.kernel.org/linux-arm-kernel/20260709121333.23507-1-vladimir.murzin@arm.com/
> 规模: 50 个文件，+1734/-614；新增 `arch/arm64/include/asm/interrupts/{common_flags,entry,masking}.h`，删除 `daifflags.h`

---

## 1. 背景与目标

FEAT_NMI（ARMv8.8 / ARMv9.4，PE 侧）配合 FEAT_GICv3_NMI（GIC 侧）提供**架构化的 NMI**（superpriority 中断 + `PSTATE.ALLINT` 屏蔽 + `ICC_NMIAR1_EL1` 应答寄存器）。内核自 2019 年起已有基于 GIC 优先级屏蔽（ICC_PMR_EL1）的 **pseudo-NMI**（CONFIG_ARM64_PSEUDO_NMI）。

系列的核心论点（cover letter）：

> "Since we already support pseudo-NMIs via priority masking, introducing another flavour of NMI on top of the existing infrastructure could easily become messy, making the code harder to follow and reason about."

因此先**重构现有的异常屏蔽逻辑**，为 FEAT_NMI 这个新"租户"腾出空间。核心思想：

> "separate the logical view of exception state from its hardware representation" —— 引入**逻辑异常上下文**（logical exception contexts），映射到具体硬件状态；硬件相关的处理收敛到少数几个地方，其余代码只按逻辑上下文操作。

由于重构非平凡、容易引入微妙行为变化，系列在 `CONFIG_DEBUG_IRQFLAGS` 下加了**大量一致性校验**（硬件状态必须与期望的逻辑状态匹配）。

## 2. 核心设计

### 2.1 三种硬件状态表示（对应三种运行模式）

| 屏蔽机制 | 由谁控制 | 说明 |
|---|---|---|
| DAIF（PSTATE） | PE | 传统屏蔽，I/F/A/D 位 |
| ICC_PMR_EL1 | GIC CPU interface | pseudo-NMI：按优先级在到达 CPU 前屏蔽 |
| ALLINT（PSTATE bit 13） | PE | FEAT_NMI：`SCTLR_EL1.{SPINTMASK=0, NMI=1}` 时 ALLINT 屏蔽 superpriority + 普通 IRQ/FIQ |

`arm64_exc_hwstate_t`（patch 10）：union { struct { u16 daif; u8 pmr; u8 pad[5] }; unsigned long flags; }。**一个 `unsigned long` 里同时编码 DAIF 和 PMR**（ALLINT 复用 daif 字段的 bit 13 = `PSR_ALLINT_BIT`，见 `save_and_disable_exceptions` 中 `orr \flags, \flags, \tmp`）。irqflags 通用 API 的 `unsigned long` 语义保持不变，但 arch 内部可以并行跟踪两种机制 —— 这是对现状（同一 flags 只跟踪一种机制）的根本修正。

### 2.2 逻辑异常上下文（patch 12，`common_flags.h`）

```c
typedef enum arm64_exc_context {
    PROCESS_CONTEXT,   // 0 屏蔽，正常任务上下文
    NOIRQ_CONTEXT,     // 屏蔽所有普通 IRQ，但 NMI 可达
    NONMI_CONTEXT,     // 屏蔽 IRQ+FIQ（NMI 可达）
    ERROR_CONTEXT,     // 屏蔽 IRQ+FIQ+SError（serror 处理）
    CRITICAL_CONTEXT,  // 全屏蔽（异常入口状态）
} arm64_exc_context_t;
```

映射到硬件状态的三个变体（`arm64_exc_hwstate_of_context()`）：

| 逻辑上下文 | 无 NMI | pseudo-NMI | FEAT_NMI |
|---|---|---|---|
| CRITICAL | DAIF 全置 | DAIF 全置 + PMR=IRQON | DAIF 全置 + ALLINT |
| ERROR | A+I+F | A+I+F + PMR=IRQON | A+I+F + ALLINT |
| NONMI | I+F | I+F + PMR=IRQON | I+F + ALLINT |
| NOIRQ | I+F | 0 + **PMR=IRQOFF** | **I+F** |
| PROCESS | 0 | 0 + PMR=IRQON | 0 |

关键差异一目了然：**pseudo-NMI 用 PMR 实现 NOIRQ，而 FEAT_NMI 用 I+F（DAIF）实现 NOIRQ、用 ALLINT 实现"连 NMI 一起屏蔽"**。这正是当前代码混乱的根源：现有实现里 `local_daif_save()` 和 `local_irq_save()` 在不同模式下语义错位（`I` 位一会儿是"屏蔽 IRQ"，一会儿是"允许 NMI"）。

### 2.3 Entry 专用 API（patch 14，`entry.h`）

Entry 代码与内核其余部分的屏蔽模式相反（entry 从全屏蔽"下降"到目标上下文，退出时"抬升"回全屏蔽；普通代码是抬升-恢复），所以单独提供：

- `arm64_unmask_exc_context(ctx)` / `arm64_mask_exc_context(prev)` / `arm64_inherit_exc_context(regs)` / `arm64_drop/lift_exc_context()`
- 每次切换内建 `arm64_debug_exc_hwstate(prev)` 校验 + `CONFIG_DEBUG_IRQFLAGS` 下对非法上下文迁移的 `WARN_ON_ONCE`（如从 PROCESS 抬升到 NOIRQ 是不合法的）
- `trace_hardirqs_on/off` 在切换 helper 内统一处理，不再散落各处

### 2.4 通用 masking API（patch 16，`masking.h`）

- `local_exceptions_save_mask()` / `local_exceptions_restore()` —— **必须成对使用**，save 时记录"原始 + 请求"状态，restore 时校验一致性（检测 save/restore 之间屏蔽状态被意外改动）。
- `local_exceptions_cpu_init_mask()` / `local_exceptions_final_mask()` —— 成对语义不适用的两个场景：CPU 初始化、CPU 下线。
- 替换全部 `local_daif_*`（patch 18-19，含 `apei_claim_sea()` 的逻辑重写、KVM nVHE switch.c 中 `GIC_PRIO_PSR_I_SET` 的删除）。

### 2.5 FEAT_NMI 使能路径（patch 30-37）

- **cpufeature**（patch 31）：`ARM64_HAS_NMI`（`ID_AA64PFR1_EL1.NMI`，boot CPU feature）+ `ARM64_NMI`（`can_use_nmi()`：要求 HAS_NMI 且 **pseudo-NMI 未启用** —— pseudo-NMI 优先，两者互斥）。`nmi_enable()`：`_allint_clear()` 后 `SCTLR_EL1.SPINTMASK=0, NMI=1`。
- **ALLINT 操作**（patch 28/32）：`SYS_ALLINT_SET/CLR` 立即数编码（`sys_reg(0,1,4,0/1,0)`，xzr 源）；entry.S 的 `save_and_disable_exceptions`/`restore_exceptions` 用 alternative 在 `ARM64_NMI` 下保存/恢复 ALLINT（存于 flags bit 13）。
- **屏蔽顺序**（`__arm64_update_exc_hwstate`）：屏蔽时先 `ALLINT_SET` 再写 DAIF；解除时先写 DAIF 再 `ALLINT_CLR` —— 保证不会出现"DAIF 已屏蔽但 NMI 认为已解除"或反向的窗口。
- **NMI 判定**（patch 34）：`is_nmi()` = `ISR_EL1.IS` 置位；`el1_interrupt`/`el0_interrupt` 据此分流到 `__el1_nmi`（`irqentry_nmi_enter/exit`）/ `__el1_irq`。FIQ 路径在 FEAT_NMI 下 `WARN_ON_ONCE(ISR_EL1.FS)`。
- **IRQ 路径简化**：无 NMI → 直接 `NOIRQ_CONTEXT`；pseudo-NMI → 保持 `NONMI_CONTEXT` 直到 GIC 判明优先级；FEAT_NMI → **直接 NOIRQ_CONTEXT**（NMI 走独立路径、不共享 IRQ 处理状态）。
- **irqflags**（patch 33）：`arch_local_save_flags()` 同时读 ALLINT；`arch_irqs_disabled_flags()` 依次检查 I 位、ALLINT、PMR。`regs_irqs_disabled()` 同样扩展。
- **配套**：kprobes 保存/恢复 ALLINT（patch 37）；suspend `INIT_PSTATE_EL1` 加 `PSR_ALLINT_BIT`（patch 35）；EFI runtime 检查含 ALLINT（patch 36）；ptrace uapi `PSR_ALLINT_BIT`（patch 29）；idreg override `id_aa64pfr1.nmi=`（patch 30）；booting.rst 要求 EL2 下 `HCRX_EL2.TALLINT=0`（patch 27）。

### 2.6 GIC 侧（patch 41-42, 44-45）

- **FEAT_GICv3_NMI**（patch 42）：`GICD_TYPER.NMI` 检测；per-interrupt NMI 使能位 **GICD_INMIR**（0x0F80 / nE 0x3B00，受 raw spinlock 保护的 RMW）；NMI 应答用 **`ICC_NMIAR1_EL1`**；`gic_handle_irq()` 中 `in_nmi()` 时走 `__gic_handle_nmi()` 独立路径（不再靠 RPR/优先级猜测）。
- **pseudo-NMI 与 FEAT_NMI 的优先级**（`gic_enable_nmi_support()`）：GIC 有 NMI 能力 → 用真 NMI；否则回落到 `gic_enable_pseudo_nmi()`。
- **GICv5（patch 44/45）**：PPI/SPI/LPI 通过**在 superpriority 与普通优先级之间切换**实现 NMI；IPI chip 加 `irq_nmi_setup/teardown`（委托 parent_data）。`irq_supports_nmi()` 去掉"只能 root irqchip"限制，允许层级继承 —— **Marc Zyngier 意见：干脆删除层级检查，信任本地 irqchip**（v3 会改）。
- 通用代码：`handle_irq_event_percpu()` 在 `in_nmi()` 时跳过 `add_interrupt_randomness()`。

### 2.7 IPI 侧（patch 38-39, 43）

- `ipi_irq_ops`（setup/enable/disable/send）抽象 SGI 与 LPI 两条路径（patch 38）。
- **NMI 请求失败回退**（patch 39）：`ipi_should_be_nmi()` 降级为"提示"；`request_percpu_nmi()`/`request_nmi()` 失败则回退普通 IRQ，per-CPU `nmi_bitmap` 记录实际结果（解决"CPU 有 FEAT_NMI 但 GIC 没有"的场景）。
- LPI-backed IPI 的 NMI 支持（patch 43）：`enable_nmi()`/`disable_nmi_nosync()` + `IRQF_PERCPU`。

### 2.8 清理类 patch（1-29 中穿插）

删除了三个历史包袱：`daifflags.h`（patch 20）、`CONFIG_ARM64_DEBUG_PRIORITY_MASKING`（patch 24，并入 `CONFIG_DEBUG_IRQFLAGS`，patch 23）、**`GIC_PRIO_PSR_I_SET`**（patch 22 —— 把 `PSR_I_BIT` 塞进 PMR 值骗过 irqflags 的 hack）。另外把 `init_IRQ()` 中依赖 `local_daif_restore()` 副作用的"DAIF→PMR 切换"改成显式的 `gic_prio_init()` helper（patch 17）；cpuidle 的 WFI 前 PMR 旁路改用新 API（patch 19）。

## 3. 上游状态（2026-08-18）

- **Will Deacon 已合入 patch 1-7** 到 `arm64/for-next/nmi`（8 月 6 日，commit b7f741717d1e 等），**8-9** 于 8 月 11 日合入（067f029c6463、39aebe0e8946）。patch 8/9 顺带修复了 Breno Leitao 报告的 **pNMI splat**（https://lore.kernel.org/all/20260807-arm64_fix-v1-1-d069ccf9d71b@debian.org/）—— 说明当前 pseudo-NMI 实现里存在真实 bug。
- **Jinjie Ruan（华为）**对除 gicv5 外的全部 patch 做了详细 review：结论"除代码风格、拆分和个别实现细节外没有大问题"，给了多个 Reviewed-by。
- Marc Zyngier 对 08/45 给了 Reviewed-by；对 45/45 提出 `irq_supports_nmi()` 应删除层级检查。
- 测试状态：主要在 QEMU / Fast Model，**真机（无 NMI / pseudo-NMI / FEAT_NMI 三种形态）待测**。

## 4. 值得关注的点 / 开放问题

1. **设计重心是"可论证的正确性"而非新功能**：逻辑上下文 + 迁移校验（WARN）+ save/restore 配对校验，试图把当前散落在 entry-common.c、irqflags.h、gic-v3.c、smp.c 里的隐式状态假设显式化。
2. **pseudo-NMI 与 FEAT_NMI 互斥**（pNMI 优先）——同一内核镜像可跑三种硬件，但一次只用一种。这对发行版（如 openEuler）意味着无需新 config 变体，`CONFIG_ARM64_NMI` default y。
3. 兼容性边界：`arch_irqentry_exit_need_resched()` 仍需读 `SYS_ALLINT` 判断 NMI；kprobes/suspend/EFI/hibernate 每个子系统都要感知新状态位 —— 状态位越多的架构，这些边角越容易漏。
4. gic-v5 支持作者自己都说是"illustration"，真机上（尤其 ITS/CMDQ 路径）风险高。
5. 尚未覆盖：SDEI（pNMI 时代就被禁用，FEAT_NMI 下关系未讨论）、perf hardlockup detector 的接入方式、KVM 中 guest 的 NMI 暴露（当前系列只动了 hyp 的屏蔽保存）。

## 5. 对 openEuler 的参考意义

- 华为（Ruan Jinjie）是本系列的主要 reviewer 之一，说明华为/欧拉社区在 arm64 NMI 方向有持续投入（openEuler 曾回移过 pseudo-NMI 相关补丁）。
- 若系列最终合入主线（预计 v7.3+），openEuler 各内核分支（当前 6.6）回移成本较高（重构面大：irqflags、entry、GIC），但 patch 1-9 的清理部分是低风险、可独立回移的候选。
- `CONFIG_ARM64_NMI` default y + 运行时检测的设计，意味着未来 NMI 能力不再需要用户显式开 `irqchip.gicv3_pseudo_nmi=1` 之类的命令行参数，对产品化更友好。
