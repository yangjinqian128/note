# Shadow Device（影子设备）研究笔记

> OLK/openEuler 内核 KVM ARM64 特性。作者：Wanghaibin (Huawei, 2019–2020)。
> 引入提交：`013f589495b7 KVM: arm64: Introduce shadow device`
> 所有代码均来自本树，行号为当前 OLK-6.6 分支；QEMU 代码在 openEuler 的
> **qemu-6.2.0 分支**（master 上没有）。

## 1. 概述（总览调用栈）

shadow device 让 QEMU **模拟设备**（virtio-net/blk/scsi）的 MSI 也能走 GICv4
direct injection：在 host 上创建虚拟 platform device，从真实 ITS 分配 host MSI
（LPI），通过 VLPImap 绑定到 guest vLPI + VPE；irqfd 触发时只对 ITS 置 pending，
硬件直投 vCPU，绕过 VM exit。

```
【创建】QEMU                              KVM                                  ITS 硬件
 guest 驱动写 MSI-X 表
   → kvm_create_shadow_device() ── ioctl(KVM_CREATE_SHADOW_DEV) ──┐
                                                                   ▼
                                      kvm_shadow_dev_create()
                                        ├─ 创建 virt_plat_dev → probe → 分配 host LPI
                                        └─ sdev_virq_bypass_active()  逐个 LPI:
                                             kvm_vgic_v4_set_forwarding()
                                               → vgic_its_resolve_lpi() (查 guest ITS 表)
                                               → its_map_vlpi()
                                                 → DISCARD + VMAPTI ──────────────► 表项→VLPImap

【注入】guest 写 virtio 门铃 → eventfd → irqfd_wakeup()
   → kvm_arch_set_irq_inatomic() → (cache 命中) shadow_dev_virq_bypass_inject()
   → irq_set_irqchip_state(host_irq, PENDING) → its_irq_set_irqchip_state()
   → its_send_vint() ──────────────────────────────────────────────────────────► ITS 查 VLPImap
                                                                                 → 直投 VPE(无 VM exit)

【删除】ioctl(KVM_DEL_SHADOW_DEV) → kvm_shadow_dev_delete()
   → workqueue: shadow_dev_destroy() → kvm_vgic_v4_unset_forwarding() → its_unmap_vlpi()
```

## 2. 使能开关

**调用栈**：模块加载 → `kvm_arm_init()`（arm.c:2985）→ `kvm_shadow_dev_init()`
（arm.c:3074）。内核参数在 early_param 阶段解析。

内核参数 `kvm-arm.virt_msi_bypass`（early_param），与 `has_gicv4` 相与得到总开关：

```c
/* arch/arm64/kvm/vgic/shadow_dev.c:307 */
static int __init early_virt_msi_bypass(char *buf)
{
	return strtobool(buf, &virt_msi_bypass);
}
early_param("kvm-arm.virt_msi_bypass", early_virt_msi_bypass);

/* arch/arm64/kvm/vgic/shadow_dev.c:313 */
void kvm_shadow_dev_init(void)
{
	/*
	 * FIXME: Ideally shadow device should only rely on a GICv4.0
	 * capable ITS, but we should also take the reserved device ID
	 * pools into account.
	 */
	sdev_enable = kvm_vgic_global_state.has_gicv4 && virt_msi_bypass;

	sdev_cleanup_wq = alloc_workqueue("kvm-sdev-cleanup", 0, 0);
	if (!sdev_cleanup_wq)
		sdev_enable = false;

	kvm_info("Shadow device %sabled\n", sdev_enable ? "en" : "dis");
}
```

```c
/* arch/arm64/kvm/arm.c:3071 */
	kvm_arm_initialised = true;

#ifdef CONFIG_VIRT_PLAT_DEV
	kvm_shadow_dev_init();
#endif
```

编译期由 `CONFIG_VIRT_PLAT_DEV` 控制（drivers/misc/Kconfig:505，
`depends on KVM && ARM64 && ARCH_HISI && ACPI_IORT`）。

## 3. 数据结构与 userspace 接口

```c
/* include/kvm/arm_vgic.h:38 */
#ifdef CONFIG_VIRT_PLAT_DEV
struct shadow_dev {
	struct kvm              *kvm;
	struct list_head        entry;

	u32                     devid;  /* guest visible device id */
	u32                     nvecs;
	unsigned long           *enable;
	int                     *host_irq;
	struct kvm_msi          *msi;

	struct platform_device  *pdev;

	struct work_struct      destroy;
};
#endif
```

链表头挂在 `vgic_dist` 里，由 `sdev_list_lock`（raw_spinlock）保护：

```c
/* include/kvm/arm_vgic.h:366 */
	raw_spinlock_t		sdev_list_lock;
	struct list_head	sdev_list_head;
```

userspace 侧（QEMU）接口：

```c
/* include/uapi/linux/kvm.h:1535 */
struct kvm_master_dev_info {
	__u32 nvectors;
	struct kvm_msi msi[];
};

/* include/uapi/linux/kvm.h:1696 */
#define KVM_CREATE_SHADOW_DEV	  _IOW(KVMIO,  0xf0, struct kvm_master_dev_info)
#define KVM_DEL_SHADOW_DEV	  _IOW(KVMIO,  0xf1, __u32)
```

`kvm_master_dev_info.msi[]` 是"主设备"（master device）的 MSI 配置模板
（guest 写进 MSI-X 表的门铃快照），shadow device 照抄这套 DevID/EventID。

## 4. 创建流程

**完整调用栈**：

