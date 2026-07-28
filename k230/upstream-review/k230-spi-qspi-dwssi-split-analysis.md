# K230 DW SSI 上游拆分分析（已合并）

> 本文内容已合并到 [K230 SPI/QSPI 上游 Review 与 v2 重构决策（合并版）](k230-spi-qspi-review-v2-decision-notes.md)。
> 后续实施、讨论和 review 回复请以合并版为唯一依据。本文件仅保留历史分析，避免旧链接失效。

<details>
<summary>展开 2026-07-27 原始拆分分析</summary>

# K230 DW SSI 上游拆分分析（历史原文）

更新时间：2026-07-27

记录上游 review（Bin Meng）要求的拆分边界研究结论，作为 `k230-spiv3.3` 拆分实施依据。

## 一、背景

### 1.1 上游 review 要求

Bin Meng 在 `k230-spiv3.2` 上对 patch 1 "hw/ssi: Add K230 DesignWare SSI register model" 的回复要求把模型拆成两层：

- 一个通用的 Synopsys DesignWare SSI 控制器模型
- 一个 K230 专有的 wrapper 模型，放在通用层之上（可选——若通用层足够让 K230 软件工作则可不必要）

目的是让 Synopsys DW SSI 控制器模型可以被未来其他使用 DW SSI IP 的 SoC 复用。

### 1.2 研究问题

拆分边界怎么定？关键是要搞清楚：

1. K230 TRM 描述的 SPI 控制器是 K230 自研，还是 Synopsys IP 实例化？
2. 如果是 IP 实例化，TRM 是否已经包含全部 IP 特性，还是需要额外找 DW SSI IP databook？
3. SDK 里的 designware_spi.c 是 K230 专有驱动，还是通用 driver？

## 二、三条关键证据

### 2.1 证据 1：K230 TRM 自己承认是 Synopsys IP

TRM 17376–17377 行明确写：

> Contains the hex representation of the **Synopsys** component version. Consists of ASCII value for each … `0x3130332a`

且 TRM 全文大量引用 IP databook 的可配置参数（`SSIC_HAS_*` / `SSIC_*`）：

- `SSIC_HAS_RX_SAMPLE_DELAY`、`SSIC_HAS_RXDS`、`SSIC_HAS_TX_RX_EN`
- `SSIC_SPI_MODE`、`SSIC_XIP_EN`、`SSIC_XIP_WRITE_EN`、`SSIC_XIP_WRITE_REG_EN`、`SSIC_XIP_CONT_XFER_EN`
- `SSIC_AXI_DW`、`SSIC_DFLT_BAUDR`、`SSIC_DFLT_AWLEN`、`SSIC_DFLT_ARLEN`
- `SSIC_RX_DLY_SR_DEPTH`、`SSIC_DM_EN`

这些都是 Synopsys DW SSI IP 的 Verilog 参数化配置项。也就是说：

> K230 TRM 12.3 = "Synopsys DW SSI IP 在 K230 选定配置下的实例化说明"，不是 K230 自研 SPI 控制器。

### 2.2 证据 2：U-Boot designware_spi.c 就是通用 driver，K230 SDK 只改一行

`build/k230-uboot-src/drivers/spi/designware_spi.c` 文件头：

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * Designware master SPI core controller driver
 *
 * Copyright (C) 2014 Stefan Roese <sr@denx.de>
 * Copyright (C) 2020 Sean Anderson <seanga2@gmail.com>
 *
 * Very loosely based on the Linux driver:
 * drivers/spi/spi-dw.c, which is:
 * Copyright (c) 2009, Intel Corporation.
 */
