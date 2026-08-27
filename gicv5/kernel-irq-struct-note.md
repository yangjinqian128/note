# 内核中断子系统数据结构全景（以 GICv5 为例）

> 完整梳理中断子系统的结构体家族：骨架四件套 + irq core 配套 + MSI 层 + GICv5 驱动私有 + KVM vgic 虚拟层。
> 内核主线 master (fc46aed51f6)。

---

## 一、全景图：六层结构与关系

```
层 0  骨架四件套
      ┌────────────┐         ┌────────────┐
      │ irq_desc   │─handle_irq→ 流控函数   │  handle_fasteoi_irq
      │  ·内嵌─────┼──► ┌────┴───────┐    │  handle_percpu_irq ...
      │  ·action───┼─┐  │ irq_data   │    │
      └────────────┘ │  │  .chip ────┼────┼──► ┌─ irq_chip ──────────────┐
                     │  │  .domain ──┼────┼──► │ 操作手册（每类一份）       │
                     │  │  .chip_data┼───┐│    │ mask/unmask/eoi/set_type/ │
                     │  │  .parent───┼─┐ ││    │ set_affinity/ipi_send...  │
                     │  └────────────┘ │ ││    └───────────────────────────┘
                     │                 │ ││
层 1  irq core 配套   │                 │ │└──► ┌─ irq_domain ─────────────┐
      ┌───────────┐  │                 │ │     │ 翻译官+分配器             │
      │ irqaction │◄─┘                 │ │     │ translate/alloc/hierarchy│
      │ (用户链)   │                    │ └────►└──────────────────────────┘
      ├───────────┤                    │
      │ common_data│◄── 内嵌于 desc ───┼── .msi_desc ──► ┌─────────────────┐
      └───────────┘                    │                 │ 层 2 MSI 子层    │
      irq_fwspec ──translate 输入──► domain               │ msi_desc/msi_msg│
      irq_affinity_desc / notify                            └─────────────────┘
                              │
层 3  GICv5 驱动私有           ▼ (chip_data 指向)
      gicv5_irs_chip_data（SPI）    gicv5_its_dev（MSI）    iaffid_entry（per-CPU）
层 4  KVM vgic 虚拟层
      vgic_irq（每虚拟中断）  vgic_v5_cpu_if（per-vCPU，内嵌 gicv5_vpe）
层 5  对照层（GICv4 的 its 家族——GICv5 已删除大半）
      its_node/its_device/its_vm/its_vpe/rdists...
```

**读图钥匙**：实线 = 指针引用；"内嵌" = 结构体内含另一个结构体；层 0 是骨架，层 1-5 是骨架挂载的配套。

---

## 二、骨架四件套

### 2.1 irq_desc：一条 Linux 中断线的户口本

```c
// include/linux/irqdesc.h:81
struct irq_desc {
	struct irq_common_data	irq_common_data;   // 公共状态（见 3.1）
	struct irq_data		irq_data;          // ★ 内嵌——与 data 永远 1:1
	unsigned int __percpu	*kstat_irqs;       // /proc/interrupts 数据源
	irq_flow_handler_t	handle_irq;        // ★ 流控函数
	struct irqaction	*action;           // ★ 用户 handler 链（见 3.2）
	unsigned int		depth;             // 嵌套 disable 计数
	raw_spinlock_t		lock;
	const char		*name;             // chip->name 拷贝
};
```

**两个 "handler" 不要混淆**：`handle_irq` = 流控函数（chip/domain 层决定"怎么处理"：何时 mask、何时 EOI）；`action->handler` = 用户函数（request_irq 注册，干设备的事）。

### 2.2 irq_data：硬件侧名片夹

```c
// include/linux/irq.h:182
struct irq_data {
	u32			mask;
	unsigned int		irq;             // virq
	unsigned long		hwirq;           // ★ 硬件号——chip 只认它
	struct irq_common_data	*common;
	struct irq_chip		*chip;           // → 操作手册
	struct irq_domain	*domain;         // → 归属域
	void			*chip_data;      // → 驱动私有（层 3）
	struct irq_data		*parent_data;    // → 父层（层级链）
	struct cpumask		*affinity;       // 期望/实际亲和性
	struct cpumask		*effective_affinity;
};
```

**hwirq 因域而异的五种含义**（GICv5）：

