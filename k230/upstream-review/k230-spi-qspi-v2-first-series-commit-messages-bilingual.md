# K230 SPI/QSPI V2 第一批上游 Series：提交说明（中英文）

最后更新：2026-08-01

本文记录当前 V2 第一批 Standard SPI series 的 5 个正式提交说明草案，供
`git commit`、`git format-patch` 和 cover letter 使用。

基线：`b428fe036233cbd15d37e3c027ab6ca4d3661a80`

当前 series 的边界：

- 通用 DesignWare SSI Standard PIO/FIFO/基础状态；
- K230 三个 SSI 实例和 PLIC 路由；
- 9 路物理 IRQ 输出保持 K230 拓扑；
- TXU、DONE、AXIE 在当前 Standard PIO 模型中保持低电平；
- Standard 1-1-1 SPI NOR Flash；
- enhanced SPI、IDMA、XIP 数据路径不进入本 series。

K230 machine 作为 qtest 的真实 carrier。测试职责按通用控制器行为、K230
实例/PLIC 集成和 Flash 集成划分，不增加测试专用 machine。

## 1. `hw/ssi: Add Synopsys DesignWare SSI standard PIO controller`

### English

```text
hw/ssi: Add Synopsys DesignWare SSI standard PIO controller

Add a reusable SysBus model for the Synopsys DesignWare SSI controller.

Implement the Standard SPI register subset, configurable chip-select and
FIFO resources, the four Standard PIO transfer modes, reset handling,
chip-select GPIOs, and RAZ/WI handling for unsupported enhanced, DMA,
and XIP registers.

Add num-cs, fifo-depth, and imr-reset properties. Save only Standard PIO
state in VMState and reject migration between devices with different
resource profiles.

The controller is not instantiated by this patch.

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
```

### 中文

```text
hw/ssi：新增 Synopsys DesignWare SSI 标准 PIO 控制器

新增可复用的 Synopsys DesignWare SSI SysBus 模型。

实现 Standard SPI 寄存器子集、可配置片选和 FIFO 资源、四种 Standard
PIO 传输模式、复位处理、片选 GPIO，以及对未实现 enhanced、DMA 和 XIP
寄存器的 RAZ/WI 语义。

新增 num-cs、fifo-depth 和 imr-reset 属性。VMState 只保存 Standard PIO
状态，并拒绝在资源配置不同的设备之间迁移。

本补丁不实例化任何控制器。

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
```

## 2. `hw/ssi: Add DesignWare SSI standard interrupt support`

### English

```text
hw/ssi: Add DesignWare SSI standard interrupt support

Implement raw and masked SSI interrupt status, threshold interrupts, and
the documented read-clear registers.

TXE and RXF are derived from FIFO levels. TXO, RXO, and RXU are latched
causes. Expose the nine physical interrupt outputs used by the K230 SSI
integration.

TXU, DONE, and AXIE depend on transfer engines outside the Standard PIO
scope and remain deasserted in this model.

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
```

### 中文

```text
hw/ssi：新增 DesignWare SSI 标准中断支持

实现 SSI 原始/屏蔽中断状态、阈值中断和手册规定的读取清除寄存器。

TXE 和 RXF 由 FIFO 水位生成；TXO、RXO 和 RXU 为锁存型原因。暴露 K230
SSI 集成所使用的 9 路物理中断输出。

TXU、DONE 和 AXIE 依赖 Standard PIO 范围之外的传输引擎，因此在当前模型
中保持低电平。

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
```

## 3. `hw/riscv/k230: Instantiate DesignWare SSI controllers`

### English

```text
hw/riscv/k230: Instantiate DesignWare SSI controllers

Instantiate the three DesignWare SSI profiles used by K230 and map their
controller MMIO regions at the documented addresses.

Configure the QSPI0, QSPI1, and SPI-OPI instances with their chip-select,
FIFO-depth, and interrupt-mask reset profiles.

Add K230 SSI qtests for Standard PIO transfers, register masks, FIFO
behaviour, interrupt state, unsupported-register semantics, and reset
profiles. K230 is used as the hardware carrier for the generic controller
tests; PLIC routing is added by the next patch.

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
```

### 中文