```
QEMU  guest 驱动写 MSI-X 表(门铃地址/data) → 使能队列
  hw/virtio/virtio-pci.c: kvm_virtio_pci_vector_vq_use()         (qemu-6.2.0 :909)
  target/arm/kvm.c: kvm_create_shadow_device()                    (:1103)
  → ioctl(KVM_CREATE_SHADOW_DEV, mdi)
KVM   virt/kvm/kvm_main.c: kvm_vm_ioctl()
  arch/arm64/kvm/arm.c: kvm_arch_vm_ioctl() case KVM_CREATE_SHADOW_DEV  (:2174)
  arch/arm64/kvm/vgic/shadow_dev.c: kvm_shadow_dev_create()       (:155)
    ├─ sdev_virt_pdev_add()                                       (:59)
    │    → platform_device_add() → (driver core 匹配)
    │      drivers/misc/virt_plat_dev.c: virt_device_probe()      (:32)
    │        → platform_msi_domain_alloc_irqs()
    │          drivers/irqchip/irq-gic-v3-its-platform-msi.c:
    │            its_pmsi_prepare()                               (:61)
    │          → ITS 为虚拟设备分配 nvec 个 host LPI
    └─ sdev_virq_bypass_active()                                  (:100)
         └─ 逐个 host LPI 调:
            arch/arm64/kvm/vgic/vgic-v4.c: kvm_vgic_v4_set_forwarding()  (:477)
              ├─ vgic_get_its()                  (:463, 按门铃地址找 vITS)
              ├─ arch/arm64/kvm/vgic/vgic-its.c: vgic_its_resolve_lpi()  (:867)
              │    (软件查 guest ITS 表: DevID/EventID → vLPI + target vCPU)
              └─ drivers/irqchip/irq-gic-v4.c: its_map_vlpi()     (:343)
                   → kernel/irq/manage.c: irq_set_vcpu_affinity()
                   → drivers/irqchip/irq-gic-v3-its.c:
                       its_irq_set_vcpu_affinity()                (:2423)
                     → its_vlpi_map()
                       → lpi_write_config() / its_send_discard() / its_send_vmapti()
                         (ITS 硬件: 该 DevID/EventID 表项换成 VLPImap)
```

### 4.1 ioctl 入口

```c
/* arch/arm64/kvm/arm.c:2174 */
#ifdef CONFIG_VIRT_PLAT_DEV
	case KVM_CREATE_SHADOW_DEV: {
		struct kvm_master_dev_info *mdi;
		u32 nvectors;
		int ret;

		if (get_user(nvectors, (const u32 __user *)argp))
			return -EFAULT;
		if (!nvectors)
			return -EINVAL;

		mdi = memdup_user(argp, sizeof(*mdi) + nvectors * sizeof(mdi->msi[0]));
		if (IS_ERR(mdi))
			return PTR_ERR(mdi);

		ret = kvm_shadow_dev_create(kvm, mdi);
		kfree(mdi);

		return ret;
	}
```

### 4.2 kvm_shadow_dev_create

```c
/* arch/arm64/kvm/vgic/shadow_dev.c:155 */
int kvm_shadow_dev_create(struct kvm *kvm, struct kvm_master_dev_info *mdi)
{
	struct vgic_dist *dist = &kvm->arch.vgic;
	struct shadow_dev *sdev;
	struct kvm_msi *msi;
	unsigned long flags;
	int ret;

	if (WARN_ON(!sdev_enable))
		return -EINVAL;

	ret = -ENOMEM;
	sdev = kzalloc(sizeof(struct shadow_dev), GFP_KERNEL);
	if (!sdev)
		return ret;

	sdev->nvecs = mdi->nvectors;

	msi = kcalloc(sdev->nvecs, sizeof(struct kvm_msi), GFP_KERNEL);
	if (!msi)
		goto free_sdev;

	sdev->msi = msi;
	sdev_msi_entry_init(mdi, sdev);          /* 拷贝 msi[] 模板 */
	sdev->devid = sdev->msi[0].devid;        /* devid 取模板第 0 个 */

	sdev->pdev = sdev_virt_pdev_add(sdev->nvecs);   /* 创建虚拟 platform device */
	if (IS_ERR(sdev->pdev)) {
		ret = PTR_ERR(sdev->pdev);
		goto free_sdev_msi;
	}

	ret = sdev_virq_bypass_active(kvm, sdev);       /* 核心：分配 host LPI 并 forwarding */
	if (ret)
		goto delete_virtdev;

	sdev->kvm = kvm;
	INIT_WORK(&sdev->destroy, shadow_dev_destroy);

	raw_spin_lock_irqsave(&dist->sdev_list_lock, flags);
	list_add_tail(&sdev->entry, &dist->sdev_list_head);
	raw_spin_unlock_irqrestore(&dist->sdev_list_lock, flags);

	kvm_info("Create shadow device: 0x%x\n", sdev->devid);
	return ret;
	...
}
```

### 4.3 创建虚拟 platform device

```c
/* arch/arm64/kvm/vgic/shadow_dev.c:59 */
static struct platform_device *sdev_virt_pdev_add(u32 nvec)
{
	struct platform_device *virtdev;
	int ret = -ENOMEM;

	virtdev = platform_device_alloc("virt_plat_dev", PLATFORM_DEVID_AUTO);
	if (!virtdev) {
		kvm_err("Allocate virtual platform device failed\n");
		goto out;
	}

	dev_set_drvdata(&virtdev->dev, &nvec);   /* 注意：栈上局部变量的地址 */

	ret = platform_device_add(virtdev);      /* 触发驱动 probe */
	...
}
```

对应的驱动 probe（drivers/misc/virt_plat_dev.c）——这里完成 **host MSI 的真实分配**：

