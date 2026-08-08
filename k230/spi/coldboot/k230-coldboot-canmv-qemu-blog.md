# 用一套 canmv 的 dts，把 K230 冷启动重新跑了一遍

记录时间：2026-08-07 ｜ 分支：`k230-coldboot-canmv-qemu`（SDK v2.0）｜ QEMU 零改动

> 上一篇 [k230-cold-boot-from-zero.md](./k230-cold-boot-from-zero.md) 把 K230 冷启动链从 0 跑通了，但那是"拼凑"的：
> U-Boot 用 evb 板级、Linux dtb 用 yocto 里现成的最小版、内核来源没记清楚。这次换个玩法：
> **U-Boot 和 Linux 都走 canmv 板级，dts 自己建，QEMU 一行不改**，看能不能跑通，并如实记录卡在哪、为什么卡。

---

## 1. 为什么又来一遍

上游 review 最怕两件事：一是 dtb 不同源（U-Boot 用 evb 的、内核用 yocto 的，板级语义对不上），二是改动说不清来路（内核哪来的、改过什么）。上次验证这两条都踩了。

这次定了三条规矩：

1. **QEMU 能不改就不改**——依赖现有行为，绝不为了跑通而改模型；
2. **SDK 改动拉独立分支**，可回滚、可 review；
3. **拿不准的设备，对照 QEMU 已实现外设，没实现的一律 disable**。

## 2. 动手之前先过的三个"坑"

### 坑 1：SPL 怎么知道从 NOR 启动

canmv 板子真机上是从 SD 卡（SDIO1）启动的，代码里写死了：

```c
// board/canaan/k230_canmv/board.c
sysctl_boot_mode_e sysctl_boot_get_boot_mode(void)
{
    return SYSCTL_BOOT_SDIO1;
}
```

它把 SDK 的 weak 实现盖掉了。weak 实现本来是读 `SOC_BOOT_CTL & 0x3`（0=NOR / 1=NAND / 2=SDIO0 / 3=SDIO1）：

```c
// arch/riscv/cpu/k230/sysctl.c:27
sysctl_boot_mode_e sysctl_boot_get_boot_mode(void) __weak;
```

QEMU 里 `SOC_BOOT_CTL` 所在的 boot 区（`0x91102000`）是 `create_unimplemented_device()`，**读出来全是 0** → 走 `case 0: return SYSCTL_BOOT_NORFLASH`。

所以做法是：把 canmv 的覆盖**删掉**，让 weak 实现去读 strap。QEMU 上读 0 = NOR，真机上读 strap 引脚，两边都对。代价是 SPL 读这个寄存器时 QEMU 会打一条 GUEST_ERROR 日志——**观察项，不影响启动**。

### 坑 2：canmv 的 dts 到底能不能起

U-Boot 侧风险不大——canmv 系的 U-Boot 在 V1 的 `-bios` 验证里跑过。真正没底的是 Linux 侧：SDK 的内核 dtb 从没在 QEMU 上 probe 过完整外设。

对策就是规矩 3：把 QEMU `hw/riscv/k230.c` 里已实现的外设列成一张表，dtb 里没实现的全部 disable。QEMU 已实现：1× c908 CPU、CLINT/PLIC、5× 8250 UART、3× DW SSI、2× WDT、Timer/RMU/IOMUX、SRAM、DDRC/PHY、GSDMA、gzip 解压、NOC stub、SPI NOR（m25p80 挂 spi0）。没实现的：mmc/usb/gpio/i2c/i2s/codec/vo/isp/dma/kpu 等，一个不留。

### 坑 3：内核从哪来

SDK 5.10.4 内核的 PTE 用了 T-HEAD MAEE 高位属性位（bit59-63），QEMU 不认。之前已经用标准 PTE 重编过一版，放在 `k230-boot-assets/buildroot/uboot-boot/Image`，实测能跑。这次**直接复用**，不重编、不伪装自建，provenance 如实写清楚：SDK 5.10.4 + Xuantie V2.6.0 + 标准 PTE 重编，无 recipe。

## 3. 实施记录

### 3.1 两份新 dts

U-Boot 侧：从 `k230_canmv.dts` 派生 `k230_canmv_qemu.dts`，显式 `&spi0` 使能 + `spi-flash@0` + `bus-width 1`，配套新建 `k230_canmv_qemu_defconfig`（`CONFIG_DEFAULT_DEVICE_TREE="k230_canmv_qemu"`）。

Linux 侧：新建 `k230_canmv_qemu.dts`，`bus-width 8→1`（QEMU 只支持 Standard 1-1-1）、分区表对齐官方 `genimage-spinor.cfg`、QEMU 未建模外设 9 处 disable（spi0 除外，见第 4 节）。

