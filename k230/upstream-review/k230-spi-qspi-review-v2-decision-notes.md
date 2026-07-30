# K230 SPI/QSPI 上游 Review 与 v2 重构决策（合并版）

首次记录：2026-07-27

合并更新：2026-07-30（同步 Step 4 Plan Final V1.3 第一批最小消费者范围）

适用分支：`qemu-camp-2026-k230/k230-spiv3.4`
问题来源：Bin Meng 对第 1 个 SSI patch 的 review
文档状态：v2 唯一决策入口，不代表 v2 代码已经实现

本文合并并取代以下两份阶段性分析：

- `k230-spi-qspi-review-v2-decision-notes.md` 原始调查记录；
- `k230-spi-qspi-dwssi-split-analysis.md` 通用层与 K230 层拆分分析。

文末保留原始记录用于追溯；实施和 review 回复应以本文前半部分的“统一结论”为准。

## 1. 最终结论摘要

下一轮上游 v2 的核心不是把 `k230_dw_ssi.c` 简单改名，而是将当前模型重构为：

1. 一个可被其他 SoC 复用的 Synopsys DesignWare SSI 通用模型；
2. K230 machine 对三个 SSI 实例的配置、地址映射和外围连接；
3. 独立的 K230 HI_SYS 模型；
4. 默认不引入 `TYPE_K230_DW_SSI` wrapper。

推荐的依赖关系为：

```text
TYPE_DW_SSI
  ├── 标准寄存器、FIFO、PIO、TMOD
  ├── 通用 IRQ
  ├── optional enhanced SPI
  ├── optional internal DMA
  ├── optional XIP engine + XIP MMIO region
  └── properties/capabilities

K230 machine
  ├── 配置三个 TYPE_DW_SSI 实例
  ├── 映射控制器地址
  ├── 连接 PLIC
  ├── 挂接 SPI NOR
  ├── 将 XIP region 映射到 0xc0000000
  └── 连接 HI_SYS 与 SSI 的抽象信号
```

只有在后续逐项审阅中发现无法通过 property、capability、GPIO、QOM link 或通用查询接口表达的 K230 控制器语义时，才增加薄 wrapper。

### 1.1 Step 4 Plan Final 裁决

Step 4 的唯一执行入口是 [Standard PIO 第一批范围 Plan Final V1.3](k230-spi-qspi-v2-step4-plan-final-instance-configurationV1.3.md)。V1.2、Plan A、Plan B、Plan C、原 Plan Final 及其 planning handoff 只保留为历史推演材料。

TRM 与 SDK 复核后，最终边界如下：

- QSPI 12.3 寄存器表从 `0x0f8 DDR_DRIVE_EDGE` 直接跳到 `0x118 SPI_CTRLR1`，不包含 `XIP_MODE_BITS`、`XIP_INCR_INST`、`XIP_WRAP_INST`；三者只在 FMC 5.3 XIP 寄存器表中定义；
- RT-Smart 公共寄存器结构体包含三个成员，只能证明最大布局和偏移占位；RT-Smart、U-Boot、Linux 的普通 enhanced/IDMA 路径均没有写 `XIP_MODE_BITS`；
- 当前 `dw_ssi_decode_enhanced_command()`、普通 enhanced mode phase，以及 IDMA 1-4-4 特判错误地消费 XIP fields。Step 4.0 必须先把普通事务恢复为 instruction → address → dummy → data；真正 XIP transaction 才允许插入 mode；
- 硬件证据保持 QSPI0/QSPI1 无 XIP、SPI-OPI/FMC 具备 XIP，但第一批不提交任何 XIP property、内部位、GPIO、第二个 MMIO region 或 K230 aperture 映射；XIP-only 寄存器统一 RAZ/WI；
- TXU 是 TX FIFO underflow，属于第一批七路基础 IRQ。DONE、AXIE output、状态和 PLIC 路由全部随 IDMA series 引入，不在第一批注册后恒低；
- 第一批不提交 DMA register layout 或 DMA 寄存器存储语义。SDK Standard PIO 在 DMA offset RAZ/WI 返回 0 时可以正常工作；实际非零写入属于 DMA/IDMA 路径；
- 第一批 `DwSsiConfig` 只保留 `num_cs`、`fifo_depth`、`imr_reset`，迁移只对 `fifo-depth` 做 equality；future property、状态和迁移约束随各自首个真实消费者引入。

## 2. Review 要求与重构目标

Bin Meng 要求把当前 K230 SSI 模型拆分为：

- 通用的 Synopsys DesignWare SSI 控制器模型；
- 可选的 K230-specific wrapper，仅在通用层不足以支持 K230 软件时保留。

Review 的本质要求是复用边界，而不是文件命名：

- DWC SSI IP 自身的寄存器和数据通路不得带 K230 依赖；
- IP 综合配置应通过实例配置表达；
- K230 物理地址、PLIC、HI_SYS 和 Flash 接线留在 SoC/machine；
- 不应仅因为某个能力目前只在 K230 上使用，就把它命名为 K230-specific。

v2 应避免声称完整覆盖全部 `DW_apb_ssi` 或 `DWC_ssi` 变体。更准确的范围是：

