# K230 V2 第四步 Plan Final：实例配置与 capability 门控

首次记录：2026-07-29

最终修订：2026-07-30（记录 Step 4.0 完成状态，并明确第一批上游 series 只覆盖 Standard SPI PIO、基础 IRQ 和 K230 集成）

适用代码检查点：`qemu-camp-2026-k230/` 分支 `k230-V2-patch-spi`，commit `c689ac865f`

文档状态：Step 4.0 已完成源码实施和 12 项 K230 SSI qtest；Step 4.1 至 Step 4.4 仍是待实施计划

本文是 [V2 实施路线](k230-spi-qspi-v2-implementation-plan.md) 第四步“引入实例配置”的唯一执行计划。Plan A、Plan B、Plan C 及其 planning handoff 只保留为历史推演材料，不再作为实施入口。架构边界以 [V2 决策记录](k230-spi-qspi-review-v2-decision-notes.md) 为准：保留单一通用 `TYPE_DW_SSI` / `DwSsiState`，K230 machine 通过 properties 配置三个实例，不增加 `TYPE_K230_DW_SSI` wrapper。

> **上游投稿边界说明（2026-07-30）**：本文描述的是完整 Step 4 的本地实施终态，不等于第一批上游 patch series 必须携带全部 capability。第一批 series 计划只提交通用 DW SSI 的 Standard SPI PIO、基础 IRQ，以及 K230 三实例和 PLIC 集成；enhanced SPI、SPI NOR、IDMA、HI_SYS 和 XIP 分批后送。详细边界见第 15 节。

---

## 摘要

Step 4 先修正当前模型把普通 enhanced/IDMA 事务错误连接到 XIP mode bits 的问题，再将 K230 固定参数改为 realize 前确定的实例配置，并把 `has-enhanced-spi`、`has-idma`、`has-xip` 三项 capability 逐项接入寄存器、IRQ、数据路径和资源生命周期。

实施分为五个可独立验证的小目标：

1. Step 4.0：普通 enhanced/IDMA 只执行 instruction → address → dummy → data，XIP mode 只留在真正的 XIP transaction；
2. Step 4.1：建立 `DwSsiConfig`、完整 properties、配置校验、动态 FIFO 和最小迁移一致性检查；
3. Step 4.2：在 K230 machine 中显式应用 QSPI0、QSPI1、SPI-OPI/FMC 三个 profile，消除 reset 值和 XIP region 的 K230 硬编码推断；
4. Step 4.3：按 enhanced SPI → IDMA → XIP 的顺序接入 capability 关闭语义，每项先写负路径测试，再实现最小门控；
5. Step 4.4：完成通用模型、K230 集成、公共头文件、迁移状态和未来 patch 归属检查。

本步不实现新的 DDR、RXDS、Octal、XIP write 或 external DMA 行为。`max-lines=8` 描述 FMC 的实例线宽上限，不改变当前模型只执行 Standard/Dual/Quad SDR、显式拒绝 Octal/DDR/RXDS 的功能边界。

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
| qtest | `k230-dw-ssi-test` 共 12 项；Step 4.0 新增 enhanced/XIP 隔离和 IDMA 1-4-4 回归，仍缺 capability `false` 组合 |

### 1.2 当前硬编码点清单

| 编号 | 当前位置 | 硬编码或耦合 | Step 4 动作 |
|---|---|---|---|
| H1 | `include/hw/ssi/dw_ssi.h:28` | `DW_SSI_XIP_WINDOW_SIZE = 0x08000000` | 改为 `xip-window-size` property |
| H2 | `include/hw/ssi/dw_ssi.h:94-95` | `num_cs`、`max_lines` 分散在运行时状态中 | 收入集中 `DwSsiConfig` |
| H3 | `hw/ssi/dw_ssi.c:27` | FIFO 固定 256 | 改为 `fifo-depth` |
| H4 | `hw/ssi/dw_ssi.c:31-37` | `IMR`、IDR、VERSION、AXI burst 固定 | reset 从实例配置读取 |
| H5 | `hw/ssi/dw_ssi.c:33-34` | 仅按 SPI/FMC 固定两个 `SPI_CTRLR0` reset | 改为 `spi-ctrlr0-reset` |
| H6 | `hw/ssi/dw_ssi.c:425-430` | `SR.TFNF/RFF` 与 256 比较 | 改为 `cfg.fifo_depth` |
| H7 | 原 `hw/ssi/dw_ssi.c:537-584`、`949-955`、`1021-1043` | 普通 enhanced/IDMA 曾错误读取 `XIP_MODE_BITS`，PIO 曾包含 XIP mode phase | Step 4.0 已完成：普通事务只保留 instruction/address/dummy/data |
| H8 | `hw/ssi/dw_ssi.c:813-984` | IDMA 寄存器和入口始终存在 | `has-idma` 关闭时 RAZ/WI、无搬运，仅 DONE/AXIE 无效；TXU 保持基础 FIFO IRQ |
| H9 | `hw/ssi/dw_ssi.c:1201-1535` | read/write switch 无 capability 分组 | 增加统一寄存器存在性辅助函数 |
| H10 | `hw/ssi/dw_ssi.c:1578-1585` | reset 使用 K230 值，并用 `max_lines == 8` 推断 FMC | 全部从 `cfg` 读取，删除推断 |
| H11 | `hw/ssi/dw_ssi.c:1677-1689` | 所有实例创建 XIP region 和 256 深度 FIFO | 移到 `realize()`，按 property 创建 |
| H12 | `hw/riscv/k230.c:246-251` | machine 只设置 CS 和线宽 | 为三个实例显式设置完整 profile |
| H13 | `hw/riscv/k230.c:293-294` | 无条件映射 `dw_ssi[2]` region 1 | 仅对 `has-xip=true` 的 FMC profile 映射 |
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

Step 4 后应改为：固定接口在 `instance_init` 注册，依赖 property 的资源在 `realize()` 校验后创建。这样 `fifo-depth`、`has-xip`、`xip-window-size` 都在资源创建前确定，设备 realize 后不允许改变。

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
| `has-enhanced-spi` | `true` | `true` | `true` |
| `has-idma` | `true` | `true` | `true` |
| `has-xip` | `false` | `false` | `true` |
| `xip-window-size` | `0` | `0` | `0x08000000` |
| 控制器 MMIO | `0x91582000` | `0x91583000` | `0x91584000` |
| XIP MMIO | 不创建/不映射 | 不创建/不映射 | `0xc0000000`，128 MiB |

`has-xip=false` 不等于 QSPI 的共享 `SPI_CTRLR0` 中所有名字含 XIP 的位都必须读零。QSPI 的 TRM reset `0x04000200` 本身包含 `XIP_MBL` 编码；Step 4 应保留整个共享 `SPI_CTRLR0` 的实例 reset，`has-xip` 只门控专用 XIP 寄存器组、XIP 数据路径和第二个 MMIO region。

