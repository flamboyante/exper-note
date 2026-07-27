# K230 SPI v3.3：11 个提交说明（中英文）

分支：`k230-spiv3.3`

基线：`f893c46c3931b3684d235d221bf8b7844ddbf1d7`（`upstream/master`）

提交数：11

本文按提交顺序整理 v3.3 的英文提交说明及中文对照。前 10 个补丁与
`k230-spiv3.2` 完全一致（提交哈希相同），第 11 个补丁为 v3.3 新增的
trace 事件补丁。本文档只记录现有 Git 提交信息，尚未对补丁本身做任何
修改。

## 1. b4144a08f2 — hw/ssi: Add K230 DesignWare SSI register model

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
hw/ssi：新增 K230 DesignWare SSI 寄存器模型

为 K230 DesignWare SSI 控制器新增 SysBus 设备模型。

依据 K230 技术参考手册实现寄存器布局、复位值、可写位掩码、FIFO
状态、片选 GPIO 和机器集成所需的 MMIO 区域。本补丁暂不实例化具体
控制器。

机器集成和 qtest 覆盖将在后续补丁中加入。

使用 riscv64-softmmu 配置完成编译验证。

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
~~~

## 2. b94be438e7 — hw/riscv/k230: Instantiate K230 SSI controllers

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
hw/riscv/k230：实例化 K230 SSI 控制器

实例化 K230 使用的三个 SSI 控制器配置，并将 MMIO 区域映射到文档
规定的地址。

新增 K230 SSI qtest，覆盖三个实例的配置差异、复位状态、片选掩码、
代表性寄存器掩码和系统复位行为。

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
~~~

## 3. 553615714c — hw/ssi: Implement K230 SSI FIFO and standard PIO transfers

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
hw/ssi：实现 K230 SSI FIFO 与标准 PIO 传输

为 TR、TO、RO 和 EEPROM Read 四种 TMOD 模式实现基于 FIFO 的标准
SPI 传输。

实现 DR 别名、帧宽截断、回环、FIFO 容量、动态状态、接收反压和
控制器禁用语义。

扩展 qtest，覆盖 8 位回环传输、只接收模式的 NDF 处理、FIFO 状态，
以及禁用控制器时的 FIFO 清理。

同时在 K230 U-Boot 和 Linux 环境中通过命令行验证标准 SPI 读写。

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
~~~

## 4. 9c8ba2d9bf — hw/ssi: Add K230 SSI interrupt controller

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
hw/ssi：新增 K230 SSI 中断控制器

实现 SSI 中断状态和 GPIO 输出。

TXE 与 RXF 根据 FIFO 阈值生成；RXU、TXO 和 RXO 保持为锁存型中断
原因，并实现手册规定的读取清除行为。

qtest 覆盖 TXE 电平中断、RXU 锁存、中断屏蔽和原因清除。PLIC 路由
由下一补丁实现。

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
~~~

## 5. a3deca2cca — hw/riscv: Route K230 SSI IRQs to the PLIC

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
hw/riscv：将 K230 SSI 中断路由至 PLIC

将每个 K230 SSI 控制器的 9 个 GPIO 输出连接到对应的 PLIC 中断源。

qtest 检查三个 SPI 实例的 TXE PLIC 路由，并验证 spi1 的 RXU 路由和
实例隔离。

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
~~~

## 6. d7d62d82cf — hw/ssi: Implement K230 enhanced QSPI transfers

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
hw/ssi：实现 K230 增强型 QSPI 传输

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

## 7. 48e05fedba — hw/riscv/k230: Attach SPI NOR flash to spi0

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
hw/riscv/k230：将 SPI NOR Flash 挂接到 spi0

新增可选的 spi-flash 机器属性，并将所选的 m25p80 兼容设备挂接到
逻辑 spi0 的片选 0。

配置 MTD 后端时使用该后端；未配置驱动器时保留擦除状态的默认 Flash。

qtest 覆盖 JEDEC 识别、标准读取与页编程、Quad Output 读取和 Quad
页编程。

同时验证由 U-Boot 管理的启动路径：从挂接的 SPI Flash 加载 Linux 和
OpenSBI 载荷，而不是通过 QEMU 命令行注入。系统进入 Linux shell。

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
~~~

## 8. 4c1e228f6e — hw/ssi: Implement K230 SSI internal DMA transfers

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
hw/ssi：实现 K230 SSI 内部 DMA 传输

为 8 位 SDR Dual 和 Quad 传输模式实现同步内部 DMA。从 SPIDR 和
SPIAR 构造增强命令，通过 AXIAR0/1 搬运数据，并通过 DONE 和 AXIE
中断原因上报完成或客户机内存访问失败。

