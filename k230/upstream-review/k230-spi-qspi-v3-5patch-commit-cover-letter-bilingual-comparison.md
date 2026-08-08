# K230 SPI/QSPI V3 五补丁：基于 V2 的最小修订拟稿

这是供审阅的拟发送文案，不是当前 `T_v3-5patch` commit message 的逐字抄录。
目标是把已发送的 V2 五个 patch 作为母本，只改 DWC 命名、`CTRLR0.TMOD` 布局、
已验证的 qtest 数量和必要的 bug-fix 概述；保留 V2 已有的 U-Boot/Linux 实测声明。
本文件不改源码、不改 commit。

## 固定边界

| 项目 | 拟定规则 |
|---|---|
| Patch 1/2 | 通用 DWC SSI patch；英文正文不出现 `K230`、`K230 SSI` 或 K230 集成表述 |
| Patch 3–5 | K230 集成 patch，可说明实例、PLIC、SPI NOR 与 SDK 验证 |
| 系列 | 仍为 V2 的同一 base、同一顺序、同一 5 个 patch |
| 测试 | 当前实际二进制注册 14 项；邮件中如实写 14，不把收敛后的数量包装成新的功能范围 |
| V2 实测 | Patch 5 和 cover letter 保留 Standard 1-1-1 经 K230 SDK U-Boot/Linux SPI 路径手工验证的声明 |
| 禁止混入 | 不加 `MAINTAINERS`、不另起 patch、不在 Patch 1/2 把通用控制器与 K230 绑定 |

## V2 → V3 提交对应

| 序号 | V2 | V3 拟定 subject | 允许的主要改动 |
|---|---|---|---|
| 1/5 | `e975addedc` | `hw/ssi: Add Synopsys DWC SSI standard PIO controller` | 命名、`CTRLR0.TMOD` 位域；bug-fix 概述只写在 cover letter |
| 2/5 | `534f116c04` | `hw/ssi: Add DWC SSI standard interrupt support` | 命名、DONE/AXIE Standard-only 边界 |
| 3/5 | `297b848a1d` | `hw/riscv/k230: Instantiate DWC SSI controllers` | 命名、现有 qtest 覆盖说明 |
| 4/5 | `8fc80e0474` | `hw/riscv/k230: Route SSI interrupts to the PLIC` | 命名、TXE/RXU 路由和实例隔离 qtest 覆盖 |
| 5/5 | `b0fc2d4893` | `hw/riscv/k230: Attach a standard SPI NOR flash` | EEPROM-read 非零地址回归；恢复 U-Boot/Linux 实测声明 |

V2 和 V3 使用同一 base：`b428fe036233cbd15d37e3c027ab6ca4d3661a80`。

## Patch 1/5 拟稿

### English

```text
hw/ssi: Add Synopsys DWC SSI standard PIO controller

Add a reusable SysBus model for the Synopsys DWC SSI controller. The model
uses the DWC CTRLR0 layout, where TMOD is at bits [11:10]; DW APB SSI
variants with TMOD at bits [9:8] are not supported.

Implement the Standard SPI register subset, configurable chip-select and
FIFO resources, the four Standard PIO transfer modes, reset handling,
chip-select GPIOs, and RAZ/WI handling for unsupported enhanced, DMA, and
XIP registers.

Add num-cs, fifo-depth, and imr-reset properties. Save only Standard PIO
state in VMState and reject migration between devices with different
resource profiles.

The Synopsys databook requires myDesignWare registration and has no stable
public URL. The K230 TRM is the public register reference for this model:

https://github.com/revyos/external-docs/blob/79b3a79072412ead81427e6755b4d9e6d9ded8d8/K230/en-us/K230_Technical_Reference_Manual_V0.3.1_20241118.pdf

The Linux driver and binding provide software and capability references:

https://github.com/torvalds/linux/blob/f9a2394a23482bfd330911e9c8295b71724feacd/drivers/spi/spi-dw-core.c

https://github.com/torvalds/linux/blob/f9a2394a23482bfd330911e9c8295b71724feacd/drivers/spi/spi-dw.h

https://github.com/torvalds/linux/blob/f9a2394a23482bfd330911e9c8295b71724feacd/Documentation/devicetree/bindings/spi/snps,dw-apb-ssi.yaml

The Intel Arria 10 HPS TRM describes the DW APB SSI family variant and is
used only for comparison with the DWC layout:

https://www.intel.com/content/www/us/en/docs/programmable/683711/21-2/hard-processor-system-technical-reference.html

Suggested-by: Bin Meng <bmeng.cn@gmail.com>
Suggested-by: Chao Liu <chao.liu@processmission.com>
Suggested-by: Anirudh Srinivasan <asrinivasan@oss.tenstorrent.com>

The controller is not instantiated by this patch.

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
```

