# GICv5 是否存在 GICv4.1 VMOVP 性能问题

## 一、GICv4.1 VMOVP 问题的本质

### 1.1 VMOVP 是什么

VMOVP（VM Move Pending state）是 GICv4.x ITS 命令集中的一条命令，用于把 VPE（Virtual Processing Element，即 vCPU 的中断虚拟化实体）从一个 Redistributor 迁移到另一个 Redistributor。内核在 vCPU 迁移时通过 `its_vpe_set_affinity()` 触发：

```
its_vpe_set_affinity()                          // irq-gic-v3-its.c:3907
  ├─ vmapp_lock (per-VM, its_list 场景)          // :3952
  ├─ vpe_to_cpuid_lock()  (col_idx 自旋锁)       // :3954
  ├─ its_send_vmovp(vpe)                         // :3976
  ├─ its_vpe_4_1_invall_locked() (华为 workaround) // :3980
  └─ its_vpe_db_proxy_move(vpe, from, cpu)       // :3982
```

### 1.2 为什么贵——内核代码中的四条证据链

**证据 ①：作者注释直接承认昂贵**

```c
/*
 * Changing affinity is mega expensive, so let's be as lazy as
 * we can and only do it if we really have to.
 */
```
— `its_vpe_set_affinity()`，irq-gic-v3-its.c:3935-3936

**证据 ②：its_list 场景下是全系统串行化点**

```c
/*
 * Yet another marvel of the architecture. If using the
 * its_list "feature", we need to make sure that all ITSs
 * receive all VMOVP commands in the same order. The only way
 * to guarantee this is to make vmovp a serialization point.
 *
 * Wall <-- Head.
 */
guard(raw_spinlock)(&vmovp_lock);               // 全局锁
desc.its_vmovp_cmd.seq_num = vmovp_seq_num++;   // 全局序号

/* Emit VMOVPs */
list_for_each_entry(its, &its_nodes, entry) {   // 广播到所有 ITS
    ...
    its_send_single_vcommand(its, its_build_vmovp_cmd, &desc);
}
```
— `its_send_vmovp()`，irq-gic-v3-its.c:1423-1445

当一个系统使用 ITS List（多个 ITS 可翻译同一 vLPI），**整个系统的所有 VM 的所有 vCPU 迁移**都会串行在 `vmovp_lock` 上，且每次迁移要向**每一个** ITS 发送一条 VMOVP 命令。系统越大（ITS 越多、VM 越多），这个串行化点越致命。

**证据 ③：VMOVP 必须走 ITS 命令队列**

ITS 命令不是寄存器写入，而是写入 GITS_CBASER 指向的内存命令队列，硬件异步执行，软件轮询 CREADR 判断完成。单条命令的完成时间取决于 ITS 负载——多个设备同时在翻译 MSI 时，VMOVP 排在队列尾部。这与 GICv5 的系统寄存器写路径（同步、无排队）形成根本差异。

**证据 ④：迁移期间 VPE 处于"墙"后**

`its_send_vmovp()` 注释中的 "Wall <-- Head" 是架构语义：VMOVP 执行期间，该 VPE 的 vLPI 投递被挂起——旧 RD 已失效，新 RD 尚未生效。vCPU 迁移的这段时间内，中断**无法投递**，只能 pending 在 VPT（Virtual Pending Table）中等待。

### 1.3 附加成本：doorbell 迁移

GICv4.0（无 rvpeid）还需要移动 doorbell：

```
its_vpe_db_proxy_move(vpe, from, to)             // irq-gic-v3-its.c:3863
  ├─ 直接注入 LPI 路径: GICR_CLRLPIR + wait_for_syncr   // :3872-3879
  └─ 代理设备路径: 自旋锁 + its_send_movi()              // :3882-3890
       └─ 又是一条 ITS 命令（MOVI，通过 proxy 设备）
```

GICv4.1 虽然去掉了 proxy 设备（`has_rvpeid` 直接 return），但 doorbell LPI 的语义仍然绑定在 RD 上——这正是下面 1.4 节 bug 的根源。

### 1.4 为什么共享 VPE 表的缓解措施失败了

