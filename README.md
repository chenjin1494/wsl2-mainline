# Linux 7.2.10 WSL2 内核适配

本项目将 Linux 主线 7.2.10 适配为可在 WSL2 中使用的内核，提供 DXGKRNL GPU 驱动、WSL 配置、移植补丁和复现说明。

从头复现、查看完整移植记录及验证范围，请阅读[移植与复现指南](PORTING.md)。

## 支持范围

- x86_64 内核已构建并在 WSL2 中启动验证。
- DXGKRNL 已编入 x86_64 内核；实测 `/dev/dxg` 存在，`nvidia-smi` 可以识别 NVIDIA GPU。
- ARM64 配置和 Hyper-V 补丁已准备，但尚未完成交叉编译或启动验证。
- 内核模块已构建，但尚未安装并制作为 WSL `modules.vhdx`。
- virtio-fs DMA 修复已移植并参与编译；当前没有实际 virtio-fs 共享挂载可用于读写测试。

## 快速构建

在 WSL Ubuntu 或其他 Linux 环境中，安装内核构建工具，克隆本项目，并把源码和构建输出放在 Linux 文件系统中：

```sh
sudo apt update
sudo apt install build-essential flex bison bc m4 libssl-dev libelf-dev dwarves git curl xz-utils

git clone https://github.com/chenjin1494/wsl2-mainline.git wsl2-mainline

export PROJECT="$HOME/wsl2-mainline"
export WORK="$HOME/wsl2-kernel-build"
export SRC="$WORK/linux-7.2.10"
export OUT="$WORK/out-x86"
mkdir -p "$SRC" "$WORK/downloads"

curl -fL 'https://cdn.kernel.org/pub/linux/kernel/v7.x/linux-7.2.10.tar.xz' \
  -o "$WORK/downloads/linux-7.2.10.tar.xz"
tar -xf "$WORK/downloads/linux-7.2.10.tar.xz" \
  --strip-components=1 -C "$SRC"

cp -a "$PROJECT/Microsoft" "$SRC/"
cp -a "$PROJECT/patches" "$SRC/"
cd "$SRC"
set -eu
for patch_file in patches/*.patch; do
  patch --batch --forward -p1 < "$patch_file"
done

make -C "$SRC" O="$OUT" \
  KCONFIG_CONFIG="$SRC/Microsoft/config-wsl" olddefconfig
make -C "$SRC" O="$OUT" \
  KCONFIG_CONFIG="$SRC/Microsoft/config-wsl" -j2 bzImage modules
```

构建完成后，先把镜像复制到 Windows 用户目录，再在 Windows 的 `.wslconfig` 中引用它。可在 WSL 中执行以下命令（要求 WSL interop 和 Windows 用户目录挂载可用）：

```sh
WIN_PROFILE=$(powershell.exe -NoProfile -Command '$env:USERPROFILE' | tr -d '\r')
WIN_PROFILE=$(wslpath "$WIN_PROFILE")
mkdir -p "$WIN_PROFILE/wsl-kernels"
cp "$OUT/arch/x86/boot/bzImage" "$WIN_PROFILE/wsl-kernels/bzImage-7.2.10"
wslpath -w "$WIN_PROFILE/wsl-kernels/bzImage-7.2.10" | sed 's/\\/\\\\/g'
```

最后一条命令输出已按 `.wslconfig` 格式转义的 Windows 路径。将实际输出写入 Windows 用户目录的 `.wslconfig`：

```ini
[wsl2]
kernel=<上一条命令输出的 Windows 完整路径>
```

不要把尖括号占位文字原样写入配置。如果已有 `[wsl2]` 节，只修改 `kernel=`，不要重复创建该节。然后在 PowerShell 重启 WSL：

```powershell
wsl --shutdown
wsl -d Ubuntu
```

在发行版中运行 `uname -r`，确认输出包含 `7.2.10-microsoft-standard-WSL2`。`wsl --version` 显示的是 WSL 安装包报告的版本，不一定是当前发行版正在运行的自定义内核版本。

## 项目内容

- `Microsoft/config-wsl`：x86_64 WSL 内核配置。
- `Microsoft/config-wsl-arm64`：ARM64 WSL 内核配置。
- `patches/`：DXGKRNL、ARM64 Hyper-V 和 virtio-fs DMA 补丁系列。
- `Microsoft/scripts/gen_artifacts_vhdx.sh`：打包 WSL 模块、头文件和 perf 工具的脚本。
- `artifacts/bzImage-7.2.10-wsl2-x86_64`：本项目已构建并启动验证的 x86_64 镜像。
- `PORTING.md`：移植过程、可复现步骤、排障记录和已知限制。

完整源码树不存放在本项目内；复现时从 kernel.org 下载上游源码，在 Linux 文件系统中解包后应用本项目补丁。请参阅[完整指南](PORTING.md)了解依赖、验证和模块盘等细节。