```

这是 Denx 维护的**上游通用 driver**，本来就要驱动所有 DW SSI 实例。K230 SDK 只在顶部硬改了一行：

```c
#define SSIC_HAS_DMA 2   //Internal DMA
```

上游 driver 用这个宏区分三种 IP 配置：

```c
#if SSIC_HAS_DMA == 1
#define DW_SPI_DMATDLR                  0x50    // 标准 external DMA 水位寄存器
#define DW_SPI_DMARDLR                  0x54
#elif SSIC_HAS_DMA == 2
#define DW_SPI_AXIAWLEN                 0x50    // 内部 AXI DMA 寄存器
#define DW_SPI_AXIARLEN                 0x54
#endif
```

也就是说：

> "internal DMA" 本身就是 DW SSI IP 的一个可选配置（`SSIC_HAS_DMA=2`），不是 K230 SoC 自己加的。K230 只是把 IP 配置成这种模式。

Linux 主线 `drivers/spi/spi-dw.c` 同样是通用 driver。

### 2.3 证据 3：QEMU 已有的 designware_i2c.c 就是这种组织方式

`iomux-v2/hw/i2c/designware_i2c.c` 是 QEMU 上游已有的 Synopsys DW I2C 通用模型，`TYPE_DESIGNWARE_I2C`，没有挂任何 SoC wrapper——因为通用层加属性就够了。

Bin Meng 让你做的就是同样的事：把 DW SSI 也做成这样的通用模型。

## 三、结论 1：不需要再找 DW SSI IP databook

仓库里现有资料已经覆盖全部 IP 特性：

| 资料 | 路径 | 角色 |
|---|---|---|
| K230 TRM | `K230_Technical_Reference_Manual_V0.3.1_20241118.txt` / `.pdf` | IP 实例化说明，含全部 `SSIC_*` 配置项 |
| TRM 12.3 中文对照 | `exper-note/k230/spi/reference/k230-trm-12.3-spi-cn.md` | 逐位依据 |
| U-Boot 通用 driver | `build/k230-uboot-src/drivers/spi/designware_spi.c` | 上游 Denx driver，证明 IP 通用性 |
| RT-Smart driver | `k230_sdk/.../drv_spi.c` | K230 SDK 对 IP 的实际调用序列 |
| QEMU DW I2C 模型 | `iomux-v2/hw/i2c/designware_i2c.c` | 通用层组织方式的参考 |

K230 TRM 描述的就是 Synopsys DW SSI IP，**不需要再找 Synopsys 原始 IP databook**。SDK 是 K230 配置下的具体调用实现，不构成 IP 边界证据。

## 四、通用层 vs K230 专有层的边界

基于上面证据修正后的边界（区别于初版分析中把"内部 DMA + XIP 寄存器"都归到 K230 专有的错误判断）：

| 项目 | 归属 | 依据 |
|---|---|---|
| 标准寄存器 CTRLR0/1, SSIENR, MWCR, SER, BAUDR, TX/RXFTLR, TX/RXFLR, SR, IMR/ISR/RISR, *ICR, IDR, VERSION, DR0..DR_END | **通用** | DW SSI databook 标准 |
| `DMACR.RDMAE/TDMAE`（external DMA） | **通用** | IP 配置 `SSIC_HAS_DMA=1` |
| `DMACR.IDMAE/ATW/AINC/ACACHE/APROT/AID` + `AXIAWLEN/AXIARLEN/SPIDR/SPIAR/AXIAR0/AXIAR1/AXIECR/DONECR` | **通用**（受 `internal-dma` 属性门控） | IP 配置 `SSIC_HAS_DMA=2`，上游 driver 用 `#if SSIC_HAS_DMA==2` 区分 |
| `RX_SAMPLE_DELAY`, `SPI_CTRLR0`, `DDR_DRIVE_EDGE`, `XIP_MODE_BITS`, `XIP_INCR_INST`, `XIP_WRAP_INST`, `XIP_CTRL`, `XIP_SER`, `XRXOICR`, `XIP_CNT_TIME_OUT`, `SPI_CTRLR1`, `SPITECR`, `XIP_WRITE_*` | **通用**（受 `xip-enabled` 属性门控） | IP 配置 `SSIC_XIP_EN`/`SSIC_XIP_WRITE_EN`，TRM 全部用 `SSIC_*` 前缀描述 |
| IRQ：TXE/RXF/TXO/RXO/RXU/MST | **通用** | 标准 DW SSI |
| IRQ：XRXO/AXIE/DONE/SPITE/TXUI | **通用**（受属性门控） | 与 `SSIC_XIP_EN` / `SSIC_HAS_DMA=2` 绑定 |
| Dual/Quad SDR 增强事务 | **通用** | IP 配置 `SSIC_SPI_MODE` |
| XIP 引擎（生成指令/地址/mode/dummy） | **通用** | IP 功能 |
| **XIP MMIO aperture @ 0xc0000000（128 MiB）** | **K230 专有** | K230 地址映射决定，IP 只产生 SPI 命令，不关心物理地址 |
| **复位值**：`IDR=0xa1b2c3d5`、`VERSION=0x3130332a`、`SPI_CTRLR0` FMC/SPI profile | **K230 专有** | K230 SoC 实例化决定 |
| **三实例地址映射**（spi0/spi1/spi2） | **K230 专有** | K230 地址映射 |
| **spi0 = FMC profile（8线），spi1/2 = SPI profile（4线）** | **K230 专有** | K230 板级配置 |
| **HI_SYS SSI_CTRL 联动**、PLIC IRQ 路由 | **K230 专有** | SoC 集成 |

