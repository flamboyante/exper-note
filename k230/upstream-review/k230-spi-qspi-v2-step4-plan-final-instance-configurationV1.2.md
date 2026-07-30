# K230 V2 第四步 Plan Final V1.2：实例配置与第一批 capability 边界

首次记录：2026-07-29

最终修订：2026-07-30（保留内部 capability 骨架，公共 capability property 随对应功能 series 引入）

适用代码检查点：`qemu-camp-2026-k230/` 分支 `k230-V2-patch-spi`，commit `c689ac865f`

文档状态：Step 4.0 已完成源码实施和 12 项 K230 SSI qtest；Step 4.1 至 Step 4.4 仍是待实施计划

本文是 [V2 实施路线](k230-spi-qspi-v2-implementation-plan.md) 第四步“引入实例配置”的唯一执行计划。Plan A、Plan B、Plan C、原 Plan Final 及其 planning handoff 只保留为历史推演材料，不再作为实施入口。架构边界以 [V2 决策记录](k230-spi-qspi-review-v2-decision-notes.md) 为准：保留单一通用 `TYPE_DW_SSI` / `DwSsiState`，K230 machine 通过 properties 配置三个实例，不增加 `TYPE_K230_DW_SSI` wrapper。

> **上游投稿边界说明（2026-07-30）**：V1.2 直接以第一批上游 series 为实施终态。第一批只提交通用 DW SSI 的标准寄存器版图、Standard SPI PIO、基础 IRQ、DMA 配置寄存器兼容性、K230 三实例与 PLIC 集成，以及独立的 Standard 1-1-1 SPI NOR 挂接；enhanced SPI、IDMA engine、HI_SYS 和 XIP 分批后送。`DwSsiConfig` 保留内部 capability 位图并固定为全零，但第一批不暴露 `has-enhanced-spi`、`has-idma`、`has-xip` 和 `xip-window-size` property；每项 public property 必须与对应正路径在同一后续 series 引入。

---

## 摘要

Step 4 先修正当前模型把普通 enhanced/IDMA 事务错误连接到 XIP mode bits 的问题，再将 K230 固定参数改为 realize 前确定的实例配置。第一批把寄存器版图与功能 capability 分开：`dma-register-layout` 决定 DMA 寄存器存在性和地址解释；内部 capability 位固定为零，保证 enhanced、IDMA engine 和 XIP 不进入数据路径，也不提前形成公共配置接口。

实施分为五个可独立验证的小目标：

1. Step 4.0：普通 enhanced/IDMA 只执行 instruction → address → dummy → data，XIP mode 只留在真正的 XIP transaction；
2. Step 4.1：建立 `DwSsiConfig`、第一批所需 properties、配置校验、动态 FIFO 和最小迁移一致性检查；
3. Step 4.2：在 K230 machine 中显式应用 QSPI0、QSPI1、SPI-OPI/FMC 三个 profile，消除 reset 值和 XIP region 的 K230 硬编码推断；
4. Step 4.3：实现第一批 capability 关闭语义契约：内部位固定为零，对应寄存器、数据路径、IRQ 和资源均保持关闭；
5. Step 4.4：完成通用模型、K230 集成、公共头文件、迁移状态和未来 patch 归属检查。

本步不实现 enhanced、DDR、RXDS、Octal、XIP、internal DMA engine 或 external DMA request 行为。`max-lines=4/4/8` 只描述三个实例的物理线宽上限；第一批数据路径只执行 Standard single-line SPI。

---

## 1. 当前状态分析

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
| capability | enhanced SPI、IDMA、XIP 均无存在性开关 |
| VMState | 保存完整寄存器数组、FIFO、enhanced、IDMA、XIP enable 等运行时状态 |
| qtest | `k230-dw-ssi-test` 共 12 项；Step 4.0 新增 enhanced/XIP 隔离和 IDMA 1-4-4 回归，仍缺第一批 capability 关闭语义覆盖 |

### 1.2 当前硬编码点清单

| 编号 | 当前位置 | 硬编码或耦合 | Step 4 动作 |
|---|---|---|---|
| H1 | `include/hw/ssi/dw_ssi.h:28` | `DW_SSI_XIP_WINDOW_SIZE = 0x08000000` | 第一批删除固定 XIP window；`xip-window-size` property 随 XIP series 引入 |
| H2 | `include/hw/ssi/dw_ssi.h:94-95` | `num_cs`、`max_lines` 分散在运行时状态中 | 收入集中 `DwSsiConfig` |
| H3 | `hw/ssi/dw_ssi.c:27` | FIFO 固定 256 | 改为 `fifo-depth` |
| H4 | `hw/ssi/dw_ssi.c:31-37` | `IMR`、IDR、VERSION、AXI burst 固定 | reset 从实例配置读取 |
| H5 | `hw/ssi/dw_ssi.c:33-34` | 仅按 SPI/FMC 固定两个 `SPI_CTRLR0` reset | 改为 `spi-ctrlr0-reset` |
| H6 | `hw/ssi/dw_ssi.c:425-430` | `SR.TFNF/RFF` 与 256 比较 | 改为 `cfg.fifo_depth` |
| H7 | 原 `hw/ssi/dw_ssi.c:537-584`、`949-955`、`1021-1043` | 普通 enhanced/IDMA 曾错误读取 `XIP_MODE_BITS`，PIO 曾包含 XIP mode phase | Step 4.0 已完成：普通事务只保留 instruction/address/dummy/data |
| H8 | `hw/ssi/dw_ssi.c:813-984` | DMA 寄存器存在性和 IDMA engine 被同一逻辑隐式绑定 | 增加 `dma-register-layout` 决定版图；第一批删除 engine、guest memory 和 DONE/AXIE 行为；TXU 保持基础 FIFO IRQ |
| H9 | `hw/ssi/dw_ssi.c:1201-1535` | read/write switch 无 layout/capability 分层 | 增加独立的寄存器 layout 辅助函数；内部 capability 位只参与第一批关闭语义 |
| H10 | `hw/ssi/dw_ssi.c:1578-1585` | reset 使用 K230 值，并用 `max_lines == 8` 推断 FMC | 全部从 `cfg` 读取，删除推断 |
| H11 | `hw/ssi/dw_ssi.c:1677-1689` | 所有实例创建 XIP region 和 256 深度 FIFO | 第一批删除 XIP region；FIFO 按 `fifo-depth` 在 `realize()` 创建 |
| H12 | `hw/riscv/k230.c:246-251` | machine 只设置 CS 和线宽 | 为三个实例显式设置完整 profile |
| H13 | `hw/riscv/k230.c:293-294` | 无条件映射 `dw_ssi[2]` region 1 | 第一批删除 region 1 映射；随 XIP series 恢复 capability-controlled 映射 |
| H14 | `tests/qtest/k230-dw-ssi-test.c:492-528` | 三实例只验证 `SPI_CTRLR0`/SER，IDR/VERSION/IMR 只抽查 spi0 | 扩充为三实例完整 reset profile |

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

Step 4 后应改为：第一批固定接口在 `instance_init` 注册，依赖已公开 property 的 FIFO 和 CS 资源在 `realize()` 校验后创建。XIP GPIO 和第二个 MMIO region 不在第一批创建；对应 property 和资源随 XIP series 同时引入。

---

## 2. 证据与 K230 三实例配置矩阵

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
| `max-lines` | 4 | 4 | 8 |
| `component-id` | `0xa1b2c3d5` | `0xa1b2c3d5` | `0xa1b2c3d5` |
| `version-id` | `0x3130332a` | `0x3130332a` | `0x3130332a` |
| `imr-reset` | `0x0000001f` | `0x0000001f` | `0x0000003f` |
| `axiawlen-reset` | `0x00000700` | `0x00000700` | `0x00000000` |
| `axiarlen-reset` | `0x00000700` | `0x00000700` | `0x00000000` |
| `spi-ctrlr0-reset` | `0x04000200` | `0x04000200` | `0x28000200` |
| `dma-register-layout` | `internal-axi` | `internal-axi` | `internal-axi` |
| 内部 capability 位图 | `0` | `0` | `0` |
| 控制器 MMIO | `0x91582000` | `0x91583000` | `0x91584000` |
| XIP MMIO | 不创建/不映射 | 不创建/不映射 | 不创建/不映射 |

