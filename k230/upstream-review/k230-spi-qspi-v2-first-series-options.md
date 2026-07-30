# K230 V2 第一阶段上游 Series 最终方案

> **历史决策记录：禁止作为当前实施依据。** 上游 review 范围复核后，本文的“全部 A”方案已由 [Step 4 Plan Final V1.3](k230-spi-qspi-v2-step4-plan-final-instance-configurationV1.3.md) 取代。V1.3 进一步删除 DMA layout、future capability 和恒低扩展 IRQ，只保留当前消费者需要的 Standard PIO 基线。

首次记录：2026-07-30

最近更新：2026-07-30

文档状态：最终选择已完成，全部采用 A 方案

本文记录 K230 SPI/QSPI V2 第一批上游 patch series 的最终边界。原先的“三个候选方案”和后续 A/B 选项已经完成收敛，废除的方案不再作为实施入口。

本文不替代 [V2 决策记录](k230-spi-qspi-review-v2-decision-notes.md) 和 [Step 4 Plan Final](k230-spi-qspi-v2-step4-plan-final-instance-configuration.md)。完成本文选择后，再据此更新 Step 4 实施顺序和最终 patch series。

## 1. 已确定的总体方向

V2 第一批的裁剪线按“通用 DW SSI IP 基线”划分，而不是按“K230 当前最小启动功能”划分：

> 保留完整的标准 DW SSI 配置骨架、标准寄存器版图、PIO/IRQ 数据路径和 K230 三实例 profile；enhanced SPI、internal DMA 和 XIP 只保留 capability 与寄存器门控，不实现对应数据通路。

可以概括为：**骨架保留，扩展功能后置；砍数据通路，不砍实例差异和 capability 边界。**

第一批必须让 reviewer 能够判断：

- `DwSsiConfig` 是否适合不同 DesignWare SSI 实例；
- capability 关闭时，寄存器、IRQ、资源和数据路径是否真正无行为；
- K230 QSPI0、QSPI1、SPI-OPI 三个 profile 是否只通过配置表达差异；
- 后续 enhanced、IDMA、XIP 是否可以作为独立 series 添加，而不重写基础模型。

## 2. 已确定保留的内容

### 2.1 通用配置骨架

第一批保留集中式 `DwSsiConfig`，至少包含：

- `num-cs`；
- `fifo-depth`；
- `max-lines`；
- component/version/reset 等实例参数；
- `spi-ctrlr0-reset`；
- `has-enhanced-spi`；
- `has-idma`；
- `has-xip`。

三项 capability 的通用默认值均为 `false`。第一批虽然保留 property 和内部 capability 位图，但**任何 capability=true 的配置都必须在 realize 阶段直接拒绝**，不能允许一个宣称具有 enhanced/IDMA/XIP 能力、实际却没有对应数据路径的实例启动。错误信息统一明确为该能力已预留、将在后续 series 支持，例如：

```text
has-enhanced-spi is reserved for a follow-up series
has-idma is reserved for a follow-up series
has-xip is reserved for a follow-up series
```

后续功能 series 在实现数据路径和正路径测试的同一个 patch 中删除对应拒绝逻辑。这样 capability 从“不可启用”变成“可启用”的 diff 是局部且可 review 的。

这里存在一个上游 review 风险：只能设置为 `false` 的 public property 可能被认为是提前暴露接口。如果 reviewer 要求严格 YAGNI，备选做法是第一批仅保留内部全零 capability 位图，把 public property 随对应功能 series 加入；但不能把 `true` 配置静默接受为无功能实例。

`max-lines` 表示实例的物理线数能力，不依赖 `has-enhanced-spi`：

- QSPI0：保留 `max-lines = 4`；
- QSPI1：保留 `max-lines = 4`；
- SPI-OPI：保留 `max-lines = 8`；
- `has-enhanced-spi = false` 时，`max-lines > 1` 仍然允许 realize；
- capability 关闭时，guest 选择非 Standard `SPI_FRF` 不得进入 enhanced 数据路径。

第一批 K230 profile 固定为：

