# K230 SSI Patch 6：控制器内部 IRQ 学习工作簿

更新时间：2026-07-18

当前状态：Patch 5 的 Standard/Enhanced 数据路径已经稳定，Patch 6 的寄存器、
状态字段、九路输出和内部 qtest 入口开始搭建学习脚手架；IRQ 事件生成仍保持
RED，等待按 P6-1 → P6-2 → P6-3 的顺序完成。本文只讨论控制器内部中断，
不包含 PLIC source ID 和 K230 machine 接线。

## 目标

Patch 6 在已经验证的 FIFO、TMOD、Standard Flash 和 Dual/Quad QSPI 路径上，
加入 K230 DWC SSI 的内部中断模型：

```text
FIFO 水位变化 / 错误事件
          |
          v
dynamic status + latched status
          |
          +---------------------> RISR
          |
          v
      & IMR & 0x9bf
          |
          +---------------------> ISR
          |
          v
TXE/TXO/RXF/RXO/TXU/RXU/MST/DONE/AXIE 九路输出
```

建议提交标题：

```text
hw/ssi: Implement K230 SSI interrupt handling
```

Patch 6 完成后的累计门禁为：

| 分组 | 数量 | 累计 |
|---|---:|---:|
| reg | 8 | 8 |
| pio | 10 | 18 |
| flash | 7 | 25 |
| qspi | 6 | 31 |
| irq/internal | 6 | 37 |

## 本 Patch 做与不做

必须完成：

- 暴露九路独立 SysBus IRQ 输出。
- 正确计算 `RISR` 和 `ISR`。
- TXE/RXF 使用 FIFO 水位动态计算，不保存成陈旧寄存器值。
- TXO/RXO/TXU/RXU/MST 使用锁存事件状态。
- 正确实现 IMR mask 和 RC 清除寄存器。
- 所有会改变中断结果的路径都调用统一的 IRQ 更新 helper。
- reset、disable、迁移恢复后输出线与内部状态一致。
- DONE/AXIE 输出存在，但内部 DMA 未实现时保持 0。

本 Patch 不实现：

- PLIC 146–172 接线。
- K230 machine 中的 IRQ base 映射。
- 27 路 PLIC routing qtest。
- IDMA 搬运、DONE/AXIE 事件来源。
- XIP FIFO overflow、SPITE dynamic wait 和多主竞争行为。
- 聚合 IRQ 或 OR gate。

控制器源码中不允许出现 `146`、`155`、`164` 等 PLIC source ID。控制器只知道
自己有九路输出；输出接到哪里属于 Patch 7 的 SoC 装配责任。

## 已搭好的学习脚手架

生产代码目标位置：

```text
include/hw/ssi/k230_dw_ssi.h
    K230DwSsiIrq
    K230_DW_SSI_IRQ_COUNT
    irqs[]
    irq_latched

hw/ssi/k230_dw_ssi.c
    P6-1  raw status：动态水位 + 锁存状态
    P6-2  事件挂点与 RC clear
    P6-3  masked status 与九路输出更新
```

测试按责任拆分：

```text
tests/qtest/k230-dw-ssi-irq-test.c
    I01–I06：Patch 6 内部寄存器/事件语义

tests/qtest/k230-dw-ssi-plic-test.c
    I07–I11：Patch 7 PLIC routing，Patch 6 阶段不注册
```

定位脚手架：

```bash
rg -n "TODO\(P6|LEARNING\(P6" \
    "hw/ssi" "include/hw/ssi" "tests/qtest"
```

## 1. 三层状态必须分开

IRQ 不能只保存一个 `isr` 字段。必须区分：

| 层次 | 含义 | 是否受 IMR 影响 |
|---|---|---|
| 原始原因 | FIFO 水位或锁存错误事件 | 否 |
| `RISR` | 当前原始中断状态 | 否 |
| `ISR` | `RISR & IMR & valid_mask` | 是 |
| 输出线 | 九个 ISR 原因分别驱动九根线 | 是 |

公式固定为：

```c
risr = dynamic_status | latched_status;
isr = risr & s->regs[R_IMR] & 0x000009bf;
```

`0x9bf` 的有效位是：

