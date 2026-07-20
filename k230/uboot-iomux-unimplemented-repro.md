# K230 QEMU 启动与 U-Boot SPI Flash 验证

## 1. 结论

截至 2026-07-18，Patch 7 基线
`e3f5c7ddb20903f36e943ef7249f5697718136c0` 已完成下面两类验证：

| 启动路径 | 结果 | 载荷在哪里 |
|---|---|---|
| QEMU 主线 direct boot | 通过 | QEMU 直接放入 RAM |
| QEMU 主线 U-Boot boot | 通过 | U-Boot、OpenSBI、Linux、DTB、initramfs 都由 QEMU 放入 RAM |
| U-Boot 从 SPI Flash 读取 Linux 载荷 | 通过 | 只有完整 U-Boot 由 `-bios` 放入 RAM；OpenSBI、Linux、DTB、initramfs 均从 Flash 读取 |
| BootROM -> SPL -> U-Boot 的完整冷启动 | 未实现 | 当前 machine 没有从 Flash 装载 SPL 的 BootROM 行为 |

第三行是本次最重要的新增结论。它证明 Patch 4--7 的 Flash、PIO 和 IRQ
链路足以让 U-Boot 把完整 Linux 启动载荷从 W25Q256 读入 RAM，并实际进入
Linux initramfs shell。它不等于真实芯片的上电 Flash 启动。

## 2. 先区分三种启动方式

```text
A. QEMU direct boot（主线默认测试思路）
   host -> RAM: OpenSBI + Linux + DTB + initramfs
   OpenSBI -> Linux

B. QEMU U-Boot boot（主线文档的 U-Boot 示例）
   host -> RAM: U-Boot + OpenSBI + Linux + DTB + initramfs
   U-Boot bootm -> OpenSBI -> Linux

C. 本文已验证的 Flash payload boot
   host -> RAM: 仅 U-Boot（-bios）
   SPI Flash -> RAM: OpenSBI + Linux + DTB + initramfs（U-Boot sf read）
   U-Boot bootm -> OpenSBI -> Linux

D. 尚未实现的真实冷启动
   BootROM -> SPI Flash 0x0 的 SPL -> DDR 初始化
           -> Flash 中的完整 U-Boot -> Flash 中的系统镜像 -> Linux
```

QEMU K230 主线文档只承诺 A、B。其 U-Boot 一节明确说明：在 SDK storage
path 被建模前，QEMU 用 loader 将 OpenSBI、Linux、DTB 和 initramfs 放到
RAM。主线流程中没有 SPL 并不是说真实 K230 没有 SPL，而是该模拟路径有意
绕过了 BootROM、SPL 和存储加载。

本文的 C 去除了这些 payload loader，但仍保留 `-bios U-Boot`；它正好验证
U-Boot 之后的 Flash 读取链路。

## 3. 资产、基线与边界

### 3.1 本机复验基线

所有本节结果都在独立 detached worktree 中取得：

```text
worktree: /tmp/k230-p7-flash-final
commit:   e3f5c7ddb20903f36e943ef7249f5697718136c0
```

该 worktree 不修改主 QEMU 工作树，也不使用含 Patch 8 脚手架的工作目录。

预构建资产位于：

```text
/tmp/k230-boot-assets-trto/
  common/u-boot                 SDK U-Boot 2022.10 ELF
  common/fw_jump.uImage         SDK OpenSBI 0.9 的 bootm uImage
  yocto/uboot-boot/Image        Yocto Linux 6.18.28
  yocto/uboot-boot/rootfs.cpio.gz
  yocto/uboot-boot/k230-canmv.dtb
```

资产仓库的 README 记录了版本来源；其中 U-Boot 与 `fw_jump.uImage` 来自
K230 SDK，Yocto kernel/rootfs 则是为 QEMU 准备的可启动产物。

本地 `/home/flamboy/qemu-camp/k230_sdk` 当前是 SDK 源码树，未包含
`output/...` 的已构建 U-Boot、SPL 或完整 SPI-NOR 镜像。因此不能把本机源码
树误称为已有可直接启动的官方 SDK 产物。

### 3.2 重要的 U-Boot DT 前置条件

原始预构建 SDK U-Boot 的内嵌 DT 将：

```text
/soc/spi@91584000
    status = "disabled"
```

保留为 disabled。即使 QEMU 已经通过
`-machine k230,spi-flash=w25q256` 接好了 W25Q256，原始 U-Boot 仍没有
SPI bus 设备。实际复验：

