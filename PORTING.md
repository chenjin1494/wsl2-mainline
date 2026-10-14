# Linux 7.2.10 WSL2 移植与复现指南

本文记录本项目将 Linux 7.2.10 适配到 WSL2 的移植内容、构建步骤、启用方法和实际验证结果。目标读者可以按步骤在 Ubuntu on WSL 中重新生成 x86_64 内核，并了解当前移植的边界。

## 1. 当前状态

- x86_64 Linux 7.2.10 WSL2 内核已完整构建，`bzImage` 与配置的可加载模块均构建成功。
- 生成的内核已在 WSL2 中启动。运行内核报告 `7.2.10-microsoft-standard-WSL2`。
- WSL 的 `/dev/dxg` 存在；`nvidia-smi` 成功查询到 NVIDIA GeForce RTX 3050。
- Windows C 盘挂载和 DNS 解析均通过检查。
- ARM64 配置已用 7.2.10 的 Kconfig 规范化，但本次没有 AArch64 交叉编译器，因此没有完成 ARM64 编译或启动测试。
- 虽然已构建 `.ko` 模块，但尚未安装并打包成 WSL `modules.vhdx`。
- 当前 Ubuntu 没有活动的 `virtiofs` 文件系统挂载，因此 virtio-fs 的实际共享文件读写路径尚未验证。

## 2. 输入与目录

本次使用的输入文件位于 Windows 的 `D:\data\linux`，在 WSL 中对应 `/mnt/d/data/linux`：

- `linux-7.2.10.tar.xz`：主线 Linux 7.2.10 源码。
- `WSL2-Linux-Kernel-linux-msft-wsl-6.18.y.zip`：微软 WSL Linux 6.18.54.1 源码快照，作为 WSL 驱动和补丁的移植参考。

项目目录是 `D:\data\git\wsl2-mainline`，在 WSL 中对应 `/mnt/d/data/git/wsl2-mainline`。项目包含配置、补丁、文档和 x86_64 构建镜像，不包含完整的上游 Linux 源码树。

内核源码及构建目录应放在 WSL 的 Linux 文件系统（ext4）中，而不是 `/mnt/d`。内核源码包含某些 Windows 文件系统无法表示的文件名；在 `/mnt/d` 上解包可能失败或产生不完整源码。

本次使用的 WSL 目录布局如下：

```text
/home/chenjin1494/wsl-kernel/
├── mainline/                 # 解包并应用补丁后的 Linux 7.2.10 源码
├── wsl-6.18/                 # 微软 6.18 参考源码快照
├── out-x86/                  # x86_64 构建输出
├── out-arm64/                # ARM64 配置规范化输出
└── local-deps/               # 无 sudo 时使用的本地构建依赖
```

以下命令将工作目录设为 `$HOME/wsl-kernel`，不要求用户名必须是 `chenjin1494`。

## 3. 移植内容与取舍

### 3.1 DXGKRNL 图形驱动

`patches/0001-dxgkrnl-port.patch` 从提供的 Microsoft WSL 6.18 源码快照移植 DXGKRNL 驱动和 D3DKMT UAPI，并接入 Linux 7.2.10：

- 加入 `drivers/hv/dxgkrnl/` 驱动实现、Kconfig 和 Makefile。
- 加入 `include/uapi/misc/d3dkmthk.h` 与 `include/uapi/linux/dxgkrnl.h`。
- 将 DXGKRNL 接入 `drivers/hv/Kconfig` 和 `drivers/hv/Makefile`。
- 在 `include/linux/hyperv.h` 中加入 DXGK 全局通道和虚拟 GPU 通道的 GUID 定义。
- 两份 WSL 配置均将 `CONFIG_DXGKRNL` 设为 `y`，将驱动编入内核。

