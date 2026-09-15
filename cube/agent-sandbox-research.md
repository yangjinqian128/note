# Agent 沙箱（Agent Sandbox）调研笔记

> 广义上，沙箱是指一个受控的、隔离的环境，容器及QEMU虚机都是沙箱。agent沙箱则是「给 LLM 生成的不可信代码/命令提供受控执行环境」的托管服务——隔离、可观测、可终止、可计费、可快照回滚。它是 AI Agent 的工具执行底座：底层复用虚拟化/容器领域的积累（KVM microVM、内存快照、CoW），但工作负载形态（高频创建销毁、短生命周期、事件驱动）与传统 IaaS/PaaS 完全不同。
>
> 调研时间：2026-09。以 Cube（腾讯云，KVM microVM 路线，2026-04 已开源）的视角做横向调研；商业产品数据以官方文档与 2025–2026 公开资料为准，关键数字建议对照文末参考链接复核。

## 一、概念与背景

### 1.1 什么是 Agent 沙箱？它解决的核心问题

**定义**：Agent 沙箱是为 AI Agent（LLM + 工具调用循环）提供的受控代码执行环境。Agent 在沙箱里运行 shell 命令、写代码、跑脚本、装依赖、起服务、操作浏览器，沙箱保证这些行为：

1. **隔离**——不可信代码破坏不了宿主/其他租户；
2. **可观测**——每一步执行的 stdout/stderr/事件流可追溯；
3. **可终止**——超时、配额、显式强杀；
4. **可计费**——多租户下按用量计量；
5. **可恢复**——快照/回滚，环境可重置、可复制。

**它解决的核心问题**：LLM 输出的代码和命令**天然不可信**。不可信来源包括：prompt injection（网页/邮件/第三方文档里的恶意指令）、被投毒的训练数据、幻觉生成的危险命令、以及第三方 MCP 工具/插件引入的供应链风险。直接让 Agent 在用户机器或共享服务器上裸跑，等于把不可信代码与真实数据、凭据放在同一信任域。

**Agent 沙箱与「跑个容器」的区别**：普通容器解决「应用怎么跑」，Agent 沙箱解决「Agent 怎么跑」——它把容器/VM 技术封装成面向 Agent 工作负载的产品语义：

| 维度 | 传统容器/VM 平台 | Agent 沙箱 |
|---|---|---|
| 生命周期 | 天/月级常驻 | 秒~分钟级，一次 tool call 可能就一个沙箱 |
| 创建频率 | 低频（发布/扩缩容） | 高频（每个会话/每步都可能新建） |
| 交互方式 | SSH / kubectl / 控制台 | SDK（Python/TS），与 Agent 框架直接集成 |
| 计费 | 按月/按实例规格 | 按秒/按分钟，闲置自动暂停（pause） |
| 状态管理 | 用户自己操心 | 模板、快照、回滚是产品内置能力 |
| 信任模型 | 跑可信应用 | 默认跑不可信代码，安全是首要需求 |

### 1.2 为什么随 AI Agent 发展，沙箱变得重要

- **工具调用（tool/function calling）成为 Agent 标配**：2023 年起 OpenAI Function Calling、2024 年末 MCP（Model Context Protocol）发布后，Agent 执行代码/命令的入口标准化了，执行环境成为每条链路都绕不开的一环。
- **代码执行是 Agent 最高危的工具**：读文件、发网络请求、改系统状态的能力都集中在它身上。没有沙箱时，一次 prompt injection 就能从「模型被忽悠」升级为「真实系统被攻破」。
- **典型产品的印证**：ChatGPT Code Interpreter / Claude 数据分析功能——模型生成 Python 跑 pandas/matplotlib，底层就是托管沙箱；Devin、OpenHands、Cline、Claude Code 等编程 Agent 需要 shell + 编辑器 + 浏览器全套环境，厂商都自建或对接沙箱；OpenAI Agents SDK、LangChain、Agno 等框架都抽象出 sandbox provider 接口（见 6.2）。
- **安全事件的推动**：2024 年 xz-utils 后门说明供应链投毒已进入实战；2025–2026 年 MCP 生态连续爆出 tool poisoning 攻击（见 6.3）；各类「Agent 被骗执行恶意命令」的演示层出不穷。业界共识从「要不要隔离」变成「隔离到什么级别、多少开销」。
- **多租户 SaaS 的刚需**：任何「帮你跑 AI 代码」的云产品（代码解释器、数据分析、自动化平台）都在多租户下运行用户代码——没有强隔离就没有上架资格。
- **2026 年上半年云厂商集体入场**：Vercel Sandbox GA（2026-01）、Cloudflare Sandbox/Containers GA（2026-04）、Google Agent Sandbox（2026-04）、AWS Lambda MicroVMs / Bedrock AgentCore（2026-06/08）——媒体称之为「四朵大云，七个月，同一个产品」。沙箱从创业公司题材变成云厂商标配。

### 1.3 与传统沙箱的区别与联系

Agent 沙箱不是新发明，是既有隔离技术的**组合封装 + 工作负载语义改造**：

| 传统沙箱 | 设计目标 | 与 Agent 沙箱的关系 |
|---|---|---|
| 进程沙箱（seccomp/namespaces/Landlock，如 Chromium sandbox、bwrap） | 限制单个程序的能力 | 最细粒度手段，常作为容器/microVM 内部的第二道防线 |
| 容器沙箱（Docker/Podman/gVisor/Kata） | 打包应用 + 轻量隔离 | 一部分 Agent 沙箱直接基于容器（OpenHands 默认 runtime、Daytona、Google Agent Sandbox）；共享内核是逃逸面 |
| 浏览器沙箱（Chromium 多进程 + OS sandbox） | 渲染不可信网页 | 浏览器自动化 Agent 需要它，但它是「浏览器里的网页」模型，管不住 Agent 跑的系统级命令 |
| 虚拟机/微虚拟机（QEMU/KVM、Firecracker、Cloud Hypervisor） | 硬件级隔离的独立内核 | **当前 Agent 沙箱的主流底座**（E2B、Fly、Vercel、Cube 均为 microVM），隔离强度与启动速度的最佳平衡点 |

一句话概括演化路径：**Agent 沙箱 = microVM（或强隔离容器）作为隔离底座 + 快照/模板作为启动优化 + SDK/事件流作为产品界面 + 自动暂停/计费作为运营语义**。

---

## 二、核心能力与技术要求

### 2.1 隔离级别：进程级 → 容器级 → 虚拟机级 → 微虚拟机级

```
隔离强度 ▲
          │  ┌─────────────────────────────────────────────┐
          │  │ 微虚拟机 microVM：KVM + 裁剪设备模型          │ ← Agent 沙箱主流
          │  │ （Firecracker / Cloud Hypervisor / Cube      │
          │  │  快照恢复 ms 级、MB 级内存开销）              │
          │  ├─────────────────────────────────────────────┤
          │  │ 虚拟机 VM：QEMU/KVM 全功能设备               │ ← 隔离强但启动秒级、密度低
          │  ├─────────────────────────────────────────────┤
          │  │ 强隔离容器：gVisor（用户态内核）/ Kata（容器  │
          │  │  接口 + VM）                                 │
          │  ├─────────────────────────────────────────────┤
          │  │ 容器：namespace + cgroup + seccomp + LSM     │ ← 共享内核，逃逸面=内核
          │  ├─────────────────────────────────────────────┤
          │  │ 进程级：seccomp-bpf / Landlock / caps        │
          │  └─────────────────────────────────────────────┘
          └────────────────────────────────────────────────→ 启动速度/部署密度 ▲
```

各层级要点与攻击面：

- **进程级**：seccomp-bpf 过滤 syscall、Landlock 限文件访问、capabilities 收权。启动零开销，但共享同一内核——**内核 CVE 即逃逸**，且用户态漏洞面全开。适合做「沙箱内的沙箱」（microVM 里再套一层限制 Agent 进程）。
- **容器级（Docker/Podman）**：namespace 隔离视图 + cgroup 限资源 + seccomp/LSM 收 syscall。逃逸面 = 宿主内核 + 容器运行时（runc CVE-2019-5736、CVE-2024-21626 都是容器逃逸）；rootless（user namespace）可显著降低风险。优点是生态、镜像、工具链最全。
- **虚拟机级（QEMU/KVM）**：独立内核、硬件虚拟化隔离。问题在于 QEMU 设备模型大（USB/显示/网络全模拟），攻击面宽，且冷启动秒级、内存 GB 级起——对「一次 tool call 一个沙箱」的场景太重。
- **微虚拟机级（microVM）**：KVM + **最小设备模型**（virtio-mmio 块/网 + vsock + serial，无 BIOS/legacy 设备）+ **轻量 VMM**（Rust 重写：Firecracker、Cloud Hypervisor、StratoVirt）+ **内存快照启动**。启动从「BIOS→内核→init」变成「restore 预 boot 内存镜像」，ms 级；内存开销 MB 级（Firecracker 官方 <5MiB 开销 + guest）。**当前 Agent 沙箱的最佳平衡点**。
  - 注意：microVM 的强隔离是**相对容器**而言，它自己的逃逸面 = VMM + KVM + guest 内核，仍需纵深防御（见 2.6）。

### 2.2 文件系统隔离与持久化

Agent 沙箱的根文件系统本质是「**只读模板 + 可写层**」，两种实现路线：