```text
K230# sf probe 0:0
Invalid bus 0 (err=-19)
```

`dm tree` 也没有 `spi@91584000`。这不是 QEMU SSI 或 Flash 模型失败，而是
U-Boot 自身 Device Tree 配置没有启用该控制器。

成功复验使用的 `/tmp/k230-p4-uboot` 是同一 SDK U-Boot 的临时派生 ELF：

```text
/soc/spi@91584000: status = "okay"
/soc/spi@91584000/flash@0: 已添加且 compatible 为 SPI NOR
```

它的用途仅是验证。产品化时应在 SDK U-Boot 的板级 DTS/配置中完成同样的最小
变更并重新生成正式 U-Boot，不应依赖 `/tmp` 中的临时 ELF。向 QEMU 传入 Linux
DTB 并不能改变 U-Boot 自己的内嵌 DT。

## 4. 主线 RAM 启动：用于确认基础环境

这条命令对应 QEMU 主线的 direct boot 思路，适合作为 Linux/QEMU 基础回归：

```bash
QEMU="/tmp/k230-p7-flash-final/build/qemu-system-riscv64"
ASSETS="/tmp/k230-boot-assets-trto"
OPENSBI="/tmp/k230-p7-flash-final/pc-bios/opensbi-riscv64-generic-fw_dynamic.bin"
DST="$ASSETS/yocto/direct-boot"

"$QEMU" \
  -machine k230 \
  -bios "$OPENSBI" \
  -kernel "$DST/Image" \
  -dtb "$DST/k230-canmv.dtb" \
  -initrd "$DST/rootfs.cpio.gz" \
  -append "console=ttyS0,115200 earlycon=sbi" \
  -nographic \
  -no-reboot
```

成功标志：

```text
Linux version 6.18.28
meta-k230 initramfs starting...
Dropping to shell...
~ #
```

这里所有启动载荷都由 host 放入 RAM，因此不能用于证明 SPI Flash 路径。

## 5. 已验证：`-bios` U-Boot，从 SPI Flash 读取 Linux 到 shell

### 5.1 本流程证明什么

```text
QEMU ROM 跳板
  -> -bios 放入 RAM 的完整 U-Boot
  -> U-Boot sf probe / sf read
  -> SPI Flash raw backend
  -> RAM 中的 OpenSBI、Linux、DTB、initramfs
  -> bootm
  -> OpenSBI 0.9
  -> Linux 6.18.28
  -> initramfs shell
```

唯一由 host 预装到 RAM 的启动组件是 U-Boot。命令中没有
`-device loader`、`-kernel`、`-initrd` 或 `-dtb`；这些选项均不用于 Flash
payload boot。

Linux `Image` 是未压缩内核，U-Boot 只负责读取。`rootfs.cpio.gz` 也由
U-Boot 原样读取并经 `/chosen/linux,initrd-*` 传给 Linux；gzip 解压由 Linux
内核完成。该流程没有使用 GSDMA 或硬件 gzip/decompression 设备。

### 5.2 Flash 布局

测试 Flash 是 W25Q256（32 MiB），使用 raw backend。下表的内容均来自上节的
预构建资产：

| 内容 | Flash offset | 大小 |
|---|---:|---:|
| OpenSBI `fw_jump.uImage` | `0x00000000` | `0x13fc8` |
| Yocto `Image` | `0x00100000` | `0x1a1fe00` |
| Yocto `rootfs.cpio.gz` | `0x01c00000` | `0x1eec20` |
| Yocto `k230-canmv.dtb` | `0x01e00000` | `0x642` |

所有范围都位于 `0x00000000..0x01ffffff`，且互不重叠。创建镜像：

```bash
QEMU="/tmp/k230-p7-flash-final/build/qemu-system-riscv64"
ASSETS="/tmp/k230-boot-assets-trto"
COMMON="$ASSETS/common"
DST="$ASSETS/yocto/uboot-boot"
FLASH="/tmp/k230-p7-sdk-assets-flash.bin"

truncate -s 32M "$FLASH"
dd if="$COMMON/fw_jump.uImage" of="$FLASH" bs=1 seek=$((0x00000000)) \
  conv=notrunc status=none
dd if="$DST/Image" of="$FLASH" bs=1 seek=$((0x00100000)) \
  conv=notrunc status=none
dd if="$DST/rootfs.cpio.gz" of="$FLASH" bs=1 seek=$((0x01c00000)) \
  conv=notrunc status=none
dd if="$DST/k230-canmv.dtb" of="$FLASH" bs=1 seek=$((0x01e00000)) \
  conv=notrunc status=none
```

