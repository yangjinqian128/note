# FEAT_TLBID 指令注入测试驱动（tlbid_inject）

> 目标：在内核态注入一条带 TLBID 域的 TLBI 指令，用 insmod 参数/退出码 + dmesg 观察硬件对 FEAT_TLBID 指令的真实行为（UNDEFINED / 静默执行 / 域生效）。
> 核心约束：不依赖新 binutils、不要求硬件已实现 FEAT_TLBID（无 TLBID 硬件同样能验证 CONSTRAINED UNPREDICTABLE 分支）、**异常即观察**——oops 不是 bug，是实验数据。
> 代码：`drivers/misc/tlbid_inject.c`（122 行，独立测试模块，与 `ipi_lat.c` 同定位）；补丁：`tlbid-inject.patch`。

---

## 1. 定位与动机

FEAT_TLBID（Armv9.7-A，2025，"TLBI Domains"）给 TLBI/TLBIP/PLBI 广播类指令增加 16 位 TLBID 域字段，把广播失效从"整个 shareability domain"缩小到"域内 PE 子集"。它是一个很新的特性，硬件实现状态未知，内核/KVM 支持也尚未上游化——所以在动手写内核/KVM 代码之前，需要先在真机上回答：

1. 这台机器实现 FEAT_TLBID 了吗？（TLBID 域字段是生效还是被忽略/拒绝）
2. 没实现时，带域的指令是 **UNDEFINED** 还是 **静默按全广播执行**（CONSTRAINED UNPREDICTABLE 的两个分支）？
3. 域值不同（0~0xFFFF）时行为如何分布？

TLBI 是特权指令（EL1+），用户态直接执行只会得到 EL0 的 UNDEFINED，测不到硬件真实行为。而**内核模块的 init 函数天然跑在内核态**——把"执行指令"藏进 insmod 的正常加载流程，用模块参数做输入、退出码和 dmesg 做输出，不需要任何 hack。

**注入的指令只有一条：`tlbi aside1is`**——这正是 OpenEuler OLK-6.6 补丁系列（`[PATCH 0/6] arm64: tlbflush: Optimize flush_tlb_mm() by using TLBID`）中 host 发 TLBID 的唯一形态，驱动 1:1 复刻生产路径（选型依据见 §4.2）。

## 2. 原理总览

```
用户态                        内核态 (特权级 EL1 / VHE-EL2)
──────                        ──────────────────────────────
insmod tlbid_inject.ko
  domain=5
      │ finit_module 系统调用
      ▼
                        模块加载器载入 .ko
                              │ module_param 机制解析命令行参数
                              │   → asid / domain 两个变量 (tlbid_inject.c:60,64)
                              ▼
                        tlbid_inject_init() (tlbid_inject.c:67)
                              │
                              ├─ preempt_disable()          (c:78) 锁核,
                              ├─ 读 CurrentEL 得 el         (c:80) 记录异常级
                              ├─ ASID 解析                  (c:88-89) 缺省取 insmod 自己的
                              ├─ 组装操作数                 (c:92-93) (ASID<<48)|domain
                              ├─ 打印注入参数               (c:95)
                              ├─ dsb(ishst)                 (c:98) 与内核真实 flush 一致
                              ├─ tlbi aside1is, xN          (c:99) ★ 注入点
                              ├─ dsb(ish); isb()            (c:100-101)
                              │
                ┌─────────────┴──────────────────────────────────┐
                │ 路径 A: 硬件正常执行                            │ 路径 B: 硬件 UNDEFINED
                │   CPU 继续执行下一条                           │   CPU 触发同步异常
                │   └─► pr_info "tlbid: OK ..." (c:107)         │   └─► el1_undef() 入口
                │   └─► return -EAGAIN (c:117)                  │       (entry-common.c:357)
                │   └─► insmod 报错但模块自动卸载               │   └─► do_el1_undef()
                │   └─► 无需 rmmod, 可立即换参数重试            │
                │                                               │       (traps.c:476)
                │                                               │   └─► die("Oops - Undefined
                │                                               │        instruction")
                │                                               │       (traps.c:487)
                │                                               │       ├─ dmesg 打 oops 日志
                │                                               │       │  (PC=注入指令地址,
                │                                               │       │   ESR=异常编码)
                │                                               │       ├─ 杀掉 insmod 进程
                │                                               │       └─ insmod 失败
                └───────────────────────────────────────────────┘
```

**oops 为什么是安全的观察方式**：`die()` 默认（`panic_on_oops=0`）只 oops 不宕机——杀掉触发它的 insmod 进程，内核继续跑。oops 日志里的 **PC** 精确指向注入指令的地址，**ESR** 的 EC 字段说明是 UNDEFINED（EC=0x00），这就是"该编码在此硬件上不被接受"的铁证。