| 域 | hwirq 含义 |
|----|-----------|
| PPI 域 | PPI 号 0-127 |
| SPI 域 | SPI INTID |
| LPI 域 | LPI INTID（IDA 动态分配） |
| IPI 域 | 域内序号 i（0..GICV5_IPIS_PER_CPU-1） |
| ITS MSI 域 | `device_id << 32 \| event_id` 打包（its.c:958 注释原文） |

### 2.3 irq_chip：操作手册（每类中断一份）

```c
// include/linux/irq.h:499
struct irq_chip {
	const char	*name;
	void	(*irq_mask)(struct irq_data *d);       // GICv5: CDDIS + GSB SYS
	void	(*irq_unmask)(struct irq_data *d);     // GICv5: CDEN
	void	(*irq_eoi)(struct irq_data *d);        // GICv5: CDDI
	int	(*irq_set_type)(struct irq_data *d, unsigned int type);
	int	(*irq_set_affinity)(struct irq_data *d, ...);
	int	(*irq_retrigger)(struct irq_data *d);   // GICv5: CDPEND(P=1)
	int	(*irq_get_irqchip_state)(...);         // GICv5: CDRCFG 读回
	int	(*irq_set_irqchip_state)(...);         // GICv5: CDPEND
	void	(*ipi_send_single)(struct irq_data *d, unsigned int cpu);
	void	(*ipi_send_mask)(struct irq_data *d, const struct cpumask *dest);
	unsigned long	flags;                        // IRQCHIP_SET_TYPE_MASKED 等
};
```

GICv5 四个 chip 实例：

| chip 实例 | 中断类 | 特征 |
|-----------|--------|------|
| `gicv5_spi_irq_chip` (:541) | SPI | set_type 走 IRS MMIO，其余 CDxx 指令 |
| `gicv5_lpi_irq_chip` (:568) | LPI | 全 CDxx 指令，无 MMIO |
| `gicv5_ppi_irq_chip` | PPI | 系统寄存器 ICC_PPI_*，EOI 走 CDDI |
| `gicv5_ipi_irq_chip` (:558) | IPI | 只有 ipi_send_single，mask/unmask/eoi 委托父层 |

**层级委托**：`irq_chip_mask_parent` / `irq_chip_retrigger_hierarchy`——沿 `parent_data` 链向上找能干的层。`gicv5_ipi_send_single`（:475）→ `irq_chip_retrigger_hierarchy` → LPI chip 的 `irq_retrigger` → CDPEND。

### 2.4 irq_domain：翻译官 + 分配器

```c
// include/linux/irqdomain.h:168
struct irq_domain {
	const struct irq_domain_ops *ops;    // 方法表（见 3.4）
	struct irq_domain	*parent;     // 父域
	struct fwnode_handle	*fwnode;     // DT/ACPI 节点
	enum irq_domain_bus_token bus_token; // DOMAIN_BUS_WIRED 等
	...                                  // hwirq↔virq 反向映射（数组/基数树）
};
```

**三种创建方式，GICv5 恰好各有一个**：

| 创建函数 | 语义 | GICv5 实例 |
|---------|------|-----------|
| `create_linear` | 连续 hwirq，数组反查 | PPI（:1046, 128 个）、SPI（:1054, spi_count 个） |
| `create_tree` | 稀疏 hwirq，基数树反查 | LPI（:849，INTID 动态分配） |
| `create_hierarchy` | 挂父域，alloc 向上委托 | IPI（:1068, parent=lpi_domain）、ITS MSI 域 |

---

## 三、irq core 配套层

### 3.1 irq_common_data：desc 与 data 的公共状态

```c
// include/linux/irq.h（irq_desc 内嵌、irq_data 指针引用）
struct irq_common_data {
	unsigned int		__private state_use_accessors;  // IRQD_* 状态位
	unsigned int		node;                          // NUMA 节点
	void			*handler_data;                  // 流控函数私有数据
	struct msi_desc		*msi_desc;     // ★ MSI 中断 → 描述符（层 2）
	cpumask_var_t		affinity;      // 期望亲和性
	cpumask_var_t		effective_affinity;  // 实际亲和性
};
```

**要点**：`state_use_accessors` 是 IRQD_* 标志位集合（如 IRQD_AFFINITY_ON_ACTIVATE），只能经 `irqd_*` 访问器读写；`msi_desc` 是 desc 通向 MSI 子层的桥。

### 3.2 irqaction：用户 handler 链节点

