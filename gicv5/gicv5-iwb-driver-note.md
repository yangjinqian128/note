# GICv5 IWB 驱动深度调研笔记

> IWB（Interrupt Wire Bridge）：GICv5 四个组件中驱动最薄的成员（301 行）。
> 职责单一：把 wire 信号翻译成 ITS 事件。无内存表、无状态管理、无路由。
> 内核主线 master (fc46aed51f6)，规范 ARM-AES-0070 00bet2。

---

## 一、架构定位：最"薄"的驱动

规范 Chapter 6（IFJFTJ）一句话定义：

> "The IWB detects changes to the state of input wires and signals an ITS. The ITS generates ITS events from the signals and translates them into interrupt events that are forwarded to an IRS."

**事件生成规则**（RWLHDQ）——IWB 的全部行为就是这张表：

```
wire enabled + Edge  + 断言      → SET_EDGE
wire enabled + Level + 断言      → SET_LEVEL
wire enabled + Level + 去断言    → CLEAR
wire disabled                    → 不生成任何事件
IWB 整体 disabled（IWBEN=0）     → 不生成任何事件
```

**硬件固定的身份**（RVKMRC/RRMRTF）：DeviceID = IWB 在关联 ITS 命名空间中的唯一 ID（所有 wire 共用）；EventID = wire 序号（硬连线）。软件无法改变这个映射——这就是 `MSI_ALLOC_FLAGS_FIXED_MSG_DATA` 的硬件根源。

**拓扑约束**：每 IWB 只关联一个 ITS（RFYRGC）；每实例最多 65536 根 wire（ILTGCB）。

---

## 二、寄存器全景：驱动只碰 5 个

**驱动使用的**（arm-gic-v5.h:280-298）：

| 寄存器 | 驱动用法 | 完成语义 |
|--------|---------|---------|
| `IWB_IDR0` | 读 IW_RANGE：wire 数 = (RANGE+1)×32 | — |
| `IWB_CR0` | **只读检查 IWBEN**——"IWB must be enabled in firmware"（:232-235） | IWBEN 0→1 转换完成后，level wire 会生成初始 SET_LEVEL/CLEAR（RVWTHK） |
| `IWB_WENABLER<n>` | 读改写 + **轮询 WENABLE_STATUSR.IDLE** | IWLHND：IDLE=1 保证"wire 状态已变 + 禁用产生的 CLEAR 已被 ITS 接受" |
| `IWB_WENABLE_STATUSR` | 轮询 IDLE | — |
| `IWB_WTMR<n>` | 读改写（1=Level, 0=Edge），**无 IDLE 轮询** | 有限时间语义（与 WENABLER 不对称） |

**驱动不碰的**（重要分工）：

- **`IWB_WDOMAINR<n>`（wire 域分配）**——固件/EL3 的活。规范 6.2（IPVVMN）：*"IWB_WDOMAINR<n> is only accessible in the MPPAS of the IWB"*。驱动假设 wire 已被分配到正确的域
- **`IWB_RESAMPLER`**——规范定义的重采样寄存器，驱动未使用
- **`IWB_WDOMAIN_STATUSR`**——域分配完成的轮询（同上，固件侧）

---

## 三、驱动逐函数解析

### 3.1 数据结构：极简

```c
// irq-gic-v5-iwb.c:18
struct gicv5_iwb_chip_data {
	void __iomem	*iwb_base;   // MMIO 配置帧基地址
	u16		nr_regs;     // WENABLER/WTMR 寄存器组数（=(IW_RANGE+1)）
};
```

对比 ITS 的节点级 xarray + 设备级 event_map，IWB 的 chip_data 只有两个字段——没有内存表可管。

### 3.2 probe 链

```
gicv5_iwb_device_probe (:252)
  ├─ devm_ioremap 配置帧
  └─ gicv5_iwb_init_bases (:216)
       ├─ IDR0 → nr_wires / nr_regs
       ├─ CR0.IWBEN 检查 ★ 固件必须先使能 IWB，驱动无权开
       ├─ 全部 WENABLER 清零 + 等 IDLE   ★ 上电静默（复位序列 SXWPYQ 的一部分）
       └─ msi_create_device_irq_domain   ★ 核心：IWB 不是 irq_domain 驱动，
           (iwb_msi_template, size=nr_wires)  而是 MSI 设备域模板
```

