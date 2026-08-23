# arm64 IPI 时延测试驱动

> 目标：调研arm64 IPI 软件发送方式，编写一个在 arm64 上测试 IPI（Inter-Processor Interrupt）时延的内核驱动。
> 核心约束：**测量值必须对应真实的硬件中断传播时延，而不是被内核快捷路径（shortcut）绕过中断后的虚假数值。**

---

## 1. IPI 是什么

### 1.1 概念与硬件承载

IPI（Inter-Processor Interrupt）是 SMP 系统中 **软件生成**的中断：一个 CPU 写一条寄存器/指令，GIC（中断控制器）向一个或多个目标 CPU 投递中断，用于核间通信（函数调用、重调度、唤醒、停机等）。与 SPI/PPI 的区别：无物理连线、由软件随时生成、目标以 CPU 列表寻址、Linux 中按 **per-CPU 中断**注册（每个 CPU 各自处理、无锁）。

本树（Linux v7.2）arm64 的两种硬件承载：

| | GICv2/v3 | GICv5（本树新硬件） |
|---|---|---|
| IPI 承载 | **SGI**（ID 0-7 被内核占用） | **per-CPU LPI**（每 (CPU × IPI类型) 一条） |
| 发送 | 写 `ICC_SGI1R_EL1`（GICv2 为 `GICD_SGIR` MMIO） | `CDPEND` GIC 指令直接置目标 PE 的 LPI pending 位 |
| 优先级 | 固定最高 0xF0 `[HARDWARE]` | — |
| 注册方式 | `request_percpu_irq`（1 类型 1 virq，所有 CPU 共享） | `request_irq` + `irq_force_affinity`（每 CPU 独立 irq） |
| 接收 handle | `handle_percpu_devid_irq`（`irq-gic-v3.c:1562`） | `handle_percpu_irq`（`irq-gic-v5.c:887`） |

arm64 通过 `set_smp_ipi_range_percpu()`（`arch/arm64/include/asm/smp.h:73`）从 irqchip 接管 IPI 中断线：GICv3 上交 8 个 SGI（`irq-gic-v3.c:1425`），GICv5 上交 `8 × nr_cpu_ids` 条 LPI（`irq-gic-v5.c:1030`，`GICV5_IPIS_PER_CPU = MAX_IPI = 8`）。

### 1.2 内核抽象与数据结构拓扑

```
enum ipi_msg_type (arch/arm64/include/asm/smp.h:53-68):
  IPI_RESCHEDULE=0  IPI_CALL_FUNC=1  IPI_CPU_STOP=2  IPI_CPU_STOP_NMI=3
  IPI_TIMER=4       IPI_IRQ_WORK=5   NR_IPI=6
  [≥NR_IPI: IPI_CPU_BACKTRACE / IPI_KGDB_ROUNDUP，MAX_IPI=8]

arch/arm64/kernel/smp.c
┌─────────────────────────────────────────────────────────────┐
│ ipi_irq_base        — IPI virq 基址                          │
│ nr_ipi = 8          — IPI 类型数                             │
│ percpu_ipi_descs    — false=SGI共享 / true=per-CPU LPI       │
│                                                              │
│ DEFINE_PER_CPU_READ_MOSTLY(struct ipi_descs, pcpu_ipi_desc)   │
│   ┌──────────────────────────────┐                           │
│   │ struct ipi_descs             │                           │
│   │   descs[MAX_IPI] ──┐         │  irq_desc *                │
│   └────────────────────┼─────────┘                           │
└────────────────────────┼─────────────────────────────────────┘
                         ▼
        SGI 模式: 所有 CPU 指向同一组 desc（每类型 1 个 irq_desc）
        LPI 模式: 每 CPU 指向各自的 desc（每类型 nr_cpu_ids 个）
irq_desc → irq_data → irq_chip:
  GICv3: gic_chip / gic_eoimode1_chip（.ipi_send_mask = gic_ipi_send_mask）
  GICv5: gicv5_ipi_irq_chip（.ipi_send_single = gicv5_ipi_send_single）
```

---

## 2. IPI 的代码栈（通用发送/接收链路）

### 2.1 发送侧通用链路

```
arch 出口（arm64）                          // arch/arm64/kernel/smp.c
  smp_cross_call(target, ipinr)             // :1030
    ├─ trace_ipi_raise(target, reason)      // :1032  发送侧 tracepoint
    └─ arm64_send_ipi(target, ipinr)        // :917
        ├─ [SGI 模式] __ipi_send_mask(desc, mask)        // :921-922
        │    └─ gic_ipi_send_mask()          // drivers/irqchip/irq-gic-v3.c:1379
        │        ├─ dsb(ishst)               // :1390  数据先于 IPI 可见
        │        ├─ gic_write_sgi1r(val)     // :1376  写 ICC_SGI1R_EL1
        │        └─ isb()                    // :1401
        └─ [LPI 模式] 逐 CPU __ipi_send_single(desc(cpu), cpu)   // :924-925
             └─ gicv5_ipi_send_single()      // irq-gic-v5.c:487
                 └─ irq_chip_retrigger_hierarchy()
                     └─ gicv5_iri_irq_write_pending_state()
                         └─ gic_insn(cdpend, CDPEND)   // :429 硬件指令
```

