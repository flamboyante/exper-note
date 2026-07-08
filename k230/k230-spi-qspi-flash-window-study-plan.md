# K230 SPI/QSPI/Flash Window 调研与实施计划

记录时间：2026-07-08

本文整理 K230 在 QEMU 中补 `SPI / QSPI / Flash Window` 的学习路线、前期参考、实现边界、测试手段和上游合入策略。

目标不是马上写代码，而是先把问题拆清楚：

```text
K230 的 SPI/QSPI 到底是什么 IP？
QEMU 现有代码有哪些可以参考？
第一版应该做到什么程度才不贪多？
怎么测试它确实有用？
怎么拆成更容易合入上游的补丁？
```

对应 issue：

- `#14 qemu-k230: model qspi`
- `#15 qemu-k230: model flash memory window`
- `#18 qemu-k230: model spi`

建议优先级：

```text
第一阶段：#14 + #15
  K230 QSPI/OPI 寄存器骨架
  SPI NOR read path
  0xC0000000 flash/XIP window 只读访问

第二阶段：#18
  普通 SPI master
  FIFO / CS / polling / 基础 IRQ
```

## 0. 一句话结论

K230 这里不要从“完整 QSPI 电气协议模拟”开始。更稳的上游第一版是：

```text
做一个 K230 DWC_ssi / DW_apb_ssi 风格控制器模型
  -> 复用 QEMU SSI bus
  -> 复用 m25p80 SPI NOR flash 后端
  -> 支持驱动 probe 和基础读 flash
  -> 支持 0xC0000000..0xC8000000 XIP / flash window 只读读取
```

第一版明确不做：

```text
不做 SPI NAND
不做 DMA
不做 program / erase / XIP write
不做完整 1-4-4 / 4-4-4 / 8-8-8 电气时序
不做 DDR/DQS/HyperBus 精确行为
不做复杂性能优化和 prefetch
```

理由是：

- QEMU SSI 框架本身是按 `ssi_transfer(..., byte)` 做逻辑字节传输。
- QEMU 里很多 SPI flash controller 也是把寄存器配置翻译成发给 `m25p80` 的字节流。
- 对训练营和第一版上游 PR 来说，能 probe、能读 JEDEC/SFDP、能从 flash window 读出镜像，比模拟四线物理时序更有价值。

## 1. 本地事实：K230 这三个地址是什么

当前 QEMU K230 machine 里，相关地址来自：

```text
/home/flamboy/qemu-camp-2026-k230/hw/riscv/k230.c
```

关键地址：

```text
QSPI0:         0x91582000, size 0x00001000
QSPI1:         0x91583000, size 0x00001000
SPI/OPI:       0x91584000, size 0x00001000
Flash window:  0xC0000000, size 0x08000000
```

当前状态：

```text
qspi0 -> create_unimplemented_device()
qspi1 -> create_unimplemented_device()
spi   -> create_unimplemented_device()
flash -> create_unimplemented_device()
```

也就是说，现在 guest 访问这些地址时只是占位行为：

- 读一般返回 `0`。
- 写一般被忽略。
- 会记录未实现访问。
- 如果驱动轮询状态位，可能卡死。

## 2. TRM 和 SDK 证据链

K230 TRM 里有两个相关章节：

```text
5.3 Flash Memory Controller
12.3 SPI
```

这两个章节都出现了典型 DWC_ssi / DW_apb_ssi 风格寄存器：

```text
CTRLR0
CTRLR1
SSIENR
SER
BAUDR
TXFTLR / RXFTLR
TXFLR / RXFLR
SR
IMR / ISR / RISR
DRx
SSIC_VERSION_ID
SPI_CTRLR0
XIP_MODE_BITS
XIP_INCR_INST
XIP_WRAP_INST
XIP_CTRL
XIP_SER
```

本地 SDK 也能确认同一件事：

```text
/home/flamboy/k230_sdk/src/big/rt-smart/kernel/bsp/maix3/board/interdrv/spi/drv_spi.c
```

里面定义了 `dw_spi_reg_t`，寄存器布局和 TRM 对得上。

SDK 中三个实例：

```text
SPI0 / OPI:  0x91584000, max_line = 8
SPI1 / QOPI: 0x91582000, max_line = 4
SPI2 / QOPI: 0x91583000, max_line = 4
```

