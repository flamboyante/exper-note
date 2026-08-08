# K230 QEMU 冷启动全流程：从 0 到 Linux initramfs shell

记录时间：2026-08-05 ｜ 状态：已验证（官方工具链重建 SPL/U-Boot；内核可复用 boot-assets 的 SDK 5.10.4 标准 PTE 重编版，BootROM → SPL → U-Boot → OpenSBI → Linux initramfs）

> 本文是"从零复现 K230 冷启动"的可操作指南。内核两条可选路径：yocto 6.18.28 或 SDK 5.10.4（见第 7 节）。所有命令以工作区根目录
> `/home/flamboy/qemu-camp` 为基准，在 WSL2（Ubuntu22.04）内执行。
> 知识背景见 [k230-cold-boot-handbook.md](./k230-cold-boot-handbook.md)。

## 验证结果（2026-08-05，官方 Xuantie 工具链）

完整冷启动链在 QEMU 上跑通，最终进入 initramfs shell（完整日志
`/tmp/k230-coldboot-full.log`）：

```text
k230 coldboot: enter
k230 coldboot: magic ok body=224356 spl=224352 -> 0x80300000
U-Boot SPL 2022.10 (Aug 05 2026 - 23:03:11)   <- 官方工具链重建
...
U-Boot 2022.10 (Aug 05 2026 - 23:03:11)  Model: kendryte k230 evb
K230# sf probe / sf read x4 / fdt set / bootm
OpenSBI v0.9
[    0.000000] Linux version 6.18.28
meta-k230 initramfs starting...
Dropping to shell...
~ #
```

## 0. 目录速览

| 项 | 路径 |
|---|---|
| 官方工具链 | `/home/flamboy/qemu-camp/toolchains/`（SDK 外） |
| 工具链软链接 | `/opt/toolchain` → `toolchains/`（deconfig 默认查找） |
| 新参考 dts | `k230_sdk/src/little/uboot/arch/riscv/dts/k230_canmv_qemu.dts`、`k230_sdk/src/little/linux/arch/riscv/boot/dts/kendryte/k230_canmv_qemu.dts` |
| dts/dt b/defconfig 归档 | `resource/dts/{original,new,dtb}` |
| 官方工具链构建产物归档 | `resource/bin/`（u-boot.bin、u-boot-spl.bin、swap_fn_u-boot-spl.bin、fn_ug_u-boot.bin） |
| QEMU 冷启动分支 | `my-qemu-camp-2026-k230` 分支 `k230-coldboot-validation`（commit `7730876ca5`） |
| QEMU 二进制 | `my-qemu-camp-2026-k230/build/qemu-system-riscv64` |
| 启动镜像资产 | `k230-boot-assets/common/fw_jump.uImage`、`k230-boot-assets/yocto/uboot-boot/{Image,rootfs.cpio.gz,k230-canmv.dtb}` |

## 1. 工具链准备

官方工具链放到 SDK 之外的 `toolchains/`，并映射 `/opt/toolchain` 软链接，
使 deconfig 中的 `/opt/toolchain/...` 路径直接可用：

```bash
mkdir -p /home/flamboy/qemu-camp/toolchains && cd /home/flamboy/qemu-camp/toolchains

# Linux 工具链（Xuantie，前缀 riscv64-unknown-linux-gnu-）
wget -q -O Xuantie-900-gcc-linux-5.10.4-glibc-x86_64-V2.6.0.tar.bz2 \
  https://kendryte-download.canaan-creative.com/k230/toolchain/Xuantie-900-gcc-linux-5.10.4-glibc-x86_64-V2.6.0.tar.bz2
tar jxf Xuantie-900-gcc-linux-5.10.4-glibc-x86_64-V2.6.0.tar.bz2

# RTT 工具链（rtsmart 用；本次不编译 rtsmart，但 deconfig 检查需要文件存在）
wget -q -O riscv64-unknown-linux-musl-rv64imafdcv-lp64d-20230420.tar.bz2 \
  https://kendryte-download.canaan-creative.com/k230/toolchain/riscv64-unknown-linux-musl-rv64imafdcv-lp64d-20230420.tar.bz2
tar jxf riscv64-unknown-linux-musl-rv64imafdcv-lp64d-20230420.tar.bz2

# /opt/toolchain 软链接（WSL 可用 root 免密）
wsl -d Ubuntu22.04 -u root -- ln -s /home/flamboy/qemu-camp/toolchains /opt/toolchain
```

验证（deconfig 的两个路径必须同时存在）：

```bash
ls /opt/toolchain/Xuantie-900-gcc-linux-5.10.4-glibc-x86_64-V2.6.0/bin/riscv64-unknown-linux-gnu-gcc
ls /opt/toolchain/riscv64-linux-musleabi_for_x86_64-pc-linux-gnu/bin/riscv64-unknown-linux-musl-gcc
```

