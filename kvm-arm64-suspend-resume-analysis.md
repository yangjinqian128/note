# arm64 deep(mem)系统休眠唤醒对 KVM 虚机功能影响分析

> 范围:deep 挂起(PSCI SYSTEM_SUSPEND),即 `/sys/power/mem_sleep = deep`。
> 结论先行:deep 路径上 KVM 的状态保存/恢复机制是**闭环的**——guest 内存天然保留、vgic/timer 状态经 vcpu_put/load 往返、EL2 与 ITS 由 syscore 钩子降级/重建。虚机功能不受损,但存在两个语义点(时钟跳跃、迟到定时器);pkvm 场景走 PSCI relay 通道,机制支持但有实测建议(§五·3)。

## 一、问题域定位

`echo mem` 挂起的是 **host 的 CPU 核心与设备**,guest 本身没有"休眠"概念。DRAM 处于自刷新状态,guest 内存完整保留。因此 KVM 的职责只有一件:**把自身占用的硬件状态降级保存、唤醒后重建**:

- nVHE 时 EL2 向量与 hyp 上下文(固件断电后丢失);
- GIC 的 CPU interface(ICH_* 系统寄存器,掉电丢失);
- vcpu 的 vgic CPU interface 状态(APR/LR/VMCR)、虚拟定时器偏移;
- ITS(翻译缓存、collection、vPE 表配置,掉电丢失);
- 维护中断、物理 vtimer 中断的 percpu 使能状态。

分析要回答三个问题:① 恢复路径是否把所有硬件状态重建了;② 休眠期间 guest 侧语义是否正确(时钟走不走、到期的定时器中断丢没丢);③ 休眠后设备是否还能用(重点直通)。

## 二、主机侧完整调用栈与 KVM 挂载点

主干是 `suspend_enter()`(`kernel/power/suspend.c:462-474`),KVM 在其中挂了三个点:**vcpu 冻结**(换出时状态落内存)、**从核热下线**(cpuhp 回调)、**引导核 syscore 钩子**(`kvm_suspend/kvm_resume`)。另有一个 **cpu_pm 通知链**作为并行机制。先给出挂载点清单,再分别画挂起入口、唤醒返回两条完整调用栈。

### 2.1 KVM 挂载点清单

| # | 挂载点 | 挂起侧 | 恢复侧 | 详见 |
|---|---|---|---|---|
| ① | vcpu 冻结/换出 | `suspend_freeze_processes`(`suspend.c:387`)→ 换出时 `kvm_arch_vcpu_put`(`arm.c:746`) | thaw 后 `vcpu_load` → vgic/timer 状态重刷硬件 | §三 |
| ② | 从核热下线/上线 | `pm_sleep_disable_secondary_cpus`(`suspend.c:446`)→ `kvm_offline_cpu`(`kvm_main.c:5629`) | `pm_sleep_enable_secondary_cpus`(`suspend.c:481`)→ `kvm_online_cpu`(`kvm_main.c:5609`) | §四 |
| ③ | 引导核 syscore | `syscore_suspend`(`suspend.c:462`)→ `kvm_suspend`(`kvm_main.c:5651`) | `syscore_resume`(`suspend.c:474`)→ `kvm_resume`(`kvm_main.c:5668`) | §五·1 |
| (并行) | cpu_pm 通知链 | `cpu_pm_suspend` syscore(`kernel/cpu_pm.c:180`)→ `CPU_PM_ENTER` | `cpu_pm_resume`(`:187`)→ `CPU_PM_EXIT` / `CPU_PM_ENTER_FAILED` | — |

### 2.2 调用栈一:挂起入口(userspace → EL3 固件)

```
用户态: echo mem > /sys/power/state
  state_store()                             kernel/power/main.c:799
  │                                          (power_attr(state), main.c:831)
  └─ pm_suspend(PM_SUSPEND_MEM)             kernel/power/suspend.c:636
      └─ enter_state()                      suspend.c:575
          ├─ mutex_trylock(system_transition_mutex)
          ├─ suspend_prepare()              suspend.c:372
          │    ├─ pm_notifier_call_chain_robust(PM_SUSPEND_PREPARE)
          │    └─ suspend_freeze_processes()  suspend.c:387
          │         ║ 挂载点①  QEMU/vcpu 线程冻结 → 换出时 vcpu_put(§三)
          ├─ suspend_devices_and_enter()    suspend.c:504
          │    ├─ platform_suspend_begin → psci_system_suspend_begin  psci.c:547
          │    ├─ dpm_suspend_start()       suspend.c:523 ← 设备常规挂起(VFIO/SMMU 在此)
          │    └─ suspend_enter()           suspend.c:420
          │         ├─ platform_suspend_prepare()     suspend.c:423
          │         ├─ dpm_suspend_late()             suspend.c:427
          │         ├─ platform_suspend_prepare_late()
          │         ├─ dpm_suspend_noirq()            suspend.c:436
          │         ├─ platform_suspend_prepare_noirq()
          │         ├─ pm_sleep_disable_secondary_cpus()  suspend.c:446
          │         │    ║ 挂载点②  从核热下线 → kvm_offline_cpu()(§四)
          │         ├─ arch_suspend_disable_irqs()   关中断(弱默认 local_irq_disable, suspend.c:397)
          │         ├─ system_state = SYSTEM_SUSPEND
          │         ├─ syscore_suspend()             suspend.c:462  逆注册序
          │         │    ├─ kvm_suspend()            kvm_main.c:5651
          │         │    │    ║ 挂载点③  timer/vgic IRQ 下线 + hyp 退回 stub(§五·1)
          │         │    ├─ its_save_disable()       irq-gic-v3-its.c:4998
          │         │    └─ cpu_pm_suspend()         kernel/cpu_pm.c:180
          │         │         └─ CPU_PM_ENTER → hyp_init_cpu_pm_notifier
          │         └─ suspend_ops->enter = psci_system_suspend_enter  psci.c:540
          │              └─ cpu_suspend(0, psci_system_suspend)  arch/arm64/kernel/suspend.c:97
          │                   ├─ __cpu_suspend_enter(&state)  sleep.S:65
          │                   │    ├─ 保存 x19-x28/sp/lr 到 sleep_stack_data
          │                   │    └─ cpu_do_suspend()        proc.S:89
          │                   │         └─ 系统寄存器保存到 cpu_suspend_ctx
          │                   └─ fn() = psci_system_suspend()  psci.c:530
          │                        └─ invoke_psci_fn(PSCI SYSTEM_SUSPEND,
          │                                          __pa_symbol(cpu_resume))  psci.c:532-535
          │                             └─ SMC/HVC → EL3:全核掉电, DRAM 自刷新
          │                                (成功则不返回——返回即失败,EOPNOTSUPP)
          └─ suspend_finish()(唤醒后才执行)  suspend.c:556
```

