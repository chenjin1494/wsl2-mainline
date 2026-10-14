# Linux 7.2.10 WSL2 移植与复现指南

本文记录本项目将 Linux 主线 7.2.10 适配到 WSL2 的移植内容、复现步骤、启用方法、排障过程和验证结果。命令使用 Linux 环境中的 `$HOME` 和项目目录变量，不依赖特定用户名、盘符或本机文件夹。

## 1. 项目状态与验证边界

- x86_64 Linux 7.2.10 WSL2 内核及配置要求的可加载模块均已成功构建。
- x86_64 内核已在 WSL2 中启动，`uname -r` 报告 `7.2.10-microsoft-standard-WSL2`。
- `/dev/dxg` 存在，`nvidia-smi` 成功识别 NVIDIA GeForce RTX 3050。
- Hyper-V/VMBus 启动、Windows 驱动器挂载和 DNS 解析已通过检查。
- ARM64 配置已用 Linux 7.2.10 的 Kconfig 规范化，但当时没有 AArch64 交叉编译器，故没有完成 ARM64 编译或启动测试。
- 已构建 `.ko` 模块，但尚未安装并打包为 WSL `modules.vhdx`。
- virtio-fs DMA 修复已参与内核编译；测试环境没有活动的 virtio-fs 共享挂载，尚未验证该共享路径的实际读写。
- 本次未运行独立 CUDA 计算程序。`nvidia-smi` 成功说明 DXG/NVIDIA 管理查询通路可用，不代表所有图形和 CUDA 工作负载均已验证。

## 2. 移植内容与取舍

### 2.1 DXGKRNL 图形驱动

`patches/0001-dxgkrnl-port.patch` 从 Microsoft WSL Linux 6.18.54.1 源码快照移植 DXGKRNL 驱动和 D3DKMT UAPI，并接入 Linux 7.2.10：

- 加入 `drivers/hv/dxgkrnl/` 驱动实现、Kconfig 和 Makefile。
- 加入 `include/uapi/misc/d3dkmthk.h` 与 `include/uapi/linux/dxgkrnl.h`。
- 将 DXGKRNL 接入 `drivers/hv/Kconfig` 和 `drivers/hv/Makefile`。
- 在 `include/linux/hyperv.h` 中加入 DXGK 全局通道和虚拟 GPU 通道的 GUID 定义。
- x86_64 和 ARM64 配置均将 `CONFIG_DXGKRNL` 设为 `y`，即将驱动编入内核。

首次 DXGKRNL 子目录编译发现目标 7.2.10 的 Hyper-V 头文件缺少 DXGK GUID 定义。补入两个 GUID 后，DXGKRNL 子目录编译通过。最终补丁也在干净的 Linux 7.2.10 基线上验证过可应用。

### 2.2 ARM64 Hyper-V hypercall

`patches/0002-hyperv-arm64.patch`（原补丁标识 `712c88941ba7cfd3c9eec3a58eef4c231226d66f`）让 ARM64 Hyper-V 使用 Hyper-V hypercall 接口，而非依赖并非所有 Windows 版本都支持的 SMCCC 接口。补丁加入 HVC 汇编实现并调整 Hyper-V 调用路径。该补丁已应用、ARM64 配置已规范化，但本项目尚无 ARM64 编译结果。

### 2.3 virtio-fs DMA 限制

`patches/0003-virtiofs-dma.patch`（原补丁标识 `3586f43a9a33f5f12306de85b51f696194778156`）在 FUSE virtio-fs 初始化时，按照传输层单次 DMA mapping 上限约束 `max_pages_limit`，防止请求缓冲区超过受限 DMA 映射能力。完整 x86_64 构建成功编译了 `fs/fuse/virtio_fs.o`。

Linux 7.2.10 已包含 virtio-fs DAX 修复，故没有重复移植。Hyper-V timer 修复也已在目标主线中，不重复添加。

### 2.4 未纳入的下游变更

Microsoft 6.18 分支中的 `hv_pci_swiotlb=<base>,<size>` 专用池改动没有纳入。它依赖 Linux 7.2.10 中不存在的早期参数及底层实现；下游补丁直接应用到目标树时有多个 hunk 无法匹配。需要另行按 7.2.10 的接口移植和验证，不应强行拼接成不完整实现。

### 2.5 移植过程中的故障与处理

