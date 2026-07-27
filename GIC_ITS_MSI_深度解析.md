# Linux 内核 GIC ITS MSI 源码深度解析

> 基于 Linux 内核源码（`drivers/irqchip/irq-gic-v3-its.c` 等），综合 ARM64 GICv3/v4 架构背景，深入剖析 ITS（Interrupt Translation Service）的 MSI 中断子系统设计与实现。

---

## 一、核心数据结构对照表

数据结构是理解内核源码的"骨架"。本节以 `drivers/irqchip/irq-gic-v3-its.c` 中的核心结构体为切入点，梳理它们在软件层面的角色与组织关系。

### 1.1 struct its_node — ITS 控制器抽象

```c
// irq-gic-v3-its.c:103
struct its_node {
    raw_spinlock_t          lock;               // 保护 ITS 共享数据结构
    struct mutex            dev_alloc_lock;     // 串行化设备分配操作
    struct list_head        entry;              // 挂载到全局 its_nodes 链表
    void __iomem            *base;              // ITS 寄存器基地址（ioremap）
    void __iomem            *sgir_base;         // GICv4.1 SGIR 寄存器基地址
    phys_addr_t             phys_base;          // ITS 物理基地址
    struct its_cmd_block    *cmd_base;          // 命令队列基地址（环形缓冲区）
    struct its_cmd_block    *cmd_write;         // 命令队列写指针
    struct its_baser        tables[GITS_BASER_NR_REGS]; // 硬件表（Device/Collection/vPE 等）
    struct its_collection   *collections;       // Per-CPU Collection 数组
    struct fwnode_handle    *fwnode_handle;     // 固件节点句柄（OF/ACPI）
    u64                     (*get_msi_base)(struct its_device *); // 计算 MSI 地址
    u64                     typer;              // GITS_TYPER 寄存器缓存
    u64                     cbaser_save;        // 命令队列基地址备份（电源管理）
    u32                     ctlr_save;          // 控制寄存器备份
    u32                     mpidr;              // GICv4.1 MPIDR
    struct list_head        its_device_list;    // 挂载在此 ITS 下的设备链表
    u64                     flags;              // 特性标志位
    unsigned long           list_nr;            // GICv4 ITS List 编号
    int                     numa_node;          // NUMA 节点
    unsigned int            msi_domain_flags;   // MSI 域标志
    u32                     pre_its_base;       // Synquacer 平台前级 ITS 基地址
    int                     vlpi_redist_offset; // vLPI Redistributor 偏移
};
```

| 成员 | 角色说明 |
|------|----------|
| `phys_base` | ITS 在物理地址空间的基址，用于向硬件写入 GITS_CBASER 等寄存器 |
| `cmd_base / cmd_write` | 命令队列（Circular Buffer）的基址与写指针。CPU 向队列填充命令，ITS 硬件从队列读取并执行。队列大小由 `ITS_CMD_QUEUE_SZ`（通常 64KB）决定 |
| `tables[]` | 对应硬件 GITS_BASERn 寄存器管理的各类表（Device Table、Collection Table、vPE Table 等）。每个 `its_baser` 描述一张表的内存位置、大小和属性 |
| `collections` | Collection 是物理 CPU 在 ITS 中的代理，每个 CPU 分配一个 Collection，`col_map[]` 中的索引即 Collection ID |
| `its_device_list` | 所有挂载在此 ITS 下的 `its_device` 通过 `entry` 链表成员串联起来 |
| `get_msi_base` | 函数指针，返回设备应向哪个地址写 MSI 消息（即 `GITS_TRANSLATER` 地址） |

**组织关系：**

```
全局链表 its_nodes
    |
    +-- its_node #1
    |       +-- its_device_list --> its_device(A) --> its_device(B) --> ...
    |       +-- collections[0..NR_CPUS-1]
    |       +-- cmd_base --> [cmd_block][cmd_block][...][cmd_block]  (环形队列)
    |       +-- tables[] --> Device Table / Collection Table / vPE Table ...
    |
    +-- its_node #2
            +-- ...
```

### 1.2 struct its_baser — BASER 表描述符

```c
// irq-gic-v3-its.c:85
struct its_baser {
    void    *base;      // 表内存基地址
    u64     val;        // 缓存的 GITS_BASERn 寄存器值
    u32     order;      // 分配页数（order of pages）
    u32     psz;        // 二级表页大小（间接模式时使用）
};
```

ITS 硬件通过 GITS_BASERn 寄存器维护若干硬件表，每张表可以是**扁平表**（Flat Table）或**两级表**（Indirect Table）。`its_baser` 是软件对每张表的描述：

- **Device Table**（`GITS_BASER_TYPE_DEVICE`）：以 DeviceID 为索引，每项指向该设备的 ITT（Interrupt Translation Table）
- **Collection Table**（`GITS_BASER_TYPE_COLLECTION`）：以 Collection ID 为索引，每项指向 Redistributor 的物理地址

### 1.3 struct event_lpi_map — LPI 事件映射

```c
// irq-gic-v3-its.c:153
struct event_lpi_map {
    unsigned long       *lpi_map;     // LPI 号分配位图
    u16                 *col_map;     // EventID → CollectionID 映射表
    irq_hw_number_t     lpi_base;     // 分配的 LPI 起始号
    int                 nr_lpis;      // 分配的 LPI 数量
    raw_spinlock_t      vlpi_lock;    // vLPI 操作锁
    struct its_vm       *vm;          // GICv4：LPI 直通到的 VM
    struct its_vlpi_map *vlpi_maps;   // GICv4：vLPI 映射数组
    int                 nr_vlpis;     // GICv4：vLPI 数量
};
```

这是 `its_device` 的核心数据成员，记录了设备 EventID 与 LPI 之间的映射关系：

- `lpi_map`：位图，每一位表示一个 LPI 是否被分配。通过 `bitmap_find_free_region()` 查找空闲区域，`bitmap_release_region()` 释放
- `col_map`：`col_map[EventID] = CollectionID`，即每个 Event 路由到哪个 CPU。这是中断亲和性的核心数据结构
- `lpi_base`：该设备在全局 LPI 空间中的起始号。设备的 EventID N 对应的硬件 LPI 号为 `lpi_base + N`

### 1.4 struct its_device — ITS 视图中的外设

```c
// irq-gic-v3-its.c:170
struct its_device {
    struct list_head    entry;           // 挂载到 its_node->its_device_list
    struct its_node     *its;            // 反向指针：所属 ITS 控制器
    struct event_lpi_map event_map;      // LPI 映射数据（见 1.3 节）
    void                *itt;            // ITT（Interrupt Translation Table）基址
    u32                 itt_sz;          // ITT 大小（字节）
    u32                 nr_ites;         // ITT Entry 数量（必须为 2 的幂）
    u32                 device_id;       // 设备的 DeviceID（由 BDF 或其他方式确定）
    bool                shared;          // 是否与其他设备共享 ITT（如 PCIe 别名）
};
```

**ITT（Interrupt Translation Table）** 是设备在 ITS 中的核心数据结构。硬件通过查表将 `(DeviceID, EventID)` 翻译为 `(LPI, CollectionID)`：

```
ITT 结构（硬件视图）:
  DeviceID ──→ Device Table ──→ ITT[EventID]
                                   ├── LPI Number
                                   └── Collection ID → Redistributor → CPU
```

**软件层面的事件流程：**
1. 设备向 `GITS_TRANSLATER` 写入 `EventID`
2. ITS 硬件查询 Device Table 找到设备的 ITT
3. 以 EventID 索引 ITTE（ITT Entry），取出 LPI 号和 Collection ID
4. ITS 向对应 Redistributor 注入 LPI

### 1.5 struct its_cmd_desc — ITS 硬件命令描述符

```c
// irq-gic-v3-its.c:434
struct its_cmd_desc {
    union {
        struct { struct its_device *dev; u32 event_id; }          its_inv_cmd;
        struct { struct its_device *dev; u32 event_id; }          its_clear_cmd;
        struct { struct its_device *dev; u32 event_id; }          its_int_cmd;
        struct { struct its_device *dev; int valid; }             its_mapd_cmd;
        struct { struct its_collection *col; int valid; }         its_mapc_cmd;
        struct { struct its_device *dev; u32 phys_id; u32 event_id; } its_mapti_cmd;
        struct { struct its_device *dev; struct its_collection *col; u32 event_id; } its_movi_cmd;
        struct { struct its_device *dev; u32 event_id; }          its_discard_cmd;
        struct { struct its_collection *col; }                    its_invall_cmd;
        // GICv4 扩展命令（略）...
    };
};
```

