# CubeSandbox 模板创建 & 沙箱启动全链路

> 调研时间:2026-09-09(代码级调研,关键结论附 file:line 出处)。

**一句话结论**:模板 = 「OCI 镜像展开后的 ext4 rootfs」+「各节点 probe 2xx 后拍的 MicroVM 内存+文件系统快照」。开沙箱 = 从这份快照恢复一台轻量 VM,而不是像容器那样跑镜像。

---

## 一、全景图

```mermaid
flowchart TB
    subgraph S1["① 模板构建 · CubeMaster 主机"]
        direction TB
        T1["OCI 镜像"] -->|"pull:docker / skopeo+umoci / native"| T2["展开 rootfs"]
        T2 -->|"注入 envd + CubeEgress CA"| T3["mkfs.ext4 打包<br/>ext4 工件 rfs-*"]
        T3 -->|"分发给各节点"| T4["临时 MicroVM<br/>跑到 HTTP probe 2xx"]
        T4 -->|"内存 + CoW 快照"| T5["模板 READY<br/>tpl-*"]
    end

    subgraph S2["② 沙箱创建 · 控制面"]
        direction TB
        C1["SDK → CubeAPI :3000<br/>或 WebUI → CubeOps"] -->|"POST /cube/sandbox"| C2["CubeMaster 模板解析<br/>ID → 容器规格/卷/节点"]
        C2 --> C3["timeout 归一化<br/>仓库默认 -1 = 永不超时"]
        C3 --> C4["调度<br/>prefilter → filter → score"]
        C4 -->|"gRPC RunCubeSandboxRequest<br/>proto 无 timeout 字段"| C5["Cubelet"]
    end

    subgraph S3["③ 节点执行 · Cubelet"]
        direction TB
        N1["4 步流水线<br/>createid→资源→cgroup→cubebox"] --> N2["containerd NewTask"]
        N2 -->|"shim v2"| N3["CubeShim<br/>内嵌 VMM"]
        N3 -->|"restore_vm / boot_vm"| N4["MicroVM"]
        N4 -->|"vsock"| N5["cube-init → cube-agent"]
    end

    subgraph S4["④ 生命周期 · CLM"]
        direction TB
        L1["Redis lifecycle 流<br/>Timeout/EndAt/AutoPause"] --> L2["sweeper 空闲到期"]
        L2 --> L3{"AutoPause?"}
        L3 -->|"pause"| L4["CoW 快照挂起"]
        L3 -->|"kill"| L5["销毁"]
        L6["CubeProxy 撞 paused"] -->|"内部 resume"| L7["restore_vm 恢复"]
    end

    T5 -.->|"模板就绪"| C2
    C5 --> N1
    C2 -.->|"创建成功 hook"| L1
    C4 -.->|"查节点本地模板快照"| T5
```

图例:实线 = 主调用流;虚线 = 状态读写 / 就绪依赖。①–④ 对应下文四节。

---

## 二、模板是怎么造出来的(4 条途径)

| 途径 | 入口 API | 产物语义 |
|---|---|---|
| **A. OCI 镜像构建**(主路径) | CubeOps `POST /api/v1/sdk/templates`(sdk.go:703)→ CubeMaster `POST /cube/template/from-image` | ext4 rootfs + 各节点内存快照 |
| **B. commit 运行中沙箱** | `POST /cube/sandbox/commit`(template_commit.go:57) | 单节点 1 副本,其余 PARTIALLY_READY |
| **C. 用户快照** | `POST /cube/snapshot` → `SubmitSandboxSnapshot` | `snap-*`,可回滚/克隆/再开沙箱 |
| **D. legacy** | `POST /cube/template`(传 CreateCubeSandboxReq) | 新前端/SDK 已不走 |

### 途径 A 构建流水线

```mermaid
flowchart TD
    subgraph Entry["① 入口 · API 转发"]
        direction TB
        A["POST /api/v1/sdk/templates<br/>CubeOps sdk.go:703"] --> B["POST /cube/template/from-image<br/>CubeMaster template_from_image.go:27"]
    end

    subgraph Build["② CubeMaster 主机构建(不是 buildkit)"]
        direction TB
        C["拉镜像三选一<br/>docker pull / skopeo+umoci / native 流式"] --> D["展开 rootfs"] --> E["注入 envd + CubeEgress CA<br/>artifact_build.go:303"] --> F["truncate + mkfs.ext4 -d 打包<br/>ext4.go:20"] --> G["内容寻址:rfs- + sha256(fingerprint)<br/>相同输入跨模板复用同一份"]
    end

    subgraph Dist["③ 分发到节点 → 拍快照"]
        direction TB
        H["CreateImage RPC"] --> I["节点下载 + sha256 校验"] --> J["起临时沙箱 templateID_0"] --> K["HTTP probe 2xx → cube-runtime snapshot<br/>内存 dump 进 CoW 卷"] --> L["节点 catalog 落库 → 模板 READY"]
    end

    B --> C
    G --> H
```

**要点**:构建发生在 CubeMaster 主机上;节点只负责下载 ext4、起临时 VM、拍快照。模板"内容"由 probe 决定——是"MicroVM 起来后等 probe 2xx 才冻的 fs+memory",不是进程刚启动的镜像。

