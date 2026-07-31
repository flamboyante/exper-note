# K230 V2 第四步 Plan Final V1.3：实例配置与第一批功能边界

首次记录：2026-07-29

V1.3 修订：2026-07-30（未实现扩展寄存器采用固定 unsupported 语义，功能配置随真实消费者引入；external DMA 不进入当前路线）

适用代码检查点：`qemu-camp-2026-k230/` 分支 `k230-V2-patch-spi`，commit `c689ac865f`

文档状态：Step 4.0 已完成源码实施和 12 项 K230 SSI qtest；Step 4.1 至 Step 4.4 仍是待实施计划

本文是 [V2 实施路线](k230-spi-qspi-v2-implementation-plan.md) 第四步“引入实例配置”的唯一执行计划。Plan A、Plan B、Plan C、原 Plan Final 及其 planning handoff 只保留为历史推演材料，不再作为实施入口。架构边界以 [V2 决策记录](k230-spi-qspi-review-v2-decision-notes.md) 为准：保留单一通用 `TYPE_DW_SSI` / `DwSsiState`，K230 machine 通过 properties 配置三个实例，不增加 `TYPE_K230_DW_SSI` wrapper。

> **上游投稿边界说明（2026-07-30）**：第一批提交通用 DW SSI 的标准寄存器版图、Standard SPI PIO、基础 IRQ、K230 三实例与 PLIC 集成，以及独立的 Standard 1-1-1 SPI NOR 挂接。K230 的 `0x04c/0x050/0x054` 按 internal-AXI `DMACR/AXIAWLEN/AXIARLEN` 版图识别，第一批固定 read 0、write ignore；不定义或实现 external `DMATDLR/DMARDLR` 语义。enhanced SPI、IDMA engine、HI_SYS 和 XIP 分批后送。第一批不增加 `has-enhanced-spi`、`has-idma`、`has-xip`、`xip-window-size` 或内部 capability 位图。

## 🧭 章节归属图例

| 标识 | 含义 | 是否进入第一批上游 series |
|---|---|---|
| 🟦 第一批 | 第一批五个 patch 的设计、实施或验收内容 | 是 |
| ✅ 已完成前置 | 为收敛第一批边界已在本地完成的纠错；最终第一批不会单独携带该 patch | 否，作为正确基线吸收 |
| 🟨 后续 | enhanced、internal IDMA、HI_SYS/XIP 等 follow-up series 的边界或防回归约束 | 否 |
| ⬜ 背景 | 当前状态、证据、编号映射、主线先例和参考入口 | 不直接形成 patch |
| 🟪 混合 | 同一章同时包含第一批正文和后续边界；以二级标题标识为准 | 按子章节判断 |

本文主体是**第一批上游 series 的完整执行计划**，不是 enhanced、IDMA、XIP 全功能实现计划。后续功能只在本文中记录接口边界、证据和防回归要求，具体实现由独立 follow-up series 计划承接。

---

## ⬜ 摘要与文档导航

Step 4 先修正当前模型把普通 enhanced/IDMA 事务错误连接到 XIP mode bits 的问题，再将 K230 固定参数改为 realize 前确定的实例配置。第一批保留完整控制器地址版图和 Standard PIO/IRQ 行为；未实现寄存器使用固定 unsupported 或 RAZ/WI 语义，不用额外配置接口表达“关闭”。

实施分为五个可独立验证的小目标，与总实施路线和最终五个 patch 的职责顺序一致：

1. Step 4.0：普通 enhanced/IDMA 只执行 instruction → address → dummy → data，XIP mode 只留在真正的 XIP transaction；
2. Step 4.1：建立 `DwSsiConfig`、配置校验、动态 FIFO、四种 Standard TMOD、基础 VMState、未实现扩展的固定语义和通用 qtest；
3. Step 4.2：实现 TXE/TXO/RXF/RXO/TXU/RXU/MST 七路基础 IRQ 及通用 IRQ qtest；
4. Step 4.3：在 K230 machine 中应用 QSPI0、QSPI1、SPI-OPI/FMC 三个 profile，映射 region 0、连接七路 PLIC，并删除第一批 XIP 接口；
5. Step 4.4：挂接 Standard 1-1-1 SPI NOR，并完成构建、通用/K230 qtest、公共头文件、迁移状态和未来 patch 归属检查。

本步不实现 enhanced、DDR、RXDS、Octal、XIP 或 internal DMA engine。external DMA 不属于当前 V2 实施路线，出现新的真实消费者和寄存器版图证据后再单独评估。第一批数据路径只执行 Standard single-line SPI；后续 series 在引入真实功能时扩展相应配置和运行状态。

---

## ⬜ 1. 当前状态分析

### 1.1 代码与测试检查点

CodeGraph 首先定位了 `k230_soc_realize()` 的实例配置、realize、IRQ 和 MMIO 映射路径；随后以当前磁盘源码逐项核对。当前 QEMU 源码工作树干净，关键状态为：

| 项目 | 当前状态 |
|---|---|
| 通用类型 | `TYPE_DW_SSI = "designware-ssi"`，状态类型 `DwSsiState` |
| 已有 properties | `num-cs`、`max-lines` |
| FIFO | TX/RX 均在 `instance_init` 固定创建为 256 帧 |
| 控制器 MMIO | 固定创建 0x1000 字节 region |
| XIP MMIO | 所有实例均固定创建 128 MiB 的第二个 region |
| K230 映射 | 三实例映射控制器 region；仅 `dw_ssi[2]` 的 region 1 映射到 `0xc0000000` |
| reset | `IMR`、AXI burst、IDR、VERSION、`SPI_CTRLR0` 仍由通用模型硬编码 |
| 可选功能配置 | enhanced SPI、IDMA、XIP 均无独立配置入口 |
| VMState | 保存完整寄存器数组、FIFO、enhanced、IDMA、XIP enable 等运行时状态 |
| qtest | `k230-dw-ssi-test` 共 12 项；Step 4.0 新增 enhanced/XIP 隔离和 IDMA 1-4-4 回归，仍缺通用 Standard PIO、unsupported 寄存器和实例配置覆盖 |

### 1.2 当前硬编码点清单

| 编号 | 当前位置 | 硬编码或耦合 | Step 4 动作 |
|---|---|---|---|
| H1 | `include/hw/ssi/dw_ssi.h:28` | `DW_SSI_XIP_WINDOW_SIZE = 0x08000000` | 第一批不创建 XIP window；`xip-window-size` property 随 XIP series 引入 |
| H2 | `include/hw/ssi/dw_ssi.h:94-95` | `num_cs`、`max_lines` 分散在运行时状态中 | `num_cs` 收入 `DwSsiConfig`；`max_lines` 随 enhanced series 引入 |
| H3 | `hw/ssi/dw_ssi.c:27` | FIFO 固定 256 | 改为 `fifo-depth` |
| H4 | `hw/ssi/dw_ssi.c:31-37` | `IMR`、IDR、VERSION、AXI burst 固定 | `IMR` 从实例配置读取；IDR/VERSION 保留通用常量；AXI reset 随 IDMA series 引入 |
| H5 | `hw/ssi/dw_ssi.c:33-34` | 仅按 SPI/FMC 固定两个 `SPI_CTRLR0` reset | 第一批固定为 unsupported/RAZ/WI；实例 reset 随 enhanced series 引入 |
| H6 | `hw/ssi/dw_ssi.c:425-430` | `SR.TFNF/RFF` 与 256 比较 | 改为 `cfg.fifo_depth` |
| H7 | 原 `hw/ssi/dw_ssi.c:537-584`、`949-955`、`1021-1043` | 普通 enhanced/IDMA 曾错误读取 `XIP_MODE_BITS`，PIO 曾包含 XIP mode phase | Step 4.0 已完成：普通事务只保留 instruction/address/dummy/data |
| H8 | `hw/ssi/dw_ssi.c:813-984` | DMA 寄存器和 IDMA engine 被同一逻辑隐式绑定 | `0x04c/0x050/0x054` 只按 K230 internal-AXI 版图保留并固定 unsupported；internal DMA engine、guest memory 和 DONE/AXIE 随 IDMA series 引入；external DMA 不进入当前路线；TXU 保持基础 FIFO IRQ |
| H9 | `hw/ssi/dw_ssi.c:1201-1535` | read/write switch 混合 Standard 与扩展功能 | Standard 寄存器保留真实语义；未实现扩展寄存器走集中 unsupported/RAZ/WI helper |
| H10 | `hw/ssi/dw_ssi.c:1578-1585` | reset 使用 K230 值，并用 `max_lines == 8` 推断 FMC | 第一批只从 `cfg.imr_reset` 读取实例 reset，删除线宽推断 |
| H11 | `hw/ssi/dw_ssi.c:1677-1689` | 所有实例创建 XIP region 和 256 深度 FIFO | 第一批不创建 XIP region；FIFO 按 `fifo-depth` 在 `realize()` 创建 |
| H12 | `hw/riscv/k230.c:246-251` | machine 只设置 CS 和线宽 | 为三个实例显式设置 `num-cs`、`fifo-depth`、`imr-reset` |
| H13 | `hw/riscv/k230.c:293-294` | 无条件映射 `dw_ssi[2]` region 1 | 第一批不映射 region 1；随 XIP series 增加可选映射 |
| H14 | `tests/qtest/k230-dw-ssi-test.c:492-528` | 三实例只验证 `SPI_CTRLR0`/SER，IDR/VERSION/IMR 只抽查 spi0 | 扩充为三实例 Standard register、FIFO 和基础 IRQ profile |

### 1.3 当前资源生命周期

```text
instance_init
  ├── 创建 SSI bus
  ├── 创建控制器 MMIO region 0
  ├── 无条件创建 128 MiB XIP region 1
  ├── 注册 9 路 IRQ
  ├── 注册 xip-enable GPIO input
  └── 无条件创建 256 深度 TX/RX FIFO

realize
  ├── 校验 num-cs
  ├── 校验 max-lines
  └── 按 num-cs 创建 CS GPIO outputs
```

Step 4 后应改为：第一批固定接口在 `instance_init` 注册，依赖已公开 property 的 FIFO 和 CS 资源在 `realize()` 校验后创建。第一批只注册七路基础 IRQ；XIP GPIO 和第二个 MMIO region 随 XIP series 同时引入。

---

## ⬜ 2. 证据与 K230 三实例配置矩阵

### 2.1 证据结论

本计划采用以下证据优先级：K230 TRM 的实例寄存器/功能章节 → 当前 firmware 实际路径与 U-Boot DTS → RT-Smart/Linux 驱动 → 当前 QEMU/qtest。单个 DTS 冲突不能覆盖多源一致证据。

已重新定位的关键事实：

- TRM 5.3 FMC 明确支持 internal DMA、Enhanced Dual/Quad/Octal 和 XIP；
- TRM 12.3 SPI 明确支持 AXI internal DMA、256 深度 TX/RX FIFO 和 Enhanced Dual/Quad，不列出 XIP aperture；
- TRM 12.3 QSPI 寄存器表从 `0x0f8 DDR_DRIVE_EDGE` 直接跳到 `0x118 SPI_CTRLR1`，没有 `XIP_MODE_BITS`、`XIP_INCR_INST`、`XIP_WRAP_INST`；
- TRM 5.3 FMC 才在 `0x0fc/0x100/0x104` 定义上述三个 XIP 寄存器，并明确限定为 XIP mode bits 和 XIP opcode；
- FMC `IMR` reset 为 `0x0000003f`，QSPI `IMR` reset 为 `0x0000001f`；
- FMC `AXIAWLEN/AXIARLEN` reset 为 `0`，QSPI 为 `0x00000700`；
- FMC `SPI_CTRLR0` reset 为 `0x28000200`，QSPI 为 `0x04000200`；
- 两类实例的 IDR 均为 `0xa1b2c3d5`，VERSION 均为 `0x3130332a`；
- RT-Smart 对三个实例统一采用 `SSIC_HAS_DMA=2`、TX/RX 深度 256，并为三实例安装 DONE/AXIE IRQ；公共寄存器结构体包含 XIP 成员只证明最大布局和偏移占位，不能证明 QSPI 实例实现这些寄存器；
- RT-Smart 的 `spi0/spi1/spi2` 最大线宽分别为 8/4/4；
- U-Boot 基础 DTS 把物理 SPI-OPI、QSPI0、QSPI1 的 `num-cs` 配置为 1/5/5；Linux DTS 写成 1/1/1，与 U-Boot、当前模型和 qtest 冲突；
- U-Boot board DTS 实际启用 `spi0`（物理 SPI-OPI/FMC）作为启动 Flash 控制器；RT-Smart、U-Boot、Linux 的普通 enhanced/IDMA 路径只配置 `SPI_CTRLR0.WAIT_CYCLES`、`SPIDR`、`SPIAR` 和 DMA 地址，没有写 `XIP_MODE_BITS`；当前源码中的“SDK 1-4-4 复用 XIP mode byte”注释与 SDK 不符。