要点：`gic_ipi_send_mask()` 对 mask 中**每个 CPU（包括发送者自己）都真实写一次寄存器**（`irq-gic-v3.c:1392-1398`）；`__ipi_send_single`/`__ipi_send_mask`（`kernel/irq/ipi.c:227-303`）无任何 self 快捷判断。**发送侧不存在软件内联捷径**——这是 arm64 与某些架构的重要区别。

### 2.2 接收侧通用链路

```
硬件投递（SGI / LPI）
  → 异常向量 kernel_ventry → el1h_64_irq_handler        // entry-common.c:525
      └─ el1_interrupt(regs, handle_arch_irq)           // :514
          └─ __el1_irq()                                // :501
              ├─ irq_enter_rcu()                        // :508
              ├─ do_interrupt_handler()                 // :152
              │   └─ gic_handle_irq                    // set_handle_irq, irq-gic-v3.c:2045
              │       └─ __gic_handle_irq_from_irqson() // :855
              │           ├─ gic_read_iar()            // :860  读 IAR
              │           ├─ gic_complete_ack(irqnr)   // :823  EOI
              │           └─ generic_handle_domain_irq() // :825
              │               └─ handle_irq_desc()
              │                   └─ handle_percpu_devid_irq() // :1562（SGI 域）
              │                       ├─ __kstat_incr_irqs_this_cpu() // chip.c:916
              │                       └─ handle_irq_event_percpu()
              │                           └─ ipi_handler()          // smp.c:1022
              │                               └─ do_handle_IPI()    // smp.c:963
              │                                   ├─ trace_ipi_entry()  // :968
              │                                   ├─ 按类型分发（见第 3 章）
              │                                   └─ trace_ipi_exit()   // :1019
              └─ irq_exit_rcu()                    // :510
```

GICv5 差异：根处理器为 `gicv5_handle_irq`（`irq-gic-v5.c:944`，经 `set_handle_irq` `:1175` 注册），IPI 域的 handle 函数为 `handle_percpu_irq`。

两个对测量至关重要的既定事实：
- **kstat 计数先于回调执行**：`handle_percpu_devid_irq()` 在跑 `ipi_handler` 之前必 `__kstat_incr_irqs_this_cpu()`（`chip.c:916`）——逐轮判决机制（5.2）的基石；
- **入口链固定**：从异常向量到 `do_handle_IPI` 之间的软件栈每次投递都完整执行，无旁路。

---

## 3. 不同使用场景下的调用栈

### 3.1 `smp_call_function_single()` —— 远程单核函数调用（测量载体之一）

用途：在指定 CPU 上执行一个函数。发送者必须进程上下文、IRQ 开启（`kernel/smp.c:695-704`）。

```
发送栈（sender CPU）：
smp_call_function_single(cpu, func, info, wait)      // kernel/smp.c:671
  ├─ get_cpu()：关抢占，防 CPU 移除（:687）
  ├─ csd_stack: {CSD_FLAG_LOCK | CSD_TYPE_SYNC}      // :675-676 栈上 csd
  └─ generic_exec_single(cpu, csd)                   // :455
      ├─ ★ cpu == self → 内联执行，无 IPI（:461-477，见 5.1-①）
      ├─ cpu 下线 → -ENXIO（:479-482）
      └─ __smp_call_single_queue(cpu, &csd->node.llist)  // :414
          ├─ llist_add() 返回 false → 不发 IPI（队列非空，:446，见 5.1-②）
          └─ 返回 true → send_call_function_single_ipi(cpu)  // :115
              ├─ call_function_single_prep_ipi()   // sched/core.c:3934，见 5.1-③
              ├─ trace_ipi_send_cpu()              // :118
              └─ arch_send_call_function_single_ipi(cpu)  // arm64 smp.c:857
                  └─ smp_cross_call → ICC_SGI1R_EL1 / CDPEND（第 2.1 节）
  └─ wait=1 → csd_lock_wait(csd)                    // :722 等回调执行完

接收栈（target CPU，第 2.2 节链路的 IPI_CALL_FUNC 分支）：
do_handle_IPI(IPI_CALL_FUNC)                        // arm64 smp.c:975-977
  └─ generic_smp_call_function_single_interrupt()   // kernel/smp.c:495
      └─ __flush_smp_call_function_queue(true)      // :514
          ├─ llist_del_all + 逆序（:529-530）
          ├─ CSD_TYPE_SYNC: csd_do_func(func, info, csd)  // :569-581 ← 回调执行点
          │    └─ csd_unlock(csd)                   // :581 释放发送者 wait
          ├─ CSD_TYPE_ASYNC: 同样执行（:605-612）
          ├─ CSD_TYPE_IRQ_WORK: irq_work_single()   // :613-614
          └─ CSD_TYPE_TTWU: sched_ttwu_pending()    // :625-628
```

### 3.2 `smp_call_function_many()` / `smp_call_function()` —— 广播

用途：在多个 CPU 上执行函数。与 3.1 的差异只在发送侧入队与 IPI 条数：