三实例内部 capability 位图固定为 `0`，因此第一批不创建第二个 MMIO region，也不映射 `0xc0000000`。这不等于共享 `SPI_CTRLR0` 必须整体读零：QSPI reset `0x04000200` 和 FMC reset `0x28000200` 仍作为 profile 契约可见；第一批只阻止扩展字段进入 enhanced/DDR/XIP 数据路径。

### 2.4 QEMU 主线组织先例

| 主线代码 | 可借鉴点 | 本计划用法 |
|---|---|---|
| `include/hw/i3c/dw-i3c.h` | 状态内集中 `cfg` 结构 | `DwSsiState` 内增加 `DwSsiConfig cfg` |
| `hw/i3c/dw-i3c.c:1806-1824` | 在 `realize()` 按 property 创建 FIFO/MMIO | 第一批动态创建 SSI FIFO；可选 XIP region 留给后续 series |
| `hw/i3c/dw-i3c.c:1782-1803` | reset 从 `cfg` 填充寄存器字段 | DW SSI reset 从实例 profile 读取 |
| `hw/i3c/dw-i3c.c:1854-1872` | properties 直接写入 `cfg` | 使用 kebab-case QOM properties |
| `hw/ssi/xilinx_spips.c:1372-1388` | 控制器在 realize 中增加线性 Flash region | 留作后续 XIP property 与资源实现参考 |
| `hw/arm/xlnx-zynqmp.c:881-885` | SoC 负责映射控制器 region 与 LQSPI region | 留作后续 XIP series 中 K230 映射 `0xc0000000` 的参考 |
| `hw/ssi/xilinx_spips.c:1397-1401` | 非法配置通过 `error_setg()` 拒绝 realize | 集中校验 property 范围与组合 |

本计划不新增 wrapper。第一批删除现有固定 XIP region；后续 XIP series 再按“通用模型实现 transaction/region、SoC 决定窗口大小与地址”的边界恢复可选 region。

### 2.5 已确认决策与实施假设

| 项目 | 结论 |
|---|---|
| 配置组织 | 使用单一 `DwSsiConfig` 和 kebab-case QOM properties；不增加 K230 wrapper |
| 默认值 | 内部 capability 位图固定为 `0`，`dma-register-layout` 默认 `none`；K230 显式选择 `internal-axi` 和全部实例 reset |
| capability 顺序 | 后续按 enhanced SPI → IDMA → XIP 分 series 开放；每项 public property 与正路径实现和测试在同一 patch 落地 |
| 负路径载体 | 使用独立 `dw-ssi-test` machine，不向 K230 产品 machine 增加测试入口 |
| XIP 归属 | 第一批不暴露 XIP property；XIP transaction、GPIO、可选 region 和 property 全部随 XIP series 后送 |
| IRQ / VMState | TXU 属于基础 FIFO IRQ，DONE/AXIE 第一批不产生；迁移只对 FIFO 深度和 DMA register layout 做 equality |
| Octal 边界 | `max-lines=8` 只表达 FMC 线宽上限，本步不实现 Octal/DDR/RXDS 数据路径 |
| 代码检查点 | Step 4.0 完成状态基于 `k230-V2-patch-spi` 的 `c689ac865f`；后续行号、QOM child 名称和默认行为若随 HEAD 变化，先重新定位，不直接套用旧行号 |

除上述检查点时效性外，本文没有依赖未确认 Databook 内容的阻塞假设。若新证据改变 K230 profile，先更新决策记录和配置矩阵，再实施代码。

---

## 3. 精确配置方案

### 3.1 `DwSsiConfig`

**文件：`include/hw/ssi/dw_ssi.h`**

新增集中配置结构，并把现有 `num_cs`、`max_lines` 移入其中：

```c
typedef struct DwSsiConfig {
    uint32_t num_cs;
    uint32_t fifo_depth;
    uint32_t max_lines;

    uint32_t component_id;
    uint32_t version_id;
    uint32_t imr_reset;
    uint32_t axiawlen_reset;
    uint32_t axiarlen_reset;
    uint32_t spi_ctrlr0_reset;

    uint32_t dma_register_layout;
    uint32_t capabilities;
    uint64_t xip_window_size;
} DwSsiConfig;
```

DMA 寄存器版图使用独立枚举，不能复用后续 IDMA engine capability：

```c
typedef enum DwSsiDmaRegisterLayout {
    DW_SSI_DMA_LAYOUT_NONE,
    DW_SSI_DMA_LAYOUT_EXTERNAL,
    DW_SSI_DMA_LAYOUT_INTERNAL_AXI,
} DwSsiDmaRegisterLayout;
```

- `NONE`：DMA 配置寄存器不存在，相关 offset RAZ/WI；
- `EXTERNAL`：`0x050/0x054` 解释为 `DMATDLR/DMARDLR`；
- `INTERNAL_AXI`：`0x050/0x054` 解释为 `AXIAWLEN/AXIARLEN`，并暴露 K230 internal-AXI 配置寄存器。

layout 只决定寄存器存在性、字段 mask 和 readback；capability 只决定功能行为。第一批内部 IDMA capability 位为 0，`INTERNAL_AXI` 布局下的配置寄存器仍可按 mask 保存和读回，但绝不触发 DMA engine。

第一批保留内部 `uint32_t` capability 位图和集中判断 helper，但不暴露 capability property，也不把固定为零的位图加入迁移 equality：

```c
#define DW_SSI_CAP_ENHANCED_SPI BIT(0)
#define DW_SSI_CAP_IDMA         BIT(1)
#define DW_SSI_CAP_XIP          BIT(2)
#define DW_SSI_CAP_VALID_MASK \
    (DW_SSI_CAP_ENHANCED_SPI | DW_SSI_CAP_IDMA | DW_SSI_CAP_XIP)

static bool dw_ssi_has_capability(const DwSsiState *s, uint32_t cap)
{
    return (s->cfg.capabilities & cap) != 0;
}
```

