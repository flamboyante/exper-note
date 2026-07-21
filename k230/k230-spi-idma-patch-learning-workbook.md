# K230 SSI IDMA Patch 学习工作簿

更新时间：2026-07-21

当前分支：`k230-ssi-idma`

当前提交：`742e76e1da hw/ssi: Implement K230 SSI internal DMA transfers`

本文用于把三层资料放在同一条学习路径上：

```text
K230 TRM 的硬件契约
        |
        v
RT-Smart / U-Boot SDK 的真实寄存器编程
        |
        v
QEMU k230_dw_ssi.c 的功能模型
```

阅读顺序固定为：先看数据流，再看寄存器，再看 SDK，最后回到 QEMU 的函数和 qtest。

## 先记住这几个结论

### 1. IDMA 不是外接的通用 DMA 控制器

K230 SSI 自己带 AXI master 接口。IDMA 开启后，SSI 控制器可以直接在 SPI 外设和
片上内存之间搬运数据，CPU 不需要逐帧读写 `DR`。

```text
CPU：配置寄存器、启动 SSI、等待中断
SSI：生成 SPI 指令/地址/时钟，并通过 AXI master 搬运数据
Flash：响应 SPI 指令和地址
内存：通过 AXIAR0/AXIAR1 指定
```

因此，IDMA 不是“给现有 PIO 路径增加一个 memcpy”。真实硬件中，SPI 数据路径、
FIFO 和 AXI master 是同时工作的；当前 QEMU 只是用临时 buffer 把这个关系压缩成
一次同步事务。

### 2. `TMOD` 决定内存方向

```text
TMOD = TO（SPI write）：
    内存 AXIAR0/1 --AXI 读取--> SSI --SPI 发送--> Flash

TMOD = RO（SPI read）：
    Flash --SPI 接收--> SSI --AXI 写入--> 内存 AXIAR0/1
```

这里的“write/read”是以 SPI 设备为观察对象：

- `TO` 是向 SPI 设备写数据；内存是源。
- `RO` 是从 SPI 设备读数据；内存是目的地。

### 3. 三组地址不要混淆

```text
SPIDR       = SPI 指令，例如 0x6b、0xeb、0x32
SPIAR       = Flash 芯片内部地址，例如 0x001234
AXIAR0/1    = CPU/Guest 内存地址，例如 0x80201000
```

`SPIDR` 和 `SPIAR` 描述 SPI 事务头；`AXIAR0/1` 描述 DMA 搬运的内存端点。

### 4. 当前 QEMU 的 IDMA 触发点

当前模型没有独立的异步 DMA engine。每次写入以下寄存器后，都会尝试调用
`k230_dw_ssi_try_idma()`：

```text
DMACR
SPIDR
SPIAR
AXIAR0 / AXIAR1
SSIENR
SER
```

只有所有前置条件满足时才真正开始事务。通常最后写 `SSIENR` 或 `SER` 会触发传输，
但 qtest 特意验证了寄存器写入顺序可以不同。

## 1. TRM 对 IDMA 的硬件定义

来源：K230 TRM 12.3.3.4，位于转换文本约 44655 行附近。

TRM 给出的硬件边界是：

- IDMA 只支持 Dual/Quad SPI 模式。
- IDMA 开启后，对 `DR` 的读写会返回 AHB error response。
- SPI 侧地址长度最大为 32 bit。
- SPI write 由 AXI 读取内存，再发送到 SPI 设备。
- SPI read 由 SPI 接收数据，再通过 AXI 写入内存。
- AXI 访问错误会立即撤销片选、关闭 SSI、清空 FIFO，并产生 AXI error 中断。
- 完成后产生 transfer done 中断。

TRM 描述的是一个真实的流式过程：

```text
内存 <-> AXI master <-> SSI FIFO <-> SPI shift engine <-> Flash
```

不能把它直接等价成：

```text
dma_memory_read/write() + for 循环
```

后者只是 QEMU 当前选择的同步功能级实现。

## 2. 关键寄存器：按“事务头”和“内存搬运”分组

### 2.1 SPI 事务头

