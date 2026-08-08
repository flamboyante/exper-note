# K230 QEMU 新增（未合入）外设模块研究

> 数据截至：2026-08-04  
> 检索范围：lore.kernel.org/qemu-devel（关键词 k230）  
> 状态：以下 11 个系列均处于评审中（patchwork 状态 New），尚未合入 master

## 总览

| # | 模块 | 作者 | 最新版本 | 补丁数 | 更新日期 | 代码路径 |
|---|------|------|----------|--------|----------|----------|
| 1 | GPIO 控制器 | guochun.wang | v1 | 2 | 2026-08-03 | `hw/gpio/` |
| 2 | DesignWare SSI (SPI PIO) | Kangjie Huang | v2 | 5 | 2026-08-01 | `hw/ssi/` |
| 3 | DW APB Timer | raoyi | v2 | 3 | 2026-07-30 | `hw/timer/` |
| 4 | SDHCI | Xin Xie | v1 | 3 | 2026-07-29 | `hw/sd/` |
| 5 | GSDMA + Decomp Gzip | Tao Ding | v2 | 7 | 2026-07-27 | `hw/dma/` `hw/misc/` |
| 6 | SPI/QSPI/IDMA/XIP | Kangjie Huang | v1 | 11 | 2026-07-26 | `hw/ssi/` `hw/misc/` |
| 7 | DW 8250 UART | WX Chen | RESEND v2 | 3 | 2026-07-25 | `hw/char/` |
| 8 | SRAM | Jian Cai | v2 | 1 | 2026-07-21 | `hw/riscv/k230.c` |
| 9 | Reset Management Unit (RMU) | Jack Wang | v2 | 2 | 2026-07-19 | `hw/misc/` |
| 10 | DDR Controller + PHY | Junze Cao | v1 | 3 | 2026-07-16 | `hw/misc/` |
| 11 | IOMUX | Kangjie Huang | v2 | 3 | 2026-07-14 | `hw/misc/` |

> **注意**：模块 2（DesignWare SSI v2）是从模块 6（SPI/QSPI/IDMA/XIP v1）中拆分出来的通用模型部分，已替代原 v1 中的 SSI 控制器实现。原 v1 中尚未拆分的 Enhanced QSPI、IDMA、XIP 功能仍在等待后续系列。

---

## 1. GPIO 控制器

**作者**：guochun.wang  
**版本**：v1 0/2（2026-08-03）

### 功能概述

K230 SoC APB GPIO 控制器模型，基于 Synopsys DesignWare APB GPIO IP（Linux `gpio-k230` 驱动基于 `gpio-dwapb.c`，寄存器布局相同），使用 Canaan 专用 compatible（`canaan,k230-apb-gpio`）。能运行 Linux `gpio-k230` 驱动并驱动外部输入/输出引脚。

实现内容：
- Port A 寄存器：SWPORTA_DR/DDR/CTL、EXT_PORTA、INTEN、INTMASK、INTTYPE_LEVEL、INT_POLARITY、INTSTATUS、RAW_INTSTATUS、DEBOUNCE、LS_SYNC、INT_BOTHEDGE、PORTA_EOI、ID_CODE、VER_ID_CODE、CONFIG_REG1/2
- 软件控制模式：DR 驱动输出引脚，DDR 选择方向，EXT_PORTA 按 DDR 多路复用
- 每引脚 IRQ 输出线（每组 32 个），边沿检测（上升/下降/双沿）、去抖、EOI 清中断
- SoC 集成 2 组 GPIO（各 32 引脚），GPIO0→PLIC 32-63，GPIO1→PLIC 64-95

### 补丁列表

1. `hw/gpio: add K230 GPIO controller model`
2. `tests/qtest: add K230 GPIO controller test`

### 代码获取

- **lore 线程**：<https://lore.kernel.org/qemu-devel/20260803152832.222022-1-guochun.wang@foxmail.com/T/>
- **mbox 下载**：<https://lore.kernel.org/qemu-devel/20260803152832.222022-1-guochun.wang@foxmail.com/t.mbox.gz>
- **patchwork 系列**：<https://patchwork.kernel.org/project/qemu-devel/list/?series=1139691>