```c
/* drivers/misc/virt_plat_dev.c:32 */
static int virt_device_probe(struct platform_device *pdev)
{
	struct msi_desc *desc;
	unsigned int *drvdata = dev_get_drvdata(&pdev->dev);
	unsigned int nvec = *drvdata;
	struct irq_domain *vp_irqdomain = vp_get_irq_domain();
	int ret;

	if (!vp_irqdomain)
		return -ENXIO;

	virtdev_info("Allocate platform msi irqs nvecs: %d\n", nvec);
	dev_set_msi_domain(&pdev->dev, vp_irqdomain);

	ret = platform_msi_domain_alloc_irqs(&pdev->dev, nvec,
					     virt_write_msi_msg);
	if (ret) {
		pr_err("Allocate platform msi irqs failed %d\n", ret);
		goto error;
	}

	virtdev_info("Allocate platform msi irqs succeed\n");
	msi_for_each_desc(desc, &pdev->dev, MSI_DESC_ALL) {
		virtdev_info("Request irq %d\n", desc->irq);
		ret = request_irq(desc->irq, virt_irq_handle, 0,
				  "virt_dev_host", pdev);
		...
	}
	...
}
```

要点：
- `virt_write_msi_msg()` 是空函数——MSI 消息内容由 shadow device 的 msi[] 模板决定；
- `virt_irq_handle()` 只返回 `IRQ_HANDLED`——这些 host LPI 被 forwarding 后
  host 侧不会再收到中断，request_irq 只是占位防误发。

### 4.4 ITS platform-MSI domain 与 reserved devid

`vp_get_irq_domain()` 返回初始化时记录的第一个 ITS pMSI domain：

```c
/* drivers/irqchip/irq-gic-v3-its-platform-msi.c:14 */
#ifdef CONFIG_VIRT_PLAT_DEV
static struct irq_domain *vp_irq_domain;
extern bool rsv_devid_pool_cap;

struct irq_domain *vp_get_irq_domain(void)
{
	if (!vp_irq_domain)
		pr_err("virtual platform irqdomain hasn't be initialized!\n");

	return vp_irq_domain;
}
EXPORT_SYMBOL_GPL(vp_get_irq_domain);
#endif
```

```c
/* drivers/irqchip/irq-gic-v3-its-platform-msi.c:138 */
#ifdef CONFIG_VIRT_PLAT_DEV
	/* Should we take other irqdomains into account? */
	if (!vp_irq_domain)
		vp_irq_domain = pmsi_irqdomain;
#endif
```

华为 ITS 支持 reserved device ID pool（`rsv_devid_pool_cap`，irq-gic-v3-its.c:109）
时，虚拟设备没有真实 DevID，由 ITS 从保留池分配：

```c
/* drivers/irqchip/irq-gic-v3-its-platform-msi.c:70 */
#ifdef CONFIG_VIRT_PLAT_DEV
	if (rsv_devid_pool_cap && !dev->of_node && !dev->fwnode) {
		WARN_ON_ONCE(domain != vp_irq_domain);
		/*
		 * virtual platform device doesn't have a DeviceID which
		 * will be allocated with core ITS's help.
		 */
		info->scratchpad[0].ul = -1;

		goto vdev_pmsi_prepare;
	}
#endif
	...
	/* ITS specific DeviceID, as the core ITS ignores dev. */
	info->scratchpad[0].ul = dev_id;

#ifdef CONFIG_VIRT_PLAT_DEV
vdev_pmsi_prepare:
#endif
	/* Allocate at least 32 MSIs, and always as a power of 2 */
	nvec = max_t(int, 32, roundup_pow_of_two(nvec));   /* :97 */
```

### 4.5 sdev_virq_bypass_active —— 逐个 host LPI 做 forwarding

```c
/* arch/arm64/kvm/vgic/shadow_dev.c:100 */
static int sdev_virq_bypass_active(struct kvm *kvm, struct shadow_dev *sdev)
{
	struct kvm_kernel_irq_routing_entry *irq_entries;
	struct msi_desc *desc;
	u32 vec = 0;

	sdev->host_irq = kcalloc(sdev->nvecs, sizeof(int), GFP_KERNEL);
	sdev->enable   = bitmap_zalloc(sdev->nvecs, GFP_KERNEL);
	irq_entries    = kcalloc(sdev->nvecs,
				 sizeof(struct kvm_kernel_irq_routing_entry),
				 GFP_KERNEL);
	...
	sdev_set_irq_entry(sdev, irq_entries);   /* 用 msi[] 模板填 routing entry */

	msi_for_each_desc(desc, &sdev->pdev->dev, MSI_DESC_ALL) {
		if (!kvm_vgic_v4_set_forwarding(kvm, desc->irq,
						&irq_entries[vec])) {
			set_bit(vec, sdev->enable);      /* 成功才置 enable 位 */
			sdev->host_irq[vec] = desc->irq; /* 记录 host LPI 号 */
		} else {
			/*
			 * Can not use shadow device for direct injection,
			 * though not fatal...
			 */
			kvm_err("Shadow device set (%d) forwarding failed",
				desc->irq);
		}
		vec++;
	}
	...
}
```

`kvm_vgic_v4_set_forwarding()` 与 VFIO 直通共用（见第 7 节），是**绑定发生的真正位置**。

### 4.6 kvm_vgic_v4_set_forwarding —— 虚拟侧解析 + 下发 VMAPTI

