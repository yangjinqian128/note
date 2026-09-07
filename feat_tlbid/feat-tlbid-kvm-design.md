# FEAT_TLBID KVM/arm64 架构设计：TLBI 域失效

> 设计任务：在 host/guest 均为 "1 vcpu = 1 domain" 的思路基础上，与 Armv9.7-A FEAT_TLBID spec 逐条对比，补齐设计难点与备选方案。
> 本文档中所有 spec 论断来自 Arm 官方文档与工具链补丁的二次文献，来源见文末；代码论断以本仓库 v7.3 代码为准，标注 `file:line`。

---

## 1. Spec 语义基础（设计的地基，先对齐再设计）

### 1.1 特性本质

FEAT_TLBID（Armv9.7-A，2025 年发布，代号 "TLBI Domains"）给 TLBI/TLBIP/PLBI 广播类指令增加一个 **16 位 TLBID 域字段**。其核心语义在 Arm 文档中表述为：

> "If FEAT_TLBID is implemented, the set of PEs is reduced to be PEs that are also within the TLBI Domain specified in the TLBID field and System register configuration."

即：**广播失效从"整个 shareability domain 的所有 PE"缩小为"域内的 PE 子集"**。本质是硬件级的 multicast TLB shootdown。

### 1.2 哪些指令能带域（关键边界）

| 类别 | 可带 TLBID | 依据 |
|---|---|---|
| `ALLE1*/ALLE2*/VMALL*/VMALLS12*/VMALLWS2*` + IS/OS | ✓ 确定 | binutils 补丁 F_TLBID_XT 列表、LLVM PR #163156 |
| `ASIDE1*`（按 ASID 失效，即 Linux `flush_tlb_mm` 用的指令） | ✓ 疑似 | ASIDE1OS 页面出现 "set of PEs is reduced" 措辞 [EXTERNAL，未完全确认] |
| `VAE1*/VALE1*/VAAE1*`（按 VA，即 `flush_tlb_range` 用的指令） | ✗ | 操作数没有 16 位空闲位 |
| `IPAS2E1*`（stage-2 IPA，KVM 热路径） | ✗ | 同上 |

**设计含义**：域优化只作用于"全量类"失效（`flush_tlb_mm`/`flush_tlb_all`/VMID 全刷）。**范围类失效（`flush_tlb_range`、KVM 的 IPA range flush）吃不到域的红利**，它们的优化路径（TLBIRANGE、逐 VA 循环）与域正交。

### 1.3 无特性时的约束（向后兼容）

- 未实现 FEAT_TLBID 时：Rt 必须为 `0b11111`，否则 CONSTRAINED UNPREDICTABLE（UNDEFINED 或按 0b11111 处理）。
- TLBID 字段位置在不同文档中分别出现 bits [15:0] 与 [31:16] 两种说法——对无操作数指令（VMALL 类）与有操作数指令（ASIDE1 类，TLBID 塞进操作数空闲位）位置可能不同 [SPEC-OPEN G3]。

### 1.4 物理域与虚拟域的两层结构（本设计最重要的 spec 事实）

`TLBIDIDR_EL1`（只读识别寄存器）把 16 位 TLBID 空间分成两层：

```
                    ┌────────────────────────────────────────────┐
                    │  TLBIDIDR_EL1 (只读, FEAT_TLBID 才存在)     │
                    │                                            │
                    │  NVOS[44:40]  ← 虚拟域位数 (OS 指令)         │
                    │  NOS [36:32]  ← 物理域位数 (OS 指令)         │
                    │  NVIS[12:8]   ← 虚拟域位数 (IS 指令)         │
                    │  NIS [4:0]    ← 物理域位数 (IS 指令)         │
                    └────────────────────────────────────────────┘
```

**合法组合约束（硬约束）**：

| NIS/NOS | NVIS/NVOS 上限 | guest 可见虚拟域数 |
|---|---|---|
| 0 | 必须为 0 | 无域功能 |
| 1–8 | ≤ 5 | ≤ 2^5 = **32** |
| 9–16 | ≤ 4 | ≤ 2^4 = **16** |
| >16 | 保留 | — |

### 1.5 虚拟→物理翻译：VTLBID 寄存器组