### 2.4 QEMU 主线组织先例

| 主线代码 | 可借鉴点 | 本计划用法 |
|---|---|---|
| `include/hw/i3c/dw-i3c.h` | 状态内集中 `cfg` 结构 | `DwSsiState` 内增加 `DwSsiConfig cfg` |
| `hw/i3c/dw-i3c.c:1806-1824` | 在 `realize()` 按 property 创建 FIFO/MMIO | 动态创建 SSI FIFO 和可选 XIP region |
| `hw/i3c/dw-i3c.c:1782-1803` | reset 从 `cfg` 填充寄存器字段 | DW SSI reset 从实例 profile 读取 |
| `hw/i3c/dw-i3c.c:1854-1872` | properties 直接写入 `cfg` | 使用 kebab-case QOM properties |
| `hw/ssi/xilinx_spips.c:1372-1388` | 控制器在 realize 中增加线性 Flash region | `has-xip=true` 时提供第二个 sysbus region |
| `hw/arm/xlnx-zynqmp.c:881-885` | SoC 负责映射控制器 region 与 LQSPI region | K230 只把 FMC region 1 映射到 `0xc0000000` |
| `hw/ssi/xilinx_spips.c:1397-1401` | 非法配置通过 `error_setg()` 拒绝 realize | 集中校验 property 范围与组合 |

本计划不把 XIP window 改成 K230 alias，也不新增 wrapper。V2 已决策为“通用模型实现 XIP SPI transaction 和 MMIO 访问语义，SoC 决定窗口大小与地址”；Step 4 只把现有固定 region 改为 capability 控制的可选 region。

### 2.5 已确认决策与实施假设

| 项目 | 结论 |
|---|---|
| 配置组织 | 使用单一 `DwSsiConfig` 和 kebab-case QOM properties；不增加 K230 wrapper |
| 默认值 | Step 4.1 默认值只保持当前 V2 中间态行为；Step 4.2 后 K230 显式设置全部字段 |
| capability 顺序 | enhanced SPI → IDMA → XIP；每项独立写负路径、实现和回归 |
| 负路径载体 | 使用独立 `dw-ssi-test` machine，不向 K230 产品 machine 增加测试入口 |
| XIP 归属 | 通用模型实现 XIP transaction 和可选 region；K230 只决定 profile、地址映射与 Flash 连接 |
| IRQ / VMState | IRQ 输出数量和 VMState schema 固定；TXU 属于基础 FIFO IRQ，IDMA 只增加 DONE/AXIE；迁移仅对 FIFO 深度和 capability profile 做 equality |
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

    uint32_t capabilities;
    uint64_t xip_window_size;
} DwSsiConfig;
```

三个 public property 仍是独立 bool，内部用一个 `uint32_t` 保存，便于集中判断和迁移 equality，不对外暴露 capability bitmask property：

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
    MemoryRegion xip;
    SSIBus *spi;

    qemu_irq *cs_lines;
    qemu_irq irqs[DW_SSI_IRQ_COUNT];

    Fifo32 tx_fifo;
    Fifo32 rx_fifo;
    uint32_t regs[DW_SSI_NUM_REGS];

    DwSsiConfig cfg;

    uint32_t irq_latched;
    uint32_t idma_completed_frames;
    uint32_t phase;
    uint32_t remaining_frames;
    DwSsiEnhancedCommand enhanced;

    int active_cs;
    bool sleep_status;
    bool xip_enabled;
};
```

properties 只表达不可变硬件配置。`SSIENR`、`SER`、`xip_enabled`、FIFO level、phase、IRQ latch 等 guest 运行时状态不得做成 property。

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
                       cfg.axiawlen_reset, 0x00000700),
    DEFINE_PROP_UINT32("axiarlen-reset", DwSsiState,
                       cfg.axiarlen_reset, 0x00000700),
    DEFINE_PROP_UINT32("spi-ctrlr0-reset", DwSsiState,
                       cfg.spi_ctrlr0_reset, 0x04000200),

    DEFINE_PROP_BIT("has-enhanced-spi", DwSsiState,
                    cfg.capabilities, 0, true),
    DEFINE_PROP_BIT("has-idma", DwSsiState,
                    cfg.capabilities, 1, true),
    DEFINE_PROP_BIT("has-xip", DwSsiState,
                    cfg.capabilities, 2, true),
    DEFINE_PROP_SIZE("xip-window-size", DwSsiState,
                     cfg.xip_window_size, 0x08000000),
};
```

这些默认值仅用于 Step 4.1 保持当前 V2 中间态行为，不宣称是所有 DWC SSI 实例的硬件默认值。Step 4.2 后 K230 machine 必须显式设置表中全部字段；上游最终版是否继续保留兼容默认值，在 patch 重组时单独审阅，不在 Step 4 内提前收紧。

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

    if (!dw_ssi_has_capability(s, DW_SSI_CAP_ENHANCED_SPI) &&
        s->cfg.max_lines != 1) {
        error_setg(errp,
                   "%s: max-lines must be 1 without enhanced SPI",
                   dev->canonical_path);
        return false;
    }

    if (dw_ssi_has_capability(s, DW_SSI_CAP_IDMA) &&
        !dw_ssi_has_capability(s, DW_SSI_CAP_ENHANCED_SPI)) {
        error_setg(errp,
                   "%s: IDMA requires the current enhanced SPI engine",
                   dev->canonical_path);
        return false;
    }

    if (dw_ssi_has_capability(s, DW_SSI_CAP_XIP) &&
        !dw_ssi_has_capability(s, DW_SSI_CAP_ENHANCED_SPI)) {
        error_setg(errp,
                   "%s: XIP requires the enhanced SPI engine",
                   dev->canonical_path);
        return false;
    }

    if (dw_ssi_has_capability(s, DW_SSI_CAP_XIP) !=
        (s->cfg.xip_window_size != 0)) {
        error_setg(errp,
                   "%s: has-xip and xip-window-size must agree",
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

`has-idma` 依赖 enhanced SPI 是当前实现约束：`dw_ssi_try_idma()` 使用 enhanced command decoder。它不是对所有 DWC SSI 硬件变体的普遍宣称；若后续独立实现 Standard-SPI IDMA，再单独放宽组合。

### 3.4 property 依赖的资源创建

`instance_init` 保留固定接口：SSI bus、控制器 MMIO、9 路 IRQ 和 `xip-enable` GPIO。以下代码是 Step 4 全部完成后的最终形态：`realize()` 先校验，再创建 CS、FIFO 和可选 XIP region。实施时 FIFO 部分在 Step 4.1 落地，`has-xip` 条件创建必须留到 Step 4.3.3，不能提前把 XIP capability 混入配置骨架 patch。

```c
static void dw_ssi_realize(DeviceState *dev, Error **errp)
{
    DwSsiState *s = DW_SSI(dev);
    SysBusDevice *sbd = SYS_BUS_DEVICE(dev);

    if (!dw_ssi_validate_config(s, errp)) {
        return;
    }

    s->cs_lines = g_new0(qemu_irq, s->cfg.num_cs);
    qdev_init_gpio_out_named(dev, s->cs_lines, "cs", s->cfg.num_cs);

    fifo32_create(&s->tx_fifo, s->cfg.fifo_depth);
    fifo32_create(&s->rx_fifo, s->cfg.fifo_depth);

    if (dw_ssi_has_capability(s, DW_SSI_CAP_XIP)) {
        memory_region_init_io(&s->xip, OBJECT(s), &dw_ssi_xip_ops, s,
                              TYPE_DW_SSI ".xip",
                              s->cfg.xip_window_size);
        sysbus_init_mmio(sbd, &s->xip);
    }
}
```

控制器固定 MMIO region 0 继续在 `instance_init` 创建。XIP region 只有 `has-xip=true` 时才作为 region 1 注册；`has-xip=false` 的 machine 不能调用 `sysbus_mmio_map(..., 1, ...)`。

现有 `dw_ssi_finalize()` 继续统一执行 `fifo32_destroy()` 和 `g_free(s->cs_lines)`。QOM 实例内存初始为零，`fifo32_destroy()` / `g_free()` 对尚未创建的空资源安全，因此不新增 `fifo_created` 一类状态位。`realize()` 必须在全部配置校验通过后才创建 FIFO、CS 和 XIP region，且资源创建后不再执行可能失败的配置检查，避免部分创建状态。

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
s->regs[R_AXIAWLEN] = s->cfg.axiawlen_reset;
s->regs[R_AXIARLEN] = s->cfg.axiarlen_reset;
s->regs[R_IDR] = s->cfg.component_id;
s->regs[R_SSIC_VERSION_ID] = s->cfg.version_id;
s->regs[R_SPI_CTRLR0] =
    dw_ssi_has_capability(s, DW_SSI_CAP_ENHANCED_SPI) ?
    s->cfg.spi_ctrlr0_reset : 0;
```