### 中文翻译

```text
hw/ssi：增加 Synopsys DWC SSI 标准 PIO 控制器

增加可复用的 Synopsys DWC SSI SysBus 模型。该模型采用 DWC 的 CTRLR0
布局，其中 TMOD 位于 [11:10]；不支持 TMOD 位于 [9:8] 的 DW APB SSI
变体。

实现 Standard SPI 寄存器子集、可配置的片选和 FIFO 资源、四种 Standard PIO
传输模式、复位处理、片选 GPIO，以及对不支持的 enhanced、DMA 和 XIP 寄存器的
RAZ/WI 处理。

增加 `num-cs`、`fifo-depth` 和 `imr-reset` 属性。VMState 只保存 Standard PIO
状态，并拒绝在资源 profile 不同的设备之间迁移。

Synopsys databook 需要 myDesignWare 注册，且没有稳定的公开 URL。K230 TRM 是
本模型的公开寄存器参考：

https://github.com/revyos/external-docs/blob/79b3a79072412ead81427e6755b4d9e6d9ded8d8/K230/en-us/K230_Technical_Reference_Manual_V0.3.1_20241118.pdf

Linux 驱动和 binding 提供软件行为与 capability 参考：

https://github.com/torvalds/linux/blob/f9a2394a23482bfd330911e9c8295b71724feacd/drivers/spi/spi-dw-core.c

https://github.com/torvalds/linux/blob/f9a2394a23482bfd330911e9c8295b71724feacd/drivers/spi/spi-dw.h

https://github.com/torvalds/linux/blob/f9a2394a23482bfd330911e9c8295b71724feacd/Documentation/devicetree/bindings/spi/snps,dw-apb-ssi.yaml

Intel Arria 10 HPS TRM 描述 DW APB SSI family variant，仅用于与 DWC 布局进行
对比：

https://www.intel.com/content/www/us/en/docs/programmable/683711/21-2/hard-processor-system-technical-reference.html

Suggested-by：Bin Meng、Chao Liu、Anirudh Srinivasan

本 patch 不实例化控制器。

Signed-off-by：Kangjie Huang
```

说明：这段保留 V2 的通用模型、属性和 VMState 叙述，只补 DWC `TMOD` 布局。公开链接
和三个 `Suggested-by` 全部保留；正文只将它们作为通用模型
的证据来源，不把控制器表述为 K230 专用设备。

## Patch 2/5 拟稿

### English

```text
hw/ssi: Add DWC SSI standard interrupt support

Implement raw and masked SSI interrupt status, threshold interrupts, and
the documented read-clear registers.

TXE and RXF are derived from FIFO levels. TXO, RXO, and RXU are latched
causes. Expose the model's nine physical interrupt outputs.

TXU, DONE, and AXIE depend on transfer engines outside the Standard PIO
scope and remain deasserted. DONECR and AXIECR are RAZ/WI.

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
```

### 中文翻译

```text
hw/ssi：增加 DWC SSI 标准中断支持

实现 SSI 原始和屏蔽后的中断状态、阈值中断，以及文档定义的读清除寄存器。

TXE 和 RXF 根据 FIFO 水位派生；TXO、RXO 和 RXU 是锁存原因。暴露模型的九个
物理中断输出。

TXU、DONE 和 AXIE 依赖 Standard PIO 范围之外的传输引擎，并保持撤销。DONECR
和 AXIECR 采用 RAZ/WI 语义。

Signed-off-by：Kangjie Huang
```

说明：V2 的中断语义保持不变；这里只把名称改为 DWC，并明确 Standard-only 下
DONE/AXIE 没有事件源。正文不提 K230。

## Patch 3/5 拟稿

### English

```text
hw/riscv/k230: Instantiate DWC SSI controllers

Instantiate the three DWC SSI profiles used by K230 and map their
controller MMIO regions at the documented addresses.

Configure the QSPI0, QSPI1, and SPI-OPI instances with their chip-select,
FIFO-depth, and interrupt-mask reset profiles.

Add K230 SSI qtests for Standard PIO transfers, register masks, FIFO
behaviour, interrupt state, unsupported-register semantics, reset profiles,
and the Standard PIO regressions fixed in this revision. K230 is used as
the hardware carrier for the generic controller tests; PLIC routing is
added by the next patch.

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
```

### 中文翻译