SDK 板级地址：

```text
/home/flamboy/k230_sdk/src/big/rt-smart/kernel/bsp/maix3/board/board.h

SPI_QOPI_BASE_ADDR       0x91582000
SPI_OPI_BASE_ADDR        0x91584000
SPI_XIP_FLASH_BASE_ADDR  0xC0000000
SPI_XIP_FLASH_IO_SIZE    0x08000000
```

所以第一版不建议写成两个完全独立的模型：

```text
k230_fmc.c
k230_spi.c
```

更建议写成一个共享模型：

```text
K230 DWC_ssi compatible controller

属性区分实例：
  max-lines = 4 或 8
  has-xip = true 或 false
  flash-window-base = 0xC0000000
  flash-window-size = 0x08000000
```

## 3. QEMU 里为什么没有 hw/spi

QEMU 里 SPI 相关代码主要叫 `SSI`：

```text
hw/ssi/
include/hw/ssi/ssi.h
```

这里的 SSI 是 synchronous serial interface。很多 SPI master 都在这个目录。

关键接口：

```c
uint32_t ssi_transfer(SSIBus *bus, uint32_t val);
```

理解这个接口很重要：

```text
QEMU SSI 框架不是在模拟真实线路上的 4 根 data line 同时传 bit。
它更像是一个逻辑字节/word 传输接口。

controller 模型负责：
  解析自己的寄存器配置
  决定 opcode/address/dummy/data 怎么发
  调用 ssi_transfer() 把字节送给 slave

slave 模型负责：
  例如 m25p80 解析 SPI NOR 命令状态机
```

所以即使叫 QSPI，第一版也不需要在框架层面实现“四线并行 bit 传输”。对 QEMU 来说，很多时候只是：

```text
guest 配置 Quad/Fast Read
  -> controller 选择合适 opcode、地址长度、dummy cycle
  -> controller 仍按 byte 调用 ssi_transfer()
  -> m25p80 返回 flash 数据
```

## 4. 必读参考代码

### 4.1 `hw/block/m25p80.c`

用途：

```text
SPI NOR flash 后端
```

它已经支持很多常见 SPI NOR 行为：

- JEDEC ID
- SFDP
- READ / FAST READ
- status register
- program / erase
- backing block device

K230 第一版应该复用它，不要自己写 flash 存储和命令状态机。

重点看：

```text
m25p80_transfer8()
m25p80_cs()
m25p80_realize()
m25p80_get_blk()
```

### 4.2 `hw/ssi/npcm7xx_fiu.c`

用途：

```text
直接 flash window + SPI NOR read/write 的很好参考
```

它有一个 direct flash memory read handler。核心思想是：

```text
CPU 读 memory-mapped flash window
  -> controller 选 CS
  -> 通过 SSI 发 read opcode
  -> 发 address
  -> 发 dummy
  -> 连续 ssi_transfer(0) 读回数据
  -> 取消 CS
```

K230 的 `0xC0000000` flash window 第一版可以参考这个结构。

### 4.3 `hw/ssi/aspeed_smc.c`

用途：

```text
Aspeed SPI flash controller，多 CS、多 segment、flash window 映射
```

适合学习：

- flash window 怎么作为 MemoryRegion 暴露。
- 多片 flash 怎么组织。
- controller register 和 direct flash access 怎么分离。

不建议第一版照搬：

- Aspeed 的 SoC 变体很多。
- segment register 和 DMA 逻辑比较重。
- K230 第一版不需要这么复杂。

### 4.4 `hw/ssi/xlnx-versal-ospi.c`

用途：

```text
复杂 OSPI / DAC / indirect / DMA 参考
```

适合学习：

- Direct Access Controller，也就是 memory-mapped flash read。
- indirect read/write 怎么把寄存器命令翻译成 flash 访问。

不建议第一版照搬：

- 代码复杂度高。
- 有 DMA、indirect SRAM、DAC/INDAC 互斥等大量 Versal 特有逻辑。
- K230 第一版只需要其中 direct read 的思路。

### 4.5 `hw/ssi/sifive_spi.c`

用途：

```text
普通 SPI master 的简洁参考
```

适合第二阶段 `#18 model spi` 学：

- FIFO
- CS
- polling 状态
- 简单 `ssi_transfer()`

## 5. 推荐实现边界

