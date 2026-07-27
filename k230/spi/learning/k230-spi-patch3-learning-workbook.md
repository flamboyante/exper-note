# K230 SSI Patch 3 学习工作簿

更新时间：2026-07-16

## 目标

本工作簿只学习 K230 DWC SSI 的 Standard SPI PIO 数据路径：

```text
DR 写入 → TX FIFO → TMOD transfer pump → RX FIFO → DR 读取
```

本阶段不实现：

- IRQ、PLIC 和错误锁存。
- Dual/Quad/Octal QSPI。
- SPI NOR 协议细节。
- XIP 和内部 DMA。

代码脚手架位于：

- `my-qemu-camp-2026-k230/hw/ssi/k230_dw_ssi.c`
- `my-qemu-camp-2026-k230/include/hw/ssi/k230_dw_ssi.h`

搜索 `LEARNING(P3-` 可以找到已完成实现旁保留的学习提示。

Patch 3 的跨 MMIO 传输状态统一为：

```c
typedef struct K230DwSsiTransferState {
    K230DwSsiPhase phase;
    uint32_t remaining_frames;
} K230DwSsiTransferState;
```

不再保存独立的 `bool busy`。当前同步 Patch 3 模型用
`transfer_state.phase != IDLE` 动态推导 `SR.BUSY`，避免 RO/EEPROM_READ
因 RX FIFO 满而暂停时 busy 被错误清零。

## 学习规则

每次只完成一个学习步骤：

1. 阅读对应测试。
2. 写出本节的输入、输出和不变量。
3. 修改不超过一个核心函数。
4. 编译。
5. 只运行本节测试。
6. 测试通过后再进入下一节。

禁止为了让测试通过而提前实现 IRQ 或 QSPI。

## 第 0 关：确认脚手架

执行：

```bash
ninja -C "my-qemu-camp-2026-k230/build" \
    qemu-system-riscv64 \
    tests/qtest/k230-dw-ssi-test
```

预期：代码能够编译，但多数 Patch 3 PIO 测试失败。

这表示“代码结构合法，但硬件行为尚未填写”，属于正常起点。

## P3-1：DR 写入与 TX FIFO

位置：`k230_dw_ssi_push_tx()`。

先填写判断题：

1. `SSIENR=0` 时，Patch 3 是否接受 DR 写入？`__不接受；采用保守的 SDK 兼容策略__`
2. `SER=0` 时，DR 写入应当丢弃还是等待后续启动？`__等待后续启动__`
3. TX FIFO 满时，Patch 3 是否需要实现 TXO IRQ？`_本patch不需要__`
4. 写入值进入 FIFO 前应当使用哪个寄存器字段限制帧宽？`__DFS__`

必须维持的不变量：

```text
0 <= TXFLR <= 256
所有 DR0..DR35 共享同一个 TX FIFO
FIFO 中保存的是“帧”，不是字节
```

验证：

```bash
"my-qemu-camp-2026-k230/tests/qtest/run-k230-dw-ssi-tests.sh" \
    "pio/dr-aliases"

"my-qemu-camp-2026-k230/tests/qtest/run-k230-dw-ssi-tests.sh" \
    "pio/fifo-depth-256"

"my-qemu-camp-2026-k230/tests/qtest/run-k230-dw-ssi-tests.sh" \
    "pio/dr-disabled-ignored"
```

完成记录：

- 我删除或调整的前置条件：`__删除 active_cs < 0；保留 SSIENR=0 时忽略__`
- FIFO 满时本 Patch 的处理：`__忽略本次写入；TXO 由 IRQ Patch 实现__`
- 测试结果：`________________`

本节的启动顺序是：

```text
配置：SSIENR=0
→ SSIENR=1、SER=0
→ 写 DR 预填 TX FIFO
→ 写 SER 选择 CS
→ transfer pump 发送 FIFO 中的数据
```

这不是“SER 允许写 FIFO”，而是“SER 让已存在的 FIFO 数据具备线路
传输条件”。

## P3-2：单帧交换 helper

统一使用一个单帧交换 helper：

函数契约：

```c
static uint32_t k230_dw_ssi_send_frame(K230DwSsiState *s, uint32_t tx);
```