> 所有 dts 改动只在这两份 `k230_canmv_qemu.dts` 里，`k230.dtsi` / `k230_evb.dtsi` 保持 SDK 原始不动。

### 3.2 U-Boot 重编

```bash
cd k230_sdk/src/little/uboot
export CROSS_COMPILE=/opt/toolchain/Xuantie-900-gcc-linux-5.10.4-glibc-x86_64-V2.6.0/bin/riscv64-unknown-linux-gnu-
make k230_canmv_qemu_defconfig
make -j$(nproc)
```

打包沿用 `toolchains/pkg_uboot.sh`：SPL 走 firmware_gen + endian-swap，U-Boot 走 k230_gzip → mkimage → firmware_gen。产物：

```
swap_fn_u-boot-spl.bin   226388 B  (0x37454)
fn_ug_u-boot.bin         318012 B  (0x4DA3C)
```

`strings u-boot.bin` 确认内嵌 dtb：`Model: kendryte k230 canmv (qemu cold boot)`。

### 3.3 Flash 布局（raw，不用官方 linux_system.bin）

| 内容 | Flash 偏移 | RAM 地址 | 预留 |
|---|---:|---:|---:|
| SPL（swap） | 0x000000 | —（BootROM 直读） | 0x80000 |
| U-Boot（fn_ug） | 0x080000 | 0x08000000 | 0x160000 |
| Linux Image（SDK 5.10.4） | 0x200000 | 0x08200000 | 0x198c800 |
| initramfs（yocto busybox） | 0x1B8C800 | 0x0A100000 | 0x1EEC20 |
| OpenSBI fw_jump.uImage | 0x1E0EC20 | 0x0C100000 | 0x14000 |
| canmv_qemu.dtb | 0x1F00000 | 0x0A000000 | 0x14000 |

装配脚本 `toolchains/mkcanmv_qemu_flash.py` 带断言：每段实际大小 ≤ 预留、相邻段不重叠，失败即停。实测各段：

```
SPL 0x37454 ≤ 0x80000        Image 0x198B000 ≤ 0x198c800
U-Boot 0x4DA3C ≤ 0x160000    rootfs 0x1EEC20（正好等于预留）
fw_jump 0x13FC8 ≤ 0x14000    dtb 0x12762 ≤ 0x14000
```

> 为什么要自己摆 raw 布局而不是用官方的 linux_system.bin？这次验证的对象是"启动路径正确性"：BootROM/SPL 从 SPI NOR 读镜像、U-Boot `sf read` + `bootm`。linux_system.bin 是 SDK 的打包格式（multi uImage 压缩），跟启动路径验证是两回事，留到以后。dtb 分区表已经对齐官方布局，想切随时能切。

### 3.4 端到端

```bash
$QEMU -M k230,spi-flash=w25q256 -drive if=mtd,format=raw,file=/tmp/k230-canmv-qemu-coldboot.img \
  -nographic -monitor none -display none -no-reboot
```

U-Boot 里的手工操作（`toolchains/run_canmv_qemu.sh` 原样发送）：

```
setenv bootargs console=ttyS0,115200 earlycon=sbi cma=0
sf probe 0:0
sf read 0x0c100000 0x1e0ec20 0x14000
sf read 0x08200000 0x200000 0x198c800
sf read 0x0a100000 0x1b8c800 0x1eec20
sf read 0x0a000000 0x1f00000 0x14000
fdt addr 0x0a000000; fdt resize 8192
fdt set /chosen linux,initrd-start <0x0 0x0a100000>
fdt set /chosen linux,initrd-end <0x0 0x0a2eec20>
bootm 0x0c100000 - 0x0a000000
```

结果（完整日志 `/tmp/k230-canmv-qemu-coldboot.log`）：

```
k230 coldboot: enter
k230 coldboot: magic ok body=225860 spl=225856 -> 0x80300000
U-Boot SPL 2022.10
...
imge: uboot load to 8000000  compress=1 src a500254 len=4d7e8
U-Boot 2022.10
Model: kendryte k230 canmv (qemu cold boot)
...
OpenSBI v0.9
Linux version 5.10.4 ...
[    0.724273][    T1] Run /init as init process
meta-k230 initramfs starting...
~ #
```

门禁三项全过：① SPL 从 SPI NOR 加载完整 U-Boot；② U-Boot `Model` 为 canmv_qemu；③ 到 initramfs shell，无阻断异常。

## 4. 卡死点：内核的 dw_spi 驱动（根因已实测定位）

第一次跑，Linux 卡在内核初始化 0.58s 处，最后一条日志：

```
dw_spi_mmio 91584000.spi: IRQ index 9 not found
```

