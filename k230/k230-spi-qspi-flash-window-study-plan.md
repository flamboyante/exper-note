# K230 SPI/QSPI QEMU 实施清单

更新时间：2026-07-13

## 文档定位

本文只回答四个问题：

```text
当前模型离 K230 SDK 兼容还缺什么？
按什么顺序实现？
每一步怎样验证？
最后怎样拆分 Patch？
```

寄存器定义、复位值、TRM 5.3/12.3 冲突及 SDK 证据统一查阅：

- [K230 TRM 12.3 SPI 中文学习版（含 5.3 FMC 对照）](k230-trm-12.3-spi-cn.md)

本文不再重复寄存器位域和冲突推导。若本清单与主文档不一致，以主文档为准。

## 1. 最终目标

最终模型严格服务 K230，而不是实现一套泛化 DWC SSI：

```text
三个 K230 SSI 实例可被 RT-Smart、U-Boot 和 Linux 正确访问；
普通 SPI PIO、Dual/Quad/Octal 事务具备正确的软件可见结果；
内部 AXI DMA、DONE/AXIE 和九路中断可运行；
spi0 可以通过 0xC0000000 XIP window 读取 SPI NOR；
HI_SYS SSI_CTRL 位于 0x91585068，不与控制器 DR2 混淆。
```

QEMU SSI bus 只模拟事务级结果，不模拟 IO0–IO7 的逐周期电平。暂不追求：

- 精确串行时钟耗时；
- DDR/RXDS 的模拟电气波形；
- HyperBus；
- SPI NAND；
- XIP write；
- 性能预取模型。

## 2. K230 实例

| SDK 逻辑实例 | 控制器基址 | 能力 | PLIC 中断 | QEMU profile |
|---|---:|---|---:|---|
| `spi0` | `0x91584000` | 8 线 OPI、XIP | 146–154 | FMC/OPI |
| `spi1` | `0x91582000` | 4 线 QOPI | 155–163 | SPI/QOPI |
| `spi2` | `0x91583000` | 4 线 QOPI | 164–172 | SPI/QOPI |

其他相关地址：

```text
HI_SYS_CONFIG      0x91585000–0x915853ff
SSI_CTRL           0x91585068
XIP Flash window   0xC0000000–0xC7FFFFFF
```

注意：QEMU 当前 `dw_ssi[]` 数组顺序按地址排列，与 SDK 逻辑 `spi0/spi1/spi2` 顺序不同。连接 PLIC 时必须显式映射，不能直接使用数组下标计算中断号。

## 3. 目标结构

### 3.1 共享控制器模型

三个实例继续共用 `K230DwSsiState`：

```text
K230DwSsiState
  ├── FMC/OPI profile
  │     max-lines = 8
  │     has-xip = true
  │     SPI_CTRLR0 reset = 0x28000200
  └── SPI/QOPI profile
        max-lines = 4
        has-xip = false
        SPI_CTRLR0 reset = 0x04000200
```

共同配置：

```text
FIFO depth          256
IMR reset           0x0000003f
IDR                 0xa1b2c3d5
SSIC_VERSION_ID     0x3130332a
Internal DMA        SSIC_HAS_DMA = 2
```

### 3.2 HI_SYS_CONFIG

`SSI_CTRL` 不进入 `k230_dw_ssi.c` 的 `0x068`：

```text
SPI base + 0x068     = DR2
HI_SYS + 0x068       = SSI_CTRL
```

单独使用轻量 `K230HiSysCfgState`，第一阶段只实现 `SSI_CTRL`，其他 SD/USB 系统配置寄存器可以继续保持未实现。

### 3.3 SPI NOR 与 XIP

```text
spi0 CS0
  -> QEMU SSI bus
  -> m25p80 SPI NOR

CPU load 0xC0000000
  -> spi0 FMC/OPI XIP path
  -> 同一个 SPI NOR 后端
```

W25Q256 可用于基础 SPI/QSPI 事务验证，但不能证明真实 Octal Flash 电气行为。

## 4. 实施顺序

### P0：寄存器与实例 profile

- [ ] 按 `has-xip` 区分 FMC/OPI 与 SPI/QOPI 复位值。
- [ ] `IMR` 复位值统一采用 `0x3f`。
- [ ] `AXIAWLEN/AXIARLEN` 使用 K230 内部 DMA 语义。
- [ ] `IDR`、`SSIC_VERSION_ID`、`BAUDR` 与主文档一致。
- [ ] FIFO 容量改为 256。
- [ ] `TXFTLR.TFT/RXFTLR.RFT` 支持 8 位。
- [ ] `TXFTLR.TXFTHR` 支持 11 位。
- [ ] `SSIENR=0` 清空 TX/RX FIFO 并终止当前事务。
- [ ] `0x108–0x11c` 按当前 SDK profile 作为保留区。
- [ ] 加入 `0x120–0x134` 内部 DMA 寄存器。

完成条件：寄存器读写、复位和保留地址行为不再与 K230 profile 冲突。

### P1：PIO、FIFO 和中断

- [ ] TX FIFO 进入实际发送路径。
- [ ] 支持 `TMOD=TR/TO/RO/EEPROM_READ`。
- [ ] TX-only 不向 RX FIFO 写入无效数据。
- [ ] RX-only 使用 dummy frame 启动时钟。
- [ ] `DFS` 决定数据帧宽度。
- [ ] `SR.BUSY/TFNF/TFE/RFNE/RFF` 反映真实状态。
- [ ] 实现 TXE/TXO/RXF/RXO/TXU/RXU/MST/DONE/AXIE 状态。
- [ ] 实现 `IMR/RISR/ISR` 的屏蔽关系。
- [ ] 实现读取清除寄存器。
- [ ] 每实例输出 9 根中断并接入正确 PLIC ID。