| 配置项 | QSPI0 | QSPI1 | SPI-OPI/FMC |
|---|---:|---:|---:|
| `max-lines` | 4 | 4 | 8 |
| `dma-register-layout` | `internal-axi` | `internal-axi` | `internal-axi` |
| `has-enhanced-spi` | `false` | `false` | `false` |
| `has-idma` | `false` | `false` | `false` |
| `has-xip` | `false` | `false` | `false` |
| `xip-window-size` | `0` | `0` | `0` |
| XIP region | 不创建/不映射 | 不创建/不映射 | 不创建/不映射 |

因此第一批不会把 `0xc0000000` 映射为 SSI XIP aperture。`spi-ctrlr0-reset` 和 `max-lines` 仍按实例 profile 保留，它们描述硬件配置和 reset 契约，不表示扩展引擎已经开放。

### 2.2 Standard SPI 功能

第一批实现：

- Standard single-line SPI；
- 4～32 bit 数据帧；
- TX/RX FIFO；
- Transmit & Receive、Transmit Only、Receive Only、EEPROM Read 四种 TMOD；
- CS 选择与生命周期；
- loopback；
- controller disable/reset；
- FIFO threshold、level 和状态寄存器；
- Standard PIO 数据路径。

### 2.3 基础 IRQ

第一批实现以下基础 IRQ：

- TXE；
- TXO；
- RXF；
- RXO；
- TXU；
- RXU；
- MST。

同时实现：

- raw status；
- interrupt mask；
- masked status；
- read-clear/clear-all；
- TXE/RXF threshold level IRQ；
- K230 到 PLIC 的实例隔离和路由测试。

TXU 属于基础 FIFO IRQ，不依赖 IDMA capability。

### 2.4 K230 三实例

第一批继续实例化：

- QSPI0；
- QSPI1；
- SPI-OPI/FMC。

K230 machine 只负责：

- 选择 profile；
- 设置 properties；
- realize 控制器；
- 映射控制器 MMIO；
- 连接基础 IRQ 到 PLIC；
- 按所选方案决定是否挂接 Standard SPI Flash。

`hw/ssi/dw_ssi.c` 不得包含 K230 地址、K230 类型判断或 HI_SYS 反向依赖。

### 2.5 通用测试

第一批保留独立 `dw-ssi-test` 测试机，不只依赖 K230 machine 测试。至少覆盖：

- property 默认值和 profile readback；
- 配置合法/非法组合；
- Standard PIO 四种 TMOD；
- FIFO 和 IRQ；
- `has-enhanced-spi = false` 的 false-path；
- `has-idma = false` 的 false-path；
- `has-xip = false` 的 false-path；
- 三实例 MMIO、IRQ 和状态隔离；
- 基础迁移状态只包含已实现的 PIO/IRQ 状态。

迁移配置至少加入以下 equality guard，并放在依赖状态之前：

```c
VMSTATE_UINT32_EQUAL(cfg.fifo_depth, DwSsiState),
VMSTATE_UINT32_EQUAL(cfg.capabilities, DwSsiState),
VMSTATE_UINT32_EQUAL(cfg.dma_register_layout, DwSsiState),
```

`fifo-depth` 决定 FIFO 状态解释；capability 决定功能边界；`dma-register-layout` 决定 `0x050/0x054` 以及相关 DMA 配置寄存器的含义。三者任何一项不一致都不能按同一迁移状态解释。

## 3. 已确定后置或废除的第一批内容

以下内容不再作为 V2 第一批候选：

- “只实现 Standard PIO、不实现 IRQ”的最小方案；
- 第一批直接实现 enhanced Dual/Quad/Octal 数据通路；
- 第一批实现 internal DMA guest memory 搬运；
- 第一批实现 XIP transaction、XIP aperture 或 XIP write；
- 第一批加入 HI_SYS 到 DW SSI 的 `xip-enable` GPIO 连接；
- 第一批加入 DDR、RXDS、Octal 数据通路；
- 第一批提前加入 enhanced/IDMA/XIP 的运行时状态和 VMState；
- 把 `0x050/0x054` 同时解释成 `DMATDLR/DMARDLR` 和 `AXIAWLEN/AXIARLEN`；
- 为了完整而一次提交 V1 的全部三千余行功能。

后续建议按以下独立 series 推进：

1. enhanced SPI SDR；
2. internal DMA；
3. XIP；
4. DDR/RXDS/Octal 和其他扩展。