```
smp_call_function_many(mask, func, info, wait)       // kernel/smp.c:932
  └─ smp_call_function_many_cond(mask, func, info, scf_flags, NULL)  // :815
      ├─ cfd_data 每 CPU 预分配 csd（:848）
      ├─ 逐 CPU：csd_lock + 填 func/info + llist_add（:864-883）
      │     └─ 首个把队列由空变非空的 CPU 记入 cpumask_ipi（:879-883）
      ├─ nr_cpus==1 → send_call_function_single_ipi()   // :891-892
      │   nr_cpus>1  → send_call_function_ipi_mask()    // :893-894 一次广播 IPI
      │       └─ arch_send_call_function_ipi_mask()     // arm64 smp.c:852
      ├─ ★ SCF_RUN_LOCAL：mask 含本 CPU 时内联执行，无 IPI（:897-905，见 5.1-①）
      └─ wait → 逐 CPU csd_lock_wait（:907-914）
```

接收侧与 3.1 完全相同——**一次广播只产生一次中断投递**，flush 处理队列里全部 csd。

### 3.3 `irq_work_queue_on()` —— 远程 irq_work（测量载体之二）

用途：向指定 CPU 投递一个在硬中断上下文执行的 work（NMI 安全入队）。

```
发送栈：
irq_work_queue_on(work, cpu)                        // kernel/irq_work.c:137
  ├─ irq_work_claim()：work 已 PENDING → 返回 false，不发 IPI（:57-70，见 5.1-④）
  └─ cpu != self → __smp_call_single_queue(cpu, &work->node.llist)  // :173
      └─ 与 3.1 完全相同的队列与 IPI 发送路径（IPI_CALL_FUNC）
接收栈：与 3.1 相同的 flush，命中 CSD_TYPE_IRQ_WORK 分支
  └─ irq_work_single(work)                          // kernel/smp.c:613-614
      ├─ 清除 IRQ_WORK_PENDING（irq_work.c:211-213）
      └─ work->func(work)                           // :221 ← 回调执行点
```

### 3.4 `irq_work_queue()` —— 本 CPU 自中断

用途：向**本 CPU** 投递 work。arm64 上自 IPI 也是真实硬件操作：

```
irq_work_queue(work)                                // kernel/irq_work.c:116
  ├─ irq_work_claim()（同 3.3）
  └─ __irq_work_queue_local(work)                   // :88
      ├─ IRQ_WORK_LAZY → lazy_list，仅 tick 停时才 raise（:96-112，见 5.1-⑤）
      └─ HARD → raised_list，队列由空变非空 → irq_work_raise()  // :107-112
          └─ arch_irq_work_raise()                  // arm64 smp.c:863-866
              └─ smp_cross_call(cpumask_of(self), IPI_IRQ_WORK)
                  └─ 写 ICC_SGI1R_EL1 给自己 → 真实硬件自中断
接收栈：do_handle_IPI(IPI_IRQ_WORK)                 // arm64 smp.c:996-998
  └─ irq_work_run() → irq_work_run_list() → irq_work_single()  // irq_work.c:259
```

### 3.5 `smp_send_reschedule()` —— 调度器 IPI

用途：通知远程 CPU 重新调度。**无用户回调挂点，不能用于测量**，列出以完整性说明"高层已规避自 IPI"：

```
__resched_curr(rq, TIF_NEED_RESCHED)                // kernel/sched/core.c:1185
  ├─ ★ cpu == self → set_ti_thread_flag，直接返回，无 IPI（:1210-1215）
  ├─ set_nr_and_not_polling()（arm64 无 polling flag → 恒 true，:1077-1080）
  └─ smp_send_reschedule(cpu)
      └─ arch_smp_send_reschedule(cpu)              // arm64 smp.c:1156-1159
          └─ smp_cross_call(cpumask_of(cpu), IPI_RESCHEDULE)   // 无条件发送
接收栈：do_handle_IPI(IPI_RESCHEDULE) → scheduler_ipi()   // arm64 smp.c:971-973
```

### 3.6 `__ttwu_queue_wakelist()` —— 跨 CPU 唤醒（TTWU）

用途：唤醒任务时把入队成本转移给目标 CPU。同样复用 CALL_FUNC 通路：

```
try_to_wake_up() → ... → ttwu_queue_wakelist()       // kernel/sched/core.c:4056
  │   （sched_feat(TTWU_QUEUE) 开关，:4058）
  └─ __ttwu_queue_wakelist(p, cpu, wake_flags)      // :3950
      ├─ WRITE_ONCE(rq->ttwu_pending, 1)            // :3956
      └─ __smp_call_single_queue(cpu, &p->wake_entry.llist)  // :3958 复用 3.1 路径
接收栈：flush 第三阶段 CSD_TYPE_TTWU → sched_ttwu_pending()  // kernel/smp.c:625-628
```

### 3.7 停机与广播 tick（简表）

| IPI | 发送 | 接收 |
|---|---|---|
| `IPI_CPU_STOP`/`IPI_CPU_STOP_NMI` | `smp_send_stop()` → `smp_cross_call`（`arm64 smp.c:1237`/`:1254`） | `local_cpu_stop()`（`:979-986`） |
| `IPI_TIMER`（tick 广播） | `tick_broadcast()`（`:1172-1177`） | `tick_receive_broadcast()`（`:989-993`） |

### 3.8 场景汇总：对测量驱动的意义