```text
bit  0 TXE
bit  1 TXO
bit  2 RXU
bit  3 RXO
bit  4 RXF
bit  5 MST
bit  7 TXU
bit  8 AXIE
bit 11 DONE
```

当前 profile 排除：

- bit 6 XRXO：并发 XIP RX FIFO overflow，Patch 9 之前不产生。
- bit 9：保留位。
- bit 10 SPITE：dynamic wait/SPITECR 条件路径未实现。

不要用 `regs[R_RISR]` 或 `regs[R_ISR]` 保存结果。FIFO 水位随 DR 访问立即变化，
保存值会在下一次读取前变成陈旧状态。

## 2. 外部 IRQ 顺序与状态位号不同

K230 DTS/RT-Smart 使用的九路外部顺序是：

| 输出 index | 名称 | RISR/ISR bit |
|---:|---|---:|
| 0 | TXE | 0 |
| 1 | TXO | 1 |
| 2 | RXF | 4 |
| 3 | RXO | 3 |
| 4 | TXU | 7 |
| 5 | RXU | 2 |
| 6 | MST | 5 |
| 7 | DONE | 11 |
| 8 | AXIE | 8 |

因此不能写成：

```c
qemu_set_irq(s->irqs[i], isr & BIT(i));
```

正确做法是建立显式映射表：

```c
static const uint32_t irq_status_mask[K230_DW_SSI_IRQ_COUNT] = {
    [K230_DW_SSI_IRQ_TXE]  = BIT(0),
    [K230_DW_SSI_IRQ_TXO]  = BIT(1),
    [K230_DW_SSI_IRQ_RXF]  = BIT(4),
    [K230_DW_SSI_IRQ_RXO]  = BIT(3),
    [K230_DW_SSI_IRQ_TXU]  = BIT(7),
    [K230_DW_SSI_IRQ_RXU]  = BIT(2),
    [K230_DW_SSI_IRQ_MST]  = BIT(5),
    [K230_DW_SSI_IRQ_DONE] = BIT(11),
    [K230_DW_SSI_IRQ_AXIE] = BIT(8),
};
```

这个映射只描述控制器输出 index 与内部状态位，不描述 PLIC 编号。

## 3. P6-1：动态水位事件

### 3.1 TXE

TXE 表示 TX FIFO level 小于等于 `TXFTLR.TFT`：

```c
txe = fifo32_num_used(&s->tx_fifo) <=
      FIELD_EX32(s->regs[R_TXFTLR], TXFTLR, TFT);
```

复位时 `TFT=0`、TX FIFO 为空，因此 `RISR.TXE=1`。IMR 复位值包含 TXE，
所以对应输出线也会拉高。这个结果不依赖 `SSIENR`；关闭控制器会清 FIFO，
从而重新满足 TXE 水位。

TXE 是动态条件，不允许锁存在 `irq_latched` 中，也不由 `TXEICR` 清除。
FIFO level 超过 threshold 后必须自动撤销；再次降到 threshold 以下时自动恢复。

### 3.2 RXF

RXF 表示 RX FIFO level 严格大于 `RXFTLR.RFT`：

```c
rxf = fifo32_num_used(&s->rx_fifo) >
      FIELD_EX32(s->regs[R_RXFTLR], RXFTLR, RFT);
```

例如 `RFT=0` 时，第一个 RX frame 进入 FIFO 就产生 RXF；Guest 读走最后一个
frame 后 RXF 自动撤销。RXF 同样不是 RC 锁存事件。

### 3.3 状态变化与 IRQ 更新边界

只要 FIFO level、threshold、IMR 或锁存事件发生变化，逻辑上的中断结果就已经变化；
但当前 `run_transfer()` 会在同一次 MMIO 回调内同步执行多次 FIFO pop/push，Guest
不能观察这些循环中间状态。因此不需要在 pump 内每处理一个 frame 就调用
`update_irq()`，应当在 pump 到达稳定的完成或阻塞点后统一更新一次。

```text
DR 写入 / SER 改变 / SSIENR enable
          |
          v
    run_transfer()
      多次 TX pop
      多次 RX push
      可能设置错误 latch
          |
          v
    update_irq() 一次
```