VMOVP 之所以必须存在且昂贵，根因是 **GICv4.x 的 VPE pending state（VPT）挂在 Redistributor 上**——迁移 VPE = 在 ITS 和两个 RD 之间同步/转移 pending state 的视图。GICv4.1 的缓解设计是：**多个 RD 共享同一 VPE 表**（`vpe_table_mask`），同一亲和组内的迁移无需真正移动状态。但这一设计在实践中被证明无法免除 VMOVP。下面逐步拆解失败机制。

#### 1.4.1 共享 VPE 表共享了什么，没共享什么

共享 VPE 表的机制：亲和组由 `compute_common_aff()`（GICR_TYPER.Affinity 按 CommonLPIAff 掩码，irq-gic-v3-its.c:2545）定义。**组范围由硬件实现决定**，通过每个 RD 的 `GICR_TYPER.CommonLPIAff`（bits [25:24]）通告：0b00 = 全系统一组；0b01 = Aff3 相同（die 级）；0b10 = Aff3.Aff2 相同（die 内分区）；0b11 = Aff3.Aff2.Aff1 相同（cluster 级）。架构规则：同组 RD 必须共享同一份 vPE 表（及 LPI Configuration 表），不同组绝不能共享（否则 UNPREDICTABLE）；ITS 侧由 `GITS_TYPER.SVPET` 通告同样的层级数（`compute_its_aff()` 据此判断 ITS-RD 同组关系，:2555）。RD 初始化时先尝试继承同组 RD 的 `VPROPBASER` 配置（`inherit_vpe_l1_table_from_rd()`，:2926），继承成功后把自己的 `vpe_table_mask` 标记为与源 RD 同组（:3015）；同组第一个 RD 自行分配 VPE 表（:2997）。组粒度只影响迁移成本大小（组越大，同组迁移越多、越便宜），不改变"rdbase 更新不可省略"的结论。

**共享的内容**：VPE 表内存本身（VPROPBASER 指向的表）。同组 RD 的 VPE 表条目地址一致，条目内的 pending 前缀与 VPT 指针对新旧 RD 同样可见——同组迁移无需在新 RD 处重建表条目，也没有"状态从旧 RD 搬到新 RD"的转移成本。注意区分两张表：**VPT 页（`vpe->vpt_page`）是每 VPE 全局唯一分配的**（`its_vpe_init()`，:4570-4575，VMAPP/VMOVP 编码其物理地址 :904），其地址与亲和组无关；**随组变化的是 VPE 表（条目）的地址**——不同亲和组各自分配一张 VPE 表，地址不同。

**没有共享的内容**：VPE 表条目中的 **rdbase 关联**。VPE 表条目记录"这个 VPE 当前属于哪个 RD"，这个关联字段只有 VMOVP 命令能更新——**即使表内存是共享的，关联字段是全局唯一的，不能共享**。而恰恰是这个关联字段，锚定了 doorbell 的生成与投递位置。（？？？未核实）

#### 1.4.2 为什么GICv4.1免VMOVP有问题

根源在于doorbell的亲和性没有改变。GICv4.1的VMOVP会做doorbell的迁移，协议中写的是：“When VMOVP is issued, if DB=1, the vPE is marked as requesting Default Doorbell generation on the new target.” 硬件具体怎么实施的不得而知。如果在同一个亲和组免VMOVP，会存在一个问题。vLPI路由到新PE上，发现vPE不在位，让GICR触发doorbell LPI去唤醒vPE。结果由于doorbell亲和性没迁移，doorbell LPI在旧PE上触发了，host handle了，kick了vPE，vPE被唤醒了。这样也没问题，vPE被唤醒的目的达到了。

但是存在一个场景，host上一个PE被offline了。这个PE上所有的vPE都会被迁移到其他PE。这个时候vLPI来了，vPE不在位的话，会把doorbell lpi路由到offline的旧PE上，这样就丢中断了。

因此，marc的解决办法是重新开启了同亲和性组的VMOVP。

这里有几个问题没搞懂
- doorbell中断的亲和性是怎么迁移的？
- doorbell中断触发路由路径是怎样的？

## 二、GICv5 的迁移模型：Residency 取代 VMOVP

