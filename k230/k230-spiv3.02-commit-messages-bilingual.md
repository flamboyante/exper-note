# K230 SPI v3.02：10 个提交信息中英文对照

基线：k230-spiv3.02。以下按提交顺序记录完整英文提交信息及中文翻译。

## 1. 3b07653da2 — hw/ssi: Add K230 DesignWare SSI register model

### English

~~~text
hw/ssi: Add K230 DesignWare SSI register model

Add a SysBus model for the K230 DesignWare SSI controllers.

The model provides the documented register layout, reset values, writable
masks, FIFO state, chip-select GPIOs, and the MMIO regions used by the
K230 integration.  It does not instantiate a controller yet.

The machine integration and qtest coverage are introduced by follow-up
commits.

Signed-off-by: flamboy <flamboyant.h.01@gmail.com>
~~~

### 中文

~~~text
hw/ssi: 新增 K230 DesignWare SSI 寄存器模型

为 K230 DesignWare SSI 控制器新增一个 SysBus 模型。

该模型提供文档规定的寄存器布局、复位值、可写位掩码、FIFO 状态、
片选 GPIO，以及 K230 集成所需的 MMIO 区域。但暂不实例化控制器。

后续提交将引入机器集成和 qtest 覆盖。

Signed-off-by: flamboy <flamboyant.h.01@gmail.com>
~~~

## 2. bbb37f8b42 — hw/riscv/k230: Instantiate K230 SSI controllers

### English

~~~text
hw/riscv/k230: Instantiate K230 SSI controllers

Instantiate the three SSI controller profiles used by K230 and map their
MMIO regions at the documented addresses.

Add the single K230 SSI qtest target and its register-contract scenario.
The scenario covers reset values, register masks, read-only registers,
and configuration writes while the controller is enabled.

Signed-off-by: flamboy <flamboyant.h.01@gmail.com>
~~~

### 中文

~~~text
hw/riscv/k230: 实例化 K230 SSI 控制器

实例化 K230 使用的三个 SSI 控制器配置，并将其 MMIO 区域映射到
文档规定的地址。

新增统一的 K230 SSI qtest 测试目标及其寄存器契约测试场景。
该场景覆盖复位值、寄存器掩码、只读寄存器，以及控制器启用期间的
配置写入行为。

Signed-off-by: flamboy <flamboyant.h.01@gmail.com>
~~~

## 3. e753ad44f7 — hw/ssi: Implement K230 SSI FIFO and standard PIO transfers

### English

~~~text
hw/ssi: Implement K230 SSI FIFO and standard PIO transfers

Implement FIFO-backed standard SPI transfers for all four TMOD modes.

Model DR aliases, frame-width truncation, loopback, FIFO capacity,
dynamic status, receive backpressure, and controller disable semantics.
Extend the existing qtest scenario with the PIO data-path coverage.

Signed-off-by: flamboy <flamboyant.h.01@gmail.com>
~~~

### 中文

~~~text
hw/ssi: 实现 K230 SSI FIFO 与标准 PIO 传输

为全部四种 TMOD 模式实现基于 FIFO 的标准 SPI 传输。

实现 DR 别名、帧宽截断、回环、FIFO 容量、动态状态、接收反压，
以及控制器禁用语义。扩展现有 qtest 场景，覆盖 PIO 数据通路。

Signed-off-by: flamboy <flamboyant.h.01@gmail.com>
~~~

## 4. 0d6434f5e6 — hw/ssi: Add K230 SSI interrupt controller

### English

~~~text
hw/ssi: Add K230 SSI interrupt controller

Implement the SSI interrupt state and GPIO outputs.

TXE and RXF are derived from FIFO thresholds; RXU, TXO, and RXO are
latched causes with their documented read-clear behaviour.  The qtest
interrupt-routing scenario covers controller-local masking and clearing.

PLIC routing is intentionally deferred to the next commit.

Signed-off-by: flamboy <flamboyant.h.01@gmail.com>
~~~

### 中文

~~~text
hw/ssi: 新增 K230 SSI 中断控制器

实现 SSI 中断状态和 GPIO 输出。