```text
hw/riscv/k230：实例化 DesignWare SSI 控制器

实例化 K230 使用的三个 DesignWare SSI 配置，并将控制器 MMIO 映射到
文档规定的地址。

为 QSPI0、QSPI1 和 SPI-OPI 实例配置片选数量、FIFO 深度和中断掩码复位
参数。

新增 K230 SSI qtest，覆盖 Standard PIO 传输、寄存器掩码、FIFO 行为、
中断状态、未支持寄存器语义和复位 profile。K230 作为通用控制器测试的
硬件载体；PLIC 路由由下一补丁实现。

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
```

## 4. `hw/riscv/k230: Route SSI interrupts to the PLIC`

### English

```text
hw/riscv/k230: Route SSI interrupts to the PLIC

Connect the nine interrupt outputs of each K230 SSI controller to the
documented PLIC source range.

The TXU, DONE, and AXIE lines are wired to preserve the physical K230
interrupt topology, but remain inactive until their transfer engines are
modeled.

Extend qtests to cover TXE routing for all instances and RXU routing and
instance isolation.

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
```

### 中文

```text
hw/riscv/k230：将 SSI 中断路由至 PLIC

将每个 K230 SSI 控制器的 9 路中断输出连接到文档规定的 PLIC 中断源范围。

TXU、DONE 和 AXIE 连线用于保留 K230 的物理中断拓扑；在对应传输引擎完成
建模前，这些线路保持不活跃。

扩展 qtest，覆盖三个实例的 TXE 路由，以及 RXU 路由和实例隔离。

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
```

## 5. `hw/riscv/k230: Attach a standard SPI NOR flash`

### English

```text
hw/riscv/k230: Attach a standard SPI NOR flash

Add the optional spi-flash machine property and attach the selected
M25P80-compatible flash device to logical spi0 chip select 0.

Use the supplied MTD backend when present and retain the erased-flash
default otherwise. Document the machine option and add qtests for Standard
1-1-1 JEDEC identification and fixed-address reads.

Standard 1-1-1 transfers were also exercised manually through the K230
SDK U-Boot and Linux SPI paths against the attached flash.

The enhanced SPI and IDMA boot paths are outside the scope of this series.

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
```

### 中文

```text
hw/riscv/k230：挂接标准 SPI NOR Flash

新增可选的 spi-flash machine 属性，并将指定的 M25P80 兼容 Flash 挂接到
逻辑 spi0 的片选 0。

存在 MTD 后端时使用该后端；否则保留擦除态的默认 Flash。补充 machine
选项文档，并新增 Standard 1-1-1 JEDEC ID 和固定地址读取 qtest。

此外，已通过 K230 SDK 的 U-Boot 和 Linux SPI 路径，对挂接 Flash 的
Standard 1-1-1 传输进行了手工验证。

本补丁不宣称支持 enhanced SPI 或 IDMA 启动路径。

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
```

## Patch-to-file 归属

| Patch | 主要文件和职责 |
|---|---|
| 1 | `hw/ssi/dw_ssi.c`、`include/hw/ssi/dw_ssi.h`、SSI Kconfig/Meson、通用 VMState/RAZ-WI |
| 2 | `dw_ssi.c` IRQ raw/masked/clear、threshold、9 路 output |
| 3 | `hw/riscv/k230.c` 三实例/profile/MMIO、K230 SSI qtest、K230 docs/MAINTAINERS |
| 4 | K230 SSI 到 PLIC 的 9 路连接和路由测试 |
| 5 | `spi-flash` property、M25P80 attachment、Flash qtest、Flash 文档、`SSI_M25P80` Kconfig |

## 发送前检查

- 每个 patch 单独 checkout 后可以构建；
- 每个 patch 的新增行为都有对应测试，测试允许使用 K230 作为 carrier；
- 第 5 个 patch 的 U-Boot/Linux 描述仅限 Standard 1-1-1 SPI，不描述 enhanced/IDMA 启动；
- `MAINTAINERS` 覆盖 `include/hw/ssi/dw_ssi.h` 和 `tests/qtest/k230-dw-ssi-test.c`；
- 每个提交包含真实作者信息和 `Signed-off-by`；
- `git diff --check`、checkpatch、通用/K230 qtest 和 Flash 测试全部通过。