行为：

1. 发送前按 `DFS+1` 生成的 frame mask 截断 TX 帧。
2. `CTRLR0.SRL=1` 时，直接返回截断后的 TX，实现内部回环。
3. 否则调用 `ssi_transfer()` 把一帧交给 SSIBus。
4. 接收返回值再次按同一个 frame mask 截断。

不要在这个 helper 中：

- push/pop FIFO。
- 判断 TMOD。
- 更新 IRQ。
- 改变 CS。

验证由下一关的 TR 测试完成。

## P3-3：TMOD=TX_AND_RX

位置：`k230_dw_ssi_run_transfer()` 的 `case 0`。

核心伪代码：

```text
while TX FIFO 非空 and RX FIFO 未满:
    tx = pop TX FIFO
    rx = send_frame(tx)
    push rx 到 RX FIFO
}
```

必须先确认 RX FIFO 有空间，再 pop TX。若 RX FIFO 已满，应保留 TX FIFO 中的
未发送帧，由 Guest 读取 DR 腾出空间后恢复；不能先发送再丢弃返回值。

输入：TX FIFO 中已有若干帧。

输出：

- TX FIFO 被消费。
- 每个发送帧最多产生一个 RX FIFO 帧。
- SRL 回环返回被 DFS 截断后的原始帧。

验证：

```bash
"my-qemu-camp-2026-k230/tests/qtest/run-k230-dw-ssi-tests.sh" \
    "pio/tmod-tr"

"my-qemu-camp-2026-k230/tests/qtest/run-k230-dw-ssi-tests.sh" \
    "pio/dfs-frame-mask"
```

回答：为什么不能直接把 Guest 写入的 32 位值原样放入 RX FIFO？

`DFS 决定真实帧宽，Guest 写入值和线路返回值都必须按 DFS+1 截断；否则保留`
`高位会让 FIFO 内容与硬件帧语义不一致。`

## P3-4：TMOD=TX_ONLY

位置：`case 1`。

它与 TR 唯一的核心区别是：

```text
线路上仍然同时产生返回值，但控制器不把返回值写入 ___rx___ FIFO。
```

核心循环：

```text
while TX FIFO 非空:
    tx = pop TX FIFO
    send_frame(tx)
    丢弃返回值
```

TO 不受 RX FIFO 是否已满的限制，因为它不写 RX FIFO；也不需要手工更新
`bool busy`，TX FIFO 消费完后动态 BUSY 自然变为 0。

验证：

```bash
"my-qemu-camp-2026-k230/tests/qtest/run-k230-dw-ssi-tests.sh" \
    "pio/tmod-to"
```

检查结果：

- 最终 TXFLR：`__0__`
- 最终 RXFLR：`__0；线路返回值被丢弃__`
- 最终 BUSY：`__0__`

## P3-5：TMOD=RX_ONLY

位置：`case 2`。

### P3-5.1 软件可见契约

RX-only 不是“看到 SER 就立即接收”。它仍需要一个 dummy DR 写入作为启动
令牌，但这个 TX FIFO 项不是数据阶段的第一项 RX 数据：

```text
dummy DR → 丢弃启动帧 → 自动产生 NDF + 1 次接收
```

启动条件必须同时满足：

```text
SSIENR = 1
active_cs >= 0
SPI_FRF = Standard
TMOD = RX_ONLY
transfer_state.phase = IDLE
TX FIFO 至少有一个 dummy 启动帧
```

如果 SER 尚未选择 CS，dummy 帧只留在 TX FIFO，不能提前调用 SSIBus。

### P3-5.2 状态字段

只使用：

```c
s->transfer_state.phase
s->transfer_state.remaining_frames
```

不要增加 `bool busy`、`rx_active` 或第二份 NDF 计数。BUSY 从 phase/FIFO 动态
推导，NDF 只在事务启动时转换一次。

### P3-5.3 启动步骤

当 phase 为 IDLE 且 TX FIFO 非空时：

1. 从 TX FIFO pop **恰好一个** dummy 启动帧。
2. 不把该帧作为普通 TR 帧写入 RX FIFO。
3. 设置 `phase = K230_DW_SSI_PHASE_RX_ONLY`。
4. 设置 `remaining_frames = CTRLR1.NDF + 1`。
5. 进入自动接收循环。