| 寄存器/字段 | 作用 | 当前 QEMU 用法 |
|---|---|---|
| `CTRLR0.TMOD` | 选择 `TO`、`RO` 等传输方向 | 决定从内存读还是向内存写 |
| `CTRLR0.SPI_FRF` | Standard、Dual、Quad、Octal 线宽格式 | 只接受当前 profile 支持的 Dual/Quad |
| `SPI_CTRLR0.TRANS_TYPE` | 指令、地址、数据分别使用几线 | 用于解析增强 SPI 命令和 dummy 长度 |
| `SPI_CTRLR0.INST_L` | 指令长度编码：0/4/8/16 bit | 解析 `SPIDR` 的有效位数 |
| `SPI_CTRLR0.ADDR_L` | 地址长度，按 4 bit 编码 | 解析 `SPIAR` 的有效位数 |
| `SPI_CTRLR0.WAIT_CYCLES` | dummy 等待周期 | 转换为 dummy byte 数 |
| `CTRLR1.NDF` | 数据帧数量减一 | 实际帧数为 `NDF + 1` |
| `SPIDR.SPI_INST` | SPI instruction code | 例如 `0x6b`、`0xeb`、`0x32` |
| `SPIAR.SDAR` | SPI device address | Flash 内部地址 |

注意：`SPIDR` 和 `SPIAR` 是 IDMA 专用的事务头寄存器，不是普通 TX FIFO 的替代品。

### 2.2 内存和 AXI 属性

| 寄存器/字段 | TRM 意图 | 当前 QEMU 状态 |
|---|---|---|
| `DMACR.IDMAE` bit 2 | 开启内部 DMA | 作为启动条件 |
| `DMACR.ATW` bits 4:3 | AXI transfer width | 保留并可读回，尚未参与搬运 |
| `DMACR.AINC` bit 6 | AXI burst/address 行为 | SDK 和当前模型都要求写 1；语义存在 TRM/SDK 差异 |
| `DMACR.ACACHE` | AXI cache 属性 | 保存，未模拟 cache 行为 |
| `DMACR.APROT` | AXI protection 属性 | 保存，未参与访问检查 |
| `DMACR.AID` | AXI ID | 保存，未建立 AXI transaction |
| `AXIAWLEN` `AWLEN` | 读操作的 destination burst length | 保存，当前 IDMA 不按 burst 搬运 |
| `AXIARLEN` `ARLEN` | 写操作的 source burst length | 保存，当前 IDMA 不按 burst 搬运 |
| `AXIAR0` | 内存地址低 32 bit | 参与计算 Guest 地址 |
| `AXIAR1` | 内存地址高位 | 与 `AXIAR0` 拼成 64 bit 地址 |

TRM 的 AXI 通道名称容易造成方向混淆。学习时先以数据方向为准：

```text
TMOD=TO：AXI 读取内存，使用 source burst 相关配置
TMOD=RO：AXI 写入内存，使用 destination burst 相关配置
```

### 2.3 完成和错误

| 寄存器 | 作用 | 当前 QEMU 状态 |
|---|---|---|
| `RISR.DONE` | 原始 transfer done 状态 | IDMA 成功后锁存 |
| `ISR.DONE` | 经过 `IMR` 屏蔽的 DONE | 可读、可接 PLIC |
| `DONECR` | 清除 DONE，中断状态为 RC | 读清已实现 |
| `RISR.AXIE` | AXI master error | 当前没有产生真实 AXI error |
| `AXIECR` | 清除 AXI error | 当前按未实现路径处理 |
| `SR.CMPLTD_DF` | 最近一次 DMA 成功传输量 | 当前没有完整实现 |

## 3. SDK 是怎样使用 IDMA 的

### 3.1 RT-Smart 驱动：`drv_spi.c`

文件：

```text
/home/flamboy/k230_sdk/src/big/rt-smart/kernel/bsp/maix3/board/interdrv/spi/drv_spi.c
```

关键路径约在 300--419 行。

先看它的约束检查：

- 指令线宽必须是单线或等于数据线宽。
- 地址长度必须是 4 bit 的倍数，最大 32 bit。
- 指令和地址至少存在一个。
- 数据长度不能超过 `0x10000`。
- 根据指令、地址和数据线宽计算 `TRANS_TYPE`。
- 数据方向决定 `TMOD`：有接收 buffer 时为 `RO`，否则为 `TO`。

然后进入 IDMA 配置：

```c
spi->dmacr = (1 << 6) | (3 << 3) | (1 << 2);
spi->ctrlr1 = length - 1;
spi->spidr = msg->instruction.content;
spi->spiar = msg->address.content;
spi->axiar0 = (uint32_t)((uint64_t)buf);
spi->axiar1 = (uint32_t)((uint64_t)buf >> 32);
```

这几行可以翻译成：

```text
DMACR：AINC=1、ATW=3、IDMAE=1
CTRLR1：数据帧数 - 1
SPIDR：SPI 指令
SPIAR：Flash 地址
AXIAR0/1：临时 DMA buffer 地址
```

RT-Smart 对 cache 的处理也很重要：

