# 内核系统休眠/唤醒全景分析（arm64 · PSCI SYSTEM_SUSPEND 路径）

> 范围：以 **arm64** 为参考架构，分析系统挂起/唤醒（suspend/resume）的整体机制，
> 硬件入睡路径为 **PSCI `SYSTEM_SUSPEND`**，并深挖 **GIC** 与 **KVM** 两条子系统的休眠唤醒实现。
> 内核版本：7.2.0-rc5（本仓库）；文件覆盖：`kernel/power/*`、`drivers/base/power/main.c`、
> `kernel/irq/pm.c`、`drivers/firmware/psci/psci.c`、`arch/arm64/kernel/suspend.c`、
> `arch/arm64/kernel/sleep.S`、`arch/arm64/mm/proc.S`、`drivers/irqchip/irq-gic-v3.c`、
> `drivers/irqchip/irq-gic-v3-its.c`、`virt/kvm/kvm_main.c`、`arch/arm64/kvm/*`。

---

## 一、背景与问题域

### 1.1 系统睡眠是一条"功耗 vs 恢复延迟"的谱系

内核把"让系统少耗电"做成一个连续体，睡得越深，软件要保存的状态越多、醒来越慢。
以 arm64 为参考架构，这条谱系长这样：

```
运行态 ──➤ runtime idle ──➤ s2idle(freeze) ──➤ mem(PSCI SYSTEM_SUSPEND) ──➤ disk ──➤ 关机
           (WFE/CPU_SUSPEND) (冻结+设备挂起)   (整系统深睡,深度由固件定)   (状态落盘)  (全断电)
  功耗递减   ────────────────────────────────────────────────────────────➤
  恢复延迟递增 ◄────────────────────────────────────────────────────────────
  软件成本递增:  无  ── 进程冻结+设备挂起 ── 平台休眠 ── 内存快照+swap 写入
```

这个谱系的内核侧出口，是 `/sys/power/state` 可写的四种状态（`kernel/power/suspend.c:37`）：

| 状态 | 含义 | 硬件行为 | arm64 可用性 |
|---|---|---|---|
| `freeze` | suspend-to-idle（s2idle） | 设备挂起、CPU 进 idle 循环 | **恒可用**，无需任何平台回调 |
| `standby` | 浅睡眠 | 平台自行决定，通常介于 freeze 与 mem | PSCI 下**不可用**（见下文） |
| `mem` | suspend-to-RAM | 整系统深睡，掉电范围由固件决定 | 需固件实现 SYSTEM_SUSPEND |
| `disk` | hibernation | 内存镜像写 swap 后断电 | 与架构无关，需 swap 与 CONFIG_HIBERNATION |

在 arm64 上，这四种软件状态背后的硬件词汇**不是 ACPI 的 S 状态，而是固件接口
PSCI（Power State Coordination Interface）**：内核只把"想睡"的意图交给固件，
**睡多深、掉什么电，由固件 state ID 决定，对内核完全透明**。PSCI 提供三个层次的睡眠原语：

```
┌──────────────────────────────────┬─────────────────────┬────────────┐
│ PSCI 睡眠原语                    │ 内核调用方          │ 典型场景    │
├──────────────────────────────────┼─────────────────────┼────────────┤
│ CPU_SUSPEND (power_state)        │ cpuidle 驱动         │ 每核深 idle │
│ SYSTEM_SUSPEND (entry_point)     │ suspend_ops->enter  │ 整系统挂起  │
│ （无 PSCI 调用，纯软件）          │ s2idle_loop         │ freeze     │
└──────────────────────────────────┴─────────────────────┴────────────┘
```

1. **CPU_SUSPEND——每核深睡的"掉多少上下文"协议**。cpuidle 的 deep state 走这里。
   `psci_cpu_suspend_enter()`（`drivers/firmware/psci/psci.c:504`）先问
   `psci_power_state_loses_context(state)`：
   - 不掉上下文 → 直接 `psci_ops.cpu_suspend(state, 0)`，内核什么都不用保存；
   - 掉上下文 → `cpu_suspend(state, psci_suspend_finisher)`
     （`arch/arm64/kernel/suspend.c:97`）把 CPU 状态存进 per-CPU 的 `sleep_save_stash`
     （`suspend.c:26`），finisher 把 `cpu_resume` 的**物理地址**交给固件（`psci.c:491`）。
     醒来时 CPU 从 reset 向量起跑、暂时用 idmap 页表，`__cpu_suspend_exit()`
     （`suspend.c:44`）卸掉 idmap、恢复真实页表（注释原文：*resuming from reset with
     the idmap active in TTBR0_EL1*）。
   `power_state` 就是固件私有的 state ID——每个 state 会掉哪些电，只有固件知道。
2. **SYSTEM_SUSPEND——整系统深睡（≈x86 的 S3）**。挂起流程的最后一步
   `suspend_ops->enter()` 走这里：`psci_system_suspend_enter()`（`psci.c:528`）先
   `pm_set_resume_via_firmware()`，再 `cpu_suspend(0, psci_system_suspend)`，finisher
   执行 `invoke_psci_fn(PSCI_FN_NATIVE(1_0, SYSTEM_SUSPEND), pa_cpu_resume, 0, 0)`
   （`psci.c:535`）——**系统从这里消失，固件决定断掉什么**：可能只关 CPU/cluster，
   也可能关掉大部分 SoC。GIC 是否保电同样由 SoC 设计决定（这是第 4 节"GICv3 永不掉电"
   假设的前提）。PSCI 只声明支持 `mem` 一种深度睡眠（`psci_suspend_ops.valid =
   suspend_valid_only_mem`，`psci.c:554`），所以 **arm64 上没有 `standby` 档位**。
3. **freeze/s2idle 的省电来自 cpuidle**。arm64 没有 S0ix 概念——s2idle 是纯软件状态：
   "冻结的进程 + 挂起的设备 + 空闲的 CPU"（`suspend.c:137` 注释原文），**不需要任何
   平台回调**（`suspend.c:194`：*Suspend-to-idle should be supported even without any
   suspend_ops*）。它省多少电取决于 CPU 在 idle 循环里能进多深的 cpuidle state，
   而 cpuidle 深度又回到第 1 点的 CPU_SUSPEND state ID——两条路殊途同归。

"丢得越多睡得越省"在 arm64 上的表达不是 S1/S3/S4 的固定档位，而是**固件 state ID 的连续谱**，
内核与固件之间关于"会丢什么"的唯一通信就是 `psci_power_state_loses_context()`。
丢失的东西必须在唤醒时重新初始化，这正是设备 PM 要分 prepare/suspend/noirq 多阶段
的原因（见 2.3 节）。

顺带说明写 `mem` 时的间接层：`state_store()` 会把 `PM_SUSPEND_MEM` 替换为
`mem_sleep_current`（`kernel/power/main.c:816`），由 `/sys/power/mem_sleep` 决定。
arm64 上只要固件支持 SYSTEM_SUSPEND，`suspend_set_ops()` 就把默认值设为 `deep`
（`suspend.c:234`，`mem_sleep_default >= PM_SUSPEND_MEM` 恒成立）——**arm64 写 mem 默认真睡**；
x86 上因 S0ix 生态默认 `s2idle`。这是两架构在"写 mem 到底睡多深"上的显著差异。

**范围声明**：`state_store()` 对 `"disk"` 走的是另一条完全独立的路
（`hibernate()`：内存快照 → 写 swap → 断电），本笔记从第二章起只讨论 arm64 上
**suspend 到 mem（SYSTEM_SUSPEND）** 的路径，hibernation 与 x86 ACPI 细节不再展开。

**x86 对照**：ACPI 把同样的谱系固化成 S0～S5 全局电源状态（S2 是"僵尸状态"——规范定义、
从 Windows 时代起无人实现；S0ix 是 Intel 在 S0 内定义的 Connected Standby 状态包、
取代 S3 成为笔记本默认，所以 x86 写 `mem` 默认反而进 s2idle）：

| ACPI 状态 | 名称 | 硬件行为 | 恢复方式 | 内核对应 |
|---|---|---|---|---|
| S0 | 正常工作 | 全功率运行 | — | 运行态 |
| S1 | Standby | CPU 停止执行但保留上下文，cache 保留 | 立即恢复执行 | `standby`（浅睡） |
| S2 | 更深的 standby | CPU 上下文与 cache 丢失，内存保留 | 需从保存的上下文恢复 | **无对应**：Linux 不支持，硬件也基本无人实现 |
| S3 | Suspend-to-RAM（STR） | 除内存外全部断电，DRAM 自刷新 | 固件唤醒路径，类冷启动但内存内容还在 | `mem`（deep） |
| S4 | Suspend-to-disk | 内存写进非易失存储后彻底断电 | 完整开机 + 读回休眠镜像 | `disk` |
| S5 | Soft-off | 完全关机 | 完整开机 | 关机 |
[来源: Documentation/admin-guide/pm/sleep-states.rst]

---

## 二、硬件抽象：内核入睡前要向三方"交代"

系统挂起要打交道的对象分三类，内核为每类准备了一张回调表。

