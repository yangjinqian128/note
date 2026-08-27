# GICv5 内存表与条目创建时间线笔记

> GICv5 全部内存结构（表 + 条目）的创建时机与调用栈。
> 物理部分为主线已合入代码；虚拟化部分为规范定义（KVM IRS 系列未合入，标注 [规范]）。
> 内核主线 master (fc46aed51f6)，规范 ARM-AES-0070 00bet2。

---

## 一、总表：全部内存结构与创建时机

| # | 结构 | 每什么一份 | 创建时机 | 创建者 | 主线状态 |
|---|------|-----------|---------|--------|---------|
| 1 | **物理 LPI IST**（表） | 每中断域一张（所有 IRS 共享） | 内核启动 | `gicv5_irs_enable` | ✅ 已合入 |
| 2 | **IST L2 表**（二级模式） | 按需 | 每条 LPI 首次分配 | `gicv5_irs_iste_alloc` | ✅ 已合入 |
| 3 | **ISTE**（L2 条目） | 每 LPI 一个 | LPI 分配时配置生效 | lpi_domain_alloc | ✅ 已合入 |
| 4 | **ITS DT**（表） | 每 ITS 一张 | ITS probe | `gicv5_its_init_devtab` | ✅ 已合入 |
| 5 | **DT L2 表**（二级模式） | 按需 | 设备注册时 | `gicv5_its_alloc_l2_devtab` | ✅ 已合入 |
| 6 | **DTE** | 每设备一个 | 设备首次申请 MSI | `gicv5_its_device_register` | ✅ 已合入 |
| 7 | **ITT**（表） | 每设备一张 | 同 DTE 时刻 | 同上 | ✅ 已合入 |
| 8 | **ITTE** | 每 EventID 一个 | 每条 MSI activate | `gicv5_its_map_event` | ✅ 已合入 |
| 9 | **VM Table**（表） | 每 IRS 域一张（所有 IRS 共享） | VM 创建 | hypervisor [规范] | ❌ KVM IRS 系列 |
| 10 | **L2_VMTE** | 每 VM 一个 | VM 创建 | hypervisor [规范] | ❌ 同上 |
| 11 | **VPE Table**（表） | 每 VM 一张 | VM 创建 | hypervisor [规范] | ❌ 同上 |
| 12 | **VPETE** | 每 VPE 一个 | VPE 创建 | hypervisor [规范] | ❌ 同上 |
| 13 | **VPE Descriptor** | 每 VPE 一个 | VPE 创建（零初始化） | hypervisor [规范] | ❌ 同上 |
| 14 | **VM Descriptor** | 每 VM 一个（可选，IRS_IDR3.VMD） | VM 创建 | hypervisor [规范] | ❌ 同上 |
| 15 | **Virtual LPI IST**（表） | 每 VM 一张（所有 IRS 共享） | VM 创建 / guest 配置时 provision | hypervisor [规范] | ❌ 同上 |
| 16 | **Virtual SPI IST**（表） | 每 VM 一张（所有 IRS 共享） | guest 首跑前 provision | hypervisor [规范] | ❌ 同上 |
| 17 | **doorbell LPI 的 ISTE** | 每 VPE 一个（专属物理 LPI） | VPE 创建 | hypervisor [规范] | ❌ 同上 |

**不在内存中的**（对照）：SPI 状态（IRS 内部寄存器）、PPI 状态（PE 系统寄存器）——只有 LPI 系和虚拟化系需要内存表。

---

## 二、物理部分调用栈（主线代码）

### 2.1 物理 LPI IST：内核启动时一次性创建

```
gicv5_irs_enable()                              irs.c:828  ★ 入口
  └─ gicv5_irs_init_ist()                       irs.c:292
       ├─ 读 IRS_IDR2: 硬件上下限 / ISTMD 元数据  :294-346
       ├─ 选 LPI_ID_BITS / ISTSZ / L2SZ / 结构
       ├─ 线性: gicv5_irs_init_ist_linear()      irs.c:64
       │    └─ 分配整表（zero 初始化）→ IRS_IST_CFGR + IRS_IST_BASER(VALID=1)
       │       → 轮询 IRS_IST_STATUSR.IDLE
       └─ 二级: gicv5_irs_init_ist_two_level()   irs.c:127
            └─ 分配 L1 → 同样 CFGR + BASER + IDLE 轮询
```