### 产物 GC(7 天 TTL 是"从最后引用起算")

```mermaid
flowchart TD
    G1["artifact 建好<br/>gc_deadline = now + 7 天<br/>job_constants.go:66"] --> G2{"被引用?<br/>模板 / replica / job"}
    G2 -->|"引用中 → 每次引用或删除都续期"| G2
    G2 -->|"最后一个引用消失"| G3["三阶段 last-owner 清理<br/>Phase1 记账 → Phase2 节点 DestroyImage → Phase3 删 master 本地"]
    G3 -->|"节点上有沙箱在用"| G4["CLEANUP_PENDING<br/>10 分钟一轮 GC 重试"]
    G4 --> G3
```

---

## 三、控制面:用模板创建沙箱

### 创建时序(两条入口汇聚同一端点)

```mermaid
sequenceDiagram
    participant SDK as SDK / 客户端
    participant API as CubeAPI :3000 或 CubeOps
    participant CM as CubeMaster
    participant SCH as 调度器
    participant CL as Cubelet
    participant R as Redis

    SDK->>API: POST /sandboxes(templateID, timeout, env, network...)
    API->>CM: POST /cube/sandbox(注解:template.id + version=v2)
    CM->>CM: dealCubeboxCreateReqWithTemplateCenter<br/>模板 ID → 容器规格/卷/节点 scope
    CM->>CM: resolveTimeoutSeconds(省略 + 集群默认 -1 → 永不超时)
    CM->>SCH: Select(选节点)
    SCH-->>CM: 选中节点
    CM->>CL: gRPC Create(RunCubeSandboxRequest)
    CL-->>CM: sandboxID / sandboxIP / 端口映射
    CM->>R: proxy map + lifecycle 事件(Timeout/EndAt/AutoPause)
    CM-->>API: 创建成功
    API-->>SDK: Sandbox
```

> 两条入口的差别:直接 SDK 经 **CubeAPI**(3000 端口,字段映射最全:env_vars→create_time_env_vars、lifecycle→auto_pause);WebUI/AgentHub 经 **CubeOps**,只转发 templateID/timeout/autoPause/metadata。Go SDK 目前没有 lifecycle 选项,只有 Python SDK 有。

### 模板解析(唯一落点,进入调度前同步完成)

```mermaid
flowchart TD
    P1["/cube/sandbox handler"] --> P2{"注解 version = v2?"}
    P2 -->|"是"| P3["ResolveTemplateIdentifier<br/>别名/短 ID 归一"]
    P2 -->|"否"| P9["legacy 本地模板配置"]
    P3 --> P4["GetTemplateRequest<br/>取出模板的 CreateCubeSandboxReq<br/>容器规格/卷/网络/注解"]
    P4 --> P5{"模板在健康节点就绪?"}
    P5 -->|"否"| P6["创建失败 NotFound"]
    P5 -->|"是"| P7{"kind?"}
    P7 -->|"template"| P8["注入组件版本注解"]
    P7 -->|"snapshot"| P10["钉节点 / DistributionScope<br/>跨节点需 S3 backend"]
    P7 -->|"pause_snapshot"| P11["拒绝(仅内部 resume 用)"]
    P8 --> P12["合并容器/卷/网络/注解<br/>→ 请求规格完备"]
    P10 --> P12
    P12 --> P13["调度:filter 里 template_locality<br/>校验节点本地模板快照"]
```

关键点:物理卷引用**不下发**给 Cubelet,只传逻辑 ID + backend,由 Cubelet 按 `RuntimeSnapshotID` 查本地 catalog(cubeboxutil.go:584-591)——防跨租户注解注入。

### timeout 的归一化与落点

```mermaid
flowchart TD
    X1{"客户端传了 timeout?"} -->|"没传"| X2{"集群 default_timeout_insec > 0?"}
    X1 -->|"传了 < 0"| X3["NeverTimeout(-1) 永不回收"]
    X1 -->|"传了 >= 0"| X4["用客户端值"]
    X2 -->|"是"| X5["用集群默认"]
    X2 -->|"否(仓库默认 -1)"| X3
    X3 --> X6["写入 Redis lifecycle meta<br/>不发给 Cubelet,sandbox_spec 也故意剔除"]
    X4 --> X6
    X5 --> X6
    X6 --> X7["CLM sweeper 到点执行<br/>AutoPause ? pause : kill"]
```

---

## 四、节点侧:沙箱真正跑起来

### Cubelet 四步流水线

```mermaid
flowchart LR
    W0["gRPC Create"] --> W1["step1<br/>createid + appsnapshot"]
    W1 --> W2["step2 并行<br/>images / storage / volume /<br/>network / cgroup 前置 / sandbox-store"]
    W2 --> W3["step3 cgroup<br/>算 VMM vCPU/内存<br/>→ OCI 注解 cube.vmmres"]
    W3 --> W4["step4 cubebox<br/>containerd NewTask<br/>→ 拉起 CubeShim"]
    W4 --> W5["envd 初始化<br/>POST sandboxIP:49983/init"]
```