> 实现由 K230 TRM、公开通用驱动和实际软件路径共同证明的 DWC SSI 子集，并用 capability 表达 K230 实例启用的可选能力。

## 3. IP 身份与关键证据

### 3.1 K230 使用 Synopsys DWC SSI IP

K230 TRM 的 `SSIC_VERSION_ID` 描述明确提到 Synopsys component version，复位值为：

```text
0x3130332a  ->  "103*"
```

TRM 还大量使用以下综合参数名：

- `SSIC_HAS_RX_SAMPLE_DELAY`、`SSIC_HAS_RXDS`、`SSIC_HAS_TX_RX_EN`；
- `SSIC_SPI_MODE`、`SSIC_XIP_EN`、`SSIC_XIP_WRITE_EN`；
- `SSIC_XIP_WRITE_REG_EN`、`SSIC_XIP_CONT_XFER_EN`；
- `SSIC_AXI_DW`、`SSIC_DFLT_BAUDR`；
- `SSIC_DFLT_AWLEN`、`SSIC_DFLT_ARLEN`；
- `SSIC_RX_DLY_SR_DEPTH`、`SSIC_DM_EN`。

这些名称表明 K230 TRM 描述的是 Synopsys DWC SSI 在 K230 上的具体综合配置，而不是 K230 自研 SPI 控制器。

### 3.2 SDK U-Boot 驱动是通用 DesignWare 驱动

K230 SDK 的 `drivers/spi/designware_spi.c` 来源于 U-Boot 通用 DesignWare SPI 驱动。K230 SDK 通过：

```c
#define SSIC_HAS_DMA 2   /* Internal DMA */
```

选择内部 AXI DMA 寄存器布局。驱动同时区分：

- `SSIC_HAS_DMA == 1`：external DMA 水位寄存器；
- `SSIC_HAS_DMA == 2`：内部 AXI DMA 寄存器。

因此 IDMA 是 DWC SSI 的可选综合能力，不是 K230 外挂的 SoC DMA 模块。

### 3.3 QEMU 已有 DesignWare 通用模型的组织先例

QEMU 已有 `TYPE_DESIGNWARE_I2C` 通用模型，由具体 SoC 负责实例化和连接。DWC SSI 应采用同样的分层思路：通用 IP 模型加实例配置，而不是默认把整个设备放在 K230 命名空间下。

### 3.4 DWC SSI 家族和版本表述

当前证据包括：

- K230 DTS 使用 `snps,dwc-ssi-1.01a`；
- `SSIC_VERSION_ID` 表现为 `1.03*`；
- TRM FMC 章节使用 `DWC_ssi` 名称；
- Linux/U-Boot 驱动同时覆盖多个 DesignWare SSI 版本。

`snps,dwc-ssi-1.01a` 与 component version `1.03*` 的准确关系仍可继续调查，但不应阻塞架构拆分。v2 使用中性的 `DW_SSI`/`dw-ssi` 命名，并在提交说明中限定实现范围，避免宣称精确模拟所有版本。

## 4. 证据层级与使用原则

| 证据 | 可以证明 | 不能单独证明 |
|---|---|---|
| DWC SSI Databook | IP 架构、寄存器通用语义、可选功能和信号语义 | K230 实例最终启用的综合参数 |
| K230 TRM | K230 暴露的寄存器、复位值、系统连接和可观察行为 | 所有 DWC SSI 实例均具备相同行为 |
| CoreConsultant 配置报告 | K230 对通用 IP 的具体参数选择 | SoC 地址、PLIC、HI_SYS 和 Flash 接线 |
| K230 SDK 驱动 | 软件实际访问顺序和功能路径 | 行为是规范、workaround 还是驱动缺陷 |
| Linux/U-Boot 通用驱动 | 多平台共性、版本和 capability 划分 | K230 的最终硬件配置 |
| 其他 SoC 文档和驱动 | 同一功能的交叉验证 | K230-specific 集成语义 |
| 实机和启动日志 | K230 的真实可观察行为 | 行为是否适用于其他 SoC |

分类时遵循以下标准：

### 4.1 DWC SSI 通用行为

满足至少两项且没有相反证据：

- DWC 文档明确规定；
- Linux/U-Boot 通用驱动在多个平台使用；
- 其他 SoC 实现与 K230 一致；
- 不依赖 K230 地址、HI_SYS 或 SoC-specific 信号。

### 4.2 DWC SSI 可配置能力

DWC 资料将其描述为可选或由综合参数决定，而 K230 TRM/SDK 给出具体取值。这些内容应进入通用模型的 property 或 capability，不应直接写成 `K230_*` 常量。

### 4.3 K230 SoC 集成

明确位于控制器外部：

- 三个 SSI 实例和 SDK 编号；
- 控制器物理地址；
- SSI IRQ 到 PLIC 的路由；
- HI_SYS `SSI_CTRL`；
- SPI NOR 挂接；
- `0xc0000000` XIP aperture 的地址和大小；
- 时钟、复位、IOMUX 等外围控制。

### 4.4 K230-specific quirk

只有同时满足以下条件，才增加 K230 quirk 或 wrapper 行为：