- `TO` 之前把发送数据复制到 DMA buffer，并执行 cache clean。
- `RO` 完成后执行 cache invalidate，再把 DMA buffer 复制到调用者的接收 buffer。
- 这说明真实 DMA 访问的是内存，而不是 CPU 私有 cache 中尚未回写的数据。

启动和等待顺序是：

```c
spi->ser = 1 << cfg->parent.cs;
spi->ssienr = 1;
rt_event_recv(... BIT(SSI_DONE) | BIT(SSI_AXIE) ...);
spi->ser = 0;
spi->ssienr = 0;
```

SDK 通过 DONE/AXIE 判断结果，而不是检查 TX/RX FIFO 是否由 CPU 消费完。

### 3.2 U-Boot：`designware_spi.c`

文件：

```text
/home/flamboy/k230_sdk/src/little/uboot/drivers/spi/designware_spi.c
```

关键路径是 `dw_spi_exec_op()`，约在 896--1113 行。

U-Boot 先根据 SPI memory operation 的线宽决定是否使用 IDMA：

```c
if ((op->data.buswidth == 1) ||
    (op->addr.buswidth == 0) ||
    (op->data.buswidth == 0))
    priv->use_idma = 0;
else
    priv->use_idma = 1;
```

也就是说，Standard 单线数据路径通常走 PIO；Dual/Quad 数据路径才走 IDMA。

U-Boot 的 IDMA 配置重点是：

- `SPIDR` 写 opcode。
- `SPIAR` 写 Flash 地址。
- `DMACR.IDMAE = 1`。
- `DMACR.AINC = 1`。
- 根据 buffer 对齐计算 `ATW`。
- 根据 SRAM/DDR 地址区间选择 AXI burst length。
- 读操作写 `AXIAWLEN`，写操作写 `AXIARLEN`。
- 刷新/失效 cache。
- 写 `AXIAR0`。
- `SSIENR = 1` 后轮询 `RISR.DONE`。
- 读取 `DONECR`，再写 `DMACR = 0`。

这里可以看到 SDK 和当前 QEMU 的共同点：都把 `AINC` 写成 1。这正是不能只按
TRM 单句描述去背寄存器含义的原因。

### 3.3 主线 QEMU 中 SPI DMA 的实现方式

主线 QEMU 没有一个所有 SPI 控制器共用的“SPI DMA 框架”。通常按硬件结构选择
下面两种建模方式。

#### 方式一：SPI 和独立 DMA 设备用 QOM/Stream 连接

参考：

```text
hw/ssi/xlnx-versal-ospi.c
hw/dma/xlnx_csu_dma.c
```

Versal OSPI 在初始化时添加 `dma-src` link，把 OSPI 和独立的 CSU DMA 设备连接起来：

```text
OSPI Flash/SRAM FIFO <-> Stream interface <-> CSU DMA <-> Guest memory
```

OSPI 侧只负责：

- 产生 SPI Flash 数据；
- 管理自己的 SRAM/FIFO；
- 根据 burst/single request 配置请求 DMA；
- 处理 indirect operation 完成和水位中断。

CSU DMA 侧负责：

- 保存 DMA 地址和剩余长度；
- 从 Guest memory 分块读取数据；
- 通过 `stream_push()` 推给 OSPI；
- 根据下游 `stream_can_push()` 处理 backpressure；
- 更新地址、剩余长度和 DONE；
- 产生 AXI response error。

CSU DMA 的核心循环约在 `xlnx_csu_dma_src_notify()`：

```text
while (还有数据 && 下游可以接收) {
    从 Guest memory 读取一块，最大 4 KiB
    stream_push() 交给 OSPI
    根据实际接收长度更新 DMA 地址和剩余长度
}
```

如果下游暂时不能接收，模型不强行把整块数据塞进去，而是记录 backpressure，
必要时启动 timeout timer，等下游重新通知。这就是比较完整的 QEMU DMA 语义：
不是只完成最终 memcpy，而是保留 DMA 和 FIFO 之间的流动关系。

#### 方式二：SPI 控制器内部 DMA，设备自己访问 AddressSpace

参考：

```text
hw/ssi/aspeed_smc.c
```

Aspeed SMC 的 DMA 是控制器内部功能，因此没有单独实例化通用 DMA 设备。SMC 自己
保存 Flash 地址、DRAM 地址和长度，然后使用两个 AddressSpace：

```text
flash_as
dram_as
```

传输时直接从一个 AddressSpace 读取，再写入另一个 AddressSpace，并且在传输过程中
更新 Guest 可见寄存器：

```text
DMA_LEN          递减
DMA_FLASH_ADDR   递增
DMA_DRAM_ADDR    递增
DMA_STATUS       运行中/完成
DMA_CHECKSUM     更新
```