移植时，首次 DXGKRNL 子目录编译发现目标 7.2.10 的 Hyper-V 头文件缺少上述 DXGK GUID 定义；补齐定义后，DXGKRNL 子目录编译通过。最终补丁也在干净的 Linux 7.2.10 基线目录上验证过可应用。

### 3.2 ARM64 Hyper-V hypercall

`patches/0002-hyperv-arm64.patch`（原补丁标识 `712c88941ba7cfd3c9eec3a58eef4c231226d66f`）让 ARM64 Hyper-V 使用 Hyper-V hypercall 接口，而不是依赖并非所有 Windows 版本都支持的 SMCCC 接口。该补丁加入 HVC 汇编实现并调整 Hyper-V 调用路径。补丁已应用并且 ARM64 配置已规范化；本环境没有 AArch64 交叉编译器，故没有验证 ARM64 编译结果。

### 3.3 virtio-fs DMA 限制

`patches/0003-virtiofs-dma.patch`（原补丁标识 `3586f43a9a33f5f12306de85b51f696194778156`）在 FUSE virtio-fs 初始化时，按照传输层单次 DMA mapping 上限约束 `max_pages_limit`，避免请求缓冲区超过受限 DMA 映射能力。补丁已应用，x86_64 完整内核构建也编译了 `fs/fuse/virtio_fs.o`。

Linux 7.2.10 已包含 virtio-fs DAX 修复，因此没有重复移植该修复。Hyper-V timer 修复也已经在目标主线中，不重复添加。

### 3.4 未纳入的下游变更

Microsoft 6.18 分支中的 `hv_pci_swiotlb=<base>,<size>` 专用池改动没有直接纳入。该变更依赖 7.2.10 中不存在的早期参数/底层实现；直接应用下游补丁时有多个 hunk 无法匹配。将它强行拼接到目标树会留下不完整功能，需另行做 API 级移植和验证。

### 3.5 移植过程中的故障与处理

- 最初尝试在 Windows 文件系统上解包 Linux 源码时失败，原因是部分 Linux 文件名无法由该文件系统表示。将源码和构建输出移至 WSL ext4 后解决。
- 初始环境缺少 `bison` 和 `gelf.h`。为避免修改系统软件包，使用 Ubuntu 对应的 flex、bison、m4 和 libelf 开发文件的本地解包副本。
- objtool 曾链接到静态 libelf，产生未解析的 `ZSTD_*` 与 `deflate*` 符号。让本地依赖路径指向系统动态 `libelf.so.1` 后继续构建；只有遇到相同链接错误时才应用该 workaround。
- 首次 DXGKRNL 编译发现 7.2.10 的 `include/linux/hyperv.h` 缺少两个 DXGK VMBus GUID。补入全局通道和虚拟 GPU 通道 GUID 后，驱动子目录构建通过。
- 在生成 DXGKRNL 补丁期间，一次零上下文修改将 GUID 定义插入现有注释，导致语法错误。修正头文件并重新生成补丁后，在干净基线上验证补丁可应用且驱动可编译。
- 来自 Microsoft 6.18 的 `hv_pci_swiotlb` 下游补丁有多个 hunk 不适用于 7.2.10，且目标树缺少其依赖的早期参数实现，因此没有强行移植。

## 4. 准备环境

在 Ubuntu WSL 发行版中准备工具链。推荐有 sudo 权限时安装发行版提供的标准依赖：

```sh
sudo apt update
sudo apt install build-essential flex bison bc libssl-dev libelf-dev dwarves python3
```

实际移植环境没有安装系统软件包，而是将 flex、bison、m4 和 libelf 头文件解包在 `$HOME/wsl-kernel/local-deps` 中。若需要保持系统不变，可在 Ubuntu 中下载与当前发行版匹配的 `.deb` 包并用 `dpkg-deb -x` 解包到该目录；具体包名和依赖版本随 Ubuntu 发行版而异。本次配置记录的工具链为 GCC 15.2.0 与 GNU Binutils 2.46。