**创建后软件不可再直接访问**（VALID=1 起由 IRS 接管）——所有后续更新走 GIC 系统指令。

### 2.2 IST L2 表 + ISTE：每条 LPI 分配时按需

```
request_irq / IPI 初始化
  └─ lpi_domain.ops->alloc                        irq-gic-v5.c:849 (tree domain)
       ├─ ida_alloc_max() 分配 LPI INTID          ← 硬件号诞生
       ├─ gicv5_irs_iste_alloc(lpi)               irs.c:189  ★ L2 表按需
       │    ├─ 二级模式才做事（!ist.l2 直接 return）
       │    ├─ index = lpi >> l2bits，查 L1 条目的 VALID
       │    └─ 不 VALID → kzalloc L2 表 + 写 L1 条目（L2 基址 + VALID）
       ├─ irq_domain_set_info(...)                ← irq_data 绑定
       └─ gicv5_hwirq_init(): CDPRI + CDAFF       ← ISTE 配置生效
```

**关键点**：ISTE 不是"创建"出来的——L2 表 kzalloc 即全零（= disabled/无路由），"条目激活"就是 hwirq_init 的两条 GIC 指令（CDPRI/CDAFF）。

### 2.3 DT：ITS probe 时一次性

```
gicv5_its_probe()                                its.c:1215
  └─ gicv5_its_init(np)                          its.c:1152
       └─ gicv5_its_init_devtab(its_node)        its.c:687  ★
            ├─ 线性: kcalloc(2^device_id_bits)    its.c:596
            ├─ 二级: kcalloc(L1)                  its.c:652
            ├─ 写 ITS_DT_CFGR（结构/L2SZ/位数）    its.c:606 / :671
            └─ 写 ITS_DT_BASER（物理基址）         its.c:609 / :674
```

### 2.4 DTE + ITT：设备首次申请 MSI 时（同一函数内）

```
设备驱动调 pci_alloc_irq_vectors()
  └─ msi_domain_alloc_irqs()
       └─ msi_domain_ops.msi_prepare = gicv5_its_msi_prepare   its.c:842
            └─ gicv5_its_alloc_device(its, nvec, dev_id)       its.c:811
                 └─ gicv5_its_device_register()                its.c:459  ★
                      ├─ 分配 ITT（此刻发生）:
                      │     线性: itt_cfg.linear.itt = kcalloc  its.c:157-177
                      │     二级: l1itt + l2ptrs                its.c:224
                      ├─ (二级 DT) gicv5_its_alloc_l2_devtab    its.c:384
                      │    └─ L1 不命中 → kcalloc L2             its.c:406
                      ├─ 写 DTE = {ITT 基址, EVENT_ID_BITS, VALID=1}
                      └─ ITS_INV_DEVICER + 轮询 ITS_STATUSR.IDLE
```

### 2.5 ITTE：每条 MSI activate 时

```
msi_domain_alloc_irqs() 的 activate 阶段
  └─ its_msi_domain.ops->activate = gicv5_its_irq_domain_activate  its.c:1007
       ├─ event_id = FIELD_GET(EVENT_ID, d->hwirq)
       ├─ lpi = d->parent_data->hwirq      ← 从父域（LPI domain）取真实 LPI INTID
       └─ gicv5_its_map_event(its_dev, event_id, lpi)              its.c:846  ★
            ├─ 检查 ITTE 未占用（VALID → -EEXIST）
            ├─ 写 ITTE = {LPI_ID, VALID}
            └─ ITS_INV_EVENTR 缓存失效 + 轮询 IDLE
```

---

## 三、虚拟化部分（规范定义，KVM IRS 系列未合入）[规范]

> 以下调用栈按规范 6.2/6.3 的寄存器协议描述，主线无代码可引，标记 [规范]。

