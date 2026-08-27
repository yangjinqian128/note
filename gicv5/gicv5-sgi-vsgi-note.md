# GICv5 IPI

> GICv5 没有 SGI 类型，因此没有 vSGI——核间通信由 IPI 模式（SPI/LPI + Targeted + Edge + CDPEND）取代。
> 内核主线 master (fc46aed51f6)，规范 ARM-AES-0070 00bet2，Fundamentals 111471 0100-02。

---

## 一、问题定位：GICv5 里 SGI 去哪了

**GICv3/v4 的 SGI 模型**（背景）：

| 属性 | GICv3/v4 SGI |
|------|--------------|
| 全称 | Software Generated Interrupt（软件生成中断） |
| INTID | 0-15，每 PE 固定 16 个 |
| 发送 | 专用系统指令 `ICC_SGI1R_EL1`（写 Distributor 的 SGI1R 寄存器，支持 target list 广播） |
| 虚拟化 | vSGI：vGICv3 用 List Register 软件模拟；GICv4.1 支持 ITS 硬件 vSGI |

**GICv5 把 SGI 类型删除了**：INTID 的 Type 字段只定义 PPI/SPI/LPI 三种编码（规范 §2.4.2），不存在 SGI 编码。规范 §2.5（DXHRZC）定义了替代方案：

> "The architecture supports inter-processor communications by using inter-processor interrupts (IPIs).
> **IPIs are either SPIs or LPIs configured as Targeted** with the Affinity specifying the destination of the IPI.
> Arm recommends that interrupts used for IPIs are not signaled by other interrupt sources."

**关键认知：IPI 不是新的中断类型，是一种"用法模式"**——把普通 SPI/LPI 按特定方式配置和使用。SGI 删除后，GICv5 系统指令集里也没有了 `ICC_SGI1R` 类指令。

---

## 二、架构定义：IPI 的三步配置 + 发送

### 2.1 配置与发送（规范 §2.5，SBDGNX 原文对照）

```
物理域 IPI（源 PE → 目标 PE）:              虚拟机 IPI（源 VPE → 目标 VPE）:
  1. 选中一个物理中断，Targeted 到目标 PE      1. 选中一个虚拟中断，Targeted 到目标 VPE
  2. Handling mode 配成 Edge                 2. Handling mode 配成 Edge
  3. 源 PE 执行 GIC CDPEND 指定 INTID        3. 源 VPE 执行 GIC CDPEND 指定 INTID
```

指令集（SFYVVP）：配置和发送只用三条已有指令——`GIC <domain>AFF`（路由）、`GIC <domain>HM`（Edge）、`GIC <domain>PEND`（发送）。**IPI 没有专用指令**。

### 2.2 发送时的流协议时序（规范附录 A7.6）

```
 IRS                              CPUIF                         PE
  │                                 │                            │
  │                                 │          GIC CDPEND, Xt     │
  │        SetPending(              │◄───────────────────────────┤
  │        Domain=NS, Virtual=0,    │                            │
  │        INTID=A, Pending=1)      │                            │
  │◄────────────────────────────────┤                            │
  │        SetAck() ───────────────►│                            │
  │                                 │  ★ SetAck 返回后，           │
  │  "Acknowledging SetPending      │    GSB SYS 才能完成          │
  │   does not guarantee that the   │                            │
  │   interrupt has yet been        │                            │
  │   forwarded to the target PE."  │                            │
```

### 2.3 三条语义要点

**① 有限时间投递**（ISZFVG）：
> "when an IPI is made pending, if the interrupt meets the conditions to be signaled by the IRI, it is signaled to the target PE or VPE **in finite time**."

**② 无异常保证**（§2.5 Note）：*"there is no guarantee that it will generate an interrupt exception and be handled by the destination (V)PE"*——目标 PE 可能屏蔽了中断或在处理更高优先级中断。IPI 是"请求"，不是"送达回执"。

**③ 显式同步要求**（Litmus B1.11/B1.21/B1.23）：
- B1.11：*"sending an IPI is only required to be observed by an acknowledge when using **explicit synchronization**"*——目标 PE 的 `CDIA` 应答不保证观察到发送方的内存写，除非配 GSB
- B1.21：IPI + GSB SYS + flag 是规范的跨 PE 消息传递模式
- B1.23（edge merging）：多次 CDPEND 同一 IPI 在目标侧**合并为一次 pending**（edge 语义天然去重）——收到一次 IPI 不代表源端只发了一次

### 2.4 数量规划（SPJWMR）

```
m 个 PE × 每 PE n 个 IPI → 分配 m*n 个 LPI
固定算术映射: PE y 的 x 号 IPI = LPI ID (y*n) + x
```