### 2.2 SDK 逻辑编号与 QEMU 数组下标

| 物理模块 | SDK 逻辑名 | QEMU `dw_ssi[]` 下标 | 控制器地址 |
|---|---|---:|---:|
| SPI-OPI/FMC | `spi0` | 2 | `0x91584000` |
| QSPI0 | `spi1` | 0 | `0x91582000` |
| QSPI1 | `spi2` | 1 | `0x91583000` |

代码、测试和文档不得用 `spi0` 指代 `dw_ssi[0]`。K230 profile 数组按 `ssi_index` 排列，注释同时写物理模块名和 SDK 逻辑名。

### 2.3 最终 K230 profile

| 配置项 | QSPI0 / `dw_ssi[0]` / SDK `spi1` | QSPI1 / `dw_ssi[1]` / SDK `spi2` | SPI-OPI/FMC / `dw_ssi[2]` / SDK `spi0` |
|---|---:|---:|---:|
| `num-cs` | 5 | 5 | 1 |
| `fifo-depth` | 256 | 256 | 256 |
| `imr-reset` | `0x0000001f` | `0x0000001f` | `0x0000003f` |
| 控制器 MMIO | `0x91582000` | `0x91583000` | `0x91584000` |
| XIP MMIO | 不创建/不映射 | 不创建/不映射 | 不创建/不映射 |

第一批只把上表中的实例差异写入 QOM property。其他已确认差异继续保留在证据矩阵中，但随对应功能 series 进入代码：

| 已确认差异 | QSPI0 | QSPI1 | SPI-OPI/FMC | 代码归属 |
|---|---:|---:|---:|---|
| 最大线宽 | 4 | 4 | 8 | enhanced SPI series |
| `SPI_CTRLR0` reset | `0x04000200` | `0x04000200` | `0x28000200` | enhanced SPI series |
| `AXIAWLEN/AXIARLEN` reset | `0x00000700` | `0x00000700` | `0x00000000` | IDMA series |
| internal-AXI DMA 布局 | 是 | 是 | 是 | IDMA series |
| XIP aperture | 无 | 无 | 128 MiB | XIP series |

IDR 和 VERSION 在三个实例上取值相同，第一批保留为通用模型常量。第一批不创建第二个 MMIO region，也不映射 `0xc0000000`。

### 2.4 QEMU 主线组织先例

| 主线代码 | 可借鉴点 | 本计划用法 |
|---|---|---|
| `include/hw/i2c/designware_i2c.h` | 运行状态完整保存在 device state，不建立恒 false capability 配置 | `DwSsiState` 保留 Standard PIO/FIFO/IRQ 的完整运行状态 |
| `hw/i2c/designware_i2c.c:597-605` | DMA 地址存在，但 read 返回 0、write 忽略并记录 unsupported | 借鉴“已知但未实现寄存器不保存 readback”的语义；SSI 的具体 offset 仍按 K230 internal-AXI 版图解释 |
| `hw/i2c/designware_i2c.c:624-638` | 能力寄存器直接报告当前真实能力 | 第一批不通过 property 或内部位图描述尚未实现的功能 |
| `include/hw/i3c/dw-i3c.h` | 状态内集中 `cfg` 结构 | `DwSsiState` 内增加 `DwSsiConfig cfg` |
| `hw/i3c/dw-i3c.c:1806-1824` | 在 `realize()` 按 property 创建 FIFO/MMIO | 第一批动态创建 SSI FIFO；可选 XIP region 留给后续 series |
| `hw/i3c/dw-i3c.c:1782-1803` | reset 从 `cfg` 填充寄存器字段 | `IMR` reset 从实例 profile 读取 |
| `hw/i3c/dw-i3c.c:1854-1872` | properties 直接写入 `cfg` | 使用 kebab-case QOM properties |
| `hw/ssi/xilinx_spips.c:1372-1388` | 控制器在 realize 中增加线性 Flash region | 留作后续 XIP property 与资源实现参考 |
| `hw/arm/xlnx-zynqmp.c:881-885` | SoC 负责映射控制器 region 与 LQSPI region | 留作后续 XIP series 中 K230 映射 `0xc0000000` 的参考 |
| `hw/ssi/xilinx_spips.c:1397-1401` | 非法配置通过 `error_setg()` 拒绝 realize | 集中校验 property 范围与组合 |

本计划不新增 wrapper。第一批只有控制器 region 0；后续 XIP series 按“通用模型实现 transaction/region、SoC 决定窗口大小与地址”的边界增加可选 region。

### 2.5 已确认决策与实施假设

| 项目 | 结论 |
|---|---|
| 配置组织 | 使用单一 `DwSsiConfig` 和 kebab-case QOM properties；不增加 K230 wrapper |
| 第一批字段 | `num_cs`、`fifo_depth`、`imr_reset`；IDR/VERSION 使用通用常量 |
| 扩展顺序 | 后续按 enhanced SPI → IDMA → XIP 分 series 引入；配置表达与正路径实现和测试同时落地 |
| 负路径载体 | 使用独立 `dw-ssi-test` machine，不向 K230 产品 machine 增加测试入口 |
| HI_SYS 边界 | 第一批不连接 HI_SYS 与 SSI，不保留 SSI getter/sleep 状态；查询和 XIP 控制接口随 HI_SYS/XIP series 引入 |
| XIP 归属 | 第一批不暴露 XIP property；XIP transaction、GPIO、可选 region 和 property 全部随 XIP series 后送 |
| DMA 寄存器 | 第一批只识别 K230 internal-AXI `DMACR/AXIAWLEN/AXIARLEN` 及后续 IDMA offset，read 返回 0、write 忽略；external DMA 不进入当前路线 |
| IRQ / VMState | TXU 属于基础 FIFO IRQ，DONE/AXIE 第一批不产生；迁移对 `num-cs`、`fifo-depth`、`imr-reset` 三项不可变配置做 equality |
| Octal 边界 | FMC 物理线宽证据保留在文档中，`max-lines` 随 enhanced series 引入 |
| 代码检查点 | Step 4.0 完成状态基于 `k230-V2-patch-spi` 的 `c689ac865f`；后续行号、QOM child 名称和默认行为若随 HEAD 变化，先重新定位，不直接套用旧行号 |

除上述检查点时效性外，本文没有依赖未确认 Databook 内容的阻塞假设。若新证据改变 K230 profile，先更新决策记录和配置矩阵，再实施代码。

---

## 🟦 3. 第一批精确配置方案

### 3.1 `DwSsiConfig`

**文件：`include/hw/ssi/dw_ssi.h`**

新增集中配置结构，把第一批确实影响实例创建、reset 或运行解释的参数收入 `cfg`：

```c
typedef struct DwSsiConfig {
    uint32_t num_cs;
    uint32_t fifo_depth;
    uint32_t imr_reset;
} DwSsiConfig;
```

V1.2 中其余候选字段的证据仍然有效，但代码归入对应功能 series：

| 字段 | 处理 |
|---|---|
| `component_id`、`version_id` | 当前三实例相同，第一批使用通用常量 |
| `max_lines`、`spi_ctrlr0_reset` | 随 enhanced SPI series 引入 |
| `dma_register_layout`、`axiawlen_reset`、`axiarlen_reset` | 随 IDMA series 引入 |
| `xip_window_size` | 随 XIP series 引入 |
| `capabilities` | 第一批不建立；后续根据真实消费者选择 bool、enum 或位图表达 |

`DwSsiState` 继续完整保存 Standard PIO/FIFO/IRQ 的运行状态，配置只保存在 `cfg`：

```c
struct DwSsiState {
    SysBusDevice parent_obj;

    MemoryRegion mmio;
    SSIBus *spi;

    qemu_irq *cs_lines;
    qemu_irq irqs[DW_SSI_IRQ_COUNT];

    Fifo32 tx_fifo;
    Fifo32 rx_fifo;
    uint32_t regs[DW_SSI_NUM_REGS];

    DwSsiConfig cfg;

    uint32_t irq_latched;
    uint32_t phase;
    uint32_t remaining_frames;

    int active_cs;
};
```

properties 只表达不可变硬件配置。`SSIENR`、`SER`、FIFO level、Standard PIO phase、IRQ latch 等 guest 运行时状态不得做成 property。enhanced command、IDMA、HI_SYS sleep 和 XIP 运行时字段由各自 series 在需要时增加。

第一批的 HI_SYS 边界必须和上述状态结构一致：`DwSsiState` 不保留 `sleep_status`，公共头文件不声明 `dw_ssi_get_spi_mode()` / `dw_ssi_is_sleeping()`，通用模型也不为后续 HI_SYS 查询预留 getter。K230 machine 第一批不得调用 `k230_hi_sys_set_ssi()`；HI_SYS 与 SSI 的 mode/sleep 查询接口随 HI_SYS series 一起恢复。

### 3.2 property 集合与兼容默认值

**文件：`hw/ssi/dw_ssi.c`**

```c
static const Property dw_ssi_properties[] = {
    DEFINE_PROP_UINT32("num-cs", DwSsiState, cfg.num_cs, 1),
    DEFINE_PROP_UINT32("fifo-depth", DwSsiState, cfg.fifo_depth, 256),
    DEFINE_PROP_UINT32("imr-reset", DwSsiState,
                       cfg.imr_reset, 0x0000003f),
};
```

通用默认实例提供 Standard PIO/FIFO/基础 IRQ。K230 machine 在 realize 前显式设置三项 property。第一批不暴露 `max-lines`、DMA layout、enhanced/IDMA/XIP property，也不在内部保存恒 false 开关。

IDR 和 VERSION 使用当前模型已经确认的通用常量。后续出现第二种需要同时建模的 component/version 时，再增加有真实实例消费者的配置。

### 3.3 realize 校验

新增单一入口 `dw_ssi_validate_config()`，所有动态资源创建前调用：

```c
#define DW_SSI_BASE_IRQ_MASK \
    (R_IMR_TXEIM_MASK | R_IMR_TXOIM_MASK | R_IMR_RXUIM_MASK | \
     R_IMR_RXOIM_MASK | R_IMR_RXFIM_MASK | R_IMR_MSTIM_MASK | \
     R_IMR_TXUIM_MASK)

static bool dw_ssi_validate_config(DwSsiState *s, Error **errp)
{
    DeviceState *dev = DEVICE(s);

    if (s->cfg.num_cs == 0 || s->cfg.num_cs > 8) {
        error_setg(errp, "%s: num-cs must be in range 1..8",
                   dev->canonical_path);
        return false;
    }

    if (s->cfg.fifo_depth < 2 || s->cfg.fifo_depth > 256) {
        error_setg(errp, "%s: fifo-depth must be in range 2..256",
                   dev->canonical_path);
        return false;
    }

    if (s->cfg.imr_reset & ~DW_SSI_BASE_IRQ_MASK) {
        error_setg(errp, "%s: imr-reset contains unsupported bits",
                   dev->canonical_path);
        return false;
    }

    return true;
}
```

TXE/TXO/RXU/RXO/RXF/MST 位于 bit 0..5，TXU 位于 bit 7，因此基础 IRQ mask 不能用连续七位代替。K230 的 `0x1f` 和 `0x3f` 都是合法 reset 值。

enhanced、IDMA 和 XIP 的组合校验不在第一批预埋。后续功能 series 引入对应配置时，再在同一 patch 中加入当时实际需要的依赖校验。

### 3.4 property 依赖的资源创建

`instance_init` 保留第一批固定接口：SSI bus、控制器 MMIO 和 TXE、TXO、RXF、RXO、TXU、RXU、MST 七路基础 IRQ。DONE/AXIE 随 IDMA series 增加；`xip-enable` GPIO 和第二个 MMIO region 随 XIP series 一起加入。

第一批 IRQ 枚举和输出顺序固定为：

```c
typedef enum DwSsiIrq {
    DW_SSI_IRQ_TXE,
    DW_SSI_IRQ_TXO,
    DW_SSI_IRQ_RXF,
    DW_SSI_IRQ_RXO,
    DW_SSI_IRQ_TXU,
    DW_SSI_IRQ_RXU,
    DW_SSI_IRQ_MST,
    DW_SSI_IRQ_COUNT,
} DwSsiIrq;
```