| 场景 | 中断类型 | 用户回调挂点 | 可作测量载体 |
|---|---|---|---|
| 3.1/3.2 smp_call_function | CALL_FUNC | ✓（func 本身） | **是（驱动唯一载体）** |
| 3.3 irq_work 远程 | CALL_FUNC | ✓（work->func） | 否——复用同一 IPI_CALL_FUNC 通路，交叉验证价值有限，已从驱动移除 |
| 3.4 irq_work 自中断 | IRQ_WORK | ✓ | 否——csd 的 self 模式（内联捷径）已覆盖负向对照 |
| 3.5 resched | RESCHEDULE | ✗ | 否 |
| 3.6 TTWU | CALL_FUNC | ✗（内核内部） | 否 |
| 3.7 stop/tick | STOP/TIMER | ✗ | 否 |

三个场景（3.1/3.2/3.3）共享同一 `call_single_queue`——这是第三方并发入队风险的来源（5.1-②）。

---

## 4. 设计方案

### 4.1 模块形态与接口

单文件 GPL 内核模块 `ipi_lat.ko`，debugfs 接口：

```
debugfs/ipi_lat/
  mode        (csd | self，默认 csd)
  sender_cpu / target_cpu
  iterations / warmup / pace_ns
  busy        (默认 1：目标 CPU 上跑忙等 kthread，消除 idle 唤醒噪声)
  run         (写 1 触发一轮测量)
  results     (读：min/avg/max/p50/p95/p99 + 10ns 直方图 + ticks 原始值
               + discarded/total 丢弃率 + 环境快照)
```

发送侧是绑定 sender_cpu 的 kthread（`smp_call_function_single` 要求进程上下文 + IRQ 开启，`kernel/smp.c:695-704`）。借鉴 ipistorm 的模式（`references/ipistorm.c`）：`kthread_bind` 绑定、`cpulist` 语义的 CPU 选择、atomic 计数同步起跑。

### 4.2 两种测量模式

| 模式 | 载体 | 测什么 |
|---|---|---|
| `csd` | `smp_call_function_single(target, h, &iter, 1)` | 端到端 smp 调用时延（唯一测量模式） |
| `self` | 同上但 target==sender | **负向对照**：csd 走内联捷径（预期 <100ns 且中断计数不增），定量证明捷径存在且主测量已绕开 |

### 4.3 核心测量循环

```c
/* 发送侧 kthread（绑定 sender_cpu）；callfunc_desc 在加载时按 6.1 扫描获得 */
valid = 0;
while (valid < warmup + iterations) {
    if (!cpu_online(target))  return -ENXIO;          // 防热插拔
    iter.t0   = __arch_counter_get_cntpct();          // t0：先打点
    before    = irq_desc_kstat_cpu(callfunc_desc, target);
    smp_call_function_single(target, ipi_lat_handler, &iter, 1);  // wait=1
    after     = irq_desc_kstat_cpu(callfunc_desc, target);
    if (after == before) { discarded++; continue; }   // 捎带样本，重采
    if (++valid > warmup)
        stats_add(iter.t1 - iter.t0);                 // 仅 warmup 之后入库
}

/* 接收侧（target CPU 硬中断上下文，handler 首条指令） */
static void ipi_lat_handler(void *info)
{
    struct iter_ctx *iter = info;
    iter->t1 = __arch_counter_get_cntpct();           // t1
    iter->cpu = smp_processor_id();
    this_cpu_inc(ipi_handled);                        // handler 计数：验证用
    // csd_unlock 的 release 语义保证 wait 返回后 sender 读到完整数据
}
```

（原 irq_work/rtt 模式已删除：远程 irq_work 复用同一 `IPI_CALL_FUNC` 中断通路，交叉验证价值有限；RTT 因全局共享计数器而冗余。）

### 4.4 时钟与打点口径

- 时钟：`__arch_counter_get_cntpct()`（`arch/arm64/include/asm/arch_timer.h:179`，单条 `mrs cntpct_el0`，~2ns）+ `arch_timer_get_cntfrq()`（`:154`）换算 ns。所有 PE 共享同一 System Counter，**跨 CPU 直接相减**，无需 ping-pong 对称化。同时输出原始 tick 值。
- **M1（主口径）**：t0 = 发送调用前一刻，t1 = 回调首条指令。含：CSD 入队软件开销 + 硬件传播 + 接收入口栈（异常入口→irq_enter→IAR/EOI→分发）。
- **M2（硬件段上界）**：用 tracepoint 对（6.4）独立核算，排除 CSD 软件开销。
- t1 在接收侧打点 ⇒ 天然不受发送者 `csd_lock_wait` 阻塞时长影响（社区实测该等待 p99 可达 16ms，见 7.3）。

### 4.5 统计输出

全量样本存 per-CPU 数组（10 万 × 8B = 800KB），跑完聚合：min/avg/max/p50/p95/p99 + 直方图 + `discarded/total`。环境快照：cpuinfo 频率、governor、`saved_command_line` 中的 `threadirqs`/`isolcpus` 检测、GIC 型号（`gic_data` 类型）、sender/target 是否同 cluster。


### 4.6 一次 run 的完整数据流（生命周期）

