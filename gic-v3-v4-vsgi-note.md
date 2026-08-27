# GICv3 / GICv4.1 vSGI 注入路径内核实现笔记

> vSGI（虚拟 SGI）在 GICv3/v4.0 是"List Register 软件注入"，在 GICv4.1 是"GITS_SGIR 硬件注入"。
> 两者共用同一 trap 解析入口，内核在 `vgic_v3_queue_sgi` 按 `irq->hw` 分流。
> 内核主线 master (fc46aed51f6)。

---

## 一、背景与总览

SGI（Software Generated Interrupt，INTID 0-15）是 GICv3 的核间中断：guest 写 `ICC_SGI1R_EL1` 生成。虚拟化场景下 vSGI 的注入有两条实现路径：

```
Guest 写 ICC_SGI1R_EL1
  │
  │  trap 到 EL2（两代共用，见第二节）
  ▼
vgic_v3_dispatch_sgi / vgic_v3_queue_sgi
  │
  ├─ irq->hw == false ──► GICv3 / GICv4.0 路径（第三节）
  │     pending_latch → AP list → 目标 vcpu 进入时写 List Register
  │
  └─ irq->hw == true ───► GICv4.1 路径（第四节）
        irq_set_irqchip_state(PENDING) → 写 GITS_SGIR → ITS 硬件注入
```

**关键前提**：GICv4.1 才支持硬件 vSGI。`its_init_v4()` 里 `sgi_ops` 仅当 `has_v4_1` 时挂载（its.c:5865-5868），GICv4.0 系统 `sgi_ops = NULL`，回落到 GICv3 的 LR 软件路径。

---

## 二、共同入口：guest SGI 指令的 trap 与解析

### 2.1 trap 入口（sys_regs.c）

guest 在 EL1 执行 `ICC_SGI1R_EL1` / `ICC_SGI0R_EL1` / `ICC_ASGI1R_EL1` 会 trap 到 EL2（虚拟 CPU 接口不实现 SGI 寄存器），sysreg 表路由到：

```c
// arch/arm64/kvm/sys_regs.c:613 access_gic_sgi
// AArch64: Op2 5 = ICC_SGI1R_EL1 (g1=true), 6 = ASGI1R, 7 = SGI0R (g1=false)
// AArch32: Op1 0 = SGI1R, 1 = ASGI1R, 2 = SGI0R
vgic_v3_dispatch_sgi(vcpu, p->regval, g1);   // g1: 是否允许生成 Group1 SGI
```

### 2.2 解析目标 vcpu 列表（vgic-mmio-v3.c:1101）

```c
void vgic_v3_dispatch_sgi(struct kvm_vcpu *vcpu, u64 reg, bool allow_group1)
{
	sgi = FIELD_GET(ICC_SGI1R_SGI_ID_MASK, reg);

	/* Broadcast（IRM 路由模式位）: 全部 vcpu 除发送者 */
	if (unlikely(reg & BIT_ULL(ICC_SGI1R_IRQ_ROUTING_MODE_BIT))) {
		kvm_for_each_vcpu(c, c_vcpu, kvm) {
			if (c_vcpu == vcpu)
				continue;
			vgic_v3_queue_sgi(c_vcpu, sgi, allow_group1);
		}
		return;
	}

	/* Targeted: Aff{3,2,1} + 16 位 Aff0 target list → kvm_mpidr_to_vcpu */
	mpidr = SGI_AFFINITY_LEVEL(reg, 3) | ...;
	target_cpus = FIELD_GET(ICC_SGI1R_TARGET_LIST_MASK, reg);
	for_each_set_bit(aff0, &target_cpus, ...) {
		c_vcpu = kvm_mpidr_to_vcpu(kvm, mpidr | aff0);
		if (c_vcpu)
			vgic_v3_queue_sgi(c_vcpu, sgi, allow_group1);
	}
}
```

注意 `ICC_SGI1R_EL1` 的 affinity 编码与 MPIDR 不同（`SGI_AFFINITY_LEVEL` 宏做移位对齐，:1043-1046），且 16 位 target list 天然支持**硬件语义的 1→N 广播**（一条 trap 内遍历多个目标）。

### 2.3 分流点（vgic-mmio-v3.c:1053）