本次构建使用的本地依赖环境变量如下。仅在采用本地解包依赖时使用；已通过 apt 安装依赖时通常不需要它们：

```sh
export WORK="$HOME/wsl-kernel"
export LOCAL="$WORK/local-deps"
export PATH="$LOCAL/usr/bin:/usr/bin:/bin"
export M4="$LOCAL/usr/bin/m4"
export BISON_PKGDATADIR="$LOCAL/usr/share/bison"
export C_INCLUDE_PATH="$LOCAL/usr/include"
export LIBRARY_PATH="$LOCAL/usr/lib/x86_64-linux-gnu"
```

本次无 sudo 构建还遇到 objtool 选择静态 libelf 后缺少 `ZSTD_*`/`deflate*` 符号的问题。通过让本地依赖查找动态 libelf 解决：

```sh
mkdir -p "$LOCAL/usr/lib/x86_64-linux-gnu"
ln -s /usr/lib/x86_64-linux-gnu/libelf.so.1 \
  "$LOCAL/usr/lib/x86_64-linux-gnu/libelf.so"
```

只有在复现相同 objtool 链接错误时才需要此 workaround；使用发行版完整开发包时不要盲目创建链接。

## 5. 解包源码并应用补丁

先确认输入路径和项目路径存在，并使用一个新的源码目录：

```sh
export WIN_DATA=/mnt/d/data/linux
export PROJECT=/mnt/d/data/git/wsl2-mainline
export WORK="$HOME/wsl-kernel"
export SRC="$WORK/mainline"
mkdir -p "$SRC"
```

解包主线源码到 WSL ext4：

```sh
tar -xf "$WIN_DATA/linux-7.2.10.tar.xz" \
  --strip-components=1 -C "$SRC"
```

把项目配置和补丁复制到源码树，然后按文件名顺序应用：

```sh
cp -a "$PROJECT/Microsoft" "$SRC/"
cp -a "$PROJECT/patches" "$SRC/"
cd "$SRC"
set -eu
for patch_file in patches/*.patch; do
  patch --batch --forward -p1 < "$patch_file"
done
```

`set -eu` 可确保某个补丁失败时流程立即停止，避免继续构建一个只应用了部分补丁的源码树。不要对已经应用过补丁的源码目录再次运行这段循环；需要重做时，从干净的 7.2.10 源码重新解包。

若要重新检查参考快照，可将 Microsoft ZIP 解包到单独目录。此步骤对使用项目中已经生成的补丁进行复现不是必需的：

```sh
mkdir -p "$WORK/wsl-6.18"
python3 - "$WIN_DATA/WSL2-Linux-Kernel-linux-msft-wsl-6.18.y.zip" "$WORK/wsl-6.18" <<'PY'
import sys
import zipfile

with zipfile.ZipFile(sys.argv[1]) as archive:
    archive.extractall(sys.argv[2])
PY
```

该快照用于移植和对照；实际构建目标仍是干净解包的 Linux 7.2.10 加上本项目补丁。

## 6. 规范化配置

配置文件最初基于 Microsoft WSL 6.18 配置。不能直接把 6.18 的 `.config` 当作 7.2.10 最终配置使用；先在已应用补丁的 7.2.10 源码上运行 `olddefconfig`，由目标 Kconfig 删除失效选项并补齐新选项。

x86_64：

```sh
export OUT_X86="$WORK/out-x86"
make -C "$SRC" O="$OUT_X86" \
  KCONFIG_CONFIG="$SRC/Microsoft/config-wsl" olddefconfig
```

ARM64 配置规范化：

```sh
export OUT_ARM64="$WORK/out-arm64"
make -C "$SRC" O="$OUT_ARM64" ARCH=arm64 \
  KCONFIG_CONFIG="$SRC/Microsoft/config-wsl-arm64" olddefconfig
```

确认 DXGKRNL 已启用并检查内核 release：

