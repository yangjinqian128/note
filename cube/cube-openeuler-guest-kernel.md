# 使用 openEuler 内核作为 CubeSandbox guest 内核 — 操作文档

> 结论：**可以**。Cube 的 guest 内核是独立组件（`cube-kernel-scf`），以 `vmlinux` 文件的形式在启动时传给 VMM，与 guest 用户态（guest image）完全解耦。默认内核基于 OpenCloudOS 9 的配置构建（`configs/kernel-oc9.<arch>.config`），换成 openEuler 内核源码 + 同一套配置即可。

## 原理速览

- 运行时内核路径：`/usr/local/services/cubetoolbox/cube-kernel-scf/vmlinux`（CubeShim 默认值，代码见 `CubeShim/shim/src/sandbox/config.rs`）
- 内核以**未压缩 vmlinux** 形式存在（aarch64 场景构建脚本会把 `Image` 改名为 `vmlinux`）
- 版本管理：节点上另有版本仓库 `/data/cubelet/root/component_versions/cube-kernel-scf/<version>/`，新模板会绑定内核版本（"稳定恢复"机制，见 `docs/zh/guide/component-multiversion.md`）

三条替换路线，按场景选：

| 路线 | 做法 | 适用场景 |
|---|---|---|
| A. 节点直接替换 | 替换 toolbox 里的 vmlinux | 开发验证、快速试验（**本档主线**） |
| B. 组件版本管理 | 导入版本仓库 + 模板绑定 | 生产化、多版本并存 |
| C. annotation 指定 | `cube.vm.kernel.path` 按沙箱覆盖 | 调试对比，内部机制 |

---

## 第一步：获取 openEuler 内核源码

```bash
git clone https://gitee.com/openeuler/kernel.git openEuler-kernel
cd openEuler-kernel
# 按需选分支（示例两个）：
git checkout openEuler-24.03-LTS   # 内核 6.6
# 或 git checkout openEuler-22.03-LTS-SP4  # 内核 5.10
```

用 openEuler **官方源码树**即可，无需 Cube 的任何内核 patch。

## 第二步：内核配置（最关键的步骤）

Cube 提供了开箱即用的构建入口：`make guest-kernel KERNEL_SRC=<openEuler 源码路径>`。它做的事（见 `scripts/build-kernel.sh`）：

1. 把 `configs/kernel-oc9.<arch>.config` 复制为 `.config`
2. 执行 `make olddefconfig` —— **这是能直接吃 openEuler 树的关键**：oc9 配置里 openEuler 内核没有的选项被丢弃，openEuler 新增的选项取默认值
3. 构建 `vmlinux`

**Cube guest 内核的必需配置清单**（已从 `configs/kernel-oc9.x86_64.config` 提取，自己核对时以此为准）：

| 配置项 | 用途 |
|---|---|
| `CONFIG_VIRTIO_BLK` / `CONFIG_VIRTIO_NET` | rootfs 块设备 / 网卡 |
| `CONFIG_VIRTIO_VSOCKETS` | vsock——shim 与 guest agent 的通信通道 |
| `CONFIG_VIRTIO_FS` | virtio-fs 共享文件系统 |
| `CONFIG_EXT4_FS` | guest rootfs 文件系统 |
| `CONFIG_DAX` | pmem 设备支持 |
| `CONFIG_KVM_GUEST` | guest 侧虚拟化支持 |
| `CONFIG_USERFAULTFD` | 内存快照/恢复依赖 |
| `CONFIG_SERIAL_8250_CONSOLE` | 串口控制台（调试必备） |
| `CONFIG_OVERLAY_FS` | 容器风格 rootfs 层叠 |
| `CONFIG_CGROUPS` 及其子系统 | 沙箱内资源隔离 |

**两个容易误解的点**：

1. `CONFIG_XFS_FS` 在 oc9 配置里**是关闭的**——不是遗漏。XFS reflink 用在**宿主侧**（CubeCoW），guest 看到的 rootfs 是 ext4，guest 内核不需要 XFS
2. `CONFIG_MEM_SOFT_DIRTY` 默认**未启用**——深挖博客里 SoftDirty 真增量快照模式需要它。想要 openEuler 内核支持该模式，可自行加上（`scripts/config --enable MEM_SOFT_DIRTY` 或改配置后重跑 olddefconfig）