```c
static void vgic_v3_queue_sgi(struct kvm_vcpu *vcpu, u32 sgi, bool allow_group1)
{
	...
	if (!irq->hw) {
		irq->pending_latch = true;                  // ★ v3/v4.0: 软件注入
		vgic_queue_irq_unlock(vcpu->kvm, irq, flags);
	} else {
		/* HW SGI? Ask the GIC to inject it */
		irq_set_irqchip_state(irq->host_irq,        // ★ v4.1: 硬件注入
				      IRQCHIP_STATE_PENDING, true);
		...
	}
}
```

`irq->hw` 只在 GICv4.1 的 `vgic_v4_enable_vsgis()` 里被置 true（见 4.1）——这一位就是两条路径的开关。

---

## 三、GICv3 / v4.0 路径：List Register 软件注入

### 3.1 数据流

```
vgic_v3_queue_sgi():
  irq->pending_latch = true          ← 软件影子状态
  vgic_queue_irq_unlock()            ← 挂 AP list（ap_list） + kick 目标 vcpu
       │
目标 vcpu 下次被调度进入 guest 前:
  vgic_flush_lr_state()              ← 遍历 AP list, 把 pending 的 vSGI 写入
       │                                ICH_LR*（List Register: INTID/priority/pending/EOI 位）
       ▼
虚拟 CPU 接口按 LR 呈现 vSGI:
  guest 的 IAR 读到 INTID → handler → EOI/DIR（围绕 LR 状态机工作）
  vgic_v3_fold_lr_state()            ← 退出时回读 LR 状态到影子
```

### 3.2 三个固有代价

1. **注入时机依赖调度**：置 pending 只是"记一笔 + 踢一脚"，vSGI 真正对 guest 可见要等目标 vcpu 被调度进来执行 flush——目标 vcpu 长时间不运行（如被抢占），vSGI 就延迟投递
2. **LR 槽位消耗**：每个 pending vSGI 占用一个 List Register——LR 是实现有限的资源（虚拟接口深度），vSGI 密集场景（如 guest 内多核 IPI 风暴）会挤占 vPPI/vLPI 的 LR 预算
3. **EOI 语义走 AP list**：`vgic_v3.c:41-42` 中 `if (!als->nr_sgi) cpuif->vgic_hcr |= ICH_HCR_EL2_vSGIEOICount;`——vSGI 的 EOI 计数与 AP list 状态耦合，无硬件 vSGI 时需要软件精确跟踪 deactivate 计数

### 3.3 GICv4.0 与 v3 完全相同

`has_gicv4_1` 是唯一门控：`vgic_v3_map_resources` 里 `if (kvm_vgic_global_state.has_gicv4_1) vgic_v4_configure_vsgis(kvm)`（vgic-v3.c:785-786）——GICv4.0 系统不创建 sgi_domain（its.c:5865-5868 置 NULL），所有 vSGI 走上述 LR 路径。

---

## 四、GICv4.1 路径：GITS_SGIR 硬件注入

### 4.1 一次性初始化：vSGI 伪装成宿主硬件中断

```c
// vgic-v4.c:190 vgic_v4_configure_vsgis → 每 vcpu 调 vgic_v4_enable_vsgis (:115)
/*
 * With GICv4.1, every virtual SGI can be directly injected. So
 * let's pretend that they are HW interrupts, tied to a host IRQ.
 */
for (i = 0; i < VGIC_NR_SGIS; i++) {
	irq->hw = true;
	irq->host_irq = irq_find_mapping(vpe->sgi_domain, i);   // 每 vSGI 绑宿主虚 IRQ

	vgic_v4_sync_sgi_config(vpe, irq);   // :108 —— enabled/group/priority
	                                     //   同步进 vpe->sgi_config[]（vSGI 配置表）
	irq_domain_activate_irq(...);        // 激活宿主侧
	irq_set_irqchip_state(host_irq,      // 软件 pending 状态转移进硬件
			      IRQCHIP_STATE_PENDING, irq->pending_latch);
	irq->pending_latch = false;
}
```

`sgi_domain` 定义在 `struct its_vpe`（include/linux/irqchip/arm-gic-v4.h:63），domain ops 是 `its_sgi_domain_ops`（its.c:4540），irq_chip 是 `its_sgi_irq_chip`（its.c:4472）。

### 4.2 注入：一条 MMIO 写 GITS_SGIR