| 方案 | 机制 | 代表 | 特点 |
|---|---|---|---|
| 文件级 CoW | overlayfs（upper/lower 层） | Docker、绝大多数容器沙箱 | 页级写时复制，简单；first-write 有性能开销，层多时元数据开销大 |
| 块级 CoW | XFS reflink（FICLONE）/ ZFS clone / 精简置备 | **Cube（CubeCoW：XFS reflink 克隆 rootfs 卷）**、E2B 云盘快照 | 克隆只复制块映射表 + 引用计数，**O(1) 与卷大小无关**；写多少占多少 |

持久化语义要分层设计，用户必须清楚「哪些数据会丢」：

- **沙箱内临时数据**：随沙箱销毁丢弃（默认语义，防账单陷阱也防数据残留）；
- **volume 挂载**：独立生命周期的持久卷（E2B cloud / Modal Volume / Fly Volumes）；
- **模板**：预装的库/工具在模板层固化，所有沙箱共享只读层——把「常用依赖安装」从每次冷启动里拿掉（E2B 的 envd 自定义模板、Cube 的模板构建流水线）；
- **对象存储归档**：会话产物（生成的图、报告、模型权重）导出到 OSS/S3/R2（Cloudflare Sandbox 支持把 R2/S3/GCS 挂载为本地文件系统）。

### 2.3 网络访问控制

三个档位，按需组合：

1. **断网**：完全离线沙箱（最高敏感度任务：私有代码、密钥材料）；
2. **白名单**：域名/IP 级 allowlist，默认拒绝（Modal `outbound_domain_allowlist`、Google Agent Sandbox 默认阻断出站）；
3. **代理出口**：所有外发流量经托管 egress proxy，做 L7 检查与审计。

工程实现要点（以 Cube 为参照）：

- 数据面用 **eBPF**（CubeVS 虚拟交换机）做内核态线速转发 + SNAT；出口流量 TPROXY 透明劫持到用户态 **CubeEgress** 做 L7 域名白名单检查；v0.5.0 起每沙箱出站流量带独立 token + 策略路由；
- **凭据注入走代理不走沙箱**：API key 等在 egress 层注入，**密钥永远不进沙箱**——沙箱内进程即使被攻破也偷不到明文凭据（Cloudflare Sandbox 的凭据注入是同一思路：agent 永不接触 token）；
- **云元数据服务防护**：必须封死 169.254.169.254（AWS/GCP/腾讯云 metadata），否则沙箱内代码可直接偷取宿主/云账号的临时凭据——这是沙箱化最容易漏的点；
- **入向流量**：沙箱内起的 dev server/数据库经统一网关代理暴露（按 sandbox_id + 端口路由），配合 auto-pause（请求来了先 ~100ms resume 再转发，用户无感）。

### 2.4 资源限制（CPU、内存、磁盘、超时）

- **CPU**：vCPU 数在 VM 配置层固定；宿主侧 cgroup v2 对 vCPU 线程限 `cpu.max`（配额）与 `cpuset`（绑核）——microVM 的 vCPU 就是宿主的普通线程，cgroup 机制完全适用（Cube 自带 NUMA 池 + 叶子 cgroup 管理，见 [cube-cgroup-cpu-binding](./cube-cgroup-cpu-binding.md)）；
- **内存**：microVM 内存预分配（无 balloon 时不超卖）或 virtio-balloon 动态回收；「预留 vs 超卖」是商业决策——超卖提高密度但引入 OOM 与噪音邻居风险；
- **磁盘**：CoW 卷 quota + IO 限速（cgroup io 控制器）；CoW 天然「写多少占多少」，但 quota 仍是必须的（防无限写爆宿主盘）；
- **超时三层**：① exec 级超时（单次命令/tool call，LLM 循环里最常用）；② 请求/会话级超时；③ 沙箱最长存活时间（到点强杀，防僵尸沙箱）。

### 2.5 快照 / 恢复 / 克隆能力

这是 Agent 沙箱区别于传统 VM 产品的**核心性能技术**，也是各家竞争的焦点：

```mermaid
flowchart LR
    subgraph 模板制备["模板制备（低频，构建时）"]
        B["启动模板 VM<br/>（内核 + init 就绪）"] --> M["保存内存快照<br/>+ rootfs 卷"]
    end
    subgraph 沙箱创建["沙箱创建（高频，<100ms）"]
        M --> C1["CoW 克隆 rootfs 卷<br/>（FICLONE，O(1)）"]
        M --> C2["restore 内存快照<br/>（跳过 BIOS/内核/init）"]
        C1 & C2 --> R["沙箱就绪<br/>（vsock ready）"]
    end
```

- **模板预 boot**：把「BIOS → 内核 → init → 依赖就绪」的慢路径放在模板构建期，创建沙箱时只做 **restore**——「唤醒冬眠的人」而不是「重新造人」。Cube 由此做到 **<60ms 创建**（官方口径：行业均值约 150ms 的 1/3；裸金属实测均值 47.8ms / P95 57.4ms）；Firecracker 官方冷启动 <125ms，快照恢复再低一个数量级。
- **克隆**：同一模板并发克隆 N 份 = CoW 卷（零数据拷贝）+ 内存快照文件；Cube 的 EPT Lazy Load + 内存 mmap CoW 让多实例共享同一物理副本，新快照只写 dirty page（基于 Linux soft-dirty 的增量快照）。
- **快照即模板、克隆即分叉、回滚即复位**（Cube v0.3.0 CubeCoW 的一等 API）：快照 ID 可直接当模板用；`clone(N)` 从运行中沙箱一行调用派生 N 个独立副本（继承内存/文件/连接）；**原地回滚保持 sandbox_id 不变**，无需重连——恢复从分钟级降到百毫秒级。这是「事件级快照回滚」的产品化落地（详见 6.4）。
- **pause / resume**：闲置沙箱暂停（CLM 自动决策），请求到达时 ~100ms 恢复——成本与体验兼得（Cube 的 AutoPause/AutoResume，v0.5.0 起开源）。

### 2.6 安全逃逸防护与攻击面分析

microVM 的逃逸面清单与对应防线：

| 攻击面 | 风险 | 防线 |
|---|---|---|
| guest 内核 | Linux 内核 CVE 高频，逃逸最常见突破口 | 内核最小化（去模块/去驱动）、快速跟进 CVE、镜像受控发布 |
| virtio 设备模拟（VMM 侧） | 恶意 guest 打设备模拟代码 | 设备模型最小化（只留 net/block/vsock）、Rust 内存安全（Firecracker/CH） |
| KVM 子系统 | KVM ioctl 面 + CPU 虚拟化漏洞 | 宿主内核 CVE 跟踪、microcode 更新 |
| VMM 进程 | 宿主上被攻破的 VMM = 宿主权限 | jailer（chroot + seccomp + cgroup + 收权，Firecracker 模式）、VMM 进程本身低特权 |
| vsock / 串口 | 内外通信通道被滥用 | 协议面收敛、长度/权限校验 |
| 宿主共享组件 | 快照文件、卷、网卡（TAP）被越权访问 | 文件权限、cgroup 隔离、独立网络命名空间 |
| 侧信道 | 多租户同机 Spectre/MDS 类攻击 | 核隔离、敏感负载独占、厂商静默修复 |

- **纵深防御组合**：microVM 隔离（第一层）→ VMM jail/seccomp（第二层）→ guest 内进程级沙箱（第三层，如跑 Python 时再套 seccomp）→ 网络 egress 审计（最后一层）。单层都不是银弹，组合才有效。
- **容器路线的前车之鉴**：runc CVE-2019-5736（覆写宿主二进制）、CVE-2024-21626（文件描述符泄漏逃逸）、Dirty Pipe（CVE-2022-0847）、CVE-2026-31431（Copy Fail，2026-04——容器方案受影响而 microVM 方案不受影响的典型案例）——共享内核决定了容器逃逸是「内核 CVE 到租户数据」的一步之遥；gVisor/Kata 通过加一层用户态内核/VM 把这条路堵上，代价是性能与兼容性。
- **供应链风险**：隔离降低爆炸半径，但**不解决**投毒——模板基础镜像、pip/npm 依赖、第三方 MCP 工具都可能是入口（xz 后门教训）。对策：模板镜像可溯源、依赖锁定、凭据不进沙箱（把偷到的东西降到最小）。

---

## 三、主流方案对比

> 表格先给全景，后面逐方案简述。数据为 2026-09 快照，商业产品以官方文档为准；「启动速度」均指官方宣称的典型值（实测口径差异见各小节与文末参考）。

