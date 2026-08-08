# K230 SPI (V2 dw_ssi) Reviewer 测试指南：从 0 复现

> 适用分支：`k230-spi-v2pacth-s1`（V2 通用 DW SSI 模型重构）
> 文档版本：2026-08-01
> 验证结论：qtest 14/14 全绿；U-Boot SPI PIO 读写通过；Linux 5.10.4 SPI PIO 读写通过（MTD PASS）

本文档面向没有本机历史上下文的 reviewer，按顺序执行即可从零复现
K230 SPI（V2 `dw_ssi`）的全部测试。所有命令默认在 WSL Ubuntu 22.04 内执行。

---

## 0. 目录速览

| 测试 | 工具 | 结果 |
|---|---|---|
| 单元/控制器行为 | qtest (`k230-dw-ssi-test`) | 14/14 PASS |
| U-Boot SPI PIO 读写 | `sf` 命令 | erase/write/read/cmp 全 OK |
| Linux SPI PIO 读写 | Buildroot Linux 5.10.4 + MTD | `K230_MTD_TEST_PASS` |

---

## 1. 环境与前置资产

### 1.1 需要的目录/仓库

```text
/home/flamboy/qemu-camp/my-qemu-camp-2026-k230/   QEMU 源码（V2 分支）
/home/flamboy/qemu-camp/k230-boot-assets/         启动镜像资产（OpenSBI/U-Boot/Linux）
/home/flamboy/qemu-camp/k230_sdk/                 K230 SDK（源码参考，测试本身不依赖）
```

### 1.2 关键镜像资产

| 文件 | 用途 |
|---|---|
| `k230-boot-assets/common/fw_jump.uImage` | OpenSBI v0.9（`bootm` 用） |
| `k230-boot-assets/common/u-boot` | 原始 SDK U-Boot（SPI 默认禁用，需启用） |
| `/tmp/u-boot-spi-enabled-v2` | **已启用 SPI0 的 U-Boot 副本（必须）** |
| `k230-boot-assets/yocto/uboot-boot/Image` | Yocto Linux 6.18.28（无 SPI 驱动，仅启动链用） |
| `k230-boot-assets/buildroot/uboot-boot/Image` | Buildroot Linux 5.10.4（**含 SPI 驱动，Linux 测试用**） |
| `/tmp/k230-spi-final-89-linux-standard-rerun.bin` | 32 MiB 测试镜像（Linux 5.10.4 + MTD 测试 initramfs） |

> `/tmp/u-boot-spi-enabled-v2` 与 `/tmp/k230-spi-final-89-linux-standard-rerun.bin`
> 是此前构建/组装的临时资产；若 /tmp 被清理，需按附录 A、B 重建。

---

## 2. 构建 QEMU（V2 分支）

```bash
cd /home/flamboy/qemu-camp/my-qemu-camp-2026-k230
git checkout k230-spi-v2pacth-s1
mkdir -p build && cd build
../configure --target-list=riscv64-softmmu --disable-werror
make -j$(nproc)
```

产物：`build/qemu-system-riscv64`

---

## 3. qtest（14 个控制器级测试）

### 3.1 构建 qtest

```bash
cd /home/flamboy/qemu-camp/my-qemu-camp-2026-k230/build
make -j$(nproc) tests/qtest/k230-dw-ssi-test
```

### 3.2 运行

```bash
QTEST_QEMU_BINARY=./qemu-system-riscv64 \
    timeout 180 ./tests/qtest/k230-dw-ssi-test
```

### 3.3 预期输出

```text
TAP version 13
1..14
ok 1 /riscv64/k230-dw-ssi/register-contract
ok 2 /riscv64/k230-dw-ssi/pio-data-path
ok 3 /riscv64/k230-dw-ssi/interrupt-controller
ok 4 /riscv64/k230-dw-ssi/plic-routing
ok 5 /riscv64/k230-dw-ssi/tx-only-mode
ok 6 /riscv64/k230-dw-ssi/eeprom-read-contract
ok 7 /riscv64/k230-dw-ssi/rx-fifo-overflow
ok 8 /riscv64/k230-dw-ssi/txfthr-start-gate
ok 9 /riscv64/k230-dw-ssi/baudr-zero
ok 10 /riscv64/k230-dw-ssi/ser-terminates-ro
ok 11 /riscv64/k230-dw-ssi/icr-total-clear
ok 12 /riscv64/k230-dw-ssi/unsupported-registers
ok 13 /riscv64/k230-dw-ssi/flash-jedec-id
ok 14 /riscv64/k230-dw-ssi/flash-fixed-read
```