这种模型仍然可以在一次寄存器写入中完成循环，但它比当前 K230 更完整，因为它
把“正在进行时寄存器是什么样”也建模出来了。

#### QEMU 的通用内存 DMA API

QEMU 设备通常通过 `system/dma.h` 中的 API 访问 Guest memory：

```c
dma_memory_read(&address_space_memory, addr, buf, len, attrs);
dma_memory_write(&address_space_memory, addr, buf, len, attrs);
```

这两个函数不是普通 Host `memcpy`，它们会经过 QEMU 的 `AddressSpace`，并返回
`MemTxResult`。因此可以表示：

- 未映射地址；
- IOMMU fault；
- 设备拒绝访问；
- DMA 读写方向；
- 基本的 DMA 访问顺序。

QEMU 是否再加 Stream、FIFO、BH 或 Timer，要看 Guest 是否能观察到这些中间行为。

#### 和当前 K230 实现对照

| 主线 QEMU 做法 | 当前 K230 IDMA |
|---|---|
| 独立 DMA 设备通过 QOM/Stream 连接 | 没有独立 DMA 对象，SSI 自己完成搬运 |
| DMA 维护分块、剩余长度、backpressure | 一次性分配 buffer 并完成整段事务 |
| `AddressSpace` 访问 Guest memory | 使用 `dma_memory_read/write()` |
| 可更新 DMA 地址/长度寄存器 | `AXIAR0/1` 只作为起始地址读取 |
| 可模拟 FIFO/Stream 流动 | 当前直接完成 SPI 循环 |
| 可产生 DMA/AXI 错误状态 | 目前主要处理内存访问失败，不完整模拟 AXIE |
| 可用 BH/Timer 延后继续 | 当前没有 IDMA 异步 continuation |

因此，当前 K230 不是“不像 QEMU”，而是选择了主线 QEMU 中也存在的功能级简化
路线；它接近“内部 DMA 直接访问 AddressSpace”的方式，但还没有达到 Aspeed SMC
或 Versal CSU DMA 那种分块、进度和 backpressure 级别。

判断是否需要继续细化，可以看 Guest 是否依赖：

```text
1. 传输期间读取 BUSY/FIFO/剩余长度；
2. DMA 地址或长度寄存器的动态变化；
3. FIFO 满/空造成的暂停或 underflow；
4. AXI error 后的部分完成量；
5. DMA 与 SPI 之间的异步中断顺序。
```

如果当前目标只是让 SDK 的 IDMA read/program 数据正确，`dma_memory_*()` 的同步
功能模型足够；如果目标是复现完整硬件状态机，就需要引入 `idma_in_progress`、
剩余长度、分块搬运和 BH/Timer 或 Stream continuation。

#### 3.3.1 `xilinx_axidma.c`：独立 DMA engine

参考：

```text
/home/flamboy/qemu-camp-2026/qemu-camp-2026-k230/hw/dma/xilinx_axidma.c
```

`xilinx_axidma.c` 是真正的独立 AXI DMA 控制器模型，不是某个 SPI 控制器内部的
DMA 快捷路径。

它的描述符位于 Guest memory：

```c
struct SDesc {
    uint64_t nxtdesc;
    uint64_t buffer_address;
    uint32_t control;
    uint32_t status;
};
```

启动后，DMA engine 根据 `CURDESC` 和 `TAILDESC` 遍历描述符：

```text
读取 CURDESC 指向的 descriptor
        |
        v
读取 buffer_address 和 length
        |
        v
Guest memory <-> Stream
        |
        v
回写 descriptor.status
        |
        v
跳到 nxtdesc
```

它建模了独立 DMA engine 的典型状态：

- 当前描述符和尾描述符；
- Memory-to-Stream / Stream-to-Memory 双向路径；
- descriptor 链表；
- 分块读写 Guest memory；
- descriptor 完成状态回写；
- `DECERR/SLVERR` 错误；
- `HALTED/IDLE/DONE/IRQ` 状态；
- completion delay timer。

关键代码：

```text
stream_desc_load()       约 197 行
stream_process_mem2s()   约 290 行
stream_process_s2mem()   约 350 行
axidma_write()           约 497 行
```

它和 K230 IDMA 的结构区别是：

```text
Xilinx AXI DMA：
    Guest memory
        <-> DMA descriptor engine
        <-> Stream peripheral

K230 IDMA：
    Guest memory
        <-> SSI 内部 AXI master
        <-> SSI FIFO/Flash
```

K230 的 `SPIDR/SPIAR/AXIAR0/1/CTRLR1` 更像“一次事务描述寄存器”，不是
Xilinx AXI DMA 那种位于 Guest memory 中、可链接的 descriptor。