（行号以 6.6 移植版 `/home/code/kernel/drivers/misc/ipi_lat.c` 为准，7.2 版结构一致）

```
用户: echo 1 > debugfs/run
  │
  ▼ ipi_lat_run_write()                          // :529（全程持 run_mutex，run 天然串行化）
  │    ├─ ipi_lat_prepare_run()                  // :390  参数校验；self 模式 target 强制 = sender
  │    ├─ kmalloc 样本数组 iterations×8B          // 每次 run 独立分配，结束即释放
  │    ├─ kthread_create + kthread_bind(sender_cpu) + wake_up_process
  │    └─ wait_for_completion(&run.done)         // 阻塞用户进程，直到测量完成
  │
  ▼ ipi_lat_run_thread()                         // :336（运行于 sender_cpu，进程上下文）
  │    ├─ ipi_lat_desc(IPI_CALL_FUNC)            // 取判决用的 irq_desc（加载时发现，§6.1）
  │    ├─ ipi_lat_busy_start(target)             // :264  busy=1 时在目标 CPU 上放忙等线程
  │    ├─ kstat_start / handled_start            // 记录两个计数器的起始值
  │    ├─ switch (mode)：
  │    │    CSD  → ipi_lat_run_csd()             // :277  主测量循环（含逐轮判决，见 4.3）
  │    │    SELF → ipi_lat_run_self()            // :315  负向对照循环（无判决，故意走捷径）
  │    ├─ kstat_end / handled_end                // 结束值 → 生成两个断言数据
  │    ├─ kthread_stop(busy)                     // 测量窗口结束才停——整个窗口内目标 CPU 非 idle
  │    └─ complete(&run.done)
  │
  ▼ 回到 ipi_lat_run_write → ipi_lat_build_results()  // :421
  │    ├─ sort(样本) → min/avg/max/p50/p95/p99（tick→ns 换算）
  │    ├─ 断言文本：handled == valid+discarded ? OK : MISMATCH
  │    │            kstat_delta >= valid ? OK : FAIL
  │    │            threadirqs 检测
  │    └─ 结果写入 results 缓冲（用户 cat results 读到）
```

流程要点：

- **跨 CPU 消息**：每轮迭代的 `ipi_lat_iter`（t0/t1/cpu）经 csd 的 info 指针在两核间传递；`wait=1` 的 `csd_lock_wait`（release/acquire 语义）保证发送者等待返回后读到接收者写入的完整数据，无需额外屏障。
- **接收侧极简**：handler（:244）只做三件事——首条指令打 t1、记 `smp_processor_id()`、`this_cpu_inc(handled)`；全部统计在发送侧进程上下文完成，无并发、无锁。
- **时序约束**：busy 线程先于测量循环启动、晚于它停止（防 6.6 轮询旁路的第一层）；run 线程与 run_write 之间用 completion 交接。
- **异常路径**：目标 CPU 中途下线 → 测量循环置 `r->err = -ENXIO` → run_write 返回错误且**不更新 results**（保留上一次的有效结果，避免半截数据被误读）。

## 5. 设计方案如何保证不存在快捷路径、确定是硬件传播时延

### 5.1 六类快捷路径 × 设计对策

| # | 快捷路径（代码证据） | 设计对策 |
|---|---|---|
| ① | **目标是本 CPU → 内联执行，零中断**：`generic_exec_single()` `kernel/smp.c:461-477`（`local_irq_save` 后直接 `csd_do_func`）；广播的 `SCF_RUN_LOCAL` 同理 `:897-905` | 强制 `target != sender`（加载即校验）；保留 `self` 模式作负向对照（6.3） |
| ② | **队列非空 → 不发新 IPI（合并投递）**：`llist_add` 判空 `kernel/smp.c:446`。`wait=1` 只能排空自己的上一轮；TTWU/RCU/其他驱动（3.6、3.8）随时可先占队列 | 串行迭代 + **逐轮 kstat 判决**（5.2）——阻止不了捎带，但保证捎带样本不入库 |
| ③ | **轮询 idle 免 IPI**：`call_function_single_prep_ipi()` `sched/core.c:3934` 对 TIF_POLLING 目标置位返回 false 不发 IPI；目标在 `do_idle` 轮询中直接 flush 队列（6.6: `kernel/sched/idle.c:370`），全程零中断 | **7.2 树：arm64 无 `TIF_POLLING_NRFLAG`**（`include/linux/sched/idle.h:20` 守卫；回退 `set_nr_if_polling()=false` `sched/core.c:1085-1088`）→ 恒发真实 IPI。**6.6 树：arm64 有 `TIF_POLLING_NRFLAG`**（`arch/arm64/include/asm/thread_info.h:79`），此旁路**真实存在**——两层防护：① `busy=1`（默认）让目标 CPU 永不进入 idle 轮询，旁路无从触发；② 即使被旁路，逐轮 kstat 判决见 `after==before`（无投递）→ 样本丢弃，绝不入库。旁证：修正该语义的 `TIF_NOTIFY_IPI` 系列未合入本树 |
| ④ | **接收侧关中断/PMR/目标下线**（不绕过中断，但使时延失真或失败） | 目标忙等且 IRQ 开启；每轮 `cpu_online`；统计分布如实呈现 |