```sh
grep '^CONFIG_DXGKRNL=y$' "$SRC/Microsoft/config-wsl"
grep '^CONFIG_DXGKRNL=y$' "$SRC/Microsoft/config-wsl-arm64"
make -s -C "$SRC" O="$OUT_X86" \
  KCONFIG_CONFIG="$SRC/Microsoft/config-wsl" kernelrelease
```

预期 x86_64 release 为：

```text
7.2.10-microsoft-standard-WSL2
```

本次 ARM64 `olddefconfig` 也成功，且其 `kernelrelease` 结果相同；这只验证配置解析，不代表 ARM64 已编译。

## 7. 构建 x86_64 内核与模块

使用与配置相同的 `KCONFIG_CONFIG` 和输出目录构建。实际验证使用 `-j2`，以兼顾 WSL 的资源占用：

```sh
make -C "$SRC" O="$OUT_X86" \
  KCONFIG_CONFIG="$SRC/Microsoft/config-wsl" -j2 bzImage

make -C "$SRC" O="$OUT_X86" \
  KCONFIG_CONFIG="$SRC/Microsoft/config-wsl" -j2 modules
```

完成后检查镜像：

```sh
file "$OUT_X86/arch/x86/boot/bzImage"
ls -lh "$OUT_X86/arch/x86/boot/bzImage"
```

本次已验证生成的镜像是 x86 boot executable，版本为 `7.2.10-microsoft-standard-WSL2`，大小约 14 MiB。镜像已复制到项目目录：

```text
artifacts/bzImage-7.2.10-wsl2-x86_64
```

本次已构建镜像的 SHA-256：

```text
a1d0765dea1c3fe57bdabee382cb35f0725f750f186ae4179d8beaa17724f29f
```

重建时内核时间戳等信息会变化，因此新构建的 SHA-256 不一定与上述值相同。`make modules` 也已成功完成；这些模块位于 `$OUT_X86` 构建输出中，但尚未安装或制成 WSL 模块盘。

`bzImage` 是压缩的 x86 启动内核镜像，不是内核源码树，也不包含所有标记为模块的驱动。启动时内核会解压；`CONFIG_DXGKRNL=y` 使本次 DXGKRNL 驱动编入镜像，而配置为 `=m` 的驱动会单独生成 `.ko` 文件。模块安装与 WSL VHDX 打包见“WSL 模块盘”一节。

## 8. ARM64 构建命令

需要安装 AArch64 交叉编译器后才可进行 ARM64 构建。例如 Ubuntu 提供的工具链：

```sh
sudo apt install gcc-aarch64-linux-gnu binutils-aarch64-linux-gnu
```

然后构建：

```sh
make -C "$SRC" O="$OUT_ARM64" ARCH=arm64 \
  CROSS_COMPILE=aarch64-linux-gnu- \
  KCONFIG_CONFIG="$SRC/Microsoft/config-wsl-arm64" -j2 Image
```

预期镜像位置为 `$OUT_ARM64/arch/arm64/boot/Image`。本次环境没有 `aarch64-linux-gnu-gcc`，所以以上交叉编译命令没有实际执行，ARM64 输出不能视为已验证产物。

## 9. 在 WSL 中启用 x86_64 镜像

将项目内镜像路径填入 Windows 用户目录 `%USERPROFILE%\.wslconfig`。配置文件属于 Windows 用户级 WSL 设置：

```ini
[wsl2]
kernel=D:\\data\\git\\wsl2-mainline\\artifacts\\bzImage-7.2.10-wsl2-x86_64
```

若已经存在 `[wsl2]` 节，只增加或替换 `kernel=` 行，不要创建重复节。保存后在 PowerShell 中重启 WSL：

```powershell
wsl --shutdown
wsl -d Ubuntu
```

将 `Ubuntu` 替换成自己的发行版名称。此设置对 WSL2 发行版全局生效。恢复微软默认内核时，从 `.wslconfig` 删除或注释 `kernel=`，再执行 `wsl --shutdown` 并重启发行版。