| 方案 | 隔离机制 | 启动速度 | 持久化 | 网络控制 | 适用场景 | 开源/商业 |
|---|---|---|---|---|---|---|
| **Cube Sandbox（腾讯云）** | KVM microVM（RustVMM/Cloud Hypervisor 分支，Shim v2 内嵌） | **<60ms**（内存快照恢复；行业均值约 150ms） | CoW 卷（XFS FICLONE）+ CubeCoW 快照/克隆/原地回滚 API | eBPF VS + L7 出口白名单/断网 + 每沙箱流量 token | 通用 Agent 沙箱（E2B SDK 兼容）、Agentic RL 底座 | 开源 Apache-2.0（2026-04 服务级开源） |
| **E2B** | Firecracker microVM（每沙箱一 VM，自托管 AWS/GCP 裸金属） | 数百 ms 级（快照恢复）；fork 单请求 ≤100 个 | 沙箱文件 + envd 模板 + Volume + 快照 | nftables egress 防火墙 + 域名黑白名单 | 代码解释器、computer use、Agent 后端 | SDK+runtime 开源（Apache-2.0，活跃），云服务商业 |
| **AgentENV（Moonshot/kvcache-ai）** | Firecracker microVM 集群 + overlaybd 按需 OCI 镜像 | boot/resume <50ms；pause/增量快照 <100ms | 增量快照；单沙箱同节点 fork ≤16 子沙箱 | —（未披露） | Agentic RL 训练（Kimi K3 底座）、评测 | 开源（MIT，2026-07） |
| **OpenSandbox（阿里）** | Docker/K8s 编排层（可挂 gVisor/Kata/Firecracker 安全运行时） | K8s 批量创建 100 个约 0.92s（Pool 池化 + BatchSandbox CRD） | 有状态代码执行（execd + Jupyter 内核） | 统一入口网关 + 逐沙箱出口控制 | 编码/GUI Agent、评估、RL 训练、多租户服务 | 开源（Apache-2.0，2025-12） |
| **Daytona** | Linux 容器为主（另有 VM/Windows/GPU 沙箱） | 宣称 <90ms（池预热）；pause/fork 仅 VM 档 | workspace 卷 + 快照 | 密钥管理、ingress/egress 分控 | 开发环境、编码 Agent、长时任务 | **已闭源（2026-06）**，旧仓库 AGPL-3.0 停维护 |
| **Modal Sandboxes** | gVisor（runsc）为主；VM Sandboxes（Alpha）补充 | ~1s 级；内存快照提速 2.5×+ | Filesystem/Directory/Memory 三类快照 + Volume | block_network / CIDR / 域名白名单 / 加密隧道 | 数据分析、GPU 计算、Claude Managed Agents 运行时 | 平台商业，SDK 开源（Apache-2.0） |
| **Fly.io（Machines / Sprites）** | Firecracker microVM | Machines ~300ms；Sprites 唤醒 warm 100–500ms | Volumes/Snapshots；Sprites 持久 ext4 + checkpoint ~300ms | 6PN 私有网络（WireGuard+Anycast）、HTTPS 唤醒 | 边缘应用、有状态 Agent「计算机」 | 商业 |
| **Cloudflare（Sandboxes / Containers）** | OCI 容器（共享内核）+ V8 isolates（轻量档） | 沙箱快照恢复 ~2s（冷 ~30s）；isolates ms 级 | 会话快照 + 持久 code interpreter + R2 挂载 | egress proxy 凭据注入、preview URL | Agent 后端、浏览器自动化（Browser Run） | 平台商业，SDK 开源 |
| **Firecracker** | KVM microVM（Rust VMM，AWS 开源） | 冷启动 <125ms，快照恢复 ms 级 | 快照文件（含 diff 快照） | 宿主侧配置（TAP/iptables） | **底层原语**（E2B/Fly/Vercel/Lambda 的底座） | 开源（Apache-2.0） |
| **gVisor** | 用户态内核（Sentry + Gofer/9P） | 容器级（秒内） | OCI 层 | 容器网络（经宿主） | 强隔离容器（App Engine/Cloud Run/Google Agent Sandbox） | 开源（Apache-2.0） |
| **Kata Containers** | 容器接口 + VM（QEMU/CH/Firecracker 后端） | 秒级（VMM 池化后数百 ms） | OCI 层 | Kubernetes CNI | K8s 上的强隔离容器 | 开源（Apache-2.0） |
| **Docker / Podman** | namespace + cgroup + seccomp + LSM | 百 ms 级 | overlay 卷 | 网桥/NAT/CNI | 通用容器、开发环境 | 开源 |
| **Wasm 沙箱（Wasmtime）** | 语言级 capability 沙箱（WASI，默认无权限） | µs~ms 级 | WASI 目录显式映射 | 默认无网络，需显式授权 | 插件、函数、确定性计算、边缘 | 开源（Apache-2.0） |
| **浏览器沙箱（Playwright + 容器）** | Chromium 多进程沙箱 + 容器/microVM 外层 | 秒级（浏览器进程启动） | 用户数据目录 | 容器网络 | 浏览器自动化 Agent | 开源（Chromium/Playwright） |

### 3.1 Cube Sandbox（腾讯云）

- **2026-04-21 全栈开源**（Apache-2.0，github.com/TencentCloud/CubeSandbox）——「服务级开源」：不只是 SDK，还包括运行时、调度系统、一键部署脚本、文档示例；2026-07 登 GitHub Trending Rust 榜首（star 10k+），列入 CNCF Landscape（AI-Native Infra 分类）。开源前 3 月在上海峰会由汤道生首次披露，定位「业内唯一兼顾硬件级强隔离与亚百毫秒启动的开源 AI Agent 沙箱服务」。
- **兼容性**：原生兼容 OpenAI Python SDK 与 E2B SDK——不改代码，改环境变量（`E2B_API_URL` / `CUBE_TEMPLATE_ID`）即可从海外闭源方案迁移。
- **性能口径（官方）**：冷启动 <60ms（行业均值约 150ms 的 1/3）；50 并发均值 67ms、P95 90ms、P99 137ms；hypervisor 层内存开销 <5MB；单台 96 vCPU 物理机 2000+ 沙箱；平台瞬时调度 100K+ 实例，分钟级拉起数万沙箱。官方「五大突破」：硬件级强隔离、亚百毫秒冷启动（资源池化预置 + 快照克隆 + EPT Lazy Load + 全栈锁优化）、极致轻量化（CoW 内存复用 + Rust 裁剪 + reflink 磁盘共享，存储消耗较传统方案降 90%+）、超大并发调度、事件级快照回滚。
- **第三方实测（供对照）**：裸金属 96C 均值 47.8ms / P95 57.4ms；PVM 16C/32G 均值 66.7ms / P95 78.2ms；WSL 本地约 750ms（受 I/O 与网络栈影响）；端到端内存摊销约 21–34MB/实例（README 的 <5MB 为 hypervisor 层开销）；HVTracker 信任分 46.3（产品较新、独立评测信号少）。
- **架构**（详见 [cube-sandbox-learning-notes](./cube-sandbox-learning-notes.md)）：CubeAPI（Rust，认证/限流）→ CubeMaster（调度中枢，无状态）→ Cubelet（节点执行者，内嵌 containerd，Shim v2 拉起 CubeShim）→ CubeShim 内嵌 RustVMM（Cloud Hypervisor 的 Rust 分支，以**库**形式嵌入）→ KVM 拉起 microVM；CubeProxy（E2B 协议兼容网关）、CubeVS（eBPF 虚拟交换机）、CubeEgress（L7 出口管控）。
- **版本线**：v0.1.0（2026-04-20 首发）→ v0.3.0（2026-06-02，CubeCoW 快照/克隆/回滚 API）→ v0.4.0（凭证保险箱 + Dashboard）→ v0.5.0（2026-07-03，**ARM64 原生全栈支持** + AutoPause/AutoResume + Terraform 一键集群部署 + 每沙箱流量 token 网络策略）→ v0.7.0（2026-08-28，跨节点 pause/resume + S3 后端快照）。
- **生产验证**：诞生于腾讯云 Serverless 体系（百亿级调用）；腾讯元宝 AI 编程迁移后资源核时消耗降低 95.8%；支撑 MiniMax Forge Agent RL——百万级吞吐、十万级并发，30,000 个 4C8G 沙箱 58 秒拉起、单镜像每分钟 60 万沙箱、QPS >10,000、交付约 80ms（较冷启动降 96%），且需同时支撑数十万异构（Linux/Windows/Android）沙箱并发。
- **部署前提**：KVM 裸金属（x86_64/aarch64）+ XFS 文件系统；PVM 嵌套虚拟化与 live migration 仅 x86_64；ARM64 需原生 KVM。
- **差异化**：硬件级隔离 + 百毫秒级快照回滚 + ARM64 + 开源 + 国内生态；对 Cube 团队自身而言，自托管所要求的「虚拟化 + 内核运维能力」正是腾讯云的存量优势（见 5.6）。

### 3.2 E2B

- 2023 年成立（FoundryLabs，捷克裔团队）；2024-10 $11.5M 种子轮，2025-07 $21M Series A（Insight Partners 领投，前 Docker CEO Scott Johnston 天使），累计约 $32M。
- 技术栈：**每个 sandbox 一个 Firecracker microVM**（独立内核 + cgroup + netns + per-sandbox nftables egress 防火墙 + SNI/Host 域名黑白名单）；模板 = 预启动 VM 快照（内存+磁盘+机器状态存对象存储），创建即恢复快照，内存页 userfaultfd 惰性加载，rootfs 只读镜像 + CoW overlay；fresh/resume/fork 走同一条快路径；**fork 可对运行中沙箱原地 checkpoint，单请求最多派生 100 个**；idle 自动 pause（diff 上传对象存储）、流量到达自动唤醒。
- 开源与自托管：SDK + **runtime（原 infra，2026-09 更名）** 全部 Apache-2.0 且高度活跃（近 4 周 246 commits）；**2025-04 全开源**；自托管 Terraform 支持 AWS/GCP（Azure BYOC 进行中），需 KVM 裸金属（Hetzner/OVH 是常见低成本选型）；单机 Linux+KVM 可 docker compose 跑完整栈（E2B Embed，官方定位评测用途）；无 GPU；ARM64 官方未确认（社区有 arm64 适配分支）；自称「唯一开源的托管 microVM 沙箱平台」。
- SDK（官方仅 Python/JS）：Sandbox.create/connect/list/kill/pause、超时与生命周期策略（Hobby 1h / Pro 24h，`onTimeout: pause + autoResume`）、文件系统/进程/PTY API、envd（VM 内 agent，Connect RPC，支持运行时热升级）、Build System 2.0（2025-10，模板构建完成即快照，加载约 80ms 且进程已在运行）、Volume/Secrets（egress 侧注入）、workload identity、**MCP 原生支持**（2025-10 与 Docker 合作）。
- 产品形态：code interpreter 沙箱（Python/JS/TS/R/Java/bash）、Desktop 沙箱（computer use：鼠标键盘/截图/桌面流）；**浏览器场景**用 kernel-browser 模板（Chromium 经 CDP）或第三方 Browserbase——无自有浏览器沙箱产品。
- 商业模型：Hobby 免费（$100 credits）→ Pro $150/月 + 按秒用量（vCPU $0.000014/s、内存 $0.0000045/s/GiB）→ Enterprise（最低约 $3,000/月，BYOC）；数据面跑在 Google Cloud（US/EU/APAC）。规模口径：1B+ 沙箱启动、10M+ SDK 月下载、94/100 Fortune 100 签约、SOC 2 Type II。
- 生态地位：**E2B SDK 正在成为 Agent 沙箱的事实标准接口**（证据与边界见 6.2）；CVE-2026-31431（Copy Fail，2026-04）中容器方案受影响而 E2B microVM 不受影响——隔离价值的具体印证。