`VTLBID<n>_EL2` / `VTLBIDOS<n>_EL2`（n=0–3，每 PE 一组，EL2 特权）是**翻译表**：

> "TD fields indicate which physical TLBI Domain corresponds to the EL1 view of one virtual TLBI Domain."
> "Transform the value of TLBID in TLBI IS/OS operations **executed at EL1** before the operation is broadcast."

- NIS ≤ 8：每寄存器 8 个 TD 字段 × 8 位；NIS ≥ 9：每寄存器 4 个 TD 字段 × 16 位。
- 表规模由 NVIS 决定：NVIS=5 → 32 个 TD 字段（4 寄存器）；NVIS=4 → 16 个。
- Warm reset 后 TD 字段为 UNKNOWN。
- **翻译只作用于 EL1 发出的 TLBI**（guest）；EL2（hypervisor 自己）直接用物理域 [INFERRED，依据 "executed at EL1" 措辞]。

```
Guest (EL1) 执行:  tlbi vmalle1is, x5     // x5[?] = 虚拟域 v (guest 自己选的值)
                     │
                     ▼
               HCRX_EL2.VTLBIDEn == 1 ?   // hypervisor 开关
                     │ yes
                     ▼
           查本 PE 的 VTLBID TD 表: 物理域 p = TD[v]     // hypervisor 在 vcpu 切换时编程
                     │
                     ▼
          广播 p: 接收方 PE 检查自己是否属于物理域 p ──► 是则失效本 PE 的 TLB（仍受 VMID/ASID 匹配约束）
                     │ no (VTLBIDEn == 0)
                     ▼
               不翻译，按 G2 语义处理（见 §6.3）
```

### 1.6 Hypervisor 控制位

- **`HCRX_EL2.VTLBIDEn/VTLBIDOSEn`**：是否对 EL1 指令启用翻译。**这是无 trap 的开关**。
- **`HCR_EL2.FB` / `HCRX_EL2.FNB`**：强制广播 / 强制不广播。KVM 现状 `HCR_GUEST_FLAGS` 已含 `HCR_FB`（`arch/arm64/include/asm/kvm_arm.h:100-101`）——**KVM 今天的默认行为就是"guest TLBI 强制全广播"**。
- **`HCR_EL2.TTLB/TTLBIS/TTLBOS`**：按类型 trap guest TLBI（`asm/kvm_arm.h:26-27,53`），目前仅嵌套虚拟化使用（`arch/arm64/kvm/hyp/vhe/switch.c:68`）。
- `SCTLR2_ELx.TLBOSNIS`：影响 OS 指令的域语义（细节未确认）。

---

## 2. 现有代码基线（改造的起点）

### 2.1 Host 侧（内核自身 TLB 失效）

- `flush_tlb_mm()`：单条 `tlbi aside1is` 全广播 + `mmu_notifier_arch_invalidate_secondary_tlbs`（`arch/arm64/include/asm/tlbflush.h:376-386`）。**今天一次 flush 打全机所有核**。
- `flush_tlb_all()`：`tlbi vmalle1is` 全广播（`tlbflush.h:368-374`）。
- `flush_tlb_range()` → `__flush_tlb_range` 循环（`tlbflush.h:613-632`），按 VA 失效，与域无关。

### 2.2 KVM 侧

- VMID 全刷：`kvm_arch_flush_remote_tlbs`（`arch/arm64/kvm/mmu.c:175-182`）→ `__kvm_tlb_flush_vmid` → `tlbi vmalls12e1is` **全机广播**（`arch/arm64/kvm/hyp/vhe/tlb.c:183-197`）。
- IPA 范围刷：`kvm_arch_flush_remote_tlbs_range` → `__kvm_tlb_flush_vmid_range`（`vhe/tlb.c:154-181`），按 IPA 范围 + `vmalle1is` 兜底，与域无关。
- 发失效前必须 `enter_vmid_context` 翻转 `HCR_EL2.TGE` 切进 guest VMID 上下文（`vhe/tlb.c:20-68`）——**域化改造要保持这个"上下文翻转 + 域"组合的正确性**。
- 通用层：`kvm_flush_remote_tlbs`（`virt/kvm/kvm_main.c:293-311`）对 arm64 直接信任广播完成（不落 IPI 路径）。

### 2.3 vCPU 切换（域表编程的唯一挂载点）