- 通用 DWC 资料没有覆盖或给出不同语义；
- K230 TRM 明确描述，或 SDK/实机可以稳定复现；
- 无法通过正常综合参数或外部连接解释。

单个 SDK 写法、单个复位值或单个 qtest 现象不足以证明 K230 quirk。

## 5. 通用层与 K230 层的最终边界

下表描述长期最终归属，不表示第一批一次提交全部功能。第一批只实现 Standard PIO/FIFO、七路基础 IRQ、K230 三实例/PLIC 和 Standard 1-1-1 NOR；可选 enhanced、DMA/IDMA、XIP 接口随各自后续 series 引入。

| 功能 | 最终归属 | v2 表达方式 |
|---|---|---|
| `CTRLR0/1`、`SSIENR`、`MWCR`、`SER`、`BAUDR` | DWC SSI 通用层 | 基础寄存器模型 |
| FIFO、DR aliases、PIO、四种 TMOD | DWC SSI 通用层 | FIFO 深度可配置 |
| `SR`、`IMR/ISR/RISR`、标准 ICR | DWC SSI 通用层 | IRQ 状态和输出 |
| `IDR`、`VERSION` 和实例复位值 | DWC SSI 通用层或实例配置 | 第一批固定已确认 component/version，仅 `imr-reset` 配置化；出现真实差异时再增加 property |
| Dual/Quad SDR、`SPI_CTRLR0` | DWC SSI 可选能力 | `has-enhanced-spi`、`max-lines` |
| 内部 DMA 寄存器和传输 | DWC SSI 可选能力 | `has-idma` |
| DONE/AXIE 等 IDMA IRQ | DWC SSI 可选能力 | capability 关闭时保持无效 |
| XIP 指令、地址、mode、dummy 生成 | DWC SSI 可选能力 | `has-xip` |
| XIP MMIO region 的访问语义 | DWC SSI 可选能力 | 通用模型第二个 MMIO region |
| XIP region 大小 | 实例/SoC 配置 | `xip-window-size` |
| XIP region 映射到 `0xc0000000` | K230 machine | `sysbus_mmio_map()` |
| XIP enable 控制 | K230 HI_SYS 到 DWC SSI | 命名 GPIO 输入/输出 |
| 三实例地址和 CS 配置 | K230 machine | 第一批设置最小 properties；线宽随 enhanced series 引入 |
| PLIC IRQ 编号和接线 | K230 machine | `sysbus_connect_irq()` |
| SPI NOR 型号和 CS 接线 | K230 machine | SSI bus/CS wiring |
| HI_SYS `SSI_CTRL` 寄存器 | K230-specific misc 设备 | 保留 `k230_hi_sys` |
| trace events | 跟随通用模型或 K230 集成归属 | 通用事件去除 K230 前缀 |

其中“复位值属于 K230 实例配置”不等于“必须由 K230 wrapper 实现”。实例参数属于 SoC 配置，但可通过通用模型 property 传入。

## 6. 推荐的代码结构

### 6.1 文件和 QOM 类型

```text
hw/ssi/dw_ssi.c
include/hw/ssi/dw_ssi.h
hw/misc/k230_hi_sys.c
include/hw/misc/k230_hi_sys.h
hw/riscv/k230.c
include/hw/riscv/k230.h
```

QOM 类型：

```c
#define TYPE_DW_SSI "dw-ssi"
OBJECT_DECLARE_SIMPLE_TYPE(DwSsiState, DW_SSI)
```

Kconfig：

```text
config DW_SSI
    bool
    select SSI
```

K230 machine 选择 `DW_SSI`，不再由 `CONFIG_K230_DW_SSI` 编译通用控制器。

### 6.2 第一批最小属性集合

第一批只实现有当前消费者的配置：

| 属性 | 类型 | 含义 |
|---|---|---|
| `num-cs` | uint32 | 实例片选数量 |
| `fifo-depth` | uint32 | TX/RX FIFO 深度 |
| `imr-reset` | uint32 | 七路基础 IRQ 的实例 reset 差异 |

第一批不增加 `max-lines`、enhanced/IDMA/XIP property、DMA layout、AXI reset 或 XIP window 配置，也不预留内部 future bit/helper。未实现 offset 使用 RAZ/WI 或明确 unsupported 语义。后续 property 必须与正路径和测试在同一 series 引入。

### 6.3 K230 已知实例配置

当前 machine 代码确认的物理实例配置是：

| 物理实例 | 当前数组下标 | CS 数 | 最大线宽 | 控制器 MMIO |
|---|---:|---:|---:|---|
| QSPI0 | 0 | 5 | 4 | `K230_DEV_QSPI0` |
| QSPI1 | 1 | 5 | 4 | `K230_DEV_QSPI1` |
| SPI-OPI/FMC | 2 | 1 | 8 | `K230_DEV_SPI` |

共同的已知值包括：

```text
fifo-depth       = 256
component-id     = 0xa1b2c3d5
version-id       = 0x3130332a
SPI profile      = 0x04000200
FMC/OPI profile  = 0x28000200
```

