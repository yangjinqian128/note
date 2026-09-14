# Cube 部署后宿主机目录结构

> 来源：CubeSandbox 仓库 `deploy/one-click/install.sh` + 各组件 Go/Rust 源码中的路径常量。
> 说明：标注 ◇ 的仅存在于对应节点类型 —— 计算节点只装 Cubelet、cube-shim、cube-kernel-scf、cube-image、cube-agent、cube-egress、cube-vs（+可选 CubeS3lvol）；控制节点为全量。

## ① 安装根目录 `/usr/local/services/cubetoolbox`

```
/usr/local/services/cubetoolbox/                    # 安装根(硬编码, CUBE_SANDBOX_INSTALL_ROOT)
├── Cubelet/                                        # cubelet 守护进程(计算节点核心)
│   ├── bin/
│   │   ├── cubelet                                 # 主进程二进制
│   │   └── cubecli                                 # 命令行工具
│   └── config/
│       ├── config.toml                             # 主配置(数据路径/网络/日志/快照)
│       ├── plugin.conf                             # 插件配置
│       └── snapshot.sh                             # 快照脚本
├── CubeMaster/                                     # ◇仅控制节点: 主控服务
│   ├── bin/
│   │   ├── cubemaster                              # 主控进程(元数据实际存 MySQL/Redis)
│   │   └── cubemastercli
│   └── conf.yaml                                   # MySQL/Redis 连接、监听地址(安装时注入)
├── CubeAPI/                                        # ◇仅控制节点
│   └── bin/cube-api                                # API 服务
├── CubeOps/                                        # ◇仅控制节点
│   └── bin/
│       ├── cubeops                                 # 运维服务(节点列表/隔离)
│       └── cubeopscli
├── webui/                                          # ◇仅控制节点: Web 控制台
│   └── nginx.generated.conf
├── cubeproxy/                                      # ◇仅控制节点: nginx 反代
│   ├── global.conf
│   └── nginx.conf
├── coredns/                                        # ◇仅控制节点: 集群 DNS
│   ├── Corefile
│   └── resolv.conf.upstream
├── cube-shim/                                      # 沙箱 shim / 运行时
│   └── bin/
│       ├── containerd-shim-cube-rs                 # OCI shim
│       └── cube-runtime                            # 运行时
├── cube-kernel-scf/                                # 客户机内核
│   ├── vmlinux-bm                                  # 普通 guest 内核
│   ├── vmlinux-pvm                                 # PVM guest 内核
│   └── vmlinux → vmlinux-bm|pvm                    # 软链, 按 CUBE_PVM_ENABLE 选择
├── cube-image/                                     # guest 镜像
├── cube-agent/                                     # guest 内 agent 的宿主侧文件
├── cube-egress/                                    # 透明 egress MITM 代理(docker 运行)
├── cube-lifecycle-manager/                         # 生命周期管理服务
├── CubeS3lvol/                                     # ◇可选: S3 卷
│   ├── bin/s3lvol_tgt
│   └── scripts/                                    # rcow 相关脚本
├── cube-vs/network/                                # cubevs 虚拟网络
│   └── bin/cubevsmapdump                           # eBPF map 转储工具
├── scripts/                                        # one-click 运维脚本(up/down/quickcheck)
├── systemd/                                        # systemd 单元模板(安装时拷到 /etc)
├── .one-click.env                                  # 持久化部署配置(0600, 含 DB/S3 密码)
├── VERSION.txt                                     # 部署版本
├── release-manifest.json                           # 各组件版本清单
├── env.example                                     # 下次升级的 three-way merge 基线
└── cubeletmnt/                                     # cubelet 挂载命名空间入口(bind mount)
    └── mnt                                         # → /proc/<pid>/ns/mnt
```

关键常量出处：

- `CUBE_SANDBOX_INSTALL_ROOT` 默认值硬编码在 `deploy/one-click/lib/common.sh`（非 `/usr/local/services/cubetoolbox` 时强制改回，即不可改）。
- `CubeMntNsDirPath = "/usr/local/services/cubetoolbox/cubeletmnt"` 在 `Cubelet/cmd/cubelet/main.go:57`。