```
        ┌────────────────────────────────────────────────────────────┐
        │  三类对象 = 三张回调表                                       │
        │                                                            │
        │  ① 设备（有 struct device）                                 │
        │     └─ dev_pm_ops: prepare/suspend/resume... 各驱动实现      │
        │        （3.4 节的五链表状态机就是编排它们的）                 │
        │                                                            │
        │  ② 平台固件（怎么睡、睡多深是厂商私事）                       │
        │     └─ platform_suspend_ops                                │
        │        arm64 的实现者: PSCI 驱动 → psci_suspend_ops         │
        │                                                            │
        │  ③ 全局设施（没有 device，但挂起时也要收摊）                  │
        │     └─ syscore_ops                                         │
        │        arm64 挂在这里的有: 时间源、irq_pm、KVM（第五节）      │
        └────────────────────────────────────────────────────────────┘
```

### 2.1 平台层：谁负责"拉电闸"

本节要回答一个问题：**内核想睡，但谁来执行最后的断电动作？** 一条因果链讲完：

```
内核想睡 → 但不知道这台机器的电源开关在哪（SoC 厂商私有）
        → 厂商把"断电"做成固件里的一个服务，并约定标准接口：PSCI
        → 内核里负责向固件喊话的代码：PSCI 驱动（drivers/firmware/psci/psci.c）
        → PM core 不直接认识 PSCI，它只认一张回调表：platform_suspend_ops
        → PSCI 驱动把表填好交给 PM core，两者就此对接
```

**第一环：内核为什么不能自己拉闸。** 每颗 arm64 SoC 的电源管理寄存器布局都是厂商
私有的，内核源码里没有、也不应该有它们的说明书。于是 arm 生态规定：**断电这类
"平台私事"统一交给固件做**——由 ATF（ARM Trusted Firmware）这类跑在 CPU 最高特权级
（EL3）的固件提供"电源服务"，内核在 EL1 管好自己的状态，最后喊一声就行。

**第二环：PSCI 是"服务菜单"的标准格式。** 内核向固件提需求需要统一说法，这个标准
就叫 PSCI（Power State Coordination Interface）。内核用一条特殊指令 SMC 陷入 EL3，
报上需求编号（例如"我要整系统睡眠"= SYSTEM_SUSPEND）。**PSCI 不是内核代码，是一份
接口规范**——固件照着实现，内核照着调用，谁也不依赖谁的内部细节。

**第三环：内核里的"喊话员"是 PSCI 驱动。** `drivers/firmware/psci/psci.c` 把
"发 SMC 指令"封装成一个个 C 函数，其中"整系统睡眠"服务的封装就是
`psci_system_suspend_enter()`。

**第四环：PM core 只认表，不认人。** 挂起流程（第三章）是 PM core 的通用代码，
它不想知道自己跑在 ARM 还是 x86 上。于是双方约定：PM core 在流程的固定位置查看一张
**函数指针表** `platform_suspend_ops`（`suspend.c:58`）——格子里有函数就调用，没有就
跳过。PSCI 驱动在探测到固件支持后把表填上（`psci.c:589`），这就是它交给 PM core 的答卷：

```c
/* drivers/firmware/psci/psci.c:554 —— PSCI 驱动填的答卷 */
static const struct platform_suspend_ops psci_suspend_ops = {
	.valid = suspend_valid_only_mem,      // PM core 问："支持哪种睡眠？" 答："只有 mem"
	.enter = psci_system_suspend_enter,   // PM core 问："怎么真睡？"   答："调 SYSTEM_SUSPEND"
	.begin = psci_system_suspend_begin,   // PM core 问："挂起前准备啥？"答："挂个标志即可"
};
```

于是整条链闭合：固件不支持 SYSTEM_SUSPEND → 表不被注册 → `/sys/power/state` 里没有
`mem`；固件支持 → 用户写 `mem` 时，PM core 走完第三章的流程，在最后一格调用
`.enter` → SMC 发出 → 机器睡着。

（`platform_s2idle_ops` 是另一张表，给"睡下去之后平台还要守夜"的场景用，arm64 的
freeze 用不到，原因见 1.1 节第 3 点。）

### 2.2 平台层在 arm64 上有多薄：流程里 8 个回调位只填了 2 个有内容的

第三章的挂起流程里有 8 个位置会停下来问"平台要不要插一脚"（`suspend.c` 里的
`platform_*` 包装函数）。arm64/PSCI 的答卷是——几乎全部空操作：

| 流程走到 | `suspend.c` 包装 | PSCI 做什么 |
|---|---|---|
| 开始挂起 | `platform_suspend_begin`（`suspend.c:307`） | `pm_set_suspend_via_firmware()`（`psci.c:534`）：挂个标志 |
| 进程冻结后 | `platform_suspend_prepare`（`suspend.c:264`） | 空操作 |
| late 阶段前 | `platform_suspend_prepare_late`（`suspend.c:270`） | 空操作（mem 不走这条） |
| noirq 阶段后 | `platform_suspend_prepare_noirq`（`suspend.c:276`） | 空操作 |
| **真正入睡** | `suspend_ops->enter`（`suspend.c:468`） | `psci_system_suspend_enter`（`psci.c:528`）：**唯一干实事的** |
| 醒来 noirq | `platform_resume_noirq`（`suspend.c:285`） | 空操作——醒来由固件直接送回 `cpu_resume`，无需回调 |
| 早期恢复 | `platform_resume_early`（`suspend.c:295`） | 空操作（s2idle 专用） |
| 收尾 | `platform_resume_finish`（`suspend.c:301`） | 空操作 |

这张表的意思是：**arm64 的挂起流程里，"平台"这个角色几乎不存在**——设备驱动各管各的
状态保存，内核通用代码管编排，固件管最后断电那一刀。对比 x86 ACPI（`prepare_late`
里一堆 ACPI 表操作），arm64 排查挂起问题几乎不用看平台胶水层，盯着设备驱动和固件就行。

### 2.3 为什么设备挂起要分四段：一条下降的台阶

从"决定要睡"到"真的睡死"，内核的能力是逐级丧失的。第三章 3.1 走的正是这条台阶：

```
可以中断、可以睡眠、runtime PM 可用
   │  prepare    （还能后悔：返回值可以拒绝这次挂起）
   ▼
可以中断、可以睡眠        ── suspend: 保存设备状态（进程已冻结，不会被打扰）
   │
   ▼
runtime PM 已禁用         ── suspend_late: 硬件收尾（main.c:1714 pm_runtime_disable）
   │
   ▼
中断已屏蔽                ── suspend_noirq: 最后写睡眠寄存器（irq/pm.c:126）
   │
   ▼
单 CPU、关中断            ── syscore: 全局设施收摊（KVM 在这里，第五节）
   │
   ▼
execute SYSTEM_SUSPEND    ── 之后内核代码不再执行，直到固件送回 cpu_resume
```

分这么细不是设计者自找麻烦，而是"能力递减"的必然结果：入睡前**必须**关中断
（睡眠序列写一半被中断打断，状态就坏了）；可一旦关了中断，设备回调里所有依赖中断的
操作都会死。所以要把"需要中断"的活（suspend）和"不需要中断"的活（late/noirq）分开，
并让每段回调清楚自己此刻能做什么、不能做什么（3.4 节的上下文对照表）。

### 2.4 睡多深：固件说了算，内核只问一句

`SYSTEM_SUSPEND` 调用**不带任何参数**（对比 `CPU_SUSPEND` 必须带 `power_state`）——
"整系统睡眠"这一档关掉哪些电源域，完全由固件决定，是 DT/ACPI 之外的平台私有语义。
内核能知道的只有两件事：`psci_power_state_loses_context()`（CPU_SUSPEND 的"掉多少"
协议，1.1 节第 1 点）和 `pm_suspend_via_firmware` 标志。**GIC 是否保电**就是这种
固件私有语义的典型例子——第四章的 GIC 分析建立在这个前提上。

---

## 三、arm64 系统挂起/唤醒整体流程

### 3.1 挂起侧调用栈（`echo mem > /sys/power/state`）

```
用户态写入
  └─ state_store()                                   kernel/power/main.c:799
      └─ pm_suspend(state)                           kernel/power/suspend.c:636
          └─ enter_state()                           kernel/power/suspend.c:576
              ├─ mutex_trylock(&system_transition_mutex)   suspend.c:591  ← 全系统唯一，
              │                                              并发写 state 直接 -EBUSY
              ├─ pm_sleep_fs_sync()                  suspend.c:600      ← sync_on_suspend=1 时
              ├─ suspend_prepare()                   suspend.c:609
              │   ├─ pm_prepare_console()            ← 挂起期间控制台切换到 PM console
              │   ├─ pm_notifier_call_chain_robust(PM_SUSPEND_PREPARE)  ← 通知链，
              │   │                                             失败时反向补发 PM_POST_SUSPEND
              │   ├─ filesystems_freeze(filesystem_freeze_enabled)   suspend.c:385
              │   └─ suspend_freeze_processes()      suspend.c:387 → power.h:277
              │       └─ freeze_processes()          kernel/power/process.c:121  (见 3.5)
              ├─ suspend_devices_and_enter()         suspend.c:618
              │   ├─ platform_suspend_begin()        suspend.c:517
              │   │   └─ psci: pm_set_suspend_via_firmware()   psci.c:534
              │   ├─ dpm_suspend_start(PMSG_SUSPEND) main.c:2334
              │   │   ├─ dpm_prepare()               main.c:2269  → 每设备 ->prepare()
              │   │   └─ dpm_suspend()               main.c:2068  → 每设备 ->suspend()
              │   └─ do { suspend_enter() } while (platform_suspend_again(state))
              │                                         suspend.c:532
              │       └─ suspend_enter()             suspend.c:419
              │           ├─ platform_suspend_prepare()  ← PSCI: 空操作
              │           ├─ dpm_suspend_late()      main.c:1779   → ->suspend_late()
              │           │   └─ pm_runtime_disable(dev) 每设备     main.c:1714
              │           ├─ dpm_suspend_noirq()     main.c:1648
              │           │   ├─ device_wakeup_arm_wake_irqs()   main.c:1652
              │           │   ├─ suspend_device_irqs()            kernel/irq/pm.c:126
              │           │   └─ ->suspend_noirq() 每设备
              │           ├─ pm_sleep_disable_secondary_cpus()  suspend.c:453
              │           │   └─ 次 CPU 下线 → psci_ops.cpu_off (PSCI CPU_OFF)  psci.c:214
              │           ├─ arch_suspend_disable_irqs()        suspend.c:457
              │           ├─ system_state = SYSTEM_SUSPEND        suspend.c:460
              │           ├─ syscore_suspend()                    suspend.c:462
              │           │   └─ kvm_suspend()  ← KVM 挂接点      kvm_main.c:5651
              │           └─ suspend_ops->enter(state)            suspend.c:468
              │               └─ psci_system_suspend_enter()      psci.c:528
              │                   └─ cpu_suspend(0, psci_system_suspend)
              │                       └─ finisher: SYSTEM_SUSPEND(pa_cpu_resume)
              │                           ═══ 系统从这里"消失"，直到固件唤醒 ═══
              └─ suspend_finish()                   suspend.c:623
                  └─ suspend_thaw_processes()       → thaw_processes()  process.c:179
```