`DwSsiState` 中保留运行时状态，配置只保存在 `cfg`：

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
    bool sleep_status;
};
```

properties 只表达不可变硬件配置。`SSIENR`、`SER`、FIFO level、Standard PIO phase、IRQ latch 等 guest 运行时状态不得做成 property。enhanced command、IDMA 和 XIP 运行时字段不在第一批结构体中预留。

### 3.2 property 集合与兼容默认值

**文件：`hw/ssi/dw_ssi.c`**

```c
static const Property dw_ssi_properties[] = {
    DEFINE_PROP_UINT32("num-cs", DwSsiState, cfg.num_cs, 1),
    DEFINE_PROP_UINT32("fifo-depth", DwSsiState, cfg.fifo_depth, 256),
    DEFINE_PROP_UINT32("max-lines", DwSsiState, cfg.max_lines, 1),

    DEFINE_PROP_UINT32("component-id", DwSsiState,
                       cfg.component_id, 0xa1b2c3d5),
    DEFINE_PROP_UINT32("version-id", DwSsiState,
                       cfg.version_id, 0x3130332a),
    DEFINE_PROP_UINT32("imr-reset", DwSsiState,
                       cfg.imr_reset, 0x0000003f),
    DEFINE_PROP_UINT32("axiawlen-reset", DwSsiState,
                       cfg.axiawlen_reset, 0),
    DEFINE_PROP_UINT32("axiarlen-reset", DwSsiState,
                       cfg.axiarlen_reset, 0),
    DEFINE_PROP_UINT32("spi-ctrlr0-reset", DwSsiState,
                       cfg.spi_ctrlr0_reset, 0x04000200),

    DEFINE_PROP_UINT32("dma-register-layout", DwSsiState,
                       cfg.dma_register_layout,
                       DW_SSI_DMA_LAYOUT_NONE),
};
```

通用默认实例只支持 Standard PIO/IRQ，不具有 DMA 寄存器布局和任何扩展 capability。K230 machine 显式设置第一批已公开的全部 property；`cfg.capabilities` 和 `cfg.xip_window_size` 保持 QOM 零初始化。后续功能 series 必须在同一 patch 中增加对应 public property、正路径、关闭路径和测试，不能提前暴露只能取单一值的接口。

### 3.3 realize 校验

新增单一入口 `dw_ssi_validate_config()`，所有资源创建前调用：

```c
static bool dw_ssi_validate_config(DwSsiState *s, Error **errp)
{
    DeviceState *dev = DEVICE(s);

    if (s->cfg.capabilities & ~DW_SSI_CAP_VALID_MASK) {
        error_setg(errp, "%s: capabilities contains unsupported bits",
                   dev->canonical_path);
        return false;
    }

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

    if (s->cfg.max_lines != 1 && s->cfg.max_lines != 2 &&
        s->cfg.max_lines != 4 && s->cfg.max_lines != 8) {
        error_setg(errp, "%s: max-lines must be 1, 2, 4, or 8",
                   dev->canonical_path);
        return false;
    }

    if (s->cfg.dma_register_layout >
        DW_SSI_DMA_LAYOUT_INTERNAL_AXI) {
        error_setg(errp,
                   "%s: invalid dma-register-layout",
                   dev->canonical_path);
        return false;
    }

    if (s->cfg.dma_register_layout != DW_SSI_DMA_LAYOUT_INTERNAL_AXI &&
        (s->cfg.axiawlen_reset != 0 || s->cfg.axiarlen_reset != 0)) {
        error_setg(errp,
                   "%s: AXI burst reset requires internal-axi DMA layout",
                   dev->canonical_path);
        return false;
    }

    if (s->cfg.imr_reset & ~DW_SSI_IRQ_VALID_MASK) {
        error_setg(errp, "%s: imr-reset contains unsupported bits",
                   dev->canonical_path);
        return false;
    }

    if (s->cfg.axiawlen_reset & ~DW_SSI_AXIAWLEN_WRITABLE_MASK ||
        s->cfg.axiarlen_reset & ~DW_SSI_AXIARLEN_WRITABLE_MASK ||
        s->cfg.spi_ctrlr0_reset & ~DW_SSI_SPI_CTRLR0_WRITABLE_MASK) {
        error_setg(errp, "%s: reset property contains unsupported bits",
                   dev->canonical_path);
        return false;
    }

    return true;
}
```

第一批不检查 `max-lines` 与 enhanced engine 的组合。`max-lines` 是物理线宽综合参数，和当前是否实现 enhanced engine 正交；第一批通过内部 capability 位固定为零、动态清除 `CTRLR0.SPI_FRF` 写入并只保留 Standard transfer dispatch 来关闭 enhanced 行为。

IDMA/XIP 的组合校验也不在第一批预埋。后续功能 series 引入对应 public property 时，再在同一 patch 中加入当时真实实现所需的依赖校验。

### 3.4 property 依赖的资源创建

`instance_init` 保留第一批固定接口：SSI bus、控制器 MMIO 和 9 路 IRQ。DONE/AXIE 两路先注册但恒低，以固定后续 IDMA series 的 sysbus IRQ 拓扑。`xip-enable` GPIO 和第二个 MMIO region 不在第一批创建，随 XIP series 一起加入。

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

删除用 `max_lines == 8` 推断 FMC 的逻辑：

```c
s->regs[R_IMR] = s->cfg.imr_reset;
s->regs[R_AXIAWLEN] =
    s->cfg.dma_register_layout == DW_SSI_DMA_LAYOUT_INTERNAL_AXI ?
    s->cfg.axiawlen_reset : 0;
s->regs[R_AXIARLEN] =
    s->cfg.dma_register_layout == DW_SSI_DMA_LAYOUT_INTERNAL_AXI ?
    s->cfg.axiarlen_reset : 0;
s->regs[R_IDR] = s->cfg.component_id;
s->regs[R_SSIC_VERSION_ID] = s->cfg.version_id;
s->regs[R_SPI_CTRLR0] = s->cfg.spi_ctrlr0_reset;
```

external layout 下 `0x050/0x054` 对应 DMATDLR/DMARDLR，reset 固定为 0；internal-axi layout 才使用 K230 profile 的 AXI burst reset。`SPI_CTRLR0` 始终保留 profile reset，但第一批不允许其扩展字段驱动数据路径。reset 末尾清除 DONE/AXIE latch，保证两路输出恒低。

---

## 4. 寄存器 layout 与 capability 分层设计

### 4.1 两类门控不能共用

禁止用单一 `dw_ssi_reg_capability()` 同时决定“寄存器是否存在”和“引擎是否运行”。第一批必须分成两层：

1. `dma-register-layout`：决定 DMA 寄存器版图、offset 解释、字段 mask、reset 和存储 readback；
2. capability：决定 enhanced/IDMA/XIP 数据路径、guest memory 访问、事件和额外资源。

DMA layout 辅助函数示意：

```c
static bool dw_ssi_dma_reg_present(const DwSsiState *s, hwaddr addr)
{
    switch (s->cfg.dma_register_layout) {
    case DW_SSI_DMA_LAYOUT_NONE:
        return false;
    case DW_SSI_DMA_LAYOUT_EXTERNAL:
        return addr == A_DMACR || addr == 0x050 || addr == 0x054;
    case DW_SSI_DMA_LAYOUT_INTERNAL_AXI:
        switch (addr) {
        case A_DMACR:
        case A_AXIAWLEN:
        case A_AXIARLEN:
        case A_SPIDR:
        case A_SPIAR:
        case A_AXIAR0:
        case A_AXIAR1:
        case A_AXIECR:
        case A_DONECR:
            return true;
        default:
            return false;
        }
    default:
        return false;
    }
}
```

`0x050/0x054` 在 C 代码中只按当前 layout 选择一个字段视图，不能生成两个同时可访问的地址。external layout 的 read/write callback 使用 DMATDLR/DMARDLR 字段，internal-axi layout 使用 AXIAWLEN/AXIARLEN 字段。

其他扩展寄存器单独分类：

```text
始终存在/profile-visible：SPI_CTRLR0
enhanced-only：DDR_DRIVE_EDGE
XIP-only：XIP_MODE_BITS、XIP_INCR_INST、XIP_WRAP_INST、XIP_CTRL、...
```

第一批 `SPI_CTRLR0` 读回 profile reset；`DDR_DRIVE_EDGE` 和 XIP-only 组 RAZ/WI。layout-visible DMA 配置寄存器按 mask 保存和读回，不因 IDMA engine 尚未实现而消失。

### 4.2 第一批精确语义

| 控制项 | 寄存器语义 | IRQ | 状态与数据路径 | realize 资源 |
|---|---|---|---|---|
| enhanced capability 位为 0 | `CTRLR0.SPI_FRF` 非零写入忽略或清零；`SPI_CTRLR0` 保留 profile reset；纯 enhanced 寄存器 RAZ/WI | 无 enhanced IRQ | 只进入 Standard PIO/TMOD | 无额外资源 |
| `dma-register-layout=external` | `DMACR/DMATDLR/DMARDLR` 按字段 mask 保存读回 | 不产生 DMA request | 无 DMA 数据通路 | 无额外资源 |
| `dma-register-layout=internal-axi`、IDMA capability 位为 0 | `DMACR/AXIAWLEN/AXIARLEN/SPIDR/SPIAR/AXIAR0/1` 按 mask 保存读回；`AXIECR/DONECR` 读 0 | DONE/AXIE 位无效；TXU 仍有效 | 不访问 guest memory，DR 保持 PIO FIFO 语义 | 无额外资源 |
| XIP capability 位为 0 | XIP-only 寄存器 RAZ/WI；共享 `SPI_CTRLR0` reset 仍可见 | XRXO/SPITE 不暴露 | 无 XIP transaction | 无 GPIO、无第二 region |

### 4.3 capability 关闭语义契约

第一批内部 capability 位图固定为 `0`，关闭语义必须落实到寄存器、数据路径、IRQ 和资源四个层面：

1. enhanced 位关闭：`CTRLR0.SPI_FRF` 保持 Standard，纯 enhanced 寄存器 RAZ/WI，只执行四种 Standard TMOD；
2. IDMA 位关闭：DMA layout-visible 寄存器仍按 mask 存储，但不访问 guest memory，不产生 DONE/AXIE，DR 保持 PIO FIFO 语义；
3. XIP 位关闭：XIP-only 寄存器 RAZ/WI，不创建 XIP GPIO、第二个 MemoryRegion 或 K230 aperture 映射；
4. system reset 后上述关闭语义保持不变。

后续 series 必须在同一个 patch 中加入对应 public property、寄存器正路径、数据路径、IRQ/资源行为和 qtest。不能先增加 property，再依赖后续 patch 补功能。

### 4.4 IDMA 后续 series 的防回归约束

第一批不保存 `idma_completed_frames` 等 IDMA 运行时状态，也不访问 guest memory。DONE/AXIE 两路输出固定注册但保持低电平；TXU 继续属于第一批基础 FIFO IRQ。

IDMA series 必须继续满足 Step 4.0 已验证的事务边界：普通 enhanced/IDMA 1-4-4 只执行：

```text
instruction → address → dummy → data
```

`XIP_MODE_BITS` 继续属于 XIP-only 寄存器组，不得改成 shared，也不得重新加入 `TRANS_TYPE=1 && WAIT_CYCLES>=2` 的 mode-byte 特判。SDK 风格 `0xeb` IDMA 回归必须随 IDMA series 一起恢复，证明 dummy 只由 `WAIT_CYCLES` 表达。

### 4.5 XIP 后续资源边界

第一批内部 XIP capability 位和 `cfg.xip_window_size` 均保持为 0，不创建 `xip-enable` GPIO 和第二个 MemoryRegion，K230 不映射 `0xc0000000`。XIP series 才同时引入：

- `has-xip`、`xip-window-size` public property 及其 realize 正路径和组合校验；
- `xip-enable` GPIO；
- 第二个 MMIO region；
- K230 `0xc0000000` 映射；
- XIP transaction 和对应迁移状态。

以下旧式集中分组在 V1.2 中明确废除，不能继续实现：

```c
/* Wrong: layout-visible registers must not be hidden by IDMA capability. */
static unsigned int dw_ssi_reg_capability(hwaddr addr)
{
    switch (addr) {
    case A_AXIAWLEN:
    case A_AXIARLEN:
    case A_SPIDR:
    case A_SPIAR:
    case A_AXIAR0:
    case A_AXIAR1:
    case A_AXIECR:
    case A_DONECR:
        return DW_SSI_REG_CAP_IDMA; /* Do not do this. */
    }
}
```

---

## 5. K230 machine 精确改动

### 5.1 profile 数据结构

**文件：`hw/riscv/k230.c`**

在 `k230_ssi_routes[]` 附近增加按 `ssi_index` 排列的 profile：

```c
typedef struct K230DwSsiProfile {
    uint32_t num_cs;
    uint32_t fifo_depth;
    uint32_t max_lines;
    uint32_t component_id;
    uint32_t version_id;
    uint32_t imr_reset;
    uint32_t axiawlen_reset;
    uint32_t axiarlen_reset;
    uint32_t spi_ctrlr0_reset;
    uint32_t dma_register_layout;
} K230DwSsiProfile;