- VHE：`kvm_vcpu_load_vhe`（`arch/arm64/kvm/hyp/vhe/switch.c:216-223`）→ `__vcpu_load_activate_traps` → `__compute_hcr`。VTLBID 编程就挂这里（对称的 `kvm_vcpu_put_vhe`，`switch.c:225-231`）。
- nVHE：对应 hyp/nvhe 路径，需要同样的保存/恢复。

---

## 3. 用户方案分析：1 vcpu = 1 domain

### 3.1 方案重述

- Host：每个 vCPU 独占一个物理域；需要广播时循环发 TLBI（带域），每条只打一个核。
- Guest：同 host，每个 vCPU 独占一个虚拟域，循环发域失效。
- 已知顾虑：guest 域数受 NVIS 限制；或改用"每条 guest TLBI 都 trap 出来由 KVM 转发"（性能顾虑）。
- 动机：vCPU 乱飘时范围绑核方案效果差，1:1 域方案"只广播真正需要的几个核"。

### 3.2 与 spec 的契合点（方案成立的部分）

1. **域随 vCPU 走、与物理位置解耦**：guest 用的是虚拟域（NVIS 空间），hypervisor 通过 VTLBID TD 表决定虚拟域→物理域映射。guest 侧的"每 vCPU 一个域"是**逻辑上稳定的**——vCPU 怎么飘，guest 的域编号不变。**这正是 spec 设计虚拟域层的本意**，用户的直觉与 spec 方向一致。
2. **接收侧零噪声**：域失效的收益不在发射侧（循环 N 条反而比 1 条广播发射成本高），而在**接收侧**——非目标 PE 的流水线/TLB 完全不受打扰。今天 `aside1is` 广播会把全机所有核都打一遍（RFC "[RFC] KVM: arm64: Don't force broadcast tlbi when guest is running" 2020-10 讨论的正是这个规模化痛点）。1:1 域把接收集压到最小，这是方案的核心价值，成立。
3. **乱飘的退化有界**：guest 侧 flush 协议要覆盖的是"所有可能残留陈旧条目的 vCPU 域"（逻辑上 = 跑过该 mm 的 vCPU 集合），与物理摆放无关。乱飘不扩大 guest 的域集合（域随 vCPU 走），最多是 hypervisor 侧映射维护成本上升。**"1:1 总是只广播需要的核"这一判断，在虚拟域层是严格成立的**。

### 3.3 硬约束与冲突点（方案必须妥协的部分）

**冲突 1：guest 域数上限是 32（甚至 16），不是"受限一点"而是架构硬顶**（§1.4 表）。
- 32 vCPU 以内：1:1 可行。
- 超过 32（且 NIS≥9 时超过 16）：必须分组（多个 vCPU 共享一个域）或回退全广播。分组时广播是超集，正确但精度随 vCPU 数单调退化。
- 注意：虚拟域空间是**每 VM 独立的**（TD 表由 hypervisor 按 vCPU 切换编程），不会随 VM 数量缩水 [INFERRED]。

**冲突 2：host 侧同样受 NIS 限制**。host 在 EL2 用物理域，1 vCPU/1 物理域在 vCPU 总数 > 2^NIS 时同样破产。且 host 内核自身（非 KVM）的 flush 也要用域——bare-metal EL1 内核没有 EL2 特权去编程 VTLBID [INFERRED]，**域功能本质上是"有 hypervisor 才完整可用"的**，与 KVM host（EL2/VHE）场景天然匹配。

**冲突 3：域只覆盖广播类指令**（§1.2）。`flush_tlb_range` 高频路径不受益；收益集中在 `flush_tlb_mm`/`flush_tlb_all`/VMID 全刷。

### 3.4 正确性核心：乱飘（迁移）下的残留覆盖协议

这是整个方案**唯一真正难的正确性问题**，分两层：

**Guest 层（虚拟域，简单）**：guest flush 协议 = "遍历跑过该 mm 的 vCPU 的虚拟域"。域随 vCPU 走，迁移对 guest 完全透明。✓

