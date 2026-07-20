# K230 SPI/QSPI 10-Patch 实施与 qtest Study

更新时间：2026-07-16

## 文档定位

本文是 K230 SPI/QSPI QEMU 模型的唯一实时实施入口，统一维护：

- 10-Patch 主系列的顺序、边界和完成状态；
- 每个 Patch 对应的 qtest；
- 当前实现差距和最后一次可运行基线；
- 本地调试、回归和提交重拆规则。

寄存器位域、复位值、TRM 5.3/12.3 冲突和 SDK 证据统一查阅：

- [K230 TRM 12.3 SPI 中文学习版（含 5.3 FMC 对照）](k230-trm-12.3-spi-cn.md)

旧的 [K230 SPI/QSPI QEMU 上游 Patch 实施计划](k230-spi-qspi-patch-plan.md)
只保留迁移说明，不再维护第二份状态表。

行为裁决优先级固定为：

```text
K230 TRM + K230 SDK 有效代码/设备树
  > 已完成的交叉裁决
  > QEMU 同类设备实现习惯
  > 当前 k230_dw_ssi.c 原型行为
```

当前原型只能用于定位差距，不能反向定义硬件契约或降低 qtest 预期。

## 1. 支持边界

### 1.1 本系列必须完成

- 三个 K230 SSI 实例及正确 profile。
- 4 KiB MMIO、复位值、writable mask、RO、RC 和 RAZ/WI。
- 256 项、每项最长 32 位的 TX/RX FIFO。
- Standard SPI 的 TR、TO、RO、EEPROM_READ PIO。
- spi0 CS0 的 W25Q256 MTD 后端。
- Dual/Quad QSPI SDR PIO 的指令、地址、mode、dummy 和 RO/TO 数据阶段。
- 九路独立中断及 PLIC 146–172 连线。
- 独立 `HI_SYS.SSI_CTRL @ 0x91585068`。
- 受 `ssi0_xip_en` 门控的 `0xC0000000–0xC7ffffff` XIP window。

### 1.2 本系列明确不实现

- 内部 AXI DMA guest memory 搬运。
- Octal/OPI 数据事务。
- DDR、DTR、RXDS 精确时序。
- concurrent XIP、prefetch、XIP write。
- BootROM 到 U-Boot/Linux 的完整冷启动 functional test。

不实现不等于静默降级：

- DMA 配置寄存器按 SDK 布局保存读回，DONE/AXIE 不得虚假触发。
- Octal/DDR/RXDS 组合必须拒绝事务，不能按 Standard SPI 发送。
- XIP write 保持只读窗口语义。

## 2. 固定实例、地址和裁决

### 2.1 实例

| SDK 实例 | 控制器基址 | `num-cs` | 最大线宽 | `SPI_CTRLR0` 复位 | PLIC 范围 | XIP |
|---|---:|---:|---:|---:|---:|---|
| `spi0` | `0x91584000` | 1 | 8 | `0x28000200` | 146–154 | 有 |
| `spi1` | `0x91582000` | 5 | 4 | `0x04000200` | 155–163 | 无 |
| `spi2` | `0x91583000` | 5 | 4 | `0x04000200` | 164–172 | 无 |

`num-cs` 存在 SDK 内部冲突：K230 U-Boot DTS 为 `1/5/5`，Linux DTS 为
`1/1/1`。本系列按当前启动软件路径和既有 qtest 采用 `1/5/5`，并在上游说明
中保留该证据差异，不能写成 TRM 唯一结论。

### 2.2 其他固定地址

| 对象 | 地址 | 说明 |
|---|---:|---|
| `HI_SYS_CONFIG` | `0x91585000–0x915853ff` | SoC 包装寄存器块 |
| `SSI_CTRL` | `0x91585068` | 不属于任一 SSI 内部 MMIO |
| `DR2` | `spiN_base + 0x068` | 三个控制器的第三个 FIFO 数据口 |
| Flash window | `0xC0000000–0xC7ffffff` | 只属于 spi0，且受 bit 0 门控 |
| PLIC | `0xF00000000` | pending 从 `+0x1000` 开始 |

### 2.3 九路外部 IRQ

| 相对序号 | 外部 IRQ | `spi0` | `spi1` | `spi2` |
|---:|---|---:|---:|---:|
| 0 | TXE | 146 | 155 | 164 |
| 1 | TXO | 147 | 156 | 165 |
| 2 | RXF | 148 | 157 | 166 |
| 3 | RXO | 149 | 158 | 167 |
| 4 | TXU | 150 | 159 | 168 |
| 5 | RXU | 151 | 160 | 169 |
| 6 | MST | 152 | 161 | 170 |
| 7 | DONE | 153 | 162 | 171 |
| 8 | AXIE | 154 | 163 | 172 |

该顺序不是 `RISR` bit 顺序。控制器输出数组、machine 接线和 qtest 必须显式映射。

## 3. 为什么顺序是 SPI → Flash → QSPI → IRQ

K230 U-Boot 的普通 `dw_spi_xfer()` 根据缓冲区方向选择 TR、TO 或 RO；
`dw_spi_exec_op()` 的 Standard read 使用 EEPROM_READ，Dual/Quad read 使用 RO，
Dual/Quad write 使用 TO。
非 IDMA 增强路径还会执行：

```text
disable
  -> 配置 CTRLR0/CTRLR1/SPI_CTRLR0
  -> enable
  -> 写入指令和地址到 TX FIFO
  -> 写 SER 选择 CS
  -> 数据阶段
```

因此：

1. FIFO、四种 TMOD、NDF 和“无 SER 时允许入队、SER 后启动”必须先稳定。
2. Standard Flash 随后验证真实 SDK 数据路径，而不是旧的字节直传捷径。
3. Dual/Quad 只扩展控制前缀，并在 DATA 阶段按 RO/TO 使用既有 RX/TX FIFO。
4. IRQ 最后观察稳定的数据路径事件，避免为迁就早期中断测试反复修改传输泵。

## 4. 10-Patch 主系列总览