### 5.3 为什么当前要执行四次 `sf read`

这不是 SPI 控制器的限制，而是本测试选择了四个独立文件、四个 Flash offset 和
四个 RAM 目标地址：

```text
Flash                              RAM
fw_jump.uImage  -- sf read -->    0xc100000
Image             -- sf read -->  0x8200000
rootfs.cpio.gz    -- sf read -->  0xa100000
k230-canmv.dtb    -- sf read -->  0xa000000
```

`bootm` 需要找到 OpenSBI uImage；OpenSBI 之后在约定地址获得 Linux `Image`；
DTB 需要在可修改的独立地址中写入 initramfs 的起止范围。因此分别读取是最直接的
验证方式。虽然可以一次读取 `0x0..0x1e00642` 的大范围到 staging buffer，再复制
四个片段到各自地址，但会连同对齐空洞一起读取，且额外引入复制步骤，没有测试价值。

SDK 量产格式采用相反的组织方式：`linux_system.bin` 是一个组合包，内部包含
OpenSBI/Linux、rootfs 和 DTB。它可以作为一个整体读取，但随后必须完成 K230
firmware header 校验、gzip 解压和内部组件解析。当前四次 `sf read` 恰好绕过这些
非 SSI 的步骤，单独证明每个 Linux 启动载荷都能从同一颗 SPI Flash 读入 RAM。

### 5.4 启动和执行脚本

这里必须使用已启用 SPI0/`flash@0` 的 SDK U-Boot 变体；直接将
`$COMMON/u-boot` 传给 `-bios` 会遇到 3.2 的 `Invalid bus` 问题。

```bash
UBOOT_FLASH="/tmp/k230-p4-uboot"

"$QEMU" \
  -machine k230,spi-flash=w25q256 \
  -drive file="$FLASH",format=raw,if=mtd \
  -bios "$UBOOT_FLASH" \
  -nographic \
  -no-reboot
```

等默认 MMC 尝试结束并回到 `K230#`，执行：

```text
setenv bootcmd_flash 'sf probe 0:0; sf read 0xc100000 0 0x13fc8; sf read 0x8200000 0x100000 0x1a1fe00; sf read 0xa100000 0x1c00000 0x1eec20; sf read 0xa000000 0x1e00000 0x642; setenv bootargs console=ttyS0,115200 earlycon=sbi; fdt addr 0xa000000; fdt resize 8192; fdt set /chosen linux,initrd-start <0x0 0xa100000>; fdt set /chosen linux,initrd-end <0x0 0xa2eec20>; bootm 0xc100000 - 0xa000000'
run bootcmd_flash
```

实测输出包含：

```text
SF: Detected w25q256 ... total 32 MiB
SF: 81864 bytes @ 0x0 Read: OK
SF: 27393536 bytes @ 0x100000 Read: OK
SF: 2026528 bytes @ 0x1c00000 Read: OK
SF: 1602 bytes @ 0x1e00000 Read: OK
Verifying Checksum ... OK
Starting kernel ...
OpenSBI v0.9
Linux version 6.18.28
Dropping to shell...
~ #
```

这已经证明“完整 U-Boot 在 RAM，Linux 启动载荷在 SPI Flash，最终到 shell”。

`sf probe` 的第一条信息：

```text
jedec_spi_nor flash@0: Software reset enable failed: -524
```

是驱动先尝试不被该器件支持的 Octal-DTR software reset、再回退 Standard SPI
的非致命信息。后续 JEDEC 识别、全部读取和 Linux 启动均成功，因此当前不作为
阻塞问题。

### 5.5 要把这条路径变成默认自动启动，还缺什么

本次已验证的是手工 `run bootcmd_flash`，不是持久化 bootcmd。需要完成的工作
只有 U-Boot 镜像/环境层面的两项，不需要修改当前 Patch 7 的 QEMU SSI 代码：

1. 在 SDK U-Boot 的正式 DTS 中启用 `spi@91584000`，并声明 CS0 `flash@0`。
2. 将经验证的 `bootcmd_flash`、Flash offset、`bootargs` 和 initrd 范围写入
   默认环境或 SPI-NOR environment；同时避免与该环境占用的 Flash 区域冲突。

当前 U-Boot 的默认环境仍优先尝试 MMC，因此会出现：

```text
MMC: no card present
```

