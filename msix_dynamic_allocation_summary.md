# VFIO/PCI 动态 MSI-X 分配 — 研究总结

## 1. 问题：为什么需要动态 MSI-X 分配

### 1.1 现状（无动态分配）

QEMU 以增量方式分配 MSI-X 中断——每当 guest unmask 一个新 vector 时（如 `request_irq()`），QEMU 需要通过 `VFIO_DEVICE_SET_IRQS` 扩展已分配的 vector 范围。

由于内核之前不支持 MSI-X 启用后新增 vector（v6.2 之前），QEMU 只能：

1. **先禁用 MSI-X 并释放所有已分配中断**
2. **重新启用 MSI-X 并分配包含新 vector 的更大范围**

**以 guest unmask vector#1 为例**（假设之前仅分配了 vector#0）：

```
步骤 4.a: VFIO_DEVICE_SET_IRQS → 禁用并释放 vector#0 的 IRQ
步骤 4.b: VFIO_DEVICE_SET_IRQS → 分配并启用 vector#0 + vector#1
```

在 4.a 到 4.b 之间，vector#0 的物理中断被拆除，但 guest 认为该 vector 是 unmasked 的 → **中断丢失风险**。

### 1.2 影响场景

- 加速器设备有多个工作队列，每个队列有专用中断
- 工作队列可在运行时任意启用/禁用
- "释放全部再重分配"在初始化阶段可接受，**运行时则中断丢失**

### 1.3 MSI-X 规范层面的论证

Thomas Gleixner 和 Alex Williamson 的讨论确认：

- MSI-X 规范**不要求**禁用再重启用来扩展 vectors（与 MSI 不同）
- MSI-X 没有类似 MSI 的"number of used vectors"控制字段
- Masked entry 不需要包含有效 message
- Unmask 时硬件必须重新读取 table entry（PCIe 5.0 spec 6.1.4.5）
- **结论**：NORESIZE 限制是内核实现层面的限制，而非 MSI-X 规范本身的限制

---

## 2. 内核侧解决方案：V5 Patch Series 详细代码分析

**作者**: Reinette Chatre (reinette.chatre@intel.com)
**标题**: [PATCH V5 00/11] vfio/pci: Support dynamic allocation of MSI-X interrupts
**4 files, +229 -113**

### Patch 01/11: Consolidate irq cleanup on MSI/MSI-X disable

**Commit**: `a65f35cfd504` | `vfio_pci_intrs.c` (+1 -2)

`vfio_msi_disable()` 原来对中断状态做两次遍历：先是 for 循环做 virqfd cleanup (unmask/mask)，再通过 `vfio_msi_set_block()` 内的循环调用 `vfio_msi_set_vector_signal()` 移除 handler。合并为单次循环：

```c
for (i = 0; i < vdev->num_ctx; i++) {
    vfio_virqfd_disable(&vdev->ctx[i].unmask);
    vfio_virqfd_disable(&vdev->ctx[i].mask);
    vfio_msi_set_vector_signal(vdev, i, -1, msix);  // 合入此调用
}
// 删除: vfio_msi_set_block(vdev, 0, vdev->num_ctx, NULL, msix);
```

### Patch 02/11: Remove negative check on unsigned vector

**Commit**: `6578ed85c7d6` | `vfio_pci_intrs.c` (+8 -7)

vector 参数是 `unsigned int`，用户空间在 `vfio_set_irqs_validate_and_prepare()` 中已做合法性检查，后续的 `< 0` 检查毫无意义。

关键改动：
- `vfio_msi_set_vector_signal()` 参数 `int vector` → `unsigned int vector`
- `if (vector < 0 || vector >= vdev->num_ctx)` → `if (vector >= vdev->num_ctx)`
- `vfio_msi_set_block()` 循环变量 `int i, j` → `unsigned int i, j`
- 错误回退从逆向遍历改为正向：`for (--j; j >= (int)start; j--)` → `for (i = start; i < j; i++)`

### Patch 03/11: Prepare for dynamic interrupt context storage

**Commit**: `d977e0f76639` | `vfio_pci_intrs.c` (+149 -66)

为数据结构切换铺路：将所有直接访问 `vdev->ctx[index]` 改为通过辅助函数间接访问。

新增三个辅助函数（此时仍操作数组）：

```c
struct vfio_pci_irq_ctx *vfio_irq_ctx_get(vdev, index) {
    if (index >= vdev->num_ctx) return NULL;
    return &vdev->ctx[index];
}

void vfio_irq_ctx_free_all(vdev) { kfree(vdev->ctx); }

int vfio_irq_ctx_alloc_num(vdev, num) {
    vdev->ctx = kcalloc(num, sizeof(...), GFP_KERNEL_ACCOUNT);
}
```

所有 `vdev->ctx[index].xxx` → `ctx->xxx`（通过 `vfio_irq_ctx_get()` 获取指针），例如：
- `vdev->ctx[0].trigger` → `ctx->trigger`（INTx）
- `vdev->ctx[vector].masked` → `ctx->masked`（MSI/MSI-X）

同时增加 `WARN_ON_ONCE(!ctx)` 安全检查，确保在 INTx mask/unmask 场景中延迟获取上下文不会导致空指针（V5 新增的 bug 修复）。

### Patch 04/11: Move to single error path

**Commit**: `8850336588fb` | `vfio_pci_intrs.c` (+10 -7)

`vfio_msi_set_vector_signal()` 改为 goto 模式统一错误处理：

```c
trigger = eventfd_ctx_fdget(fd);
if (IS_ERR(trigger)) {
    ret = PTR_ERR(trigger);
    goto out_free_name;          // 不再直接 kfree(ctx->name) + return
}

ret = request_irq(irq, vfio_msihandler, 0, ctx->name, trigger);
vfio_pci_memory_unlock_and_restore(vdev, cmd);
if (ret)
    goto out_put_eventfd_ctx;    // 不再就地 cleanup

ctx->trigger = trigger;
return 0;

out_put_eventfd_ctx:
    eventfd_ctx_put(trigger);
out_free_name:
    kfree(ctx->name);
    return ret;
```

为后续动态上下文分配引入更多步骤（如 `vfio_irq_ctx_alloc`）时减少重复错误处理。

### Patch 05/11: Use xarray for interrupt context storage ★核心结构性改动

**Commit**: `b156e48fffa9` | `vfio_pci_core.c` (+1), `vfio_pci_intrs.c` (+48 -46), `vfio_pci_core.h` (+1 -1)

**核心变更**：中断上下文存储从静态数组切换为 xarray。