- **LPI 优先**（IWHWNY）：*"Arm expects software to use LPIs allocated by system software to send IPIs"*
- **SPI 后备**（IBNDTT）：无 LPI 实现时，用未接线的 SPI 做 IPI

---

## 三、内核实现：物理 IPI 全链路

### 3.1 数据结构：IPI domain 层级挂 LPI domain 之上

```
IPI Domain (hierarchical, parent = LPI domain)     irq-gic-v5.c:1067-1076
  ├─ 大小 = GICV5_IPIS_PER_CPU * nr_cpu_ids
  ├─ GICV5_IPIS_PER_CPU = MAX_IPI                  arm-gic-v5.h:15
  └─ handler = handle_percpu_irq

LPI Domain (parent)
  └─ 每个 (CPU, IPI 类型) 组合一个专属 LPI，Targeted 到该 CPU
```

### 3.2 分配路径

```c
// irq-gic-v5.c:862 gicv5_irq_ipi_domain_alloc
ret = irq_domain_alloc_irqs_parent(domain, virq, nr_irqs, arg);  // ① 先向父域申请 LPI
for (...) {
    irq_domain_set_hwirq_and_chip(domain, virq, i,
                                  &gicv5_ipi_irq_chip, NULL);   // ② hwirq = 域内序号 i
    irq_set_handler(virq, handle_percpu_irq);                    // ③ 每 CPU 投递语义
}

// irq-gic-v5.c:1013-1024 — 启动时一次分配全部
unsigned int num_ipis = GICV5_IPIS_PER_CPU * nr_cpu_ids;
base_ipi_virq = irq_domain_alloc_irqs(gicv5_global_data.ipi_domain, num_ipis, ...);
set_smp_ipi_range_percpu(base_ipi_virq, GICV5_IPIS_PER_CPU, nr_cpu_ids);
```

### 3.3 发送链：一次 IPI = 一条 CDPEND

```
smp_send_reschedule() / smp_cross_call()         kernel/smp.c
  └→ irq chip .ipi_send_single
      └→ gicv5_ipi_send_single()                irq-gic-v5.c:475
          └→ irq_chip_retrigger_hierarchy(d)     // 委托父域 LPI chip
              └→ gicv5_lpi_irq_retrigger()       irq-gic-v5.c:468
                  └→ gicv5_lpi_irq_set_irqchip_state(PENDING, true)
                      └→ gicv5_iri_irq_write_pending_state()  irq-gic-v5.c:434
                          └→ gic_insn(cdpend, CDPEND)  ★ 规范 A7.6 的 PE 侧动作
                              （CPU-IF 转成 SetPending 流协议消息发往 IRS）
```

```c
// irq-gic-v5.c:434 — CDPEND 操作数 = TYPE | ID | PENDING
cdpend = FIELD_PREP(GICV5_GIC_CDPEND_TYPE_MASK, hwirq_type)	|
	 FIELD_PREP(GICV5_GIC_CDPEND_ID_MASK, d->hwirq)		|
	 FIELD_PREP(GICV5_GIC_CDPEND_PENDING_MASK, state);
gic_insn(cdpend, CDPEND);
```

### 3.4 接收链：普通 LPI 投递，无特殊路径

IPI LPI 是 Targeted 到目标 CPU 的普通 LPI——接收走 LPI 热路径（`CDIA → GSB_ACK → ISB → CDEOI → handle_irq_per_domain → generic_handle_domain_irq`），handler 是 `handle_percpu_irq`（irq-gic-v5.c:876）。**GICv5 硬件和驱动都没有"IPI 快速通道"——IPI 的延迟与普通 LPI 相同。**

### 3.5 与 GICv3 实现对比

| | GICv3（内核） | GICv5（内核） |
|--|--------------|--------------|
| 发送实现 | `gic_ipi_send_single` → 写 `ICC_SGI1R_EL1`（irq-gic-v3.c:1359-1372） | `gicv5_ipi_send_single` → `GIC CDPEND`（irq-gic-v5.c:475） |
| 目标编码 | MPIDR 式 Aff0-3 + target list（可 1→N 广播） | IAFFID（Targeted，1→1） |
| 中断载体 | SGI INTID 0-15（专用类型） | LPI（软件分配的普通中断） |
| 数量 | 每 PE 固定 16 | `MAX_IPI × nr_cpu_ids`，软件可扩 |

---

## 四、虚拟侧：vSGI 的消失与虚拟 IPI 取代

### 4.1 vSGI 不存在的三层证据

**① 类型层面**：规范 §2.4 的 INTID Type 编码只有 PPI/SPI/LPI——没有 SGI 编码，vSGI 无从定义。