| Patch | 单一职责 | 新增 qtest | 累计门禁 |
|---:|---|---:|---:|
| 1 | 寄存器模型、QOM 和稳定设备接口 | 编译门禁 | 0 |
| 2 | 三个实例、地址和 profile | `reg` 8 项 | 8 |
| 3 | Fifo32、Standard PIO 和四种 TMOD | `pio` 10 项 | 18 |
| 4 | spi0 W25Q256、MTD 和 Standard Flash | `flash` 7 项 | 25 |
| 5 | Dual/Quad QSPI SDR PIO 读写 | `qspi` 6 项 | 31 |
| 6 | 控制器内部九路中断 | `irq` 内部 6 项 | 37 |
| 7 | PLIC 146–172 接线 | `irq/plic-*` 5 项 | 42 |
| 8 | HI_SYS `SSI_CTRL` | `hi-sys` 4 项 | 46 |
| 9 | 受控 XIP window | `xip` 5 项 | 51 |
| 10 | 上游英文文档 | 文档门禁 | 51 |

依赖关系固定为：

```text
Patch 1  寄存器/QOM
  -> Patch 2  machine 实例
  -> Patch 3  Standard SPI/FIFO/TMOD
  -> Patch 4  Standard SPI NOR
  -> Patch 5  Dual/Quad QSPI
  -> Patch 6  内部 IRQ
  -> Patch 7  PLIC
  -> Patch 8  HI_SYS SSI_CTRL
  -> Patch 9  XIP
  -> Patch 10 文档
```

## 5. Patch 详细实施卡

### Patch 1：寄存器模型、QOM 与稳定接口

建议标题：

```text
hw/ssi: Add a K230 DesignWare SSI register model
```

文件边界：

- `hw/ssi/k230_dw_ssi.c`
- `include/hw/ssi/k230_dw_ssi.h`
- `hw/ssi/Kconfig`
- `hw/ssi/meson.build`

必须完成：

- 建立 `0x000–0x148` 已裁决寄存器和 4 KiB MMIO。
- 使用 implemented/writable mask，不让保留位进入普通 `regs[]`。
- 建立 profile、`num-cs`、`max-lines` 等稳定属性。
- 建立 SSIBus 和按 `num-cs` 暴露的 CS 接口，但不执行数据事务。
- 固定只读 `IDR=0xa1b2c3d5`、`SSIC_VERSION_ID=0x3130332a`。
- spi0 和 spi1/spi2 使用各自的 `SPI_CTRLR0` 复位 profile。
- DMA 配置寄存器只提供 MMIO 兼容；DONE/AXIE 保持 0。
- `SSIENR` 禁用时恢复静态状态；不提前加入最终 FIFO、IRQ 或 XIP 状态。

本 Patch 禁止：

- 修改 K230 machine。
- 接入 Flash、PLIC 或 XIP window。
- 创建临时聚合 IRQ。
- 使用 Fifo8 或临时字节直传作为后续接口。

完成条件：

- `qemu-system-riscv64` 独立构建通过。
- 无未使用的后续功能状态机。
- 迁移状态只包含本 Patch 已存在的状态。

### Patch 2：三个 SSI 实例和 profile

建议标题：

```text
hw/riscv/k230: Instantiate the K230 SSI controllers
```

文件边界：

- `hw/riscv/k230.c`
- `include/hw/riscv/k230.h`
- `hw/riscv/Kconfig`
- qtest 公共入口、`reg` 测试和 Meson 接入

必须完成：

- SoC 状态按 SDK 逻辑编号组织 `spi0/spi1/spi2`，不按地址排序。
- 映射 `0x91584000/0x91582000/0x91583000`。
- 设置 FMC/QSPI profile、最大线宽和 `num-cs=1/5/5`。
- machine 只做 child、realize 和 MMIO map。
- `0x91585000` 仍由现有占位区域覆盖，不提前实现 `SSI_CTRL`。

qtest：

- R01 `reg/reset-values`
- R02 `reg/profile-reset-values`
- R03 `reg/write-masks`
- R04 `reg/ser-num-cs`
- R05 `reg/read-only-and-razwi`
- R06 `reg/enabled-write-contract`
- R07 `reg/internal-dma-passive`
- R08 `reg/system-reset`

完成条件：

- `reg` 8/8 PASS。
- 没有 PLIC、Flash 或 XIP 映射。
- 三实例状态互相隔离。

### Patch 3：Fifo32 与 Standard SPI

建议标题：

```text
hw/ssi: Implement K230 SSI FIFO and standard PIO transfers
```

必须完成：

- TX/RX 使用容量 256 项的 `Fifo32`，一项代表一个 4–32 bit 帧。
- `DR0–DR35` 访问同一对 FIFO，写入按 `DFS+1` 截断。
- 统一传输泵处理 TR、TO、RO、EEPROM_READ。
- `CTRLR1.NDF+1` 控制 RO/EEPROM_READ 接收帧数。
- `SR/TXFLR/RXFLR` 全部动态计算。
- `SSIENR 1→0` 停止事务、撤销 CS、清 FIFO 和阶段状态。
- `SSIENR=1、SER=0` 时允许先向 TX FIFO 写入指令/地址；`SER` 后续置位时才启动事务。
- `SER` 写入不丢弃已排队帧；多个有效 CS 位必须拒绝，不能静默选择最低位。
- Standard loopback 和外部 SSIBus 共用同一帧/FIFO路径。
- 所有 FIFO 修改集中到少量 helper，为 Patch 6 增加事件锁存提供稳定挂点。

内部实施顺序：

1. Fifo32、DR 别名、frame mask、禁用清理。
2. TR/TO。
3. RO/EEPROM_READ 和 NDF。
4. deferred SER 启动、动态状态和回归。

qtest：

- P01 `pio/dr-aliases`
- P01a `pio/dr-disabled-ignored`
- P02 `pio/fifo-depth-256`
- P03 `pio/dfs-frame-mask`
- P04 `pio/tmod-tr`
- P05 `pio/tmod-to`
- P06 `pio/tmod-ro`
- P07 `pio/tmod-eeprom-read`
- P08 `pio/disable-clears-fifo`
- P09 `pio/dynamic-status`