`has-idma`、`has-xip` 和增强能力是否对三个实例全部启用，应在编码时按 TRM 实例配置表再次核对，不能因为当前模型对所有实例硬编码寄存器就默认全部为 `true`。

### 6.4 编号映射注意事项

SDK 逻辑编号和当前 C 数组下标不是同一个概念：SDK 的逻辑 `spi0` 对应物理 SPI-OPI/FMC 实例，而物理数组下标 0 是 QSPI0。

v2 的代码、测试和 commit message 应统一使用清晰术语：

- `logical_index`：SDK/HI_SYS 的 spi0、spi1、spi2；
- `ssi_index`：QEMU `dw_ssi[]` 数组下标；
- `QSPI0/QSPI1/SPI-OPI`：物理模块名称。

不能把拆分分析中简化的“`num-cs=1`”应用到全部实例。

## 7. HI_SYS 解耦方案

当前模型存在反向依赖：DWC SSI include K230 HI_SYS，并保存 `K230HiSysState *`，XIP 读路径直接查询 `k230_hi_sys_xip_enabled()`。

v2 应删除：

```c
#include "hw/misc/k230_hi_sys.h"
K230HiSysState *hi_sys;
k230_dw_ssi_set_hi_sys();
```

推荐依赖方向：

```text
错误：DWC SSI -> K230 HI_SYS
正确：K230 HI_SYS -> DWC SSI 的抽象接口
```

具体接口建议：

1. DWC SSI 提供名为 `xip-enable` 的 GPIO input；
2. K230 HI_SYS 提供对应 GPIO output；
3. HI_SYS reset 或写 `SSI_CTRL` 后更新输出电平；
4. XIP 读路径只检查 DWC SSI 自身的 `xip_enabled` 状态；
5. HI_SYS 读取 mode/sleep 状态时，通过 `DwSsiState` link 和通用 getter 查询。

通用 getter 可以保留为：

```c
uint32_t dw_ssi_get_spi_mode(const DwSsiState *s);
bool dw_ssi_is_sleeping(const DwSsiState *s);
```

依赖只允许 K230 HI_SYS 引用通用 DWC SSI，通用模型不能引用任何 K230 类型。

## 8. XIP aperture 最终决策

历史分析提出两个方案：

| 方案 | 描述 | 问题 |
|---|---|---|
| A | 通用层只暴露 `xip_read()`，K230 wrapper 创建 MMIO region | 为单纯映射和 enable 联动引入 wrapper，职责偏薄且重复 |
| B | 通用层提供完整 XIP region，K230 只决定映射地址 | 需要让窗口大小可配置，避免写死 K230 信息 |

最终采用 **方案 B 的修正版**：

- XIP 命令生成、SPI transaction 和 MMIO read 语义放在通用 DWC SSI；
- 通用模型在 `has-xip=true` 时提供第二个 sysbus MMIO region；
- region 大小由 `xip-window-size` 配置；
- 通用模型不知道 `0xc0000000`；
- K230 machine 将 SPI-OPI 实例的第二个 region 映射到 `K230_DEV_FLASH`；
- XIP 是否可访问由 `xip-enable` GPIO 控制。

这样保持了清晰边界：

```text
根据 XIP 寄存器生成 SPI 事务 = DWC SSI 行为
窗口位于何处、大小是多少       = SoC 集成配置
```

仅映射第二个 sysbus MMIO region 不构成引入 K230 wrapper 的充分理由。

## 9. 当前代码的主要重构点

### 9.1 全局命名通用化

需要系统性调整：

- `K230DwSsiState` -> `DwSsiState`；
- `TYPE_K230_DW_SSI` -> `TYPE_DW_SSI`；
- `K230_DW_SSI_*` -> `DW_SSI_*`；
- `k230_dw_ssi_*()` -> `dw_ssi_*()`；
- trace event 去掉 `k230_` 前缀；
- migration VMState 名称使用通用类型名。

### 9.2 硬编码参数配置化

第一批只配置当前存在实例差异且有消费者的参数：

- FIFO 256；
- CS 数；
- `IMR` reset。

IDR/version 采用当前确认的通用常量。`max-lines`、SPI/FMC `SPI_CTRLR0` profile、DMA layout 和 XIP window 没有第一批消费者，随对应后续 series 引入。

### 9.3 可选寄存器门控

第一批不建立 capability 门控框架。Standard PIO/IRQ 寄存器实现真实语义；enhanced、DMA/IDMA、XIP offset 统一 RAZ/WI 或走集中 unsupported helper。后续功能 series 引入真实正路径时，再同时引入对应配置和门控，不预留无消费者 helper。

### 9.4 IRQ 输出

第一批只注册 TXE、TXO、RXF、RXO、TXU、RXU、MST 七路基础 IRQ。DONE/AXIE 和 XIP IRQ 随对应功能 series 增加，不能为保持未来拓扑而提前注册恒低 output。K230 machine 第一批只连接七路基础 IRQ。

## 10. v2 Patch 系列组织

第一批固定为 5 个可独立使用和测试的 patch：