```c
/* arch/arm64/kvm/vgic/vgic-v4.c:477 */
int kvm_vgic_v4_set_forwarding(struct kvm *kvm, int virq,
			       struct kvm_kernel_irq_routing_entry *irq_entry)
{
	struct vgic_its *its;
	struct vgic_irq *irq;
	struct its_vlpi_map map;
	unsigned long flags;
	int ret;

	if (!vgic_supports_direct_msis(kvm))
		return 0;

	/*
	 * Get the ITS, and escape early on error (not a valid
	 * doorbell for any of our vITSs).
	 */
	its = vgic_get_its(kvm, irq_entry);   /* 按 MSI address 找到对应 vITS */
	if (IS_ERR(its))
		return 0;

	mutex_lock(&its->its_lock);

	/* Perform the actual DevID/EventID -> LPI translation. */
	ret = vgic_its_resolve_lpi(kvm, its, irq_entry->msi.devid,
				   irq_entry->msi.data, &irq);
	if (ret)
		goto out;

	/*
	 * Emit the mapping request. If it fails, the ITS probably
	 * isn't v4 compatible, so let's silently bail out. Holding
	 * the ITS lock should ensure that nothing can modify the
	 * target vcpu.
	 */
	map = (struct its_vlpi_map) {
		.vm		= &kvm->arch.vgic.its_vm,
		.vpe		= &irq->target_vcpu->arch.vgic_cpu.vgic_v3.its_vpe,
		.vintid		= irq->intid,
		.properties	= ((irq->priority & 0xfc) |
				   (irq->enabled ? LPI_PROP_ENABLED : 0) |
				   LPI_PROP_GROUP1),
		.db_enabled	= true,
	};

	ret = its_map_vlpi(virq, &map);
	if (ret)
		goto out;

	irq->hw		= true;      /* 记账：该 vLPI 已绑定 host LPI */
	irq->host_irq	= virq;
	atomic_inc(&map.vpe->vlpi_count);

	/* Transfer pending state */
	raw_spin_lock_irqsave(&irq->irq_lock, flags);
	if (irq->pending_latch) {
		ret = irq_set_irqchip_state(irq->host_irq,
					    IRQCHIP_STATE_PENDING,
					    irq->pending_latch);
		WARN_RATELIMIT(ret, "IRQ %d", irq->host_irq);

		irq->pending_latch = false;
		vgic_queue_irq_unlock(kvm, irq, flags);
	} else {
		raw_spin_unlock_irqrestore(&irq->irq_lock, flags);
	}
	...
}
```

关键点：`map.vpe` 取的是 `irq->target_vcpu` 的 VPE（即 guest 侧该 vLPI 的投递目标
vCPU），`map.vintid` 是 guest 的 vLPI 号。**绑定 = host LPI ↔ (VPE, vINTID)**。

`vgic_its_resolve_lpi()` 在 guest 的 vITS 表里做翻译（软件模拟 ITS 查表）：

```c
/* arch/arm64/kvm/vgic/vgic-its.c:867 */
int vgic_its_resolve_lpi(struct kvm *kvm, struct vgic_its *its,
			 u32 devid, u32 eventid, struct vgic_irq **irq)
{
	struct kvm_vcpu *vcpu;
	struct its_ite *ite;

	if (!its->enabled)
		return -EBUSY;

	ite = find_ite(its, devid, eventid);   /* 查 vITS 的 ITT:DevID/EventID → ite */
	if (!ite || !its_is_collection_mapped(ite->collection))
		return E_ITS_INT_UNMAPPED_INTERRUPT;

	vcpu = kvm_get_vcpu(kvm, ite->collection->target_addr);  /* Collection → vCPU */
	if (!vcpu)
		return E_ITS_INT_UNMAPPED_INTERRUPT;

	if (!vgic_lpis_enabled(vcpu))
		return -EBUSY;

	vgic_its_cache_translation(kvm, its, devid, eventid, ite->irq);

	*irq = ite->irq;
	return 0;
}
```

### 4.7 硬件绑定：its_map_vlpi → its_vlpi_map（DISCARD + VMAPTI）

```c
/* drivers/irqchip/irq-gic-v4.c:343 */
int its_map_vlpi(int irq, struct its_vlpi_map *map)
{
	struct its_cmd_info info = {
		.cmd_type = MAP_VLPI,
		{
			.map      = map,
		},
	};
	int ret;

	/*
	 * The host will never see that interrupt firing again, so it
	 * is vital that we don't do any lazy masking.
	 */
	irq_set_status_flags(irq, IRQ_DISABLE_UNLAZY);

	ret = irq_set_vcpu_affinity(irq, &info);
	...
}
```

经 `irq_set_vcpu_affinity` 落到 ITS domain 的 `its_irq_set_vcpu_affinity`：

```c
/* drivers/irqchip/irq-gic-v3-its.c:2423 */
static int its_irq_set_vcpu_affinity(struct irq_data *d, void *vcpu_info)
{
	...
	guard(raw_spinlock)(&its_dev->event_map.vlpi_lock);

	/* Unmap request? */
	if (!info)
		return its_vlpi_unmap(d);

	switch (info->cmd_type) {
	case MAP_VLPI:
		return its_vlpi_map(d, info);
	...
}
```

```c
/* drivers/irqchip/irq-gic-v3-its.c: its_vlpi_map() */
static int its_vlpi_map(struct irq_data *d, struct its_cmd_info *info)
{
	struct its_device *its_dev = irq_data_get_irq_chip_data(d);
	u32 event = its_get_event_id(d);

	if (!info->map)
		return -EINVAL;

	if (!its_dev->event_map.vm) {
		struct its_vlpi_map *maps;

		maps = kcalloc(its_dev->event_map.nr_lpis, sizeof(*maps),
			       GFP_ATOMIC);
		if (!maps)
			return -ENOMEM;

		its_dev->event_map.vm = info->map->vm;
		its_dev->event_map.vlpi_maps = maps;
	} else if (its_dev->event_map.vm != info->map->vm) {
		return -EINVAL;    /* 同一 ITS device 上只能转发给同一个 VM */
	}

	/* Get our private copy of the mapping information */
	its_dev->event_map.vlpi_maps[event] = *info->map;

	if (irqd_is_forwarded_to_vcpu(d)) {
		/* Already mapped, move it around */
		its_send_vmovi(its_dev, event);
	} else {
		/* Ensure all the VPEs are mapped on this ITS */
		its_map_vm(its_dev->its, info->map->vm);

		/*
		 * Flag the interrupt as forwarded so that we can
		 * start poking the virtual property table.
		 */
		irqd_set_forwarded_to_vcpu(d);

		/* Write out the property to the prop table */
		lpi_write_config(d, 0xff, info->map->properties);

		/* Drop the physical mapping */
		its_send_discard(its_dev, event);   /* 删原 MAPTI */

		/* and install the virtual one */
		its_send_vmapti(its_dev, event);    /* 装 VLPImap ← 真正的绑定 */

		/* Increment the number of VLPIs */
		its_dev->event_map.nr_vlpis++;
	}

	return 0;
}
```