**Hypervisor 层（物理域，难）**：vCPU 从 PE_A 迁到 PE_B 后：
- PE_B 需要新 vCPU 的域映射 → `vcpu_load` 时重编程 VTLBID 表。
- **PE_A 上残留的旧 guest 条目，未来 guest 的域失效必须还能打到它**。若 PE_A 的表已换给下一个 VM，域失效就打不到了 → 陈旧条目泄漏 → 正确性 bug。

**模型无关的协议（推荐，无论 G1 两种模型都成立）——"迁移即冲刷"**：

```
vCPU 迁移 PE_A ──► PE_B 时序:

PE_A (旧)                              PE_B (新)
───────                                ───────
① vcpu_put:
   tlbi vmalle1  (本地, 冲刷本 PE 的      │
   guest stage-1 残留)                   │
   ic iallu                              │
② (之后本 PE 的 VTLBID 表可安全换给       │
    下一个 VM)                           │
                                       ③ vcpu_load:
                                         写 VTLBID 表 = 新 VM 的
                                         TD 映射 (含该 vCPU 的域)
```

- 成本：每次迁移多 1 条本地 `vmalle1`（几十周期，vcpu_put 路径本来就很重，`__kvm_flush_cpu_context` 已有同类操作可复用，`vhe/tlb.c:199-212`）。
- **把"乱飘"的成本从 O(每次 flush 的广播面) 转成 O(迁移次数 × 1 条指令)**——这正是用户直觉"乱飘对 TLBID 效果一般"的真正解法：不需要一一绑核，域方案 + 迁移冲刷即可同时吃掉精确性和乱飘。
- 安全网：VMID 轮转时的全广播 `vmalls12e1is` 必须保持**不带域**（§5.2），它是所有残留条目的最终兜底。

### 3.5 性能模型

```
                    现状(全广播)          1:1 域循环
发射侧成本          1 × (TLBI+DSB)        N × TLBI + 1 × DSB(可 batch)
                                    ↑ MAX_DVM_OPS 已有批处理模式可复用 (tlbflush.h:412)
接收侧成本          全机所有 PE           仅 |目标集| 个 PE
                    (包括无关核的        无关核零打扰
                     TLB 流水线抖动)
```

- 发射侧 N 条 TLBI 可通过"连续发射 + 单次 DSB"摊销（与现有 `__flush_tlb_range` 循环同构）。
- 交叉点：目标集小时域循环赢（VM 内 flush、单进程 exit）；目标集接近全机时（`flush_tlb_all`、系统级操作）应回退单条全广播。**阈值自适应是必须内置的**，与 OpenEuler OLK-6.6 补丁（mm_cpumask 少时用域）方向一致 [EXTERNAL]。

---

## 4. 关于"每条 guest TLBI 都 trap 出来转发"的评估

**结论：作为主路径否决；作为兜底路径保留。** 理由：

1. **现状 guest TLBI 零 trap**：KVM 靠 `HCR_FB` 强制广播保证正确（`kvm_arm.h:101`），guest TLBI 原生执行（VMID 隔离保证安全）。每条 TLBI 一次 VM exit（数百~数千周期）+ 高频触发（上下文切换、munmap、THP 拆分）→ 量级不可接受。
2. **而且根本不需要 trap 才能"转发"**：`HCRX_EL2.{VTLBIDEn,VTLBIDOSEn}` + `HCR_EL2.FB` 给了 hypervisor **两个寄存器位级别的无 trap 开关**：
   - 域配置未就绪 → `VTLBIDEn=0, FB=1` → guest TLBI 变回全广播（正确性零风险）。
   - 域配置就绪 → `VTLBIDEn=1` → guest 域失效直接生效。
   - 这就是"转发"的硬件实现——**翻译在硬件里做，hypervisor 只负责预先编程映射表**。
3. 选择性 trap（`HCR_EL2.TTLBIS/TTLBOS`，`kvm_arm.h:26-27`）可作为**最后手段**：只 trap IS/OS 广播形式、放行本地形式，用于嵌套虚拟化或疑难调试场景，而非正常 guest 运行。
4. 若 guest 想表达"请帮我失效这组 vCPU"且不想自己循环：已有 PV TLB shootdown 类机制的思路可复用（本树未找到实现，[EXTERNAL]），SMC 批量通道 + hypervisor 侧用域失效执行，是一个与 trap 无关的补充通道。

---

## 5. 其他设计难点（补齐清单）

### 5.1 域分配器与 ID 生命周期