static const K230DwSsiProfile k230_dw_ssi_profiles[] = {
    [0] = { /* QSPI0, SDK spi1 */
        .num_cs = 5,
        .fifo_depth = 256,
        .max_lines = 4,
        .component_id = 0xa1b2c3d5,
        .version_id = 0x3130332a,
        .imr_reset = 0x0000001f,
        .axiawlen_reset = 0x00000700,
        .axiarlen_reset = 0x00000700,
        .spi_ctrlr0_reset = 0x04000200,
        .dma_register_layout = DW_SSI_DMA_LAYOUT_INTERNAL_AXI,
    },
    [1] = { /* QSPI1, SDK spi2: values identical to QSPI0 */
        .num_cs = 5,
        .fifo_depth = 256,
        .max_lines = 4,
        .component_id = 0xa1b2c3d5,
        .version_id = 0x3130332a,
        .imr_reset = 0x0000001f,
        .axiawlen_reset = 0x00000700,
        .axiarlen_reset = 0x00000700,
        .spi_ctrlr0_reset = 0x04000200,
        .dma_register_layout = DW_SSI_DMA_LAYOUT_INTERNAL_AXI,
    },
    [2] = { /* SPI-OPI/FMC, SDK spi0 */
        .num_cs = 1,
        .fifo_depth = 256,
        .max_lines = 8,
        .component_id = 0xa1b2c3d5,
        .version_id = 0x3130332a,
        .imr_reset = 0x0000003f,
        .axiawlen_reset = 0,
        .axiarlen_reset = 0,
        .spi_ctrlr0_reset = 0x28000200,
        .dma_register_layout = DW_SSI_DMA_LAYOUT_INTERNAL_AXI,
    },
};
```

QSPI0/QSPI1 值相同但仍分别列出，避免以后实例差异出现时依赖共享可变对象，也让 review 能直接对照三个硬件实例。

### 5.2 property 设置 helper

新增 `k230_configure_dw_ssi()`，只负责从 K230 profile 设置通用 property：

```c
static void k230_configure_dw_ssi(DwSsiState *ssi,
                                  const K230DwSsiProfile *profile)
{
    DeviceState *dev = DEVICE(ssi);

    qdev_prop_set_uint32(dev, "num-cs", profile->num_cs);
    qdev_prop_set_uint32(dev, "fifo-depth", profile->fifo_depth);
    qdev_prop_set_uint32(dev, "max-lines", profile->max_lines);
    qdev_prop_set_uint32(dev, "component-id", profile->component_id);
    qdev_prop_set_uint32(dev, "version-id", profile->version_id);
    qdev_prop_set_uint32(dev, "imr-reset", profile->imr_reset);
    qdev_prop_set_uint32(dev, "axiawlen-reset", profile->axiawlen_reset);
    qdev_prop_set_uint32(dev, "axiarlen-reset", profile->axiarlen_reset);
    qdev_prop_set_uint32(dev, "spi-ctrlr0-reset",
                         profile->spi_ctrlr0_reset);
    qdev_prop_set_uint32(dev, "dma-register-layout",
                         profile->dma_register_layout);
}
```

在三个 SSI realize 前循环调用，删除当前六个分散的 `num-cs` / `max-lines` 设置。内部 capability 位图和 `xip_window_size` 不进入 K230 profile，也不由 machine 直接写入；第一批统一依赖 QOM 零初始化。

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

三个 profile 均只有 region 0。第一批删除 K230 到 SSI 的 `xip-enable` GPIO 连接，不调用 `sysbus_mmio_map(..., 1, ...)`，也不把 `0xc0000000` 映射为 XIP aperture。

### 5.4 Standard 1-1-1 SPI Flash 挂接

最终选择 5.5-A：第一批保留 `spi-flash` machine property，并把 M25P80-compatible NOR 挂到选定 SSI bus/CS。该 patch 只验证 Standard 1-1-1 PIO 访问：

- 不使用 `SPI_FRF=Dual/Quad`；
- 不设置 `IDMAE`；
- 不创建或访问 XIP aperture；
- 不把 U-Boot enhanced/IDMA 启动作为第一批合入条件。

Flash 挂接必须是独立 patch，不能与通用 `dw_ssi.c` 寄存器模型混在一起。测试至少验证 JEDEC ID 或固定地址普通读，证明 K230 实例具备真实 SSI peripheral consumer。

---

## 6. capability 关闭路径测试载体决策

### 6.1 选择独立通用 qtest

K230 三实例在第一批均使用内部全零 capability 位图，可以覆盖产品 profile 的关闭行为；仍保留独立通用测试机，用于覆盖 `none/external/internal-axi` 三种 DMA layout、不同 FIFO 深度，以及 enhanced/IDMA/XIP 不可用时的寄存器、数据路径、IRQ 和资源语义，不向 K230 产品 machine 增加测试专用 property。

新增文件：

| 文件 | 职责 |
|---|---|
| `hw/ssi/dw_ssi-test.c` | `CONFIG_DW_SSI && CONFIG_TEST_DEVICES` 下注册无 CPU 的最小 `dw-ssi-test` machine，实例化一个 `TYPE_DW_SSI` |
| `tests/qtest/dw-ssi-test.c` | 通过 `-preconfig` + QMP `qom-set` 建立不同配置，验证 property、FIFO、DMA layout、RAZ/WI、基础 IRQ 和 capability 关闭路径 |
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

配置类通用测试都从 `-preconfig` 开始，通过 QMP 设置本用例需要的第一批 properties。有效配置执行 `x-exit-preconfig`，由 machine init realize 控制器并映射 region；非法配置直接对 child 设置 `realized=true` 并断言 QMP error。property 尚未实现时，setup `qom-set` 先失败；property 存在但行为或校验尚未实现时，后续断言失败，两种状态都能形成明确的 TDD 红灯且不会阻塞 QMP。capability 关闭路径不依赖 `qom-set`，在默认全零位图上直接验证 guest-visible 行为。

注册非法路径使用 `/dw-ssi/config/invalid/<case>`；例如 `num-cs-zero`、`fifo-depth-too-large`、`dma-layout-invalid` 和 `reset-mask-invalid`。每个 data case 列出属性数组；`error_text` 只省略设备 canonical path，保留核心错误文本。

典型配置：

```c
static const DwSsiTestProperty standard_only[] = {
    { "max-lines", 4 },
    { "dma-register-layout", DW_SSI_DMA_LAYOUT_INTERNAL_AXI },
};