第一批删除 `DW_SSI_IRQ_DONE`、`DW_SSI_IRQ_AXIE`。IRQ output 顺序与 RISR bit 顺序不同，必须继续通过显式映射表关联，不能用 `BIT(irq_index)` 推导：

```c
static const uint32_t dw_ssi_irq_status_mask[DW_SSI_IRQ_COUNT] = {
    [DW_SSI_IRQ_TXE] = R_RISR_TXEIR_MASK,
    [DW_SSI_IRQ_TXO] = R_RISR_TXOIR_MASK,
    [DW_SSI_IRQ_RXF] = R_RISR_RXFIR_MASK,
    [DW_SSI_IRQ_RXO] = R_RISR_RXOIR_MASK,
    [DW_SSI_IRQ_TXU] = R_RISR_TXUIR_MASK,
    [DW_SSI_IRQ_RXU] = R_RISR_RXUIR_MASK,
    [DW_SSI_IRQ_MST] = R_RISR_MSTIR_MASK,
};
```

IDMA series 在 `MST` 之后追加 DONE、AXIE，保持第一批七路 output 的编号不变。

```c
static void dw_ssi_realize(DeviceState *dev, Error **errp)
{
    DwSsiState *s = DW_SSI(dev);
    if (!dw_ssi_validate_config(s, errp)) {
        return;
    }

    s->cs_lines = g_new0(qemu_irq, s->cfg.num_cs);
    qdev_init_gpio_out_named(dev, s->cs_lines, "cs", s->cfg.num_cs);

    fifo32_create(&s->tx_fifo, s->cfg.fifo_depth);
    fifo32_create(&s->rx_fifo, s->cfg.fifo_depth);
}
```

控制器固定 MMIO region 0 继续在 `instance_init` 创建。第一批任何实例都只有 region 0，machine 不得调用 `sysbus_mmio_map(..., 1, ...)`。

现有 `dw_ssi_finalize()` 继续统一执行 `fifo32_destroy()` 和 `g_free(s->cs_lines)`。QOM 实例内存初始为零，`fifo32_destroy()` / `g_free()` 对尚未创建的空资源安全，因此不新增 `fifo_created` 一类状态位。`realize()` 必须在全部配置校验通过后才创建 FIFO 和 CS，且资源创建后不再执行可能失败的配置检查，避免部分创建状态。

### 3.5 动态 FIFO 的全部影响点

删除 `DW_SSI_FIFO_CAPACITY`，所有容量判断统一使用 `s->cfg.fifo_depth`：

- `fifo32_create()` 的容量；
- `SR.TFNF`：`tx_used < fifo_depth`；
- `SR.RFF`：`rx_used == fifo_depth`；
- `TXFLR` / `RXFLR` 的最大可见 level；
- `TXFTLR.TFT`、`TXFTLR.TXFTHR`、`RXFTLR.RFT` 写入合法性；
- reset 后 FIFO 清空；
- VMState 目的端 FIFO 容量由同一 property 在 realize 时建立。

阈值写入不能只依赖固定 8/11 位 mask。新增辅助函数，超过当前 `fifo-depth - 1` 的阈值保持旧值并记录 guest error：

```c
static bool dw_ssi_fifo_threshold_valid(DwSsiState *s, uint32_t value)
{
    return value < s->cfg.fifo_depth;
}
```

`TXFTLR.TFT`、`TXFTLR.TXFTHR`、`RXFTLR.RFT` 分别提取字段后验证，再写回对应字段。K230 深度 256 时继续允许 `0..255`，通用测试使用深度 8 验证 `0..7`。

### 3.6 reset 从实例配置填充

删除用 `max_lines == 8` 推断 FMC 的逻辑。第一批 reset 只从实例配置读取 `IMR`，IDR 和 VERSION 使用通用常量：

```c
s->regs[R_IMR] = s->cfg.imr_reset & DW_SSI_BASE_IRQ_MASK;
s->regs[R_IDR] = DW_SSI_COMPONENT_ID;
s->regs[R_SSIC_VERSION_ID] = DW_SSI_VERSION_ID;
```

`DMACR`、`0x050/0x054`、`SPI_CTRLR0`、internal-AXI 和 XIP-only 寄存器第一批不保存状态，read 固定返回 0、write 忽略。reset 清空 FIFO、Standard PIO phase 和七路基础 IRQ latch，不包含 DONE/AXIE 或 XIP 状态。

---

## 🟪 4. 寄存器地址版图、第一批语义与后续边界

### 🟦 4.1 地址版图与运行状态分开

控制器 region 0 保持完整 MMIO aperture。寄存器地址可以被识别，但只有第一批已经实现的寄存器进入运行状态和 VMState。

第一批分为三类：

1. Standard PIO/FIFO/基础 IRQ 寄存器：实现 reset、write mask、动态 readback 和副作用；
2. internal-AXI DMA、enhanced、IDMA、XIP 已知 offset：read 返回 0，write 忽略并记录 unsupported；
3. aperture 外或未定义访问：记录 guest error。

DesignWare I2C 使用同样的边界。`DW_IC_DMA_CR`、`DW_IC_DMA_TDLR`、`DW_IC_DMA_RDLR` 地址存在，但 unsupported callback 不保存写入值：

```c
static uint32_t dw_ssi_unsupported_read(DwSsiState *s,
                                        hwaddr addr)
{
    qemu_log_mask(LOG_UNIMP,
                  "%s: unsupported read at 0x%" HWADDR_PRIx "\n",
                  DEVICE(s)->canonical_path, addr);
    return 0;
}

static void dw_ssi_unsupported_write(DwSsiState *s,
                                     hwaddr addr,
                                     uint32_t value)
{
    qemu_log_mask(LOG_UNIMP,
                  "%s: unsupported write at 0x%" HWADDR_PRIx
                  " value 0x%08x\n",
                  DEVICE(s)->canonical_path, addr, value);
}
```

这些 helper 只表达 guest-visible 语义，不读取 property 或内部 feature bit。

### 🟦 4.2 DMA 基础 offset 采用 internal-AXI 版图

K230 三实例由 SDK 明确选择 `SSIC_HAS_DMA == 2`，因此第一批只按 internal-AXI 版图命名和识别 DMA 基础 offset：

```c
REG32(DMACR,   0x04c)
REG32(AXIAWLEN, 0x050)
REG32(AXIARLEN, 0x054)
```

第一批 read/write 分派：

```c
case A_DMACR:
case A_AXIAWLEN:
case A_AXIARLEN:
    return dw_ssi_unsupported_read(s, addr);
```

```c
case A_DMACR:
case A_AXIAWLEN:
case A_AXIARLEN:
    dw_ssi_unsupported_write(s, addr, value);
    return;
```

精确契约：

- read 始终返回 0；
- write 不改变 `regs[]`；
- 不保存 IDMA enable、AXI burst 或其他 DMA 配置；
- 不访问 guest memory；
- 不形成可变 guest-visible 状态；若继续迁移完整 `regs[]`，这些槽位必须始终保持 0；
- 不需要 `dma-register-layout`。

IDMA series 实现 internal-AXI 正路径时，再把这些 offset 从 unsupported 分派移入真实寄存器语义，并同时增加 engine、guest-memory 状态、DONE/AXIE、迁移字段和 qtest。external DMA 的 `DMATDLR/DMARDLR` 布局不在当前路线中定义；以后出现真实消费者时再独立评估，不能与 internal-AXI 名称同时占用 `0x050/0x054`。

### 🟦 4.3 enhanced 与 XIP 寄存器的第一批 RAZ/WI 语义

`CTRLR0.SPI_FRF` 属于共享控制寄存器。第一批只接受 Standard 值：

```c
value &= ~R_CTRLR0_SPI_FRF_MASK;
dw_ssi_write_masked(s, R_CTRLR0, value,
                    DW_SSI_CTRLR0_STANDARD_WRITABLE_MASK);
```

以下扩展 offset 第一批保持 unsupported/RAZ/WI：

```text
SPI_CTRLR0
DDR_DRIVE_EDGE
XIP_MODE_BITS
XIP_INCR_INST
XIP_WRAP_INST
XIP_CTRL
XIP_SER
XRXOICR
```

其中 `XIP_MODE_BITS`、`XIP_INCR_INST`、`XIP_WRAP_INST` 始终属于 XIP transaction。ordinary enhanced 或 IDMA 1-4-4 不得读取这些寄存器。

### 🟦 4.4 internal DMA/IDMA 寄存器的第一批 RAZ/WI 语义

K230 internal-AXI 扩展包括：

```text
AXIAWLEN / AXIARLEN
SPIDR / SPIAR
AXIAR0 / AXIAR1
AXIECR / DONECR
```

这些地址和字段定义可以保留在寄存器目录与注释中，方便对照 TRM，但第一批 read 返回 0、write 忽略，不产生 DONE/AXIE，也不保存 `idma_completed_frames`、地址或 burst 状态。

第一批 TXU 仍然是有效基础 IRQ。TXU 表示 TX FIFO underflow，不依赖 IDMA。

### 🟦 4.5 第一批精确语义

| 类别 | 寄存器语义 | IRQ | 状态与数据路径 | 资源 |
|---|---|---|---|---|
| Standard PIO/FIFO | 真实 reset、mask、readback、副作用 | 七路基础 IRQ | 四种 TMOD、FIFO、CS | region 0、SSI bus、CS |
| internal-AXI DMA 基础 offset | 地址识别，read 0、write ignore | 无 DONE/AXIE | 不保存寄存器值、不访问 guest memory | 无 DMA engine |
| enhanced SPI | `SPI_FRF` 固定 Standard；扩展 offset RAZ/WI | 无扩展 IRQ | 不进入 enhanced decoder | 无额外资源 |
| internal DMA/IDMA | internal-AXI offset RAZ/WI | 无 DONE/AXIE output | 不访问 guest memory | 无额外资源 |
| XIP | XIP-only offset RAZ/WI | 无 XIP IRQ | 无 XIP transaction | 无 GPIO、无 region 1 |

system reset 后，上表中的固定语义不变。测试只观察 readback、数据路径、IRQ output 和资源，不查询内部实现开关。

### 🟦 4.6 Standard PIO/P1 寄存器契约

第一批必须显式锁定以下 guest-visible 契约，不能只在实现代码中隐含：

| 寄存器/功能 | 第一批契约 | 必需测试 |
|---|---|---|
| `CTRLR0`、`CTRLR1`、`MWCR`、`BAUDR` | 仅在 `SSIENR=0` 时接受配置写；enable 期间写入保持旧值并记录 guest error | enable 前写入成功；enable 后写入不改变 readback |
| `CTRLR0.TMOD` | 支持 TX/RX、TX-only、RX-only、EEPROM-read 四种 Standard 模式 | 四个独立 PIO 用例 |
| `CTRLR1.NDF` | RX-only/EEPROM-read 产生 `NDF + 1` 个接收 frame | `NDF=0` 和非零边界 |
| `SSIENR` | enable 启动已配置事务；disable 终止事务、拉高 active CS、清空 FIFO 和 phase | 传输中 disable 后回到 idle |
| `SER` | 只保留 `num-cs` 范围内位；`SER=0` 不选择从设备；非法多 CS 行为固定并记录错误 | 5/5/1 mask、无 CS 排队、非法多 CS |
| `DR` | disabled 时写忽略；enabled 且无 active CS 时可排队；FIFO 满时不增长并锁存 TXO；RX 空读返回 0 并锁存 RXU | 深度、overflow、underflow、frame mask |
| `TXFTLR`、`RXFTLR` | 阈值必须小于 `fifo-depth`，非法值保持旧值 | 深度 8 的 `0..7` 与非法 8 |
| `SR`、`TXFLR`、`RXFLR` | 从 FIFO、phase 和 active transfer 动态计算，不保存可被 guest 写入的影子值 | empty/full/busy/RFNE/TFE/TFNF |
| `IMR`、`ISR`、`RISR`、清除寄存器 | TXE/RXF 为 level；TXO/RXO/TXU/RXU 为 latched；MST 保留 mask/output 但第一批 raw 恒 0；`ISR = RISR & IMR` | raw/masked、threshold 翻转、逐项 read-clear、总清除 |
| 保留位和只读寄存器 | 写入不改变 guest-visible 值 | `UINT32_MAX` write-mask 测试 |

