# K230 SPI/QSPI 最终处理报告

## 交付范围

分支 `k230_spi` 基于 `master`，保留 `spi-patch` 原分支不变。正式系列固定为 9 个提交；Flash、HI_SYS、XIP 位于最后三个提交，便于后续决定是否上游化。

| # | 提交 | 主题 | 验证基线 |
|---|---|---|---|
| 1 | `d2eed9d37b` | SSI register model | 构建 |
| 2 | `8128694c31` | K230 SSI instances | 构建 + `register-contract` |
| 3 | `69a77da082` | FIFO and PIO | 构建 + `pio-data-path` |
| 4 | `d282f7a03d` | interrupt controller | 构建 + 控制器内部 IRQ 断言 |
| 5 | `a800ac6e73` | PLIC routing | 构建 + PLIC 路由/隔离断言 |
| 6 | `b5ea90b196` | enhanced QSPI | 构建 + 无 Flash 接线的拒绝路径 |
| 7 | `7e12e2c72c` | SPI NOR attachment | 构建 + `spi-nor`/`qspi-sdr` |
| 8 | `92e3d3f9e0` | HI_SYS SSI control | 构建 + `hi-sys` |
| 9 | `985f53798a` | XIP read window | 构建 + `xip-read-window` |

清理工作仅涉及 SPDX/维护者元数据、英文注释、无效教学脚手架、格式和 qtest 组织；不会混入未经审阅的设备功能语义变更。

## 能力清单

| 分类 | 能力 | 证据 |
|---|---|---|
| 已实现 | 三个 SSI 实例、地址、FIFO、DR 别名、4--32 bit 帧、四种 `TMOD` | [TRM 12.3](K230_Technical_Reference_Manual_V0.3.1_20241118.txt)，[模型](../../my-qemu-camp-2026-k230/hw/ssi/k230_dw_ssi.c)，[控制器 qtest](../../my-qemu-camp-2026-k230/tests/qtest/k230-dw-ssi-test.c) |
| 已实现 | 标准 PIO、loopback、动态状态、watermark、中断 RC、PLIC 九路映射 | [SDK 驱动](../../k230_sdk/src/big/rt-smart/kernel/bsp/maix3/board/interdrv/spi/drv_spi.c)，[控制器 qtest](../../my-qemu-camp-2026-k230/tests/qtest/k230-dw-ssi-test.c) |
| 已实现 | Dual/Quad SDR PIO：指令、地址、mode bits、dummy cycles、读写和 RX FIFO 恢复 | [U-Boot 驱动](../../k230_sdk/src/little/uboot/drivers/spi/designware_spi.c)，[qtest](../../my-qemu-camp-2026-k230/tests/qtest/k230-dw-ssi-test.c) `qspi-sdr` |
| 已实现 | SPI NOR 挂接后的 JEDEC、读、页写、扇区擦、CS 命令边界 | [qtest](../../my-qemu-camp-2026-k230/tests/qtest/k230-dw-ssi-test.c) `spi-nor` |
| 已实现 | `SSI_CTRL` 的 XIP enable、三实例模式/休眠状态 | [HI_SYS 模型](../../my-qemu-camp-2026-k230/hw/misc/k230_hi_sys.c)，[qtest](../../my-qemu-camp-2026-k230/tests/qtest/k230-dw-ssi-test.c) `hi-sys` |
| 有限制支持 | `spi0` XIP 只读窗口，支持 Standard/Dual/Quad SDR、地址、mode bits、dummy cycles 和 1/2/4/8 字节读 | [TRM 5.3](K230_Technical_Reference_Manual_V0.3.1_20241118.txt)，[XIP 模型](../../my-qemu-camp-2026-k230/hw/ssi/k230_dw_ssi.c)，[qtest](../../my-qemu-camp-2026-k230/tests/qtest/k230-dw-ssi-test.c) `xip-read-window` |
| 有限制支持 | `BAUDR`、`RX_SAMPLE_DELAY`、`SCPOL/SCPH/SSTE` 的寄存器契约 | [TRM 12.3](K230_Technical_Reference_Manual_V0.3.1_20241118.txt)，[模型](../../my-qemu-camp-2026-k230/hw/ssi/k230_dw_ssi.c)；不模拟引脚波形/时间 |
| 有限制支持 | `SPI_CTRLR0`、DMA 相关寄存器保存与掩码 | [SDK 结构](../../k230_sdk/src/big/rt-smart/kernel/bsp/maix3/board/interdrv/spi/drv_spi.c)，[寄存器审阅](k230-spi-qspi-register-audit.md) |
| 未实现 | 内部 AXI DMA 实际 guest-memory 搬运、DONE/AXIE 事件 | [TRM 12.3.3.4](K230_Technical_Reference_Manual_V0.3.1_20241118.txt)，[审阅表](k230-spi-qspi-register-audit.md) |
| 未实现 | Octal、DDR、RXDS/HyperBus 数据事务 | [TRM 5.3](K230_Technical_Reference_Manual_V0.3.1_20241118.txt)，[审阅表](k230-spi-qspi-register-audit.md) |
| 未实现 | concurrent XIP、dynamic wait、XIP 写扩展 | [RT-Smart 条件布局](../../k230_sdk/src/big/rt-smart/kernel/bsp/maix3/board/interdrv/spi/drv_spi.c)，[审阅表](k230-spi-qspi-register-audit.md) |

## qtest 收敛与覆盖

qtest 从 P2 起即收敛为一个源文件：[k230-dw-ssi-test.c](../../my-qemu-camp-2026-k230/tests/qtest/k230-dw-ssi-test.c)，Meson 也只引用该文件。

原有细粒度断言按寄存器、PIO、IRQ/PLIC、SPI NOR、QSPI SDR、HI_SYS、XIP 合并为 7 个端到端场景：`register-contract`、`pio-data-path`、`interrupt-routing`、`spi-nor`、`qspi-sdr`、`hi-sys`、`xip-read-window`；最终 TAP 为 **1..7，7/7** 通过。

## 审阅结果与复现

详细寄存器/行为结论见：[K230 SPI/QSPI 寄存器与行为审阅表](k230-spi-qspi-register-audit.md)。本轮无“疑似错误”，因此没有未经用户确认的功能修正。

```sh
ninja -C build qemu-system-riscv64 tests/qtest/k230-dw-ssi-test
TMPDIR=/tmp/qemu-k230-qtest-final \
QTEST_QEMU_BINARY=$PWD/build/qemu-system-riscv64 \
build/tests/qtest/k230-dw-ssi-test --tap

git diff --check
git format-patch --stdout --cover-letter --numbered master..k230_spi | \
  perl scripts/checkpatch.pl --no-tree -
```

最终还应以 `git range-diff master...spi-patch master...k230_spi` 确认生产功能差异仅来自既定 9 个补丁，清理后的差异只包含英文/格式、SPDX/MAINTAINERS 与单一 qtest 文件重构。