static const DwSsiTestProperty fifo_depth_8[] = {
    { "fifo-depth", 8 },
};

static const DwSsiTestProperty invalid_dma_layout[] = {
    { "dma-register-layout", DW_SSI_DMA_LAYOUT_INTERNAL_AXI + 1 },
};
```

---

## 7. Step 4.0：解除普通 enhanced/IDMA 与 XIP 的错误耦合

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
- Step 4.0 完成前不得开始 capability 门控，以免用 `has-xip` 掩盖现有数据路径错误。

---

## 8. Step 4.1：建立第一批配置骨架

### 目标

引入配置表达、DMA register layout、动态 FIFO 和通用测试载体。通用默认实例从本步开始就是第一批契约：内部 capability 位图为 `0`、layout 为 `none`，不再保持当前 V2 中间态的 enhanced/IDMA/XIP 正路径。

### TDD 顺序

1. 新增 `dw-ssi-test` 最小 machine 和 qtest 骨架；
2. 先写 property 默认值、`fifo-depth=8`、三种 DMA layout 和非法取值测试；
3. 增加 `DwSsiConfig` 和 properties；
4. 增加 `dw_ssi_validate_config()`；
5. 把 FIFO 创建移到 realize，并替换所有固定容量判断；
6. reset 改从 `cfg` 和 layout 读取；
7. 运行通用定向 qtest 和仍属于第一批范围的 K230 Standard PIO/IRQ 回归。

### 失败断言

`/dw-ssi/config/fifo-depth` 使用 `fifo-depth=8`：

- SSI enable、SER 保持 0，连续写 DR 8 次，`TXFLR == 8`；
- 第 9 次写入不增加 level，锁存 TX overflow；
- `SR.TFNF == 0`；
- 写 `TXFTLR.TFT=8`、`TXFTLR.TXFTHR=8` 或 `RXFTLR.RFT=8` 不改变原值；
- system reset 后 `TXFLR == 0`、`SR.TFNF == 1`。

`/dw-ssi/config/invalid/<case>` 至少覆盖：

| 参数 | 非法值 | 预期错误 |
|---|---:|---|
| `num-cs` | 0、9 | `num-cs must be in range 1..8` |
| `fifo-depth` | 1、257 | `fifo-depth must be in range 2..256` |
| `max-lines` | 3 | `max-lines must be 1, 2, 4, or 8` |
| `dma-register-layout` | 超出 `none/external/internal-axi` | `invalid dma-register-layout` |
| `dma-register-layout=none/external` | 非零 `axiawlen-reset/axiarlen-reset` | `AXI burst reset requires internal-axi DMA layout` |
| `imr-reset` | `0x80000000` | `imr-reset contains unsupported bits` |
| `axiawlen-reset` / `axiarlen-reset` | `1` | `reset property contains unsupported bits` |
| `spi-ctrlr0-reset` | `0x80000000` | `reset property contains unsupported bits` |

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
- `max-lines=4/8` 的实例均可正常 realize，内部 capability 位图保持为 0；
- `none/external/internal-axi` 的寄存器视图和 readback 测试通过；
- 仍属于第一批范围的 K230 Standard PIO/IRQ 回归通过。

---

## 9. Step 4.2：应用 K230 三实例 profile

### 目标

证明同一 `TYPE_DW_SSI` 可以表达 QSPI0、QSPI1、FMC 的实例差异，通用模型不再通过 `max-lines` 或 K230 常量猜测 reset。

### TDD 顺序

1. 扩展 `K230SsiInstance` 测试表，先写三实例完整 expected profile；
2. 让 `test_register_contract()` 对每个实例读取 IMR、AXI burst、IDR、VERSION、`SPI_CTRLR0`；
3. 新增 `/k230-dw-ssi/fifo-depth`，实际写入 256 帧并验证第 257 帧不增加 level；
4. 通过 QMP `qom-get` 验证每个 child 的 `fifo-depth`、`max-lines` 和 `dma-register-layout`；通过寄存器/资源行为验证内部 capability 关闭语义；
5. 在 `k230.c` 增加 profile 数组和设置 helper；
6. 删除 reset 中 `max_lines == 8` 的 FMC 推断；
7. 删除 FMC region 1 映射和 HI_SYS `xip-enable` GPIO 连接；
8. system reset 后重新执行三实例 profile 断言。

### 三实例 reset 断言

每个实例都验证：

```text
CTRLR0
SSIENR
IMR
AXIAWLEN
AXIARLEN
IDR
SSIC_VERSION_ID
SPI_CTRLR0
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

第一批 properties 均按数值读取，使用 `qom-get` QMP response 中的 `QNum` 验证。这样可以直接验证 FMC 的 `max-lines=8`，而不把 Octal 数据路径实现错误地纳入 Step 4。

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
- 三实例均为 `internal-axi` layout，内部 capability 位图为 `0`；
- 三实例都只创建和映射 region 0，`0xc0000000` 不映射；
- Standard PIO/IRQ 和寄存器兼容性测试通过。

---

## 10. Step 4.3：锁定第一批 capability 关闭语义

第一批不暴露 capability property。每项只验证内部 capability 位为 0 时的 guest-visible 关闭语义，不能存在未实现寄存器、数据路径、IRQ 或资源表现为可用的中间态。

### 10.1 enhanced SPI 关闭路径

#### 关闭路径测试

`/dw-ssi/capability/enhanced-off`：

- 启动 Standard-only profile；
- 向 `CTRLR0.SPI_FRF` 写 Quad，读回仍为 Standard；
- `SPI_CTRLR0` 先读到 profile reset；写扩展字段后不得进入 enhanced path；`DDR_DRIVE_EDGE` 读回 0；
- 配置 Standard loopback，PIO 收发仍正常；
- 尝试 enhanced command 后 TX FIFO 不被 enhanced engine 消费；
- `CTRLR0.SPI_FRF` 读回 Standard，间接证明 `dw_ssi_get_spi_mode()` 的输入已收敛为 0；不为内部 getter 增加测试专用 property。

#### 最小实现