```c
// vfio_pci_core.h
struct vfio_pci_irq_ctx *ctx;  →  struct xarray ctx;

// vfio_pci_core_init_dev()
xa_init(&vdev->ctx);           // 初始化 xarray
```

辅助函数从数组操作改为 xarray 操作：

```c
// 获取
struct vfio_pci_irq_ctx *vfio_irq_ctx_get(vdev, index) {
    return xa_load(&vdev->ctx, index);     // 从 &vdev->ctx[index] → xa_load
}

// 单个释放
void vfio_irq_ctx_free(vdev, ctx, index) {
    xa_erase(&vdev->ctx, index);           // 从 kfree(vdev->ctx) → xa_erase + kfree
    kfree(ctx);
}

// 单个分配
struct vfio_pci_irq_ctx *vfio_irq_ctx_alloc(vdev, index) {
    ctx = kzalloc(sizeof(*ctx), GFP_KERNEL_ACCOUNT);
    ret = xa_insert(&vdev->ctx, index, ctx, GFP_KERNEL_ACCOUNT);
    if (ret) { kfree(ctx); return NULL; }
    return ctx;
}
```

**关键语义变更**：中断上下文不再在 MSI/MSI-X enable 时预分配整个范围，改为在 `vfio_msi_set_vector_signal()` 中按需单个分配：

```c
// vfio_msi_set_vector_signal() — 启用路径（fd >= 0）
ctx = vfio_irq_ctx_get(vdev, vector);  // 先查是否已有
if (ctx) {                              // 已启用 → 先关闭
    free_irq(irq, ctx->trigger);
    kfree(ctx->name);
    eventfd_ctx_put(ctx->trigger);
    vfio_irq_ctx_free(vdev, ctx, vector); // 释放上下文
}
if (fd < 0) return 0;                   // fd < 0 = 仅关闭，到此结束

ctx = vfio_irq_ctx_alloc(vdev, vector); // 按需分配新上下文
if (!ctx) return -ENOMEM;
```

**只有启用的中断才有上下文 = "Active Contexts"**。`pci_irq_vector()` 返回值判断中断是否可被启用。

`vfio_msi_enable()` 删除 `vfio_irq_ctx_alloc_num()` 预分配调用。`vfio_msi_disable()` 改用 `xa_for_each()` 遍历所有 active contexts。

`vfio_pci_set_msi_trigger()` 中的 `!ctx->trigger` 检查简化为 `!ctx`——ctx 存在即意味着 trigger 存在（active context）。

### Patch 06/11: Remove interrupt context counter

**Commit**: `63972f63a63f` | `vfio_pci_intrs.c` (+1 -13), `vfio_pci_core.h` (-1)

xarray 稀疏存储下，`num_ctx` 不再是可靠的上界索引。删除 `struct vfio_pci_core_device::num_ctx` 字段及所有赋值/检查：

删除的代码：
- `vdev->num_ctx = 1` (intx_enable), `vdev->num_ctx = 0` (intx_disable)
- `vdev->num_ctx = nvec` (msi_enable), `vdev->num_ctx = 0` (msi_disable)
- `if (vector >= vdev->num_ctx) return -EINVAL` (set_vector_signal)
- `if (start >= vdev->num_ctx || start + count > vdev->num_ctx) return -EINVAL` (set_block)
- `if (!irq_is(vdev, index) || start + count > vdev->num_ctx)` → `if (!irq_is(vdev, index))`

合法性判断改用 `pci_irq_vector()` 返回值。

### Patch 07/11: Update stale comment

**Commit**: `9387cf59dc6f` | `vfio_pci_intrs.c` (+3 -5)

修正 `vfio_msi_set_vector_signal()` 中关于 MSI-X vector table 的注释：

```c
// 之前 (不准确):
// "The MSIx vector table resides in device memory which may be cleared
//  via backdoor resets. We don't allow direct access to the vector
//  table so even if a userspace driver attempts to save/restore around
//  such a reset it would be unsuccessful. To avoid this, restore the
//  cached value of the message prior to enabling."

// 之后 (准确):
// "If the vector was previously allocated, refresh the on-device
//  message data before enabling in case it had been cleared or
//  corrupted (e.g. due to backdoor resets) since writing."
```

删除了关于"不允许直接访问 vector table"的不准确描述，聚焦于刷新 on-device message data 的目的。

### Patch 08/11: Use bitfield for struct vfio_pci_core_device flags

**Commit**: `9cd0f6d5cbb6` | `vfio_pci_core.h` (+11 -11)

11 个 `bool` 标志各占 1 byte = 11 bytes，改为 `bool xxx:1` 位域仅占约 2 bytes：

```c
bool pci_2_3;              →  bool pci_2_3:1;
bool virq_disabled;        →  bool virq_disabled:1;
bool reset_works;          →  bool reset_works:1;
bool extended_caps;        →  bool extended_caps:1;
bool bardirty;             →  bool bardirty:1;
bool has_vga;              →  bool has_vga:1;
bool needs_reset;          →  bool needs_reset:1;
bool nointx;               →  bool nointx:1;
bool needs_pm_restore;     →  bool needs_pm_restore:1;
bool pm_intx_masked;       →  bool pm_intx_masked:1;
bool pm_runtime_engaged;   →  bool pm_runtime_engaged:1;
```

为新 flag `has_dyn_msix` 腾出空间而不增加结构体大小。

### Patch 09/11: Probe and store ability to support dynamic MSI-X

**Commit**: `dd27a7070038` | `vfio_pci_core.c` (+5 -1), `vfio_pci_core.h` (+1)

在 `vfio_pci_core_enable()` 中一次性探测并存储：

```c
if (pdev->msix_cap) {
    vdev->msix_bar = table & PCI_MSIX_TABLE_BIR;
    vdev->msix_offset = table & PCI_MSIX_TABLE_OFFSET;
    vdev->msix_size = ((flags & PCI_MSIX_FLAGS_QSIZE) + 1) * 16;
    vdev->has_dyn_msix = pci_msix_can_alloc_dyn(pdev);  // 新增
} else {
    vdev->msix_bar = 0xFF;
    vdev->has_dyn_msix = false;                           // 新增
}
```

新增 `bool has_dyn_msix:1` 位域标志。避免后续每次操作都查询 `pci_msix_can_alloc_dyn()`。

### Patch 10/11: Support dynamic MSI-X ★核心功能实现

**Commit**: `e4163438e015` | `vfio_pci_intrs.c` (+41 -6)

新增 `vfio_msi_alloc_irq()` 函数——动态分配的核心：

