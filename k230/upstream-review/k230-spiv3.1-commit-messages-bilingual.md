# K230 SPI v3.1：10 个提交说明（中英文）

分支：`k230-spiv3.1`

本文按提交顺序整理 v3.1 的英文提交说明草案及中文对照。提交哈希来自
当前分支，正文已按精简后的 qtest 覆盖范围调整。这里只更新文档，尚未
修改 Git 提交信息。

## 1. 43dc3cb578 — hw/ssi: Add K230 DesignWare SSI register model

### English

~~~text
hw/ssi: Add K230 DesignWare SSI register model

Add a SysBus model for the K230 DesignWare SSI controllers.

Implement the documented register layout, reset values, writable masks,
FIFO state, chip-select GPIOs, and the MMIO regions needed by the K230
machine. The controller is not instantiated by this patch.

Machine integration and qtest coverage follow in later patches.

Build-test the model with the riscv64-softmmu target.

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
~~~

### 中文

~~~text
hw/ssi: 新增 K230 DesignWare SSI 寄存器模型

为 K230 DesignWare SSI 控制器新增 SysBus 设备模型。

依据 K230 技术参考手册实现寄存器布局、复位值、可写位掩码、FIFO
状态、片选 GPIO 和机器集成所需的 MMIO 区域。本补丁暂不实例化
具体控制器。

机器集成和 qtest 覆盖将在后续补丁中加入。

使用 riscv64-softmmu 配置完成编译验证。

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
~~~

## 2. 23275df2b3 — hw/riscv/k230: Instantiate K230 SSI controllers

### English

~~~text
hw/riscv/k230: Instantiate K230 SSI controllers

Instantiate the three SSI controller profiles used by K230 and map their
MMIO regions at the documented addresses.

Add the K230 SSI qtest target. Cover the three instance profiles, reset
state, chip-select masks, representative register masks, and system
reset behaviour.

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
~~~

### 中文

~~~text
hw/riscv/k230: 实例化 K230 SSI 控制器

实例化 K230 使用的 spi0、spi1 和 spi2 三个 SSI 控制器配置，并将
MMIO 区域映射到文档规定的地址。

新增 K230 SSI qtest，检查三个实例的配置差异、复位状态、片选掩码、
代表性寄存器掩码和系统复位行为。

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
~~~

## 3. 071b4d8277 — hw/ssi: Implement K230 SSI FIFO and standard PIO transfers

### English

~~~text
hw/ssi: Implement K230 SSI FIFO and standard PIO transfers

Implement FIFO-backed standard SPI transfers for all four TMOD modes.

Model DR aliases, frame-width truncation, loopback, FIFO capacity,
dynamic status, receive backpressure, and controller disable semantics.

Extend the qtest with an 8-bit loopback transfer, receive-only NDF
handling, FIFO status checks, and FIFO cleanup on controller disable.

Also exercise standard SPI reads and writes from U-Boot and Linux.

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
~~~

### 中文

~~~text
hw/ssi: 实现 K230 SSI FIFO 与标准 PIO 传输

为 TR、TO、RO 和 EEPROM Read 四种 TMOD 模式实现基于 FIFO 的标准
SPI 传输。

实现 DR 别名、帧宽截断、回环、FIFO 容量、动态状态、接收反压和
控制器禁用语义。

扩展 qtest，覆盖 8 位回环传输、只接收模式的 NDF 处理、FIFO 状态，
以及禁用控制器时的 FIFO 清理。

同时在 K230 U-Boot 和 Linux 环境中通过命令行验证标准 SPI 读写。

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
~~~

## 4. 3d80856ce0 — hw/ssi: Add K230 SSI interrupt controller

### English

~~~text
hw/ssi: Add K230 SSI interrupt controller

Implement the SSI interrupt state and GPIO outputs.

Derive TXE and RXF from the FIFO thresholds. Keep RXU, TXO, and RXO as
latched causes with their documented read-clear behaviour.

Cover TXE level signalling, RXU latching, interrupt masking, and cause
clearing in qtest. PLIC routing is added by the next patch.

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
~~~

### 中文

~~~text
hw/ssi: 新增 K230 SSI 中断控制器

实现 SSI 中断状态和 GPIO 输出。

TXE 与 RXF 根据 FIFO 阈值生成；RXU、TXO 和 RXO 保持为锁存型中断
原因，并实现手册规定的读取清除行为。

qtest 覆盖 TXE 电平中断、RXU 锁存、中断屏蔽和原因清除。PLIC 路由
由下一补丁实现。

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
~~~