### 5.1 第一阶段：QSPI + Flash Window

第一阶段覆盖 `#14` 和 `#15`。

目标：

```text
K230 QSPI/OPI 寄存器不再是 unimplemented。
guest 可以完成最小 probe。
guest 可以通过控制器读 SPI NOR 的 JEDEC/SFDP/数据。
guest 可以从 0xC0000000 flash window 读出 MTD flash image 内容。
```

建议支持寄存器：

```text
0x000 CTRLR0
0x004 CTRLR1
0x008 SSIENR
0x010 SER
0x014 BAUDR
0x018 TXFTLR
0x01c RXFTLR
0x020 TXFLR
0x024 RXFLR
0x028 SR
0x02c IMR
0x030 ISR
0x034 RISR
0x038 TXEICR
0x03c RXOICR
0x040 RXUICR
0x044 MSTICR
0x048 ICR
0x04c DMACR
0x058 IDR
0x05c SSIC_VERSION_ID
0x060..0x0ec DRx
0x0f0 RX_SAMPLE_DELAY
0x0f4 SPI_CTRLR0
0x0fc XIP_MODE_BITS
0x100 XIP_INCR_INST
0x104 XIP_WRAP_INST
0x108 XIP_CTRL
0x10c XIP_SER
0x110 XRXOICR
0x114 XIP_CNT_TIME_OUT
0x118 SPI_CTRLR1
```

第一版可以简化：

```text
多数配置寄存器保存写入值，读回。
只读寄存器返回合理固定值或状态。
FIFO 可以先做足够小的 byte FIFO。
SR 至少表达：
  TX FIFO not full
  TX FIFO empty
  RX FIFO not empty
  busy
```

需要特别注意：

```text
SSIC_VERSION_ID 建议返回 TRM 里的 0x3130332a。
CTRLR0 reset value 建议按 TRM 返回 0x00004007。
SPI_CTRLR0 reset value 要区分 Flash Controller / SPI 章节差异，第一版文档中说明来源。
```

### 5.2 Flash window 第一版怎么定义

K230 flash window：

```text
0xC0000000..0xC8000000
```

建议第一版：

```text
只读。
读访问转成 SPI NOR read 命令。
默认挂 CS0。
默认使用 m25p80 后端。
如果没有 -drive if=mtd，读到 erased value 0xff 或全 0xff backing。
```

窗口读流程：

```text
guest load 0xC0000100
  -> QEMU 命中 K230 flash window MemoryRegion
  -> controller 选择 CS0
  -> 发送 read opcode
  -> 发送地址 0x00000100
  -> 根据配置发送 dummy
  -> 读回 size 个 byte
  -> 拼成 little-endian 返回值
```

这里不需要真实执行 XIP 指令，只需要让 CPU 从该地址读数据时得到 flash 内容。

### 5.3 第二阶段：普通 SPI master

第二阶段覆盖 `#18`。

目标：

```text
0x91584000 普通 SPI/OPI 实例不再是 unimplemented。
支持基础 SPI master 传输。
可以挂 QEMU SSI 外设。
可以通过 polling FIFO 完成字节收发。
```

建议支持：

- `CTRLR0 / CTRLR1 / SSIENR / SER / BAUDR`
- TX/RX FIFO
- `DRx` 读写触发 transfer
- `SR / TXFLR / RXFLR`
- 基础 interrupt mask/status

暂不支持：

- DMA
- full-duplex 边界细节
- slave mode
- Microwire
- precise clock divider timing

## 6. 代码结构建议

建议新增一个共享设备模型，而不是为 qspi/spi 分别写两套。

推荐文件：

```text
include/hw/ssi/k230_dw_ssi.h
hw/ssi/k230_dw_ssi.c
```

构建接入：

```text
hw/ssi/Kconfig
hw/ssi/meson.build
hw/riscv/Kconfig
```

K230 SoC 接入：

```text
include/hw/riscv/k230.h
hw/riscv/k230.c
```

设备属性建议：

```text
max-lines
has-xip
flash-window-size
```

K230 三个实例：

```text
qspi0:
  base = 0x91582000
  max-lines = 4
  has-xip = false 或按实际用途决定

qspi1:
  base = 0x91583000
  max-lines = 4
  has-xip = false

spi/opi:
  base = 0x91584000
  max-lines = 8
  has-xip = true
  flash-window = 0xC0000000..0xC8000000
```