```c
// include/linux/interrupt.h:123
struct irqaction {
	irq_handler_t		handler;       // 主 handler（hardirq 上下文）
	union {
		void		*dev_id;       // 用户 cookie（共享中断时靠它区分）
		void __percpu	*percpu_dev_id; // per-CPU cookie（IRQF_PERCPU）
	};
	const struct cpumask	*affinity;     // 用户侧亲和性约束
	struct irqaction	*next;         // ★ 链——共享中断多条 action
	irq_handler_t		thread_fn;     // 线程化 handler
	struct task_struct	*thread;       // 内核线程（thread_fn 跑这里）
	unsigned int		irq;           // 所属 virq
	unsigned int		flags;         // IRQF_*
	const char		*name;         // 显示名
	struct proc_dir_entry	*dir;
};
```

**要点**：`next` 链支撑 IRQF_SHARED；`dev_id` 是 handler 第二参数（`action->handler(irq, action->dev_id)`，kernel/irq/handle.c:206）——doorbell 场景 vcpu 指针就是塞在这里。`free_irq(irq, dev_id)` 靠 dev_id 在链上定位自己的 action。

### 3.3 irq_fwspec：固件中断描述的载体

```c
// include/linux/irqdomain.h:41
struct irq_fwspec {
	struct fwnode_handle *fwnode;        // 设备节点
	int param_count;                     // 3（GICv5 DT 3-cell）
	u32 param[IRQ_DOMAIN_IRQ_SPEC_PARAMS]; // [类型, INTID, 触发标志]
};
```

DT 的 `interrupts = <GIC_SPI 42 IRQ_TYPE_LEVEL_HIGH>` 解析成 fwspec 后交给 domain 的 `translate`（irq-gic-v5.c:581）产出 hwirq + type。**fwspec 是"固件世界 → 内核世界"的入口数据结构。**

### 3.4 irq_domain_ops：domain 的方法表

```c
// include/linux/irqdomain.h:99
struct irq_domain_ops {
	int (*match)(...);      // 匹配 fwnode 与 domain
	int (*select)(...);     // ★ "这个 fwspec 归不归我管"（gicv5_irq_spi_domain_select :751）
	int (*map)(...);        // 老式映射接口
	int (*alloc)(...);      // ★ 分配 virq + 填 irq_data
	void (*free)(...);
	int (*activate)(...);   // ★ 写硬件表（ITS: 写 ITTE）
	void (*deactivate)(...);
	int (*translate)(...);  // ★ fwspec → hwirq + type
};
```

### 3.5 亲和性三兄弟

```
irq_data.affinity          → 期望亲和性（用户写 smp_affinity 的落点）
irq_data.effective_affinity → 实际生效亲和性（chip 在 set_affinity 里更新）
irq_affinity_desc          → MSI 多向量分布描述（managed 中断 + 各向量 mask，include/linux/interrupt.h）
irq_affinity_notify        → /proc/irq/N/smp_affinity 变更通知（用户态监听机制）
```

### 3.6 irq_devres：devm 自动释放包装

```c
// kernel/irq/devres.c
struct irq_devres {
	unsigned int irq;
	void *dev_id;
};
```

`devm_request_irq` 的私有记录——设备 remove 时自动 `free_irq`，不需要驱动手动清理。

---

## 四、MSI 子层：套在 irq_domain 之上的语义层

### 4.1 msi_desc：每 MSI 中断的档案

```c
// include/linux/msi.h
struct msi_desc {
	unsigned int			irq;         // virq（与 irq_data.irq 对应）
	unsigned int			nvec_used;
	struct device			*dev;        // 所属设备
	struct msi_msg			msg;         // ★ 要写进设备 MSI-X 表的消息
	struct irq_affinity_desc	*affinity;
	void (*write_msi_msg)(struct msi_desc *entry, void *data);
	void *write_msi_msg_data;              // 平台特定写表回调
	u16				msi_index;
	union {
		struct pci_msi_desc	pci;       // PCI 特有（msi_attrib、mask_pos...）
		struct msi_desc_data	data;      // 平台 MSI 特有
	};
};
```

**桥接关系**：`irq_common_data->msi_desc` 指到它；`msi_desc->irq` 指回 virq——desc 与 msi_desc 双向可达。

### 4.2 msi_msg：写进设备硬件的三字节

```c
// include/linux/msi.h
struct msi_msg {
	u32 address_lo;   // MSI 目标地址低 32 位
	u32 address_hi;
	u32 data;         // MSI 数据（GICv5: 就是 EventID）
};
```

