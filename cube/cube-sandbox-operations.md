# CubeSandbox 沙箱操作速查

> 按「操作场景 → 用什么工具/命令」组织的速查手册。
> 命令与参数均提取自源码（Cubelet/cmd/cubecli、CubeMaster/cmd/cubemastercli、CubeOps/cmd/cubeopscli、openapi.yml）。
> 目录结构见 [[cube-deploy-directory-layout]]，性能测试见 [[cube-performance-testing]]。

## 1. 工具与入口总览

四个操作入口：

| 工具 | 定位 | 默认地址 | 全局参数 |
|---|---|---|---|
| `cubecli` | 节点侧（直连 cubelet gRPC） | `/data/cubelet/cubelet.sock` | `--address/-a`、`--timeout`、`--namespace/-n`、`--debug` |
| `cubemastercli` | 控制面（直连 cubemaster HTTP） | `0.0.0.0:8089` | `--address/-a`、`--port/-p`、`--timeout`(35s) |
| `cubeopscli` | 节点运维（直连 CubeOps） | `127.0.0.1:3010` | `--a/--address`(逗号分隔多 IP)、`--p/--port`、`--timeout` |
| CubeAPI | HTTP API（E2B 兼容，SDK 走这里） | `http://<node-ip>:3000` | — |

顶层命令组（cubecli）：`cubebox`(b)、`container`(c/ctr)、`exec`、`image`(i)、`volume`(v)、`storage`(s)、`network`(n)、`vm`、`meta`(m)、`unsafe`(u)、`version`、`multirun`、`containerd-ctr`。

「什么操作该用哪个工具」速查矩阵：

| 操作 | cubecli | cubemastercli | cubeopscli | CubeAPI |
|---|---|---|---|---|
| 创建/管理沙箱实例 | ✓(create/list/exec/logs) | ✓(list/info/destroy) | | ✓ |
| 模板管理 | | ✓(template) | | ✓(/templates) |
| 快照/回滚 | ✓(debug 级) | ✓(snapshot/rollback) | | ✓ |
| 镜像管理 | ✓(image) | | | |
| 节点隔离/下线 | | | ✓(node) | |
| 压测 | ✓(multirun) | ✓(multirun) | | cube-bench |

### 1.1 连接所需环境变量（客户端）

SDK（`sdk/python/cubesandbox/_config.py`）与示例统一从环境变量读取连接配置：

| 变量 | 用途 | 默认/说明 |
|---|---|---|
| `CUBE_API_URL` | CubeAPI 地址（Cube 自家 SDK 读这个） | `http://127.0.0.1:3000` |
| `E2B_API_URL` | 同上，E2B SDK 兼容名（quickstart 的 `.env.example` 用这个） | 无 |
| `CUBE_API_KEY` | API key（Cube 自家 SDK 读这个） | 未启用 auth 时留空；发 http 到非回环地址时会告警 |
| `E2B_API_KEY` | 同上，E2B SDK 兼容名 | auth 关闭时**任意非空字符串**即可（如 `e2b_000000`） |
| `CUBE_TEMPLATE_ID` | 要启动的沙箱模板 ID | `create-from-image` 之后获得（见 §3） |
| `SSL_CERT_FILE` | 指向 mkcert 根证书，使用 `cube.app` 内建证书的 https 时需要 | 如 `/root/.local/share/mkcert/rootCA.pem` |
| `CUBE_PROXY_NODE_IP` | 代理节点 IP（SDK 内部用） | 无 |
| `CUBE_PROXY_PORT_HTTP` | 代理 HTTP 端口（SDK 内部用） | `80` |
| `CUBE_SANDBOX_DOMAIN` | 沙箱域名后缀（SDK 内部用） | `cube.app` |

典型 `.env`（示例脚本会自动加载脚本旁的 `.env` 或 cwd 的 `.env`，不覆盖已存在的进程环境变量）：

