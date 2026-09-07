# FEAT_TLBID 域语义探测驱动设计（tlbid_probe）

> 目标：实测硬件对 TLBID 域语义的执行——证明带 domain=D 的 TLBI 指令恰好清了该清的 PE（既不过清也不漏清），输出完整的 **domain→PE 映射表**。
> 接口约束：**无 fs 接口**，全部通过 insmod 参数 + dmesg；驱动加载"不成功"（完成后 return -EAGAIN 自卸载），可反复 insmod 重试。
> 代码：`drivers/misc/tlbid_probe.c`；补丁：`tlbid-probe.patch`。

---

## 1. 核心机制：fault 探针（TLB-only 走钢丝）

把每个 PE 上的 TLB 条目变成"被清即断"的线。条目被清时，下次访问**必然 fault**（页表已被摘除，TLB 是唯一通路），观测信号是二值的：

```
arm (布哨):
  共享 guard mm ──► N 个 kthread 各钉死一个 PE, 全部 kthread_use_mm(同一 mm)
                    → 同一 ASID; 每个 PE 的 TLB 都填入探测页条目
  然后 pte_clear(摘页表, 不发 TLBI)
                    → 探测条目进入 TLB-only 状态

inject:
  tlbi aside1is, (ASID<<48) | domain=D    (在固定注入 PE 上执行)

report (观测): 每个 PE 的 kthread 再访问探测页 (__ex_table fixup 包住):
  TLB hit  → 条目还在 → 该 PE 不在 domain D 内
  fault    → 条目被清 → 该 PE 在 domain D 内  ✓
```

**内置控制列**：TLBI 必清发出它的 PE（本地 PE 不参与域匹配）——注入 PE 那一列每轮必为 ✓，若不对说明观测机制有 bug，本轮作废。

## 2. 接口与两种模式（insmod 参数，无 fs）

```
insmod tlbid_probe.ko mode=scan                     # 模式1: 全量遍历
insmod tlbid_probe.ko mode=probe domain=5 [pes_mask=0xff]  # 模式2: 单域

pes_mask: 参与探测的 PE 位图 (bit N = CPU N); 0 = 全部在线 CPU (上限 64)
```

驱动加载后全程在 `module_init` 内同步完成：检测 → 建哨 → 探测 → dmesg 输出 → 销毁 → `return -EAGAIN` 自卸载。insmod 报错是预期行为，结果看 dmesg。

### 模式1（scan）：全量遍历

```
① system_supports_tlbid() 检查   → 不支持: pr_err + return -ENODEV
② 读 TLBIDIDR_EL1 得 NIS         → 域总数 = 2^NIS
③ for domain in 0 .. 2^NIS-1:
       arm + verify (见 §4) → 摘页表 → inject(domain) → report
   → 每行输出: domain=xx pes=<位图>
```

NIS=8 时 256 轮（秒级）；NIS=16 时 65536 轮（分钟级）——insmod 会阻塞到跑完，测试场景可接受。

### 模式2（probe）：单域探测

同一套 arm/inject/report 原语只跑一轮，输出该域的 PE 位图。用于复核可疑域、对照实验（如 VTLBIDEn 开/关各探测一次）。

## 3. 检测链（按用户指定的方式）

```c
/* OpenEuler OLK-6.6 TLBID 补丁系列提供; 主线 v7.3 暂无, 构建时依赖 */
if (!system_supports_tlbid()) {
	pr_err("FEAT_TLBID not supported on this CPU\n");
	return -ENODEV;
}
nis = FIELD_GET(GENMASK(4, 0), read_sysreg_s(SYS_TLBIDIDR_EL1));
```

`SYS_TLBIDIDR_EL1` 宏同样来自补丁系列。这两个符号在 OpenEuler 树上存在即可（构建目标 = 有 TLBID 补丁的树）。

## 4. 模块内部结构

