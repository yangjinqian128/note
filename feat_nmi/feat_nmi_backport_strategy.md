# 笔记：主线 FEAT_NMI v2 方案回合 openEuler 6.6 的策略分析

> 问题：将 Murzin RFC v2（45 patches）回合到 openEuler 6.6 内核，最佳方案是什么？整体回合还是部分回合？部分回合时哪些是必须的？
> 日期：2026-08-18。分析对象：`/home/code/kernel`（openEuler 6.6.0）

---

## 1. 结论先行

**推荐：分期分批回合，不整体回合。**

| 档位 | 内容 | 时机 |
|---|---|---|
| 第一档（必须） | 清理与 bug 修复组：patch 1-9（已核查大部分未回移）、29、35、37、39 思想的适配版、27 | 现在即可，风险低 |
| 第二档（强烈建议） | "ALLINT 进 irqflags" 的最小化适配（patch 10/11/33 思想的 6.6 老框架版）+ 屏蔽顺序修正 | 第一档完成后，重点回归后 |
| 第三档（暂缓） | 框架重构主体（12/14/15/16/18-26/28）、entry NMI 重构（34）、smp/gic-v5（38/43/44/45） | 等上游 v3 定稿 + 先回移 Marc 2025 pNMI 重构底座，或在新内核分支上直接跟随上游 |

## 2. 为什么不能整体回合

### 2.1 系列本身未定稿
RFC v2 还在 bikeshed 阶段：Marc Zyngier 已要求删除 `irq_supports_nmi()` 的层级检查（v3 必改）；命名（`arm64_exc_*`、`ipi_irq_ops`）待议；gic-v5 部分作者自称"illustration"。整体回合一个 RFC 到产品内核，意味着 v3/v4 每版都要追，回移成本翻倍且无法保证 ABI/行为稳定。

### 2.2 基础版本鸿沟（patch 10-29 无法直接应用）
系列基于 v7.2-rc5，其重构对象（irqflags/daifflags/entry-common/gic-v3）在 6.6 中处于**截然不同的形态**：

| 上游（系列假设的基础） | openEuler 6.6 现状 |
|---|---|
| 2025 Marc Zyngier pNMI 重构：runtime selectable、`ppi_nmi_refs` 已删、PPI 统一 `handle_percpu_devid_irq` | 老框架：`ppi_nmi_refs`、`handle_percpu_devid_fasteoi_nmi`、`GIC_PRIO_PSR_I_SET`、`CONFIG_ARM64_DEBUG_PRIORITY_MASKING` **全在** |
| entry-common.c 接近纯净上游（+347/-xx 行重构的基座） | 被华为 FAST_IRQ/Xint 重度修改：`el0_xint`、`gic_handle_irq_noack`、自研 `arm64_enter_nmi`（含 `ct_nmi_enter`）交织其中 |
| smp.c 有 per-CPU `ipi_descs` + LPI-backed IPI | 老的 `ipi_desc[NR_IPI]` + SGI（无 `percpu_ipi_descs`/`ipi_setup_lpi`），patch 38/43 无根基 |
| `handle_arch_nmi_irq` 等自研分流不存在 | 有（华为自研），与系列的主线式分流互斥 |

因此整体回合的**实际成本 = 先回移 Marc 2025 pNMI 重构（约 15+ patches，还依赖 genirq 演进） + 重做 FAST_IRQ/Xint 适配 + 再应用 45 patches 并适配**。这不是"回合一个系列"，是三个系列加一次重构。

### 2.3 等价物已部分存在，整体回合会重复/冲突
已核查（`/home/code/kernel`）：
- `idreg-override.c` 已有 `FIELD("nmi", ...)`（≈ patch 30）；
- Jinjie 2026-06 的 "Fix ALLINT masking logic by decoupling from GIC NMI support" 已在树（部分覆盖 patch 33/42 的意图）；
- hulk 补丁 "arm64/entry: Mask DAIF in cpu_switch_to(), call_on_irq_stack()"（!17730）部分覆盖 patch 7 意图；
- GIC 的 FEAT_GICv3_NMI 实现（INMIR/NMIAR1/`gic_setup_nmi_handler`）已存在（Mark Brown 2022 原版 + 华为修复）—— patch 41/42 对 openEuler 而言基本是**已实现功能**。

## 3. 第一档：必须回合清单（已逐项核查 6.6 树状态）