qtest 边界调整：

- P02 只验证 FIFO 深度、`TXFLR` 和 `SR.TFNF`，不验证 TXO。
- P09 只验证 `TXFLR/RXFLR/SR` 动态变化，不读取 `RISR`。
- TXO 和 TXE 水位中断全部由 Patch 6 的 `irq` 测试负责。

完成条件：

- `reg + pio` 累计 18/18 PASS。
- 没有 IRQ 输出或 PLIC 编号。
- Flash 和 QSPI 后续只能扩展该路径，不能再引入直接字节传输旁路。

### Patch 4：Standard SPI NOR 与 MTD

配套逐步学习文稿：

- [K230 SSI Patch 4：Standard SPI NOR 学习工作簿](k230-spi-patch4-learning-workbook.md)

建议标题：

```text
hw/riscv/k230: Attach SPI NOR flash to K230 spi0
```

必须完成：

- 增加显式 `-machine k230,spi0-flash=<m25p80-model>`；默认不创建 Flash。
- 指定 `w25q256` 时连接到 spi0 CS0。
- `-drive if=mtd` 绑定 Flash backend；指定型号但无镜像时创建擦除态 Flash。
- 控制器不硬编码 Flash 型号和 opcode 状态机。
- Standard read 使用 SDK 的 EEPROM_READ/NDF 序列。
- WREN、page program、sector erase 使用 TO 数据路径。
- 事务结束和 `SSIENR=0` 均正确撤销 CS。
- 后续 XIP 必须经同一 spi0 SSIBus/CS0 访问该从设备，不能直映射 backend。

寄存器写锁采用最小、可证实集合：仅 `CTRLR0/CTRLR1/MWCR/BAUDR/SPI_CTRLR0`
要求在 `SSIENR=0` 时更新。TRM 未明确列为 disabled-only 的 DMA、AXI、采样时序
和 XIP 配置允许在 enabled-idle 状态写入，由 guest 驱动保护活动事务边界；后续实现
IDMA/XIP 时依据专用 active 状态和回归测试再决定是否收紧。

qtest：

- F01 `flash/jedec-id`
- F02 `flash/read-3byte`
- F03 `flash/read-4byte`
- F04 `flash/page-program`
- F05 `flash/sector-erase`
- F06 `flash/cs-restarts-command`
- F07 `flash/enhanced-reset-fallback-read-id`

qtest helper 调整：

- 不再用一个 TR helper 模拟所有 Flash 操作。
- 读命令按“控制帧入 TX FIFO → SER → NDF 数据”执行。
- 写命令按 TO 执行，避免用无意义 RX 数据掩盖 TMOD 错误。

完成条件：

- 累计 25/25 PASS。
- 本 Patch 原则上不修改通用 FIFO/TMOD 架构；若暴露基础缺陷，回修 Patch 3。

### Patch 5：Dual/Quad QSPI

建议标题：

```text
hw/ssi: Implement K230 enhanced QSPI transfers
```

必须完成：

- 事务阶段包含 instruction、address、mode、dummy、data。
- `SPI_FRF` 支持 Standard/Dual/Quad。
- `TRANS_TYPE/INST_L/ADDR_L/WAIT_CYCLES/XIP_MD_BIT_EN/TMOD/NDF` 进入真实事务。
- QSPI read 使用 RO/NDF，RX FIFO 满时暂停并由 DR read 恢复。
- QSPI write 使用 TO/NDF，TX FIFO 空时等待并由后续 DR write 恢复。
- `remaining_frames==0` 是 RO/TO 的完成条件，FIFO 暂时满/空不是完成条件。
- 建立可复用的“增强命令描述/阶段生成”helper，Patch 9 XIP 必须复用。
- Octal、DDR、DTR、RXDS、TT3 和增强 TR/EEPROM_READ 明确拒绝。
- 不模拟 2/4 根物理线逐周期电平。

qtest：

- Q01 `qspi/dual-quad-output-read`
- Q02 `qspi/mode-bits-dummy`
- Q03 `qspi/quad-page-program`
- Q04 `qspi/rx-fifo-resume`
- Q05 `qspi/prefix-atomic`
- Q06 `qspi/unsupported-configs`

完成条件：

- 累计 31/31 PASS。
- Standard Flash 7 项持续通过。
- 没有第二套 FIFO、CS 或 Flash 事务引擎。

### Patch 6：控制器内部九路 IRQ

配套逐步学习文稿：

- [K230 SSI Patch 6：控制器内部 IRQ 学习工作簿](k230-spi-patch6-learning-workbook.md)

建议标题：

```text
hw/ssi: Implement K230 SSI interrupt handling
```

必须完成：

- 暴露 TXE/TXO/RXF/RXO/TXU/RXU/MST/DONE/AXIE 九路输出。
- TXE/RXF 由稳定 FIFO 水位动态产生。
- TXO/RXO/TXU/RXU/MST 使用独立锁存状态。
- `RISR` 不经过 mask，`ISR = RISR & IMR & 0x9bf`。
- IMR 写入和 RC 读取后立即重算输出。
- RC 清除范围符合 TRM；读取 ISR/RISR 不清事件。
- DONE/AXIE 输出存在但在无 DMA 状态机时始终为 0。
- 控制器内部不出现任何 PLIC source ID。

qtest：

- I01 `irq/watermark-mask`
- I02 `irq/rxu-read-clear`
- I03 `irq/txo-read-clear`
- I04 `irq/rxo-read-clear`
- I05 `irq/icr-clear-scope`
- I06 `irq/inactive-causes`

I01 同时验证 TXE 随 `TXFTLR/TXFLR` 动态拉高和撤销，替代原 PIO 中的
`RISR` 断言。

完成条件：

- 累计 37/37 PASS。
- 加 IRQ 不改变 Patch 3–5 的事务结果。

### Patch 7：PLIC 接线

配套逐步学习文稿：

- [K230 SSI Patch 7：PLIC 接线学习工作簿](k230-spi-patch7-learning-workbook.md)

建议标题：