`CTRLR1.NDF` 保存的是“接收帧数减一”，所以必须先加一再进入循环。不要在
每次 pump 恢复时重新读取 NDF 并覆盖 `remaining_frames`。

### P3-5.4 自动接收循环

QEMU SSIBus 仍需要一个发送值来产生一次同步线路交换。Patch 3 可统一使用
DFS 截断后的 `0` 作为内部 dummy 帧；它不进入 TX FIFO，也不改变 TXFLR。

伪代码：

```text
if phase == RX_ONLY:
    while remaining_frames > 0 and RX FIFO 未满:
        rx = send_frame(0)
        push rx 到 RX FIFO
        remaining_frames--

    if remaining_frames == 0:
        phase = IDLE
```

循环不允许在 RX FIFO 已满时继续调用 `ssi_transfer()`，否则从设备已经前进，
但返回值无法保存，会造成不可恢复的数据丢失。

### P3-5.5 完整状态转移

```text
IDLE
  --SSIENR/CS 有效且消费一个 dummy TX 帧-->
RX_ONLY，remaining_frames = CTRLR1.NDF + 1

RX_ONLY
  --RX FIFO 有空间，完成一次 send_frame(0)-->
RX FIFO push 一帧，remaining_frames--

RX_ONLY
  --RX FIFO 满且 remaining_frames > 0-->
保持 RX_ONLY 和 remaining_frames，暂停 pump

RX_ONLY
  --remaining_frames == 0-->
IDLE
```

完成只表示线路事务结束。RX FIFO 中可以仍有 Guest 尚未读取的数据；这些数据
不会让 BUSY 保持为 1。

如果 RX FIFO 满，当前 Patch 不实现 RXO IRQ。应保留未完成计数，等待 Guest
读取 DR 后由 P3-7 恢复。

验证：

```bash
"my-qemu-camp-2026-k230/tests/qtest/run-k230-dw-ssi-tests.sh" \
    "pio/tmod-ro"
```

回答：`CTRLR1.NDF=3` 为什么应得到 4 帧？

`NDF 编码的是接收数据帧数减一；硬件语义是 NDF + 1，所以 3 表示 4 帧。`

## P3-6：TMOD=EEPROM_READ

位置：`case 3`。

### P3-6.1 为什么必须是两阶段

EEPROM_READ 是 DesignWare SSI 为串行存储器提供的硬件辅助模式：Guest 只把
opcode/address 等控制帧写入 TX FIFO，并通过 `CTRLR1.NDF` 声明随后要接收的
数据长度。控制器负责在命令发送完毕后自动继续产生读时钟。

因此它不能使用 TR 的“一项 TX 对应一项 RX”逻辑：

- 命令阶段线路返回值没有数据意义，不能进入 RX FIFO。
- 数据阶段没有对应的 Guest TX FIFO 项，控制器必须自动产生 dummy 时钟。
- `NDF+1` 只统计数据阶段，不包含 opcode、address 或 dummy command bytes。

本次真实 U-Boot `sf probe` 取证已经验证该契约：Read-ID 时
`CTRLR0.TMOD=3`、`CTRLR1=5`，U-Boot 等待 6 个 RX 帧；由于模型没有进入
EEPROM 数据阶段，`RXFLR` 始终为 0，PC 卡在 `dw_reader()`。

### P3-6.2 启动条件

EEPROM_READ 事务只能在以下条件下启动：

```text
SSIENR = 1
active_cs >= 0
SPI_FRF = Standard
TMOD = EEPROM_READ
transfer_state.phase = IDLE
TX FIFO 非空，至少包含一个命令帧
```

SDK 支持先在 `SER=0` 时预填 opcode/address，再写 SER 启动。无 SER 时必须保留
TX FIFO，不能发送、不能清空，也不能提前把 phase 切到 EEPROM_DATA。

### P3-6.3 命令阶段

事务启动后设置：

```text
phase = EEPROM_COMMAND
```

然后消费当前 TX FIFO 中的命令帧：