```c
static int vfio_msi_alloc_irq(struct vfio_pci_core_device *vdev,
                              unsigned int vector, bool msix)
{
    struct pci_dev *pdev = vdev->pdev;
    struct msi_map map;
    int irq;
    u16 cmd;

    irq = pci_irq_vector(pdev, vector);
    if (WARN_ON_ONCE(irq == 0))    // irq==0 视为 kernel bug
        return -EINVAL;
    if (irq > 0 || !msix || !vdev->has_dyn_msix)
        return irq;                 // 已有IRQ或不支持动态 → 直接返回

    // 动态分配新IRQ
    cmd = vfio_pci_memory_lock_and_enable(vdev);
    map = pci_msix_alloc_irq_at(pdev, vector, NULL);
    vfio_pci_memory_unlock_and_restore(vdev, cmd);

    return map.index < 0 ? map.index : map.virq;
}
```

逻辑：
1. `pci_irq_vector()` 查询已有 IRQ → `> 0` 直接复用
2. `== 0` → kernel bug (WARN)
3. `< 0` 且 MSI-X + `has_dyn_msix` → 调用 `pci_msix_alloc_irq_at()` 动态分配
4. `< 0` 且不支持动态 → 返回负值错误

`vfio_msi_set_vector_signal()` 修改为使用此函数：

```c
static int vfio_msi_set_vector_signal(vdev, vector, fd, msix)
{
    int irq = -EINVAL, ret;   // irq 初始值改为 -EINVAL

    ctx = vfio_irq_ctx_get(vdev, vector);

    if (ctx) {               // 已有上下文 → 先关闭
        irq = pci_irq_vector(pdev, vector);  // 关闭时需要 IRQ 号
        free_irq(irq, ctx->trigger);
        /* Interrupt stays allocated, will be freed at MSI-X disable. */
        kfree(ctx->name);
        eventfd_ctx_put(ctx->trigger);
        vfio_irq_ctx_free(vdev, ctx, vector);
    }

    if (fd < 0) return 0;    // 仅关闭，到此结束

    if (irq == -EINVAL) {    // 关闭后 irq 仍为初始值 → 需分配新 IRQ
        /* Interrupt stays allocated, will be freed at MSI-X disable. */
        irq = vfio_msi_alloc_irq(vdev, vector, msix);
        if (irq < 0) return irq;
    }

    ctx = vfio_irq_ctx_alloc(vdev, vector);   // 分配新上下文
    ...
}
```

**为什么没有 `vfio_msi_free_irq()`？** 注释明确说明：
> Allocated interrupts are maintained, essentially forming a cache that subsequent allocations can draw from. Interrupts are freed using pci_free_irq_vectors() when MSI/MSI-X is disabled.

动态分配的 IRQ 不主动释放，仅在 MSI-X disable 或设备释放时通过 `pci_free_irq_vectors()` 统一释放。优势：降低延迟、提高可靠性、简化实现。

### Patch 11/11: Clear VFIO_IRQ_INFO_NORESIZE for MSI-X ★用户空间接口

**Commit**: `6c8017c6a58d` | `vfio_pci_core.c` (+4 -1), `include/uapi/linux/vfio.h` (+3)

`vfio_pci_ioctl_get_irq_info()` 中条件化 NORESIZE 标志：

```c
// 之前: MSI 和 MSI-X 都设置 NORESIZE
else
    info.flags |= VFIO_IRQ_INFO_NORESIZE;

// 之后: MSI-X 支持动态时清除 NORESIZE
else if (info.index != VFIO_PCI_MSIX_IRQ_INDEX || !vdev->has_dyn_msix)
    info.flags |= VFIO_IRQ_INFO_NORESIZE;
```

`include/uapi/linux/vfio.h` 新增文档：
```c
/*
 * ...
 * Absence of the NORESIZE flag indicates that vectors can be enabled
 * and disabled dynamically without impacting other vectors within the
 * index.
 */
struct vfio_irq_info {
    __u32   argsz;
    __u32   flags;
    ...
};
```

---

## 3. QEMU 侧解决方案

Jing Liu 提出的 [PATCH v3 0/4]：

| # | Patch | 内容 |
|---|-------|------|
| 1 | detect the support of dynamic MSI-X allocation | 通过 NORESIZE 标志检测动态 MSI-X 能力 |
| 2 | enable vector on dynamic MSI-X allocation | 支持动态时单独启用 vector |
| 3 | use an invalid fd to enable MSI-X | 用 vector 0 + 无效 fd 启用 MSI-X |
| 4 | enable MSI-X in interrupt restoring on dynamic allocation | 迁移恢复时同样使用动态策略 |

Patch 4 的核心代码（`hw/vfio/pci.c`）：

```c
// vfio_enable_vectors() — 迁移恢复时
if (msix && !vdev->msix->noresize) {
    ret = vfio_enable_msix_no_vec(vdev);  // 用 vector 0 + 无效 fd 仅启用 MSI-X
    if (ret) return ret;
}
```

策略：检测到 !NORESIZE → 先启用 MSI-X（不分配实际中断），再按需逐个分配 guest unmasked 的 vector。

---

## 4. 完整流程对比

### 之前（NORESIZE）

```
guest unmask vector#1 (当前仅分配 vector#0):
  1. VFIO_DEVICE_SET_IRQS → 释放所有中断 + 禁用 MSI-X
  2. VFIO_DEVICE_SET_IRQS → 重分配 vector#0~#1 + 启用 MSI-X
  ⚠ 步骤 1→2 间 vector#0 中断丢失风险
```

### 之后（!NORESIZE / 动态 MSI-X）

```
guest unmask vector#0 (初始启用 MSI-X):
  VFIO_DEVICE_SET_IRQS → vector 0 + 无效 fd → 仅启用 MSI-X

guest unmask vector#1:
  VFIO_DEVICE_SET_IRQS → vfio_msi_alloc_irq() 动态分配 vector#1 IRQ
  ✓ vector#0 不受影响，无中断丢失

guest unmask vector#2:
  VFIO_DEVICE_SET_IRQS → vfio_msi_alloc_irq() 动态分配 vector#2 IRQ
  ✓ vector#0~#1 不受影响
```

---

## 5. 参考链接

### 内核 Patch Series（所有版本）