reset 末尾调用 capability 状态收敛函数，保证 absent capability 的寄存器、状态和 IRQ 均为无效状态。

---

## 4. capability 门控设计

### 4.1 集中寄存器分组

禁止在 read/write switch 的每个 case 中散落 `if (s->cfg.has_...)`。新增寄存器分组辅助函数：

```c
typedef enum DwSsiRegCapability {
    DW_SSI_REG_CAP_NONE = 0,
    DW_SSI_REG_CAP_ENHANCED = BIT(0),
    DW_SSI_REG_CAP_IDMA = BIT(1),
    DW_SSI_REG_CAP_XIP = BIT(2),
} DwSsiRegCapability;

static unsigned int dw_ssi_reg_capability(hwaddr addr)
{
    switch (addr) {
    case A_SPI_CTRLR0:
    case A_DDR_DRIVE_EDGE:
        return DW_SSI_REG_CAP_ENHANCED;

    case A_DMACR:
    case A_AXIAWLEN:
    case A_AXIARLEN:
    case A_SPIDR:
    case A_SPIAR:
    case A_AXIAR0:
    case A_AXIAR1:
    case A_AXIECR:
    case A_DONECR:
        return DW_SSI_REG_CAP_IDMA;

    case A_XIP_MODE_BITS:
    case A_XIP_INCR_INST:
    case A_XIP_WRAP_INST:
    case A_XIP_CTRL:
    case A_XIP_SER:
    case A_XRXOICR:
    case A_XIP_CNT_TIME_OUT:
    case A_XIP_WRITE_INCR_INST:
    case A_XIP_WRITE_WRAP_INST:
    case A_XIP_WRITE_CTRL:
        return DW_SSI_REG_CAP_XIP;

    default:
        return DW_SSI_REG_CAP_NONE;
    }
}
```

`dw_ssi_reg_present()` 只在 read/write 入口调用一次；capability 关闭时直接 RAZ/WI。当前本来就未实现的 concurrent XIP、dynamic wait 和 XIP write 寄存器继续 RAZ/WI，不因 `has-xip=true` 自动变成已实现。

### 4.2 capability 关闭语义

| capability | 关闭时寄存器语义 | IRQ | 状态与数据路径 | realize 资源 |
|---|---|---|---|---|
| `has-enhanced-spi=false` | `CTRLR0.SPI_FRF` 读 0/写忽略；`SPI_CTRLR0`、`DDR_DRIVE_EDGE` RAZ/WI | 当前无独立 enhanced IRQ | 不进入 enhanced phase；Standard PIO/TMOD 保持可用 | 无额外资源变化 |
| `has-idma=false` | `DMACR`、`AXIAWLEN`、`AXIARLEN`、`SPIDR`、`SPIAR`、`AXIAR0/1`、`AXIECR`、`DONECR` RAZ/WI | 仅 DONE、AXIE 对应 IMR/ISR/RISR 位无效且输出恒低；TXU 仍是有效的 TX FIFO underflow IRQ | `idma_completed_frames=0`；不访问 guest memory；DR 恢复普通 FIFO 语义 | 无额外资源变化 |
| `has-xip=false` | 专用 XIP 寄存器组 RAZ/WI；共享 `SPI_CTRLR0` 仍按 enhanced profile 可见 | 当前模型没有实现独立 XRXO/SPITE 输出，不新增 IRQ | `xip_enabled=false`；GPIO 电平被忽略；无 XIP transaction | 不创建第二个 MMIO region |

### 4.3 `has-enhanced-spi`

需要同时门控三个位置：

1. `CTRLR0` 读写 mask：关闭时清除 `SPI_FRF` 和 `SPI_HYPERBUS_EN`；
2. `SPI_CTRLR0` / `DDR_DRIVE_EDGE` 寄存器存在性；
3. `dw_ssi_run_transfer()`：只有 capability 开启且 `SPI_FRF != 0` 才进入 enhanced engine。

`dw_ssi_get_spi_mode()` 在 capability 关闭时返回 0，保证 HI_SYS 不会观察到不存在的 enhanced mode。

### 4.4 `has-idma`

`dw_ssi_idma_enabled()` 成为 IDMA 的单一运行时入口：

```c
static bool dw_ssi_idma_enabled(const DwSsiState *s)
{
    return dw_ssi_has_capability(s, DW_SSI_CAP_IDMA) &&
           FIELD_EX32(s->regs[R_DMACR], DMACR, IDMAE);
}
```

