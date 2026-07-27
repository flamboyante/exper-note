# K230 SPI/QSPI 上游 Review 与 v2 修改决策记录

记录时间：2026-07-27
适用分支：`qemu-camp-2026-k230/k230-spiv3.4`
问题来源：Bin Meng 对第 1 个 SSI patch 的 review
文档状态：调查与决策记录，不代表 v2 代码已经实现

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

- [K230 TRM 文本](K230_Technical_Reference_Manual_V0.3.1_20241118.txt)
- [K230 Linux DTS](../../k230_sdk/src/little/linux/arch/riscv/boot/dts/kendryte/k230.dtsi)
- [K230 U-Boot DesignWare SPI 驱动](../../k230_sdk/src/little/uboot/drivers/spi/designware_spi.c)
- [K230 Linux DesignWare SPI 核心](../../k230_sdk/src/little/linux/drivers/spi/spi-dw-core.c)
- [K230 Linux-specific SPI 扩展](../../k230_sdk/src/little/linux/drivers/spi/spi-dw-core-k230.c)
- [RT-Smart SPI 驱动](../../k230_sdk/src/big/rt-smart/kernel/bsp/maix3/board/interdrv/spi/drv_spi.c)
- [当前 K230 SSI 模型](../../qemu-camp-2026-k230/hw/ssi/k230_dw_ssi.c)
- [当前 K230 SSI 头文件](../../qemu-camp-2026-k230/include/hw/ssi/k230_dw_ssi.h)
- [当前 K230 machine 集成](../../qemu-camp-2026-k230/hw/riscv/k230.c)
- [当前 HI_SYS 模型](../../qemu-camp-2026-k230/hw/misc/k230_hi_sys.c)

## 11. 变更记录

### 2026-07-27

- 记录 Bin Meng 对 patch 1 的通用 DesignWare SSI 拆分意见。
- 确认 K230 TRM、SDK、Linux/U-Boot 通用驱动的证据边界不同。
- 建立 DWC 共性、综合参数、K230 集成和 K230 quirk 的分类标准。
- 形成 v2 重构的初步边界和待决策问题。