## 第三步：构建

```bash
# 本机构建（x86_64 构建 x86_64）
make guest-kernel KERNEL_SRC=/path/to/openEuler-kernel

# 交叉编译 aarch64
make guest-kernel KERNEL_SRC=/path/to/openEuler-kernel KERNEL_TARGET_ARCH=aarch64
```

- 构建在 Cube 统一的 builder 镜像（Docker）内进行，工具链自动备齐
- 产物：`_output/kernel/<arch>/vmlinux`（几十 MB 量级）
- 构建时长视机器而定，一般几十分钟

## 第四步：部署

### 路线 A：节点直接替换（开发验证，最简单）

```bash
# 1. 备份原内核（务必！）
sudo cp /usr/local/services/cubetoolbox/cube-kernel-scf/vmlinux \
        /usr/local/services/cubetoolbox/cube-kernel-scf/vmlinux.bak

# 2. 替换
sudo cp _output/kernel/x86_64/vmlinux \
        /usr/local/services/cubetoolbox/cube-kernel-scf/vmlinux

# 3. 回滚（出问题时）
sudo cp /usr/local/services/cubetoolbox/cube-kernel-scf/vmlinux.bak \
        /usr/local/services/cubetoolbox/cube-kernel-scf/vmlinux
```

- **只影响之后新建的沙箱**，已运行的沙箱不受影响（它们的内存里已经有自己的内核）
- 替换后立刻生效，无需重启任何服务

### 路线 B：组件版本管理（生产化）

1. 将 vmlinux 放入版本仓库：`/data/cubelet/root/component_versions/cube-kernel-scf/<新版本号>/`
2. 通过 WebUI 仓库导入一键包（或参考 `docs/zh/guide/component-multiversion.md` 的导入/预装流程）
3. 新模板在构建时会绑定该内核版本——之后从该模板创建的沙箱都用 openEuler 内核

### 路线 C：annotation 按沙箱指定（调试用）

CubeShim 支持 annotation `cube.vm.kernel.path` 覆盖默认内核路径（可配合 `cube.vm.kernel.cmdline.append` 追加内核参数）。这是 containerd Create 层的内部 annotation，透传能力以当前 API 支持为准，适合本地调试对比。

## 第五步：验证

```bash
# 建一个沙箱并确认内核版本
python examples/code-sandbox-quickstart/exec_code.py \
  --code "import platform; print(platform.release())"
```

**功能验证清单**（openEuler 内核务必全过一遍）：

| 验证项 | 说明 |
|---|---|
| 跑代码 / 读写文件 | vsock + rootfs 通路正常 |
| 外网访问 | VIRTIO_NET + SNAT 链路正常 |
| **快照 / 克隆 / 回滚** | 重点验证！内存快照对内核特性有依赖 |
| auto-pause / auto-resume | 恢复路径正常 |
| 长时间运行 | 稳定性观察 |

## 注意事项与风险

1. **只替换 guest 内核，绝不碰宿主侧内核**——宿主侧内核（PVM 场景的 host deb/rpm 等）与此操作无关
2. 内核格式必须是 **vmlinux**（非 bzImage/压缩格式）
3. 模板会**绑定内核版本**：升级/混用不同内核版本的节点时注意"稳定恢复"机制的行为（见 component-multiversion 文档）
4. 快照跨内核版本恢复：同一沙箱的快照最好用同一内核版本恢复，跨版本恢复可能出现意料之外的行为，验证清单里务必覆盖
5. openEuler 特有配置（如安全加固项）可能与沙箱启动流程冲突（如 selinux 策略），首次构建建议**保持 oc9 配置的选项不动，只换源码树**——olddefconfig 会处理差异
6. 故障排查顺序：沙箱起不来 → 看 CubeShim 日志；能起但功能异常 → 对照上方必需配置清单逐项核对