**② 规范层面**：AES0070 里 SGI 字样全部出现在 GICv3 legacy 兼容上下文（见第五节），正文虚拟中断类型只有 vPPI/vLPI/vSPI（Fundamentals §6.1）。

**③ 内核层面**：`vgic-v5.c` / `vgic-v5-sr.c` / `irq-gic-v5*.c` 中 grep SGI 零结果——SGI 代码全部在 `vgic-v3.c`（GICv2/v3 legacy 路径）。

### 4.2 替代机制：虚拟 IPI（规范 §2.5 SBDGNX 第二段 + A7.6 虚拟部分）

```
虚拟 IPI = vLPI（或 vSPI）配置 Targeted（affinity = 目标 VPE ID）+ Edge

 IRS                            CPUIF                        PE
  │                               │                           │
  │                               │   写 ICH_CONTEXTR_EL2       │
  │                               │   使 VPE 0 of VM 0 resident │
  │    SetResident(...)           │                           │
  │◄──────────────────────────────┤                           │
  │                               │   ...guest 运行中...        │
  │                               │        GIC CDPEND, Xt      │
  │    SetPending(Domain=NS,      │◄──────────────────────────┤
  │    Virtual=1, INTID=A,        │                           │
  │    Pending=1)                 │                           │
  │◄──────────────────────────────┤                           │
  │    SetAck() ─────────────────►│                           │
  │                               │                           │
  ★ "For virtual IPIs, the VPE is taken from the previous
     SetResident() command" —— Virtual=1 命令隐式使用
     当前 resident VPE 的上下文
```

**与物理 IPI 完全同构**——唯一差别：命令里 `Virtual=1`、affinity 字段装 VPE ID。Guest 里的 `smp_cross_call` 用同一条 `CDPEND` 指令，硬件直通，零 hypervisor 陷阱。这正是 GICv5 "物理/虚拟统一编程接口"（Fundamentals ch.6）在 IPI 上的体现。

### 4.3 GICv4 的 vSGI 三形态（对照背景）

| 形态 | 机制 | 内核证据 |
|------|------|---------|
| vGICv3 软件模拟 | SGI 走 List Register（中断"注入"由 hypervisor 软件完成） | `vgic-v3.c` 的 `is_v2_sgi` 分支（:77-110） |
| GICv4.1 硬件 vSGI | ITS 支持 vSGI，配置同步到 vPE | `vgic_v4_sync_sgi_config`（vgic-v4.c:108） |
| **GICv5** | **无 vSGI——虚拟 IPI（vLPI/vSPI + CDPEND）直通** | vgic-v5.c 零 SGI 代码 |

GICv3，guest写ICC_VSGIR -> **trap** -> KVM LR注入 -> **kick vcpu** -> guest handle
GICv4.1，guest写ICC_VSGIR -> **trap** -> KVM 直接注入 -> guest handle
GICv5，guest发CDPEND -> 硬件中断pending -> geust handle

GICv3 -> GICv4.1 -> GICv5 ，vSGI的性能是越来越好的。

### 4.4 主线合入状态

- **物理 IPI**：已合入（Host Driver v7 系列）
- **虚拟 IPI / vLPI 基础设施**：**未合入**——`vgic-v5.c` 目前只实现 vPPI（DVI）部分，没有任何 vLPI/vIPI 代码；Virtual LPI IST、VM/VPE 表、`VDPEND` 注入全部在 KVM IRS 系列（v4, 48 patches）里。合入前，GICv5 模式下 guest 的 IPI 无法直通 [INFERRED：legacy 模式（V3=1）guest 可用第五节陷阱模拟的 SGI]

---

## 五、Legacy 兼容：GICv3 SGI 的陷阱模拟

GICv5 CPU 接口支持可选向后兼容（`ICH_VCTLR_EL2.V3=1`）。为 legacy guest 服务的 SGI 痕迹：

**① `ICH_HCR_EL2.TC` 陷阱**（规范 §9.7）——GICv5 模式下 trap GICv3 寄存器访问：

> "This affects accesses to **ICC_SGI0R_EL1, ICC_SGI1R_EL1, ICC_ASGI1R_EL1**, ICC_CTLR_EL1, ICC_DIR_EL1, ICC_PMR_EL1, ICC_RPR_EL1, ICV_CTLR_EL1, ..."

即 GICv5 guest 执行 GICv3 的 SGI 指令 → 硬件不实现该寄存器 → trap 给 hypervisor 模拟。

**② `ICH_HCR_EL2.vSGIEOICount`**（规范 §9.7）——GICv4.1 遗留字段，控制"虚拟 SGI 去激活是否计入 EOIcount"，仅当 GICv4.1 实现时存在。

