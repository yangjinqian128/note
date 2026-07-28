-v0.1 2026.7.27 Jinqian init

简介
----

gicv4.0相对于gicv3新增了vLPI直通，GICv4.1又在GICv4.0的基础上新增了vSGI的直通功能。
不过，GICv4.0和GICv4.1是完全不兼容的，架构上GICv4.1做了改进。这篇文章用于整理他们
之间的改进，背景及解决方案。

vLPI中断路由
----

直接从vLPI中断路由路径看两者之间的差异和改进。

**GICv4.0：**
- 外设发MSI写ITS_TRANSLATER，写入信息是eventid。ITS通过查device table获取ITT地址。
- ITT entry根据eventid返回vINTID和vPEID。
- ITS在vPE table 查vPEID对应的GICR和virtual pending table。`GITS_BASER<n>`存vPE table.
- ITS将 vINTID + vPEID + VPT 信息发给GICR。
- GICv4.0中，GICR_VPENDBASER记录此vPE的virtual pending table的基地址。如果GICR_VPENDBASER.valid==1且GICR_VPENDBASER.paddr和ITS转发过来的VPT相匹配，则认为VPE在位。
- - 如果vPE在位，GICR直接在VPT中标记pending。通过GICV注入虚拟中断。
- - 如果vPE不在位，GICR也会在VPT中标记pending。触发doorbell给pCPU，pCPU会进入hypervisor调度
vCPU上位。
- guest读IAR寄存器后，会触发GICR先将自身的**缓存**中的VPT表置为0，代表中断状态从pending变为active或active&pending。后续会写回内存。

这里延迟写回VPT不会出问题是因为，只要这个vPE在位，中断来了之后都是优先看缓存的。在vcpu下位的时候
hypersivor会写GICR_VPENDBASER.valid=0，此时会强制硬件将缓存写回内存。

总结下GICv4.0各种表的关系
```
========================================================================================
                               GICv4.0 内存结构架构图
========================================================================================

 [ GITS (ITS 侧) - 全局私有表 ]

 GITS_BASER<n> ────────┐
                       ▼
             ┌─────────────────────────┐
             │    ITS Device Table     │
             └────────────┬────────────┘
                          │ (DeviceID 索引)
                          ▼
             ┌─────────────────────────┐
             │ Interrupt Trans Table   │ (ITT, 按设备独占)
             └────────────┬────────────┘
                          │ (EventID 索引)
                          ▼
             ┌─────────────────────────┐
             │     ITS vPE Table       │ (ITS 私有全局表)
             │ ┌─────────────────────┐ │
             │ │ vPEID=0: VPT_addr_0 │ │ ──┐
             │ ├─────────────────────┤ │   │
             │ │ vPEID=1: VPT_addr_1 │ │ ──┼──┐ (记录每个 vPE 独占的 VPT 物理地址)
             │ └─────────────────────┘ │   │  │
             └─────────────────────────┘   │  │
                                           │  │
───────────────────────────────────────────┼──┼─────────────────────────────────────────
 [ Memory / DRAM (物理内存区) ]            │  │
                                           │  │
   VM 1 共享内存区                          │  │
   ┌───────────────────────────────────┐   │  │
   │    vLPI Configuration Table      │   │  │ (存放优先级 Priority & 使能 Enable)
   └─────────────────▲─────────────────┘   │  │
                     │                     │  │
   vPE 私有内存区    │                     │  │
   ┌─────────────────┴─────────────────┐   │  │
   │   vLPI Pending Table 0 (vPT_0)    │◄──┘  │ (vPE_0 独占, bit 记录 Pending 状态)
   └───────────────────────────────────┘      │
   ┌───────────────────────────────────┐      │
   │   vLPI Pending Table 1 (vPT_1)    │◄─────┘ (vPE_1 独占)
   └───────────────────────────────────┘
                     ▲
─────────────────────┼──────────────────────────────────────────────────────────────────
 [ GICR (Redistributor 侧) - 无全局表，纯寄存器驱动 ]
                     │
 GICR_VPROPBASER ────┘ (调度切入 vPE_0 时，KVM 写入 VM 共享的 Config Table 地址)
 GICR_VPENDBASER ──────(调度切入 vPE_0 时，KVM 写入 vPT_0 的物理基地址 VPT_addr_0)
 ```

**GICv4.1：**
- 外设发MSI写ITS_TRANSLATER，写入信息是eventid。ITS通过查device table获取ITT地址。
- ITT entry根据eventid返回vINTID和vPEID。
- ITS在**vPE configuration table**查vPEID对应的GICR。`GITS_BASER<n>`存vPE table.
- ITS将 vINTID + vPEID信息发给GICR。
- **GICR会将vPEID和GICR_VPENDBASER中保存的vPEID进行对比**。GICv4.1中，GICR_VPENDBASER记录在位的vPEID。如果GICR_VPENDBASER.valid==1且vPEID对比一致，则认为VPE在位。
- - 如果vPE在位，GICR直接在VPT中标记pending。通过GICV注入虚拟中断。
- - 如果vPE不在位，GICR也会在VPT中标记pending。触发doorbell给pCPU，pCPU会进入hypervisor调度
vCPU上位。
- guest读IAR寄存器后，会触发GICR先将自身的**缓存**中的VPT表置为0，代表中断状态从pending变为active或active&pending。后续会写回内存。

GICv4.1把 ITS 侧的私有 vPE Table 升级并重构为全局唯一的 vPE Configuration Table，同时让 ITS（通过 GITS_BASER2） 和 GICR（通过 GICR_VPROPBASER） 的基地址寄存器同时指向物理内存中的这同一张表。