```text
hw/riscv/k230: Connect K230 SSI interrupts to the PLIC
```

必须完成：

- spi0 连接 146–154。
- spi1 连接 155–163。
- spi2 连接 164–172。
- machine 按逻辑实例选择 IRQ base，不按 MMIO 地址推导。
- 27 路逐一连接，不使用 OR gate。
- 本 Patch 不修改控制器事件算法。

qtest：

- I07 `irq/plic-txe-reset-routing`
- I08 `irq/plic-rxu-isolation`
- I09 `irq/plic-rxf-routing`
- I10 `irq/plic-txo-routing`
- I11 `irq/plic-rxo-routing`

完成条件：

- 累计 42/42 PASS。
- 27 路无重复、无越界、无实例错位。

### Patch 8：HI_SYS SSI_CTRL

建议标题：

```text
hw/misc: Model the K230 HI_SYS SSI control register
```

必须完成：

- 独立设备映射 `0x91585000–0x915853ff`。
- `SSI_CTRL` 只位于 `0x91585068`。
- reset、implemented/writable mask 与 SDK 位图一致。
- `ssi0_xip_en` 通过窄接口导出给 spi0。
- mode/sleep 状态从三个 SSI 实例动态获得。
- RXDS edge/delay 只保存读回，不产生线路行为。
- `spiN_base+0x068` 始终是 `DR2`。

qtest：

- H01 `hi-sys/reset-mask`
- H02 `hi-sys/mode-status`
- H03 `hi-sys/sleep-status`
- H04 `hi-sys/dr2-independent`

完成条件：

- 累计 43/43 PASS。
- HI_SYS 不直接访问 machine 全局变量。

### Patch 9：只读 XIP window

建议标题：

```text
hw/ssi: Add the K230 SPI flash XIP window
```

必须完成：

- 只为 spi0 暴露 `0xC0000000–0xC7ffffff`。
- `ssi0_xip_en=0` 时不返回 Flash 数据。
- opcode 来自 `XIP_INCR_INST/XIP_WRAP_INST`。
- 地址长度、mode bits 和 dummy 复用 Patch 5 的增强阶段生成器。
- 支持 1/2/4/8 字节 little-endian MMIO read。
- PIO 与 XIP 共享同一 Flash 和 CS，不保留陈旧事务。
- XIP write 记录 guest error并忽略。

qtest：

- X01 `xip/enable-gate`
- X02 `xip/read-widths`
- X03 `xip/address-width`
- X04 `xip/mode-bits-dummy`
- X05 `xip/pio-coordination`

完成条件：

- 51/51 qtest PASS。
- XIP 不按 opcode 猜测地址长度和 dummy。
- Standard Flash、QSPI 和 IRQ 全量回归不退化。

### Patch 10：上游文档

建议标题：

```text
docs: Document K230 SPI and QSPI support
```

Patch 10 不新增生产行为或 qtest。它只声明前九个 Patch 已经通过的能力，
并明确排除 DMA 搬运、Octal/OPI、DTR/RXDS、concurrent XIP、XIP write
和完整冷启动。本 Study 不继续展开 RST 写作计划。

## 6. qtest 组织与 Patch 归属

### 6.1 最终逻辑模块

所有测试仍链接成一个 `k230-dw-ssi-test`，但按七个能力模块组织：

| 模块 | 用例数 | Patch |
|---|---:|---:|
| `reg` | 8 | 2 |
| `pio` | 10 | 3 |
| `flash` | 7 | 4 |
| `qspi` | 6 | 5 |
| `irq` | 11 | 6–7 |
| `hi-sys` | 4 | 8 |
| `xip` | 5 | 9 |
| 合计 | 51 | 2–9 |

重拆提交时，测试源码目标结构为：

```text
k230-dw-ssi-test.c
k230-dw-ssi-test.h
k230-dw-ssi-test-common.c
k230-dw-ssi-reg-test.c
k230-dw-ssi-pio-test.c
k230-dw-ssi-flash-test.c
k230-dw-ssi-qspi-test.c
k230-dw-ssi-irq-test.c
k230-dw-ssi-plic-test.c
k230-dw-ssi-hi-sys-test.c
k230-dw-ssi-xip-test.c
```

当前 `flash` 文件中的 QSPI 测试、`xip` 文件中的 HI_SYS 测试只做机械迁移，
不改变已经裁决的断言语义。入口注册顺序与 Patch 顺序一致。

### 6.2 测试加入规则

- Patch 2 只加入公共 helper 和 `reg`。
- Patch 3 加入 `pio`。
- Patch 4 加入 `flash`。
- Patch 5 加入 `qspi`。
- Patch 6 注册内部 `irq` I01–I06，Patch 7 再注册独立的 PLIC I07–I11。
- Patch 8 加入 `hi-sys`。
- Patch 9 加入 `xip`。
- 早期 Patch 不携带后续 RED 测试或后续生产实现。
- 每个 Patch 必须运行本组以及之前所有组，不能只跑新增用例。

## 7. 当前状态和差距

### 7.1 最后一次可运行基线

2026-07-14 的最后一次完整逐项结果：

| 指标 | 结果 |
|---|---:|
| 注册用例 | 46 |
| PASS | 16 |
| FAIL | 30 |
| TIMEOUT | 0 |

| 组 | PASS | FAIL | 当时结论 |
|---|---:|---:|---|
| `reg` | 7 | 1 | `spi1/spi2 num-cs=5` 未完成 |
| `pio` | 0 | 9 | Fifo32、TMOD、DFS 未完成 |
| `flash` | 6 | 0 | 旧字节直传原型可以访问 W25Q256 |
| `qspi` | 1 | 2 | 只有配置读回，增强事务未完成 |
| `irq` | 2 | 9 | 简化水位存在，错误锁存和 PLIC 未完成 |
| `hi-sys` | 0 | 4 | `SSI_CTRL` 未实现 |
| `xip` | 0 | 5 | 窗口永久可读，缺少门控和配置驱动 |