## 3. 模块结构

### 3.1 参数（输入接口）

| 参数 | 类型 | 含义 | 定义 |
|---|---|---|---|
| `asid` | u64 | ASID 值，编码进操作数 [63:48]。**缺省 = 自动使用 insmod 进程自己的 ASID**（`ASID(current->mm)`，零副作用）；显式传值则用手动 ASID | `tlbid_inject.c:60` |
| `domain` | u64 | 16 位 TLBID 域值，编码进操作数 [15:0] | `tlbid_inject.c:64` |

`module_param` 是内核给"用户态向模块传参"的标准通道：insmod 命令行的 `key=value` 由内核模块加载器解析后写入模块内变量，在 `module_init` 运行前完成赋值。

### 3.2 执行体（tlbid_inject.c:67-118）

```
tlbid_inject_init()
  ├─ preempt_disable()                 锁核：保证打印的 cpu 就是执行核
  ├─ cpu = smp_processor_id()          记录执行核
  ├─ el = read_sysreg(CurrentEL) >> 2  记录异常级 (EL1=1 / EL2=2)
  ├─ own_asid 时: asid = ASID(current->mm)
  ├─ operand = (asid<<48) | domain     操作数组装 (OpenEuler __TLBI_DOMAIN 编码)
  ├─ dsb(ishst)                        屏障与 flush_tlb_mm 一致
  ├─ tlbi aside1is, xN                 ★ 真正执行指令 (c:99)
  ├─ dsb(ish); isb()
  └─ OK 打印 + return -EAGAIN          故意失败 → 模块自动卸载 → 无需 rmmod 即可重试
```

**el 字段是设计点**：`read_sysreg(CurrentEL)` 在 arm64 是宏 stringify（`mrs x, CurrentEL`），gas 原生接受 CurrentEL 名字（树内 `virt.h:137` 同款用法）。同一份代码在非 VHE 上 `el=1`、VHE host 上 `el=2`、guest 里 `el=1`——spec 说 TLBID 翻译只作用于 EL1 发出的指令，所以这个对照本身就是实验数据。

### 3.3 ASID 选择与副作用（缺省零影响设计）

`tlbid_inject.c:69,88-89`：`asid` 参数的缺省值 `(u64)-1` 是哨兵，init 里解析为 insmod 进程自己的 ASID：

```c
bool own_asid = (asid == (u64)-1);
...
if (own_asid)
	asid = ASID(current->mm);   /* asm/mmu.h:56 */
```

**为什么这是零影响**：insmod 进程在核上运行，它的 ASID 一定已分配（正在运行 = 已写入 TTBR0）且此刻**全局唯一属于这个 mm**。注入 `aside1is` 后 IS 广播发到全机，接收核只清"打这个 ASID 标签"的条目——而这样的条目只有 insmod 进程自己的。其他进程（ASID 互不相同）、内核（ASID 0 保留）一个都不碰。

| ASID 方案 | 副作用 |
|---|---|
| `asid=0`（显式传） | IS 广播清全机内核条目（KPTI 下内核映射以 ASID 0 打非全局标签）→ 抖动 |
| 手动固定 `asid=N` | N 随时可能被真进程占用 → 清它一轮 TLB（安全但非零影响） |
| **缺省：`ASID(current->mm)`** | **零影响**：只清 insmod 自己的条目 |

`asid` 参数缺省为 `(u64)-1`，`param_set_ullong` 不接收负数字面量，哨兵无法从命令行误设。打印行里带 `(own)` 标记（`tlbid_inject.c:95-96`），实验记录可对上号。guest 里同样成立：ASID 是 guest 自己的空间，外加 VMID 隔离，影响天然困在本 VM。

## 4. 编码原理

### 4.1 操作数布局（OpenEuler `__TLBI_DOMAIN` 同款）

`tlbi aside1is` 的 64 位操作数布局：

```
63             48 47              16 15               0
┌────────────────┬──────────────────┬─────────────────┐
│     ASID       │     (RES0)       │  TLBID domain   │
└────────────────┴──────────────────┴─────────────────┘
        ↑                                    ↑
  asid << 48                          domain & 0xFFFF
```

模块里（`tlbid_inject.c:92-93`）：

```c
operand = ((asid << 48) & GENMASK_ULL(63, 48)) |
	  (domain & GENMASK_ULL(15, 0));
asm volatile("tlbi aside1is, %0" :: "r"(operand));
```

指令语义：让"属于该 TLBID 域"的核，把 TLB 里该 ASID 的条目失效。aside1is 操作数是架构标准形式，任何版本 binutils 都接受，无需特殊处理。

### 4.2 为什么只有 aside1is（以 host 代码为准的选型）

OpenEuler host 发 TLBID 时**只有这一种形态**（补丁原文）：

