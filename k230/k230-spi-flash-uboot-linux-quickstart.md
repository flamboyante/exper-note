# K230 U-Boot 从 SPI Flash 启动 Linux：最小复现

记录时间：2026-07-22；XIP 验证：2026-07-24

## 中文 PR 实现说明

> 本系列为 K230 machine 增加 Standard SPI 控制器与 SPI NOR Flash 支持。
> machine 通过可选的 `spi-flash` 属性，将 M25P80 兼容 Flash 接到
> `spi0@91584000` 的 CS0，并使用 `-drive if=mtd` 提供 Flash 后端。
> 集成验证使用现成的 OpenSBI、Yocto Linux、initrd 和 DTB 组成 32 MiB
> W25Q256 镜像。U-Boot 可以通过 `sf probe`、`sf read` 将四个镜像读入
> RAM，再使用 `bootm` 启动 Linux 6.18.28，最终进入 initramfs shell。
> 该验证覆盖 U-Boot 从 Standard SPI NOR 加载 Linux 的路径，不覆盖
> BootROM 从 Flash 加载 U-Boot，也不包含 QSPI 启动。`k230-spiv3.1`
> 另外验证了 XIP window 可供 U-Boot 的 `bootm` 读取 OpenSBI。

## 结论

在 QEMU 分支 `k230-spiv2` 上，U-Boot 可以通过标准 SPI NOR 的 `sf probe`
和 `sf read` 从 Flash 读取 OpenSBI、Yocto Linux、initrd 与 DTB，并用
`bootm` 启动到 initramfs shell。

在 QEMU 分支 `k230-spiv3.1` 上，额外验证了 SPI0 的 XIP read window。
U-Boot 仍使用 `sf read` 加载 Yocto Image、initrd 与 DTB；OpenSBI uImage
则直接从 Flash 映射窗口 `0xc0000000` 交给 `bootm`。

本页只验证：

```text
QEMU -bios U-Boot
  -> U-Boot 从 SPI NOR 读镜像
  -> OpenSBI
  -> Linux + initramfs
```

不验证 BootROM 从 Flash 加载 U-Boot，也不讨论 QSPI 或 MMIO 模型实现。
XIP 仅验证 U-Boot 可从映射窗口读取 OpenSBI，不表示 Linux 从 Flash 直接执行。

## 前置条件

以下命令默认从工作区根目录 `/home/flamboy/qemu-camp-2026` 执行。

需要当前分支已经构建出：

```text
qemu-camp-2026-k230/build/qemu-system-riscv64
```

现成镜像来自 `k230-boot-assets`：

```text
common/fw_jump.uImage
yocto/uboot-boot/Image
yocto/uboot-boot/rootfs.cpio.gz
yocto/uboot-boot/k230-canmv.dtb
```

`common/u-boot` 的控制 DTB 默认禁用了 `spi0@91584000`，不能直接用于本页。
请使用一个控制 DTB 已启用 SPI0 的 U-Boot 副本。其最小 DTS 条件如下：

```dts
&spi0 {
    status = "okay";

    spi-flash@0 {
        compatible = "jedec,spi-nor";
        reg = <0>;
        spi-max-frequency = <50000000>;
        spi-tx-bus-width = <1>;
        spi-rx-bus-width = <1>;
        status = "okay";
    };
};
```

这里的 `spi0` 是 `0x91584000`。本验证使用标准单线 SPI，必须保持两个
`spi-*-bus-width` 都为 `1`。

## 准备 Flash 镜像

使用 QEMU 的 `w25q256`，容量固定为 32 MiB。以下布局不重叠：

| 内容 | Flash 偏移 | RAM 地址 | `sf read` 长度 |
|---|---:|---:|---:|
| OpenSBI uImage | `0x00000000` | `0x0c100000` | `0x14000` |
| Yocto Image | `0x00100000` | `0x08200000` | `0x1a1fe00` |
| Yocto initrd | `0x01c00000` | `0x0a100000` | `0x1eec20` |
| Yocto DTB | `0x01f00000` | `0x0a000000` | `0x1000` |