- `SPI_CTRLR0` 保持 profile-visible，只有 `DDR_DRIVE_EDGE` 等纯扩展寄存器 RAZ/WI；
- CTRLR0 read/write mask 动态清除 enhanced 字段；
- `dw_ssi_run_transfer()` 第一批只包含 Standard path；
- 第一批 VMState 不保存 enhanced command/phase 状态。

#### 第一批回归

```bash
TMPDIR=/tmp/qemu-k230-qtest-v2-step4 \
QTEST_QEMU_BINARY=$PWD/build/qemu-system-riscv64 \
build/tests/qtest/dw-ssi-test \
  -p /dw-ssi/capability/enhanced-off -v
TMPDIR=/tmp/qemu-k230-qtest-v2-step4 \
QTEST_QEMU_BINARY=$PWD/build/qemu-system-riscv64 \
build/tests/qtest/k230-dw-ssi-test \
  -p /k230-dw-ssi/register-contract -v
```

### 10.2 IDMA 关闭路径

#### 关闭路径测试

`/dw-ssi/capability/idma-off`：

- `dma-register-layout=internal-axi` 时，`DMACR`、`AXIAWLEN`、`AXIARLEN`、`SPIDR`、`SPIAR`、`AXIAR0/1` 按字段 mask 保存和读回；
- `AXIECR`、`DONECR` 读取为 0；
- 向 IMR 写 DONE/AXIE 位，读回仍为 0；TXU 位仍可写入和读回；
- 尝试写 `IDMAE`、SSIENR、SER 后不访问 guest memory；
- `SR.CMPLTD_DF == 0`；
- 用 `qtest_irq_intercept_out()` 拦截 `/machine/dw-ssi`，DONE/AXIE 两路保持低；再制造 TX FIFO underflow，确认 TXU raw status 和输出仍可见；
- DR 写入继续进入普通 TX FIFO，证明关闭 IDMA 没有破坏 PIO。

#### 最小实现

- DMA layout helper 决定寄存器存在性和字段 mask，不能用内部 IDMA capability 位隐藏 layout-visible 寄存器；
- 第一批不保留可启动的 `dw_ssi_idma_enabled()` 数据路径；
- DONE/AXIE 两路输出固定注册但保持低电平；
- reset 清 DONE/AXIE latch；
- 第一批 VMState 不保存 IDMA engine 运行时字段。

#### 关闭路径回归

```bash
TMPDIR=/tmp/qemu-k230-qtest-v2-step4 \
QTEST_QEMU_BINARY=$PWD/build/qemu-system-riscv64 \
build/tests/qtest/dw-ssi-test \
  -p /dw-ssi/capability/idma-off -v
TMPDIR=/tmp/qemu-k230-qtest-v2-step4 \
QTEST_QEMU_BINARY=$PWD/build/qemu-system-riscv64 \
build/tests/qtest/k230-dw-ssi-test \
  -p /k230-dw-ssi/register-contract -v
```

### 10.3 XIP 关闭路径

#### 关闭路径测试

`/dw-ssi/capability/xip-off`：

- `XIP_MODE_BITS`、`XIP_INCR_INST`、`XIP_WRAP_INST` 和 XIP 扩展寄存器全部 RAZ/WI；
- QMP `qom-list /machine/dw-ssi` 中没有名为 `designware-ssi.xip[0]`、类型为 `child<memory-region>` 的条目；
- system reset 后行为不变。
- device 不注册 `xip-enable` GPIO，K230 不映射 region 1。

#### 最小实现

- 专用 XIP 寄存器分组 RAZ/WI；
- 第一批删除 XIP GPIO、第二个 MemoryRegion 和 K230 `0xc0000000` 映射。

#### 关闭路径回归

```bash
TMPDIR=/tmp/qemu-k230-qtest-v2-step4 \
QTEST_QEMU_BINARY=$PWD/build/qemu-system-riscv64 \
build/tests/qtest/dw-ssi-test \
  -p /dw-ssi/capability/xip-off -v
TMPDIR=/tmp/qemu-k230-qtest-v2-step4 \
QTEST_QEMU_BINARY=$PWD/build/qemu-system-riscv64 \
build/tests/qtest/k230-dw-ssi-test \
  -p /k230-dw-ssi/register-contract -v
```

### Step 4.3 完成标准

- 三个内部 capability 位都有关闭行为测试；
- K230 三实例均为内部 capability 全零、layout 为 `internal-axi`；
- layout-visible DMA 寄存器可读写，但无 engine、guest memory 和 DONE/AXIE 行为；
- 无 XIP GPIO、region 1 和 `0xc0000000` 映射；
- 后续 series 可以逐项增加对应 public property，不需要修改 `max-lines` 或 DMA register layout 设计。

---

## 11. VMState 与迁移边界

### 11.1 最小配置 equality

QOM properties 在 realize 前确定，和 machine type/profile 一起由源端、目的端分别创建。它们不作为 guest 可变状态迁移，但以下两项会改变动态状态或寄存器存储的解释，必须作为 equality guard 放在依赖字段之前：

```c
VMSTATE_UINT32_EQUAL(cfg.fifo_depth, DwSsiState),
VMSTATE_UINT32_EQUAL(cfg.dma_register_layout, DwSsiState),
```

`VMSTATE_FIFO32` 使用目的端已经创建的 FIFO capacity 解释数据，因此 `fifo-depth` 不一致必须拒绝。`dma-register-layout` 决定同一 offset 的寄存器含义，也必须一致。内部 capability 位图第一批固定为 0，不作为迁移字段；`num-cs`、ID、VERSION、reset 值和 XIP 配置不在本步增加 equality。

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

当前 V2 分支尚未上游发布，Step 4 不承诺与未发布中间态进行跨版本迁移。Plan Final V1.2 只加入 FIFO 深度和 DMA layout equality，不把全部 cfg 参数塞进迁移流。测试覆盖同 profile 成功、FIFO 深度不一致失败和 DMA layout 不一致失败三种必要情况。

---

## 12. TDD 测试矩阵

| 测试路径 | 配置 | 核心断言 |
|---|---|---|
| `/dw-ssi/config/defaults` | 默认通用实例 | capability 全关、DMA layout=none、只支持 Standard PIO/IRQ |
| `/dw-ssi/config/fifo-depth` | `fifo-depth=8` | 8 帧满，第 9 帧不增长，阈值边界，reset 清空 |
| `/dw-ssi/config/invalid/*` | `-preconfig` 下每个路径一个非法组合 | QMP `realized=true` 返回精确配置错误 |
| `/dw-ssi/layout/none` | DMA layout=none | DMA 配置地址 RAZ/WI |
| `/dw-ssi/layout/external` | DMA layout=external | DMACR/DMATDLR/DMARDLR 按 mask 存储，无 DMA request |
| `/dw-ssi/layout/internal-axi` | DMA layout=internal-axi、内部 IDMA 位为 0 | AXI/IDMA 配置寄存器存储，无 guest memory/DONE/AXIE |
| `/dw-ssi/capability/enhanced-off` | 默认内部 capability 位图为 0 | SPI_FRF 保持 Standard，SPI_CTRLR0 reset 可见，Standard PIO 正常 |
| `/dw-ssi/capability/idma-off` | internal-axi、内部 IDMA 位为 0 | layout-visible 寄存器可读写，无 guest memory/DONE/AXIE，TXU 仍可见 |
| `/dw-ssi/capability/xip-off` | 内部 XIP 位为 0 | XIP-only 寄存器 RAZ/WI，无 GPIO和第二 region |
| `/dw-ssi/migration/same-profile` | 相同 FIFO/layout | 迁移成功，FIFO、寄存器和 IRQ 状态恢复 |
| `/dw-ssi/migration/fifo-depth-mismatch` | 8 → 16 | equality 拒绝迁移 |
| `/dw-ssi/migration/dma-layout-mismatch` | external → internal-axi | equality 拒绝迁移 |
| `/k230-dw-ssi/register-contract` | K230 三实例 | 完整 reset profile、SER 位宽、properties |
| `/k230-dw-ssi/fifo-depth` | K230 256 深度 | 256 帧满，第 257 帧不增长 |
| `/k230-dw-ssi/profile-capabilities` | K230 三实例 | max-lines=4/4/8、internal-axi、内部 capability 全零、无 XIP region |
| `/k230-dw-ssi/standard-flash` | M25P80-compatible NOR | Standard 1-1-1 JEDEC ID/固定地址读取，不访问 XIP aperture |
| K230 WDT qtest | K230 SoC | SSI 资源数量变化不破坏其他设备 realize/mapping |