TXE 与 RXF 由 FIFO 阈值推导得出；RXU、TXO 与 RXO 是锁存型中断原因，
并具有文档规定的读取清除行为。qtest 的中断路由场景覆盖控制器本地的
中断屏蔽与清除行为。

PLIC 路由有意留待下一次提交实现。

Signed-off-by: flamboy <flamboyant.h.01@gmail.com>
~~~

## 5. 450728e358 — hw/riscv: Route K230 SSI IRQs to the PLIC

### English

~~~text
hw/riscv: Route K230 SSI IRQs to the PLIC

Connect each of the nine GPIO outputs from every K230 SSI controller to
the corresponding PLIC source.

Use an explicit logical spi0/spi1/spi2 routing table instead of relying
on device-array order.  Extend the existing interrupt-routing qtest
scenario with PLIC pending-state and instance-isolation checks.

Signed-off-by: flamboy <flamboyant.h.01@gmail.com>
~~~

### 中文

~~~text
hw/riscv: 将 K230 SSI 中断路由至 PLIC

将每个 K230 SSI 控制器的 9 个 GPIO 输出连接到对应的 PLIC 中断源。

使用显式的逻辑 spi0/spi1/spi2 路由表，而非依赖设备数组顺序。
扩展现有的中断路由 qtest 场景，加入 PLIC pending 状态和实例隔离检查。

Signed-off-by: flamboy <flamboyant.h.01@gmail.com>
~~~

## 6. 1273bb17af — hw/ssi: Implement K230 enhanced QSPI transfers

### English

~~~text
hw/ssi: Implement K230 enhanced QSPI transfers

Add controller-side Dual and Quad SDR transfer phases for instruction,
address, mode, dummy, and data fields.

Reject unsupported Octal, DDR, RXDS, and invalid transfer configurations
without consuming FIFO data.  The qtest coverage exercises only these
controller-local rejection paths; SPI NOR success paths follow once the
machine attaches a flash device.

Signed-off-by: flamboy <flamboyant.h.01@gmail.com>
~~~

### 中文

~~~text
hw/ssi: 实现 K230 增强型 QSPI 传输

为指令、地址、模式、空周期和数据字段新增控制器侧的 Dual 与 Quad
SDR 传输阶段。

拒绝不支持的 Octal、DDR、RXDS 及非法传输配置，且不消耗 FIFO 数据。
qtest 仅覆盖这些控制器本地的拒绝路径；待机器挂接 Flash 设备后，
再覆盖 SPI NOR 成功传输路径。

Signed-off-by: flamboy <flamboyant.h.01@gmail.com>
~~~

## 7. 6d5430c3b3 — hw/riscv/k230: Attach SPI NOR flash to spi0

### English

~~~text
hw/riscv/k230: Attach SPI NOR flash to spi0

Add the optional spi-flash machine property and attach the selected m25p80
compatible device to logical spi0 chip select 0.

Accept an MTD backend when provided and retain the erased-flash default
when no drive is configured.  Complete the SPI NOR and QSPI SDR qtest
scenarios with JEDEC, read, program, erase, mode/dummy, and FIFO-resume
coverage.

Signed-off-by: flamboy <flamboyant.h.01@gmail.com>
~~~

### 中文

~~~text
hw/riscv/k230: 将 SPI NOR Flash 挂接到 spi0

新增可选的 spi-flash 机器属性，并将所选的 m25p80 兼容设备挂接到
逻辑 spi0 的片选 0。

提供 MTD 后端时接收并使用它；未配置驱动器时则保留擦除状态的 Flash
默认行为。补全 SPI NOR 与 QSPI SDR qtest 场景，覆盖 JEDEC、读取、
编程、擦除、模式/空周期以及 FIFO 恢复。

Signed-off-by: flamboy <flamboyant.h.01@gmail.com>
~~~

## 8. 14a0a12905 — hw/ssi: Implement K230 SSI internal DMA transfers

### English

~~~text
hw/ssi: Implement K230 SSI internal DMA transfers

Implement synchronous internal DMA for the verified 8-bit SDR Dual and
Quad transfer modes. Build enhanced commands from SPIDR and SPIAR, move
data through AXIAR0/1, and report completion or guest-memory failures
through the DONE and AXIE interrupt causes.

