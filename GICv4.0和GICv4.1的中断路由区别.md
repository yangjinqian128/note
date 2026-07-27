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

总结下各种表的关系
```
[ ITS 侧 - 全局级 (GITS) ]                 [ GICR 侧 - 局部/vCPU 级 (GICR_VLPI) ]

┌─────────────────────────┐                ┌──────────────────────────────┐
│       Device Table      │                │   vLPI Configuration Table   │
│   (GITS_BASER<n>, IDT)  │                │     (GICR_VPROPBASER)        │
└────────────┬────────────┘                └──────────────┬───────────────┘
             │                                            │
             ▼                                            │ (按 VM 或 vPEID 共享)
┌─────────────────────────┐                               │
│  Interrupt Trans Table  │ (ITT)                         │
│ (分配给特定 PCIe Device)│                               │
└────────────┬────────────┘                               │
             │                                            │
             ▼                                            ▼
┌─────────────────────────┐                ┌──────────────────────────────┐
│       vPE Table         │                │     vLPI Pending Table       │
│   (GITS_BASER<n>, VPT)  │───────────────>│     (GICR_VPENDBASER)        │
└─────────────────────────┘  (记录其指针)  └──────────────────────────────┘
                                                 (每 vPE 唯一私有)
```

这里还有个问题，多ITS的时候，这些表格是如何共享同步的？

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