发送侧另经审计确认**不存在**其他旁路：`__ipi_send_single`/`__ipi_send_mask`（`kernel/irq/ipi.c:227-303`）无 self 判断；`gic_ipi_send_mask` 对 mask 内每个 CPU 包括自己都写寄存器（`irq-gic-v3.c:1392-1398`）；arm64 连自 IPI 都走真实硬件（3.4）。

### 5.2 逐轮 kstat 判决——保证"窗口内有真实投递"的核心机制

基石：`handle_percpu_devid_irq()` 在跑回调**之前**必执行 `__kstat_incr_irqs_this_cpu()`（`kernel/irq/chip.c:916`）。每轮对比目标 CPU 的 `IPI_CALL_FUNC` 计数（读取顺序：`before` 必须在 `t0` **之后**读取，保证任何 `after > before` 都对应一次发生在 t0 之后的投递）：

| 情况 | 发生了什么 | 计数 | 处置 |
|---|---|---|---|
| A 干净发送 | 我们的 `llist_add` 见队列空 → 写 SGI1R/CDPEND → 投递（计数++）→ flush 跑回调 | after > before | ✓ 保留 |
| B1 捎带-已投递 | 第三方 IPI 已投递（计数已++）但目标正关中断、flush 被推迟；我们入队不发 IPI，回调等目标开中断才跑 | after == before | ✗ 丢弃（虚高样本） |
| B2 捎带-在途 | 第三方 IPI 尚在投递途中；它投递（计数++）后 flush 跑我们回调 | after > before | ✓ 保留——t0 后确有真实投递，时延无偏 |
| C 抢先投递 | 我们发了 IPI，第三方 IPI 先到并顺手 flush 掉我们的回调 | after > before | ✓ 保留——窗口内真实投递，时延无偏 |

丢弃只发生在"无投递"或"被旧中断捎带"两种情形；保留的每个样本都满足：**t0 之后、回调执行之前，发生了一次真实的 SGI/LPI 投递**。

### 5.3 闭环结论与口径边界

设计闭环：`target ≠ sender`（①）+ 串行化 + kstat 判决（②）+ arm64 无轮询旁路（③）+ HARD work 与 claim 检查（④⑤）+ 目标在线且中断开启（⑥）⇒ **每个入库样本的 delta 都包含一段真实的硬件中断传播**（发送侧 `ICC_SGI1R_EL1`/`CDPEND` 写 → GIC 投递 → 接收侧异常入口）。

诚实边界：delta = CSD 入队软件 + 硬件传播 + 接收入口栈，是**软件可见时延**而非纯硬件传播时延；纯硬件段可用 M2（6.4）界定，负向对照（6.3）给出软件捷径的底线值。两者一减，可近似分离硬件段。

---

## 6. 验证手段

### 6.1 模块内逐轮判决（kstat）

即 5.2 机制本身：`after > before` 才入库，`discarded` 单独计数并随 results 输出。丢弃率 >5% 提示系统不够静默，建议 `isolcpus`/`nohz_full`（只影响重采成本，不影响正确性）。

模块加载时扫描一次 IPI 中断线（`irq_to_desc` 为 `EXPORT_SYMBOL_GPL`，`irqdesc.c:417`）：

```c
/* 扫描 desc->action->name == "IPI" 且 chip 具备 ipi_send_mask/ipi_send_single 的 irq
 * SGI 模式: 恰好 8 个连续 virq → base+1 = IPI_CALL_FUNC, base+5 = IPI_IRQ_WORK
 * GICv5 模式: 8*nr_cpu_ids 个 → 按 irq_data 有效亲和选出 target 的 base+cpu*8+1   */
```

扫描失败 → 模块**拒绝加载**（宁可不测，不测假数据）。

### 6.2 `/proc/interrupts` 汇总核对（使用规程）

IPI 行由 `arch_show_interrupts()` 打印（`arm64/kernel/smp.c:837-850`）：`IPI1 Function call interrupts`、`IPI5 IRQ work interrupts`。跑前跑后目标 CPU 列增量 **≥ 有效样本数**（第三方 IPI 也会 ++，故为 ≥；有效样本数由模块输出）。增量 < 有效样本数 ⇒ 判决机制失效（不应发生），立即停止使用。

### 6.3 负向对照（self 模式）

- csd self：走 `generic_exec_single` 内联路径（`smp.c:461-477`），预期 <100ns 且 `/proc/interrupts` 计数不增——定量证明捷径存在，并反证远程 µs 级数据确实经过了中断。
- irq_work self：走 `IPI_IRQ_WORK` 真实自 SGI（计数 +1），给出自 SGI 往返基线。

### 6.4 ftrace tracepoint 独立核算（M2）

发送侧 `ipi:ipi_raise`（`smp.c:1032`）/`ipi:ipi_send_cpu`（`kernel/smp.c:118`）→ 接收侧 `ipi:ipi_entry`（`smp.c:968`）的时间差，界定"寄存器写 → handler 入口"段（排除 CSD 软件，含接收入口栈）。arm64 的 `sched_clock` 源自全局 arch timer（`arm_arch_timer.c:951`），跨 CPU 时间戳直接可比。注意：模块无法注册该 tracepoint 探针（未 `EXPORT_TRACEPOINT_SYMBOL`），只能经 ftrace 用户态工具离线完成；tracepoint 常开有 ~百 ns 开销，仅用于少量验证轮次。