该 `16/46` 只作为历史 RED 基线。2026-07-15 已将当前工作树收敛到
Patch 3 生产代码边界：保留寄存器、三实例、Fifo32 和 Standard TMOD，
移除 Flash、增强 QSPI 行为、IRQ、PLIC、HI_SYS/XIP 和 Patch 10 RST。

2026-07-16 为隔离 K230 SDK 对 TMOD 的真实依赖，只补齐 Standard SPI 的
TR/TO，并使用未修改的预构建 U-Boot 和 Linux 做了 Flash 探测实验。当前状态：

```text
qemu-system-riscv64 BUILD PASS
qtest 未执行
TR/TO 已实现
RO/EEPROM_READ 仍未实现
16/46 仍仅为清理前的历史结果，不能代表当前工作树
```

实验期间临时连接的 W25Q256、Kconfig 依赖和派生 DTB 均不属于最终代码；
取证完成后已撤销 QEMU machine 临时接线。现有 qtest 按用户要求保留原样，
本次未修改也未运行。

### 7.2 当前原型的主要差距

| 所属 Patch | 差距 |
|---:|---|
| 2 | 三实例、地址、profile 和 `num-cs=1/5/5` 已保留 |
| 3 | Fifo32、DR 写入门控、deferred SER、Standard TR/TO 已实现；RO/EEPROM_READ 和 NDF 阶段仍未实现 |
| 4 | SPI NOR/MTD 生产代码已移除，等待 Patch 4 重新加入 |
| 5 | 增强 QSPI 行为已移除，只保留 Patch 1 的寄存器访问契约 |
| 6 | IRQ 输出和动态中断行为已移除，只保留寄存器外形 |
| 7 | SSI 到 PLIC 的接线尚未加入 |
| 8 | `SSI_CTRL` 仍由 HI_SYS 占位区域覆盖 |
| 9 | 功能性 XIP MemoryRegion 已移除，`0xC0000000` 恢复为原始 unimplemented 占位 |

## 8. 运行方法

### 8.1 构建 QEMU 和 qtest

不能只链接测试程序；主命令必须同时构建被测 QEMU：

```bash
ninja -C "build" \
    "qemu-system-riscv64" \
    "tests/qtest/k230-dw-ssi-test"
```

### 8.2 列出全部用例

```bash
QTEST_QEMU_BINARY="./build/qemu-system-riscv64" \
    "./build/tests/qtest/k230-dw-ssi-test" -l
```

最终应看到 46 个 `/riscv64/k230-dw-ssi/...` 路径。

### 8.3 跑单项或能力组

```bash
QTEST_QEMU_BINARY="./build/qemu-system-riscv64" \
    "./build/tests/qtest/k230-dw-ssi-test" \
    -p "/riscv64/k230-dw-ssi/pio/fifo-depth-256" --tap
```

```bash
QTEST_QEMU_BINARY="./build/qemu-system-riscv64" \
    "./build/tests/qtest/k230-dw-ssi-test" \
    -p "/riscv64/k230-dw-ssi/pio" --tap
```

### 8.4 Meson 完整入口

```bash
./build/pyvenv/bin/meson test -C "build" \
    --no-rebuild --print-errorlogs \
    qtest-riscv64/k230-dw-ssi-test
```

### 8.5 本地逐项 runner

`run-k230-dw-ssi-tests.sh` 是本地学习和调试工具，不属于上游 10-Patch
功能边界。它会构建 QEMU 和 qtest，然后逐项收集 PASS/FAIL/TIMEOUT。

```bash
bash "tests/qtest/run-k230-dw-ssi-tests.sh"
bash "tests/qtest/run-k230-dw-ssi-tests.sh" "pio"
bash "tests/qtest/run-k230-dw-ssi-tests.sh" "irq/plic"
```

## 9. 分支和提交重拆规则

当前分支可作为重拆载体，已有原型提交用于取回代码和 qtest 意图。最终提交栈遵守：

- 每个 Patch 独立编译。
- 功能实现与对应 qtest 位于同一 Patch。
- 后续实现和后续测试不提前出现在早期 Patch。
- 不用注释、`#if 0` 或永远不调用的 helper 保存超前功能。
- 只允许提前建立后续必需且已经被当前 Patch 使用的稳定结构接口。
- Flash 暴露基础传输缺陷时回修 Patch 3。
- QSPI 不得替换 Standard SPI 引擎。
- IRQ 不得改变传输结果。
- XIP 不得复制 QSPI 阶段生成器。

任何 reset、历史重写、批量删除或重新生成 Patch 的操作，执行前必须单独进行
危险操作确认；本文只定义目标提交边界，不记录 commit/push 结果。

## 10. 完成标准

- [ ] Patch 1–9 均独立构建。
- [ ] 51/51 qtest PASS，且无 TIMEOUT。
- [ ] 每个 Patch 只包含本期实现和本期 qtest。
- [ ] Standard Flash 使用 TO/EEPROM_READ，不依赖旧字节直传。
- [ ] Dual/Quad 复用 Standard FIFO/CS，并为 XIP 提供统一阶段生成器。
- [ ] IRQ 在 SPI/QSPI 稳定后接入，不反向改写数据路径。
- [ ] 三实例地址、profile、CS 和 PLIC 映射与裁决一致。
- [ ] 未支持能力有明确拒绝或静默契约。
- [ ] TRM 只维护证据，Study 是唯一实时计划与状态入口。
- [ ] Patch 10 只声明前九个 Patch 和 qtest 已证明的能力。

最终结论格式：

```text
K230 SPI/QSPI 10-Patch 主系列的 51 项 qtest 全部通过；
支持 Standard/Dual/Quad、九路 IRQ、W25Q256、HI_SYS 和受控 XIP；
不包含内部 DMA 搬运、Octal/OPI、DTR/RXDS 和完整冷启动。
```

## 11. 仅实现 TR/TO 的 K230 SDK Flash 实验

### 11.1 实验目的和边界

本实验回答以下问题：只实现 Standard SPI 的 TR/TO，不实现 RO 和
EEPROM_READ，是否足以让未修改的 K230 SDK U-Boot/Linux 使用 Flash，
以及失败时真正依赖哪个 TMOD。

固定边界：