```bash
# 获取并应用补丁
curl -sL "https://lore.kernel.org/qemu-devel/20260803152832.222022-1-guochun.wang@foxmail.com/t.mbox.gz" | gunzip | git am
```

---

## 2. DesignWare SSI (SPI PIO)

**作者**：Kangjie Huang  
**版本**：v2 0/5（2026-08-01）  
**前序版本**：v1 0/11（SPI/QSPI/IDMA/XIP 系列，2026-07-26，已被拆分替代）

### 功能概述

可复用的 Synopsys DesignWare SSI 控制器模型，用于建模 K230 上的三个 SSI 实例。v2 将通用 DesignWare SSI 模型与 K230 集成层分离，使其他使用相同 IP 的 SoC 也能复用。

实现范围（仅 Standard 单线 SPI PIO）：
- 可配置片选数量和 FIFO 深度，四种 TMOD 模式
- FIFO 与状态处理、Standard 中断状态、迁移状态
- K230 每实例 9 条 SSI 中断源线全部接线到 PLIC
- 在 spi0 CS0 上挂载可选 M25P80 兼容 SPI NOR flash（仅 Standard 1-1-1 访问）

**暂未实现**（推迟到后续系列）：Enhanced SPI、内部 DMA/IDMA、XIP 事务。

### 补丁列表

1. `hw/ssi: Add Synopsys DesignWare SSI standard PIO controller`
2. `hw/ssi: Add DesignWare SSI standard interrupt support`
3. `hw/riscv/k230: Instantiate DesignWare SSI controllers`
4. `hw/riscv/k230: Route SSI interrupts to the PLIC`
5. `hw/riscv/k230: Attach a standard SPI NOR flash`

### 代码获取

- **lore 线程**：<https://lore.kernel.org/qemu-devel/20260801192848.30606-1-flamboyant.h.01@gmail.com/T/>
- **mbox 下载**：<https://lore.kernel.org/qemu-devel/20260801192848.30606-1-flamboyant.h.01@gmail.com/t.mbox.gz>
- **patchwork 系列**：<https://patchwork.kernel.org/project/qemu-devel/list/?series=1138717>

```bash
curl -sL "https://lore.kernel.org/qemu-devel/20260801192848.30606-1-flamboyant.h.01@gmail.com/t.mbox.gz" | gunzip | git am
```

---

## 3. DW APB Timer

**作者**：raoyi  
**版本**：v2 0/3（2026-07-30）  
**前序版本**：v1（2026-07-26）

### 功能概述

K230 DesignWare APB 定时器模型。使用 QEMU Clock 框架输入频率，为未来 CMU（时钟管理单元）集成做准备。模型包含 6 个定时器通道，支持周期/单次模式、中断屏蔽、EOI 清中断、PWM LoadCount2 寄存器等。

### 评审状态：v2 未满足架构要求，仍需 v3

**Bin Meng 在 v1 和 v2 中均提出相同的架构要求**：建模为通用 DesignWare APB Timer，再视需要叠加 K230 定制包装层（与 SSI 系列相同的拆分模式）。v2 发布当天（7/30），Bin Meng 再次回复：

> This one should be modeled as a generic DesignWare APB Timer, then optionally a K230 customized wrapper model.

v2 **未完成此拆分**，仅做了代码层面改进：
- 拆分为 3 个补丁（模型/板级集成/qtest）
- 去除 `goto` 写法
- 修正 include 顺序，补 QOM `<private>`/`<public>` 标记
- 修正版权头

代码仍为 K230 专用（对比 SSI v2 已拆为 `dw_ssi.c` 通用模型）：
- 文件名：`k230_dwapb_timer.c`（非 `dw_apb_timer.c`）
- 类型名：`TYPE_K230_TIMER` = `"riscv.k230.timer"`
- Kconfig：`CONFIG_K230_TIMER`（非 `CONFIG_DW_APB_TIMER`）
- 所有函数前缀：`k230_timer_*`
- 仍在使用已弃用的 `device_class_set_legacy_reset`