### 2.1 GICv5 没有 VMOVP——命令集对比

GICv5 ITS 的完整命令集（从寄存器定义推导）：

```
GICv5 ITS (arm-gic-v5.h:181-240):
  DIDR / EIDR          ← 选择 deviceID/eventID
  INV_EVENTR           ← 失效单个 ITT 条目缓存
  INV_DEVICER          ← 失效设备表条目缓存
  SYNCR / SYNC_STATUSR ← 同步

GICv4.x ITS (额外拥有):
  VMAPP / VMAPI / VMAPTI / VMOVI / VMOVP / VINVALL ...
  VPE 表、VPT、doorbell 代理 —— 全部不存在于 GICv5
```

GICv5 的 ITS 退化为**纯翻译组件**（deviceID+eventID → LPI INTID），不持有任何 VPE 状态。VPE 相关的一切——pending 状态、residency、doorbell——全部移到 IRS 和 PE 的 IRI 上。

### 2.2 中断状态在共享内存 IST 中

GICv4.1 的根因是 pending state per-RD。GICv5 的 **IST（Interrupt State Table）是系统内存中的共享表**，由 IRS 硬件管理（驱动在 `gicv5_irs_init_ist()` 中分配，irq-gic-v5-irs.c:292，物理地址写入 `IRS_IST_BASER`）。任何 IRS 读取任何 VPE 的中断状态都是直接读内存——**不存在"状态属于某个 RD，迁移时要转移"的问题**。

### 2.3 Residency 切换 = 两次系统寄存器写 + ISB

GICv5 中 vCPU 迁移的完整操作序列（来自 GICv5 规范语义 + KVM IRS 系列实现）：

```
旧 PE 侧（make non-resident）:
  写 ICH_CONTEXTR_EL2: V=0, DB=1, DBPM=<门铃优先级掩码>
    └─ 同一操作内请求 doorbell：此后若该 VPE 出现
       高于 DBPM 优先级的 pending SPI/LPI，IRS 会投递 doorbell 唤醒 vCPU
  ISB

新 PE 侧（make resident）:
  写 ICH_CONTEXTR_EL2: V=1, VM=<vmid>, VPE=<vpeid>
  ISB
  （可选）读回 F 位检查操作是否成功/失败
```

关键对比：

| 维度 | GICv4.1 VMOVP | GICv5 residency 切换 |
|------|---------------|---------------------|
| 操作介质 | ITS 命令队列（异步） | EL2 系统寄存器（同步） |
| 涉及组件 | 所有 ITS + 新旧 RD | 本 PE 的 IRI + 本地 IRS |
| 全局串行化 | `vmovp_lock` 全系统串行 | 无（每 PE 独立） |
| pending 状态 | 需要转移/失效视图 | 不需要（IST 共享内存） |
| doorbell 配置 | 额外 MOVI 命令/proxy 移动 | 嵌入同一条寄存器写（DB 位） |
| 中断投递窗口 | "Wall <-- Head" 挂起 | IRI 作为本地边界，ISB 后即生效 |

### 2.4 内核侧的印证：struct gicv5_vpe 只有一个 bool

GICv4.1 内核侧 VPE 状态机（`struct its_vpe`，irq-gic-v3-its.c）维护：`vpe_id`、`col_idx`（当前 RD 列）、`vpe_db_lpi`（doorbell LPI）、`vpe_proxy_event`、`vmapp_count`、`its_vm`（含 `vmapp_lock`、`vlpi_count`、`nr_db_lpis` 等）——迁移涉及跨组件一致性和多把锁。

GICv5 内核侧的 VPE 状态（arm-gic-v5.h:391）：

```c
/* Embedded in kvm.arch */
struct gicv5_vpe {
    bool resident;
};
```

**整个 VPE 的迁移相关状态就是一个 `resident` 布尔**。当前已合入的 vgic-v5.c 中，load/put 的 resident 跟踪就是全部：

```
vgic_v5_load():  if (resident) return; ... resident = true    // vgic-v5.c:452-457
vgic_v5_put():   if (!resident) return; ... resident = false  // vgic-v5.c:470-475
```