它不影响手工 Flash boot，但说明“无交互自动 Flash boot”尚未产品化。上述两项
完成后，应再次复测“上电后不输入命令，自动到 `~ #`”。

## 6. 仍未覆盖：真实 SPI Flash 冷启动

真实 SDK SPI-NOR 镜像的布局与第 5 节故意不同：

```text
Flash 0x000000: swap_fn_u-boot-spl.bin
Flash 0x080000: fn_ug_u-boot.bin
Flash 0x1e0000: U-Boot environment
Flash 0x0fc0000: Linux system image
```

SDK 的 `genimage-spinor.cfg` 说明前 0x80000 是 SPL、后面才是完整 U-Boot。
完整冷启动应为：

```text
CPU reset
  -> BootROM（选择 SPI NOR）
  -> 从 Flash 0x0 装载 SPL
  -> SPL 初始化 DDR
  -> SPL 从 Flash 0x80000 装载完整 U-Boot
  -> U-Boot 从 Flash 自动加载系统
  -> OpenSBI -> Linux
```

当前 QEMU 不具备第一步：`0x91200000` 的 ROM 只包含 QEMU 生成的跳转 stub，
用于跳到 `-bios`；它不会读取 Flash、不会识别 SPL、不会选择 boot source。

因此第 5 节成功后，完整冷启动还需要：

1. 明确真实 BootROM 的首级读取机制（PIO copy 或 XIP）。SDK 打包文件只证明
   SPL 位于 Flash 0，不证明 ROM 采用哪种协议。
2. 实现或提供 BootROM 行为，使其从 Flash 0 装载/跳转 SPL；若目标是功能验证，
   可以先使用明确标注为“功能型”的 ROM stub，不能声称等同真实 ROM。
3. 使用 SDK 格式的 SPL、完整 U-Boot、environment 和系统镜像，而非第 5 节的
   raw payload layout。
4. 让 SPL 实际运行并逐项验证其早期依赖：DDRC、时钟/复位、IOMUX、boot mode
   寄存器与 SPI NOR。外部同事实现的 DDRC 模型对此非常有帮助，但尚未纳入本机
   Patch 7 基线，需在独立 worktree 集成后以真实 SPL 验证。
5. 建模硬件 gzip 与 GSDMA，或明确改用 `IH_COMP_NONE` 的非 SDK 打包格式。
   SDK 默认的 `gen_linux_bin()` 和 `gen_uboot_bin()` 都通过 `k230_priv_gzip`
   生成 gzip 的 U-Boot legacy image；SPL/U-Boot 的 `k230_img.c` 对
   `IH_COMP_GZIP` 调用 `gunzip()`，而 K230 的 `unzip.c` 通过
   `GSDMA @ 0x80800000` 和 `decomp-gzip @ 0x80808000` 完成解压。当前 QEMU
   这两个 MMIO 仍是 unimplemented，因此使用原样 SDK SPI-NOR 镜像时，这会是
   SPL 装载完整 U-Boot、以及 U-Boot 装载 `linux_system.bin` 的明确阻碍。

Patch 8 的 HI_SYS `SSI_CTRL` 和 Patch 9 的 XIP window 不属于第 5 节的硬前置。
只有确认 BootROM/SPL 通过 `0xc0000000` 的 XIP window 首级读取或取指时，才需要
按 P8 -> P9 完成它们。

## 7. IOMUX 小节

IOMUX 不是本文件的主线，但可用 SDK U-Boot 进行快速回归：

```bash
QEMU="/tmp/k230-p7-flash-final/build/qemu-system-riscv64"
UBOOT="/tmp/k230-boot-assets-trto/common/u-boot"

timeout 10 "$QEMU" \
  -machine k230 \
  -bios "$UBOOT" \
  -nographic \
  -d unimp \
  -D /tmp/k230-iomux-unimp.log \
  > /tmp/k230-iomux-console.log 2>&1

rg -n '^iomux:' /tmp/k230-iomux-unimp.log
```

预期 U-Boot 到达 `K230#`，且最后一条 `rg` 没有输出。`timeout` 返回 124 表示
主机在 U-Boot 停留时终止 QEMU，不表示失败。

## 8. Patch 9 最终 SPI/QSPI 验证与上游评审（2026-07-20）

本节是对 Patch 1--9 完整系列的独立复验；第 1--7 节保留其当时的
`/tmp/k230-p7-flash-final` 历史结果，不修改或混用。这里的基线、资产和结论以本节
为准。

### 8.1 固定输入、构建例外和测试边界