### 3.4 测试覆盖说明

| 测试 | 覆盖点 |
|---|---|
| register-contract | 寄存器默认值/复位值契约 |
| pio-data-path | TR 模式 PIO 收发（FIFO 深度 64） |
| interrupt-controller | 9 路中断（TXE/RXF level；TXO/RXO/TXU/RXU latched） |
| plic-routing | K230 三实例到 PLIC 的路由（146~172） |
| tx-only-mode | TO 模式写入后 TFE/TXFLR 行为 |
| eeprom-read-contract | EEPROM_READ 模式 NDF 帧数契约 |
| rx-fifo-overflow | 写入 256+1 帧触发 RXO 并可清除 |
| txfthr-start-gate | TXFTHR 水位不足不启动传输（TRM 启动门槛） |
| baudr-zero | SCKDV=0 关闭 sclk_out，不传输 |
| ser-terminates-ro | SER 切换终止待决 RO/EEPROM 事务且不清 FIFO |
| icr-total-clear | RXU+RXO 总清除 |
| unsupported-registers | DMACR/XIP_MODE_BITS/SPI_CTRLR0 只读 0 |
| flash-jedec-id | spi0 挂接 m25p80 的 JEDEC 识别 0x20 0x20 0x14 |
| flash-fixed-read | 固定地址 Standard 1-1-1 读返回擦除态 0xff |

---

## 4. U-Boot SPI PIO 读写验证

### 4.1 准备测试镜像

使用 flash 镜像 `/tmp/k230-spi-v2-flash.img`（32 MiB，从 k230-boot-assets 组装）。
重建方法：

```bash
ASSETS=/home/flamboy/qemu-camp/k230-boot-assets
FLASH=/tmp/k230-spi-v2-flash.img
truncate -s 32M "$FLASH"
dd if="$ASSETS/common/fw_jump.uImage" of="$FLASH" conv=notrunc status=none
dd if="$ASSETS/yocto/uboot-boot/Image" of="$FLASH" bs=1M seek=1 conv=notrunc status=none
dd if="$ASSETS/yocto/uboot-boot/rootfs.cpio.gz" of="$FLASH" bs=1M seek=28 conv=notrunc status=none
dd if="$ASSETS/yocto/uboot-boot/k230-canmv.dtb" of="$FLASH" bs=1M seek=31 conv=notrunc status=none
```

### 4.2 自动验证脚本

`run-uboot-spi-pio.py`（位于 `qemu-camp/spi-v2-test-scripts/`）流程：
`sf probe 0:0` → `mw.b 0x1100000 0x5a 0x10000` → `sf erase 0x200000 0x10000`
→ `sf write 0x1100000 0x200000 0x10000` → `sf read 0x1200000 ...` → `cmp.b`

```bash
cd /home/flamboy/qemu-camp/spi-v2-test-scripts
timeout 300 python3 run-uboot-spi-pio.py
```

> 注意：`sf write` 目标偏移 0x200000 位于 Yocto Image 区域内，运行后 Flash
> 会被改写。验证后应恢复基线：`cp /tmp/k230-spi-v2-flash.img.bak-yocto /tmp/k230-spi-v2-flash.img`

### 4.3 预期输出

```text
=== U-Boot ready ===
[1.probe] SF: Detected w25q256 ...
[2.erase] SF: 65536 bytes @ 0x200000 Erased: OK
[3.write] SF: 65536 bytes @ 0x200000 Written: OK
[4.readback] SF: 65536 bytes @ 0x200000 Read: OK
[5.compare] （无 mismatch 输出）
```

---

## 5. Linux SPI PIO 读写验证（MTD）

### 5.1 关键点

必须使用 **Buildroot Linux 5.10.4**（`dw_spi_mmio` + `spi-nor` 驱动齐全）。
Yocto 6.18.28 镜像没有编译 SPI 驱动（`snps,dwc-ssi-1.01a` 出现 0 次），
不能用于本测试。K230 SDK 5.10 内核有驱动但无法在 QEMU 上启动
（`exec_page_fault`，SDK 内核与 QEMU 模型不兼容），也不要使用。

### 5.2 测试镜像布局（/tmp/k230-spi-final-89-linux-standard-rerun.bin）