IDMA 使能期间禁止 DR 访问 FIFO。实现读取清除状态寄存器，每次传输
结束时禁用 SSI 并释放片选，并对已完成帧计数进行迁移保存。

qtest 覆盖向客户机 RAM 的 Quad 读取、已完成帧上报、DONE 路由与
清除，以及非法客户机地址触发的 AXIE 路径。

在 U-Boot 和 Linux 中验证 QSPI 读写命令。在设备树中将 spi0 配置为
QSPI 后，从 QSPI Flash 启动 Linux 镜像。

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
~~~

## 9. 071a9e18ad — hw/misc: Add K230 HI_SYS SSI control

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
hw/misc：新增 K230 HI_SYS SSI 控制

建模 HI_SYS SSI_CTRL 包装寄存器，包括其复位值、可写掩码以及三个
逻辑 SSI 控制器的动态模式与睡眠状态。

K230 软件通过该寄存器控制 XIP 使能并观察 SSI 实例的模式与睡眠
状态，仅靠控制器本地寄存器无法提供完整的客户机可见接口。

复用机器的 SSI 路由表，将逻辑控制器编号关联到物理实例。IDMA 在
完成或发生 AXI 错误后禁用 SSI 时，同步更新睡眠指示。

将该包装寄存器映射到此前未实现的 HI_SYS 区域，并迁移其可写状态和
控制器睡眠状态。qtest 覆盖寄存器约定、三实例模式路由和睡眠状态
切换。

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
~~~

## 10. 1a2ef9249d — hw/ssi: Add K230 SSI XIP read window

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
hw/ssi：新增 K230 SSI XIP 读取窗口

将逻辑 spi0 的 128 MiB Flash 窗口作为第二个 SSI MMIO 区域暴露。
通过 HI_SYS XIP_EN 控制访问，并根据 XIP 指令、地址、模式和空
周期寄存器构造 Standard、Dual 或 Quad SDR 读命令。

PIO 和 IDMA 覆盖显式 SPI 事务，而 K230 固件也通过内存映射访问
读取 spi0 Flash。建模 XIP 窗口使该接口对客户机可见。

保持窗口只读，每次访问结束后释放片选，并在发起 XIP 命令前丢弃
陈旧的 PIO 状态。

qtest 覆盖 XIP 使能门控、写拒绝、24 位与 32 位寻址、Quad 模式与
空周期，以及 PIO/XIP 共享。

同时验证 U-Boot 能通过 XIP 读取路径启动 Linux。

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
~~~

## 11. 0b93b3d973 — hw/ssi: Add trace events for K230 DesignWare SSI（v3.3 新增）

### English

~~~text
hw/ssi: Add trace events for K230 DesignWare SSI

Add trace events for the K230 DesignWare SSI model covering:

- register accesses using the effective masked or returned value;
- transaction boundaries for standard and enhanced PIO transfers;
- interrupt output levels driven toward the PLIC;
- IDMA start, completion, and AXI error paths; and
- XIP reads with the final returned value.

The events avoid tracing individual FIFO data frames so that traces
remain usable during U-Boot and Linux SPI NOR transfers.

The events only observe model behaviour and do not change it.

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
~~~

### 中文

~~~text
hw/ssi：为 K230 DesignWare SSI 新增 trace 事件

为 K230 DesignWare SSI 模型新增 trace 事件，覆盖：

- 寄存器访问，使用经过掩码处理或最终返回的值；
- 标准与增强 PIO 传输的事务边界；
- 输出到 PLIC 的中断电平；
- IDMA 启动、完成和 AXI 错误路径；
- XIP 读取及其最终返回值。

事件不追踪单个 FIFO 数据帧，避免在 U-Boot 和 Linux SPI NOR 传输
期间产生过多日志，保持 trace 可用性。

这些事件仅观察模型行为，不改变其语义。

Signed-off-by: Kangjie Huang <flamboyant.h.01@gmail.com>
~~~

---

## 改动建议

下表汇总 v3.3 各 commit 当前状态及是否仍需修改：