### 6.5 环境快照与可复现性

results 输出环境快照（4.5）——回应社区对 IPI 基准可复现性的要求（7.2）：测试方法、硬件、频率、拓扑、中断控制器型号缺一不可。

---

## 7. 先例与要点（压缩）

- **ipistorm**（Anton Blanchard，源码已存 `references/ipistorm.c`）：内核社区 IPI 压测事实标准（每 CPU `kthread_bind` 线程循环 `smp_call_function_single(wait=1)`，modified 版 `total/numipi` 得均值）。**无逐样本打点、无"中断确实发出"验证、时延口径受发送侧 wait 阻塞污染**——本设计补齐这三点；其 kthread_bind/同步起跑/cpulist 模式被沿用（4.1）。
- **`b2a02fc43a1f` 回溯案例**（K Prateek Nayak，2024-01）：轮询旁路在 x86 上把 ipistorm 数据扭曲了 30%+，`TIF_NOTIFY_IPI` 系列修复（EPYC 上 -81%）；该系列未合入本树。**快捷路径污染测量有实测事故背书**——5.1-③ 的旁证。
- **KVM vSGI RFC**（Xu Zhao，2023-08）：arm64 虚拟机 SGI 基准（`vgic_v3_dispatch_sgi` ~1.5µs@4vCPU）；Marc Zyngier 要求"声明测试、硬件、复现方法"——6.5 的由来。
- **csd_lock_wait 可抢占化系列**（Chuyi Zhou，2026-02）：发送侧不可抢占等待 p99 达 16ms（TLB shootdown）——佐证"发送者往返总时长 ≠ IPI 时延"，本设计 t1 打点在接收侧（4.4）。
- **DragonFly** `debug.ipiq.latency_test` + `ipitest`：内核内建 IPI 时延测试 + sysctl 驱动的形态参考 `[EXTERNAL: 未抓取源码]`。

---

## 8. 使用方法（含 Linux 6.6 移植说明）

### 8.1 代码位置与两个内核版本的差异

| | 本树（Linux 7.2，`drivers/misc/ipi_lat.c`） | 移植版（Linux 6.6，`/home/code/kernel/drivers/misc/ipi_lat.c`） |
|---|---|---|
| IPI 承载 | GICv3 SGI 共享 / GICv5 per-CPU LPI 双模式 | 仅 GICv2/v3 SGI 共享模式（6.6 无 GICv5） |
| IPI 注册 | `set_smp_ipi_range_percpu`，8 条（LPI 模式 8×nr_cpu_ids 条） | `set_smp_ipi_range`（`arch/arm64/kernel/smp.c:1036`），`nr_ipi=min(8, NR_IPI)=7` 条（enum 序 `smp.c:74-85`，CALL_FUNC=1 / IRQ_WORK=5 两版一致） |
| **轮询 idle 旁路** | **不存在**（arm64 无 `TIF_POLLING_NRFLAG`） | **存在**（`arch/arm64/include/asm/thread_info.h:79`）→ `call_function_single_prep_ipi()`（`kernel/sched/core.c:3776`）可对 idle 目标免发 IPI |
| desc 发现 | 8 或 8×nr_cpu_ids 两种计数 | 连续 "IPI" 命名 irq 串，≥7 条即可（兼容 SMT_QOS 的 8 条） |

6.6 上旁路依然被两层防护覆盖：**busy=1（默认）让目标 CPU 永不 idle，旁路无从触发**；即使 busy=0 被旁路，逐轮 kstat 判决也会把"无投递"的样本全部丢弃（`after==before`），不产生虚假数据。

### 8.2 构建（6.6，arm64 交叉环境）

```bash
# 树内已接线：drivers/misc/Kconfig（config IPI_LAT）与 drivers/misc/Makefile
scripts/config -e DEBUG_FS -m IPI_LAT          # 在既有 arm64 config 上开启
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- olddefconfig
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc) modules
# 产物: drivers/misc/ipi_lat.ko
```

### 8.3 上板使用（推荐顺序）

```bash
insmod drivers/misc/ipi_lat.ko                          # dmesg 应见 "7 contiguous IPI irqs (SGI mode), chip GICv3"

D=/sys/kernel/debug/ipi_lat
echo 0 > $D/sender_cpu; echo 1 > $D/target_cpu            # 选一对 CPU（先同 cluster）

# ① 负向对照（必须先做）：内联捷径，预期 <100ns 且中断计数不增
echo self > $D/mode; echo 1 > $D/run; cat $D/results

# ② 主测量
echo csd > $D/mode; echo 1 > $D/run; cat $D/results
```

### 8.4 汇总核对（使用规程，跑前跑后各一次）

```bash
grep "Function call interrupts" /proc/interrupts    # 目标 CPU 列
# 判据：增量 >= results 中的 valid（第三方 IPI 也会 ++，故为 >=；< valid 说明判决失效，立即停用）
# self 模式判据：增量 ≈ 0（内联路径不产生中断）
```

### 8.5 results 解读要点