```text
QEMU worktree: /tmp/k230-spi-final-89
QEMU baseline: 89f718b5a339e5fc7ec8b30bcb3d63c8047257f9
assets:        https://github.com/zevorn/k230-boot-assets.git
assets commit: c3c32fb46e8307c5063f13e8f367c98bf9273cd1
assets path:   /home/flamboy/qemu-camp/k230-boot-assets
```

这纠正了早先章节中 `/tmp/k230-boot-assets-trto` 的使用：它仅是旧 Patch 7
复验的历史输入，不能再作为本节或后续复验的资产来源。当前资产仓库在固定提交上
且工作树干净。

默认构建启用 Werror 时在 GCC 13 失败：

```text
hw/ssi/k230_dw_ssi.c:1103: -Wimplicit-fallthrough
```

因此为取得运行时证据，临时 build 使用 `-Dwerror=false`；未修改 QEMU 源码。
这不是可上游提交的构建结果，见 8.5。该 build 的 K230 SSI qtest 全量通过
`54/54`，包括 Standard/Quad Flash、PIO、IRQ、HI_SYS 和 XIP 测试。

Linux 测试使用 Buildroot Linux 5.10.4（提供 `dw_spi_mmio` 与 SPI-NOR 驱动）和
基于 Yocto 小 initramfs 的测试载荷，以适配 32 MiB W25Q256。它们是为测试拼装的
混合载荷，不是 SDK 发布镜像；不得据此声称 SDK SPI-NOR 整机启动已通过。

所有下列启动仍是 Flash payload boot：完整 U-Boot 由 `-bios` 置入 RAM，OpenSBI、
kernel、initramfs 与 DTB 由 U-Boot `sf read` 从 Flash 读取。它不是
BootROM -> SPL 的冷启动，前文第 6 节的限制仍然适用。

### 8.2 端到端矩阵

| 客户端与线宽 | 设备枚举 | 实际擦写/读回 | 结果 |
|---|---|---|---|
| U-Boot，Standard SPI | `SF: Detected w25q256` | `sf erase`、256 B `sf write`/`sf read`、`cmp.b` 都成功 | 通过 |
| U-Boot，Quad SPI | 已识别 W25Q256；HI_SYS `SSI_CTRL=0x00004040` | write/read 均超时，`risr=0x1 0x6`，返回 `-110` | 失败 |
| Linux 5.10.4，Standard SPI | `dw_spi_mmio` 与 `spi-nor spi0.0: w25q256` probe，建立 `qemu-test` MTD 分区 | `/dev/mtd0` 写入 EIO，增强 memory-op 重试失败，回读不符 | 失败 |
| Linux 5.10.4，Quad SPI | 同上；Quad DT 将 tx/rx bus width 设为 4 | `/dev/mtd0` 读写失败 | 失败 |

U-Boot Standard 的成功日志在
`/tmp/k230-spi-final-89-uboot-standard.log`：`SSI_CTRL=0x00004000`，随后可从 Flash
加载 OpenSBI、Linux 6.18.28、rootfs 和 DTB，并到达 `Dropping to shell`。这同时覆盖
了 U-Boot 标准 SPI 的实际读、擦、写、读回和启动载荷读取。

U-Boot Quad 的失败日志在 `/tmp/k230-spi-final-89-uboot-quad.log`。`0x00004040`
证明 DT 的 Quad 设置已真正写入 HI_SYS，而非 SPI0/Flash 节点未启用；失败位于
QEMU 增强 QSPI 事务与 U-Boot `dw_spi` 驱动的交互，不能视为 DTS 配置问题。

Linux Standard 的完整复跑日志为
`/tmp/k230-spi-final-89-linux-standard-rerun.log`。关键错误是：

```text
spi_master spi0: CS de-assertion on Tx
spi_master spi0: Retry of enh_mem_op failed
dd: error writing '/dev/mtd0': Input/output error
K230_MTD_TEST_FAIL mode=standard device=/dev/mtd0
```

Linux Quad 已使用标准 U-Boot 仅负责可靠地把 Quad Linux DT 载入 RAM，之后由 Linux
的 Quad DT 驱动 SPI；这避免把已知的 U-Boot Quad `-110` 掩盖为 Linux 启动失败。
复跑日志 `/tmp/k230-spi-final-89-linux-quad-rerun.log` 显示 DTB 已正确读到 RAM：