作者尚未回复 v2 反馈，至少需要一轮 v3 完成通用模型拆分。

### 补丁列表

1. `hw/timer: add k230 dwapb timer model`
2. `hw/riscv: integrate k230 dwapb timer into k230 board`
3. `tests/qtest: add k230 dwapb timer test`

### 代码获取

- **lore 线程**：<https://lore.kernel.org/qemu-devel/20260730133713.3253-1-rao232328@gmail.com/T/>
- **mbox 下载**：<https://lore.kernel.org/qemu-devel/20260730133713.3253-1-rao232328@gmail.com/t.mbox.gz>
- **patchwork 系列**：<https://patchwork.kernel.org/project/qemu-devel/list/?series=1137448>

```bash
curl -sL "https://lore.kernel.org/qemu-devel/20260730133713.3253-1-rao232328@gmail.com/t.mbox.gz" | gunzip | git am
```

---

## 4. SDHCI

**作者**：Xin Xie  
**版本**：v1 0/3（2026-07-29）

### 功能概述

K230 SDHCIv3 模型，基于 QEMU 通用 SDHCI 模型。K230 包含两个 Synopsys DesignWare Core Mobile Storage Host Controller，此前其 MMIO 范围以 `unimplemented` 设备暴露，软件无法访问 SD 存储。

实现内容：
- 复用通用命令、PIO、SDMA 和 ADMA2 数据路径
- K230 0x1000 字节寄存器布局，文档化的能力和复位值
- 只读 Preset Value 和扩展指针寄存器，可写的 PHY/厂商寄存器
- 始终就绪的 `PHY_PWRGOOD` 状态位（K230 SDK 复位序列需要）
- 报告为 v3（非 v4.20），因为通用 QEMU SDHCI 不支持 ADMA3

### 补丁列表

1. `hw/sd: add Kendryte K230 SDHCI controller`
2. `hw/riscv: enable SDHCI controllers on K230`
3. `tests/qtest: add K230 SDHCI tests`

### 代码获取

- **lore 线程**：<https://lore.kernel.org/qemu-devel/20260729105904.272939-1-xinxie908@gmail.com/T/>
- **mbox 下载**：<https://lore.kernel.org/qemu-devel/20260729105904.272939-1-xinxie908@gmail.com/t.mbox.gz>
- **patchwork 系列**：<https://patchwork.kernel.org/project/qemu-devel/list/?series=1136613>

```bash
curl -sL "https://lore.kernel.org/qemu-devel/20260729105904.272939-1-xinxie908@gmail.com/t.mbox.gz" | gunzip | git am
```

---

## 5. GSDMA + Decomp Gzip

**作者**：Tao Ding  
**版本**：v2 0/7（2026-07-27）  
**前序版本**：v1 0/7（2026-07-21）

### 功能概述

为 K230 板添加 GSDMA（通用安全 DMA）和解压缩引擎（decomp_gzip），使 K230 能在 U-Boot 阶段使用 `k230_unzip` 解压文件。

实现内容：
- **GSDMA**：支持 SDMA 模式（GDMA 暂不支持），含端序转换、2D 模式、LLT 链表
- **decomp_gzip**：与 SDMA 协同工作的 gzip 硬件解压器，支持 Dynamic Huffman 校验
- 额外添加 noc-stub 区域

v2 修复：端序转换、复位方法替换、DMA 完成后清除使能寄存器、2D 模式 + SRC_FIXED 地址递增 bug、Dynamic Huffman 格式校验、DECOMP_START 只写位修复等。

### 补丁列表

1. `hw/dma: add K230 gsdma`
2. `hw/riscv: k230: add gsdma in K230 board`
3. `tests/qtest: add test for K230 gsdma`
4. `hw/misc: add K230 decomp gzip`
5. `hw/riscv: k230: add decomp gzip in K230 board`
6. `tests/qtest: add test for K230 decomp gzip`
7. `hw/riscv: k230: add a noc stub region in K230 board`

### 代码获取

