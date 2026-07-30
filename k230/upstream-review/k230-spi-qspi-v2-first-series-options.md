# K230 V2 第一阶段上游 Series 范围与待决策项

首次记录：2026-07-30

最近更新：2026-07-30

文档状态：基础方向已确定，等待项目 owner 选择剩余实现策略

本文用于确定 K230 SPI/QSPI V2 第一批上游 patch series 的最终边界。原先的“三个候选方案”已经根据后续讨论收敛：不再比较“最小 PIO”“PIO + IRQ”“完整寄存器目录 + PIO/IRQ”三个互斥方案，而是先固定一条共同基线，再只保留仍有实际取舍的决策项。

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

三项 capability 的通用默认值均为 `false`。第一批可以配置和验证这些 capability，但不会因为结构中存在 capability 就提前加入对应运行时状态。

`max-lines` 表示实例的物理线数能力，不依赖 `has-enhanced-spi`：

- QSPI0：保留 `max-lines = 4`；
- QSPI1：保留 `max-lines = 4`；
- SPI-OPI：保留 `max-lines = 8`；
- `has-enhanced-spi = false` 时，`max-lines > 1` 仍然允许 realize；
- capability 关闭时，guest 选择非 Standard `SPI_FRF` 不得进入 enhanced 数据路径。

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
| external/internal DMA 配置寄存器 | 根据第 5.2 节选择，仅提供寄存器兼容性，不产生 DMA 行为 |
| `SPI_CTRLR0` | 根据第 5.3 节选择 reset/readback 和写入门控 |
| enhanced/IDMA/XIP 专用寄存器 | 定义 offset/field；capability 关闭时 RAZ/WI |
| DONE/AXIE 等扩展 IRQ | 根据第 5.4 节选择是否提前保留物理输出，第一批不产生事件 |

“寄存器兼容性”只表示合法字段可以保存和读回，不表示：

- 已产生 external DMA request；
- 已访问 guest memory；
- 已启动 internal DMA；
- 已产生 DONE/AXIE；
- 已完成对应的数据通路。

## 5. 等待项目 owner 选择的决策项

每个决策项只选择一个选项。可以直接回复类似：`5.1-A，5.2-A，5.3-A，5.4-A，5.5-B，5.6-A`。

### 5.1 寄存器建模方式

- [ ] **5.1-A（推荐）：继续使用 `REG32/FIELD` + 集中 read/write helper。** 保留完整 offset/field 目录，用少量辅助函数统一 reset、mask 和 RAZ/WI；不在第一批引入完整 `RegisterAccessInfo` 重构。预计代码量较小，接近现有模型，reviewer 更容易对照 V1。
- [ ] **5.1-B：采用 DW I2C 风格 `RegisterAccessInfo`。** 使用 descriptor table、unsupported callback 和 `register_init_block32()` 管理寄存器。长期结构更集中，但第一批会额外增加一次寄存器框架 review。

选择影响：5.1-B 预计比 5.1-A 增加约 200～400 行结构代码和测试调整，也可能通过删除大 switch 抵消一部分增量。

### 5.2 `0x050/0x054` DMA 寄存器布局

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

- [ ] **5.2-A（推荐）：第一批增加显式 DMA register-layout 配置。** 定义 `none/external/internal-axi` 布局；external 和 internal-axi 均只实现寄存器存储，不产生 DMA 行为；K230 三实例选择 `internal-axi`。优点是通用 IP 边界最清楚，代价是增加 profile、字段 mask 和 false-path 测试。
- [ ] **5.2-B：第一批只实现 K230 的 internal-axi 布局。** `0x050/0x054` 只按 `AXIAWLEN/AXIARLEN` 解释，external DMA layout 后续在有第二个用户时加入。代码最小，符合 YAGNI，但第一批的“通用标准 DW SSI”范围需要在 cover letter 中限定为 K230 已综合配置。
- [ ] **5.2-C：第一批让 `0x050/0x054` 全部 RAZ/WI。** 不保存 `AXIAWLEN/AXIARLEN`，等 IDMA series 再实现。最小，但弱化 K230 profile 的寄存器完整性，也不能覆盖 SDK 对这两个寄存器的访问。

无论选择哪一项，第一批都不实现 DMA request、DMA engine 或 guest memory 搬运。

### 5.3 `SPI_CTRLR0` 的第一批行为

K230 已知实例 reset 存在差异：普通 SPI/QSPI 为 `0x04000200`，FMC/OPI 为 `0x28000200`。

- [ ] **5.3-A（推荐）：保留 offset 和 profile reset，扩展写入受 capability 门控。** guest 可以读到实例 reset 差异；`has-enhanced-spi = false` 时，写入 enhanced/DDR/XIP 字段不产生行为，并按定义忽略或清零不可写字段。
- [ ] **5.3-B：`has-enhanced-spi = false` 时整个寄存器 RAZ/WI。** 行为最简单，但三个 profile 的 `spi-ctrlr0-reset` 在第一批失去可见意义，后续启用 enhanced 时需要改变 guest-visible readback。

5.3-A 更符合“保留实例差异、后置数据通路”的原则。