```c
domain = trans_cpumask_to_domain(mm_cpumask(mm));   /* cpumask → 域值 */
asid = __TLBI_DOMAIN(ASID(mm), domain);             /* (ASID<<48)|domain[15:0] */
__tlbi(aside1is, asid);                             /* 单条 aside1is 带域广播 */
```

全树统计（`grep __tlbi( arch/arm64/`，含 KVM）内核实际发出的 TLBI 变体：

| 变体 | 全树次数 | 用途 |
|---|---|---|
| `vmalle1is` | 6 | `flush_tlb_all`、KVM stage-1 全刷 |
| `vmalle1` | 6 | `local_flush_tlb_all`、`__kvm_flush_cpu_context`（本地） |
| `aside1is` | 3 | **`flush_tlb_mm` 主力，OpenEuler TLBID 的载体** |
| `vmalls12e1is` / `ipas2e1is` | KVM 专属 | VMID/S2，需 TGE 翻转上下文才有意义 |
| **OS 变体（aside1os/vmalle1os）** | **0** | 内核从不发 |

砍掉的其他候选：
- `aside1os` / `vmalle1os`：内核零使用，测了也不是 host 真实路径；
- `aside1` / `vmalle1`：NS 本地变体**不在域指令集里**（TLBID 只对 IS/OS 广播有意义），对 TLBID 测试无价值；
- `vmalle1is`：虽然也是 host 真实指令（`flush_tlb_all`/KVM stage-1 全刷），但它是**另一种编码机制**（可选操作数指令，Rt≠0b11111 的经典 CONSTRAINED UNPREDICTABLE 案例，且老 binutils 只接受 xzr 操作数需 `.inst` 绕行）。当前目标是验证 host TLBID 生产路径（aside1is），vmalle1is 留到 KVM stage-1 全刷域化（L3 阶段）时再加回，约 10 行。

驱动与 OpenEuler host 实现的逐项对照：

| 项 | OpenEuler host | 本驱动 |
|---|---|---|
| 指令 | `tlbi aside1is`（单条带域广播） | 同 ✓ |
| 操作数 | `(ASID<<48) \| domain&0xFFFF` | 逐位等价 ✓ |
| 屏障 | 标准 `dsb(ishst)/dsb(ish)/isb` | 同 ✓ |
| `__tlbi_user` 双发 | KPTI 下再发一条带 `USER_ASID_FLAG` 的 | 驱动未做——注入验证仅观测单条指令行为，这是唯一差异点 |

## 5. 异常路径：oops 作为观察仪器

内核态 UNDEFINED 的完整路径：

```
CPU 执行 tlbi (UNDEFINED)
  └─► 同步异常, 进入异常向量 (EL1h)
      └─► el1_undef()          arch/arm64/kernel/entry-common.c:357
          └─► do_el1_undef()   arch/arm64/kernel/traps.c:476
              ├─ aarch64_insn_read(regs->pc)  读出出错指令
              ├─ try_emulate_el1_ssbs()        (仅 SSBS 仿真, 与 TLBI 无关)
              └─► die("Oops - Undefined instruction", regs, esr)  traps.c:487
                  └─► oops 日志 + do_exit 杀当前进程 (insmod)
```

**关键设计判断**：v7.3 的 `do_el1_undef` **不查 `__ex_table` fixup**（fixup_exception 只存在于内存 fault 等路径），所以内核态 UNDEFINED 无法用 fixup 干净捕获——驱动不捕获，oops 本身就是观察结果。这与"有问题时挂死也 ok"的验证需求一致，同时砍掉了所有复杂模式。

oops 之后的模块状态：init 没跑完，模块留在"加载中"状态，`rmmod -f tlbid_inject` 强制清理，**不用重启**即可换参数继续下一轮。

## 6. 使用方式

```sh
# 构建（out-of-tree，模块只依赖稳定 API，v7.3 / OLK-6.6 树均可）
echo 'obj-m := tlbid_inject.o' > /tmp/tlbid/Makefile
cp drivers/misc/tlbid_inject.c /tmp/tlbid/
make -C /home/code/atomgit/linux M=/tmp/tlbid modules

# 注入（taskset 锁执行核；不指定就用 insmod 所在核）
# asid 缺省 = 自动用 insmod 自己的 ASID（零副作用）；显式传值则手动指定
taskset -c 3 insmod /tmp/tlbid/tlbid_inject.ko domain=5
dmesg | tail -2
# 注意: 执行成功时 insmod 也会报错 (EAGAIN, 故意失败让模块自动卸载),
# 这是预期行为——结果只看 dmesg: "OK" 行 = 执行成功, oops 日志 = UNDEFINED

# 重复验证循环（扫域空间, 全程无需 rmmod）
for d in $(seq 0 31); do
	insmod tlbid_inject.ko domain=$d 2>/dev/null   # 每次都"失败": 成功=EAGAIN, 异常=oops
	dmesg --since '1 second ago' | grep -q "OK:" \
		&& echo "domain=$d: executed" \
		|| echo "domain=$d: oops"
	rmmod -f tlbid_inject 2>/dev/null              # 仅 oops 后清残留, 成功路径无模块
done
```