- 不修改 K230 SDK 源码。
- 不重新编译或覆盖原始 U-Boot、Linux、DTB 和 rootfs 产物。
- 不通过修改 SDK 驱动规避硬件模型缺失。
- qtest 不属于本次实验范围，不修改也不运行。
- QEMU 中的 W25Q256 和 U-Boot 派生 DTB 仅用于取证，实验后撤销或保留在
  `/tmp`，不进入最终 Patch 3。
- 本实验验证“系统已经在 RAM 中运行后访问 Flash”的软件路径，不等同于
  BootROM 冷启动或 CPU XIP 取指验证。

本次最小实现包括：

- `SSIENR=0` 时忽略 DR 写入。
- `SSIENR=1、SER=0` 时允许预填 TX FIFO。
- SER 选择有效 CS 后启动已排队事务。
- TR 消费 TX FIFO，并把每帧线路返回值压入 RX FIFO。
- TO 消费 TX FIFO并发送，但丢弃线路返回值。
- RX FIFO 满时 TR 保留未发送 TX 帧，避免越界和数据丢失。

构建结果：

```bash
ninja -C "build" "qemu-system-riscv64"
git diff --check
```

```text
BUILD PASS
diff check PASS
```

### 11.2 启动资产和方法

实验复用 [K230 完整启动与 IOMUX 验证](uboot-iomux-unimplemented-repro.md)
中已经验证过的 U-Boot/Linux 启动方法。临时使用的公开预构建资产位于：

```text
/tmp/k230-boot-assets-trto/common/u-boot
/tmp/k230-boot-assets-trto/buildroot/direct-boot/Image
/tmp/k230-boot-assets-trto/buildroot/direct-boot/k230.dtb
/tmp/k230-boot-assets-trto/buildroot/direct-boot/rootfs.cpio.gz
```

预构建 U-Boot 的内嵌 DT 原本没有启用 `spi@91584000`。实验从 ELF 提取 DTB，
只在 `/tmp/k230-uboot-spi-test.dtb` 中启用该节点并添加 `flash@0`，然后通过
QEMU loader 覆盖 Guest RAM 中的 DTB：

```bash
-device loader,file=/tmp/k230-uboot-spi-test.dtb,addr=0x807a610,force-raw=on
```

该方法没有修改磁盘上的原始 U-Boot。QEMU machine 中临时把 `w25q256`
连接到 SDK `spi0 @ 0x91584000` 的 CS0；没有恢复 XIP window、IRQ 或 IDMA。

### 11.3 U-Boot 实验：确定阻塞在 EEPROM_READ

U-Boot 启动后，`dm tree` 能看到：

```text
spi       dw_spi          spi@91584000
spi_flash jedec_spi_nor   flash@0
```

执行：

```text
sf probe 0:0
```

首先输出：

```text
jedec_spi_nor flash@0: Software reset enable failed: -524
```

`524` 是 `ENOTSUPP`。该警告来自启动阶段尝试的 Octal DTR 软件复位，
源码明确规定复位失败不是致命错误，随后仍会使用 Standard SPI 读取 JEDEC ID。
真正的阻塞发生在后续 Read-ID。

阻塞现场的 SSI 寄存器：

```text
spi0 base = 0x91584000

CTRLR0 = 0x00000c07  -> TMOD[11:10] = 3，EEPROM_READ
CTRLR1 = 0x00000005  -> NDF + 1 = 6 个接收帧
SSIENR = 0x00000001  -> 控制器已启用
SER    = 0x00000001  -> CS0 已选择
TXFLR  = 0x00000000
RXFLR  = 0x00000000  -> 没有任何接收数据
SR     = 0x00000006  -> TX FIFO empty，RX FIFO empty
```

U-Boot 运行时重定位偏移为 `0x17f57000`。阻塞时：

```text
runtime PC = 0x1ff8e056
link PC    = 0x08037056
```

`addr2line/objdump` 将该地址精确定位到：

```text
dw_reader()
  -> readl(priv->regs + DW_SPI_RXFLR)

return address
  -> poll_transfer()
```

因此阻塞链路已经由控制流和寄存器双重确认：

```text
sf probe
  -> spi_nor_read_id()
  -> Standard Data-IN
  -> dw_spi_exec_op() 选择 TMOD=EEPROM_READ
  -> CTRLR1 请求 6 帧
  -> QEMU 的 TMOD=3 尚未生成接收时钟和 RX 数据
  -> dw_reader() 永久轮询 RXFLR
```

这说明 TO 可以发送纯命令或 Data-OUT，但不能让 `sf probe` 完成。Flash write
即使数据阶段使用 TO，也依赖先 probe Flash，并通常要读取状态寄存器等待 WIP
清除，所以 TR/TO 不能组成可用的 `sf read/write/erase` 链路。

### 11.4 Linux 实验：原始 DT 走增强 RO，而不是 Standard TR

使用未修改的 Linux 5.10.4、原始 `k230.dtb` 和 rootfs 启动，Linux 本身没有
整体阻塞，但 SPI NOR probe 快速失败。关键日志：

```text
dw_spi_mmio 91584000.spi: IRQ index 9 not found
dw_spi_mmio 91584000.spi: Detected FIFO size: 0 bytes
dw_spi_mmio 91584000.spi: registered master spi0
spi spi0.0: setup: ignoring unsupported mode bits 6000
spi_master spi0: CS de-assertion on Tx
spi_master spi0: Retry of enh_mem_op failed
spi-nor: probe of spi0.0 failed with error -2
```

原始 K230 DT 声明 8-line TX/RX，因此 SDK 选择增强 `enh_mem_op`：

```text
Standard Data-IN  -> EEPROM_READ (TMOD=3)
Enhanced Data-IN  -> RO          (TMOD=2)
Data-OUT          -> TO          (TMOD=1)
```

当前 `SPI_FRF != Standard` 会停止传输泵，而且原始 SDK 在总线宽度大于 1 时还会
启用控制器内部 IDMA。因此 Linux 结果说明 TR/TO 对原始 K230 QSPI/OSPI 路径
不充分，但该现场同时存在 FIFO 深度为 0、缺少增强格式、IDMA 和 IRQ 等干扰，
不能把 Linux 的失败单独归因于 TMOD=2。