- 虚拟域：per-VM 分配（vCPU 映射），vCPU 销毁即回收；回收的域 ID 被新 vCPU 复用前，需确保旧域关联的 PE 已被冲刷（§3.4 协议 + VMID 轮转兜底）。
- 物理域：全局稀缺资源（2^NIS），host 与各 VM 共享。建议物理域不按 vCPU 分配而**按"当前活跃上下文"分配**（活跃 VM 一组、host 一组），复用 + 回收都要走冲刷协议。
- 跨 host 迁移：域不随迁，目标机重新分配（与 VMID/ASID 一样处理）。

### 5.2 VMID 轮转必须保持全广播

VMID 复用时的 `vmalls12e1is` 是**最终安全网**——它必须无条件打到所有 PE（含域外 PE），因此这条 TLBI **永远不带 TLBID**（或按 G2 语义取"全域"值）。任何域化改造都不能碰这条路径。

### 5.3 stage-2 与 KVM 自身失效

- IPA 范围失效（KVM 热路径）与域无关，现状不动（§1.2）。
- KVM 的 VMID 全刷 `vmalls12e1is`（`vhe/tlb.c:192`）可域化：只打"当前运行该 VM vCPU 的 PE"——但 VMID 全刷本身低频（VMID 轮转/表折叠），收益有限，**先不动**，等 L3 阶段再评估。

### 5.4 VTLBID 上下文保存/恢复

- 每 vCPU 需保存/恢复最多 8 个 sysreg（VTLBID0-3 + VTLBIDOS0-3），VHE 挂 `kvm_vcpu_load_vhe/put_vhe`（`vhe/switch.c:216-231`），nVHE 挂对应路径。
- 优化：VM 内 vCPU 之间表相同 → 只需在 **VM 切换**时写（与 stage-2 MMU 切换同频），可复用"上一个 mmu 是谁"的对比逻辑（`enter_vmid_context` 里已有 `vcpu->arch.hw_mmu` 判断，`vhe/tlb.c:28-31`）。
- 宿主机内核自身（EL1/VHE 的 host 部分）如果用域，还需 host↔guest 切换时恢复 host 域配置。

### 5.5 嵌套虚拟化（NV）

- 现状：NV 下 KVM 已用 `HCR_TTLB` trap L1 hypervisor 的 TLBI 并软件转换（`__kvm_tlbi_s1e2`，`vhe/tlb.c:230-366`）。
- L1 若也用域：域翻译对 EL1 生效一次，L2 的域经 L1 再翻译会产生**双层映射**问题；NV 与 FEAT_TLBID 叠加的语义 spec 尚未看到明确说明 [SPEC-OPEN G4]。**建议 NV 场景直接禁用域优化（VTLBIDEn=0 + FB=1）**，保持现状语义。

### 5.6 Guest 可见性与工具链

- guest 如何发现 FEAT_TLBID（ID 寄存器暴露位）尚不明确 [SPEC-OPEN G5]；QEMU/KVM 需要对应的 ID 位与 VTLBID 仿真。
- 工具链已就绪：LLVM `+tlbid`（2025-10 合入）、binutils `+tlbid`（2026-01 合入）；但两者架构门控不一致（LLVM 挂 armv9.7-a，binutils 测试在 armv9.4-a 上开）——内核 cpufeature 检测时注意。

### 5.7 安全面

- 域失效仍受 VMID 匹配约束：guest 随便猜域 ID 只能命中自己 VM 的条目，无跨 VM 风险。
- guest 无法用域"拒收"host 的失效（host 用物理域/全广播，不走 guest 的虚拟域翻译）。
- 域表编程是 EL2 特权，guest 不可触碰。

---

## 6. 推荐方案：三层递进