GICv5 的填充（its.c:717 `gicv5_its_compose_msi_msg`）：`msg->data = EventID`，`msg->address = its_trans_phys_base`（ITS 翻译寄存器物理地址）。**msi_msg 是内核与设备硬件之间的最终字节契约。**

### 4.3 msi_domain_info / msi_domain_ops

```c
// include/linux/msi.h
struct msi_domain_ops {
	msi_alloc_info_t *(*get_hwirq)(...);      // 分配 hwirq 的钩子
	int (*msi_prepare)(...);                  // ★ GICv5: 设备注册（its_dev）
	void (*set_desc)(...);
	int (*domain_alloc_irqs)(...);
	int (*domain_free_irqs)(...);
};
struct msi_domain_info {
	unsigned long	flags;
	const struct msi_domain_ops *ops;
	struct irq_chip *chip;                    // 顶层 chip
	void *chip_data;                          // 顶层 chip_data
	...
};
```

MSI 层是**给通用 irq_domain 加的 MSI 语义外衣**：`msi_domain_alloc_irqs` 先调 `msi_domain_ops`（prepate/alloc），再落入底层 irq_domain 的 alloc/activate。GICv5 里 `gicv5_its_msi_prepare` 创建 `gicv5_its_dev` 就是这一层干的。

---

## 五、GICv5 驱动私有层（chip_data 家族）

### 5.1 gicv5_irs_chip_data：IRS 实例（SPI 的 chip_data）

```c
// include/linux/irqchip/arm-gic-v5.h:324
struct gicv5_irs_chip_data {
	struct list_head	entry;           // irs_nodes 链表
	struct fwnode_handle	*fwnode;
	void __iomem		*irs_base;      // ★ IRS MMIO 配置帧基地址
	struct resource		res;
	u32			flags;
	u32			spi_min;         // 本 IRS 管理的 SPI 范围（IDR7/IDR6）
	u32			spi_range;
	raw_spinlock_t		spi_config_lock; // ★ SELR 选择窗口的串行锁
};
```

每 IRS 一个实例，挂在全局 `irs_nodes` 链表；SPI 中断的 `chip_data` 指向它（irq-gic-v5.c:731），`gicv5_spi_irq_set_type` 靠它拿 `irs_base` 走 select-write-poll 协议。配套 per-CPU 变量 `per_cpu_irs_data`（irs.c:353）记录"本 CPU 连的 IRS"。

### 5.2 gicv5_its_chip_data + gicv5_its_dev：ITS 的两级视图

```c
// drivers/irqchip/irq-gic-v5-its.c:27 / :38
struct gicv5_its_chip_data {                 // ITS 节点级（每 ITS 一个）
	struct xarray			its_devices;    // ★ DeviceID → its_dev 索引
	struct mutex			dev_alloc_lock;
	struct gicv5_its_devtab_cfg	devtab_cfgr;    // DT 基址/结构/L2SZ
	void __iomem			*its_base;      // ITS MMIO 基地址
	u32				flags;
	unsigned int			msi_domain_flags;
};

struct gicv5_its_dev {                       // 设备级（MSI 中断的 chip_data）
	struct gicv5_its_chip_data	*its_node;  // 回指所属 ITS
	struct gicv5_its_itt_cfg	itt_cfg;    // ITT 基址/结构/大小
	unsigned long			*event_map; // ★ EventID 位图（分配器）
	u32				device_id;
	u32				num_events;
	phys_addr_t			its_trans_phys_base; // ★ MSI 写目标（compose 用它）
};
```

**两级职责**：节点级管"这个 ITS 的硬件与设备集合"；设备级管"这个 PCIe 设备的事件位图和 ITT"——`gicv5_its_compose_msi_msg` 从 `its_dev->its_trans_phys_base` + event_map 分配的 EventID 合成 msi_msg。

### 5.3 iaffid_entry：CPU ↔ IAFFID 映射（per-CPU）

```c
// drivers/irqchip/irq-gic-v5-irs.c:368
struct iaffid_entry {
	u16	iaffid;    // 本 CPU 的 16 位 PE 中断亲和 ID
	bool	valid;
};
static DEFINE_PER_CPU(struct iaffid_entry, cpu_iaffid);
```

