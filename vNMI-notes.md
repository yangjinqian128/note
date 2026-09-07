# vNMI（FEAT_GIC_NMI 虚拟化）使能逻辑与代码架构笔记

> 适用：openEuler OLK-6.6（`/home/code/kernel`）
> 补丁来源：Marc Zyngier `arm64/nmi` 分支（15 patches），openEuler PR `!5815 v2`（caijian 移植，合并提交 `856685970332`），系列头 `d46e52452afc`
> follow-up：`81c70de3a179`（Jinqian Yang，补 `ID_AA64PFR1_EL1.NMI` 的 `.val` 使其真正可写）

---

## 1. 硬件背景（GICv3.3 / FEAT_GIC_NMI）

| 硬件概念 | 说明 |
|---|---|
| `GICD_INMIR` / `GICR_INMIR0` | 每 IRQ 1 bit 的 NMI 标记（dist 域 SPI / redist 域 SGI+PPI） |
| NMI 语义 | 超级优先级：抢占一切普通中断；**不受 PMR 屏蔽**，只被 `ALLINT` 屏蔽 |
| `ICH_AP1R0_EL2` | 从 32 位升级为 **64 位**，bit[63] 编码 NMI 激活优先级 |
| `ICH_LR.NMI`（bit 59） | LR 内标记该中断为 NMI，置位时硬件忽略 Priority 字段 |
| `ICC_NMIAR1_EL1` | 取 NMI（对应普通中断的 IAR） |
| `ALLINT` / `ALLINT.SET` / `ALLINT.CLR` | 屏蔽/恢复所有中断（含 NMI）的 PSR 位 |
| `ID_AA64PFR1_EL1.NMI` | CPU 侧特性广告（0x1 = IMP） |

---

## 2. 使能逻辑（三级门控 + 注入判定）

```
┌──────────────── Host 全局 ────────────────┐
│ gic_data.has_nmi                        │ ← GICD_TYPER.NMI 硬件位
│         &&                              │   (irq-gic-v3.c:2483)
│ cpus_have_const_cap(ARM64_HAS_NMI)      │ ← 纯 CPUID 特性: PFR1.NMI≥IMP
│         &&                              │   (cpufeature.c:3151, matches=has_cpuid_feature)
│ vgic_nmi                                │ ← 本地总开关: static bool, 默认 false
│         &&                              │   (vgic-v3.c:32，改 true 启用)
│ !vgic_v3_cpuif_trap                     │ ← CPU interface 需要 trap 时自动禁用
└───────────────────┬─────────────────────┘   (vgic-v3.c:740-747)
                    ▼
   kvm_vgic_global_state.has_nmi          (vgic-v3.c:731)
                    │  vgic_init() 默认继承 (vgic-init.c:362-369)
                    ▼
┌──────────────── VM 级 ─────────────────────┐
│ dist->has_nmi                            │ ← userspace: IMP_REV_4 + 写 GICD_TYPER.NMI
│  · 读 GICD_TYPER 时广告 NMI 位            │   (vgic-mmio-v3.c:81, 170-193)
│  · INMIR 写 handler 的门控                │   (vgic-mmio-v3.c:644)
│  · 选 IMP_REV_2/3 强制清 has_nmi          │   (vgic-mmio-v3.c:187-190)
└───────────────────┬───────────────────────┘
                    ▼
┌──────────────── vCPU 级 ────────────────────┐
│ kvm->arch.pfr1_nmi                        │ ← kvm_arch_init_vm 默认 IMP
│  · userspace 写 ID_AA64PFR1_EL1.NMI       │   (arm.c:384-385)
│  · set_id_aa64pfr1_el1: 只许改 NMI 字段    │   (sys_regs.c:1767-1780)
│  · 读 PFR1 时 FIELD_PREP 进 sanitised 值   │   (sys_regs.c:1415)
└───────────────────┬───────────────────────┘
                    ▼
         ┌────── 注入判定 ──────┐
         │ vgic_v3_populate_lr │ (vgic-v3.c:181-190)
         │  pfr1_nmi==IMP      │
         │    && irq->nmi      │ ──► ICH_LR_NMI (bit 59)，不写 Priority
         └─────────┬───────────┘
                   ▼
         ┌────── 直通判定 ──────┐
         │ HCRX_EL2.TALLINT    │ (hyp/switch.h:228-230, 263-265)
         │  HAS_NMI && pfr1_nmi│ ──► 清 TALLINT：ALLINT 直通 guest
         │  否则保持 trap       │ ──► guest 访问 ALLINT 触发 UNDEF
         └─────────────────────┘
```