要点：

- **串行化靠 `system_transition_mutex`**：`mutex_trylock`（不是 lock），已有挂起在途时
  新请求直接失败返回 `-EBUSY`（`suspend.c:591`）。驱动注册回调（如 `suspend_set_ops()`）
  用 `lock_system_sleep()` 取同一把锁（`kernel/power/main.c:67`）。
- **唤醒检查点遍布全程**：`try_to_freeze_tasks` 轮询循环里（`process.c:69`）、每个设备的
  `device_suspend`/`device_suspend_late` 回调前（`main.c:1956,1701`）、`syscore_suspend`
  之后入睡之前（`suspend.c:464`）——任何一点发现唤醒事件，整条链以 `-EBUSY` 回滚。

### 3.2 arm64 硬件入睡与醒来：cpu_suspend 的双返回

`enter()` 的 arm64 实现是 `psci_system_suspend_enter()`（`psci.c:528`），它复用
`cpu_suspend()`（`arch/arm64/kernel/suspend.c:97`）——后者是 arm64 上**所有**
掉电睡眠（每核 idle 和整系统挂起）的公共通道，核心是一个"一次调用、两次返回"的技巧：

```c
/* arch/arm64/kernel/suspend.c:142 */
if (__cpu_suspend_enter(&state)) {
	ret = fn(arg);              // ① 第一次返回（睡下去）: 执行 finisher
	if (!ret)                   //    fn = psci_system_suspend，成功则不返回
		ret = -EOPNOTSUPP;      //    走到这里 = 固件拒绝睡眠，转错误路径
	ct_cpuidle_exit();
} else {
	ct_cpuidle_exit();          // ② 第二次返回（醒来）: 从 cpu_resume 恢复后
	__cpu_suspend_exit();       //    x0=0 落到这个分支
}
```

注释原文（`suspend.c:146`）：*Successful cpu_suspend() should return from cpu_resume(),
returning through this code path is considered an error*。

**保存侧**（睡下去之前）由两段汇编完成：

- `__cpu_suspend_enter()`（`arch/arm64/kernel/sleep.S:65`）：把 callee-saved 寄存器
  x19～x28、fp、lr、sp 存入 `sleep_stack_data`；用 `mpidr_hash` 计算下标，把上下文
  指针写进 per-CPU 的 `sleep_save_stash`（`sleep.S:79`，数组在 `suspend.c:26` 分配，
  因为醒来时 MMU 尚未恢复，只能按 CPU 身份查物理地址找上下文）；随后调
  `cpu_do_suspend()`（`arch/arm64/mm/proc.S:89`）保存系统寄存器：
  `tpidr_el0/tpidrro_el0/contextidr_el1/cpacr_el1/tcr_el1/vbar_el1/mdscr_el1/sctlr_el1/
  sp_el0/x18`（Shadow Call Stack 平台寄存器）等；最后返回 1。
- finisher `psci_system_suspend()`（`psci.c:519`）：算出 `cpu_resume` 的**物理地址**
  （`__pa_symbol(cpu_resume)`），`invoke_psci_fn(SYSTEM_SUSPEND, pa_cpu_resume, 0, 0)`
  陷入固件。此后 CPU 断电，内核代码不再执行。

**醒来侧**：

- 固件把 CPU 从 reset 向量重新启动，并直接跳转到当时交给它的 `cpu_resume`
  （`sleep.S:101`，`SYM_CODE_START`——从 reset 起跑，**不能假设任何栈/页表状态**）：
  `init_kernel_el` 初始化异常级别 → `__cpu_setup` 配置 CPU → 用 `idmap_pg_dir +
  swapper_pg_dir` **提前打开 MMU**（注释原文：*enable the MMU early - so we can access
  sleep_save_stash by va*）→ 跳 `_cpu_resume`（`sleep.S:116`）按 `sleep_save_stash`
  恢复保存的寄存器，最终"返回"到 `__cpu_suspend_enter` 的调用点，但 **x0=0**——
  这就是 `cpu_suspend()` 走到 else 分支的机制。
- `__cpu_suspend_exit()`（`suspend.c:44`）收尾：`cpu_uninstall_idmap()` 卸掉 idmap
  换回真实页表（注释：*resuming from reset with the idmap active in TTBR0_EL1*）、
  恢复 CnP/DIT/PAN、**在 debug 异常重新打开之前**恢复硬件断点寄存器
  （`suspend.c:29` 注释：notifier 跑的时候 debug 异常可能已开，寄存器状态未知）、
  重新套用 Spectre 缓解、MTE/SME/PTRAUTH 各自的 `suspend_exit`。

完整时序：

```
挂起侧（cpu_suspend）                  固件                    醒来侧
──────────────────                    ────                    ──────
__cpu_suspend_enter(): 保存 x19-x28、
  sp、系统寄存器 → sleep_save_stash → 返回 1
fn(arg) = psci_system_suspend()
  invoke_psci_fn(SYSTEM_SUSPEND, pa_cpu_resume)
    ═══════════════════➤ SMC/HVC 陷入固件
                                     [固件按平台策略断电：
                                      关 CPU/cluster/部分 SoC]
                                     [唤醒事件 → 固件重新上电]
                                     [CPU 从 reset 向量起跑]
    ◄══════════════════════════════════════ 跳到 cpu_resume(物理地址)
                                     init_kernel_el → __cpu_setup
                                     → 早期开 MMU(idmap+swapper)
                                     → _cpu_resume 恢复上下文
__cpu_suspend_enter 调用点"返回"，x0=0
cpu_suspend() 走 else 分支:
  __cpu_suspend_exit(): 卸 idmap、
  恢复调试/MTE/SME/PTRAUTH
  local_daif_restore() → 回到 suspend_enter()
  的恢复路径（3.3 节）
```

### 3.3 恢复侧镜像逆序

```
唤醒（固件把 CPU 送回 cpu_resume，最终回到 suspend_enter 的恢复路径）
  └─ syscore_resume()                              suspend.c:474
  │   └─ kvm_resume()  ← KVM 恢复点                kvm_main.c:5668
  ├─ system_state = SYSTEM_RUNNING                 suspend.c:477
  ├─ arch_suspend_enable_irqs()                    suspend.c:479
  ├─ pm_sleep_enable_secondary_cpus()              suspend.c:483
  │   └─ 次 CPU 上线 → psci_ops.cpu_on (PSCI CPU_ON)  psci.c:217
  ├─ platform_resume_noirq()                       suspend.c:486  ← PSCI: 空操作
  ├─ dpm_resume_noirq(PMSG_RESUME)                 suspend.c:487 → main.c:937
  │   ├─ ->resume_noirq() 每设备
  │   ├─ resume_device_irqs()                      main.c:941 → kernel/irq/pm.c:246
  │   └─ device_wakeup_disarm_wake_irqs()          main.c:942
  ├─ dpm_resume_early(PMSG_RESUME)                 suspend.c:493 → main.c:1033
  │   └─ pm_runtime_enable(dev) 每设备             main.c:1005
  └─ dpm_resume_end(PMSG_RESUME)                   suspend.c:538 → main.c:1357
      ├─ dpm_resume()  → ->resume()                main.c:1212
      └─ dpm_complete() → ->complete()             main.c:1315
          └─ device_unblock_probing()              main.c:1346
  └─ console_resume_all()                          suspend.c:541
  └─ suspend_finish() → thaw 进程
```

挂起与恢复严格对称：**挂起顺序的逆序就是恢复顺序**，由五链表状态机保证（3.4 节）。
arm64 的对称里只有一个不对称点：入睡靠 `enter()` 主动调用，醒来却是**固件直接把 CPU
送进 `cpu_resume`**——`platform_resume_noirq()` 在 PSCI 下是空操作（2.2 节表）。

### 3.4 设备 PM：五链表状态机

`drivers/base/power/main.c:56` 定义的五个链表，就是设备挂起进度的具象化：

```
挂起:  dpm_list ──①prepare──➤ prepared ──②suspend──➤ suspended ──③late──➤ late_early ──④noirq──➤ noirq
恢复:  dpm_list ◄──⑧complete── prepared ◄──⑦resume── suspended ◄──⑥early── late_early ◄──⑤noirq── noirq
```