EEPROM-read 是 Standard 1-1-1 SPI NOR 的关键软件契约：guest 只向 TX FIFO 提交 command/address，控制器保持 CS，并自动产生 `NDF + 1` 次 dummy transfer 把返回数据填入 RX FIFO。Flash 集成测试会再次覆盖这一路径，但通用 qtest 必须先独立证明控制器状态机。

### 🟨 4.7 后续 series 的状态扩展

第一批 `DwSsiState` 已完整保存 Standard 运行状态。后续功能按实际实现扩展，而不是预先放空字段。

Enhanced SPI series 增加：

```c
typedef struct DwSsiEnhancedCommand {
    uint32_t instruction;
    uint32_t address;
    uint32_t instruction_bits;
    uint32_t address_bits;
    uint32_t wait_cycles;
    uint32_t data_frames;
    uint32_t spi_frf;
    uint32_t trans_type;
    uint32_t tmod;
} DwSsiEnhancedCommand;

/* DwSsiState */
DwSsiEnhancedCommand enhanced;
```

DMA/IDMA series 再引入寄存器布局和运行状态。例如：

```c
typedef enum DwSsiDmaRegisterLayout {
    DW_SSI_DMA_LAYOUT_EXTERNAL,
    DW_SSI_DMA_LAYOUT_INTERNAL_AXI,
} DwSsiDmaRegisterLayout;

typedef struct DwSsiDmaConfig {
    DwSsiDmaRegisterLayout register_layout;
    uint32_t axiawlen_reset;
    uint32_t axiarlen_reset;
} DwSsiDmaConfig;

typedef struct DwSsiIdmaState {
    uint32_t completed_frames;
    uint64_t address;
    uint32_t remaining_frames;
} DwSsiIdmaState;
```

IDMA series 实现 internal-AXI layout、guest-memory 进度与 DONE/AXIE。external DMA 不进入当前 V2 路线；若未来出现消费者，作为独立需求重新设计，不复用 internal-AXI 配置。

XIP series 增加 XIP 配置与资源：

```c
typedef struct DwSsiXipConfig {
    uint64_t window_size;
} DwSsiXipConfig;

/* DwSsiState */
MemoryRegion xip;
bool xip_enabled;
```

同时增加 XIP command、`xip-enable` GPIO 和 K230 aperture 映射。每次扩展都要同步更新 reset、VMState、post-load 校验和 qtest。

是否使用 bool、enum 或 capability 位图，在对应 series 根据当时实际组合决定。第一批不固定该内部表示。

### 🟨 4.8 IDMA 与 XIP 防回归约束

IDMA series 必须继续满足 Step 4.0 已验证的普通事务顺序：

```text
instruction → address → dummy → data
```

真正 XIP transaction 才允许：

```text
instruction → address → optional mode → dummy → data
```

`XIP_MODE_BITS` 不得改成 shared，也不得恢复 `TRANS_TYPE=1 && WAIT_CYCLES>=2` 的 mode-byte 特判。SDK 风格 `0xeb` IDMA 回归和 FMC XIP 正路径分别随对应 series 保留。

---

## 🟦 5. K230 machine 第一批精确改动

### 5.1 profile 数据结构

**文件：`hw/riscv/k230.c`**

在 `k230_ssi_routes[]` 附近增加按 `ssi_index` 排列的 profile：

```c
typedef struct K230DwSsiProfile {
    uint32_t num_cs;
    uint32_t fifo_depth;
    uint32_t imr_reset;
} K230DwSsiProfile;

static const K230DwSsiProfile k230_dw_ssi_profiles[] = {
    [0] = { /* QSPI0, SDK spi1 */
        .num_cs = 5,
        .fifo_depth = 256,
        .imr_reset = 0x0000001f,
    },
    [1] = { /* QSPI1, SDK spi2 */
        .num_cs = 5,
        .fifo_depth = 256,
        .imr_reset = 0x0000001f,
    },
    [2] = { /* SPI-OPI/FMC, SDK spi0 */
        .num_cs = 1,
        .fifo_depth = 256,
        .imr_reset = 0x0000003f,
    },
};
```

QSPI0/QSPI1 值相同但仍分别列出，让 review 可以直接对照三个硬件实例。最大线宽、`SPI_CTRLR0` reset、internal-AXI reset 和 XIP aperture 继续保留在第 2.3 节证据矩阵中，由后续 series 扩展 profile。

### 5.2 property 设置 helper

新增 `k230_configure_dw_ssi()`，只负责从 K230 profile 设置通用 property：

```c
static void k230_configure_dw_ssi(DwSsiState *ssi,
                                  const K230DwSsiProfile *profile)
{
    DeviceState *dev = DEVICE(ssi);

    qdev_prop_set_uint32(dev, "num-cs", profile->num_cs);
    qdev_prop_set_uint32(dev, "fifo-depth", profile->fifo_depth);
    qdev_prop_set_uint32(dev, "imr-reset", profile->imr_reset);
}
```

在三个 SSI realize 前循环调用，替换当前分散的 `num-cs` / `max-lines` 设置：

```c
for (int i = 0; i < ARRAY_SIZE(s->dw_ssi); i++) {
    k230_configure_dw_ssi(&s->dw_ssi[i],
                          &k230_dw_ssi_profiles[i]);
    if (!sysbus_realize(SYS_BUS_DEVICE(&s->dw_ssi[i]), errp)) {
        return;
    }
}
```

profile 不包含恒 false 字段，也不设置只能取单一值的 property。后续 series 扩展 profile 时，property、寄存器正路径和测试必须在同一提交中出现。

### 5.3 第一批只映射控制器 region 0

```c
for (int i = 0; i < ARRAY_SIZE(s->dw_ssi); i++) {
    k230_configure_dw_ssi(&s->dw_ssi[i], &k230_dw_ssi_profiles[i]);
    if (!sysbus_realize(SYS_BUS_DEVICE(&s->dw_ssi[i]), errp)) {
        return;
    }
}

sysbus_mmio_map(SYS_BUS_DEVICE(&s->dw_ssi[0]), 0,
                memmap[K230_DEV_QSPI0].base);
sysbus_mmio_map(SYS_BUS_DEVICE(&s->dw_ssi[1]), 0,
                memmap[K230_DEV_QSPI1].base);
sysbus_mmio_map(SYS_BUS_DEVICE(&s->dw_ssi[2]), 0,
                memmap[K230_DEV_SPI].base);

```

三个 profile 均只有 region 0。第一批不建立 K230 到 SSI 的 `xip-enable` GPIO 连接，不调用 `sysbus_mmio_map(..., 1, ...)`，也不把 `0xc0000000` 映射为 XIP aperture。

### 5.4 HI_SYS 与 SSI 的第一批边界

第一批不提交 HI_SYS 与 SSI 的 mode/sleep/XIP 集成。不能只删除 `xip-enable` GPIO，却继续保留 `k230_hi_sys_set_ssi()` 和通用 SSI getter。

最终第一批 patch 必须满足：

- `hw/riscv/k230.c` 不调用 `k230_hi_sys_set_ssi()`，不建立任何 HI_SYS→SSI 指针连接；
- `include/hw/ssi/dw_ssi.h` 不声明 `dw_ssi_get_spi_mode()`、`dw_ssi_is_sleeping()`；
- `DwSsiState` 不保存 `sleep_status`，VMState 不包含 HI_SYS 查询状态；
- 第一批不连接 HI_SYS `xip-enable` output，也不为 SSI 注册对应 input；
- K230 SSI qtest 不验证 HI_SYS mode/sleep/XIP 联动。

若当前本地开发树仍编译 `k230_hi_sys`，必须同时移除其对 `hw/ssi/dw_ssi.h` 的 include、`DwSsiState *ssi[]`、`k230_hi_sys_set_ssi()` 和动态 mode/sleep 查询；不能依赖空指针或临时兼容 getter 让第一批通过。HI_SYS 自身若仍作为独立 K230 MMIO block 保留，只提供不依赖 DW SSI 的本地寄存器语义。

后续 HI_SYS/XIP series 在同一批次重新引入：

- HI_SYS 到 SSI 的抽象查询/控制接口；
- mode/sleep 的真实消费者和测试；
- `xip-enable` GPIO；
- XIP region、K230 aperture 映射和迁移状态。

### 5.5 Standard 1-1-1 SPI Flash 挂接

最终选择 5.5-A：第一批保留 `spi-flash` machine property，并把 M25P80-compatible NOR 明确挂到 `dw_ssi[2]`（物理 SPI-OPI/FMC、SDK `spi0`）的 CS0。该 patch 只验证 Standard 1-1-1 PIO 访问：

- 不使用 `SPI_FRF=Dual/Quad`；
- 不设置 `IDMAE`；
- 不创建或访问 XIP aperture；
- 不把 U-Boot enhanced/IDMA 启动作为第一批合入条件。

Flash 挂接必须是独立 patch，不能与通用 `dw_ssi.c` 寄存器模型混在一起。测试至少验证 JEDEC ID 或固定地址普通读，证明 K230 实例具备真实 SSI peripheral consumer。

---

## 🟦 6. 第一批通用测试载体决策

### 6.1 选择独立通用 qtest

K230 三实例用于验证产品 profile、PLIC 和 Flash 集成；仍保留独立通用测试机，用于覆盖不同 FIFO 深度、Standard PIO/TMOD、基础 IRQ，以及 enhanced、DMA/IDMA、XIP 未实现时的固定寄存器和资源语义，不向 K230 产品 machine 增加测试专用 property。

新增文件：

| 文件 | 职责 |
|---|---|
| `hw/ssi/dw_ssi-test.c` | `CONFIG_DW_SSI && CONFIG_TEST_DEVICES` 下注册无 CPU 的独立 `dw-ssi-test` machine，实例化一个 `TYPE_DW_SSI` |
| `tests/qtest/dw-ssi-test.c` | 通过 `-preconfig` + QMP `qom-set` 建立不同配置，验证 property、FIFO、Standard PIO、unsupported/RAZ-WI 和基础 IRQ |
| `hw/ssi/meson.build` | 条件编译测试 machine |
| `tests/qtest/meson.build` | 构建并注册 `dw-ssi-test` |

测试 machine 只做三件事：创建一个 DW SSI、realize、映射 region 0。它不连接 K230 HI_SYS、PLIC 或 Flash，不包含 K230 常量。

```c
#include "qemu/osdep.h"
#include "qapi/error.h"
#include "qemu/module.h"
#include "hw/core/boards.h"
#include "hw/core/sysbus.h"
#include "hw/ssi/dw_ssi.h"

#define DW_SSI_TEST_MMIO_BASE 0x10000000
#define TYPE_DW_SSI_TEST_MACHINE MACHINE_TYPE_NAME("dw-ssi-test")
OBJECT_DECLARE_SIMPLE_TYPE(DwSsiTestMachineState, DW_SSI_TEST_MACHINE)

struct DwSsiTestMachineState {
    MachineState parent_obj;
    DwSsiState ssi;
};

static void dw_ssi_test_machine_instance_init(Object *obj)
{
    DwSsiTestMachineState *s = DW_SSI_TEST_MACHINE(obj);

    object_initialize_child(obj, "dw-ssi", &s->ssi, TYPE_DW_SSI);
}

static void dw_ssi_test_machine_init(MachineState *machine)
{
    DwSsiTestMachineState *s = DW_SSI_TEST_MACHINE(machine);
    SysBusDevice *sbd = SYS_BUS_DEVICE(&s->ssi);

    sysbus_realize(sbd, &error_fatal);
    sysbus_mmio_map(sbd, 0, DW_SSI_TEST_MMIO_BASE);
}

static void dw_ssi_test_machine_class_init(ObjectClass *oc,
                                           const void *data)
{
    MachineClass *mc = MACHINE_CLASS(oc);

    mc->desc = "DesignWare SSI qtest machine";
    mc->init = dw_ssi_test_machine_init;
    mc->max_cpus = 1;
    mc->default_ram_size = 0;
    mc->default_ram_id = "dw-ssi-test.ram";
    mc->no_serial = 1;
    mc->no_parallel = 1;
    mc->no_floppy = 1;
    mc->no_cdrom = 1;
}

static const TypeInfo dw_ssi_test_machine_info = {
    .name = TYPE_DW_SSI_TEST_MACHINE,
    .parent = TYPE_MACHINE,
    .instance_size = sizeof(DwSsiTestMachineState),
    .instance_init = dw_ssi_test_machine_instance_init,
    .class_init = dw_ssi_test_machine_class_init,
};

static void dw_ssi_test_machine_register_types(void)
{
    type_register_static(&dw_ssi_test_machine_info);
}

type_init(dw_ssi_test_machine_register_types)
```