### 11.5 对启动和系统命令的结论

| 场景 | TR/TO-only 结果 | 第一必要能力 | 其他阻碍 |
|---|---|---|---|
| U-Boot 中 `sf probe/read/write/erase` | Read-ID 阶段阻塞 | Standard EEPROM_READ | 正确 CS、FIFO、SR、Flash 接线 |
| U-Boot 从 Flash 加载 Linux | 无法先完成 probe/read | Standard EEPROM_READ，或启动配置实际使用的增强 read | Flash 镜像和板级启动链 |
| Linux Standard SPI NOR | Data-IN 无法完成 | EEPROM_READ | FIFO/IRQ 和完整 spi-mem 行为 |
| 原始 K230 Linux 8-line Flash | 增强 probe 失败 | RO + Octal/增强阶段 | IDMA、FIFO 深度、可能的 IRQ 完整性 |
| CPU 通过 Flash window 访问或取指 | 未验证，TMOD 实现不会自动提供 XIP | 独立 XIP MemoryRegion | 门控、地址映射、协议状态、Flash backend |
| BootROM 冷启动到 U-Boot/Linux | 未验证 | 取决于 ROM 实际选择的 PIO/QSPI/XIP 路径 | ROM、复位态、镜像布局和完整板级链路 |

本实验支持的严格表述是：

- 对当前 K230 U-Boot Standard SPI NOR 探测路径，TO 是必要但不充分；
  EEPROM_READ 是完成 Read-ID 的必要条件。
- 对原始 K230 Linux 8-line 路径，TR/TO 不充分；读取至少需要 RO 和增强
  SPI/Octal，未修改 SDK 时还需要模型支持其 IDMA 使用方式。
- TMOD=2/3 不是所有平台和所有 Flash 操作的普遍必要条件；它们是当前
  K230 SDK 对应软件分支的明确依赖。
- 本实验不能证明 EEPROM_READ、RO 或 XIP 单项实现后的数学“充分性”，因为
  完整链路还依赖 CS、FIFO、状态、Flash backend、增强配置和启动阶段。

### 11.6 后续实现顺序

若目标是先让 U-Boot Standard Flash 命令工作，最小下一步为：

1. 实现 EEPROM_READ 的命令阶段和 `NDF+1` 自动接收阶段。
2. RX FIFO 满时暂停，Guest 读取 DR 后恢复传输。
3. 完成 BUSY、TXFLR、RXFLR、SR 和事务结束/禁用语义。
4. 在后续 Patch 4 恢复 W25Q256/MTD 接线，验证 JEDEC ID、read、program、erase。

若目标是让未修改的原始 K230 Linux 8-line DT 使用 Flash，还必须扩大当前
10-Patch 的既定支持边界：

1. 实现 RO 及 NDF 自动接收。
2. 实现 Octal/增强 instruction、address、mode、dummy 和 data 阶段。
3. 实现 SDK 实际使用的内部 IDMA/AXI 寄存器和完成语义。
4. 修正 FIFO 深度探测，并补齐 Linux 运行所需 IRQ 行为。

这与本文第 1.2 节“暂不实现内部 DMA、Octal 和完整冷启动”的现有边界冲突；
在不修改 SDK/DT 产物的前提下，必须先明确扩大模型范围，不能把原始 Linux
Flash 支持写入当前 10-Patch 的完成声明。

最后，真正的 Flash/XIP 冷启动必须在上述命令路径稳定后单独验证，不能用
`sf probe` 或 direct boot 成功替代。

## 12. 为什么很多 QEMU SPI 很简单，而 K230 SDK 不能只做 DR 直传

### 12.1 `ssi_transfer()` 不是硬件直通

QEMU 的 `ssi_transfer(bus, value)` 是一个同步、事务级的模拟接口：控制器模型
给模拟从设备发送一帧，函数立即返回这一帧线路上的接收值。它不是把 Guest
请求直接转发给 Host SPI 控制器，也不会自动理解 Guest 的 TMOD、NDF、FIFO、
CS、dummy cycle 或 XIP 配置。

控制器模型仍然负责把 Guest 可见的寄存器协议翻译成若干次
`ssi_transfer()`：

```text
Guest MMIO/FIFO/TMOD/CS
  -> 控制器模型解释事务
  -> 一次或多次 ssi_transfer()
  -> 模拟 SPI NOR/外设
  -> 控制器模型更新 RX FIFO/SR/IRQ
```

因此所谓“直接通过”通常只是：Guest 驱动本身已经把每个线路帧都写进 TX FIFO，
QEMU 控制器只需逐项调用 `ssi_transfer()`；并不是控制器寄存器语义可以省略。

### 12.2 常见简单 SPI 与 K230 DesignWare SSI 的软件契约对比

| 类型 | Guest 如何产生读时钟 | QEMU 模型通常怎么做 | 是否需要 TMOD/NDF 状态机 |
|---|---|---|---|
| PL022、SiFive SPI、Xilinx SPI 等普通全双工 PIO | 软件为命令、地址和每个待读字节持续写 TX/dummy | 每消费一个 TX FIFO 项调用一次 `ssi_transfer()`，返回值进入 RX FIFO | 通常不需要独立 EEPROM_READ |
| 简单 SPI 外设收发 | 软件明确提交等长 TX/RX transfer | DR 写一次，线路交换一次 | TR 模型通常足够 |
| DesignWare SSI Standard spi-mem | 软件只提交命令/地址，并通过 `CTRLR1.NDF` 声明接收长度 | 命令阶段结束后，控制器必须自动产生 `NDF+1` 次 dummy 传输 | 需要 EEPROM_READ/RO 状态机 |
| Flash 专用控制器，如 ASPEED SMC、NPCM FIU | 软件写 command/address/dummy/data 配置，不逐字节写完整事务 | 模型主动合成命令、地址、dummy 和数据阶段 | 同样需要阶段模型，实际并不简单 |
| K230 增强 QSPI/OSPI | SDK 写 `SPI_CTRLR0`、`SPIDR`、`SPIAR`、NDF 和 IDMA 地址 | 模型按配置生成多线命令阶段并完成数据搬运 | 需要 RO/TO、增强阶段和 IDMA 契约 |

