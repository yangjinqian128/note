# arm-smmu-v3 系统挂起/恢复分析

> 范围:物理 SMMUv3 在 deep 挂起下的状态丢失与恢复支持情况。本文从《kvm-arm64-suspend-resume-analysis》§五·5 拆分而来——SMMU 属于 IOMMU 子系统,与 KVM 无代码关系,独立成篇。配套阅读:KVM 笔记 §五·5(VFIO 侧的设备状态难题)。

## 一、问题定位

SMMU 的硬件状态分两类,挂起时命运不同:

| 状态 | 挂起时 | 后果 |
|---|---|---|
| 页表(Stream Table / STE / CD / Stage 表) | 内存,DRAM 自刷新保留 | 数据不丢 |
| 寄存器(SMMU_CR0、Stream Table Base、队列基址) | 随掉电归零 | 指针丢失 |
| 内部缓存(STE/TLB) | 丢失 | 可重建,性能损失 |

**危害形态**:复位态下 `SMMUEN=0` → SMMU 旁路(IOVA 被当物理地址直写内存,**静默数据损坏**);若固件/系统配置了 `GBPA.AbortAll` 则表现为 DMA 全部 abort(设备超时)。两种形态都意味着直通设备的数据面失效,且挂起/唤醒本身**不报错**。

## 二、当前支持状态:mainline 无,补丁三轮未合入

- **本树**:`arm_smmu_driver`(arm-smmu-v3.c:5651-5659)只有 `probe/remove/shutdown`,无 `.pm` 字段、无 syscore/cpu_pm 注册;
- **上游补丁史**:
  - 2021 初版(驱动内缓存 `msi_msg` + 恢复寄存器):Marc Zyngier 打回——MSI 消息缓存应在 genirq 核心做,不应在驱动;
  - 2024-03 Peng Fan `[PATCH 0/3] iommu/smmu-v3: support suspend/resume`(i.MX95 测试,MSI 部分未测,评审停滞);
  - 2026-05 `[PATCH v7 00/11] Implement Runtime/System Sleep ops`(最全面:**仍在评审,未合入**);
- Xen 生态确认:arm-smmu-v3 驱动的 suspend 处理返回 `-ENOSYS`(未实现)。

## 三、真实复杂度:从 v7 补丁看恢复不是"重写寄存器"

```
挂起序列(多阶段):
  停流量:SMMUEN=0 + GBPA=Abort      ← 防止复位后"旁路乱写",主动转 abort
  → gate/flush CMDQ(命令队列门控,CMDQ_PROD_STOP_FLAG)
  → 排水(处理在途事务)
  → 软件静止
恢复序列:
  完整 device reset → 重写 Stream Table Base 等寄存器 → 重建队列
  → MSI 消息重写 → 失效缓存
```

## 四、MSI 是核心难题(Thomas Gleixner,2024-03 讨论)

- MSI 消息**只在中断激活时写一次**;之后的 affinity 变更只更新 ITS 表,不会重写消息;
- resume 后消息可能丢失 → 需要在 genirq 核心加 `IRQD_RESUMING` 标志,在核心恢复路径强制重写消息;
- **结论:SMMU 的挂起恢复不是驱动单方面能完成的,依赖 genirq 核心配合**——这也是多轮补丁迟迟未合入的结构性原因。

## 五、恢复顺序硬依赖

```
resume(顺序不可颠倒):
  ① SMMU reset + 寄存器恢复 + MSI 消息重写   ← 翻译硬件先活
  ② IOMMU domain 重新生效(页表在内存)
  ③ 设备 resume(重新初始化、重新使能 DMA 与中断)
     ↑ 设备一活就开始 DMA,①必须在此之前完成
```

## 六、vSMMU 两种形态与挂起的关系

- **纯软件模拟**(QEMU 模拟 SMMU + virtio 模拟设备):DMA 不经过物理 SMMU,挂起**无影响**;
- **硬件辅助嵌套**(guest 跑真 SMMU 驱动,本树 arm-smmu-v3.c:3117 有 `vsmmu->s2_parent` 嵌套 domain 支持):guest 表(guest RAM)与 host stage-2(host 内存)都保留,但物理 SMMU 寄存器与"设备→嵌套 domain→stage-2 父级"挂接关系照丢——**同根因,恢复链更长**;
- s2idle 下电源不断,寄存器保持,直通可用;**deep 是分水岭**。

## 七、结论

1. SMMU 侧是**可解的工程问题**(页表在内存,恢复=重指寄存器+重建队列+MSI 重写),但上游 mainline 至今未合入,且依赖 genirq 核心的 `IRQD_RESUMING` 机制;
2. 在补丁合入前,带直通设备的节点做 deep 挂起的代价是"唤醒后 DMA 数据面失效(静默损坏或全 abort)";
3. 与 VFIO 侧(设备状态不可保存,架构性难题)叠加后,直通场景短期内只有"阻止挂起"或"guest 配合 quiesce/重初始化"两条路。

## 参考链接

- [iommu/arm-smmu-v3: Implement Runtime/System Sleep ops(v7 系列,未合入)](https://lwn.net/Articles/1075131/)
- [iommu/smmu-v3: support suspend/resume(Peng Fan 2024,评审停滞,含 Thomas Gleixner 的 MSI 核心障碍讨论)](https://lkml.indiana.edu/hypermail/linux/kernel/2403.3/00091.html)
- [Re: [PATCH 3/3] iommu/arm-smmu-v3: support suspend/resume(Thomas Gleixner 的 MSI 消息恢复分析)](http://yhbt.net/lore/all/87msqkeotk.ffs@tglx/)

---

*分析时间:2026-09-10。基于本树 v7.3-rc1。上游补丁状态以分析当日为准。*