```text
while phase == EEPROM_COMMAND and TX FIFO 非空:
    tx = pop TX FIFO
    send_frame(tx)
    丢弃线路返回值
```

命令帧必须按 FIFO 顺序完整发送。不能 push RX FIFO，也不能递减
`remaining_frames`，因为接收计数尚未开始。

在本 Patch 的 deferred-SER 契约下，EEPROM_READ 推荐在命令已预填、SER 写入
使 CS 首次有效时启动。不要在 `SER=0` 的每次 DR 写入后把“暂时 TX FIFO 空”
误判为命令已经完整发送。

### P3-6.4 COMMAND 到 DATA 的唯一切换点

只有命令阶段已经启动且 TX FIFO 全部消费完，才执行：

```text
phase = EEPROM_DATA
remaining_frames = CTRLR1.NDF + 1
```

NDF 只初始化一次。RX FIFO 满后重新进入 pump 时，只能继续使用保存的
`remaining_frames`，不能再次恢复为 `NDF+1`。

### P3-6.5 数据阶段

EEPROM_DATA 与 RX_ONLY 的自动接收循环相同，但不再查看或消费命令 TX FIFO：

```text
if phase == EEPROM_DATA:
    while remaining_frames > 0 and RX FIFO 未满:
        rx = send_frame(0)
        push rx 到 RX FIFO
        remaining_frames--

    if remaining_frames == 0:
        phase = IDLE
```

这里的 `send_frame(0)` 表示为 SSIBus 产生一帧读取时钟，不代表 Guest 又写了
DR，也不应增加 TXFLR。外部 SPI NOR 已经在命令阶段收到 opcode/address，数据
阶段返回的才是有效内容。

### P3-6.6 完整状态机

```text
IDLE
  --SSIENR/CS 有效且 TX FIFO 含命令-->
EEPROM_COMMAND

EEPROM_COMMAND：
  --每消费一个命令 TX 帧-->
  send_frame(tx)，丢弃返回值，不修改 remaining_frames

EEPROM_COMMAND
  --TX FIFO 已空-->
EEPROM_DATA，remaining_frames = CTRLR1.NDF + 1

EEPROM_DATA
  --RX FIFO 有空间-->
  send_frame(0)，push RX，remaining_frames--

EEPROM_DATA
  --RX FIFO 满且 remaining_frames > 0-->
  保持 EEPROM_DATA 和 remaining_frames，暂停 pump

EEPROM_DATA
  --remaining_frames == 0-->
IDLE
```

关键区别：

| 模式 | TX FIFO 内容 | 数据阶段如何启动 | RX 计数 |
|---|---|---|---|
| TR | 每个实际发送帧 | 每个 TX 项直接交换 | TX 项数决定 |
| TO | 每个实际发送帧 | 每个 TX 项直接发送 | 不保存 RX |
| RX_ONLY | 一个 dummy 启动令牌 | 消费令牌后自动开始 | `NDF+1` |
| EEPROM_READ | opcode/address 等命令帧 | 命令 FIFO 发完后自动开始 | `NDF+1`，不含命令帧 |

验证：

```bash
"my-qemu-camp-2026-k230/tests/qtest/run-k230-dw-ssi-tests.sh" \
    "pio/tmod-eeprom-read"
```

回答：为什么 EEPROM 命令阶段不能使用 TR 的逻辑把返回值放入 RX FIFO？

`命令阶段返回的是 opcode/address 发送期间的无效线路值；只有自动数据阶段的`
`NDF+1 帧属于 Guest 请求的数据。若按 TR 保存，RX FIFO 会混入命令返回值，`
`导致长度和字节位置全部错位。`

## P3-7：RX FIFO 腾出空间后恢复

位置：DR 读取路径。

Guest pop 一个 RX 帧后，如果：

```text
transfer_state.phase 是 RX_ONLY 或 EEPROM_DATA
并且 transfer_state.remaining_frames > 0
```

应再次调用 transfer pump，使暂停的 RO/EEPROM 数据阶段继续运行。

调用顺序必须是：

```text
先 pop RX FIFO
  -> RX FIFO 出现至少一个空位
  -> 再调用 transfer pump
  -> pump 最多填回可用空间
```