```c
/* drivers/irqchip/irq-gic-v3-its.c:1740 */
static void its_send_vmapti(struct its_device *dev, u32 id)
{
	struct its_vlpi_map *map = dev_event_to_vlpi_map(dev, id);
	struct its_cmd_desc desc;

	desc.its_vmapti_cmd.vpe = map->vpe;            /* VPE 号 */
	desc.its_vmapti_cmd.dev = dev;
	desc.its_vmapti_cmd.virt_id = map->vintid;     /* guest vLPI 号 */
	desc.its_vmapti_cmd.event_id = id;             /* 该 host LPI 的 EventID */
	desc.its_vmapti_cmd.db_enabled = map->db_enabled;  /* GICv4.0 用 doorbell */

	its_send_single_vcommand(dev->its, its_build_vmapti_cmd, &desc);
}
```

## 5. 注入路径（bypass）

**完整调用栈**：

```
【触发】guest 写 virtio 门铃 → irqfd 的 eventfd 被写
  virt/kvm/eventfd.c: irqfd_wakeup()            (:209, eventfd 唤醒回调, 原子上下文)
  → arch/arm64/kvm/vgic/vgic-irqfd.c: kvm_arch_set_irq_inatomic()  (:144)
      └─ (cache 命中) kvm_arch_set_irq_bypass()                   (:125)
          → arch/arm64/kvm/vgic/shadow_dev.c:
              shadow_dev_virq_bypass_inject()                     (:22)
            → kernel/irq/manage.c: irq_set_irqchip_state(host_irq, PENDING)
            → drivers/irqchip/irq-gic-v3-its.c:
                its_irq_set_irqchip_state()                       (:2209)
              (irqd_is_forwarded_to_vcpu → its_send_vint)         (:1844)
              → ITS 查 VLPImap → 硬件直投 VPE（无 VM exit）
  └─ (cache 未命中 → -EWOULDBLOCK) schedule_work(irqfd->inject)
      → irqfd_inject() → kvm_set_irq() → e->set = kvm_set_msi (vgic-irqfd.c:77)
      → vgic_its_inject_msi() 软件注入（有 VM exit, 回退路径）

【缓存建立】QEMU 配置 MSI 路由: ioctl(KVM_SET_GSI_ROUTING)
  virt/kvm/irqchip.c: kvm_set_irq_routing()
  → virt/kvm/eventfd.c: kvm_irq_routing_update()  (~:640, 遍历所有已注册 irqfd)
    → irqfd_update()                              (:279)
      → arch/arm64/kvm/vgic/vgic-irqfd.c:
          kire_arch_cached_data_update()          (:16)
        → kvm_shadow_dev_get() (shadow_dev.c:37, 按 devid/eventid 匹配 sdev)
        → cache->valid/data 写入 routing entry
  注: KVM_IRQFD ioctl 时 kvm_irqfd_assign() 也会调一次 irqfd_update()
      (eventfd.c:437), 即 irqfd 注册那一刻就建立缓存。
```

### 5.1 routing entry 缓存

```c
/* arch/arm64/kvm/vgic/vgic-irqfd.c:16 */
void kire_arch_cached_data_update(struct kvm *kvm,
			struct kvm_kernel_irq_routing_entry *e)
{
	struct vgic_dist *dist = &kvm->arch.vgic;
	struct kire_data *cache = &e->cache;
	struct shadow_dev *sdev;
	struct kvm_msi msi;

	kvm_populate_msi(e, &msi);

	raw_spin_lock(&dist->sdev_list_lock);
	sdev = kvm_shadow_dev_get(kvm, &msi);
	raw_spin_unlock(&dist->sdev_list_lock);

	cache->valid = !!sdev;
	cache->data = sdev;
}
```

```c
/* arch/arm64/kvm/vgic/shadow_dev.c:37 */
struct shadow_dev *kvm_shadow_dev_get(struct kvm *kvm, struct kvm_msi *msi)
{
	struct vgic_dist *dist = &kvm->arch.vgic;
	struct shadow_dev *sdev;

	if (!sdev_enable)
		return NULL;

	list_for_each_entry(sdev, &dist->sdev_list_head, entry) {
		if (sdev->devid != msi->devid)     /* DevID 必须匹配 */
			continue;

		if (sdev->nvecs <= msi->data ||    /* EventID 越界 */
		    !test_bit(msi->data, sdev->enable))  /* 该 vec 未成功 forwarding */
			break;

		return sdev;
	}

	return NULL;
}
```

### 5.2 irqfd 快速路径

```c
/* arch/arm64/kvm/vgic/vgic-irqfd.c:144 */
int kvm_arch_set_irq_inatomic(struct kvm_kernel_irq_routing_entry *e,
			      struct kvm *kvm, int irq_source_id, int level,
			      bool line_status)
{
	if (!level)
		return -EWOULDBLOCK;

	switch (e->type) {
	case KVM_IRQ_ROUTING_MSI: {
		struct kvm_msi msi;

		if (!vgic_has_its(kvm))
			break;

#ifdef CONFIG_VIRT_PLAT_DEV
		if (!kvm_arch_set_irq_bypass(e, kvm))
			return 0;                    /* bypass 成功，直接返回 */
#endif
		kvm_populate_msi(e, &msi);
		return vgic_its_inject_cached_translation(kvm, &msi);  /* 回退软件注入 */
	}
	...
}
```