**要点：**

- **本地总开关默认关**：`static bool vgic_nmi`（vgic-v3.c:32，默认 false）→ `global.has_nmi=false` → `dist->has_nmi`/`pfr1_nmi` 默认全关，且 userspace 写 PFR1.NMI、GICD_TYPER.NMI 均被拒绝（sys_regs.c:1776-1780、vgic-mmio-v3.c:170-173）。改成 `= true` 后恢复上游语义：userspace 什么都不写时 `dist->has_nmi = global.has_nmi`、`pfr1_nmi = IMP`，即默认开启。
- **`ARM64_HAS_NMI` vs `ARM64_USES_NMI`**（当前树，比 backport 系列更新）：
  - `ARM64_HAS_NMI` = 纯 CPUID 特性（`has_cpuid_feature`），**KVM vNMI 的门控**；
  - `ARM64_USES_NMI` = `use_nmi()`（cpufeature.c:2273-2297），受伪 NMI 冲突与 `CONFIG_ARM64_NMI` 影响，**host 自己用 NMI 的门控**（`system_uses_nmi()`，cpufeature.h:865-869）。
- **`CONFIG_ARM64_NMI=n` 不影响 vNMI**：`b8c8255e1d74` 解耦，`use_nmi()` 在无 config 时照样返回 true（"using NMIs for guests only"）。
- **伪 NMI 冲突**：`irqchip.gicv3_pseudo_nmi=1` 只关 `ARM64_USES_NMI`（host 侧），**不关 `ARM64_HAS_NMI`，因此也不关 vNMI**。
- **`vgic_v3_cpuif_trap` 是自动降级点**：只要 CPU interface 需要 trap（例如无 VHE 直通条件），`has_nmi` 强制为 false，dmesg 打 "disabled due to trapping"。
- host 侧非 KVM 路径用 `has_v3_3_nmi() = gic_data.has_nmi && system_uses_nmi()`（irq-gic-v3.c:165-167）控制 ALLINT/PMR 管理。

---

## 3. 代码架构（数据流）

```
 guest 写 INMIR (GICD_INMIR / GICR_INMIR0)
   │  vgic_mmio_write_nmi (vgic-mmio-v3.c:637)     ← 要求 dist->has_nmi
   ▼
 vgic_irq.nmi (include/kvm/arm_vgic.h:183)
   │
   ├─► ① AP-list 排序: vgic_irq_cmp (vgic.c:302-303)
   │      nmi 不同 → NMI 排最前（super-priority），同 nmi 再比 priority
   │
   ├─► ② LR 编码: vgic_v3_populate_lr (vgic-v3.c:189)
   │      nmi → ICH_LR_NMI，priority 字段不写（硬件忽略）
   │
   ├─► ③ Priority 读写 RES0: vgic_mmio_read/write_priority (vgic-mmio.c:754, 783)
   │      NMI IRQ 读 IPRIORITYR 返回 0，写被忽略
   │
   ├─► ④ GICv4.1 直通 SGI: vgic_update_vsgi (vgic-mmio.c:64-68)
   │      → its_prop_update_vsgi(irq, prio, group, nmi)
   │      → its_sgi_set_vcpu_affinity 存 sgi_config.nmi (irq-gic-v3-its.c:4996)
   │      → VSGI 命令编码 bit[11] (its_encode_sgi_nmi, irq-gic-v3-its.c:971-973)
   │      （NMI 状态变化时才调：vgic-mmio-v3.c:658）
   │
   └─► ⑤ debugfs: print_irq_state 的 "N" 列 (vgic-debug.c:223)
```

**hyp 层 save/restore（64 位 AP1R）：**

