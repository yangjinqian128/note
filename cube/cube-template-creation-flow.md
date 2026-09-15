# CubeSandbox 模板创建 & 沙箱启动全链路

> 调研时间:2026-09-09(代码级调研,关键结论附 file:line 出处);2026-09-13 对照 HEAD f7b2317c 勘误修订:修正文件路径与语义误差,全文精简为「模板创建 + 沙箱通过模板启动」两条主线。

**一句话结论**:模板 = 「OCI 镜像展开后的 ext4 rootfs」+「各节点 probe 200–399 后拍的 MicroVM 内存快照 + reflink rootfs 卷」。开沙箱 = 恢复内存与设备状态(文件系统是 CoW 克隆,不是快照恢复),而不是像容器那样跑镜像。

---

## 一、全景图

```mermaid
flowchart TB
    subgraph S1["① 模板构建 · CubeMaster 主机"]
        direction TB
        T1["OCI 镜像"] -->|"pull:native 流式(默认)/ skopeo+umoci / docker"| T2["展开 rootfs"]
        T2 -->|"注入 envd + CubeEgress CA"| T3["mkfs.ext4 打包<br/>ext4 工件 rfs-*"]
        T3 -->|"分发给各节点"| T4["临时 MicroVM<br/>跑到 HTTP probe 2xx–3xx"]
        T4 -->|"内存 + CoW 快照"| T5["模板 READY<br/>tpl-*"]
    end

    subgraph S2["② 沙箱创建 · 控制面"]
        direction TB
        C1["SDK → CubeAPI :3000<br/>或 WebUI → CubeOps"] -->|"POST /cube/sandbox"| C2["CubeMaster 模板解析<br/>ID → 容器规格/卷/节点"]
        C2 --> C3["调度选节点<br/>(prefilter → filter → score)"]
        C3 -->|"gRPC RunCubeSandboxRequest"| C4["Cubelet"]
    end

    subgraph S3["③ 节点执行 · Cubelet"]
        direction TB
        N1["Cubelet 流水线<br/>(资源/cgroup 等准备)"] --> N2["containerd NewTask"]
        N2 -->|"shim v2"| N3["CubeShim<br/>内嵌 VMM"]
        N3 -->|"restore_vm / boot_vm"| N4["MicroVM"]
        N4 -->|"vsock"| N5["cube-init → cube-agent"]
    end

    T5 -.->|"模板就绪"| C2
    C4 --> N1
    C3 -.->|"查节点本地模板快照"| T5
```

图例:实线 = 主调用流;虚线 = 状态读写 / 就绪依赖。①–③ 对应下文三节。

---

## 二、模板是怎么造出来的(OCI 镜像构建)

**一句话**:把 OCI 镜像交给 Cube,Cube 先把它展开成 ext4 根文件系统,再在每个节点上起一台临时 MicroVM、等沙箱里的服务通过健康检查后拍下「内存 + 文件系统快照」——这份"预热好的存档"就是模板。

### 构建流水线

```mermaid
flowchart TD
    subgraph Entry["① 入口 · 收请求"]
        direction TB
        A["用户提交 OCI 镜像<br/>(经 CubeOps 转发)"] --> B["CubeMaster 收到构建请求"]
    end

    subgraph Build["② CubeMaster 主机上 · 镜像 → ext4 文件"]
        direction TB
        C["拉取镜像,展开成 rootfs 目录"] --> D["注入 envd + CubeEgress CA<br/>(沙箱启动器 + 出网信任证书)"] --> E["打包成 ext4 文件<br/>(内容寻址:相同镜像复用同一份)"]
    end

    subgraph Dist["③ 各节点 · 试跑一次 → 拍快照"]
        direction TB
        F["下载 ext4 并校验"] --> G["起临时 MicroVM<br/>等里面的服务通过健康检查"] --> H["拍「内存 + 文件系统」快照"] --> I["登记进节点 catalog<br/>→ 模板 READY"]
    end

    B --> C
    E -->|"CreateImage RPC"| F
```

**要点**:构建发生在 CubeMaster 主机上;节点只负责下载 ext4、起临时 VM、拍快照。模板"内容"由 probe 决定——是"MicroVM 起来后等 probe 200–399 才冻的 fs+memory"(默认 GET :port/health、30s 预算、500ms 周期、连续失败 60 次放弃),不是进程刚启动的镜像。