固件表（DT `arm,iaffids` / ACPI MADT GICC）填充；`gicv5_irs_cpu_to_iaffid()` 查它——所有 `CDAFF` 操作（set_affinity、hwirq_init 默认路由）都从这里拿 IAFFID。

---

## 六、KVM vgic 虚拟层

### 6.1 vgic_irq：每虚拟中断的状态机本体

```c
// include/kvm/arm_vgic.h:233
struct vgic_irq {
	raw_spinlock_t irq_lock;      // 保护整个结构
	u32 intid;                    // Guest 可见 INTID
	struct rcu_head rcu;
	struct list_head ap_list;     // ★ 挂 AP 列表（待注入队列）

	struct kvm_vcpu *vcpu;        // SGI/PPI: 所属 vcpu；SPI/LPI: ap_list 宿主
	struct kvm_vcpu *target_vcpu; // ★ 路由目标（affinity 解析结果）

	bool pending_latch:1;         // ★ 软件 pending 锁存（edge 注入的主载体）
	enum vgic_irq_config config:1; // Level or edge
	bool line_level:1;            // Level 型的实时电平
	bool enabled:1;
	bool active:1;
	bool hw:1;                    // ★ 直通标记（DVI/VSGI/GICv4 硬件注入）
	bool on_lr:1;                 // 已在某个 List Register 中
	refcount_t refcount;
	u32 hwintid;                  // 直通对应的硬件 INTID
	unsigned int host_irq;        // 直通对应的宿主 virq
	u32 mpidr;                    // v3 亲和性目标
	u8 priority;
	u8 group;
	const struct irq_ops *ops;    // ★ v5 的 PPI ops（set_direct_injection 等）
	void *owner;                  // 设备归属（reserve 用）
};
```

**这是虚拟侧与物理侧镜像的枢纽**：`hw=0` 时 pending 由 `pending_latch/line_level` 软件维护（GICv3 LR 路径）；`hw=1` 时硬件直通（GICv5 PPI 的 DVI、GICv4.1 的 vSGI），`host_irq/hwintid` 建立与宿主中断的绑定。

### 6.2 vgic_v5_cpu_if：per-vCPU 的 GICv5 影子状态

```c
// include/kvm/arm_vgic.h:475
struct vgic_v5_cpu_if {
	u64	vgic_apr;      // ICH_APR_EL2 影子
	u64	vgic_vmcr;     // ICH_VMCR_EL2 影子

	/* PPI 寄存器状态（64 位位图，与 ICH_PPI_* 寄存器一一对应） */
	DECLARE_BITMAP(vgic_ppi_dvir, VGIC_V5_NR_PRIVATE_IRQS);    // DVI 位图
	DECLARE_BITMAP(vgic_ppi_activer, VGIC_V5_NR_PRIVATE_IRQS); // Active
	DECLARE_BITMAP(vgic_ppi_enabler, VGIC_V5_NR_PRIVATE_IRQS); // Enable
	u64	vgic_ppi_priorityr[VGIC_V5_NR_PRIVATE_IRQS / 8]; // Priority

	u64	vgic_icsr;     // ICC_ICSR_EL1 影子（host/guest 复用，hyp 保存）

	struct gicv5_vpe gicv5_vpe;  // ★ 内嵌——GICv5 的 VPE 抽象
};
```

所在位置：`struct vgic_cpu` 的 union（arm_vgic.h:505）——`vgic_v2 / vgic_v3 / vgic_v5` 三选一，按 vgic 版本互斥。**这个结构就是之前 vPPI 笔记里 DVI 位图、save/restore 流程的"影子家"**：guest 运行期间寄存器归 guest，切出时读到这些影子，切入时写回。

### 6.3 gicv5_vpe：整个 VPE 就一个 bool

```c
// include/linux/irqchip/arm-gic-v5.h:393
struct gicv5_vpe {
	bool			resident;
};
```

对比 GICv4 的 `struct its_vpe`（include/linux/irqchip/arm-gic-v4.h：vpe_id、vpe_table_mask、sgi_config、sgi_domain、vpt...）——**GICv5 把 VPE 的全部迁移相关状态收敛为一个 resident 布尔**，这是"Residency 取代 VMOVP"（见 gicv5-vmovp-analysis.md）在数据结构上的终极体现。

---

## 七、对照层：GICv4 的 its 家族（GICv5 删除清单）