**③ 内核 legacy 路径**：`vgic-v3.c:41-42` 在无 SGI 的 AP list 场景设置 `ICH_HCR_EL2_vSGIEOICount`；SGI 注入/回读的 `is_v2_sgi` 分支全在 vgic-v3.c。**这些代码在 GICv5 模式（V3=0）下不参与工作。**

---

## 六、设计权衡：为什么删 SGI

**删除的理由**：

1. **类型统一**：GICv5 中断模型收敛为 PPI/SPI/LPI 三种，SGI 在状态机、路由、虚拟化上都是特例（GICv3 的 SGI 状态在 Distributor 里按 target PE 复制，pending 模型与 SPI/LPI 不同）——删除后整个模型只剩"同一套 Pending/Active + 路由"语义
2. **数量可扩展**：SGI 固定 16 个，IPI 模式是软件分配的 m×n 个 LPI（SPJWMR），随系统规模扩展
3. **虚拟化零陷阱**：guest 的 `CDPEND` 直通硬件（Virtual Domain），而 GICv3 的 `ICC_SGI1R` 要么 trap 模拟要么依赖 GICv4.1 硬件 vSGI
4. **硬件简化**：GICD 的 SGI 广播/分发路径删除，IPI 复用既有 LPI 投递管道

**付出的代价**：

1. **广播语义损失**：GICv3 的 `ICC_SGI1R` 支持 target list 1→N 广播（一条指令通知多个 PE）；GICv5 的 IPI 定义为 Targeted（SBDGNX），1→1——广播需软件逐 PE 循环发 N 次 CDPEND。1ofN 路由解决不了这个问题（那是"任意一个"，不是"全部"） [来源: 规范 §2.5 定义 + ICC_SGI1R target list 语义对比]
2. **消耗 LPI 资源**：每 CPU `MAX_IPI` 个 LPI 占用 IST 条目（换来的是数量可编程）
3. **同步语义靠 GSB 兜底**：B1.11/B1.21 表明 IPI 的"被观察"需要显式同步——这与 SGI 时代一致，但 IPI 模式下"SetAck 不代表已投递"（A7.6）使软件必须更清楚地区分"命令被接受"与"中断被投递"

---

## 七、对比总表

| 维度 | GICv4 SGI/vSGI | GICv5 IPI/虚拟 IPI |
|------|---------------|-------------------|
| 中断类型 | 专用 SGI（INTID 0-15） | 无新类型——复用 SPI/LPI |
| 发送指令 | `ICC_SGI1R_EL1`（专用） | `GIC CDPEND`（复用） |
| 底层消息 | 写 Distributor SGI1R | `SetPending` 流协议消息 + `SetAck` |
| 目标 | MPIDR Aff + target list（可广播） | IAFFID / VPE ID（Targeted 1→1） |
| 数量 | 每 PE 16，固定 | m×n 软件分配，可扩展 |
| 虚拟化 | LR 模拟 / GICv4.1 硬件 vSGI | 虚拟 IPI（CDPEND 直通，零陷阱） |
| 内核发送 | `gic_ipi_send_single` → SGI1R | `gicv5_ipi_send_single` → retrigger → CDPEND |
| 投递保证 | 无（同一语义） | 无（有限时间 + 可能不产生异常） |

---

*笔记时间：2026-08-27*
*内核版本：master (fc46aed51f6)*
*规范：ARM-AES-0070 00bet2 §2.5、§9.7、附录 A7.6、Litmus B1.11/B1.21/B1.23；Fundamentals 111471 0100-02*

## 参考链接

**规范：**
- [Arm GICv5 Architecture Specification (AES0070) — developer.arm.com](https://developer.arm.com/documentation/aes0070)
- [Learn the Architecture: GICv5 Overview (111471) — developer.arm.com](https://developer.arm.com/documentation/111471)

**内核代码：**
- [irq-gic-v5.c — IPI domain / ipi_send_single / CDPEND 路径](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/irqchip/irq-gic-v5.c)
- [irq-gic-v3.c — GICv3 ICC_SGI1R 发送路径](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/irqchip/irq-gic-v3.c)
- [vgic-v3.c — legacy SGI 模拟](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/arm64/kvm/vgic/vgic-v3.c)
- [vgic-v4.c — GICv4.1 vSGI 配置同步](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/arm64/kvm/vgic/vgic-v4.c)

**邮件列表：**
- [KVM: arm64: Add GICv5 IRS support (v4) — 虚拟 IPI/vLPI 未合入系列](https://lists.infradead.org/pipermail/linux-arm-kernel/2026-July/)