### VM 启动(有快照走 restore,没快照冷启动)

```mermaid
sequenceDiagram
    participant CT as containerd
    participant SH as CubeShim(shim v2)
    participant VMM as cube-hypervisor(内嵌库)
    participant K as Linux KVM
    participant GI as guest: cube-init
    participant AG as cube-agent

    CT->>SH: Create + Start(ttrpc)
    SH->>SH: 解析 OCI 注解<br/>vmmres / net / disk / pmem
    alt 有模板快照(<60ms 的秘密)
        SH->>VMM: restore_vm(内存镜像)
    else 冷启动
        SH->>VMM: boot_vm
    end
    VMM->>K: ioctl(/dev/kvm)
    K-->>GI: 内核启动
    GI->>AG: exec cube-agent
    AG-->>SH: VsockServerReady(vsock)
    SH->>AG: create_sandbox(netns/网卡/路由)
    AG-->>SH: 就绪,沙箱可用
```

> 设备形态:virtio-net(host TAP,fd 经 unix socket SCM_RIGHTS 交接)、virtio-blk、virtiofs、**pmem0=guest OS 镜像、pmem1=cube-agent.ext4**、vsock。guest 的 `/sbin/init` 就是 cube-init。

### 网络数据面(访问沙箱 & 沙箱外发)

```mermaid
flowchart LR
    U["客户端"] -->|"Host: port-sandboxid.domain"| P["CubeProxy OpenResty<br/>查 Redis 路由"]
    P -->|"同节点直连 / 跨节点 DNAT"| VS["cubevs eBPF<br/>内嵌 Cubelet"]
    VS -->|"TAP"| VM["沙箱内服务"]
    VM -->|"外发"| VS
    VS -->|"SNAT / egress 策略默认拒绝"| I["公网"]
    VS -.->|"可选挂 CubeEgress<br/>域名白名单/凭据注入"| I
```

---

## 五、生命周期接管(CLM)

```mermaid
flowchart TD
    L1["创建成功 hook → Redis lifecycle 流"] --> L2["sweeper 每轮<br/>idle = max(LastActive, CreatedAt)"]
    L2 --> L3{"TimeoutSeconds?"}
    L3 -->|"< 0"| L4["永不处理"]
    L3 -->|"已到期"| L5{"AutoPause?"}
    L5 -->|"pause"| L6["SETNX 抢锁 → 推 pausing 给各 Proxy<br/>CubeMaster pause → Cubelet 拍 CoW 快照 + 收 shim"]
    L5 -->|"kill"| L7["kill → Proxy 返 410 Gone"]
    L6 --> L8["Proxy 撞 paused → CLM /internal/resume"]
    L8 --> L9["SETNX resuming → CubeMaster resume<br/>同 sandboxID 瘦 Create 走 restore_vm"]
    L9 --> L10["推 running 给 Proxy → 流量恢复"]
    L10 --> L2
```

> Proxy 状态门:`pausing→503 Retry-After`、`paused→内部 resume 子请求`、`killed→410`。用户访问被暂停的沙箱时,resume ~100ms,基本无感。

---

## 六、勘误清单(最容易误解的 5 点)

1. **CubeOps 的 warehouse(S3 blobstore/tar.gz 上传)与模板无关**——它是节点组件(agent 一键包)仓库。模板 rootfs 存在 CubeMaster 本机磁盘,经 CubeMaster 自己的 HTTP 端点下发。
2. **模板不是 OCI 镜像,是 VM 快照**:开沙箱是"从模板快照恢复 MicroVM"(docs/guide/templates.md:11)。
3. **构建工具不是 buildkit**:CubeMaster 主机上的 docker / skopeo+umoci + `mkfs.ext4 -d`;仓库没有 Dockerfile 构建入口。
4. **模板 ID 规格不可变**:同 ID 重提不同规格会报错(template_image.go:124-129),要改规格换新 ID 或 redo。
5. **状态存储分散**:业务状态机分在 CubeMaster 实例缓存表(`t_cube_instance_info.ins_state`)、Redis lifecycle state(CLM 视角)、节点本地 cubeboxstore 三处,不在 CubeDB 的某张 sandbox 表里。

---

## 附:常见疑问——沙箱有默认生存时间吗?

**没有。** 仓库出厂 `default_timeout_insec: -1`(configs/single-node/cubemaster.yaml:32),客户端不传 `timeout` 的沙箱**永不因空闲被回收**:

| 传入值 | 行为 |
|---|---|
| 省略 | 集群 `default_timeout_insec`;仓库默认 -1 → 永不超时 |
| `NEVER_TIMEOUT`(-1) | 永不超时 |
| `0` | 立即回收 |
| 正整数 N | 空闲 N 秒后触发(`on_timeout` 默认 kill,可设 pause) |

归一化唯一落点:`resolveTimeoutSeconds`(CubeMaster/pkg/service/sandbox/util.go:254);执行端是 CLM sweeper。集群运维把 `default_timeout_insec` 改为正数(如 300)即可自动回收不传 TTL 的沙箱,需重启 CubeMaster。