1. `hw/ssi: Add a Synopsys DesignWare SSI standard PIO controller`
2. `hw/ssi: Add DesignWare SSI standard interrupt support`
3. `hw/riscv/k230: Instantiate DesignWare SSI controllers`
4. `hw/riscv/k230: Route SSI interrupts to the PLIC`
5. `hw/riscv/k230: Attach a standard SPI flash to the K230 SSI`

Patch 1 必须已经包含 Standard 寄存器、FIFO、四种 TMOD、PIO、reset、基础 VMState 和通用 qtest，不能先实例化空壳再由后续 patch 补基本数据路径。Patch 2 只增加七路基础 IRQ。测试跟随首次引入相应行为的 patch。

enhanced SPI、DMA/IDMA、HI_SYS/XIP 和 trace 按独立后续 series 发送，每批随首个真实消费者引入其 property、IRQ、GPIO、MMIO、状态和测试。
- Patch 5/7/9 中允许出现 K230 地址、PLIC、HI_SYS 和设备接线；
- Patch 10 同时完成通用 XIP region、GPIO enable 和 K230 地址映射；
- Patch 11 的通用 trace 使用 `dw_ssi_*` 命名，K230-only trace 留在对应 K230 文件。

## 11. 测试重构

测试应区分“通用 IP 契约”和“K230 集成契约”。

### 11.1 通用 DWC SSI 测试

候选覆盖：

- 基础寄存器 reset/write mask；
- DR aliases；
- FIFO 深度和 TX/RX level；
- 四种 TMOD；
- 标准 IRQ；
- capability 关闭时寄存器 RAZ/WI；
- enhanced SPI；
- IDMA success/error；
- XIP 命令生成和读取。

如果暂时没有方便的独立 test machine，可以先复用 K230 machine 访问实例，但测试函数和断言要明确标记为 DWC SSI 行为。后续再评估独立 `dw-ssi-test.c` 或最小测试设备。

### 11.2 K230 集成测试

保留并聚焦：

- 三实例物理地址；
- SDK 逻辑编号路由；
- PLIC source 路由；
- HI_SYS reset/write mask；
- mode/sleep 状态；
- HI_SYS `XIP_EN` 到 SSI 的连接；
- SPI NOR 挂载；
- `0xc0000000`、128 MiB XIP aperture。

现有十个场景可以继续使用，但建议按归属重命名或拆组，避免把 K230 集成行为描述成通用 DWC SSI 规范。

## 12. Databook 与证据闸门

DWC SSI Databook 和 K230 CoreConsultant 配置报告仍然是高价值资料，但不建议阻塞 v2。

### 12.1 当前可接受的实现标准

- 通用模型只覆盖已有多源证据支持的功能；
- 对未确认的 DWC 变体明确写出限制；
- 不以“完整 DWC SSI 模型”作为提交宣称；
- K230-specific 行为有 TRM、SDK 或实机证据；
- 未确认字段保持 RAZ/WI 或不实现，而不是猜测；
- capability 取值在 K230 machine 中有明确依据。

### 12.2 Databook 的定位

- 能获得时用于验证版本差异、可选寄存器和信号语义；
- 受许可限制时不提交原始文档，只记录必要结论；
- reviewer 对具体寄存器语义提出疑问时，再针对性补证据；
- 不因为暂时缺少 Databook 而保留明显错误的 K230 反向依赖。

## 13. 实施顺序与检查点

实施路线已细化到独立文档：[K230 SPI/QSPI V2 实施路线](k230-spi-qspi-v2-implementation-plan.md)。

核心原则：分刀推进，每一刀只做一类改动，保持中间态可编译可回归。不在 v3.4 顶部追加单一重构 patch；第一批重组为 5 个自洽提交，未来功能使用独立 follow-up series。

五步概要：

1. 冻结 v3.4 基线（重新构建 + qtest + 记录）；
2. 行为不变的通用化（重命名 + 依赖整理）；
3. 解除 HI_SYS 反向依赖（GPIO + getter）；
4. 引入实例配置（属性化）；
5. 重组第一批 5 个提交。

每步的检查点和注意点见实施路线文档。

## 14. Cover letter 与 review 回复要点

v2 cover letter 应直接说明：

- 已按 review 将控制器重构为通用 `DW_SSI`；
- K230 只负责配置和 SoC 集成；
- 第一批只提交 Standard PIO 和七路基础 IRQ；enhanced SPI、DMA/IDMA 和 XIP 接口随独立 follow-up series 引入；
- 删除了通用模型对 K230 HI_SYS 的依赖；
- XIP 地址由 K230 machine 映射，XIP 事务由通用模型实现；
- 当前模型范围是 K230 所需且有证据支持的 DWC SSI 子集。

回复 Bin Meng 时可以简洁概括为：

> The controller model has been split into a reusable DesignWare SSI
> device and K230 machine integration. K230-specific addresses, PLIC
> routing, HI_SYS control and flash wiring remain in the K230 machine,
> while the first series implements only the reusable Standard PIO and
> interrupt baseline. Enhanced SPI, DMA/IDMA and XIP will follow with
> their first functional consumers.

## 15. 参考资料

