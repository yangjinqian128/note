# Cube(CubeSandbox)性能测试笔记

> 整理自仓库内两篇基准博客与 hypervisor 文档,覆盖 CubeSandbox 全部性能测试类别、方法、工具与实测数据。
> 数据均来自 2026-06 基准报告,测试对象为 2 vCPU / 2 GiB 规格沙箱,单位除注明外均为毫秒(ms)。

---

## 1. 测试体系总览

CubeSandbox 面向 AI Agent 代码执行场景,**超快冷启动**与**高并发**是两个最核心的指标。性能测试分两大层面:

| 层面 | 内容 | 工具/入口 | 文档来源 |
|------|------|-----------|----------|
| 核心操作测试(API 层) | 冷启动、并发扩展、单机密度、快照操作 | `examples/cube-bench`(Go)、`examples/snapshot-rollback-clone/`(Python) | 2026-06-01 / 2026-06-03 两篇基准博客 |
| 底层 Hypervisor 指标 | 微VM 启动时间、块设备 IO、网络吞吐/延迟 | Cloud Hypervisor `performance-metrics`(`dev_cli.sh tests --metrics`) | `hypervisor/docs/performance_metrics.md` |

---

## 2. 核心操作性能测试(API 层)

### 2.1 测试环境(两套对照)

| 项 | 裸金属(BMI5) | 云主机 PVM(SA9.4XLARGE32) |
|----|--------------|---------------------------|
| 机型 | 腾讯云内存型裸金属 BMI5 | 腾讯云标准 CVM SA9.4XLARGE32 |
| 内核 | 6.6.119(TencentOS Server 4) | 6.6.69-opencloudos9.cubesandbox.pvm.host |
| CPU | Intel Xeon Platinum 8255C,2 路×24 核×2 线程 = **96 逻辑核**,2 NUMA | AMD EPYC 9K65,1 路×16 核 = **16 逻辑核**,1 NUMA |
| 内存 | 375 GiB DDR4 ECC | 32 GiB |
| 磁盘 | 3.84 TB NVMe(XFS,/data) | 200 GiB 增强型 SSD(XFS) |
| 沙箱规格 | 2 vCPU / 2 GiB | 2 vCPU / 2 GiB |
| 存储 | CoW reflink(XFS) | CoW reflink(XFS) |
| 内存跟踪 | soft-dirty(`/proc/PID/clear_refs`) | soft-dirty |

模板构建命令(两环境一致):

```bash
cubemastercli tpl create-from-image \
  --image cube-sandbox-cn.tencentcloudcr.com/cube-sandbox/sandbox-code:latest \
  --writable-layer-size 1G --expose-port 49999 --expose-port 49983 --probe 49999
```

### 2.2 通用指标定义与方法论

| 指标 | 含义 |
|------|------|
| avg | 所有轮次的均值 |
| min / max | 观测到的最小 / 最大值 |
| p95 | 95 分位(95% 请求在此时间内完成);PVM 报告额外有 P50/P90/P99 |
| wall | 整批端到端耗时(首个请求发出 → 最后一个完成),用于并发场景 |
| per | 摊薄每操作耗时(wall ÷ 批内操作数),用于并发场景 |

方法论约定:

- 每个场景前跑 **warm-up 轮次**,结果丢弃,消除 page cache 冷读噪声
- 并发测试轮次**串行执行**,避免轮间互相干扰
- 各并发 tier 独立测试,之间清理全部沙箱并让资源池恢复
- 所有测试 100% 成功率
- 所有数据高度依赖硬件与工作负载,需按自身环境评估

---

### 2.3 沙箱冷启动(Template 创建)

**测什么**:调用 `POST /sandboxes`(带 `template_id`)到沙箱达到 `running` 的端到端时间——最常见的用法。