测试机不创建 CPU、RAM、PLIC 或 Flash。控制器 child 必须在 machine `instance_init` 创建，而不是在 `MachineClass::init` 才创建：这样正常启动仍由 machine init realize 和映射；`-preconfig` 启动时则已经存在未 realize 的 `/machine/dw-ssi`，非法 property 测试可以通过 QMP 触发 realize 并接收正常 error response。

Meson 条件：

```meson
system_ss.add(
  when: ['CONFIG_DW_SSI', 'CONFIG_TEST_DEVICES'],
  if_true: files('dw_ssi-test.c'))

qtests_riscv64 += (
  config_all_devices.has_key('CONFIG_DW_SSI') and
  config_all_devices.has_key('CONFIG_TEST_DEVICES') ?
  ['dw-ssi-test'] : [])

qtests += {'dw-ssi-test': files('dw-ssi-test.c')}
```

第一段放入 `hw/ssi/meson.build`；后两段分别放在 `tests/qtest/meson.build` 的 `qtests_riscv64` 定义之后、现有 `qtests = { ... }` 字典闭合之后。不得在 C 源码中使用 `#ifdef CONFIG_TEST_DEVICES`；该符号由 Meson/Kconfig 选择测试源文件。

### 6.2 测试启动方式

```c
typedef struct DwSsiTestProperty {
    const char *name;
    uint64_t value;
} DwSsiTestProperty;

static void dw_ssi_qom_set_property(QTestState *qts,
                                    const DwSsiTestProperty *prop)
{
    QDict *args = qdict_new();

    qdict_put_str(args, "path", "/machine/dw-ssi");
    qdict_put_str(args, "property", prop->name);
    qdict_put_int(args, "value", prop->value);
    qtest_qmp_assert_success(
        qts, "{'execute':'qom-set','arguments':%p}", args);
}

static QTestState *dw_ssi_start_preconfig(void)
{
    return qtest_init("-machine dw-ssi-test -preconfig");
}

static QTestState *dw_ssi_start(const DwSsiTestProperty *props,
                                size_t num_props)
{
    QTestState *qts = dw_ssi_start_preconfig();

    for (size_t i = 0; i < num_props; i++) {
        dw_ssi_qom_set_property(qts, &props[i]);
    }
    qtest_qmp_assert_success(qts, "{'execute':'x-exit-preconfig'}");
    return qts;
}

typedef struct DwSsiInvalidConfig {
    const DwSsiTestProperty *props;
    size_t num_props;
    const char *error_text;
} DwSsiInvalidConfig;

static void test_invalid_config(gconstpointer opaque)
{
    const DwSsiInvalidConfig *test = opaque;
    QTestState *qts;
    QDict *response;
    QDict *error;

    qts = dw_ssi_start_preconfig();
    for (size_t i = 0; i < test->num_props; i++) {
        dw_ssi_qom_set_property(qts, &test->props[i]);
    }
    response = qtest_qmp(
        qts,
        "{'execute':'qom-set','arguments':{"
        "'path':'/machine/dw-ssi','property':'realized','value':true}}");

    g_assert_true(qmp_rsp_is_err(response));
    error = qdict_get_qdict(response, "error");
    g_assert_nonnull(strstr(qdict_get_str(error, "desc"),
                            test->error_text));

    qobject_unref(response);
    qtest_quit(qts);
}
```

配置类通用测试都从 `-preconfig` 开始，通过 QMP 设置本用例需要的第一批 properties。有效配置执行 `x-exit-preconfig`，由 machine init realize 控制器并映射 region；非法配置直接对 child 设置 `realized=true` 并断言 QMP error。property 尚未实现时，setup `qom-set` 先失败；property 存在但行为或校验尚未实现时，后续断言失败，两种状态都能形成明确的 TDD 红灯且不会阻塞 QMP。未实现寄存器测试直接验证 read 0、write ignore、无额外 IRQ/GPIO/MMIO 资源。

注册非法路径使用 `/dw-ssi/config/invalid/<case>`；例如 `num-cs-zero`、`fifo-depth-too-large` 和 `imr-reset-invalid`。每个 data case 列出属性数组；`error_text` 只省略设备 canonical path，保留核心错误文本。

典型配置：

```c
static const DwSsiTestProperty fifo_depth_8[] = {
    { "fifo-depth", 8 },
};

static const DwSsiTestProperty imr_reset_1f[] = {
    { "imr-reset", 0x1f },
};

static const DwSsiTestProperty invalid_imr_reset[] = {
    { "imr-reset", 0x00000100 }, /* AXIE bit */
};
```

---

## ✅ 7. 已完成前置 Step 4.0：解除普通 enhanced/IDMA 与 XIP 的错误耦合

**实施状态：已完成。** 对应代码检查点为 `c689ac865f`。新增 `/k230-dw-ssi/enhanced-xip-isolation` 和 `/k230-dw-ssi/idma-quad-io`，完整 K230 SSI qtest 12/12 PASS。

### 目标

先修正当前源码中的既有行为错误，再开始实例配置重构。普通 enhanced PIO 和 IDMA 都只执行：

```text
instruction → address → dummy → data
```

只有 `dw_ssi_prepare_xip_command()` 驱动的真正 XIP transaction 才允许执行：

```text
instruction → address → optional mode → dummy → data
```

### TDD 顺序

1. 在 `k230-dw-ssi-test` 增加普通 enhanced 与 XIP 字段隔离测试：使用 `0xeb`、`TRANS_TYPE=1`、24-bit address、8-bit instruction、`WAIT_CYCLES=6`，故意设置 `XIP_MD_BIT_EN=1`、8-bit `XIP_MBL` 和非零 `XIP_MODE_BITS`；普通传输结果不得变化；
2. 增加 SDK 风格 IDMA 1-4-4 回归：`XIP_MD_BIT_EN=0`，全部 mode/dummy 时钟由 `WAIT_CYCLES` 表达；
3. 保留现有 FMC XIP `0xeb` 正路径，确认 XIP transaction 仍使用 `XIP_INCR_INST` 和 `XIP_MODE_BITS`；
4. `dw_ssi_decode_enhanced_command()` 删除对 `XIP_MD_BIT_EN`、`XIP_MBL`、`XIP_MODE_BITS` 的读取；
5. 删除 `DW_SSI_PHASE_ENHANCED_MODE`，PIO 从 ADDRESS 直接进入 DUMMY；
6. 删除 IDMA 中 `trans_type == 1 && wait_cycles >= 2` 的特殊分支及错误 SDK 注释；
7. 从 VMState 删除不再属于持久 enhanced 状态的 mode 字段；XIP command 继续作为 MMIO 访问期间的局部状态使用。

### 完成标准

- 普通 enhanced/IDMA 路径不再读取任何 `XIP_*` 寄存器；
- 普通 enhanced 四阶段测试和 SDK 风格 IDMA 1-4-4 回归通过；
- FMC XIP mode bits 正路径保持通过；
- Step 4.0 完成前不得用配置或 unsupported 分支掩盖现有数据路径错误；先证明 ordinary 与 XIP transaction 的边界正确。

---

## 🟦 8. Step 4.1：建立配置、Standard PIO/FIFO 与基础 VMState

### 目标

引入配置表达、动态 FIFO、四种 Standard TMOD、基础 VMState 和通用测试载体。通用默认实例提供 Standard PIO/FIFO；internal-AXI DMA、enhanced、IDMA 和 XIP 寄存器使用固定 unsupported/RAZ-WI 语义。基础 IRQ 的完整输出和清除语义在 Step 4.2 增加。

### TDD 顺序

1. 新增独立 `dw-ssi-test` machine 和 qtest 骨架；
2. 先写 property 默认值、`fifo-depth=8`、四种 TMOD、写保护、状态寄存器、unsupported 寄存器和非法取值测试；
3. 增加 `DwSsiConfig` 和 properties；
4. 增加 `dw_ssi_validate_config()`；
5. 把 FIFO 创建移到 realize，并替换所有固定容量判断；
6. reset 改从 `cfg.imr_reset` 和通用 ID/VERSION 常量读取；
7. 运行通用 Standard PIO/FIFO 定向 qtest 和仍属于第一批范围的 K230 Standard PIO 回归。

### 失败断言

`/dw-ssi/config/fifo-depth` 使用 `fifo-depth=8`：

- 写 `SSIENR=1`、保持 `SER=0`，连续写 DR 8 次，`TXFLR == 8`；
- 第 9 次写入不增加 level，锁存 TX overflow；
- `SR.TFNF == 0`；
- 写 `TXFTLR.TFT=8`、`TXFTLR.TXFTHR=8` 或 `RXFTLR.RFT=8` 不改变原值；
- system reset 后 `TXFLR == 0`、`SR.TFNF == 1`。

`/dw-ssi/config/invalid/<case>` 至少覆盖：

| 参数 | 非法值 | 预期错误 |
|---|---:|---|
| `num-cs` | 0、9 | `num-cs must be in range 1..8` |
| `fifo-depth` | 1、257 | `fifo-depth must be in range 2..256` |
| `imr-reset` | `0x80000000` | `imr-reset contains unsupported bits` |

### 局部验证

```bash
ninja -C build qemu-system-riscv64 tests/qtest/dw-ssi-test
mkdir -p /tmp/qemu-k230-qtest-v2-step4
TMPDIR=/tmp/qemu-k230-qtest-v2-step4 \
QTEST_QEMU_BINARY=$PWD/build/qemu-system-riscv64 \
build/tests/qtest/dw-ssi-test -p /dw-ssi/config -v
```

### 完成标准

- `DwSsiConfig` 与第一批 property 集合存在；
- `fifo-depth=8` 的通用测试通过；
- 四种 Standard TMOD、写保护、状态/FIFO 和保留位契约测试通过；
- internal-AXI DMA、enhanced、IDMA、XIP unsupported read/write 测试通过；
- `DwSsiState` 保留 Standard regs、FIFO、phase、remaining frames 和 active CS；`irq_latched` 随 Step 4.2 加入；
- 仍属于第一批范围的 K230 Standard PIO 回归通过。

---

## 🟦 9. Step 4.2：增加七路基础 IRQ

### 目标

在 Step 4.1 的 Standard PIO/FIFO 状态机上实现 TXE、TXO、RXF、RXO、TXU、RXU、MST 七路基础 IRQ。DONE、AXIE 和 XIP 相关 output 不在第一批注册。

### IRQ 精确语义

| IRQ | 类型 | 置位条件 | 清除条件 |
|---|---|---|---|
| TXE | level | `TXFLR <= TXFTLR.TFT` | TX level 超过 threshold |
| RXF | level | `RXFLR > RXFTLR.RFT` | RX level 降到 threshold 以下或等于 threshold |
| TXO | latched | TX FIFO 满时继续写 DR | 对应 clear register 或总清除 |
| RXO | latched | 控制器产生 RX frame 时 RX FIFO 已满 | 对应 clear register 或总清除 |
| TXU | latched | Standard TX 状态机需要 frame 但 TX FIFO 为空 | TX clear register 或总清除 |
| RXU | latched | guest 在 RX FIFO 为空时读 DR | RXU clear register 或总清除 |
| MST | latched/output 保留 | 第一批没有可模拟的 multi-master contention source，raw 保持 0 | MST clear register 或总清除为无害操作 |

`RISR` 返回 raw 状态，`ISR` 返回 `RISR & IMR`。TXE/RXF 由 FIFO level 动态计算，不放入 `irq_latched`；其余基础错误位进入 `irq_latched`。`IMR` 只接受 `DW_SSI_BASE_IRQ_MASK`，DONE/AXIE/XIP 位写入后读回为 0。

### 接口级改动

第一批必须同步修改所有依赖 `DW_SSI_IRQ_COUNT` 的位置：

- `include/hw/ssi/dw_ssi.h`：IRQ enum 收缩为七路；
- `hw/ssi/dw_ssi.c`：IRQ status mapping、`sysbus_init_irq()`、reset/update 循环只遍历七路；
- `hw/riscv/k230.c`：`k230_connect_ssi_irqs()` 的范围断言和连接循环使用新的 `DW_SSI_IRQ_COUNT`；
- K230 每个实例只占用 `irq_base + 0..6`，原 `irq_base + 7/+8` 在第一批保持未连接；
- tests：通用 output 编号和 K230 PLIC source 均按七路断言。