### 5.4 DONE/AXIE 物理 IRQ 输出

- [ ] **5.4-A（推荐）：第一批保留全部 9 路 IRQ 输出。** 七路基础 IRQ 正常工作；DONE/AXIE 输出恒低，IMR/ISR/RISR 对应位无效；K230 PLIC 路由一次固定下来，IDMA series 只增加事件产生逻辑。
- [ ] **5.4-B：第一批只暴露七路基础 IRQ。** device interface 更精确，IDMA series 再增加 DONE/AXIE 输出并修改 K230 PLIC 路由。第一批更小，但后续会改变 sysbus IRQ 拓扑。

XRXO/SPITE 不在第一批暴露独立物理输出，其寄存器状态位保持无效。

### 5.5 K230 Standard SPI Flash 挂接

- [ ] **5.5-A：第一批包含 Flash 挂接。** 保留 `spi-flash` machine property，挂接 M25P80 compatible device，只验证 Standard 1-1-1 PIO 访问；不宣称 Dual/Quad、IDMA 或 XIP 启动。优点是提供真实设备集成证明，代价是增加 machine property 和 Flash 测试 review。
- [ ] **5.5-B（推荐）：第一批不包含 Flash 挂接。** 使用通用 SSI test device 验证 PIO，在后续 enhanced series 或独立 K230 Flash integration series 中加入 NOR。第一批边界最集中，但缺少真实 NOR 使用场景。

如果选择 5.5-A，Flash 挂接应作为独立 patch，不能混入通用 DW SSI 模型 patch。

### 5.6 Patch series 拆分粒度

- [ ] **5.6-A（推荐）：6 个功能 patch。** 每个 patch 职责更单一，测试跟随首次实现该行为的 patch。
- [ ] **5.6-B：5 个功能 patch。** 合并一部分 K230 集成或测试 patch，series 更短，但单个 patch 更大。

推荐的 6-patch 结构：

1. `hw/ssi: Add a configurable DesignWare SSI register model`
2. `hw/ssi: Implement FIFO-backed Standard SPI PIO`
3. `hw/ssi: Add DesignWare SSI standard interrupt support`
4. `hw/riscv/k230: Instantiate DesignWare SSI controllers`
5. `hw/riscv/k230: Route SSI interrupts to the PLIC`
6. 根据 5.5 的选择，加入 Standard Flash integration，或把通用/K230 qtest 独立整理为最后一个 patch。

测试不得全部堆到最后一个 patch：Patch 1～5 必须各自带上首次覆盖对应行为的测试；第 6 个测试 patch 只允许补充跨模块集成场景。

## 6. 不同选择对应的预计规模

| 组合 | 预计新增代码 | 特点 |
|---|---:|---|
| 5.1-A + 5.2-B + 5.4-B + 5.5-B | 1500～1800 行 | 最小的标准 IP 基线 |
| 5.1-A + 5.2-A + 5.4-A + 5.5-B | 1700～2100 行 | 通用边界和后续稳定性较平衡 |
| 5.1-B + 5.2-A + 5.4-A + 5.5-A | 2000～2500 行 | 寄存器框架、DMA layout 和真实 Flash 集成最完整 |

这些数字不包含 cover letter。无论选择哪种组合，第一批都应明显小于 V1 的三千余行完整功能 series。

## 7. 当前推荐组合

如果优先考虑“让 reviewer 能确认通用边界，同时控制第一批体积”，当前推荐：

```text
5.1-A  REG32/FIELD + 集中 helper
5.2-A  显式 DMA register-layout
5.3-A  保留 SPI_CTRLR0 profile reset
5.4-A  保留 9 路 IRQ，DONE/AXIE 恒低
5.5-B  第一批不挂接 Flash
5.6-A  拆成 6 个功能 patch
```

这套组合预计约 1700～2100 行。它提供完整的标准 DW SSI 配置和寄存器边界、PIO/IRQ 闭环以及 K230 三实例证明，但不会把 enhanced、IDMA、XIP 数据路径带回第一批。

## 8. Cover letter 必须主动说明的边界

无论最终选择哪组方案，cover letter 都应明确说明：

> This series introduces a configurable DesignWare SSI model covering the standard PIO and interrupt baseline used by the K230 SSI instances. Enhanced SPI transfers, the internal DMA engine, and the XIP aperture are represented by disabled capability boundaries and will be added in follow-up series. Registers belonging to disabled capabilities are either read-as-zero/write-ignored or expose only profile-defined reset values; they do not provide the corresponding data-path functionality.

还应单独说明 `0x050/0x054` 是随综合配置变化的互斥 DMA register layout，避免 reviewer 把 K230 的 `AXIAWLEN/AXIARLEN` 和普通 external DMA 的 `DMATDLR/DMARDLR` 当成同一实例应同时存在的寄存器。

---

选择完成后，需要同步更新：

- [Step 4 Plan Final](k230-spi-qspi-v2-step4-plan-final-instance-configuration.md)；
- [V2 implementation plan](k230-spi-qspi-v2-implementation-plan.md)；
- 最终 cover letter 和 patch 拆分表。