这样可以避免在一次同步 MMIO 回调中产生 Guest 本不应观察到的短暂 IRQ 电平，并且
仍能保证回调返回时 `RISR`、`ISR` 和九路输出与稳定 FIFO 状态一致。

推荐更新时机：

| 时机 | 更新原因 |
|---|---|
| system reset 完成 | latch、FIFO、threshold 和 IMR 恢复复位状态 |
| `run_transfer()` 返回 | TX/RX FIFO 可能经过多次 pop/push，或设置 RXO |
| Guest 向满 TX FIFO 写 DR | TXO 立即锁存，本次写入不进入 pump |
| Guest 从空 RX FIFO 读 DR | RXU 立即锁存；若随后 pump 恢复接收，在 pump 后统一更新 |
| 写 `TXFTLR` / `RXFTLR` | 水位条件可能立即改变 |
| 写 `IMR` | `RISR` 不变，但 `ISR` 和输出可能立即改变 |
| 读取 RC clear 寄存器 | latch 被清除，输出必须立即重算 |
| `SSIENR` disable/abort | FIFO、phase 和 CS 状态变化，动态水位需要重算 |
| 迁移恢复 | 依据迁移后的 FIFO、IMR、threshold 和 latch 重建输出电平 |

不要把 `update_irq()` 复制到四种 TMOD 分支的每一处，也不要让普通 FIFO helper
每次 pop/push 都直接驱动 IRQ。优先让 helper 只修改状态，由 MMIO 入口、pump 出口、
RC clear、reset/abort 等少量稳定边界统一更新。

## 4. P6-2：锁存错误事件

Patch 6 的锁存状态建议使用一个 `uint32_t irq_latched`，位号与 RISR 一致。
不要为每个事件增加一个独立 `bool`，否则 clear mask、VMState 和 raw status 会出现
重复代码。

| 事件 | 当前可达挂点 | 行为 |
|---|---|---|
| TXO | Guest 向已满 TX FIFO 写 DR | 忽略本次写，锁存 TXO |
| RXO | 新接收 frame 到达时 RX FIFO 已满 | 丢弃新 frame，锁存 RXO |
| RXU | Guest 从空 RX FIFO 读 DR | 返回 0，锁存 RXU |
| TXU | 当前 PIO 模型无可靠硬件原因 | 保持 0 |
| MST | 固定 master profile 无多主竞争 | 保持 0 |
| DONE | 无内部 DMA | 保持 0 |
| AXIE | 无内部 DMA | 保持 0 |

### 4.1 TXO 测试为什么必须保持 SER=0

当前 pump 在一次 MMIO DR 写入中同步消费可发送的 TX frame。如果测试先选择 CS，
TR/TO 数据可能每写一项就立即发走，永远填不满 TX FIFO。

稳定的 TXO 触发方式：

```text
SSIENR=1
SER=0
写满 256 个 TX FIFO item
再写第 257 个 item
```

SER=0 允许预填 FIFO，但 pump 不会发送；第 257 次写入才是确定的 overflow。

### 4.2 RXO 与背压的区别

RO/Enhanced RO 在 RX FIFO 满时主动暂停生成 dummy clock，这是正常背压，不是 RXO。
RXO 只在模型已经得到一个新的 RX frame、但 FIFO 无空间保存时产生。当前最直接的
可达路径是 TR loopback：Guest 在 RX FIFO 已满后继续发送一个 frame。

不要为了制造 RXO，让 RO pump 在 FIFO 已满时继续无限生成 dummy；那会破坏
Patch 3/5 已验证的暂停恢复语义。

### 4.3 TXE 与 TXU 不能混淆

TXE 是正常流控：TX FIFO level 小于等于 threshold，提示 Guest 可以继续填数据。
TXU 是错误事件：控制器已经知道当前事务仍然需要发送普通 TX frame，但此时取不到
数据。正常发送结束后 TX FIFO 为空只产生 TXE，不能顺便锁存 TXU。

当前 Standard TR/TO 没有独立的预期发送总帧数，TX FIFO 耗尽通常只表示本轮已没有
待发送数据，因此没有可靠条件产生 TXU。Enhanced TO 虽然有 `remaining_frames`，
当前 Patch 仍将 TX FIFO 空视为可恢复的暂停点，不为了覆盖中断而提前改变 Patch 5
事务语义。TXU 输出、状态位和清除路径需要存在，但无可靠来源时保持 0。