```bash
export E2B_API_URL="http://<your-node-ip>:3000"
export E2B_API_KEY="e2b_000000"
export CUBE_TEMPLATE_ID="<template-id>"
export SSL_CERT_FILE="/root/.local/share/mkcert/rootCA.pem"   # 可选
```

部署侧环境变量（MySQL/Redis/S3/监听端口等）不在客户端配置，而持久化在宿主机 `/usr/local/services/cubetoolbox/.one-click.env`（0600，见 [[cube-deploy-directory-layout]] §①），其中与启动沙箱直接相关的几个：`CUBE_API_BIND=0.0.0.0:3000`（API 监听）、`CUBEMASTER_ADDR=127.0.0.1:8089`、`CUBE_OPS_BIND=0.0.0.0:3010`、`CUBE_API_SANDBOX_DOMAIN=cube.app`。

## 2. 沙箱生命周期

### 2.1 创建

```bash
# CLI: 由 JSON 请求文件创建
cubecli cubebox create <request.json> [--rm] [--sleep/-s <秒>]

# SDK (E2B 风格)
from e2b_code_interpreter import Sandbox
sandbox = Sandbox.create(template=template_id, timeout=600)

# CubeAPI
curl -X POST http://<node-ip>:3000/sandboxes \
  -H "Content-Type: application/json" \
  -d '{"templateID": "<id>", "timeout": 600}'
```

**创建时注入环境变量**（注入后沙箱内命令可直接读取）：

```python
# SDK: envs= 参数
sandbox = Sandbox.create(
    template=template_id,
    envs={"API_TOKEN": "demo-token", "SESSION_ID": "user-session-test"},
)
sandbox.commands.run("echo $SESSION_ID")   # -> user-session-test
```

```jsonc
// API: envVars 字段（canonical 名；envs 是兼容别名）
{"templateID": "<id>", "envVars": {"API_TOKEN": "demo-token"}}
```

对应示例：`examples/code-sandbox-quickstart/create_with_envs.py`。

### 2.2 查询

```bash
cubecli cubebox list [--quiet/-q] [--all/-a] [--last/-n] [--latest/-l] [--no-trunc] [--wide/-w]
cubecli cubebox sandboxes          # 按沙箱视角列出
cubecli cubebox inspect <ID>       # 查看元数据(注意: 命令名是 inspect, 不是 metadata)
cubemastercli list [--filter] [--hostid/-t] [--sandboxid] [--wide/-w]
cubemastercli info --sandboxid/-s <id>
```

`cubecli` 侧 ID 支持前缀匹配（`resolveSandboxIDFromList`），前缀能唯一识别即可。

### 2.3 容器操作与执行命令

```bash
cubecli container list [--sandbox/-s <沙箱ID>]
cubecli container info <ID> [--spec]
cubecli container delete <ID> [--keep-snapshot]
cubecli exec <容器ID> <cmd...> [--tty/-t] [--interactive/-i] [--workdir/-w <dir>] [--detach/-d]
```

### 2.4 日志

```bash
cubecli logs <ID> [--tail/-t N] [--head/-H N] [--stderr/-e] [--all/-a]
cubecli logs --tpl <模板ID>        # 读模板日志(/data/log/template/<id>_0/)
```

### 2.5 暂停 / 恢复 / 超时（CubeAPI）

```
POST /sandboxes/{sandboxID}/pause      # 暂停
POST /sandboxes/{sandboxID}/resume     # 恢复
POST /sandboxes/{sandboxID}/timeout    # 调整超时
```

### 2.6 销毁 ⚠️

```bash
cubecli unsafe destroy --name/-n <名> | --annotation <k=v> [--all] [--force]
cubecli unsafe destroyall [--force]        # 全删
cubemastercli destroy <ID>                  # 控制面销毁(别名 rm)
```

⚠️ `destroy` 不在 `cubebox` 组——只注册在 `unsafe` 组下（见 §10）。