- [K230 TRM 文本](../reference/K230_Technical_Reference_Manual_V0.3.1_20241118.txt)
- [TRM 12.3 中文对照](../spi/reference/k230-trm-12.3-spi-cn.md)
- [寄存器审阅表](../spi/k230-spi-qspi-register-audit.md)
- [K230 Linux DTS](../../../k230_sdk/src/little/linux/arch/riscv/boot/dts/kendryte/k230.dtsi)
- [K230 U-Boot DesignWare SPI 驱动](../../../k230_sdk/src/little/uboot/drivers/spi/designware_spi.c)
- [K230 Linux DesignWare SPI 核心](../../../k230_sdk/src/little/linux/drivers/spi/spi-dw-core.c)
- [K230 Linux-specific SPI 扩展](../../../k230_sdk/src/little/linux/drivers/spi/spi-dw-core-k230.c)
- [RT-Smart SPI 驱动](../../../k230_sdk/src/big/rt-smart/kernel/bsp/maix3/board/interdrv/spi/drv_spi.c)
- [当前通用 DW SSI 模型](../../../qemu-camp-2026-k230/hw/ssi/dw_ssi.c)
- [当前通用 DW SSI 头文件](../../../qemu-camp-2026-k230/include/hw/ssi/dw_ssi.h)
- [当前 K230 machine 集成](../../../qemu-camp-2026-k230/hw/riscv/k230.c)
- [当前 HI_SYS 模型](../../../qemu-camp-2026-k230/hw/misc/k230_hi_sys.c)
- [当前 v3.4 cover letter](current/k230-spiv3.4-cover-letter.md)

## 16. 合并版变更记录

### 2026-07-28

- 合并原 v2 决策记录和 DW SSI 拆分分析；
- 明确默认不增加 K230 SSI wrapper；
- 采用“通用 XIP region + 可配置大小 + K230 machine 映射”的方案；
- 明确 HI_SYS 通过 GPIO/link 与通用模型解耦；
- 修正全部实例 `num-cs=1` 的过度简化，记录实际 5/5/1 配置；
- 增加 SDK 逻辑编号、QEMU 数组下标和物理模块名的区分；
- 增加当前代码重构点、patch 顺序、测试边界和 review 回复建议；
- 将 Databook 定位为增强证据，而不是架构拆分的前置阻塞项。

---

<details>
<summary>历史原始记录：2026-07-27 v2 调查与决策记录</summary>

> 以下内容按原样保留用于追溯，其中的“待决策”或阶段性结论可能已被上文取代。

## 1. Review 背景

上游 review 要求将当前 K230 SSI 模型拆分为：

1. 可供其他机器复用的 Synopsys DesignWare SSI 通用模型；
2. 可选的 K230-specific wrapper，只有在通用模型不足以满足 K230 软件时才保留。

当前模型文件名为 `k230_dw_ssi.c`，但其中同时包含寄存器模型、FIFO/PIO、IRQ、增强 QSPI、IDMA、XIP 和 K230 HI_SYS 耦合。下一版的核心任务不是简单改名，而是用证据区分：

- DWC SSI IP 的共性；
- DWC SSI 的综合配置参数；
- K230 SoC 的板级集成；
- 仅在 K230 上观察到的特殊行为。

## 2. 当前结论

K230 TRM 和 SDK 是必要证据，但不足以单独完成通用模型边界划分。还需要 DWC SSI 相关资料，至少需要确认：

- K230 使用的是哪个 DWC SSI IP 家族和版本；
- FIFO、CS、最大数据线数、IDMA、XIP 等综合参数；
- 哪些寄存器和信号是可选功能；
- 哪些行为属于 IP 规范，哪些是 K230 软件 workaround。

目前证据显示，K230 实例更接近 `DWC_ssi` 系列，而不能笼统称为旧式 `DW_apb_ssi`：

- K230 DTS 使用 `snps,dwc-ssi-1.01a` 兼容串；
- `SSIC_VERSION_ID` 复位值为 `0x3130332a`，对应版本字符串 `1.03*`；
- TRM 的 FMC 章节直接使用 `DWC_ssi` 名称，并提到 CoreConsultant 生成的 ID 和可变综合参数。

## 3. 证据层级

| 证据 | 可以证明 | 不能单独证明 |
|---|---|---|
| DWC SSI Databook | IP 架构、寄存器通用语义、可选功能、信号语义 | K230 实例究竟启用了哪些综合参数 |
| K230 TRM | K230 对外暴露的寄存器、复位值、系统连接和可观察行为 | 所有其他 DWC SSI 实例都具备相同行为 |
| CoreConsultant 配置或生成报告 | K230 对通用 IP 的具体参数选择 | SoC 外围地址、PLIC、HI_SYS 和 Flash 接线 |
| K230 SDK 驱动 | K230 软件实际访问的寄存器、顺序和功能路径 | 行为是硬件规范、驱动 workaround 还是驱动缺陷 |
| Linux/U-Boot 通用驱动 | 多平台共性、版本和 capability 划分 | K230 的最终硬件配置 |
| 其他 SoC 的 TRM/驱动 | 同一 DWC SSI 功能的交叉验证 | K230-specific 集成语义 |
| 实机或启动日志 | K230 的真实可观察行为 | 行为是否适用于其他机器 |