```text
hw/riscv/k230：实例化 DWC SSI 控制器

实例化 K230 使用的三个 DWC SSI profile，并将控制器 MMIO 区域映射到文档规定
的地址。

为 QSPI0、QSPI1 和 SPI-OPI 实例配置片选数量、FIFO 深度和中断屏蔽复位 profile。

增加 K230 SSI qtest，覆盖 Standard PIO 传输、寄存器 mask、FIFO 行为、中断状态、
unsupported-register 语义、复位 profile，以及本轮修复的 Standard PIO 回归。
K230 用作通用控制器测试的硬件载体；PLIC 路由由下一个 patch 增加。

Signed-off-by：Kangjie Huang
```

## Patch 4/5 拟稿

### English

```text
hw/riscv/k230: Route SSI interrupts to the PLIC

Connect the nine interrupt outputs of each K230 DWC SSI controller to the
documented PLIC source range.

The TXU, DONE, and AXIE lines are wired to preserve the physical K230
interrupt topology, but remain inactive until their transfer engines are
modelled. Extend qtests to cover TXE routing for all instances, RXU routing,
and instance isolation.

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
```

### 中文翻译

```text
hw/riscv/k230：将 SSI 中断路由到 PLIC

将每个 K230 DWC SSI 控制器的九个中断输出连接到文档规定的 PLIC source 范围。

TXU、DONE 和 AXIE 线路保留 K230 的物理中断拓扑，但在其传输引擎完成建模前保持
不活动。扩展 qtest，覆盖所有实例的 TXE 路由、RXU 路由和实例隔离。

Signed-off-by：Kangjie Huang
```

## Patch 5/5 拟稿

### English

```text
hw/riscv/k230: Attach a standard SPI NOR flash

Add the optional spi-flash machine property and attach the selected
M25P80-compatible flash device to logical spi0 chip select 0.

Use the supplied MTD backend when present and retain the erased-flash
default otherwise. Document the machine option and add qtests for Standard
1-1-1 JEDEC identification, fixed-address reads, and EEPROM-read with a
multi-byte command and non-zero address.

Standard 1-1-1 transfers were also exercised manually through the K230
SDK U-Boot and Linux SPI paths against the attached flash.

The enhanced SPI and IDMA boot paths are outside the scope of this series.

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
```

### 中文翻译

```text
hw/riscv/k230：挂接标准 SPI NOR flash

增加可选的 `spi-flash` machine property，并将选定的 M25P80 兼容 flash 设备
挂接到逻辑上的 spi0 CS0。

如果提供了 MTD backend，则使用该 backend；否则保留擦除状态的 flash 默认内容。
补充 machine option 文档，并增加 qtest，覆盖 Standard 1-1-1 JEDEC 识别、固定
地址读取，以及带多字节 command 和非零地址的 EEPROM-read。

还通过 K230 SDK 的 U-Boot 和 Linux SPI 路径，针对已挂接 flash 手工验证了
Standard 1-1-1 传输。

enhanced SPI 和 IDMA 启动路径不在本系列范围内。

Signed-off-by：Kangjie Huang
```

说明：恢复 V2 原有的 U-Boot/Linux 手工验证声明；唯一新增的是 EEPROM-read
非零地址 qtest 的事实描述。

## Cover letter 拟稿

### English

```text
Subject: [PATCH v3 0/5] hw/riscv: Add K230 DWC SSI Standard PIO support

Hi,

This series adds a reusable Standard PIO model for the Synopsys DWC SSI
controller. The K230 machine has three such controllers, and the later
patches add their machine integration.

The first patch adds the Standard SPI register subset, configurable FIFO
and chip-select resources, four transfer modes, reset and migration state,
and RAZ/WI handling for unsupported enhanced SPI, DMA, and XIP registers.
It uses the DWC CTRLR0 layout, where TMOD is at bits [11:10]. The second
patch adds the Standard PIO interrupt support. The remaining patches add
the K230 instances, PLIC routing, and an optional SPI NOR on spi0 CS0.

Changes since v2:

* Rename the controller model and integration from DesignWare/DW to DWC.
* Clarify the DWC CTRLR0.TMOD layout and keep DW APB SSI variants out of
  scope.
* Fix Standard PIO transfer progress for TX-only, RX FIFO overrun in all
  PIO modes, and RX-only dummy frames.
* Merge the associated regression assertions into the existing qtests,
  including a non-zero SPI NOR EEPROM-read address.

Enhanced SPI, internal IDMA, HI_SYS, and XIP remain separate follow-up
series. The older DW APB SSI TMOD encoding is not part of this series.

Testing:

* Built qemu-system-riscv64 and the K230 DWC SSI qtest target.
* Ran all 14 K230 DWC SSI qtests, including Standard PIO modes, register
  contracts, interrupts and PLIC routing, and Standard SPI NOR reads.
* Standard 1-1-1 transfers were also exercised manually through the K230
  SDK U-Boot and Linux SPI paths against the attached flash.
* git diff --check passed.
* checkpatch.pl reported no code errors; its MAINTAINERS coverage warnings
  are expected because this series does not modify MAINTAINERS.

Kangjie Huang (5):
  hw/ssi: Add Synopsys DWC SSI standard PIO controller
  hw/ssi: Add DWC SSI standard interrupt support
  hw/riscv/k230: Instantiate DWC SSI controllers
  hw/riscv/k230: Route SSI interrupts to the PLIC
  hw/riscv/k230: Attach a standard SPI NOR flash

base-commit: b428fe036233cbd15d37e3c027ab6ca4d3661a80
--
2.43.0
```