```c
/* arch/arm64/kvm/vgic/vgic-irqfd.c:125 */
static int kvm_arch_set_irq_bypass(struct kvm_kernel_irq_routing_entry *e,
				  struct kvm *kvm)
{
	struct kire_data *cache = &e->cache;

	/*
	 * FIXME: is there any race against the irqfd_update(),
	 * where the cache data will be updated?
	 */
	if (!cache->valid)
		return -EWOULDBLOCK;

	return shadow_dev_virq_bypass_inject(kvm, e);
}
```

### 5.3 直通注入：置 pending

```c
/* arch/arm64/kvm/vgic/shadow_dev.c:22 */
int shadow_dev_virq_bypass_inject(struct kvm *kvm,
				  struct kvm_kernel_irq_routing_entry *e)
{
	struct shadow_dev *sdev = e->cache.data;
	u32 vec = e->msi.data;
	u32 host_irq = sdev->host_irq[vec];
	int ret;

	ret = irq_set_irqchip_state(host_irq, IRQCHIP_STATE_PENDING, true);
	WARN_RATELIMIT(ret, "IRQ %d", host_irq);

	return ret;
}
```

### 5.4 ITS 侧：转发中断的 pending 走 VINT 命令

```c
/* drivers/irqchip/irq-gic-v3-its.c:2209 */
static int its_irq_set_irqchip_state(struct irq_data *d,
				     enum irqchip_irq_state which,
				     bool state)
{
	struct its_device *its_dev = irq_data_get_irq_chip_data(d);
	u32 event = its_get_event_id(d);

	if (which != IRQCHIP_STATE_PENDING)
		return -EINVAL;

	if (irqd_is_forwarded_to_vcpu(d)) {
		if (state)
			its_send_vint(its_dev, event);   /* ← shadow device 走这里 */
		else
			its_send_vclear(its_dev, event);
	} else {
		if (state)
			its_send_int(its_dev, event);
		else
			its_send_clear(its_dev, event);
	}

	return 0;
}
```

```c
/* drivers/irqchip/irq-gic-v3-its.c:1844 */
static void its_send_vint(struct its_device *dev, u32 event_id)
{
	struct its_cmd_desc desc;

	/*
	 * There is no real VINT command. This is just a normal INT,
	 * with a VSYNC instead of a SYNC.
	 */
	desc.its_int_cmd.dev = dev;
	desc.its_int_cmd.event_id = event_id;

	its_send_single_vcommand(dev->its, its_build_vint_cmd, &desc);
}
```

ITS 收到 INT 后查表：该 DevID/EventID 的映射已被第 4.7 节的 VMAPTI 换成
`(VPE, vINTID)`，于是硬件直接更新该 VPE 的 vLPI pending 表——vCPU 在跑（VPE
resident）就直投，无 VM exit；不在跑则由 doorbell LPI（GICv4.0，`vpe_db_lpi`）
唤醒 host 调度 vCPU。这就是 "shadow device 用软件伪造硬件 MSI 源头" 的含义。

## 6. 删除流程

**完整调用栈**：

```
QEMU  hw/virtio/virtio-pci.c: kvm_virtio_pci_vector_vq_release() (qemu-6.2.0 :971)
      target/arm/kvm.c: kvm_delete_shadow_device()                  (:1129)
  → ioctl(KVM_DEL_SHADOW_DEV, &devid)
KVM   arm.c: kvm_arch_vm_ioctl() case KVM_DEL_SHADOW_DEV             (:2194)
  → shadow_dev.c: kvm_shadow_dev_delete()                           (:261)
    → queue_work(kvm-sdev-cleanup, &sdev->destroy) + flush_workqueue
    → shadow_dev_destroy() (work 回调)                              (:248)
      ├─ sdev_virq_bypass_deactive()                                (:215)
      │    → vgic-v4.c: kvm_vgic_v4_unset_forwarding()              (:552)
      │      → irq-gic-v4.c: its_unmap_vlpi()                       (:378)
      │        → its_irq_set_vcpu_affinity(info=NULL)
      │          → its_vlpi_unmap() (ITS 硬件: 撤销 VLPImap)
      └─ sdev_virt_pdev_delete() → platform_device_unregister()
           → virt_plat_dev.c: virt_device_remove()

【VM 销毁】kvm_arch_destroy_vm() → arm.c: kvm_arch_pre_destroy_vm()  (:2978)
  → kvm_shadow_dev_delete_all()                                     (:286)
```

```c
/* arch/arm64/kvm/vgic/shadow_dev.c:261 */
void kvm_shadow_dev_delete(struct kvm *kvm, u32 devid)
{
	struct vgic_dist *dist = &kvm->arch.vgic;
	struct shadow_dev *sdev, *tmp;
	unsigned long flags;

	if (WARN_ON(!sdev_enable))
		return;

	raw_spin_lock_irqsave(&dist->sdev_list_lock, flags);
	WARN_ON(list_empty(&dist->sdev_list_head)); /* shouldn't be invoked */

	list_for_each_entry_safe(sdev, tmp, &dist->sdev_list_head, entry) {
		if (sdev->devid != devid)
			continue;

		list_del(&sdev->entry);
		queue_work(sdev_cleanup_wq, &sdev->destroy);
		break;
	}
	raw_spin_unlock_irqrestore(&dist->sdev_list_lock, flags);

	flush_workqueue(sdev_cleanup_wq);
}
```