| 原 patch | 内容 | 6.6 树核查结果 | 必须回合的理由 |
|---|---|---|---|
| 02 | debug-monitors: mdscr_write 不屏蔽 DAIF | **未回移**（仍是 `local_daif_save/restore` 版，debug-monitors.c:37-43） | 上游已合（7cf2d6efb86f）；消除伪 NMI 屏蔽窗口 |
| 03/04 | hibernate: mask DAIF + 错误路径恢复 DAIF | **未回移**（hibernate.c:338/388 仍是老用法） | 真实 bug 修复；hibernate 恢复后 DAIF/PMR 状态错误会直接破坏 pNMI/NMI |
| 05/06 | suspend: 依赖 daif helper 处理 PMR + resume 时初始化 PMR | **未回移**（suspend.c:116/162 老用法，无 PMR 初始化） | pNMI 下 resume 后 PMR 状态不确定 → 后续中断可能被永久屏蔽 |
| 07 | entry: 从 C EL1 handler 返回前 mask DAIF | 部分等价（hulk !17730 覆盖 cpu_switch_to/call_on_irq_stack 路径） | 评估等价性后补齐缺口；上游已合（0d774e0517f3） |
| 08 | gic-v3: `gic_unmask_pnmis()` 显式化 | **未回移**（`gic_arch_enable_irqs()` 仍在 2 处：irq-gic-v3.c:938 和 **1043 的 FAST_IRQ noack 路径**） | **修复 Breno 报告的 pNMI splat**（上游 067f029c6463）。注意 openEuler 有两处调用点，FAST_IRQ 路径（`gic_handle_irq_noack`）也要同步改并单独验证 |
| 09 | entry: kernel exit 避免多余 `local_irq_disable()` | **未回移** | 与 08 配套的同一 bug 修复（上游 39aebe0e8946）；依赖 08 先合 |
| 29 | uapi: `PSR_ALLINT_BIT` | **缺失**（uapi/asm/ptrace.h 无） | 一行；pstate 打印/用户态 ABI 完整性 |
| 35 | suspend: `INIT_PSTATE_EL1` 加 `PSR_ALLINT_BIT` | **缺失** | 一行；suspend 恢复后 ALLINT 未屏蔽 → NMI 状态错误 |
| 37 | kprobes: 保存/恢复 ALLINT | **缺失**（kprobes.c 仅 DAIF_MASK） | 独立小 patch；NMI 上下文中 kprobe 单步的正确性 |
| 39 思想 | IPI NMI 请求失败回退普通 IRQ | **无此机制**（`set_smp_dynamic_ipi()` 失败即静默放弃） | 需**适配**到 `ipi_nmi.c`（不依赖 patch 38 的 ops 抽象）：request 失败 → 回退普通 IRQ backtrace + 记录实际模式；顺带修 Kconfig 错误绑定（见下） |
| — | Kconfig: `IPI_AS_NMI` 解绑 `ARM64_PSEUDO_NMI` | 现状：`depends on ARM64_PSEUDO_NMI`（Kconfig:2604-2606） | 纯硬件 NMI 平台（未开 PSEUDO_NMI）也能用 NMI backtrace/kgdb——`gic_irq_nmi_setup()` 已支持 `has_v3_3_nmi()` 路径，只是被 Kconfig 挡了 |
| 01 | ptrace: Remove `INIT_PSTATE_EL2` | **未回移**（ptrace.h:21 仍在） | 死代码清理 |
| 27 | booting.rst: HCRX_EL2.TALLINT 要求 | — | 纯文档 |

第一档除 08→09 的先后依赖外，各 patch 相互独立、冲突面小（08 的 FAST_IRQ 适配是唯一需要华为侧验证的点）。

## 4. 第二档：强烈建议——"ALLINT 进 irqflags"最小化适配

**不引入完整框架，只把根因修掉。**

### 4.1 为什么必须做
第一档修的是症状，第二档修的是根因。6.6 树中 ALLINT 类 bug 的共同根因是：**`unsigned long flags` 里没有 ALLINT**，导致：
- `local_daif_restore()` 用 `PSR_A_BIT` 隐式推断 ALLINT（daifflags.h:126-128）——A 位与 NMI 屏蔽无语义关联，NMI 上下文中配对 save/restore 会错误清 ALLINT；
- `entry.S` 三处 `save_and_disable_daif`（1073/1124/1142）不保存 ALLINT；
- `arch_local_save_flags()` 无法表达"NMI 屏蔽中"状态。

2026-06 该区域反复 revert 4 次，说明补丁式修复已经到头。

### 4.2 做法（patch 10/11/33 思想的 6.6 老框架版）
1. `irqflags.h`：`arch_local_save_flags()` 在 `system_uses_nmi()` 时把 `SYS_ALLINT` 的 bit 13 并入 flags（`arm64_exc_hwstate_t` union 思路的简化版，无需引入新头文件）；
2. `daifflags.h`：
   - `local_daif_save_flags()` 读 ALLINT 并入 flags；
   - `local_daif_restore()` 按 **flags 中的 ALLINT 位**恢复（替换 A 位推断）；
   - `local_daif_inherit()` 同步处理；