IDMA series 增加 DONE/AXIE 时，把它们追加为 output 7/8，并连接既有 `irq_base + 7/+8`，不得改变第一批 output 0..6 的含义。

### TDD 顺序

1. 新增 `/dw-ssi/irq/level`，覆盖 TXE/RXF threshold 上下翻转；
2. 新增 `/dw-ssi/irq/latched`，分别制造 TXO/RXO/TXU/RXU，并验证 raw、masked 和 output；MST 验证 mask/output 存在但 raw 保持 0；
3. 新增 `/dw-ssi/irq/clear`，逐项验证 read-clear 和总清除寄存器；
4. 用 `qtest_irq_intercept_out()` 验证七路基础 output 的编号和相互隔离；DONE/AXIE 通过 IMR/RISR/ISR 固定为 0 证明无行为，不依赖越界 IRQ 编号证明“output 不存在”；
5. 实现 raw status、mask、clear helper 和七路 `sysbus_init_irq()`；
6. 运行通用 IRQ qtest，确认 Step 4.1 的四种 TMOD、FIFO 和 reset 回归不变。

### 完成标准

- 七路基础 IRQ 的 level/latched/mask/clear 语义全部有独立断言；
- TXE/RXF 不被错误锁存，latched 错误不会因 level 变化自动消失；
- TXU 保持 Standard FIFO underflow 语义，不依赖 IDMA；
- DONE/AXIE output、状态和清除寄存器行为未进入第一批；
- `irq_latched` 加入 VMState，并在 post-load 时拒绝第一批不可能出现的位；
- 通用 PIO/FIFO/IRQ qtest 全部通过。

---

## 🟦 10. Step 4.3：应用 K230 三实例 profile、PLIC 与可选资源裁剪

### 目标

证明同一 `TYPE_DW_SSI` 可以表达 QSPI0、QSPI1、FMC 的第一批实例差异，通用模型不再通过 K230 下标或常量猜测 `IMR` reset。

### TDD 顺序

1. 扩展 `K230SsiInstance` 测试表，先写三实例 expected profile；
2. 让 `test_register_contract()` 对每个实例读取 IMR、IDR、VERSION 和 Standard 寄存器；
3. 新增 `/k230-dw-ssi/fifo-depth`，实际写入 256 帧并验证第 257 帧不增加 level；
4. 通过 QMP `qom-get` 验证每个 child 的 `num-cs`、`fifo-depth` 和 `imr-reset`；通过寄存器/资源行为验证 unsupported 语义；
5. 在 `k230.c` 增加 profile 数组和设置 helper；
6. 删除 reset 中 `max_lines == 8` 的 FMC 推断；
7. 删除 FMC region 1 映射、全部 `k230_hi_sys_set_ssi()` 调用和 HI_SYS `xip-enable` GPIO 连接；
8. 让 `k230_connect_ssi_irqs()` 对每个实例只连接 output 0..6 到 `irq_base + 0..6`；
9. system reset 后重新执行三实例 profile 和 PLIC 路由断言。

### 三实例 reset 断言

每个实例都验证：

```text
CTRLR0
SSIENR
IMR
IDR
SSIC_VERSION_ID
SER mask
```

原测试只在 `spi0` 抽查 IDR/VERSION/IMR，必须改为循环验证三个物理实例。QSPI 的 `IMR=0x1f` 和 FMC 的 `IMR=0x3f` 是本小目标最重要的差异断言之一。

### property 内省

测试 QOM path：

```text
/machine/soc/k230-qspi0
/machine/soc/k230-qspi1
/machine/soc/k230-spi-opi
```

第一批 properties 均按数值读取，使用 `qom-get` QMP response 中的 `QNum` 验证。最大线宽和扩展 reset 继续由第 2.3 节记录，后续功能 series 再增加对应 QOM 断言。

### 局部验证

```bash
ninja -C build qemu-system-riscv64 tests/qtest/k230-dw-ssi-test
mkdir -p /tmp/qemu-k230-qtest-v2-step4
TMPDIR=/tmp/qemu-k230-qtest-v2-step4 \
QTEST_QEMU_BINARY=$PWD/build/qemu-system-riscv64 \
build/tests/qtest/k230-dw-ssi-test \
  -p /k230-dw-ssi/register-contract -v
TMPDIR=/tmp/qemu-k230-qtest-v2-step4 \
QTEST_QEMU_BINARY=$PWD/build/qemu-system-riscv64 \
build/tests/qtest/k230-dw-ssi-test \
  -p /k230-dw-ssi/fifo-depth -v
```

### 完成标准

- K230 machine 显式设置第一批全部 property；
- QSPI0/QSPI1/FMC 的 reset profile 与配置矩阵一致；
- 5/5/1 个 CS 和 256 深度 FIFO 有行为断言；
- 三实例 internal-AXI DMA、enhanced、IDMA、XIP offset 使用相同固定 unsupported 语义；
- 三实例都只创建和映射 region 0，`0xc0000000` 不映射；
- K230 machine 与 HI_SYS 不保存 SSI 指针或查询连接，通用 SSI 不暴露 mode/sleep getter；
- 三实例各自只连接七路基础 PLIC source，`irq_base + 7/+8` 保持未连接；
- Standard PIO/IRQ 和寄存器兼容性测试通过。

---

### 10.1 K230 未实现扩展的集成回归

通用 unsupported/RAZ-WI 语义已经在 Step 4.1 实现。Step 4.3 不重复建立 capability property 或内部位图，只验证 K230 三实例均继承相同语义，并确认 machine 已删除未实现功能的 IRQ、GPIO、region 和地址映射。

#### 10.1.1 enhanced SPI 寄存器

##### 寄存器与数据路径测试

`/dw-ssi/register/unsupported-enhanced`：

- 启动 Standard-only profile；
- 向 `CTRLR0.SPI_FRF` 写 Quad，读回仍为 Standard；
- `SPI_CTRLR0`、`DDR_DRIVE_EDGE` 读回 0，写入不保存；
- 配置 Standard loopback，PIO 收发仍正常；
- 尝试 enhanced command 后 TX FIFO 不被 enhanced engine 消费；
- `CTRLR0.SPI_FRF` 读回 Standard，不增加测试专用 property。

##### 实现要求

- `SPI_CTRLR0`、`DDR_DRIVE_EDGE` 等扩展寄存器走 unsupported/RAZ-WI；
- CTRLR0 read/write mask 动态清除 enhanced 字段；
- `dw_ssi_run_transfer()` 第一批只包含 Standard path；
- 第一批 VMState 不保存 enhanced command/phase 状态。

##### 第一批回归

```bash
TMPDIR=/tmp/qemu-k230-qtest-v2-step4 \
QTEST_QEMU_BINARY=$PWD/build/qemu-system-riscv64 \
build/tests/qtest/dw-ssi-test \
  -p /dw-ssi/register/unsupported-enhanced -v
TMPDIR=/tmp/qemu-k230-qtest-v2-step4 \
QTEST_QEMU_BINARY=$PWD/build/qemu-system-riscv64 \
build/tests/qtest/k230-dw-ssi-test \
  -p /k230-dw-ssi/register-contract -v
```

#### 10.1.2 DMA/IDMA 寄存器

##### 寄存器、IRQ 与数据路径测试

`/dw-ssi/register/unsupported-dma`：

- `DMACR`、`AXIAWLEN`、`AXIARLEN`、`SPIDR`、`SPIAR`、`AXIAR0/1`、`AXIECR`、`DONECR` read 返回 0、write 忽略；
- 第一批不定义或测试 external `DMATDLR/DMARDLR` 寄存器含义；`0x050/0x054` 只按 internal-AXI offset 识别；
- 向 IMR 写 DONE/AXIE 位，读回仍为 0；TXU 位仍可写入和读回；
- 尝试写 DMA/IDMA 寄存器后不访问 guest memory；
- `SR.CMPLTD_DF == 0`；
- 用 `qtest_irq_intercept_out()` 拦截 `/machine/dw-ssi` 的七路基础输出，再制造 TX FIFO underflow，确认 TXU raw status 和输出仍可见；DONE/AXIE output 不存在由设备初始化与 K230 接线代码检查保证，qtest 只验证其 IMR/RISR/ISR 位固定无效；
- DR 写入继续进入普通 TX FIFO，证明关闭 IDMA 没有破坏 PIO。

##### 实现要求

- internal-AXI 地址进入集中 unsupported helper；
- 第一批不保留可启动的 `dw_ssi_idma_enabled()` 数据路径；
- DONE/AXIE output 随 IDMA series 引入；
- 第一批 VMState 不保存 IDMA engine 运行时字段。

##### 第一批回归

```bash
TMPDIR=/tmp/qemu-k230-qtest-v2-step4 \
QTEST_QEMU_BINARY=$PWD/build/qemu-system-riscv64 \
build/tests/qtest/dw-ssi-test \
  -p /dw-ssi/register/unsupported-dma -v
TMPDIR=/tmp/qemu-k230-qtest-v2-step4 \
QTEST_QEMU_BINARY=$PWD/build/qemu-system-riscv64 \
build/tests/qtest/k230-dw-ssi-test \
  -p /k230-dw-ssi/register-contract -v
```

#### 10.1.3 XIP 寄存器与资源

##### 寄存器与资源测试

`/dw-ssi/register/unsupported-xip`：

- `XIP_MODE_BITS`、`XIP_INCR_INST`、`XIP_WRAP_INST` 和 XIP 扩展寄存器全部 RAZ/WI；
- QMP `qom-list /machine/dw-ssi` 中没有名为 `designware-ssi.xip[0]`、类型为 `child<memory-region>` 的条目；
- system reset 后行为不变。
- device 不注册 `xip-enable` GPIO，K230 不映射 region 1。

##### 实现要求

- 专用 XIP 寄存器分组 RAZ/WI；
- 第一批不注册 XIP GPIO、不创建第二个 MemoryRegion，K230 不映射 `0xc0000000`。

##### 第一批回归

```bash
TMPDIR=/tmp/qemu-k230-qtest-v2-step4 \
QTEST_QEMU_BINARY=$PWD/build/qemu-system-riscv64 \
build/tests/qtest/dw-ssi-test \
  -p /dw-ssi/register/unsupported-xip -v
TMPDIR=/tmp/qemu-k230-qtest-v2-step4 \
QTEST_QEMU_BINARY=$PWD/build/qemu-system-riscv64 \
build/tests/qtest/k230-dw-ssi-test \
  -p /k230-dw-ssi/register-contract -v
```

### Step 4.3 完成标准

- enhanced、internal DMA/IDMA 和 XIP unsupported 行为都有测试；
- K230 三实例使用相同的固定 unsupported 语义；
- DMA 寄存器不保存 readback，无 engine、guest memory 和 DONE/AXIE 行为；
- 无 XIP GPIO、region 1 和 `0xc0000000` 映射；
- 后续 series 按真实功能逐项增加配置、状态、IRQ、资源和测试。

---

## 🟦 11. 第一批 VMState 与迁移边界

### 11.1 配置 equality

QOM properties 在 realize 前确定，和 machine type/profile 一起由源端、目的端分别创建。它们不作为 guest 可变状态迁移，但三项配置都会改变迁移后硬件行为，必须作为 equality guard 放在依赖字段之前：

```c
VMSTATE_UINT32_EQUAL(cfg.num_cs, DwSsiState),
VMSTATE_UINT32_EQUAL(cfg.fifo_depth, DwSsiState),
VMSTATE_UINT32_EQUAL(cfg.imr_reset, DwSsiState),
```

`fifo-depth` 决定 `VMSTATE_FIFO32` 的容量和 level 解释；`num-cs` 决定 `SER`、`active_cs` 和 CS GPIO 拓扑；`imr-reset` 决定迁移后下一次 system reset 的 guest-visible IRQ mask。任一项不一致都拒绝迁移。ID 和 VERSION 第一批为通用常量，不需要额外 equality 字段。

### 11.2 第一批只迁移已实现状态

第一批 VMState 只保存：

- `regs[]`；
- TX/RX FIFO；
- Standard PIO phase/remaining frames；
- IRQ latch；
- active CS 和其他基础状态。

第一批不提前保存 enhanced command、`idma_completed_frames`、DMA engine phase 或 `xip_enabled`。后续 series 增加对应功能时，通过 VMState version/subsection 加入新增运行时字段，不把未实现状态预埋到第一批 schema。