- Windows 文件系统无法表示部分 Linux 源码文件名。最初在 Windows 文件系统上解包失败；把源码和构建输出放到 WSL 的 Linux 文件系统后解决。
- 初始构建环境缺少 `bison` 和 `gelf.h`。移植时没有安装系统软件包，而是使用本地解包的构建依赖；一般复现可直接安装发行版开发包。
- objtool 曾链接到静态 libelf，产生未解析的 `ZSTD_*` 和 `deflate*` 符号。改为让本地依赖路径指向系统动态 libelf 后解决；此 workaround 仅适用于遇到相同错误的本地依赖环境。
- DXGKRNL 首次编译缺少两个 Hyper-V DXGK GUID。补入定义后驱动编译通过。
- 生成 DXGKRNL 补丁期间，一次零上下文修改把 GUID 定义插入了已有注释，导致语法错误。修正头文件、重新生成补丁后，在干净基线上确认补丁能应用并编译。
- Microsoft 6.18 的 `hv_pci_swiotlb` 补丁依赖目标树不存在的实现且无法完全匹配，因此没有强行移植。

## 3. 目录约定

以下目录名均由复现者自行设置。示例把项目克隆和 Linux 源码、构建输出分别放在 `$HOME` 下：

```text
$HOME/
├── wsl2-mainline/       # 本项目克隆
└── wsl2-kernel-build/
    ├── downloads/       # 下载的上游源码压缩包
    ├── linux-7.2.10/    # 解包并应用本项目补丁后的内核源码
    ├── out-x86/         # x86_64 构建输出
    └── out-arm64/       # ARM64 构建输出
```

源码和输出目录应位于 Linux 文件系统（例如 WSL Ubuntu 的 ext4 家目录），不要放在 Windows 挂载盘或其他不支持 Linux 文件名/符号链接语义的文件系统上。

## 4. 准备工具

在 Ubuntu WSL 或其他 Debian/Ubuntu Linux 环境中安装构建工具：

```sh
sudo apt update
sudo apt install build-essential flex bison bc m4 libssl-dev libelf-dev dwarves git curl xz-utils python3
```

这些是常见构建依赖。不同发行版的包名可能略有区别；遇到构建提示缺少其他工具时，按发行版包管理器补装相应开发包。

本次构建使用 GCC 15.2.0 与 GNU Binutils 2.46。多数现代 Linux 发行版可使用自身的原生 x86_64 GCC 和 Binutils，不要求版本字符串完全一致；如遇到编译器版本或警告升级为错误，应先确认目标内核和工具链的兼容性。

### 4.1 无 sudo 的本地依赖（可选）

若不能安装系统软件包，可下载与当前发行版匹配的 `.deb` 包，使用 `dpkg-deb -x` 解包到独立目录。包名、依赖和库目录会因发行版及 CPU 架构不同而变化，因此不要复制其他机器的包路径。准备好后按实际位置设置环境变量，例如：

```sh
export WORK="$HOME/wsl2-kernel-build"
export LOCAL="$WORK/local-deps"
export PATH="$LOCAL/usr/bin:/usr/bin:/bin"
export M4="$LOCAL/usr/bin/m4"
export BISON_PKGDATADIR="$LOCAL/usr/share/bison"
export C_INCLUDE_PATH="$LOCAL/usr/include"
export LIBRARY_PATH="$LOCAL/usr/lib/x86_64-linux-gnu"
```

上面的本地目录结构仅供示例；应按实际解包布局调整。遇到 objtool 选择静态 libelf 并出现未解析 `ZSTD_*`/`deflate*` 符号时，可检查系统是否提供动态 libelf，并在本地依赖目录中配置正确链接。不要在未遇到该问题时盲目添加符号链接。

## 5. 获取本项目和 Linux 源码

先设置本项目克隆地址。将占位值替换为项目托管平台给出的 HTTPS 或 SSH 克隆地址：

```sh
git clone https://github.com/chenjin1494/wsl2-mainline.git "$HOME/wsl2-mainline"
export PROJECT="$HOME/wsl2-mainline"
```

从 kernel.org 获取 Linux 7.2.10 主线源码，并解包到 Linux 文件系统：

```sh
export WORK="$HOME/wsl2-kernel-build"
export SRC="$WORK/linux-7.2.10"
mkdir -p "$WORK/downloads" "$SRC"

curl -fL 'https://cdn.kernel.org/pub/linux/kernel/v7.x/linux-7.2.10.tar.xz' \
  -o "$WORK/downloads/linux-7.2.10.tar.xz"
tar -xf "$WORK/downloads/linux-7.2.10.tar.xz" \
  --strip-components=1 -C "$SRC"
```

项目补丁已由 Microsoft WSL 6.18.54.1 源码快照移植生成。复现补丁应用和构建不需要另行下载这份参考快照；若要审阅上游参考源码，可从 Microsoft 官方 WSL2-Linux-Kernel 仓库获取对应的 `linux-msft-wsl-6.18.y` 源码。参考源码只用于理解移植来源，构建目标仍是干净的 Linux 7.2.10。

## 6. 应用移植补丁

把项目配置和补丁复制到目标源码树，再按文件名顺序应用补丁：