## 5. 941f7ef6c1 — hw/riscv: Route K230 SSI IRQs to the PLIC

### English

~~~text
hw/riscv: Route K230 SSI IRQs to the PLIC

Connect the nine GPIO outputs from each K230 SSI controller to their PLIC
sources.

Check the TXE PLIC route for all three instances and verify RXU routing
and instance isolation for spi1 in qtest.

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
~~~

### 中文

~~~text
hw/riscv: 将 K230 SSI 中断路由至 PLIC

将每个 K230 SSI 控制器的 9 个 GPIO 输出连接到对应的 PLIC 中断源。

qtest 检查三个 SPI 实例的 TXE PLIC 路由，并验证 spi1 的 RXU 路由和
实例隔离。

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
~~~

## 6. bcb797b163 — hw/ssi: Implement K230 enhanced QSPI transfers

### English

~~~text
hw/ssi: Implement K230 enhanced QSPI transfers

Add controller-side Dual and Quad SDR phases for instruction, address,
mode, dummy, and data fields.

Reject unsupported Octal, DDR, RXDS, and invalid transfer configurations
without consuming FIFO data.

Use qtest to check an accepted Quad SDR configuration and representative
Octal and DDR rejection paths.

According to the K230 SDK driver, multi-line QSPI data transfers use the
controller's internal IDMA path. That SDK-facing path is implemented by
a later patch. Flash-backed PIO coverage follows once the machine
attaches a SPI NOR device.

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
~~~

### 中文

~~~text
hw/ssi: 实现 K230 增强型 QSPI 传输

为指令、地址、模式、空周期和数据字段实现控制器侧的 Dual 与 Quad
SDR 传输阶段。

拒绝不支持的 Octal、DDR、RXDS 和非法传输配置，且不消耗 FIFO 数据。

qtest 检查一组可接受的 Quad SDR 配置，以及代表性的 Octal、DDR
拒绝路径。

根据 K230 SDK 驱动的实现，多线 QSPI 数据传输使用控制器内部 IDMA
路径。面向 SDK 的这条访问路径将在后续补丁中实现；机器挂接 SPI NOR
设备后，下一补丁先补充基于 PIO 的 Flash qtest 覆盖。

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
~~~

## 7. 7df643f426 — hw/riscv/k230: Attach SPI NOR flash to spi0

### English

~~~text
hw/riscv/k230: Attach SPI NOR flash to spi0

Add the optional spi-flash machine property and attach the selected
m25p80-compatible device to logical spi0 chip select 0.

Use an MTD backend when supplied and keep the erased-flash default when
no drive is configured.

Cover JEDEC identification, standard read and page program, Quad output
read, and Quad page program in qtest.

Also verify the U-Boot-managed boot path by loading the Linux and OpenSBI
payloads from the attached SPI flash instead of injecting them via the
QEMU command line. The system reaches a Linux shell.

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
~~~

### 中文

~~~text
hw/riscv/k230: 将 SPI NOR Flash 挂接到 spi0

新增可选的 spi-flash 机器属性，并将所选的 m25p80 兼容设备挂接到
逻辑 spi0 的片选 0。

配置 MTD 后端时使用该后端；未配置驱动器时保留擦除状态的默认 Flash。

qtest 覆盖 JEDEC 识别、标准读取与页编程、Quad Output 读取和 Quad
页编程。

同时验证由 U-Boot 管理的启动路径：从挂接的 SPI Flash 加载 Linux 和
OpenSBI 载荷，而不是通过 QEMU 命令行注入。系统能够进入 Linux shell。

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
~~~

## 8. 100409c5db — hw/ssi: Implement K230 SSI internal DMA transfers

### English

~~~text
hw/ssi: Implement K230 SSI internal DMA transfers

Implement synchronous internal DMA for 8-bit SDR Dual and Quad transfer
modes. Build enhanced commands from SPIDR and SPIAR, move data through
AXIAR0/1, and report completion or guest-memory failures through the DONE
and AXIE interrupt causes.

Keep DR accesses out of the FIFO while IDMA is enabled. Implement the
read-clear status registers, terminate each transaction with SSI
disabled and chip select inactive, and migrate the completed-frame
count.

Cover a Quad read into guest RAM, completed-frame reporting, DONE
routing and clearing, and the AXIE path for an invalid guest address.

Exercise QSPI read and write commands from U-Boot and Linux. With spi0
configured for QSPI in the device tree, boot a Linux image from QSPI
flash.

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
~~~

### 中文