这里有一个需要后续实现前再确认的点：

```text
flash window 最终绑定到 SPI0/OPI 还是 QSPI0？
```

从 SDK 地址看，`SPI_OPI_BASE_ADDR = 0x91584000`，`SPI_XIP_FLASH_BASE_ADDR = 0xC0000000`，所以更倾向于绑定 `0x91584000` 的 OPI 实例。

## 7. 测试手段

### 7.1 qtest：第一优先级

新增测试建议：

```text
tests/qtest/k230-dw-ssi-test.c
```

测试组 1：reset value

```text
读 0x91582000 + CTRLR0
期望 0x00004007

读 SSIENR
期望 0

读 SSIC_VERSION_ID
期望 0x3130332a
```

测试组 2：寄存器读写

```text
写 CTRLR1 / SER / BAUDR / SPI_CTRLR0 / XIP_INCR_INST
再读回
确认普通 R/W 寄存器保存值
确认只读位没有被错误写入
```

测试组 3：FIFO 和状态位

```text
写 DRx
检查 TXFLR / RXFLR / SR
读 DRx
检查 RX FIFO 状态变化
```

测试组 4：JEDEC ID

```text
创建临时 raw flash image
用 -drive file=...,format=raw,if=mtd 启动 k230
通过 controller FIFO 发 0x9f
读回 m25p80 的 JEDEC ID
```

测试组 5：SFDP

```text
发 0x5a + 3-byte address + dummy
读回 SFDP magic
```

测试组 6：flash window read

```text
临时 flash image offset 0x100 写入固定 pattern
guest 读 0xC0000000 + 0x100
确认读回 pattern
覆盖 1/2/4/8 byte read size
```

测试组 7：write 不支持行为

```text
向 flash window 写入
不崩溃
记录 guest error 或忽略
文档说明 read-only
```

### 7.2 裸机 smoke test

写一个很小的 RISC-V 裸机程序：

```text
读 SSIC_VERSION_ID
配置 BAUDR / SER / SSIENR
发 JEDEC ID command
通过 UART 打印读到的 ID
读 0xC0000000 的前几个字节
通过 UART 打印
```

这个测试不是上游必须，但对自己调试很有帮助。

### 7.3 SDK / U-Boot / Linux probe 日志

如果 qtest 通过，再拿真实软件栈做 smoke：

```text
U-Boot 或 SDK 相关驱动 probe 不再卡死在 SPI/QSPI 寄存器访问。
Linux/RT-Smart 驱动访问 qspi/spi 时不再大量 unimplemented。
```

这一步重点不是保证完整启动，而是收集证据：

```text
哪些寄存器还被访问但没实现？
是否有状态位轮询？
是否需要补 writable mask / reset value？
```

## 8. 推荐学习顺序

### 第 1 天：确认事实

读：

```text
K230 TRM 5.3 Flash Memory Controller
K230 TRM 12.3 SPI
/home/flamboy/k230_sdk/src/big/rt-smart/kernel/bsp/maix3/board/interdrv/spi/drv_spi.c
/home/flamboy/k230_sdk/src/big/rt-smart/kernel/bsp/maix3/board/board.h
/home/flamboy/qemu-camp-2026-k230/hw/riscv/k230.c
```

产出：

```text
一张寄存器表
一张 K230 三个 SPI/QSPI/OPI 实例表
一张第一版支持/不支持边界表
```

### 第 2 天：读 QEMU SSI 和 m25p80

读：

```text
include/hw/ssi/ssi.h
hw/ssi/ssi.c
hw/block/m25p80.c
hw/block/m25p80_sfdp.c
```

弄懂：

```text
ssi_create_bus()
ssi_transfer()
ssi_create_peripheral()
m25p80 的 CS 和 transfer 状态机
```

产出：

```text
画出 controller -> SSI bus -> m25p80 的调用链
```

### 第 3 天：读 direct flash window 参考

读：

```text
hw/ssi/npcm7xx_fiu.c
hw/ssi/aspeed_smc.c
hw/ssi/xlnx-versal-ospi.c
```

重点：

```text
MemoryRegionOps 怎么做 flash window
读 window 时怎么发 opcode/address/dummy/data
CS 怎么选中和取消
```

