# 笔记：openEuler 当前硬件 NMI 方案 vs Murzin FEAT_NMI RFC v2 —— 对比与问题清单

> 分析对象：
> - 当前方案：openEuler 6.6 内核（`/home/code/kernel`，6.6.0）中的硬件 NMI（FEAT_NMI）实现
> - 对比对象：Vladimir Murzin [RFC PATCH v2 00/45] arm64: Add support for FEAT_NMI（基于 v7.2-rc5）
> 日期：2026-08-18

---

## 0. 两套方案的谱系（关键背景）

| | openEuler 6.6（当前） | Murzin RFC v2 |
|---|---|---|
| 技术源头 | **Mark Brown 2022 原始 FEAT_NMI 系列**（2022-10/11 合入 openEuler：cpufeature 检测、superpriority NMI 处理、ALLINT+DAIF 管理）+ **Sumit Garg 2023 IPI-as-NMI 框架**（Linaro，从未进主线） | 同一拨人的**重构版**：Mark Brown 参与、Ada Couprie Diaz 主力、Murzin 汇总发布（2026-07 v1 → v2） |
| 演进方式 | 在老 daifflags/irqflags 框架上**打补丁式**接入 ALLINT，华为持续维护（Xint/FAST_IRQ、watchdog、SDEI、2026-06 反复修 ALLINT bug） | 先**重构异常屏蔽框架**（逻辑上下文 + 双状态 flags），再接入 FEAT_NMI |
| 上游状态 | 与主线分叉：上游 2025 已合入 Marc Zyngier 的 pNMI 重构（runtime selectable、PPI 用 handle_percpu_devid_irq），2026-08 已合入 Murzin patch 1-9 | 部分已进主线 arm64 for-next/nmi |

一句话概括两者差异：**openEuler 是把 ALLINT "塞进"旧框架；Murzin 系列先建新框架再"装入" ALLINT**。RFC cover letter 的原话：

> "introducing another flavour of NMI on top of the existing infrastructure could easily become messy, making the code harder to follow and reason about."

## 1. 架构对比

| 维度 | openEuler 6.6（当前） | Murzin RFC v2 |
|---|---|---|
| **状态表示** | irqflags 仍只跟踪一种机制（PMR 或 DAIF）；ALLINT 不进 `unsigned long flags` | `arm64_exc_hwstate_t`：一个 flags 同时编码 DAIF(16b)+PMR(8b)+ALLINT(bit 13) |
| **逻辑上下文** | 无。`DAIF_PROCCTX` 等只是 DAIF 常量 | `arm64_exc_context`（PROCESS/NOIRQ/NONMI/ERROR/CRITICAL）× 三套硬件映射（无 NMI / pNMI / FEAT_NMI） |
| **状态迁移校验** | 无系统性校验（仅 pNMI 的 `DEBUG_PRIORITY_MASKING` 少量 WARN） | drop/lift/inherit 每次切换校验 + 非法迁移 WARN + save/restore 配对校验（DEBUG_IRQFLAGS） |
| **NMI 判定** | entry-common.c 里 `ISR_EL1.IS` 判一次；FAST_IRQ 路径 `el0_xint` 再判一次 | 统一 `is_nmi()` helper |
| **NMI 处理期间屏蔽** | **全程保持屏蔽**（`__el1_nmi` 无任何 unmask → 不支持 NMI 嵌套） | `irqentry_nmi_enter` 后 unmask 到 NONMI_CONTEXT（允许嵌套 NMI） |
| **普通 IRQ 路径 ALLINT** | **GIC 驱动里清**（`irq-gic-v3.c` 的 `_allint_clear()`，arch 与 irqchip 职责混杂） | entry 层统一切换到 NOIRQ 上下文，职责收回 arch |
| **IRQ/NMI 分流机制** | 自研：`handle_arch_nmi_irq` 函数指针 + `set_handle_nmi_irq()`（irq.c） | 主线风格：复用 `handle_arch_irq`，GIC 内 `in_nmi()` 分流读 NMIAR1 |
| **NMI enter/exit** | 自研 `arm64_enter_nmi/exit_nmi` + per-CPU `nmi_ctx`（含华为 ct_nmi_enter 扩展） | 复用主线 `irqentry_nmi_enter/exit` |
| **IPI NMI** | Sumit Garg 框架：额外 IPI 通道 + `set_smp_dynamic_ipi`（`CONFIG_IPI_AS_NMI` **depends on ARM64_PSEUDO_NMI**） | `ipi_irq_ops` 抽象 SGI/LPI + 失败回退普通 IRQ + per-CPU `nmi_bitmap` |
| **NMI 请求失败回退** | 无（`set_smp_dynamic_ipi` 失败则静默无 NMI backtrace） | patch 39：`request_percpu_nmi` 失败显式回退 `request_irq` |
| **kprobes** | 无 ALLINT 处理 | patch 37：保存/恢复 ALLINT |
| **suspend** | `INIT_PSTATE_EL1` 不含 `PSR_ALLINT_BIT` | patch 35：恢复时显式初始化 ALLINT |
| **KVM** | 需额外回移 2024-04 的 "Decouple KVM from CONFIG_ARM64_NMI" + hyp-stub/switch.h 手工 TALLINT 处理 | 系列内置（hyp switch、HCRX_EL2.TALLINT boot 要求文档化） |
| **perf PMU NMI** | `request_percpu_nmi` + `arm_pmu_irq_is_nmi()` | 同机制（上游已有） |
| **hardlockup detector** | 自研 `watchdog_hld.c`（perf + cpufreq 自适应采样周期）+ `watchdog_sdei.c`（secure timer + SDEI） | 未涉及（上游 perf 版） |
| **GIC INMIR 并发保护** | 全局 `irq_controller_lock` | 专用 `gic_nmi_lock` |