### 3.3 irq_chip 回调：全部"wire 层 + 父层"双动作

```c
// :86 gicv5_iwb_irq_enable —— 两层使能
gicv5_iwb_enable_wire(iwb_node, d->hwirq);   // ① WENABLER 置位 + 等 IDLE
irq_chip_enable_parent(d);                    // ② 父层 CDEN（LPI 可投递）

// :78 gicv5_iwb_irq_disable —— 对称的两层禁用
gicv5_iwb_disable_wire(...);                  // ① 清 WENABLER + 等 IDLE
irq_chip_disable_parent(d);                   // ② 父层 CDDIS

// :94 gicv5_iwb_set_type —— 触发类型
wtmr 读改写: LEVEL_* → 置位；EDGE_* → 清零   // 无 IDLE 轮询

// :169 —— 空函数！
static void gicv5_iwb_write_msi_msg(struct irq_data *d, struct msi_msg *msg) {}
```

**注释点破的关键分层**（:52-55）：*"Enable IWB wire/pin at this point. Note: **This is not the same as enabling the interrupt**"*——wire 使能（IWB 硬件开始监测线）和中断使能（父层 CDEN 让 LPI 可投递）是两层，缺一不可。

---

## 四、MSI 集成机制（核心设计）

IWB 借用 MSI 框架表达"固定消息"语义——**把 wire 伪装成一个 MSI 设备**。

### 4.1 DeviceID 从 DT 的 msi-parent cell 来

```dts
iwb {
	compatible = "arm,gic-v5-iwb";
	msi-parent = <&its0 64>;    // ★ 第三个 cell = DeviceID！
};
```

### 4.2 完整数据流

```
msi-parent cell(64)
  → of_pmsi_get_msi_info()                its-msi-parent.c:226
  → its_v5_pmsi_prepare()                 its-msi-parent.c:218
      ├─ scratchpad[0] = dev_id           :235
      ├─ scratchpad[1] = ITS 翻译帧地址    :237
      └─ → gicv5_its_msi_prepare()        its.c:801
            └─ gicv5_its_alloc_device(its, nvec, dev_id=64)  its.c:811
                 ★ 该 IWB 的 DTE/ITT 诞生（设备注册协议，见内存表笔记）
```

挂载链：`gic_v5_its_msi_parent_ops`（its-msi-parent.c:332，`.init_dev_msi_info = its_v5_init_dev_msi_info` :296）在 `DOMAIN_BUS_WIRED_TO_MSI` 时把 `msi_prepare` 指到 `its_v5_pmsi_prepare`（:310）——IWB 与普通平台 MSI 共用同一条 prepare 链，唯一差别是总线类型。

### 4.3 FIXED_MSG_DATA 的三处联动

```c
// ① 模板声明 (:200): EventID 固定，由硬件身份决定
.alloc_info.flags = MSI_ALLOC_FLAGS_FIXED_MSG_DATA

// ② ITS 侧 (:888): 固定模式跳过 event_map bitmap，EventID = hwirq = wire 号
if (!(info->flags & MSI_ALLOC_FLAGS_FIXED_MSG_DATA))
	event_id_base = bitmap_find_free_region(its_dev->event_map, ...);  // 动态
else
	event_id_base = info->hwirq;   // ★ 固定 = wire 号（且 nr_irqs 必须 == 1）

// ③ chip 侧 (:169): 消息无需写——没有 MSI-X 表可写
static void gicv5_iwb_write_msi_msg(...) {}
```

### 4.4 wire 号的流转：icookie

```c
// :130 gicv5_iwb_domain_set_desc
alloc_info->hwirq = (u32)desc->data.icookie.value;

// msi.c:1584 msi_device_domain_alloc_wired —— icookie 的来源
icookie.value = ((u64)type << 32) | hwirq;   // ★ wire 号经 icookie 流入
```