### 3.3 Daytona

- 2023 年成立（创始人来自 Codeanywhere），2025-04 战略转向《From Dev Environments to AI Runtimes》——从「开发环境管理器」转为「AI 代码运行时」；定位「给每个 Agent 一台可组合计算机」（start/pause/fork/snapshot/destroy）。
- **底层：默认 Linux 容器（OCI 兼容，<90ms 创建）**，另有 Linux VM / Windows sandbox（独立内核）/ GPU sandbox（H100/B200/MI355X，独占分配）多形态——pause/resume、fork、hot snapshot 是 **VM-only** 能力；隔离边界三层（Runtime 资源硬限 / Network 进出分控 / Organization 多租户）。
- **2026-06-11 宣布闭源**（理由：安全——AI 已能系统性挖掘开源漏洞，「卖隔离边界的产品不能公开蓝图」）；旧仓库 daytonaio/daytona（71.7k stars，AGPL-3.0 仅存于 v0.190.0 tag）停止维护，SDK/文档迁至 github.com/daytona——「开源自托管」卖点实质上消失，选型需重新评估。
- SDK 覆盖 5 语言（Python/TS/Ruby/Go/Java）+ MCP server + SSH/VNC；2026-02 $24M Series A（FirstMark 领投，Datadog/Figma Ventures 战略投资，累计约 $31M），2026-08 传闻追加 $48.3M（SEC 文件来源，未官宣，待核实）；纯按量计费（vCPU $0.0504/h、内存 $0.0162/GiB·h，按秒，$200 免费额度）。
- 与 E2B 的核心差异：**容器为默认形态**（E2B 纯 microVM）、平台广度更大（Windows/GPU）、开源策略相反（E2B 活跃开源，Daytona 闭源）；OpenAI Agents SDK 对其定位「SDK 标准 workspace、snapshot 友好、长时任务推荐」。生态：Devin Outposts（2026-07）、Cursor Self-Hosted Machines（2026-09）均在 Daytona 落地。

### 3.4 Modal Sandboxes

- 2025-01-21 GA 的 serverless 平台（GPU/CPU）沙箱原语：**gVisor（runsc）隔离** + 自研 Rust runtime + 分布式 FUSE 惰性加载文件系统；2026-06 新增 **VM Sandboxes（Alpha）**——真实 Linux 内核，支持沙箱内 Docker-in-Docker、eBPF、systemd（对「Agent 要啥装啥」的兼容性补课）。
- **三层快照体系**：Filesystem（`snapshot_filesystem()`，产出 Image，按 base 差分、基本永久保存）/ Directory（Beta，30 天）/ Memory（Alpha，基于 gVisor checkpoint/restore——`import torch` 从 ~5s 降到 ~1.05s）；GPU Memory Snapshots 基于 NVIDIA CUDA checkpoint/restore API（部分函数启动最高快 10×）。
- 网络：`block_network`、`cidr_allowlist`、`outbound_domain_allowlist`（2026-06）、加密端口隧道、短期 connect token。
- 规模与生态：创建吞吐压测 1000/s（2025-01），报道称重构后 1 分钟启动 100 万沙箱、压测 100 万并发；SWE-bench 官方集成；**Claude Managed Agents 一等集成**（2026-05 Anthropic 自托管沙箱公测的支持方之一，单客户 10 万+ 并发）。
- 定价：按秒计费（CPU ~$0.00003942/core/s，约标准 Functions 的 3 倍）；Starter $30/月免费额度、Team $250/月。

### 3.5 Fly.io Machines / Sprites

- **Machines**：Firecracker microVM（2023 年起从 Nomad/flyd 迁移到 Firecracker），~300ms 冷启动、suspend/resume <100ms（suspend 仅支持 ≤2GiB 内存、无 GPU）；Volumes/Snapshots；6PN 私有网络（WireGuard mesh + Anycast）；GPU Machines（A100/L40S）；按秒计费。
- **Sprites（2026-01-09 发布）**——Fly 真正面向 Agent 的产品线，定位「有状态沙箱环境 + checkpoint/restore」：持久 ext4（最多 100GB，NVMe 直连 + 对象存储，只对写入的块计费）、**checkpoint ~300ms / restore ~1s**（CoW 只抓可写层，最近 5 个 checkpoint 常驻挂载在 `/.sprite/checkpoints`）、空闲自动睡眠、唯一 HTTPS URL 唤醒（warm 100–500ms / cold 1–2s）、按秒计费且空闲不计算力、原生 MCP 端点（2026-03）、预装 Claude Code/Codex/Gemini CLI；刻意不支持 GPU 与 OCI 镜像。CEO 定位语：「The age of sandboxes is over. The time of the disposable computer has come.」
- 与 Cube/E2B 的路线分歧值得注意：Sprites 主张「沙箱应该是持久的计算机」，靠 checkpoint 兜底可逆性；Cube/E2B 主张「沙箱是一次性环境」，靠模板克隆保证重建速度——两条路在 6.4 的「事件级快照回滚」上汇合。

### 3.6 Cloudflare Sandbox / Workers

- 三层产品栈：
  - **Workers / Dynamic Workers**：V8 isolates（workerd），**语言级沙箱**——每 isolate 独立内存空间，无进程/系统调用暴露，ms 级冷启动，128MB 内存上限；不能跑任意 Linux 二进制（Node.js 兼容层 + Wasm）；
  - **Browser Run**（原 Browser Rendering，2026-08 更名）：托管 headless Chrome，Puppeteer/Playwright/CDP/Stagehand 直控，浏览器自动化 Agent 无需自建浏览器集群；
  - **Containers**：OCI 容器平台（2025-06 beta → 2026-04-13 GA），在 Cloudflare 网络跑完整 Linux，需 Workers Paid 计划；共享内核隔离（非 microVM）。
- **Cloudflare Sandboxes**（2025-06 beta → 2026-04-13 GA）：以 Containers 为底座的持久隔离环境——按名字请求、未运行即启动、空闲自动 sleep、请求到来唤醒，同 ID 全球任意位置访问；官方口径冷启动约 30s vs 快照恢复约 2s；持久化 code interpreter（Python/JS 上下文跨调用保状态）；凭据经可编程 egress proxy 注入（agent 不接触 token）；PTY 真终端 + preview URL + 文件系统 watch；SDK 1.0 处于 preview。
- 定价与规模：Active-CPU 计费 $0.00002/vCPU-s；标准计划约 15,000 并发 lite 实例；生产案例 Figma Make。
- 生态：OpenAI Agents SDK 一等后端、Claude Managed Agents 模板、Project Think（`createSandboxTools`）。
- 一句话：Cloudflare 走「isolate（轻）+ container（全）」双引擎 + 边缘网络，安全模型上 isolates 最强（无内核攻击面），但「完整 Linux 环境」档位是共享内核容器，隔离强度弱于 microVM 路线。

### 3.7 Firecracker / gVisor / Kata Containers（底层原语组）

- **Firecracker**：AWS 2018 年开源的 Rust microVM VMM，Serverless（Lambda/Fargate）验证过的底座；特性 = 最小设备模型 + jailer 加固 + 快照/diff 快照；**它不是产品**，是 E2B/Fly/Vercel/Lambda MicroVMs 等产品的地基。v1.15（2026-03）起对 aarch64 持续补课（PVTime、dma-coherent、GSI 修正——见 6.5）。
- **gVisor**：Google 开源，用户态内核（Sentry）拦截 syscall + Gofer 经 9P 提供文件系统；隔离强于容器、弱于 VM，性能损耗（syscall 开销 + 9P IO）是主要代价；在 App Engine/Cloud Run/Modal/Google Agent Sandbox 大规模生产验证——**腾讯云生产环境每天运行数百万 gVisor 沙箱**（gVisor 官方博客披露），与 Cube 的 microVM 路线并存。
- **Kata Containers**：OCI 容器接口 + 轻量 VM 后端（QEMU/Cloud Hypervisor/Firecracker/StratoVirt），「K8s 原生的强隔离容器」；3.x 起 Rust 重写 runtime，VMM 池化 + 预启动内核优化启动；适合存量 K8s 体系里升级隔离强度，而非专门为 Agent 工作负载设计（arm64 上仍有集成摩擦：Kata agent 的 root-bus 探测只认 QEMU 风格 PCI 节点，Cloud Hypervisor 后端需补丁）。