### 中文翻译

```text
主题：[PATCH v3 0/5] hw/riscv：增加 K230 DWC SSI Standard PIO 支持

你好：

本系列为 Synopsys DWC SSI 控制器增加可复用的 Standard PIO 模型。K230 machine
有三个这样的控制器，后续 patch 再增加它们的 machine 集成。

第一个 patch 增加 Standard SPI 寄存器子集、可配置 FIFO 和片选资源、四种传输模式、
复位和 migration 状态，以及对不支持的 enhanced SPI、DMA 和 XIP 寄存器的 RAZ/WI
处理。它采用 DWC CTRLR0 布局，TMOD 位于 [11:10]。第二个 patch 增加 Standard PIO
中断支持。其余 patch 增加 K230 实例、PLIC 路由，以及 spi0 CS0 上可选的 SPI NOR。

相对 v2 的改动：

* 将控制器模型和集成中的 DesignWare/DW 命名统一为 DWC。
* 明确 DWC `CTRLR0.TMOD` 布局，并继续将 DW APB SSI 变体排除在范围外。
* 修复 TX-only、RX FIFO overrun（TR/RO/EEPROM-read 三种模式）和
  RX-only dummy frame 的 Standard PIO 传输推进问题。
* 将关联的回归断言并入既有 qtest，包括 SPI NOR EEPROM-read 非零地址。

Enhanced SPI、内部 IDMA、HI_SYS 和 XIP 仍作为后续独立系列。旧 DW APB SSI 的
TMOD 编码不属于本系列。

测试：

* 构建 `qemu-system-riscv64` 和 K230 DWC SSI qtest target。
* 运行全部 14 个 K230 DWC SSI qtest，包括 Standard PIO 模式、寄存器契约、
  中断和 PLIC 路由，以及 Standard SPI NOR 读取。
* 还通过 K230 SDK 的 U-Boot 和 Linux SPI 路径，针对已挂接 flash 手工验证
  Standard 1-1-1 传输。
* `git diff --check` 通过。
* `checkpatch.pl` 未报告代码错误；由于本系列不修改 `MAINTAINERS`，仅有新增文件的
  maintainer coverage 提示。

Kangjie Huang（5）：
  hw/ssi：增加 Synopsys DWC SSI 标准 PIO 控制器
  hw/ssi：增加 DWC SSI 标准中断支持
  hw/riscv/k230：实例化 DWC SSI 控制器
  hw/riscv/k230：将 SSI 中断路由到 PLIC
  hw/riscv/k230：挂接标准 SPI NOR flash

base-commit：b428fe036233cbd15d37e3c027ab6ca4d3661a80
```

## 对比与待你确认的点

| 项目 | V2 原样保留 | V3 最小修改 |
|---|---|---|
| Patch 1/2 的职责 | 通用控制器与通用 IRQ | 仅改为 DWC 名称、补 `CTRLR0.TMOD`；不出现 K230 |
| K230 语义 | 只在 machine/flash patch 出现 | 仅 Patch 3–5 与 cover letter 出现 |
| Patch 5 验证 | U-Boot/Linux 手工验证 | 恢复原句，并加入 EEPROM-read 非零地址 qtest |
| qtest | V2 为 14 项 | V3 仍为 14 项；新增断言并入已有用例，邮件与实际注册数一致 |
| bug 说明 | 无 | 仅在 cover letter 用四条简短 bullet 概述 |
| `MAINTAINERS` | V2 未改 | 不进入 V3；当前五个 patch 不修改该文件 |

Patch 1 保留公开链接和三个 `Suggested-by`，但不在正文用 K230 SSI 说明通用模型。
这将“证据来源”与“控制器归属”分开：链接仍可供 reviewer 核查，Patch 1/2 的模型
身份仍保持通用 DWC SSI。