```c
// drivers/irqchip/irq-gic-v3-its.c:4380 its_sgi_set_irqchip_state
if (state) {
	val  = FIELD_PREP(GITS_SGIR_VPEID, vpe->vpe_id);
	val |= FIELD_PREP(GITS_SGIR_VINTID, d->hwirq);
	writeq_relaxed(val, its->sgir_base + GITS_SGIR - SZ_128K);
	//     ★ 一条 64 位 MMIO 写: {VPEID, VINTID} → ITS 硬件把 vSGI 注入目标 vPE
	//       不占 LR、不等目标 vcpu 调度、fire-and-forget
}
```

**注意**：置 pending 走的是 **GITS_SGIR 寄存器**，不是 VSGI 命令——VSGI 命令只用于配置和清 pending（见 4.3）。这也是内核注释里"GICv4.1 allows us to send VSGI commands to any ITS as long as the destination VPE is mapped there"（its.c:4344）所指的机制家族。

### 4.3 清 pending 与配置同步：VSGI 命令

```c
// its.c:4332 its_configure_sgi —— 构造 VSGI 命令（GITS_CMD_VSGI, :1081 编码）
desc.its_vsgi_cmd.priority = vpe->sgi_config[d->hwirq].priority;
desc.its_vsgi_cmd.enable   = vpe->sgi_config[d->hwirq].enabled;
desc.its_vsgi_cmd.group    = vpe->sgi_config[d->hwirq].group;
desc.its_vsgi_cmd.clear    = clear;

// its.c:4526 —— 内核注释点破 VSGI 命令的尴尬语义:
/*
 * The VSGI command is awkward:
 *  - To change the configuration, CLEAR must be set to false,
 *    leaving the pending bit unchanged.
 *  - To clear the pending bit, CLEAR must be set to true, leaving
 *    the configuration unchanged.
 * You just can't do both at once, hence the two commands below.
 */
vpe->sgi_config[d->hwirq].enabled = false;
its_configure_sgi(d, false);   // 改配置（disable）
its_configure_sgi(d, true);    // 清 pending
```

**一条命令不能同时改配置和清 pending**——mask 一个 vSGI 需要发两条 VSGI 命令。配置更新走 `its_sgi_mask/unmask_irq`（its.c:4354/:4363）和 `its_sgi_set_vcpu_affinity` 的 `PROP_UPDATE_VSGI` 分支（its.c:4461）。

### 4.4 读状态：GICR_VSGIR + 忙轮询

```c
// its.c:4403 its_sgi_get_irqchip_state —— 双锁保护
cpu = vpe_to_cpuid_lock(vpe, &flags);           // 防 vPE 亲和性并发变化
raw_spin_lock(&gic_data_rdist_cpu(cpu)->rd_lock); // 防 VSGIPENDR 并发访问
writel_relaxed(vpe->vpe_id, base + GICR_VSGIR);
do {
	status = readl_relaxed(base + GICR_VSGIPENDR);
	if (!(status & GICR_VSGIPENDR_BUSY)) goto out;
	count--; ... udelay(1);                    // 1s 超时兜底
} while (count);
```

读 pending 需要"选 vPE → 轮询 BUSY → 读位图"三步 MMIO 序列，且有两把锁（vPE 迁移锁 + RD 锁）——这是 vSGI 硬件化带来的新同步面（LR 时代读软件 pending_latch 一行搞定）。

### 4.5 反向转移：disable 时把硬件 pending 迁回软件

```c
// vgic-v4.c:158 vgic_v4_disable_vsgis
irq->hw = false;
irq_get_irqchip_state(irq->host_irq, IRQCHIP_STATE_PENDING, &pending);  // 读硬件
irq->pending_latch = pending;         // 迁回软件影子
irq_domain_deactivate_irq(...);       // 停宿主侧
```

与 4.1 的 enable 方向对称——vSGI 在"硬件注入"和"软件注入"两种模式间切换时，pending 状态必须双向无损转移（这也是 `vgic_v4_configure_vsgis` 开头 `kvm_arm_halt_guest(kvm)` 停住所有 vcpu 的原因：转移期间 guest 不能同时在跑）。

---

## 五、两条路径对比总表