## 2. 当前 openEuler 方案存在的问题清单

### A. 正确性风险（真实/潜在 bug）

1. **`local_daif_restore()` 用 A 位隐式推断 ALLINT 状态**（`daifflags.h:126-128`）：
   `local_daif_save()` 保存的 flags **不含 ALLINT 位**，restore 时却按 `flags & PSR_A_BIT` 决定 `_allint_set()/_allint_clear()`。A 位语义是"SError 屏蔽"，与"NMI 屏蔽"毫无逻辑关系。任何在 A=0 但 ALLINT=1 状态（NMI 上下文中）配对 save/restore 都会**错误清除 ALLINT**，导致 NMI 提前解除。Murzin patch 33 把 ALLINT 显式编码进 flags（bit 13）正是根除这一问题的做法。

2. **entry.S 的 `save_and_disable_daif` 不保存 ALLINT**（`entry.S:1073/1124/1142` 三处）：汇编级保存/恢复 DAIF 的路径上 ALLINT 状态被丢弃，恢复时 ALLINT 仍停留在入口值，与 C 侧的期望状态可能不一致。对比 Murzin patch 32 的 `save_and_disable_exceptions`（alternative 下保存 ALLINT 到 flags[13] 再恢复）。

3. **ALLINT 屏蔽逻辑与 GIC NMI 支持耦合**：Jinjie Ruan 2026-06-05 的提交 "irqchip/gic-v3: Fix ALLINT masking logic by decoupling from GIC NMI support" 在 4 天内经历了 **4 次 revert/重提**（!23676 → !23746 → !23750 → …）。说明在"CPU 有 FEAT_NMI 但 GIC 无 FEAT_GICv3_NMI"的平台上 ALLINT 屏蔽逻辑有真实 bug，且补丁式修复本身不稳定。Murzin 方案通过 `ARM64_HAS_NMI`（PE 能力）与 `ARM64_USES_NMI`（真正启用）分离 + IPI 回退（patch 39）在架构层面解决。

4. **普通 IRQ 处理中 ALLINT 由 GIC 驱动负责清除**（`irq-gic-v3.c:941-944`）：irqchip 驱动侵入 arch 屏蔽状态管理。任何不走 `gic_handle_irq` 常规路径的 IRQ（如 FAST_IRQ noack 路径、其他 irqchip）都不会清 ALLINT → IRQ handler 运行期间 NMI 被意外屏蔽。Murzin patch 8/14 将解除逻辑统一收回 entry 层（`gic_unmask_pnmis` 只负责 pNMI 自身的显式解除）。

5. **NMI handler 全程屏蔽、不支持嵌套**（`__el1_nmi` 仅 enter/do/exit，无 unmask）：NMI 处理耗时（如 backtrace 打印）期间新 NMI 被屏蔽；x86 及 Murzin 方案（unmask 到 NONMI_CONTEXT）都允许嵌套 NMI。对 hardlockup 检测本身影响有限，但会丢失 NMI 事件、放大 NMI 延迟。

6. **kprobes 完全不处理 ALLINT**（`kprobes.c` 仅保存 DAIF_MASK）：在 NMI 上下文中执行 kprobe 单步时，ALLINT 的保存/恢复不受控（Murzin patch 37 修复项）。

7. **suspend 恢复路径不初始化 ALLINT**（`INIT_PSTATE_EL1` 无 `PSR_ALLINT_BIT`）：与 Murzin patch 35 对应的修复缺失。