> 注意 1：官方 Xuantie-900 V2.6.0 binutils 较老，**不支持**
> `-march=..._zicsr_zifencei` 后缀。之前为新工具链加的
> `linux/arch/riscv/Makefile` `_zicsr_zifencei` hack 必须回退；而
> `mhcr/mcor` 数字 CSR hack（`csrr 0x7c1/0x7c2`）两种汇编器都接受，可保留。
>
> 注意 2：canmv 板级 SPL 固定从 SDIO/MMC 启动（`board.c` 覆盖 boot mode），
> 冷启动 SPI NOR 需用 **k230_evb** defconfig（原生读 boot strap → SPI NOR）。
> `k230_canmv_qemu.dts` 保留为 canmv 家族参考，且已单独编译验证
> （`resource/dts/dtb/k230_canmv_qemu.dtb` 内含 spi0 okay + spi-flash@0 +
> bus-width 1）。

## 2. 参考 dts 准备

canmv 派生新 dts，供 U-Boot/SPL 与 Linux 使用（详见 `resource/README.md`）：

- U-Boot/SPL：`k230_canmv_qemu.dts`——canmv + 显式使能 spi0 + spi-flash@0 +
  Standard 1-1-1 + `u-boot,dm-pre-reloc`
- Linux：`k230_canmv_qemu.dts`——canmv + bus-width 8→1 + 分区表对齐官方
  `genimage-spinor.cfg`
- U-Boot defconfig：`k230_canmv_qemu_defconfig`
  （`CONFIG_DEFAULT_DEVICE_TREE="k230_canmv_qemu"`）

已验证产物归档在 `resource/dts/dtb/`、`resource/dts/new/`。

## 3. 构建 U-Boot/SPL（官方工具链，已验证）

```bash
cd /home/flamboy/qemu-camp/k230_sdk/src/little/uboot
export CROSS_COMPILE=/opt/toolchain/Xuantie-900-gcc-linux-5.10.4-glibc-x86_64-V2.6.0/bin/riscv64-unknown-linux-gnu-

make k230_evb_defconfig        # SPI NOR 冷启动用 EVB（canmv 走 MMC）
make -j$(nproc)
# 产物：spl/u-boot-spl.bin、u-boot.bin、tools/mkimage
```

## 4. 官方格式打包（已验证）

SDK 官方 `gen_uboot_bin` 逻辑（`board/common/gen_image_script/gen_image_comm_func.sh`），
脚本见 `toolchains/pkg_uboot.sh`：

```bash
K230_SDK=/home/flamboy/qemu-camp/k230_sdk
UBOOT=$K230_SDK/src/little/uboot; cd /tmp/pkg

# U-Boot：k230_gzip -> mkimage uImage -> K230 firmware head -> fn_ug_u-boot.bin
cp $UBOOT/u-boot.bin .
$K230_SDK/tools/k230_priv_gzip -n8 -f -k u-boot.bin
sed -i -e "1s/\x08/\x09/" u-boot.bin.gz
$UBOOT/tools/mkimage -A riscv -C gzip -O u-boot -T firmware \
  -a 0x8000000 -e 0x8000000 -n uboot -d u-boot.bin.gz ug_u-boot.bin
python3 $K230_SDK/tools/firmware_gen.py -i ug_u-boot.bin -o fn_ug_u-boot.bin -n

# SPL：firmware head + endian swap -> swap_fn_u-boot-spl.bin
cp $UBOOT/spl/u-boot-spl.bin .
python3 $K230_SDK/tools/firmware_gen.py -i u-boot-spl.bin -o fn_u-boot-spl.bin -n
$UBOOT/tools/endian-swap.py fn_u-boot-spl.bin swap_fn_u-boot-spl.bin
```

## 5. Flash 镜像装配（已验证布局）

`swap_fn_u-boot-spl.bin` 在 0x0（BootROM 直读），`fn_ug_u-boot.bin` 在
0x80000（SPL 读），Linux 段按可放置的连续空间排布（脚本 `toolchains/mknewflash.py`）：

| 内容 | Flash 偏移 | RAM 地址 | 长度 |
|---|---:|---:|---:|
| SPL（swap 格式） | 0x000000 | —（BootROM 直读） | 0x80000 |
| 完整 U-Boot（fn_ug 格式） | 0x080000 | 0x08000000 | 0x160000 |
| Yocto Image | 0x200000 | 0x08200000 | 0x1a1fe00 |
| rootfs.cpio.gz | 0x1c1fe00 | 0x0a100000 | 0x1eec20 |
| OpenSBI fw_jump.uImage | 0x1e0ec20 | 0x0c100000 | 0x14000 |
| k230-canmv.dtb | 0x1f00000 | 0x0a000000 | 0x1000 |