注意：SDK 不是纯粹的“K230 特殊实现”。U-Boot 的 `designware_spi.c` 同时支持多个 `snps,dw-apb-ssi-*` 和 `snps,dwc-ssi-*` 版本；Linux SDK 同时包含通用 `spi-dw-core.c` 和 K230 派生代码；RT-Smart 驱动则更接近 K230 软件使用路径。

## 4. 资料需求

### 4.1 优先级最高

- Synopsys `DWC_ssi` Databook，优先寻找 1.03 或接近版本。
- K230 对应的 CoreConsultant 配置报告或等价综合参数说明。

如果资料受 Synopsys 许可限制，不应把原始文档提交到公开仓库；只在本地分析中记录章节、结论和必要引用。

### 4.2 可接受的替代证据

如果暂时拿不到原始 Databook，则使用以下组合进行交叉验证：

1. K230 TRM FMC/SPI 章节；
2. K230 Linux/U-Boot/RT-Smart 驱动；
3. 上游 Linux `spi-dw` 核心和设备树 binding；
4. 其他公开使用 DWC SSI 的 SoC 文档和驱动；
5. K230 实机启动、Flash 读写和 XIP 验证结果。

这种替代方案可以支撑“当前 K230 QEMU 所需的最小通用模型”，但不能宣称已经完整覆盖所有 Synopsys DWC SSI 变体。

## 5. 分类标准

### 5.1 DWC SSI 通用行为

满足下列条件中的至少两项，并且没有相反证据：

- DWC 文档明确规定；
- Linux/U-Boot 通用驱动在多个平台使用；
- 其他 SoC 的实现与 K230 行为一致；
- 不依赖 K230 地址、HI_SYS 或 SoC-specific 信号。

候选内容：寄存器访问框架、DR/FIFO、标准 PIO、TMOD、基本状态位、通用 IRQ 状态和标准增强 SPI 阶段。

### 5.2 DWC SSI 可配置能力

DWC 文档将其标记为可选、`Varies` 或由综合参数决定，而 K230 TRM/SDK 给出了具体值。

候选内容：

- FIFO 深度；
- CS 数量；
- 最大数据线数；
- DFS 最大宽度；
- ID/version；
- enhanced SPI、IDMA、XIP、DDR/RXDS 等能力是否存在；
- XIP 读写扩展是否综合进实例。

这类内容应进入通用模型的 device property、class configuration 或 capability，而不是直接写成 `K230_*` 常量。

### 5.3 K230 SoC 集成

明确属于控制器外部或板级连接：

- 三个 SSI 实例及其 SDK 编号；
- `0x91582000` 等物理地址；
- SSI IRQ 到 K230 PLIC 的路由；
- HI_SYS `SSI_CTRL`；
- SPI NOR 挂接；
- `0xc0000000` 的 XIP aperture 地址和大小；
- 时钟、复位、IOMUX 和其他 SoC 外围控制。

### 5.4 K230-specific quirk

只有在以下条件同时满足时，才将行为标记为 K230 特性：

- 通用 DWC 资料没有覆盖或给出不同语义；
- K230 TRM 明确描述，或 SDK/实机可以稳定复现；
- 不能通过正常综合参数解释。

### 5.5 证据不足

仅凭一个 SDK 驱动的写法、一个寄存器复位值或一个 qtest 现象，不足以将行为命名为 K230-specific。此类内容先标记为“待确认”，不能提前固化进 wrapper 接口。

## 6. 当前系列的初步归类

| 当前功能 | 初步归属 | v2 决策方向 | 需要补证据 |
|---|---|---|---|
| `0x00--0x5c` 基础寄存器 | DWC SSI 共性，部分字段受配置影响 | 放入通用模型 | DWC 版本和寄存器变体 |
| DR aliases、FIFO、PIO、四种 TMOD | DWC SSI 共性 | 放入通用模型 | FIFO 深度和帧宽上限 |
| RISR/ISR/IMR、IRQ 输出 | DWC SSI 共性，IRQ 数量可能是配置项 | 放入通用模型，K230 负责 PLIC 接线 | 输出数量和事件使能参数 |
| `SPI_CTRLR0`、Dual/Quad SDR | DWC SSI enhanced capability | 通用模型 capability | `DWC_ssi` 与 `DW_apb_ssi` 的版本差异 |
| `SPIDR`、`SPIAR`、`AXIAR*`、DONE/AXIE | DWC SSI 可选 IDMA capability | 通用模型 capability | IDMA 规范和 K230 实例是否完整启用 |
| XIP 指令/地址/mode/dummy 机制 | DWC SSI 可选 XIP capability | 通用模型；XIP enable 通过抽象接口输入 | XIP signal 和 aperture 语义 |
| XIP 128 MiB 地址窗口 | K230 SoC 集成 | 留在 K230 machine mapping | TRM memory map |
| HI_SYS `SSI_CTRL` | K230-specific | 留在 `k230_hi_sys` | TRM/SDK |
| 三实例、物理地址、PLIC route | K230-specific | 留在 `hw/riscv/k230.c` | K230 memory map/interrupt map |
| W25Q256 挂接 | K230 machine integration | 留在 machine/Flash wiring | 启动链路证据 |
| trace events | 观察通用模型行为 | 通用 trace 命名，避免 K230 前缀 | 最终通用类型名 |