## 10. 运行后验证

在 Ubuntu WSL 中依次检查：

```sh
uname -a
ls -l /dev/dxg
/usr/lib/wsl/lib/nvidia-smi
findmnt -T /mnt/c -o TARGET,SOURCE,FSTYPE,OPTIONS
getent hosts www.microsoft.com
findmnt -rn -t virtiofs -o TARGET,SOURCE
```

本次实测结果：

- `uname -a` 报告 `7.2.10-microsoft-standard-WSL2`，与构建的 x86_64 镜像一致。
- `/dev/dxg` 为字符设备，主/次设备号 `10, 258`。
- `nvidia-smi` 成功返回 NVIDIA-SMI `615.78.05`、KMD `617.42`、CUDA UMD `13.4`，并识别到 NVIDIA GeForce RTX 3050（4 GiB）。这验证了 DXG/NVIDIA 管理接口可用，但本次没有运行独立 CUDA 计算程序。
- 内核日志报告 Hyper-V hypervisor、Hyper-V clocksource/hypercall、VMBus 和 virtiofs `wslg` tag。VMBus 设备目录有枚举项。
- `/mnt/c` 以 `9p`/`drvfs` 挂载，且 `getent hosts www.microsoft.com` 成功。
- `/mnt/wslg` 下存在 WSLg runtime 文件和 socket；它当前显示为 `tmpfs`。本次 `findmnt -t virtiofs` 没有列出活动的 virtio-fs 挂载，因此没有验证用户共享目录上的 virtio-fs 读写。

启动日志中有 DXG 适配器/特性查询 ioctl 返回 `-22`、`-2`，也有未知 DXGK 通道 GUID 的记录。由于 `nvidia-smi` 在同一运行内核下成功识别 GPU，这些日志没有阻断已验证的 GPU 查询路径；但应在验证具体图形或 CUDA 应用时继续观察，不能据此宣称所有 DXG ioctl 或图形工作负载均已覆盖。

## 11. WSL 模块盘

本次执行了 `make modules` 并确认成功，但没有运行 `modules_install`、准备配套 headers/perf，也没有生成 `modules.vhdx`。项目中的 `Microsoft/scripts/gen_artifacts_vhdx.sh` 用于将已安装的模块树、headers 和 perf 工具打包为 WSL 可用的 VHDX；该脚本要求五个参数（模块目录、headers 目录、perf 目录、kernel version、输出文件），本次没有执行它。

因此，若依赖内核模块，请先为当前 release 安装并准备匹配的模块/headers/perf，再生成并配置相应 `modules.vhdx`。不要假定其他 WSL 内核版本的模块与该内核 ABI 兼容。当前已证明的是内核本体启动、DXG/NVIDIA 查询、Windows 驱动器挂载和 DNS 等路径。

## 12. 项目文件清单

```text
README.md                              项目简介与简要构建说明
PORTING.md                             本移植和复现全过程
Microsoft/config-wsl                   Linux 7.2.10 x86_64 WSL 配置
Microsoft/config-wsl-arm64             Linux 7.2.10 ARM64 WSL 配置
Microsoft/scripts/gen_artifacts_vhdx.sh WSL 模块/headers/perf VHDX 打包脚本
patches/0001-dxgkrnl-port.patch         DXGKRNL 驱动和 UAPI 移植
patches/0002-hyperv-arm64.patch         ARM64 Hyper-V hypercall 适配
patches/0003-virtiofs-dma.patch         virtio-fs DMA 请求大小限制
artifacts/bzImage-7.2.10-wsl2-x86_64    已构建并启动验证的 x86_64 镜像
```

完整 Linux 7.2.10 上游源码树不会复制进项目仓库：它应从输入 tarball 解包到 WSL ext4；本项目通过补丁和配置保存适配改动，便于审阅和复现。