**工具**:[`examples/cube-bench`](https://github.com/TencentCloud/CubeSandbox/tree/master/examples/cube-bench)(Go)——用 goroutine 驱动 CubeAPI,输出完整百分位统计。

```bash
cd examples/cube-bench && make   # 需要 Go 1.21+, 产出 ./bin/cube-bench

export E2B_API_URL=http://<server-ip>:3000
export E2B_API_KEY=e2b_000000
export CUBE_TEMPLATE_ID=<template-id>

./bin/cube-bench -c 1 -n 20 -w 3 -m create-only   # 1 并发,20 个,-w 3 = 3 轮热身丢弃
./bin/cube-bench -c 50 -n 500 -w 3 -m create-only -o report_c50.json
```

**裸金属结果**:

| 并发 | 请求数 | avg | min | p95 | max | 摊薄/沙箱 | 吞吐 |
|:--:|:--:|--:|--:|--:|--:|--:|--:|
| 1 | 20 | 47.8 | 43.5 | 57.4 | 60.4 | 55.8 ms | 17.9 /s |
| 10 | 200 | 88.7 | 45.8 | 116.9 | 119.1 | 9.9 ms | 101.4 /s |
| 20 | 300 | 98.1 | 47.7 | 175.8 | 232.6 | 5.5 ms | **180.9 /s** |
| 50 | 500 | 276.1 | 60.6 | 508.4 | 681.3 | 6.8 ms | 147.6 /s |

**PVM 结果**:

| 并发 | 请求数 | avg | min | P50 | P90 | P95 | P99 | max |
|:--:|:--:|--:|--:|--:|--:|--:|--:|--:|
| 1 | 20 | 66.7 | 55.9 | 64.5 | 77.5 | 78.2 | 80.2 | 80.2 |
| 10 | 200 | 170.9 | 85.4 | 168.5 | 206.4 | 216.7 | 286.1 | 323.5 |
| 20 | 300 | 364.6 | 116.5 | 356.2 | 459.0 | 521.4 | 673.8 | 744.0 |

**结论**:

- 串行创建 ~48ms(裸金属)/ ~67ms(PVM),稳定低于 100ms
- 并发带来显著吞吐收益:裸金属 20 并发吞吐 180.9/s,摊薄降到 5.5ms——是该机器的**延迟-吞吐甜点**
- 50 并发时队列深度推高单请求延迟(p95 508ms),但吞吐仍有 147.6/s

---

### 2.4 单机部署密度(内存开销)

**测什么**:利用内核共享 + CoW 压缩单实例开销,方法为"清空机器 → 分批创建 → 记录内存变化":

```
每 VM 摊薄开销 = (当前 used - 基线 used) ÷ VM 数量
```

**方法**:基线 `free -h` → `cube-bench -m create-only` 分批创建并保持存活(100→300→500→1000)→ 每批后 `free -h` 记录。

> ⚠️ 每批前必须 `free -h` 确认余量、小批推进,防止 OOM Killer 破坏环境。

**裸金属结果**(0→1000 沙箱):

| 存活沙箱 | 系统可用内存 | 每 VM 摊薄开销 |
|:--:|--:|--:|
| 0(基线) | 359.5 GiB | — |
| 100 | 357.4 GiB | ~21.5 MB |
| 300 | 352.5 GiB | ~23.8 MB |
| 500 | 347.3 GiB | ~25.0 MB |
| 1000 | 334.3 GiB | ~25.7 MB |

**PVM 结果**(0→20 沙箱):摊薄开销稳定在 **~27–34 MB**。

**结论**:

- 2 GiB 沙箱空闲时不预分配全部内存(CoW 按需分配),每实例仅几十 MB 开销,裸金属 1000 个沙箱只消耗 ~25 GiB
- 容量估算:
  - 满载场景(每沙箱写满 2 GiB):375 GiB ÷ (2 GiB + 25MB) ≈ **185 个**
  - 空闲/轻载场景:由 ~25MB 摊薄开销主导,可上**数千**,实测 1000 个稳定

**注意**:`tap_init_num`(cubelet 配置,默认 500)实际由 **network-agent** 启动时消费,用于预创建 TAP 设备;要压 1000 个沙箱需先调大该值并 `systemctl restart cube-sandbox-network-agent.service`(重启 network-agent 而非 cubelet)。

---

### 2.5 快照操作

快照是核心特性:对运行中沙箱做**内存 + 文件系统**快照,可近乎即时恢复(Clone / Rollback)。前置环境变量:

```bash
export CUBE_API_URL=http://<server-ip>:3000
export CUBE_TEMPLATE_ID=<template-id>
export CUBE_PROXY_NODE_IP=<cubeproxy-ip>   # 本机跑用 127.0.0.1
export CUBE_PROXY_PORT_HTTP=80
```

#### 2.5.1 快照创建 vs 并发(`bench_snapshot_concurrency.py`)

对 N 个独立沙箱各发一个快照请求,测全部完成的 wall 时间(单沙箱内部会串行化快照请求,故并发测的是不同沙箱)。

**裸金属**(基准脏页 ~7MB):

| 并发 | 轮次 | wall avg | wall min | wall p95 | wall max | 每快照摊薄 |
|:--:|:--:|--:|--:|--:|--:|--:|
| 1 | 5 | 49.8 | 47.3 | 54.1 | 54.1 | 49.8 ms |
| 5 | 5 | 71.0 | 62.7 | 81.0 | 81.0 | 14.2 ms |
| 10 | 5 | 127.2 | 79.6 | 155.6 | 155.6 | 12.7 ms |

**PVM**(基准脏页 ~8MB):串行 41.4ms;5 并发摊薄 11.6ms;10 并发摊薄 11.4ms(但 p95 有 285ms 尾延迟)。

#### 2.5.2 快照创建 vs 脏页大小(`bench_snapshot_dirty.py`)

**原理**:soft-dirty 机制只保存上次快照后修改过的内存页;实际写入量 = 脏页数 × 4KiB,通常远小于沙箱总内存。测试通过预写 `/dev/shm`(tmpfs)精确控制脏页量,实际脏页从 `/data/log/CubeVmm/vmm.log` 的 `PagemapAnon snapshot saved` 读取。

```bash
python bench_snapshot_dirty.py -d 0    -n 3   # -d = 写入量 MB
python bench_snapshot_dirty.py -d 1024 -n 3 --no-header
# 串行模式;每个数据点先丢 1 次热身,再取 3 次实测平均
```

**裸金属结果**:

| 写入量 | 实际脏页 | 快照 avg | 快照 p95 | 快照 max | 从快照创建 avg |
|--:|--:|--:|--:|--:|--:|
| 0 MB | 7.1 MB | 45.7 | 47.4 | 47.4 | 64.8 |
| 10 MB | 38.9 MB | 75.7 | 79.2 | 79.2 | 60.7 |
| 50 MB | 120.7 MB | 107.7 | 112.3 | 112.3 | 64.4 |
| 100 MB | 195.0 MB | 138.6 | 139.9 | 139.9 | 66.5 |
| 200 MB | 296.7 MB | 174.2 | 176.2 | 176.2 | 63.7 |
| 500 MB | 602.5 MB | 289.4 | 293.1 | 293.1 | 64.0 |
| 800 MB | 908.4 MB | 392.8 | 394.1 | 394.1 | 60.9 |
| 1024 MB | 1136.4 MB | 486.9 | 510.8 | 510.8 | 68.4 |

**PVM 结果**:同样近线性——基线(8.3MB 脏页)42.1ms,1024MB 写入(1136MB 脏页)257.5ms,斜率约 +22ms/100MB(裸金属约 +40ms/100MB,差异来自存储性能)。

**结论**:

- **快照耗时与脏页量近线性**:基线 ~47ms,每 +100MB 脏页约 +40ms,1GB 脏页 ~487ms
- **从快照恢复与脏页量无关**:稳定 60–85ms——恢复走 CoW 按需加载,不依赖快照大小

#### 2.5.3 从快照创建沙箱(`bench_create_concurrency.py`)

先打一个快照,再并发 `POST /sandboxes`(带 `snapshot_id`),测全部达到 `running` 的 wall 时间。

**裸金属**:

| 并发 | 总数 | wall avg | wall p95 | wall max | 每沙箱摊薄 |
|:--:|:--:|--:|--:|--:|--:|
| 1 | 1 | 63.9 | 66.1 | 66.1 | 63.9 ms |
| 10 | 10 | 89.9 | 93.6 | 93.6 | 9.0 ms |
| 20 | 20 | 118.9 | 167.1 | 167.1 | 5.9 ms |
| 50 | 50 | 180.3 | 260.7 | 260.7 | **3.6 ms** |

**PVM**:串行 66.7ms;10 并发摊薄 38.8ms;20 并发摊薄 35.1ms(小机器受存储带宽限制,扩展性弱于裸金属)。

#### 2.5.4 Rollback 回滚(`bench_rollback_concurrency.py`)

对运行中沙箱调用 `POST /sandboxes/{id}/rollback`,**原地**恢复内存+文件系统到指定快照,无需重建沙箱。

> 约束:沙箱只能回滚到**自己创建**的快照,故每个并发沙箱独立完成"create_snapshot → rollback"全流程,快照不能复用。

**裸金属**:

| 并发 | 轮次 | wall avg | wall p95 | wall max | 每次回滚摊薄 |
|:--:|:--:|--:|--:|--:|--:|
| 1 | 5 | 81.6 | 97.4 | 97.4 | 81.6 ms |
| 5 | 5 | 189.6 | 243.2 | 243.2 | 37.9 ms |
| 10 | 5 | 266.1 | 305.1 | 305.1 | 26.6 ms |

**PVM**:串行 90.0ms;5 并发摊薄 65.1ms;10 并发摊薄 82.1ms(10 并发摊薄反而回升,小机器 IO 竞争)。

#### 2.5.5 Clone 克隆(`bench_clone_concurrency.py`)

从**运行中**源沙箱 fork N 个新沙箱,完整保留源的内存与文件系统状态(含脏页)。测试时磁盘文件已在 Page Cache 中,结果不含冷读 IO 开销。

**裸金属**:

| 场景 | n | 并发 | wall avg | wall p95 | wall max | 每次克隆摊薄 |
|---|---|:--:|--:|--:|--:|--:|
| 1 沙箱 | 1 | 1 | 219.6 | 234.7 | 234.7 | 219.6 ms |
| 100 沙箱 | 100 | 10 | 870.4 | 880.2 | 880.2 | 8.7 ms |
| 100 沙箱 | 100 | 20 | 638.6 | 656.3 | 656.3 | 6.4 ms |
| 100 沙箱 | 100 | 50 | 540.9 | 590.5 | 590.5 | **5.4 ms** |

**PVM**(源沙箱脏页 ~10MB):单克隆 270.6ms;10 个 @5 并发摊薄 54.2ms;20 个 @10 并发摊薄 39.5ms。

**反直觉发现**:n 固定为 100 时,10 并发 wall(870ms)**慢于** 20/50 并发(639/541ms)——低并发需要串行 10 批、累积调度开销;高并发批次少,且源沙箱内存页在 Page Cache 中复用更充分。该规模下 Clone 不受并发瓶颈限制,加大并发有利。

#### 2.5.6 Pause / Resume(`bench_pause_resume_concurrency.py`)

并发创建 N 个沙箱 → 全部 `POST /sandboxes/{id}/pause` → 全部 `POST /sandboxes/{id}/resume`,分别记录 wall 与摊薄延迟。

> ⚠️ **当前实现为全量内存拷贝模式**:pause 时把沙箱全部匿名内存页写入持久化存储,延迟随内存大小线性增长(裸金属 2 GiB 约 558ms/个)。未来将升级 **soft-dirty 增量模式**(只写上次检查点后脏的页),空闲沙箱预计降 80–90%,与快照创建(~60ms)持平。

**裸金属 Pause**:

| 并发 | wall avg | wall p95 | wall max | 每次摊薄 |
|:--:|--:|--:|--:|--:|
| 1 | 558.4 | 590.3 | 590.3 | 558.4 ms |
| 5 | 656.9 | 683.2 | 683.2 | 131.4 ms |
| 10 | 682.1 | 699.3 | 699.3 | **68.2 ms** |

**裸金属 Resume**:

| 并发 | wall avg | wall min | wall p95 | 每次摊薄 |
|:--:|--:|--:|--:|--:|
| 1 | 41.8 | 18.7 | 65.1 | 41.8 ms |
| 5 | 28.2 | 17.6 | 34.2 | 5.6 ms |
| 10 | 35.7 | 30.6 | 41.7 | **3.6 ms** |

**PVM**:Pause 串行 370.8ms、10 并发摊薄 158.6ms;Resume 串行 18.9ms、10 并发摊薄 2.7ms。

**结论**:

- **Resume 极快且并发扩展好**:单次 ~42ms(裸金属)/ ~19ms(PVM),10 并发摊薄 3.6 / 2.7ms
- **Pause 是当前瓶颈**:全量拷贝模式下单次 558 / 371ms;并发下 wall 增幅温和(裸金属 NVMe 并行 IO 好),摊薄 68 / 159ms
- soft-dirty 增量模式落地后,Pause 预计降到 ~60ms,10 并发摊薄进入个位数毫秒

---

## 3. 底层 Hypervisor 指标测试

`hypervisor/docs/performance_metrics.md` 描述使用 Cloud Hypervisor 的 `performance-metrics` 工具,在自己的环境生成指标数据(需 Docker):

```bash
# 全部测试(boot time、block I/O 吞吐、network 吞吐与延迟),输出 JSON
./scripts/dev_cli.sh tests --metrics -- -- --report-file /tmp/metrics.json

# 列出可用测试
./scripts/dev_cli.sh tests --metrics -- -- --list-tests

# 只测启动时间
./scripts/dev_cli.sh tests --metrics -- -- --report-file /tmp/metrics.json --test-filter boot_time
```

覆盖三类指标:

- **boot time** — 微VM 启动时间
- **block I/O throughput** — 块设备吞吐
- **network throughput & latency** — 网络吞吐与延迟

---

## 4. 关键结论速查

| 操作 | 串行(裸金属) | 并发摊薄(裸金属) | 特点 |
|------|--:|--:|------|
| 模板创建沙箱 | ~48 ms | 5.5 ms @20 并发(吞吐 180.9/s) | 20 并发为甜点 |
| 单机内存开销 | — | ~25 MB/实例 | CoW 按需分配,1000 个仅 ~25 GiB |
| 快照创建(空闲) | ~50 ms | ~13 ms @10 并发 | 随脏页近线性:+~40ms/100MB |
| 从快照创建 | ~64 ms | 3.6 ms @50 并发 | 与快照大小无关(CoW) |
| Rollback | ~82 ms | 26.6 ms @10 并发 | 仅能回滚到自己的快照 |
| Clone | ~220 ms | 5.4 ms @50 并发 | 低并发反而更慢(批次调度) |
| Pause | ~558 ms | 68.2 ms @10 并发 | 全量拷贝,待 soft-dirty 优化至 ~60ms |
| Resume | ~42 ms | 3.6 ms @10 并发 | 极快 |

---

## 5. 工具与脚本索引

| 工具/脚本 | 语言 | 位置 | 用途 |
|-----------|------|------|------|
| cube-bench | Go | `examples/cube-bench/` | 模板并发创建压测(百分位统计、JSON 报告) |
| bench_snapshot_concurrency.py | Python | `examples/snapshot-rollback-clone/` | 快照创建 vs 并发 |
| bench_snapshot_dirty.py | Python | 同上 | 快照创建 vs 脏页大小 |
| bench_create_concurrency.py | Python | 同上 | 从快照并发创建 |
| bench_rollback_concurrency.py | Python | 同上 | Rollback vs 并发 |
| bench_clone_concurrency.py | Python | 同上 | Clone vs 并发 |
| bench_pause_resume_concurrency.py | Python | 同上 | Pause/Resume vs 并发 |
| performance-metrics | Rust | `hypervisor/scripts/dev_cli.sh tests --metrics` | 微VM boot/IO/网络指标 |

---

## 6. 参考文档

- 裸金属基准报告:`docs/blog/posts/2026-06-01-cubesandbox-perf-benchmark.md`(BMI5,96 核/375GiB)
- 云主机 PVM 基准报告:`docs/blog/posts/2026-06-03-cubesandbox-perf-benchmark-pvm.md`(SA9.4XLARGE32,16 核/32GiB)
- Hypervisor 指标:`hypervisor/docs/performance_metrics.md`
- 博客在线版:https://cubesandbox.com/blog/posts/2026-06-01-cubesandbox-perf-benchmark