~~~text
hw/ssi: 实现 K230 SSI 内部 DMA 传输

为 8 位 SDR Dual 与 Quad 模式实现同步内部 DMA。根据 SPIDR 和
SPIAR 构建增强型命令，通过 AXIAR0/1 搬运数据，并使用 DONE 和 AXIE
中断原因报告完成状态或客户机内存访问失败。

IDMA 启用时不再通过 DR 访问 FIFO。实现读取清除型状态寄存器；每笔
事务结束时禁用 SSI 并释放片选；同时迁移已完成帧计数。

qtest 覆盖 Quad 读取到客户机内存、已完成帧计数、DONE 路由与清除，
以及非法客户机地址触发的 AXIE 路径。

在 U-Boot 和 Linux 命令行中验证 QSPI 读写。将设备树中的 spi0 配置
为 QSPI 后，从 QSPI Flash 启动 Linux 镜像。

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
~~~

## 9. 6728bb3346 — hw/misc: Add K230 HI_SYS SSI control

### English

~~~text
hw/misc: Add K230 HI_SYS SSI control

Model the HI_SYS SSI_CTRL wrapper register, including its reset value,
write mask, and dynamic mode and sleep status for the three logical SSI
controllers.

K230 software uses this register to control XIP enable and observe the
mode and sleep state of the SSI instances, so controller-local registers
alone do not provide the complete guest-visible interface.

Reuse the machine SSI routing table to associate logical controller
numbers with physical instances. Keep the sleep indication synchronized
when IDMA disables SSI after completion or an AXI error.

Map the wrapper over the previously unimplemented HI_SYS region and
migrate its writable state and the controller sleep state. Cover the
register contract, three-instance mode routing, and sleep transitions in
qtest.

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
~~~

### 中文

~~~text
hw/misc: 新增 K230 HI_SYS SSI 控制

建模 HI_SYS SSI_CTRL 包装寄存器，包括复位值、写掩码，以及三个逻辑
SSI 控制器的动态模式和休眠状态。

K230 软件通过该寄存器控制 XIP 开关，并读取 SSI 实例的模式和休眠
状态，因此仅建模控制器自身寄存器还不能提供完整的客户机可见接口。

复用机器的 SSI 路由关系，将逻辑控制器编号对应到物理实例。当 IDMA
在完成或 AXI 错误后禁用 SSI 时，同步更新休眠状态指示。

将包装寄存器映射到原先未实现的 HI_SYS 区域，并迁移其可写状态和
控制器休眠状态。

qtest 覆盖寄存器行为、三个实例的模式路由和休眠状态切换。

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
~~~

## 10. 99c3ebb428 — hw/ssi: Add K230 SSI XIP read window

### English

~~~text
hw/ssi: Add K230 SSI XIP read window

Expose logical spi0's 128 MiB flash window as a second SSI MMIO region.
Gate accesses through HI_SYS XIP_EN and build Standard, Dual, or Quad
SDR read commands from the XIP instruction, address, mode, and
dummy-cycle registers.

PIO and IDMA cover explicit SPI transactions, while K230 firmware also
uses memory-mapped accesses to read spi0 flash. Model the XIP aperture so
this interface is visible to the guest.

Keep the window read-only, end each access with chip select inactive,
and discard stale PIO state before issuing an XIP command.

Cover the XIP gate, write rejection, 24-bit and 32-bit addressing, Quad
mode and dummy cycles, and PIO/XIP sharing in qtest.

Also verify that U-Boot can boot Linux through the XIP read path.

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
~~~

### 中文

~~~text
hw/ssi: 新增 K230 SSI XIP 读取窗口

为逻辑 spi0 实现只读 XIP Flash 窗口，使客户机能够像访问内存一样
直接读取 Flash 内容。通过 HI_SYS XIP_EN 控制窗口开关，并根据 XIP
指令、地址、模式和空周期寄存器构建 Standard、Dual 或 Quad SDR
读取命令。

PIO 和 IDMA 用于显式发起 SPI 事务，而 K230 固件还会通过内存映射
方式读取 spi0 Flash。建模 XIP 窗口，使这条接口对客户机可见。

窗口写入不生效；每次访问结束后释放片选；发出 XIP 命令前丢弃残留
的 PIO 状态。

qtest 覆盖 XIP 开关、写入拒绝、24 位和 32 位地址、Quad 模式与空周期，
以及 PIO/XIP 共用 Flash 的场景。

同时验证 U-Boot 能够通过 XIP 读取路径启动 Linux。

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
~~~
