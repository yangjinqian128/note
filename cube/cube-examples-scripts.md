# CubeSandbox 示例脚本速查（examples/ 玩具箱）

> 仓库 `examples/` 目录 = 项目的"玩具箱"，所有可运行的玩法脚本都在这里。
> 跑之前先看各目录的 README 配好环境（API URL / KEY / 模板），`env.py` / `env_utils.py` 里有默认配置。

## 1. 入门基础（`code-sandbox-quickstart/`，15 个脚本）

| 脚本 | 玩法 |
|---|---|
| `create.py` | 最基础：建一个沙箱 |
| `exec_code.py` / `cmd.py` / `read.py` | 跑 Python 代码 / shell 命令 / 读文件 |
| `create_with_envs.py` | 带环境变量建沙箱 |
| `pause.py` / `auto-resume.py` | 手动暂停 / 自动恢复演示 |
| `auto-kill.py` | 超时自动销毁 |
| `mask-request-host.py` | path-based 路由（不用泛域名） |
| `network_no_internet.py` / `_allowlist.py` / `_denylist.py` | 断网 / 出网白名单 / 黑名单三连 |
| `network_l7_custom_port_echo.py` | 沙箱内起自定义端口服务再从外面访问 |
| `restrict_public_access.py` | 限制公网访问沙箱 |

## 2. 快照·克隆·回滚（`snapshot-rollback-clone/`，20 个脚本）★ 对应学习主题

- **编号教程 01–11**（按顺序 = 手把手教程）：
  01 创建快照 → 02 列快照 → 03 从快照克隆 → 04 验证状态保留 → 05 快照比沙箱活得久 → 06 一次克隆 N 份 → 07 并发克隆 → 08 三轴分叉 → 09 回滚 → 10 回滚后继续跑 → 11 删快照
- **`bench_*.py` × 6**：快照/克隆/回滚/创建/暂停恢复的**并发压测**
  - `bench_snapshot_dirty.py` 直接验证"脏页越多快照越慢"——和深挖博客的增量页原理互为印证
- `clone_demo.py` / `rollback_demo.py`：现成演示脚本（分享 Demo 素材）

## 3. 网络玩法

| 目录 | 脚本 | 玩法 |
|---|---|---|
| `network-policy/` | 4 个 | 断网/白名单/黑名单/**动态更新**策略 |
| `route-aware-egress/` | `dual_nic.py`、`gre_tunnel_gateway.py` | 双网卡 + GRE 隧道做路由感知出口 |
| `grpc-ingress/` | `grpc_plaintext.py` | CubeProxy 的 gRPC 明文入口（9090） |

## 4. Agent 框架集成（生态演示）

| 目录 | 玩法 |
|---|---|
| `openai-agents-example/` | OpenAI Agents SDK 的 `E2BSandboxClient` 直连 Cube |
| `openai-agents-code-interpreter/` | 数据分析 Agent：LLM 在沙箱里真跑 pandas/matplotlib 并画图 |
| `langchain-integration/` | LangChain 0.x 和 1.x 两版 agent 演示 |
| `pi-agent-integration/` | 跑 Pi coding agent（`run_pi_agent.py`）、预热（`run_pi_warmup.py`）、断点恢复（`resume_pi_agent.py`） |
| `openclaw-integration/` | OpenClaw 技能配置 |
| `claude-code-integration/` | Claude Code 的 bash 执行透明隔离进沙箱（`hooks/` 是 hook 脚本） |
| `mini-rl-training/` | SWE-bench RL 训练：`scripts/run-concurrent.py` 并发跑题 |

## 5. 虚拟化向玩具

| 目录 | 脚本 | 玩法 |
|---|---|---|
| `ivshmem/` | `ivshmem_ring_demo.py` + `ivshmem_benchmark.py` | 宿主↔guest 共享内存通道 + benchmark |
| `host-mount/` | `create_with_mount.py` | 宿主目录只读/读写挂进沙箱 |
| `e2b-dev-sidecar/` | `dev_sidecar.py` | 本地 `e2b_code_interpreter` SDK 经 sidecar 直连 Cube |
| `browser-sandbox/` | `browser.py` | 沙箱里跑无头 Chromium，Playwright 远程控制 |
| `volume/` | `cos/` + `s3/` | 卷框架对接 COS / S3 后端 |

## 6. 压测工具

- `cube-bench/`：CLI 基准工具，按配置的并发度测沙箱创建/销毁延迟（benchmark 数据同款）
- `cubesandbox-base-nginx/`：配套的自定义镜像示例（`test_files.py`）

## 上手顺序建议

1. 快照教程 01–11（配合快照主题学习）
2. `bench_snapshot_dirty.py`（验证增量页原理）
3. `ivshmem`（纯虚拟化玩具）
4. 分享 Demo 素材：`clone_demo.py` / `rollback_demo.py`