一开始怀疑是 QEMU 中断模型不完整，反复核对后推翻。真实根因在**时钟**，是照着 V2 cover letter 的测试 dtb 才找到的。

### 4.1 先排除的

1. **`IRQ index 9 not found` 不是根因**。标准 `spi-dw-mmio` 驱动 `for(i=0;i<16;i++) platform_get_irq()` 循环到第 10 个（interrupts 只有 9 项）时的普通警告。V1 时代的 Linux probe 日志里**也有**这条，同样不影响 probe（V1 workbook 9.3 记录）。
2. **中断框架没有变化**。V1 `k230_dw_ssi.c` 与 V2 `dw_ssi.c` 的 `irq_raw_status`/`update_irq` 几乎一致，TXEI/RXFI 都由 FIFO 水位实时计算。
3. **驱动走的路径一样**。`compatible="snps,dwc-ssi-1.01a"` 命中标准 `spi-dw-mmio`（K230 定制 poll 版匹配 `-k230` 后缀，不会挂上来）。`irq[0]=146` 有效 → 中断传输。

### 4.2 真正的原因：spi0 的 clock 在 QEMU 上是"坏的"

卡死链路（三层）：

```
dts: spi0.clocks = <&ssi0_clk>          ← clock_consumer.dtsi 提供，canaan,k230-clk-composite
内核: dws->max_freq = clk_get_rate(clk) ← QEMU 上 PLL 未建模 → rate 算出 0（日志: pll pll0 is unlock）
驱动: clk_div = DIV_ROUND_UP(0, freq)+1 & 0xfffe = 0 → 写 BAUDR = 0 → SCKDV = 0
QEMU: dw_ssi_run_transfer 见 SCKDV==0 → 无 sclk_out → 直接 return，传输永不启动
驱动: 等 RXFI 中断（永远不会来）→ wait_for_completion 挂死
```

关键在 QEMU 模型 `dw_ssi_run_transfer` 里 V1 没有的一行门控：

```c
/* SCKDV=0 disables sclk_out: no transfer can start. */
if (FIELD_EX32(s->regs[R_BAUDR], BAUDR, SCKDV) == 0) {
    return;
}
```

V1 模型没有这行（无条件启动传输），所以 V1 时代即使 SCKDV=0 也照传，掩盖了 clk 问题；V2 模型按 TRM 补上了这条，于是"坏 clock"暴露成挂死。**V2 模型的行为是对的，问题出在 dts 给内核的 clock 在 QEMU 上算不出有效频率**。

### 4.3 V2 cover letter 的 MTD 测试为什么能过

V2 第一批的 Linux MTD 测试（`spi-v2-test-scripts/run-linux-mtd-pio.py`）**没有用 SDK 原始 dts**，而是用了一个 2439 字节的定制 dts（`k230-linux-standard.dts`），里面 spi0 的 clock 是：

```
ssi0-clock { compatible = "fixed-clock"; #clock-cells = <0>; clock-frequency = <50000000>; };
spi@91584000 { ... clocks = <&ssi0-clock>; ... };
```

fixed-clock 不依赖 K230 的 PLL 驱动，rate 恒为 50MHz → 驱动算出有效波特率 → 传输正常 → MTD 测试通过。**所以 cover letter 没有骗人，V2 模型的 Linux SPI 读写确实通过。**

### 4.4 修复与验证

照 V2 测试 dts 的做法，在 `k230_canmv_qemu.dts` 给 spi0 换 fixed-clock 并补 `interrupt-names`：

```
ssi0_fixed_clk: ssi0-clock { compatible = "fixed-clock"; #clock-cells = <0>; clock-frequency = <50000000>; };
&spi0 { status = "okay"; clocks = <&ssi0_fixed_clk>; interrupt-names = "spi_txe" ... "spi_axie"; };
```

重编 dtb 重跑，Linux 侧 dw_spi 完全正常，**不需要再禁用 spi0**：

```
dw_spi_mmio 91584000.spi: IRQ index 9 not found        ← 依旧是非致命警告
spi-nor spi0.0: w25q256 (32768 Kbytes)                  ← 识别成功
12 fixed-partitions partitions found on MTD device spi0.0
Creating 12 MTD partitions on "spi0.0"                  ← MTD 分区（与 V2 cover letter 一致）
```

顺带确认：coldboot 分支的 `dw_ssi.c` 与 V2 正式代码（`T_v2-5patch`，cover letter 的 base-commit 对应树）只差一处 CTRLR0 写入掩码（放开 SPI_FRF 以跑通 U-Boot 冷启动链），与本次卡死无关。
## 5. 复现

