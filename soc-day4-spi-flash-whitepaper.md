# Day 4 白皮书：SPI Flash，从字节传输到 Flash 命令状态机

## 来源

这份 Day4 固定以 WSL 实验仓库里的真实 qtest 为最高优先级，官方手册作为背景参考。

| 来源 | 用法 |
| --- | --- |
| `/home/flamboy/qemu-camp/qemu-camp-2026-exper-flamboyante/tests/gevico/qtest/test-spi-jedec.c` | SPI 控制器最小传输和 JEDEC ID |
| `/home/flamboy/qemu-camp/qemu-camp-2026-exper-flamboyante/tests/gevico/qtest/test-flash-read.c` | Flash polling 模式的 erase / program / read |
| `/home/flamboy/qemu-camp/qemu-camp-2026-exper-flamboyante/tests/gevico/qtest/test-flash-read-interrupt.c` | SPI TXE/RXNE interrupt 和 PLIC IRQ 5 |
| `/home/flamboy/qemu-camp/qemu-camp-2026-exper-flamboyante/tests/gevico/qtest/test-spi-cs.c` | CS0/CS1 双 Flash、容量边界和状态隔离 |
| `/home/flamboy/qemu-camp/qemu-camp-2026-exper-flamboyante/tests/gevico/qtest/test-spi-overrun.c` | RX overrun、W1C、ERRIE 到 IRQ 5 |
| [G233 SoC 硬件手册](https://github.com/gevico/qemu-camp-tutorial/blob/main/docs/exercise/2026/stage1/soc/g233-datasheet.md) | SPI/Flash 的硬件背景、地址和中断模型 |
| [qemu-hw-study.md](./qemu-hw-study.md) | `SysBusDevice`、`MemoryRegionOps`、read/write 回调 |
| [qemu-clock-study.md](./qemu-clock-study.md) | `QEMUTimer` 和 `qtest_clock_step` |
| [qemu-board-study.md](./qemu-board-study.md) | board map、PLIC IRQ 接线 |

⚠️ 重要提醒：`g233.c` 里已经有 `virt.flash0/1`，但它们是 `TYPE_PFLASH_CFI01`，映射在 `VIRT_FLASH = 0x20000000`，用于 board 固件/pflash 这条线。Day4 的 SPI Flash 是挂在 `SPI_BASE = 0x10018000` 后面的串行 Flash，访问入口是 `SPI_CR1/CR2/SR/DR`，不是 `RISCVG233State->flash[2]`。本轮不要复用现有 pflash，默认在 `G233SPIState` 内嵌两个 SPI flash state。

## 阅读说明

Day2 的 GPIO 是“寄存器 + IRQ”。Day3 的 PWM/WDT 是“寄存器 + 虚拟时间”。Day4 的 SPI Flash 会再多一个难点：协议状态机。

```text
GPIO:
  write offset -> 改 state

PWM/WDT:
  write offset -> 改 state -> timer 到期 -> 改 state/IRQ

SPI Flash:
  write SPI_DR 一个 byte
  -> SPI controller 处理 TXE/RXNE/OVERRUN/IRQ
  -> 根据 CS 选择一片 Flash
  -> Flash command state machine 消费这个 byte
  -> 产生 MISO 返回 byte
  -> 读 SPI_DR 拿到 rx byte
```

这份文档的目标不是让你背 SPI 规范，而是让你能把 5 个 qtest 翻译成实现规格，然后按阶段写出可维护的 QEMU 设备模型。

## 一句话结论

SPI Flash 不是一个更大的 MMIO 寄存器表，而是：

```text
SPI controller 状态机
+ CS 路由
+ Flash command 状态机
+ storage / busy / WEL
+ TXE/RXNE/OVERRUN/IRQ side effect
```

默认实现方案：

```text
G233SPIState 一个 QEMU 设备
  master/controller 部分:
    CR1 / CR2 / SR / DR / rx_byte / tx_byte / transfer_timer / irq

  内嵌 slave 部分:
    flash[0] = CS0 W25X16, 2MB, JEDEC EF 30 15
    flash[1] = CS1 W25X32, 4MB, JEDEC EF 30 16
```

---

## Day4 核心心智模型

```text
qtest / driver
  |
  | qtest_writel(SPI_DR, tx)
  v
SPI MemoryRegionOps.write(offset=SPI_DR)
  |
  | 保存 tx byte
  | 清/更新 TXE
  | 安排 transfer_timer
  v
QEMUTimer on virtual time
  |
  | qtest_clock_step(qts, 1000)
  v
g233_spi_transfer_done()
  |
  | cs = SPI_CR2 & 0x3
  | flash = &s->flash[cs]
  | rx = g233_spi_flash_xfer(flash, tx)
  v
SPI controller state
  |
  | s->rx = rx
  | SPI_SR.RXNE = 1
  | SPI_SR.TXE = 1
  | 如果旧 RXNE 还没清又来了新 byte: OVERRUN = 1
  v
g233_spi_update_irq()
  |
  | (TXEIE && TXE) || (RXNEIE && RXNE) || (ERRIE && OVERRUN)
  v
PLIC IRQ 5
```

关键分层：

| 层 | 负责什么 | 不要混淆 |
| --- | --- | --- |
| qtest/driver | 写 `CR1/CR2/DR`，读 `SR/DR` | 它看不到内部 flash state，只通过 SPI 协议交互 |
| SPI master/controller | MMIO、TXE/RXNE、CS、IRQ、overrun | 不应该直接把 Flash storage 暴露成 MMIO |
| SPI flash slave | JEDEC/status/read/program/erase/storage/busy | 它不关心 PLIC，只返回一个 rx byte |
| board | map `0x10018000`，接 PLIC IRQ 5 | 不应该写 Flash 命令逻辑 |

---

## 1. 今天的任务边界

今天做：

| 测题 | 目标 |
| --- | --- |
| `test-spi-jedec` | SPI init、byte transfer、JEDEC ID、TXE/RXNE |
| `test-flash-read` | polling 模式读 status、sector erase、page program、read data |
| `test-flash-read-interrupt` | TXEIE/RXNEIE 触发 PLIC IRQ 5 |
| `test-spi-cs` | CS0/CS1 双 Flash 独立、不同 JEDEC、容量边界 |
| `test-spi-overrun` | RXNE 未清时第二个 byte 完成，置 OVERRUN，ERRIE 时 IRQ |

今天不做：

| 不做 | 为什么 |
| --- | --- |
| Rust SPI 测题 | 那是另一条扩展路线，先不混入 Day4 |
| I2C | I2C 是后续模块，不属于 SPI Flash 主线 |
| 复用 `virt.flash0/1` pflash | 它们是 memory-mapped pflash，不是 SPI serial flash |
| 完整 SSI bus/QOM slave 拆分 | 当前训练营优先过 qtest，默认方案 A 内嵌 flash state |
| 完整真实 SPI timing | qtest 只要求短延迟后 `TXE/RXNE/BUSY` 状态变化 |

---

## 2. SPI 驱动机理

### 2.1 SPI 主从关系是什么

SPI 是同步串行总线。最核心的概念是：master 发一个 byte 的同时，也会从 slave 收一个 byte。

```text
MOSI: master -> slave, 本次 tx byte
MISO: slave  -> master, 本次 rx byte
SCLK: master 产生时钟
CS:   master 选择哪一个 slave
```

在真实硬件里，一次 SPI transfer 是同时发送和接收。软件通常写 data register 启动传输，然后等状态位告诉它传输完成：

```text
等待 TXE=1
写 SPI_DR = tx
等待 RXNE=1
读 SPI_DR = rx
```

测题里的 helper 正是这个模型：

```c
static uint8_t spi_transfer_byte(QTestState *qts, uint8_t tx)
{
    spi_wait_txe(qts);
    qtest_writel(qts, SPI_DR, tx);
    spi_wait_rxne(qts);
    return (uint8_t)qtest_readl(qts, SPI_DR);
}
```

### 2.2 `TXE` 和 `RXNE` 怎么理解

| 状态位 | 含义 | 软件动作 |
| --- | --- | --- |
| `TXE` | TX buffer empty，可以写下一个 byte | 写 `SPI_DR` 前轮询它 |
| `RXNE` | RX buffer not empty，有 byte 可读 | 读 `SPI_DR` 拿 rx byte |
| `OVERRUN` | RXNE 还没清，又收到新 byte | 说明软件读得太慢 |

最小状态流：

```text
reset:
  TXE=1, RXNE=0, OVERRUN=0

write SPI_DR:
  TXE=0
  安排一次 transfer

transfer done:
  如果 RXNE 已经是 1:
    OVERRUN=1
  rx_byte = flash_xfer(tx_byte)
  RXNE=1
  TXE=1

read SPI_DR:
  返回 rx_byte
  RXNE=0
```

### 2.3 为什么建议 transfer 用 QEMUTimer

很多测题都有：

```c
while (!(qtest_readl(qts, SPI_SR) & SPI_SR_RXNE) && --timeout) {
    qtest_clock_step(qts, 1000);
}
```

这说明测题允许你把“一次 byte transfer”建模成一个短暂的未来事件。推荐写法：

```text
write SPI_DR
  -> 清 TXE
  -> 保存 tx
  -> timer_mod(xfer_timer, now + 1000ns)

qtest_clock_step(1000)
  -> transfer_done callback
  -> 设置 RXNE/TXE
```

同步完成也可能过一些测试，但它会让你少练一块重要能力：带虚拟延迟的设备行为。Day4 默认用 `QEMUTimer`，这样和 Flash busy 也统一。

---

## 3. Flash 驱动机理

### 3.1 SPI Flash 为什么是命令状态机

SPI Flash 没有一堆 MMIO 寄存器给 CPU 直接读写。CPU 想操作它，只能通过 SPI 一字节一字节发送命令。

常见流程：

```text
读 JEDEC:
  CS low
  send 0x9F
  read 3 dummy bytes -> flash 返回 JEDEC ID
  CS high

读数据:
  CS low
  send 0x03
  send addr[23:16], addr[15:8], addr[7:0]
  read N dummy bytes -> flash 返回 storage[addr++]
  CS high

写数据:
  send 0x06         -> write enable
  send 0x02 + addr + data bytes
  flash busy 一段时间
  read status 等 BUSY 清除
```

所以 Flash 内部必须记住“我现在收到的是命令、地址第几个 byte，还是 data 阶段”。这就是 command state machine。

### 3.2 Day4 需要支持哪些命令

| 命令 | 值 | 行为 |
| --- | --- | --- |
| `JEDEC_ID` | `0x9F` | 后续读出 3 字节 JEDEC |
| `READ_STATUS` | `0x05` | 返回 status，至少要有 BUSY bit |
| `WRITE_ENABLE` | `0x06` | 设置 WEL，允许后续 erase/program |
| `READ_DATA` | `0x03` | 收 3 字节地址后连续读 storage |
| `PAGE_PROGRAM` | `0x02` | 收 3 字节地址后写 data |
| `SECTOR_ERASE` | `0x20` | 收 3 字节地址后擦 4KB sector |

Flash status 建议：

| bit | 名称 | 当前测题是否显式检查 |
| --- | --- | --- |
| `0` | `BUSY` | 检查，`flash_wait_busy()` 等它清 0 |
| `1` | `WEL` | 测题不直接 assert，但建议实现，避免语义混乱 |

`WEL/BUSY` 生命周期建议固定成这条线：

```text
WRITE_ENABLE:
  WEL = 1
  CS deassert 不清 WEL

PAGE_PROGRAM / SECTOR_ERASE:
  只有 WEL=1 时才允许修改 storage
  CS deassert 时 finalize transaction
  进入 BUSY

busy_timer 到期:
  BUSY = 0
  WEL = 0
```

BUSY 期间的命令处理建议：

| 命令 | BUSY=1 时建议 |
| --- | --- |
| `READ_STATUS` | 必须可用，返回 `BUSY=1`，否则 `flash_wait_busy()` 会卡死 |
| `JEDEC_ID` / `READ_DATA` | 本轮可返回 `0xFF` 或保持简化行为 |
| `WRITE_ENABLE` / `PAGE_PROGRAM` / `SECTOR_ERASE` | 本轮可忽略，避免 busy 期间重入修改 |

🧠 我的理解：`READ_STATUS` 是 Flash 忙的时候给软件看的“呼吸灯”。如果 BUSY 后连 status 都读不了，驱动就不知道什么时候能继续。

### 3.3 storage 语义

Flash 初始值是 `0xFF`。这是 NOR Flash 的关键特性：擦除把 bit 置回 1，编程只能把 1 写成 0，不能把 0 写回 1。

```text
erase:
  storage[sector] = 0xFF

program:
  storage[addr+i] = storage[addr+i] & data[i]
```

虽然测题会先 erase 再 program，所以直接赋值也可能过，但文档建议用 `old & new`，这是更像真实 flash 的写法。

### 3.4 dummy clock 和 dummy byte 怎么区分

SPI 是 full-duplex。master 每发一个 byte，slave 也会同时返回一个 byte。Flash 没有“单独读”的动作，所谓读数据，本质上仍然是 master 继续发送 byte，给总线提供 SCLK，slave 才能在 MISO 上吐出数据。

Flash 手册或时序图里的 **dummy clock** 是器件层面的说法：命令和地址之后，Flash 可能需要若干个 SCLK 周期准备数据。在这些 clock 上，MOSI 的值通常没有意义，MISO 可能也还不是有效数据。

但从软件和 SPI controller 的视角看，软件不能直接“发送 clock”。master 只有写 `SPI_DR` 发送一个 byte，控制器才会产生 8 个 SCLK。所以测题里这些 `0x00` 不是为了把 0 写进 Flash，而是用一个 dummy byte 触发一组 clock：

```c
id[0] = spi_transfer_byte(qts, 0x00);
id[1] = spi_transfer_byte(qts, 0x00);
id[2] = spi_transfer_byte(qts, 0x00);
```

这几个 byte 在代码层面是 dummy byte：

```text
MOSI: 0x00 只是占位
SCLK: 由 master/controller 因这次 transfer 产生
MISO: Flash 在同一拍返回 JEDEC / data / status
```

🧠 我的理解：SPI 里“读”也必须“写”。这不是写内存，也不是 Flash 自己产生 clock，而是 master 用 MOSI 上的 dummy byte 产生 clock，再换 MISO 上的有效数据。

---

## 4. 寄存器和测试规格

### 4.1 SPI 寄存器

| 寄存器 | 地址 / offset | 语义 |
| --- | --- | --- |
| `SPI_BASE` | `0x10018000` | SPI 控制器基地址 |
| `SPI_CR1` | `0x00` | enable、master、interrupt enable |
| `SPI_CR2` | `0x04` | bits `[1:0]` 选择 CS |
| `SPI_SR` | `0x08` | RXNE/TXE/OVERRUN |
| `SPI_DR` | `0x0C` | 数据寄存器，写 tx，读 rx |

`SPI_CR1` bit：

| bit | 名称 | 作用 |
| --- | --- | --- |
| `0` | `SPE` | SPI enable |
| `2` | `MSTR` | master mode |
| `5` | `ERRIE` | error interrupt enable |
| `6` | `RXNEIE` | RXNE interrupt enable |
| `7` | `TXEIE` | TXE interrupt enable |

`SPI_SR` bit：

| bit | 名称 | 行为 |
| --- | --- | --- |
| `0` | `RXNE` | transfer 完成后置 1，读 `DR` 清 0 |
| `1` | `TXE` | reset/init 后为 1，写 `DR` 后可短暂清 0，完成后置 1 |
| `4` | `OVERRUN` | RXNE 未清又完成新 byte 时置 1，写 1 清除 |

### 4.2 推荐宏定义清单

这些宏建议集中放在 `g233_spi.h` 或 `g233_spi.c` 顶部。后面的伪代码都会使用这些名字。

```c
#define TYPE_G233_SPI "g233-spi"

#define G233_SPI_SIZE 0x100
#define G233_SPI_NR_FLASH 2

#define G233_SPI_CR1 0x00
#define G233_SPI_CR2 0x04
#define G233_SPI_SR  0x08
#define G233_SPI_DR  0x0C

#define SPI_CR1_SPE    (1u << 0)
#define SPI_CR1_MSTR   (1u << 2)
#define SPI_CR1_ERRIE  (1u << 5)
#define SPI_CR1_RXNEIE (1u << 6)
#define SPI_CR1_TXEIE  (1u << 7)

#define SPI_SR_RXNE    (1u << 0)
#define SPI_SR_TXE     (1u << 1)
#define SPI_SR_OVERRUN (1u << 4)

#define SPI_CR2_CS_MASK 0x3u

#define SPI_PLIC_IRQ 5

#define FLASH_CMD_WRITE_ENABLE  0x06
#define FLASH_CMD_READ_STATUS   0x05
#define FLASH_CMD_READ_DATA     0x03
#define FLASH_CMD_PAGE_PROGRAM  0x02
#define FLASH_CMD_SECTOR_ERASE  0x20
#define FLASH_CMD_JEDEC_ID      0x9F

#define FLASH_SR_BUSY (1u << 0)
#define FLASH_SR_WEL  (1u << 1)

#define FLASH_SECTOR_SIZE 4096
#define FLASH_PAGE_SIZE   256
#define FLASH_CS0_SIZE    (2 * MiB)
#define FLASH_CS1_SIZE    (4 * MiB)
```

### 4.3 Flash 参数

| CS | 型号 | 容量 | JEDEC |
| --- | --- | --- | --- |
| CS0 | W25X16 | 2MB | `EF 30 15` |
| CS1 | W25X32 | 4MB | `EF 30 16` |

### 4.4 测题到规格映射

| 测题 | 测试函数 | 规格 |
| --- | --- | --- |
| `test-spi-jedec` | `g233/spi/init` | `CR1.SPE/MSTR` 写后读回 |
| `test-spi-jedec` | `g233/spi/jedec_id` | CS0 下 `0x9F` 后读 `EF 30 15` |
| `test-spi-jedec` | `g233/spi/transfer_byte` | 初始 `TXE=1`，写 `DR` 后 `RXNE=1`，读 `DR` 清 RXNE |
| `test-flash-read` | `read_status` | idle status 不 busy |
| `test-flash-read` | `sector_erase` | sector erase 后读出 `0xFF` |
| `test-flash-read` | `page_program` | erase -> program 256B -> read back 一致 |
| `test-flash-read` | `write_test_data` | erase -> program 64B -> read back 一致 |
| `test-flash-read-interrupt` | `txe_interrupt` | `TXEIE && TXE` 触发 PLIC IRQ 5 |
| `test-flash-read-interrupt` | `rxne_interrupt` | transfer 后 `RXNEIE && RXNE` 触发 PLIC IRQ 5 |
| `test-flash-read-interrupt` | `jedec_interrupt` | interrupt 模式仍能读 JEDEC |
| `test-flash-read-interrupt` | `write_test_data` | interrupt 模式下 erase/program/read 正常 |
| `test-spi-cs` | `identification` | CS0 JEDEC `EF3015`，CS1 JEDEC `EF3016` |
| `test-spi-cs` | `individual/cross/alternating` | 两片 flash storage 和 command state 独立 |
| `test-spi-cs` | `capacity` | CS0 2MB 边界、CS1 4MB 边界可写读 |
| `test-spi-cs` | `concurrent_status` | 两片 idle status 都不 busy |
| `test-spi-overrun` | `interrupt` | `ERRIE && OVERRUN` 触发 PLIC IRQ 5 |
| `test-spi-overrun` | `polling` | `OVERRUN` 可 W1C 清除 |

---

## 5. 默认实现方案：控制器内嵌两个 Flash state

### 5.1 为什么采用方案 A

方案 A 不是“不区分 SPI 主从”，而是“逻辑上区分主从，代码上放在一个 QEMU 设备里”。

```text
G233SPIState
  |
  | master/controller:
  |   CR1 / CR2 / SR / DR / xfer_timer / irq
  |
  | embedded slave:
  |   flash[0] = CS0
  |   flash[1] = CS1
```

对训练营来说，这个方案最合适：

| 对比 | 内嵌 flash state | 拆 SSI bus + flash device |
| --- | --- | --- |
| 文件数量 | 少 | 多 |
| QOM 复杂度 | 低 | 高 |
| 当前 qtest 适配 | 直接 | 需要更多 bus glue |
| 主从逻辑 | 仍然清晰 | 更正式 |
| 推荐程度 | 本轮默认 | 后续增强 |

### 5.2 推荐 state

```c
#define TYPE_G233_SPI "g233-spi"

#define G233_SPI_SIZE 0x100
#define G233_SPI_NR_FLASH 2

typedef enum G233FlashPhase {
    FLASH_PHASE_IDLE,
    FLASH_PHASE_JEDEC,
    FLASH_PHASE_ADDR,
    FLASH_PHASE_READ,
    FLASH_PHASE_PROGRAM,
    FLASH_PHASE_STATUS,
} G233FlashPhase;

typedef struct G233SPIFlash {
    uint8_t *storage;
    uint32_t size;
    uint8_t jedec[3];
    uint8_t status;
    uint8_t cmd;
    uint32_t addr;
    int addr_bytes;
    int jedec_pos;
    bool program_touched;
    bool erase_pending;
    uint32_t erase_addr;
    G233FlashPhase phase;
    QEMUTimer *busy_timer;
} G233SPIFlash;

typedef struct G233SPIState {
    SysBusDevice parent_obj;
    MemoryRegion mmio;
    qemu_irq irq;

    uint32_t cr1;
    uint32_t cr2;
    uint32_t sr;
    uint8_t tx;
    uint8_t rx;

    QEMUTimer *xfer_timer;
    G233SPIFlash flash[G233_SPI_NR_FLASH];
} G233SPIState;
```

字段归属表：

| 字段 | 属于哪一层 | 说明 |
| --- | --- | --- |
| `cr1/cr2/sr` | SPI master/controller | MMIO 寄存器影子 |
| `tx/rx` | SPI master/controller | 当前 transfer 的发送/接收 byte |
| `xfer_timer` | SPI master/controller | 一次 byte transfer 的完成事件 |
| `irq` | SPI master/controller | 汇总 TXE/RXNE/ERR 到 PLIC IRQ 5 |
| `flash[i].storage` | SPI slave flash | 每片 Flash 的持久存储内容 |
| `flash[i].jedec` | SPI slave flash | 每片 Flash 的身份 |
| `flash[i].status` | SPI slave flash | BUSY/WEL 等 Flash 状态 |
| `flash[i].cmd/addr/phase` | SPI slave flash transaction | 当前 CS transaction 的临时状态 |
| `flash[i].busy_timer` | SPI slave flash | erase/program 结束后的内部 busy 延迟 |

⚠️ 容易踩坑：`rx` 是 SPI controller 的接收缓冲，不是 Flash 的 storage；`cmd/addr/phase` 是某一片 Flash 的协议状态，不应该放成全局唯一变量，否则 CS0/CS1 会互相污染。

### 5.3 reset 默认值

```text
SPI:
  cr1 = 0
  cr2 = 0
  sr = TXE
  tx = 0
  rx = 0
  irq = low
  xfer_timer stopped

Flash:
  storage 全 0xFF
  status = 0
  cmd = 0
  addr = 0
  phase = IDLE
  CS0 jedec = EF 30 15, size = 2MB
  CS1 jedec = EF 30 16, size = 4MB
```

⚠️ 容易踩坑：`TXE` 初始必须是 1。`test_spi_transfer_byte` 会先 assert 初始 `TXE` 非 0。

---

## 6. 新 API 和 QEMU 机制讲解

### 6.1 `memory_region_init_io`

用途：创建一个 MMIO 区域，把 guest 访问映射到你的 read/write 回调。

```c
memory_region_init_io(&s->mmio, OBJECT(dev), &g233_spi_ops, s,
                      TYPE_G233_SPI, G233_SPI_SIZE);
sysbus_init_mmio(SYS_BUS_DEVICE(dev), &s->mmio);
```

你要理解这几个参数：

| 参数 | 意义 |
| --- | --- |
| `&s->mmio` | 设备自己的 MemoryRegion |
| `OBJECT(dev)` | owner，通常就是设备对象 |
| `&g233_spi_ops` | read/write 回调表 |
| `s` | opaque，回调里会拿回 `G233SPIState *` |
| `TYPE_G233_SPI` | region 名字，调试用 |
| `G233_SPI_SIZE` | MMIO 窗口大小 |

`MemoryRegionOps`：

```c
static const MemoryRegionOps g233_spi_ops = {
    .read = g233_spi_read,
    .write = g233_spi_write,
    .endianness = DEVICE_LITTLE_ENDIAN,
    .impl.min_access_size = 4,
    .impl.max_access_size = 4,
};
```

当前 qtest 全部用 `qtest_readl/writel`，所以 4 字节访问足够。

### 6.2 `sysbus_init_irq` 和 `sysbus_connect_irq`

设备内部声明一根 IRQ 输出线：

```c
sysbus_init_irq(SYS_BUS_DEVICE(dev), &s->irq);
```

board 里把这根线接到 PLIC source 5：

```c
sysbus_connect_irq(sbd, 0, qdev_get_gpio_in(mmio_irqchip, SPI_PLIC_IRQ));
```

设备里通过 `qemu_set_irq()` 改变这根线电平：

```c
qemu_set_irq(s->irq, level);
```

SPI IRQ 规则：

```c
static void g233_spi_update_irq(G233SPIState *s)
{
    bool level =
        ((s->cr1 & SPI_CR1_TXEIE) && (s->sr & SPI_SR_TXE)) ||
        ((s->cr1 & SPI_CR1_RXNEIE) && (s->sr & SPI_SR_RXNE)) ||
        ((s->cr1 & SPI_CR1_ERRIE) && (s->sr & SPI_SR_OVERRUN));

    qemu_set_irq(s->irq, level);
}
```

⚠️ PLIC pending 是 sticky 的。qtest 只读 pending，不做 claim/complete，所以你只要把线拉起来，pending 就会出现。

### 6.3 `timer_new_ns` / `timer_mod` / `timer_del`

SPI transfer 和 Flash busy 都推荐用 `QEMUTimer`。

创建 timer：

```c
s->xfer_timer = timer_new_ns(QEMU_CLOCK_VIRTUAL,
                             g233_spi_transfer_done, s);
```

安排未来事件：

```c
int64_t now = qemu_clock_get_ns(QEMU_CLOCK_VIRTUAL);
timer_mod(s->xfer_timer, now + 1000);
```

取消未来事件：

```c
timer_del(s->xfer_timer);
```

常见使用点：

| timer | 触发时机 | callback 做什么 |
| --- | --- | --- |
| `xfer_timer` | 写 `SPI_DR` 后短延迟 | 完成一次 byte transfer，设置 `RXNE/TXE` |
| `flash.busy_timer` | erase/program 后短延迟 | 清 `BUSY`，清 `WEL` |

### 6.4 `g_malloc0` / `g_free`

Flash storage 建议动态分配：

```c
flash->storage = g_malloc0(flash->size);
memset(flash->storage, 0xFF, flash->size);
```

`g_malloc0` 会先分配并清零，但 Flash 初始值不是 0，所以还要 `memset(..., 0xFF, ...)`。

这句话很重要：`realize/flash_init` 分配 storage 后必须 `memset(..., 0xFF, ...)`，因为每个新 QEMU 进程里的 Flash 初始内容应该像擦除后的 NOR Flash。`reset` 不重新分配 storage，也不把 storage 清 0；它只清当前 SPI transaction、状态位和 timer。

释放可以放在 unrealize：

```c
g_free(flash->storage);
```

训练营测题不一定覆盖 unrealize，但写上会更像正式设备。

### 6.5 realize / reset / unrealize 分工

这三个生命周期函数很容易混。建议按职责分清：

| 函数 | 做什么 | 不做什么 |
| --- | --- | --- |
| `instance_init` | 初始化 MMIO、IRQ 输入输出这类对象骨架 | 不依赖外部配置，不启动业务行为 |
| `realize` | 分配 storage、创建 timer、初始化每片 flash 的固定参数 | 不把寄存器 reset 逻辑散落在这里 |
| `reset` | 清寄存器、清 phase、停 timer、恢复默认状态 | 不重新分配 storage |
| `unrealize` | 释放 `storage`、释放 timer | 不改 guest 可见状态 |

推荐分工：

```c
static void g233_spi_init(Object *obj)
{
    G233SPIState *s = G233_SPI(obj);
    SysBusDevice *sbd = SYS_BUS_DEVICE(obj);

    memory_region_init_io(&s->mmio, obj, &g233_spi_ops, s,
                          TYPE_G233_SPI, G233_SPI_SIZE);
    sysbus_init_mmio(sbd, &s->mmio);
    sysbus_init_irq(sbd, &s->irq);
}
```

```c
static void g233_spi_realize(DeviceState *dev, Error **errp)
{
    G233SPIState *s = G233_SPI(dev);

    s->xfer_timer = timer_new_ns(QEMU_CLOCK_VIRTUAL,
                                 g233_spi_transfer_done, s);
    g233_spi_flash_init(&s->flash[0], 2 * MiB, 0xEF, 0x30, 0x15);
    g233_spi_flash_init(&s->flash[1], 4 * MiB, 0xEF, 0x30, 0x16);
}
```

```c
static void g233_spi_reset(DeviceState *dev)
{
    G233SPIState *s = G233_SPI(dev);

    s->cr1 = 0;
    s->cr2 = 0;
    s->sr = SPI_SR_TXE;
    s->tx = 0;
    s->rx = 0;
    timer_del(s->xfer_timer);
    qemu_set_irq(s->irq, 0);

    for (int i = 0; i < G233_SPI_NR_FLASH; i++) {
        g233_spi_flash_reset_transaction(&s->flash[i]);
    }
}
```

```c
static void g233_spi_unrealize(DeviceState *dev)
{
    G233SPIState *s = G233_SPI(dev);

    timer_free(s->xfer_timer);
    for (int i = 0; i < G233_SPI_NR_FLASH; i++) {
        timer_free(s->flash[i].busy_timer);
        g_free(s->flash[i].storage);
    }
}
```

注意：realize/flash_init 必须分配 storage 并初始化为 `0xFF`；reset 不要重新 `g_malloc` storage，也不要把已经 program 的 Flash 内容清成 0。对当前 qtest 来说，每个测试都是新 QEMU 进程，影响不大；但从设备模型习惯看，分清生命周期会让代码更稳。

最后还要把这些函数挂到 QOM 类型上。否则你写了 `realize/reset/unrealize`，QEMU 也不会调用它们：

```c
static void g233_spi_class_init(ObjectClass *klass, const void *data)
{
    DeviceClass *dc = DEVICE_CLASS(klass);

    dc->realize = g233_spi_realize;
    dc->unrealize = g233_spi_unrealize;
    device_class_set_legacy_reset(dc, g233_spi_reset);
}

static const TypeInfo g233_spi_info = {
    .name = TYPE_G233_SPI,
    .parent = TYPE_SYS_BUS_DEVICE,
    .instance_size = sizeof(G233SPIState),
    .instance_init = g233_spi_init,
    .class_init = g233_spi_class_init,
};

static void g233_spi_register_types(void)
{
    type_register_static(&g233_spi_info);
}

type_init(g233_spi_register_types)
```

你可以把这段理解成“把 C 结构体和 QEMU 设备类型注册起来”。`qdev_new(TYPE_G233_SPI)` 能工作，靠的就是这里的 `TypeInfo`。

### 6.6 `qemu_log_mask`

未知 offset 或不支持命令时，不建议直接 assert。用 guest error log 更稳：

```c
qemu_log_mask(LOG_GUEST_ERROR,
              "G233 SPI bad read offset 0x%" HWADDR_PRIx "\n", offset);
```

这类日志不会改变测题逻辑，但 debug 时很有用。

---

## 7. SPI controller 实现手册

### 7.1 read/write 行为

read：

| offset | 返回 |
| --- | --- |
| `SPI_CR1` | `s->cr1` |
| `SPI_CR2` | `s->cr2` |
| `SPI_SR` | `s->sr` |
| `SPI_DR` | `s->rx`，并清 `RXNE` |

write：

| offset | 行为 |
| --- | --- |
| `SPI_CR1` | 保存 `SPE/MSTR/ERRIE/RXNEIE/TXEIE`，更新 IRQ |
| `SPI_CR2` | 保存 CS，切换 CS 时结束当前 flash transaction |
| `SPI_SR` | W1C 清 `OVERRUN`，更新 IRQ |
| `SPI_DR` | 保存 tx，清 TXE，安排 transfer timer |

写 `SPI_CR1` 后必须立刻 `g233_spi_update_irq()`。原因是 `TXE` reset 后本来就是 1，如果 guest 写入 `TXEIE`，`TXEIE && TXE` 立刻成立，不需要等下一次 transfer。

```c
case SPI_CR1:
    s->cr1 = value & (SPI_CR1_SPE |
                      SPI_CR1_MSTR |
                      SPI_CR1_ERRIE |
                      SPI_CR1_RXNEIE |
                      SPI_CR1_TXEIE);
    g233_spi_update_irq(s);
    break;
```

`test_flash_read_interrupt` 的 `txe_interrupt` 就依赖这个行为。

### 7.2 写 `SPI_DR`

```c
static void g233_spi_write_dr(G233SPIState *s, uint8_t value)
{
    if (!(s->cr1 & SPI_CR1_SPE)) {
        return;
    }

    if (!(s->sr & SPI_SR_TXE)) {
        qemu_log_mask(LOG_GUEST_ERROR,
                      "G233 SPI write DR while TXE=0\n");
        return;
    }

    s->tx = value;
    s->sr &= ~SPI_SR_TXE;

    int64_t now = qemu_clock_get_ns(QEMU_CLOCK_VIRTUAL);
    timer_mod(s->xfer_timer, now + 1000);
    g233_spi_update_irq(s);
}
```

为什么只保存 `uint8_t`：SPI 是 byte transfer，qtest 虽然用 `qtest_writel` 写 32 位，但有效数据是低 8 位。

当前测题都会等 `TXE=1` 后再写 `DR`。如果 guest 在 `TXE=0` 时又写 `DR`，本轮不需要做 FIFO；建议直接忽略并打一条 guest error log，避免覆盖还没完成的 `tx`。

### 7.3 transfer done callback

```c
static void g233_spi_transfer_done(void *opaque)
{
    G233SPIState *s = opaque;
    unsigned cs = s->cr2 & 0x3;
    uint8_t rx = 0xFF;

    if (cs < G233_SPI_NR_FLASH) {
        rx = g233_spi_flash_xfer(&s->flash[cs], s->tx);
    } else {
        /*
         * CS=2/3 are not connected in this training model.
         * Return erased-bus style data and, most importantly, do not
         * index flash[cs].
         */
        rx = 0xFF;
    }

    if (s->sr & SPI_SR_RXNE) {
        s->sr |= SPI_SR_OVERRUN;
    }

    s->rx = rx;
    s->sr |= SPI_SR_RXNE | SPI_SR_TXE;
    g233_spi_update_irq(s);
}
```

⚠️ 容易踩坑：overrun 要在“新 byte 完成”时判断，而不是在写 `DR` 时立刻判断。测题第二次写 `DR` 后会 `qtest_clock_step(qts, 100000)` 再查 `OVERRUN`，这正好对应 transfer done。

### 7.4 读 `SPI_DR`

```c
case SPI_DR:
    ret = s->rx;
    s->sr &= ~SPI_SR_RXNE;
    g233_spi_update_irq(s);
    return ret;
```

读 `DR` 清 `RXNE` 是 `test_spi_transfer_byte` 和 overrun 测题的关键。如果你忘了清，后面几乎所有 transfer 都会变成 overrun。

### 7.5 写 `SPI_CR2` 和 CS transaction 边界

`SPI_CR2 bits[1:0]` 选择 CS。测题常用这个模式：

```text
select CS0
send command/data
select CS1  // 用另一个 CS 表示释放当前片选
```

这是一种训练营简化片选模型。真实硬件里通常是单独的 CS 引脚拉低/拉高；当前测题没有单独的 CS high/low 寄存器，而是通过写 `SPI_CR2` 从 0 切到 1 来模拟“释放 CS0”，或者从 1 切到 0 模拟“释放 CS1 再选中 CS0”。

所以文档里说的 `CS deassert`，在当前实现里就是：

```text
old_cs != new_cs
  -> old_cs 对应 Flash 的 transaction 结束
```

所以写 `CR2` 时，如果 CS 发生变化，建议结束旧 flash 的 transaction：

```c
static void g233_spi_write_cr2(G233SPIState *s, uint32_t value)
{
    unsigned old_cs = s->cr2 & 0x3;
    unsigned new_cs = value & 0x3;

    if (old_cs < G233_SPI_NR_FLASH && old_cs != new_cs) {
        g233_spi_flash_cs_deassert(&s->flash[old_cs]);
    }

    s->cr2 = new_cs;
}
```

`cs_deassert` 负责 finalize 当前 transaction，再把 `cmd/phase/addr_bytes` 等临时状态清掉，但不要清 storage、status、busy 这类持久状态。

invalid CS 处理建议：

| CS | 行为 |
| --- | --- |
| `0` | 选择 CS0 |
| `1` | 选择 CS1 |
| `2/3` | 无有效 Flash，transfer 返回 `0xFF`，也可以视为所有 CS 都释放 |

测题只使用 `0/1`。实现上建议对 `2/3` 保守处理，不要数组越界。

代码级原则：

```c
if (cs < G233_SPI_NR_FLASH) {
    rx = g233_spi_flash_xfer(&s->flash[cs], s->tx);
} else {
    rx = 0xFF;
}
```

不要写 `&s->flash[s->cr2 & 0x3]` 后再判断，否则 `CS=2/3` 已经越界了。

---

## 8. Flash command 状态机

### 8.1 状态机总览

```text
IDLE
  |
  | tx = command
  v
CMD decoded
  |
  +-- 0x9F JEDEC_ID ------> JEDEC phase
  +-- 0x05 READ_STATUS ---> STATUS phase
  +-- 0x06 WRITE_ENABLE --> set WEL, back IDLE
  +-- 0x03 READ_DATA -----> ADDR phase -> READ phase
  +-- 0x02 PAGE_PROGRAM --> ADDR phase -> PROGRAM phase -> CS deassert finalize + BUSY
  +-- 0x20 SECTOR_ERASE --> ADDR phase -> mark erase pending -> CS deassert finalize + BUSY
```

### 8.2 flash xfer 主入口

```c
static uint8_t g233_spi_flash_xfer(G233SPIFlash *f, uint8_t tx)
{
    switch (f->phase) {
    case FLASH_PHASE_IDLE:
        return g233_spi_flash_decode_cmd(f, tx);
    case FLASH_PHASE_JEDEC:
        return g233_spi_flash_read_jedec(f);
    case FLASH_PHASE_STATUS:
        return f->status;
    case FLASH_PHASE_ADDR:
        return g233_spi_flash_collect_addr(f, tx);
    case FLASH_PHASE_READ:
        return g233_spi_flash_read_data(f);
    case FLASH_PHASE_PROGRAM:
        return g233_spi_flash_program_data(f, tx);
    default:
        return 0xFF;
    }
}
```

### 8.3 command decode

```c
static uint8_t g233_spi_flash_decode_cmd(G233SPIFlash *f, uint8_t cmd)
{
    f->cmd = cmd;
    f->addr = 0;
    f->addr_bytes = 0;
    f->jedec_pos = 0;

    switch (cmd) {
    case FLASH_CMD_JEDEC_ID:
        f->phase = FLASH_PHASE_JEDEC;
        return 0xFF;
    case FLASH_CMD_READ_STATUS:
        f->phase = FLASH_PHASE_STATUS;
        return 0xFF;
    case FLASH_CMD_WRITE_ENABLE:
        f->status |= FLASH_SR_WEL;
        f->phase = FLASH_PHASE_IDLE;
        return 0xFF;
    case FLASH_CMD_READ_DATA:
    case FLASH_CMD_PAGE_PROGRAM:
    case FLASH_CMD_SECTOR_ERASE:
        f->phase = FLASH_PHASE_ADDR;
        return 0xFF;
    default:
        f->phase = FLASH_PHASE_IDLE;
        return 0xFF;
    }
}
```

### 8.4 地址收集

Flash 命令里的地址是 24-bit，大端顺序发送：

```text
addr[23:16]
addr[15:8]
addr[7:0]
```

```c
static uint8_t g233_spi_flash_collect_addr(G233SPIFlash *f, uint8_t tx)
{
    f->addr = (f->addr << 8) | tx;
    f->addr_bytes++;

    if (f->addr_bytes < 3) {
        return 0xFF;
    }

    f->addr %= f->size;

    if (f->cmd == FLASH_CMD_READ_DATA) {
        f->phase = FLASH_PHASE_READ;
    } else if (f->cmd == FLASH_CMD_PAGE_PROGRAM) {
        f->phase = FLASH_PHASE_PROGRAM;
    } else if (f->cmd == FLASH_CMD_SECTOR_ERASE) {
        f->erase_pending = true;
        f->erase_addr = f->addr;
        f->phase = FLASH_PHASE_IDLE;
    }

    return 0xFF;
}
```

### 8.5 JEDEC / status / read / program

JEDEC：

```c
static uint8_t g233_spi_flash_read_jedec(G233SPIFlash *f)
{
    if (f->jedec_pos < 3) {
        return f->jedec[f->jedec_pos++];
    }
    return 0xFF;
}
```

read data：

```c
static uint8_t g233_spi_flash_read_data(G233SPIFlash *f)
{
    uint8_t v = f->storage[f->addr % f->size];
    f->addr = (f->addr + 1) % f->size;
    return v;
}
```

program data：

```c
static uint8_t g233_spi_flash_program_data(G233SPIFlash *f, uint8_t tx)
{
    if (f->status & FLASH_SR_WEL) {
        f->storage[f->addr % f->size] &= tx;
        f->addr = (f->addr + 1) % f->size;
        f->program_touched = true;
    }
    return 0xFF;
}
```

⚠️ 正确性重点：不要在每个 data byte 里调用 `g233_spi_flash_set_busy()`。测题每传一个 byte 都可能 `qtest_clock_step(1000)`，256B page program 过程中虚拟时间会推进很多。如果你过早启动 busy timer，它可能在后续 data byte 还没传完时清掉 `WEL`，导致后半页写不进去。

更合理的模型：

```text
PAGE_PROGRAM:
  CS low
  收 0x02
  收 3 字节 addr
  收 data bytes，持续写入 storage 或缓存
  CS high / CR2 切换时，结束 transaction
  此时再置 BUSY，启动 busy_timer
  busy done 后清 BUSY 和 WEL
```

### 8.6 CS deassert 如何 finalize transaction

`cs_deassert` 是 Flash 命令的收尾点。它应该处理“这个 transaction 到此结束后，Flash 内部是否进入 busy”。

```c
static void g233_spi_flash_cs_deassert(G233SPIFlash *f)
{
    if (f->cmd == FLASH_CMD_PAGE_PROGRAM && f->program_touched) {
        g233_spi_flash_set_busy(f);
    }

    if (f->cmd == FLASH_CMD_SECTOR_ERASE && f->erase_pending) {
        g233_spi_flash_erase_sector_now(f, f->erase_addr);
        if (f->status & FLASH_SR_WEL) {
            g233_spi_flash_set_busy(f);
        }
    }

    f->cmd = 0;
    f->addr = 0;
    f->addr_bytes = 0;
    f->jedec_pos = 0;
    f->program_touched = false;
    f->erase_pending = false;
    f->phase = FLASH_PHASE_IDLE;
}
```

注意：`READ_DATA`、`READ_STATUS`、`JEDEC_ID` 这类读命令在 CS deassert 时只需要清 transaction 临时状态，不需要 busy。

erase/program finalize 都必须受 `WEL` 约束：没有先执行 `WRITE_ENABLE`，就算收到了 `SECTOR_ERASE` 或 `PAGE_PROGRAM`，也应该忽略写入/擦除，不应该置 BUSY。上面的 `erase_sector_now()` 里也会检查 `WEL`，这里再判断一次是为了避免“没有实际 erase 却启动 busy timer”。

### 8.7 erase 和 busy

sector erase：

```c
static void g233_spi_flash_erase_sector_now(G233SPIFlash *f, uint32_t addr)
{
    if (!(f->status & FLASH_SR_WEL)) {
        return;
    }

    uint32_t base = addr & ~(SECTOR_SIZE - 1);
    for (uint32_t i = 0; i < SECTOR_SIZE && base + i < f->size; i++) {
        f->storage[base + i] = 0xFF;
    }

}
```

busy timer：

```c
static void g233_spi_flash_set_busy(G233SPIFlash *f)
{
    f->status |= FLASH_SR_BUSY;

    int64_t now = qemu_clock_get_ns(QEMU_CLOCK_VIRTUAL);
    timer_mod(f->busy_timer, now + 100000);
}
```

busy done：

```c
static void g233_spi_flash_busy_done(void *opaque)
{
    G233SPIFlash *f = opaque;

    f->status &= ~FLASH_SR_BUSY;
    f->status &= ~FLASH_SR_WEL;
}
```

`flash_wait_busy()` 会循环读 status，并每次 `qtest_clock_step(qts, 100000)`。所以 busy 不要用 host sleep，必须靠虚拟时间清除。

### 8.8 三条逐字节 trace

#### JEDEC ID

```text
前置:
  CR2 = 0, 选择 CS0
  CR1 = SPE | MSTR

byte 0:
  MOSI tx = 0x9F
  flash phase: IDLE -> JEDEC
  MISO rx = 0xFF   // 命令阶段返回值不重要

byte 1:
  MOSI tx = 0x00   // dummy byte，触发 master 产生时钟
  flash phase: JEDEC
  MISO rx = 0xEF

byte 2:
  MOSI tx = 0x00
  flash phase: JEDEC
  MISO rx = 0x30

byte 3:
  MOSI tx = 0x00
  flash phase: JEDEC
  MISO rx = 0x15
```

#### READ_DATA

```text
byte 0:
  MOSI tx = 0x03
  phase: IDLE -> ADDR
  MISO rx = 0xFF

byte 1:
  MOSI tx = addr[23:16]
  phase: ADDR, addr_bytes=1
  MISO rx = 0xFF

byte 2:
  MOSI tx = addr[15:8]
  phase: ADDR, addr_bytes=2
  MISO rx = 0xFF

byte 3:
  MOSI tx = addr[7:0]
  phase: ADDR -> READ
  MISO rx = 0xFF

byte 4..:
  MOSI tx = 0x00 dummy byte
  phase: READ
  MISO rx = storage[addr++]
```

#### PAGE_PROGRAM

```text
transaction A:
  CS0 select
  tx 0x06
  phase: IDLE, set WEL
  CS deassert

transaction B:
  CS0 select
  tx 0x02
  phase: IDLE -> ADDR

  tx addr[23:16]
  tx addr[15:8]
  tx addr[7:0]
  phase: ADDR -> PROGRAM

  tx data[0]
  storage[addr] &= data[0]
  program_touched = true

  tx data[1..N]
  storage[addr++] &= data[i]
  program_touched = true

  CS deassert
  finalize:
    if program_touched:
      set BUSY
      start busy_timer

busy done:
  clear BUSY
  clear WEL
```

---

## 9. IRQ 和 overrun

### 9.1 SPI IRQ 5

三个条件汇总到一根 IRQ 线：

```text
TXEIE  && TXE
RXNEIE && RXNE
ERRIE  && OVERRUN
```

```c
static void g233_spi_update_irq(G233SPIState *s)
{
    bool txe_irq = (s->cr1 & SPI_CR1_TXEIE) && (s->sr & SPI_SR_TXE);
    bool rxne_irq = (s->cr1 & SPI_CR1_RXNEIE) && (s->sr & SPI_SR_RXNE);
    bool err_irq = (s->cr1 & SPI_CR1_ERRIE) && (s->sr & SPI_SR_OVERRUN);

    qemu_set_irq(s->irq, txe_irq || rxne_irq || err_irq);
}
```

对应测题：

| 条件 | 测题 |
| --- | --- |
| `TXEIE && TXE` | `g233/flash-int/txe_interrupt` |
| `RXNEIE && RXNE` | `g233/flash-int/rxne_interrupt` |
| `ERRIE && OVERRUN` | `g233/spi-overrun/interrupt` |

### 9.2 overrun

overrun 的本质是：软件还没读走上一个 rx byte，硬件又收到一个新的 rx byte。

```text
write DR 0xAA
  -> transfer done
  -> RXNE=1

不读 DR

write DR 0xBB
  -> transfer done
  -> 发现 RXNE 仍然是 1
  -> OVERRUN=1
```

W1C 清除：

```c
case SPI_SR:
    s->sr &= ~(value & SPI_SR_OVERRUN);
    g233_spi_update_irq(s);
    break;
```

⚠️ 容易踩坑：读 `DR` 应该清 `RXNE`，但不应该清 `OVERRUN`。`OVERRUN` 只能写 `SPI_SR` 的 bit4 清。

---

## 10. board / meson / 头文件接入提示

建议文件：

| 项 | 建议 |
| --- | --- |
| 头文件 | `hw/riscv/g233_feat/g233_spi.h` |
| 源文件 | `hw/riscv/g233_feat/g233_spi.c` |
| meson | `hw/riscv/meson.build` 里 `CONFIG_GEVICO_G233` 增加 `g233_feat/g233_spi.c` |
| board include | `hw/riscv/g233.c` include `g233_spi.h` |
| memmap | `VIRT_SPI = 0x10018000, size 0x100` |
| IRQ | `SPI_PLIC_IRQ = 5` |

board helper：

```c
static void g233_spi_create(hwaddr addr, qemu_irq irq)
{
    DeviceState *dev = qdev_new(TYPE_G233_SPI);
    SysBusDevice *s = SYS_BUS_DEVICE(dev);

    sysbus_realize_and_unref(s, &error_fatal);
    sysbus_mmio_map(s, 0, addr);
    sysbus_connect_irq(s, 0, irq);
}
```

machine init 里：

```c
g233_spi_create(s->memmap[VIRT_SPI].base,
                qdev_get_gpio_in(mmio_irqchip, SPI_PLIC_IRQ));
```

⚠️ 不要把 SPI Flash 写进 `RISCVG233State->flash[2]`。那个字段已经被 pflash 使用。

---

## 11. 分阶段执行记录

### 阶段 1：SPI 基础寄存器和 TXE/RXNE

目标：先让 `test-spi-jedec` 的 init 和 transfer byte 走通。

运行命令：

```bash
make -C build/tests/gevico/qtest run-spi-jedec
```

期望输出：

```text
g233/spi/init PASS
g233/spi/transfer_byte PASS
```

我的实际输出：

```text
<运行后贴 3-10 行关键日志>
```

失败定位：

| 现象 | 优先排查 |
| --- | --- |
| `CR1` 读不回 | read/write offset |
| 初始 `TXE` 不是 1 | reset 默认值 |
| 写 `DR` 后没有 `RXNE` | `xfer_timer` 没创建、没 `timer_mod`、callback 没执行 |
| 读 `DR` 后 `RXNE` 不清 | `SPI_DR` read 逻辑 |

### 阶段 2：JEDEC ID

目标：CS0 下 `0x9F` 返回 `EF 30 15`。

期望输出：

```text
g233/spi/jedec_id PASS
qtest-riscv64/test-spi-jedec PASS
```

我的实际输出：

```text
<运行后贴 3-10 行关键日志>
```

失败定位：

| 现象 | 优先排查 |
| --- | --- |
| 返回全 `0xFF` | `FLASH_CMD_JEDEC_ID` 没进入 JEDEC phase |
| 第一个 byte 对，后面错 | `jedec_pos` 没递增或 CS transaction 被提前清 |
| CS1 影响 CS0 | `flash[cs]` 选择错 |

### 阶段 3：Flash polling erase/program/read

目标：`test-flash-read` 全过。

运行命令：

```bash
make -C build/tests/gevico/qtest run-flash-read
```

期望输出：

```text
g233/flash/read_status PASS
g233/flash/sector_erase PASS
g233/flash/page_program PASS
g233/flash/write_test_data PASS
qtest-riscv64/test-flash-read PASS
```

我的实际输出：

```text
<运行后贴 3-10 行关键日志>
```

失败定位：

| 现象 | 优先排查 |
| --- | --- |
| 一直 busy | `busy_timer` 没清 `BUSY` |
| erase 后不是 `0xFF` | sector base 计算或 storage 初始化 |
| program 后读不回 | `WRITE_ENABLE` / `WEL` / `PROGRAM` phase |
| 只写了第一个 byte | program phase 被提前结束 |

### 阶段 4：SPI interrupt

目标：`TXEIE/RXNEIE` 能进入 PLIC IRQ 5。

运行命令：

```bash
make -C build/tests/gevico/qtest run-flash-read-interrupt
```

期望输出：

```text
g233/flash-int/txe_interrupt PASS
g233/flash-int/rxne_interrupt PASS
g233/flash-int/jedec_interrupt PASS
g233/flash-int/write_test_data PASS
qtest-riscv64/test-flash-read-interrupt PASS
```

我的实际输出：

```text
<运行后贴 3-10 行关键日志>
```

失败定位：

| 现象 | 优先排查 |
| --- | --- |
| TXEIE 不触发 | `TXE` 初始值和 `update_irq()` |
| RXNEIE 不触发 | transfer done 后是否调用 `update_irq()` |
| 状态对但 PLIC 没 pending | `sysbus_init_irq` / `sysbus_connect_irq` / IRQ 号 5 |

### 阶段 5：CS0/CS1 双 Flash

目标：`test-spi-cs` 全过。

运行命令：

```bash
make -C build/tests/gevico/qtest run-spi-cs
```

期望输出：

```text
g233/spi-cs/identification PASS
g233/spi-cs/individual PASS
g233/spi-cs/cross PASS
g233/spi-cs/alternating PASS
g233/spi-cs/capacity PASS
g233/spi-cs/concurrent_status PASS
qtest-riscv64/test-spi-cs PASS
```

我的实际输出：

```text
<运行后贴 3-10 行关键日志>
```

失败定位：

| 现象 | 优先排查 |
| --- | --- |
| CS1 JEDEC 还是 `EF3015` | 两片 jedec 没独立初始化 |
| CS0/CS1 数据串了 | storage 共用或 `flash[cs]` 选择错 |
| alternating 失败 | CS 切换没结束旧 transaction |
| capacity 失败 | size / addr wrap / 边界处理 |

### 阶段 6：overrun

目标：`test-spi-overrun` 全过。

运行命令：

```bash
make -C build/tests/gevico/qtest run-spi-overrun
```

期望输出：

```text
g233/spi-overrun/interrupt PASS
g233/spi-overrun/polling PASS
qtest-riscv64/test-spi-overrun PASS
```

我的实际输出：

```text
<运行后贴 3-10 行关键日志>
```

失败定位：

| 现象 | 优先排查 |
| --- | --- |
| overrun 不置位 | transfer done 时没检查旧 `RXNE` |
| polling 清不掉 | `SPI_SR` W1C 没实现 |
| interrupt 不触发 | `ERRIE && OVERRUN` 没进入 `update_irq()` |

---

## 12. 调试观察点

SPI Flash 最难 debug 的不是某个寄存器，而是“byte 到底推进到了哪个 phase”。建议先加临时 `qemu_log_mask()`，跑通后再考虑改成正式 trace-events。

推荐观察点：

| 位置 | 打什么 |
| --- | --- |
| write `SPI_DR` | `tx`、`cs`、写前 `SR` |
| `transfer_done` | `tx`、`rx`、写后 `SR` |
| `flash_xfer` | `cs`、`cmd`、`phase`、`addr`、`tx`、`rx` |
| `cs_deassert` | `cmd`、`program_touched`、`erase_pending`、`status` |
| `busy_done` | 清 BUSY/WEL 前后的 `status` |
| `update_irq` | `CR1`、`SR`、最终 `level` |

临时日志示例：

```c
qemu_log_mask(LOG_GUEST_ERROR,
              "g233-spi xfer cs=%u tx=0x%02x rx=0x%02x sr=0x%08x\n",
              cs, s->tx, s->rx, s->sr);
```

⚠️ 不要长期保留大量 guest error 日志。它们适合学习和定位，最终代码更建议用 `trace-events`，或者只保留异常路径日志。

---

## 13. Day4 结论速查

### SPI controller

| 问题 | 答案 |
| --- | --- |
| SPI base 是多少 | `0x10018000` |
| 数据寄存器是什么 | `SPI_DR = 0x0C` |
| 写 `DR` 表示什么 | 发起一次 byte transfer |
| 读 `DR` 表示什么 | 取出 rx byte，并清 `RXNE` |
| 初始 `TXE` 应该是多少 | 1 |
| overrun 什么时候发生 | `RXNE=1` 时又完成新 byte |
| overrun 怎么清 | 写 `SPI_SR.OVERRUN=1` |
| SPI IRQ 是几号 | `SPI_PLIC_IRQ=5` |

### Flash

| 问题 | 答案 |
| --- | --- |
| CS0 是什么 | W25X16，2MB，JEDEC `EF 30 15` |
| CS1 是什么 | W25X32，4MB，JEDEC `EF 30 16` |
| storage 初始值 | `0xFF` |
| `0x9F` | JEDEC ID |
| `0x05` | read status |
| `0x06` | write enable |
| `0x03` | read data |
| `0x02` | page program |
| `0x20` | sector erase |
| program 语义 | `old & new` |
| erase 语义 | sector 全部回到 `0xFF` |

### 实现顺序

```text
1. 空 SPI 设备 + map + CR1/CR2/SR/DR
2. TXE/RXNE + xfer_timer
3. CS0 JEDEC ID
4. Flash status / erase / program / read
5. TXEIE/RXNEIE IRQ 5
6. CS0/CS1 双 flash 独立
7. OVERRUN + ERRIE IRQ + W1C
```

### 最容易踩坑的 10 件事

| 坑 | 正确做法 |
| --- | --- |
| 把 `virt.flash0/1` 当 SPI flash | 不复用 pflash，SPI 内嵌 `flash[2]` |
| 忘记初始 `TXE=1` | reset 时设置 |
| 读 `DR` 不清 `RXNE` | `SPI_DR` read 后清 bit0 |
| 写 `DR` 同步完成但忘记 IRQ | 无论同步/异步，完成后都要 `update_irq()` |
| CS 切换不结束 transaction | `CR2` 变化时清旧 flash 临时 phase |
| 两片 flash 共用 storage | 每片独立分配 |
| program 只写第一个 byte | `PROGRAM` phase 保持到 CS deassert |
| busy 永远不清 | 用 `busy_timer` 绑定虚拟时间 |
| overrun 在写 `DR` 时判断 | 应在 transfer done 时判断旧 `RXNE` |
| 状态位对但 PLIC 没 pending | 查 `sysbus_connect_irq` 和 IRQ 5 |

---

## 14. 下一步怎么接

Day4 完成后，SoC 主线会进入收尾阶段：

```text
GPIO:
  基础 MMIO + IRQ

PWM/WDT:
  虚拟时间 + timer/down-counter

SPI Flash:
  字节传输 + 协议状态机 + storage + IRQ
```

接下来你应该能开始写真实实现了。推荐 commit 拆分：

```text
commit 1: add G233 SPI shell device and MMIO map
commit 2: implement SPI CR/SR/DR and byte transfer
commit 3: implement CS0 JEDEC state machine
commit 4: implement flash erase/program/read polling
commit 5: add SPI IRQ 5 support
commit 6: add CS0/CS1 independent flash state
commit 7: add overrun and W1C
```

这样每个 commit 都能对应一组 qtest，你学习时也能沿着 commit 一步步复盘，不会被一个“大而全”的 SPI 实现淹没。