```bash
ASSETS=./k230-boot-assets
FLASH=/tmp/k230-yocto-spi-w25q256.img

truncate -s 32M "$FLASH"
dd if="$ASSETS/common/fw_jump.uImage" of="$FLASH" conv=notrunc status=none
dd if="$ASSETS/yocto/uboot-boot/Image" of="$FLASH" \
   bs=1M seek=1 conv=notrunc status=none
dd if="$ASSETS/yocto/uboot-boot/rootfs.cpio.gz" of="$FLASH" \
   bs=1M seek=28 conv=notrunc status=none
dd if="$ASSETS/yocto/uboot-boot/k230-canmv.dtb" of="$FLASH" \
   bs=1M seek=31 conv=notrunc status=none
```

## 启动 QEMU

将 `UBOOT_SPI` 指向满足前述 DTS 条件的 U-Boot 副本：

```bash
QEMU=./qemu-camp-2026-k230/build/qemu-system-riscv64
UBOOT_SPI=/tmp/u-boot-spi-enabled

"$QEMU" \
    -machine k230,spi-flash=w25q256 \
    -drive if=mtd,format=raw,file="$FLASH" \
    -bios "$UBOOT_SPI" \
    -serial stdio -monitor none -display none -no-reboot
```

等待 `K230#` 提示符后输入：

```text
setenv bootargs console=ttyS0,115200 earlycon=sbi
sf probe 0:0
sf read 0x0c100000 0x0 0x14000
sf read 0x08200000 0x100000 0x1a1fe00
sf read 0x0a100000 0x1c00000 0x1eec20
sf read 0x0a000000 0x1f00000 0x1000
fdt addr 0x0a000000
fdt resize 8192
fdt set /chosen linux,initrd-start <0x0 0x0a100000>
fdt set /chosen linux,initrd-end <0x0 0x0a2eec20>
bootm 0x0c100000 - 0x0a000000
```

## 自动执行

手动命令确认无误后，可以把同一组命令编入 U-Boot 默认 `bootcmd`。Flash
镜像及偏移不变，也不需要额外生成 environment 镜像。

修改 U-Boot 的 `include/configs/k230_evb.h`，将原来的：

```c
#define DEFAULT_BOOTCMD_ENV "bootcmd=k230_boot auto auto_boot; \0"
```

替换为：

```c
#define DEFAULT_BOOTCMD_ENV \
    "bootcmd=" \
    "setenv bootargs console=ttyS0,115200 earlycon=sbi; " \
    "sf probe 0:0; " \
    "sf read 0x0c100000 0x0 0x14000; " \
    "sf read 0x08200000 0x100000 0x1a1fe00; " \
    "sf read 0x0a100000 0x1c00000 0x1eec20; " \
    "sf read 0x0a000000 0x1f00000 0x1000; " \
    "fdt addr 0x0a000000; " \
    "fdt resize 8192; " \
    "fdt set /chosen linux,initrd-start <0x0 0x0a100000>; " \
    "fdt set /chosen linux,initrd-end <0x0 0x0a2eec20>; " \
    "bootm 0x0c100000 - 0x0a000000; \0"
```

然后使用系统中的 RISC-V 交叉工具链重新构建 U-Boot：

```bash
UBOOT_SRC=./build/k230-uboot-src
CROSS_COMPILE=riscv64-linux-gnu-

make -C "$UBOOT_SRC" CROSS_COMPILE="$CROSS_COMPILE" k230_canmv_defconfig
make -C "$UBOOT_SRC" CROSS_COMPILE="$CROSS_COMPILE" -j"$(nproc)"
```

将 QEMU 命令中的 U-Boot 改为新构建的 ELF：

```bash
UBOOT_SPI="$UBOOT_SRC/u-boot"

"$QEMU" \
    -machine k230,spi-flash=w25q256 \
    -drive if=mtd,format=raw,file="$FLASH" \
    -bios "$UBOOT_SPI" \
    -serial stdio -monitor none -display none -no-reboot
```