- 链表迁移点：① `dpm_prepare()` 内 `list_move_tail`（`main.c:2308`）；② `dpm_suspend()`
  内 `list_move`（`main.c:2098`）；③ `main.c:1807`；④ `main.c:1602`；⑤ `main.c:907`；
  ⑥ `main.c:1057`；⑦ `main.c:1236`；⑧ `dpm_complete()` 最后 `list_splice` 回
  `dpm_list`（`main.c:1340`）。
- **排序即依赖**：`dpm_list` 按发现顺序（父先子后）维护；挂起时**从尾部**取设备
  （子设备先挂），恢复时**从头部**取（父设备先恢复）。device_link 的 supplier/consumer
  关系在 `dpm_wait_for_subordinate()`/`dpm_wait_for_superior()`（`main.c:297,355`）中
  以等待 completion 的方式强化同一约束。
- **回调选择次序**固定：`pm_domain → type → class → bus → driver`，每个设备只执行一个
  层级的回调（`device_suspend()` 内，`main.c:1989`）。
- **异步挂起**：`dev->power.async_suspend` 且 `pm_async_enabled` 时，回调交给
  `async_schedule_dev_nocall()` 在 async worker 里跑（`main.c:686`）；依赖同步靠每个
  设备的 `power.completion`。叶设备（无子无 consumer，`dpm_leaf_device()` `main.c:1368`）
  被优先并行发起。

四段的执行上下文差异是驱动写回调时的铁律：

| 阶段 | 中断 | 进程 | 睡眠 | runtime PM | 典型用途 |
|---|---|---|---|---|---|
| prepare | 开 | 已冻结 | 可 | 可用 | 检查是否可以挂、direct_complete 判定 |
| suspend | 开 | 已冻结 | 可 | 已 `pm_runtime_barrier`（`main.c:1954`） | 保存设备状态、停 DMA |
| suspend_late | 开 | 已冻结 | 谨慎 | **已禁用**（`main.c:1714`） | 与中断无关的硬件收尾 |
| suspend_noirq | **关**（`suspend_device_irqs` 之后） | 已冻结 | **不可** | 已禁用 | 关芯片电源、写睡眠寄存器 |
| syscore | 关、次 CPU 下线 | — | 不可 | — | KVM、时间源等全局设施 |

### 3.5 进程冻结器（freezer）

`kernel/power/process.c` 实现：

- `freeze_processes()`（`process.c:121`）：先 `__usermodehelper_disable(UMH_FREEZING)`
  禁止新的 usermode helper，给自己打 `PF_SUSPEND_TASK` 豁免冻结，置 `pm_freezing` 并
  打开 `freezer_active` 静态分支，然后 `try_to_freeze_tasks(true)`。
- `try_to_freeze_tasks()`（`process.c:28`）：持 `tasklist_lock` 遍历所有任务，对用户态
  任务 `freeze_task()` 发送**伪造信号**（`kernel/freezer.c:164`），任务在 `get_signal()`
  返回前进入 `__refrigerator()`（`kernel/freezer.c:63`）挂起；内核线程则靠**协作式
  检查点** `try_to_freeze()`。每轮遍历后 `usleep_range` 指数退避重试（1ms→8ms），
  超时 20s（`freeze_timeout_msecs`，`process.c:26`）或发现唤醒事件即放弃。
- 冻结期间 `oom_killer_disable()`（`process.c:149`）防止 OOM killer 杀正在冻结的任务
  造成死锁。
- 解冻 `thaw_processes()`（`process.c:179`）：恢复 UMH、`thaw_workqueues()`、遍历
  `__thaw_task()` 全部任务、`schedule()` 让被解冻的任务先跑，最后清自己的
  `PF_SUSPEND_TASK`。

### 3.6 唤醒中止竞态与 s2idle

```
任意中断上下文                              挂起路径的检查点
────────────────                            ────────────────
wakeup_source_activate()                      pm_wakeup_pending()   wakeup.c:871
  └─ ws->wakeup_count++                          ├─ events_check_enabled?
  └─ pm_system_wakeup()  wakeup.c:895               ├─ cnt != saved_count || inpr > 0
      ├─ atomic_inc(&pm_abort_suspend)               │    └─ ret=true，且一次性失效
      └─ s2idle_wake()  suspend.c:163                └─ ret || pm_abort_suspend>0
          └─ s2idle_state=WAKE; swake_up_one     → 检查点返回 true → 路径 -EBUSY 回滚
```

- 检查点返回 true 后，错误沿 `resume_event()` 的事件映射回滚：`SUSPEND→RESUME`、
  `FREEZE→RECOVER`、`HIBERNATE→RESTORE`（`main.c:1451`）——挂到哪个阶段，就按对应
  事件的逆序恢复哪个阶段。
- 用户态协议 `/sys/power/wakeup_count`（`kernel/power/main.c:862`）：先读计数 → 干完
  自己的事 → 写回。写失败说明期间有事件，禁止写 `state`；写成功则挂起途中任何新事件
  都会中止挂起。这是"读-检查-写"的经典二段提交。
- **s2idle 的特殊性**：没有硬件睡死点，事件在"检查 pending 之后、进入 wait 之前"到达
  会永远丢失。`s2idle_enter()`（`suspend.c:91`）用 `s2idle_lock` 关闭这个窗口
  （`suspend.c:105` 拿锁后再查一次 pending，注释明确：保证 `pm_system_wakeup()` 不能
  整体插入"检查"与"状态更新"之间），然后当前 CPU `swait_event_exclusive()` 睡在
  simple waitqueue 上（`suspend.c:115`）——**这就是 s2idle 的"睡眠"**。中断上下文调
  `pm_system_wakeup()` → `s2idle_wake()`（`suspend.c:163`）拿同一把锁置
  `S2IDLE_STATE_WAKE` 并唤醒它。
- 在 arm64 上，s2idle 是"固件不支持 SYSTEM_SUSPEND 时唯一可用的系统挂起"，
  深睡与浅睡两条路共享同一套设备挂起流程，只在 `enter()` 处分叉
  （`suspend.c:448`：`state == PM_SUSPEND_TO_IDLE` 走 `s2idle_loop()`）。

---

## 四、GIC 的挂起与恢复（CPU 级 + 系统级）

### 4.1 CPU 级别：deep idle（CPU_SUSPEND）路径

**挂起侧调用栈**：

```
__psci_enter_domain_idle_state()                drivers/cpuidle/cpuidle-psci.c:64
  ├─ cpu_pm_enter()                             cpuidle-psci.c:75
  │    └─ notifier 链: gic_cpu_pm_notifier()    irq-gic-v3.c:1482（CPU_PM_ENTER 分支）
  │         ├─ gic_write_grpen1(0)              ← 写 ICC_IGRPEN1_EL1 = 0
  │         └─ gic_enable_redist(false)         irq-gic-v3.c:367
  │              └─ 写 GICR_WAKER.ProcessorSleep = 1
  ├─ psci_cpu_suspend_enter(state)              psci.c:504
  │    └─ cpu_suspend(state, psci_suspend_finisher)   arch/arm64/kernel/suspend.c:97
  │         └─ finisher: psci_ops.cpu_suspend(power_state, pa_cpu_resume)  psci.c:491
  └─ [CPU 掉电]
```

**恢复侧调用栈**：

```
  [唤醒，cpu_resume 恢复上下文]
  └─ cpu_pm_exit()                              cpuidle-psci.c:98
       └─ notifier 链: gic_cpu_pm_notifier()    irq-gic-v3.c:1482（CPU_PM_EXIT 分支）
            ├─ gic_enable_redist(true)          ← 写 GICR_WAKER.ProcessorSleep = 0
            ├─ gic_cpu_sys_reg_enable()         irq-gic-v3.c:1139
            │    └─ gic_enable_sre()            ← 写 ICC_SRE_EL1
            └─ gic_cpu_sys_reg_init()           irq-gic-v3.c:1153
                 ├─ 写 ICC_PMR_EL1
                 ├─ 写 ICC_BPR1_EL1 = 0
                 ├─ 写 ICC_CTLR_EL1（EOImode）
                 ├─ 写 ICC_AP0Rn_EL1 = 0 / ICC_AP1Rn_EL1 = 0
                 ├─ gic_write_grpen1(1)         ← 写 ICC_IGRPEN1_EL1 = 1
                 └─ 记录 per_cpu(has_rss, cpu)
```

**各函数代码与行为**：

`gic_cpu_pm_notifier()`（`irq-gic-v3.c:1482`，注册于 `gic_cpu_pm_init()` `irq-gic-v3.c:1503`）：

```c
static int gic_cpu_pm_notifier(struct notifier_block *self,
			       unsigned long cmd, void *v)
{
	if (cmd == CPU_PM_EXIT || cmd == CPU_PM_ENTER_FAILED) {
		if (gic_dist_security_disabled())
			gic_enable_redist(true);        // 唤醒 redistributor
		gic_cpu_sys_reg_enable();           // 写 ICC_SRE_EL1
		gic_cpu_sys_reg_init();             // 重建 CPU 接口寄存器
	} else if (cmd == CPU_PM_ENTER && gic_dist_security_disabled()) {
		gic_write_grpen1(0);               // 写 ICC_IGRPEN1_EL1 = 0
		gic_enable_redist(false);           // redistributor 进休眠
	}
	return NOTIFY_OK;
}
```

行为：