同时新增 `dw_ssi_irq_valid_mask()`：TXE、TXO、RXF、RXO、TXU、RXU、MST 始终属于基础 IRQ，只有 `has-idma=true` 时再加入 DONE、AXIE。`dw_ssi_irq_raw_status()`、IMR 读写和 `dw_ssi_update_irq()` 全部使用这一动态 mask，避免仅在数据路径阻止 IDMA、却仍让 guest 写出可见 DONE/AXIE 位。

IDMA 关闭后仍保留 `idma_completed_frames`、enhanced command 等结构字段和 VMState 字段，避免按 capability 条件改变迁移流结构。reset 清空 absent capability 的正常状态；post-load 遇到与 capability 冲突的迁移状态时拒绝加载，不静默修改迁移流。

### 4.5 `has-xip`

XIP capability 分为三层，不得混淆：

1. `has-xip`：实例是否具有专用 XIP 寄存器和 XIP MMIO 访问接口；
2. `xip-window-size`：region 大小，必须与 `has-xip` 同时存在；
3. `xip-enable` GPIO：运行时访问开关，不决定 region 是否创建。

`has-xip=false` 时：

- `XIP_MODE_BITS`、`XIP_INCR_INST`、`XIP_WRAP_INST` 及未实现扩展寄存器全部 RAZ/WI；
- `dw_ssi_xip_enable_handler()` 强制保持 `xip_enabled=false`；
- `dw_ssi_xip_read()` 不可达，因为没有第二个 MemoryRegion；
- machine 不映射 region 1。

`has-xip=true` 时只恢复当前已有的 XIP read window 行为；XIP write 仍记录 guest error，concurrent XIP/dynamic wait/write-register 扩展仍保持 RAZ/WI。

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
    bool has_enhanced_spi;
    bool has_idma;
    bool has_xip;
    uint64_t xip_window_size;
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
        .has_enhanced_spi = true,
        .has_idma = true,
        .has_xip = false,
        .xip_window_size = 0,
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
        .has_enhanced_spi = true,
        .has_idma = true,
        .has_xip = false,
        .xip_window_size = 0,
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
        .has_enhanced_spi = true,
        .has_idma = true,
        .has_xip = true,
        .xip_window_size = 0x08000000,
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
    qdev_prop_set_bit(dev, "has-enhanced-spi",
                      profile->has_enhanced_spi);
    qdev_prop_set_bit(dev, "has-idma", profile->has_idma);
    qdev_prop_set_bit(dev, "has-xip", profile->has_xip);
    qdev_prop_set_uint64(dev, "xip-window-size",
                         profile->xip_window_size);
}
```

在三个 SSI realize 前循环调用，删除当前六个分散的 `num-cs` / `max-lines` 设置。

### 5.3 条件映射 XIP region

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

for (int i = 0; i < ARRAY_SIZE(s->dw_ssi); i++) {
    g_assert(sysbus_has_mmio(SYS_BUS_DEVICE(&s->dw_ssi[i]), 1) ==
             k230_dw_ssi_profiles[i].has_xip);
}
sysbus_mmio_map(SYS_BUS_DEVICE(&s->dw_ssi[2]), 1,
                memmap[K230_DEV_FLASH].base);
```

QSPI profile 的 `has-xip=false` 会使其只暴露 region 0。region 数量与 profile 的断言在 Step 4.3.3 随 XIP 资源门控一起加入；Step 4.2 只先停止映射 QSPI region 1，不能提前要求该断言成立。HI_SYS 的 `xip-enable` GPIO 仍只连接 `dw_ssi[2]`，与 Step 3 的依赖方向保持一致。

---

## 6. capability 负路径测试载体决策

### 6.1 选择独立通用 qtest

K230 三实例均具备 enhanced SPI 和 IDMA，无法覆盖这两项 capability 的 `false`。本计划采用独立通用测试机，不向 K230 产品 machine 增加测试专用 property。

新增文件：

| 文件 | 职责 |
|---|---|
| `hw/ssi/dw_ssi-test.c` | `CONFIG_DW_SSI && CONFIG_TEST_DEVICES` 下注册无 CPU 的最小 `dw-ssi-test` machine，实例化一个 `TYPE_DW_SSI` |
| `tests/qtest/dw-ssi-test.c` | 通过 `-preconfig` + QMP `qom-set` 建立不同配置，验证 property、FIFO、RAZ/WI、IRQ 和 XIP region 生命周期 |
| `hw/ssi/meson.build` | 条件编译测试 machine |
| `tests/qtest/meson.build` | 构建并注册 `dw-ssi-test` |

测试 machine 只做三件事：创建一个 DW SSI、realize、映射 region 0；若 `sysbus_has_mmio(sbd, 1)` 为真，再把 region 1 映射到固定测试地址。它不连接 K230 HI_SYS、PLIC 或 Flash，不包含 K230 常量。

```c
#include "qemu/osdep.h"
#include "qapi/error.h"
#include "qemu/module.h"
#include "hw/core/boards.h"
#include "hw/core/sysbus.h"
#include "hw/ssi/dw_ssi.h"

#define DW_SSI_TEST_MMIO_BASE 0x10000000
#define DW_SSI_TEST_XIP_BASE  0x20000000

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
    if (sysbus_has_mmio(sbd, 1)) {
        sysbus_mmio_map(sbd, 1, DW_SSI_TEST_XIP_BASE);
    }
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

测试机不创建 CPU、RAM、PLIC 或 Flash。控制器 child 必须在 machine `instance_init` 创建，而不是在 `MachineClass::init` 才创建：这样正常启动仍由 machine init realize 和映射；`-preconfig` 启动时则已经存在未 realize 的 `/machine/dw-ssi`，非法 property 测试可以通过 QMP 触发 realize 并接收正常 error response。region 1 是否存在只由控制器 realize 后的 `sysbus_has_mmio()` 决定。

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
    bool is_bool;
} DwSsiTestProperty;

static void dw_ssi_qom_set_property(QTestState *qts,
                                    const DwSsiTestProperty *prop)
{
    QDict *args = qdict_new();

    qdict_put_str(args, "path", "/machine/dw-ssi");
    qdict_put_str(args, "property", prop->name);
    if (prop->is_bool) {
        qdict_put_bool(args, "value", prop->value);
    } else {
        qdict_put_int(args, "value", prop->value);
    }
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

所有通用测试都从 `-preconfig` 开始，通过 QMP 设置本用例需要的 properties。有效配置执行 `x-exit-preconfig`，由 machine init realize 控制器并映射 region；非法配置直接对 child 设置 `realized=true` 并断言 QMP error。properties 尚未实现时，setup `qom-set` 先失败；properties 存在但行为或校验尚未实现时，后续断言失败，两种状态都能形成明确的 TDD 红灯且不会阻塞 QMP。

注册非法路径使用 `/dw-ssi/config/invalid/<case>`；例如 `num-cs-zero`、`fifo-depth-too-large`、`idma-without-enhanced`。每个 data case 列出属性数组；`error_text` 只省略设备 canonical path，保留核心错误文本。

典型配置：

```c
static const DwSsiTestProperty standard_only[] = {
    { "max-lines", 1, false },
    { "has-enhanced-spi", false, true },
    { "has-idma", false, true },
    { "has-xip", false, true },
    { "xip-window-size", 0, false },
};

