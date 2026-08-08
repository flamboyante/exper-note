# K230 canmv QEMU v4 官方 SPI-NOR 布局冷启动复现

本文独立说明如何从当前已验证基线生成 32 MiB SPI-NOR 镜像，并在不向串口输入任何
命令的情况下启动到 JFFS2 用户空间。

## 1. 能力边界

本文验证的是：

- QEMU 现有 K230 host-assisted loader 从 Flash 取出 SPL；
- SPL 和完整 U-Boot 从同一 SPI NOR 启动；
- U-Boot 从 `0x1e0000` 读取有效 environment；
- `bootcmd=k230_boot auto auto_boot` 按 SDK 官方 offset 加载配置包和
  `linux_system.bin`；
- Linux 从 `/dev/mtdblock11` 挂载 JFFS2，并进入最小用户空间。

本文不验证完整 CanMV 产品 rootfs、RT-Smart、UBI/UBIFS 或真实 BootROM。六个产品配置
分区使用 SDK 合法容器中的最小占位 payload；`rtt` 分区不执行。

## 2. 目录与已验证基线

工作区：

```text
/home/flamboy/qemu-camp
```

基线：

```text
QEMU: my-qemu-camp-2026-k230
  branch k230-coldboot-validation
  HEAD   7730876ca51b303bd424d9ae71d9a932936ba208

SDK: k230_sdk
  branch k230-coldboot-canmv-qemu
  HEAD   90f1f26b2ec079bcb4471964cf934b2fefaa34ad

boot assets: k230-boot-assets
  HEAD c3c32fb46e8307c5063f13e8f367c98bf9273cd1
```

需要的 host 工具：

- bash/sh、GNU make、git、tar、gzip、cpio、sed、`rg`
- Python 3
- Xuantie V2.6.0 RISC-V 工具链
- `mkfs.jffs2`（本轮使用 mtd-utils 2.3.1）
- SDK 自带 `mkenvimage`、`mkimage`、`k230_priv_gzip`、`firmware_gen.py`

本轮工具链目录：

```text
toolchains/Xuantie-900-gcc-linux-5.10.4-glibc-x86_64-V2.6.0/bin
```

## 3. 新增文件

SDK 内：

```text
k230_sdk/configs/k230_canmv_qemu_spinor_defconfig
k230_sdk/board/common/env/k230_canmv_qemu_spinor.jffs2.env
```

工作区根目录：

```text
toolchains/build_canmv_qemu_v4_cfg_parts.sh
toolchains/build_canmv_qemu_v4_rootfs.sh
toolchains/k230-coldboot-v4-rootfs-init
toolchains/k230-coldboot-v4-rootfs-marker
toolchains/mkcanmv_qemu_v4_flash.py
toolchains/run_canmv_qemu_v4.sh
```

注意：根目录 `toolchains/` 不属于 SDK Git 仓库，提交前必须明确其版本控制归属。

## 4. 生成顶层派生配置

以 `k230_canmv_defconfig` 为基线复制，然后只做以下差异：

```diff
-CONFIG_UBOOT_DEFCONFIG="k230_canmv"
+CONFIG_UBOOT_DEFCONFIG="k230_canmv_qemu"
-CONFIG_LINUX_DTB="k230_canmv"
+CONFIG_LINUX_DTB="k230_canmv_qemu"
-# CONFIG_SPI_NOR is not set
+CONFIG_SPI_NOR=y
+CONFIG_SPI_NOR_SUPPORT_CFG_PARAM=y
-CONFIG_SUPPORT_RTSMART=y
+# CONFIG_SUPPORT_RTSMART is not set
```

保留：

```text
CONFIG_LINUX_DEFCONFIG="k230_canmv"
CONFIG_SUPPORT_LINUX=y
CONFIG_MEM_LINUX_SYS_BASE=0x08000000
CONFIG_MEM_LINUX_SYS_SIZE=0x08000000
```

在 SDK 根目录生成配置：

```bash
cd /home/flamboy/qemu-camp/k230_sdk
make CONF=k230_canmv_qemu_spinor_defconfig prepare_memory
```

必须确认生成文件同源，不能直接手改 `sdk_autoconf.h`：