## 3. 镜像与模板

### 3.1 镜像（cubecli image）

```bash
cubecli image pull <image> [--username/-u <u>] [--creds <文件>] [--auth <文件>] [--annotation/-a k=v] [--pull-timeout/-pt]
cubecli image ls [--verbose/-v] [--filter/-f <k=v>] [--output/-o json|table] [--pinned]
cubecli image inspect <image> [--mode/-m dockercompat|native]
cubecli image inspecti <image...> [--output/-o] [--template <tpl>]
cubecli image rmi <image...> [--all/-a] [--prune/-q]
cubecli image emount <image[:tag]> <目标目录>    # 把镜像和层挂到目录
CUBEMNT=1 cubecli image fix [--skip-cfs] [--cubelet-path]   # 修复损坏镜像
cubecli image imagefsinfo [--output/-o]
```

已废弃：顶层 `cubecli images`、`cubecli load`（分别改用 `image ls` / `image pull`）。

### 3.2 模板（cubemastercli template / tpl）

```bash
# 从沙箱提交模板 / 在健康节点上建模板快照
cubemastercli template commit --sandbox-id <id> --file/-f <请求.json> [--detach/--no-wait] [--json]
cubemastercli template create --file/-f <请求.json> [--node <id>...] [--json]

# 从 OCI 镜像一步建模板(拉镜像→构建 ext4 rootfs→异步建模板), 快速上手首选
cubemastercli template create-from-image --image <镜像> [--memory <MB>] [--with-cube-ca] \
    [--enable-inject-envd] [--envd-path <路径>] [--detach/--no-wait] [--json]
cubemastercli template status --job-id <id>     # 跟踪 create-from-image 作业
cubemastercli template watch --job-id <id>

# 分发/维护
cubemastercli template redo <template-id>              # 全量/指定/失败节点上重建
cubemastercli template delete <template-id> [...]      # 删元数据+各节点副本
cubemastercli template set-alias --template-id <id> --alias <名> [--clear]
cubemastercli template list / info --template-id <id> [--include-request] / render [--file/-f]

# 沙箱 commit 的构建进度
cubemastercli template build-status --build-id <id>
cubemastercli template build-watch --build-id <id>
```

## 4. 快照与回滚

```bash
cubemastercli snapshot create --sandbox-id <id>      # 从运行中沙箱打快照
cubemastercli snapshot list / info <snapshot-id> / delete <snapshot-id>

cubemastercli sandbox rollback --sandbox-id <id> --snapshot-id <snap> [--instance-type <t>] [--json]

# 进度跟踪
cubemastercli operation status --operation-id <id>
cubemastercli operation watch --operation-id <id> [--interval <秒>] [--json]
cubemastercli storage status                        # 快照存储聚合状态

# 节点侧 DEBUG 级(直连 cubelet, 绕过 master)
cubecli cubebox snapshot [--snapshot-dir] [--json]                     # AGS 应用快照
cubecli cubebox debug-commit --sandbox-id <id> --template-id <tid> [--json]
cubecli cubebox debug-rollback --sandbox-id <id> --snapshot-id <sid> [--from-commit-result] [--json]
```

完整生命周期演示（含克隆、并发 fork、回滚后继续运行）：`examples/snapshot-rollback-clone`。

## 5. 网络操作

```bash
cubecli network ls [--config/-c]
cubecli container taps [--http-address <addr>] [--json]    # tap 运行时状态
```

出网策略（创建时设置，执行于 cubelet tap 层，沙箱内无法绕过）：

- 完全断网 / CIDR 允许列表 / CIDR 拒绝列表 → `examples/network-policy`
- 运行中动态更新策略 → 同上示例
- L7 自定义端口（eBPF skb->mark，与 `/etc/cubeegress/l7-marks.conf` 联动）→ `examples/code-sandbox-quickstart/network_l7_*.py`

## 6. 卷与存储