3. `assembler.h`/`entry.S`：`save_and_disable_daif` 三处改用保存 ALLINT 的变体（patch 32 的 `alternative_if ARM64_HAS_NMI` 宏思路）；
4. 屏蔽顺序修正（patch 32 注释的等价物）：mask 时先 `ALLINT_SET` 再写 DAIF，unmask 时先写 DAIF 再 `ALLINT_CLR`——消除"DAIF 已屏蔽但 ALLINT 未置"的 NMI 窗口（6.6 当前 `local_daif_mask()` 顺序是 daifset → PMR → `_allint_set()`，存在该窗口）。

**不引入**：common_flags.h 上下文框架、entry.h drop/lift API、masking.h 配对 API、DEBUG_PRIORITY_MASKING 删除（这些都依赖 2025 pNMI 重构底座，属于第三档）。

### 4.3 风险控制
- 影响面：irqflags.h + daifflags.h + assembler.h + entry.S（3 处）+ 全内核 irqflags 语义回归；
- 必须的测试矩阵：无 NMI / pNMI / 硬件 NMI 三配置 × FAST_IRQ on/off × CONFIG_DEBUG_IRQFLAGS；
- 重点回归：hibernate/suspend、kprobes、perf NMI（hardlockup detector）、NMI backtrace、KVM（TALLINT）、华为特有 watchdog_hld/SDEI/Xint 路径。

## 5. 第三档：暂缓——框架重构与其余部分

| 原 patch | 暂缓理由 |
|---|---|
| 12/14/15/16/18/19/20/21/22/24/25/26/28（上下文框架 + 删 daifflags.h/GIC_PRIO_PSR_I_SET/DEBUG_PRIORITY_MASKING） | 重构对象在 6.6 里是老形态；依赖先回移 Marc 2025 pNMI 重构做底座 |
| 34（entry NMI 处理重构、嵌套 NMI 行为） | 依赖框架；且必须重做 FAST_IRQ/Xint（`el0_xint`、`arm64_enter_nmi`）适配，工作量大 |
| 38/43（smp ops 抽象 + LPI IPI NMI） | openEuler smp.c 无 LPI-backed IPI 根基（老 `ipi_desc[]` + SGI） |
| 44/45（gic-v5） | 上游作者自称 illustration；Fast Model 之外无验证 |
| 41/42（GIC prepare/implement） | **功能已存在**（Mark Brown 2022 + Jinjie 修复），无需回合；可选项仅为 `gic_nmi_lock` 细化和命名统一 |
| 23/24（并入 CONFIG_DEBUG_IRQFLAGS） | 依赖 15/18 完成 |

### 触发时机
- **上游 v3 发布且结构稳定后**：重新评估第二档是否升级为"直接跟随上游框架版"（届时 common_flags/entry/masking 三头文件结构定型）；
- **若 openEuler 规划 7.x 新内核分支**：第三档不应在 6.6 上做，而应在新分支上直接跟踪上游（华为 Jinjie Ruan 已是上游该系列主要 reviewer，可推动同步）。

## 6. 附：建议的执行顺序与依赖

```
第一档（可并行，除标注）:
  [02] [03] [04] [05] [06] [07*] [29] [35] [37] [01] [27] [Kconfig 解绑]
        └── [08] ──→ [09]   (*先评估 hulk !17730 与 07 的等价性)
        └── [39 思想 → ipi_nmi.c 适配版]

第二档（第一档完成后）:
  [10/11/33 思想的最小化适配] → [assembler/entry.S 变体] → [屏蔽顺序修正]
        └── 三配置 × FAST_IRQ × DEBUG_IRQFLAGS 测试矩阵

第三档（等上游 v3 + 前置底座）:
  [Marc 2025 pNMI 重构] → [12/14/15/16/18-26/28] → [34] → [38/43/44/45]
```

### 决策要点回顾
1. **不要整体回合**：RFC 未定稿 + 基础版本差两年演进 + 华为扩展交织，成本是三个系列之和；
2. **第一档必须做**：patch 1-9 里 8/9 修复真实 pNMI bug 且 openEuler 的 gic_arch_enable_irqs 原样保留着这个 bug；35/37 是 NMI 正确性的两处明确缺口；
3. **第二档是性价比最高的一步**：用 ~4 个文件的改动根治 ALLINT 类 bug 的根因，避免 2026-06 式反复 revert；
4. **第三档留给新分支或上游定稿后**：在 6.6 上重构框架不划算，功能等价物（GIC NMI）已经存在。