## ② 运行时数据盘 `/data`

```
/data/
├── cubelet/                                        # cubelet 数据根(强制 XFS, 整体 virtiofs 共享给 guest)
│   ├── root/                                       # containerd 式运行时根
│   │   ├── io.containerd.runtime.v2/task/          # 运行中的沙箱任务
│   │   ├── io.cubelet.internal.v1.cubebox/         # cubebox 内部数据
│   │   ├── io.containerd.metadata.v1.bolt/meta.db  # containerd 元数据(boltdb)
│   │   ├── volume/                                 # 卷插件目录
│   │   ├── component_versions/                     # 组件版本清单(升级前先快照到这里)
│   │   └── (镜像层、快照器等 containerd 数据)
│   ├── state/                                      # 运行状态目录
│   ├── storage/                                    # 存储插件数据(storage.data_path)
│   ├── hostdir/                                    # hostdir 卷插件基目录
│   ├── fifo/                                       # exec 用的 FIFO
│   ├── shimlog/                                    # shim 日志
│   ├── cleanup/                                    # 清理目录
│   ├── log/                                        # 沙箱日志
│   ├── rcow/
│   │   └── wal_bdev.img                            # S3lvol WAL 稀疏镜像(默认 ~512GiB)
│   ├── network-agent/state/                        # 旧 network-agent 状态(已废弃)
│   ├── rainbow.toml                                # Redis 配置
│   ├── s3.cfg                                      # S3 凭证(s3lvol 读取)
│   ├── cubelet.sock                                # gRPC socket(cubecli 连这里)
│   ├── cubetap.sock                                # tap socket
│   └── cubelet-operation.sock                      # 运维 socket
├── log/                                            # 各组件日志
│   ├── Cubelet/
│   ├── CubeShim/                                   # 含 cube-shim-req.log
│   ├── CubeVmm/                                    # vmm.json + vmm.log
│   ├── CubeAPI/                                    # ◇仅控制节点
│   ├── CubeOps/                                    # ◇仅控制节点
│   ├── CubeMaster/                                 # ◇仅控制节点
│   ├── cube-proxy/                                 # ◇仅控制节点
│   ├── rcow/
│   └── cubecow/                                    # cubecow.log
├── cube-shim/
│   ├── disks/                                      # 沙箱磁盘镜像
│   └── snapshot/                                   # 快照状态(SnapshotStatusPath)
├── snapshot_pack/
│   └── disks/                                      # 快照打包临时目录
├── cube-shared/
│   └── volume/                                     # 卷插件共享目录(volume_plugin_base_dir)
├── shared/                                         # 通用共享
│   └── agenthub/
│       ├── openclaw/
│       └── openclaw-snapshots/
├── CubeMaster/
│   └── storage/                                    # 模板中心制品存储(guest image 等)
└── stop.dat                                        # bench 停机标志文件
```

关键点：

- `/data/cubelet` 必须位于 XFS 文件系统（`install.sh` 的 `check_cubelet_fs_preflight` 强制检查）。
- 目录创建见 `install.sh` 的 `mkdir -p` 块（log 目录、cube-shim/disks、snapshot_pack/disks、cube-shared、shared 等）。
- `/data/cubelet/root/component_versions`：升级时旧版本组件先 inventory 到这里再替换。
- `wal_bdev.img` 只在首次安装且 `ONE_CLICK_ENABLE_S3LVOL=1` 时一次性创建（稀疏文件，大小 = WAL + journal + cache）。
- virtiofs 共享路径：`Cubelet/pkg/container/virtiofs/virtiofs.go` 中 `virtioFsSharePath = "/data/cubelet/"`。

## ③ 系统级目录