- `CPU_PM_ENTER`：`gic_write_grpen1(0)` 关闭该 CPU 的 Group1 中断投递；`gic_enable_redist(false)` 让 redistributor 休眠。两者都以 `gic_dist_security_disabled()`（GICD_CTLR.DS==1）为条件——GICD 处于 security 禁用状态时这些寄存器才对 EL1 可见可写。
- `CPU_PM_EXIT` / `CPU_PM_ENTER_FAILED`：按顺序执行 redistributor 唤醒 → SRE 使能 → CPU 接口寄存器重建。

`gic_enable_redist()`（`irq-gic-v3.c:367`）：

```c
static void gic_enable_redist(bool enable)
{
	void __iomem *rbase;
	u32 val;
	int ret;

	if (gic_data.flags & FLAGS_WORKAROUND_GICR_WAKER_MSM8996)
		return;

	rbase = gic_data_rdist_rd_base();

	val = readl_relaxed(rbase + GICR_WAKER);
	if (enable)
		val &= ~GICR_WAKER_ProcessorSleep;   /* Wake up this CPU redistributor */
	else
		val |= GICR_WAKER_ProcessorSleep;
	writel_relaxed(val, rbase + GICR_WAKER);

	if (!enable) {		/* Check that GICR_WAKER is writeable */
		val = readl_relaxed(rbase + GICR_WAKER);
		if (!(val & GICR_WAKER_ProcessorSleep))
			return;	/* No PM support in this redistributor */
	}

	ret = readl_relaxed_poll_timeout_atomic(rbase + GICR_WAKER, val,
						enable ^ (bool)(val & GICR_WAKER_ChildrenAsleep),
						1, USEC_PER_SEC);
	if (ret == -ETIMEDOUT) {
		...
	}
}
```

行为：

- `FLAGS_WORKAROUND_GICR_WAKER_MSM8996` 置位时直接返回（MSM8996 的 GICR_WAKER 不可用）。
- 读 GICR_WAKER，按 `enable` 清/置 `ProcessorSleep` 位后写回。
- `enable == false` 时读回验证：位没有被置上说明该 redistributor 没有 PM 支持，直接返回。
- 轮询 GICR_WAKER：睡眠时期望 `ChildrenAsleep` 出现，唤醒时期望消失；上限 1s（每步 1µs），超时走 `-ETIMEDOUT` 分支。

`gic_cpu_sys_reg_enable()`（`irq-gic-v3.c:1139`）：

```c
static void gic_cpu_sys_reg_enable(void)
{
	if (!gic_enable_sre())
		pr_err("GIC: unable to set SRE (disabled at EL2), panic ahead\n");
}
```

行为：调 `gic_enable_sre()` 写 `ICC_SRE_EL1`（使能系统寄存器接口）；返回 false（SRE 被 EL2 禁止）时打印错误，之后的中断寄存器操作将无法工作。

`gic_cpu_sys_reg_init()`（`irq-gic-v3.c:1153`）关键片段：

```c
static void gic_cpu_sys_reg_init(void)
{
	...
	pribits = gic_get_pribits();
	group0 = gic_has_group0();

	/* Set priority mask register */
	if (!gic_prio_masking_enabled()) {
		write_gicreg(DEFAULT_PMR_VALUE, ICC_PMR_EL1);
	} else if (gic_supports_nmi()) {
		WARN_ON(group0 != cpus_have_group0);
		WARN_ON(gic_dist_security_disabled() != cpus_have_security_disabled);
	}

	gic_write_bpr1(0);                    /* 写 ICC_BPR1_EL1 = 0 */

	if (static_branch_likely(&supports_deactivate_key)) {
		gic_write_ctlr(ICC_CTLR_EL1_EOImode_drop);       /* EOI 只降优先级 */
	} else {
		gic_write_ctlr(ICC_CTLR_EL1_EOImode_drop_dir);   /* EOI 同时 deactivate */
	}

	/* Always whack Group0 before Group1 */
	if (group0) {
		switch (pribits) {
		case 8: case 7:
			write_gicreg(0, ICC_AP0R3_EL1);
			write_gicreg(0, ICC_AP0R2_EL1);
			fallthrough;
		case 6:
			write_gicreg(0, ICC_AP0R1_EL1);
			fallthrough;
		case 5: case 4:
			write_gicreg(0, ICC_AP0R0_EL1);
		}
		isb();
	}
	... /* ICC_AP1Rn_EL1 按同样的 pribits 级联清零 */ ...

	isb();
	gic_write_grpen1(1);                  /* 写 ICC_IGRPEN1_EL1 = 1 */

	per_cpu(has_rss, cpu) = !!(gic_read_ctlr() & ICC_CTLR_EL1_RSS);
	... /* 遍历在线 CPU，校验 SGI 可达性（RSS），不可达打印 pr_crit */ ...
}
```

行为：写 `ICC_PMR_EL1`（默认值 `DEFAULT_PMR_VALUE`；NMI 模式只做一致性校验）→ 写 `ICC_BPR1_EL1 = 0` → 写 `ICC_CTLR_EL1` 恢复 EOImode → 按优先级位数清零 `ICC_AP0Rn_EL1`/`ICC_AP1Rn_EL1`（active priority 寄存器，复位后内容未知）→ `isb()` → 写 `ICC_IGRPEN1_EL1 = 1` 重新打开 Group1 投递 → 读 `ICC_CTLR_EL1.RSS` 记入 `per_cpu(has_rss)` 并校验各 CPU 间 SGI 可达性。

### 4.2 系统级别：SYSTEM_SUSPEND 路径

**挂起侧调用栈**：

```
suspend_enter()                                 kernel/power/suspend.c:419
  ├─ dpm_suspend_noirq()                        drivers/base/power/main.c:1648
  │    ├─ device_wakeup_arm_wake_irqs()         main.c:1652
  │    │    └─ 对 wakeup source 已激活的设备: irqd_set(IRQD_WAKEUP_ARMED)
  │    └─ suspend_device_irqs()                 kernel/irq/pm.c:126
  │         └─ suspend_device_irq() 逐 irq_desc 处理   irq/pm.c:65
  ├─ pm_sleep_disable_secondary_cpus()          suspend.c:453
  ├─ arch_suspend_disable_irqs()                suspend.c:457
  ├─ syscore_suspend()                          drivers/base/syscore.c:47
  │    └─ its_save_disable()                    irq-gic-v3-its.c:4992
  └─ suspend_ops->enter()                       suspend.c:468
       └─ psci_system_suspend_enter()           psci.c:528
            └─ cpu_suspend(0, psci_system_suspend)
                 └─ invoke_psci_fn(SYSTEM_SUSPEND, pa_cpu_resume)  psci.c:535
```

**恢复侧调用栈**：

```
cpu_resume（固件交回）
  └─ syscore_resume()                           drivers/base/syscore.c:93
  │    └─ its_restore_enable()                  irq-gic-v3-its.c:5028
  ├─ platform_resume_noirq()                    suspend.c:486
  ├─ dpm_resume_noirq()                         main.c:937
  │    ├─ ->resume_noirq() 每设备
  │    ├─ resume_device_irqs()                  irq/pm.c:246
  │    │    └─ resume_irq() 逐 irq_desc         irq/pm.c:144
  │    └─ device_wakeup_disarm_wake_irqs()      main.c:942
  └─ ...
```

**中断核心层代码与行为**：

`suspend_device_irq()`（`irq/pm.c:65`）：

```c
static bool suspend_device_irq(struct irq_desc *desc)
{
	unsigned long chipflags = irq_desc_get_chip(desc)->flags;
	struct irq_data *irqd = &desc->irq_data;

	if (!desc->action || irq_desc_is_chained(desc) ||
	    desc->no_suspend_depth)
		return false;

	if (irqd_is_wakeup_set(irqd)) {
		irqd_set(irqd, IRQD_WAKEUP_ARMED);

		if ((chipflags & IRQCHIP_ENABLE_WAKEUP_ON_SUSPEND) &&
		     irqd_irq_disabled(irqd)) {
			__enable_irq(desc);
			irqd_set(irqd, IRQD_IRQ_ENABLED_ON_SUSPEND);
		}
		return true;
	}

	desc->istate |= IRQS_SUSPENDED;
	__disable_irq(desc);

	if (chipflags & IRQCHIP_MASK_ON_SUSPEND)
		mask_irq(desc);
	return true;
}
```

行为（GICv3 的 `gic_chip.flags` 含 `IRQCHIP_MASK_ON_SUSPEND` 与 `IRQCHIP_SKIP_SET_WAKE`，`irq-gic-v3.c:1524`）：

- 无 action / 链式 IRQ / `no_suspend_depth > 0`（`IRQF_NO_SUSPEND`）：不做任何处理，返回 false。
- `irqd_is_wakeup_set()`（wakeup source 已激活）：置 `IRQD_WAKEUP_ARMED`。若芯片声明 `IRQCHIP_ENABLE_WAKEUP_ON_SUSPEND` 且当前是 disabled，则 `__enable_irq()` 临时打开并置 `IRQD_IRQ_ENABLED_ON_SUSPEND`。返回 true，调用方 `suspend_device_irqs()`（`irq/pm.c:126`）随后执行 `synchronize_irq()` 保证 `IRQD_WAKEUP_ARMED` 全局可见。
- 其余中断：置 `IRQS_SUSPENDED` + `__disable_irq()`；GICv3 声明了 `IRQCHIP_MASK_ON_SUSPEND`，因此再执行 `mask_irq()` 做芯片级屏蔽。