```text
0a000000: edfe0dd0
spi spi0.0: setup: ignoring unsupported mode bits a00
spi-nor spi0.0: w25q256 (32768 Kbytes)
spi_master spi0: Retry of enh_mem_op failed
K230_MTD_TEST_FAIL mode=quad device=/dev/mtd0
```

因此先前 Quad 日志中的 `FDT_ERR_BADMAGIC` 是一次测试镜像/加载过程问题，而不是
DTB 布局错误：Flash `0x01e00000` 中的 DTB 已与源文件逐字节校验一致，且测试分区
`0x01f00000..0x01f10000` 不重叠。重新生成 Flash 并复跑后 Linux 已启动，结论是
实际 Quad MTD I/O 失败。

### 8.3 本轮测试命令的关键约束

所有场景均使用：

```bash
QEMU="/tmp/k230-spi-final-89/build/qemu-system-riscv64"
ASSETS="/home/flamboy/qemu-camp/k230-boot-assets"

"$QEMU" \
  -machine k230,spi-flash=w25q256 \
  -drive file="$FLASH",format=raw,if=mtd \
  -bios "$UBOOT" \
  -nographic -no-reboot
```

Flash 保持前文的 raw payload 分区方式，另留 `0x01f00000..0x01f10000` 作为 Linux
`qemu-test` MTD 分区。U-Boot 读写测试先在该分区执行：

```text
sf erase 0x1f00000 0x10000
mw.b 0x8300000 0x5a 0x100
sf write 0x8300000 0x1f00000 0x100
sf read 0x8310000 0x1f00000 0x100
cmp.b 0x8300000 0x8310000 0x100
```

Linux initramfs 对同一分区的 `/dev/mtd*` 执行写入、读取和 `cmp`。所以“probe 成功”
与“读写通过”在本节被严格区分，后者当前没有任何模式通过。

### 8.4 功能结论与应实现的范围

本系列的 QSPI 模型 qtest 覆盖并不等价于真实软件栈兼容性。当前可接受的功能结论
仅为 U-Boot Standard SPI 与 Flash payload boot 通过；不能宣称“终极 SPI/QSPI
考验通过”，也不能宣称 U-Boot Quad、Linux Standard 或 Linux Quad 的 SPI-NOR
读写已通过。

后续修复应首先用上述四格矩阵回归，并明确限制模型能力：只覆盖 SDR 1/2/4-bit
事务；OPI 与 DDR/Octal-DTR 不在本系列实现范围。XIP window 也是只读，不应暗示它
能用于写 Flash。`Software reset enable failed: -524` 是 U-Boot Standard 路径中
对不支持 Octal-DTR reset 的可回退探测，因随后的识别、读写和启动均成功，单独不
作为阻塞项。

### 8.5 以 QEMU 主线 review 标准的结论：不应提交

结论为 **Request changes / 不建议发送上游**，至少应解决以下阻塞项后再重新评审：

1. 默认 Werror 构建必须通过；为 switch fallthrough 添加符合 QEMU 规范的显式标注，
   不应依赖 `-Dwerror=false`。
2. 在本基线对 `30e8a06b64..89f718b5a3` 运行
   `git format-patch --stdout --no-stat ... | scripts/checkpatch.pl --strict -`，结果是
   **42 errors、118 warnings、6616 lines checked**。问题包括超过 90 字符的行、
   `if(s->)` 格式、中文/`LEARNING(...)` 教学脚手架和冗长注释。它们需在拆分为
   可评审的补丁前清理为主线风格的英文、简洁说明。
3. `checkpatch` 同时提示新 SSI/HI_SYS/qtest 文件缺少 MAINTAINERS 覆盖；当前
   `K230 Machines` 条目只覆盖 `hw/riscv/k230.c`、文档和 watchdog。应增加本系列
   新增 SSI、HI_SYS、头文件和 qtest 文件的维护范围，并复查提交者/邮件列表。
4. 最重要的是端到端功能回归：修复 U-Boot Quad `-110`，以及 Linux Standard/Quad
   的增强 memory-op 失败，直到 8.2 四格矩阵全部通过。仅 qtest `54/54` 不能替代
   上游对真实 guest 驱动兼容性的要求。

在这些问题解决前，Patch 8/9 的 HI_SYS 与 XIP 功能应保留为开发分支实验，而不是
最终上游补丁。修复后应以干净 worktree、默认 Werror 构建、qtest 和本节四类
U-Boot/Linux 实测日志重新记录结果。

### 8.6 已定位的代码根因（2026-07-20）