| 内容 | Flash 偏移 | RAM 地址 | 长度 |
|---|---:|---:|---:|
| OpenSBI uImage | `0x0` | `0xc100000` | `0x13fc8` |
| Buildroot Image (5.10.4) | `0x100000` | `0x8200000` | `0x198b000` |
| initramfs（含 MTD 测试） | `0x1c00000` | `0xa100000` | `0x1eb50a` |
| 定制 DTB（含 spi@91584000 + qemu-test 分区） | `0x1e00000` | `0xa000000` | `0x987` |

> DTB 由 `k230-linux-standard.dts` 编译：`spi@91584000` 挂 `flash@0`
> （`jedec,spi-nor`），分区 `qemu-test @ 0x1f00000 size 0x10000`。

### 5.3 自动验证脚本

`run-linux-mtd-pio.py`（位于 `qemu-camp/spi-v2-test-scripts/`）：

```bash
cd /home/flamboy/qemu-camp/spi-v2-test-scripts
timeout 320 python3 run-linux-mtd-pio.py
```

### 5.4 预期输出（关键行）

```text
[4] Linux 5.10.4 started
  | dw_spi_mmio 91584000.spi: IRQ index 9 not found
  | spi-nor spi0.0: w25q256 (32768 Kbytes)
  | 1 fixed-partitions partitions found on MTD device spi0.0
  | 0x000001f00000-0x000001f10000 : "qemu-test"
  | 1+0 records in / 1+0 records out (x3)
  | K230_MTD_TEST_PASS mode=standard
[5] LINUX MTD PIO TEST PASS
```

含义：Linux 内核通过 `dw_spi_mmio` 驱动 probe 到 SPI 控制器，`spi-nor`
识别 W25Q256，建立 MTD 分区；initramfs 中的测试程序对该分区执行
**erase → write(256B) → readback → 逐字节校验**，全部成功。

> `IRQ index 9 not found` 是 DTB 中断数量与驱动期望的差异警告，不影响
> PIO 读写，可忽略。这是 V1→V2 差异之一：V1 曾在此处报
> `Retry of enh_mem_op failed` + `/dev/mtd0` EIO，V2 已修复。

---

## 6. 排错指南

| 现象 | 原因与处理 |
|---|---|
| `QTEST_QEMU_BINARY required` | 忘记设置环境变量，见 3.2 |
| qtest 某些用例失败 | 检查 `hw/ssi/dw_ssi.c` 与 `k230.c` 的 IRQ 路由/阈值实现 |
| `Invalid bus 0 (err=-19)` | U-Boot 控制 DTB 未启用 `spi0@91584000`，用 `/tmp/u-boot-spi-enabled-v2` |
| `No SPI flash selected` | `sf probe 0:0` 未成功，先确认 U-Boot 副本含 SPI 支持 |
| Linux 无 `/proc/mtd` | 内核无 SPI 驱动：换 Buildroot 5.10.4 镜像，勿用 Yocto 6.18 |
| Linux 启动卡 OpenSBI 后 | 误用 SDK 5.10 内核：换 Buildroot Image |
| MTD 写入 EIO | V1 已知问题；V2 应 PASS。若复现，检查 `enh_mem_op`/CS 时序 |
| `Software reset enable failed: -524` | U-Boot jedec 探测警告，随后出现 `SF: Detected w25q256` 即正常 |

---

## 附录 A：重建 SPI 启用的 U-Boot

U-Boot 控制 DTB 需启用 `spi0@91584000` 并声明 `flash@0`：

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

编译方法见 `exper-note/k230/spi/k230-spi-flash-uboot-linux-quickstart.md`。

## 附录 B：重建 Linux 测试镜像

1. Buildroot Image/initramfs 来自 `k230-boot-assets/buildroot/`。
2. 定制 DTB：`/tmp/k230-linux-standard.dts`（含 spi 节点 + qemu-test 分区），
   `dtc -I dts -O dtb` 编译。
3. initramfs 内置 `k230-linux-mtd-test.c`（编译为静态二进制，由 `/init`
   自动执行；执行 erase + pwrite + pread + 校验）。
4. 按 5.2 布局用 `dd` 组装 32 MiB 镜像。

---

## 附录 C：参考文档

- 启动链复现笔记：`exper-note/k230/spi/k230-spi-flash-uboot-linux-quickstart.md`
- V1 报告（含 V1 Linux 失败记录）：`exper-note/k230/spi/k230-spi-qspi-final-report.md`
- V2 决策/计划：`exper-note/k230/upstream-review/k230-spi-qspi-review-v2-decision-notes.md`
- V1 博客：`exper-note/k230/spi/k230-spi-v3.4-blog.md`