```sh
cp -a "$PROJECT/Microsoft" "$SRC/"
cp -a "$PROJECT/patches" "$SRC/"
cd "$SRC"
set -eu
for patch_file in patches/*.patch; do
  patch --batch --forward -p1 < "$patch_file"
done
```

`set -eu` 可确保某个补丁失败时立即停止，避免继续构建只应用部分补丁的源码。不要在已成功打补丁的源码树上重复运行；需要从头复现时，先从原始压缩包重新解包到空目录。

补丁和配置可以检查 DXGKRNL 是否启用：

```sh
grep '^CONFIG_DXGKRNL=y$' "$SRC/Microsoft/config-wsl"
grep '^CONFIG_DXGKRNL=y$' "$SRC/Microsoft/config-wsl-arm64"
```

## 7. 规范化配置

配置最初以 Microsoft WSL 6.18 配置为基础。必须在打过补丁的 Linux 7.2.10 源码树中运行 `olddefconfig`，由目标版本的 Kconfig 移除失效选项并补充新选项。不要直接把其他内核版本生成的 `.config` 当作 Linux 7.2.10 的最终配置。

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

确认内核 release：

```sh
make -s -C "$SRC" O="$OUT_X86" \
  KCONFIG_CONFIG="$SRC/Microsoft/config-wsl" kernelrelease
```

预期结果：

```text
7.2.10-microsoft-standard-WSL2
```

本项目的 ARM64 `olddefconfig` 已成功运行，release 字符串也已确认；这只代表配置可解析，不代表 ARM64 内核已编译或启动。

## 8. 构建 x86_64 内核和模块

使用与配置相同的输出目录和 `KCONFIG_CONFIG` 构建。已验证的构建命令如下：

```sh
make -C "$SRC" O="$OUT_X86" \
  KCONFIG_CONFIG="$SRC/Microsoft/config-wsl" -j2 bzImage

make -C "$SRC" O="$OUT_X86" \
  KCONFIG_CONFIG="$SRC/Microsoft/config-wsl" -j2 modules
```

`-j2` 表示并行使用两个构建任务；可按机器资源调整并行数。构建成功后检查镜像：

```sh
file "$OUT_X86/arch/x86/boot/bzImage"
ls -lh "$OUT_X86/arch/x86/boot/bzImage"
```

内核镜像位于：

```text
$OUT_X86/arch/x86/boot/bzImage
```

`bzImage` 是压缩的 x86 启动内核镜像，不是源码树，也不包含所有模块。启动时内核会解压。`CONFIG_DXGKRNL=y` 将 DXGKRNL 编入内核；配置为 `=m` 的驱动则作为独立 `.ko` 文件生成。

本项目附带一份已构建并启动验证的镜像：

```text
artifacts/bzImage-7.2.10-wsl2-x86_64
```

该文件约 14 MiB。本次构建产物的 SHA-256 为：

```text
a1d0765dea1c3fe57bdabee382cb35f0725f750f186ae4179d8beaa17724f29f
```

重建时内核时间戳等信息会变化，故新构建文件的 SHA-256 不一定相同。`make modules` 已成功完成，但模块尚未安装或打包为 WSL 模块盘。

## 9. ARM64 构建（未实测）

ARM64 构建需要 AArch64 交叉编译器。Ubuntu 上可安装：

```sh
sudo apt install gcc-aarch64-linux-gnu binutils-aarch64-linux-gnu
```

规范化 ARM64 配置后，执行：

```sh
make -C "$SRC" O="$OUT_ARM64" ARCH=arm64 \
  CROSS_COMPILE=aarch64-linux-gnu- \
  KCONFIG_CONFIG="$SRC/Microsoft/config-wsl-arm64" -j2 Image
```

预期镜像位置为 `$OUT_ARM64/arch/arm64/boot/Image`。本项目开发环境没有 AArch64 编译器，以上编译命令尚未实测，不能把 ARM64 输出视为已验证产物。

## 10. 在 WSL 中启用内核

构建源码应留在 Linux 文件系统。配置 WSL 使用内核时，需要把镜像复制到 Windows 用户目录，再将它的 Windows 路径写入 `.wslconfig`。在 WSL 中执行以下命令（要求 WSL interop 可用）：

```sh
WIN_PROFILE=$(powershell.exe -NoProfile -Command '$env:USERPROFILE' | tr -d '\r')
WIN_PROFILE=$(wslpath "$WIN_PROFILE")
mkdir -p "$WIN_PROFILE/wsl-kernels"
cp "$OUT_X86/arch/x86/boot/bzImage" "$WIN_PROFILE/wsl-kernels/bzImage-7.2.10"
wslpath -w "$WIN_PROFILE/wsl-kernels/bzImage-7.2.10" | sed 's/\\/\\\\/g'
```