Keep DR accesses out of the FIFO while IDMA is enabled, implement the
read-clear status registers, terminate each transaction with SSI
disabled and chip select inactive, and migrate the completed-frame
count. Extend the existing K230 SSI qtest with Quad read/program,
interrupt, error, reset, and unsupported-mode coverage.

Signed-off-by: flamboy <flamboyant.h.01@gmail.com>
~~~

### 中文

~~~text
hw/ssi: 实现 K230 SSI 内部 DMA 传输

为已验证的 8 位 SDR Dual 与 Quad 传输模式实现同步内部 DMA。
根据 SPIDR 与 SPIAR 构建增强型命令，通过 AXIAR0/1 搬运数据，
并通过 DONE 和 AXIE 中断原因报告完成或客户机内存访问失败。

在 IDMA 启用期间禁止 DR 访问 FIFO，实现读取清除型状态寄存器；
每笔事务结束时均禁用 SSI 并使片选失效，同时迁移已完成帧计数。
扩展现有 K230 SSI qtest，覆盖 Quad 读取/编程、中断、错误、复位，
以及不支持模式。

Signed-off-by: flamboy <flamboyant.h.01@gmail.com>
~~~

## 9. 809022018c — hw/misc: Add K230 HI_SYS SSI control

### English

~~~text
hw/misc: Add K230 HI_SYS SSI control

Model the HI_SYS SSI_CTRL wrapper register, including its reset value,
write mask, and dynamic mode and sleep status for the three logical SSI
controllers. Reuse the machine's SSI route table to connect logical
controller numbering to the physical instances.

Keep the sleep indication synchronized when IDMA disables SSI after
completion or an AXI error.

Map the wrapper over the former unimplemented HI_SYS region, migrate its
writable state and each controller's sleep status, and extend the K230
SSI qtest with register, routing, sleep, IDMA, and DR2-alias coverage.

Signed-off-by: flamboy <flamboyant.h.01@gmail.com>
~~~

### 中文

~~~text
hw/misc: 新增 K230 HI_SYS SSI 控制

建模 HI_SYS SSI_CTRL 包装寄存器，包括其复位值、写掩码，以及
三个逻辑 SSI 控制器的动态模式和休眠状态。复用机器的 SSI 路由表，
将逻辑控制器编号关联到物理实例。

当 IDMA 在完成或 AXI 错误后禁用 SSI 时，保持休眠状态指示同步。

将该包装器映射到先前未实现的 HI_SYS 区域；迁移其可写状态以及每个
控制器的休眠状态；扩展 K230 SSI qtest，覆盖寄存器、路由、休眠、
IDMA 和 DR2 别名。

Signed-off-by: flamboy <flamboyant.h.01@gmail.com>
~~~

## 10. d35a73707c — hw/ssi: Add K230 SSI XIP read window

### English

~~~text
hw/ssi: Add K230 SSI XIP read window

Expose logical spi0's 128 MiB flash window as a second SSI MMIO region.
Gate accesses through HI_SYS XIP_EN and build Standard, Dual, or Quad
SDR read commands from the existing XIP instruction, address, mode, and
dummy-cycle registers.

Keep the window read-only, terminate every access with chip select
inactive, and discard stale PIO state before issuing an XIP command. Add
qtest coverage for the gate, access widths, write rejection, 24/32-bit
addressing, Quad mode/dummy fields, and PIO/XIP sharing.

Signed-off-by: flamboy <flamboyant.h.01@gmail.com>
~~~

### 中文

~~~text
hw/ssi: 新增 K230 SSI XIP 读取窗口

将逻辑 spi0 的 128 MiB Flash 窗口暴露为第二个 SSI MMIO 区域。
通过 HI_SYS 的 XIP_EN 控制访问开关，并基于现有的 XIP 指令、地址、
模式和空周期寄存器构建 Standard、Dual 或 Quad SDR 读取命令。

保持该窗口只读；每次访问结束时均使片选失效；在发出 XIP 命令前清除
残留的 PIO 状态。新增 qtest 覆盖 XIP 开关、访问宽度、写入拒绝、
24/32 位地址、Quad 模式/空周期字段，以及 PIO/XIP 共存。

Signed-off-by: flamboy <flamboyant.h.01@gmail.com>
~~~