### 节点侧执行:AppSnapshot 步骤

> ③ 里「起临时沙箱 → probe → 快照 → catalog 落库」在节点上的具体实现是 Cubelet 的 `service.AppSnapshot`(Cubelet/services/cubebox/appsnapshot.go:57),完整步骤:

```mermaid
flowchart TD
    subgraph S0["阶段一 · 准备:起临时沙箱(Step0–3)"]
        direction TB
        A["0. 前置校验<br/>注解 / backend=CoW / templateID"] --> B["Step1 起临时沙箱 templateID_0<br/>probe 通过才返回,PreConditionFailed 销毁重试"] --> C["Step2 取规格 resource/disk/pmem/kernel"] --> D["Step3 建 CoW 内存卷 tpl-<id>-memory<br/>(空卷,XfsCow 后端)"]
    end

    subgraph S1["阶段二 · 快照:冻结这台 VM(Step4)"]
        direction TB
        E["收集 envd 版本<br/>(冻结前,快照期间禁 exec)"] --> F["cube-runtime 全量内存快照<br/>vm.pause → RAM 写进内存卷(无压缩)→ vm.resume"] --> G["提交 rootfs 卷<br/>build-rootfs → tpl-<id>-rootfs(FICLONE)"]
    end

    subgraph S2["阶段三 · 收尾:拆临时沙箱 + 落位(Step5–7)"]
        direction TB
        H["Step5 销毁临时沙箱<br/>去激活 CoW 对象"] --> I["Step6 tmp rename 落位<br/>+ shim spec 链接"] --> J["Step7 写状态标志 /data/cube-shim/snapshot(+i)"]
    end

    K["节点 catalog 落库 → 返回 success"] --> L["CubeMaster: replica READY"]

    D --> E
    G --> H
    J --> K
```

> 失败保护:`forceDestroyCubebox` 闭包在 appsnapshot.go:187-206,真正的 defer 注册在 :208-212(`!snapshotSuccess && !temporaryCubeboxDestroyed` 时 force destroy),任何一步失败都先清理再报错。注意该 defer 在 Create **成功之后**才注册;Create 失败(含 probe 失败)由 Cubelet workflow 的 failover 清理。

## 三、控制面:用模板创建沙箱

### 创建时序(三条入口汇聚同一端点)

```mermaid
sequenceDiagram
    participant SDK as SDK / 客户端
    participant API as CubeAPI :3000 或 CubeOps
    participant CM as CubeMaster
    participant CL as Cubelet
    participant R as Redis

    SDK->>API: 创建沙箱请求<br/>(templateID, timeout, env, network...)
    API->>CM: POST /cube/sandbox<br/>(带 templateID)
    CM->>CM: 模板解析(dealCubeboxCreateReqWithTemplateCenter)<br/>ID → 容器规格/卷/节点 scope
    CM->>CM: 调度(内置模块,prefilter→filter→score)<br/>选中一个节点
    CM->>CL: gRPC Create(RunCubeSandboxRequest)
    CL-->>CM: sandboxID / sandboxIP / 端口映射
    CM->>R: 登记路由表 + 生命周期元数据
    CM-->>API: 创建成功
    API-->>SDK: Sandbox
```

> 入口实为三条:直接 SDK 经 **CubeAPI**(3000 端口,字段映射最全:env_vars→create_time_env_vars、lifecycle→auto_pause);**WebUI** 经 CubeOps,只转发 templateID/timeout(>0)/autoPause/metadata;**AgentHub** 经 CubeOps 的另一条路(agenthub.go:246-293),另转发 network_config/distribution_scope,timeout 硬编码 86400。Go SDK 目前没有 lifecycle 选项,只有 Python SDK 有。

---

## 四、节点侧:沙箱真正跑起来

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
    alt 有模板快照(RAM mmap 惰性填充,README 口径 ~60ms)
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

> 设备形态:virtio-net(host TAP,fd 先从 Cubelet tap 池取——unix socket SCM_RIGHTS,失败才 open /dev/net/tun)、virtio-blk(业务数据卷 → guest `/dev/vdX`)、virtiofs(tag cubeShared,插件卷/共享目录)、**pmem0=ext4 rootfs 镜像(`root=/dev/pmem0 rootflags=dax,ro`;内核是独立 vmlinux,由 VMM load_kernel 加载)、pmem1=cube-agent.ext4(guest 挂 `/run/support`)**、vsock。guest 的 `/sbin/init` 就是 cube-init。