| 版本 | Cover Letter |
|------|-------------|
| V5 (最终合入) | https://lore.kernel.org/lkml/cover.1683740667.git.reinette.chatre@intel.com/ |
| V4 | https://lore.kernel.org/lkml/cover.1682615447.git.reinette.chatre@intel.com/ |
| V3 | https://lore.kernel.org/lkml/cover.1681837892.git.reinette.chatre@intel.com/ |
| V2 | https://lore.kernel.org/lkml/cover.1680038771.git.reinette.chatre@intel.com/ |
| RFC V1 | https://lore.kernel.org/lkml/cover.1678911529.git.reinette.chatre@intel.com/ |

### V5 各 Patch 链接

| # | Patch | lore.kernel.org 链接 |
|---|-------|----------------------|
| 01 | Consolidate irq cleanup | https://lore.kernel.org/r/837acb8cbe86a258a50da05e56a1f17c1a19abbe.1683740667.git.reinette.chatre@intel.com |
| 02 | Remove negative check on unsigned vector | https://lore.kernel.org/r/28521e1b0b091849952b0ecb8c118729fc8cdc4f.1683740667.git.reinette.chatre@intel.com |
| 03 | Prepare for dynamic interrupt context storage | https://lore.kernel.org/r/eab289693c8325ede9aba99380f8b8d5143980a4.1683740667.git.reinette.chatre@intel.com |
| 04 | Move to single error path | https://lore.kernel.org/r/72dddae8aa710ce522a74130120733af61cffe4d.1683740667.git.reinette.chatre@intel.com |
| 05 | Use xarray for interrupt context storage | https://lore.kernel.org/r/40e235f38d427aff79ae35eda0ced42502aa0937.1683740667.git.reinette.chatre@intel.com |
| 06 | Remove interrupt context counter | https://lore.kernel.org/r/e27d350f02a65b8cbacd409b4321f5ce35b3186d.1683740667.git.reinette.chatre@intel.com |
| 07 | Update stale comment | https://lore.kernel.org/r/5b605ce7dcdab5a5dfef19cec4d73ae2fdad3ae1.1683740667.git.reinette.chatre@intel.com |
| 08 | Use bitfield for struct flags | https://lore.kernel.org/r/cf34bf0499c889554a8105eeb18cc0ab673005be.1683740667.git.reinette.chatre@intel.com |
| 09 | Probe and store dynamic MSI-X ability | https://lore.kernel.org/r/f1ae022c060ecb7e527f4f53c8ccafe80768da47.1683740667.git.reinette.chatre@intel.com |
| 10 | Support dynamic MSI-X | https://lore.kernel.org/r/956c47057ae9fd45591feaa82e9ae20929889249.1683740667.git.reinette.chatre@intel.com |
| 11 | Clear VFIO_IRQ_INFO_NORESIZE for MSI-X | https://lore.kernel.org/r/fd1ef2bf6ae972da8e2805bc95d5155af5a8fb0a.1683740667.git.reinette.chatre@intel.com |

### LWN 公告

- https://lwn.net/Articles/931679/

### 问题起源讨论线程

- 原始邮件 (Kevin Tian → Alex Williamson): https://lore.kernel.org/kvm/MWHPR11MB188603D0D809C1079F5817DC8C099@MWHPR11MB1886.namprd11.prod.outlook.com/#t
- Alex Williamson 首次回复: https://lore.kernel.org/kvm/20210623204828.2bc7e6dc.alex.williamson@redhat.com/
- Thomas Gleixner 评论 (MSI-X 规范分析): https://lore.kernel.org/kvm/87im22uncn.ffs@nanos.tec.linutronix.de/
- Alex Williamson 关于 NORESIZE 的回应: https://lore.kernel.org/kvm/20210624115236.309d6b48.alex.williamson@redhat.com/
- Thomas Gleixner 关于删除 NORESIZE 的激进提案: https://lore.kernel.org/kvm/87czs9vdzl.ffs@nanos.tec.linutronix.de/
- Alex Williamson 关于 MSI-X spec 的详细分析: https://lore.kernel.org/kvm/20210624154434.11809b8f.alex.williamson@redhat.com/

### IMS 相关参考

- IMS irqchip 讨论: https://lore.kernel.org/kvm/87im2lyiv6.ffs@nanos.tec.linutronix.de/
- IMS generic irqchip 实现: https://lore.kernel.org/linux-hyperv/20200826112335.202234502@linutronix.de/

### QEMU 侧 Patch

- https://patchew.org/QEMU/20230926021407.580305-1-jing2.liu@intel.com/20230926021407.580305-5-jing2.liu@intel.com/

### 前置内核 Commit

- `34026364df8e` — PCI/MSI: Provide post-enable dynamic allocation interfaces for MSI-X (v6.2)
- `195d8e5da3ac` — PCI/MSI: Provide missing stub for pci_msix_can_alloc_dyn() (v6.3-rc7)

### 本地导出的 Patch 文件

- `/home/yangjinqian/ai_output/0001*.patch` ~ `0011*.patch` (11 个 V5 版本完整 diff)

---

## 6. ARM64 上的动态 MSI-X 问题与进展

### 6.1 现状

ARM64 GIC ITS 的 PCI MSI domain **未设置** `MSI_FLAG_PCI_MSIX_ALLOC_DYN` flag：

```c
// x86: arch/x86/include/asm/msi.h:65
#define X86_VECTOR_MSI_FLAGS_SUPPORTED  \
    (MSI_GENERIC_FLAGS_MASK | MSI_FLAG_PCI_MSIX | MSI_FLAG_PCI_MSIX_ALLOC_DYN)

// ARM64: drivers/irqchip/irq-gic-v3-its-msi-parent.c:17
#define ITS_MSI_FLAGS_SUPPORTED  \
    (MSI_GENERIC_FLAGS_MASK | MSI_FLAG_PCI_MSIX | MSI_FLAG_MULTI_PCI_MSI)
    // ← 没有 MSI_FLAG_PCI_MSIX_ALLOC_DYN !
```

因此在 ARM64 上 `pci_msix_can_alloc_dyn()` 返回 false，VFIO 的 `has_dyn_msix` 为 false，
NORESIZE 标志不会被清除。VFIO PCI 直通场景仍受"禁用→重分配→启用"旧方式限制，存在中断丢失风险。

### 6.2 为什么 x86 解决了而 ARM64 还没解决

核心原因是 **x86 和 ARM64 的中断硬件架构完全不同**。

#### x86：天然支持，加个 flag 即可

x86 的 MSI-X 直接写入 local APIC：

- 每个 MSI message 包含 `{目标 APIC ID, vector号}` → APIC 直接路由到目标 CPU
- 不需要任何中间翻译表
- `pci_msi_prepare()` 只做一件事：`init_irq_alloc_info(arg, NULL)`，3 行代码
- 新增 vector = 在目标 CPU 上分配一个空闲 vector 号，**无需任何全局结构变更**