TXU 属于基础 TX FIFO underflow 状态，继续保存在基础 IRQ latch 中。DONE/AXIE 位第一批始终无效；保存前应保证它们为 0，加载时若迁移流包含不可能出现的位则返回 `-EINVAL`，不能静默接受损坏状态。

### 11.3 兼容性结论

当前 V2 分支尚未上游发布，Step 4 不承诺与未发布中间态进行跨版本迁移。Plan Final V1.3 对第一批三项不可变配置全部增加 equality。测试覆盖同 profile 成功，以及 `num-cs`、`fifo-depth`、`imr-reset` 任一不一致时迁移失败。

---

## 🟦 12. 第一批 TDD 测试矩阵

| 测试路径 | 配置 | 核心断言 |
|---|---|---|
| `/dw-ssi/config/defaults` | 默认通用实例 | 三项 property、Standard PIO/FIFO/基础 IRQ |
| `/dw-ssi/config/fifo-depth` | `fifo-depth=8` | 8 帧满，第 9 帧不增长，阈值边界，reset 清空 |
| `/dw-ssi/config/invalid/*` | `-preconfig` 下每个路径一个非法组合 | QMP `realized=true` 返回精确配置错误 |
| `/dw-ssi/pio/tx-rx` | Standard TX/RX | 全双工 frame、DFS mask、FIFO level、CS 生命周期 |
| `/dw-ssi/pio/tx-only` | Standard TX-only | TX FIFO 消费、无 RX frame、完成后 idle |
| `/dw-ssi/pio/rx-only` | Standard RX-only | `NDF + 1` 自动接收、RX FIFO 和 underflow |
| `/dw-ssi/pio/eeprom-read` | Standard EEPROM-read | command/address 后自动 dummy，CS 保持，返回 `NDF + 1` 帧 |
| `/dw-ssi/register/write-protect` | enable 前后写配置寄存器 | enable 期间 `CTRLR0/1`、`MWCR`、`BAUDR` 保持旧值 |
| `/dw-ssi/register/status-mask` | FIFO/phase/保留位 | SR/TXFLR/RXFLR 动态值、RO 和 writable mask |
| `/dw-ssi/irq/level` | TXE/RXF | threshold 上下翻转，raw/masked/output 一致 |
| `/dw-ssi/irq/latched` | TXO/RXO/TXU/RXU，MST 无 source | 错误锁存且不因 level 自动清除；MST raw 保持 0 |
| `/dw-ssi/irq/clear` | 各 clear 寄存器 | 逐项 read-clear 与总清除 |
| `/dw-ssi/register/unsupported-enhanced` | 默认通用实例 | SPI_FRF 保持 Standard，扩展 offset read 0/write ignore |
| `/dw-ssi/register/unsupported-dma` | 默认通用实例 | internal-AXI DMA offset read 0/write ignore，无 guest memory/DONE/AXIE |
| `/dw-ssi/register/unsupported-xip` | 默认通用实例 | XIP-only 寄存器 RAZ/WI，无 GPIO 和第二 region |
| `/dw-ssi/migration/same-profile` | 三项配置相同 | 迁移成功，FIFO、寄存器和 IRQ 状态恢复 |
| `/dw-ssi/migration/fifo-depth-mismatch` | 8 → 16 | equality 拒绝迁移 |
| `/dw-ssi/migration/num-cs-mismatch` | 1 → 2 | equality 拒绝迁移 |
| `/dw-ssi/migration/imr-reset-mismatch` | `0x1f` → `0x3f` | equality 拒绝迁移 |
| `/k230-dw-ssi/register-contract` | K230 三实例 | 完整 reset profile、SER 位宽、properties |
| `/k230-dw-ssi/fifo-depth` | K230 256 深度 | 256 帧满，第 257 帧不增长 |
| `/k230-dw-ssi/plic-isolation` | K230 三实例七路 IRQ | output 0..6 对应 `irq_base + 0..6`，实例间不串扰，`+7/+8` 未连接 |
| `/k230-dw-ssi/unsupported-registers` | K230 三实例 | 三实例固定 unsupported 语义一致、无 XIP region |
| `/k230-dw-ssi/standard-flash` | M25P80-compatible NOR | Standard 1-1-1 JEDEC ID/固定地址读取，不访问 XIP aperture |
| K230 WDT qtest | K230 SoC | SSI 资源数量变化不破坏其他设备 realize/mapping |

系统 reset 后至少重新运行：三实例 reset profile、FIFO 清空、unsupported 寄存器语义和 Standard Flash 访问。

Step 4.0 的 enhanced/XIP 隔离与 IDMA `0xeb` 测试属于当前 V2 中间态的已完成回归，不进入第一批最终 patch。后续 enhanced/IDMA series 恢复对应功能时必须同时恢复这两项测试，且 `XIP_MODE_BITS` 仍保持 XIP-only。

---

## 🟦 13. Step 4.4：挂接 Standard Flash 并收敛验证

先按 §5.5 将 M25P80-compatible NOR 明确挂到 `dw_ssi[2]`（物理 SPI-OPI/FMC、SDK `spi0`）的 CS0。测试使用 Standard TX-only 和 EEPROM-read 读取 JEDEC ID/固定地址，不使用 enhanced、IDMA 或 XIP。Flash 正路径通过后再执行以下完整收敛验证。

### 13.1 编译

```bash
ninja -C build qemu-system-riscv64
ninja -C build tests/qtest/dw-ssi-test
ninja -C build tests/qtest/k230-dw-ssi-test
ninja -C build tests/qtest/k230-wdt-test
```

### 13.2 通用与 K230 qtest

```bash
mkdir -p /tmp/qemu-k230-qtest-v2-step4

TMPDIR=/tmp/qemu-k230-qtest-v2-step4 \
QTEST_QEMU_BINARY=$PWD/build/qemu-system-riscv64 \
build/tests/qtest/dw-ssi-test -v

TMPDIR=/tmp/qemu-k230-qtest-v2-step4 \
QTEST_QEMU_BINARY=$PWD/build/qemu-system-riscv64 \
build/tests/qtest/k230-dw-ssi-test -v

TMPDIR=/tmp/qemu-k230-qtest-v2-step4 \
QTEST_QEMU_BINARY=$PWD/build/qemu-system-riscv64 \
build/tests/qtest/k230-wdt-test -v
```

成功标准：第一批通用配置、PIO、IRQ、unsupported 寄存器和迁移测试全部 PASS；K230 三实例 profile、PLIC、Standard Flash 测试全部 PASS；WDT qtest PASS；无 FAIL/ERROR/SKIP。当前 V2 中间态的 enhanced/IDMA/XIP 测试不计入第一批通过口径。

### 13.3 公共头文件独立包含

```bash
printf '#include "qemu/osdep.h"\n#include "hw/ssi/dw_ssi.h"\n' | \
cc -fsyntax-only -x c -I. -Iinclude -Ibuild -Ibuild/include \
  $(pkg-config --cflags glib-2.0) -
```

### 13.4 通用层 K230 依赖残留

```bash
rg -n 'k230|K230' hw/ssi/dw_ssi.c include/hw/ssi/dw_ssi.h
rg -n 'k230_hi_sys_set_ssi|dw_ssi_get_spi_mode|dw_ssi_is_sleeping|sleep_status' \
  hw/ssi/dw_ssi.c include/hw/ssi/dw_ssi.h \
  hw/riscv/k230.c hw/misc/k230_hi_sys.c include/hw/misc/k230_hi_sys.h
```

预期：第一条除版权作者或说明文档明确允许的文本外，不出现 K230 类型、地址、reset、HI_SYS 或 machine 依赖；第二条无结果，证明第一批没有残留 HI_SYS↔SSI 指针、getter 或 sleep 状态。通用测试 machine 也不得包含 K230 常量。

### 13.5 格式与静态检查

```bash
git diff --check
scripts/checkpatch.pl -f include/hw/ssi/dw_ssi.h
scripts/checkpatch.pl -f hw/ssi/dw_ssi.c
scripts/checkpatch.pl -f hw/ssi/dw_ssi-test.c
scripts/checkpatch.pl -f hw/riscv/k230.c
scripts/checkpatch.pl -f tests/qtest/dw-ssi-test.c
scripts/checkpatch.pl -f tests/qtest/k230-dw-ssi-test.c
```

本计划不执行 commit、push 或分支操作。`git diff --check` 和 checkpatch 仅作为未来实施时的验证命令。

---

## 🟦 14. 第一批实施清单（含已完成前置）

### ✅ Step 4.0：纠正现有数据路径

- [x] 普通 enhanced/IDMA 解码不读取 `XIP_MD_BIT_EN`、`XIP_MBL`、`XIP_MODE_BITS`
- [x] 删除普通 enhanced mode phase 和 IDMA 1-4-4 特判
- [x] 普通 enhanced/IDMA 四阶段回归通过
- [x] FMC XIP mode bits 正路径保持通过

### 🟦 Step 4.1：配置、Standard PIO/FIFO 与基础 VMState

- [ ] 新增 `DwSsiConfig`，把 `num_cs`、`fifo_depth` 和 `imr_reset` 收入 `cfg`
- [ ] `DwSsiState` 保留完整 Standard regs、FIFO、phase、remaining frames 和 active CS
- [ ] 增加 `dw_ssi_validate_config()`
- [ ] 增加独立 `dw-ssi-test` machine、Meson 注册与 qtest
- [ ] 非法配置按 case 使用 `-preconfig` + QMP realize error 验证
- [ ] FIFO 按 `fifo-depth` 在 realize 创建
- [ ] 所有 FIFO 容量与阈值逻辑改用 `cfg.fifo_depth`
- [ ] 保留 finalize 统一销毁 FIFO 和动态 CS 数组
- [ ] reset 从 `cfg.imr_reset` 和通用 ID/VERSION 常量读取
- [ ] 四种 Standard TMOD、`NDF + 1`、CS 生命周期和 frame mask 测试通过
- [ ] enable 期间配置写保护、RO/保留位、SR/TXFLR/RXFLR 动态语义测试通过
- [ ] 增加 `num-cs`、`fifo-depth`、`imr-reset` VMState equality
- [ ] 同 profile 迁移成功，三项配置任一不一致时迁移失败
- [ ] internal-AXI DMA、enhanced、IDMA、XIP 地址走 unsupported/RAZ-WI 测试

### 🟦 Step 4.2：七路基础 IRQ

- [ ] TXE/RXF level IRQ 随 threshold 动态翻转
- [ ] TXO/RXO/TXU/RXU latched IRQ 的置位和保持语义正确；MST 在无 contention source 时保持 0
- [ ] RISR、IMR、ISR、逐项 clear 和总 clear 测试通过
- [ ] 只注册 TXE/TXO/RXF/RXO/TXU/RXU/MST 七路 output
- [ ] DONE/AXIE 位固定无效，TXU 不依赖 IDMA
- [ ] `irq_latched` 加入 VMState，post-load 拒绝第一批无效位

### 🟦 Step 4.3：K230 profile、PLIC 与可选资源裁剪

- [ ] 在 `k230.c` 增加三实例完整 profile
- [ ] 三实例 realize 前显式设置三项 property
- [ ] 删除 `max_lines == 8` 推断 FMC reset
- [ ] 删除全部 `k230_hi_sys_set_ssi()` 调用、SSI mode/sleep getter、HI_SYS `xip-enable` 连接和 XIP region 映射
- [ ] 若本地仍编译 `k230_hi_sys`，移除其 DW SSI include、SSI 指针数组和动态查询接口
- [ ] 三实例七路基础 IRQ 连接到正确 PLIC source
- [ ] `DW_SSI_IRQ_COUNT=7`，所有 IRQ mapping/init/update/route 循环同步收缩
- [ ] `irq_base + 7/+8` 第一批未连接，保留给后续 DONE/AXIE
- [ ] 三实例 reset、SER、FIFO、unsupported/property 和 PLIC 隔离测试通过
- [ ] SPI_FRF 保持 Standard，enhanced offset read 0/write ignore
- [ ] internal-AXI offset read 0/write ignore；不定义 external DMA layout
- [ ] 无 guest memory 访问和 DMA engine 行为
- [ ] DONE/AXIE output 不在第一批注册，但 TXU 仍可产生和上报
- [ ] XIP-only 寄存器 RAZ/WI，无 GPIO、region 1 和地址映射

### 🟦 Step 4.4：Standard Flash 与收敛