### 恢复视图:快照里什么是旧的、什么是新的

> 从模板恢复沙箱的本质是「旧世界原样复活 + 现场接线」:进程/内核/挂载状态照单全收(快照的收益),stdio/时钟/熵/设备后端逐项换新(快照的代价)。「换新」全部通过恢复后的 guest 内 RPC 执行(ResetVm / CreateSandbox / CreateContainer 恢复分支)——这就是「冻结时刻 guest 活跃度 → 恢复时延」的落点。

```mermaid
flowchart TD
    subgraph OLD["❄ 旧世界 · 模板快照(构建时冻结,恢复时原样复活)"]
        direction TB
        O1["进程状态:应用 / envd / agent 的<br/>代码+数据页、vCPU/寄存器"]
        O2["内核状态:page cache、dentry、<br/>netns/路由、容器挂载表"]
        O3["容器内挂载:模板 rootfs(只读基础)<br/>+ 模板构建期的卷"]
        O4["应用 stdio:握着旧 shim 的<br/>vsock 连接 —— 对端已死 ✗"]
        O5["guest 时钟:冻结在构建时刻 ✗"]
        O6["RNG 状态:恢复后会重放 ✗"]
        O7["设备槽位 vdX / net-i:<br/>背后是模板卷、模板期 TAP"]
    end

    subgraph WIRE["🔌 接线 · 创建路径(逐个修复 / 换新)"]
        direction TB
        W1["restore_vm:内存灌回,<br/>vdX 同槽位换成 sb-rootfs 克隆、<br/>net-i 换新 TAP(guest 无感)"]
        W2["connect_agent:建新 vsock 控制通道<br/>(旧通道随旧 shim 已死)"]
        W3["ResetVm:SetGuestDateTime(校时)<br/>+ ReseedRandomDev(重播种)"]
        W4["CreateSandbox(RESTORE):<br/>add_virtiofs_storages 挂新卷"]
        W5["CreateContainer(restore 分支):<br/>find 进程 + passfd 重连<br/>+ mount propagation 进容器 ns"]
    end

    subgraph NEW["🌱 新世界 · 这个沙箱专属的环境"]
        direction TB
        N1["新 shim / containerd / sandboxID<br/>新日志管道"]
        N2["新 rootfs(sb-rootfs FICLONE 克隆)<br/>+ 新可写层 emptyDir"]
        N3["新用户卷 / 插件卷(virtiofs)"]
        N4["新 stdio 通道(日志续上)"]
        N5["新时间 / 新熵"]
        N6["新 TAP(同槽位网卡,新 IP/路由)"]
    end

    O1 -->|"不换,原样复活"| W1
    O2 -->|"不换,原样复活"| W1
    O7 -->|"换后端,guest 无感"| W1
    W1 --> N2
    W1 --> N6
    O4 -->|"死连接 → 换新"| W5
    O3 -->|"缺新卷 → 补挂"| W5
    W5 --> N4
    W5 --> N2
    O5 -->|"冻结时钟 → 校"| W3
    O6 -->|"重放熵 → 重播"| W3
    W3 --> N5
    W4 --> N3
    W2 --> N1
```

**读图要点**:三样「原样复活、不换」——进程状态、内核状态、容器内模板挂载(快照的全部价值);五样「现场换新/补新」:

| 旧的(冻在快照里) | 为什么必须换 | 换新动作 |
|---|---|---|
| stdio vsock 连接 | 对端旧 shim 已死 | passfd 重连 |
| 容器 ns 缺新卷 | 新沙箱有专属可写层/用户卷 | mount propagation |
| guest 时钟 | 冻结在构建时刻 | SetGuestDateTime |
| RNG 状态 | 所有沙箱会重放同一随机序列 | ReseedRandomDev |
| 设备后端(vdX / TAP) | 模板卷被共享、网卡接新网络 | 同槽位换后端(guest 无感) |

关键不对称:「不换」的是快照的收益来源,「必须换」的是快照的代价来源——代价全部通过恢复后的 guest 内执行支付。