以上只是 v2 的初步决策，不替代逐寄存器审阅。

## 7. v2 重构原则

### 7.1 通用模型边界

建议形成 `DwSsiState` / `TYPE_DW_SSI`，通用模型不应：

- include K230 HI_SYS 头文件；
- 保存 `K230HiSysState *`；
- 读取 K230 物理地址或 PLIC 编号；
- 用 `K230_*` 常量表达综合参数；
- 为 K230 的 machine mapping 直接创建 SoC-specific aperture。

### 7.2 K230 集成边界

K230 machine 负责：

- 初始化和配置三个通用 SSI 实例；
- 设置 CS 数量、最大线宽和其他 K230 综合参数；
- 连接 PLIC；
- 连接 HI_SYS；
- 映射控制器和 XIP aperture；
- 挂接 SPI NOR。

### 7.3 是否保留 K230 wrapper

当前默认方案是不增加独立 K230 SSI wrapper：

- HI_SYS 已经是 K230-specific 外围设备；
- 大部分当前“特性”看起来是 DWC SSI capability 或综合参数；
- 通过 property、capability 或抽象 GPIO/link 可以去除通用模型对 HI_SYS 的依赖。

只有当逐项审阅后发现无法通过配置或通用接口表达的 K230 控制器语义，才增加薄 wrapper，并限制其职责范围。

## 8. v2 Patch 组织方向

这不是只修改第 1 个 patch 的文件名，后续 patch 也要保持命名和依赖一致：

1. 通用 DWC SSI 寄存器模型和基础配置。
2. K230 machine 实例化通用模型。
3. 通用 FIFO/PIO/TMOD。
4. 通用 SSI IRQ。
5. K230 PLIC 路由。
6. 通用 enhanced SPI/QSPI。
7. K230 SPI NOR 挂接。
8. 通用可选 IDMA。
9. K230 HI_SYS `SSI_CTRL`。
10. 通用 XIP capability；K230 负责 aperture 和使能连接。
11. 通用 trace events。

具体拆分仍需根据 DWC 资料确认，不能在资料不足时过度抽象。

## 9. 待办与决策闸门

### 9.1 编码前必须完成

- [ ] 确认 `DWC_ssi 1.03*` 与 `snps,dwc-ssi-1.01a` 的关系。
- [ ] 对照 TRM、SDK 和通用驱动整理逐寄存器证据矩阵。
- [ ] 确认 FIFO 深度、CS 数、最大线宽和 capability 参数。
- [ ] 区分 IDMA/XIP 的通用协议与 K230 外围连接。
- [ ] 设计通用模型向 HI_SYS 暴露的最小抽象接口。
- [ ] 决定 XIP aperture 是通用设备 region 属性还是 K230 machine alias。

### 9.2 可以接受的 v2 证据标准

- 通用模型只声称支持已被 DWC 资料或多源实现支撑的能力；
- 对未确认的 DWC 变体明确写出限制；
- K230-specific 行为有 TRM、SDK 或实机证据之一，并说明证据类型；
- qtest 覆盖的是模型契约，不把 K230 集成行为伪装成通用 IP 规范。

## 10. 参考入口

- [K230 TRM 文本](../reference/K230_Technical_Reference_Manual_V0.3.1_20241118.txt)
- [K230 Linux DTS](../../../k230_sdk/src/little/linux/arch/riscv/boot/dts/kendryte/k230.dtsi)
- [K230 U-Boot DesignWare SPI 驱动](../../../k230_sdk/src/little/uboot/drivers/spi/designware_spi.c)
- [K230 Linux DesignWare SPI 核心](../../../k230_sdk/src/little/linux/drivers/spi/spi-dw-core.c)
- [K230 Linux-specific SPI 扩展](../../../k230_sdk/src/little/linux/drivers/spi/spi-dw-core-k230.c)
- [RT-Smart SPI 驱动](../../../k230_sdk/src/big/rt-smart/kernel/bsp/maix3/board/interdrv/spi/drv_spi.c)
- [当前通用 DW SSI 模型](../../../qemu-camp-2026-k230/hw/ssi/dw_ssi.c)
- [当前通用 DW SSI 头文件](../../../qemu-camp-2026-k230/include/hw/ssi/dw_ssi.h)
- [当前 K230 machine 集成](../../../qemu-camp-2026-k230/hw/riscv/k230.c)
- [当前 HI_SYS 模型](../../../qemu-camp-2026-k230/hw/misc/k230_hi_sys.c)

## 11. 变更记录

### 2026-07-27

- 记录 Bin Meng 对 patch 1 的通用 DesignWare SSI 拆分意见。
- 确认 K230 TRM、SDK、Linux/U-Boot 通用驱动的证据边界不同。
- 建立 DWC 共性、综合参数、K230 集成和 K230 quirk 的分类标准。
- 形成 v2 重构的初步边界和待决策问题。

</details>