### 3.1 VM 创建时：五张表一次配齐

```
hypervisor 创建 VM:
  ① 分配 VM Descriptor（若 IRS_IDR3.VMD=1）并零初始化
  ② 分配 VPE Table（每 VM 一张）
  ③ 分配 Virtual LPI IST + Virtual SPI IST（每 VM 各一张）
  ④ 写 L2_VMTE:
       L2_VMTE.VPE_TABLE_ADDR = VPE Table 基址
       L2_VMTE.LPI_IST_ADDR  = vLPI IST 基址（+ STRUCTURE/LPI_ID_BITS）
       L2_VMTE.SPI_IST_ADDR  = vSPI IST 基址（+ SPI_ID_BITS）
  ⑤ 使能 VM Table:
       写 IRS_VMT_BASER.ADDR（VM Table 基址）
       写 IRS_VMT_BASER.VALID 0→1 → 轮询 IRS_VMT_STATUSR.IDLE
       （VALID=1 起 IRS 才被允许访问全部虚拟化数据结构）
```

**二级 VM Table 的按需 L2**（规范 6.2 三步协议）：

```
软件:  L1_VMTE.L2_ADDR = 新 L2 表地址（VALID 保持 0）
软件:  IRS_VMAP_L2_VMTR.M = 1
硬件:  IRS 回写 L1_VMTE.VALID 0→1 + IDLE=1 报告完成
```

### 3.2 VPE 创建时：三项

```
hypervisor 创建 VPE:
  ① 分配 VPE Descriptor: kzalloc(IRS_IDR4.VPED_SZ)，必须全零
     （创建后不可搬动——IRS 往里面记账，规范 IYWLWM/RDLKTX）
  ② 写 VPETE（经 IRS_VMAP_VPER）→ IRS 回写 VALID
  ③ 分配专属 doorbell 物理 LPI + 配置其 ISTE（路由到调度 CPU）
     写 IRS_VPE_DBR.INTID（门铃设置生效）
```

### 3.3 vLPI / vSPI IST 的两种 provision 时机

规范 Fundamentals 6.3.1 原话：

- **vSPI IST**：*"The SPI IST is provisioned by the hypervisor **before the first run of the guest**"*——VM 创建时一次配齐
- **vLPI IST**：*"The LPI IST is provisioned (or wrapped) by the hypervisor **in response to the guest configuring the LPI IST** via the emulated IRS Memory Mapped IO interface"*——guest 启动后自己配置 LPI IST（写模拟 MMIO）时才 provision/wrap，因为 vLPI 数量（LPI_ID_BITS）由 guest 决定

```
vLPI IST 使能（规范 6.3.1）:
  hypervisor 写 IRS_VMAP_VISTR
    → IRS 回写 L2_VMTE.LPI_IST_VALID 0→1 + IDLE=1

二级 vLPI IST 按需 L2（IRS_VMAP_L2_VISTR.TYPE=LPI）:
  与 VM Table L2 同样的"软件写地址 → IRS 回写 VALID"协议
```

### 3.4 doorbell LPI 的 ISTE：复用物理 IST 的普通条目

doorbell 不是新表——就是**物理 LPI IST 里的一个普通条目**（RKDQNS: SET_EDGE targeting physical LPIs），VPE 创建时分配专属 LPI 并配置 Targeted 路由，约束：不得与 ITS 映射或他人门铃共用（RDHHRM）。

---

## 四、创建时机全景时间轴