要点:`cpu_suspend` 的 finisher 语义——`__cpu_suspend_enter` 返回非零后调用 finisher,finisher **永不返回**(成功路径);只有挂起失败才返回,`cpu_suspend` 强制转成 `-EOPNOTSUPP`(`arch/arm64/kernel/suspend.c:144-152`)。`cpu_resume` 的物理地址在掉电前经 PSCI 交给固件,是唤醒跳转的唯一凭据。

### 2.3 调用栈二:唤醒返回(EL3 → userspace)

```
唤醒中断 → EL3 固件恢复电源 → 按挂起时传入的物理地址跳转(关 MMU)
  cpu_resume                            sleep.S:101 (.idmap.text)
  ├─ init_kernel_el()                   EL2/EL1 最小化初始化(此时 KVM hyp 向量尚未重装)
  ├─ __cpu_setup() + __enable_mmu()     先经 idmap 开 MMU
  └─ _cpu_resume()                      sleep.S:116
      ├─ finalise_el2()                 恢复 EL2 基础状态(仍非 KVM hyp)
      ├─ 按 mpidr_hash 从 sleep_save_stash 取回 sleep_stack_data
      └─ cpu_do_resume()                proc.S:125  恢复系统寄存器
          → ret 回到 cpu_suspend() 的 else 分支   suspend.c:154
              └─ __cpu_suspend_exit()   suspend.c:44(替代物修补、local_daif_restore)
  ── 逐级返回到 suspend_ops->enter 调用点,suspend_enter 继续:
  syscore_resume()                       suspend.c:474  注册序
      ├─ cpu_pm_resume() → CPU_PM_EXIT  kernel/cpu_pm.c:187  ║ 挂载点③回
      ├─ its_restore_enable()           irq-gic-v3-its.c:5034  host ITS 重建
      └─ kvm_resume()                   kvm_main.c:5668   ║ 挂载点③回
           └─ kvm_arch_enable_virtualization_cpu()  arm.c:2309
                └─ cpu_hyp_reinit() → kvm_vgic_cpu_up()/kvm_timer_cpu_up()(§五·1)
  arch_suspend_enable_irqs()
  pm_sleep_enable_secondary_cpus()       suspend.c:481
      └─ 每核 cpuhp 上线 → kvm_online_cpu()  kvm_main.c:5609  ║ 挂载点②回(§四)
  dpm_resume_end()                       suspend.c:538 ← 设备恢复(VFIO/SMMU 在此)
  console_resume_all() → platform_resume_end()
  suspend_finish()                       suspend.c:556
      └─ suspend_thaw_processes()       suspend.c:562
           └─ vcpu 线程恢复运行 → vcpu_load
                └─ __vgic_v3_restore_state / timer_restore_state  ║ 挂载点①回(§三/§六)
```

要点:**hyp 的重装时机晚于 EL2 基础初始化**——`finalise_el2`(固件跳转后的第一程)只恢复 EL2 最小状态,KVM 的 hyp 向量/上下文由 `kvm_resume` 中的 `cpu_hyp_reinit` 重建,两者之间系统运行在 hyp-stub 上,这段窗口内不会进入 guest(进程仍冻结、IRQ 尚未开启)。

### 2.4 时序总览(KVM 挂载点在时间轴上的位置)