所以 x86 启用 `MSI_FLAG_PCI_MSIX_ALLOC_DYN` 就是加一个 flag，**零额外实现**。

#### ARM64：三层翻译架构带来三个阻塞点

GIC ITS 使用 **DeviceID + EventID → LPI** 的两级翻译：

```
PCI设备 → MSI-X Table(MSI地址+数据) → ITS翻译 → LPI → CPU
```

##### 阻塞点 1：ITT 必须一次性分配，大小在创建时固定

`its_create_device()` 在 `msi_prepare()` 时创建设备，一次性：

- 分配 ITT（大小 = `2^ceil(log2(nvecs))`，**必须是 2 的幂**）
- 一次性分配 LPI bitmap + collection map
- 发送 MAPD 命令给 ITS 硬件绑定设备与 ITT

关键代码 (`drivers/irqchip/irq-gic-v3-its.c:3868`):

```c
nr_ites = max(2, nvecs);
sz = nr_ites * (FIELD_GET(GITS_TYPER_ITT_ENTRY_SIZE, its->typer) + 1);
sz = max(sz, ITS_ITT_ALIGN);
itt = itt_alloc_pool(its->numa_node, sz);
```

ITT **大小在创建时就固定了**，ITS 硬件不支持在线 resize ITT。
动态分配意味着每次新增 vector 时可能需要更大的 ITT，但无法在线扩展。
解决方案只能在 `msi_prepare()` 时预分配足够大的 ITT（覆盖 hwsize），但这又需要
`msi_prepare()` 在正确的时机被调用并拿到正确的大小参数。

##### 阻塞点 2：`msi_prepare()` 调用时机错误

ITS 的 `msi_prepare()` 有严格要求：**必须在设备生命周期内只调用一次**，在任何 MSI 分配之前。
但旧 MSI 核心代码的调用时机是 "semi-random" 的（Marc Zyngier 的描述），导致 ITS
不得不加了一个 ugly hack 来确保 ITT 足够大：

```c
// 旧代码 (irq-gic-v3-its-msi-parent.c，已被删除)
msi_info = msi_get_domain_info(domain);
if (msi_info->hwsize > nvec)
    nvec = msi_info->hwsize;  // 强制向上取到 hwsize
```

这个 hack 恰恰是动态分配要**去掉**的东西——去掉后 ITT 大小才由实际分配的 nvec 决定，
但去掉它需要先修好 `msi_prepare()` 的调用时机，否则 ITT 大小会不正确。

##### 阻塞点 3：LPI 分配的全局性

- x86 的 vector 是 **per-CPU** 的（每 CPU 256 个），分配局部、无竞争
- ARM64 的 LPI 是 **全局共享池**（8192+ 范围），`its_lpi_alloc()` 需要全局 bitmap 操作
- 动态分配时每次新增 vector 都要做全局 LPI 分配，需确保与已有 LPI 分配状态一致

### 6.3 解决进展

| 时间 | x86 | ARM64 |
|------|-----|-------|
| 2022.11 | Gleixner 重构 MSI 核心框架，x86 天然适配，直接加 `MSI_FLAG_PCI_MSIX_ALLOC_DYN` | — |
| 2022.12 | x86 动态 MSI-X 合入 v6.2 | — |
| 2023.05 | VFIO PCI 动态分配 patch (Reinette) 合入 | ITS 仍使用旧的 pci-msi.c 分层 |
| 2024.06 | — | Gleixner 开始 ITS MSI parent 重构 (`b5712bf89b4b` 等) |
| 2025.05 | — | Marc Zyngier 修复 `msi_prepare()` 调用时机 + 删除 ugly hack (`7dd20bf2f010`) |
| 至今 | — | 前置修复已合入，但 **`MSI_FLAG_PCI_MSIX_ALLOC_DYN` 尚未正式启用** |

#### Marc Zyngier 的前置修复 Patch Series

**标题**: [PATCH v2 0/5] genirq/msi: Fix device MSI prepare/alloc sequencing
**日期**: 2025-05-13
**已合入 commit**: `7dd20bf2f010`

| # | Patch | 内容 |
|---|-------|------|
| 1 | Add .msi_teardown() callback | 作为 .msi_prepare() 的逆操作 |
| 2 | ITS: Implement .msi_teardown() | ITS 级别实现 |
| 3 | Move prepare() call to per-device allocation | 修复 msi_prepare() 调用时机 |
| 4 | Engage .msi_teardown() on domain removal | domain 销毁时调用 teardown |
| 5 | ITS: Use allocation size from the prepare call | 删除 ugly hack |

#### Gleixner 与 Marc Zyngier 关于启用动态 MSI-X 的讨论 (2025-05)

Gleixner 在 review patch 5/5 时提出：

> FWIW, while looking at something related, it occured to me that with
> this change you can enable MSI_FLAG_PCI_MSIX_ALLOC_DYN now on GIC ITS.

Marc Zyngier 的疑问：

> What is the endpoint driver allowed to expect in terms of continuity
> of allocation in the IRQ space? If this is solely limited to MSI-X,
> then the answer probably is "none whatsoever".
>
> Can any other MSI-like mechanism end-up with multiple allocations and
> require extra alignment/contiguity guarantees in the hwirq space?

Gleixner 的确认：

> It's only relevant to MSI-X today. That's the only facility, which
> actually provides an interface _if_ the underlying parent supports it.
> MSI-X 不需要 hwirq 连续性，driver 只需管理 MSI descriptor index。

**结论**：前置修复（msi_prepare 调用时机 + ugly hack 删除）已合入。
技术上 GIC ITS 已具备启用 `MSI_FLAG_PCI_MSIX_ALLOC_DYN` 的条件，
但 Marc Zyngier 尚未正式提交启用该 flag 的 patch。
ITT 预分配策略的最终方案（预分配到 hwsize vs. 动态扩展）仍需明确。

### 6.4 参考链接

#### ITS msi_prepare 修复

- Cover letter v2: https://lore.kernel.org/lkml/20250513163144.2215824-1-maz@kernel.org/
- Patch 5/5 (删除 ugly hack): https://lore.kernel.org/lkml/20250513163144.2215824-6-maz@kernel.org/

#### Gleixner 与 Marc 关于动态 MSI-X 的讨论