```c
/* arch/arm64/kvm/vgic/shadow_dev.c:248 */
static void shadow_dev_destroy(struct work_struct *work)
{
	struct shadow_dev *sdev = container_of(work, struct shadow_dev, destroy);
	struct kvm *kvm = sdev->kvm;

	sdev_virq_bypass_deactive(kvm, sdev);   /* unmap vlpi */
	sdev_virt_pdev_delete(sdev->pdev);      /* unregister platform device */

	sdev->nvecs = 0;
	kfree(sdev->msi);
	kfree(sdev);
}
```

```c
/* arch/arm64/kvm/vgic/shadow_dev.c:215 */
static void sdev_virq_bypass_deactive(struct kvm *kvm, struct shadow_dev *sdev)
{
	...
	sdev_set_irq_entry(sdev, irq_entries);

	msi_for_each_desc(desc, &sdev->pdev->dev, MSI_DESC_ALL) {
		if (!kvm_vgic_v4_unset_forwarding(kvm, desc->irq,
						  &irq_entries[vec])) {
			clear_bit(vec, sdev->enable);
			sdev->host_irq[vec] = 0;
		} else {
			kvm_err("Shadow device unset (%d) forwarding failed",
				desc->irq);
		}
		vec++;
	}

	kfree(sdev->host_irq);
	kfree(sdev->enable);
	kfree(irq_entries);

	/* FIXME: no error handling */
}
```

`kvm_vgic_v4_unset_forwarding()`（vgic-v4.c:552）做反向操作：`its_unmap_vlpi(virq)`
（发 VMAPI 置 invalid 或 DISCARD），`irq->hw = false`、`vlpi_count--`。

## 7. 与 VFIO 直通的绑定对比

**VFIO 完整调用栈**：

```
【绑定】irq_bypass 框架撮合 (token = 同一个 eventfd_ctx)
QEMU  VFIO: ioctl(VFIO_DEVICE_SET_IRQS) → vfio_msi_set_block()
        drivers/vfio/pci/vfio_pci_intrs.c:544
        → irq_bypass_register_producer({irq=host LPI, token=eventfd})  (:523)
      KVM:  ioctl(KVM_IRQFD) → kvm_irqfd_assign()
        virt/kvm/eventfd.c:455
        → irq_bypass_register_consumer(add_producer=kvm_arch_irq_bypass_add_producer)
      virt/lib/irqbypass.c: token 匹配 → 回调双方
        → arch/arm64/kvm/arm.c: kvm_arch_irq_bypass_add_producer()    (:2942)
        → kvm_vgic_v4_set_forwarding()  ← 与 shadow device 汇合，后续同 4.6/4.7

【投递】物理设备 MSI → 硬件写 GITS_TRANSLATER → ITS 查 VLPImap → 直投 VPE；
        vCPU 未 resident → doorbell LPI (GICv4.0) 唤醒 host 调度
```

```c
/* arch/arm64/kvm/arm.c:2942 */
int kvm_arch_irq_bypass_add_producer(struct irq_bypass_consumer *cons,
				      struct irq_bypass_producer *prod)
{
	struct kvm_kernel_irqfd *irqfd =
		container_of(cons, struct kvm_kernel_irqfd, consumer);

	return kvm_vgic_v4_set_forwarding(irqfd->kvm, prod->irq,
					  &irqfd->irq_entry);
}
```

对比表：

| | VFIO 直通 | shadow device |
|---|---|---|
| 入口 | `kvm_arch_irq_bypass_add_producer()`（arm.c:2942，irq_bypass 框架撮合 VFIO producer 与 irqfd consumer） | `sdev_virq_bypass_active()` 循环（shadow_dev.c:121） |
| 绑定函数 | `kvm_vgic_v4_set_forwarding()` | **同一个** `kvm_vgic_v4_set_forwarding()` |
| host LPI 来源 | 直通设备真实 MSI | 虚拟 platform device 的 pMSI |
| 中断触发 | 硬件写 GITS_TRANSLATER | 软件 `irq_set_irqchip_state(PENDING)` → `its_send_vint` |
| ITS 表项 | DISCARD + VMAPTI | 同样 DISCARD + VMAPTI |

## 8. QEMU 侧：mdi 是什么、何时传

QEMU 代码在 openEuler 的 **qemu-6.2.0 分支**（master 无此特性）。

**调用栈**：guest 驱动写 MSI-X 表（门铃地址/data）→ 使能队列 →
`kvm_virtio_pci_vector_vq_use()`（virtio-pci.c:909）→ `kvm_create_shadow_device()`
（target/arm/kvm.c:1103）→ ioctl。释放向量时 `kvm_virtio_pci_vector_vq_release()`
（:971）→ `kvm_delete_shadow_device()`（:1129）→ ioctl(KVM_DEL_SHADOW_DEV)。

### 8.1 mdi 的构造（target/arm/kvm.c:1103）

```c
int kvm_create_shadow_device(PCIDevice *dev)
{
    KVMState *s = kvm_state;
    struct kvm_master_dev_info *mdi;
    MSIMessage msg;
    uint32_t vector, nvectors = msix_nr_vectors_allocated(dev);
    uint32_t request_id;
    int ret;

    if (!kvm_vm_check_extension(s, KVM_CAP_ARM_VIRT_MSI_BYPASS) || !nvectors) {
        return 0;
    }

    mdi = g_malloc0(sizeof(uint32_t) + sizeof(struct kvm_msi) * nvectors);
    mdi->nvectors = nvectors;
    request_id = pci_requester_id(dev);

    for (vector = 0; vector < nvectors; vector++) {
        msg = msix_get_message(dev, vector);
        mdi->msi[vector].address_lo = extract64(msg.address, 0, 32);
        mdi->msi[vector].address_hi = extract64(msg.address, 32, 32);
        mdi->msi[vector].data = le32_to_cpu(msg.data);
        mdi->msi[vector].flags = KVM_MSI_VALID_DEVID;
        mdi->msi[vector].devid = request_id;
        memset(mdi->msi[vector].pad, 0, sizeof(mdi->msi[vector].pad));
    }

    ret = kvm_vm_ioctl(s, KVM_CREATE_SHADOW_DEV, mdi);
    g_free(mdi);
    return ret;
}
```