挂起期间唤醒中断到达时的处理 `irq_pm_handle_wakeup()`（`irq/pm.c:16`，由中断 flow handler 调用）：

```c
void irq_pm_handle_wakeup(struct irq_desc *desc)
{
	irqd_clear(&desc->irq_data, IRQD_WAKEUP_ARMED);
	desc->istate |= IRQS_SUSPENDED | IRQS_PENDING;
	desc->depth++;
	irq_disable(desc);
	pm_system_irq_wakeup(irq_desc_get_irq(desc));
}
```

行为：清 `IRQD_WAKEUP_ARMED` → 置 `IRQS_SUSPENDED|IRQS_PENDING` → `irq_disable()`（此刻不执行 action 的 ISR，中断处理推迟到 resume 之后）→ `pm_system_irq_wakeup()` 通知 PM core（触发 `pm_system_wakeup()`，`wakeup.c:895`：`atomic_inc(&pm_abort_suspend)` + `s2idle_wake()`）。

恢复时的 `resume_irq()`（`irq/pm.c:144`，由 `resume_device_irqs()` `irq/pm.c:246` 逐 desc 调用）：

```c
static void resume_irq(struct irq_desc *desc)
{
	struct irq_data *irqd = &desc->irq_data;

	irqd_clear(irqd, IRQD_WAKEUP_ARMED);

	if (irqd_is_enabled_on_suspend(irqd)) {
		__disable_irq(desc);
		irqd_clear(irqd, IRQD_IRQ_ENABLED_ON_SUSPEND);
	}

	if (desc->istate & IRQS_SUSPENDED)
		goto resume;

	if (!desc->force_resume_depth)
		return;

	desc->depth++;
	irq_state_set_disabled(desc);
	irq_state_set_masked(desc);
resume:
	desc->istate &= ~IRQS_SUSPENDED;
	__enable_irq(desc);
}
```

行为：清 `IRQD_WAKEUP_ARMED` → 对挂起期间被临时打开的唤醒 IRQ（`IRQD_IRQ_ENABLED_ON_SUSPEND`）恢复原 disabled 态 → 对 `IRQS_SUSPENDED` 的中断清标志并 `__enable_irq()`；`IRQF_FORCE_RESUME`（`force_resume_depth`）的中断即使挂起前是 disabled 也强制打开。`IRQF_EARLY_RESUME` 的中断另由 syscore 提前恢复：`irq_pm_syscore_ops.resume`（`irq/pm.c:218`，注册于 `irq/pm.c:233`）。

丢失唤醒事件的补偿 `rearm_wake_irq()`（`irq/pm.c:198`）与 GICv3 的 `gic_retrigger()`（`irq-gic-v3.c:1478`）：

```c
void rearm_wake_irq(unsigned int irq)
{
	...
	if (!(desc->istate & IRQS_SUSPENDED) || !irqd_is_wakeup_set(&desc->irq_data))
		return;

	desc->istate &= ~IRQS_SUSPENDED;
	irqd_set(&desc->irq_data, IRQD_WAKEUP_ARMED);
	__enable_irq(desc);
}
```

行为：对"挂起期间被武装但错过投递窗口"的唤醒 IRQ：清 `IRQS_SUSPENDED`、重新置 `IRQD_WAKEUP_ARMED`、重新使能。GICv3 侧配套 `gic_retrigger()`（`irq-gic-v3.c:1478`）通过 `gic_irq_set_irqchip_state(IRQCHIP_STATE_PENDING, true)`（`irq-gic-v3.c:521`）在 GIC 层把中断重新置为 pending 重新投递。

**ITS 的 syscore 代码与行为**：

`its_syscore_ops`（`irq-gic-v3-its.c:5088`，`its_init()` 中 `register_syscore()`，`its.c:5867`）：

```c
static const struct syscore_ops its_syscore_ops = {
	.suspend = its_save_disable,
	.resume  = its_restore_enable,
};
```

`its_save_disable()`（`irq-gic-v3-its.c:4992`）：

```c
static int its_save_disable(void *data)
{
	struct its_node *its;
	int err = 0;

	raw_spin_lock(&its_lock);
	list_for_each_entry(its, &its_nodes, entry) {
		void __iomem *base;

		base = its->base;
		its->ctlr_save = readl_relaxed(base + GITS_CTLR);
		err = its_force_quiescent(base);
		if (err) {
			pr_err("ITS@%pa: failed to quiesce: %d\n",
			       &its->phys_base, err);
			writel_relaxed(its->ctlr_save, base + GITS_CTLR);
			goto err;
		}

		its->cbaser_save = gits_read_cbaser(base + GITS_CBASER);
	}
	...
}
```

行为：持 `its_lock` 遍历 `its_nodes`——保存 `GITS_CTLR` 到 `ctlr_save` → `its_force_quiescent()` 停摆 ITS；失败则把 `ctlr_save` 写回 GITS_CTLR 并沿反向链表回滚（`its.c:5016`）；成功则保存 `GITS_CBASER` 到 `cbaser_save`。

`its_force_quiescent()`（`irq-gic-v3-its.c:5012` 附近）：

```c
static int its_force_quiescent(void __iomem *base)
{
	u32 count = 1000000;	/* 1s */
	u32 val;

	val = readl_relaxed(base + GITS_CTLR);
	if ((val & GITS_CTLR_QUIESCENT) && !(val & GITS_CTLR_ENABLE))
		return 0;

	/* Disable the generation of all interrupts to this ITS */
	val &= ~(GITS_CTLR_ENABLE | GITS_CTLR_ImDe);
	writel_relaxed(val, base + GITS_CTLR);

	while (1) {
		val = readl_relaxed(base + GITS_CTLR);
		if (val & GITS_CTLR_QUIESCENT)
			return 0;

		count--;
		if (!count)
			return -EBUSY;

		cpu_relax();
		udelay(1);
	}
}
```

行为：已 quiescent 且 disabled 则直接返回 0 → 写 `GITS_CTLR` 清 `ENABLE|ImDe`（停止向该 ITS 生成中断）→ 轮询 `GITS_CTLR.QUIESCENT`，每次 1µs、上限 1s，超时返回 `-EBUSY`。

`its_restore_enable()`（`irq-gic-v3-its.c:5028`）：

```c
static void its_restore_enable(void *data)
{
	struct its_node *its;
	int ret;

	raw_spin_lock(&its_lock);
	list_for_each_entry(its, &its_nodes, entry) {
		void __iomem *base;
		int i;

		base = its->base;

		/*
		 * Make sure that the ITS is disabled. If it fails to quiesce,
		 * don't restore it since writing to CBASER or BASER<n>
		 * registers is undefined according to the GIC v3 ITS
		 * Specification.
		 *
		 * Firmware resuming with the ITS enabled is terminally broken.
		 */
		WARN_ON(readl_relaxed(base + GITS_CTLR) & GITS_CTLR_ENABLE);
		ret = its_force_quiescent(base);
		if (ret) {
			pr_err("ITS@%pa: failed to quiesce on resume: %d\n",
			       &its->phys_base, ret);
			continue;
		}

		gits_write_cbaser(its->cbaser_save, base + GITS_CBASER);

		/*
		 * Writing CBASER resets CREADR to 0, so make CWRITER and
		 * cmd_write line up with it.
		 */
		its->cmd_write = its->cmd_base;
		gits_write_cwriter(0, base + GITS_CWRITER);

		/* Restore GITS_BASER from the value cache. */
		for (i = 0; i < GITS_BASER_NR_REGS; i++) {
			struct its_baser *baser = &its->tables[i];

			if (!(baser->val & GITS_BASER_VALID))
				continue;

			its_write_baser(its, baser, baser->val);
		}
		writel_relaxed(its->ctlr_save, base + GITS_CTLR);

		/*
		 * Reinit the collection if it's stored in the ITS. This is
		 * indicated by the col_id being less than the HCC field.
		 * CID < HCC as specified in the GIC v3 Documentation.
		 */
		if (its->collections[smp_processor_id()].col_id <
		    GITS_TYPER_HCC(gic_read_typer(base + GITS_TYPER)))
			its_cpu_init_collection(its);
	}
	raw_spin_unlock(&its_lock);
}
```

行为，逐项：

- `WARN_ON(GITS_CTLR & ENABLE)`：固件唤醒时把 ITS 留在 enabled 状态（注释："Firmware resuming with the ITS enabled is terminally broken"）。
- `its_force_quiescent()` 失败则 `continue` 跳过该 ITS（不恢复）。
- 写 `GITS_CBASER = cbaser_save`；写 CBASER 会清零 CREADR，因此同步把软件指针 `its->cmd_write = its->cmd_base` 并写 `GITS_CWRITER = 0`。
- 遍历 `GITS_BASER_NR_REGS` 个寄存器：`baser->val` 有效（`GITS_BASER_VALID`）的用 `its_write_baser()`（`its.c:2372`）写回——`baser->val` 是建表时缓存的寄存器值。
- 写 `GITS_CTLR = ctlr_save`。
- 若 collection 存储在 ITS 内部（`col_id < GITS_TYPER_HCC`），调 `its_cpu_init_collection()` 重建当前 CPU 的 collection 映射。

**GICD/GICR 在系统级路径上的代码**：`drivers/irqchip/irq-gic-v3.c` 没有注册任何 suspend/resume 回调或 syscore ops。系统级挂起时对 GICD/GICR 的操作只有 `suspend_device_irqs()` 经 `gic_mask_irq`/`gic_unmask_irq` 写 GICD/GICR 的使能位，以及恢复时 `resume_device_irqs()` 写回使能位。