QEMU 中可直接看到两类实现差异：

- `hw/ssi/pl022.c`、`hw/ssi/sifive_spi.c`、`hw/ssi/xilinx_spi.c` 都是在 TX FIFO
  非空时逐帧调用 `ssi_transfer()`，每个 TX 项最多对应一个 RX 项。
- `hw/ssi/aspeed_smc.c`、`hw/ssi/npcm7xx_fiu.c` 会根据 Flash 控制寄存器主动
  发送 opcode、地址和 dummy，再读取或写入数据；这些 Flash 控制器并没有
  依靠单次 DR 访问“自动直通”。

所以复杂度的分界不是“K230 与其他 SoC”，而是：

```text
Guest 软件是否把线路上的每一帧都显式写出来
                    versus
Guest 软件是否依赖控制器自动合成事务阶段
```

### 12.3 为什么只在 DR 写入时发送会卡住 K230 U-Boot

普通全双工驱动常见序列：

```text
写 opcode
写 address byte 0
写 address byte 1
写 address byte 2
写 dummy
写 dummy
写 dummy
读取对应 RX FIFO
```

这里每个待读字节都有一个 dummy TX 项。只要模型在每次 DR 写入时调用
`ssi_transfer()`，就能自然得到 RX 数据。

K230 U-Boot 的 Standard `spi-mem` Read-ID 序列则是：

```text
CTRLR0.TMOD = EEPROM_READ
CTRLR1.NDF  = 5
SSIENR      = 1
DR          = 0x9f          # 只写命令
SER         = 1
poll RXFLR                 # 等待 6 个返回字节
```

Guest 没有再写 6 个 dummy 字节，因为真实 DesignWare SSI 会在命令阶段后自动
产生接收时钟。若 QEMU 只实现“DR 写入一次就 `ssi_transfer()` 一次”，最多只能
发出 `0x9f`，之后没有任何 MMIO 写入触发剩余传输，`RXFLR` 必然永久为 0。

这也是本次实验能精确定位到 `TMOD=3` 的原因：问题不是 Flash 从设备不会返回
JEDEC ID，而是控制器模型根本没有产生读取 ID 所需的后续时钟。

### 12.4 哪部分是 DesignWare 共性，哪部分是 K230 SDK 特性

必须分开看待。

DesignWare SSI 共性：

- TR、TO、RO、EEPROM_READ 四种 TMOD。
- RO/EEPROM_READ 根据 NDF 自动接收。
- TX/RX FIFO、BUSY、阈值和 CS 行为。
- Standard `spi-mem` 驱动使用 EEPROM_READ 并非 K230 独有概念。

K230 SDK/集成特性：

- 原始 Linux DT 默认声明 8-line TX/RX，优先进入增强 OSPI 路径。
- SDK 增强驱动使用 `SPI_CTRLR0`、`SPIDR`、`SPIAR` 等扩展寄存器。
- 总线宽度大于 1 时，SDK 会选择控制器内部 IDMA，而不是普通 PIO dummy。
- K230 另有 `HI_SYS.SSI_CTRL`、九路外部 IRQ 和独立 XIP window。
- U-Boot 启动时还会尝试 Octal DTR 软件复位，然后回退到 Standard Read-ID。

因此本次 U-Boot 的 `TMOD=3` 阻塞主要属于 DesignWare spi-mem 契约；原始 Linux
所要求的 Octal、IDMA、HI_SYS 和 XIP 才是 K230 集成让模型明显变复杂的部分。

### 12.5 不需要做周期精确模拟

满足 K230 SDK 不等于模拟真实 SPI 时钟的每个边沿。当前模型仍可保持 KISS，
使用同步、事务级实现。

必须保留的 Guest 可观察契约：

- SSIENR 和 SER 决定事务能否启动以及 CS 边界。
- TX/RX FIFO 的容量、顺序和动态水位。
- TR/TO/RO/EEPROM_READ 对 TX/RX FIFO 的不同影响。
- `CTRLR1.NDF+1` 的接收数量。
- RX FIFO 满时暂停，Guest 读取后继续。
- SR.BUSY、TFE、TFNF、RFNE、RFF 能让轮询正确结束。
- 增强事务的 opcode、address、mode、dummy、data 顺序。

可以简化或暂不实现的物理细节：

- BAUDR 对真实时间流逝的影响。
- SCLK 每个上升沿/下降沿。
- Dual/Quad/Octal 每根数据线的逐 bit 电平。
- DMA 总线逐拍仲裁和精确延迟。
- BUSY 保持多少真实纳秒；同步传输完成后可立即撤销。

推荐的最小模型仍然是：

```text
模式感知的事务级状态机
  + 真实 FIFO/CS/NDF/SR 语义
  + 同步 ssi_transfer() 帧交换
  - 不做周期和引脚级模拟
```

这比“每次 DR 写入直接传一帧”多了一层阶段控制，但远小于周期精确硬件模型，
同时足以满足 U-Boot/Linux 的轮询和 spi-mem 契约。

### 12.6 最终对比结论

很多 QEMU SPI 看起来简单，是因为 Guest 驱动承担了事务展开工作：它主动写
dummy，每个 TX 帧都能直接映射为一次同步 `ssi_transfer()`。

K230 SDK 使用 DesignWare SSI 的硬件卸载能力：Standard read 依赖
EEPROM_READ/NDF 自动产生时钟，增强 read 依赖 RO、SPI_CTRLR0 和 IDMA。
Guest 没有提交那些可以让简单模型继续运行的 dummy DR 写入，所以 QEMU 必须
补上真实硬件原本负责的事务阶段，否则软件轮询没有任何机会自行恢复。

换句话说：

```text
不是 K230 必须做周期精确模拟，
而是 K230 SDK 把更多工作交给了控制器硬件，
QEMU 至少要实现这些可观察的硬件自动行为。
```