KVM IRS 系列（未合入，v4 48 patches）在 load/put 上叠加两个 hyp call——make resident / make non-resident——分别对应上文的两条 `ICH_CONTEXTR_EL2` 写。

### 2.5 为什么 GICv5 可以这样做

架构逻辑链：

1. **SPI/LPI 状态在 IST（共享内存）**，由 IRS 管理 → 迁移 VPE 不需要移动任何状态
2. **IRS 按 PE 直接投递**（1-of-N 路由 + IAFFID）→ 中断可以从任何 IRS 投递到任何 PE，不需要经过 VPE 所在的 RD
3. **residency 只是"投递许可"标记**：resident 的 VPE，IRS 直接选中其 SPI/LPI 投递到宿主 PE；non-resident 的 VPE，IRS 评估 doorbell 条件唤醒 hypervisor
4. **doorbell 请求与 residency 切换原子绑定**（contextr 的 DB 位）→ 没有独立的 doorbell 迁移步骤

## 三、GICv5 残余成本（诚实评估）

GICv5 消除了 VMOVP 问题，但不是零成本。逐项评估：

### 3.1 contextr 写 + ISB 的本体成本

每次 VPE 切换 = 2 次系统寄存器写 + 2 次 ISB（+ 可选 1 次 F 读回）。ISB 冲刷流水线，典型开销数十 ns 量级。**这是 vCPU 切换路径上本来就要付出的 CPU 接口切换成本的一部分**，与 VMOVP 的"命令队列 + 跨组件同步"不在一个数量级。

### 3.2 Residency 生效的有限时间语义

GICv5 遵循 relaxed ordering：residency 变化的传播由 GSB 类同步保证"有限时间内完成"。`ICH_CONTEXTR_EL2.F` 位可供软件轮询确认操作结果。这意味着 make-resident 不是瞬时完成的——但这是**本地 PE-IRS 对**的同步，不存在 GICv4.1 的跨 ITS 一致性协议。 [来源: GICv5 spec §4.10 VPE residency + KVM IRS 系列对 F 位的处理]

### 3.3 Doorbell 条件评估

VPE 变为 non-resident（DB=1）后，IRS 需要评估"该 VPE 是否有候选 HPPI ≥ DBPM"。这是一次硬件侧的 doorbell 条件检查（IRS trace 中的 `GICV5_DOORBELL_CONDITION_MET`）。注意：这是**唤醒路径**的组成部分，不是迁移路径——GICv4.1 的 non-resident VPE 同样需要 doorbell 唤醒。两架构在此处是对等的。

### 3.4 多 IRS 场景 [INFERRED]

GICv5 系统可以有多个 IRS。residency 由 PE 的 IRI 经 GSP 通知其**本地 IRS**；其它 IRS 是否需要感知某 VPE 的 residency 变化，规范层面无法从内核代码完全验证。推断依据：IST 和 VPE 表是共享内存，IRS 按 PE 直接投递（不经过"VPE 所在的 RD"这一中转），因此远端 IRS 不需要 residency 视图即可完成投递或 doorbell。若此推断成立，则 GICv5 连"多组件一致性协议"都不需要；即使不成立，也只需要 doorbell 条件的分布式评估，仍无状态转移。标记 [INFERRED] 以明确边界。

### 3.5 唤醒延迟（两架构共有，非 GICv5 新引入）

VPE non-resident 时，新中断的投递路径是：IRS 评估 doorbell → 投递 doorbell LPI → hypervisor 唤醒 vCPU → 调度 → make resident → 中断投递。这段延迟在 GICv4.1 中同样存在（doorbell LPI 机制）。它是"vCPU 未运行时的中断唤醒成本"，与"vCPU 迁移成本"（VMOVP）正交。

## 四、结论

**GICv5 架构性地消除了 GICv4.1 的 VMOVP 性能问题。**

论证链：