- `vgic_v3_cpu_if.vgic_ap1r[4]` 为 `u64`（arm_vgic.h:385），AP0R 仍 32 位
- save/restore：`__vgic_v3_read/write_ap1rn`（hyp/vgic-v3-sr.c:137, 180），在 360-400 行处展开 4 个寄存器

**trap 侧：**

| 寄存器 | 有 FEAT_NMI 的 guest | 无 NMI 的 guest |
|---|---|---|
| `ALLINT` / `SET` / `CLR` | 直通（清 `HCRX_EL2.TALLINT`，hyp/switch.h:228-230） | trap → `undef_access`（sys_regs.c:2298-2299, 2568） |
| `ICC_NMIAR1_EL1` | hyp `__vgic_v3_perform_cpuif_access` 直接 `return 0`（"Here's an UNDEF for you"，hyp/vgic-v3-sr.c:1051-1053）；sys_regs 侧 `undef_access`（sys_regs.c:2629） | 同左（trap 路径） |
| `GICD_TYPER.NMI` | 读返回 `has_nmi` 广告位（vgic-mmio-v3.c:81） | 0 |

**特性广告（读侧）：**

- `GICD_TYPER.NMI`：`vgic->has_nmi`（vgic-mmio-v3.c:81）
- `ID_AA64PFR1_EL1.NMI`：sanitised 值 OR `kvm->arch.pfr1_nmi`（sys_regs.c:1415）

---

## 4. 关键文件索引

| 文件 | 符号 / 位置 | 作用 |
|---|---|---|
| `drivers/irqchip/irq-gic-v3.c` | `gic_data.has_nmi` (:2483)、`system_is_nmi_capable()` (:170-172)、`:2653/:3009` | host 探测 NMI 能力，填 `gic_kvm_info.has_nmi` |
| `arch/arm64/kernel/cpufeature.c` | `use_nmi()` (:2273)、`ARM64_HAS_NMI` (:3151)、`ARM64_USES_NMI` (:3161) | cpucap 定义；HAS=cpuid 纯特性，USES=use_nmi |
| `arch/arm64/include/asm/cpufeature.h` | `system_uses_nmi()` (:865) | host 侧 NMI 使用判定（CONFIG && USES && !prio_masking） |
| `arch/arm64/kvm/vgic/vgic-v3.c` | `vgic_nmi` 总开关 (:32)、`vgic_v3_probe()` (:740)、`vgic_v3_populate_lr()` (:110, :189) | vNMI 总开关（默认关）；全局 has_nmi 门控；LR 编码 ICH_LR_NMI |
| `arch/arm64/kvm/vgic/vgic-init.c` | `vgic_init()` (:362-369) | dist->has_nmi 默认继承 global 值 |
| `arch/arm64/kvm/vgic/vgic-mmio-v3.c` | `read/write_nmi` (:617/:637)、`TYPER` (:81, :170-193)、INMIR 注册 (:739, :826) | INMIR MMIO 仿真 + VM 级选择 |
| `arch/arm64/kvm/vgic/vgic-mmio.c` | `vgic_update_vsgi()` (:64)、priority RES0 (:754, :783) | NMI 时 priority 不可见/不可写；VSGI 传播 |
| `arch/arm64/kvm/vgic/vgic.c` | `vgic_irq_cmp()` (:302) | AP-list 排序：NMI 超级优先级 |
| `arch/arm64/kvm/vgic/vgic-v4.c` | `vgic_v4_sync_sgi_config()` (:113) | 直通 SGI 配置同步 nmi 位 |
| `drivers/irqchip/irq-gic-v3-its.c` | `its_encode_sgi_nmi()` (:971)、`its_configure_sgi()` (:4837)、`:4996` | VSGI 命令 bit[11] 编码 |
| `arch/arm64/kvm/arm.c` | `kvm_arch_init_vm()` (:384-385) | vCPU 默认 pfr1_nmi=IMP |
| `arch/arm64/kvm/sys_regs.c` | `set_id_aa64pfr1_el1()` (:1767)、`:1415`、ALLINT (:2298, :2568)、NMIAR (:2629) | vCPU 级 NMI 开关 + ALLINT/NMIAR trap |
| `arch/arm64/kvm/hyp/include/hyp/switch.h` | `__activate/deactivate_traps_common` (:228-230, :263-265) | TALLINT 直通/恢复 |
| `arch/arm64/kvm/hyp/vgic-v3-sr.c` | `__vgic_v3_read/write_ap1rn` (:137, :180)、NMIAR trap (:1051) | 64 位 AP1R save/restore；NMIAR 返回 0 |
| `include/kvm/arm_vgic.h` | `vgic_irq.nmi` (:183)、`vgic_ap1r[4]` u64 (:385)、`has_nmi` (:324)、`vgic_global.has_nmi` (:111) | 核心数据结构 |
| `arch/arm64/kvm/vgic/vgic-debug.c` | `print_irq_state` (:223) | debugfs N 列 |