## 5. P6-3：RC 清除寄存器

TRM 的读取清除范围固定如下：

| 寄存器 | 读取返回 | 清除范围 |
|---|---|---|
| `TXEICR` | TXO/TXU 是否有任一 active | TXO、TXU |
| `RXOICR` | RXO 是否 active | RXO |
| `RXUICR` | RXU 是否 active | RXU |
| `MSTICR` | MST 是否 active | MST |
| `ICR` | 四类错误是否有任一 active | TXO、RXU、RXO、MST |
| `AXIECR` | AXIE 是否 active | AXIE |
| `DONECR` | DONE 是否 active | DONE |

共同规则：

- 读返回 bit 0，其他位为 0。
- 先取旧状态作为返回值，再清除 latch。
- 清除后立即调用 `update_irq()`。
- 对这些寄存器写入没有效果。
- 读取 `ISR/RISR` 不清除任何事件。
- `ICR` 不清 TXE、RXF、TXU、DONE 或 AXIE。
- 当前无 IDMA/AXI 事件来源，因此 `AXIECR`、`DONECR` 读取返回 0，但仍保留正确的
  RC 接口，写入同样无效果。

动态 TXE/RXF 不能通过 RC 寄存器清除；只要水位条件仍成立，输出就必须保持 active。

## 6. 九路输出更新

推荐只保留一个更新函数：

```c
static void k230_dw_ssi_update_irq(K230DwSsiState *s)
{
    uint32_t status = k230_dw_ssi_irq_raw_status(s) &
                      s->regs[R_IMR] & K230_DW_SSI_IRQ_VALID_MASK;

    for (int i = 0; i < K230_DW_SSI_IRQ_COUNT; i++) {
        qemu_set_irq(s->irqs[i], !!(status & irq_status_mask[i]));
    }
}
```

不要生成一个聚合 `irq`。K230 对外就是九路独立信号，Patch 7 需要把每一路连接到
不同的 PLIC source。

IMR 只影响 `ISR` 和外部输出，不影响 `RISR`。因此：

```text
事件 active + IMR=0
    RISR=1
    ISR=0
    output=0

事件 active + IMR=1
    RISR=1
    ISR=1
    output=1
```

## 7. reset、disable 与迁移

reset 必须：

- 清除 `irq_latched`。
- 恢复 `IMR=0x3f`。
- 清空 TX/RX FIFO。
- 按空 FIFO 水位重新计算 TXE/RXF。
- 更新九路输出。

`SSIENR 1→0` 已经负责清 FIFO、终止 phase 和撤销 CS。Patch 6 不应复制一套禁用
逻辑，只需在统一 abort 完成后重新计算 IRQ。锁存错误是否随 disable 清除必须保持
统一契约；当前学习方案选择由 reset/RC 清除锁存错误，disable 只通过 FIFO 变化影响
动态事件，避免 Guest 通过一次 disable 无意吞掉尚未读取的错误原因。

迁移状态保存：

- `irq_latched`。
- 已有 `regs[]` 中的 IMR/threshold。
- 已有 TX/RX FIFO。

`qemu_irq` 线路句柄本身不能进入 VMState。迁移恢复后根据上述状态调用
`update_irq()` 重建线路电平。

## 8. 从 Guest 到控制器的中断流程

Patch 6 只产生控制器内部九路输出。Guest 真正进入 Linux IRQ handler，还需要
Patch 7 把这些输出逐根接到 PLIC：

```text
FIFO 水位 / 错误 latch
        |
        v
RISR -> IMR -> ISR
        |
        v
九路 SSI IRQ 输出             Patch 6
        |
        v
PLIC source 146--172          Patch 7
        |
        v
RISC-V external interrupt
        |
        v
Linux PLIC irqchip claim
        |
        v
K230 DW SSI irq handler
```

PLIC 的 claim/complete 只表示 CPU 已经处理该 PLIC source，并不会替外围设备清除
原因。只要 SSI 的电平型输出仍然为高，PLIC complete 后仍可能再次 pending。真正让
输出下降的是：

