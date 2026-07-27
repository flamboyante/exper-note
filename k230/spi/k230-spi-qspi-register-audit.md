# K230 SPI/QSPI 寄存器与行为审阅表

## 范围与证据

- 英文原始 TRM：[K230 Technical Reference Manual V0.3.1](K230_Technical_Reference_Manual_V0.3.1_20241118.txt)，重点为 12.3 SPI（PDF 778--808 页）和 5.3 FMC/XIP。
- SDK：RT-Smart [drv_spi.c](../../k230_sdk/src/big/rt-smart/kernel/bsp/maix3/board/interdrv/spi/drv_spi.c)、U-Boot [designware_spi.c](../../k230_sdk/src/little/uboot/drivers/spi/designware_spi.c)、[U-Boot DTS](../../k230_sdk/src/little/uboot/arch/riscv/dts/k230.dtsi)、[Linux DTS](../../k230_sdk/src/little/linux/arch/riscv/boot/dts/kendryte/k230.dtsi)。
- 逐位依据和条件布局见既有学习文档：[TRM 12.3 中文对照](k230-trm-12.3-spi-cn.md)。该文档用于定位，不以 QEMU 实现反证硬件。

结论含义：`吻合` 表示所实现的寄存器契约/模型行为与证据一致；`明确未实现` 是已经验证并对外声明的范围边界；`疑似错误` 需要用户审阅后才允许变更功能代码。

| 审阅项 | TRM 英文定位 | SDK 证据 | 当前 QEMU | 结论 |
|---|---|---|---|---|
| 三实例地址、FIFO 深度、数据宽度 | 12.3.1、12.3.5 | `drv_spi.c` 的 `dr[36]`；两个 DTS | `spi0/1/2`、256 深度、4--32 bit DFS；[模型](../../my-qemu-camp-2026-k230/hw/ssi/k230_dw_ssi.c) | 吻合 |
| `CTRLR0`/`CTRLR1`/`SSIENR`/`SER` 写掩码与禁用时配置 | 12.3.4/12.3.5 | `designware_spi.c` 在配置前写 `SSIENR=0` | 保存掩码、使能保护、片选范围按实例限制；[qtest](../../my-qemu-camp-2026-k230/tests/qtest/k230-dw-ssi-test.c) `register-contract` | 吻合 |
| 四种 `TMOD` 与 PIO FIFO | 12.3.3.1 | `drv_spi.c` 的 read/write/read-write 路径 | TR/TO/RO/EEPROM-read、FIFO 满/空和暂停恢复 | 吻合 |
| `BAUDR`、`RX_SAMPLE_DELAY`、标准 SPI 时序字段 | 12.3.3.2--12.3.3.3 | SDK 写入分频及采样延迟 | 掩码/读回支持；QEMU SSI 总线不生成引脚级时钟波形 | 有限制支持 |
| 水位、原始/屏蔽中断、RC 清除、九路 PLIC 映射 | 12.3.4（`IMR` 至 `ICR`） | DTS IRQ 编号和 RT 驱动布局 | 动态 TXE/RXF、TXO/RXO/RXU、RC、每实例九路 IRQ；[控制器 qtest](../../my-qemu-camp-2026-k230/tests/qtest/k230-dw-ssi-test.c) `interrupt-routing` | 吻合 |
| Dual/Quad SDR 增强事务 | 12.3.3 增强 SPI；5.3 FMC | RT/U-Boot 的 `SPI_CTRLR0` 配置 | 指令、地址、mode bits、dummy cycles、RO/TO、FIFO 恢复；[qtest](../../my-qemu-camp-2026-k230/tests/qtest/k230-dw-ssi-test.c) `qspi-sdr` | 吻合 |
| `SPI_CTRLR0` DDR、RXDS、Octal | 5.3 FMC 与 12.3 SPI | SDK 定义这些字段，`spi0` 为八线实例 | 字段可读写；DDR/RXDS/Octal 事务被显式拒绝且不消费 FIFO | 明确未实现 |
| 内部 AXI DMA：`DMACR`、`SPIDR`、`SPIAR`、`AXIAR*`、DONE/AXIE | 12.3.3.4、12.3.4 | SDK `SSIC_HAS_DMA=2` 和 DMA 配置路径 | 寄存器掩码与读回支持；不访问 guest memory，不产生 DONE/AXIE | 明确未实现 |
| `spi0` XIP 读窗口和 HI_SYS `SSI_CTRL` | 5.3 FMC、地址图 | `board.h` 的 `0xc0000000`，RT 布局 | XIP enable gate、1/2/4 线 SDR、地址/模式/空等待；[qtest](../../my-qemu-camp-2026-k230/tests/qtest/k230-dw-ssi-test.c) `xip-read-window` | 有限制支持 |
| concurrent XIP、dynamic wait、XIP write 扩展 | 5.3 FMC | RT 条件布局未启用 `0x108--0x11c` 和 `0x140--0x148` | RAZ/WI；XIP 写记录 guest error | 明确未实现 |
| Flash 外设协议 | 外部 Flash 不是 SSI TRM 的寄存器语义 | U-Boot SPI-NOR 使用路径 | 使用 QEMU `m25p80`；JEDEC/read/program/erase 均有 qtest | 有限制支持 |

## 疑似错误

本次依据英文 TRM 与三套 SDK 代码的只读审阅，**未发现需要先由用户裁决的功能语义疑似错误**。以下不是错误结论，而是后续需要真机或更完整 SDK 证据才能升级实现的项目：

1. `DMACR.AINC` 的 TRM 表述与地址映射说明存在方向性文字冲突；SDK 实际连续传输均置位 bit 6。当前模型只保存该字段，尚未执行 DMA，因此没有擅自选择运行时语义。
2. 12.3 对 Dual/Quad 的描述与 5.3/SDK 中 `spi0` 八线 OPI 能力范围不同。当前模型保留 `spi0` 的八线 profile，但拒绝 Octal 数据事务。
3. `SCPOL`、`SCPH`、`SSTE` 和采样延迟属于线级可观察时序；QEMU 的 SSI 抽象总线不暴露波形，当前只保证控制器与外设事务级语义。

上述项目若要改动行为，应先提供真机读回、波形或 SDK 实际调用序列，并经审阅后修改对应功能提交。