GICv4.1各种表的关系：
```
========================================================================================
                               GICv4.1 内存结构架构图
========================================================================================

 [ GITS (ITS 侧) ]                        [ GICR (Redistributor 侧) ]
    GITS_BASER2 ──────────────┐              ┌────────────── GICR_VPROPBASER
                              │              │
                              ▼              ▼
──────────────────────────────┼──────────────┼──────────────────────────────────────────
 [ Memory / DRAM (全局共享数据区) ]
                              │              │
                              ▼              ▼
                     ┌───────────────────────────────────┐
                     │    vPE Configuration Table    │  (全局共享表)
                     │ (indexed directly by vPEID)       │
                     └─────────────────┬─────────────────┘
                                       │
     ┌─────────────────────────────────┴─────────────────────────────────┐
     │                                                                   │
     ▼ Entry 0 (vPEID=0)                                                 ▼ Entry 1 (vPEID=1)
┌─────────────────────────────────────────┐             ┌─────────────────────────────────────────┐
│ [Word 0]                                │             │ [Word 0]                                │
│  • Valid (1b)                           │             │  • Valid (1b)                           │
│  • VPT_addr ────────┐                   │             │  • VPT_addr ────────┐                   │
│  • vIDBits / Size   │                   │             │  • vIDBits / Size   │                   │
│ [Word 1]            │                   │             │ [Word 1]            │                   │
│  • VCONF_addr ──────┼────────┐          │             │  • VCONF_addr ──────┼────────┐          │
│  • Target GICR ID   │        │          │             │  • Target GICR ID   │        │          │
│  • Doorbell pINTID  │        │          │             │  • Doorbell pINTID  │        │          │
│  • DBE (Enable)     │        │          │             │  • DBE (Enable)     │        │          │
└─────────────────────┼────────┼──────────┘             └─────────────────────┼────────┼──────────┘
                      │        │                                              │        │
                      │        │                                              │        │
──────────────────────┼────────┼──────────────────────────────────────────────┼────────┼────────
 [ Memory / DRAM (挂起表与配置表) ]                                            │        │
                      │        │                                              │        │
                      ▼        │                                              │        │
   ┌──────────────────────┐    │                            ┌─────────────────┼────┐   │
   │ vLPI Pending Table 0 │    │                            │ vLPI Pending Table 1 │   │
   │  (vPE_0 独占 vPT)     │    │                            │  (vPE_1 独占 vPT)     │   │
   └──────────────────────┘    │                            └──────────────────────┘   │
                               ▼                                                       ▼
   ┌──────────────────────────────────────────────────────────────────────────────────────┐
   │                       vLPI Configuration Table (VM 共享)                             │
   └──────────────────────────────────────────────────────────────────────────────────────┘
```

在 GICv4.0 规范中，并没有全局统一的 vPE 表。ITS 侧有一个私有的 vPE Table（存 vPEID 到 VPT_addr 的映射），而 GICR 侧则完全依赖 Hypervisor 动态写入的 GICR_VPENDBASER / GICR_VPROPBASER 本地寄存器。

导致的问题;
1. 每次外设触发 vLPI，ITS 在向 GICR 转发中断请求时，报文中必须携带巨大的物理地址信息：(vPEID, vINTID, VPT_addr)。在现代高并发数据中心中，海量的中断报文频繁携带 64-bit 的物理地址，对 GIC 内部互联总线（Interconnect/IRI）造成了巨大的物理带宽浪费。
2. 当 Hypervisor 将一个 vCPU (vPE) 从物理 CPU-A 调度迁移到物理 CPU-B 时，因为 ITS 侧的私有 vPE Table 记录了该 vPE 对应的物理 GICR 路由，Hypervisor 必须向 ITS 命令队列发送 VMOVP 指令来更新 ITS 私有表中的目标 GICR 节点。
3. 当 vPE 在位（Resident）状态发生变化或需要触发 Doorbell 唤醒时，Hypervisor 需要手动遍历并更新大量分散的状态，软件逻辑极其复杂。一旦刷盘（Flush）时机或寄存器写入时序出现微小偏差，极其容易导致“中断 Pending 位丢失”或“Doorbell 被错误重复触发”。

这些问题都在GICv4.1得到解决：
1. GICR知道全局共享表，只需要vPEID就可以索引到VPT_addr。
2. vPEID 到物理 GICR 的映射关系直接保存在共享表的 Entry 中。Hypervisor 在做 vCPU 迁移时，硬件支持直接去修改共享表项，或者使用硬件级/单指令的轻量级 VMOVP。
3. 在共享表的 Entry 中，直接集成存放了该 vPE 专属的 Doorbell pINTID 和使能状态 DBE。当外设发来 vLPI 时，ITS 查全局共享表：
- - 在位（Resident）： 硬件直接注入 vIRQ，Doorbell 压根不触发。
- - 不在位（Non-Resident）： 硬件自动读取表里的 DB_pINTID 抛出 Doorbell，并且硬件会自动将该 vPE 的 DBE (Doorbell Enable) 位自动清零（防抖），防止后续的中断继续轰炸 Host。

GICv4.1本质是用“共享内存”取代“跨硬件消息”，将分散的 vPE 上下文收拢为全局唯一的内存真理源（Single Source of Truth）——Hypervisor 仅需以轻量级内存写去更新映射，而 ITS 和 GICR 硬件则通过 vPEID 查表实现低延迟的路由、直投和 Doorbell 自动化。