```
主机侧 (kernel/power/suspend.c)              KVM / 相关驱动
──────────────────────────────              ──────────────────────────────
suspend_devices_and_enter()
 └─ suspend_freeze_processes()   :387        QEMU 线程被冻结 → vcpu 停止执行
     └─ vcpu 任务被换出                         [§三: vcpu_put 状态落内存]
 └─ suspend_enter()             :462
     ├─ dpm 设备挂起(noirq)                    VFIO/SMMU 设备驱动各自 pm 回调
     ├─ pm_sleep_disable_secondary_cpus() :446  ── 每个从核热下线
     │    └─ cpuhp teardown                     [§四: kvm_offline_cpu]
     ├─ arch_suspend_disable_irqs() :457
     ├─ syscore_suspend() :462  ← 逆注册序执行
     │    ├─ kvm_suspend()          [kvm_main.c:5651]
     │    │    └─ kvm_arch_disable_virtualization_cpu()  [§五·1]
     │    │         ├─ kvm_timer_cpu_down()   [arch_timer.c:1139]
     │    │         ├─ kvm_vgic_cpu_down()    [vgic-init.c:717]
     │    │         └─ cpu_hyp_uninit() → cpu_hyp_reset()  [arm.c:2301]
     │    ├─ its_save_disable()     [irq-gic-v3-its.c:4998]  host ITS 存状态
     │    └─ cpu_pm_suspend()       [kernel/cpu_pm.c:180]
     │         └─ CPU_PM_ENTER → hyp_init_cpu_pm_notifier [arm.c:2339]
     │              └─ kvm_hyp_initialized 已为 0 → 空操作
     ├─ suspend_ops->enter() :468
     │    = psci_system_suspend_enter()      [drivers/firmware/psci/psci.c:540]
     │    └─ cpu_suspend(0, psci_system_suspend)  [arch/arm64/kernel/suspend.c:97]
     │         └─ PSCI SYSTEM_SUSPEND → 固件断电(全核掉电, DRAM 自刷新)
     │
     │  ════════════ 唤醒中断 → 固件恢复 → cpu_resume ════════════
     │
     ├─ syscore_resume() :474  ← 注册序执行
     │    ├─ cpu_pm_resume()         [kernel/cpu_pm.c:187]
     │    │    └─ CPU_PM_EXIT → 空操作(kvm_hyp_initialized 仍为 0)
     │    ├─ its_restore_enable()    [irq-gic-v3-its.c:5034]  host ITS 重建
     │    └─ kvm_resume()            [kvm_main.c:5668]
     │         └─ kvm_arch_enable_virtualization_cpu()  [arm.c:2309]
     │              ├─ cpu_hyp_init() → cpu_hyp_reinit()
     │              ├─ kvm_vgic_cpu_up()     [vgic-init.c:711]
     │              └─ kvm_timer_cpu_up()    [arch_timer.c:1132]
     └─ pm_sleep_enable_secondary_cpus() :481
          └─ 从核重新上线 → kvm_online_cpu()  [kvm_main.c:5609]
               └─ kvm_arch_enable_virtualization_cpu() 每核重建
```

**syscore 执行顺序证据**:挂起走 `list_for_each_entry_reverse`(`drivers/base/syscore.c:62`),恢复走正序(`:101`)。注册时序:`cpu_pm_init` 是 `core_initcall`(`kernel/cpu_pm.c:208`,开机即注册)→ host ITS 在 `its_init` 注册(`irq-gic-v3-its.c:5879`)→ `kvm_syscore` 在首个虚机启动时才注册(`kvm_main.c:5702`)。所以**挂起顺序 = KVM → ITS → cpu_pm,恢复顺序相反**——KVM 先降级、ITS 后断电;恢复时 ITS 先重建、KVM 最后恢复,顺序天然正确。

## 三、挂载点一:vcpu 冻结——断电瞬间没有 vcpu 在运行

deep 路径上有两道保险保证"固件断电时没有任何 vcpu 在执行 guest 代码":**freezer**(协作式,主要路径)与**从核热下线强制迁移**(兜底)。两者殊途同归:vcpu 任务的任何一次换出都经过 `kvm_sched_out`(`kvm_main.c:6389`,经 `kvm_preempt_ops.sched_out = kvm_sched_out` 注册于 `:6521`)→ `kvm_arch_vcpu_put`(`arch/arm64/kvm/arm.c:746`),vgic/timer 状态由此落内存。

### 3.1 冻结的本质:freezer 无法"抢下"正在跑的 CPU

`freeze_task`(`kernel/freezer.c:164`)只做两件事,对两类任务分别生效:

1. **正在 CPU 上运行的任务**(如正在跑 guest 的 vcpu 线程):`__set_task_frozen` 里 `task_is_runnable(p)` 直接返回 0(`freezer.c:117-118`)——**状态不动**,只能经 `fake_signal_wake_up`(`freezer.c:99`)→ `signal_wake_up(p, 0)` 置一个 `TIF_SIGPENDING` 标记;
2. **睡眠中的可冻结任务**:直接把 `__state` 换成 `TASK_FROZEN`(`freezer.c:139-140`)。

所以冻结运行中的 vcpu 是**协作式**的:freezer 只打标记,vcpu 任务自己走到检查点、自己走进冰箱。

### 3.2 调用栈一:freezer 侧(谁发的冻结请求)

```
echo mem > /sys/power/state
  state_store()                             kernel/power/main.c:799
  └─ pm_suspend → enter_state               suspend.c:636 → :575
      └─ suspend_prepare()                  suspend.c:372
          └─ suspend_freeze_processes()     suspend.c:387
              └─ freeze_processes()         kernel/power/process.c:121
                  └─ try_to_freeze_tasks(true)   process.c:28
                      └─ for_each_process_thread: freeze_task(p)  freezer.c:164
                          ├─ __freeze_task → __set_task_frozen    freezer.c:113
                          │    ├─ 任务在跑(task_is_runnable)→ 返回 0,不碰状态
                          │    └─ 任务睡眠且可冻结 → __state = TASK_FROZEN
                          └─ 用户任务: fake_signal_wake_up(p)     freezer.c:99
                               └─ signal_wake_up(p, 0) → 置 TIF_SIGPENDING
                      (循环重试,直到全部冻结或 20s 超时)
```

### 3.3 调用栈二:vcpu 正在跑 guest——检查点在"下一轮进 guest 之前"

`TIF_SIGPENDING` 不会打断 guest 执行;vcpu 每轮进入 guest 之前必经一次检查:

```
  kvm_arch_vcpu_ioctl_run 主循环            arm.c:1285+
    └─ kvm_xfer_to_guest_mode_handle_work(vcpu)   arm.c:1292
        └─ xfer_to_guest_mode_handle_work()  kernel/entry/virt.c:28
            └─ xfer_to_guest_mode_work(): ti_work &
                 (_TIF_SIGPENDING | _TIF_NOTIFY_SIGNAL)
                 → return -EINTR              virt.c:10-11
        └─ kvm_handle_signal_exit(vcpu)      kvm_host.h:2478-2485
             └─ run->exit_reason = KVM_EXIT_INTR
    → ret = -EINTR 退出主循环 → KVM_RUN ioctl 返回 -EINTR 给 QEMU
QEMU vcpu 线程返回用户态:
  do_signal → get_signal()
    └─ try_to_freeze()                        signal.c:2825
        └─ __refrigerator(false)              freezer.c:63
            ├─ __state = TASK_FROZEN(任务把自己标为冻结)
            └─ freezing(current) → schedule()  ← 睡进冰箱
                 │
                 └─ ★ 换出瞬间: kvm_sched_out  kvm_main.c:6389
                     └─ kvm_arch_vcpu_put(vcpu)  arm.c:746   ← "下位"完成
```