```bash
cmp include/generated/autoconf.h \
  src/little/uboot/board/canaan/common/sdk_autoconf.h

rg 'CONFIG_SPI_NOR_(LK|LR)_(BASE|SIZE)' include/generated/autoconf.h
```

期望：

```text
CONFIG_SPI_NOR_LK_BASE 0xfc0000
CONFIG_SPI_NOR_LK_SIZE 0x700000
CONFIG_SPI_NOR_LR_BASE 0x16c0000
CONFIG_SPI_NOR_LR_SIZE 0x900000
```

## 5. 从干净 U-Boot 源码构建 SPL/U-Boot

如果 SDK 工作树还有其他用户修改，不要 clean、stash 或 reset。用 `git archive` 从 HEAD
导出干净 U-Boot 子树，再只注入上一步生成的配置头：

```bash
ROOT=/home/flamboy/qemu-camp
SDK="$ROOT/k230_sdk"
BUILD_ROOT=$(mktemp -d /tmp/k230-v4-clean-uboot.XXXXXX)
UBOOT_SRC="$BUILD_ROOT/source"
UBOOT_OUT="$BUILD_ROOT/output"

mkdir -p "$UBOOT_SRC" "$UBOOT_OUT"
git -C "$SDK" archive HEAD:src/little/uboot | tar -x -C "$UBOOT_SRC"
install -m 0755 \
  "$SDK/src/little/uboot/board/canaan/common/sdk_autoconf.h" \
  "$UBOOT_SRC/board/canaan/common/sdk_autoconf.h"

export PATH="$ROOT/toolchains/Xuantie-900-gcc-linux-5.10.4-glibc-x86_64-V2.6.0/bin:$PATH"
export CROSS_COMPILE=riscv64-unknown-linux-gnu-
export ARCH=riscv

make -C "$UBOOT_SRC" O="$UBOOT_OUT" k230_canmv_qemu_defconfig
make -C "$UBOOT_OUT" -j8
```

所需输出：

```text
$UBOOT_OUT/u-boot-spl-k230-swap.bin
$UBOOT_OUT/fn_u-boot.img
```

复制到 evidence 输入目录时使用官方布局脚本期待的名称：

```bash
EVIDENCE="$ROOT/.trae/evidence/k230-coldboot-canmv-qemu-v4"
mkdir -p "$EVIDENCE"
install -m 0644 "$UBOOT_OUT/u-boot-spl-k230-swap.bin" \
  "$EVIDENCE/swap_fn_u-boot-spl.bin"
install -m 0644 "$UBOOT_OUT/fn_u-boot.img" \
  "$EVIDENCE/fn_ug_u-boot.bin"
```

## 6. 生成 SPI environment

从 SDK `board/common/env/spinor.jffs2.env` 派生 QEMU 文件，只把 rootfs 分区号改为
`mtdblock11`；原文件已经包含正确 `bootcmd` 和 `quick_boot=false`：

```text
bootcmd=k230_boot auto auto_boot;
bootargs=root=/dev/mtdblock11 rw rootwait rootfstype=jffs2 console=ttyS0,115200 earlycon=sbi;
quick_boot=false
```

生成 64 KiB environment：

```bash
"$SDK/src/little/uboot/tools/mkenvimage" -s 0x10000 \
  -o "$EVIDENCE/jffs2.env" \
  "$SDK/board/common/env/k230_canmv_qemu_spinor.jffs2.env"
```

environment 分区为 128 KiB；装配器只写入前 64 KiB，其余保持 `0xff`。

## 7. 生成六个合法配置占位包

```bash
"$ROOT/toolchains/build_canmv_qemu_v4_cfg_parts.sh" \
  "$SDK" "$EVIDENCE/cfg_part"
```

脚本生成：

```text
fn_ug_quick_boot.bin
fn_ug_face_data.bin
fn_ug_sensor_cfg.bin
fn_ug_ai_mode.bin
fn_ug_speckle.bin
fn_ug_fastboot_app.elf
rtt_system.bin
```

前六项是带 gzip uImage 和 K230 firmware header 的合法一字节 payload；最后一项是
关闭 RT-Smart 后不会执行的最小占位文件。