| # | Commit                              | 是否需要改 | 说明 |
|---|-------------------------------------|------------|------|
| 1 | b4144a08f2 寄存器模型                | 是（建议） | 应同步加入 trace-events 文件骨架与寄存器读写 trace（见下文方案 A/B） |
| 2 | b94be438e7 实例化控制器              | 否         | 与 trace 无关，保持现状即可 |
| 3 | 553615714c FIFO 与标准 PIO           | 是（建议） | 应在该补丁中加入 transaction_start/end trace 调用 |
| 4 | 9c8ba2d9bf 中断控制器                | 是（建议） | 应在该补丁中加入 irq_update trace 调用 |
| 5 | a3deca2cca PLIC 路由                 | 否         | 仅连线，无新模型行为可 trace |
| 6 | d7d62d82cf 增强 QSPI                 | 否         | QSPI 配置拒绝路径可选 trace，但非必须 |
| 7 | 48e05fedba 挂接 SPI NOR              | 否         | 机器集成补丁，无新 trace 需求；但建议把正文 Linux shell 改为 Linux initramfs shell，与 cover letter 表述一致 |
| 8 | 4c1e228f6e 内部 DMA                  | 是（建议） | 应在该补丁中加入 idma_start/done/error trace 调用 |
| 9 | 071a9e18ad HI_SYS 控制               | 否         | 包装寄存器无独立 trace 需求 |
| 10| 1a2ef9249d XIP 读取窗口              | 是（建议） | 应在该补丁中加入 xip_read trace 调用 |
| 11| 0b93b3d973 trace 事件（v3.3 新增）   | 是（必须） | 当前作为系列末尾补丁存在，trace 调用与对应功能实现分离在多个补丁中，详见下文分析 |

### 核心问题：commit 11 的位置与拆分

`0b93b3d973` 是 v3.3 相对 v3.2 唯一的新增补丁，但它在 `k230_dw_ssi.c` 中
追溯式插入了 26 行 trace 调用，覆盖的却是 commit 1/3/4/8/10 中实现的
功能：

- 寄存器读写 trace → 对应 commit 1（寄存器模型）
- transaction_start/end trace → 对应 commit 3（FIFO/PIO）和 commit 6（增强 QSPI）
- irq_update trace → 对应 commit 4（中断控制器）
- idma_start/done/error trace → 对应 commit 8（IDMA）
- xip_read trace → 对应 commit 10（XIP 窗口）

对上游 review 而言，这种先实现、最后统一加 trace 的写法存在两个隐患：

1. bisect 不友好：在 commit 1～10 之间 bisect 时，trace 事件不可用，
   调试早期问题只能靠 printf。
2. 改动跨多个补丁：commit 11 一次性触碰了 6 个前序补丁引入的代码路径，
   review 时难以判断 trace 点位置是否合理。

#### 推荐方案（按优先级排序）

方案 A：把 trace 调用拆分回各功能补丁（推荐，最干净）

- 在 commit 1 中新增 `hw/ssi/trace-events` 文件骨架与
  `k230_dw_ssi_reg_read/write` 两条事件，并在 `k230_dw_ssi.c` 的
  读写回调中调用。
- commit 3 加入 `k230_dw_ssi_transaction_start/end`。
- commit 4 加入 `k230_dw_ssi_irq_update`。
- commit 8 加入 `k230_dw_ssi_idma_start/done/error`。
- commit 10 加入 `k230_dw_ssi_xip_read`。
- 删除独立的 commit 11，系列回到 10 个补丁，但每个功能补丁自带 trace。

方案 B：把 commit 11 整体前移到 commit 1 之后

- 保留 commit 11 作为一个独立补丁，但放在 commit 1 之后、commit 2 之前。
- 这样 trace 基础设施在所有功能补丁之前就位，后续补丁即使不调用 trace
  也至少能编译进 trace 框架。
- 缺点：commit 11 中的 idma/xip trace 调用此时引用的函数还不存在，
  需要把这些 trace 调用拆到对应功能补丁中。本质上等价于方案 A。

方案 C：保持现状，但在 commit 11 正文中说明理由

- 在 commit message 中补一句类似：
  Added after the model stabilized to avoid churning trace definitions during development.
- 这是最省事的方案，但 review 时仍可能被要求重排，建议至少在 cover letter
  中预先解释。

### 其他小问题

- commit 7 正文：The system reaches a Linux shell. 与 cover letter
  中 reached the Linux initramfs shell. 不一致，建议改为
  The system reaches a Linux initramfs shell.，与 cover letter 及
  commit 3/8/10 的表述对齐。
- cover letter（非 commit）：v3.2 的 cover letter 仍是 10 补丁版本，
  v3.3 需要单独更新——主题改为 [PATCH 00/11]、补丁列表加入 trace 事件、
  diffstat 加入 hw/ssi/trace-events 的 11 行与 k230_dw_ssi.c 的 26 行
  （合计 +37）、测试小节补一句说明 trace 在 U-Boot/Linux SPI NOR 传输
  期间可用。