这不是 Flash 后端、DTB 或 Quad 线宽设置未生效。当前失败来自两个尚未实现/不兼容的
控制器行为。

1. **Quad 的 IDMA/DONE 路径未实现。** SDK U-Boot 在 data bus width 大于 1 时将
   `use_idma=1`，把命令/地址写到 `SPIDR`/`SPIAR`、buffer 地址写到 `AXIAR0`，再使能
   `DMACR.IDMAE`，并轮询 `RISR.DONE`。Linux K230 `dw_spi` 的增强 memory-op 同样
   配置 `SPIDR`/`SPIAR`/`AXIAR*`/`DMACR`，等待 completion/DONE。

   QEMU 当前对 `DMACR.IDMAE` 的行为是明确记录
   `internal DMA is not implemented`；`SPIDR`、`SPIAR` 与 `AXIAR*` 仅保存寄存器值，
   不会发起 SSI 事务或搬运 guest memory；`DONECR` 读为零，代码中没有任何路径置
   `RISR.DONER`。因此 U-Boot 一直等 DONE 并以 `-110` 超时，Linux 的增强 memory-op
   等待 completion 后重试并以 EIO 失败。这是 U-Boot Quad 与 Linux Quad 的直接根因。

2. **Standard PIO 的 FIFO/CS 时序模型过度同步。** Linux Standard 的 256 B page
   program 走普通 memory-op PIO。其驱动先填满 256-entry TX FIFO，再置 `SER`；随后它
   要求在继续填余下的命令/数据前 `TXFLR` 仍非零，否则认定硬件已经因 FIFO 空而撤销
   native CS，打印 `CS de-assertion on Tx`。

   QEMU 的 `k230_dw_ssi_run_transfer()` 在每次 DR 写或 SER 写的同一 MMIO 回调内
   `while` 循环清空整个 TX FIFO。故驱动读取 `TXFLR` 时立即见到零，正好进入该 EIO
   分支。U-Boot Standard 使用不同的轮询/填充策略，所以没有触发这一时序缺口。

Patch 5 的 qtest 仅覆盖“把 opcode/address/data 作为 DR PIO 写入”的简化 Quad
transaction，未覆盖 IDMA 寄存器、DONE IRQ、真实 Linux FIFO refill，也未用 SDK
U-Boot 驱动回归；所以 qtest `54/54` 与本节失败并不矛盾。

修复顺序应为：先实现或显式拒绝 IDMA（若实现，需支持 guest memory 搬运、`SPIDR`/
`SPIAR` 描述、DONE latch/clear 和 IRQ）；再将 PIO 发送改为可推进的时序模型，保证
native CS 在 FIFO refill 期间保持 asserted，并新增对应 Linux/SDK U-Boot MMIO 序列的
qtest。仅伪造 DONE 不足以正确实现 Quad 读写。

### 8.7 后续复验更正：载荷尺寸与 IDMA/PIO 依赖

后续独立 worktree 复验曾错误地把 Yocto Linux 6.18.28 的载荷尺寸用于本节的
Buildroot Linux 5.10.4 Flash 镜像：

```text
错误：kernel=0x1a1fe00 initrd=0x1eec20 DTB=0x642
正确：kernel=0x198b000 initrd=0x1eb50a DTB=0x987
```

错误尺寸会截断 DTB、设置错误的 `linux,initrd-end`，表现为 U-Boot 已打印
`Starting kernel ...`，但没有 OpenSBI 或 Linux 输出。该现象不是 SSI 数据损坏：
U-Boot 对读入 RAM 的 OpenSBI、kernel、initrd 和 DTB 计算的 CRC 与 Flash 原始内容
一致。使用正确尺寸后，`spi-patch`、IDMA 分支和 PIO timing 分支都能正常进入
Linux 5.10.4 并 probe `dw_spi_mmio`/SPI-NOR。

修正尺寸后的 Standard PIO 结果为：

```text
spi-patch: Linux 启动，复现 CS de-assertion on Tx 和 MTD FAIL
889ece7647: Linux 启动，256 B 与 4 KiB MTD write/read/cmp 均通过
```

IDMA 分支 `a30307581d` 使用本节 Quad DT 复验时，U-Boot Quad 的 256 B/4 KiB
write/read/cmp 通过，但 Linux page program 仍在进入 IDMA 数据阶段前失败。原因是
SPI-NOR 先发送无数据的 WREN；K230 Linux 驱动对该操作回退 Standard PIO，而独立
IDMA commit 按任务边界没有包含 Standard PIO pacing 修复。因此日志中的
`CS de-assertion on Tx` 来自 WREN 的 Standard PIO 前置操作，不能据此判定 IDMA
guest-memory 搬运本身失败。完整 Linux Quad page program 验证需要同时组合 IDMA 与
PIO timing 两个独立修复。