`DOMAIN_BUS_WIRED_TO_MSI` 是专为 IWB 新增的总线类型（msi.c:1584/1731 的两处 WARN 守卫就是它）。消费方视角：设备节点把 IWB 当 interrupt-controller 引用（2-cell `[wire, type]`），`gicv5_iwb_irq_domain_translate`（:136）解析 hwirq=wire——**对外呈现"有线中断控制器"，对内是 MSI 机制**。ACPI 侧 wire 号从 GSI 编码提取（`GICV5_GSI_IWB_WIRE` [15:0]，:162）。

---

## 五、规范语义的精细之处（驱动行为的硬件依据）

1. **使能 level wire = 电平采样**（RSDJBH）：WENABLER 置 1 时，level wire 若当前断言则**立刻**生成 SET_LEVEL（去断言则 CLEAR）——设备状态在使能瞬间被采样；edge wire 无初始事件
2. **禁用 level wire = 补发 CLEAR**（SDDQDS）：*"The IWB is required to generate a CLEAR event as a result of individually disabling a level-sensitive wire"*——目的是重新采样电平、防止 pending 残留。**且 IDLE=1 之前这些 CLEAR 必须已被 ITS 接受**（IWLHND）——这就是 WENABLER 写必须轮询的原因
3. **edge 合并**（RPKSVB）：多次断言在事件发出前可合并为**一个** SET_EDGE（edge 去重，与 IPI 的合并同源）
4. **IWBEN=0 时 WENABLER 写只是存值**（无事件）——驱动"先清 WENABLER、固件再开 IWBEN"的顺序是安全的
5. **IWBEN 0→1 转换**（RVWTHK）：level wire 在转换完成时生成初始 SET_LEVEL/CLEAR；edge wire 无事件
6. **IWBEN 1→0 转换**（RDRQYB）：*"the IWB has generated a CLEAR event for every level-sensitive wire where it had sent a SET_LEVEL event and no corresponding CLEAR"*——关电前结清电平账
7. **MPPAS 访问控制**（IVFDHJ）：WENABLER/WTMR 只能从 MPPAS 或 wire 所属域的 PAS 写；WDOMAINR 仅 MPPAS（IPVVMN）——域隔离在寄存器访问层面强制
8. **域分配可固定可编程**（SMMLFL）：软件尝试改写 WDOMAINR 后读回验证——读回值没变说明硬件固定

---

## 六、并发与坑

1. **读改写无锁**：`__gicv5_iwb_set_wire_enable`（:40）和 `gicv5_iwb_set_type`（:94）的 read-modify-write **没有锁保护**——两个不同 wire（同一 32 位寄存器的不同位）从不同 CPU 并发操作会丢更新。对比 IRS 驱动的 `spi_config_lock` 保护 SELR 窗口。irq core 只串行化**同一**中断的操作，跨中断并发是真实窗口 [INFERRED：规范未给原子性要求，驱动应加锁]
2. **WTMR 写无完成确认**：与 WENABLER 的 IDLE 轮询不对称——触发类型切换是"有限时间生效"语义，驱动直接返回
3. **IWBEN 依赖固件**：probe 失败路径直接报 "IWB must be enabled in firmware"（:233）——GICv5 分层配置的又一个实例：固件管组件使能和域分配，驱动管 wire 级操作
4. **写 WTMR 的 PAS 约束**：驱动运行在 Non-secure，只能操作分配给 Non-secure 域的 wire——固件域分配错误时驱动静默失效（写被忽略）

---

## 七、ACPI probe deferral 系列演进（v1 → 已合入）

**解决的问题**：ACPI 下消费驱动可能先于 IWB 驱动 probe → 中断解析失败（wire 无法解析为 virq）。