```
mode=csd sender=0 target=1
chip=GICv3 iterations=100000 warmup=1000 pace_ns=0
cntfrq=50000000 Hz ticks=... valid=100000 discarded=12 (0.01%)   ← 丢弃率 <5% 为健康
handled=100012 expect=100012 OK       ← handler 计数 == valid+discarded
kstat_delta=100015 expect>=100000 OK  ← 目标 IPI 计数增量 >= valid
min=812ns avg=845ns max=1034ns p50=831ns p95=870ns p99=903ns
hist(10ns/bin): ...
```

- 口径：t1-t0 = CSD 入队软件 + 硬件传播 + 接收入口栈（软件可见时延）；纯硬件段用 ftrace `ipi_raise`→`ipi_entry` 核算
- `self` 模式若 >100ns 或 `handled`/计数异常增长，说明环境有干扰，先查 CPU 隔离
- 出现 `WARNING: threadirqs` 或 `MISMATCH`/`FAIL` 即数据不可信

## 9. 参考链接

- [ipistorm 源码（Anton Blanchard）](https://github.com/antonblanchard/ipistorm)
- [LKML: [PATCH] sched/fair: Skip newidle_balance() when an idle CPU is woken up to process an IPI（b2a02fc43a1f 回溯案例）](https://lkml.org/lkml/2024/1/19/130)
- [patchew: [RFC PATCH 00/14] Introducing TIF_NOTIFY_IPI flag](https://patchew.org/linux/20240220171457.703-1-kprateek.nayak@amd.com/)
- [LKML archive: [RFC] KVM: arm/arm64: optimize vSGI injection performance](https://lkml.indiana.edu/2308.2/03337.html)
- [LKML: [PATCH 00/11] Allow preemption during IPI completion waiting to improve real-time performance（Chuyi Zhou, 2026-02）](https://lkml.org/lkml/2026/2/3/836)
- [DragonFly BSD kernel list: IPI Benchmark Data?（debug.ipiq.latency_test / ipitest）](https://lists.dragonflybsd.org/pipermail/kernel/2016-August/255513.html)
- [linux-stable: kernel/irq 历史提交（ipi-mux 相关）](https://git.zx2c4.com/linux-stable/log/kernel/irq?id=befaa609f4c784f505c02ea3ff036adf4f4aa814&ofs=50&showmsg=1)
- [Cregit: Linux 6.14 kernel/irq/ipi-mux.c 源码](https://cregit.linuxsources.org/code/6.14/kernel/irq/ipi-mux.c.html)
- [patchew: [PATCH v6 3/7] genirq: Add mechanism to multiplex a single HW IPI（RISC-V IPI mux 起源）](https://patchew.org/linux/20220418105305.1196665-1-apatel@ventanamicro.com/20220418105305.1196665-4-apatel@ventanamicro.com/)
- [LKML: Marc Zyngier 论 GIC 为何不接入 IPI mux（SGI 已天然跨 CPU）](https://marc.info/?l=linux-kernel&m=165840375002191&q=raw)
- [lore: [PATCH v5] Generic IPI sending tracepoint（ipi_send_cpu/ipi_send_cpumask 合入系列）](https://lore-kernel.gnuweeb.org/linux-hexagon/20230307143558.294354-6-vschneid@redhat.com/)
- [patchew: [RFC PATCH 5/5] treewide: Rename and trace arch-definitions of smp_send_reschedule()](https://patchew.org/linux/20221007154145.1877054-1-vschneid@redhat.com/20221007154533.1878285-5-vschneid@redhat.com/)
- [LKML archive: [RFC PATCH v2 2/8] trace: Add trace_ipi_send_cpumask()](https://lkml.rescloud.iu.edu/2211.0/date.html)

---

*调研基准：本树 `master`（v7.2-rc6 附近，2026-08-21）。覆盖文件：`arch/arm64/kernel/smp.c`、`arch/arm64/include/asm/smp.h`、`arch/arm64/kernel/entry-common.c`、`arch/arm64/include/asm/arch_timer.h`、`drivers/irqchip/irq-gic-v3.c`、`drivers/irqchip/irq-gic-v5.c`、`include/linux/irqchip/arm-gic-v5.h`、`kernel/smp.c`、`kernel/sched/core.c`、`kernel/irq_work.c`、`kernel/irq/ipi.c`、`kernel/irq/chip.c`、`kernel/irq/irqdesc.c`、`kernel/irq/handle.c`、`include/trace/events/ipi.h`、`include/linux/sched/idle.h`、`include/linux/irqdesc.h`、`drivers/clocksource/arm_arch_timer.c`；先例源码：`references/ipistorm.c`。6.6 移植基准：`/home/code/kernel`（v6.6.0）的 `arch/arm64/kernel/smp.c`（enum 74-85、`set_smp_ipi_range` 1036）、`arch/arm64/include/asm/thread_info.h`（`TIF_POLLING_NRFLAG` 79）、`drivers/irqchip/irq-gic-v3.c`（`gic_smp_init` 1749）、`kernel/smp.c`（`generic_exec_single` 390）、`kernel/sched/core.c`（`call_function_single_prep_ipi` 3776）、`kernel/irq/chip.c`（kstat 902）、`include/linux/irqdesc.h`（`irq_desc_kstat_cpu` 134）。*