- [ ] Flash 挂到 `dw_ssi[2]` / SPI-OPI/FMC / SDK `spi0` 的 CS0
- [ ] Standard TX-only 与 EEPROM-read 的 JEDEC ID/固定地址读取通过
- [ ] 通用 qtest、K230 SSI qtest、K230 WDT qtest 全过
- [ ] 公共头文件独立包含通过
- [ ] 通用 SSI 无 K230 依赖
- [ ] VMState 只保存已实现的 Standard PIO/IRQ 状态，不预埋 enhanced/IDMA/XIP 字段
- [ ] `git diff --check` 与相关 checkpatch 通过
- [ ] 每个小目标的未来 patch 归属明确

---

## 🟪 15. 第一批上游 Series 与后续提交策略

### 🟦 15.1 第一批 Series 的目标

第一批上游 series 不追求一次完成 K230 SPI Flash 启动链，而是先提交一个边界清晰、可独立 review 的基础控制器：

```text
通用 DesignWare SSI
  ├── Standard single-line SPI PIO
  ├── TX/RX FIFO
  ├── 基础中断控制器
  └── 可配置实例参数
            ↓
K230 machine
  ├── 创建三个 SSI 实例
  ├── 显式设置基础 profile
  ├── 映射控制器 MMIO
  ├── 把 7 路基础 IRQ 输出连接到 PLIC
  └── 挂接 Standard 1-1-1 SPI NOR
```

这批 series 的核心 review 问题有三个：

1. `dw_ssi.c` 是否已经成为不依赖 K230 类型、地址和 HI_SYS 的通用设备模型；
2. 未实现寄存器是否使用稳定且一致的 unsupported/RAZ-WI 语义；
3. K230 是否只负责实例配置、地址映射、PLIC 接线和 Flash 挂接。

第一批包含 Standard 1-1-1 SPI NOR 挂接，但不以 SDK U-Boot 完整启动作为合入条件，因为现有启动链还会使用 enhanced SPI 和 IDMA。完成标准是通用寄存器/FIFO/PIO/IRQ/unsupported qtest、K230 实例/PLIC 集成 qtest 和 Standard Flash 读取测试全部通过。

### 🟦 15.2 第一批包含和不包含的内容

| 类别 | 第一批包含 | 第一批不包含 |
|---|---|---|
| 通用设备 | `TYPE_DW_SSI`、SSI bus、控制器 region 0、CS outputs、realize/finalize、reset、基础 VMState | K230 wrapper、HI_SYS 指针、K230 地址常量 |
| 配置 | `num-cs`、`fifo-depth`、`imr-reset` | enhanced、DMA/IDMA、XIP 配置 |
| 数据路径 | Standard single-line SPI，四种 TMOD、frame width、loopback、FIFO level/status、Standard 1-1-1 NOR | Dual/Quad/Octal、enhanced command、IDMA、XIP transaction |
| IRQ | TXE、TXO、RXF、RXO、TXU、RXU、MST 的基础语义和 K230 PLIC 路由 | DONE/AXIE 的产生和清除逻辑 |
| 资源 | 控制器 MMIO、SSI bus、CS、7 路基础 IRQ、Flash attachment | DONE/AXIE、XIP region、`xip-enable` GPIO |
| 测试 | 通用寄存器/PIO/IRQ/unsupported，K230 三实例、PLIC、Standard Flash | enhanced、IDMA、HI_SYS、XIP 正路径 |

`TXU` 属于基础 TX FIFO underflow，必须随第一批 IRQ 一起实现。第一批只注册并路由七路基础输出；DONE/AXIE 的寄存器位固定无效，物理 output 随 IDMA series 引入。

### 🟦 15.3 “寄存器占位但不提供功能”的精确定义

第一批可以保留完整控制器 MMIO aperture，但不能通过一组半实现的寄存器暗示 enhanced、IDMA 或 XIP 已受支持。占位采用以下规则：

1. **地址空间占位**：region 0 大小保持稳定；未实现 offset 统一 RAZ/WI，不为每个未来寄存器增加空 case；
2. **基础寄存器真实实现**：Standard PIO、FIFO、状态、基础 IRQ、IDR/VERSION 所依赖的寄存器必须具有明确 reset、write mask 和副作用；
3. **地址与实现分离**：K230 internal-AXI DMA 和其他扩展 offset 可以存在于寄存器目录中；第一批 read 0、write ignore，不保存状态；external DMA layout 不进入当前路线；
4. **有证据才暴露非零契约**：如果 firmware 枚举确实需要读取某个扩展寄存器 reset，可以单独实现只读/reset 语义并写测试，但 commit message 必须声明“寄存器可见不等于数据路径已实现”；
5. **配置随功能出现**：第一批不增加恒 false property、profile 字段或内部开关；后续 series 必须随正路径、资源、IRQ、迁移和测试一起引入配置；
6. **XIP-only 保持专用**：`XIP_MODE_BITS` 第一批 RAZ/WI，后续 IDMA series 也不得把它改为 shared 或重新用于 1-4-4 mode byte。

因此，“先占位”精确表达为：**地址可以被识别，但未实现寄存器不保存状态、不产生数据路径或资源；read 0、write ignore。**

### 🟦 15.4 第一批仍需具备的通用基础

除了 PIO 和 IRQ，第一批还需要以下基础，否则“通用 DW SSI”拆分仍不完整：

- `DwSsiConfig`、realize 前配置校验和完整 `DwSsiState` Standard 运行状态；
- FIFO、CS 动态资源的明确创建和销毁生命周期；
- 基础 VMState：寄存器、FIFO、PIO phase、remaining frames、IRQ latch、active CS，以及 `num-cs`、`fifo-depth`、`imr-reset` equality；
- 固定且有文档说明的 SSI bus、CS output 和 IRQ output 接口；
- 通用 qtest 载体，用于证明 PIO/IRQ 语义不依赖 K230 machine；
- K230 集成 qtest 只验证三实例 profile、地址映射、PLIC source 和实例隔离，避免与通用测试重复；
- Standard 1-1-1 Flash 测试提供真实 SSI consumer；
- 非法 `num-cs`、`fifo-depth` 等配置在 realize 阶段返回清晰错误。

第一批不需要 trace events 或启动镜像。trace 可以随对应功能加入，避免单独增加只为调试服务的 review 面积。

### 🟦 15.5 第一批 Commit 顺序

第一批保持五个可编译、可回归的提交，每个提交只增加一类职责：

1. `hw/ssi: Add a Synopsys DesignWare SSI standard PIO controller`
   - 建立通用 QOM 类型、配置、Standard 寄存器、动态 FIFO、四种 TMOD、PIO、reset、VMState、unsupported 语义和通用 qtest；
2. `hw/ssi: Add DesignWare SSI standard interrupt support`
   - 实现基础 raw/masked/clear 语义、threshold IRQ 和通用 IRQ qtest；
3. `hw/riscv/k230: Instantiate DesignWare SSI controllers`
   - 创建三个实例，设置第一批所需 profile，映射 region 0，并增加实例测试；
4. `hw/riscv/k230: Route SSI interrupts to the PLIC`
   - 完成三实例 PLIC 接线和路由隔离 qtest。
5. `hw/riscv/k230: Attach a standard SPI flash to the K230 SSI`
   - 保留 `spi-flash` property，挂接 M25P80-compatible NOR，并增加 Standard 1-1-1 读取测试。

测试应跟随首次实现该行为的 commit，不把所有测试集中到最后一个 patch。每个 commit 完成后至少执行增量构建、对应定向 qtest 和 `git diff --check`；完成整个 series 后再运行完整通用/K230 qtest。

本地开发 commit 可以按 Step 4.0、4.1、4.2 等小目标逐步积累；发送上游前再重组到上述五个职责提交。Step 4.0 当前提交属于本地纠错历史：第一批 series 不包含 enhanced/IDMA，因此不需要单独携带这个修复 patch；后续 enhanced/IDMA series 应从首次出现开始就是正确实现，不能先引入错误 mode phase 再补修复。

### 🟨 15.6 后续 Series

第一批合入或架构 review 基本稳定后，再顺序发送：

1. enhanced SPI 数据路径，同时引入 `max-lines`、`spi-ctrlr0-reset`、command state 和 Dual/Quad SDR；
2. internal IDMA：internal-AXI 寄存器、guest memory 搬运、DONE/AXIE；external DMA 不在当前路线；
3. K230 HI_SYS + optional XIP region、GPIO 和 XIP transaction。

IDMA series 必须恢复 SDK 风格 `0xeb` 1-4-4 回归，并保持 `XIP_MODE_BITS` 为 XIP-only；dummy/mode 不得重新耦合。

后续 series 不应同时并发发送，否则 reviewer 同一时间仍需理解完整三千余行改动，失去分批投稿的意义。

---

## 🟪 16. 第一批与完整 V2 patch 系列的关系

Step 4 的实现内容在第五步重组时按职责回填，不形成一个覆盖所有功能的“大配置 patch”：

| Step 4 内容 | 最终 patch 归属 |
|---|---|
| `DwSsiConfig`、完整 Standard state、寄存器/FIFO/PIO、配置校验、通用测试 machine | Patch 1：通用 Standard PIO 控制器 |
| 七路基础 IRQ | Patch 2：通用 IRQ |
| K230 三实例 profile、控制器 MMIO | Patch 3：K230 实例化通用控制器 |
| K230 七路 PLIC 路由 | Patch 4：PLIC 集成 |
| Standard 1-1-1 Flash 挂接 | Patch 5：K230 Flash integration |
| 引入 `has-enhanced-spi` property 并实现正路径 | 后续 enhanced SPI series |
| 引入 `has-idma` property 并实现 engine | 后续 optional IDMA series |
| 引入 `has-xip`/`xip-window-size` property、增加 region/GPIO/`0xc0000000` | 后续 optional XIP series |

每个最终 patch 必须包含自己需要的 property、测试和 machine 配置；后续 enhanced/IDMA/XIP series 不得修复第一批已经引入的不可编译、不可 realize 或不完整 Standard 状态。

---

## ⬜ 17. 参考入口

- [V2 决策记录](k230-spi-qspi-review-v2-decision-notes.md)
- [V2 实施路线](k230-spi-qspi-v2-implementation-plan.md)
- [Step 3 HI_SYS 解耦计划](k230-spi-qspi-v2-step3-hi-sys-decoupling.md)
- [K230 TRM 原始文本](../reference/K230_Technical_Reference_Manual_V0.3.1_20241118.txt)
- [TRM 12.3 中文对照](../spi/reference/k230-trm-12.3-spi-cn.md)
- [寄存器审计](../spi/k230-spi-qspi-register-audit.md)
- [当前 DW SSI 头文件](../../../qemu-camp-2026-k230/include/hw/ssi/dw_ssi.h)
- [当前 DW SSI 模型](../../../qemu-camp-2026-k230/hw/ssi/dw_ssi.c)
- [当前 K230 machine](../../../qemu-camp-2026-k230/hw/riscv/k230.c)
- [当前 K230 SSI qtest](../../../qemu-camp-2026-k230/tests/qtest/k230-dw-ssi-test.c)
- [QEMU DesignWare I2C 先例](../../../qemu-camp-2026-k230/hw/i2c/designware_i2c.c)
- [QEMU DesignWare I3C 先例](../../../qemu-camp-2026-k230/hw/i3c/dw-i3c.c)
- [QEMU Xilinx QSPI 线性窗口先例](../../../qemu-camp-2026-k230/hw/ssi/xilinx_spips.c)
- [K230 U-Boot DTS](../../../k230_sdk/src/little/uboot/arch/riscv/dts/k230.dtsi)
- [K230 Linux DTS](../../../k230_sdk/src/little/linux/arch/riscv/boot/dts/kendryte/k230.dtsi)
- [K230 U-Boot DesignWare SPI 驱动](../../../k230_sdk/src/little/uboot/drivers/spi/designware_spi.c)
- [K230 RT-Smart SPI 驱动](../../../k230_sdk/src/big/rt-smart/kernel/bsp/maix3/board/interdrv/spi/drv_spi.c)

---

## ⬜ 18. 与总决策文档的关系

本文只细化 V2 第四步。通用层与 K230 层边界、XIP aperture 归属、默认不增加 wrapper，以及配置必须有真实消费者的证据标准仍以 [V2 决策记录](k230-spi-qspi-review-v2-decision-notes.md) 为准。

若实施中发现 K230 某个 reset、寄存器或功能消费者与本矩阵冲突，先回到 TRM/SDK 定位证据并更新决策记录，再调整 profile 或后续 series；不得在 `dw_ssi.c` 中重新加入 K230 分支或地址常量。