- TXE：Guest 继续填 TX FIFO，或驱动 mask TXE。
- RXF：Guest 读取 RX FIFO，使 level 低于 threshold。
- TXO/RXO/RXU/TXU/MST：驱动读取对应 RC clear 寄存器，或 mask 对应原因。
- reset：清 latch，并根据复位后的 FIFO 水位重新计算动态原因。

### 8.1 spi0 的外部顺序

有效 DTS 中 spi0 的九个 PLIC source 应按控制器输出顺序排列：

| 名称 | spi0 source |
|---|---:|
| TXE | 146 |
| TXO | 147 |
| RXF | 148 |
| RXO | 149 |
| TXU | 150 |
| RXU | 151 |
| MST | 152 |
| DONE | 153 |
| AXIE | 154 |

spi1、spi2 分别使用 155--163 和 164--172。这里用于解释 Guest 路径；这些数字不得
进入控制器设备代码。

### 8.2 Linux PIO handler 的基本行为

标准中断处理流程是：

1. handler 读取 `ISR`，获得已经经过 IMR 的 active 原因。
2. 每次进入 handler 都先读取 RX FIFO，防止接收数据积压。
3. 如果 `ISR.TXE=1`，继续向 DR 填 TX FIFO。
4. 所有待发送数据已经填完后 mask TXE，避免空 FIFO 引起持续中断。
5. 所有 RX 数据取完后 mask 中断，并调用 SPI core 完成本次 transfer。

增强 handler 会根据 RXF 读取 RX FIFO，根据 TXE 继续发送；DONE 只属于内部 DMA
完成路径。Patch 6 没有 IDMA 状态机，所以 Guest 不应依赖 DONE 完成普通 PIO 请求。

### 8.3 当前 SDK 集成注意点

当前 SDK 驱动与 DTS 之间还有几处独立于 Patch 6 控制器算法的集成条件：

- 驱动通过名字获取 `spi_txe`、`spi_rxf`、`spi_done` 三条 IRQ，但当前基础
  `k230.dtsi` 的 SSI 节点只有九个 `interrupts`，没有对应的 `interrupt-names`。
- 驱动只为 TXE、RXF、DONE 注册 handler，却会在部分配置中 unmask TXO、RXU、
  RXO。九路信号不能在 QEMU 控制器内 OR 聚合，因此 Guest 驱动需要为所有可能
  unmask 的独立 source 建立一致的处理方式。
- 当前普通 PIO transfer 路径中的 `dw_spi_irq_setup(...dw_spi_transfer_handler)` 被
  注释，代码会先 mask 中断再直接填 TX FIFO。因此普通 PIO 是否实际依赖 TXE IRQ，
  需要以最终启用的驱动和 DTB 为准。

这些问题可能导致 Guest 在 probe 或传输流程中没有进入预期 handler，但不能通过
修改 Patch 6 的 `RISR/ISR` 语义、错误地 OR 九路输出或伪造 DONE 来绕过。

## 9. 六项内部 qtest

| 编号 | 用例 | 证明范围 |
|---|---|---|
| I01 | `irq/watermark-mask` | TXE/RXF 动态水位、IMR 与 RISR/ISR 关系 |
| I02 | `irq/rxu-read-clear` | 空读锁存 RXU，RXUICR 读清除 |
| I03 | `irq/txo-read-clear` | SER=0 填满 TX FIFO，TXEICR 清 TXO |
| I04 | `irq/rxo-read-clear` | TR loopback 溢出 RX FIFO，RXOICR 清除 |
| I05 | `irq/icr-clear-scope` | ICR 只清 TRM 指定的四类错误，不清动态 TXE |
| I06 | `irq/inactive-causes` | MST/TXU/DONE/AXIE 无来源时保持 0 |

Patch 6 阶段不注册：

```text
irq/plic-txe-reset-routing
irq/plic-rxu-isolation
irq/plic-rxf-routing
irq/plic-txo-routing
irq/plic-rxo-routing
```

这些用例属于 Patch 7。否则 Patch 6 的控制器内部实现即使完全正确，也会因为 machine
尚未接线而保持 RED，破坏单 Patch 门禁。

## 10. 推荐实现顺序

### P6-1：状态读取

1. 完成 TXE/RXF 动态计算。
2. `RISR` 返回 raw status。
3. `ISR` 返回 raw & IMR & valid mask。
4. 先让 I01 从 RED 变 GREEN。