### 4.3 总结思考

GIC中，各种翻译表、配置表、pending表都是存在内存中的，系统级别的休眠唤醒不用去管这些内存表，就像在ITS恢复时，重新将配置表地址写进ITS_CBASER。但是GIC中的各种寄存器怎么在系统级的休眠唤醒中保存和恢复呢？代码中似乎没有答案。CPU级别的休眠唤醒代码中做了ICC寄存器的保存和恢复，但是系统级的休眠唤醒并没有。是不是完全由固件去做？

---

## 五、KVM 的挂起与恢复（CPU 级 + 系统级）

### 5.1 CPU 级别：deep idle（CPU_PM）路径

**挂起/恢复调用栈**：

```
__psci_enter_domain_idle_state()                drivers/cpuidle/cpuidle-psci.c:64
  ├─ cpu_pm_enter()                             cpuidle-psci.c:75
  │    └─ notifier 链: hyp_init_cpu_pm_notifier()  arch/arm64/kvm/arm.c:2339
  │         └─ CPU_PM_ENTER 分支: cpu_hyp_reset()
  ├─ psci_cpu_suspend_enter(state)              psci.c:504
  └─ [CPU 掉电]
  [唤醒，cpu_resume]
  └─ cpu_pm_exit()                              cpuidle-psci.c:98
       └─ notifier 链: hyp_init_cpu_pm_notifier()  arm.c:2339
            └─ CPU_PM_EXIT 分支: cpu_hyp_reinit()
```

**代码与行为**：

`hyp_init_cpu_pm_notifier()`（`arm.c:2339`，注册于 `cpu_pm_register_notifier(&hyp_init_cpu_pm_nb)`，`arm.c:2379`）：

```c
static int hyp_init_cpu_pm_notifier(struct notifier_block *self,
				    unsigned long cmd, void *v)
{
	/*
	 * kvm_hyp_initialized is left with its old value over
	 * PM_ENTER->PM_EXIT. It is used to indicate PM_EXIT should
	 * re-enable hyp.
	 */
	switch (cmd) {
	case CPU_PM_ENTER:
		if (__this_cpu_read(kvm_hyp_initialized))
			/*
			 * don't update kvm_hyp_initialized here
			 * so that the hyp will be re-enabled
			 * when we resume. See below.
			 */
			cpu_hyp_reset();

		return NOTIFY_OK;
	case CPU_PM_ENTER_FAILED:
	case CPU_PM_EXIT:
		if (__this_cpu_read(kvm_hyp_initialized))
			/* The hyp was enabled before suspend. */
			cpu_hyp_reinit();
	}
	return NOTIFY_OK;
}
```

行为：`CPU_PM_ENTER` 时，若 per-CPU 标志 `kvm_hyp_initialized` 为真（该 CPU 的 hyp 此前已初始化），调 `cpu_hyp_reset()`，**不修改** `kvm_hyp_initialized`；`CPU_PM_EXIT` / `CPU_PM_ENTER_FAILED` 时，标志仍为真则调 `cpu_hyp_reinit()`。

`cpu_hyp_reset()` / `cpu_hyp_reinit()`（`arm.c`）：

```c
static void cpu_hyp_reset(void)
{
	if (!is_kernel_in_hyp_mode())
		__hyp_reset_vectors();
}

static void cpu_hyp_reinit(void)
{
	cpu_hyp_reset();
	cpu_hyp_init_context();
	cpu_hyp_init_features();
}
```

`cpu_hyp_init_context()`（`arm.c`）：

```c
static void cpu_hyp_init_context(void)
{
	kvm_init_host_cpu_context(host_data_ptr(host_ctxt));
	kvm_init_host_debug_data();

	if (!is_kernel_in_hyp_mode())
		cpu_init_hyp_mode();
}
```

`cpu_hyp_init_features()`（`arm.c`）：

```c
static void cpu_hyp_init_features(void)
{
	cpu_set_hyp_vector();

	if (is_kernel_in_hyp_mode()) {
		kvm_timer_init_vhe();
		kvm_debug_init_vhe();
	}

	if (vgic_present)
		kvm_vgic_init_cpu_hardware();
}
```

行为，逐函数：

- `cpu_hyp_reset()`：nVHE（`!is_kernel_in_hyp_mode()`）下调 `__hyp_reset_vectors()` 把 EL2 异常向量指回 hyp stub；VHE 下不做任何事。
- `cpu_hyp_init_context()`：`kvm_init_host_cpu_context()` 初始化宿主 CPU 上下文（`host_ctxt`）+ `kvm_init_host_debug_data()` 初始化宿主调试数据；nVHE 时 `cpu_init_hyp_mode()` 重建 EL2 栈与地址翻译配置。
- `cpu_hyp_init_features()`：`cpu_set_hyp_vector()` 设置 EL2 异常向量；VHE 时 `kvm_timer_init_vhe()` / `kvm_debug_init_vhe()` 初始化虚拟定时器与调试的 VHE 配置；`vgic_present` 时 `kvm_vgic_init_cpu_hardware()`。

`kvm_vgic_init_cpu_hardware()`（`vgic-init.c:756`）：

```c
void kvm_vgic_init_cpu_hardware(void)
{
	BUG_ON(preemptible());

	if (kvm_vgic_global_state.type == VGIC_V2) {
		vgic_v2_init_lrs();
	} else if (kvm_vgic_global_state.type == VGIC_V3 ||
		   kvm_vgic_global_state.has_gcie_v3_compat) {
		kvm_call_hyp(__vgic_v3_init_lrs);
	}
}
```

行为：VGIC_V2 时 `vgic_v2_init_lrs()` 清零 `GICH_LR*`（List Register）；VGIC_V3 时 `kvm_call_hyp(__vgic_v3_init_lrs)` 在 hyp 侧清零 `ICH_LR*` 寄存器。

deep idle 路径不涉及 vGIC maintenance PPI 与虚拟定时器 PPI——这两个 per-CPU 中断在 deep idle 期间保持使能状态（CPU 掉电期间无法触发，唤醒后无需额外动作）。

### 5.2 系统级别：SYSTEM_SUSPEND 路径

**挂起侧调用栈**：

```
suspend_enter()                                 kernel/power/suspend.c:419
  ├─ pm_sleep_disable_secondary_cpus()          suspend.c:453
  │    └─ 次 CPU 下线: cpuhp → kvm_offline_cpu()  virt/kvm/kvm_main.c:5630
  │         └─ kvm_disable_virtualization_cpu()  kvm_main.c:5619
  ├─ arch_suspend_disable_irqs()                suspend.c:457
  ├─ syscore_suspend()                          drivers/base/syscore.c:47
  │    └─ kvm_suspend()                         kvm_main.c:5651
  │         └─ kvm_disable_virtualization_cpu(NULL)   kvm_main.c:5619
  │              └─ kvm_arch_disable_virtualization_cpu()  arch/arm64/kvm/arm.c:2329
  └─ suspend_ops->enter() → SYSTEM_SUSPEND
```

**恢复侧调用栈**：

```
cpu_resume（固件交回）
  └─ syscore_resume()                           drivers/base/syscore.c:93
  │    └─ kvm_resume()                          kvm_main.c:5668
  │         └─ kvm_enable_virtualization_cpu()  kvm_main.c:5594
  │              └─ kvm_arch_enable_virtualization_cpu()  arm.c:2309
  ├─ pm_sleep_enable_secondary_cpus()           suspend.c:483
  │    └─ 次 CPU 上线: cpuhp → kvm_online_cpu()  kvm_main.c:5610
  │         └─ kvm_enable_virtualization_cpu()
  └─ ...
```

**代码与行为**：

`kvm_syscore_ops`（`kvm_main.c:5676`，`kvm_init()` 中 `register_syscore(&kvm_syscore)`，`kvm_main.c:5702`）：

```c
static int kvm_suspend(void *data)
{
	/*
	 * Secondary CPUs and CPU hotplug are disabled across the suspend/resume
	 * callbacks, i.e. no need to acquire kvm_usage_lock to ensure the usage
	 * count is stable. Assert that kvm_usage_lock is not held to ensure
	 * the system isn't suspended while KVM is enabling hardware. Hardware
	 * enabling can be preempted, but the task cannot be frozen until it has
	 * dropped all locks (userspace tasks are frozen via a fake signal).
	 */
	lockdep_assert_not_held(&kvm_usage_lock);
	lockdep_assert_irqs_disabled();

	kvm_disable_virtualization_cpu(NULL);
	return 0;
}

static void kvm_resume(void *data)
{
	lockdep_assert_not_held(&kvm_usage_lock);
	lockdep_assert_irqs_disabled();

	WARN_ON_ONCE(kvm_enable_virtualization_cpu());
}

static const struct syscore_ops kvm_syscore_ops = {
	.suspend = kvm_suspend,
	.resume = kvm_resume,
	.shutdown = kvm_shutdown,
};
```

行为：`kvm_suspend()` 断言未持 `kvm_usage_lock`、断言中断已关闭（syscore 阶段为单 CPU + 关中断上下文），然后对当前 CPU 执行 `kvm_disable_virtualization_cpu(NULL)`。`kvm_resume()` 对称执行 `kvm_enable_virtualization_cpu()`，失败时 `WARN_ON_ONCE`。`kvm_shutdown()` 用于 reboot/关机路径。