## 五、结论 2：K230 wrapper 会非常薄

按上面的边界，K230 wrapper 几乎只做四件事：

1. 设 QOM 属性：`num-cs=…`、`max-lines=1|4|8`、`fifo-depth=256`、`internal-dma=true`、`xip-enabled=true`、`idr-reset=0xa1b2c3d5`、`version-id=0x3130332a`、`spi-ctrlr0-reset=0x28000200|0x04000200`
2. 注册 spi0 那块 128 MiB XIP MMIO aperture（命令由通用层 XIP 引擎生成，地址由 wrapper 决定）
3. 暴露 HI_SYS SSI_CTRL 联动接口
4. （可选）覆盖 reset 来填 K230 复位值——但其实靠 `*-reset` 属性就够了

这正好对应 Bin Meng 括号里说的 "optional, if the generic model is enough to make the k230 software happy"——很可能 XIP MMIO 这一项就使 wrapper 必须存在（因为它需要单独的 sysbus MMIO region），但除此之外几乎全是属性配置。

## 六、推荐目标结构

### 6.1 文件与 QOM 关系

```
hw/ssi/dw_ssi.c                  # 通用 DW SSI
hw/ssi/k230_dw_ssi.c             # K230 wrapper（极薄）
include/hw/ssi/dw_ssi.h          # DwSsiState, TYPE_DW_SSI, 属性声明
include/hw/ssi/k230_dw_ssi.h

hw/ssi/Kconfig:
    config DW_SSI
        bool
        depends on SSI
    config K230_DW_SSI
        bool
        select DW_SSI
```

QOM 继承链：`TYPE_K230_DW_SSI` → `TYPE_DW_SSI` → `SYS_BUS_DEVICE`。

`DwSsiState` 持有通用寄存器数组、FIFO、CS、SSI bus；`K230DwSsiState` extends 它，加 XIP MMIO region、HI_SYS 联动接口，覆盖 reset 填 K230 复位值。

### 6.2 通用层 QOM 属性建议

| 属性 | 类型 | 含义 | K230 取值 |
|---|---|---|---|
| `num-cs` | uint32 | 片选数 | 1 |
| `max-lines` | uint32 | 最大数据线数 1/2/4/8 | spi0=8，spi1/2=4 |
| `fifo-depth` | uint32 | FIFO 容量 | 256 |
| `internal-dma` | bool | 是否实现 `SSIC_HAS_DMA=2` 内部 AXI DMA | true |
| `xip-enabled` | bool | 是否实现 `SSIC_XIP_EN` XIP 寄存器组 | true |
| `idr-reset` | uint32 | `IDR` 复位值 | 0xa1b2c3d5 |
| `version-id` | uint32 | `VERSION` 复位值 | 0x3130332a |
| `spi-ctrlr0-reset` | uint32 | `SPI_CTRLR0` 复位值 | 0x28000200 / 0x04000200 |