```
════════ 内核启动 ════════════════════════════════════════════════
  gicv5_irs_enable → gicv5_irs_init_ist        ★ 物理 LPI IST（每域一张）
  gicv5_its_probe → gicv5_its_init_devtab      ★ ITS DT（每 ITS 一张）
  （SPI/PPI 无内存表）

════════ 每条 LPI 分配（request_irq / IPI 初始化）════════════════
  lpi_domain_alloc → gicv5_irs_iste_alloc      ★ IST L2 表（按需）
                   → gicv5_hwirq_init          ★ ISTE 配置生效（CDPRI/CDAFF）

════════ 设备首次申请 MSI（pci_alloc_irq_vectors）═══════════════
  gicv5_its_msi_prepare → alloc_device → device_register
                   → kcalloc ITT               ★ ITT（每设备一张）
                   → 写 DTE + INV_DEVICER       ★ DTE
                   → alloc_l2_devtab（按需）     ★ DT L2 表

════════ 每条 MSI activate ══════════════════════════════════════
  gicv5_its_irq_domain_activate → map_event
                   → 写 ITTE + INV_EVENTR       ★ ITTE

════════ VM 创建（hypervisor）[规范] ════════════════════════════
                   ★ VM Table / L2_VMTE / VPE Table
                   ★ vSPI IST（guest 首跑前）/ VM Descriptor（可选）

════════ VPE 创建（hypervisor）[规范] ═══════════════════════════
                   ★ VPETE / VPE Descriptor（零初始化）
                   ★ doorbell LPI 的 ISTE

════════ guest 运行中配置 LPI IST（模拟 MMIO）[规范] ════════════
                   ★ vLPI IST 的 provision/wrap（IRS_VMAP_VISTR）
```

---

## 五、规律总结

1. **表级一次性、条目级按需**：DT/IST/VM Table 在"组件初始化"时建骨架（全 invalid/零），DTE/ISTE/ITTE/L2_VMTE/VPETE 在"对象注册"时填充
2. **L2 表全部按需分配**：物理 IST L2（iste_alloc）、DT L2（alloc_l2_devtab）、VM Table L2（IRS_VMAP_L2_VMTR）——统一的"L1 条目不 VALID → 软件分配 L2 → 硬件回写 VALID"协议，这正是"Split/Span 支持稀疏 ID 空间"的落地
3. **硬件写 VALID 的模式**：软件只提供"地址 + 配置"，VALID 位常由**硬件回写**（VPETE、L2_VMTE.LPI_IST_VALID、L1 条目的 VALID）或**软件置位后轮询 IDLE 确认**（IST_BASER、VMT_BASER）——内存表的"生效"全部有明确的完成信号
4. **虚拟表的 provision 时机不对称**：vSPI 表在 guest 首跑前（数量由 hypervisor 定），vLPI 表在 guest 配置时（数量由 guest 定）——因为 vLPI 的 LPI_ID_BITS 是 guest 的决策
5. **主线边界**：物理侧 8 项全部已合入（代码可查）；虚拟侧 9 项全部依赖 KVM IRS 系列（v4, 48 patches 未合入），当前只有规范协议可依

---

*笔记时间：2026-08-27*
*内核版本：master (fc46aed51f6)*
*证据文件：irq-gic-v5-irs.c（IST/L2/ISTE）、irq-gic-v5-its.c（DT/DTE/ITT/ITTE）、规范 §4.7/§4.9/§6.2/§6.3、Fundamentals 5.2/6.2/6.3*

## 参考链接

**内核代码：**
- [irq-gic-v5-irs.c — IST 创建与 iste_alloc](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/irqchip/irq-gic-v5-irs.c)
- [irq-gic-v5-its.c — DT/DTE/ITT/ITTE 创建链](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/irqchip/irq-gic-v5-its.c)

**规范：**
- [Arm GICv5 Architecture Specification (AES0070) — §4.7 IST / §4.9 虚拟化数据结构 / §6.x](https://developer.arm.com/documentation/aes0070)
- [Learn the Architecture: GICv5 Overview (111471) — §5.2 ITS / §6.2-6.3 虚拟化](https://developer.arm.com/documentation/111471)

**邮件列表：**
- [KVM: arm64: Add GICv5 IRS support (v4) — 虚拟侧表创建代码所在](https://lists.infradead.org/pipermail/linux-arm-kernel/2026-July/)

**关联笔记：**
- [GICv5 架构与实现笔记（1.5 关键数据结构 / 3.2 初始化流程）](gic-v5-arch-note.md)
- [内核中断子系统数据结构全景（表挂在哪些结构体上）](kernel-irq-struct-note.md)