#### 3.3.2 `npcm7xx_fiu.c`：不是 DMA，而是直接 Flash window

参考：

```text
/home/flamboy/qemu-camp-2026/qemu-camp-2026-k230/hw/ssi/npcm7xx_fiu.c
```

这个文件没有实现 DMA engine。它提供的是一个 Flash 直接访问
`MemoryRegion`，并实现 UMA 命令访问。

初始化时，每个片选对应一个直接访问窗口：

```c
memory_region_init_io(&flash->direct_access, ...);
```

Guest 读取 Flash window 时，QEMU 会同步执行：

```text
Guest load
    |
    v
npcm7xx_fiu_flash_read()
    +-- 选择 CS
    +-- 发送读 opcode
    +-- 发送地址
    +-- 发送 dummy
    +-- ssi_transfer() 读取数据
    +-- 释放 CS
    v
返回读取值
```

Guest 写入窗口时则执行对应的 SPI 写命令和数据发送。UMA 路径也是同样的同步模型：
Guest 写 `EXEC_DONE` 后，`npcm7xx_fiu_uma_transaction()` 立即发送命令、地址、
dummy 和数据，然后更新 `RDYST`。

因此 NPCM FIU 的结构是：

```text
Guest load/store
    -> QEMU MemoryRegion callback
    -> 拼 SPI 命令
    -> ssi_transfer()
    -> 返回结果
```

不是：

```text
Guest 配置 DMA
    -> 控制器后台搬运
    -> DMA 完成中断
```

NPCM FIU 适合学习 QEMU 如何建模 Flash memory window 和同步 SPI 事务，不适合作为
K230 IDMA 的 DMA engine 架构模板。

#### 3.3.3 三个例子的定位

| 文件 | 实际模型 | 对 K230 的学习价值 |
|---|---|---|
| `hw/dma/xilinx_axidma.c` | 独立 DMA engine + descriptor + Stream | 学 DMA 状态机、描述符、错误和完成语义 |
| `hw/ssi/npcm7xx_fiu.c` | SPI Flash window + UMA 同步访问 | 学 MemoryRegion 到 SPI 事务的映射 |
| `hw/ssi/k230_dw_ssi.c` | SSI 内部 IDMA 的简化模型 | 学如何把内部 AXI 搬运接入 SSI 事务 |

因此，K230 不应该直接复制 `xilinx_axidma.c` 的 descriptor/Stream 结构，也不应把
`npcm7xx_fiu.c` 的 MemoryRegion callback 误认为 DMA。更合理的演进路线是：保留
K230 的内部 DMA寄存器接口，在需要时逐步增加 `idma_in_progress`、剩余长度、分块
搬运、FIFO backpressure 和 AXIE 错误状态。

## 4. 当前 QEMU 实现逐段阅读

文件：

```text
/home/flamboy/qemu-camp-2026/qemu-camp-2026-k230/hw/ssi/k230_dw_ssi.c
```

### 4.1 先把命令解析抽出来

`k230_dw_ssi_parse_enhanced_command()` 约在 623--679 行。

这次 IDMA patch 把原来只服务 PIO 的增强命令解析拆成共享 helper。它从寄存器中
解码：

```text
INST_L       -> instruction_bits
ADDR_L       -> address_bits
XIP_MBL      -> mode_bits
WAIT_CYCLES  -> wait_cycles
CTRLR1.NDF   -> data_frames = NDF + 1
TRANS_TYPE   -> trans_type
TMOD         -> tmod
```

这样 PIO、IDMA 和后续 XIP 使用同一种 `K230DwSsiEnhancedCommand`，避免三套路径
分别解释寄存器，符合 DRY 原则。

### 4.2 IDMA 何时算 ready

`k230_dw_ssi_idma_ready()` 约在 735--745 行。它要求：

```text
DMACR.IDMAE = 1
DMACR.AINC  = 1
SSIENR 已启用
SER 非零
SPIDR 已写
SPIAR 已写
AXIAR0 已写
上一次 IDMA 没有完成
phase = IDLE
TX/RX FIFO 都为空
```

`idma_spidr_written`、`idma_spiar_written`、`idma_axiar0_written` 不是硬件寄存器，
而是 QEMU 用来判断 Guest 是否已经完成编程的辅助状态。

当前没有 `idma_axiar1_written`，因为低 32 bit 地址是启动 IDMA 的必要条件，高 32 bit
可以保持为 0。若要严格建模 64 bit 地址写入顺序，这里是后续需要补充的状态点。

### 4.3 `k230_dw_ssi_try_idma()` 的真实执行顺序

这个函数约在 761--858 行。