```
tlbid_probe_init()                     [insmod 进程上下文, 全程同步]
  ├─ 检测 + 读 NIS
  ├─ st.mm = current->mm               (复用 insmod 自己的 mm/ASID)
  ├─ guard_setup(): vm_mmap 探测页 → get_user_pages 建 PTE → 手走页表定位 ptep
  │    ★ fault-in 必须走 GUP: EL1 访问用户地址的缺页不走 handle_mm_fault
  │      (__do_kernel_fault 里 fixup_exception 优先级最高, 见 §4 机制表)
  │    (只用 *_offset()/pte_offset_map() 内联——分配类 __ 函数未导出)
  ├─ kthread_create × N + kthread_bind (每 PE 一个, 钉核后 wake)
  │    └─ 各线程: kthread_use_mm(insmod 的 mm) → complete(ready)
  ├─ 循环 (每轮):
  │    arm:    set_pte(恢复 orig_pte) → poke 各 PE 访问 → 校验全部 hit
  │            (任一 fault = 环境冲刷 → 重试 ≤3 次)
  │    clear:  pte_clear(无 TLBI) + dsb(ishst)     → TLB-only
  │    inject: smp_call_function_single(注入PE) 执行
  │            dsb(ishst) → tlbi aside1is → dsb(ish) → isb
  │    report: poke 各 PE 再访问 → fault 位图
  ├─ pr_info 输出映射表
  ├─ 销毁: complete_all + kthread_stop + mmput
  └─ return -EAGAIN (自动卸载)
```

关键机制与内核先例：

| 机制 | 先例/依据 |
|---|---|
| 共享 mm 同 ASID 跨 PE | `kthread_use_mm` → `activate_mm` → `check_and_switch_context` 分配 ASID；同 mm 稳态下各 PE 同 ASID |
| fault 捕获 | `__ex_table` fixup 对 **EL1 数据缺页生效**（`__do_kernel_fault` 里 `fixup_exception` **优先级最高**、先于一切页表处理）——fixup 只用于观测（PTE 摘除后的访问），**fault-in 必须走 `get_user_pages`**，否则 fixup 会吞掉建页表的首次访问 |
| TLB-only | 恢复**相同 PTE 值** + 不刷 TLB → 已有条目继续命中；ptep 手术用 `pte_offset_map_lock`（arm64 上即 identity 映射，ptep 长期有效） |
| 注入固定 PE | `smp_call_function_single` → IPI 上下文执行，天然免迁移 |

## 5. 环境干扰与防御（"有没有别的进程全刷 TLB"）

| 干扰源 | 是否碰哨兵 | 概率 |
|---|---|---|
| 其他进程 `aside1is`（自己的 ASID） | 否 | 排除 |
| 按 VA 范围失效（自己的 VA） | 否 | 排除 |
| 全刷 `vmalle1is`（flush_tlb_all 等） | **是** | 正常运行时极罕见 |
| KVM `alle1is`（VMID 轮转） | **是** | 罕见 |
| 8-bit ASID 代滚动（4K/16K 页） | **是** | 繁忙机器分钟级；16-bit ASID（64K 页）几乎不可能 |

防御（两层，做进实现）：

1. **verify-before-inject**：arm 之后、注入之前校验所有 PE 哨兵在位——若注入前哨兵已丢 = 环境冲刷，该轮作废自动重试（≤3 次），不误报成硬件行为。
2. **异常结果复核**：report 显示全 PE 被清（含域外 PE）时重试一轮，排除环境全刷再下结论。

实验纪律：安静测试机 / 专用安静 guest；单轮毫秒级。综合：环境误杀概率极低，且一旦发生会被 verify 捕获而非污染结论。

## 6. host / guest 双态（同一份代码）

| | 注入的域空间 | 观测粒度 | 前提 |
|---|---|---|---|
| host | 物理域（NIS 位） | 物理 PE | VHE 内核 EL2 直用物理域；非 VHE 裸机 VTLBIDEn=0 不翻译 |
| guest | 虚拟域（NVIS 位） | vCPU（host 侧把 vCPU 1:1 绑核后对应到物理 PE） | KVM 编程 VTLBID 表 + 开 VTLBIDEn；VMID 隔离保证安全 |