不要在返回 DR 值之前 pop 第二项，也不要在 RX FIFO 仍满时递归调用 pump。

需要防止：

- RX FIFO 仍满时死循环。
- `remaining_frames` 未递减时无限调用。
- 已完成事务被重复启动。

## P3-8：禁用与动态状态

检查 `k230_dw_ssi_abort_transfer()` 是否统一恢复：

```text
CS 撤销
TX FIFO 清空
RX FIFO 清空
transfer_state.phase = IDLE
transfer_state.remaining_frames = 0
```

不存在需要清理的 `bool busy`。禁用后的 BUSY 应由动态谓词自然得到 0。

### P3-8.1 BUSY 动态推导

建议建立只读 helper，而不是在各 TMOD 分支手工赋值：

```text
line_ready = SSIENR=1 && active_cs>=0 && SPI_FRF=Standard

SR.BUSY = (transfer_state.phase != IDLE)
```

该定义在同步 QEMU 模型中的含义：

- SER=0 时即使 TX FIFO 已预填，也没有线路事务，BUSY=0。
- TR/TO 在当前同步模型中完成 FIFO 消费后，BUSY=0。
- RO/EEPROM_DATA 因 RX FIFO 满暂停时，phase 非 IDLE，所以 BUSY=1。
- 自动接收全部生成完后 phase 回到 IDLE，即使 RX FIFO 仍有未读数据，BUSY=0。
- `SSIENR=0` 时无条件 BUSY=0。

不要把 `RXFLR != 0` 作为 BUSY 条件；RX FIFO 中有未读数据不表示线路仍在传输。

### P3-8.2 其他 SR 位

所有状态均从 FIFO 动态计算：

```text
TFNF = TXFLR < 256
TFE  = TXFLR == 0
RFNE = RXFLR != 0
RFF  = RXFLR == 256
```

TR/TO/RO/EEPROM_READ 分支只修改 FIFO 和 transfer_state，不直接保存 SR 位。

### P3-8.3 `dynamic-status` qtest

`pio/dynamic-status` 使用 RO、`NDF=256`，所以总接收帧数是 257：

```text
启动后：RXFLR=256、RFF=1、RFNE=1、BUSY=1
读取一个 DR 后：pump 补入最后一帧，RXFLR 仍为 256、BUSY=0
全部读空后：RXFLR=0、RFNE=0、RFF=0、TFE=1、TFNF=1
```

它验证的是动态状态，而不是 IRQ；RXO/RISR 仍属于 Patch 6。

验证：

```bash
"my-qemu-camp-2026-k230/tests/qtest/run-k230-dw-ssi-tests.sh" \
    "pio/dynamic-status"
```

### P3-8.4 reset 和迁移

reset/abort 只需要重置 `transfer_state.phase` 和 `remaining_frames`。VMState 只序列化
这两个跨 MMIO 状态字段，不再包含 `VMSTATE_BOOL(busy, ...)`。迁移恢复后 BUSY
会根据 phase 重新推导。

验证：

```bash
"my-qemu-camp-2026-k230/tests/qtest/run-k230-dw-ssi-tests.sh" \
    "pio/disable-clears-fifo"
```

## 最终 Patch 3 验收

只运行 PIO 组：

```bash
"my-qemu-camp-2026-k230/tests/qtest/run-k230-dw-ssi-tests.sh" "pio"
```

然后检查：

```bash
git -C "my-qemu-camp-2026-k230" diff --check
```

自评：

| 问题 | 能否脱离代码说明 |
|---|---|
| DR 写入和线路传输为什么必须分开？ | `SER=0 时 SDK 需要预填命令；DR 只入 FIFO，SSIENR/CS/TMOD 决定何时上线。` |
| TR 和 TO 的唯一区别是什么？ | `两者都在线路上完成同步交换；TR 保存返回值到 RX FIFO，TO 丢弃返回值。` |
| RO 为什么需要状态字段？ | `RX FIFO 满时必须跨 MMIO 暂停并保存尚未生成的 NDF 帧数。` |
| EEPROM_READ 为什么是两阶段？ | `命令阶段消费 TX 且丢弃 RX；数据阶段不消费 TX，而按 NDF 自动生成 RX。` |
| IRQ 为什么不应该写进 TMOD switch？ | `TMOD 只负责数据路径；IRQ 应统一观察 FIFO 水位和锁存事件，避免四个分支重复。` |