```
┌────────────────────────────────────────────────────────────────┐
│ L0 基线 (零行为变化)                                           │
│   检测 FEAT_TLBID → 读 TLBIDIDR_EL1 记录 NIS/NVIS/NOS/NVOS     │
│   保持 HCR_FB=1；VTLBIDEn/VTLBIDOSEn=0                         │
│   → 系统行为与今天完全一致（域失效被强制广播取代）              │
├────────────────────────────────────────────────────────────────┤
│ L1 host 自用 (guest 无感知)                                    │
│   host 侧域失效：mm_cpumask 小 → 域循环；大 → 全广播（阈值）    │
│   迁移协议落地：vcpu_put 本地 vmalle1 + vcpu_load 重编程        │
│   （注：host 能否安全域化依赖 G1，见 §6.3）                    │
├────────────────────────────────────────────────────────────────┤
│ L2 guest-aware 虚拟域协议                                       │
│   guest 1 vcpu/1 domain (vCPU ≤ 2^NVIS)，超限分组回退           │
│   KVM 编程 TD 表：虚拟域 → 物理域；VTLBIDEn=1                   │
│   域表复用/回收协议 + VMID 全广播安全网                         │
├────────────────────────────────────────────────────────────────┤
│ L3 演进                                                        │
│   PV 通道 (批量 SMC + hypervisor 域失效执行)                    │
│   KVM VMID 全刷域化、NV 域语义（等 spec G4）                    │
└────────────────────────────────────────────────────────────────┘
```

### 6.1 关键数据结构（草案）

```c
/* per-kvm：该 VM 的域配置 */
struct kvm_tlbid_cfg {
	u8	nvis;			/* 从 TLBIDIDR_EL1 读出，恒等实现值 */
	u8	vdom_bits;		/* 本 VM 实际使用的虚拟域位数（≤ nvis） */
	u64	vtlbid[4];		/* 组装好的 VTLBID0-3_EL2 值 */
	u64	vtlbidos[4];		/* 同上，OS 变体 */
	DECLARE_BITMAP(vcpu_vdom, 32);	/* vCPU id → 虚拟域 id 映射 */
};

/* per-vcpu：迁移冲刷标记 */
	/* vcpu_put 时若发生过迁移（cpu != last_cpu），执行本地 vmalle1 */
```

### 6.2 迁移协议伪代码

```c
/* PE_A: kvm_vcpu_put_vhe 路径 */
if (vcpu->cpu != smp_processor_id() && kvm_uses_tlbid(kvm)) {
	__kvm_flush_cpu_context(&vcpu->arch.hw_mmu);   /* 复用现有: vmalle1 + ic iallu */
}

/* PE_B: kvm_vcpu_load_vhe 路径 */
if (kvm_uses_tlbid(kvm) && mmu_switched) {
	write_sysreg(kvm->tlbid.vtlbid[0], vtlbid0_el2);   /* … 共 4/8 个 */
	/* HCRX_EL2.VTLBIDEn = 1 已在 __compute_hcr 中设置 */
}
```

### 6.3 动手前必须验证的 spec 未决项（设计分支点）

| 编号 | 问题 | 影响 |
|---|---|---|
| **G1** | PE 的**物理域成员关系**如何确定：接收方 PE 的 VTLBID TD 表内容（双职责模型）还是静态 fabric/firmware ID？ | **host 侧域化（L1）的正确性证明完全取决于此**。双职责模型下 host flush 打不到正在跑 guest 的 PE（表里没有 host 域），需 host 域常驻所有表（烧 TD 槽位）或回退广播；静态 ID 模型下 host 域化天然安全 |
| **G2** | FEAT_TLBID 实现时 TLBID=`0b11111` 的语义：全域广播还是具体域？ | 决定"回退全广播"的编码方式与 guest 旧代码的兼容行为（倾向全广播，但需 spec 确认） |
| G3 | TLBID 字段位置：无操作数指令 [15:0] vs 有操作数指令 [31:16]？ | 汇编封装与内核内联 asm 的写法 |
| G4 | NV（嵌套虚拟化）与域翻译的叠加语义 | L3 是否碰 NV |
| G5 | guest 可见的 ID 寄存器位 | QEMU 暴露与 guest 检测 |

**G1/G2 两个问题查清之前，L1/L2 只能做正确性中立的部分**（检测、保存/恢复、协议骨架），不能打开域化开关。

---

## 7. 结论