中断输出顺序固定为：

```text
TXE, TXO, RXF, RXO, TXU, RXU, MST, DONE, AXIE
```

完成条件：轮询和中断驱动的普通 SPI 事务均能结束，不残留 FIFO 或中断状态。

### P2：增强 SPI 与内部 AXI DMA

- [ ] 使用 `CTRLR0.SPI_FRF` 区分 Standard/Dual/Quad/Octal。
- [ ] 使用 `SPI_CTRLR0.TRANS_TYPE/INST_L/ADDR_L/WAIT_CYCLES` 组织事务阶段。
- [ ] 使用 `CTRLR1.NDF + 1` 决定数据帧数量。
- [ ] 实现 `SPIDR` 指令与 `SPIAR` 设备地址。
- [ ] 使用 `AXIAR0/1` 访问 guest memory。
- [ ] `AINC=1` 时递增内存地址。
- [ ] SPI 读：从 Flash 收数据并写入 guest memory。
- [ ] SPI 写：从 guest memory 取数据并发送到 Flash。
- [ ] 成功置位 DONE，读取 `DONECR` 清除。
- [ ] 内存访问错误置位 AXIE，读取 `AXIECR` 清除。
- [ ] 错误时撤销片选、清 FIFO 并禁用控制器。

完成条件：RT-Smart、U-Boot 和 Linux 的多线有数据传输路径不再因 DMA/DONE/AXIE 缺失而超时。

### P3：XIP 与 HI_SYS

- [ ] 在 `0x91585068` 实现 `SSI_CTRL`。
- [ ] 使用 `ssi0_xip_en` 门控 XIP window。
- [ ] XIP 仅连接 `spi0@0x91584000`。
- [ ] 使用 `SPI_CTRLR0` 决定指令、地址长度和 dummy cycles。
- [ ] 使用 `XIP_INCR_INST/XIP_WRAP_INST/XIP_MODE_BITS`。
- [ ] 删除按 opcode 硬编码地址长度和 dummy 的主要路径。
- [ ] 协调 XIP 与 PIO/DMA 的片选和 BUSY 状态。

完成条件：CPU 对 `0xC0000000` 的读取能够按 FMC/OPI 配置生成正确 SPI NOR 事务。

## 5. 测试清单

### 5.1 qtest

寄存器与 profile：

- [ ] 三个实例的 `CTRLR0/SR/IMR/IDR/VERSION` 复位值。
- [ ] FMC 与 SPI 的 `SPI_CTRLR0` 复位值不同。
- [ ] `0x050/0x054` 为 `AXIAWLEN/AXIARLEN`。
- [ ] `0x108–0x11c` 保留地址行为。
- [ ] `0x120–0x134` 可访问。
- [ ] `0x91585068` 与三个控制器 `base+0x68` 相互独立。

FIFO 与 PIO：

- [ ] 阈值 `0xff` 可以完整读回。
- [ ] FIFO 可以容纳 256 项。
- [ ] `SSIENR=0` 清空已有 RX/TX 数据。
- [ ] 四种 TMOD 的 RX/TX 结果。
- [ ] TX/RX overflow、underflow 和读取清除。

DMA 与中断：

- [ ] SPI NOR read 写回 guest memory。
- [ ] SPI NOR program 从 guest memory 取数据。
- [ ] AINC 地址递增。
- [ ] DONE/AXIE 状态及清除。
- [ ] 27 根 SPI 中断连接到 PLIC 146–172。

XIP：

- [ ] `ssi0_xip_en=0/1` 门控窗口。
- [ ] 三字节和四字节地址。
- [ ] mode bits 和 dummy cycles。
- [ ] 1/2/4/8 字节 CPU 读取。

### 5.2 SDK 验证

- [ ] RT-Smart `spi0/spi1/spi2` 初始化完成。
- [ ] U-Boot `sf probe` 与基础读取完成。
- [ ] Linux SPI host probe 不超时。
- [ ] SPI NOR JEDEC/SFDP/数据读取正确。
- [ ] 多线 DMA 事务产生 DONE 而不是超时。

不得把单个 qtest 绿色结果写成“完整支持 K230 QSPI/OPI”。

## 6. Patch 拆分

建议按能力边界拆分：

```text
Patch 1  hw/ssi: add K230 SSI register profiles and reset values
Patch 2  hw/ssi: implement K230 256-entry FIFO and PIO modes
Patch 3  hw/ssi: implement K230 interrupt status and clear behavior
Patch 4  hw/riscv: wire K230 SPI interrupt lines to the PLIC
Patch 5  hw/ssi: implement K230 enhanced SPI transaction formatting
Patch 6  hw/ssi: implement K230 internal AXI DMA
Patch 7  hw/misc: model K230 HI_SYS SSI_CTRL
Patch 8  hw/ssi: implement the K230 FMC XIP window
Patch 9  tests: add K230 SSI/QSPI register, DMA and XIP coverage
Patch 10 docs: document K230 FMC/SPI profiles and limitations
```

每个 Patch 应满足：

- 单一职责；
- 独立编译；
- 对应测试与实现放在同一能力边界；
- commit message 明确说明本 Patch 尚未支持的部分。

## 7. 当前下一步

从 P0 开始，不继续扩展新功能：

```text
1. 修正 IMR reset 为 0x3f；
2. 按 has-xip 区分 SPI_CTRLR0 reset；
3. 修正 256 FIFO 和阈值字段；
4. 实现 SSIENR=0 清 FIFO；
5. 收紧 0x108–0x11c，补 0x120–0x134。
```

完成 P0 后再进入 PIO、中断和 DMA，避免继续在错误寄存器 profile 上叠加行为。