#### 第一步：检查启动条件和片选

```c
if (!k230_dw_ssi_idma_ready(s))
    return;

k230_dw_ssi_update_cs(s);
```

多个 `SER` 位同时置位时，当前模型拒绝广播片选；没有有效片选也不会启动。

#### 第二步：验证增强配置并生成命令描述

```c
k230_dw_ssi_parse_enhanced_command(s, s->regs[R_SPIDR],
                                   s->regs[R_SPIAR], &command)
```

当前 profile 只接受 Dual/Quad SDR 的 `RO`/`TO`，拒绝 Octal、DDR、RXDS 和其他未
实现的增强模式。

#### 第三步：计算内存范围并准备临时 buffer

```c
length = command.data_frames;
address = AXIAR0 | ((uint64_t)AXIAR1 << 32);
buffer = g_malloc(length);
```

当前模型把每个数据帧按 1 byte 处理，因此这里实际上是“字节型 IDMA”。`ATW`、
`DFS` 与多字节 frame 还没有参与 buffer 长度和搬运宽度计算。

#### 第四步：`TO` 先从 Guest 内存取数据

```c
result = dma_memory_read(&address_space_memory, address, buffer,
                         length, MEMTXATTRS_UNSPECIFIED);
```

这是：

```text
Guest memory -> QEMU temporary buffer
```

读取失败时不发送 SPI 数据，不产生 DONE。

#### 第五步：发送 SPI 指令、地址、mode 和 dummy

```text
instruction
address
mode bits（如果启用）
dummy bytes
```

`k230_dw_ssi_send_enhanced_field()` 按大端字节顺序逐字节调用 `ssi_transfer()`。
QEMU 的 `SSIBus`/Flash 模型是字节级的，所以 Dual/Quad 线宽被保存在命令描述中，
实际底层模型仍按字节交换。

#### 第六步：按 `TMOD` 搬运数据

`RO`：

```c
for (i = 0; i < length; i++)
    buffer[i] = ssi_transfer(s->spi, 0);

dma_memory_write(&address_space_memory, address, buffer, length, ...);
```

数据流是：

```text
Flash -> SSI -> temporary buffer -> Guest memory
```

`TO`：

```c
for (i = 0; i < length; i++)
    ssi_transfer(s->spi, buffer[i]);
```

数据流是：

```text
Guest memory -> temporary buffer -> SSI -> Flash
```

#### 第七步：结束片选并产生 DONE

```c
k230_dw_ssi_deselect(s);
k230_dw_ssi_idma_complete(s);
```

`k230_dw_ssi_idma_complete()` 会：

```text
idma_completed = true
irq_latched |= DONE
更新内部 IRQ 输出
```

然后 Guest 可以读取 `DONECR` 清除 DONE 状态。

## 5. 用 qtest 手推一次 Quad IDMA read

用例：

```text
/home/flamboy/qemu-camp-2026/qemu-camp-2026-k230/tests/qtest/k230-dw-ssi-idma-test.c
```

对应 `test_quad_idma_read_and_done()`，约在 70--93 行。

测试配置：

```text
TMOD        = RO
SPI_FRF     = Quad
TRANS_TYPE  = 0
WAIT_CYCLES = 8
SPIDR       = 0x6b（Quad Output Read）
SPIAR       = Flash pattern address
AXIAR0      = 0x80201000
length      = 4
CTRLR1.NDF  = 3
```

启动后的逻辑：

```text
1. NDF=3，所以 data_frames=4
2. SPIDR=0x6b，发送读指令
3. SPIAR 指向 Flash pattern 区域
4. WAIT_CYCLES=8，当前模型折算为 1 个 dummy byte
5. 从 Flash 收 4 个 byte
6. 写入 Guest 地址 0x80201000
7. 锁存 DONE
8. 测试读取 DONECR，确认 DONE 被清除
9. 读取 0x80201000，比较得到期望数据
```

这个测试同时验证了三个不同结果：

- SPI 数据是否正确。
- Guest memory 是否真的被写入。
- DONE/RISR/ISR/PLIC 是否正确联动。

## 6. `TO` 和 `RO` 的区别

| 项目 | `TMOD=TO` | `TMOD=RO` |
|---|---|---|
| SPI 方向 | 向 Flash 写 | 从 Flash 读 |
| 内存角色 | 源 | 目的地 |
| QEMU 首个内存操作 | `dma_memory_read()` | 无，先接收 SPI |
| QEMU 最后内存操作 | 无 | `dma_memory_write()` |
| SDK cache 重点 | 发送前 clean | 接收后 invalidate |
| 常见 Flash 操作 | Quad Page Program | Quad Output/IO Read |
| 完成条件 | 所有数据发送后 DONE | 所有数据接收并写回后 DONE |