| 版本 | 时间 | 内容 |
|------|------|------|
| v1 | 2026-05 | 2 patches：把 RISC-V 的 autodep 方案搬进通用 ACPI IRQ 层 |
| v2 | 2026-06 | 6 patches 拆分，Rafael 审阅 |
| v4 | 2026-07 | 7 patches，新增 `acpi_device_clear_deps()` |
| **已合入** | **2026-08-17** | `5a64611747687`（IORT 基础设施）+ `d2aa7b179a711`（IWB ACPI 探测排序）+ `5b4fa95e425b6`（注释修正） |

**方案本质**：自动建立"消费设备 → IWB（HID `ARMH0003`）"的 probe 依赖，IWB probe 完成后 `acpi_device_clear_deps()`（iwb.c:272）解除依赖。复用了 RISC-V 的 autodep 机制并下沉到通用 ACPI 层（`iort_iwb_handle`，iort.c:792；`gic_v5_get_gsi_handle`，irq-gic-v5.c:1229）。

---

## 八、与兄弟驱动对比

| | IWB | ITS | IRS |
|--|-----|-----|-----|
| 行数 | 301 | 1343 | 968 |
| 内存表 | 无 | DT/ITT | IST |
| 域模型 | MSI 设备域模板（msi_domain_template） | irq_domain 家族 | 无域（MMIO 窗口 + 指令） |
| 固件依赖 | **IWBEN + WDOMAINR 都是固件的** | 无 | 域分配是 EL3 的 |
| 独有机制 | FIXED_MSG_DATA + WIRED_TO_MSI | — | — |
| 状态管理 | 无（事件生成即职责全部） | 缓存失效协议 | 状态 + 路由 + 门铃 |

**一句话**：IWB 驱动是 GICv5 家族里的"翻译前端"——用 301 行代码把一个"wire 事件发生器"包装成 MSI 设备域，靠 `MSI_ALLOC_FLAGS_FIXED_MSG_DATA` 复用整个 ITS/LPI 投递管道；它的所有复杂性都外包给了三个对象：固件（IWBEN/域分配）、ITS（翻译表）、MSI core（固定消息语义）。

---

*笔记时间：2026-08-27*
*内核版本：master (fc46aed51f6)*
*证据文件：drivers/irqchip/irq-gic-v5-iwb.c（全文）、irq-gic-its-msi-parent.c、irq-gic-v5-its.c、kernel/irq/msi.c、include/linux/irqchip/arm-gic-v5.h、DT binding arm,gic-v5-iwb.yaml、规范 Chapter 6 / 6.1 / 6.2*

## 参考链接

**内核代码：**
- [irq-gic-v5-iwb.c — IWB 驱动全文](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/irqchip/irq-gic-v5-iwb.c)
- [irq-gic-its-msi-parent.c — its_v5_pmsi_prepare / gic_v5_its_msi_parent_ops](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/irqchip/irq-gic-its-msi-parent.c)
- [arm,gic-v5-iwb.yaml — DT 绑定（msi-parent cell = DeviceID）](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/devicetree/bindings/interrupt-controller/arm,gic-v5-iwb.yaml)

**规范：**
- [Arm GICv5 Architecture Specification (AES0070) — Chapter 6 IWB](https://developer.arm.com/documentation/aes0070)
- [Learn the Architecture: GICv5 Overview (111471) — §5.2.1 IWB](https://developer.arm.com/documentation/111471)

**邮件列表（ACPI deferral 系列）：**
- [irqchip/ACPI: Arm GICv5 IWB ACPI IRQ probe deferral v4（已合入）— patchew](https://patchew.org/linux/20260709-gic-v5-acpi-iwb-probe-deferral-v4-0-48dae790f871@kernel.org/)
- [ACPI/IORT: Implement ACPI infrastructure to enable GICv5 IWB probe deferral — lkml](https://lkml.iu.edu/hypermail/linux/kernel/2606.0/05544.html)

**关联笔记：**
- [GICv5 内存表与条目创建时间线（IWB 触发 DTE/ITT 创建的环节）](gicv5-memory-tables-note.md)
- [GICv5 架构与实现笔记（2.1 Step 1b IWB 路径）](gic-v5-arch-note.md)