系统 reset 后至少重新运行：三实例 reset profile、FIFO 清空、capability 关闭语义、DMA layout readback 和 Standard Flash 访问。

Step 4.0 的 enhanced/XIP 隔离与 IDMA `0xeb` 测试属于当前 V2 中间态的已完成回归，不进入第一批最终 patch。后续 enhanced/IDMA series 恢复对应功能时必须同时恢复这两项测试，且 `XIP_MODE_BITS` 仍保持 XIP-only。

---

## 13. Step 4.4：收敛验证

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

成功标准：第一批通用配置/layout/PIO/IRQ/迁移测试全部 PASS；K230 三实例 profile、PLIC、Standard Flash 测试全部 PASS；WDT qtest PASS；无 FAIL/ERROR/SKIP。当前 V2 中间态的 enhanced/IDMA/XIP 测试不计入第一批通过口径。

### 13.3 公共头文件独立包含

```bash
printf '#include "qemu/osdep.h"\n#include "hw/ssi/dw_ssi.h"\n' | \
cc -fsyntax-only -x c -I. -Iinclude -Ibuild -Ibuild/include \
  $(pkg-config --cflags glib-2.0) -
```

### 13.4 通用层 K230 依赖残留

```bash
rg -n 'k230|K230' hw/ssi/dw_ssi.c include/hw/ssi/dw_ssi.h
```

预期：除版权作者或说明文档明确允许的文本外，不出现 K230 类型、地址、reset、HI_SYS 或 machine 依赖。通用测试 machine 也不得包含 K230 常量。

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

## 14. 实施清单（收敛版）

### Step 4.0：纠正现有数据路径

- [x] 普通 enhanced/IDMA 解码不读取 `XIP_MD_BIT_EN`、`XIP_MBL`、`XIP_MODE_BITS`
- [x] 删除普通 enhanced mode phase 和 IDMA 1-4-4 特判
- [x] 普通 enhanced/IDMA 四阶段回归通过
- [x] FMC XIP mode bits 正路径保持通过

### Step 4.1：配置骨架

- [ ] 新增 `DwSsiConfig`，把 `num_cs`、`max_lines`、`dma_register_layout` 和内部 `capabilities` 收入 `cfg`
- [ ] 通用默认内部 capability 位图为 0、DMA layout=none
- [ ] 增加 `dw_ssi_validate_config()`
- [ ] 增加独立 `dw-ssi-test` machine、Meson 注册与 qtest
- [ ] 非法配置按 case 使用 `-preconfig` + QMP realize error 验证
- [ ] FIFO 按 `fifo-depth` 在 realize 创建
- [ ] 所有 FIFO 容量与阈值逻辑改用 `cfg.fifo_depth`
- [ ] 保留 finalize 统一销毁 FIFO 和动态 CS 数组
- [ ] reset 从 `cfg` 和 DMA layout 读取，`SPI_CTRLR0` 保留 profile reset
- [ ] 增加 `fifo-depth`、`dma-register-layout` 两项 VMState equality
- [ ] 同 profile 迁移成功，FIFO/layout 不一致迁移失败
- [ ] 内部 capability 位图固定为 0，并通过关闭语义测试

### Step 4.2：K230 profile

- [ ] 在 `k230.c` 增加三实例完整 profile
- [ ] 三实例 realize 前显式设置全部 property
- [ ] 删除 `max_lines == 8` 推断 FMC reset
- [ ] 三实例 `max-lines=4/4/8`、layout=`internal-axi`、内部 capability 位图为 0
- [ ] 删除全部 XIP region 映射和 HI_SYS `xip-enable` GPIO 连接
- [ ] 三实例 reset、SER、FIFO、layout/property 测试通过

### Step 4.3：capability

- [ ] 内部 enhanced capability 位为 0 时 `max-lines=4/8` 可 realize，SPI_FRF 保持 Standard
- [ ] `SPI_CTRLR0` profile reset 可见，扩展写入不进入数据路径
- [ ] DMA layout 决定寄存器存在性，IDMA capability 不隐藏 layout-visible 寄存器
- [ ] 内部 IDMA capability 位为 0 时无 guest memory 访问和 engine 行为
- [ ] 内部 IDMA capability 位为 0 时 DONE/AXIE 无效，但 TXU 仍可产生和上报
- [ ] 内部 XIP capability 位为 0 时 XIP-only 寄存器 RAZ/WI，无 GPIO、region 1 和地址映射
- [ ] Standard 1-1-1 Flash 挂接和读取测试通过

### Step 4.4：收敛

- [ ] 通用 qtest、K230 SSI qtest、K230 WDT qtest 全过
- [ ] 公共头文件独立包含通过
- [ ] 通用 SSI 无 K230 依赖
- [ ] VMState 只保存已实现的 Standard PIO/IRQ 状态，不预埋 enhanced/IDMA/XIP 字段
- [ ] `git diff --check` 与相关 checkpatch 通过
- [ ] 每个小目标的未来 patch 归属明确

---

## 15. 第一批上游 Series 边界与渐进提交策略

### 15.1 第一批 Series 的目标

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
  ├── 把 9 路 IRQ 输出连接到 PLIC
  └── 挂接 Standard 1-1-1 SPI NOR