### P6-2：错误锁存与 RC

1. 在 TX full、RX empty read、RX full receive 三个稳定挂点设置 latch。
2. 完成核心错误 RC read helper，并保留 AXIECR/DONECR 的无事件 RC 接口。
3. 依次通过 I02、I03、I04、I05、I06。

### P6-3：外部输出与生命周期

1. 初始化九路 SysBus IRQ。
2. 建立显式 status-bit mapping。
3. 在所有状态变化入口调用统一 update helper。
4. 完成 reset 和迁移恢复线路重算。
5. 运行 reg/pio/flash/qspi 全部回归。

## 11. 验证命令

重新配置过的 build 目录中运行：

```bash
ninja -C "build" \
    "qemu-system-riscv64" \
    "tests/qtest/k230-dw-ssi-test"

QTEST_QEMU_BINARY="./build/qemu-system-riscv64" \
    ./build/tests/qtest/k230-dw-ssi-test \
    -p /riscv64/k230-dw-ssi/irq
```

逐项调试：

```bash
bash "tests/qtest/run-k230-dw-ssi-tests.sh" \
    "irq/watermark-mask"
```

回归门禁：

```text
reg       8/8
pio      10/10
flash     7/7
qspi      6/6
irq       6/6
累计     37/37
```

## 12. 完成标准

- [ ] P6-1 动态水位和 RISR/ISR 完成。
- [ ] P6-2 锁存事件和 RC 清除完成。
- [ ] P6-3 九路输出、reset 和迁移恢复完成。
- [ ] I01–I06 全部通过。
- [ ] Patch 2–5 的 31 项测试持续通过。
- [ ] 累计 37/37 PASS，无 TIMEOUT。
- [ ] 控制器源码没有 PLIC source ID。
- [ ] 没有聚合 IRQ 或 OR gate。
- [ ] DONE/AXIE 在无 IDMA 时保持 0。
- [ ] `ISR/RISR` 读取无清除副作用。
- [ ] RC 寄存器写入无效果。

达到以上条件后，Patch 6 才算完成，可以进入 Patch 7 的 27 路 PLIC 接线。

## 13. 附录：脱离 Patch 边界的统一推进状态机

本节不是 Patch 6 的合入条件，而是从完整控制器角度记录 Standard SPI、Enhanced
SPI/QSPI、后续 DMA 和 XIP 如何共用一套传输引擎。

### 13.1 当前模型属于哪一种

当前实现使用同步 `ssi_transfer()`，没有 timer/BH，但不同 TMOD 的推进边界不同：

| 路径 | 当前推进方式 |
|---|---|
| Standard TR | 清空当前 TX FIFO；RX 满时继续发送、丢弃新 RX 并锁存 RXO |
| Standard TO | 清空当前 TX FIFO，忽略线路返回值 |
| Standard RO | 运行到 RX FIFO 满或 `remaining_frames=0` |
| EEPROM_READ | 先清空命令 TX，再运行到 RX 满或计数完成 |
| Enhanced RO | instruction/address/mode/dummy 后运行到 RX 满或计数完成 |
| Enhanced TO | 运行到 TX FIFO 空或计数完成 |

因此当前已经是“同步 + 部分 run-until-blocked”的混合模型，而不是必须推倒重写的
单纯 FIFO flush 实现。

### 13.2 统一的含义

“统一”不是把所有逻辑塞进一个巨大 switch，而是让所有模式服从相同生命周期：

```text
prepare -> start -> run -> blocked/complete
                       ^       |
                       |       |
                 DR 访问恢复    +-> abort/reset
```

- `prepare`：根据 CTRLR0、CTRLR1、SPI_CTRLR0 和数据来源构造事务描述。
- `start`：确认 SSIENR、CS、格式和启动数据满足要求。
- `run`：按 phase 推进 instruction/address/mode/dummy/data。
- `blocked`：等待 TX 数据或 RX 空间，保留 phase 和 remaining count。
- `complete`：事务计数归零，回到 IDLE；需要完成中断时由对应前端产生。
- `abort`：disable/reset 时统一撤销 CS、FIFO、phase 和 descriptor。