```bash
cubecli volume resetvolumeref <卷>           # 重置卷引用
cubecli storage ls [--bucket/-b] [--raw]
cubecli storage cleanup [--bucket/-b] [--format <f>] [--dry-run]   # 清理孤儿 emptydir
cubecli unsafe volumedb [--config/-c]        # 扫描 volumedb
```

卷驱动示例：`examples/volume`（`cos/` 腾讯云 COS、`s3/` 自实现 S3 驱动，含 `volume-*.conf.example`）；宿主目录挂载进沙箱：`examples/host-mount`（`Sandbox.create(metadata={"host-mount": ...})`）。

## 7. 压测与性能

```bash
# 节点侧批量压测(直接打 cubelet)
cubecli multirun <请求.json> --runcnt <n> --runcc <并发> [--addrm] [--norm] [--percents] \
    [--fail_exit] [--rmimage/-i <镜像>] [--disk-state] [--tag-key k --tag-value v] \
    [--sleep_before_del <秒>] [--sleep_after_del <秒>] [--dynamic-config-path <路径>] [--same] [--delcc]

# 控制面批量压测(打 cubemaster)
cubemastercli multirun <请求.json> --runcnt <n> --runcc <并发> [--hostid/-t] [--hostip/-s] \
    [--testmocksch] [--biztype <t>] [--testmultireq] [--async_retry_max <n>] [--norm] [--percents]

# 打 CubeAPI 的压测工具(Go, 带 TUI)
# examples/cube-bench: --concurrency/-c --total/-n --template/-t --warmup/-w --mode \
#   --output/-o --api-url --api-key --host-mount --network-policy/--np [--no-tui] [--dry-run]
```

性能分析方法论见 [[cube-performance-testing]]。

## 8. 节点运维（cubeopscli）

```bash
cubeopscli node list [--hostid <id>] [--score-only] [--show-local-templates] [--json]
cubeopscli node isolate <node-id> [...]      # 隔离(cordon), 不再调度新沙箱
cubeopscli node unisolate <node-id> [...]
cubeopscli node delete <node-id> [...] [--force]   # 删除已隔离且为空的节点
```

## 9. 排障与诊断

```bash
# 节点侧 unsafe / meta / vm
cubecli unsafe init                          # 初始化 cubelet
cubecli unsafe restoredb [--config/-c]       # 从备份库恢复元数据
cubecli meta dbs [--o json|table]            # 列出所有 bucket
cubecli meta view --db <db> --key <k> [--o]  # 只读查看 DB 数据
cubecli vm counter <容器>
cubecli container info <ID> [--spec]

# 集群脚本(安装目录 /usr/local/services/cubetoolbox/scripts/one-click/)
quickcheck.sh                                # 部署自检
cube-check.sh / check-deps.sh / check-procs.sh / collect-logs.sh   # 诊断三件套
```

日志位置速查（详见 [[cube-deploy-directory-layout]] §②）：cubelet `/data/log/Cubelet`、shim `/data/log/CubeShim`、VMM `/data/log/CubeVmm`（vmm.json + vmm.log）、控制面 `/data/log/CubeMaster`、`/data/log/CubeAPI`、`/data/log/CubeOps`；沙箱日志 `/data/cubelet/log`；模板日志 `/data/log/template/<id>_0/`。

## 10. 常见易错点

1. **`destroy` 不在 `cubebox` 组**：是 `cubecli unsafe destroy` / `unsafe destroyall`。
2. **没有 `cubebox rmimg` 命令**：`rmimg` 只是 `multirun` 的 `--rmimage/-i` flag。
3. **`cubebox resolve` 不是命令**：是 ID 前缀解析的内部辅助函数。
4. **"metadata" 子命令实际叫 `inspect`**：`cubecli cubebox inspect`。
5. **已废弃命令**：`cubecli ls`、`cubecli images`、`cubecli load`。
6. **`E2B_API_KEY` 在 auth 关闭时也要非空**：SDK 检查非空才发请求，随意填（如 `e2b_000000`）。
7. **`--json` 与位置参数**：`template commit/create` 的请求体走 `--file`；`rollback`/`debug-*` 走 flag。
8. **`snapshot create` 前沙箱必须在运行**；回滚目标快照必须属于该沙箱。