dmesg 解读：

```
tlbid: inject asid=0x1 (own) domain=0x5 (cpu=3 el=2)   ← 执行前参数记录, (own)=自动 ASID
tlbid: OK: aside1is executed, no exception (cpu=3 el=2) ← 成功标识
──────────────────────── 或 ────────────────────────
Internal error: Oops - Undefined instruction: 00000000... ← UNDEFINED 分支
pc : tlbid_inject_init+0x30/0x7c [tlbid_inject]           ← 出错指令地址
```

## 7. 能回答哪些问题（验证矩阵）

| 实验 | 命令 | 回答的问题 |
|---|---|---|
| 域扫描 | `domain=0..31` 循环 | 该硬件对 TLBID 域值的接受度分布 |
| CONSTRAINED UNPREDICTABLE | 无 TLBID 硬件上 `domain≠0` | oops = UNDEFINED 分支；OK = 静默执行分支 |
| 有 TLBID 硬件 | 任意 `domain` | OK 且按域广播（是否真按域需探测实验，见下） |
| EL 对照 | 同命令在 VHE host / 非 VHE / guest 各跑一遍 | 翻译只对 EL1 生效的假设（看 el 字段） |
| ASID 副作用对照 | `asid=0` vs 缺省 | 内核条目被清（抖动）vs 零影响 |

**盲区**：CONSTRAINED UNPREDICTABLE 的"静默执行"分支与"真域生效"从异常上看一模一样（都打印 OK）——区分它们需要 TLB 时延探测实验（各核填充探测页 TLB → 注入域失效 → 各核重访问测时延，被刷的核变慢），是后续 `probe` 命令的扩展方向。详见 [feat-tlbid-kvm-design.md](feat-tlbid-kvm-design.md) §6.3 的 G1/G2 未决项。

## 8. 限制与注意

- **IS 广播真刷 TLB**：有域硬件按域广播、无域硬件全广播，都会对其他核产生 TLB 抖动——但缺省 ASID（insmod 自己的）保证**被清的只有注入进程自己的条目**，其他进程与内核零影响；显式 `asid=0` 才会碰内核条目（KPTI 下全机抖动）。测试机上跑。
- **模块残留**：成功路径故意 `return -EAGAIN` 自动卸载，**无需 rmmod 即可连续重试**；仅 oops 后模块楔在"加载中"状态，需 `rmmod -f` 清理。
- **G3 已按 OpenEuler 生产代码定案**：aside1is 的 TLBID 字段位置 = 操作数 [15:0]（`__TLBI_DOMAIN` 宏的 `GENMASK_ULL(15, 0)` 即产品级答案）。[31:16] 的说法出自 PLBI 指令，与 TLBI 无关。
- 模块刻意不做 fixup / expect 翻转 / 多 op 等模式，保持"成功打印标识、异常即 oops"的最小语义。

---

## 参考链接

- [FEAT_TLBID KVM 架构设计文档（本目录）](feat-tlbid-kvm-design.md)
- [OpenEuler OLK-6.6 补丁系列：Optimize flush_tlb_mm() by using TLBID](https://mailweb.openeuler.org/archives/list/kernel@openeuler.org/thread/7JKBJYHMRJQ6J4YACTTK75FTSMFYQ6TS/)
- [OpenEuler 补丁 2/6：Track CPUs that a task has run on for TLBID optimization](https://mailweb.openeuler.org/archives/list/kernel@openeuler.org/message/VSODAF7RGDJN4KMEJQIS3K2WHWXVAPYF/)
- [OpenEuler v2 系列：Support feature TLBI（0/9，找 trans_cpumask_to_domain 实现）](https://mailweb.openeuler.org/hyperkitty/list/kernel@openeuler.org/thread/OQUJHFXYTOSYP7M5K7NPRJZMFLSDQJXV/?sort=date)
- [Arm: TLBIDIDR_EL1 — TLBI Domains Identification Register](https://developer.arm.com/documentation/ddi0601/2025-12/AArch64-Registers/TLBIDIDR-EL1--TLBI-Domains-Identification-Register--EL1-)
- [binutils 补丁系列（+tlbid，2026-01 合入）](https://sourceware.org/pipermail/binutils/2026-January/147699.html)

---

*笔记时间：2026-09-02；代码覆盖：drivers/misc/tlbid_inject.c（v7.3 树新增文件）、arch/arm64/kernel/traps.c:476-487、entry-common.c:357、asm/sysreg.h:40。*