组合复验还发现 IDMA commit 的 DONE 清除语义与 SDK Linux 驱动不一致。驱动的 DONE
IRQ handler 通过读取 `DONECR` 清除中断，然后清零 `DMACR`；原实现却让 `DONECR`
读取恒为零，仅在写 `DONECR` 时清除 `RISR.DONER`。这会让 U-Boot 遗留的 DONE latch
在 Linux 打开 DONE IRQ 后成为陈旧中断，提前完成尚未启动的新事务。修正为
`DONECR` read-clear，并与 Standard PIO pacing 组合后，Linux 5.10.4 Quad 的 256 B
和 4 KiB MTD write/read/cmp 均通过。验证日志为：

```text
/tmp/k230-idma-pio-linux510-donecr-readclear-256-4k.log
```

该日志同时包含 U-Boot Quad 256 B/4 KiB write/read/cmp、Linux 5.10.4 probe、
`K230_MTD_TEST_PASS_256` 和 `K230_MTD_TEST_PASS_4K`，且不包含
`DMACR.IDMAE ... not implemented`、`ERROR -110`、`Retry of enh_mem_op failed`、
`CS de-assertion on Tx` 或 `K230_MTD_TEST_FAIL`。这证明 IDMA 数据路径可用，但也明确
说明完整 Linux Quad 流程依赖两个正交修复：Standard PIO pacing 负责 WREN 等无数据
命令，IDMA 实现及正确的 DONE read-clear 语义负责 Quad 数据搬运与完成中断。

### 8.8 主线基线更新暴露的 `WAIT_CYCLES` 语义错误

将 K230 SPI 系列从 `30e8a06` 重放到 QEMU 基线 `f893c46` 后，首个失败是
`/riscv64/k230-dw-ssi/qspi/dual-quad-output-read` 的 `actual != expected`。rebase
无冲突，构建也成功；失败来自 K230 SSI 与上游 `m25p80` 对 dummy phase 的旧契约不再
一致。

K230 qtest 对 `0x3b`/`0x6b` 使用 `SPI_CTRLR0.WAIT_CYCLES=8`。该字段的单位是
dummy clock cycles，但旧模型把它直接作为 `ssi_transfer()` 的次数，即错误地发送
8 个 SSI 字节。QEMU 主线随后修复了 Winbond、Numonyx/Micron、Macronix 和 Spansion
flash 的 dummy-byte 换算；对 W25Q256 的 `0x3b`/`0x6b`，8 个 dummy clocks 只应发送
1 个 SSI dummy byte。多余 7 个零字节会在 flash 已进入数据阶段后消耗真实数据，因而
控制器读到偏移后的内容。

正确换算必须依据 dummy phase 宽度：`TRANS_TYPE=0`（1-1-2/4 output read）使用
单线 dummy phase；`TRANS_TYPE=1/2` 按 Dual/Quad 的 2/4 线宽计算。发送次数为
`ceil(WAIT_CYCLES * dummy_phase_lines / 8)`。Quad I/O 的 mode byte 仍是独立字段，
不能重复计入 dummy bytes。

因此在重放后的 `spi-patch` 添加共同提交
`hw/ssi: Adapt dummy cycles to byte-wide flash model`；PIO timing 和 IDMA 分支均以它为
父提交，并分别只保留各自一个功能提交。该次基线调整只复跑 qtest：PIO 相关组、IDMA
组和 QSPI 组通过；按本轮约束，未重新运行 U-Boot 或 Linux 端到端测试。

## 9. 参考

- QEMU 主线 RAM boot 说明：
  `qemu-camp-2026-k230/docs/system/riscv/k230.rst`
- 预构建资产和原始主线命令：
  `/tmp/k230-boot-assets-trto/README.md`
- SDK SPI-NOR 量产布局：
  `k230_sdk/board/common/gen_image_cfg/genimage-spinor.cfg`
- SDK SPL 板级代码：
  `k230_sdk/src/little/uboot/board/canaan/common/k230_spl.c`
- 本节固定资产库：
  `https://github.com/zevorn/k230-boot-assets.git`（提交
  `c3c32fb46e8307c5063f13e8f367c98bf9273cd1`）