## 6. 运行 QEMU（全流程，已验证）

```bash
QEMU=./my-qemu-camp-2026-k230/build/qemu-system-riscv64
FLASH=/tmp/k230-newcoldboot.img

"$QEMU" -M k230,spi-flash=w25q256 \
  -drive if=mtd,format=raw,file="$FLASH" \
  -nographic -monitor none -display none -no-reboot
```

到 `K230#` 后输入（自动喂命令脚本 `toolchains/run_final.sh`）：

```text
setenv bootargs console=ttyS0,115200 earlycon=sbi
sf probe 0:0
sf read 0x0c100000 0x1e0ec20 0x14000
sf read 0x08200000 0x200000 0x1a1fe00
sf read 0x0a100000 0x1c1fe00 0x1eec20
sf read 0x0a000000 0x1f00000 0x1000
fdt addr 0x0a000000
fdt resize 8192
fdt set /chosen linux,initrd-start <0x0 0x0a100000>
fdt set /chosen linux,initrd-end <0x0 0x0a2eec20>
bootm 0x0c100000 - 0x0a000000
```

## 7. 常见问题

- `Software reset enable failed: -524`：无害警告，随后出现 `SF: Detected w25q256` 即可继续。
- `Invalid bus 0 (err=-19)`：U-Boot dts 未启用 `spi0`/`spi-flash@0`。
- `spl_board_init_f() failed: 3` / `uboot boot failed`：SPL 启动介质选择错误。
  canmv 固定走 MMC（QEMU 无卡），改用 k230_evb defconfig（SPI NOR）。

## 8. 官方布局升级路径（后续）

官方 `sysimage-spinor32m.img` 在 0xfc0000 放 `linux_system.bin`（7MB 分区），
内部是 `mkimage -T multi -C gzip` 封装 `fw_payload.bin.gz:rd:k230.dtb`
（OpenSBI+Linux+DTB+rootfs）。该布局要求压缩后 ≤ 7MB。内核可直接用 boot-assets 的 SDK 5.10.4 重编版
（第 9 节），但需确认其 gzip 后大小 ≤ 7MB（此前 SDK Image 22MB gzip ≈ 8.3MB 超限，
buildroot 26.7MB 版更大，大概率仍需裁内核/关 DEBUG_INFO 重编）。官方打包函数见
`gen_image_comm_func.sh: gen_linux_bin`。
---

## 9. SDK 5.10.4（官方内核）复用验证（2026-08-05）

`k230-boot-assets/buildroot/uboot-boot/Image` 就是 **SDK T-HEAD 5.10.4 + Xuantie V2.6.0 编译 + 已按 QEMU 标准 PTE 重编**的产物（README 明示 "rebuilt for QEMU PTEs"），可直接复用，**无需重编**。它含 `dw_spi_mmio` + `spi-nor` 驱动（此前 MTD 测试已用）。

```text
Linux version 5.10.4 (root@8eb9db8d89ea) (riscv64-unknown-linux-gnu-gcc
(Xuantie-900 linux-5.10.4 glibc gcc Toolchain V2.6.0 B-20220715) 10.2.0,
GNU ld (GNU Binutils) 2.35) #5 SMP Fri May 8 2026
```

验证布局（buildroot rootfs 26.4MB 超 32MB Flash，故 rootfs 用 yocto 的 2MB busybox initramfs——initramfs 与内核版本无关可混用）：

| 内容 | Flash 偏移 | RAM 地址 | 长度 |
|---|---:|---:|---:|
| SPL（swap 格式） | 0x000000 | — | 0x80000 |
| 完整 U-Boot（fn_ug 格式） | 0x080000 | 0x08000000 | 0x160000 |
| **SDK 5.10.4 Image** | 0x200000 | 0x08200000 | 0x198c800 |
| busybox initramfs | 0x1b8c800 | 0x0a100000 | 0x1eec20 |
| OpenSBI fw_jump.uImage | 0x1e0ec20 | 0x0c100000 | 0x14000 |
| k230-canmv.dtb | 0x1f00000 | 0x0a000000 | 0x1000 |

U-Boot 命令与第 6 节相同，仅 `sf read` 偏移/长度按上表。bootargs 建议加 `cma=0`（SDK 内核否则会为 initramfs 预留过多内存）。脚本：`toolchains/mk510flash.py`、`toolchains/run_510.sh`。日志：`/tmp/k230-sdk510-coldboot.log`。

> 说明：boot-assets 里只有"标准 PTE 重编"的**产物**，README 未记录重编 recipe。要从 SDK 源码复现重编，需自行把 `arch/riscv/include/asm/pgtable-bits.h` 的 `_PAGE_SEC/SHARE/BUF/CACHE/SO`（bit 59-63）从页表属性中移除。