### 3.4 调用栈三:vcpu 阻塞在 WFI(guest 在等中断)

任务睡在 `TASK_INTERRUPTIBLE`(`kvm_vcpu_block`),freezer 两条路任选其一:

```
  ├─ __set_task_frozen 直接把 __state 换成 TASK_FROZEN(睡眠任务,直接冻)
  └─ fake_signal_wake_up 唤醒
      → kvm_vcpu_check_block: signal_pending(current) → -EINTR   kvm_main.c:3627
      → ioctl 返回 → 同 3.3 后半段: get_signal → try_to_freeze
          → refrigerator → schedule() 换出 → kvm_sched_out → vcpu_put
```

### 3.5 vcpu_put:状态落内存的关键点

`kvm_arch_vcpu_put`(`arm.c:746`)依次执行 `kvm_vgic_put`(`vgic.c:1200`)、`kvm_timer_vcpu_put`(`arch_timer.c:918`):

- `kvm_vgic_put → vgic_v3_put` 通过 `kvm_call_hyp(__vgic_v3_save_aprs, cpu_if)` 把 ICH_APxR 读回内存,并 `vgic_v4_put` 把 vPE 置为 non-resident(`vgic-v4.c:358` → `its_make_vpe_non_resident`);
- `kvm_timer_vcpu_put` 把 CNTV_CVAL/CTL/OFF 读回软件态(`timer_save_state`);若 vcpu 即将阻塞(guest 在 WFI),`kvm_timer_blocking`(`arch_timer.c:595`)会启动 **bg_timer** 后台 hrtimer(`CLOCK_MONOTONIC`,`hrtimer_setup` 于 `arch_timer.c:1113`),保证"guest 定时器到期时即使 vcpu 没在跑也能被唤醒"。

### 3.6 超时兜底与 thaw

1. **"下位"的完成标志是换出,不是收到信号**:`vcpu_put` 发生在任务 `schedule()` 换出的瞬间,与"收到假信号"、"走进冰箱"同链但不同时。
2. **20s 超时兜底**:guest 忙循环 + 关中断的极端情况下 vcpu 可能长时间不 exit、不检查 `TIF_SIGPENDING`。freezer 有 20 秒超时(`freeze_timeout_msecs = 20 * MSEC_PER_SEC`,process.c:26),超时后打印 `Freezing user space processes failed` 并继续 suspend——由下一关 `pm_sleep_disable_secondary_cpus` 强制迁移任务、逼其换出走 `vcpu_put`(§四)。
3. **thaw 是逆过程**:`suspend_thaw_processes` → `__thaw_task` 把 `TASK_FROZEN` 换回 `saved_state`(process.c:201)→ refrigerator 里的 `schedule()` 返回 → `try_to_freeze` 返回 → QEMU 线程回到主循环重新 `ioctl(KVM_RUN)` → `vcpu_load` → vgic/timer 状态重刷硬件。

**结论**:断电瞬间 vcpu 的 vgic CPU interface 状态、定时器状态已经全部落在内存里,这是休眠后能完整恢复的前提。休眠本身不需要任何额外的 vgic/timer 保存动作。

## 四、挂载点二:从核热下线

`pm_sleep_disable_secondary_cpus()`(`kernel/power/suspend.c:446`)对每个从核走 cpuhp 下线,`kvm_offline_cpu`(`kvm_main.c:5629`)→ `kvm_disable_virtualization_cpu`(`:5619`)→ `kvm_arch_disable_virtualization_cpu`(`arm.c:2329`):

```
kvm_arch_disable_virtualization_cpu()          arm.c:2329
  ├─ kvm_timer_cpu_down()                      arch_timer.c:1139
  │    └─ disable_percpu_irq(host_vtimer_irq)  物理 vtimer 中断下线
  ├─ kvm_vgic_cpu_down()                       vgic-init.c:717
  │    └─ disable_percpu_irq(maint_irq)        vgic 维护中断下线
  └─ cpu_hyp_uninit()                          arm.c:2301
       └─ cpu_hyp_reset() + kvm_hyp_initialized = 0
            └─ nVHE: __hyp_reset_vectors()      退回 hyp-stub 向量
               VHE : 空操作(无独立 EL2 向量)    arm.c:2229
```

从核下线后,该核在 `arch_cpu_idle_dead` 中进入 PSCI CPU_OFF 或深 idle,EL2 上下文随断电消失——这没关系,因为 `kvm_hyp_initialized` 已被清零,重新上线时会整体重建。

## 五、挂载点三:引导核 syscore + cpu_pm 双钩子

引导核(执行 suspend 的核)是唯一"不断电流程中仍带电"的核,它靠 **syscore 钩子为主、cpu_pm 通知链为辅**两个机制处理。

### 5.1 syscore 钩子:真正干活的路径

`kvm_syscore_ops`(`kvm_main.c:5676-5680`)注册于首个虚机启动时:

```
kvm_suspend()                                kvm_main.c:5651
  └─ kvm_disable_virtualization_cpu(NULL)    kvm_main.c:5619
       └─ kvm_arch_disable_virtualization_cpu()  arm.c:2329
            (与 §四 相同: timer/vgic percpu IRQ 下线 + hyp 退回 stub)
            + virtualization_enabled = false  kvm_main.c:5626

kvm_resume()                                 kvm_main.c:5668
  └─ WARN_ON_ONCE(kvm_enable_virtualization_cpu())  kvm_main.c:5594
       └─ kvm_arch_enable_virtualization_cpu()      arm.c:2309
            ├─ cpu_hyp_init()                      arm.c:2293
            │    └─ kvm_hyp_initialized == 0 → cpu_hyp_reinit()  arm.c:2286
            │         ├─ cpu_hyp_reset()           先确保 stub 状态
            │         ├─ cpu_hyp_init_context()    arm.c:2264
            │         │    ├─ kvm_init_host_cpu_context()  重建 EL2 host 上下文
            │         │    └─ nVHE: cpu_init_hyp_mode()  arm.c:2213 重装 EL2 向量
            │         └─ cpu_hyp_init_features()   arm.c:2273
            │              ├─ kvm_timer_init_vhe() / kvm_debug_init_vhe()
            │              └─ kvm_vgic_init_cpu_hardware()  vgic-init.c:756
            │                   └─ __vgic_v3_init_lrs()  清空并重初始化 ICH LR
            ├─ kvm_vgic_cpu_up()                   vgic-init.c:711
            │    └─ enable_percpu_irq(maint_irq)
            └─ kvm_timer_cpu_up()                  arch_timer.c:1132
                 └─ enable_percpu_irq(host_vtimer_irq / host_ptimer_irq)
```

注意两个 flag 的接力:`kvm_hyp_initialized` 挂起时被 `cpu_hyp_uninit` 清零(`arm.c:2305`),恢复时 `cpu_hyp_init` 据此判断需要整体重装 hyp;`virtualization_enabled` 同理驱动 `kvm_resume` 侧的 `kvm_enable_virtualization_cpu`(`kvm_main.c:5596`)。

### 5.2 nVHE vs VHE:同一时间轴的两种命运

两种模式走同一条挂起/恢复时间轴,唯一的分叉点在 `cpu_do_suspend`(`proc.S:89-124`):它用 **EL1 名字**保存系统寄存器(`mrs x8, vbar_el1` 等),在 E2H 重定向下这些名字命中 EL2 寄存器——所以 **VHE 的 EL2 状态被通用挂起机制顺带保存,nVHE 的 EL2 无人可救**(EL1 代码架构上读不到 EL2 寄存器)。后续所有动作都由这一个分叉决定:

- **挂起侧**:nVHE 必须降级——`cpu_hyp_reset` 把 `VBAR_EL2` 指回 hyp-stub(`arm.c:2227-2231`),让断电瞬间 EL2 处于无页表/TLB 依赖的干净状态;VHE 的 reset 是**空操作**,它的"降级"发生在更早的 vcpu_put 处——`__vgic_v3_deactivate_traps` 关掉 HCR_EL2 的 guest 陷阱位,EL2 回到纯 host 态。
- **唤醒早期**(`init_kernel_el`,head.S:270-331):把 EL2 恢复到"开机态"——nVHE 装回 hyp-stub 向量(head.S:304-306)后 eret 到 EL1;VHE 经 `init_el2_hcr`(`el2_setup.h:19-66`)探测后设 `HCR_E2H`,直接留在 EL2。
- **唤醒后期**(`kvm_resume`):nVHE 靠 `__kvm_hyp_init` 从每 CPU 内存模板 `kvm_nvhe_init_params`(`kvm_asm.h:207-217`,常驻 DRAM 不掉)整体重灌 EL2;VHE 的 EL2 已被 `cpu_do_resume` 恢复,KVM 只补刷**四个不在通用保存清单里的专属位**:

| VHE 补刷位 | 作用 | 代码 |
|---|---|---|
| `cpu_set_hyp_vector()` | Spectre 加固向量 slot | arm.c:2253 |
| `kvm_timer_init_vhe()` | `CNTHCTL_EL2.ECV`(复位丢失) | arch_timer.c:1630 |
| `kvm_debug_init_vhe()` | 清 `PMSCR_EL1.E{0,1}SPE`(复位后 UNKNOWN) | debug.c:110 |
| `kvm_vgic_init_cpu_hardware()` | 清空并重初始化 ICH LR | vgic-init.c:756 |

| | nVHE | VHE |
|---|---|---|
| EL2 是谁的 | KVM 的"租界"(独立向量/页表/栈) | 内核的家(E2H=1) |
| `cpu_do_suspend` 救谁 | 只救 EL1,**EL2 无人可救** | E2H 重定向 → 顺带救了 EL2 |
| 挂起侧 KVM 动作 | 降级:VBAR_EL2 → hyp-stub | vcpu_put 关 guest 陷阱;reset 空操作 |
| 唤醒早期 | 装回 hyp-stub | 设 E2H 留 EL2 |
| 唤醒后期 | 从内存模板**整体重建** | 已恢复,**补刷四个专属位** |
| 机制本质 | 拆 → 装(两条路径都真实干活) | 随大流 → 打补丁 |

**分析/验证提示**:先确认平台跑哪个模式(`is_kernel_in_hyp_mode()`);两种模式各自的薄弱点不同——VHE 漏刷四个专属位会静默失效(如 ECV 丢失导致 CNTPOFF 语义错误),nVHE 的薄弱点是"stub 窗口"(唤醒早期到 `kvm_resume` 之间 EL2 停在 hyp-stub,安全前提是进程冻结 + IRQ 关闭)。x86 可视为"天生 VHE"(虚拟化状态在内存 VMCS 中,恢复只需 VMXON + VMPTRLD),无 nVHE 式问题。

### 5.3 pkvm:不走 cpu_pm 通道,走 hyp 侧 PSCI relay 通道