最后一条命令会打印已按 `.wslconfig` 格式转义的 Windows 完整路径。复制实际输出，在 Windows 用户目录的 `.wslconfig` 中填写：

```ini
[wsl2]
kernel=<上一条命令输出的 Windows 完整路径>
```

如果 WSL 禁用了 interop，可通过 Windows 文件资源管理器，从 WSL 网络共享 `\\wsl.localhost\<发行版名称>\...` 把镜像复制到 Windows 用户目录，然后使用镜像的 Windows 完整路径。不要把尖括号占位文字原样写入配置。

如果已有 `[wsl2]` 节，只需修改其中的 `kernel=`，不要重复添加节。然后在 PowerShell 中关闭并重启 WSL：

```powershell
wsl --shutdown
wsl -d Ubuntu
```

将 `Ubuntu` 替换为自己的发行版名称。此配置对 WSL2 发行版生效。恢复 WSL 自带的默认内核时，删除 `.wslconfig` 中的 `kernel=` 行，再执行 `wsl --shutdown` 并重新启动发行版。

在发行版中查看实际运行内核：

```sh
uname -r
```

预期包含：

```text
7.2.10-microsoft-standard-WSL2
```

`wsl --version` 显示的是 WSL 安装包报告的内核版本，不一定是发行版当前运行的内核。验证自定义内核应以发行版中的 `uname -r` 为准。

## 11. 功能验证

在 WSL 发行版中运行基础检查：

```sh
uname -a
ls -l /dev/dxg
/usr/lib/wsl/lib/nvidia-smi
findmnt -rn -t 9p -o TARGET,SOURCE,FSTYPE,OPTIONS
getent hosts www.microsoft.com
findmnt -rn -t virtiofs -o TARGET,SOURCE
```

工具或挂载点可能因发行版、Windows 配置和硬件不同而不存在；缺少 NVIDIA GPU 时，`nvidia-smi` 检查不适用。

本项目开发机器的实测结果：

- `uname -a` 报告 `7.2.10-microsoft-standard-WSL2`。
- `/dev/dxg` 是字符设备，主/次设备号为 `10, 258`。
- `/usr/lib/wsl/lib/nvidia-smi` 报告 NVIDIA-SMI `615.78.05`、KMD `617.42`、CUDA UMD `13.4`，识别到 NVIDIA GeForce RTX 3050（4 GiB）。没有运行独立 CUDA 计算程序。
- 内核日志报告检测到 Microsoft Hyper-V，Hyper-V clocksource/hypercall 和 VMBus 已启动；VMBus 设备目录中存在设备。
- Windows 系统盘挂载并可访问，DNS 查询成功。
- WSLg runtime 文件和 socket 存在；`/mnt/wslg` 显示为 `tmpfs`。virtio-fs `wslg` tag 出现在内核日志中，但没有活动的 `virtiofs` 挂载，所以尚未验证共享目录读写。

启动日志中出现若干 DXG 适配器/特性查询 ioctl 返回 `-22`、`-2`，并有未知 DXGK 通道 GUID 记录。`nvidia-smi` 在同一内核下仍成功识别 GPU，因此这些日志未阻断已验证的管理查询路径；仍需用目标图形或 CUDA 工作负载验证更完整功能。

## 12. WSL 模块盘

本次 `make modules` 已成功，但没有运行 `modules_install`、准备匹配的 headers/perf，也没有生成 `modules.vhdx`。项目脚本 `Microsoft/scripts/gen_artifacts_vhdx.sh` 可将已安装的模块树、头文件和 perf 工具打包为 WSL VHDX；脚本需要五个参数：模块目录、headers 目录、perf 目录、内核版本和输出文件。请先阅读脚本并准备与当前内核 release 完全匹配的文件，再生成模块盘。

不要假定其他 WSL 内核版本的模块与本内核 ABI 兼容。需要加载模块时，须为本次内核安装并打包对应的模块和工具。

## 13. 项目文件

```text
README.md                               中文项目首页和快速构建说明
PORTING.md                              完整移植记录与复现指南
Microsoft/config-wsl                    x86_64 WSL 内核配置
Microsoft/config-wsl-arm64              ARM64 WSL 内核配置
Microsoft/scripts/gen_artifacts_vhdx.sh WSL 模块/headers/perf VHDX 打包脚本
patches/0001-dxgkrnl-port.patch          DXGKRNL 驱动和 UAPI 移植
patches/0002-hyperv-arm64.patch          ARM64 Hyper-V hypercall 适配
patches/0003-virtiofs-dma.patch          virtio-fs DMA 请求大小限制
artifacts/bzImage-7.2.10-wsl2-x86_64     已构建并启动验证的 x86_64 镜像
```

完整 Linux 上游源码树不存放在本项目中。复现时从 kernel.org 下载源码、在 Linux 文件系统中解包，并应用本项目补丁和配置。