全部能够说明后，再进入 IRQ 或 QSPI Patch。

## 会话复盘：Patch 3 的关键裁决与答疑

本节不是实现步骤的重复，而是把本次学习中容易混淆的判断收敛为一份
“为什么这样做”的索引。阅读 Patch 4/5 时可以回到这里复习。

### 1. Patch 3 的边界是什么

Patch 3 只实现 Standard SPI 的 PIO 数据路径：

```text
DR 写入 → TX FIFO → TMOD transfer pump → RX FIFO → DR 读取
```

完成目标是 `TR`、`TO`、`RO`、`EEPROM_READ` 四种 TMOD 和通用 FIFO/CS
路径，而不是 SPI NOR、QSPI、IRQ、DMA 或 XIP。

因此，Patch 3 不能直接“读 Flash 镜像”，因为 Flash 从设备绑定属于 Patch 4；
也不能处理 Dual/Quad 线宽、mode bits、dummy cycles，因为这些属于 Patch 5；
更不会提供 `0xc0000000` 的 XIP window，那是 Patch 9。

### 2. SSIENR、SER 和 DR 分别决定什么

本模型采用的保守契约为：

```text
SSIENR = 0：忽略 DR 写入。
SSIENR = 1，SER = 0：DR 写入 TX FIFO，但不产生线路传输。
SSIENR = 1，SER 有一个有效 CS：DR 写入 TX FIFO，pump 可以推进传输。
```

`SSIENR` 是控制器是否接受本 Patch 的 DR 写和是否允许引擎运行的门；`SER`
选择从设备/CS，并决定线路是否已经具备开始条件；它不应成为 TX FIFO 是否接收
数据的门。

这样做是因为 K230 SDK 会先 enable，再在 `SER=0` 时预填 opcode/address，最后
写 `SER` 启动。证据见：

- `k230_sdk/src/little/linux/drivers/spi/spi-dw-core-k230.c` 的
  `dw_spi_write_then_read()`；
- `k230_sdk/src/little/uboot/drivers/spi/designware_spi.c` 的
  `dw_spi_exec_op()`。

TRM/SDK 没有直接证明 disabled 时 DR 写必须被忽略；这是为了避免禁用状态积累
陈旧数据而选择的、已由 `pio/dr-disabled-ignored` 固化的模型裁决。若未来有反证，
应修改裁决与 qtest，而不是在调用点私自绕过。

`DR0` 到 `DR35` 不是 36 个不同的 FIFO：它们是同一 TX/RX FIFO 数据口的地址别名。
一项 FIFO 项代表一个 `DFS+1` 位的帧，写入与读取都必须按 frame mask 截断。

### 3. 四种 TMOD 到底差在哪里

| TMOD | TX FIFO 的含义 | RX FIFO 的含义 | 谁决定数据帧数 |
|---|---|---|---|
| TR | 有效发送数据 | 每次交换的返回值有效 | TX FIFO 项数 |
| TO | 有效发送数据 | 线路返回值丢弃 | TX FIFO 项数 |
| RO | 一个 dummy word 只用于启动 | 自动接收的数据有效 | `CTRLR1.NDF + 1` |
| EEPROM_READ | opcode/address 等控制帧 | 控制帧 RX 无效；数据阶段 RX 有效 | 控制帧后由 `NDF + 1` 决定 |

SPI 物理上总是全双工交换。TO 的“仅发送”不是总线不返回比特，而是控制器不把
返回帧放进 RX FIFO。EEPROM_READ 的 command 阶段也会有线路返回值，但它们不是
Guest 请求的 Flash 数据，因此必须丢弃，否则 RX 数据会整体错位。

### 4. NDF、dummy 和 `remaining_frames`

`CTRLR1.NDF` 编码的是“接收帧数减一”，所以：

```text
NDF = 0      → 1 帧
NDF = 3      → 4 帧
NDF = 0xffff → 65536 帧
```

计算必须使用 `uint32_t`，否则 `0xffff + 1` 会在 16 位类型中溢出为 0。