`kvm_disable_virtualization_cpu()` / `kvm_enable_virtualization_cpu()`（`kvm_main.c:5619,5594`）：

```c
static void kvm_disable_virtualization_cpu(void *ign)
{
	if (!__this_cpu_read(virtualization_enabled))
		return;

	kvm_arch_disable_virtualization_cpu();

	__this_cpu_write(virtualization_enabled, false);
}

static int kvm_enable_virtualization_cpu(void)
{
	if (__this_cpu_read(virtualization_enabled))
		return 0;

	if (kvm_arch_enable_virtualization_cpu()) {
		pr_info("kvm: enabling virtualization on CPU%d failed\n",
			raw_smp_processor_id());
		return -EIO;
	}

	__this_cpu_write(virtualization_enabled, true);
	return 0;
}
```

行为：以 per-CPU 标志 `virtualization_enabled` 做幂等控制——为真时 disable 直接返回、enable 直接返回 0；disable 先调架构拆解再清标志；enable 先调架构使能（失败返回 `-EIO`）再置标志。次 CPU 下线/上线走同一对函数：`kvm_offline_cpu()`（`kvm_main.c:5630`）、`kvm_online_cpu()`（`kvm_main.c:5610`，cpuhp 状态 `CPUHP_AP_KVM_ONLINE`）。

arm64 架构层 `kvm_arch_disable_virtualization_cpu()` / `kvm_arch_enable_virtualization_cpu()`（`arm.c:2329,2309`）：

```c
void kvm_arch_disable_virtualization_cpu(void)
{
	kvm_timer_cpu_down();
	kvm_vgic_cpu_down();

	if (!is_protected_kvm_enabled())
		cpu_hyp_uninit(NULL);
}

int kvm_arch_enable_virtualization_cpu(void)
{
	preempt_disable();

	cpu_hyp_init(NULL);

	kvm_vgic_cpu_up();
	kvm_timer_cpu_up();

	preempt_enable();

	return 0;
}
```

`cpu_hyp_init()` / `cpu_hyp_uninit()`（`arm.c:2293,2301`）：

```c
static void cpu_hyp_init(void *discard)
{
	if (!__this_cpu_read(kvm_hyp_initialized)) {
		cpu_hyp_reinit();
		__this_cpu_write(kvm_hyp_initialized, 1);
	}
}

static void cpu_hyp_uninit(void *discard)
{
	if (!is_protected_kvm_enabled() && __this_cpu_read(kvm_hyp_initialized)) {
		cpu_hyp_reset();
		__this_cpu_write(kvm_hyp_initialized, 0);
	}
}
```

行为：`cpu_hyp_init()` 在 `kvm_hyp_initialized` 为假时执行 `cpu_hyp_reinit()`（见 5.1）并置标志；`cpu_hyp_uninit()` 在非 pKVM 且标志为真时执行 `cpu_hyp_reset()` 并清标志。pKVM（`is_protected_kvm_enabled()`）时 `cpu_hyp_uninit()` 内什么都不做。

`kvm_vgic_cpu_down()` / `kvm_vgic_cpu_up()`（`vgic-init.c:717,711`）：

```c
void kvm_vgic_cpu_up(void)
{
	enable_percpu_irq(kvm_vgic_global_state.maint_irq, 0);
}

void kvm_vgic_cpu_down(void)
{
	disable_percpu_irq(kvm_vgic_global_state.maint_irq);
}
```

行为：禁用/使能 vGIC maintenance 中断（KVM 的 per-CPU PPI，申请于 `request_percpu_irq()`，`vgic-init.c:832`，申请时无 `IRQF_NO_SUSPEND` 标志）。

`kvm_timer_cpu_up()` / `kvm_timer_cpu_down()`（`arch_timer.c:1132,1139`）：

```c
void kvm_timer_cpu_up(void)
{
	enable_percpu_irq(host_vtimer_irq, host_vtimer_irq_flags);
	if (host_ptimer_irq)
		enable_percpu_irq(host_ptimer_irq, host_ptimer_irq_flags);
}

void kvm_timer_cpu_down(void)
{
	disable_percpu_irq(host_vtimer_irq);
	if (host_ptimer_irq)
		disable_percpu_irq(host_ptimer_irq);
}
```

行为：禁用/使能虚拟定时器（vtimer）与物理定时器（ptimer）的 per-CPU IRQ。拆解顺序为 timer → vgic → hyp，重建顺序为 hyp → vgic → timer。

vCPU 重新调度上 CPU 时的恢复（`kvm_arch_vcpu_load()`，`arm.c:700` 附近）：

```c
	/* The timer must be loaded before the vgic to correctly set up physical
	 * interrupt deactivation in nested state (e.g. timer interrupt). */
	kvm_timer_vcpu_load(vcpu);
	kvm_vgic_load(vcpu);
	...
	if (is_protected_kvm_enabled()) {
		kvm_call_hyp_nvhe(__pkvm_vcpu_load,
				  vcpu->kvm->arch.pkvm.handle,
				  vcpu->vcpu_idx, vcpu->arch.hcr_el2);
		kvm_call_hyp(__vgic_v3_restore_vmcr_aprs,
			     &vcpu->arch.vgic_cpu.vgic_v3);
	}
```

行为：系统恢复后 vCPU 重新被调度上 CPU 时，`kvm_timer_vcpu_load()` 装载虚拟定时器状态，`kvm_vgic_load()` 把内存中的 vGIC 状态刷入寄存器（vGIC 状态保存在内存的 `struct vgic_*` 中，挂起期间无额外保存动作；vCPU 下 CPU 时由 `kvm_vgic_put()`，`arm.c:758`，写回内存）；pKVM 额外执行 `__pkvm_vcpu_load` 与 `__vgic_v3_restore_vmcr_aprs`（恢复 hyp 侧保存的 VMCR/APR，下 CPU 时由 `__vgic_v3_save_aprs` 保存，`arm.c:749`）。

per-VM 的 PM notifier（`kvm_init_pm_notifier()`，`kvm_main.c:906`）：

```c
static void kvm_init_pm_notifier(struct kvm *kvm)
{
	kvm->pm_notifier.notifier_call = kvm_pm_notifier_call;
	/* Suspend KVM before we suspend ftrace, RCU, etc. */
	kvm->pm_notifier.priority = INT_MAX;
	register_pm_notifier(&kvm->pm_notifier);
}
```

行为：每个 kvm 实例注册一个 PM notifier，优先级 `INT_MAX`，回调为 `kvm_pm_notifier_call`（收到 `PM_SUSPEND_PREPARE` / `PM_POST_SUSPEND` 等事件时通知该 VM）。

### 5.3 小结：CPU 级与系统级各自做了什么

**CPU 级**（单核深睡）：每颗 CPU 进 deep idle 之前，把 EL2 的异常向量指回安全桩，
防止掉电过渡期有异常落入 EL2 执行到悬空代码；醒来后重建 EL2 运行环境——异常向量、
宿主寄存器上下文、虚拟中断控制器的硬件初始化（清空中断投递寄存器），并用一个"睡过"
标记记住"回来需要重建"。这级不碰虚拟定时器和虚拟中断的 per-CPU 中断，它们保持使能
（CPU 睡着期间反正也触发不了）。

**系统级**（整系统挂起）：次 CPU 随下线流程逐个收起各自的虚拟化环境；boot CPU 在入睡前
最后一刻（中断已关、只剩一颗 CPU）做完整拆解——先把两个 per-CPU 中断（虚拟定时器、
虚拟中断维护中断）关掉，再把 EL2 复位到安全状态并清标记，然后系统才睡；醒来后严格
逆序：重建 EL2 环境 → 重开两个中断 → 次 CPU 上线时逐个重建。

**两级都不需要保存 vCPU 的世界**：vCPU 的寄存器、虚拟中断状态、虚拟定时器的主副本
本来就在内存里，vCPU 被冻结、下 CPU 时已经写回；挂起期间唯一会随断电丢失的是宿主
自己的 EL2 上下文——所以 KVM 的休眠唤醒工作，就是"拆掉再重建宿主虚拟化环境"这一件事。

KVM相关的寄存器主要是虚拟化的一些寄存器，比较广且复杂。但是vCPU的状态在上下位时会自
己恢复和保存，因此与guest相关的寄存器不需要另外保存恢复。另外，KVM的数据也都是存在
内存，看起来保存和恢复比GIC简单一些。

---

## 参考链接

### 内核文档（本仓库）

- [Documentation/admin-guide/pm/sleep-states.rst](../../Documentation/admin-guide/pm/sleep-states.rst)
- [Documentation/admin-guide/pm/suspend-flows.rst](../../Documentation/admin-guide/pm/suspend-flows.rst)
- [Documentation/power/freezing-of-tasks.rst](../../Documentation/power/freezing-of-tasks.rst)
- [Documentation/power/basic-pm-debugging.rst](../../Documentation/power/basic-pm-debugging.rst)

### 规范与社区讨论

- [ARM PSCI Specification (DEN0022)](https://developer.arm.com/documentation/den0022/latest/) — SYSTEM_SUSPEND/CPU_SUSPEND 语义
- [linux-pm: Fundamental flaw in system suspend, exposed by freezer removal (2008)](https://lists.linuxfoundation.org/pipermail/linux-pm/2008-February/016779.html) — 设备挂起与父子注册竞态、`dpm_list` 设计渊源

---

*笔记生成时间: 2026-08-21 · 内核版本: 7.2.0-rc5 · 参考架构: arm64（PSCI SYSTEM_SUSPEND 路径）*