```

这批 series 的核心 review 问题有三个：

1. `dw_ssi.c` 是否已经成为不依赖 K230 类型、地址和 HI_SYS 的通用设备模型；
2. DMA register layout 与 enhanced/IDMA/XIP engine capability 是否真正正交；
3. K230 是否只负责实例配置、地址映射、PLIC 接线和 Flash 挂接。

第一批包含 Standard 1-1-1 SPI NOR 挂接，但不以 SDK U-Boot 完整启动作为合入条件，因为现有启动链还会使用 enhanced SPI 和 IDMA。完成标准是通用寄存器/layout/FIFO/PIO/IRQ qtest、K230 实例/PLIC 集成 qtest 和 Standard Flash 读取测试全部通过。

### 15.2 第一批包含和不包含的内容

| 类别 | 第一批包含 | 第一批不包含 |
|---|---|---|
| 通用设备 | `TYPE_DW_SSI`、SSI bus、控制器 region 0、CS outputs、realize/finalize、reset、基础 VMState | K230 wrapper、HI_SYS 指针、K230 地址常量 |
| 配置 | 完整 `DwSsiConfig`、`max-lines`、DMA layout、profile reset、内部 capability 位图 | capability 正路径 |
| 数据路径 | Standard single-line SPI，四种 TMOD、frame width、loopback、FIFO level/status、Standard 1-1-1 NOR | Dual/Quad/Octal、enhanced command、IDMA、XIP transaction |
| IRQ | TXE、TXO、RXF、RXO、TXU、RXU、MST 的基础语义和 K230 PLIC 路由 | DONE/AXIE 的产生和清除逻辑 |
| 资源 | 控制器 MMIO、SSI bus、CS、9 路 IRQ、Flash attachment | XIP region、`xip-enable` GPIO |
| 测试 | 通用寄存器/layout/PIO/IRQ，K230 三实例、PLIC、Standard Flash | enhanced、IDMA、HI_SYS、XIP 正路径 |

`TXU` 属于基础 TX FIFO underflow，必须随第一批 IRQ 一起实现。第一批固定注册并路由完整 9 路输出；DONE/AXIE 必须恒低，其 IMR/ISR/RISR 位在 IDMA series 到来前不得表现为有效功能。

### 15.3 “寄存器占位但不提供功能”的精确定义

第一批可以保留完整控制器 MMIO aperture，但不能通过一组半实现的寄存器暗示 enhanced、IDMA 或 XIP 已受支持。占位采用以下规则：

1. **地址空间占位**：region 0 大小保持稳定；未实现 offset 统一 RAZ/WI，不为每个未来寄存器增加空 case；
2. **基础寄存器真实实现**：Standard PIO、FIFO、状态、基础 IRQ、IDR/VERSION 所依赖的寄存器必须具有明确 reset、write mask 和副作用；
3. **layout 与 engine 分离**：`dma-register-layout` 决定 external/internal-axi 寄存器是否存在和如何读写；后续 `has-idma` property 只决定 engine、guest memory 和事件；
4. **有证据才暴露非零契约**：如果 firmware 枚举确实需要读取某个扩展寄存器 reset，可以单独实现只读/reset 语义并写测试，但 commit message 必须声明“寄存器可见不等于数据路径已实现”；
5. **capability 不虚假开放**：第一批只保留内部全零 capability 位图，不暴露只能取单一值的 public property；后续 series 必须随正路径、关闭路径、资源、IRQ 和测试一起引入对应 property；
6. **XIP-only 保持专用**：`XIP_MODE_BITS` 第一批 RAZ/WI，后续 IDMA series 也不得把它改为 shared 或重新用于 1-4-4 mode byte。

因此，“先占位”精确表达为：**版图存在性由 profile/layout 决定，寄存器兼容性不等于数据路径实现；纯扩展 offset 在 capability 未开放时 RAZ/WI。**

### 15.4 第一批仍需具备的通用基础

除了 PIO 和 IRQ，第一批还需要以下基础，否则“通用 DW SSI”拆分仍不完整：

- 完整 `DwSsiConfig`、DMA register layout 和 realize 前配置校验；
- FIFO、CS 动态资源的明确创建和销毁生命周期；
- 基础 VMState：寄存器、FIFO、PIO phase、remaining frames、IRQ latch、active CS，以及 FIFO/layout equality；
- 固定且有文档说明的 SSI bus、CS output 和 IRQ output 接口；
- 通用 qtest 载体，用于证明 PIO/IRQ 语义不依赖 K230 machine；
- K230 集成 qtest 只验证三实例 profile、地址映射、PLIC source 和实例隔离，避免与通用测试重复；
- Standard 1-1-1 Flash 测试提供真实 SSI consumer；
- 非法 `num-cs`、`fifo-depth` 等配置在 realize 阶段返回清晰错误。

第一批不需要 trace events 或启动镜像。trace 可以随对应功能加入，避免单独增加只为调试服务的 review 面积。

### 15.5 第一批 Commit 顺序

最终选择 5.6-A：第一批保持六个可编译、可回归的提交，每个提交只增加一类职责：

1. `hw/ssi: Add a Synopsys DesignWare SSI standard register model`
   - 建立通用 QOM 类型、最小配置、基础寄存器、reset、VMState 和通用 register qtest；
2. `hw/riscv/k230: Instantiate DesignWare SSI controllers`
   - 创建三个实例，设置第一批所需 profile，映射 region 0；
3. `hw/ssi: Implement DesignWare SSI FIFO and standard PIO transfers`
   - 实现动态 FIFO、四种 TMOD、loopback 和 PIO qtest；
4. `hw/ssi: Add DesignWare SSI standard interrupt support`
   - 实现基础 raw/masked/clear 语义、threshold IRQ 和通用 IRQ qtest；
5. `hw/riscv/k230: Route SSI interrupts to the PLIC`
   - 完成三实例 PLIC 接线和路由隔离 qtest。
6. `hw/riscv/k230: Attach a standard SPI flash to the K230 SSI`
   - 保留 `spi-flash` property，挂接 M25P80-compatible NOR，并增加 Standard 1-1-1 读取测试。

测试应跟随首次实现该行为的 commit，不把所有测试集中到最后一个 patch。每个 commit 完成后至少执行增量构建、对应定向 qtest 和 `git diff --check`；完成整个 series 后再运行完整通用/K230 qtest。

本地开发 commit 可以按 Step 4.0、4.1、4.2 等小目标逐步积累；发送上游前再重组到上述六个职责提交。Step 4.0 当前提交属于本地纠错历史：第一批 series 不包含 enhanced/IDMA，因此不需要单独携带这个修复 patch；后续 enhanced/IDMA series 应从首次出现开始就是正确实现，不能先引入错误 mode phase 再补修复。

### 15.6 后续 Series

第一批合入或架构 review 基本稳定后，再顺序发送：

1. enhanced SPI 数据路径，在已有 `max-lines` / `spi-ctrlr0-reset` / NOR 挂接上增加 Dual/Quad SDR；
2. optional internal DMA engine + guest memory 搬运 + DONE/AXIE；
3. K230 HI_SYS + optional XIP region、GPIO 和 XIP transaction。

IDMA series 必须恢复 SDK 风格 `0xeb` 1-4-4 回归，并保持 `XIP_MODE_BITS` 为 XIP-only；dummy/mode 不得重新耦合。

后续 series 不应同时并发发送，否则 reviewer 同一时间仍需理解完整三千余行改动，失去分批投稿的意义。

---

## 16. 与最终 V2 patch 系列的关系

Step 4 的实现内容在第五步重组时按职责回填，不形成一个覆盖所有功能的“大配置 patch”：

| Step 4 内容 | 最终 patch 归属 |
|---|---|
| `DwSsiConfig`、DMA layout、内部 capability 位图、配置校验、通用测试 machine | Patch 1：通用寄存器模型与基础配置 |
| K230 三实例 profile、控制器 MMIO | Patch 2：K230 实例化通用控制器 |
| 动态 FIFO 深度 | Patch 3：FIFO/PIO |
| 动态 IRQ mask 的基础部分 | Patch 4：通用 IRQ |
| K230 9 路 PLIC 路由 | Patch 5：PLIC 集成 |
| Standard 1-1-1 Flash 挂接 | Patch 6：K230 Flash integration |
| 引入 `has-enhanced-spi` property 并实现正路径 | 后续 enhanced SPI series |
| 引入 `has-idma` property 并实现 engine | 后续 optional IDMA series |
| 引入 `has-xip`/`xip-window-size` property、增加 region/GPIO/`0xc0000000` | 后续 optional XIP series |

每个最终 patch 必须包含自己需要的 property、测试和 machine 配置；后续 enhanced/IDMA/XIP series 不得修复第一批已经引入的不可编译、不可 realize 或虚假 capability 中间态。

---

## 17. 参考入口

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
- [QEMU DesignWare I3C 先例](../../../qemu-camp-2026-k230/hw/i3c/dw-i3c.c)
- [QEMU Xilinx QSPI 线性窗口先例](../../../qemu-camp-2026-k230/hw/ssi/xilinx_spips.c)
- [K230 U-Boot DTS](../../../k230_sdk/src/little/uboot/arch/riscv/dts/k230.dtsi)
- [K230 Linux DTS](../../../k230_sdk/src/little/linux/arch/riscv/boot/dts/kendryte/k230.dtsi)
- [K230 U-Boot DesignWare SPI 驱动](../../../k230_sdk/src/little/uboot/drivers/spi/designware_spi.c)
- [K230 RT-Smart SPI 驱动](../../../k230_sdk/src/big/rt-smart/kernel/bsp/maix3/board/interdrv/spi/drv_spi.c)

---

## 18. 与总决策文档的关系

本文只细化 V2 第四步。通用层与 K230 层边界、XIP aperture 归属、默认不增加 wrapper、capability 证据标准仍以 [V2 决策记录](k230-spi-qspi-review-v2-decision-notes.md) 为准。

若实施中发现 K230 某个 reset 或 capability 与本矩阵冲突，先回到 TRM/SDK 定位证据并更新决策记录，再调整 profile；不得在 `dw_ssi.c` 中重新加入 K230 分支或地址常量。