RO 需要 Guest 向 DR 写入**恰好一个** dummy word 来触发事务。模型在启动时只
`pop` 这个 FIFO 项，不对它额外调用 `send_frame()`；随后按 `NDF+1` 次调用
`send_frame(0)` 来产生时钟并接收数据。若对启动 dummy 也做一次交换，就会错误地
收到 `NDF+2` 帧。

这里的 dummy word 不是增强 SPI 的 dummy cycle：前者是 RO 模式下写入 TX FIFO
的启动帧；后者由 Patch 5 的 `SPI_CTRLR0.WAIT_CYCLES` 描述，单位和事务阶段都不同。

`remaining_frames` 用来记录未生成的 RO/EEPROM 数据帧数。RX FIFO 满后，pump 必须
结束本次调用，保留 `phase + remaining_frames`；Guest 读 DR 腾出空间后再调用 pump。
这不是为了模拟异步时间，而是为了在多次 MMIO 访问之间保持硬件可观察的事务状态。

### 5. 为什么需要 phase，而不需要独立 busy

Patch 3 的跨 MMIO 状态为：

```text
IDLE
RX_ONLY
EEPROM_COMMAND
EEPROM_DATA
```

`phase` 区分 EEPROM 的“命令尚未发完”和“开始自动读数据”两个阶段；只保存
`remaining_frames` 无法表达这个区别。RO/EEPROM 在数据阶段被 RX FIFO 容量打断时，
也必须靠这两个字段从正确位置继续。

不保存独立 `bool busy`，避免 `busy` 与 `phase` 忘记同步。当前同步模型中：

```text
SR.BUSY = (phase != IDLE)
```

因此，RX FIFO 中仍有未读数据并不表示 BUSY；只要线路数据已经全部生成，phase
回到 IDLE，BUSY 就应为 0。`pio/dynamic-status` 覆盖了 RX FIFO 满、读 DR 恢复、
最后一帧完成和 FIFO 读空的连续状态变化。

SDK 同时说明，真实硬件若软件不能及时取走 RX FIFO，可能发生 RX overflow 并丢失
后续输入。Patch 3 只建立 FIFO/phase 路径；RXO 锁存、RISR 和 IRQ 语义由 Patch 6
补齐，不能提前塞进 TMOD 分支。

### 6. 其他 QEMU SPI 模型能借鉴什么

普通 FIFO SPI 控制器通常非常朴素：

```text
pop TX → ssi_transfer(tx) → push RX
```

可参考：

- `hw/ssi/sifive_spi.c`、`hw/ssi/pl022.c`、`hw/ssi/xilinx_spi.c`：普通逐帧交换；
- `hw/ssi/ibex_spi_host.c`：以 command length 保存剩余传输数量；
- `hw/ssi/xlnx-versal-ospi.c`：读 Flash 时向 TX FIFO 填 0 产生读时钟；
- `hw/ssi/xilinx_spips.c`：保存 command/address/dummy 的阶段状态。

这些模型不是 K230 寄存器语义的直接模板。普通控制器多数要求驱动自己写 N 个
dummy byte；DWC SSI 的 RO/EEPROM_READ 使用 NDF 让硬件自动进行数据阶段。因此，
Patch 3 借鉴的是“阶段机与 FIFO pump”的结构，而不是复制其他控制器的寄存器行为。

当前 QEMU 树中没有可直接照抄的成熟 DWC APB SSI 外设模型；这正是 K230 的
`phase + remaining_frames` 看起来比普通 SPI 模型多一些状态的原因。

### 7. 只实现 TR/TO 是否足够支持 Flash

从 SPI 协议角度，TR 可以读 Flash：驱动发 opcode/address 后，再显式写 N 个
`0x00` 或 `0xff` dummy 帧，交换得到 N 个 RX 帧；TO 可以用于 WREN、page program、
erase 和状态寄存器等发送型操作。

但**当前 K230 软件栈不是这样使用控制器的**：

```text
K230 U-Boot Standard read → EEPROM_READ + NDF
K230 U-Boot Dual/Quad read → RO + NDF
K230 Linux spi-mem Data-IN → EEPROM_READ + NDF
```