### 3.8 Docker / Podman

- 生态最全、工具链最成熟、镜像供给无限；但**共享宿主内核**是硬伤——多租户跑不可信代码时，一次内核 CVE 即可横向打穿。
- 在 Agent 场景的合理位置：① 内部工具/可信场景；② OpenHands 等框架的默认本地 runtime；③ microVM 沙箱**内部**跑容器工作负载（套娃式双重隔离）。

### 3.9 Wasm 沙箱（Wasmtime 等）

- **能力模型安全**：WASI 默认零权限，目录/网络必须显式授权（capability-based），内存安全（无指针越界），µs~ms 级启动——安全与启动速度的天花板。
- 代价：**不能跑完整 Linux 生态**——需要重编译为 Wasm（WASI SDK / 组件模型），Python/系统工具的支持靠专门移植（CPython 已有 WASI 构建）；对「Agent 要啥装啥」的场景不适用。
- 合适位置：插件系统、单函数执行、边缘计算、确定性重放；趋势上正与 microVM 融合（见 6.1）。

### 3.10 浏览器沙箱（Playwright + 容器）

- 自建路线：容器（或 microVM）里跑 Chromium + Playwright/Puppeteer，浏览器自身多进程沙箱 + 外层容器双层防护。
- 痛点：浏览器启动慢（秒级）、内存大（GB 级）、浏览器 0day 是真实威胁（每年都有在野利用）；规模化需要浏览器池化/预热。
- 托管替代：E2B Desktop、Cloudflare Browser Run、Browserbase 等把浏览器栈做成服务，Agent 只需要 CDP/Playwright 协议。

### 3.11 补充：2026 年入场的云厂商与新兴项目

用户列出的 11 项之外，2026 年新增的重要玩家（多为云厂商）：

| 方案 | 隔离机制 | 关键指标 | 定位 | 状态 |
|---|---|---|---|---|
| Vercel Sandbox | Firecracker microVM（自研 Hive 平台，与构建同套基础设施） | 毫秒级启动、`Sandbox.fork()`、持久化 GA（2026-05）；Active CPU $0.128/vCPU-h | 前端/全栈生态的 Agent 沙箱 | GA（2026-01-30） |
| AWS Lambda MicroVMs | Firecracker + 快照 | 状态跨 suspend 保留 8h、独立 HTTPS 端点、**仅 ARM64（Graviton）** | 把 microVM 做成 Lambda 级原语 | 发布（2026-06-22） |
| AWS Bedrock AgentCore | EC2 托管（Runtime Instances / Code Interpreter） | 会话最长 14 天、Code Interpreter 启动约 100ms 级、每会话独立容器化 microVM | Bedrock Agent 的一等运行时 | GA（2026-08-06） |
| Google Agent Sandbox（GKE） | gVisor（GKE Sandbox） | 300 sandbox/s、亚秒延迟、TTL 最长 14 天、出站默认阻断 | 开源 K8s SIG Apps 子项目，任意 K8s 可跑 | 发布（2026-04，2026-07 起计费） |
| microsandbox | libkrun microVM（自托管、rootless） | <100ms 启动、OCI 镜像、MCP server | 开源自托管沙箱（Apache-2.0） | beta |
| OpenSandbox（阿里） | Docker/K8s 编排层（可挂 gVisor/Kata/Firecracker 安全运行时） | 批量创建 100 个约 0.92s（Pool/BatchSandbox CRD）；统一入口网关 + 逐沙箱出口控制 | 国内开源方案（横评定位：抽象层而非新隔离底座） | 开源（Apache-2.0，2025-12） |
| AgentENV（Moonshot/kvcache-ai） | Firecracker microVM 集群 + overlaybd 按需 OCI 镜像 | boot/resume <50ms、pause/增量快照 <100ms、单沙箱同节点 fork ≤16 | Kimi K3 的 agentic RL 训练底座（E2B 兼容 API） | 开源（MIT，2026-07） |

收购/整合信号：Baseten 收购 Blaxel（2026-09，microVM 沙箱 suspend/resume 约 25ms）；OpenAI 收购 Ona（原 Gitpod，2026）——沙箱正在成为 AI 基础设施厂商的收购标的。

其中 **AgentENV** 值得单独一提：它是目前**唯一以「RL 训练」为第一场景**的开源沙箱基础设施（Kimi K3 的 agentic RL 训练底座），且**对外暴露 E2B 兼容 API**——与 Cube 同属「microVM + E2B 协议」路线，但目标负载聚焦训练：同节点 fork ≤16 个子沙箱直接服务树状 rollout / 分支采样（与 4.6 的 BPO 类算法同构）。部署要求 Linux 6.8+ / KVM。

---

## 四、典型应用场景

### 4.1 AI 代码解释器 / 数据分析 Agent

- 形态：模型生成 Python → 沙箱执行 → 返回结果/图表（ChatGPT Code Interpreter、Claude 数据分析、E2B Data Analysis 沙箱、Cloudflare 持久化 code interpreter）。
- 需求特征：**短任务高频执行**（一次对话几十次 exec）、常用库预热（模板里装好 pandas/numpy/matplotlib，省掉每次冷启动 import 时间）、产物导出（图/CSV 落到对象存储）。
- 安全要点：用户上传的数据文件可能含注入（CSV 公式注入、恶意文件），沙箱 + 超时 + 资源限制是标配。

### 4.2 自主编程 Agent（Devin、OpenHands、Claude Code）

- 形态：长时运行的会话（小时级），需要 shell + 编辑器 + git + 浏览器全套，产出真实代码仓库变更。
- 需求特征：**状态持久化**（工作目录、git 状态不能丢）、**可观察性**（回放执行历史）、**网络分级**（git 推送要放行、扫描内网要拦截）。
- 落地现状：OpenHands 默认 runtime 是 Docker（V0 时代曾通过 plugin 支持 E2B/Daytona/Modal 等远程 runtime，2026-04 V1 迁移时**移除了全部第三方 runtime**、改为自研 Sandbox API）；Devin 自建 VM 基础设施、Devin Outposts 以 E2B/Daytona 为执行层（2026-07）；Cursor 提供 Self-Hosted Machines（E2B/Daytona 模板）；Claude Code 默认跑在用户环境，靠权限模型约束，企业场景建议容器/沙箱化部署（2026-06 Anthropic 发布 Claude Sandboxes + MCP Tunnels）。
- 为什么不能裸跑用户机器：仓库代码、SSH key、`.env` 全在信任域内，一次恶意命令（哪怕是无意的 `rm -rf`）就是事故。

### 4.3 浏览器自动化 Agent

- 形态：Agent 操作浏览器（点击/填表/爬取），需要完整浏览器栈 + CDP/Playwright 接口。
- 需求特征：浏览器生命周期管理（启动/崩溃恢复）、截屏与 DOM 流（给模型看）、下载内容的安全处理（下载的文件先进隔离区）。
- 分层防护：浏览器进程沙箱（防网页打浏览器）→ 容器/microVM（防浏览器打系统）→ egress 管控（防浏览器偷凭据外发）。
- 托管选项：Cloudflare Browser Run（Playwright MCP / Stagehand 直控）、E2B Desktop、Browserbase。

### 4.4 多 Agent 协作中的工具执行环境

- 粒度选择：**每 Agent 一沙箱**（强隔离，互相不污染，但共享数据要靠对象存储/共享卷）vs **共享沙箱 + 内部权限分层**（协作直接，出事故互相牵连）；Cube 的 clone(N)、Vercel 的 fork() 提供了「分叉式协作」的新选项——从同一个环境状态派生出多个独立探索分支。
- 编排层（LangGraph、crewAI 等）通过 sandbox provider 统一创建/销毁/暂停沙箱，把沙箱当成可调度资源池；沙箱间通信走受控网络（私有网段 + 白名单）。

### 4.5 强化学习 / 评测环境（Agentic RL）

**评测侧**：基准的执行环节必须隔离——跑的是模型生成的代码，评测平台不敢在裸宿主上执行：

- SWE-bench：每个 instance 一个独立 Docker 镜像（`swebench/sweb.eval.x86_64.*`），经 SandboxSpec 三级机制解析，生命周期为 seed → solve(sandbox) → verify(sandbox) → release；
- KernelBench：在 Docker sandbox 内评分且 `network_mode: none`，只拷贝 `eval_runner.py` 进容器以最小化 reward hacking 面；
- OSWorld：靠 VM 快照恢复实现任务间重置，基础设施从本地 VM 迁到 AWS 后约 50× 并行加速；
- SWE-rebench 的量化研究：OS 层执行（工具调用、容器/Agent 初始化）占端到端延迟的 56–74%；**内存（而非 CPU）是并发瓶颈**（工具调用内存峰值达平均值的 15.4 倍）——需要 cgroup/eBPF 级内核态资源控制。

**RL 训练侧**：每个 rollout 一个干净环境，训练吞吐 = 环境重置速度 × 并发沙箱数。2026 年公开案例：

- **MiniMax Forge Agent RL** 跑在腾讯云 Agent Runtime（Cube）：百万级吞吐、十万级并发，30,000 个 4C8G 沙箱 58 秒拉起，单镜像每分钟 60 万沙箱，交付约 80ms（较冷启动降 96%），镜像按需加载使实际读取降至全量的约 10%；
- **Moonshot AgentENV**（2026-07 开源，MIT，用于 Kimi K3 的 agentic RL）：Firecracker microVM 集群 + overlaybd 按需 OCI 镜像，boot/resume <50ms、pause 与增量快照 <100ms、单沙箱可同节点 fork 16 个子沙箱，**暴露 E2B 兼容 API**；
- 快手 KwaiEnv（数万并发沙箱）；Modal 定位 tool-use/terminal-agent RL（gVisor + snapshot/restore，100 万并发压测）；Dressage/SkyRL/OpenClaw-RL 等框架提供可插拔沙箱后端（bubblewrap 本地 / E2B 远程）。