> 前置说明:pkvm 是 **nVHE 专属**模式,与 VHE 互斥——`early_kvm_mode_cfg` 在 VHE 下拒绝 protected 模式("Protected KVM not available with VHE",`arm.c:3122-3126`),因为 VHE 下 host 内核自身运行在 EL2,pkvm 赖以生存的"host 待在 EL1、hyp 独占 EL2"隔离边界不存在。本节所述机制天然是 nVHE 场景。

#### 5.3.1 调用栈:挂起入口(host 的 SMC 被 hyp 截获)

```
[host, EL1] 挂起路径照常走,只有一处不同:syscore 的 kvm_suspend
            只关 timer/vgic percpu IRQ,不碰 EL2(见 5.3.3 解释)
  suspend_ops->enter = psci_system_suspend_enter       psci.c:540
  └─ cpu_suspend → fn = psci_system_suspend            psci.c:530
      └─ invoke_psci_fn → SMC 指令
          ║  pkvm 下 HCR_EL2.TSC 置位 → host 的 SMC 全部陷入 EL2
          ▼
[EL2, hyp] handle_host_smc(host_ctxt)                  hyp-main.c:815
  └─ kvm_host_psci_handler(host_ctxt, func_id)         psci-relay.c:294
      └─ 函数号 = SYSTEM_SUSPEND → psci_system_suspend()  psci-relay.c:180
          ├─ suspend_args->pc/r0 = host 的恢复地址(每 CPU 私有)
          └─ psci_call(SYSTEM_SUSPEND,
                       __hyp_pa(&kvm_hyp_cpu_resume),  ← 唤醒入口换成 hyp 的
                       __hyp_pa(init_params))          ← 附上 EL2 重建模板
              └─ EL2 发 SMC → EL3 固件断电
```

#### 5.3.2 调用栈:唤醒返回(hyp 自重建后再交还 host)

```
固件 → 按挂起时传入的入口跳到 kvm_hyp_cpu_resume(EL2)   hyp-init.S:180
  ├─ init_el2_hcr + __kvm_init_el2_state                EL2 基线初始化
  └─ ___kvm_hyp_init(init_params)                       hyp-init.S:70-167
      └─ 用内存模板重灌 EL2:TTBR0/TCR/MAIR/HCR/SCTLR/VBAR/TPIDR + 开 MMU
  └─ blr → __kvm_host_psci_cpu_resume_entry             psci-relay.c:233
      └─ __kvm_host_psci_cpu_entry(保存的 pc, r0)       psci-relay.c:216
          └─ ELR/SPSR = host 挂起前的恢复点 → __host_enter
              → host 回到 cpu_resume 路径继续(EL1,MMU 关)
```

#### 5.3.3 解释:为什么 pkvm 不用 cpu_pm 通道

1. **cpu_pm 通知链的本质是"host 侧拆装 EL2"**:`CPU_PM_ENTER → cpu_hyp_reset` 把 VBAR_EL2 指回 boot stub,`CPU_PM_EXIT → cpu_hyp_reinit` 从模板重灌——这套机制的前提是 EL2 是 host 的财产、host 有权拆装;
2. **pkvm 的威胁模型是 host 不可信**:EL2 及其上的受保护状态不允许 host 触碰。因此 `hyp_cpu_pm_init` 不注册通知链(`arm.c:2378`)、`cpu_hyp_uninit` 跳过 reset 与 flag 清零(`arm.c:2303`)、`do_pkvm_init` 注释明说防止 host 之后 re-init(`arm.c:2560-2562`);
3. **替代机制是 PSCI relay**(仅 pkvm 分支初始化,`arm.c:2881`):pkvm 下 HCR_EL2.TSC 让 host 的 SMC 全部陷入 EL2,由 hyp 代理后再发真正的 PSCI,并把唤醒入口换成 hyp 自己的 `kvm_hyp_cpu_resume` + 重建模板 `kvm_nvhe_init_params`(常驻 DRAM)——**EL2 的存亡由 hyp 自己负责,host 全程无接触**。

#### 5.3.4 支持性结论

pkvm 下 deep 休眠**机制上支持**,走 relay 通道而非 cpu_pm 通道。该路径是上游活跃维护的代码:2020 年 David Brazdil 系列引入 relay 即为 pkvm 的 CPU hotplug 服务;2023 年 Quentin Perret 修复恢复后 EL2 状态未 finalise 的 bug;2025 年 Mark Rutland/Ahmed Genidi 修复 CPU_ON/CPU_SUSPEND/SYSTEM_SUSPEND 三个冷入口的缺失初始化(v6.14-rc3,有真实崩溃报告)——三者共同证明 SYSTEM_SUSPEND relay 路径在真实环境使用。两点保留:① 受保护 VM 跨休眠的端到端语义仍建议实测;② web 资料称 pkvm 禁用 host 的 kexec 与 hibernation(S4),本树未找到对应禁用代码 `[EXTERNAL]`。

### 5.4 GICv4.0 vs GICv4.1:直投注入的挂起风险分析

**架构事实**:GICv4.0 与 GICv4.1 都有 per-vPE 的 VPT 与 `GICR_VPENDBASER` 寄存器,VLPI pending 权威存储于内存。代码证据:VMAPP 命令的 VPT 地址/大小字段两代通用(编码在 `is_v4_1()` 判断之前,its-v3-its.c:903-909);`its_vpe_schedule` 无条件写 `GICR_VPENDBASER`(its-v3-its.c:4008-4040),由 `vgic_v4_load → irq_set_affinity(vpe->irq)` 触发(its-v3-its.c:3904、4081)。GICv4.1 的新增:VMAPP 的 VCONF_ADDR、VPENDBASER 的 Dirty/IDAI 位、vSGI(替代 doorbell)、unmap 自同步(无需 VSYNC,its-v3-its.c:896-898)。