- Gleixner 首次提出可启用: https://lore.kernel.org/lkml/8734d1iwcp.ffs@tglx/
- Marc 的疑问: https://lore.kernel.org/lkml/86wmacewjr.wl-maz@kernel.org/
- Gleixner 解释动态分配意义: https://lore.kernel.org/lkml/87zff8hk1x.ffs@tglx/
- Marc 关于连续性保证的追问: https://lore.kernel.org/lkml/86v7pwekum.wl-maz@kernel.org/
- Gleixner 确认仅 MSI-X 需要: https://lore.kernel.org/lkml/871psirnh1.ffs@tglx/

#### ITS MSI parent 重构

- Provide MSI parent infrastructure: commit `48f71d56e2b8`
- Provide MSI parent for PCI/MSI[-X]: commit `b5712bf89b4b`

---

## 7. Marc Zyngier Patch Series 详细分析：为什么修完后可以动态分配

**标题**: [PATCH v2 0/5] genirq/msi: Fix device MSI prepare/alloc sequencing
**日期**: 2025-05-13
**已合入 commit**: `7dd20bf2f010` (patch 5/5)

### 7.1 旧代码的根本问题：msi_prepare() 被多次错误调用

旧 MSI 核心代码中，`msi_prepare()` 在每次 MSI 分配时都被调用：

```
__msi_domain_alloc_irqs()
    → msi_domain_prepare_irqs(domain, dev, ctrl->nirqs, &arg)
        → ops->msi_prepare(domain, dev, nvec, arg)
```

这意味着：

1. **nvme 驱动场景**：先分配 1 个 MSI，用完后再释放，再分配一批 MSI。
   旧代码在第二次分配时会再次调用 `msi_prepare()`。
   对 ITS 来说，第二次调用时 `its_find_device()` 发现设备已存在，只是标记 `shared=true`
   然后跳过创建——这靠巧合凑巧没崩，但语义是错误的。

2. **wire-MSI 设备场景**：每条输入线都触发一次 `msi_prepare()` 调用。
   Marc 在 cover letter 中说："for which .msi_prepare() is called for. each. input. line."

3. **对 ITS 的影响**：因为 `msi_prepare()` 调用时机不确定，传入的 `nvec` 参数是
   "semi-random" 的——可能只是当前分配的 1 个 vector，而不是设备的全部 vector 数量。
   ITS 的 `its_create_device()` 需要用 `nvec` 来决定 ITT 大小和 LPI 分配数量，
   如果只传入 1，ITT 就只有 2 个 entry（`max(2, nvecs)`），后续 vector 就装不下了。

**这就是 ugly hack 存在的原因**：

```c
// 旧代码 irq-gic-v3-its-msi-parent.c
msi_info = msi_get_domain_info(domain);
if (msi_info->hwsize > nvec)
    nvec = msi_info->hwsize;  // 强制向上取到 hwsize（MSI-X 的最大 vector 数）
```

这个 hack 的作用是：即便 `msi_prepare()` 只传入 `nvec=1`，
也强制用 `hwsize`（MSI-X 的最大 vector 数，如 2048）来创建 ITT 和分配 LPI。
这样 ITT 就足够大，后续动态新增 vector 时不会溢出。

**但这和动态分配是矛盾的**：
- 动态分配的核心思想是 **按需分配**——只分配当前需要的 vector，不一次性分配全部
- ugly hack 反其道而行——一次性预分配最大容量
- 去掉 hack 是动态分配的前提，但去掉 hack 需要先修好 `msi_prepare()` 的调用时机

### 7.2 Patch 1/5: Add .msi_teardown() callback

**改动**: `include/linux/msi.h`, `kernel/irq/msi.c`

新增 `msi_teardown()` 回调，作为 `msi_prepare()` 的逆操作：

```c
struct msi_domain_ops {
    ...
    int  (*msi_prepare)(struct irq_domain *domain, struct device *dev,
                        int nvec, msi_alloc_info_t *arg);
    void (*msi_teardown)(struct irq_domain *domain, msi_alloc_info_t *arg);  // 新增
    ...
};

struct msi_domain_info {
    ...
    msi_alloc_info_t  *alloc_data;  // 新增：关联的分配数据
    ...
};
```

同时新增默认实现 `msi_domain_ops_teardown()`（空操作）。

**为什么需要 teardown**：旧代码中 ITS 在 `its_irq_domain_free()` 里做设备清理——
释放最后一个 LPI 时拆除整个 `its_device`。这个清理时机是错误的：
- 它发生在 per-IRQ free 层面，而不是 per-device 层面
- 与 `msi_prepare()` 的 per-device 创建不对称
- 限制了"释放→重分配"的灵活性（nvme 驱动就是先释放再重分配）

**注意**：此 patch 只定义了 callback，并未调用它。"In order to avoid breaking the
ITS driver that suffers from related issues, do not call the callback just yet."

### 7.3 Patch 2/5: ITS: Implement .msi_teardown() ★关键结构变更

**改动**: `irq-gic-v3-its-msi-parent.c`, `irq-gic-v3-its.c`

#### ITS 层面的 msi_teardown 实现

```c
static void its_msi_teardown(struct irq_domain *domain, msi_alloc_info_t *info)
{
    struct its_device *its_dev = info->scratchpad[0].ptr;

    guard(mutex)(&its_dev->its->dev_alloc_lock);

    if (its_dev->shared)
        return;  // 共享设备不拆除

    // LPI 应已全部释放
    if (WARN_ON_ONCE(!bitmap_empty(its_dev->event_map.lpi_map,
                                   its_dev->event_map.nr_lpis)))
        return;

    its_lpi_free(its_dev->event_map.lpi_map,
                 its_dev->event_map.lpi_base,
                 its_dev->event_map.nr_lpis);

    its_send_mapd(its_dev, 0);  // 向 ITS 硬件发送 MAPD(0) 解除映射
    its_free_device(its_dev);   // 释放 its_device 结构
}
```

#### 从 its_irq_domain_free() 中移除设备拆除逻辑

**旧代码** (`its_irq_domain_free`) 在释放最后一个 IRQ 时：

```c
mutex_lock(&its->dev_alloc_lock);
if (!its_dev->shared &&
    bitmap_empty(its_dev->event_map.lpi_map, its_dev->event_map.nr_lpis)) {
    its_lpi_free(...);
    its_send_mapd(its_dev, 0);
    its_free_device(its_dev);
}
mutex_unlock(&its->dev_alloc_lock);
```

**新代码**：删除了这段逻辑。设备拆除现在只在 `msi_teardown()` 中发生，
时机是 domain 被移除时（而不是 per-IRQ free 时）。