U-Boot 等待 `bootdelay` 结束后会自动执行 `sf probe`、`sf read` 和 `bootm`，
不再需要在 `K230#` 手动输入命令。当前 QEMU 没有 MMC，U-Boot 读取 MMC
environment 失败后会使用这里编译进去的默认 `bootcmd`。

## XIP 读取 OpenSBI（spi3.1）

本节使用 QEMU 分支 `k230-spiv3.1`（验证提交 `99c3ebb428`），复用前文的
W25Q256 Flash 镜像和启用 SPI0 的 U-Boot。Flash 布局不变。

与前面的标准 SPI 路径相比，只有 OpenSBI 的加载方式变化：不执行
`sf read 0x0c100000 0x0 0x14000`，而是通过 SPI0 的 XIP window
`0xc0000000` 直接让 `bootm` 读取 Flash 中偏移 `0x0` 的 OpenSBI uImage。
Yocto Image、initrd 和 DTB 仍必须先读入 RAM。

在 `K230#` 输入：

```text
setenv bootargs console=ttyS0,115200 earlycon=sbi
sf probe 0:0
sf read 0x08200000 0x100000 0x1a1fe00
sf read 0x0a100000 0x1c00000 0x1eec20
sf read 0x0a000000 0x1f00000 0x1000
fdt addr 0x0a000000
fdt resize 8192
fdt set /chosen linux,initrd-start <0x0 0x0a100000>
fdt set /chosen linux,initrd-end <0x0 0x0a2eec20>
mw.l 0x91584008 0
mw.l 0x91584000 0x4007
mw.l 0x91584100 0x13
mw.l 0x915840f4 0x100220
mw.l 0x91585068 0x4001
md.l 0xc0000000 1
bootm 0xc0000000 - 0x0a000000
```

`sf probe` 会让 W25Q256 进入 4-byte address mode。因此，XIP 寄存器必须
在所有 `sf read` 完成后重新配置；其中 `0x13` 是 4-byte Read opcode，
`0x100220` 表示 8-bit instruction 加 32-bit address。若仍使用 3-byte read
配置，XIP 读取会失败或错位。

关键地址如下：

| 名称 | 地址 |
|---|---:|
| SPI0 MMIO | `0x91584000` |
| HI_SYS SSI_CTRL | `0x91585068` |
| XIP Flash window | `0xc0000000` |

以下是本次实际运行的关键输出：

```text
c0000000: 56190527
## Booting kernel from Legacy Image at c0000000 ...
Verifying Checksum ... OK
Starting kernel ...

OpenSBI v0.9
[    0.000000] Linux version 6.18.28
meta-k230 initramfs starting...
~ #
```

`c0000000: 56190527` 与随后成功的 checksum 表明，U-Boot 已通过 XIP window
读取到 Flash 中的 OpenSBI uImage。Linux 内核和 initramfs 仍由前面的
`sf read` 放入 RAM 后启动；本节不是 Linux XIP 执行测试。

## 关键日志

以下来自本次实际运行：

```text
SF: Detected w25q256 with page size 256 Bytes, erase size 64 KiB, total 32 MiB
SF: 81920 bytes @ 0x0 Read: OK
SF: 27393536 bytes @ 0x100000 Read: OK
SF: 2026528 bytes @ 0x1c00000 Read: OK
SF: 4096 bytes @ 0x1f00000 Read: OK

## Booting kernel from Legacy Image at 0c100000 ...
Starting kernel ...

OpenSBI v0.9
[    0.000000] Linux version 6.18.28
meta-k230 initramfs starting...
Dropping to shell...
~ #
```

`sf probe` 前可能出现 `Software reset enable failed: -524`。只要随后出现
`SF: Detected w25q256`，该警告不影响本次标准 SPI 读和 Linux 启动。

## 首先检查

- `Invalid bus 0 (err=-19)`：U-Boot 控制 DTB 仍未启用 `spi0@91584000`，或缺少
  `spi-flash@0`。
- `No SPI flash selected`：前一条 `sf probe 0:0` 未成功，不能继续执行 `sf read`。