页编程 qtest `test_quad_idma_page_program()` 展示了完整链路：

```text
1. 先用 PIO 发送 WREN
2. 把 256 byte 发送数据写入 Guest memory
3. 用 IDMA + TMOD=TO 执行 Quad Page Program
4. 等待 Flash 内部编程完成
5. 用 IDMA + TMOD=RO 读回同一页
6. 比较发送和读回的数据
```

这说明 IDMA 只负责“数据块搬运”，Flash 的 WREN、busy、program 等协议状态仍由
Flash 设备模型负责。

## 7. 当前 QEMU 已实现与未实现

### 已实现

- Dual/Quad SDR IDMA 的 `RO` 和 `TO` 功能路径。
- `SPIDR`、`SPIAR`、`AXIAR0/1` 地址和命令组合。
- `CTRLR1.NDF + 1` 数据长度。
- Guest memory 到 SPI、SPI 到 Guest memory 的搬运。
- IDMA 完成后 DONE 锁存。
- `DONECR` 读清。
- DONE 经过 `IMR`、SSI IRQ 输出和 K230 PLIC 路由。
- 失败的内存访问不产生 DONE。
- 寄存器写入顺序不固定时，使用辅助状态等待所有必要字段齐备。

### 当前模型的简化

- 在一个 MMIO 回调中同步完成整个传输。
- 使用临时 buffer，未模拟 SSI FIFO 在 IDMA 中的真实持续供给/排空过程。
- 未使用 `ATW` 决定实际内存访问宽度。
- 未使用 `AXIAWLEN/AXIARLEN` 产生 AXI burst。
- 未模拟 `AID`、`APROT`、`ACACHE` 的 AXI 信号。
- 当前数据帧按 byte 处理，未完整覆盖 `DFS` 大于 8 的 IDMA 数据宽度。
- 没有完整实现 AXI error、`AXIE` 事件和错误中止后的 `CMPLTD_DF`。
- 没有完全复现真实硬件完成后自动清 `SSIENR` 的行为。
- IDMA 开启后对 `DR` 的 AHB error response 尚未作为独立行为实现。

### “同步完成”与真实硬件的区别

这里的“同步完成”描述的是 QEMU 模型的执行方式，不是说真实 SSI/IDMA 硬件
也是同步外设。

真实硬件的时间线更接近：

```text
CPU 写 SSIENR=1
        |
        | MMIO 写入返回
        v
SSI 硬件继续运行：AXI burst -> FIFO -> SPI 时钟 -> Flash 数据
        |
        v
DONE/AXIE 中断
```

当前 QEMU 的时间线则是：

```text
CPU 写 SSIENR=1
        |
        v
k230_dw_ssi_try_idma()
        |
        +-- dma_memory_read/write()
        +-- ssi_transfer() 循环
        +-- 设置 DONE
        v
MMIO 写入返回
```

也就是说，在当前模型中，Guest 写完最后一个启动寄存器后，整个事务已经完成；
真实硬件中，启动寄存器写入返回后，AXI、FIFO 和 SPI 传输通常还在继续。

这是一种常见的功能级 QEMU 简化。它可以快速得到最终数据和完成状态，但不会自然
产生下面这些中间状态：

- `SR.BUSY=1` 持续一段硬件时间；
- FIFO 逐渐增加或减少；
- CPU 在传输过程中观察到未完成的 `DONE`；
- AXI burst 之间的 backpressure；
- SPI 时钟、片选和 SSI 时钟之间的真实时序。

因此，当前 qtest 可以在配置函数返回后直接检查 DONE；真实驱动通常必须轮询 DONE
或等待 DONE/AXIE 中断。

### 为什么仍然需要 AXI 语义

IDMA 的本质不是简单的：

```text
buffer = memory[address : address + length]
```

而是 SSI 作为 AXI master 对系统内存发起访问。AXI 语义决定了以下问题：

| AXI 语义 | 可能影响 |
|---|---|
| `ATW` | 访问宽度、地址对齐和数据拼接 |
| `AINC` | 每次 burst 后地址如何变化 |
| `AXIAWLEN/AXIARLEN` | 每个 burst 的长度和 FIFO 供给节奏 |
| `ACACHE/APROT/AID` | 总线属性、权限和事务标识 |
| AXI response | 访问失败、传输中止、AXIE 和部分完成量 |

QEMU 不一定要模拟 AXI 的每根信号线，但必须根据 Guest 可观察需求选择要保留的
语义。当前模型已经抽象实现了最小的一组：