## 8. 生成 OpenSBI payload 与 linux_system.bin

本轮 Linux payload 使用：

```text
k230-boot-assets/buildroot/uboot-boot/Image
SHA256 113b5d2c01526566dd9ef37abb57788c9510f3c253b68276c784284d3e2b5cff
```

构建 OpenSBI：

```bash
OPENSBI_OUT="$EVIDENCE/opensbi-build"
make -C "$SDK/src/common/opensbi" \
  O="$OPENSBI_OUT" \
  PLATFORM=generic \
  CROSS_COMPILE=riscv64-unknown-linux-gnu- \
  FW_PAYLOAD_PATH="$ROOT/k230-boot-assets/buildroot/uboot-boot/Image"

install -m 0644 \
  "$OPENSBI_OUT/platform/generic/firmware/fw_payload.bin" \
  "$EVIDENCE/fw_payload.bin"
```

使用 Linux QEMU DTS 生成 DTB：

```bash
LINUX="$SDK/src/little/linux"
cpp -nostdinc -I "$LINUX/include" -I "$LINUX/arch" \
  -undef -x assembler-with-cpp \
  "$LINUX/arch/riscv/boot/dts/kendryte/k230_canmv_qemu.dts" | \
  "$LINUX/scripts/dtc/dtc" -I dts -O dtb \
  -o "$EVIDENCE/k230_canmv_qemu.dtb"
```

按 SDK 的 multi-uImage + K230 header 链打包：

```bash
LINUX_PACK="$EVIDENCE/linux-pack"
mkdir -p "$LINUX_PACK"
install -m 0644 "$EVIDENCE/fw_payload.bin" "$LINUX_PACK/fw_payload.bin"

(
  cd "$LINUX_PACK"
  "$SDK/tools/k230_priv_gzip" -n8 -f -k fw_payload.bin
  sed -i -e '1s/\x08/\x09/' fw_payload.bin.gz
  printf 'a\n' > rd
  "$SDK/src/little/uboot/tools/mkimage" \
    -A riscv -O linux -T multi -C gzip \
    -a 0x8000000 -e 0x8000000 -n linux \
    -d "fw_payload.bin.gz:rd:$EVIDENCE/k230_canmv_qemu.dtb" \
    ulinux.bin
  python3 "$SDK/tools/firmware_gen.py" \
    -i ulinux.bin -o "$EVIDENCE/linux_system.bin" -n
)

test "$(stat -c %s "$EVIDENCE/linux_system.bin")" -le $((0x700000))
```

`mkimage` 对真正的空 `rd` 会直接报 `Input file rd is empty`；该失败保留在
`linux-system-build.log:1` 作为 red 证据。最终采用 2 字节 `a\n` 占位段，Linux 不把它
作为 initramfs 使用，且 `linux_system.bin` 格式和启动结果均已重新验证。

## 9. 生成最小 JFFS2 rootfs

本轮复用的基线 archive：

```text
k230-boot-assets/yocto/uboot-boot/rootfs.cpio.gz
SHA256 4e1869a99a232ee60324f71f3a9e84a79b03ccabb5b73f8a727c5ff5be5c0914
```

生成命令：

```bash
MKFS_JFFS2=${MKFS_JFFS2:-$(command -v mkfs.jffs2)}
test -n "$MKFS_JFFS2" -a -x "$MKFS_JFFS2"
"$ROOT/toolchains/build_canmv_qemu_v4_rootfs.sh" \
  "$ROOT/k230-boot-assets/yocto/uboot-boot/rootfs.cpio.gz" \
  "$MKFS_JFFS2" \
  "$EVIDENCE/rootfs.jffs2"

test "$(stat -c %s "$EVIDENCE/rootfs.jffs2")" -le $((0x900000))
```

脚本先把 Yocto archive 裁剪到 BusyBox shell、init 所需 applet、动态 loader、`libc`
和 `libm`，再安装相同的启动文件到 `/init` 和 `/sbin/init`，并写入
`/etc/K230_V4_JFFS2_ROOTFS_OK`。`mkfs.jffs2` 使用 64 KiB erase block 与 4 KiB
page size；不要使用 `-s 1`，否则该 rootfs 会膨胀到约 252 MiB。