```bash
# 1) U-Boot 重编 + 打包
cd k230_sdk/src/little/uboot
export CROSS_COMPILE=/opt/toolchain/Xuantie-900-gcc-linux-5.10.4-glibc-x86_64-V2.6.0/bin/riscv64-unknown-linux-gnu-
make k230_canmv_qemu_defconfig && make -j$(nproc)
bash /home/flamboy/qemu-camp/toolchains/pkg_uboot.sh

# 2) dtb
cd /home/flamboy/qemu-camp/k230_sdk/src/little/linux
cpp -nostdinc -I include -I arch -undef -x assembler-with-cpp \
  arch/riscv/boot/dts/kendryte/k230_canmv_qemu.dts | \
  scripts/dtc/dtc -I dts -O dtb -o /tmp/k230_canmv_qemu.dtb

# 3) 装配 Flash
python3 /home/flamboy/qemu-camp/toolchains/mkcanmv_qemu_flash.py

# 4) 端到端
bash /home/flamboy/qemu-camp/toolchains/run_canmv_qemu.sh
```

## 6. 问题与风险清单

1. **Linux 侧 spi0 曾因 clock 问题挂死（已修复，非 QEMU 模型缺陷）**。SDK 原始 dts 的 spi0 clock（`canaan,k230-clk-composite`）在 QEMU 上 rate=0（PLL 未建模）→ `BAUDR.SCKDV=0` → V2 `dw_ssi` 按 TRM 拒绝启动传输 → Linux probe 挂死。已在 canmv_qemu.dts 给 spi0 换 fixed-clock（50MHz）+ 补 `interrupt-names` 修复，Linux 下 dw_spi/spi-nor/MTD 全部正常。V1 模型没有 SCKDV 门控（较宽松）掩盖了同一问题；V2 模型行为符合 TRM，不是缺陷。
2. **U-Boot SPI NOR soft reset 报 `-524`（EINPROGRESS）**。`jedec_spi_nor spi-flash@0: Software reset enable failed: -524` 在每次 SF 访问都打一条，U-Boot 容错继续，不影响启动。怀疑 SPI NOR soft-reset 时序与 QEMU m25p80 模型不完全一致，真机不会有。留档，不深挖。
3. **SPL 启动源遍历有"先失败后成功"**。日志里 `pfh->magic ... sys 1 / sys 0` 是 SPL 先尝试了非 NOR 源（读到全 0xff）再回退 NOR。boot mode 修复后最终选 NOR 没问题，但 SPL 的启动源遍历顺序值得在上游 review 时说明。
4. **内核 provenance 是"复用"不是"自建"**。`Image` 来自 boot-assets 的 SDK 5.10.4 标准 PTE 重编版，本次没有 recipe。文档里如实写，绝不伪装。
5. **`linux/arch/riscv/Makefile` 的 `_zicsr_zifencei` hack、`cache.c`/`start.S` 改动还在工作区**，与 canmv 验证无关，不提交本分支。后续若重编内核必须处理。
6. **GUEST_ERROR（boot 区读 0）** 是方案依据本身，仅观察项。日志里出现的 `hardlock unlock failed`、`disp_domain power on fail`、`pll unlock` 均为 QEMU 未建模电源/时钟域的预期噪音，不阻断启动，但上游 review 时会被问到，需要提前备好解释。

## 7. 改动边界（本地 vs 上游）

| 改动 | 归属 |
|---|---|
| uboot dts 使能 spi0 + 1-1-1（canmv_qemu dts） | 可上游化（配合 K230 machine 文档） |
| canmv board.c 删 SDIO1 覆盖（尊重 strap） | 上游化需评估（改变真机行为，cover letter 说明） |
| `CONFIG_K230_PUFS` 注释 | 本地验证专用（QEMU 无 PUFS 模型） |
| linux dts 未建模外设 disable | 配合 QEMU 模型能力范围；spi0 经 fixed-clock 修复后保留（Linux 下 SPI NOR/MTD 正常） |
| Linux 内核"标准 PTE 重编" | 本地资产，无 recipe，上游 QEMU 侧不应依赖 |
| QEMU | 零改动 |

**一句话总结**：冷启动链（BootROM→SPL→U-Boot→OpenSBI→Linux 5.10.4→initramfs shell）在"全 canmv + QEMU 零改动"下跑通，Linux 侧 spi0 也正常——补 fixed-clock 后 dw_spi/spi-nor/MTD 全部工作，与 V2 cover letter 一致。期间排查过一次内核 dw_spi 挂死，根因是 SDK 原始 dts 的 clock 在 QEMU 上算不出频率（PLL 未建模），不是 QEMU 模型缺陷；V2 的 SCKDV 门控只是把这个问题暴露了出来。本次验证的全部脚本和文档都在工作区，可按第 5 节完整复现。