**VPT 基址的保存与恢复**:`GICR_VPENDBASER` **无寄存器快照**——挂起时无任何软件保存它:host GIC 驱动的 cpu_pm 回调仅重初始化 CPU interface 与 re-enable redistributor(irq-gic-v3.c:1479-1495),不碰 GICR 数据寄存器;vPE 层由 vcpu_put → `vgic_v4_put` 置 non-resident,寄存器随掉电丢失。恢复靠**软件从内存页重算**:

```
vcpu_load → vgic_v4_load                        vgic-v4.c:368
  └─ irq_set_affinity(vpe->irq, cpumask_of(cpu))
      └─ its_vpe_set_affinity                   its-v3-its.c:3904
          └─ its_vpe_schedule                   its-v3-its.c:4005
              ├─ 从 its_vm->vprop_page 重算 → 写 GICR_VPROPBASER
              └─ 从 vpe->vpt_page 重算     → 写 GICR_VPENDBASER
  └─ its_make_vpe_resident                      irq-gic-v4.c:261
      └─ SCHEDULE_VPE(VMAPP,自带 VPT 地址)→ ITS 重新挂接、从内存读回 pending
```

即"寄存器是影子、内存页是本源":`vpt_page`/`vprop_page` 在 DRAM 中保留,唤醒后由软件重算写回——与 host 自身 GICR PPI 配置(无快照、resume 时各用户按需重新 enable)同一策略。

| | 两代共同 | GICv4.1 独有 |
|---|---|---|
| VLPI pending 本源 | **内存 VPT**(per-vPE),掉电不丢 | — |
| VPT 基址恢复 | vcpu_load 时重写 VPENDBASER + VMAPP 重挂接 | VCONF_ADDR、unmap 自同步 |

**两代差异**:`vgic_v3_save_pending_tables` 里 `vlpi_avail` 仅在 `has_gicv4_1` 时置位(vgic-v3.c:608-610)——v4.1 的 unmap 自同步使"unmap 后读 VPT"安全;v4.0 上 host 读 VPT 与 ITS 并发更新有竞态,故 KVM 在 v4.0 不读回。这是 KVM 读回能力的限制,不影响挂起恢复(v4.0 的恢复同样走 VMAPP + VPENDBASER 重写)。