### 6.3 对现有 10 个 patch 的影响

| Patch | 影响 |
|---|---|
| 1 寄存器模型 | **大改**：拆成 `dw_ssi.c` + `k230_dw_ssi.c`（wrapper 极薄） |
| 2 machine 实例化 | 改 `TYPE_K230_DW_SSI` 即可，外部看不变 |
| 3 FIFO/PIO | 逻辑进通用层（这是标准 DW SSI TMOD 行为） |
| 4 内部 DMA | **整 patch 进通用层**（受 `internal-dma` 属性门控） |
| 5 IRQ 控制器 | 通用层（DW SSI 标准中断）+ K230 的 `XRXOIR`/`AXIER`/`DONER`/`SPITER`/`TXUIR` 也在通用层（受属性门控） |
| 6 PLIC 接线 | 不变 |
| 7 QSPI 增强事务 | 通用层（DW SSI 增强 SPI） |
| 8 SPI NOR 挂载 | 不变 |
| 9 HI_SYS SSI_CTRL | 已经是独立的 `k230_hi_sys` misc 设备，不动 |
| 10 XIP 读窗口 | **XIP 引擎逻辑进通用层**，**128 MiB MMIO aperture 进 K230 wrapper** |

### 6.4 qtest 拆分建议

- `tests/qtest/dw-ssi-test.c`：测通用层最小配置（`num-cs=1, max-lines=1, fifo-depth=256`，不开 DMA/XIP），证明通用性
- `tests/qtest/k230-dw-ssi-test.c`：保留现有 10 个场景，测 K230 整机行为

Bin Meng 没强制要求加通用层 qtest，但加上会更稳，验收更顺。

## 七、待决策点：XIP MMIO aperture 放哪一层

动手前需要拍板：XIP 读窗口的 128 MiB MMIO region 放在通用层还是 K230 wrapper？两个方案：

| 方案 | 通用层责任 | K230 wrapper 责任 | 优缺点 |
|---|---|---|---|
| A. 通用层不提供 XIP MMIO，wrapper 自己实现 | 只暴露 XIP 寄存器配置 + 一个 `xip_read(addr, len)` API | 自己映射 128 MiB region，调用通用层 API 生成命令 | 通用层更干净；wrapper 要写一点 XIP 命令组装逻辑 |
| B. 通用层提供 XIP MMIO region，wrapper 只决定地址 | 完整 XIP 引擎 + MMIO region | `sysbus_init_mmio` 时把通用层的 region 映射到 0xc0000000 | 复用性最高；但通用层要知道窗口大小，略偏 SoC |

倾向 **A**：XIP MMIO aperture 是 SoC 地址映射决定的（不同 SoC 可能不同大小、不同地址、甚至不存在），通用层只提供"给定 XIP 配置，生成一次读命令"的 API，wrapper 负责把内存访问翻译成 API 调用。这与 `k230-spiv3.2` patch 10 现有的 `k230_dw_ssi_xip_read` 实现吻合。

## 八、参考资料

- 上游 review 邮件：Bin Meng，2026-07-27 12:45
- 现有 series：`k230-spiv3.2` 分支，base `f893c46c3931b3684d235d221bf8b7844ddbf1d7`
- patch 1 实现：`iomux-v2` 仓库 commit `b4144a08f2` "hw/ssi: Add K230 DesignWare SSI register model"
- 寄存器审阅表：[k230-spi-qspi-register-audit.md](../spi/k230-spi-qspi-register-audit.md)
- TRM 12.3 中文对照：[k230-trm-12.3-spi-cn.md](../spi/reference/k230-trm-12.3-spi-cn.md)
- Cover letter：[k230-spiv3.2-cover-letter.md](archive/v3.2/k230-spiv3.2-cover-letter.md)

</details>