## 10. 装配严格 32 MiB Flash

```bash
python3 "$ROOT/toolchains/mkcanmv_qemu_v4_flash.py" \
  "$EVIDENCE" \
  "$EVIDENCE/k230_canmv_qemu_v4_spinor32m.img" \
  "$EVIDENCE/manifest.txt" \
  --metadata "$EVIDENCE/manifest_metadata.txt"
```

装配器逐段检查 K230 header、JFFS2 magic、实际大小、分区重叠和 Flash 越界，并把未写
区域初始化为 `0xff`。最终必须满足：

```bash
test "$(stat -c %s "$EVIDENCE/k230_canmv_qemu_v4_spinor32m.img")" \
  -eq $((0x2000000))
```

当前已验证最终镜像：

```text
size   33554432
SHA256 e7728c24a7edef22c85d57dcd41c7c429de974a7337fde5d4eb02b0da689911a
```

由于 gzip/uImage 和 JFFS2 可包含构建时间信息，重建哈希可能变化；每次应以新生成的
`manifest.txt` 为准。

## 11. 无人干预启动

推荐直接使用 runner：

```bash
cd "$ROOT"
toolchains/run_canmv_qemu_v4.sh \
  "$EVIDENCE/k230_canmv_qemu_v4_spinor32m.img" \
  "$EVIDENCE/v4-qemu-no-serial.log"
```

runner 等价于：

```bash
timeout 60s my-qemu-camp-2026-k230/build/qemu-system-riscv64 \
  -M k230,spi-flash=w25q256 \
  -drive "if=mtd,format=raw,file=$EVIDENCE/k230_canmv_qemu_v4_spinor32m.img" \
  -nographic -monitor none -display none -no-reboot -snapshot \
  </dev/null
```

成功时 runner 返回 0。QEMU 自身必须返回 124，因为 guest 到 shell 后保持运行，host
timeout 负责结束采集；其他退出状态直接失败。`-snapshot` 保证 JFFS2 运行时写入不改变
交付镜像，运行前后 SHA256 必须一致。

## 12. 成功判据

日志必须同时满足：

```text
stdin=/dev/null
Loading Environment from SPIFlash
无 bad CRC
quick_boot_cfg / face_db / sensor_cfg / ai_mode / speckle / rtapp 均出现 load
imge: linux load to 8000000
不存在 off=0xffffffff
Kernel command line 包含 root=/dev/mtdblock11 和 rootfstype=jffs2
VFS: Mounted root (jffs2 filesystem) on device 31:11.
K230_V4_JFFS2_ROOTFS_OK
K230 v4 JFFS2 userspace ready
```

## 13. 分阶段排错

| 停止位置 | 优先检查 |
|---|---|
| 无 `k230 coldboot: magic ok` | SPL K230 header、endian swap、Flash offset 0 |
| SPL 未到 U-Boot | `0x80000` U-Boot 分区、gzip/K230 header、SPI NOR boot mode |
| U-Boot 停在 autoboot | environment CRC、`0x1e0000`、`bootcmd` |
| 配置包 header/load error | 六个 `fn_ug_*` 文件是否由 helper 生成、offset 是否匹配 manifest |
| `off=0xffffffff` | `sdk_autoconf.h` 是否由派生顶层配置重新生成 |
| Linux 容器错误 | `linux_system.bin` K230 header、multi-uImage 三段、7 MiB 上限 |
| Linux 起但不挂根 | DTS MTD 分区、`mtdblock11`、JFFS2 驱动和 magic |
| 15 秒停在配置包后 | 不是足够稳定的门限；使用已验证的 60 秒 |

U-Boot 的 `Software reset enable failed: -524` 在本基线中是非阻断日志；只有后续无法识别
w25q256、无法读 environment 或无法继续启动时，才把它升级为失败。

## 14. v3 回归

v4 完成后还应运行历史 v3 raw/initramfs runner。已验证日志必须出现：

```text
Run /init as init process
meta-k230 initramfs starting...
Dropping to shell...
```

本轮 v3 runner 返回 0。v3 的旧 environment bad CRC 和旧布局
`off=0xffffffff` 只属于历史链路，不得与 v4 门禁混用。