### 13.3 建议显式表达退出原因

当前 `run_transfer()` 返回 `void`，调用者无法区分正常静止、等待数据、接收背压和
真正完成。完整模型可以在内部使用类似结果：

```c
typedef enum K230DwSsiRunResult {
    K230_DW_SSI_PROGRESS,
    K230_DW_SSI_BLOCK_DISABLED,
    K230_DW_SSI_BLOCK_NO_CS,
    K230_DW_SSI_BLOCK_NEED_TX,
    K230_DW_SSI_BLOCK_RX_FULL,
    K230_DW_SSI_COMPLETE,
    K230_DW_SSI_UNSUPPORTED,
} K230DwSsiRunResult;
```

这使模型能够回答：TX FIFO 空是正常静止、等待 Guest 继续填充，还是已知仍有剩余
发送帧而发生真正 TXU。Standard TR/TO 没有已知总发送长度，不能只因 TX FIFO 空
就报告 TXU；有明确剩余计数的事务才有条件进一步定义 underflow。

### 13.4 单步推进与运行到阻塞点

可以把协议推进拆成“一个可解释步骤”和“循环到阻塞”：

```c
static K230DwSsiRunResult k230_dw_ssi_step(K230DwSsiState *s);

static K230DwSsiRunResult
k230_dw_ssi_run_until_blocked(K230DwSsiState *s)
{
    K230DwSsiRunResult result;

    do {
        result = k230_dw_ssi_step(s);
    } while (result == K230_DW_SSI_PROGRESS);

    k230_dw_ssi_update_irq(s);
    return result;
}
```

`step()` 可以推进一个 phase 或一个 data frame；外层仍在一次 MMIO 回调中同步运行，
因此不要求引入 timer。只有在 timer/BH 的两次回调之间 Guest 能够执行时，才需要
在每个可观察批次后更新 IRQ。

### 13.5 各模式的阻塞和完成条件

| 模式 | 阻塞点 | 完成条件 |
|---|---|---|
| Standard TR/TO | 当前 TX FIFO 空 | 没有内部总长度；由 Guest/CS 控制事务边界 |
| Standard RO | RX FIFO 满 | `remaining_frames=0` |
| EEPROM data | RX FIFO 满 | `remaining_frames=0` |
| Enhanced RO | RX FIFO 满 | `remaining_frames=0` |
| Enhanced TO | TX FIFO 空 | `remaining_frames=0` |

Standard TR 在 RX FIFO 满时可以继续线路发送并锁存 RXO，因此 RX 满不一定是所有
模式的公共阻塞点。阻塞策略必须按 TMOD 和硬件语义决定。

### 13.6 集中 FIFO 事件语义

完整模型可以通过少量 helper 统一 FIFO 状态修改：

```text
tx_push_from_guest()       满写时锁存 TXO
tx_pop_for_transfer()      传输引擎消费 TX frame
rx_push_from_transfer()    无空间时锁存 RXO
rx_pop_for_guest()         空读时锁存 RXU
```

这些 helper 只修改 FIFO 和 latch，不必在每次调用时驱动 IRQ。MMIO 入口、pump 出口、
RC clear、reset/abort 负责在稳定边界统一调用 `update_irq()`。

### 13.7 PIO、DMA 与 XIP 共用协议引擎

长期结构可以让不同前端生成同一种事务描述：

```text
PIO DR 写入 ----+
DMA FIFO 搬运 --+--> transaction descriptor --> phase engine --> SSI bus
XIP window 访问 -+
```

- PIO 通过 DR/FIFO 提供和接收数据。
- DMA 负责内存与 FIFO 之间搬运，并在计数完成或 AXI 出错时产生 DONE/AXIE。
- XIP 根据窗口地址生成 instruction/address/mode/dummy/data 描述。
- Standard/Enhanced 的 instruction、address、dummy 和 data 执行逻辑只有一套。

只有在必须让 `BAUDR` 影响传输时间、让 `SR.BUSY` 持续可观察、模拟 CPU 与 SPI
serializer 竞争或精确产生 TXU 时，才需要进一步引入 timer/BH。功能级主线模型可以
继续使用同步 `ssi_transfer()` 和 run-until-blocked，保持确定性和较低复杂度。
