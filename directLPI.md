Direct LPI是个GICv3/GICv4的一个特性，允许直接通过GICR（GICR_INVLPIR/GICR_SETLPIR/GICR_CLRLPIR）来对LPI进行失效、置位、清空，完全绕过了ITS。（注意只针对LPI，而不保护vLPI）

LPI pending table是per-cpu的，存在内存中，基地址存在GICR_PENDBASER。另外，GICR有自己的cache，会将pending信息放到cache中。上面的失效、置位、清空操作都是针对cache的，比写内存来的更快。

---
题外话，vLPI的情况是什么样：
GICR_VPROPBASER中存了vPE configuration table的寄地址，可以从内存获取数据，但是实际GICR会从cache中存取数据。

vPE上位的时候，GICR_VPENDBASER.valid会置为1，GICR感知到后会从内存中将vPE的vLPI中断状态都读进GICR内部的缓存中。

GICv4.0中GICR_VPENDBASER直接存的vLPI pending table，GICv4.1中GICR_VPENDBASER存的是VPEID，需要从vPE configuration table中索引到vLPI pending table。vLPI pending table是per-vpe的，

---

举个使用场景：vpe迁移时，对应的doorbell中断也要迁移。

doorbell中断是lpi中断，vpe通过`its_vpe_set_affinity`来设置新的亲和性，从当前cpu迁移到新的cpu。vpe的迁移通过vmovp完成，doorbell的迁移通过`its_vpe_db_proxy_move`完成。

在`its_vpe_db_proxy_move`中有三条分支：
1. gicv4.1直接返回，gicv4.1的vmovp直接完成了doorbell中断状态的迁移，不需要软件介入。
2. gicv4.0+directLPI，直接通过写GICR_CLRLPIR清掉原先cpu上的doorbell pending状态。
3. gicv4.0无directLPI，使用vpe proxy设备发movi命令。

directLPI内容大致就这些，另外，`its_vpe_set_irqchip_state`中修改doorbell中断状态也会用到这个特效。

接下来顺便看看vpe proxy这个设备。

这个设备来源于gicv4.0的一个缺陷，gicv4.0架构上没有规范操作doorbell LPI的机制。如果设备没有实现directLPI特性，要操作doorbell LPI状态就只能通过ITS命令。但是ITS命令必须让每个LPI绑定一个设备，有对应的devid和eventid。doorbell LPI只能通过软件来模拟一个，因此在its初始化的时候（`its_init_vpe_domain`），创建了一个vpe proxy设备，设备deviceid是支持的最大设备数(2^(GITS_TYPE.DEVBITS+1))-1，一般是0xffff。

```
struct {
	raw_spinlock_t lock;
	struct its_device *dev;
	strcut its_vpe **vpes;
	int next_victim;
} vpe_proxy;
```

- lock 保护proxy设备数据的并发访问；
- dev 模拟的设备指针；
- vpes vpe指针数组，每个eventid和VPE的映射，索引是eventid；
- next_victim 下一个可分配/可抢占的eventid，用于解决eventid不够的问题；

对于一个ITT表，他的eventid数量是有限的，vpe的数量可能会大于eventid的最大值，无法保证eventid和vpe能够一一对应。vpe->vpe_proxy_event报错当前vpe占用的eventid，vpe使用前需确保占了一个eventid，如果没占，就先发discard踢掉next_victim对应的vpe，再发MAPTI把自己挂到ITT表。

看下vpe迁移时，第三条分支做了什么：
```
static void its_vpe_db_proxy_move(struct its_vpe *vpe, int from, int to)
{
······

raw_spin_lock_irqsave(&vpe_proxy.lock, flags);

// 确保vpe占了一个eventid
its_vpe_db_proxy_map_locked(vpe);

// MOVI <devid=proxy.devid><eventid=vpe_proxy_event><collection=to.col>
target_col = &vpe_proxy.dev->its->collections[to];
its_send_movi(vpe_proxy.dev, target_col, vpe->vpe_proxy_event);

// col_map是设备软件侧存的eventid和cpu的映射关系，也需要更新一下
vpe_proxy.dev->event_map.col_map[vpe->vpe_proxy_event] = to;

raw_spin_lock_irqsave(&vpe_proxy.lock, flags);
}
```