```text
传输方向
内存起始地址
传输长度
内存读写是否成功
成功后产生 DONE
```

当前模型没有实现 burst、访问宽度和 AXI sideband，因此它适合验证“数据最终搬到
哪里”和“基本 DONE 是否产生”，不适合验证真实总线时序和所有错误边界。

可以用下面的标准判断一个被保存但未参与行为的字段是否合理：

```text
1. 它是否影响最终数据内容？
2. 它是否影响 Guest 可观察的地址或长度？
3. 它是否影响错误、中断或部分完成量？
4. 它是否只是性能、总线属性或物理时序字段？
```

前 3 类字段最终需要实现，或者明确拒绝不支持的配置；第 4 类字段在功能级模型中
可以先保存和读回。保存不等于实现，只是保留了寄存器接口契约。

### 必须单独标记的 TRM/SDK 差异

TRM 的 AXI 控制信号表写的是：

```text
AINC=0 -> INCR burst
AINC=1 -> FIXED burst
```

但当前 SDK：

```text
RT-Smart：spi->dmacr = ... | (1 << 6) | ...
U-Boot：  dmacr.dmacr.ainc = 1
QEMU：    k230_dw_ssi_idma_ready() 要求 AINC=1
```

结论只能写成：

```text
K230 SDK 的可运行 IDMA 配置使用 AINC=1。
AINC=1 在 K230 实际集成中的地址/burst 语义，需要以实机或更具体的硬件资料确认。
```

不要在学习笔记中把它简化成“AINC=1 表示地址递增”。

## 8. qtest 覆盖和阅读入口

当前 IDMA qtest 有四组：

| 用例 | 重点 |
|---|---|
| `idma/quad-read-done` | Quad read、Guest memory 写回、DONE 清除 |
| `idma/quad-page-program` | `TO` 写入和 `RO` 读回的闭环 |
| `idma/quad-io-read` | `TRANS_TYPE=1` 的 Quad IO read |
| `idma/bad-address` | 内存地址错误时不产生 DONE |

建议按下面顺序读代码：

```text
1. qtest configure_idma()
2. qtest test_quad_idma_read_and_done()
3. k230_dw_ssi_idma_ready()
4. k230_dw_ssi_try_idma()
5. k230_dw_ssi_idma_complete()
6. DONECR read-clear
7. K230 PLIC DONE 路由
```

定位命令：

```bash
rg -n "k230_dw_ssi_idma|IDMAE|AINC|SPIDR|SPIAR|AXIAR|DONECR" \
    "/home/flamboy/qemu-camp-2026/qemu-camp-2026-k230/hw/ssi" \
    "/home/flamboy/qemu-camp-2026/qemu-camp-2026-k230/include/hw/ssi" \
    "/home/flamboy/qemu-camp-2026/qemu-camp-2026-k230/tests/qtest"
```

## 9. 后续学习路线

下一步不要先看所有 AXI 字段，先完成下面三次手推：

### 手推 A：Quad Output Read

```text
TMOD=RO，TRANS_TYPE=0，0x6b，地址，dummy，数据写内存
```

重点看：`SPIDR/SPIAR` 如何组成事务头，`AXIAR0/1` 如何成为目的地址。

### 手推 B：Quad Page Program

```text
先 WREN，再 TMOD=TO，内存读出数据，发送到 Flash
```

重点看：为什么 `TO` 必须先读 Guest memory，为什么 SDK 要先 clean cache。

### 手推 C：AXI error

```text
AXI 访问失败 -> 片选撤销 -> SSI 关闭 -> FIFO 清空 -> AXIE
```

重点看：当前 QEMU 哪些状态已经有入口，哪些还只是寄存器占位。

## 10. 资料索引

```text
TRM：
/home/flamboy/qemu-camp-2026/K230_Technical_Reference_Manual_V0.3.1_20241118.pdf
/home/flamboy/qemu-camp-2026/exper-note/k230/K230_Technical_Reference_Manual_V0.3.1_20241118.txt

QEMU 控制器：
/home/flamboy/qemu-camp-2026/qemu-camp-2026-k230/hw/ssi/k230_dw_ssi.c
/home/flamboy/qemu-camp-2026/qemu-camp-2026-k230/include/hw/ssi/k230_dw_ssi.h

QEMU qtest：
/home/flamboy/qemu-camp-2026/qemu-camp-2026-k230/tests/qtest/k230-dw-ssi-idma-test.c

K230 SDK：
/home/flamboy/k230_sdk/src/big/rt-smart/kernel/bsp/maix3/board/interdrv/spi/drv_spi.c
/home/flamboy/k230_sdk/src/little/uboot/drivers/spi/designware_spi.c
```