**联合体（union）设计的优势：**
- 每种命令类型仅携带必要的参数，避免空间浪费
- 传递给 `BUILD_SINGLE_CMD_FUNC` 宏生成的命令发送函数，通过 `its_cmd_builder_t` 回调构建硬件命令块

**主要 ITS 硬件命令对照表：**

| 命令宏 | 编码 | 用途 | `its_cmd_desc` 成员 | 使用场景 |
|--------|------|------|---------------------|----------|
| `GITS_CMD_MAPD` | 0x08 | 映射设备到 ITT | `its_mapd_cmd` | 设备初始化，建立 DeviceID ↔ ITT 映射 |
| `GITS_CMD_MAPTI` | 0x0A | 映射 EventID 到 LPI | `its_mapti_cmd` | 使能中断，建立 EventID ↔ LPI ↔ Collection 映射 |
| `GITS_CMD_MAPI` | 0x0B | 映射 EventID（不含 LPI） | 无直接对应 | 已废弃，由 MAPTI 取代 |
| `GITS_CMD_INV` | 0x0C | 无效化 LPI 配置缓存 | `its_inv_cmd` | 修改 LPI 属性后通知 ITS 刷新缓存 |
| `GITS_CMD_INVALL` | 0x0D | 无效化 Collection 所有 LPI | `its_invall_cmd` | 批量刷新（较少使用） |
| `GITS_CMD_INT` | 0x03 | 软件触发中断 | `its_int_cmd` | 测试 / 调试 / `irq_set_irqchip_state` |
| `GITS_CMD_CLEAR` | 0x04 | 清除中断 pending 状态 | `its_clear_cmd` | `irq_set_irqchip_state` |
| `GITS_CMD_DISCARD` | 0x0F | 取消映射 | `its_discard_cmd` | 设备释放中断时 |
| `GITS_CMD_MOVI` | 0x01 | 迁移中断到其他 Collection | `its_movi_cmd` | CPU 亲和性变更 |
| `GITS_CMD_SYNC` | 0x05 | 同步命令（等待前序命令完成） | （自动生成） | 每次命令发送后自动追加 |

### 1.6 struct its_cmd_block — 硬件命令块

```c
// irq-gic-v3-its.c:532
struct its_cmd_block {
    union {
        u64     raw_cmd[4];       // 4 × 64 位 = 32 字节（CPU 字节序）
        __le64  raw_cmd_le[4];    // Little-Endian 视图（ITS 硬件要求 LE）
    };
};
```

每次向命令队列写入 32 字节的命令块。`its_encode_*()` 函数族（如 `its_encode_cmd`、`its_encode_devid`、`its_encode_event_id` 等）通过 `its_mask_encode()` 将字段值编码到 `raw_cmd[0..3]` 的指定位域。发送前，`its_fixup_cmd()` 将 `raw_cmd` 转换为 LE 格式写入 `raw_cmd_le`。

---

## 二、理解 Irq Domain（中断域）的层级架构

Linux 内核通过 **层级中断域（Hierarchical Irq Domains）** 机制管理 GIC ITS MSI。这一设计与硬件流水线（ITS → Redistributor → CPU Interface）精确对应。