**共性工程模式**：~100ms 级快照恢复、资源池化（池化替代逐任务供给）、镜像去重与按需加载、单机数千沙箱密度、E2B SDK 兼容——这正好是 microVM 沙箱的核心竞争力清单，也是 RL 场景为什么不用「docker run + 环境初始化」的结构性原因（快照克隆比冷启动+初始化快 1~2 个数量级）。

### 4.6 沙箱后训练（Sandbox Post-Training）——方法侧

> 4.5 讲「谁在用什么规模的沙箱」，本节讲「后训练方法怎么把沙箱本身当成训练原语」。（2026-09-12 补充调研）

**为什么沙箱成为后训练（RL）的默认执行环境**：RL 需要可验证奖励，而「代码在沙箱里跑测试用例」给出的奖励零噪声、零 API 成本、reward hacking 空间最小（RLVR 范式）——沙箱是 RLVR 不可替代的执行层。2026 年该方向密集产出：

- **LLM-in-Sandbox / LLM-in-Sandbox-RL**（arXiv:2601.16206）：把 LLM 放进轻量 Python 沙箱自主探索（文件系统/脚本/网络/状态持久化）；关键发现——**仅用非具身 SFT 数据（UltraFeedback + self-instruct）构造奖励**就能稳定提升规划与工具调用能力，无需昂贵的人工「思维-动作」轨迹标注；沙箱启动 <100ms、单次交互内存 <80MB、并发沙箱池，已开源为 PyPI 包 `llm_in_sandbox` 与 Qwen3-4B-Instruct-2507 训练 checkpoint。
- **BPO：Branching Policy Optimization**（arXiv:2607.14171）——「沙箱原生」的 agent RL：**直接利用沙箱 checkpoint/restore 构建树状 rollout + sibling-baseline advantage**（有可证明的方差缩减）；WebShop/ALFWorld/SWE-bench 全面超过 PPO/RLOO/GRPO/VinePPO（SWE-bench +4.7），达到基线最终性能所需梯度步数减少约 38.7%、墙钟时间降 35–40%。**这是「事件级快照回滚」直接变成训练算法的案例**——树状探索的前提是廉价的 checkpoint/restore，正是 Cube v0.3.0 clone(N)/原地回滚提供的原语。
- **ReTool / Tool-Call RL**（THUDM slime）：SFT（学会何时进沙箱验证计算、测试假设）→ RL（优化求解）两阶段，基于 verl/slime，Qwen2.5-32B/Qwen3-4B 在 AIME 2024 上支持 GRPO、PRM+RL 与多节点训练。
- **DeepCoder（rLLM 框架）**：模型生成候选程序 → 沙箱跑测试 → 结果形成奖励 → GRPO 更新；DeepCoder-14B 在 LiveCodeBench 达 60.6% Pass@1（对齐 o3-mini 水平）、DeepScaleR-1.5B 在 AIME 2024 达 43.1%、DeepSWE-32B 在 SWE-bench Verified 达 59%。
- **Wasm 沙箱训练**（axolotl + GRPO）：用 Wasm 运行时本地安全执行不可信 Python（「fuel」资源配额机制），多进程异步奖励计算约 10× 提速——Wasm 在「训练期奖励计算」这个细分场景找到了与 microVM 互补的位置（见 6.1）。
- **AZRL（Absolute Zero RL）**：零外部数据的自博弈推理框架，模型自任提议者/求解者、在安全沙箱验证 Python 代码，TRR++/PPO 更新。
- **环境接口标准化**：OpenEnv（`reset()/step()/state()` 的 client-server 接口，容器 provider 覆盖 Docker/UV/Daytona/K8s）、Dressage（bwrap/E2B/K8s 可插拔）、AWS AgentCore 的 ART RL Toolkit、RLVE（400 个程序化可验证环境）。

**后训练对沙箱的需求清单**（与 4.5 互补）：

1. **checkpoint/restore 是一等原语**——BPO 的树状 rollout 直接以沙箱快照为 API；
2. **确定性执行 + 资源硬限**：超时（如 120s）、内存（如 4GB）、模块白名单、危险模式检测（拦截 os/sys/subprocess/`__import__`）——防 reward hacking 也防训练事故；
3. **高并发短生命周期**：<100ms 创建 + 并发沙箱池（SandboxFusion 本地 128 并行 worker、LLM-in-Sandbox 的并发池）；
4. **框架生态**：verl/slime/rLLM/OpenEnv/ART 都在抽象 sandbox backend——沙箱厂商提供官方 backend 插件，是进入训练工作负载的入口。

**对 Cube 的含义**：<60ms 创建 + clone(N) + 原地回滚 + MiniMax 生产验证（4.5）已覆盖 BPO 类算法需要的原语；缺口在生态侧——为 verl/slime/OpenEnv 提供官方 backend、沉淀「训练级」资源控制模板（超时/内存/白名单组合策略），是把「沙箱」卖进「后训练」工作负载的关键。

---

## 五、风险与挑战

### 5.1 沙箱逃逸与供应链风险

- 逃逸是概率问题不是有无问题：microVM 逃逸面（2.6 节清单）需要**持续的 CVE 跟踪与镜像更新机制**，自托管团队必须有人盯 guest 内核/KVM/VMM 的安全公告；
- 供应链：基础镜像、依赖包、MCP 工具都是投毒入口，隔离只能限爆半径——所以「凭据不进沙箱」是第一原则；
- 逃逸后的止损：宿主横向隔离（不同租户不同宿主池）、审计日志、快速吊销。

### 5.2 冷启动延迟

- 「冷启动」在 Agent 场景 = 模板缓存未命中时的一次真实 boot（秒级~十秒级）+ 依赖安装（分钟级）——对交互式 Agent 体验是灾难；
- 实测数据谱系：Cube 60ms（行业均值约 150ms）；Cloudflare Sandbox 快照恢复约 2s vs 冷启动约 30s；Fly Sprites 唤醒 warm 100–500ms / cold 1–2s；AWS Lambda MicroVM 启动约 10s 但 suspend/resume 约 2.6s（第三方实测）——**快照热启动是当前最有效的优化杠杆，模板缓存命中率决定 P50 体验**；
- 对策组合：模板预 boot + 快照恢复（把慢路径挪到构建期）、常用模板常驻热缓存、模板瘦身（只装基础层，重依赖用 volume）、同机克隆复用（FICLONE）。

### 5.3 状态管理与持久化难题

- 内存快照的跨节点迁移成本（快照文件可能几百 MB~GB，恢复节点无本地缓存时要拉取）；
- 「沙箱销毁即丢数据」与「用户以为数据还在」之间的语义鸿沟——需要清晰的持久化分层（2.2 节）与产品提示；Fly Sprites 用「默认持久 + checkpoint 可回滚」绕开这个问题，但换来磁盘计费；
- 事件流/会话状态的持久化与重放（用于审计、断点续跑、回滚）；
- 快照格式与 VMM 版本的兼容性（升级 VMM 后旧快照能否恢复——自研/自托管必须自己维护）。

### 5.4 多租户下的资源争抢与计费

- 超卖 vs 预留的权衡：超卖提升毛利但制造噪音邻居（同一宿主的沙箱互相抢 CPU/内存带宽）；QoS 需要 cgroup 细粒度调控（Cube 的 NUMA 池 + 叶子 cgroup 管理）；
- **计费模型在 2026 年明显分化**：Active-CPU 计费（Cloudflare/Vercel：只为实际执行计费）、按秒计费 + 空闲不计（Fly Sprites/Modal）、状态保留 TTL 计价（AWS Lambda MicroVMs 状态保留 8h、Bedrock AgentCore 与 Google 最长 14 天）——「暂停省成本」与「恢复要状态」的耦合成为产品设计点；
- 账单不可预测性：Agent 循环失控（死循环调用工具）会烧钱——超时、配额、熔断是刚需。

### 5.5 合规与数据隐私

- 多租户下跑用户代码 = 处理用户数据：数据驻留（region 约束）、审计日志留存、隔离强度证明（SOC 2/等保）；
- 模型提供商与沙箱提供商的信任边界：用户数据在「模型 → 沙箱 → 用户」链路中的每一跳都要有明文约定（Claude Managed Agents 的「agent loop 在 Anthropic、执行在客户自托管沙箱」模式是边界清晰的样板）；
- 金融/医疗等强监管场景倾向自托管/专有部署（也是 Cube、E2B self-host 的市场逻辑）。

### 5.6 自托管方案的隐性成本

自托管（E2B infra Terraform、自建 Cube 集群）买的是**控制权与单位成本**，付的是**人力与稳定性**，容易被低估的成本项：

1. **运维值班**：7×24 值班体系，故障响应 SLA；
2. **内核更新**：guest 内核与宿主内核的 CVE 跟踪、灰度升级、滚动迁移（升级宿主内核 = 迁移所有沙箱）；
3. **集群故障转移**：节点宕机时沙箱状态丢失，需模板快速重建 + 用户会话续接机制；
4. **快照/镜像供应链**：模板构建流水线、依赖安全扫描、镜像仓库存续；
5. **容量规划**：沙箱密度调优（单机可跑上千 microVM，但内存带宽/网络是约束）、峰值突发扩容（RL 场景的十万级并发不是裸机堆得起的，调度系统本身就是核心资产）；
6. **安全攻防**：自己承担逃逸防护（2.6 全清单）。

