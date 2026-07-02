# Virtio-Net 源码深度分析笔记

> 基于 Linux Kernel 主线与 QEMU master 分支源码，逐函数逐结构体剖析 virtio-net 的控制面与数据面实现。

---

## 目录

1. [拓扑架构与初始化（Control Plane）](#一拓扑架构与初始化control-plane)
2. [数据面深度剖析：发送数据包（TX Data Path）](#二数据面深度剖析发送数据包tx-data-path)
3. [数据面深度剖析：接收数据包（RX Data Path）](#三数据面深度剖析接收数据包rx-data-path)
4. [核心数据结构与内存共享（Vring 机制）](#四核心数据结构与内存共享vring-机制)

---

## 引子：核心文件索引

| 层次 | 文件 | 角色 |
|------|------|------|
| Guest 驱动 | `drivers/net/virtio_net.c` | Linux 内核 virtio-net 网卡驱动 |
| Guest 驱动 | `include/uapi/linux/virtio_net.h` | virtio-net Feature Bits 与 config/header 结构体定义 |
| Guest 驱动 | `include/uapi/linux/virtio_ring.h` | vring 描述符/avail/used 结构体定义 |
| Guest 驱动 | `drivers/virtio/virtio_ring.c` | virtqueue 核心操作（add/get/kick/notify） |
| Guest 驱动 | `include/linux/virtio_config.h` | 配置空间读取（**virtio_cread**）与 Feature 协商辅助 |
| Host 模拟 | `hw/net/virtio-net.c` | QEMU virtio-net 设备模拟 |
| Host 模拟 | `include/hw/virtio/virtio-net.h` | QEMU 侧 **VirtIONet** 结构体定义 |
| Host 模拟 | `hw/virtio/virtio.c` | QEMU virtio 通用框架（**virtqueue_pop** / **virtqueue_push** / **virtio_notify**） |

---

## 一、拓扑架构与初始化（Control Plane）

### 1.1 Guest 端（Linux）驱动注册：**virtnet_probe**

**核心文件：** `drivers/net/virtio_net.c:6735`

当 virtio-pci 总线扫描到 virtio-net 设备后，调用 **virtnet_probe**。整个流程分为以下几个阶段：

#### 阶段一：Feature 协商与队列对数量确定

```c
// 1. 检查多队列/RSS支持
if (virtio_has_feature(vdev, VIRTIO_NET_F_MQ) || virtio_has_feature(vdev, VIRTIO_NET_F_RSS))
    max_queue_pairs = virtio_cread16(vdev, offsetof(struct virtio_net_config, max_virtqueue_pairs));

// 2. 合法性校验：队列对数必须在 [1, 0x8000] 之间，且需要 VIRTIO_NET_F_CTRL_VQ
if (max_queue_pairs < 1 || max_queue_pairs > 0x8000 || !virtio_has_feature(vdev, VIRTIO_NET_F_CTRL_VQ))
    max_queue_pairs = 1;
```

代码首先通过 **virtio_has_feature** 读取 Host 通告的 Feature Bits，检查是否支持多队列（**VIRTIO_NET_F_MQ**）或 RSS（**VIRTIO_NET_F_RSS**）。若支持，则通过 **virtio_cread16** 从 **struct virtio_net_config** 的配置空间中读取 `max_virtqueue_pairs` 字段，获取 Host 支持的最大队列对数。

#### 阶段二：分配 **net_device** 与 **struct virtnet_info**

```c
// 3. 分配网卡设备（内部包含 virtnet_info 空间）
dev = alloc_etherdev_mq(sizeof(struct virtnet_info), max_queue_pairs);

// 4. 设置 netdev_ops（核心回调函数表）
dev->netdev_ops = &virtnet_netdev;

// 5. 获取指向 virtnet_info 的指针
vi = netdev_priv(dev);
vi->dev = dev;
vi->vdev = vdev;
vdev->priv = vi;
```

**alloc_etherdev_mq** 一次性分配了两部分内存：
- **struct net_device**：Linux 网络栈的通用网卡抽象。
- **struct virtnet_info**：virtio-net 驱动私有的设备上下文，紧跟在 net_device 之后分配，通过 **netdev_priv** 获取。

#### 阶段三：逐个 Feature Bit 检查与设置

**virtnet_probe** 接着逐项检查每个 Feature Bit，并设置对应的 `dev->hw_features` / `dev->features` 位掩码：

| Feature Bit | Linux 侧映射 | 含义 |
|-------------|-------------|------|
| **VIRTIO_NET_F_CSUM** | `NETIF_F_HW_CSUM \| NETIF_F_SG` | 开启硬件校验和与 Scatter-Gather |
| **VIRTIO_NET_F_GSO** | `NETIF_F_TSO \| NETIF_F_TSO6 \| NETIF_F_TSO_ECN` | 开启 TCP Segmentation Offload |
| **VIRTIO_NET_F_HOST_TSO4** | `NETIF_F_TSO` | Host 可处理 IPv4 TSO |
| **VIRTIO_NET_F_HOST_TSO6** | `NETIF_F_TSO6` | Host 可处理 IPv6 TSO |
| **VIRTIO_NET_F_MRG_RXBUF** | `vi->mergeable_rx_bufs = true` | 开启 mergeable receive buffer |
| **VIRTIO_NET_F_CTRL_VQ** | `vi->has_cvq = true` | 存在控制 virtqueue |
| **VIRTIO_NET_F_MAC** | 读取 mac[6] 到 dev_addr | 从配置空间读取 MAC 地址 |

#### 阶段四：确定 virtio header 长度

代码根据协商的 Features 确定 **vi->hdr_len**（virtio 网络包头的长度）：

```c
if (UDP_TUNNEL_GSO)
    vi->hdr_len = sizeof(struct virtio_net_hdr_v1_hash_tunnel);
else if (has_rss_hash_report)
    vi->hdr_len = sizeof(struct virtio_net_hdr_v1_hash);
else if (MRG_RXBUF || VERSION_1)
    vi->hdr_len = sizeof(struct virtio_net_hdr_mrg_rxbuf);
else
    vi->hdr_len = sizeof(struct virtio_net_hdr);
```

这意味着 virtio header 的长度是动态的，取决于协商的 Feature —— 从最基础的 10 字节到最复杂的包含 hash/tunnel 信息的大结构体。

#### 阶段五：分配与初始化 virtqueues

```c
// 6. 核心：分配队列并调用 find_vqs
err = init_vqs(vi);
```

**init_vqs** (`drivers/net/virtio_net.c:6540`) 的调用链为：

```
init_vqs
  -> virtnet_alloc_queues    // 分配 send_queue/rx_queue 数组
  -> virtnet_find_vqs        // 调用 virtio_find_vqs 与 Host 协商创建 virtqueue
  -> virtnet_set_affinity    // 设置 virtqueue 的 CPU 亲和性
```

**virtnet_find_vqs** 内部调用 **virtio_find_vqs**，后者通过 virtio-pci 传输层与 Host 通信，为每个 TX/RX queue pair 以及 control queue 创建对应的 **struct virtqueue** 实例，并将它们关联到 **send_queue.vq** 和 **receive_queue.vq** 指针。

#### 阶段六：注册网络设备并通知 Host 就绪

```c
// 7. 注册到 Linux 网络栈
register_netdevice(dev);

// 8. 通知 Host：设备已经就绪（设置 DRIVER_OK 状态位）
virtio_device_ready(vdev);

// 9. 设置活跃的队列对数
virtnet_set_queues(vi, vi->curr_queue_pairs);
```

调用 **virtio_device_ready** 后，Host 端 QEMU 会收到 **VIRTIO_CONFIG_S_DRIVER_OK** 状态位，标志着设备初始化完成，双方可以开始数据通信。

---

### 1.2 Host 端（QEMU）设备模拟：**virtio_net_device_realize**

**核心文件：** `hw/net/virtio-net.c:3867`

在 QEMU 命令行指定 `-device virtio-net` 后，QEMU 的设备框架调用 **virtio_net_device_realize** 创建虚拟网卡。

#### 核心逻辑流程

1. **设置 Host Features**：根据配置选项（如 mtu、duplex、speed、failover）设置对应的 Host Feature Bits：

   ```c
   if (n->net_conf.mtu)
       n->host_features |= (1ULL << VIRTIO_NET_F_MTU);
   if (n->failover)
       n->host_features |= (1ULL << VIRTIO_NET_F_STANDBY);
   ```

2. **初始化 VirtIO 基础设施**：

   ```c
   virtio_net_set_config_size(n, n->host_features);
   virtio_init(vdev, VIRTIO_ID_NET, n->config_size);
   ```

3. **分配 VirtIONetQueue 数组**：

   ```c
   n->vqs = g_new0(VirtIONetQueue, n->max_queue_pairs);
   ```

4. **创建 virtqueues** (调用 **virtio_net_add_queue**，`virtio-net.c:2976`)：

   ```c
   // RX queue - 当 Host 有数据要发给 Guest 时触发
   n->vqs[index].rx_vq = virtio_add_queue(vdev, rx_queue_size,
                                           virtio_net_handle_rx);

   // TX queue - 当 Guest 发送数据时触发
   // 支持两种 TX 模式：timer 模式和 bh (bottom-half) 模式
   if (tx == "timer")
       n->vqs[index].tx_vq = virtio_add_queue(vdev, tx_queue_size,
                                               virtio_net_handle_tx_timer);
   else
       n->vqs[index].tx_vq = virtio_add_queue(vdev, tx_queue_size,
                                               virtio_net_handle_tx_bh);
   ```

   每个 TX 队列的 handler（**virtio_net_handle_tx_bh** 或 **virtio_net_handle_tx_timer**）是 Guest kick 信号进入 QEMU 后的入口点。`bh` 模式使用 QEMU 的 bottom-half 机制延迟批量处理；`timer` 模式使用定时器合并多次 kick。

5. **创建 control virtqueue**：

   ```c
   n->ctrl_vq = virtio_add_queue(vdev, 64, virtio_net_handle_ctrl);
   ```

6. **创建 NetClientState 并关联后端**：

   ```c
   n->nic = qemu_new_nic(&net_virtio_info, &n->nic_conf,
                         object_get_typename(OBJECT(dev)), dev->id,
                         &dev->mem_reentrancy_guard, n);
   ```

   **qemu_new_nic** 创建了 NICState，其中的每个 **NetClientState** 代表一个网络客户端。对于 virtio-net，每个 queue pair 有一个对应的 **NetClientState**，通过 `nc->peer` 连接到后端（如 tap 设备、vhost-net 等）。

7. **检测后端 vnet_hdr 能力**：

   ```c
   peer_test_vnet_hdr(n);
   if (peer_has_vnet_hdr(n))
       n->host_hdr_len = sizeof(struct virtio_net_hdr);
   else
       n->host_hdr_len = 0;
   ```

   如果后端（如 tap）支持 `VHDR` (virtio net header)，则 Host 可以直接传递带外信息（checksum offload、GSO 等）。否则 Host 需要剥离这些信息。

---

### 1.3 握手与 Feature 协商

Feature 协商是 virtio 设备初始化的核心环节，发生在 Guest 驱动 **virtnet_probe** 执行到 **virtio_find_vqs** 阶段。

**核心文件：** `include/linux/virtio_config.h:258`

#### 协商流程

```
Guest (Linux)                                    Host (QEMU)
    |                                                |
    |  virtio_find_vqs                               |
    |  -> 读取 device feature bits                   |
    |     (通过 PCI config space)                     |
    |  <---------------------------------------------|
    |                                                |
    |  驱动检查 Feature Bits，决定接受哪些             |
    |  写入 driver feature bits                      |
    |  -------------------------------------------->|
    |                                                |
    |                          QEMU 检查 feature_ok  |
    |  virtio_device_ready (设置 DRIVER_OK)          |
    |  -------------------------------------------->|
```

#### 关键读写函数

- **virtio_has_feature(vdev, fbit)** — 检查某个 feature bit 是否已被双方协商接受。实际调用 `__virtio_test_bit(vdev, fbit)`。
- **virtio_cread16(vdev, offset) / virtio_cread_bytes(vdev, offset, buf, len)** — 从设备配置空间（如 **struct virtio_net_config**）中读取字段。这些读写通过 virtio-pci 传输层的 BAR 空间完成。

#### 核心 Feature Bits 速查

**文件：** `include/uapi/linux/virtio_net.h:36-110`

| Feature Bit | 编号 | 含义 |
|-------------|------|------|
| **VIRTIO_NET_F_CSUM** | 0 | Guest 可处理带 checksum offload 的包 |
| **VIRTIO_NET_F_GUEST_CSUM** | 1 | Guest 可接收部分校验和的包 |
| **VIRTIO_NET_F_MAC** | 5 | 配置空间中包含 MAC 地址 |
| **VIRTIO_NET_F_GSO** | 6 | Guest 可处理 GSO 包 |
| **VIRTIO_NET_F_GUEST_TSO4** | 7 | Guest 可接收 TSOv4 包 |
| **VIRTIO_NET_F_GUEST_TSO6** | 8 | Guest 可接收 TSOv6 包 |
| **VIRTIO_NET_F_STATUS** | 16 | 配置空间中包含 link status 字段 |
| **VIRTIO_NET_F_CTRL_VQ** | 17 | 存在控制 virtqueue |
| **VIRTIO_NET_F_MQ** | 22 | 支持多队列 |
| **VIRTIO_NET_F_MTU** | 3 | 配置空间中包含 MTU 字段 |
| **VIRTIO_NET_F_MRG_RXBUF** | 15 | Host 可合并多个接收缓冲区 |
| **VIRTIO_NET_F_RSS** | 60 | 支持 RSS 散列 |
| **VIRTIO_F_VERSION_1** | 32 | 使用 virtio 1.0 规范 |

---

### 1.4 核心结构体总览

#### **struct virtnet_info**（`drivers/net/virtio_net.c:382`）

```c
struct virtnet_info {
    struct virtio_device *vdev;   // 指向 virtio 设备
    struct virtqueue *cvq;        // 控制 virtqueue
    struct net_device *dev;       // Linux 网络设备
    struct send_queue *sq;        // 发送队列数组
    struct receive_queue *rq;     // 接收队列数组
    unsigned int status;          // 链路状态 VIRTIO_NET_S_LINK_UP

    u16 max_queue_pairs;          // 最大 queue pair 数
    u16 curr_queue_pairs;         // 当前活跃 queue pair 数
    bool mergeable_rx_bufs;       // 是否启用 mergeable rx buffers
    bool big_packets;             // 是否启用 big packets

    bool has_cvq;                 // 是否有控制 VQ
    u8 hdr_len;                   // virtio net header 长度

    bool any_header_sg;           // Host 支持 header 和数据分离的 scatter-gather
    struct work_struct config_work;   // 配置变更 work
    struct control_buf *ctrl;     // 控制 VQ buffer
    // ... 更多字段
};
```

#### **struct send_queue**（`drivers/net/virtio_net.c:295`）

```c
struct send_queue {
    struct virtqueue *vq;                          // 关联的 virtqueue
    struct scatterlist sg[MAX_SKB_FRAGS + 2];      // 预分配的 scatterlist 数组
    char name[16];                                 // "output.$index"
    struct virtnet_sq_stats stats;                 // 统计计数器
    struct napi_struct napi;                       // NAPI 结构（用于 TX NAPI）
    bool reset;                                    // 重置标志
    struct xsk_buff_pool *xsk_pool;                // AF_XDP 零拷贝缓冲池
};
```

#### **struct receive_queue**（`drivers/net/virtio_net.c:320`）

```c
struct receive_queue {
    struct virtqueue *vq;                          // 关联的 virtqueue
    struct napi_struct napi;                       // NAPI 结构
    struct bpf_prog __rcu *xdp_prog;               // XDP 程序
    struct virtnet_rq_stats stats;                 // 统计计数器
    struct dim dim;                                // 动态中断调节
    struct page *pages;                            // 已回收页面链表
    struct page_pool *page_pool;                   // Page Pool（高效内存管理）
    struct scatterlist sg[MAX_SKB_FRAGS + 2];      // 预分配的 scatterlist 数组
    unsigned int min_buf_len;                      // 最小单 buffer 长度
    char name[16];                                 // "input.$index"
};
```

#### **struct VirtIONet**（QEMU 侧，`include/hw/virtio/virtio-net.h:170`）

```c
struct VirtIONet {
    VirtIODevice parent_obj;          // 继承自 VirtIODevice
    uint8_t mac[ETH_ALEN];            // MAC 地址
    uint16_t status;                  // 链路状态
    VirtIONetQueue *vqs;              // queue pair 数组
    VirtQueue *ctrl_vq;               // 控制 VQ
    NICState *nic;                    // 网卡状态（包含 NetClientState 数组）
    uint32_t mergeable_rx_bufs;       // 是否启用 mergeable rx
    uint16_t max_queue_pairs;         // 最大 queue pair 数
    uint16_t curr_queue_pairs;        // 当前活跃 queue pair 数
    int multiqueue;                   // 是否启用多队列
    VIRTIO_DECLARE_FEATURES(host_features);  // Host 支持的特性位
    // ... 更多字段
};
```

#### **struct VirtIONetQueue**（QEMU 侧，`include/hw/virtio/virtio-net.h:158`）

```c
typedef struct VirtIONetQueue {
    VirtQueue *rx_vq;            // 接收 virtqueue
    VirtQueue *tx_vq;            // 发送 virtqueue
    QEMUTimer *tx_timer;         // TX 定时器（timer 模式）
    QEMUBH *tx_bh;               // TX bottom-half（bh 模式）
    uint32_t tx_waiting;         // 是否有待刷新数据
    struct {
        VirtQueueElement *elem;  // 异步发送中的元素
    } async_tx;
    struct VirtIONet *n;         // 回指 VirtIONet
} VirtIONetQueue;
```

---

## 二、数据面深度剖析：发送数据包（TX Data Path）

### 2.1 核心调用流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Guest (Linux Kernel)                          Host (QEMU)                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  net_dev->ndo_start_xmit                                                   │
│      │                                                                      │
│      ▼                                                                      │
│  start_xmit                                    ┌─── 事件循环 ──────────┐   │
│   (virtio_net.c:3330)                          │                       │   │
│      │                                         │  io_eventfd /          │   │
│      │① free_old_xmit (回收已完成TX)            │  kvmtool kick_handler  │   │
│      │                                         │         │              │   │
│      │② xmit_skb (构建virtio header + sg)       │         ▼              │   │
│      │       │                                  │  virtio_net_handle_tx  │   │
│      │       │                                  │  _bh / _tx_timer       │   │
│      │       │                                  │   (virtio-net.c:2851)  │   │
│      │       ▼                                  │         │              │   │
│      │  virtnet_add_outbuf                      │         ▼              │   │
│      │   -> virtqueue_add_outbuf                │  virtio_net_tx_bh      │   │
│      │    -> virtqueue_add                      │   (virtio-net.c:2927)  │   │
│      │     (virtio_ring.c:2896)                 │         │              │   │
│      │                                         │         │              │   │
│      │③ virtqueue_kick (通知 Host)              │  ◄─ kick ─┘           │   │
│      │    -> virtqueue_kick_prepare              │                       │   │
│      │    -> virtqueue_notify                   │  virtio_net_flush_tx   │   │
│      │       -> vq->notify(_vq)                 │   (virtio-net.c:2718)  │   │
│      │          ─── VM Exit (WRITE to           │         │              │   │
│      │               io_eventfd) ──►            │         │              │   │
│      │                                         │ ① virtqueue_pop(vq)    │   │
│      │                                         │    从 vring 取出        │   │
│      │                                         │    VirtQueueElement     │   │
│      │                                         │         │              │   │
│      │                                         │ ② qemu_sendv_packet    │   │
│      │                                         │    _async(nc, sg, n)    │   │
│      │                                         │    -> tap_write /       │   │
│      │                                         │       vhost_net         │   │
│      │                                         │          │              │   │
│      │                                         │         ▼              │   │
│      │                                         │  virtqueue_push         │   │
│      │                                         │   + virtio_notify       │   │
│      │                                         │   (标记已用 + 注入中断)  │   │
│      │                                         └───────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Guest 端发送：从 **ndo_start_xmit** 到 VM Exit

**核心文件：** `drivers/net/virtio_net.c:3330`

#### 入口函数：**start_xmit**

Linux 协议栈通过 `dev_queue_xmit` --> `__dev_queue_xmit` --> `dev_hard_start_xmit` --> `ndo_start_xmit` 最终到达 **start_xmit**（即 **virtnet_netdev.ndo_start_xmit**）。

这个函数的核心逻辑分三步：

##### 第一步：回收已完成的 TX buffer

```c
if (!use_napi)
    free_old_xmit(sq, txq, false);   // 同步模式下直接回收
else
    virtqueue_disable_cb(sq->vq);    // NAPI 模式下禁用回调
```

**free_old_xmit** (`drivers/net/virtio_net.c:953`) 调用 **virtnet_free_old_xmit**，遍历 vring 的 Used Ring，找出 Host 已经处理完成的 buffer，释放对应的 **sk_buff**。这一步确保 virtqueue 中有空闲 slot 用于新的发送。

##### 第二步：构建 scatterlist 并添加到 virtqueue

```c
err = xmit_skb(sq, skb, !use_napi);
```

**xmit_skb** (`drivers/net/virtio_net.c:3272`) 执行以下子步骤：

1. 构建 **struct virtio_net_hdr_v1_hash_tunnel** 头部，填充 checksum offload、GSO 信息等。
2. 决定是否将 virtio header "push" 到 skb 的 headroom 中（零拷贝优化）：

   ```c
   can_push = vi->any_header_sg &&
       !((unsigned long)skb->data & (__alignof__(*hdr) - 1)) &&
       !skb_header_cloned(skb) && skb_headroom(skb) >= hdr_len;

   if (can_push) {
       __skb_push(skb, hdr_len);                         // 扩展 skb->data 包含 header
       num_sg = skb_to_sgvec(skb, sq->sg, 0, skb->len);  // 将 skb 转换为 scatterlist
       __skb_pull(skb, hdr_len);                         // 恢复 skb->data
   } else {
       sg_set_buf(sq->sg, hdr, hdr_len);                 // 第一个 sg 指向 header
       num_sg = skb_to_sgvec(skb, sq->sg + 1, 0, skb->len); // 后续 sg 指向数据
       num_sg++;
   }
   ```

3. 调用 **virtnet_add_outbuf** --> **virtqueue_add_outbuf** --> **virtqueue_add**，将 scatterlist 写入 vring 的 Descriptor Table。

**virtqueue_add** (`drivers/virtio/virtio_ring.c:2826`) 是底层实现，它会：
- 遍历每个 scatterlist entry，在 vring 中分配一个描述符（**struct vring_desc**）。
- 将 buffer 的 GPA（Guest Physical Address）写入 `desc->addr`。
- 将 buffer 长度写入 `desc->len`。
- 如果需要链接多个描述符，设置 `desc->flags |= VRING_DESC_F_NEXT` 并写入 `desc->next`。
- 最后在 Available Ring 中写入新的 head 描述符索引，更新 `avail->idx`。

##### 第三步：通知 Host（Kick）

```c
kick = use_napi ? __netdev_tx_sent_queue(txq, skb->len, xmit_more) :
                  !xmit_more || netif_xmit_stopped(txq);
if (kick) {
    if (virtqueue_kick_prepare(sq->vq) && virtqueue_notify(sq->vq)) {
        // 统计 kick 次数
    }
}
```

**virtqueue_kick** (`drivers/virtio/virtio_ring.c:3099`) 分为两阶段：

1. **virtqueue_kick_prepare** (`virtio_ring.c:3055`): 检查是否需要通知 Host（考虑 VRING_AVAIL_F_NO_INTERRUPT 等通知抑制机制）。
2. **virtqueue_notify** (`virtio_ring.c:3071`): 调用 `vq->notify(_vq)` 回调。对于 virtio-pci，这个回调会写入 PCI 的 Queue Notify 寄存器。

写入 Queue Notify 寄存器会触发：
- **传统模式**：VM Exit，KVM 捕获 MMIO/PIO 写操作。
- **现代模式（virtio 1.0+）**：通过 **io_eventfd** 直接通知 QEMU/KVM，无需 VM Exit。

QEMU 通过监听 **io_eventfd** 或 virtqueue 的 kick fd，在事件循环中收到通知。

---

### 2.3 Host 端接收与转发：从 Kick 到 Tap

**核心文件：** `hw/net/virtio-net.c:2851`

#### Kick 信号的接收

当 QEMU 收到 virtqueue kick 的通知后，根据队列注册时的 handler 调用对应的处理函数：

- **BH 模式**：调用 **virtio_net_handle_tx_bh** (`virtio-net.c:2851`)。它先检查 `q->tx_waiting` 避免重复调度，然后调用 `replay_bh_schedule_event(q->tx_bh)` 调度 **virtio_net_tx_bh** 在 QEMU 的 bottom-half 中执行。
- **Timer 模式**：调用 **virtio_net_handle_tx_timer** (`virtio-net.c:2822`)。它设置 `q->tx_waiting = 1`，然后启动/重设定时器，定时器到期后调用 **virtio_net_tx_timer**。

两种模式最终都调用 **virtio_net_flush_tx** (`virtio-net.c:2718`) 进行批量处理。

#### **virtio_net_flush_tx** 逐包处理循环

```c
for (;;) {
    // ① 从 vring 中弹出待处理的描述符链
    elem = virtqueue_pop(q->tx_vq, sizeof(VirtQueueElement));
    if (!elem) break;  // 没有更多待发送的数据

    // ② 如果需要字节序转换，处理 virtio header
    if (n->needs_vnet_hdr_swap) {
        virtio_net_hdr_swap(vdev, &vhdr);
    }

    // ③ 裁剪 guest_hdr_len 到 host_hdr_len（如果后端不需要完整 header）
    if (n->host_hdr_len != n->guest_hdr_len) {
        // 复制 host 关心的部分 header + 全部 payload
        iov_copy(sg, ..., out_sg, out_num, 0, n->host_hdr_len);
        iov_copy(sg + ..., ..., out_sg, out_num, n->guest_hdr_len, -1);
    }

    // ④ 发送到后端（tap 设备）
    ret = qemu_sendv_packet_async(qemu_get_subqueue(n->nic, queue_index),
                                  out_sg, out_num, virtio_net_tx_complete);
    if (ret == 0) {
        // 异步发送 - 后端暂时无法接收，保存 elem 等待 tx_complete 回调
        q->async_tx.elem = elem;
        return -EBUSY;
    }

    // ⑤ 同步完成 - 立即 push 回 Used Ring 并发送中断通知 Guest
drop:
    virtqueue_push(q->tx_vq, elem, 0);
    virtio_notify(vdev, q->tx_vq);
    g_free(elem);

    // ⑥ 达到 burst 限制则暂停（防止饥饿）
    if (++num_packets >= n->tx_burst) break;
}
```

**virtqueue_pop** 是 **virtqueue_add_outbuf** 的逆操作：
1. 从 `avail->ring[last_avail_idx % num]` 读取 head 描述符索引。
2. 遍历描述符链（通过 `desc->flags & VRING_DESC_F_NEXT`），构建 **VirtQueueElement**（包含 iovec 数组）。
3. 递增 `last_avail_idx`。

**qemu_sendv_packet_async** 将数据通过 **NetClientState** 发送到其后端（peer）：
- 如果 peer 是 tap 设备，数据通过 `write()` 系统调用写入 `/dev/net/tun`，进入宿主机的网络协议栈。
- 如果 peer 是 vhost-net，数据直接由内核处理，绕过 QEMU 用户态。

完成发送后，QEMU 调用 **virtqueue_push** 将元素写回 vring 的 Used Ring，然后 **virtio_notify** 向 Guest 注入中断（MSI-X vector），通知 Guest 该 buffer 已被消费（可用于回收）。

---

## 三、数据面深度剖析：接收数据包（RX Data Path）

### 3.1 核心调用流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Host (QEMU)                                  Guest (Linux Kernel)          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ① Tap fd 可读                                                              │
│      │                                                                      │
│      ▼                                                                      │
│  tap_read / net_handle_rx                                                   │
│      │                                                                      │
│      ▼                                                                      │
│  virtio_net_receive                         MSI-X 中断注入 ──►              │
│   (virtio-net.c:2672)                            │                          │
│      │                                            ▼                          │
│      │② virtqueue_pop (从 RX vring 取出空 buf)    virtnet_interrupt          │
│      │                                            -> napi_schedule           │
│      │③ iov_from_buf (将数据拷贝到 Guest 内存)     -> __napi_schedule        │
│      │        │                                                             │
│      │④ virtqueue_fill + virtqueue_flush          ┌── NAPI Poll ─────────┐  │
│      │   (将填充好的 buf push 回 Used Ring)       │                      │  │
│      │        │                                   │ virtnet_poll         │  │
│      │⑤ virtio_notify (中断注入，如 MSI-X) ──►    │  (virtio_net.c:3001) │  │
│      │                                            │      │               │  │
│      │                                            │      │① virtnet_poll │  │
│      │                                            │      │  _cleantx      │  │
│      │                                            │      │  (回收TX buf)  │  │
│      │                                            │      │               │  │
│      │                                            │      │② virtnet_     │  │
│      │                                            │      │  receive       │  │
│      │                                            │      │  -> receive_buf│  │
│      │                                            │      │  -> page_to_skb│  │
│      │                                            │      │               │  │
│      │                                            │      │③ try_fill_recv │  │
│      │                                            │      │  (补充空buf)   │  │
│      │                                            │      │       │        │  │
│      │                                            │      │       │        │  │
│      │   下次 Host 收包时会 pop 到                │      │ virtqueue_add  │  │
│      │   这些新补充的 buffer ◄─────────────────────┼──────  _inbuf        │  │
│      │                                            │      │ virtqueue_kick │  │
│      │                                            └──────────────────────┘  │
│      │                                                         │            │
│      │                                            netif_receive_skb         │
│      │                                            -> ip_rcv / netif_rx     │
│      │                                                   │                │
│      │                                            Linux 协议栈继续处理      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Host 端注入：从 Tap 可读到通知 Guest

**核心文件：** `hw/net/virtio-net.c:2672`

#### 入口函数：**virtio_net_receive**

当 QEMU 主事件循环检测到 tap 设备的 fd 可读时，调用 `tap_receive` --> `qemu_deliver_packet` --> `nc->info->receive` --> **virtio_net_receive**（如果启用了 RSC 则先经过 **virtio_net_receive_rcu**）。

```c
static ssize_t virtio_net_receive(NetClientState *nc, const uint8_t *buf, size_t size)
{
    VirtIONet *n = qemu_get_nic_opaque(nc);
    if ((n->rsc4_enabled || n->rsc6_enabled))
        return virtio_net_rsc_receive(nc, buf, size);
    else
        return virtio_net_do_receive(nc, buf, size);
}
```

实际的大头在 **virtio_net_receive_rcu** (`virtio-net.c:1904`)：

```c
// ① RSS 处理 - 选择目标队列
if (n->rss_data.enabled && n->rss_data.enabled_software_rss) {
    int index = virtio_net_process_rss(nc, buf, size, &extra_hdr);
    if (index >= 0)
        nc = qemu_get_subqueue(n->nic, index % n->curr_queue_pairs);
}

// ② 检查是否有空闲 rx buffer
if (!virtio_net_has_buffers(q, size + n->guest_hdr_len - n->host_hdr_len))
    return 0;  // 丢包

// ③ 逐个从 vring 弹出空 buffer，将数据拷贝进去
while (offset < size) {
    elem = virtqueue_pop(q->rx_vq, sizeof(VirtQueueElement));

    // 第一个sg entry写入 virtio header (含 num_buffers 等)
    if (i == 0) {
        receive_header(n, sg, elem->in_num, buf, size);
        offset = n->host_hdr_len;
    }

    // 将数据拷贝到 Guest 内存
    len = iov_from_buf(sg, elem->in_num, guest_offset,
                       buf + offset, size - offset);
    offset += len;
    elems[i] = elem;
    lens[i] = total;
    i++;
}

// ④ 批量写回 Used Ring
for (j = 0; j < i; j++) {
    virtqueue_fill(q->rx_vq, elems[j], lens[j], j);
}
virtqueue_flush(q->rx_vq, i);

// ⑤ 注入中断通知 Guest
virtio_notify(vdev, q->rx_vq);
```

**virtqueue_fill + virtqueue_flush** 的组合操作：**virtqueue_fill** 将元素信息写入 Used Ring（设置 `used->ring[used_idx].id` 和 `used->ring[used_idx].len`），然后 **virtqueue_flush** 更新 `used->idx`（加写屏障确保可见性）。

**virtio_notify** 触发 MSI-X 中断注入：对于 virtio-pci，这一般是通过设置 MSI-X 的 Pending Bit 然后注入中断到 Guest 的 LAPIC。

---

### 3.3 Guest 端收包与 NAPI

**核心文件：** `drivers/net/virtio_net.c:3001`

#### 中断处理与 NAPI 调度

Guest 收到 MSI-X 中断后，执行注册的中断处理函数（如 **virtnet_config_changed** 或 per-VQ 中断处理函数）。中断处理函数调用：

```
napi_schedule(&rq->napi)
  -> __napi_schedule
    -> ____napi_schedule
      -> 将 napi 加入 softirq 的 poll_list
      -> 触发 NET_RX_SOFTIRQ
```

在 ksoftirqd 上下文中，NET_RX_SOFTIRQ 的处理函数 `net_rx_action` 会遍历 poll_list，调用 NAPI 的 **poll** 函数 -- 即 **virtnet_poll**。

#### NAPI 轮询函数：**virtnet_poll**

```c
static int virtnet_poll(struct napi_struct *napi, int budget)
{
    struct receive_queue *rq = container_of(napi, struct receive_queue, napi);

    // ① 先处理 TX 完成（清理已发送但未回收的 skb）
    virtnet_poll_cleantx(rq, budget);

    // ② 接收数据包
    received = virtnet_receive(rq, budget, &xdp_xmit);

    // ③ XDP 重定向刷新
    if (xdp_xmit & VIRTIO_XDP_REDIR)
        xdp_do_flush();

    // ④ 如果没有更多包可收，完成 NAPI
    if (received < budget) {
        napi_complete = virtqueue_napi_complete(napi, rq->vq, received);
        if (napi_complete && rq->dim_enabled)
            virtnet_rx_dim_update(vi, rq);
    }

    // ⑤ XDP_TX 重定向发送
    if (xdp_xmit & VIRTIO_XDP_TX) {
        sq = virtnet_xdp_get_sq(vi);
        virtqueue_kick(sq->vq);
        virtnet_xdp_put_sq(vi, sq);
    }

    return received;
}
```

**virtnet_poll_cleantx** 是一种优化：在 RX NAPI 上下文中同步回收 TX buffer，避免 TX 侧单独触发中断。

#### 接收核心：**virtnet_receive** --> **receive_buf**

**virtnet_receive** (`virtio_net.c:2914`) 调用 **virtnet_receive_packets** (`virtio_net.c:2886`) 循环处理 each buffer：

```c
while (packets < budget &&
       (buf = virtqueue_get_buf(rq->vq, &len)) != NULL) {
    receive_buf(vi, rq, buf, len, ctx, xdp_xmit, stats);
    packets++;
}
```

**virtqueue_get_buf** (`virtio_ring.c:3133`) 从 vring 的 Used Ring 中取出 Host 已经填充好的 buffer：

1. 读取 `used->ring[last_used_idx % num]` 获取 buffer id 和长度。
2. 根据 id 找到对应的描述符链。
3. 更新 `last_used_idx`，返回 buffer 关联的 `data` 指针。

**receive_buf** (`virtio_net.c:2538`) 根据接收模式分发：

```c
if (vi->mergeable_rx_bufs)
    skb = receive_mergeable(dev, vi, rq, buf, ctx, len, xdp_xmit, stats);
else if (vi->big_packets)
    skb = receive_big(dev, vi, rq, buf, len, stats);
else
    skb = receive_small(dev, vi, rq, buf, ctx, len, xdp_xmit, stats);
```

三种模式的区别：
- **receive_small**：单个 buffer 包含完整帧，buffer 大小固定。
- **receive_big**：单个 buffer 包含完整帧，但可能需要使用多个页面（通过 `give_pages` / `get_a_page` 机制）。
- **receive_mergeable**：一个帧可能跨越多个 buffer（由 **VIRTIO_NET_F_MRG_RXBUF** 支持），Host 使用多个 buffer 来存储一个大帧。

**receive_mergeable** 的核心逻辑：

```c
// 第一个 buffer 包含 virtio header + 帧开头
struct virtio_net_hdr_mrg_rxbuf *hdr = buf;
int num_buf = virtio16_to_cpu(vi->vdev, hdr->num_buffers);

// 通过 page_pool 构建 skb
head_skb = page_to_skb(vi, rq, page, offset, len, truesize, headroom);

// 后续 buffer 作为 skb 的 frags 追加
while (--num_buf) {
    buf = virtnet_rq_get_buf(rq, &len, &ctx);
    curr_skb = virtnet_skb_append_frag(rq, head_skb, curr_skb,
                                       page, buf, len, truesize);
}
```

**page_to_skb** 将 page_pool 分配的页面转换为 **sk_buff**。对于 XDP 路径，**receive_mergeable_xdp** 会在构建 skb 之前先运行 XDP 程序。

---

#### 最终：将数据送入协议栈

**receive_buf** 最后调用 **virtnet_receive_done** (`virtio_net.c:2497`)：

```c
virtnet_receive_done(vi, rq, skb, flags)
  -> napi_gro_receive(&rq->napi, skb)    // 支持 GRO
  或 netif_receive_skb(skb)              // 直接送入协议栈
```

**napi_gro_receive** 会尝试将多个小包合并（Generic Receive Offload），然后通过 `netif_receive_skb` 进入 Linux 网络协议栈：

```
netif_receive_skb
  -> __netif_receive_skb / __netif_receive_skb_core
    -> ip_rcv / ip6_rcv / arp_rcv   (根据以太网帧类型分发)
      -> ... (协议栈上层处理，直到 socket)
```

---

### 3.4 关键细节：Buffer 预填充机制

Guest 需要预先在 RX virtqueue 中放置空 buffer，Host 才能将接收到的数据填入。这个预填充通过 **try_fill_recv** (`virtio_net.c:2755`) 完成：

```c
static bool try_fill_recv(struct virtnet_info *vi, struct receive_queue *rq, gfp_t gfp)
{
    do {
        if (vi->mergeable_rx_bufs)
            err = add_recvbuf_mergeable(vi, rq, gfp);
        else if (vi->big_packets)
            err = add_recvbuf_big(vi, rq, gfp);
        else
            err = add_recvbuf_small(vi, rq, gfp);
    } while (rq->vq->num_free);

    virtqueue_kick(rq->vq);  // 通知 Host 有新 buffer 可用
    return err != -ENOMEM;
}
```

**add_recvbuf_mergeable** (`virtio_net.c:2710`) 的核心：

```c
// 通过 page_pool 分配一页
buf = page_pool_alloc_va(rq->page_pool, &alloc_len, gfp);
buf += headroom;  // XDP headroom

// 提交到 virtqueue（作为 input buffer）
err = virtnet_rq_submit(rq, buf, len, ctx, gfp);
  -> sg_init_one(rq->sg, buf, len);
  -> virtqueue_add_inbuf(rq->vq, rq->sg, 1, ctx, gfp);
```

**virtqueue_add_inbuf** 与 **virtqueue_add_outbuf** 的区别在于：inbuf 是 "write-only" （从 Host 视角），即设置了 `VRING_DESC_F_WRITE` 标志，表示 Host 可以写入这些 buffer。

---

## 四、核心数据结构与内存共享（Vring 机制）

### 4.1 Split Ring vs Packed Ring 对比

**核心文件：** `include/uapi/linux/virtio_ring.h`

| 特性 | Split Ring | Packed Ring |
|------|-----------|------------|
| **Descriptor Table** | 独立区域，Guest 分配，Host 只读 | 与 Available/Used 合并为单一 Ring |
| **Available Ring** | 独立区域，由 Guest 写入，Host 读取 | 内嵌于 Descriptor Ring，通过 `flags` 中的 AVAIL/USED 位标记 |
| **Used Ring** | 独立区域，由 Host 写入，Guest 读取 | 同上，Host 置 USED 位 |
| **内存布局** | 三个独立区域（desc[], avail, used） | 单一环形数组 |
| **缓存行为** | 三次缓存行访问（desc → avail → used） | 一次缓存行访问 |
| **事件抑制** | 通过 avail→flags / used→flags | 通过 Event 描述符 |
| **兼容性** | virtio 0.9.5+ (legacy + modern) | virtio 1.1+ |
| **适用场景** | 广泛使用，兼容性好 | 高性能场景，减少缓存争用 |

#### Split Ring 内存布局

```
+-------------------+
| Descriptor Table  |  desc[0], desc[1], ..., desc[num-1]
|   (Guest 可读写,   |  每个 16 字节: { addr(64), len(32), flags(16), next(16) }
|    Host 只读)      |
+-------------------+
| Available Ring    |  flags(16) | idx(16) | ring[num](16) | used_event(16)
|   (Guest 写入)     |   ↑ Guest 在此发布新 buffer head
+-------------------+
|  ... padding ...  |  (对齐到 VRING_USED_ALIGN_SIZE)
+-------------------+
| Used Ring         |  flags(16) | idx(16) | ring[num]({id(32), len(32)}) | avail_event(16)
|   (Host 写入)      |   ↑ Host 在此标记已消费的 buffer
+-------------------+
```

#### Packed Ring 内存布局

```
+-------------------+
| Descriptor Ring   |  单一环形数组，每个 16 字节: { addr(64), len(32), id(16), flags(16) }
| (Guest 写入,       |  - flags bit 7 (AVAIL) : Guest 置位
|  Host 写入)        |  - flags bit 15 (USED)  : Host 置位
+-------------------+
| Driver Event      |  { off_wrap(16), flags(16) } — Guest 写入，通知抑制
+-------------------+
| Device Event      |  { off_wrap(16), flags(16) } — Host 写入，事件抑制
+-------------------+
```

---

### 4.2 Split Ring 核心结构体源码片段

**文件：** `include/uapi/linux/virtio_ring.h:104-163`

```c
/* 描述符：16 字节，可链式连接 */
struct vring_desc {
    __virtio64 addr;         // Buffer 的 Guest 物理地址 (GPA)
    __virtio32 len;          // Buffer 长度
    __virtio16 flags;        // VRING_DESC_F_NEXT(1) | VRING_DESC_F_WRITE(2) | VRING_DESC_F_INDIRECT(4)
    __virtio16 next;         // 链中下一个描述符的索引
};

/* Available Ring: Guest 发布待处理的 buffer */
struct vring_avail {
    __virtio16 flags;        // VRING_AVAIL_F_NO_INTERRUPT
    __virtio16 idx;          // 单调递增的可用 buffer 索引
    __virtio16 ring[];       // 可用 buffer 的 head 描述符索引数组
};

/* Used Ring 中的每个条目 */
struct vring_used_elem {
    __virtio32 id;           // 已使用的描述符链的 head 索引
    __virtio32 len;          // Host 实际写入的字节数
};

/* Used Ring: Host 发布已消费的 buffer */
struct vring_used {
    __virtio16 flags;        // VRING_USED_F_NO_NOTIFY
    __virtio16 idx;          // 单调递增的已用 buffer 索引
    vring_used_elem_t ring[];
};

/* vring: 整合三个区域的抽象 */
struct vring {
    unsigned int num;               // Queue 大小 (必须是 2 的幂)
    vring_desc_t *desc;            // Descriptor Table 指针
    vring_avail_t *avail;          // Available Ring 指针
    vring_used_t *used;            // Used Ring 指针
};
```

---

### 4.3 零拷贝（Zero-copy）安全共享内存机制

virtio 的零拷贝机制通过以下关键设计实现：

#### 1. Guest 物理地址（GPA）直接传递

当 Guest 调用 **virtqueue_add_outbuf** 时，scatterlist 中的地址是 Guest 物理地址（GPA）。QEMU/KVM 通过 EPT/NPT 页表可以访问这些物理地址：

- **vhost-net 路径**（高性能）：vhost 内核模块使用 `copy_from_user` / `get_user_pages` 等方法直接访问 Guest 内存，完全绕过 QEMU 用户态。
- **QEMU 用户态路径**：QEMU 通过 `address_space_map` / `cpu_physical_memory_map` 将 GPA 映射到 QEMU 进程的虚拟地址空间，使用 `iov_from_buf` / `iov_to_buf` 进行 DMA 操作。

#### 2. 描述符链的三态所有权

每个 **vring_desc** 通过 `flags` 字段约定所有权：

| 状态 | 所有权 | 含义 |
|------|--------|------|
| Guest 写入 avail ring 后 | **Host** | Guest 完成了 buffer 内容的填充，Host 可以消费 |
| Host 写入 used ring 后 | **Guest** | Host 完成了 buffer 的消费/填充，Guest 可以读取/回收 |

内存屏障（memory barriers）确保状态切换的可见性：
- Guest 写入 avail ring 后使用写屏障（`wmb()`），确保 Host 看到完整描述符内容后才看到 avail idx 更新。
- Host 写入 used ring 后使用写屏障，确保 Guest 看到完整数据后才看到 used idx 更新。

#### 3. DMA 映射与 IOMMU

现代 virtio 支持 IOMMU（如 Intel VT-d / AMD IOMMU）：

- Guest 侧：**virtqueue_add** 对 scatterlist 进行 DMA 映射（`dma_map_sg`），将 GPA 转换为 IOMMU 视角的 IOVA。
- Host 侧：QEMU/vhost 通过 IOMMU 页表访问 Guest 内存。

IOMMU 提供了额外的隔离和安全保障，Host 不能访问未经 Guest 授权的内存区域。

#### 4. Indirect Descriptors（间接描述符）

当 `VRING_DESC_F_INDIRECT` 标志设置时，描述符的 `addr` 指向的不是数据 buffer，而是一张嵌套的间接描述符表。这让 Guest 可以为单个 virtqueue 操作提供任意数量的 scatterlist entries，突破了 vring 大小的限制。

#### 5. 通知抑制（Event Suppression）

为了减少 VM Exit 开销，vring 支持通知抑制：

- **VRING_AVAIL_F_NO_INTERRUPT**：Guest 告诉 Host 不要在使用 buffer 后发送中断。
- **VRING_USED_F_NO_NOTIFY**：Host 告诉 Guest 不要在添加 buffer 后发送 kick。
- **VIRTIO_RING_F_EVENT_IDX**：更精细的控制——Guest 设置 `used_event_idx`，Host 只在 used idx 超过此值后才通知；Host 设置 `avail_event_idx`，Guest 只在 avail idx 超过此值后才 kick。

---

### 4.4 Chained Descriptors 工作示例

假设 Guest 要发送一个两部分的包（header + payload），且每个部分占一个物理页：

```
desc[0]:  addr=0x1000_0000, len=20,  flags=F_NEXT, next=5
desc[5]:  addr=0x2000_0000, len=1500, flags=0,       next=0

avail.ring[avail.idx % 256] = 0    (head = desc[0])
avail.idx = avail.idx + 1
```

Host 消费后：

```
used.ring[used.idx % 256] = { id=0, len=1520 }
used.idx = used.idx + 1
```

Guest 通过 `virtqueue_get_buf` 读取 used.idx 的变化，得知 desc[0] (链式包含 desc[5]) 已被处理，释放对应的 sk_buff 和 DMA 映射。

---

## 附录 A：关键函数索引

### Linux 内核侧

| 函数 | 文件:行号 | 角色 |
|------|----------|------|
| **virtnet_probe** | `drivers/net/virtio_net.c:6735` | 驱动初始化入口 |
| **init_vqs** | `drivers/net/virtio_net.c:6540` | 分配队列并调用 find_vqs |
| **virtnet_open** | `drivers/net/virtio_net.c:3180` | 网卡打开回调 |
| **start_xmit** | `drivers/net/virtio_net.c:3330` | `ndo_start_xmit` 入口 |
| **xmit_skb** | `drivers/net/virtio_net.c:3272` | 构建 virtio header + sg |
| **free_old_xmit** | `drivers/net/virtio_net.c:953` | 回收已完成的 TX buffer |
| **virtnet_poll** | `drivers/net/virtio_net.c:3001` | NAPI poll 函数 |
| **virtnet_receive** | `drivers/net/virtio_net.c:2914` | RX 数据接收 |
| **receive_buf** | `drivers/net/virtio_net.c:2538` | 分发到 small/big/mergeable |
| **receive_mergeable** | `drivers/net/virtio_net.c:2373` | mergeable buffer 处理 |
| **try_fill_recv** | `drivers/net/virtio_net.c:2755` | 预填充 RX buffer |
| **add_recvbuf_mergeable** | `drivers/net/virtio_net.c:2710` | 分配 page_pool buffer 并添加到 RX VQ |
| **virtqueue_add_outbuf** | `drivers/virtio/virtio_ring.c:2896` | 将 out buffer 加入 vring |
| **virtqueue_get_buf** | `drivers/virtio/virtio_ring.c:3133` | 从 used ring 获取完成的 buffer |
| **virtqueue_kick** | `drivers/virtio/virtio_ring.c:3099` | 通知 Host |
| **virtio_has_feature** | `include/linux/virtio_config.h:258` | 检查 Feature Bit |
| **virtio_cread16** | `include/linux/virtio_config.h:xxx` | 从配置空间读取 16 位值 |

### QEMU 侧

| 函数 | 文件:行号 | 角色 |
|------|----------|------|
| **virtio_net_device_realize** | `hw/net/virtio-net.c:3867` | 设备创建入口 |
| **virtio_net_add_queue** | `hw/net/virtio-net.c:2976` | 添加队列对 |
| **virtio_net_receive** | `hw/net/virtio-net.c:2672` | 从后端收包入口 |
| **virtio_net_receive_rcu** | `hw/net/virtio-net.c:1904` | 实际收包处理（含 RSS） |
| **virtio_net_handle_tx_bh** | `hw/net/virtio-net.c:2851` | TX kick 的 BH 入口 |
| **virtio_net_handle_tx_timer** | `hw/net/virtio-net.c:2822` | TX kick 的 timer 入口 |
| **virtio_net_flush_tx** | `hw/net/virtio-net.c:2718` | 批量刷新 TX 数据 |
| **virtio_net_tx_bh** | `hw/net/virtio-net.c:2927` | BH 模式 TX 批处理 |
| **virtio_net_tx_complete** | `hw/net/virtio-net.c:2685` | 异步发送完成回调 |
| **virtqueue_pop** | `hw/virtio/virtio.c` | 从 vring avail ring 获取元素 |
| **virtqueue_push** | `hw/virtio/virtio.c` | 将元素写回 used ring |
| **virtio_notify** | `hw/virtio/virtio.c` | 向 Guest 注入中断 |
| **virtqueue_fill** | `hw/virtio/virtio.c` | 填充单个 used ring entry |
| **virtqueue_flush** | `hw/virtio/virtio.c` | 批量提交 used ring entries |

---

## 附录 B：Feature Bit 速查表

**文件：** `include/uapi/linux/virtio_net.h:36-110`

| 宏 | 值 | 说明 |
|----|-----|------|
| VIRTIO_NET_F_CSUM | 0 | Device handles pkts w/ partial checksum |
| VIRTIO_NET_F_GUEST_CSUM | 1 | Driver handles pkts w/ partial checksum |
| VIRTIO_NET_F_CTRL_GUEST_OFFLOADS | 2 | Control channel offloads reconfig support |
| VIRTIO_NET_F_MTU | 3 | Device MTU is in config |
| VIRTIO_NET_F_MAC | 5 | Device MAC is in config |
| VIRTIO_NET_F_GUEST_TSO4 | 7 | Driver can receive TSOv4 |
| VIRTIO_NET_F_GUEST_TSO6 | 8 | Driver can receive TSOv6 |
| VIRTIO_NET_F_GUEST_ECN | 9 | Driver can receive TSO w/ ECN |
| VIRTIO_NET_F_GUEST_UFO | 10 | Driver can receive UFO |
| VIRTIO_NET_F_HOST_TSO4 | 11 | Device can receive TSOv4 |
| VIRTIO_NET_F_HOST_TSO6 | 12 | Device can receive TSOv6 |
| VIRTIO_NET_F_HOST_ECN | 13 | Device can receive TSO w/ ECN |
| VIRTIO_NET_F_HOST_UFO | 14 | Device can receive UFO |
| VIRTIO_NET_F_MRG_RXBUF | 15 | Driver can merge receive buffers |
| VIRTIO_NET_F_STATUS | 16 | Config status field available |
| VIRTIO_NET_F_CTRL_VQ | 17 | Control channel available |
| VIRTIO_NET_F_CTRL_RX | 18 | Control channel RX mode support |
| VIRTIO_NET_F_CTRL_VLAN | 19 | Control channel VLAN filtering |
| VIRTIO_NET_F_GUEST_ANNOUNCE | 21 | Driver can send gratuitous pkts |
| VIRTIO_NET_F_MQ | 22 | Device supports multiqueue |
| VIRTIO_NET_F_CTRL_MAC_ADDR | 23 | Set MAC address via control VQ |
| VIRTIO_NET_F_RSS | 60 | Device supports RSS |
| VIRTIO_NET_F_HASH_REPORT | 57 | Device can report hash value |
| VIRTIO_F_VERSION_1 | 32 | Compliant to virtio 1.0 spec |
| VIRTIO_RING_F_INDIRECT_DESC | 28 | Indirect descriptors supported |
| VIRTIO_RING_F_EVENT_IDX | 29 | Event suppression via index |

---

*笔记生成时间：2026-07-02，基于 Linux 内核主线与 QEMU master 分支源码*