本节先阐述 irq_domain 子系统的设计理念（参考 [`Documentation/core-api/irq/irq-domain.rst`](https://www.kernel.org/doc/Documentation/IRQ-domain.txt)），再结合 ITS 源码说明各层职责。

### 2.1 irq_domain 设计理念：为什么需要这一层抽象

#### 2.1.1 核心问题：hwirq 与 Linux IRQ 号的分离

内核中每个中断源需要一个唯一的 Linux IRQ 号。早期系统中只有一根中断控制器，硬件 IRQ 线和 Linux IRQ 号可以直接一一对应。但在现代 SoC 中，多种中断控制器（GIC、GPIO、IOMMU 等）形成了**级联（Cascading）**结构：

```
设备 → GPIO 控制器 → GIC → CPU
        ↑              ↑
    (hwirq 0-31)   (hwirq 0-1023)
```

每个中断控制器都有自己局部的硬件中断号（hwirq），如果直接暴露给内核，会导致 IRQ 号冲突且无法追溯中断来源。`irq_domain` 的设计初衷就是：
- **建立 hwirq → Linux IRQ 号的映射表**（反向映射）
- **将 `struct irq_fwspec`（固件描述，如 Device Tree 中的 interrupts 属性）翻译为 hwirq**
- **为层级中断控制器提供标准的域级联框架**

#### 2.1.2 四种映射类型（仅作背景了解）

内核文档定义了四种 irq_domain 映射类型，ITS 域使用的是 **Tree** 类型（通过 `irq_domain_create_tree()` 创建），因为 LPI 中断号范围极大（8192–65535），不适合用固定大小的线性表：

| 类型 | 创建函数 | 适用场景 | ITS 是否使用 |
|------|---------|---------|-------------|
| Linear | `irq_domain_create_linear()` | 最大 hwirq < 256，固定大小表 | 否 |
| Tree | `irq_domain_create_tree()` | hwirq 号很大且稀疏（如 LPI） | **是** |
| No Map | `irq_domain_create_nomap()` | hwirq 硬件可编程 | 否（已不推荐） |
| Legacy | `irq_domain_create_legacy()` | 固定 IRQ 映射的旧平台 | 否（已废弃） |

#### 2.1.3 Hierarchy irq_domain 的设计哲学：软件架构匹配硬件拓扑

内核文档明确指出 hierarchy irq_domain 并非 x86 专用，它在 ARM/ARM64 中被大量使用。其核心思想是：

> **为每个中断控制器（interrupt controller）构建一个 `irq_domain`，然后将这些 domain 组织成层级结构。离设备最近的 domain 是子域（child），离 CPU 最近的 domain 是父域（parent）。**

以 x86 为例：
```
Device → IOAPIC → Interrupt Remapping Controller → Local APIC → CPU
  ↑        ↑                  ↑                          ↑
  │   ioapic_domain    remapping_domain             lapic_domain
  │   (child of remap)  (child of lapic)            (root)
```

以 **GIC ITS MSI** 对应到 ARM64：
```
PCIe Device ──MSI──→ ITS ──LPI──→ Redistributor + CPU Interface ──IRQ──→ CPU
   │                  ↑                         ↑
   │             its_domain                its_parent
   │             (child)              (parent / root,
   │             manages:              irq-gic-v3.c)
   │             - 命令队列               manages:
   │             - ITT/Device Table      - LPI Configuration Table
   │             - MAPD/MAPTI            - LPI Pending Table
   │             - MOVI (亲和性)          - IAR/EOIR 寄存器
   │                                     - 最终 EOI
   │
   └── 设备自身不直接对应 domain，而是通过 PCI MSI domain 接入 its_domain
```

**关键点：** Redistributor 和 CPU Interface 由同一个 GIC domain（`its_parent`）统一管理，而非两个独立域。

#### 2.1.4 四大核心接口与关键数据结构

层级 irq_domain 定义了四大接口，每个 domain 的驱动可以选择性实现：

| 接口 | `irq_domain_ops` 回调 | 职责 |
|------|----------------------|------|
| `irq_domain_alloc_irqs()` | `.alloc()` | 分配 irq_desc 和中断控制器相关资源（如 LPI、ITT 表项） |
| `irq_domain_free_irqs()` | `.free()` | 释放资源，回收中断号 |
| `irq_domain_activate_irq()` | `.activate()` | 激活中断控制器硬件以投递此中断（如发送 MAPTI） |
| `irq_domain_deactivate_irq()` | `.deactivate()` | 停用硬件，中断控制器不再投递此中断（如发送 DISCARD） |

**关键数据结构：**

- **`domain->parent`**：`struct irq_domain` 中的 `parent` 指针，维护 domain 之间的父子关系。GIC ITS 中 `its_domain->parent = its_parent`（即 GIC domain）。
- **`irq_data->parent_data`**：`struct irq_data` 中的 `parent_data` 指针，为每个 IRQ 构建与 domain 层级对应的 `irq_data` 链条。每条 `irq_data` 存储了属于自己 domain 的 hwirq 号和 domain 指针。

```
一个 LPI 中断的 irq_data 链条：
  irq_data (ITS domain, hwirq=EventID, chip=its_irq_chip)
    └─→ parent_data (GIC domain, hwirq=LPI号, chip=gic_irq_chip)
```

#### 2.1.5 Stacked irq_chip（堆叠式中断控制器操作）

层次化 irq_domain 的上层抽象是 **Stacked irq_chip**：每个 domain 级别关联一个 `irq_chip`，子 chip 可以自行处理或**委托给父 chip**。ITS 代码中大量使用这种委托模式：

```c
// irq-gic-v3-its.c:2072
.irq_eoi = irq_chip_eoi_parent,  // EOI 操作委托给父域（GIC）的 chip->irq_eoi
```

这种委托机制使得每层驱动只需关注自己管理的硬件，无需知晓父域的实现细节。

### 2.2 ITS 三层软件域架构总览

注意：**"ITS MSI Domain" 和 "its_domain" 并非两个独立的 irq_domain，而是同一个域的两组操作集**。`its_init_domain()` 通过 `msi_create_parent_irq_domain()` 创建的这个域同时承载：
- `irq_domain_ops = its_domain_ops`（.alloc/.free/.activate/.deactivate）—— 硬件层操作
- `host_data = msi_domain_info { .ops = its_msi_domain_ops }`（.msi_prepare/.msi_teardown）—— MSI 抽象操作

因此，实际的软件域层级为 **3 层**：

```
┌──────────────────────────────────────────────────────┐
│  外设驱动层                                            │
│  request_irq() / pci_alloc_irq_vectors()              │
└──────────────────────┬───────────────────────────────┘
                       │
┌──────────────────────▼───────────────────────────────┐
│  Layer 2: PCI/Platform MSI Domain（总线适配层）        │
│  • bus_token = DOMAIN_BUS_PCI_DEVICE_MSI              │
│  • 由 gic_v3_its_msi_parent_ops.bus_select_mask       │
│    自动匹配（MATCH_PCI_MSI | MATCH_PLATFORM_MSI）       │
│  • its_pci_msi_prepare() → 提取 DevID                  │
│  • 文件: kernel/irq/msi.c / drivers/pci/msi/          │
└──────────────────────┬───────────────────────────────┘
                       │  parent
┌──────────────────────▼───────────────────────────────┐
│  Layer 1: its_domain（ITS 操作域，合并了两组 ops）      │
│                                                       │
│  ┌─ irq_domain_ops (its_domain_ops) ──────────────┐  │
│  │  .alloc      = its_irq_domain_alloc()          │  │
│  │  .free       = its_irq_domain_free()           │  │
│  │  .activate   = its_irq_domain_activate()       │  │
│  │  .deactivate = its_irq_domain_deactivate()     │  │
│  │  .select     = msi_lib_irq_domain_select()     │  │
│  └────────────────────────────────────────────────┘  │
│  ┌─ msi_domain_ops (its_msi_domain_ops) ──────────┐  │
│  │  .msi_prepare  = its_msi_prepare()             │  │
│  │  .msi_teardown = its_msi_teardown()            │  │
│  └────────────────────────────────────────────────┘  │
│  ┌─ irq_chip (its_irq_chip) ──────────────────────┐  │
│  │  .irq_mask/unmask/eoi/set_affinity/...         │  │
│  └────────────────────────────────────────────────┘  │
│                                                       │
│  由 its_init_domain() 调用                              │
│  msi_create_parent_irq_domain() 一次性创建              │
└──────────────────────┬───────────────────────────────┘
                       │  parent
┌──────────────────────▼───────────────────────────────┐
│  Layer 0: GIC IRQ Domain（硬件分发层，its_parent）      │
│  • 在 irq-gic-v3.c 中初始化                             │
│  • 管理 Redistributor LPI Configuration Table          │
│    和 LPI Pending Table                                │
│  • 管理 CPU Interface: IAR/EOIR 寄存器                  │
│  • gic_handle_irq() 从 IAR 读取 INTID，                 │
│    通过 generic_handle_domain_irq() 向上分发             │
│  • 提供最终 EOI（irq_chip_eoi_parent 委托到此）          │
└──────────────────────────────────────────────────────┘
```

**对比旧版内核：** 在引入 `msi_create_parent_irq_domain()` 之前，ITS 域和 MSI 域是分离的（通过 `its_pci_msi_init_domain()` 手动级联）。新版将二者合并为单个域，由 MSI 父域框架统一管理总线匹配（`bus_select_mask`），简化了域层级。

### 2.3 关键代码：域初始化

**its_init_domain() — ITS 域的创建入口：**

```c
// irq-gic-v3-its.c:5131
static int its_init_domain(struct its_node *its)
{
    struct irq_domain_info dom_info = {
        .fwnode         = its->fwnode_handle,   // 从 OF/ACPI 获取的固件节点
        .ops            = &its_domain_ops,       // ITS 硬件操作函数集
        .domain_flags   = its->msi_domain_flags,
        .parent         = its_parent,            // 父域：GIC IRQ Domain
    };
    struct msi_domain_info *info;

    info = kzalloc_obj(*info);
    info->ops = &its_msi_domain_ops;             // MSI 抽象操作函数集
    info->data = its;                            // 将 its_node 绑定到 msi_domain_info
    dom_info.host_data = info;

    // 关键：通过 msi_create_parent_irq_domain 创建统合的父 MSI 域
    if (!msi_create_parent_irq_domain(&dom_info,
                                       &gic_v3_its_msi_parent_ops)) {
        kfree(info);
        return -ENOMEM;
    }
    return 0;
}
```

**`gic_v3_its_msi_parent_ops` — MSI 父域操作集：**

```c
// irq-gic-its-msi-parent.c:321
const struct msi_parent_ops gic_v3_its_msi_parent_ops = {
    .supported_flags    = ITS_MSI_FLAGS_SUPPORTED,   // 支持的 MSI 特性标志
    .required_flags     = ITS_MSI_FLAGS_REQUIRED,    // 要求的 MSI 特性标志
    .chip_flags         = MSI_CHIP_FLAG_SET_EOI,     // 需要 SET_EOI 机制
    .bus_select_token   = DOMAIN_BUS_NEXUS,           // 作为总线的连接点
    .bus_select_mask    = MATCH_PCI_MSI | MATCH_PLATFORM_MSI, // 匹配 PCI 和平台 MSI
    .prefix             = "ITS-",                     // 中断名称前缀
    .init_dev_msi_info  = its_init_dev_msi_info,      // 设备 MSI 信息初始化
};
```

**其 `bus_select_mask` 包含 `MATCH_PCI_MSI | MATCH_PLATFORM_MSI`**，这意味着 PCI 设备和平台设备创建 MSI 中断域时，内核 MSI 框架会自动匹配到 ITS 域，无需像旧版内核那样显式调用 `its_pci_msi_init_domain()`。

### 2.4 域操作函数集注册逻辑

**its_domain_ops — ITS 硬件域操作（irq_domain_ops）：**

```c
// irq-gic-v3-its.c:3774
static const struct irq_domain_ops its_domain_ops = {
    .select     = msi_lib_irq_domain_select,   // 域选择（MSI 库辅助）
    .alloc      = its_irq_domain_alloc,         // ★ 分配中断：LPI 分配 + GIC 映射
    .free       = its_irq_domain_free,          // 释放中断
    .activate   = its_irq_domain_activate,      // ★ 激活中断：发送 MAPTI 命令
    .deactivate = its_irq_domain_deactivate,    // 去激活：发送 DISCARD 命令
};
```

**its_msi_domain_ops — MSI 抽象层操作（msi_domain_ops）：**

```c
// irq-gic-v3-its.c:3655
static struct msi_domain_ops its_msi_domain_ops = {
    .msi_prepare    = its_msi_prepare,    // ★ 准备阶段：创建/查找 its_device
    .msi_teardown   = its_msi_teardown,   // 拆除阶段：释放 its_device
};
```

**its_irq_chip — 中断控制器操作（irq_chip）：**

```c
// irq-gic-v3-its.c:2068
static struct irq_chip its_irq_chip = {
    .name                   = "ITS",
    .irq_mask               = its_mask_irq,          // 屏蔽：发送 DISCARD
    .irq_unmask             = its_unmask_irq,        // 解除屏蔽：发送 MAPTI
    .irq_eoi                = irq_chip_eoi_parent,   // EOI 委托给父域（GIC）
    .irq_set_affinity       = its_set_affinity,      // 亲和性：发送 MOVI
    .irq_compose_msi_msg    = its_irq_compose_msi_msg, // 组合 MSI 消息
    .irq_set_irqchip_state  = its_irq_set_irqchip_state, // 获取/设置中断状态
    .irq_retrigger          = its_irq_retrigger,     // 重新触发
    .irq_set_vcpu_affinity  = its_irq_set_vcpu_affinity,  // GICv4 vCPU 亲和性
};
```

### 2.5 域层级的串联方式

```
pci_device->msi_domain
    │  通过 bus_select_mask 自动匹配
    │  parent = its_init_domain() 创建的域
    ▼
its_domain (irq_domain)
    │   ops = its_domain_ops (.alloc / .activate / ...)
    │   host_data = msi_domain_info { .ops = its_msi_domain_ops,
    │                                  .data = its_node }
    │   parent = its_parent (GIC domain)
    ▼
GIC irq_domain (irq-gic-v3.c 初始化)
    │   ops = gic_irq_domain_ops
    │   处理 LPI 配置表（Redistributor 内存中的 LPI Configuration Table）
    ▼
硬件：ITS → Redistributor → CPU Interface
```

---

## 三、核心源码文件位置分布

| 文件路径 | 职责 |
|----------|------|
| `drivers/irqchip/irq-gic-v3-its.c` | **核心驱动**：ITS 控制器初始化、命令队列管理、`its_device`/`its_node` 生命周期、中断域操作（alloc/free/activate/deactivate）、`its_irq_chip` 实现 |
| `drivers/irqchip/irq-gic-its-msi-parent.c` | **MSI 父域操作**：`gic_v3_its_msi_parent_ops` 定义、`its_init_dev_msi_info()`、`its_pci_msi_prepare()`（PCI MSI 准备阶段） |
| `drivers/irqchip/irq-gic-v3.c` | **GICv3 主驱动**：Distributor/Redistributor 初始化、`gic_handle_irq()`（中断处理总入口）、LPI 配置表的建立、`its_parent` 域的创建 |
| `kernel/irq/msi.c` | **内核通用 MSI 架构**：`msi_domain_alloc_irqs()`、`msi_create_parent_irq_domain()`、`pci_msi_create_irq_domain()` 等 |
| `include/linux/msi.h` | **MSI API 头文件**：`msi_domain_info`、`msi_domain_ops`、`msi_alloc_info_t`、`msi_parent_ops` 等结构体定义 |
| `include/linux/irqchip/arm-gic-v3.h` | **GICv3 寄存器定义**：`GITS_CMD_*`（ITS 命令编码）、`GITS_BASER_*`（寄存器字段）、`GITS_TYPER_*` 等 |
| `drivers/irqchip/irq-gic-its-msi-parent.h` | **MSI 父域接口声明**：声明 `gic_v3_its_msi_parent_ops` 和 `gic_v5_its_msi_parent_ops` |
| `drivers/pci/msi/msi.c` | **PCI MSI 层**：`pci_alloc_irq_vectors()` 等 PCI MSI API 实现 |

---

## 四、追踪三大核心业务流程

### 4.1 各级 Irq Domain 创建流程（初始化阶段）

此流程描述从内核启动到设备 MSI 域就绪的完整 domain 层级构建过程。

#### 流程概览

```
内核启动
  │
  ├─ Step 0: gic_init_bases()                 ← irq-gic-v3.c
  │    ├─ irq_domain_create_tree()            → 创建 GIC domain (根域, DOMAIN_BUS_WIRED)
  │    └─ its_init(handle, rdists, gic_data.domain, ...)
  │
  ├─ Step 1: its_init()                        ← irq-gic-v3-its.c
  │    ├─ its_parent = parent_domain          ← 记录父域（GIC domain）
  │    ├─ its_of_probe() / its_acpi_probe()    → 遍历设备树/ACPI 描述符
  │    │    └─ its_probe_one()                ← 每个 ITS 资源节点
  │    │         ├─ 分配 cmd_base (命令队列)
  │    │         ├─ its_alloc_tables()        ← 初始化 Device/Collection Table
  │    │         ├─ 配置 GITS_CBASER, 启用 GITS_CTLR
  │    │         └─ its_init_domain()         ← ★ 创建 ITS domain
  │    │              └─ msi_create_parent_irq_domain(&dom_info,
  │    │                    &gic_v3_its_msi_parent_ops)
  │    │                    parent = its_parent (GIC domain)
  │    │                    ops = its_domain_ops
  │    │                    host_data = msi_domain_info { .ops = its_msi_domain_ops }
  │    │
  │    └─ its_init_vpe_domain()               ← GICv4: vPE proxy device
  │
  ├─ Step 2: PCI 设备枚举（运行时）
  │    └─ pci_device_add() → pcibios_add_device()
  │         └─ dev_set_msi_domain(&pdev->dev, its_domain)
  │            ★ 将 ITS domain 设置为设备的 MSI parent domain
  │
  └─ Step 3: 设备驱动调用 pci_alloc_irq_vectors()
       └─ pci_setup_msi_device_domain()       ← 仅在首次调用时执行
            └─ pci_create_device_domain()
                 └─ msi_create_device_irq_domain(dev, ..., &pci_msi_template, ...)
                      │  parent = dev->msi.domain (即 its_domain)
                      │  ★ bus_token = DOMAIN_BUS_PCI_DEVICE_MSI
                      │  ★ 匹配 gic_v3_its_msi_parent_ops.bus_select_mask
                      │
                      ├─ pops->init_dev_msi_info()  → its_init_dev_msi_info()
                      │     └─ 根据 bus_token 注入 its_pci_msi_prepare / its_msi_prepare
                      │
                      ├─ __msi_create_irq_domain()  → 创建 PCI MSI domain
                      │     parent = its_domain
                      │
                      └─ msi_domain_prepare_irqs()  → 触发 msi_prepare 回调
                            └─ its_pci_msi_prepare()  → 创建/查找 its_device
```

#### Step 0: GIC Domain 创建 — 硬件分发层的根域

GIC domain 是整棵 domain 树的根，由 `gic_init_bases()` 在 `irq-gic-v3.c` 中创建：

```c
// irq-gic-v3.c:2019
gic_data.domain = irq_domain_create_tree(handle, &gic_irq_domain_ops, &gic_data);

// 标记为 wired 类型（传统有线中断 + LPI 均由该域管理）
irq_domain_update_bus_token(gic_data.domain, DOMAIN_BUS_WIRED);
```

该域使用 **Tree 类型**（radix tree）反向映射，因为 LPI 的 INTID 范围是 8192–65535，稀疏且范围大。创建完成后，GIC domain 作为根域不设 parent（`parent = NULL`）。

随后，GIC domain 作为 `parent_domain` 参数传递给 ITS 初始化：

```c
// irq-gic-v3.c:2057
if (gic_dist_supports_lpis()) {
    its_init(handle, &gic_data.rdists, gic_data.domain, dist_prio_irq);
}
```

#### Step 1: ITS Domain 创建 — 合并 ITS 硬件操作与 MSI 抽象

`its_init()` 设置全局变量后，遍历固件描述的 ITS 节点，调用 `its_probe_one()` 为每个 ITS 硬件实例执行初始化，最终由 `its_init_domain()` 创建 domain：

```c
// irq-gic-v3-its.c:5828
its_parent = parent_domain;   // ★ 记录父域为 GIC domain

// 遍历固件中的 ITS 节点
its_of_probe(of_node);        // Device Tree 路径
// 或 its_acpi_probe();       // ACPI 路径

// 为每个 ITS 节点调用 its_probe_one()
```

**`its_probe_one()` — 单个 ITS 硬件的初始化：**

1. 分配命令队列：`its->cmd_base = page_address(page)`（通常 64KB 环形缓冲区）
2. 分配硬件表：`its_alloc_tables(its)` — 初始化 Device Table、Collection Table 等 BASER 表
3. 分配 Collection 数组：`its_alloc_collections(its)` — 每个 CPU 一个 Collection
4. 写入 `GITS_CBASER` 寄存器，指向命令队列的物理地址
5. 启用 ITS：设置 `GITS_CTLR.Enable` 位
6. **创建 domain**：`its_init_domain(its)` — 见下方
7. 挂载到全局链表：`list_add(&its->entry, &its_nodes)`

**`its_init_domain()` — Domain 的软件构建：**

```c
// irq-gic-v3-its.c:5131
static int its_init_domain(struct its_node *its)
{
    struct irq_domain_info dom_info = {
        .fwnode       = its->fwnode_handle,     // 固件节点（OF/ACPI）
        .ops          = &its_domain_ops,         // ★ 硬件层 ops
        .domain_flags = its->msi_domain_flags,
        .parent       = its_parent,              // ★ 父域 = GIC domain
    };
    struct msi_domain_info *info;

    info = kzalloc_obj(*info);
    info->ops = &its_msi_domain_ops;             // ★ MSI 抽象层 ops
    info->data = its;                            // 绑定 its_node
    dom_info.host_data = info;

    // ★ 一次调用创建同时承载两组 ops 的单个 irq_domain
    if (!msi_create_parent_irq_domain(&dom_info, &gic_v3_its_msi_parent_ops)) {
        kfree(info);
        return -ENOMEM;
    }
    return 0;
}
```

**关键点：** `msi_create_parent_irq_domain()` 创建的是一个 **MSI Parent Domain**——它既是 `irq_domain`（承载 `its_domain_ops`），又是 MSI 框架的父域（承载 `gic_v3_its_msi_parent_ops`）。这意味着 ITS domain 之后可以作为多个设备 MSI domain 的 parent，通过 `bus_select_mask` 自动匹配 PCI 和平台 MSI 设备。

`gic_v3_its_msi_parent_ops` 的关键字段：

| 字段 | 值 | 作用 |
|------|-----|------|
| `bus_select_mask` | `MATCH_PCI_MSI \| MATCH_PLATFORM_MSI` | 该域作为 PCI 和平台 MSI 设备的父域 |
| `bus_select_token` | `DOMAIN_BUS_NEXUS` | 作为总线连接的枢纽节点 |
| `prefix` | `"ITS-"` | 设备中断域的名称前缀 |
| `init_dev_msi_info` | `its_init_dev_msi_info()` | 设备创建 MSI 域时，定制其 `msi_prepare` |

#### Step 2: 设备 MSI Parent Domain 的绑定

设备枚举时（PCI 或平台设备），内核通过 `dev_set_msi_domain()` 将 ITS domain 绑定到设备的 `msi.domain` 字段。对于 PCI 设备，这是通过 firmware 提供的 `msi-map` 属性（Device Tree）或 IORT 表（ACPI）来确定的。

```
dev_set_msi_domain(&pdev->dev, its_domain)
  → pdev->dev.msi.domain = its_domain
```

此时设备尚未创建自己的 MSI domain——只是记录了"我的 MSI parent 是哪个 ITS"。

#### Step 3: PCI MSI Device Domain 创建 — 触发设备级 MSI 域

当设备驱动首次调用 `pci_alloc_irq_vectors()` 时：

```c
// drivers/pci/msi/irqdomain.c:239
bool pci_setup_msi_device_domain(struct pci_dev *pdev, unsigned int hwsize)
{
    // 检查是否已存在匹配的 domain
    if (pci_match_device_domain(pdev, DOMAIN_BUS_PCI_DEVICE_MSI))
        return true;

    // 创建新的设备 MSI domain
    return pci_create_device_domain(pdev, &pci_msi_template, hwsize);
}
```

`pci_create_device_domain()` 调用：

```c
// drivers/pci/msi/irqdomain.c:207
msi_create_device_irq_domain(&pdev->dev, MSI_DEFAULT_DOMAIN,
                              tmpl, hwsize, NULL, NULL);
```

`msi_create_device_irq_domain()` 是通用 MSI 框架的函数：

```c
// kernel/irq/msi.c:1029
bool msi_create_device_irq_domain(...)
{
    struct irq_domain *parent = dev->msi.domain;  // ★ 即 its_domain
    pops = parent->msi_parent_ops;                // ★ 即 gic_v3_its_msi_parent_ops

    // ① 初始化 domain info（设置 bus_token = DOMAIN_BUS_PCI_DEVICE_MSI）
    // ② pops->init_dev_msi_info() → its_init_dev_msi_info()
    //      ↓
    //      根据 info->bus_token 匹配到 PCI 分支，注入:
    //        info->ops->msi_prepare  = its_pci_msi_prepare
    //        info->ops->msi_teardown = its_msi_teardown
    //
    // ③ __msi_create_irq_domain(fwnode, &info, ..., parent=its_domain)
    //      创建 PCI MSI domain，parent 指向 its_domain
    //
    // ④ msi_domain_prepare_irqs() → ops->msi_prepare()
    //      即 its_pci_msi_prepare() → 提取 DevID → its_create_device()
}
```

#### 最终形成的中断域树

```
GIC irq_domain (DOMAIN_BUS_WIRED, root, parent=NULL)
  │
  ├─ its_domain (MSI parent, parent=GIC domain)
  │    │  domain_ops = its_domain_ops
  │    │  msi_parent_ops = gic_v3_its_msi_parent_ops
  │    │  host_data->ops = its_msi_domain_ops
  │    │
  │    ├─ PCI MSI domain for Device A (DOMAIN_BUS_PCI_DEVICE_MSI, parent=its_domain)
  │    │     msi_prepare = its_pci_msi_prepare
  │    │
  │    ├─ PCI MSI domain for Device B (DOMAIN_BUS_PCI_DEVICE_MSI, parent=its_domain)
  │    │
  │    └─ Platform MSI domain (DOMAIN_BUS_DEVICE_MSI, parent=its_domain)
  │
  └─ (GICv4) its_vpe_domain (parent=GIC domain)
```

**小结：**
- GIC domain 和 its_domain 在**内核初始化阶段**一次性创建
- PCI/Platform MSI device domain 在**设备首次申请中断时按需创建**
- 各层 domain 通过 `parent` 指针串成树，中断分配请求从叶子域逐层向上传播

---

### 4.2 PCIe 设备使能 MSI 的过程（设备准备阶段）

#### 流程概览

```
pci_alloc_irq_vectors()              ← 设备驱动调用
  │
  ├─→ __pci_enable_msi_range()
  │     ├─→ msi_domain_alloc_irqs()
  │     │     ├─→ its_init_dev_msi_info()       ← 根据 bus_token 定制 msi_prepare
  │     │     ├─→ its_pci_msi_prepare()         ← Step 1: 提取 DevID，创建 its_device
  │     │     ├─→ its_irq_domain_alloc()        ← Step 2: 分配 LPI，构建层级域 IRQ
  │     │     └─→ ... (irq_startup 等触发 activate)
  │     │           └─→ its_irq_domain_activate() ← Step 3: 发送 MAPTI 建立映射
  │     └─→ 返回 virq 范围给设备驱动
  └─→ 驱动调用 request_irq() 注册 ISR
```

#### Step 1: its_pci_msi_prepare() — PCI 设备准备

```c
// irq-gic-its-msi-parent.c:67
static int its_pci_msi_prepare(struct irq_domain *domain, struct device *dev,
                                int nvec, msi_alloc_info_t *info)
{
    struct pci_dev *pdev = to_pci_dev(dev);
    struct pci_dev *alias_dev;
    int alias_count = 0;

    // 处理 PCIe 别名（DMA alias）：Bridge 下游多设备可能共享 DevID
    pci_for_each_dma_alias(pdev, its_get_pci_alias, &alias_dev);
    if (alias_dev != pdev) {
        if (alias_dev->subordinate)
            pci_walk_bus(alias_dev->subordinate,
                         its_pci_msi_vec_count, &alias_count);
        info->flags |= MSI_ALLOC_FLAGS_PROXY_DEVICE;
    }

    // ★ 核心：从 PCI 设备提取 ITS DeviceID（Requester ID）
    info->scratchpad[0].ul = pci_msi_domain_get_msi_rid(domain->parent, pdev);
    // ...
}
```

**关键点：** `pci_msi_domain_get_msi_rid()` 从 PCIe 配置空间的 BDF（Bus/Device/Function）提取 Requester ID。ARM64 系统通常直接使用 BDF 作为 ITS DeviceID。

#### Step 2: its_msi_prepare() → its_create_device() — 设备与 ITT 创建

```c
// irq-gic-v3-its.c:3572
static int its_msi_prepare(struct irq_domain *domain, struct device *dev,
                           int nvec, msi_alloc_info_t *info)
{
    u32 dev_id = info->scratchpad[0].ul;     // Step 1 中提取的 DevID
    msi_info = msi_get_domain_info(domain);
    its = msi_info->data;                    // 取出绑定的 its_node

    mutex_lock(&its->dev_alloc_lock);

    // ★ 先查是否已存在该 DevID 的设备（如 PCI 别名共享）
    its_dev = its_find_device(its, dev_id);
    if (its_dev) {
        its_dev->shared = true;              // 标记为共享
        goto out;                            // 复用已有 ITT
    }

    // ★ 核心：创建新的 its_device
    its_dev = its_create_device(its, dev_id, nvec, true);
    // ...
    info->scratchpad[0].ptr = its_dev;       // 传递给后续 alloc 阶段
}
```

**its_create_device() 内部细节：**

```c
// irq-gic-v3-its.c:3466
static struct its_device *its_create_device(struct its_node *its, u32 dev_id,
                                            int nvecs, bool alloc_lpis)
{
    // ① 在 Device Table 中为此 DevID 分配表项
    if (!its_alloc_device_table(its, dev_id))
        return NULL;

    // ② 对齐 nvecs 到 2 的幂（ITS 硬件要求）
    if (WARN_ON(!is_power_of_2(nvecs)))
        nvecs = roundup_pow_of_two(nvecs);

    // ③ 限制 nvecs 不超过 ITS 硬件支持的 EventID 位数
    id_bits = FIELD_GET(GITS_TYPER_IDBITS, its->typer) + 1;
    nvecs = min_t(unsigned int, nvecs, BIT(id_bits));
    nr_ites = max(2, nvecs);                       // 最少 2 个 ITTE

    // ④ 分配 ITT 内存（通过 gen_pool 或直接申请页面）
    sz = nr_ites * (FIELD_GET(GITS_TYPER_ITT_ENTRY_SIZE, its->typer) + 1);
    sz = max(sz, ITS_ITT_ALIGN);
    itt = itt_alloc_pool(its->numa_node, sz);

    // ⑤ 分配 LPI 号范围（从全局 LPI 池中获取）
    if (alloc_lpis) {
        lpi_map = its_lpi_alloc(nvecs, &lpi_base, &nr_lpis);
        col_map = kcalloc(nr_lpis, sizeof(*col_map), GFP_KERNEL);
    }

    // ⑥ 填充 its_device 结构体
    dev->its = its;
    dev->itt = itt;
    dev->itt_sz = sz;
    dev->nr_ites = nr_ites;
    dev->event_map.lpi_map = lpi_map;
    dev->event_map.col_map = col_map;
    dev->event_map.lpi_base = lpi_base;
    dev->event_map.nr_lpis = nr_lpis;
    dev->device_id = dev_id;

    // ⑦ 挂载到 ITS 的设备链表
    raw_spin_lock_irqsave(&its->lock, flags);
    list_add(&dev->entry, &its->its_device_list);
    raw_spin_unlock_irqrestore(&its->lock, flags);

    // ⑧ ★ 发送 MAPD 命令：告诉 ITS 硬件此 DeviceID → ITT 的映射关系
    its_send_mapd(dev, 1);

    return dev;
}
```

**ITT 内存分配策略：**
- 大块（`≥ PAGE_SIZE`）：直接通过 `its_alloc_pages_node()` 分配物理页面
- 小块（`< PAGE_SIZE`）：通过 `gen_pool`（通用内存池）分配，避免内部碎片

#### Step 3: its_irq_domain_alloc() — LPI 分配与域级联

```c
// irq-gic-v3-its.c:3684
static int its_irq_domain_alloc(struct irq_domain *domain, unsigned int virq,
                                unsigned int nr_irqs, void *args)
{
    msi_alloc_info_t *info = args;
    struct its_device *its_dev = info->scratchpad[0].ptr;
    irq_hw_number_t hwirq;

    // ① 从其 its_device 的 lpi_map 中分配连续 LPI 号
    err = its_alloc_device_irq(its_dev, nr_irqs, &hwirq);

    // ② IOMMU DMA 准备（如果系统启用了 SMMU）
    err = iommu_dma_prepare_msi(info->desc, its->get_msi_base(its_dev));

    for (i = 0; i < nr_irqs; i++) {
        // ③ ★ 向父域（GIC Domain）申请中断：
        //   构建 fwspec = {GIC_IRQ_TYPE_LPI, hwirq+i, IRQ_TYPE_EDGE_RISING}
        //   委托父域配置 LPI Configuration Table
        its_irq_gic_domain_alloc(domain, virq + i, hwirq + i);

        // ④ 设置 irq_data 的 chip 为 its_irq_chip
        irq_domain_set_hwirq_and_chip(domain, virq + i,
                                       hwirq + i, &its_irq_chip, its_dev);
        irqd = irq_get_irq_data(virq + i);
        irqd_set_single_target(irqd);
        irqd_set_affinity_on_activate(irqd);  // activate 时才设置亲和性
        irqd_set_resend_when_in_progress(irqd);
    }
    return 0;
}
```

**its_alloc_device_irq() — 从设备 LPI 位图中分配：**

```c
// irq-gic-v3-its.c:3556
static int its_alloc_device_irq(struct its_device *dev, int nvecs,
                                 irq_hw_number_t *hwirq)
{
    // 在位图中查找 nvecs 个连续的空闲位（2^order 对齐）
    idx = bitmap_find_free_region(dev->event_map.lpi_map,
                                   dev->event_map.nr_lpis,
                                   get_count_order(nvecs));
    if (idx < 0)
        return -ENOSPC;

    *hwirq = dev->event_map.lpi_base + idx;
    return 0;
}
```

#### Step 4: its_irq_domain_activate() — 发送 MAPTI 建立映射

```c
// irq-gic-v3-its.c:3722
static int its_irq_domain_activate(struct irq_domain *domain,
                                    struct irq_data *d, bool reserve)
{
    struct its_device *its_dev = irq_data_get_irq_chip_data(d);
    u32 event = its_get_event_id(d);          // event = hwirq - lpi_base
    int cpu;

    // ① 选择目标 CPU（NUMA 感知，非托管中断优先本地节点）
    cpu = its_select_cpu(d, cpu_online_mask);

    // ② 更新亲和性记录
    its_dev->event_map.col_map[event] = cpu;
    irq_data_update_effective_affinity(d, cpumask_of(cpu));

    // ③ ★ 发送 MAPTI 命令：建立 (DevID, EventID, LPI, Collection) 的映射
    its_send_mapti(its_dev, d->hwirq, event);

    return 0;
}
```

**its_send_mapti() → 命令队列路径：**

```
its_send_mapti(dev, irq_id, id)
  │
  └─→ its_send_single_command(dev->its, its_build_mapti_cmd, &desc)
        │
        ├─→ raw_spin_lock_irqsave(&its->lock)
        ├─→ its_allocate_entry(its)           ← 在环形命令队列中分配一个槽位
        ├─→ its_build_mapti_cmd(its, cmd, desc)
        │     ├─→ its_encode_cmd(cmd, GITS_CMD_MAPTI)      ← 命令编码 0x0a
        │     ├─→ its_encode_devid(cmd, dev->device_id)     ← bits [63:32]
        │     ├─→ its_encode_event_id(cmd, event_id)        ← bits [31:0]
        │     ├─→ its_encode_phys_id(cmd, phys_id)          ← bits [63:32]
        │     ├─→ its_encode_collection(cmd, col->col_id)   ← bits [15:0]
        │     └─→ its_fixup_cmd(cmd)               ← 字节序转换（BE→LE）
        ├─→ its_flush_cmd(its, cmd)              ← Cache Flush 确保 ITS 可见
        ├─→ its_build_sync_cmd(its, sync_cmd, col) ← ★ 追加 SYNC 命令
        ├─→ its_post_commands(its)               ← 更新 GITS_CWRITER 通知 ITS
        └─→ raw_spin_unlock_irqrestore(&its->lock)
```

**MAPD 命令编码示例（its_build_mapd_cmd）：**

```c
// irq-gic-v3-its.c:708
static struct its_collection *its_build_mapd_cmd(struct its_node *its,
                                                  struct its_cmd_block *cmd,
                                                  struct its_cmd_desc *desc)
{
    u8 size = ilog2(desc->its_mapd_cmd.dev->nr_ites);
    itt_addr = virt_to_phys(desc->its_mapd_cmd.dev->itt);

    its_encode_cmd(cmd, GITS_CMD_MAPD);        // raw_cmd[0][7:0]   = 0x08
    its_encode_devid(cmd, dev->device_id);      // raw_cmd[0][63:32] = DevID
    its_encode_size(cmd, size - 1);             // raw_cmd[1][4:0]   = ITT 条目数-1
    its_encode_itt(cmd, itt_addr);              // raw_cmd[2][51:8]  = ITT 物理地址
    its_encode_valid(cmd, 1);                   // raw_cmd[2][63]    = 1 (Valid)

    its_fixup_cmd(cmd);  // 转为 LE
    return NULL;         // MAPD 不需要 SYNC（返回 NULL 则不发 SYNC）
}
```

### 4.3 中断触发与处理流程（运行时阶段）

#### 流程概览

```
设备向 GITS_TRANSLATER 写入 EventID
  │
  ├─→ ITS 硬件翻译：
  │     Device Table[DevID] → ITT Base
  │     ITT[EventID] → {LPI, CollectionID}
  │     Collection Table[CollectionID] → Redistributor 物理地址
  │
  ├─→ Redistributor 将 LPI 注入 CPU Interface
  │
  ├─→ CPU 感知中断，进入异常向量
  │     异常入口代码 → gic_handle_irq()    ← GICv3 中断处理总入口
  │
  └─→ gic_handle_irq()
        │
        ├─→ gic_read_iar()                ← 读取 ICC_IAR0_EL1 获取 INTID
        ├─→ gic_complete_ack(irqnr)       ← 写 ICC_EOIR0_EL1（优先级丢弃）
        └─→ generic_handle_domain_irq(gic_data.domain, irqnr)
              │
              ├─→ 通过 irq_domain 层级找到 irq_desc
              └─→ desc->handle_irq(desc)  ← 即 handle_fasteoi_irq()
                    │
                    ├─→ mask_irq(desc)            ← ONESHOT 模式下 mask
                    ├─→ handle_irq_event(desc)    ← 调用设备驱动 ISR
                    │     └─→ action->handler()    ← 驱动注册的中断处理函数
                    ├─→ cond_unmask_eoi_irq()     ← EOI + Unmask
                    │     └─→ chip->irq_eoi()      ← irq_chip_eoi_parent → GIC
                    └─→ check_irq_resend()        ← 检查是否需要重发
```

#### 详细源码追踪

**① 中断总入口 gic_handle_irq()：**

```c
// irq-gic-v3.c:915
static void __exception_irq_entry gic_handle_irq(struct pt_regs *regs)
{
    if (unlikely(gic_supports_nmi() && !interrupts_enabled(regs)))
        __gic_handle_irq_from_irqsoff(regs);   // NMI 上下文
    else
        __gic_handle_irq_from_irqson(regs);    // 普通 IRQ 上下文
}
```

**② 普通 IRQ 处理 `__gic_handle_irq_from_irqson()`：**

```c
// irq-gic-v3.c:855
static void __gic_handle_irq_from_irqson(struct pt_regs *regs)
{
    irqnr = gic_read_iar();                    // ★ 读 IAR，获取中断号
    is_nmi = gic_rpr_is_nmi_prio();           // 检查是否是 NMI 优先级

    if (is_nmi) {
        nmi_enter();
        __gic_handle_nmi(irqnr, regs);
        nmi_exit();
    }

    if (!is_nmi)
        __gic_handle_irq(irqnr, regs);         // 普通中断分发
}
```

**③ `__gic_handle_irq()` 分发中断：**

```c
// irq-gic-v3.c:818
static void __gic_handle_irq(u32 irqnr, struct pt_regs *regs)
{
    if (gic_irqnr_is_special(irqnr))           // 跳过特殊中断号 (1020-1023)
        return;

    gic_complete_ack(irqnr);                   // 优先级丢弃（Priority Drop）

    // ★ 核心：通过域分发中断
    if (generic_handle_domain_irq(gic_data.domain, irqnr)) {
        WARN_ONCE(true, "Unexpected interrupt (irqnr %u)\n", irqnr);
        gic_deactivate_unhandled(irqnr);       // 未处理的中断需要反激活
    }
}
```

**④ LPI 中断的 irqnr 范围与识别：**

LPI 的中断号在 GICv3 规范中固定为 **8192 ~ 65535**（即 INTID 8192 及以上）。`gic_handle_irq()` 读取 IAR 得到的 INTID 对 LPI 来说就是 LPI 号本身。由于 `its_irq_gic_domain_alloc()` 已经向 GIC 域注册了 `(GIC_IRQ_TYPE_LPI, hwirq)` 的映射关系，`generic_handle_domain_irq()` 能正确找到对应的 `irq_desc`。

**⑤ handle_fasteoi_irq() — LPI 中断的 flow handler：**

```c
// kernel/irq/chip.c:740
void handle_fasteoi_irq(struct irq_desc *desc)
{
    struct irq_chip *chip = desc->irq_data.chip;

    raw_spin_lock(&desc->lock);

    // A) 检查 PM（Power Management）状态
    if (!irq_can_handle_pm(desc)) { ... return; }

    // B) 检查是否有有效的 action（处理函数）
    if (!irq_can_handle_actions(desc)) {
        mask_irq(desc);
        cond_eoi_irq(chip, &desc->irq_data);   // 直接 EOI，没有 action
        return;
    }

    kstat_incr_irqs_this_cpu(desc);            // 统计计数

    if (desc->istate & IRQS_ONESHOT)
        mask_irq(desc);                         // ONESHOT：先屏蔽

    // C) ★ 调用设备驱动注册的处理函数
    handle_irq_event(desc);

    // D) ★ EOI + 可能的 Unmask
    cond_unmask_eoi_irq(desc, chip);

    // E) 检查是否需要在其他 CPU 上重发（亲和性迁移竞争）
    if (unlikely(desc->istate & IRQS_PENDING))
        check_irq_resend(desc, false);

    raw_spin_unlock(&desc->lock);
}
```

**⑥ EOI 路径：**

对于 LPI 中断，`its_irq_chip.irq_eoi` 设置为 `irq_chip_eoi_parent`：

```c
.irq_eoi = irq_chip_eoi_parent,   // 委托父域（GIC Domain）执行 EOI
```

这会调用到 GICv3 的 EOI 处理（写 ICC_EOIR0_EL1），完成中断优先级丢弃，允许后续中断（包括优先级更低的 LPI）被触发。

#### LPI 中断从触发到 ISR 的完整路径图

```
┌─────────────────────────────────────────────────────────────────┐
│  PCIe 设备                                                        │
│  写 MSI 消息到 GITS_TRANSLATER (EventID = N)                      │
└──────────────────┬──────────────────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────────────────┐
│  ITS 硬件                                                         │
│  ① 从写入地址识别 ITS                                              │
│  ② Device Table[DevID] → ITT 基址                                 │
│  ③ ITT[EventID=N] → {LPI_NUM=8192+K, CollectionID=C}              │
│  ④ Collection Table[C] → Redistributor 物理地址                    │
│  ⑤ 向目标 Redistributor 注入 LPI_NUM                               │
└──────────────────┬──────────────────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────────────────┐
│  GIC Redistributor                                                │
│  ① 将 LPI_NUM 写入 LPI Pending Table（设置 pending 位）            │
│  ② 检查 LPI Configuration Table[K]：Enable + Priority              │
│  ③ 若优先级足够高，向 CPU Interface 发送中断信号                    │
└──────────────────┬──────────────────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────────────────┐
│  CPU Interface / 异常入口                                          │
│  ① CPU 进入 IRQ 异常向量                                           │
│  ② gic_handle_irq() → gic_read_iar() 获取 INTID=8192+K            │
└──────────────────┬──────────────────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────────────────┐
│  内核 IRQ 子系统                                                   │
│  ① generic_handle_domain_irq(GIC_domain, 8192+K)                 │
│  ② irq_find_mapping() → virq                                      │
│  ③ desc = irq_to_desc(virq)                                       │
│  ④ desc->handle_irq(desc) = handle_fasteoi_irq(desc)              │
│  ⑤ handle_irq_event(desc) → action->handler() [设备 ISR]          │
│  ⑥ irq_chip_eoi_parent() → GIC EOI                                │
└─────────────────────────────────────────────────────────────────┘
```

---

## 五、源码调试与追踪技巧

### 5.1 内核配置项

确保以下配置项启用，以获取完整的动态调试能力：

```kconfig
CONFIG_GENERIC_IRQ_DEBUGFS=y      # /sys/kernel/debug/irq/ 下的中断信息
CONFIG_IRQ_DOMAIN_DEBUG=y         # /sys/kernel/debug/irq_domain_mapping
CONFIG_GENERIC_MSI_IRQ=y          # MSI 基础设施
CONFIG_DYNAMIC_DEBUG=y            # 支持 pr_debug 动态开启
```

### 5.2 pr_debug 动态打印

在 ITS 驱动源码中预置了大量 `pr_debug()` 打印。通过 debugfs 动态开启：

```bash
# 开启 ITS 驱动所有 pr_debug 输出
echo 'file irq-gic-v3-its.c +p' > /sys/kernel/debug/dynamic_debug/control

# 仅开启特定函数相关打印
echo 'func its_irq_domain_alloc +p' > /sys/kernel/debug/dynamic_debug/control
echo 'func its_create_device +p' > /sys/kernel/debug/dynamic_debug/control
echo 'func its_msi_prepare +p' > /sys/kernel/debug/dynamic_debug/control

# 查看输出
dmesg -w
```

关键 `pr_debug` 位置：
- `its_irq_domain_alloc()`: `pr_debug("ID:%d pID:%d vID:%d\n", event_id, pID, vID)` — 打印 EventID / LPI / virq 的映射关系
- `its_msi_prepare()`: `pr_debug("Reusing ITT for devID %x\n", ...)` 和 `pr_debug("ITT %d entries, %d bits\n", ...)`

### 5.3 DebugFS 信息节点

```bash
# 查看中断域映射关系
cat /sys/kernel/debug/irq_domain_mapping

# 查看每个 IRQ 的详细信息（chip name、handler、affinity 等）
cat /sys/kernel/debug/irq/irqs/<irq_num>

# 查看 ITS 设备状态
cat /sys/kernel/debug/irq/domains/
```

### 5.4 内核 Tracepoint 追踪

ITS 驱动虽然未定义专用 tracepoint，但可以通过以下通用 tracepoint 追踪：

```bash
# 追踪中断处理
echo 1 > /sys/kernel/debug/tracing/events/irq/irq_handler_entry/enable
echo 1 > /sys/kernel/debug/tracing/events/irq/irq_handler_exit/enable

# 追踪 MSI 设置
echo 1 > /sys/kernel/debug/tracing/events/msi/enable

# 查看 trace 输出
cat /sys/kernel/debug/tracing/trace

# 使用 trace-cmd 进行高级追踪
trace-cmd record -e irq -e msi -e irq_vectors
trace-cmd report
```

### 5.5 ftrace 函数追踪

```bash
# 追踪 ITS 核心函数调用图
echo its_irq_domain_alloc > /sys/kernel/debug/tracing/set_graph_function
echo its_create_device >> /sys/kernel/debug/tracing/set_graph_function
echo its_send_single_command >> /sys/kernel/debug/tracing/set_graph_function
echo function_graph > /sys/kernel/debug/tracing/current_tracer

# 查看调用图
cat /sys/kernel/debug/tracing/trace

# 追踪完毕后关闭
echo nop > /sys/kernel/debug/tracing/current_tracer
```

### 5.6 硬件寄存器观察

在开发板上，可以通过 `/dev/mem` 或 JTAG 直接观察 ITS 硬件寄存器状态：

```bash
# 使用 busybox devmem 观察 ITS 寄存器（需要知道 ITS 物理地址）
# 例如观察 GITS_CTLR（偏移 0x0000）
devmem <ITS_PHYS_BASE>

# 观察 GITS_CBASER（偏移 0x0080）- 命令队列基址
devmem $((ITS_PHYS_BASE + 0x0080)) 64

# 观察 GITS_CREADR（偏移 0x0088）- 命令读取指针
devmem $((ITS_PHYS_BASE + 0x0088)) 64

# 观察 GITS_CWRITER（偏移 0x0090）- 命令写指针
devmem $((ITS_PHYS_BASE + 0x0090)) 64
```

### 5.7 常见问题排查清单

| 症状 | 可能原因 | 排查方向 |
|------|---------|---------|
| `pci_alloc_irq_vectors()` 返回 -ENOSPC | LPI 耗尽或 ITS 表空间不足 | 检查 `/proc/interrupts` 的 ITS 中断数；确认 `its_lpi_alloc()` 中 `lpi_range_list` 的状态 |
| MSI 中断不触发 | MAPTI 未正确发送或 DeviceID 错误 | 通过 ftrace 确认 `its_send_mapti` 被调用；检查 `pci_msi_domain_get_msi_rid()` 返回的 DevID 是否与硬件匹配 |
| 中断亲和性不生效 | MOVI 命令发送失败或 col_map 未更新 | 确认 `its_set_affinity()` 中的 `its_send_movi()` 被正确执行 |
| ITS 命令队列超时 | ITS 硬件未启用或命令队列空满 | 检查 `GITS_CTLR.Enabled` 位；观察 `GITS_CREADR` 与 `GITS_CWRITER` 的差值 |
| 设备初始化时 `pr_err("ITS@%pa: failed probing\n")` | `its_probe_one()` 中某个步骤失败 | 逐项排查：基地址映射 → 命令队列分配 → BASER 表分配 → Collection 分配 → 域初始化 |

---

## 附录：ITS 硬件命令队列工作机制补充

ITS 命令队列是一个**环形缓冲区（Circular Buffer）**，CPU 是生产者（写入命令），ITS 硬件是消费者（读取并执行命令）。

```
cmd_base                                                  cmd_base + ITS_CMD_QUEUE_SZ
  │                                                           │
  ├── [cmd0][cmd1][cmd2][...][cmdN] ──── (空闲) ──── [cmdM]...┤
  │                            │                     │        │
  │                       cmd_write               GITS_CREADR │
  │                       (SW 写指针)              (HW 读指针)  │
  └───────────────────────────────────────────────────────────┘
```

**命令发送的关键步骤（宏 `BUILD_SINGLE_CMD_FUNC` 生成）：**

1. `its_allocate_entry(its)`：检查队列是否满，分配一个命令槽位。如果队列满，等待 ITS 硬件消费
2. `builder(its, cmd, desc)`：调用命令构建函数（如 `its_build_mapti_cmd()`）编码命令
3. `its_flush_cmd(its, cmd)`：Cache Flush，确保命令对 ITS 硬件可见
4. 若 builder 返回了一个 `sync_obj`，则追加一条 SYNC 命令（确保前序命令执行完成）
5. `its_post_commands(its)`：更新 `GITS_CWRITER` 寄存器，通知 ITS 硬件有新命令
6. 等待命令完成（轮询 `GITS_CREADR`）