| GICv4 结构 | 职责 | GICv5 去向 |
|-----------|------|-----------|
| `its_node` | ITS 实例（命令队列 CMDQ、CBASER） | 退化为 `gicv5_its_chip_data`（无命令队列） |
| `its_device` | 设备视图（itt 基址、事件位图） | `gicv5_its_dev`（保留核心字段） |
| `its_collection` | 目标 RD 集合 | 删除（无 RD 概念） |
| `its_cmd_block` | 命令队列块 | 删除（无异步命令，写寄存器 + 轮询 IDLE） |
| `its_vm` | VM 的 vPE 集合 + vmapp_lock | 删除（VM 状态进 IRS 的 VM Table） |
| `its_vpe` | VPE（vpt、doorbell、col_idx） | 收敛为 `gicv5_vpe { bool resident }` |
| `rdists` | Redistributor 集合 | 删除（无 RD，IRS 取代） |

这张删除清单本身就是一篇架构对比：**GICv4 的复杂度集中在"ITS 命令队列 + VPE 状态机"，GICv5 把前者换成"寄存器写 + IDLE 轮询"、后者换成"共享内存表 + residency 布尔"。**

---

## 八、关系总表：谁嵌入谁、谁指向谁

| 关系 | 类型 | 说明 |
|------|------|------|
| irq_desc ⊃ irq_data | 内嵌 | 1:1，户口本贴名片 |
| irq_desc ⊃ irq_common_data | 内嵌 | 公共状态 |
| irq_data.common → common_data | 指针 | 与上同一份 |
| irq_desc.action → irqaction 链 | 指针 | 用户 handler（可多条，共享中断） |
| irq_desc.handle_irq → 流控函数 | 函数指针 | chip/domain 层设置 |
| irq_data.chip → irq_chip | 多对一 | 一类中断一份手册 |
| irq_data.domain → irq_domain | 多对一 | 归属域 |
| irq_data.chip_data → 层 3 私有 | 指针 | SPI→irs_data；MSI→its_dev |
| irq_data.parent_data → 父 irq_data | 指针 | 层级链（IPI→LPI） |
| irq_domain.parent → 父 domain | 指针 | 层级链 |
| irq_domain.fwnode → DT/ACPI 节点 | 指针 | 静态域的固件锚点 |
| common_data.msi_desc → msi_desc | 指针 | 中断通向 MSI 层的桥 |
| msi_desc.msg → msi_msg | 内嵌 | 写进设备的字节 |
| msi_desc.dev → device | 指针 | 所属设备 |
| msi_desc.irq ↔ irq_data.irq | 同号 | virq 双向对应 |
| irqaction.dev_id → 用户 cookie | 指针 | handler 第二参数 |
| vgic_irq.target_vcpu → kvm_vcpu | 指针 | 虚拟路由目标 |
| vgic_irq.host_irq ↔ 宿主 irq_data | 同号 | hw=1 直通绑定 |
| vgic_irq.ap_list → 队列 | 链表 | 待注入队列 |
| vgic_cpu ⊃ vgic_v5_cpu_if | union 内嵌 | v2/v3/v5 三选一 |
| vgic_v5_cpu_if ⊃ gicv5_vpe | 内嵌 | VPE 抽象 |
| gicv5_its_dev → gicv5_its_chip_data | 指针 | 设备回指节点 |
| gicv5_its_chip_data ⊃ xarray | 内嵌 | DeviceID → its_dev 索引 |
| per_cpu cpu_iaffid / per_cpu_irs_data | per-CPU | CPU 本地拓扑缓存 |

---

## 九、数据流实例：结构体如何在两条路径上流转

### 9.1 MSI 注册路径

```
pci_alloc_irq_vectors()
  └─ msi_domain_alloc_irqs()                    ← 层 2 msi_domain_info/ops
       ├─ msi_prepare → 创建 gicv5_its_dev       ← 层 3
       │     └─ xa_store(its_devices, device_id)  ← 层 3 节点级索引
       ├─ domain alloc（ITS MSI 域）→ parent（LPI 域）
       │     ├─ ida_alloc LPI INTID               ← hwirq 诞生
       │     └─ irq_domain_set_info(...)          ← 填 irq_data{chip, chip_data}
       │          + irq_alloc_descs               ← irq_desc 诞生（内嵌 data）
       ├─ activate → 写 ITTE                       ← 硬件表落盘
       └─ compose_msi_msg → msi_msg{addr, data}   ← 层 2 产物
              → 写设备 MSI-X 表                    ← 硬件侧契约完成
```

### 9.2 中断到达路径