U-Boot 的直接证据是 `designware_spi.c:dw_spi_exec_op()`：Standard read 选择
`CTRLR0_TMOD_EPROMREAD`，其他读选择 `CTRLR0_TMOD_RO`，并将读取长度写入
`CTRLR1`。它只写命令、地址和协议 dummy，不再为数据阶段手工写 N 个 dummy。

所以只实现 TR/TO 并非“所有 Flash 协议都不可能”，但会让未经修改的 K230
U-Boot/Linux 常规 Flash read 等不到正确 RX 数据，从而导致 probe、读数据或启动
失败。RO/EEPROM_READ 是 DWC SSI 的硬件能力，不是 SDK 随意增加的私有协议。

这也解释了依赖关系：

```text
Patch 3 的 RO/EEPROM_READ/NDF
    → Patch 4 Standard Flash read
    → Patch 5 Dual/Quad QSPI read
    → Patch 9 XIP read window
```

TR/TO 不实现 QSPI 线宽与阶段，因此不能替代 Patch 5；没有 Patch 5 的增强读路径，
也不能正确实现 XIP。Patch 3 本身不产生 XIP 映射，所以它不会单独改变现有 XIP 行为。

### 8. transfer pump 的触发点与结束条件

在 Patch 3 中，调用 pump 的地方有三个：

```text
DR 写入：新数据可能让 TR/TO/RO/EEPROM 开始或继续。
SER 写入：此前在 SER=0 预填的数据首次获得线路条件。
DR 读取：RX FIFO 腾出空间，RO/EEPROM 的暂停数据阶段可能恢复。
```

`SSIENR` 从 0 写到 1 也需要检查是否已存在可启动的数据；从 1 写到 0 则是 abort：
撤销 CS、清 TX/RX FIFO、清 phase 和 remaining_frames。禁止只清 FIFO 而遗留 phase，
否则下次 enable 会出现“幽灵事务”。

多个 SER 位同时有效时，当前 profile 不支持广播片选，应拒绝并撤销活动 CS，不能
静默选择最低有效位。

### 9. 嵌套 switch 是否符合 QEMU 风格

外层 `switch (tmod)`，内层 `switch (phase)` 是合理的状态机分派；QEMU 没有禁止
嵌套 switch。当前状态数很少，使用 switch 比状态表或函数指针更直接。

需要注意的不是“嵌套”本身，而是：

- 有意从 `IDLE` 落入下一 phase 时，最终整理版应使用 `QEMU_FALLTHROUGH;` 或
  清晰的 fallthrough 注释；
- phase switch 应有 `default: g_assert_not_reached();`，避免未知迁移状态被静默忽略；
- RO 与 EEPROM_DATA 的“按 remaining_frames 接收”循环未来可抽为小 helper，减少
  重复；不应为此过度设计成复杂框架。

这些是可读性整理，不改变 Patch 3 的硬件行为。学习版保留中文 `LEARNING(P3-*)`
注释是允许的；已完成事项不再保留 `TODO(P3-*)`，以免误导后续阅读。

### 10. Patch 3 的验证与进入下一阶段

Patch 3 的 PIO 测试现在包括 10 项：DR alias、disabled DR、FIFO 深度、DFS mask、
TR、TO、RO、EEPROM_READ、disable cleanup、dynamic status。`git diff --check` 应持续
通过。

本次环境的构建验证被 Meson 版本阻塞：项目要求 Meson `>= 1.5.0`，环境仅有 1.3.2。
切换到满足版本的 Meson 后，重新配置、编译并运行：

```bash
meson setup --reconfigure build
ninja -C build qemu-system-riscv64 tests/qtest/k230-dw-ssi-test
QTEST_QEMU_BINARY=./build/qemu-system-riscv64 \
    ./build/tests/qtest/k230-dw-ssi-test -p /riscv64/k230-dw-ssi/pio
```

运行结果应为 10/10 PASS。完成该环境验证后，Patch 3 可正式封口；Patch 4 只应在
既有 FIFO/TMOD 路径上接入 `spi0` 的 W25Q256/MTD，不应重新实现一条绕过 Patch 3 的
字节直传路径。