**联合对照**：host 表（物理域→PE）+ guest 表（虚拟域→vCPU）+ 绑核对照表（vCPU→PE），三表一对 → TD 表的虚拟→物理翻译被直接实测，设计文档 G1/G2/翻译语义一次性闭环。guest 侧扫描上限应取 **NVIS**（虚拟域空间），实现时按场景切换。

## 7. 输出与判读

```
tlbid_probe: TLBIDIDR_EL1=0x... nis=6 nvis=5 nos=6 nvos=5
tlbid_probe: scan pes=8 asid=0x1 injector=0
tlbid_probe: domain=0x00 pes=0,0x1   ← 只有注入 PE(0) 被清? 说明域 0 = {PE0}
tlbid_probe: domain=0x01 pes=0,0x3   ← 域 1 = {PE0, PE1}, PE0 是控制列
tlbid_probe: domain=0x3f pes=0,0xff  ← 全 1 域: 全部响应 → G2 = 全域广播
```

判读规则：
- 每行去掉注入 PE 控制列后 = 该域的真实成员集合
- 映射表形态（每域一核 / 多核共享）= G1 的答案
- 全 1 域是否全响应 = G2 的答案
- 某域**漏了**预期 PE = 漏清证据（正确性风险）；**多了**预期外 PE = 过清证据（性能损失）

## 8. 与 tlbid_inject.ko 的关系

`tlbid_inject.ko`（单发注入器）继续承担"快速冒烟"：一条 insmod 看 oops-vs-OK，测 CONSTRAINED UNPREDICTABLE 分支。`tlbid_probe.ko` 承担映射表测绘。两者独立、可共用同一套 ASID/编码约定。

## 9. 构建依赖与边界

- 依赖 `system_supports_tlbid()` 与 `SYS_TLBIDIDR_EL1`（OpenEuler TLBID 补丁系列提供）——在含补丁的树（OLK）上构建；主线 v7.3 上这两个符号不存在，编译会失败，属预期。
- 公共框架适配性已逐项核对 OLK-6.6：**`mm_alloc` / `__pud_alloc` / `__pmd_alloc` / `__pte_offset_map_lock` 均未导出**（v7.3 同样）→ 驱动复用 insmod 自己的 mm + `*_offset()`/`pte_offset_map()` 手走页表绕开；`kthread_use_mm/unuse_mm`（GPL）、`vm_mmap`、`mmap_read_lock`、`kthread_bind/stop`（含 TIF_NOTIFY_SIGNAL 机制）、`fixup_exception`（EL1 数据缺页两处调用点 fault.c:490,940）、`read_sysreg_s`、`%*pb` 等全部适配。
- 快照语义：映射表是测量时刻的值；域成员若可编程则只对当时有效。
- pes_mask 上限 64 PE（u64 位图），够测试用。
- 8-bit ASID 系统长时间 scan 可能跨代滚动——每轮 arm 前重读 `ASID(mm)` 并依赖 verify 兜底。

---

## 参考链接

- [FEAT_TLBID KVM 架构设计文档（本目录）](feat-tlbid-kvm-design.md)
- [tlbid_inject 驱动笔记（本目录）](tlbid-inject-driver.md)
- [OpenEuler OLK-6.6 TLBID 补丁系列](https://mailweb.openeuler.org/archives/list/kernel@openeuler.org/thread/7JKBJYHMRJQ6J4YACTTK75FTSMFYQ6TS/)
- [Arm: TLBIDIDR_EL1 寄存器文档](https://developer.arm.com/documentation/ddi0601/2025-12/AArch64-Registers/TLBIDIDR-EL1--TLBI-Domains-Identification-Register--EL1-)

---

*设计时间：2026-09-02。*