## 4. 第一批寄存器语义基线

| 寄存器类别 | 第一批语义 |
|---|---|
| Standard PIO/IRQ | 实现 reset、write mask、动态 readback 和副作用 |
| IDR/VERSION/公共只读寄存器 | 按 profile 返回只读值 |
| external/internal DMA 配置寄存器 | 由 `dma-register-layout` 决定存在性和地址解释；仅提供寄存器兼容性，不产生 DMA 行为 |
| `SPI_CTRLR0` | 保留 offset 和 profile reset；扩展字段写入不产生行为，`SPI_FRF` 非零写入忽略或清零 |
| enhanced/IDMA/XIP 专用寄存器 | 定义 offset/field；capability 关闭时 RAZ/WI |
| DONE/AXIE 等扩展 IRQ | 根据第 5.4 节选择是否提前保留物理输出，第一批不产生事件 |

“寄存器兼容性”只表示合法字段可以保存和读回，不表示：

- 已产生 external DMA request；
- 已访问 guest memory；
- 已启动 internal DMA；
- 已产生 DONE/AXIE；
- 已完成对应的数据通路。

## 5. 最终选择

项目 owner 最终确认全部采用 A 选项：`5.1-A，5.2-A，5.3-A，5.4-A，5.5-A，5.6-A`。

### 5.1 寄存器建模方式

- [x] **5.1-A：继续使用 `REG32/FIELD` + 集中 read/write helper。** 保留完整 offset/field 目录，用少量辅助函数统一 reset、mask、layout 和 RAZ/WI；第一批不引入完整 `RegisterAccessInfo` 重构。

### 5.2 `0x050/0x054` DMA 寄存器布局（已确定）

K230 的 `SSIC_HAS_DMA = 2` 使用：

```text
0x050 -> AXIAWLEN
0x054 -> AXIARLEN
```

普通 external DMA 变体使用：

```text
0x050 -> DMATDLR
0x054 -> DMARDLR
```

二者是互斥布局，不能在同一个 profile 中同时出现。

- [x] **5.2-A：增加显式 `dma-register-layout` 配置。** 定义 `none/external/internal-axi` 布局。layout 决定寄存器版图：

- `none`：DMA 配置寄存器不存在，相关地址 RAZ/WI；
- `external`：`0x050/0x054` 按 `DMATDLR/DMARDLR` 解释；
- `internal-axi`：`0x050/0x054` 按 `AXIAWLEN/AXIARLEN` 解释，K230 三实例使用此布局。

layout 还决定 `DMACR` 字段视图以及 `SPIDR/SPIAR/AXIAR0/1` 等 internal-AXI 配置寄存器是否存在。`has-idma` 不参与寄存器存在性判断，只控制 internal DMA 引擎触发、guest memory 访问和 DONE/AXIE 事件。

无论选择哪一项，第一批都不实现 DMA request、DMA engine 或 guest memory 搬运。

### 5.3 `SPI_CTRLR0` 的第一批行为（已确定）

K230 已知实例 reset 存在差异：普通 SPI/QSPI 为 `0x04000200`，FMC/OPI 为 `0x28000200`。

- [x] **5.3-A：保留 offset 和 profile reset。** guest 可以读到 `0x04000200` 与 `0x28000200` 的实例差异。`has-enhanced-spi=false` 时：

- `CTRLR0.SPI_FRF` 非零写入忽略或清零，读回保持 Standard；
- `SPI_CTRLR0` 的扩展写入不进入 enhanced/DDR/XIP 数据路径；
- 不把整个 `SPI_CTRLR0` 机械归入 absent capability 后 RAZ/WI；
- `DDR_DRIVE_EDGE` 等纯扩展寄存器仍可 RAZ/WI。

### 5.4 DONE/AXIE 物理 IRQ 输出

- [x] **5.4-A：第一批保留全部 9 路 IRQ 输出。** 七路基础 IRQ 正常工作；DONE/AXIE 输出恒低，IMR/ISR/RISR 对应位无效；K230 PLIC 路由一次固定下来，IDMA series 只增加事件产生逻辑。

XRXO/SPITE 不在第一批暴露独立物理输出，其寄存器状态位保持无效。