产出：

```text
K230 flash window read 的伪流程图
```

### 第 4 天：写实现设计，不写代码

产出设计文档：

```text
设备类型名
state 结构体字段
寄存器数组和特殊寄存器
MMIO region 数量
flash window region 怎么挂
qtest 列表
patch 拆分
```

### 第 5 天以后：按 TDD 实现

实现顺序：

```text
1. qtest reset/register 失败
2. 最小寄存器模型让它通过
3. qtest FIFO/Jedec 失败
4. 接 SSI bus + m25p80
5. qtest XIP window 失败
6. 加 flash window read-only
7. 文档和 smoke test
```

## 9. Patch 拆分建议

不要一个巨大 commit 把所有东西塞进去。建议拆：

```text
Patch 1: hw/ssi: add K230 DWC_ssi register skeleton
Patch 2: tests/qtest: add K230 DWC_ssi register tests
Patch 3: hw/riscv: wire K230 QSPI/SPI controller instances
Patch 4: hw/ssi: attach m25p80 read path and flash window
Patch 5: tests/qtest: add K230 flash window read tests
Patch 6: docs: document K230 SPI/QSPI limitations
```

如果 reviewer 偏好测试先行，也可以把 qtest patch 放到模型 patch 后面，但本地开发仍建议先写失败测试。

## 10. 上游合入时要诚实说明的限制

文档和 commit message 里建议明确写：

```text
This models the K230 DWC_ssi compatible SPI/QSPI controller enough for
register access and SPI NOR read-only flash window use.
```

限制说明：

```text
Supported:
  - DWC_ssi style register access
  - basic FIFO/polling transfer
  - SPI NOR via m25p80
  - read-only memory-mapped flash window

Not supported:
  - DMA
  - SPI NAND
  - flash program/erase through XIP window
  - accurate multi-lane electrical timing
  - DDR/DQS/HyperBus details
```

这样比声称“完整 QSPI/OSPI”更容易被接受。

## 11. 方案风险

### 风险 1：flash window 绑定哪个实例

SDK 证据倾向于：

```text
0x91584000 SPI0/OPI 负责 0xC0000000 XIP flash window
```

但 issue 名称里有 qspi/flash window，容易误以为 window 属于 `0x91582000`。

实现前要再确认：

```text
TRM address map
SDK board.h
U-Boot platform.h
实际驱动里使用哪个 base 做 XIP
```

### 风险 2：DWC_ssi 寄存器字段很多

不要第一版试图实现所有字段副作用。优先处理：

```text
probe 会读的固定值
driver 会写后读回的配置值
状态轮询需要的 SR/TXFLR/RXFLR
读 flash 必需的 SPI_CTRLR0/XIP opcode/address/dummy 信息
```

### 风险 3：m25p80 对 opcode 支持和 K230 配置不完全匹配

如果 K230 驱动使用的 opcode 是 m25p80 已支持的标准命令，问题不大。

如果遇到特殊 OPI/Octal DDR opcode，第一版可以：

```text
先用配置降级到标准 READ/FAST_READ
或只在 XIP read path 中选择 m25p80 支持的命令
并在文档里说明未实现 octal DDR 精确模式
```

### 风险 4：测试过度依赖真实 SDK

第一版不能只靠“某个 SDK 启动更远了”证明正确。必须有 qtest 固定行为。

推荐证据层级：

```text
qtest 寄存器行为
qtest flash image window read
裸机 smoke
SDK/U-Boot/Linux probe 日志
```

## 12. 最终推荐目标

如果只选一个最稳目标：

```text
K230 SPI0/OPI + QSPI register skeleton
+ m25p80 SPI NOR read path
+ 0xC0000000 flash window read-only
+ qtest
+ docs limitations
```

如果时间更紧，进一步压缩：

```text
只做 #15 flash memory window read-only
  但底层仍以 K230 DWC_ssi/SSI/m25p80 的结构写
  不写 fake flat memory
```

如果时间更充足，再加：

```text
#18 普通 SPI master
基础 IRQ
更多 qtest
SDK probe smoke
```

第一版最重要的是边界清晰：

```text
我们不是在声明完整模拟 K230 QSPI/OSPI。
我们是在给 K230 补一个可验证、可复用、能支撑 SPI NOR read-only/XIP window 的最小控制器模型。
```