mdi 各字段：

| 字段 | 值 | 来源 | 内核侧用途 |
|---|---|---|---|
| nvectors | 已分配 MSI-X 向量数 | `msix_nr_vectors_allocated(dev)` | 分配 host LPI 个数 |
| msi[i].address_lo/hi | ITS 门铃 GPA（64 位拆分） | `msix_get_message()` —— **guest 驱动填的** | `vgic_get_its()` 定位 vITS |
| msi[i].data | MSI data（EventID） | `msix_get_message()` —— guest 填的 | `vgic_its_resolve_lpi()` 的 eventid |
| msi[i].flags | `KVM_MSI_VALID_DEVID` | 硬编码 | devid 有效标志 |
| msi[i].devid | PCI requester ID（BDF+segment） | `pci_requester_id(dev)` | vITS 表 device ID + sdev 链表键 |

要点：mdi 是 **guest 配置 MSI-X 时写出的门铃配置快照**，QEMU 原样转交。
guest 已用同样 DevID/EventID 配过自己的 ITS 表，所以 forwarding 时
`vgic_its_resolve_lpi()` 才能解析成功；且 QEMU 的 irqfd routing entry 与
mdi 同源，所以 `kvm_shadow_dev_get()` 能命中缓存。

### 8.2 触发时机（hw/virtio/virtio-pci.c，qemu-6.2.0）

```c
/* :896 —— 仅三个模拟设备支持 */
static bool shadow_device_supported(VirtIODevice *vdev)
{
    return !strcmp(vdev->name, "virtio-net") ||
           !strcmp(vdev->name, "virtio-blk") ||
           !strcmp(vdev->name, "virtio-scsi");
}

/* :909 —— guest 启用 queue 向量（irqfd 建立）时创建，失败在 :922 回滚 */
/* :971 —— guest 释放向量时删除 */
```

时序：guest 驱动配置 MSI-X 表 → 使能队列 → QEMU 建 irqfd 时创建 shadow
device。cap 门为 `KVM_CAP_ARM_VIRT_MSI_BYPASS`（799，kernel uapi kvm.h:1283，
arm.c:565 返回 `sdev_enable`）。

## 9. FIXME / 风险点

1. shadow_dev.c:315 —— 理想上只依赖 GICv4 的 ITS，但需考虑 reserved device ID pool
2. vgic-irqfd.c:130 —— irqfd_update() 与 cache 数据更新的竞争未解决
3. shadow_dev.c:245 —— `sdev_virq_bypass_deactive()` kcalloc 失败无错误处理
4. shadow_dev.c:271 —— `WARN_ON(list_empty())` 注释 "shouldn't be invoked"
5. shadow_dev.c:70 —— `dev_set_drvdata(&virtdev->dev, &nvec)` 传栈上局部变量地址
   （probe 在 add 过程中同步完成，目前安全但脆弱）
6. `its_vlpi_map()` 里 `GFP_ATOMIC` 分配 vlpi_maps，失败则静默 -ENOMEM，
   forwarding 未建立，回退软件注入

## 10. 相关文件清单

| 文件 | 内容 |
|---|---|
| arch/arm64/kvm/vgic/shadow_dev.c | 核心实现：create/delete/inject/get/init |
| arch/arm64/kvm/vgic/vgic-irqfd.c | 注入快速路径 + routing entry 缓存 |
| arch/arm64/kvm/vgic/vgic-v4.c | `kvm_vgic_v4_set/unset_forwarding`（:477/:552）、`vgic_get_its`（:463） |
| arch/arm64/kvm/vgic/vgic-its.c | `vgic_its_resolve_lpi`（:867）、vITS 表维护 |
| arch/arm64/kvm/arm.c | KVM_CREATE/DEL_SHADOW_DEV ioctl（:2174）、init、delete_all、irq_bypass 钩子（:2942）、cap 799（:565） |
| include/kvm/arm_vgic.h | `struct shadow_dev`（:39）、sdev_list、函数声明 |
| include/uapi/linux/kvm.h | `kvm_master_dev_info`（:1535）、两个 ioctl（:1696） |
| drivers/misc/virt_plat_dev.c | 虚拟 platform 设备驱动（probe 分配 host MSI） |
| drivers/irqchip/irq-gic-v3-its-platform-msi.c | `vp_get_irq_domain()`、reserved devid 处理 |
| drivers/irqchip/irq-gic-v3-its.c | `its_vlpi_map`（VMAPTI）、`its_irq_set_irqchip_state`（:2209）、`its_send_vint`（:1844） |
| drivers/irqchip/irq-gic-v4.c | `its_map_vlpi`（:343）、`its_unmap_vlpi`（:378） |
| virt/kvm/eventfd.c | `irqfd_wakeup`（:209）、`irqfd_update`（:279）、`kvm_irq_routing_update`（~:640） |
| virt/kvm/irqchip.c | `kvm_set_irq_routing`（→ `kvm_irq_routing_update`，:221） |
| virt/lib/irqbypass.c | producer/consumer 撮合 |
| (QEMU qemu-6.2.0) target/arm/kvm.c | `kvm_create/delete_shadow_device`（:1103/:1129） |
| (QEMU qemu-6.2.0) hw/virtio/virtio-pci.c | 触发点（:909/:922/:971）、白名单（:896） |