| 维度 | GICv3 / GICv4.0 | GICv4.1 |
|------|-----------------|---------|
| 注入介质 | List Register（`ICH_LR*`，软件写） | **`GITS_SGIR` 一条 MMIO 写**（`{VPEID, VINTID}`） |
| 内核抽象 | 软件中断（`pending_latch` + AP list） | 伪装成宿主硬件中断（`irq->hw=true` + `host_irq`） |
| 生效时机 | 目标 vcpu **下次调度进入**时 flush 才可见 | **即时生效**（fire-and-forget MMIO） |
| LR 消耗 | 每 pending vSGI 占 1 个 LR 槽 | 零 |
| 配置通道 | 无（软件状态） | VSGI 命令 + `vpe->sgi_config[]` 同步 |
| 清 pending | `pending_latch = false`（软件） | VSGI 命令（CLEAR=1，与配置不可同命令——两条） |
| 读 pending | 读软件状态 | `GICR_VSGIR` + `VSGIPENDR.BUSY` 轮询 + 双锁 |
| Guest 侧 trap | trap（`access_gic_sgi`） | **仍 trap**——硬件化的是注入，不是 guest 直接生成 |
| EOI 计数 | 走 AP list（`vSGIEOICount` 置位，vgic-v3.c:41-42） | 硬件管理，无需软件计数 |
| 模式切换 | — | enable/disable vsgis 时 pending 双向转移 + halt guest |
| 门控 | 默认路径 | `has_gicv4_1`（vgic-v3.c:786；its.c:5868 sgi_ops 仅 4.1 挂载） |

---

## 六、关键设计点与坑

1. **trap 解析两代共用**：`vgic_v3_dispatch_sgi` 的 affinity 解析（含 IRM 广播位、target list 遍历）在 v4.1 下依然执行——GICv4.1 的硬件 vSGI 只替代"注入"环节，guest SGI 指令仍被 trap，hypervisor 仍是"目标列表计算器"
2. **VSGI 命令的 awkward 语义**（its.c:4526 注释）：配置与清 pending 互斥，一次 mask 要两条命令——软件 API 简洁性的反面是命令语义的割裂
3. **GITS_SGIR 的 fire-and-forget**：置 pending 的 MMIO 写无回读、无 BUSY 轮询（对比读路径的 VSGIPENDR 轮询）——写入即完成，投递由 ITS 异步保证
4. **1→N 广播的语义保留**：GICv3 的 `ICC_SGI1R` target list / IRM 广播位在 trap 解析层完整保留（`vgic_v3_dispatch_sgi` 的广播分支与位图遍历），与 GICv5 删除广播形成对照（见 `gicv5/gicv5-sgi-vsgi-note.md`）
5. **状态转移的正确性前提**：`vgic_v4_configure_vsgis` 的 `kvm_arm_halt_guest` + 双向 pending 转移——从软件中断切换为硬件中断的瞬间，任何丢失的 pending 都会变成丢中断

---

*笔记时间：2026-08-27*
*内核版本：master (fc46aed51f6)*
*证据文件：arch/arm64/kvm/sys_regs.c、arch/arm64/kvm/vgic/vgic-mmio-v3.c、vgic-v3.c、vgic-v4.c、drivers/irqchip/irq-gic-v3-its.c、include/linux/irqchip/arm-gic-v4.h*

## 参考链接

**内核代码：**
- [vgic-mmio-v3.c — vgic_v3_dispatch_sgi / vgic_v3_queue_sgi](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/arm64/kvm/vgic/vgic-mmio-v3.c)
- [sys_regs.c — access_gic_sgi（SGI trap 入口）](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/arm64/kvm/sys_regs.c)
- [vgic-v4.c — configure/enable/disable vsgis、sgi_config 同步](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/arm64/kvm/vgic/vgic-v4.c)
- [irq-gic-v3-its.c — its_sgi_set_irqchip_state（GITS_SGIR）、its_configure_sgi（VSGI 命令）](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/irqchip/irq-gic-v3-its.c)

**规范：**
- [Arm GICv3/v4 Architecture Specification（IHI 0069）](https://developer.arm.com/documentation/ihi0069)
- [Arm A-profile Registers — ICH_HCR_EL2 / ICH_LR / GITS_SGIR / GICR_VSGIR](https://developer.arm.com/documentation/ddi0601)

**关联笔记：**
- [GICv5 IPI 笔记（vSGI 在 GICv5 的消失）](gicv5/gicv5-sgi-vsgi-note.md)