### 5.5 K230 Standard SPI Flash 挂接

- [x] **5.5-A：第一批包含 Flash 挂接。** 保留 `spi-flash` machine property，挂接 M25P80-compatible device，只验证 Standard 1-1-1 PIO 访问；不宣称 Dual/Quad、IDMA 或 XIP 启动。

Flash 挂接作为独立 patch，不能混入通用 DW SSI 模型 patch。

### 5.6 Patch series 拆分粒度

- [x] **5.6-A：6 个功能 patch。** 每个 patch 职责更单一，测试跟随首次实现该行为的 patch。

推荐的 6-patch 结构：

1. `hw/ssi: Add a configurable DesignWare SSI register model`
2. `hw/ssi: Implement FIFO-backed Standard SPI PIO`
3. `hw/ssi: Add DesignWare SSI standard interrupt support`
4. `hw/riscv/k230: Instantiate DesignWare SSI controllers`
5. `hw/riscv/k230: Route SSI interrupts to the PLIC`
6. `hw/riscv/k230: Attach a standard SPI flash to the K230 SSI`，加入 Standard 1-1-1 Flash integration 测试。

测试不得全部堆到最后一个 patch：Patch 1～5 必须各自带上首次覆盖对应行为的测试；第 6 个测试 patch 只允许补充跨模块集成场景。

## 6. 不同选择对应的预计规模

以下估算**包含通用/K230 qtest、测试 machine 和 Meson 注册**，不包含 cover letter。按测试跟随功能 patch 的拆分方式，测试代码预计占总新增行数的 30%～40%。

| 最终组合 | 预计新增代码（含 qtest） | 特点 |
|---|---:|---|
| 全部 A | 1850～2250 行 | REG32/helper、显式 DMA layout、9 路 IRQ、Standard Flash、6 patches |

该估算仍应明显小于 V1 的三千余行完整功能 series。

## 7. 最终组合摘要

```text
5.1-A  REG32/FIELD + 集中 helper
5.2-A  显式 DMA register-layout
5.3-A  保留 SPI_CTRLR0 profile reset
5.4-A  保留 9 路 IRQ，DONE/AXIE 恒低
5.5-A  第一批挂接 Standard 1-1-1 Flash
5.6-A  拆成 6 个功能 patch
```

最终组合预计约 1850～2250 行（含 qtest）。它提供完整的标准 DW SSI 配置和寄存器边界、PIO/IRQ 闭环、K230 三实例证明以及真实 Standard Flash consumer，但不会把 enhanced、IDMA、XIP 数据路径带回第一批。

## 8. Cover letter 必须主动说明的边界

无论最终选择哪组方案，cover letter 都应明确说明：

> This series introduces a configurable DesignWare SSI model covering the standard PIO and interrupt baseline used by the K230 SSI instances. Enhanced SPI transfers, the internal DMA engine, and the XIP aperture are reserved for follow-up series; attempts to enable those capabilities are rejected at realize time in this version. Register layout is selected independently from engine capabilities, so layout-visible DMA configuration registers may provide storage/readback without triggering DMA requests, guest-memory accesses, or DONE/AXIE events.

还应单独说明 `0x050/0x054` 是随综合配置变化的互斥 DMA register layout，避免 reviewer 把 K230 的 `AXIAWLEN/AXIARLEN` 和普通 external DMA 的 `DMATDLR/DMARDLR` 当成同一实例应同时存在的寄存器。

后续 IDMA series 必须继承 Step 4.0 已确认的隔离约束：普通 enhanced/IDMA 的 1-4-4 路径只执行 instruction → address → dummy → data，不读取 `XIP_MODE_BITS`。`XIP_MODE_BITS` 继续属于 XIP 专用组，不得因为 IDMA 加入而改成 shared 寄存器；IDMA series 应保留 SDK 风格 `0xeb` 回归，防止旧的 mode-byte 特判重新出现。

---

选择完成后，需要同步更新：

- [Step 4 Plan Final](k230-spi-qspi-v2-step4-plan-final-instance-configuration.md)；
- [V2 implementation plan](k230-spi-qspi-v2-implementation-plan.md)；
- 最终 cover letter 和 patch 拆分表。