8. **IPI-as-NMI 的 Kconfig 绑定错误**：`CONFIG_IPI_AS_NMI` depends on `ARM64_PSEUDO_NMI`。但 `gic_irq_nmi_setup()` 实际已支持 `has_v3_3_nmi()`（INMIR 路径）——纯硬件 NMI（不开 PSEUDO_NMI）的平台因此**拿不到 NMI backtrace / kgdb roundup**，尽管硬件完全支持。Murzin patch 43/45 直接在 smp/gic 层支持，无此绑定。

9. **`set_smp_dynamic_ipi()` 失败无回退**：`request_percpu_nmi` 失败则 `ipi_nmi_desc` 为空，backtrace 静默退化，无 WARN 无 fallback（Murzin patch 39 显式回退 + 记录 per-CPU `nmi_bitmap`）。

### B. 可维护性 / 架构问题

10. **无逻辑上下文框架，状态转换散落四处**：ALLINT/PMR/DAIF 的转换逻辑分布在 `daifflags.h`（PMR+GIC_PRIO_PSR_I_SET+ALLINT hack）、`entry-common.c`（直接 `write_sysreg(DAIF_PROCCTX_NOIRQ, daif)`）、`irq-gic-v3.c`（`_allint_clear()`）、`assembler.h`（disable/enable_daif）——没有任何一处能回答"当前处于什么逻辑上下文、合法迁移是什么"。这正是 Murzin 系列"separate the logical view of exception state from its hardware representation"要解决的。

11. **`GIC_PRIO_PSR_I_SET` hack 仍然存在**（pNMI 侧，把 `PSR_I_BIT` 塞进 PMR 值骗过 irqflags），Murzin patch 22 已删除。

12. **自研 NMI enter/exit 与主线 irqentry 框架不兼容**：`arm64_enter_nmi` 内嵌华为 `ct_nmi_enter` 等扩展。若上游 v7.3 合入 Murzin 系列，openEuler 的 entry-common.c 无法直接跟随主线演进，后续每个上游 NMI 改动都需要大量手工适配（技术债）。

13. **FAST_IRQ/Xint 双路径维护成本**：`el0_xint` 里重复 NMI 判定，`gic_handle_irq_noack/gic_handle_nmi_noack` 复制 GIC 主路径逻辑。每引入一个屏蔽状态位（ALLINT 已是例子），两条路径都要改，漏改风险高。

14. **KVM 支持靠额外回移**（2024-04 Marc Zyngier "Decouple KVM from CONFIG_ARM64_NMI"）：说明 KVM/NMI 解耦是后期补丁，主线后续 KVM 演进（如 TALLINT 相关修复）不会自动获得。

### C. 维护事实（git 证据）

15. **与上游分叉持续扩大**：
    - 上游 2025 已合入 Marc Zyngier pNMI 重构（runtime selectable、删除 `ppi_nmi_refs`/`handle_percpu_devid_fasteoi_nmi`）——openEuler 6.6 仍是老框架；
    - 上游 2026-08 已合入 Murzin patch 1-9（含两个真实 bug 修复：Breno 报告的 pNMI splat）；
    - 每次上游演进，openEuler 回移成本递增。
16. **2026-06 ALLINT 修复的反复 revert** 表明该实现已进入"打补丁-出 bug-再打补丁"循环，重构比修补更经济。

### D. 客观优点（避免片面）

- 产品化增强：`watchdog_hld.c` 的 cpufreq 自适应采样周期（华为，上游没有）、SDEI watchdog（secure physical timer）、FAST_IRQ 低延迟 EL0 中断路径；
- 在 openEuler 发行版上有真机验证和发布经验；
- 相比 Mark Brown 2022 原版已积累不少修复（Yicong Yang 的 NMI withdraw 竞态修复等）。

## 3. 建议

1. **短期**：回移 Murzin patch 1-9（主线 arm64 for-next/nmi 已合入，低风险清理 + 修复 pNMI splat 与 kernel-exit 的 local_irq_disable 问题），openEuler 当前 6.6 框架与之兼容性较好。
2. **中期**：跟踪 Murzin v3/最终版。华为（Jinjie Ruan）已是该系列上游主要 reviewer 且除 gic-v5 外无异议，可推动其在 openEuler 新内核（如 7.x 分支）上直接落地，并把 FAST_IRQ/watchdog 等华为扩展在新框架（`asm/interrupts/*`）上重做。
3. **风险提示**：若继续维持 6.6 老框架，ALLINT 耦合类 bug（见 A.3）大概率还会复发；上游 NMI 演进（逻辑上下文框架）早晚要跟上，越晚迁移成本越高。