---

## 5. 如何关闭

| 层级 | 方法 | 效果 |
|---|---|---|
| **VM（推荐）** | userspace 选 `KVM_VGIC_IMP_REV_3`（或写 GICD_TYPER 清 NMI 位）+ 写 `ID_AA64PFR1_EL1.NMI = 0` | 该 VM 完全无 vNMI；两级都要清，否则 guest 看到的 CPU/IRQ 能力不一致 |
| guest | 不写 INMIR | 每 IRQ 逐位 opt-in，guest 不用就无 NMI 行为 |
| **host 全局（本地总开关）** | `vgic-v3.c:32` 的 `static bool vgic_nmi;` —— 默认 false=关；改 `= true` 启用 | 全 VM 默认无 vNMI，userspace 也强制不开 |
| host 运行时（上游树） | 无专用开关。`irqchip.gicv3_pseudo_nmi=1` **只关 host 侧 USES_NMI，不关 vNMI** | — |
| host 编译期（备选） | `system_is_nmi_capable()` 恒 false（irq-gic-v3.c） | 与 vgic_nmi 开关等价，二选一即可 |
| 硬件 | `gic_data.has_nmi` 来自 GICD_TYPER.NMI，不可配置 | — |

**注意坑点：**
- `CONFIG_ARM64_NMI=n` **关不掉 vNMI**（补丁 14 特意解耦）
- 没有 `CONFIG_KVM_ARM64_NMI` 之类的 Kconfig
- （本地修改后）userspace 什么都不做时**默认关闭**——`vgic_nmi` 默认 false；改 true 后才是"默认开启"语义

**验证：** guest 内 `GICD_TYPER.NMI == 0`、`ID_AA64PFR1_EL1.NMI == 0`；host dmesg 无 "GICv3 NMI support enabled"；vgic debugfs N 列。

---

## 6. 已知限制 / 注意点

1. **NMIAR 未仿真（原型行为）**：guest 读 `ICC_NMIAR1_EL1` 得到"无 NMI pending"（hyp 返回 0 / sys_reg UNDEF）。vNMI 目前只支持"收 NMI + 超级优先级抢占"，guest 无法通过 NMIAR 显式 ack。Marc 后续版本有改进，此 backport 未包含。
2. **两级开关一致性**：`dist->has_nmi`（GICD_TYPER.NMI）与 `pfr1_nmi`（PFR1.NMI）独立，userspace 需要成对设置。
3. **注入判定在 vCPU 级**：`populate_lr` 要求 `pfr1_nmi == IMP` 才写 `ICH_LR_NMI`，否则退化为普通中断注入（只写 priority）。
4. **默认开启语义**（上游）：依赖 `vgic_init()` 的默认继承逻辑，userspace 显式写 IMP_REV_2/3 或 TYPER 可覆盖。本地用 `vgic_nmi` 总开关（默认关）压过该语义。
5. **`vgic_v3_cpuif_trap` 自动禁用**：CPU interface 需要 trap 的系统上 vNMI 静默不可用（ALLINT/NMIAR 无法直通），dmesg 有提示。
6. **伪 NMI 与真 NMI 的关系**（当前树）：`ARM64_HAS_NMI` 是纯 CPUID，伪 NMI 只影响 `ARM64_USES_NMI`（host 使用）；两者理论上可共存，但 host 侧 `system_uses_nmi()` 加了 `!system_uses_irq_prio_masking()` 保护。