**中间状态的副作用**："Nothing calls it yet, so a side effect is that
the its_dev structure will not be freed and that the DID will stay mapped.
Not a big deal, and this will be solved in following patches."
即：此时 its_dev 在 LPI 全部释放后不会被拆除——这为后续"按需重分配 LPI"
留出了空间，因为设备结构（ITT + 映射）仍在。

### 7.4 Patch 3/5: Move prepare() call to per-device allocation ★★核心修复

**改动**: `include/linux/msi.h`, `kernel/irq/msi.c`

这是整个 series 的核心——修复 `msi_prepare()` 的调用时机。

#### 新增 msi_domain_template.alloc_info 字段

```c
struct msi_domain_template {
    char            name[48];
    struct irq_chip chip;
    struct msi_domain_ops ops;
    struct msi_domain_info info;
    msi_alloc_info_t      alloc_info;  // 新增：MSI 分配数据
};
```

`alloc_info` 的生命周期与 domain template 相同——domain 创建时填充，
domain 销毁时随 template 一起消失。

#### msi_prepare() 从"每次分配"移到"domain 创建"

```c
// msi_create_device_irq_domain() — 新增部分
domain->dev = dev;
dev->msi.data->__domains[domid].domain = domain;

if (msi_domain_prepare_irqs(domain, dev, hwsize, &bundle->alloc_info)) {
    dev->msi.data->__domains[domid].domain = NULL;
    irq_domain_remove(domain);
    return false;
}
```

关键变化：
- `msi_prepare()` 现在在 **domain 创建时** 被调用一次，传入 `hwsize` 参数
- `hwsize` 是 MSI-X domain 的最大 vector 数（来自 MSI-X capability 的 Table Size 字段）
- 之前是每次 `__msi_domain_alloc_irqs()` 时调用，传入当前 `nirqs`（可能只是 1）

#### populate_alloc_info() 替换 msi_domain_prepare_irqs()

```c
static int populate_alloc_info(struct irq_domain *domain, struct device *dev,
                               unsigned int nirqs, msi_alloc_info_t *arg)
{
    struct msi_domain_info *info = domain->host_data;

    // 如果有 template alloc_info（device MSI 场景），
    // 直接复制之前在 domain 创建时准备好的数据，不再调用 msi_prepare()
    if (!info->alloc_data)
        return msi_domain_prepare_irqs(domain, dev, nirqs, arg);  // 旧路径（非 device MSI）

    *arg = *info->alloc_data;  // 直接复制已准备好的数据
    return 0;
}
```

```c
// __msi_domain_alloc_irqs() — 修改
// 旧: ret = msi_domain_prepare_irqs(domain, dev, ctrl->nirqs, &arg);
// 新: ret = populate_alloc_info(domain, dev, ctrl->nirqs, &arg);
```

**对 ITS 的意义**：现在 `its_msi_prepare()` 在 domain 创建时只被调用一次，
传入的 `nvec` 参数是 `hwsize`（MSI-X 的最大 vector 数），不再是 semi-random 的值。

### 7.5 Patch 4/5: Engage .msi_teardown() on domain removal

**改动**: `kernel/irq/msi.c` (+3)

```c
// msi_remove_device_irq_domain()
dev->msi.data->__domains[domid].domain = NULL;
info = domain->host_data;

info->ops->msi_teardown(domain, info->alloc_data);  // 新增：domain 移除时调用 teardown

if (irq_domain_is_msi_device(domain))
    fwnode = domain->fwnode;
irq_domain_remove(domain);
```

现在 `msi_teardown()` 和 `msi_prepare()` 形成对称：
- `msi_prepare()`: domain 创建时调用 → `its_create_device()` + `its_send_mapd(1)`
- `msi_teardown()`: domain 移除时调用 → `its_send_mapd(0)` + `its_free_device()`

这修复了旧代码里在 `its_irq_domain_free()` 做 device teardown 的错误时机。

### 7.6 Patch 5/5: ITS: 删除 ugly hack ★★★打通动态分配

**改动**: `irq-gic-v3-its-msi-parent.c` (-19)

删除了两处 ugly hack：

```c
// its_pci_msi_prepare() — 删除
- msi_info = msi_get_domain_info(domain);
- if (msi_info->hwsize > nvec)
-     nvec = msi_info->hwsize;

// its_pmsi_prepare() — 删除
- msi_info = msi_get_domain_info(domain);
- if (msi_info->hwsize > nvec)
-     nvec = msi_info->hwsize;
```

**为什么现在可以删除**：

Patch 3/5 修复后，`msi_prepare()` 在 domain 创建时只调用一次，
传入的 `nvec` 参数就是 `hwsize`（MSI-X 最大 vector 数）。
因此 `nvec` 本身就是正确的值，不再需要 hack 来强制向上取。

Marc 的 commit message 说的就是这个：

> "Now that .msi_prepare() gets called at the right time and not
> with semi-random parameters, remove the ugly hack that tried
> to fix up the number of allocated vectors.
> It is now correct by construction."

### 7.7 串联分析：为什么修完这些后可以动态分配

#### 问题链条

```
旧流程（无法动态分配）:

msi_prepare() 每次分配时调用，传入 nvec=1
  → ITS 创建 ITT 大小=max(2,1)=2 entries
  → 后续新增 vector 时 ITT 溢出
  → 需要 ugly hack 强制用 hwsize 创建 ITT
  → 但这意味着一次性预分配全部 LPI（浪费、非动态）
  → 动态分配不可行
```

#### 修复后链条

```
新流程（可以动态分配）:

1. domain 创建时 msi_prepare() 被调用一次，传入 nvec=hwsize
   → ITS 创建 ITT 大小=hwsize（足够容纳所有 MSI-X vector）
   → LPI bitmap 预分配 hwsize 范围
   → 不需要 ugly hack，因为 nvec 本身就是 hwsize

2. 每次 VFIO_DEVICE_SET_IRQS 动态分配单个 vector 时：
   → populate_alloc_info() 直接复制已准备好的 alloc_data
   → 不再调用 msi_prepare()（不会重新创建 device）
   → 调用 __msi_domain_alloc_irqs() 分配单个 IRQ
   → ITS 的 its_alloc_device_irq() 在预分配的 LPI bitmap 中找一个空闲位
   → 不影响已分配的其他 vector

3. domain 移除时 msi_teardown() 被调用一次：
   → ITS 拆除 its_device（释放 ITT + LPI bitmap + MAPD）
   → 完整对称的生命周期管理
```

#### 对比 VFIO 动态分配的需求