**上游支持情况**:Linux **没有专门的 GICv4 挂起/恢复代码**——支持是间接的:数据层靠内存表(VPT/prop 页,DRAM 保留),寄存器层靠恢复路径重建。寄存器重建有两条路:① CPU 重新上线时 `its_cpu_init_lpis`(本树 its-v3-its.c:3150,调用点 :5433)初始化 `VPROPBASER.IDbits`、清 `VPENDBASER.Valid`——上游 [commit 6479450f "Fix occasional VLPI drop"](https://git.raptorcs.com/git/blackbird-obmc-linux/commit/?id=6479450f72c1391c03f08affe0d0110f41ae7ca0) 修复的正是"IDbits 未知 → 超范围 INTID 被 GIC **静默丢弃**"这类寄存器残留/未知状态问题,休眠唤醒恰是此类场景的放大器;② vcpu_load → `its_vpe_schedule` 重写 + VMAPP 重挂接(见上)。相关上游修复还有:[b321c31c9b7b "Make the doorbell request robust w.r.t preemption"](https://git.armlinux.org.uk/cgit/linux.git/commit/include/kvm?id=b321c31c9b7b309dcde5e8854b741c8e6a9a05f0)(2023,Cc stable,doorbell 丢失致 VM 无法启动)与 f66b7b151e00 "Try to save VLPI state in save_pending_tables" 及其竞态修复——后者证实 **VLPI 状态读回是 GICv4.1 专属**(unmap 全部 vPE 后读 VPT),与上述 `vlpi_avail` 门控一致。

**结论**:两代 VLPI pending 都在内存 VPT,挂起/恢复理论上都闭环,且上游无"GICv4 不支持休眠"的任何声明;但 VPENDBASER/VPROPBASER 的历史 bug 全部集中在"状态丢失/残留"类,休眠唤醒让每颗核经历一次"寄存器归零再重建",建议实测覆盖:v4.0 doorbell(vpe_proxy 映射在 ITS device 表,BASER 恢复后仍在;doorbell pending 仅是唤醒提示)与 v4.1 vSGI 配置在恢复后的重建衔接。

### 5.5 VFIO + SMMU:直通设备在 deep 挂起下不可用

问题不在 KVM,在 SMMUv3 驱动:**本树 SMMUv3 没有任何系统挂起支持**——`arm_smmu_driver`(arm-smmu-v3.c:5651-5659)只有 `probe/remove/shutdown`,没有 `.pm` 字段,也没有 syscore/cpu_pm 注册。

```
挂起: dpm_suspend → 设备驱动(如 NVMe)suspend    设备被停掉
      SMMU 随电源域掉电:Stream Table Base 寄存器、STE/CD 缓存、
      命令队列、SMMU_CR0 配置 —— 全部丢失,无人保存

恢复: dpm_resume → 设备驱动 resume 重新初始化设备、重新申请 MSI
      ├─ MSI 路径:host ITS 由 its_restore_enable 恢复 → 中断链路 ✅ 能活
      └─ DMA 路径:设备恢复 DMA 后,事务经过 SMMU
                   → 无有效 Stream Table/STE → SMMU abort(DMA fault)
                   → 直通设备在 guest 里彻底不可用 ❌
```

- **VFIO 侧**(vfio_iommu_type1.c)同样没有任何 resume 逻辑——domain/映射都在内存里,但没有驱动侧恢复目标可言;
- **s2idle 不受影响**:电源不断,SMMU 状态保持,直通跨 s2idle 可用;**deep 是分水岭**;
- 故障形态:挂起/唤醒本身成功,唤醒后 guest 里 passthrough 设备 DMA 失败(SMMU event queue 报 `C_BAD_STE`/abort 类错误),host 需要重启或 SMMU 重新初始化;
- 解决路径:为 SMMUv3 驱动补系统挂起支持(syscore/cpu_pm 保存恢复 SMMU 寄存器、重建队列与 STE/CD,或整表保存)——这是设备驱动侧工程,与本树 KVM 无关,但直接决定"VFIO 虚机能否跨 deep 休眠"。



## 六、子系统状态清单(save/restore 配对表)

| 子系统 | 状态保存介质 | 恢复点 | 风险评估 |
|---|---|---|---|
| guest RAM | DRAM 自刷新,天然保留 | 无 | ✓ 无风险 |
| vgic CPU interface(APR/LR/VMCR) | vcpu 换出时 `__vgic_v3_save_aprs` 存入 `vgic_v3_cpu_if`(`vgic-v3.c` vgic_v3_put) | vcpu 重新调度 → `kvm_vgic_load`(`vgic.c:1172`)→ `vgic_v3_load` → restore | ✓ 闭环;⚠️ 宽度截断会被此往返放大(如 AP1R 的 64bit 截断) |
| vgic 维护中断 | percpu IRQ | `kvm_vgic_cpu_up`(`vgic-init.c:711`) | ✓ 闭环 |
| 虚拟/物理定时器 | `timer_save_state` 存入软件态;`bg_timer` hrtimer 记录最早到期 | `kvm_timer_vcpu_load` → `timer_restore_state` 重写 CNTV_CTL/CVAL/OFFSET | ✓ 闭环(恢复后对活动计数器重新采样注入,迟到但必达) |
| host GIC CPU interface | — | `gic_cpu_pm_notifier`(`irq-gic-v3.c:1479`→ `:1482` CPU_PM_EXIT 分支重初始化) | ✓ 闭环(与 KVM 恢复顺序:先 GIC 后 KVM) |
| host ITS | `its_save_disable`(`irq-gic-v3-its.c:4998`) | `its_restore_enable`(`:5034`),先于 `kvm_resume` 执行 | ✓ 闭环 |
| GICv4 vPE/VLPI | vcpu_put 时 vPE 已 non-resident(`vgic-v4.c:358`);pending 位在内存 VPT(per-vPE,两代都有) | vcpu_load → `vgic_v4_load`(`vgic-v4.c:368`)→ `its_vpe_schedule` 重写 VPENDBASER + VMAPP 重挂接 | ✅ 详见 §五·4:两代都理论闭环;仅建议实测 doorbell/vSGI 衔接 |
| virtio(模拟/kernel 态) | 队列全在 guest RAM | 无 | ✓ 低风险,验证 IO 连续性即可 |
| VFIO 直通 + SMMU | 设备驱动 pm 回调 + ITS restore | 设备驱动 pm 回调 | ❌ 详见 §五·5:SMMUv3 驱动无挂起支持,deep 下 DMA 路径不可用;MSI 路径经 ITS restore 可恢复 |
| pkvm | — | — | ✓ 机制支持:EL2 经 PSCI relay 由 hyp 自重建(§五·3,有上游修复记录);受保护 VM 跨休眠语义建议实测 |


**结论**:deep + VFIO 直通在 arm64 上目前**不可用**(SMMU 驱动缺失挂起支持);GICv4.0 直投有 pending 丢失风险待实测;两者都不影响纯虚拟化(无直通/无 GICv4.0 直投)场景的结论。

## 参考链接

- [arm64 KVM cpu_pm ENTER_FAILED 修复(James Morse)](https://android.googlesource.com/kernel/common/+/58d6b15e9da5042a99c9c30ad725792e4569150e%5E%21/virt/kvm/arm/arm.c)
- [arm/arm64 KVM: checkpoint/restore 竞态致 timer 中断丢失](https://android.googlesource.com/kernel/common.git/+/1a74847885cc87857d631f91cca4d83924f75674)
- [arm/arm64 KVM: Fix arch timer behavior for disabled interrupts](https://git.kernel.dk/cgit/linux/commit/virt/kvm?id=cff9211eb1a1f58ce7f5a2d596b617928fd4be0e)
- [kvm: arm64: Intercept host's CPU_SUSPEND PSCI SMCs(David Brazdil,pkvm relay 引入)](https://lkml.org/lkml/2020/11/16/1378)
- [KVM: arm64: Finalise EL2 state from pKVM PSCI relay(Quentin Perret)](https://lkml.iu.edu/hypermail/linux/kernel/2302.0/00356.html)
- [KVM: arm64: PSCI relay fixes(Mark Rutland / Ahmed Genidi,v6.14-rc3,覆盖 CPU_ON/CPU_SUSPEND/SYSTEM_SUSPEND 冷入口)](http://lists.infradead.org/pipermail/linux-arm-kernel/2025-February/1005885.html)
- [irqchip/gic-v4: Fix occasional VLPI drop(6479450f,IDbits/VPENDBASER 残留状态)](https://git.raptorcs.com/git/blackbird-obmc-linux/commit/?id=6479450f72c1391c03f08affe0d0110f41ae7ca0)
- [KVM: arm64: vgic-v4: Make the doorbell request robust w.r.t preemption(b321c31c9b7b)](https://git.armlinux.org.uk/cgit/linux.git/commit/include/kvm?id=b321c31c9b7b309dcde5e8854b741c8e6a9a05f0)
- [GICv4 直接注入与 VPENDBASER/VPT 上下文切换协议(GICv3 软件 overview)](https://code84.com/831910.html)

---

*分析时间:2026-09-03。基于本树 v7.3-rc1(分支 `vgic-ap1r-64bit`,含 GICv5、`soft_timer` 重写的 arch_timer、NV 嵌套等下游特性,与 mainline 有差异)。代码引用以分析时工作区行号为准。*