static const DwSsiTestProperty enhanced_no_idma[] = {
    { "max-lines", 4, false },
    { "has-enhanced-spi", true, true },
    { "has-idma", false, true },
    { "has-xip", false, true },
    { "xip-window-size", 0, false },
};

static const DwSsiTestProperty full_xip[] = {
    { "max-lines", 4, false },
    { "has-enhanced-spi", true, true },
    { "has-idma", true, true },
    { "has-xip", true, true },
    { "xip-window-size", 0x01000000, false },
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

## 8. Step 4.1：建立配置骨架并保持当前行为

### 目标

先引入配置表达、校验、动态 FIFO 和通用测试载体，但默认值保持当前 behavior。此阶段不让 capability 改变寄存器可见性；现有 K230 machine 即使尚未设置新 property，12 项 qtest 也必须保持通过。

### TDD 顺序

1. 新增 `dw-ssi-test` 最小 machine 和 qtest 骨架；
2. 先写 property 默认值、`fifo-depth=8` 和非法取值测试，确认当前代码因 property 不存在或 FIFO 仍为 256 而失败；
3. 增加 `DwSsiConfig` 和 properties；
4. 增加 `dw_ssi_validate_config()`；
5. 把 FIFO 创建移到 realize，并替换所有固定容量判断；
6. reset 改从 `cfg` 读取，但默认值与当前常量一致；
7. 运行通用定向 qtest 和现有 K230 12 项回归。

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
| `has-enhanced-spi=off` | `max-lines=4` | `max-lines must be 1 without enhanced SPI` |
| `has-idma=on` | `has-enhanced-spi=off,max-lines=1,has-xip=off,xip-window-size=0` | `IDMA requires the current enhanced SPI engine` |
| `has-xip=on` | `has-enhanced-spi=off,max-lines=1,has-idma=off,xip-window-size=16M` | `XIP requires the enhanced SPI engine` |
| `has-xip=on` | `xip-window-size=0` | `has-xip and xip-window-size must agree` |
| `has-xip=off` | `xip-window-size=16M` | `has-xip and xip-window-size must agree` |
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

- `DwSsiConfig` 与完整 property 集合存在；
- `fifo-depth=8` 的通用测试通过；
- 默认启动 K230 的 12 项 qtest 全部通过；
- capability 仍未改变寄存器可见性，避免在同一小目标混入门控行为。

---

## 9. Step 4.2：应用 K230 三实例 profile

### 目标

证明同一 `TYPE_DW_SSI` 可以表达 QSPI0、QSPI1、FMC 的实例差异，通用模型不再通过 `max-lines` 或 K230 常量猜测 reset。

### TDD 顺序

1. 扩展 `K230SsiInstance` 测试表，先写三实例完整 expected profile；
2. 让 `test_register_contract()` 对每个实例读取 IMR、AXI burst、IDR、VERSION、`SPI_CTRLR0`；
3. 新增 `/k230-dw-ssi/fifo-depth`，实际写入 256 帧并验证第 257 帧不增加 level；
4. 通过 QMP `qom-get` 验证每个 child 的 `fifo-depth`、`max-lines` 和三项 capability property；
5. 在 `k230.c` 增加 profile 数组和设置 helper；
6. 删除 reset 中 `max_lines == 8` 的 FMC 推断；
7. 仅为 FMC 映射 region 1；
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

布尔 property 使用 `qtest_qom_get_bool()`；数值 property 使用 `qom-get` QMP response 中的 `QNum`。这样可以直接验证 FMC 的 `max-lines=8`，而不把 Octal 数据路径实现错误地纳入 Step 4。

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

- K230 machine 显式设置全部 property；
- QSPI0/QSPI1/FMC 的 reset profile 与配置矩阵一致；
- 5/5/1 个 CS 和 256 深度 FIFO 有行为断言；
- K230 只映射 FMC 的 XIP region；QSPI 的 `has-xip=false` 已写入 profile，但“不创建 region 1”的资源门控留到 Step 4.3.3；
- 现有 12 项测试仍全过。

---

## 10. Step 4.3：逐项接入 capability 门控

三个 capability 必须形成三个独立、可验证的小改动。每项先在 `dw-ssi-test` 写 `false` 负路径，再实现门控，最后运行 K230 正路径回归。

### 10.1 `has-enhanced-spi`

#### 失败测试

`/dw-ssi/capability/enhanced-off`：

- 启动 Standard-only profile；
- 向 `CTRLR0.SPI_FRF` 写 Quad，读回仍为 Standard；
- 向 `SPI_CTRLR0`、`DDR_DRIVE_EDGE` 写全 1，读回 0；
- 配置 Standard loopback，PIO 收发仍正常；
- 尝试 enhanced command 后 TX FIFO 不被 enhanced engine 消费；
- `CTRLR0.SPI_FRF` 读回 Standard，间接证明 `dw_ssi_get_spi_mode()` 的输入已收敛为 0；不为内部 getter 增加测试专用 property。

#### 最小实现

- `dw_ssi_reg_present()` 门控 `SPI_CTRLR0`、`DDR_DRIVE_EDGE`；
- CTRLR0 read/write mask 动态清除 enhanced 字段；
- `dw_ssi_run_transfer()` 在 capability 关闭时只走 Standard path；
- reset/post-load 清 enhanced phase 和 command 状态。

#### 正路径回归

```bash
TMPDIR=/tmp/qemu-k230-qtest-v2-step4 \
QTEST_QEMU_BINARY=$PWD/build/qemu-system-riscv64 \
build/tests/qtest/dw-ssi-test \
  -p /dw-ssi/capability/enhanced-off -v
TMPDIR=/tmp/qemu-k230-qtest-v2-step4 \
QTEST_QEMU_BINARY=$PWD/build/qemu-system-riscv64 \
build/tests/qtest/k230-dw-ssi-test \
  -p /k230-dw-ssi/qspi-config -v
TMPDIR=/tmp/qemu-k230-qtest-v2-step4 \
QTEST_QEMU_BINARY=$PWD/build/qemu-system-riscv64 \
build/tests/qtest/k230-dw-ssi-test \
  -p /k230-dw-ssi/qspi-sdr -v
```

### 10.2 `has-idma`

#### 失败测试

`/dw-ssi/capability/idma-off`：

- `DMACR`、`AXIAWLEN`、`AXIARLEN`、`SPIDR`、`SPIAR`、`AXIAR0/1` 写后读回 0；
- `AXIECR`、`DONECR` 读取为 0；
- 向 IMR 写 DONE/AXIE 位，读回仍为 0；TXU 位仍可写入和读回；
- 尝试写 `IDMAE`、SSIENR、SER 后不访问 guest memory；
- `SR.CMPLTD_DF == 0`；
- 用 `qtest_irq_intercept_out()` 拦截 `/machine/dw-ssi`，DONE/AXIE 两路保持低；再制造 TX FIFO underflow，确认 TXU raw status 和输出仍可见；
- DR 写入继续进入普通 TX FIFO，证明关闭 IDMA 没有破坏 PIO。

#### 最小实现

- 寄存器分组统一返回 RAZ/WI；
- `dw_ssi_idma_enabled()` 加 capability 条件；
- 动态 IRQ mask只排除 DONE/AXIE；
- reset 清 `idma_completed_frames` 和 DONE/AXIE latch；post-load 遇到冲突状态返回错误；
- 不删除 IDMA 字段，不条件化 VMState。

#### 正路径回归

```bash
TMPDIR=/tmp/qemu-k230-qtest-v2-step4 \
QTEST_QEMU_BINARY=$PWD/build/qemu-system-riscv64 \
build/tests/qtest/dw-ssi-test \
  -p /dw-ssi/capability/idma-off -v
TMPDIR=/tmp/qemu-k230-qtest-v2-step4 \
QTEST_QEMU_BINARY=$PWD/build/qemu-system-riscv64 \
build/tests/qtest/k230-dw-ssi-test \
  -p /k230-dw-ssi/idma -v
```

### 10.3 `has-xip`

#### 失败测试

`/dw-ssi/capability/xip-off`：

- `XIP_MODE_BITS`、`XIP_INCR_INST`、`XIP_WRAP_INST` 和 XIP 扩展寄存器全部 RAZ/WI；
- QMP `qom-list /machine/dw-ssi` 中没有名为 `designware-ssi.xip[0]`、类型为 `child<memory-region>` 的条目；
- 调用 `qtest_set_irq_in(qts, "/machine/dw-ssi", "xip-enable", 0, 1)` 后仍无 XIP region，专用寄存器与数据路径不产生可见效果；
- system reset 后行为不变。

`/dw-ssi/capability/xip-on-resource`：

- 使用 `has-xip=on,xip-window-size=16M`；
- QMP `qom-list /machine/dw-ssi` 中存在 `designware-ssi.xip[0]`，类型为 `child<memory-region>`；
- 测试 machine 能映射 region 1；
- XIP 未 enable 时读取测试窗口返回 0；
- 本测试不挂 Flash，不替代 K230 的完整 XIP transaction 正路径。

`designware-ssi.xip[0]` 不是推测名称：当前源码以 `TYPE_DW_SSI ".xip"` 调用 `memory_region_init_io()`，并已用当前构建的 QMP `qom-list` 复核。测试按 `name` 和 `type` 搜索 QList 条目，不依赖返回顺序。

#### 最小实现

- `realize()` 条件创建第二个 MemoryRegion；
- 专用 XIP 寄存器分组 RAZ/WI；
- GPIO handler 在 capability 关闭时保持 `xip_enabled=false`；
- K230 只映射 FMC region 1；
- 不改变 XIP write 与未实现扩展的现有边界。

#### 正路径回归

```bash
TMPDIR=/tmp/qemu-k230-qtest-v2-step4 \
QTEST_QEMU_BINARY=$PWD/build/qemu-system-riscv64 \
build/tests/qtest/dw-ssi-test \
  -p /dw-ssi/capability/xip-off -v
TMPDIR=/tmp/qemu-k230-qtest-v2-step4 \
QTEST_QEMU_BINARY=$PWD/build/qemu-system-riscv64 \
build/tests/qtest/dw-ssi-test \
  -p /dw-ssi/capability/xip-on-resource -v
TMPDIR=/tmp/qemu-k230-qtest-v2-step4 \
QTEST_QEMU_BINARY=$PWD/build/qemu-system-riscv64 \
build/tests/qtest/k230-dw-ssi-test \
  -p /k230-dw-ssi/xip-read-window -v
```

### Step 4.3 完成标准

- 三个 capability 都有 `false` 负路径；
- K230 对应正路径仍通过；
- capability 关闭时寄存器、IRQ、状态、数据路径和资源行为均有断言；
- 每项 capability 可单独 review，不依赖下一项修复前一项。

---

## 11. VMState 与迁移边界

### 11.1 最小配置 equality

QOM properties 在 realize 前确定，和 machine type/profile 一起由源端、目的端分别创建。它们不作为 guest 可变状态迁移，但以下两项会改变动态状态的解释，必须作为 equality guard 放在依赖字段之前：

```c
VMSTATE_UINT32_EQUAL(cfg.fifo_depth, DwSsiState),
VMSTATE_UINT32_EQUAL(cfg.capabilities, DwSsiState),
```

`VMSTATE_FIFO32` 使用目的端已经创建的 FIFO capacity 解释数据，因此 `fifo-depth` 不一致必须拒绝。capability profile 决定寄存器、IRQ 和活动数据路径是否存在，也必须整体一致。`num-cs`、ID、VERSION、reset 值和 `xip-window-size` 不在本步增加 equality：K230 machine 已固定这些配置，它们也不改变当前 VMState 字段的内存解释。

### 11.2 保持 VMState 字段列表稳定

Step 4 不按 capability 条件删除以下字段：

- `regs[]`；
- TX/RX FIFO；
- `idma_completed_frames`；
- enhanced command/phase；
- `xip_enabled`。

这样 K230 三实例虽然 capability 不同，仍使用同一 VMState schema。reset 负责清空 absent capability 的正常状态；post-load 只验证迁移状态，不通过静默清零掩盖配置或状态冲突：

```c
static int dw_ssi_validate_loaded_state(DwSsiState *s)
{
    if (!dw_ssi_has_capability(s, DW_SSI_CAP_ENHANCED_SPI) &&
        (s->phase >= DW_SSI_PHASE_ENHANCED_INSTRUCTION ||
         s->remaining_frames != 0)) {
        return -EINVAL;
    }
    if (!dw_ssi_has_capability(s, DW_SSI_CAP_IDMA) &&
        (s->idma_completed_frames != 0 ||
         (s->irq_latched & (R_RISR_DONER_MASK |
                            R_RISR_AXIER_MASK)))) {
        return -EINVAL;
    }
    if (!dw_ssi_has_capability(s, DW_SSI_CAP_XIP) && s->xip_enabled) {
        return -EINVAL;
    }
    return 0;
}
```

TXU 不在 IDMA 冲突检查中，因为它是基础 TX FIFO underflow 状态。`dw_ssi_post_load()` 在现有 phase、active CS 范围检查后调用该函数；全部验证通过后再恢复 CS 和 IRQ 电平。

### 11.3 兼容性结论

当前 V2 分支尚未上游发布，Step 4 不承诺与未发布中间态进行跨版本迁移。Plan Final 在最终 schema 中加入两项 equality，但不按 capability 增加条件字段，也不把全部 cfg 参数塞进迁移流。测试只覆盖同 profile 成功、FIFO 深度不一致失败、capability profile 不一致失败三种必要情况。

---

## 12. TDD 测试矩阵

| 测试路径 | 配置 | 核心断言 |
|---|---|---|
| `/dw-ssi/config/defaults` | 默认通用实例 | property 默认值保持当前行为 |
| `/dw-ssi/config/fifo-depth` | `fifo-depth=8` | 8 帧满，第 9 帧不增长，阈值边界，reset 清空 |
| `/dw-ssi/config/invalid/*` | `-preconfig` 下每个路径一个非法组合 | QMP `realized=true` 返回精确配置错误 |
| `/dw-ssi/capability/enhanced-off` | enhanced/idma/xip 全关 | enhanced 寄存器 RAZ/WI，Standard PIO 正常 |
| `/dw-ssi/capability/idma-off` | enhanced 开、IDMA/XIP 关 | IDMA 寄存器 RAZ/WI，DONE/AXIE 无效；制造 TX FIFO underflow 后 TXU 仍可见 |
| `/dw-ssi/capability/xip-off` | enhanced/IDMA 开、XIP 关 | XIP 寄存器 RAZ/WI，无第二 region |
| `/dw-ssi/capability/xip-on-resource` | XIP 开、16 MiB | 创建并映射第二 region，未 enable 读 0 |
| `/dw-ssi/migration/same-profile` | 相同 FIFO/capability | 迁移成功，FIFO、寄存器和 IRQ 状态恢复 |
| `/dw-ssi/migration/fifo-depth-mismatch` | 8 → 16 | equality 拒绝迁移 |
| `/dw-ssi/migration/capability-mismatch` | capability profile 不同 | equality 拒绝迁移 |
| `/k230-dw-ssi/enhanced-xip-isolation` | 普通 `0xeb` + 非零 XIP fields | ordinary enhanced 忽略 XIP mode fields，数据不偏移 |
| `/k230-dw-ssi/idma-quad-io` | SDK 风格 `0xeb` IDMA | mode/dummy 只由 `WAIT_CYCLES` 表达，数据正确 |
| `/k230-dw-ssi/register-contract` | K230 三实例 | 完整 reset profile、SER 位宽、properties |
| `/k230-dw-ssi/fifo-depth` | K230 256 深度 | 256 帧满，第 257 帧不增长 |
| `/k230-dw-ssi/qspi-config` | FMC enhanced 配置 | 非法 Octal/DDR 仍拒绝，Quad SDR 正常 |
| `/k230-dw-ssi/idma` | FMC IDMA | DONE/AXIE 正负路径保持通过 |
| `/k230-dw-ssi/xip-read-window` | FMC XIP 128 MiB | enable 前 0，enable 后 Flash 数据，窗口和 PIO 并存 |
| `/k230-dw-ssi/hi-sys` | K230 HI_SYS | mode/sleep 和 XIP enable GPIO 连接不回退 |
| K230 WDT qtest | K230 SoC | SSI 资源数量变化不破坏其他设备 realize/mapping |

系统 reset 后至少重新运行：三实例 reset profile、FIFO 清空、capability RAZ/WI、FMC XIP enable 默认低电平。

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

成功标准：通用测试全部 PASS；K230 当前 12 项加新增 profile/FIFO 测试全部 PASS；WDT qtest PASS；无 FAIL/ERROR/SKIP。

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

- [ ] 新增 `DwSsiConfig`，把 `num_cs`、`max_lines` 和内部 `capabilities` 收入 `cfg`
- [ ] 增加全部 QOM properties 和兼容默认值
- [ ] 增加 `dw_ssi_validate_config()`
- [ ] 增加独立 `dw-ssi-test` machine、Meson 注册与 qtest
- [ ] 非法配置按 case 使用 `-preconfig` + QMP realize error 验证
- [ ] FIFO 按 `fifo-depth` 在 realize 创建
- [ ] 所有 FIFO 容量与阈值逻辑改用 `cfg.fifo_depth`
- [ ] 保留 finalize 统一销毁 FIFO 和动态 CS 数组
- [ ] reset 从 `cfg` 读取，默认行为不变
- [ ] 增加 `fifo-depth` 和 `capabilities` 两项 VMState equality
- [ ] 同 profile 迁移成功，FIFO/capability 不一致迁移失败
- [ ] 通用 config qtest 与 K230 12 项回归通过

### Step 4.2：K230 profile

- [ ] 在 `k230.c` 增加三实例完整 profile
- [ ] 三实例 realize 前显式设置全部 property
- [ ] 删除 `max_lines == 8` 推断 FMC reset
- [ ] QSPI0/QSPI1 不映射 XIP region，profile 显式设置 `has-xip=false`
- [ ] FMC profile 设置 128 MiB XIP window，并把现有 region 1 映射到 `0xc0000000`
- [ ] 三实例 reset、SER、FIFO、property 测试通过

### Step 4.3：capability

- [ ] `has-enhanced-spi=false` 的 RAZ/WI 与 Standard PIO 负路径通过
- [ ] `has-idma=false` 的寄存器、状态、guest memory、IRQ 负路径通过
- [ ] `has-idma=false` 时 DONE/AXIE 无效，但 TXU 仍可产生和上报
- [ ] `has-xip=false` 的寄存器、GPIO、region 负路径通过
- [ ] `has-xip=true` 的资源创建和 K230 XIP 正路径通过
- [ ] QSPI0/QSPI1 最终不创建 region 1，FMC 按 128 MiB property 创建 region 1
- [ ] 三项门控分别完成局部与完整回归

### Step 4.4：收敛

- [ ] 通用 qtest、K230 SSI qtest、K230 WDT qtest 全过
- [ ] 公共头文件独立包含通过
- [ ] 通用 SSI 无 K230 依赖
- [ ] VMState 字段列表不按 capability 分叉，post-load 对冲突状态返回错误而非静默 sanitize
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
  └── 把 IRQ 输出连接到 PLIC
```

这批 series 的核心 review 问题只有两个：

1. `dw_ssi.c` 是否已经成为不依赖 K230 类型、地址和 HI_SYS 的通用设备模型；
2. K230 是否只负责实例配置、地址映射和 PLIC 接线。

第一批不以 SDK U-Boot 从 SPI NOR 启动作为合入条件，因为 SPI NOR、enhanced SPI 和 IDMA 不在本批范围。完成标准是通用寄存器/FIFO/PIO/IRQ qtest 与 K230 实例/PLIC 集成 qtest 全部通过。

### 15.2 第一批包含和不包含的内容

| 类别 | 第一批包含 | 第一批不包含 |
|---|---|---|
| 通用设备 | `TYPE_DW_SSI`、SSI bus、控制器 region 0、CS outputs、realize/finalize、reset、基础 VMState | K230 wrapper、HI_SYS 指针、K230 地址常量 |
| 配置 | `num-cs`、`fifo-depth`、`component-id`、`version-id`、`imr-reset` 及必要校验 | `max-lines`、`spi-ctrlr0-reset`、AXI burst reset、enhanced/IDMA/XIP capability properties |
| 数据路径 | Standard single-line SPI，四种 TMOD、frame width、loopback、FIFO level/status | Dual/Quad/Octal、enhanced command、SPI NOR、IDMA、XIP transaction |
| IRQ | TXE、TXO、RXF、RXO、TXU、RXU、MST 的基础语义和 K230 PLIC 路由 | DONE/AXIE 的产生和清除逻辑 |
| 资源 | 固定控制器 MMIO、SSI bus、CS 和 IRQ 接口 | XIP region、`xip-enable` GPIO、Flash attachment |
| 测试 | 通用寄存器/PIO/IRQ 测试，K230 三实例 profile、MMIO 和 PLIC 隔离测试 | enhanced、Flash、IDMA、HI_SYS、XIP 正负路径 |

`TXU` 属于基础 TX FIFO underflow，必须随第一批 IRQ 一起实现。DONE、AXIE 属于 IDMA；为保持 K230 物理接线和 QOM GPIO output 数量稳定，可以在第一批注册并路由完整 9 路输出，但 DONE/AXIE 必须恒低，其 IMR/ISR/RISR 位在 IDMA series 到来前不得表现为有效功能。

### 15.3 “寄存器占位但不提供功能”的精确定义

第一批可以保留完整控制器 MMIO aperture，但不能通过一组半实现的寄存器暗示 enhanced、IDMA 或 XIP 已受支持。占位采用以下规则：

1. **地址空间占位**：region 0 大小保持稳定；未实现 offset 统一 RAZ/WI，不为每个未来寄存器增加空 case；
2. **基础寄存器真实实现**：Standard PIO、FIFO、状态、基础 IRQ、IDR/VERSION 所依赖的寄存器必须具有明确 reset、write mask 和副作用；
3. **扩展寄存器不伪装功能**：`SPI_CTRLR0` 扩展字段、internal DMA/AXI、XIP 专用寄存器在对应 series 前不进入有效数据路径；
4. **有证据才暴露非零契约**：如果 firmware 枚举确实需要读取某个扩展寄存器 reset，可以单独实现只读/reset 语义并写测试，但 commit message 必须声明“寄存器可见不等于数据路径已实现”；
5. **不提前增加未来 property**：property 随首次使用它的功能 patch 引入，避免 Patch 1 出现没有消费者的 capability scaffolding。

因此，“先占位”推荐表达为：**MMIO aperture 已存在，未实现扩展 offset RAZ/WI**；不推荐提前复制完整寄存器表、写入全部 K230 reset 值，却让实际功能留到数个 series 之后。

### 15.4 第一批仍需具备的通用基础

除了 PIO 和 IRQ，第一批还需要以下基础，否则“通用 DW SSI”拆分仍不完整：

- `DwSsiConfig` 的第一批最小子集和 realize 前配置校验；
- FIFO、CS 动态资源的明确创建和销毁生命周期；
- 基础 VMState：寄存器、FIFO、PIO phase、remaining frames、IRQ latch、active CS；
- 固定且有文档说明的 SSI bus、CS output 和 IRQ output 接口；
- 通用 qtest 载体，用于证明 PIO/IRQ 语义不依赖 K230 machine；
- K230 集成 qtest 只验证三实例 profile、地址映射、PLIC source 和实例隔离，避免与通用测试重复；
- 非法 `num-cs`、`fifo-depth` 等配置在 realize 阶段返回清晰错误。

第一批不需要 trace events、SPI NOR 或启动镜像。trace 可以随对应功能加入，避免单独增加只为调试服务的 review 面积。

### 15.5 第一批 Commit 顺序

第一批建议保持五个可编译、可回归的提交，每个提交只增加一类职责：

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

测试应跟随首次实现该行为的 commit，不把所有测试集中到最后一个 patch。每个 commit 完成后至少执行增量构建、对应定向 qtest 和 `git diff --check`；完成整个 series 后再运行完整通用/K230 qtest。

本地开发 commit 可以按 Step 4.0、4.1、4.2 等小目标逐步积累；发送上游前再重组到上述五个职责提交。Step 4.0 当前提交属于本地纠错历史：第一批 series 不包含 enhanced/IDMA，因此不需要单独携带这个修复 patch；后续 enhanced/IDMA series 应从首次出现开始就是正确实现，不能先引入错误 mode phase 再补修复。

### 15.6 后续 Series

第一批合入或架构 review 基本稳定后，再顺序发送：

1. enhanced SPI + `max-lines` / `spi-ctrlr0-reset` + SPI NOR；
2. optional internal DMA + AXI 配置 + DONE/AXIE；
3. K230 HI_SYS + optional XIP region、GPIO 和 XIP transaction。

后续 series 不应同时并发发送，否则 reviewer 同一时间仍需理解完整三千余行改动，失去分批投稿的意义。

---

## 16. 与最终 V2 patch 系列的关系

Step 4 的实现内容在第五步重组时按职责回填，不形成一个覆盖所有功能的“大配置 patch”：

| Step 4 内容 | 最终 patch 归属 |
|---|---|
| `DwSsiConfig`、基础 properties、配置校验、通用测试 machine | Patch 1：通用寄存器模型与基础配置 |
| K230 三实例 profile、控制器 MMIO | Patch 2：K230 实例化通用控制器 |
| 动态 FIFO 深度 | Patch 3：FIFO/PIO |
| 动态 IRQ mask 的基础部分 | Patch 4：通用 IRQ |
| `has-enhanced-spi` 门控 | Patch 6：enhanced SPI |
| `has-idma` 门控 | Patch 8：optional IDMA |
| `has-xip`、可选 region、K230 `0xc0000000` 映射 | Patch 10：optional XIP |

每个最终 patch 必须包含自己需要的 property、测试和 machine 配置，不允许 Patch 8 或 Patch 10 修复 Patch 1 已经引入的不可编译/不可 realize 中间态。

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