1. **用户的 1 vcpu/1 domain 思路与 spec 的"虚拟域层"设计方向一致**，且"乱飘不损精度"的直觉在虚拟域层严格成立——域随 vCPU 走，物理迁移由 hypervisor 吸收。
2. **方案的三处妥协点**：guest 域数硬顶 32/16（超限分组或回退）；域只覆盖广播类指令（range flush 不受益）；host 侧域化受 G1 约束（先验证 spec）。
3. **trap-转发方案否决为主路径**：`HCRX_EL2.VTLBIDEn + HCR_EL2.FB` 两个寄存器位就是无 trap 的"转发开关"，硬件翻译取代软件转发；`TTLBIS/TTLBOS` 选择性 trap 只留作兜底。
4. **核心协议是"迁移即冲刷"**：vcpu_put 本地 `vmalle1` + vcpu_load 重编程 TD 表，把乱飘成本从广播面压成 O(迁移次数)。VMID 轮转全广播是不可碰的安全网。
5. **落地顺序**：L0（零变化基线）→ L1（host 自用，需 G1 验证）→ L2（guest-aware 1:1 域协议）→ L3（PV/NV 演进）。OpenEuler OLK-6.6 的 mm_cpumask 域化补丁是 L1 的现成参考实现 [EXTERNAL]。

---

## 参考链接

**Spec 文档（Arm 官方）**
- [TLBIDIDR_EL1 — TLBI Domains Identification Register](https://developer.arm.com/documentation/ddi0601/2025-12/AArch64-Registers/TLBIDIDR-EL1--TLBI-Domains-Identification-Register--EL1-)
- [VTLBID<n>_EL2 — Virtual TLBI Domain Registers](https://developer.arm.com/documentation/111107/2026-03/AArch64-Registers/VTLBID-n--EL2--Virtual-TLBI-Domain-Registers)
- [VTLBIDOS<n>_EL2 — Virtual TLBI Domain Outer Shareable Registers](https://developer.arm.com/documentation/111107/2026-03/AArch64-Registers/VTLBIDOS-n--EL2--Virtual-TLBI-Domain-Outer-Shareable-Registers)
- [TLBI VMALLE1IS 指令页](https://developer.arm.com/documentation/ddi0595/2020-12/AArch64-Instructions/TLBI-VMALLE1IS--TLBI-VMALLE1ISNXS--TLB-Invalidate-by-VMID--All-at-stage-1--EL1--Inner-Shareable)
- [PLBI ALLE1IS 指令页（TLBID 字段位置）](https://developer.arm.com/documentation/111107/2025-09/AArch64-Instructions/PLBI-ALLE1IS--PLBI-ALLE1ISNXS--PLB-Invalidate-All--EL1--Inner-Shareable)
- [2025 Architecture Extensions 文档](https://developer.arm.com/documentation/109697/2025_09/2025-Architecture-Extensions)

**工具链**
- [LLVM PR #163156 — Armv9.7-A: Add support for TLBI Domains (FEAT_TLBID)](https://github.com/llvm/llvm-project/pull/163156)
- [LLVM PR #177995 / #178913 — Gate TLBIP insns with +tlbid or +d128](https://github.com/llvm/llvm-project/pull/177995)
- [binutils 补丁系列 cover letter（+tlbid）](https://sourceware.org/pipermail/binutils/2026-January/147699.html)

**内核/虚拟化相关**
- [RFC KVM: arm64: Don't force broadcast tlbi when guest is running（2020-10，广播失效规模化痛点）](http://lists.openwrt.org/pipermail/linux-arm-kernel/2020-October/611049.html)
- [OpenEuler OLK-6.6: arm64: mm: Track CPUs for TLBID optimization（mm_cpumask 域化 flush_tlb_mm）](https://mailweb.openeuler.org/archives/list/kernel@openeuler.org/message/VSODAF7RGDJN4KMEJQIS3K2WHWXVAPYF/)
- [gem5: Implement HCR_EL2 force broadcast for EL1&0 TLBIs](https://github.com/gem5/gem5/commit/c4ed23a10b51f2f4f3c40fac060e8b34f5b18848)

**第三方 sysreg 镜像**
- [arm.jonpalmisc.com — VTLBID<n>_EL2](https://arm.jonpalmisc.com/latest_sysreg/AArch64-vtlbidn_el2)
- [arm.jonpalmisc.com — TLBIDIDR_EL1](https://arm.jonpalmisc.com/latest_sysreg/AArch64-tlbididr_el1)

---

*文档生成时间：2026-09-01；基于仓库 v7.3（branch vgic-ap1r-64bit）代码与截至当日可获得的网络资料。*