- **lore 线程**：<https://lore.kernel.org/qemu-devel/20260727155702.36484-1-dingtao0430@163.com/T/>
- **mbox 下载**：<https://lore.kernel.org/qemu-devel/20260727155702.36484-1-dingtao0430@163.com/t.mbox.gz>
- **patchwork 系列**：<https://patchwork.kernel.org/project/qemu-devel/list/?series=1135263>

```bash
curl -sL "https://lore.kernel.org/qemu-devel/20260727155702.36484-1-dingtao0430@163.com/t.mbox.gz" | gunzip | git am
```

---

## 6. SPI / QSPI / IDMA / XIP（原始完整系列）

**作者**：Kangjie Huang  
**版本**：v1 00/11（2026-07-26）  
**状态**：已被模块 2（DesignWare SSI v2）部分替代，剩余 Enhanced QSPI / IDMA / XIP 部分等待后续系列

### 功能概述

K230 DesignWare SSI 控制器的完整实现系列，覆盖 SPI 全功能。v1 将通用控制器行为与 K230 SoC 集成混合在一个设备中，评审后按 Bin Meng 要求拆分为通用模型 + K230 包装层（即模块 2）。

原始 v1 包含的功能：
- SSI 寄存器模型、FIFO 与 Standard PIO 传输
- SSI 中断控制器、PLIC 路由
- Enhanced QSPI 传输（双线/四线）
- SPI NOR flash 挂载
- 内部 DMA（IDMA）传输
- HI_SYS SSI 控制
- XIP 读窗口
- Trace 事件

### 补丁列表

1. `hw/ssi: Add K230 DesignWare SSI register model`
2. `hw/riscv/k230: Instantiate K230 SSI controllers`
3. `hw/ssi: Implement K230 SSI FIFO and standard PIO transfers`
4. `hw/ssi: Add K230 SSI interrupt controller`
5. `hw/riscv: Route K230 SSI IRQs to the PLIC`
6. `hw/ssi: Implement K230 enhanced QSPI transfers`
7. `hw/riscv/k230: Attach SPI NOR flash to spi0`
8. `hw/ssi: Implement K230 SSI internal DMA transfers`
9. `hw/misc: Add K230 HI_SYS SSI control`
10. `hw/ssi: Add K230 SSI XIP read window`
11. `hw/ssi: Add trace events for K230 DesignWare SSI`

### 代码获取

- **lore 线程**：<https://lore.kernel.org/qemu-devel/cover.1785064312.git.flamboyant.h.01@gmail.com/T/>
- **mbox 下载**：<https://lore.kernel.org/qemu-devel/cover.1785064312.git.flamboyant.h.01@gmail.com/t.mbox.gz>
- **patchwork 系列**：<https://patchwork.kernel.org/project/qemu-devel/list/?series=1134650>

```bash
curl -sL "https://lore.kernel.org/qemu-devel/cover.1785064312.git.flamboyant.h.01@gmail.com/t.mbox.gz" | gunzip | git am
```

---

## 7. DW 8250 UART

**作者**：WX Chen  
**版本**：RESEND v2 0/3（2026-07-25）  
**前序版本**：v1（zhenbaii，2026-07-24）

### 功能概述

K230 SoC DesignWare 8250 兼容 UART 控制器模型，能运行 Linux `8250_dw` 驱动并提供交互式 shell。

实现内容：
- 标准 16550 寄存器（RBR/THR/DLL, IER/DLH, IIR/FCR, LCR, MCR, LSR, MSR, SCR）
- DesignWare 专用寄存器（USR, TFL, RFL, SRR, SRTS, SBCR, SDMAM, SFE, SRT, STET, HTX, CPR, UCV, CTR）
- DLAB 切换，32 字节 TX/RX FIFO，同步发送与背压处理
- 回环模式，BREAK 经 CHR_EVENT_BREAK 路由到 LSR.BI/FE
- 四级优先中断方案（ELSI / RX Data / Timeout / THRE），THRE 边沿触发
- RX 字符超时中断

### 补丁列表