一句话：**托管沙箱买的是「别人替你熬这些」**；自托管只有在这支团队本来就有虚拟化/内核运维能力（如腾讯云自身）时才是划算的。

---

## 六、趋势与展望

### 6.1 microVM + Wasm（/isolate）的融合

- 双引擎分工在 2026 年清晰化：**microVM 管「重负载」**（完整 Linux、任意二进制、GPU），**Wasm/V8 isolates 管「轻计算」**（插件、单函数、确定性执行、边缘）——同一平台按负载路由到不同引擎：
  - Cloudflare 是最完整的商业样板：V8 isolates（Workers/Dynamic Workers）+ Containers/Sandboxes 双轨，并新增 Dynamic Workflows；
  - Modal 反向补课：gVisor 之外 2026-06 增加 VM Sandboxes（Alpha，真内核、支持 Docker-in-Docker）；
  - AWS 把 microVM 原语化：Lambda MicroVMs（Firecracker + 快照 + 独立端点）；
  - 腾讯云同样两条腿：Cube（microVM）+ gVisor 沙箱（生产环境每日数百万）。
- 判断：不会互相替代——Wasm 解决不了「Agent 要 apt install 一切」，microVM 解决不了 µs 级启动与毫秒级边缘分发；融合为「按负载路由」的多引擎平台是主流叙事，Wasm 组件模型在模板/工具分发上的可移植性有增量价值。

### 6.2 Agent 沙箱协议 / 标准：E2B SDK 正在成为事实标准

- **协议级兼容是「事实标准」的最硬证据**：
  - 腾讯云 CubeSandbox README 直接打 "API: E2B Compatible" 徽章（改环境变量迁移）；
  - **阿里云 FC Agent Sandbox 不提供自有 SDK/OpenAPI，E2B SDK/CLI 兼容是唯一接口**（仅高级项 Snapshot/Volume 等不兼容）；
  - Moonshot AgentENV 暴露 E2B 兼容 API；ComputeSDK/lelantos 复用 e2b npm SDK 对接自建 Firecracker 控制面；NVIDIA NeMo Gym 的 `api_url` 可指向「任意 E2B-compatible gateway」——协议已成为可替换的接口层。
- **一线框架与 Agent 产品内置集成（仓库级实证，2026-09）**：OpenAI Agents SDK 官方内置 `E2BSandboxClient`（与 Blaxel/Cloudflare/Daytona/Modal/Runloop/Vercel 并列 7 家 provider）；Google ADK `integrations/e2b`；LangChain / Agno / CrewAI / HuggingFace smolagents / AutoGPT 内置 E2B 工具；Claude Managed Agents self-hosted 环境、Cursor Self-Hosted Machines、Devin/Devin Outposts 以 E2B 为执行层。规模口径：1B+ 沙箱启动、10M+ SDK 月下载、94/100 Fortune 100 签约；HVTracker 信任分 78.4 居沙箱类第一。
- **「标准」的边界（同样重要）**：OpenAI Agents SDK 同时内置 7 家 provider（E2B 是默认之一而非唯一）；**OpenHands 2026-04 已移除 E2B 等第三方 runtime**（V1 自研 Sandbox API）；Cline 无 E2B 集成；Codex CLI 不接受第三方沙箱替换；AWS 侧是竞品（AgentCore）而非集成方——「事实标准」是接口层的既成事实，不是排他协议。
- **原因与风险**：开源早（2025-04 全开源、runtime 持续活跃）、接口贴近 Agent 工作负载（不是「虚拟机 API」而是「执行环境 API」）、云/自托管双模式；风险——事实标准≠正式标准，E2B 的演进方向直接影响下游兼容成本；云厂商自有 SDK（Vercel/Cloudflare/AWS）与 MCP 工具协议也在争夺接入层，长期看「sandbox-as-tool」可能经 MCP 二次标准化。

### 6.3 与 MCP、工具调用协议的结合

- **现状**：MCP 规范本身不含沙箱（2025-06/2025-11 两版 changelog 均无沙箱条目）；stdio 本地 server 作为子进程运行、继承启动用户的全部权限——恶意 `npx` 包可无提示读取 SSH 密钥、`~/.aws` 凭证、浏览器 cookie。
- **攻击面已成体系**：OWASP MCP Top 10（MCP03:2025 tool poisoning）；Deadbugz 元数据投毒（2026-08 发现活跃攻击：23 个 GitHub PR 诱导 agent 窃取 SSH key/AWS 凭证/k8s 配置）；Windsurf 零点击 prompt injection（CVE-2026-30615）；Anthropic 官方 Git MCP server 三连 CVE（2025-12 修复）；MCPTox 基准平均攻击成功率 36.5%。
- **业界响应四条线**：① MCP server 分级治理（第三方 server 强制沙箱 + 版本锁定）；② 容器加固参数成为社区模板（`--read-only --cap-drop ALL --network none` 等，约 90% 团队适用的默认档）；③「网关 + 沙箱」双屏障架构（网关管「谁能调什么」，OS 级沙箱管「真能做什么」）；④ 沙箱产品原生 MCP 化：Fly Sprites MCP 端点（agent 自己创建/管理沙箱）、microsandbox MCP server、Anthropic 2026-06 发布 Claude Sandboxes + MCP Tunnels、Google Agent Sandbox 的 OSS MCP server。
- **结论**：MCP 把「工具宿主」标准化后，**沙箱成为协议边界上的安全锚点**——「MCP server 跑在沙箱里」从社区最佳实践走向产品特性，工具调用协议与沙箱 API 正在收敛。

### 6.4 事件级快照回滚能力的普及

- 从「创建时快照」（模板恢复）演进到「**事件级快照**」：每步工具调用后打点，支持**任意步回滚、会话分叉、审计重放**。2026 年已全面产品化：
  - **Cube**：2026-04 官方宣布「毫秒级事件级快照与状态回滚」规划，v0.3.0 CubeCoW 把 snapshot/clone/rollback 做成一等 API——快照 ID 即模板、clone(N) 一行分叉、**原地回滚保持 sandbox_id 无需重连**，恢复从分钟级降到百毫秒级；
  - **Fly Sprites**：checkpoint ~300ms，最近 5 个 checkpoint 常驻挂载（`/.sprite/checkpoints`）；
  - **Vercel Sandbox**：`Sandbox.fork()` + 快照派生 + 持久化（2026-05 GA）；
  - **Cloudflare Sandboxes**：会话快照与 fork；
  - **开源/学术**：Shepherd（把执行历史当 Git 式版本化对象，OS 级沙箱 fork 比 Docker 快 5×）、DeltaBox（ms 级 checkpoint/rollback 论文）、Agent VCR / Smithers / AgentOS（replay/fork/resume 的 time-travel 工具）、StateWeave（agent 状态的 git）、AgentRewind（上下文记录 + 环境检查点恢复）。
- **价值场景**：Agent 跑挂环境后回到上一个干净状态（替代销毁重建，省分钟级时间）、RL 训练的精确重置、多 Agent 协作的试错分叉、time-travel 调试与审计重放。
- **技术前提**：ms 级内存快照 + CoW 零拷贝存储 + 事件流持久化——microVM 路线天然占优（内存快照本就是它的启动机制，Cube 的 EPT Lazy Load/soft-dirty 增量快照直接复用）。
- 判断：**2026–2027 年「回滚」将从运维能力变成 Agent 编程原语**（回到上一步重新采样），与 6.2 的协议收敛叠加，成为沙箱产品的标配卖点。

### 6.5 ARM64 vs x86 性能生态支持对比

- **虚拟化底座**：Firecracker v1.15（2026-03）对 aarch64 持续补课（guest 缓存目录缺失修复、dma-coherent、PVTime steal time、GSI 编号修正后最多 92 个设备）；Cloud Hypervisor 的 arm64 是一等公民（官方 arm64 内核/云镜像文档齐全）；gVisor arm64 生产可用但仍有长尾问题（Raspbian 39-bit VA 配置、epoll_pwait 死循环等）。
- **产品化信号（2026 年 ARM 明显升温）**：AWS Lambda MicroVMs **仅 ARM64（Graviton）**；Google GKE Agent Sandbox 跑在 Axion 上性价比最高提升 30%；AWS 官方博客实测 agentic RL 沙箱层（Python 沙箱/浏览器/SQL 仓库，CPU-bound fan-out fleet）从 m7i 迁 Graviton4 成本降约 30%、Graviton5 降约 41%；**Cube v0.5.0 原生 ARM64 全栈支持**（腾讯云 IaaS 团队与 Arm 工程团队联合开发，20+ commits 合入主线，架构专属 guest kernel config）。
- **成本账**：腾讯云 SR1（Ampere Altra，2021 年首款 ARM 实例）对比 x86 标准型算力性价比最高提升 83%（腾讯云口径）；阿里倚天 710 在 Java/Python/Nginx 负载提升约 30%。
- **生态差距在 guest 里**：x86 仍是默认目标——大量 dev 工具/预编译 wheel/OCI import 路径 x86-first（甚至有沙箱工具在 arm64 上直接硬失败）；GitHub Agentic Workflows 收敛到 Cloud Hypervisor 但仍仅 x86_64（preview）；Kata arm64+CH 的 root-bus 集成摩擦是缩影。
- **策略判断**：双架构支持 = 双倍镜像供应链成本（模板要双架构构建/验证）；务实路线是「**ARM 承载规模化自营负载**（RL/成本敏感、镜像可控），**x86 兜底长尾生态**（兼容性优先）」；对外宣称的架构支持矩阵（哪些功能仅 x86_64——如 Cube 的 PVM 嵌套虚拟化/live migration）将成为沙箱产品的选型维度之一。