1. VMOVP 昂贵的根因是 GICv4.x 的 pending state 挂在 Redistributor 上，迁移 VPE 需要跨 ITS/多 RD 同步状态视图——这决定了它必须是昂贵的广播式命令（全系统 `vmovp_lock` 串行化、每 ITS 一条命令、迁移窗口内中断挂起）
2. GICv4.1 的架构缓解（共享 VPE 表）被实践证明无效：共享表只消除了"pending state 转移"这一个成本分量，rdbase/doorbell 关联变更无法共享；`af9acbfc` 修复确认 elide VMOVP 会导致 doorbell 投递到旧 CPU（负载失衡）甚至被丢弃（旧 CPU offline 后 VM 永久失去中断唤醒）——详见 §1.4
3. GICv5 把 pending state 移到共享内存 IST，迁移 VPE 不再涉及状态转移；residency 只是投递许可标记，通过两条 EL2 系统寄存器写（`ICH_CONTEXTR_EL2`）+ ISB 完成，doorbell 请求原子嵌入同一操作
4. GICv5 的 ITS 命令集根本没有 VMOVP（也无 VMAPP/VINVALL 等 VPE 命令）——ITS 退化为纯翻译组件
5. 内核侧印证：GICv4.1 需要一整套 `its_vpe` 状态机 + 三把锁 + 多组件一致性协议；GICv5 的 `struct gicv5_vpe` 只有一个 `bool resident`

残余成本（contextr 写 + ISB、doorbell 条件评估、F 位确认）属于 vCPU 切换路径的固有开销，数量级远低于 VMOVP 的命令队列 + 跨组件广播，且**随系统规模（ITS 数 × VM 数）恒为 O(1)**——而 VMOVP 的成本随 ITS 数量线性增长、并受全局锁串行化约束。

**边界声明**：
- GICv5 目前仅有 FVP 平台，无量产硅片，缺乏实测数据。以上是**架构级**结论，基于 GICv5 spec 语义、内核代码证据和 KVM IRS 系列的实现设计
- 多 IRS 的 residency 传播语义标注为 [INFERRED]
- 本文分析的是"迁移操作本身的成本"；vCPU 未运行时的中断唤醒延迟（doorbell 路径）两架构共有，不在对比范围内

---

*分析时间：2026-08-02*
*内核版本：master (fc46aed51f6)*
*证据文件：irq-gic-v3-its.c（GICv4.1 VMOVP 实现）、arm-gic-v5.h（GICv5 VPE 模型）、vgic-v5.c（已合入的 residency 跟踪）、KVM IRS 系列（未合入，v4 48 patches）*

## 参考链接

**GICv4.1 VMOVP 相关：**
- [irqchip/gic-v3-its: Fix GICv4.1 VPE affinity update (af9acbfc) — lore](https://lore.kernel.org/all/20240213101206.2137483-4-maz@kernel.org/)
- [内核源码 its_vpe_set_affinity / its_send_vmovp — drivers/irqchip/irq-gic-v3-its.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/irqchip/irq-gic-v3-its.c)

**GICv5 residency 相关：**
- [KVM: arm64: Add GICv5 IRS support (v2, 39 patches) — lists.infradead.org](https://lists.infradead.org/pipermail/linux-arm-kernel/2026-May/1132232.html)
- [KVM: arm64: gic-v5: Add resident/non-resident hyp calls (v2) — lists.infradead.org](https://lists.infradead.org/pipermail/linux-arm-kernel/2026-May/1132252.html)
- [KVM: arm64: gic-v5: Add VPE doorbell domain (v2) — lists.infradead.org](https://lists.infradead.org/pipermail/linux-arm-kernel/2026-May/1132243.html)
- [KVM: arm64: gic-v5: Implement VPE IRS MMIO Ops (v2) — lists.infradead.org](https://lists.infradead.org/pipermail/linux-arm-kernel/2026-May/1132250.html)

**规范与 TRM：**
- [Arm GICv5 Specification (beta) — developer.arm.com](https://developer.arm.com/documentation/aes0070)
- [ICH_CONTEXTR_EL2 寄存器定义 — Arm A-profile Registers](https://developer.arm.com/documentation/ddi0601/2025-12/AArch64-Registers/ICH-CONTEXTR-EL2--Interrupt-Controller-Virtual-Context-Register)
- [Arm CoreLink GIC-700 TRM — Direct injection: Residency and VMOVP](https://developer.arm.com/documentation/101516/0400/Operation-of-GIC-700/Direct-injection/Residency-and-VMOVP?lang=en)