```
gicv5_handle_irq: CDIA（返回 hwirq 编码）
  └─ generic_handle_domain_irq(domain, id)
       └─ irq_desc[virq].handle_irq(d)          ← 流控（handle_fasteoi_irq）
            ├─ handle_irq_event(desc)
            │    └─ for_each action 链:
            │         action->handler(irq, action->dev_id)   ← 用户 handler
            └─ d->chip->irq_eoi(d)               ← gicv5_lpi_irq_eoi → CDDI
```

### 9.3 vPPI 直通的虚拟侧流转（层 4 全景）

```
map: kvm_vgic_map_phys_irq → irq->hw=true, irq->host_irq=... → ops->set_direct_injection
     → vgic_v5_set_ppi_dvi: 置 vgic_v5_cpu_if.vgic_ppi_dvir 位图   ← 影子
VPE 切入: restore → 位图写 ICH_PPI_DVIR0_EL2                        ← 硬件落盘
运行中: 物理 pending → DVI 硬件 → vPPI pending → vIRQ
VPE 切出: save → ICH_PPI_* 读回 vgic_v5_cpu_if 位图 + 清零 DVIR
```

---

## 十、常见困惑点澄清

1. **chip_data vs dev_id**：chip_data = chip 回调的私有数据（irs_data 的 MMIO 基地址，domain alloc 设置）；dev_id = 用户 handler 的 cookie（request_irq 传入）。毫无关系。
2. **两个 handler**：`desc->handle_irq`（flow，管"怎么处理"）vs `action->handler`（用户，管"干什么"）。
3. **hwirq 不是全局唯一的**：同一数值在不同 domain 语义不同；全局唯一标识是 virq。
4. **startup vs unmask**：startup 首次使能（可含芯片初始化），unmask 纯解除屏蔽。
5. **hierarchy 域的 hwirq 层层重译**：ITS MSI 域 hwirq = 打包(device,event)，parent LPI 域 hwirq = LPI INTID。
6. **pending 的三种影子**：物理侧在硬件（SPI 在 IRS 内部/LPI 在 IST/PPI 在寄存器）；虚拟侧在 `vgic_irq.pending_latch`（软件）或硬件直通（hw=1）；两者以 vgic_irq.hw 为界划分权威。
7. **msi_desc 与 irq_data 的双向**：common_data->msi_desc 下行、msi_desc->irq 上行——查"这个中断是 MSI 吗"从 desc 出发，查"这个 MSI 写什么消息"从 msi_desc 出发。

---

*笔记时间：2026-08-27*
*内核版本：master (fc46aed51f6)*
*证据文件：include/linux/irq.h、irqdesc.h、irqdomain.h、interrupt.h、msi.h、include/kvm/arm_vgic.h、include/linux/irqchip/arm-gic-v5.h、arm-gic-v4.h、drivers/irqchip/irq-gic-v5.c、irq-gic-v5-its.c、irq-gic-v5-irs.c、kernel/irq/handle.c、devres.c*

## 参考链接

**内核文档：**
- [Linux generic IRQ handling — kernel.org](https://www.kernel.org/doc/html/latest/core-api/genericirq.html)
- [irq_domain 文档 — kernel.org](https://www.kernel.org/doc/html/latest/core-api/irq/irq-domain.html)
- [MSI 与 irqdomain 关系（Thomas Gleixner 的 irqdomain 文档注释）](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/irqdomain.h)

**内核代码：**
- [include/linux/irq.h — irq_data / irq_chip / irq_common_data](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/irq.h)
- [include/linux/irqdesc.h — irq_desc](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/irqdesc.h)
- [include/linux/irqdomain.h — irq_domain / irq_domain_ops / irq_fwspec](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/irqdomain.h)
- [include/linux/msi.h — msi_desc / msi_msg / msi_domain_info](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/msi.h)
- [include/kvm/arm_vgic.h — vgic_irq / vgic_v5_cpu_if / vgic_cpu](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/kvm/arm_vgic.h)
- [irq-gic-v5-its.c — gicv5_its_chip_data / gicv5_its_dev](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/irqchip/irq-gic-v5-its.c)

**关联笔记：**
- [GICv5 架构与实现笔记](gic-v5-arch-note.md)（流程视角，与本篇结构视角互补）
- [GICv5 VMOVP 分析（its_vpe 家族为何被收敛为 bool resident）](gicv5-vmovp-analysis.md)
