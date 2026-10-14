# wsl2-mainline

WSL2 kernel build configuration for Linux mainline 7.2.10.

See [PORTING.md](PORTING.md) for the complete porting record, reproduction
steps, validation results, and known limitations.

## Target

Linux 7.2.10, using the Microsoft WSL configuration as the starting point.
The x86_64 and arm64 configs in `Microsoft/` are derived from the supplied
`linux-msft-wsl-6.18.y` source snapshot. They must be reconciled against the
7.2.10 Kconfig before building; do not use the 6.18-generated files as a final
`.config` without this step.

## Source setup

The source archives are under `/mnt/d/data/linux` when accessed from WSL.
Extract Linux 7.2.10 into the Linux filesystem, then copy this repository's WSL
configuration files into the source tree:

```sh
mkdir -p "$HOME/linux-7.2.10"
tar -xf /mnt/d/data/linux/linux-7.2.10.tar.xz \
  --strip-components=1 -C "$HOME/linux-7.2.10"
cp -a /mnt/d/data/git/wsl2-mainline/Microsoft "$HOME/linux-7.2.10/"
cp -a /mnt/d/data/git/wsl2-mainline/patches "$HOME/linux-7.2.10/"
cd "$HOME/linux-7.2.10"
set -e
for patch_file in patches/*.patch; do
  patch --batch --forward -p1 < "$patch_file"
done
```

The DXGKRNL driver and UAPI are ported from the supplied Microsoft WSL
6.18 source snapshot, with Kconfig and Hyper-V GUID integration for 7.2.10.
The series also applies the WSL ARM64 Hyper-V hypercall and virtio-fs DMA
fixes. The virtio-fs DAX fix is already present in Linux 7.2.10 and is not
duplicated.

## Scope

The vendor-only `hv_pci_swiotlb=<base>,<size>` pool change from the 6.18 WSL
branch is not included: Linux 7.2.10 does not contain its underlying early
parameter implementation, so it needs a separate port. The x86_64 image has
been booted in WSL and reports `7.2.10-microsoft-standard-WSL2`; ARM64 has not
been cross-compiled or boot-tested. The modules have not been packaged into a
WSL `modules.vhdx`.

## Build

Build from a Linux filesystem (for example, inside Ubuntu on WSL). The Linux
source archive contains filenames that Windows filesystems cannot represent.
Install the standard kernel build dependencies first (`build-essential`,
`flex`, `bison`, `bc`, `libssl-dev`, `libelf-dev`, and `dwarves`).

For x86_64:

```sh
make KCONFIG_CONFIG=Microsoft/config-wsl olddefconfig
make -j"$(nproc)" KCONFIG_CONFIG=Microsoft/config-wsl
```

The WSL boot image is `arch/x86/boot/bzImage`. This workspace's verified image
is copied to `artifacts/bzImage-7.2.10-wsl2-x86_64`.

For arm64, use an AArch64 cross compiler and the matching config:

```sh
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- \
  KCONFIG_CONFIG=Microsoft/config-wsl-arm64 olddefconfig
make -j"$(nproc)" ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- \
  KCONFIG_CONFIG=Microsoft/config-wsl-arm64
```

The arm64 image is `arch/arm64/boot/Image`. Use the resulting image through
WSL's `kernel=` setting in `.wslconfig`.