```
/etc/
├── systemd/system/
│   ├── cube-sandbox-control.target                 # 控制节点开机自启入口
│   ├── cube-sandbox-compute.target                 # 计算节点开机自启入口
│   ├── cube-sandbox-cubelet.service                # ExecStart → toolbox/scripts/systemd/cubelet-start.sh
│   ├── cube-sandbox-cubemaster.service
│   ├── cube-sandbox-cube-api.service
│   ├── cube-sandbox-cubeops.service
│   ├── cube-sandbox-webui.service
│   ├── cube-sandbox-cube-proxy.service
│   ├── cube-sandbox-dns.service
│   ├── cube-sandbox-coredns.service
│   ├── cube-sandbox-cube-egress.service
│   ├── cube-sandbox-cube-egress-net.service
│   ├── cube-sandbox-cube-lifecycle-manager.service
│   ├── cube-sandbox-mysql.service                  # docker MySQL(外接数据库时被 mask)
│   ├── cube-sandbox-redis.service                  # docker Redis(外接时被 mask)
│   ├── cube-sandbox-minio.service                  # docker MinIO(计算节点/关闭时被 mask)
│   └── cube-sandbox-s3lvol.service                 # 可选, 按 ONE_CLICK_ENABLE_S3LVOL 启用
├── cubeegress/
│   └── l7-marks.conf                               # L7 eBPF skb->mark 值(cubelet 网络运行时 + iptables TPROXY 共用)
└── cube/
    └── ca/
        └── cube-root-ca.crt                        # cube-egress 根证书(master 下载/模板烘培用)

/usr/local/bin/                                     # CLI 软链
├── cube-runtime                → …/cubetoolbox/cube-shim/bin/cube-runtime
├── containerd-shim-cube-rs     → …/cubetoolbox/cube-shim/bin/containerd-shim-cube-rs
├── cubecli                     → …/cubetoolbox/Cubelet/bin/cubecli
├── cubevsmapdump               → …/cubetoolbox/cube-vs/network/bin/cubevsmapdump
├── cubemastercli               → …/cubetoolbox/CubeMaster/bin/cubemastercli   ◇仅控制节点
└── cubeopscli                  → …/cubetoolbox/CubeOps/bin/cubeopscli         ◇仅控制节点
```

关键点：

- 所有 service 单元 `EnvironmentFile=-/usr/local/services/cubetoolbox/.one-click.env`，配置由该文件驱动。
- 外接 MySQL/Redis/S3 时本地容器单元会被 mask（先删单元文件再 `systemctl mask`，见 `mask_local_dep_service`）。
- `/etc/cube/ca` 相关：`CubeMaster/pkg/service/httpservice/cube/ca_download.go`（caRootDir 常量）与 `cube_egress_ca_bake.go`（模板烘培时 bake 进 guest）。

## ④ 其余

```
/var/lib/docker/                # MySQL / Redis / MinIO / cube-egress 容器数据(由 docker volume 管理,
                                # 安装脚本不直接碰; 外接 MySQL/Redis/S3 时这些容器被 mask 不启动)
/sys/fs/bpf/                    # eBPF pinned maps(安装前置强制要求 bpffs)
/dev/kvm                        # KVM 设备(安装前置强制要求)

【guest 沙箱 VM 内】
/                               # virtiofs 只读挂载宿主机 /data/cubelet
/etc/cube/ca/cube-root-ca.crt   # 烘培进模板的 egress 根证书
```

## 附：安装前置检查对应的路径要求

| 检查 | 要求 |
|---|---|
| `check_cubelet_fs_preflight` | `/data/cubelet` 必须位于 XFS |
| `check_cgroup_cpu_preflight` | cgroup v2 必须暴露 cpu controller |
| `check_bpf_fs_preflight` | `/sys/fs/bpf` 必须是 bpffs 挂载 |
| `check_hardware_preflight` | `/dev/kvm` 必须存在（否则提示走 PVM 方案） |
| `check_pvm_consistency_preflight` | 加载了 kvm_pvm 模块时 CUBE_PVM_ENABLE 必须为 1 |