VFIO 的动态 MSI-X 分配（笔记第 2 节 patch 10/11）调用链：

```c
vfio_msi_set_vector_signal(vdev, vector, fd, msix)
    → vfio_msi_alloc_irq(vdev, vector, msix)
        → pci_irq_vector(pdev, vector)  // 查询已有 IRQ
        → pci_msix_alloc_irq_at(pdev, vector, NULL)  // 动态分配新 IRQ
            → msi_domain_alloc_irq_at(&dev->dev, MSI_DEFAULT_DOMAIN, index, ...)
                → populate_alloc_info()  // 复制已准备的 alloc_data，不再调 msi_prepare
                → __msi_domain_alloc_irqs()  // 分配单个 IRQ
```

关键：`msi_domain_alloc_irq_at()` 不调用 `msi_prepare()`，
而是通过 `populate_alloc_info()` 复制已准备好的分配数据。
这意味着 ITS 的 `its_device` 在整个设备生命周期中只创建一次，
后续的动态 vector 分配只是在已有的 ITT/LPI bitmap 中增加一个 entry。

**这就是 Gleixner 说"with this change you can enable MSI_FLAG_PCI_MSIX_ALLOC_DYN
now on GIC ITS"的原因**：

1. ✅ ITT 在 domain 创建时预分配到 hwsize，足够容纳所有 MSI-X vector
2. ✅ 动态新增 vector 时不会触发 `msi_prepare()`，不会重新创建 device
3. ✅ LPI 分配在已有的 bitmap 中按需进行，不影响其他 vector
4. ✅ domain 销毁时通过 `msi_teardown()` 完整拆除

#### 唯一遗留问题

Marc Zyngier 的疑问：ITT 预分配到 hwsize 的策略意味着即使设备只用 1 个 MSI-X vector，
ITT 也会预分配到最大容量（如 2048 entries）。这在内存上可能浪费，
但对功能性没有影响（动态分配的核心目标是避免中断丢失，而非节省 ITT 内存）。

如果要进一步优化 ITT 内存使用，需要 ITS 硬件支持在线 resize ITT
（目前不支持），或者采用某种分级扩展策略——但这超出了当前讨论范围，
且不影响 `MSI_FLAG_PCI_MSIX_ALLOC_DYN` 的启用。

### 7.8 参考链接

- Cover letter v2: https://lore.kernel.org/lkml/20250513163144.2215824-1-maz@kernel.org/
- Patch 1/5 (msi_teardown callback): https://lore.kernel.org/lkml/20250513163144.2215824-2-maz@kernel.org/
- Patch 2/5 (ITS implement msi_teardown): https://lore.kernel.org/lkml/20250513163144.2215824-3-maz@kernel.org/
- Patch 3/5 (move prepare to per-device): https://lore.kernel.org/lkml/20250513163144.2215824-4-maz@kernel.org/
- Patch 4/5 (engage msi_teardown on removal): https://lore.kernel.org/lkml/20250513163144.2215824-5-maz@kernel.org/
- Patch 5/5 (ITS remove ugly hack): https://lore.kernel.org/lkml/20250513163144.2215824-6-maz@kernel.org/

---

## 8. 回合策略：从主线适配到 OLK 内核

### 8.1 核心架构差异

| 方面 | 主线 (linux) | OLK (kernel) |
|------|-------------|-------------|
| ITS PCI MSI 层 | `irq-gic-its-msi-parent.c` (MSI parent 架构) | `irq-gic-v3-its-pci-msi.c` (旧分层架构) |
| ITS domain 创建 | `msi_create_device_irq_domain()` per-device | `pci_msi_create_irq_domain()` 全局 |
| PCI MSI domain ops | 通过 `its_init_dev_msi_info()` 动态设置 | 静态 `its_pci_msi_ops` |
| MSI irq chip | `msi_lib` 通用 chip | 自定义 `its_msi_irq_chip` |
| `MSI_FLAG_PCI_MSI_MASK_PARENT` | 有 (required) | 无 |
| VFIO PCI 代码 | 使用 MSI parent `msi_domain_alloc_irq_at()` | 使用旧的 `VFIO_DEVICE_SET_IRQS` 扩展方式 |

### 8.2 适配要点

**Patch 1/5 (msi_teardown callback)**: 改 `include/linux/msi.h` 和 `kernel/irq/msi.c`，
这两个文件架构相同，但 OLK 有 KABI 保护。需用 `#include <linux/kabi.h>` 的
`KABI_REPLACE` / `KABI_RESERVE` 机制来安全地添加 `.msi_teardown` 和 `.alloc_data`。

**Patch 2/5 (ITS msi_teardown)**: 主线改 `irq-gic-its-msi-parent.c`，
OLK 没有 this file → 需要将 teardown 逻辑适配到旧架构：
- 在 `irq-gic-v3-its-pci-msi.c` 的 `its_pci_msi_ops` 中增加 `.msi_teardown`
- 在 `irq-gic-v3-its.c` 中增加 `its_msi_teardown()` 实现
- 从 `its_irq_domain_free()` 中移除 per-IRQ 的设备拆除逻辑

**Patch 3/5 (move prepare)**: 核心改动。需要：
- 在 `msi_domain_template` 中增加 `alloc_info` 字段（KABI）
- 在 `msi_create_device_irq_domain()` 中增加 `msi_domain_prepare_irqs()` 调用
- 新增 `populate_alloc_info()` 替换 `__msi_domain_alloc_irqs()` 中的直接调用
- 在 `msi_domain_info` 中增加 `alloc_data` 指针（KABI）

**Patch 4/5 (engage teardown)**: 在 `msi_remove_device_irq_domain()` 中增加 teardown 调用。

**Patch 5/5 (remove ugly hack)**: 主线改 `irq-gic-its-msi-parent.c`，
OLK 没有 → 需要在 `irq-gic-v3-its-pci-msi.c` 的 `its_pci_msi_prepare()` 中删除相同的 hack。

**启用 MSI_FLAG_PCI_MSIX_ALLOC_DYN**: 在 `irq-gic-v3-its-pci-msi.c` 的
`its_pci_msi_domain_info.flags` 中加入 `MSI_FLAG_PCI_MSIX_ALLOC_DYN`。
同时需要确认 VFIO PCI 的 `vfio_pci_core_enable()` 中已有 `pci_msix_can_alloc_dyn()` 检查
（笔记中 patch 09/11），以及 VFIO 的 `vfio_msi_alloc_irq()` 和 NORESIZE 清除逻辑
（patch 10/11 和 11/11）已在当前内核中存在。