## 附录 A：命令速查表

| 组 | 命令 | 用途 |
|---|---|---|
| cubecli cubebox | list / sandboxes / inspect / create / snapshot / debug-commit / debug-rollback / update | 沙箱管理 |
| cubecli container | list / info / delete / taps | 容器管理 |
| cubecli(顶层) | exec / logs / multirun / version | 执行、日志、压测 |
| cubecli image | pull / ls / inspect / inspecti / rmi / emount / fix / imagefsinfo / ctr-image | 镜像 |
| cubecli network | ls | 网络 |
| cubecli storage | ls / cleanup | 存储 |
| cubecli volume | resetvolumeref | 卷 |
| cubecli unsafe | init / restoredb / rmi / volumedb / destroy / destroyall | 危险操作 |
| cubecli meta | dbs / view | 元数据(只读) |
| cubecli vm | counter | VM 计数 |
| cubemastercli | list / info / destroy / multirun / listinventory | 沙箱总览 |
| cubemastercli snapshot | create / list / info / delete | 快照 |
| cubemastercli rollback | (别名 sandbox-rollback) | 回滚 |
| cubemastercli operation | status / watch | 操作跟踪 |
| cubemastercli storage | status | 存储状态 |
| cubemastercli template | create / commit / create-from-image / redo / delete / set-alias / status / watch / build-status / build-watch / list / info / render | 模板 |
| cubemastercli volume | list / get / delete | 卷 |
| cubeopscli node | list / isolate / unisolate / delete | 节点运维 |

## 附录 B：CubeAPI 端点速查表（openapi.yml）

| 端点 | 用途 |
|---|---|
| `GET /health` | 健康检查 |
| `POST /sandboxes`、`GET /sandboxes`、`GET /sandboxes/{id}` | 创建 / 列出 / 查询沙箱 |
| `POST /sandboxes/{id}/connect` | 建立连接 |
| `GET /sandboxes/{id}/logs` | 日志 |
| `GET /sandboxes/{id}/network`、`POST /sandboxes/{id}/pause`、`POST /sandboxes/{id}/resume`、`POST /sandboxes/{id}/timeout` | 网络 / 暂停 / 恢复 / 超时 |
| `GET /sandboxes/{id}/refreshes`、`POST /sandboxes/{id}/rollback`、`POST /sandboxes/{id}/snapshots` | 刷新 / 回滚 / 快照 |
| `GET/POST /snapshots` | 快照列表与操作 |
| `GET/POST /templates`、`GET/POST /templates/{id}`、`POST /templates/{id}/alias`、`GET /templates/aliases/{alias}`、`GET /templates/compat`、`POST /templates/compat/{id}/adopt-baseline` | 模板管理 |
| `GET /templates/{id}/builds/{buildID}`、`.../logs`、`.../status` | 模板构建作业 |
| `GET/POST /v2/sandboxes`、`GET /v2/sandboxes/{id}/logs` | v2 沙箱接口 |
| `GET/POST /volumes`、`GET/DELETE /volumes/{volumeID}` | 卷 |

## 附录 C：相关笔记互链

- [[cube-deploy-directory-layout]] — 宿主机目录结构（日志位置、socket、配置）
- [[cube-template-creation-flow]] — 模板创建全流程
- [[cube-performance-testing]] — 压测方法与工具（cube-bench 等）
- [[cube-examples-scripts]] — 示例脚本用法
- `examples/code-sandbox-quickstart/README_zh.md` — 快速上手完整步骤（建模板→配 env→跑示例）