---

## 参考来源

**Cube / 腾讯云**
- [TencentCloud/CubeSandbox（GitHub）](https://github.com/TencentCloud/CubeSandbox) · [开源新闻稿（腾讯云国际站）](https://intl.cloud.tencent.com/dynamic/news-details/101123) · [品玩：汤道生披露开源计划与五大突破](https://www.pingwest.com/a/313176) · [v0.5.0 changelog（ARM64/AutoPause/网络策略）](https://raw.githubusercontent.com/TencentCloud/CubeSandbox/master/docs/changelog/v0.5.0.md) · [CubeCoW v0.3.0 官方博客](https://www.tencentcloud.com/dynamic/blogs/sample-article/101259) · [Cube 架构笔记（skyyao）](https://skyao.net/learning-ai-agent/infra/environment/sandbox/cubesandbox/_print/) · [ARM64 联合开发（腾讯云开发者社区）](https://cloud.tencent.com.cn/developer/article/2707492) · [MiniMax Forge Agent RL 实践](https://cloud.tencent.cn/developer/article/2642705)

**评测与横评**
- [AI Agent Sandbox 云产品横评（pondero, 2026-07）](https://pondero.ai/agents/guides/best-ai-agent-sandbox-cloud-comparison-july-2026/) · [中文横评：CubeSandbox vs E2B vs OpenSandbox vs Daytona（CSDN）](https://devpress.csdn.net/wuhan/6a4efc4a662f9a54cb8c5d01.html) · [HVTracker 沙箱信任分](https://hvtracker.net/categories/sandboxes-runtimes/) · [第三方 Cube 实测（devpress）](https://devpress.csdn.net/wuhan/6a4efc4a662f9a54cb8c5d01.html)

**E2B / Daytona**
- [e2b.dev](https://e2b.dev) · [e2b-dev/runtime（GitHub）](https://github.com/e2b-dev/runtime) · [E2B 定价](https://e2b.dev/pricing) · [E2B Series A 官宣](https://e2b.dev/blog/series-a) · [阿里云 FC Agent Sandbox 的 E2B 兼容说明](https://www.alibabacloud.com/help/en/functioncompute/e2b-compatibility-description) · [Daytona Series A](https://www.daytona.io/dotfiles/daytona-raises-24m-series-a-to-give-every-agent-a-computer) · [Daytona 闭源公告（2026-06）](https://www.daytona.io/dotfiles/updates/daytona-is-going-closed-source) · [OpenHands 移除第三方 runtime（PR #14119）](https://github.com/OpenHands/OpenHands/pull/14119) · [OpenAI Agents SDK sandbox 扩展（7 家 provider）](https://github.com/openai/openai-agents-python/tree/main/src/agents/extensions/sandbox)

**Modal**
- [Sandboxes GA 博客](https://modal.com/blog/sandbox-launch) · [内存快照](https://modal.com/blog/mem-snapshots) · [VM Sandboxes 更新（2026-06）](https://modal.com/blog/product-updates-vm-sandboxes-domain) · [Claude Managed Agents × Modal](https://modal.com/blog/introducing-claude-managed-agents-with-modal-sandboxes) · [定价](https://modal.com/pricing) · [Modal 谈 RL 沙箱选型](https://modal.com/resources/best-sandboxes-rl-environments)

**Fly.io**
- [Machines 文档](https://fly.io/docs/machines/) · [sprites.dev](https://sprites.dev/) · [Simon Willison 评测 Sprites（2026-01）](https://simonwillison.net/2026/Jan/9/sprites-dev/) · [suspend/resume 参考](https://fly.io/docs/reference/suspend-resume/)

**Cloudflare**
- [Sandbox GA 博客（2026-04）](https://blog.cloudflare.com/sandbox-ga/) · [Sandbox 文档](https://developers.cloudflare.com/sandbox/) · [Containers 文档](https://developers.cloudflare.com/containers/) · [Browser Run 文档](https://developers.cloudflare.com/browser-run/) · [InfoQ 报道](https://www.infoq.com/news/2026/04/cloudflare-sandboxes-ga/)

**云厂商新入场者**
- [Vercel Sandbox GA](https://vercel.com/blog/vercel-sandbox-is-now-generally-available) · [Vercel Sandbox 文档/定价](https://vercel.com/docs/vercel-sandbox) · [AWS Lambda MicroVMs 发布](https://aws.amazon.com/cn/about-aws/whats-new/2026/06/aws-lambda-microvms/) · [AWS Bedrock AgentCore Runtime Instances GA](https://aws.amazon.com/cn/about-aws/whats-new/2026/08/aws-bedrock-agentcore-runtime-instances-generally-available/) · [Gemini Enterprise Agent Platform](https://cloud.google.com/blog/products/ai-machine-learning/introducing-gemini-enterprise-agent-platform) · [GKE Agent Sandbox（Next '26）](https://cloud.google.com/blog/products/containers-kubernetes/whats-new-in-gke-at-next26/) · [microsandbox（GitHub）](https://github.com/dwongdev/microsandbox) · [OpenSandbox（GitHub，阿里开源）](https://github.com/alibaba/OpenSandbox)

**底层技术**
- [Firecracker FAQ/发布页](https://github.com/firecracker-microvm/firecracker) · [gVisor 官方博客（腾讯生产规模披露）](https://gvisor.dev/blog/index.xml) · [Kata arm64+CH 集成摩擦](https://github.com/AlexanderMattTurner/agent-glovebox/pull/6229)

**Agentic RL / 评测**
- [AgentENV（Moonshot，Kimi K3 RL）](https://www.opensourceforu.com/2026/07/moonshot-ai-kvcache-ai-open-source-agentenv-to-scale-agentic-reinforcement-learning/) · [AWS Graviton 优化 agentic RL 沙箱层](https://aws.amazon.com/cn/blogs/china/graviton-optimize-agentic-rl-layer-architecture-cost-analytics/) · [KernelBench sandbox PR（inspect_evals）](https://github.com/UKGovernmentBEIS/inspect_evals/pull/1159) · [SWE-bench Sandbox 架构（NVIDIA NeMo）](https://docs.nvidia.com/nemo/evaluator/architecture/sandbox)

**沙箱后训练**
- [LLM-in-Sandbox（BAAI Hub / arXiv:2601.16206）](https://hub.baai.ac.cn/paper/f14623a6-fc56-47a0-a028-0b1615e544ab) · [BPO：Branching Policy Optimization（arXiv:2607.14171）](https://arxiv.org/abs/2607.14171) · [ReTool（THUDM slime）](https://thudm.github.io/slime/_examples_synced/retool/README.html) · [OpenEnv（GitHub）](https://github.com/rycerzes/OpenEnv) · [Wasm 沙箱训练（axolotl×HuggingFace）](https://huggingface.co/blog/axolotl-ai-co/training-llms-w-interpreter-feedback-wasm)

**MCP 安全**
- [MCP 规范 changelog（2025-11-25）](https://modelcontextprotocol.io/specification/2025-11-25/changelog.md) · [CSA Deadbugz 研究笔记（2026-09）](https://labs.cloudsecurityalliance.org/research/csa-research-note-deadbugz-mcp-metadata-poisoning-20260902-c/) · [MCP tool poisoning 研究（lyrie.ai）](https://lyrie.ai/research/research/mcp-stdio-tool-poisoning-rugpull-150m-downloads) · [MCP server 安全清单（safeguard.sh）](https://safeguard.sh/resources/blog/securing-model-context-protocol-mcp-servers)

**快照回滚 / time-travel**
- [DeltaBox 论文（ms 级 checkpoint/rollback）](https://www.semanticscholar.org/paper/DeltaBox%3A-Scaling-Stateful-AI-Agents-with-Sandbox-Dong-He/b1660afbd7c1ed0789cb6f7279530d08047d47ad) · [Shepherd（Northeastern+Stanford）](https://www.opensourceforu.com/2026/08/northeastern-stanford-shepherd/) · [agent-vcr（GitHub）](https://github.com/ixchio/agent-vcr) · [smithers（GitHub）](https://github.com/smithersai/smithers)

**收购/整合**
- [Baseten 收购 Blaxel](https://www.baseten.co/blog/blaxel-is-joining-baseten-to-build-the-future-of-agentic-cloud/) · [OpenAI 收购 Ona](https://www.dutchstartup.ai/en/news/openai-mikt-op-lang-draaiende-agents-met-overname-van-ona)

## 延伸笔记

- [CubeSandbox 架构讲解](./cube-sandbox-learning-notes.md)——Cube 控制面/数据面全链路（E2B SDK 兼容、<60ms 创建、eBPF 网络）
- [模板创建 & 沙箱启动全链路](./cube-template-creation-flow.md)——模板 4 条制备途径、ext4+内存快照、CLM 接管
- [Cube 沙箱 cgroup 绑核笔记](./cube-cgroup-cpu-binding.md)——宿主/沙箱双层资源限制实操
- [Cube 性能测试](./cube-performance-testing.md)、[pvspinlock 性能测试](./cube-pvspinlock-perf-testing.md)——沙箱内性能评估方法
- [openEuler guest 内核](./cube-openeuler-guest-kernel.md)——ARM64 guest 内核支持