1. `hw/char: add K230 DW 8250-compatible UART`
2. `hw/riscv: k230: connect DW 8250 UART`
3. `tests/qtest: add K230 UART test`

### 代码获取

- **lore 线程**：<https://lore.kernel.org/qemu-devel/20260725-feat-k230-uart-v2-v2-0-d5fe82c47c28@gmail.com/T/>
- **mbox 下载**：<https://lore.kernel.org/qemu-devel/20260725-feat-k230-uart-v2-v2-0-d5fe82c47c28@gmail.com/t.mbox.gz>
- **patchwork 系列**：<https://patchwork.kernel.org/project/qemu-devel/list/?series=1134375>

```bash
curl -sL "https://lore.kernel.org/qemu-devel/20260725-feat-k230-uart-v2-v2-0-d5fe82c47c28@gmail.com/t.mbox.gz" | gunzip | git am
```

---

## 8. SRAM

**作者**：Jian Cai  
**版本**：v2 单补丁（2026-07-21）  
**前序版本**：RFC 0/3（2026-07-20）

### 功能概述

K230 共享 SRAM（2 MB，位于 0x80200000）是纯片上 RAM 块，无软件可见的控制器寄存器，通过 AXI 总线访问。v2 按评审意见放弃了 SysBusDevice 模型，改为在 `k230.c` 中使用 `k230_sram_create()` 静态辅助函数（与现有 `k230_create_plic()`、`k230_create_uart()` 模式一致），替换原有的内联 `memory_region_init_ram` 调用。

### 补丁列表

1. `hw/riscv/k230: add k230_sram_create() helper function`

### 代码获取

- **lore 线程**：<https://lore.kernel.org/qemu-devel/20260721060409.9688-2-lingqian_gi@163.com/T/>
- **mbox 下载**：<https://lore.kernel.org/qemu-devel/20260721060409.9688-2-lingqian_gi@163.com/raw>
- **patchwork 系列**：<https://patchwork.kernel.org/project/qemu-devel/list/?series=1131319>

```bash
curl -sL "https://lore.kernel.org/qemu-devel/20260721060409.9688-2-lingqian_gi@163.com/raw" | git am
```

---

## 9. Reset Management Unit (RMU)

**作者**：Jack Wang  
**版本**：v2 0/2（2026-07-19）  
**前序版本**：RFC 0/2（2026-07-09）

### 功能概述

K230 复位管理单元（RMU）模型，位于 0x91101000 的一组复位控制寄存器。替换原有的 `create_unimplemented_device("rmu")` stub，使 guest 复位驱动（`drivers/reset/reset-k230.c`）能对真实模型工作。

实现内容：
- 使用 Resettable API（`phases.hold`，替代已弃用的 `device_class_set_legacy_reset()`）
- 真实复位传播：写入 PERI0 复位位时通过 `wdt0`/`wdt1` QOM 链接冷复位两个看门狗
- CPU1 复位请求为两步 assert/deassert（非自清零），匹配硬件行为
- 寄存器复位值和保留位掩码来自 K230 TRM 第 2.1 章 "Reset"
- 复位时间控制寄存器（`*_RST_TIM`）作为纯存储

### 补丁列表

1. `hw/misc/k230_rmu: add Kendryte K230 Reset Management Unit model`
2. `hw/riscv/k230: wire up the RMU device`

### 代码获取

- **lore 线程**：<https://lore.kernel.org/qemu-devel/20260719180247.8660-1-163wangjack@gmail.com/T/>
- **mbox 下载**：<https://lore.kernel.org/qemu-devel/20260719180247.8660-1-163wangjack@gmail.com/t.mbox.gz>
- **patchwork 系列**：<https://patchwork.kernel.org/project/qemu-devel/list/?series=1130359>

```bash
curl -sL "https://lore.kernel.org/qemu-devel/20260719180247.8660-1-163wangjack@gmail.com/t.mbox.gz" | gunzip | git am
```

---

## 10. DDR Controller + PHY

**作者**：Junze Cao  
**版本**：v1 0/3（2026-07-16）

### 功能概述

K230 DDR 控制器（DDRC）和 DDR PHY 模型。K230 SDK U-Boot SPL 在 DRAM 可用前需要配置 DDRC CFG 和 DDR PHY 寄存器。

实现内容：
- 为两个寄存器范围分别建立 SysBus 设备
- **控制器**：复位值、DFI 和软件更新握手
- **PHY**：寄存器所有权、训练邮箱（training mailbox）、DFI 完成
- 两个设备均包含迁移状态

### 补丁列表

1. `hw/misc: Add K230 DDR controller and PHY models`
2. `hw/riscv: Connect K230 DDR controller and PHY models`
3. `tests/qtest: Add K230 DDR controller tests`

### 代码获取

- **lore 线程**：<https://lore.kernel.org/qemu-devel/20260716132423.931427-1-caojunze424@gmail.com/T/>
- **mbox 下载**：<https://lore.kernel.org/qemu-devel/20260716132423.931427-1-caojunze424@gmail.com/t.mbox.gz>
- **patchwork 系列**：<https://patchwork.kernel.org/project/qemu-devel/list/?series=1128852>

```bash
curl -sL "https://lore.kernel.org/qemu-devel/20260716132423.931427-1-caojunze424@gmail.com/t.mbox.gz" | gunzip | git am
```

---

## 11. IOMUX

**作者**：Kangjie Huang  
**版本**：v2 0/3（2026-07-14）  
**前序版本**：RFC 1/1（2026-07-10）

### 功能概述

K230 IOMUX 模型，为 K230 技术参考手册中记录的 64 个 Function IO 配置寄存器建立 SysBus 模型。

实现内容：
- 保持可写配置字段在 32 位对齐访问中的值
- 应用文档化的复位值，忽略对只读和保留字段的写入
- 文档未记录的偏移读零且忽略写入
- 供 Kendryte SDK U-Boot `pinctrl-single` 驱动和 Linux IOMUX 配置代码使用

### 补丁列表

1. `hw/misc/k230_iomux: add Kendryte K230 IOMUX model`
2. `hw/riscv/k230: wire up the IOMUX device`
3. `tests/qtest: add test for K230 IOMUX`

### 代码获取

- **lore 线程**：<https://lore.kernel.org/qemu-devel/cover.1784022244.git.flamboyant.h.01@gmail.com/T/>
- **mbox 下载**：<https://lore.kernel.org/qemu-devel/cover.1784022244.git.flamboyant.h.01@gmail.com/t.mbox.gz>
- **patchwork 系列**：<https://patchwork.kernel.org/project/qemu-devel/list/?series=1127309>

```bash
curl -sL "https://lore.kernel.org/qemu-devel/cover.1784022244.git.flamboyant.h.01@gmail.com/t.mbox.gz" | gunzip | git am
```

---

## 附：批量获取所有补丁

```bash
# 下载所有 11 个系列的 mbox 并依次 apply
declare -a SERIES=(
  "20260803152832.222022-1-guochun.wang@foxmail.com"
  "20260801192848.30606-1-flamboyant.h.01@gmail.com"
  "20260730133713.3253-1-rao232328@gmail.com"
  "20260729105904.272939-1-xinxie908@gmail.com"
  "20260727155702.36484-1-dingtao0430@163.com"
  "cover.1785064312.git.flamboyant.h.01@gmail.com"
  "20260725-feat-k230-uart-v2-v2-0-d5fe82c47c28@gmail.com"
  "20260721060409.9688-2-lingqian_gi@163.com"
  "20260719180247.8660-1-163wangjack@gmail.com"
  "20260716132423.931427-1-caojunze424@gmail.com"
  "cover.1784022244.git.flamboyant.h.01@gmail.com"
)

for id in "${SERIES[@]}"; do
  echo "=== Fetching $id ==="
  curl -sL "https://lore.kernel.org/qemu-devel/${id}/t.mbox.gz" | gunzip | git am --3way || echo "FAILED: $id"
done
```

> **注意**：模块 2 和模块 6 有重叠（SSI 部分），不建议同时 apply。建议只 apply 模块 2（v2 拆分版）。
