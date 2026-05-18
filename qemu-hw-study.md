# QEMU 外设建模流程学习资料

## 来源

- 讲义：<https://qemu.gevico.online/tutorial/2026/ch2/qemu-hw/>
- 视频：<https://www.bilibili.com/video/BV1Ae9wBjEiD/>
- 视频标题校验：`QEMU 训练营 | 专业阶段 QEMU 外设建模流程 [SOC 建模]`
- 作者 / UP：`绝对是泽文啦`
- 讲义作者：`@zevorn`
- 本地处理：已抓取讲义正文，已下载视频音频并用本地 `faster-whisper` 转写，视频时长约 `11:51`

⚠️ 容易踩坑：你计划里写的 `BV1p99cBCEzV` 实际对应“专业阶段 QEMU 时钟系统 [SOC 建模]”，不是本章。这里已校正为 `BV1Ae9wBjEiD`，后面“时钟系统”正好接到下一章学习。

本章没有放 PPT 截图。原因是讲义已经给出核心代码与结构，视频重点也是口头解释“建模主线”；普通标题页和过渡页放进来只会让笔记膨胀。

---

## 阅读说明

- `🎥 视频/作者`：讲者在视频里强调的理解方式。
- `📘 讲义`：网页文档里可当手册反复查的稳定内容。
- `🧠 我的理解`：我把概念翻译成你写工程代码时的判断方式。
- `🧩 补充知识`：讲义和视频没完全展开，但做 SoC 设备必须知道的背景。
- `🔗 和 SoC 训练营的关系`：落到 G233 的测试、寄存器、设备模型。
- `✅ 你现在就做`：看完这一节后立刻动手的小任务。
- `⚠️ 容易踩坑`：QEMU 初学和外设建模里最容易写错的地方。

---

## 一句话结论

QEMU 外设建模不是“写几个寄存器读写函数”，而是把一个硬件外设拆成：`QOM 类型`、`设备状态`、`MMIO 寄存器协议`、`IRQ/timer/bus 副作用`、`machine 集成`、`迁移状态` 这条完整链路。

🧠 我的理解：你后面做 SoC 模块时，GPIO、PWM、WDT、SPI 都可以套这条线。先把设备“长什么样”放进 state，再把 guest 的 `readl/writel` 转成 state 变化，最后把 IRQ、timer、下游 flash 这种副作用接出去。

---

## 本章结构

```text
QOM 类型注册
  -> DeviceState / SysBusDevice / 自己的 State 结构体
  -> instance_init: MMIO / IRQ / clock 骨架
  -> class_init: realize / reset / vmstate / properties
  -> MemoryRegionOps: read/write 寄存器协议
  -> FIFO / IRQ / timer / bus side effect
  -> machine: 地址映射、IRQ 连接、设备树暴露
  -> qtest_readl/qtest_writel 从测试打到设备模型
```

🎥 视频/作者：视频开头强调，很多人刚写 QEMU 设备时会把 QOM、MMIO、IRQ、设备树混在一起，不知道从哪开始。本章目的就是建立一条“设备建模主线”。

📘 讲义：讲义用 `PL011` UART 做例子，因为它是典型 `SysBusDevice`，同时包含寄存器、FIFO、IRQ、clock、chardev 和迁移状态。

🧠 我的理解：PL011 不只是串口例子，它是一块“外设建模样板板”。你以后写 GPIO 会砍掉 FIFO 和 chardev，写 PWM/WDT 会强化 timer，写 SPI 会把 FIFO/状态位/片选/下游 flash 做得更复杂。

---

## 1. 外设模型在 QEMU 里是什么

🎥 视频/作者：讲者把主线概括成：注册类型、定义状态、初始化硬件骨架、实现 MMIO 行为、补齐 FIFO 和中断、接入 machine、支持迁移。

📘 讲义：PL011 的实现位于 `hw/char/pl011.c`，状态定义在 `include/hw/char/pl011.h`。它通过 QOM 注册类型，父类是 `TYPE_SYS_BUS_DEVICE`。

🧠 我的理解：QEMU 里的“外设”不是一个裸 C struct，也不是一组散函数。它是一个 QOM 对象。对象类型告诉 QEMU“我是什么设备”，状态结构体保存“我现在处于什么状态”，MMIO 回调定义“guest 读写我时发生什么”。

🧩 补充知识：可以先记三层对象关系。

```text
Object
  -> DeviceState
    -> SysBusDevice
      -> 你的设备状态结构体，比如 G233GPIOState / G233PWMState
```

🧩 补充知识：`SysBusDevice` 适合 SoC 内部平台设备，比如 UART、GPIO、timer、WDT、SPI controller。它通常不是可枚举总线设备，不像 PCIe 设备那样靠 PCI 配置空间发现，而是由 machine 直接创建、映射地址、连接 IRQ。

🔗 和 SoC 训练营的关系：G233 SoC 里的 GPIO、PWM、WDT、SPI 都更像 `SysBusDevice`。测试里用固定 MMIO 地址访问，例如 GPIO `0x10012000`、PWM `0x10015000`、WDT `0x10010000`、SPI `0x10018000`，这说明它们应该由 machine 明确映射到系统地址空间。

✅ 你现在就做：在脑子里把每个外设写成一句话：“这是一个 `SysBusDevice`，它暴露一块 MMIO 区域，里面有一组寄存器，guest 读写这些寄存器会改变设备状态，并可能触发 IRQ/timer/bus 行为。”

⚠️ 容易踩坑：不要以为“写了设备 C 文件”设备就存在了。视频里特别强调，设备代码不等于设备已经接入 machine。没有 `qdev_new`、`sysbus_realize`、`memory_region_add_subregion`、`sysbus_connect_irq`，guest 看不到它。

---

## 2. 设备状态结构体怎么设计

🎥 视频/作者：视频里反复强调一个点：设备建模本质上是状态建模。guest 看到的所有行为，本质都是设备内部状态变化后的结果。

📘 讲义：PL011 的状态结构体里有这些典型成员：`SysBusDevice parent_obj`、`MemoryRegion iomem`、寄存器状态、FIFO 状态、`CharFrontend chr`、多条 `qemu_irq`、`Clock *clk`、迁移相关字段。

🧠 我的理解：设计 state 结构体时，不要先按“函数怎么写”思考，要先按“硬件有哪些状态”思考。

```text
设备身份：parent_obj
MMIO 入口：MemoryRegion iomem
寄存器影子：ctrl/status/enable/pending/load/value 等字段
内部队列：FIFO、buffer、读写指针
外部连线：qemu_irq、Clock、下游设备指针
时间行为：QEMUTimer、周期、deadline、计数值
迁移状态：哪些字段需要保存和恢复
```

🧩 补充知识：并不是每个寄存器都必须有一个同名字段。有些寄存器是“真实存储”，比如 `GPIO_DIR`、`SPI_CR1`；有些寄存器是“派生视图”，比如 `GPIO_IN` 可以由 `DIR/OUT/外部输入` 计算出来，`WDT_VAL` 可以由 timer 和 load 计算出来。

🔗 和 SoC 训练营的关系：你的 G233 设备 state 可以这样分。

| 设备 | state 里必须先想清楚的东西 |
| --- | --- |
| GPIO | `dir/out/in/ie/is/trig/pol`，上一拍输入值，IRQ 线 |
| PWM | 全局寄存器、4 个 channel 的 `ctrl/period/duty/cnt/done`，timer |
| WDT | `ctrl/load/val/status/locked`，feed key，timeout timer，IRQ 线 |
| SPI | `cr1/cr2/sr/dr`，RX/TX 状态，当前 CS，flash 子设备，overrun 状态，IRQ 线 |

✅ 你现在就做：先不要写 read/write。拿一张纸只写 state 字段，并标注每个字段属于“寄存器值”“派生状态”“外部连线”“timer 状态”中的哪一类。

⚠️ 容易踩坑：状态字段不清楚，后面 read/write 一定会变成一坨 if/switch。尤其是中断，不能只用一个 `bool irq_pending` 糊过去，要分清“中断原因”“中断使能”“最终 IRQ 线电平”。

---

## 3. MMIO 和 MemoryRegionOps

🎥 视频/作者：视频里有一句特别适合记住：MMIO 表面上像访存，本质上是设备协议。guest 的 load/store 不是真的读写 RAM，而是在触发设备行为。

📘 讲义：PL011 通过 `MemoryRegionOps` 暴露 read/write 入口，指定小端，访问宽度是 4 字节。`memory_region_init_io` 把回调和设备 state 绑定起来，`sysbus_init_mmio` 把这块区域挂到 sysbus 设备上。

🧠 我的理解：`MemoryRegionOps` 是“地址空间”和“设备模型”的接口。guest 访问某个物理地址时，QEMU 地址空间查找会发现这段地址属于你的 `MemoryRegion`，然后调用你的 `.read` 或 `.write`。

```text
qtest_writel(qts, GPIO_OUT, 1)
  -> QTest 协议发给 QEMU
  -> QEMU 对 guest 物理地址执行写访问
  -> 地址空间命中 GPIO 的 MemoryRegion
  -> 调用 gpio_write(opaque, offset, value, size)
  -> offset = 0x04，value = 1
  -> 更新 GPIO state，必要时更新 IRQ
```

🧩 补充知识：`opaque` 通常就是你的设备 state。`offset` 是相对 MMIO base 的偏移，不是绝对物理地址。比如 GPIO base 是 `0x10012000`，guest 写 `0x10012004`，回调里通常看到的是 `offset = 0x04`。

🔗 和 SoC 训练营的关系：qtest 里所有 `qtest_readl/qtest_writel` 最终都在逼你实现正确的 `MemoryRegionOps`。如果 `GPIO_OUT` 写了以后 `GPIO_IN` 读不到变化，不是 qtest 神秘，而是你的 write 回调没有更新 state，或 read 回调没有正确组合返回值。

✅ 你现在就做：找一个测试地址，手工算一次 offset。

```text
GPIO_BASE = 0x10012000
GPIO_IS   = GPIO_BASE + 0x10
write callback 里看到的 offset 应该是 0x10
```

⚠️ 容易踩坑：不要在回调里拿绝对地址做 switch。正确习惯是对 `offset` 做 switch。这样设备 base 改了也不用改设备内部逻辑。

---

## 4. 寄存器行为怎么写成 read/write

🎥 视频/作者：视频强调“重点不是地址，而是寄存器语义和副作用”。读某个寄存器可能清状态位，写某个寄存器可能触发中断，写 data register 可能启动一次传输。

📘 讲义：PL011 的 read/write 会解析寄存器偏移，更新 flags、FIFO 和中断状态。FIFO 深度受 `LCR_FEN` 控制，中断由 `int_level & int_enabled` 决定，最后映射到多条 IRQ 线。

🧠 我的理解：写寄存器时可以按这套顺序拆。

```text
1. offset 是否合法
2. size 是否符合设备要求
3. value 哪些 bit 可写，哪些 bit 只读，哪些 bit 保留
4. 写入是否改变内部状态
5. 是否有副作用，比如清中断、启动 timer、发送 SPI 字节
6. 是否需要调用 update_irq / update_status / update_timer
```

🧩 补充知识：常见寄存器语义要单独建模。

| 语义 | 写法要点 |
| --- | --- |
| reset 默认值 | `reset()` 里统一恢复，别散落在 init 和 write 里 |
| read-only | write 忽略，read 返回内部状态或派生值 |
| write-only | read 返回 0、保留值，或按规格返回 |
| RW mask | 只接受可写 bit，保留 bit 不应被 guest 写脏 |
| W1C | `reg &= ~(value & clear_mask)`，不是 `reg = value` |
| mirror bit | 由别的状态计算，别让 guest 随便写 |
| write trigger | 写入某个 magic value 或 data register 会触发动作 |

🔗 和 SoC 训练营的关系：这些语义会直接出现在测试里。

| 测试 | 寄存器语义 |
| --- | --- |
| `test-gpio-basic` | reset 默认值、`GPIO_IN` 派生读、`DIR/OUT` 可写 |
| `test-gpio-int` | `GPIO_IS` W1C，edge/level/polarity，IE mask |
| `test-pwm-basic` | `PWM_GLB` mirror bit，DONE W1C，CNT read-only |
| `test-wdt-timeout` | `WDT_VAL` read-only，`WDT_KEY` magic value，`WDT_SR` W1C |
| `test-spi-jedec` | `SPI_DR` 写触发传输，读 `SPI_DR` 消耗 RX 数据 |
| `test-spi-overrun` | `SPI_SR.OVERRUN` W1C，RXNE 未清又收到新字节触发错误 |

✅ 你现在就做：给 `GPIO_IS` 写一个伪代码。

```c
case GPIO_IS:
    /* Edge interrupt status is write-1-to-clear. */
    s->is &= ~(value & s->is_w1c_mask);
    gpio_update_irq(s);
    break;
```

⚠️ 容易踩坑：W1C 最容易被写成普通赋值。这样 `qtest_writel(qts, GPIO_IS, 0x1)` 之后状态不但没清，可能还被错误写成 `1`。

---

## 5. IRQ / timer / bus 这些副作用怎么理解

🎥 视频/作者：视频里把中断拆成三个层次：事件发生、更新内部状态位、把状态位投影到 IRQ 线。IRQ 线只看最终结果，真正原因在设备内部。

📘 讲义：PL011 维护 `int_level` 和 `int_enabled`，再用 mask 把不同中断原因映射到多条 IRQ 线，最后通过 `qemu_set_irq` 拉高或拉低。

🧠 我的理解：副作用不要直接塞在 read/write 的局部逻辑里，最好有统一的 update 函数。

```text
gpio_update_irq(s)
  -> 看 IS 和 IE
  -> 算出 active
  -> qemu_set_irq(s->irq, active)

pwm_update_timer(s, ch)
  -> 看 EN / PERIOD / DUTY
  -> 重新安排 QEMUTimer

spi_do_transfer(s, tx)
  -> 看 SPE / MSTR / CS
  -> 找到下游 flash
  -> 更新 RXNE/TXE/OVERRUN
  -> 可能更新 IRQ
```

🧩 补充知识：timer 和 IRQ 的关系很常见。PWM 的 counter、WDT 的 countdown、SPI flash 的 busy 都不是“写完寄存器立刻静态返回”能表达完整的，它们需要虚拟时间推进。qtest 里的 `qtest_clock_step` 就是在驱动这些虚拟时间事件。

🔗 和 SoC 训练营的关系：`test-pwm-basic`、`test-wdt-timeout`、`test-spi-overrun` 都会用 `qtest_clock_step`。这说明测试不是只看寄存器保存值，还在验证你有没有把 timer 和状态转换接起来。

✅ 你现在就做：每个设备都先设计一个 `update_xxx()`。

```text
GPIO: gpio_update_input_and_irq()
PWM:  pwm_update_channel_timer()
WDT:  wdt_update_timer_and_irq()
SPI:  spi_update_status_and_irq()
```

⚠️ 容易踩坑：不要在 5 个 write case 里各自手写一份 IRQ 计算。后面修一个 bit 的时候会漏。统一 update 函数是防止寄存器副作用失控的保险丝。

---

## 6. 对 SoC 测题的直接帮助

这一节是本章最实战的部分：你不是为了“懂 PL011”而学 PL011，而是为了知道 G233 外设题该怎么拆。

### `test-gpio-basic`

🎥 视频/作者：视频强调 state 是设备行为的本体。GPIO 这题最适合练“寄存器值和派生值”的区别。

📘 讲义：PL011 把寄存器、FIFO、中断状态放在 state 中；GPIO 可以照这个方法，把 `DIR/OUT/IE/IS/TRIG/POL` 放入 state。

🧠 我的理解：`GPIO_IN` 不一定是一个普通可写字段。测试里设置 `GPIO_DIR` 为 output 后写 `GPIO_OUT`，再读 `GPIO_IN`，期望输出 pin 能反映到输入视图。也就是说 read 回调里要组合 state，而不是只返回一个旧变量。

🔗 和 SoC 训练营的关系：`test-gpio-basic` 检查 reset 默认值、方向寄存器、多 pin 位操作、`OUT -> IN` 反映。它对应本章的“状态结构体设计”和“read/write 回调”。

✅ 你现在就做：先画寄存器表，再写 `gpio_read()` 的 `GPIO_IN` 分支，确认它不是盲目返回 `s->in`。

⚠️ 容易踩坑：`GPIO_OUT` 写了，`GPIO_IN` 没变。这个 bug 的本质是没有把硬件“输出回读”的派生关系建模出来。

### `test-gpio-int`

🎥 视频/作者：视频说中断线是内部状态的投影。GPIO interrupt 正好是这个观点的练习题。

📘 讲义：PL011 的中断不是一个裸 bool，而是 `int_level & int_enabled` 再映射到 IRQ 线。

🧠 我的理解：GPIO interrupt 至少要分四层：输入变化事件、`IS` 状态位、`IE` 使能、PLIC IRQ 线。edge/level、rising/falling/high/low 都是在决定“什么时候设置或清除 IS”。

🔗 和 SoC 训练营的关系：`test-gpio-int` 会检查 rising edge、level high、W1C 清 `GPIO_IS`、IE mask、PLIC IRQ 2 pending/claim/complete。它对应本章的“中断状态分层”和“machine IRQ 连接”。

✅ 你现在就做：给 GPIO 写一个统一流程。

```text
输出或输入变化
  -> 根据 TRIG/POL 判断是否产生事件
  -> 根据 IE 判断是否允许设置 IS
  -> gpio_update_irq()
  -> PLIC pending 变化
```

⚠️ 容易踩坑：edge 和 level 不能用同一种清除策略。edge 状态通常 sticky，靠 W1C 清；level 状态要跟电平条件联动。

### `test-pwm-basic`

🎥 视频/作者：视频里提到后面会进入 clock/timer，本章先把外设骨架讲清楚。PWM 是你第一次真正把外设骨架和虚拟时间揉在一起。

📘 讲义：PL011 的 state 里有 clock，迁移状态也可以包含 clock。PWM 的核心就是 channel 状态和 timer。

🧠 我的理解：PWM 不只是 `PERIOD` 和 `DUTY` 两个整数。它还要有 channel enable、counter、done flag、global mirror bit。`PWM_GLB` 里的 `CHn_EN` 是 mirror，`CHn_DONE` 是 W1C，这两个语义不一样。

🔗 和 SoC 训练营的关系：`test-pwm-basic` 检查配置保存、enable 后 GLB mirror、虚拟时间推进后 CNT 增长、周期完成后 DONE 置位、DONE W1C、多通道独立、极性位保存。

✅ 你现在就做：state 里把 4 个 channel 做成数组，而不是写 4 份字段。

```text
channels[4].ctrl
channels[4].period
channels[4].duty
channels[4].cnt
channels[4].timer
```

⚠️ 容易踩坑：`qtest_clock_step` 后 CNT 还是 0。这个不是 qtest 问题，是 timer 没注册、没 rearm，或者 enable 后没有启动计数。

### `test-wdt-timeout`

🎥 视频/作者：视频强调 class_init 里绑定 reset 和迁移，state 里要区分哪些状态需要保存。WDT 很适合练 reset、timer、lock、W1C。

📘 讲义：PL011 的 vmstate 明确保存寄存器和 FIFO 状态。WDT 未来如果支持迁移，也要考虑 `LOAD/CTRL/SR/locked` 以及 timer 剩余时间。

🧠 我的理解：WDT 是“寄存器协议 + 时间副作用”的组合。写 `LOAD` 是配置，写 `CTRL.EN` 是启动，写 `KEY.FEED` 是 reload，写 `KEY.LOCK` 是改变后续写权限，timeout 后设置 `SR.TIMEOUT` 并可能触发 IRQ。

🔗 和 SoC 训练营的关系：`test-wdt-timeout` 检查配置、倒计时、feed 重新装载、timeout flag、W1C 清 flag、lock 后禁用被忽略、INTEN 触发 PLIC IRQ 4。

✅ 你现在就做：给 WDT 写状态机。

```text
disabled
  -> write LOAD
  -> write CTRL.EN
running
  -> clock step: VAL 下降
  -> KEY.FEED: VAL 回到 LOAD
  -> timeout: SR.TIMEOUT = 1
locked
  -> 某些控制写入被忽略
```

⚠️ 容易踩坑：锁定不是一个寄存器值那么简单，而是“后续写行为”的条件。也就是说 lock 会改变 write 回调如何处理 `WDT_CTRL`。

### `test-spi-jedec / test-spi-cs / test-spi-overrun`

🎥 视频/作者：视频讲 PL011 的 FIFO 和中断时提醒，FIFO 不是优化，而是设备正确性的一部分。SPI 也一样，RXNE/TXE/OVERRUN 这些状态位就是协议本身。

📘 讲义：PL011 的 read/write 会驱动 FIFO 和中断状态。SPI 的 `DR` 读写也要驱动状态位、下游 flash、片选和错误状态。

🧠 我的理解：SPI controller 不是 flash 本体。SPI controller 是 master，`CR2` 选 CS，`DR` 写入一个 byte，controller 把它送给当前 flash，再把返回 byte 放进 RX buffer，并更新 `SR.RXNE/TXE/OVERRUN`。

🔗 和 SoC 训练营的关系：`test-spi-jedec` 检查 init、CS0 上 W25X16 的 JEDEC ID、TXE/RXNE 行为；`test-spi-cs` 检查 CS0/CS1 两片 flash 的 ID、容量、擦写读独立性；`test-spi-overrun` 检查 RXNE 未读时第二个字节导致 OVERRUN，并在 ERRIE 打开时触发 PLIC IRQ 5。

✅ 你现在就做：画 SPI 一次 byte transfer。

```text
guest write SPI_DR
  -> 检查 SPE/MSTR
  -> 如果 RXNE 还没清，置 OVERRUN
  -> 根据 CR2 选 flash
  -> flash 接收 tx byte，返回 rx byte
  -> rx 放入 DR/RX buffer
  -> 置 RXNE，置 TXE
  -> spi_update_irq()
```

⚠️ 容易踩坑：把 CS 当成普通数值保存就结束了。`test-spi-cs` 会验证两片 flash 的状态隔离，CS 必须真的决定当前访问的是哪个下游设备。

---

## 7. 最小练习

✅ 你现在就做：练习一，跟一遍 PL011。

```text
打开 hw/char/pl011.c 和 include/hw/char/pl011.h
找到 TypeInfo
找到 PL011State
找到 memory_region_init_io
找到 pl011_read / pl011_write
找到 pl011_update
找到 machine 里创建和连接 PL011 的位置
```

✅ 你现在就做：练习二，给 G233 GPIO 写一张“实现前表格”。

| offset | name | read 语义 | write 语义 | 副作用 |
| --- | --- | --- | --- | --- |
| `0x00` | `GPIO_DIR` | 返回方向 | 保存可写 bit | 可能影响 IN 派生 |
| `0x04` | `GPIO_OUT` | 返回输出值 | 更新输出 | 可能触发 edge/level |
| `0x08` | `GPIO_IN` | 返回输入视图 | 忽略 | 无 |
| `0x0C` | `GPIO_IE` | 返回使能 | 更新使能 | 更新 IRQ |
| `0x10` | `GPIO_IS` | 返回状态 | W1C | 更新 IRQ |
| `0x14` | `GPIO_TRIG` | 返回触发类型 | 更新 | 影响后续事件判断 |
| `0x18` | `GPIO_POL` | 返回极性 | 更新 | 影响后续事件判断 |

✅ 你现在就做：练习三，手动追踪一个 qtest。

```text
qtest_writel(qts, GPIO_OUT, 0x1)
  -> gpio_write(offset=0x04, value=0x1)
  -> s->out = 0x1
  -> 更新 GPIO_IN 派生视图
  -> 根据旧值和新值判断 edge
  -> 可能设置 GPIO_IS
  -> gpio_update_irq()
```

✅ 你现在就做：练习四，写一个“寄存器语义清单”，每次实现新外设前都过一遍。

```text
默认值是什么
哪些 bit 只读
哪些 bit 可写
哪些 bit 是 W1C
哪些 read 有副作用
哪些 write 有副作用
哪些字段需要 timer
哪些状态需要 IRQ update
哪些状态未来可能需要迁移
```

🧠 我的理解：这 4 个练习做完，你再写 SoC 外设题会稳很多。你不是靠感觉写 switch，而是先把硬件协议翻译成 state + callback + side effect。

⚠️ 容易踩坑：不要一上来就追求“像真实硬件一样完整”。训练营测试通常先要求一个可验证的最小模型。先让 qtest 证明你的寄存器语义正确，再逐步补复杂行为。

---

## 8. 下一章怎么接

🎥 视频/作者：视频最后把设备集成到 machine 作为收束点：创建设备、设置属性、realize、映射 MMIO、连接 IRQ、写设备树。设备模型写完，还必须让 guest 能发现它。

📘 讲义：讲义在 machine 集成部分用 ARM virt 的 PL011 举例：`qdev_new` 创建设备，`sysbus_realize_and_unref` 激活设备，`memory_region_add_subregion` 映射 MMIO，`sysbus_connect_irq` 接到中断控制器，并通过设备树暴露给 guest。

🧠 我的理解：本章解决“一个外设内部怎么写”。下一章“主板建模流程”会解决“这些外设怎么组成一块 SoC 板子”。再下一章“时钟系统”会补上 PWM/WDT/SPI 这种依赖虚拟时间和 clock 的部分。

🔗 和 SoC 训练营的关系：建议你后续顺序这样学。

| 顺序 | 章节 | 为什么 |
| --- | --- | --- |
| 1 | 外设建模流程 | 先会写一个 MMIO 设备 |
| 2 | 主板建模流程 | 把 GPIO/PWM/WDT/SPI 挂到 G233 machine |
| 3 | 时钟系统 | 理解 PWM/WDT 计时、clock、timer |
| 4 | kernel 运行机制 | 理解 guest driver 怎么通过 MMIO/IRQ 使用这些设备 |

✅ 你现在就做：学下一章前，先确认自己能解释这句话。

```text
qtest 不是在直接调用我的设备函数，
而是在模拟 guest 访问物理地址；
QEMU 地址空间把这个访问路由到 MemoryRegionOps；
我的 read/write 回调再把访问翻译成设备状态变化。
```

⚠️ 容易踩坑：如果后面某个 qtest 读不到设备，不要只查 read/write。也要查 machine 是否创建设备、MMIO base 是否一致、IRQ 号是否接对、设备是否 realize。

---

## 我的点评和延伸

🧠 我的理解：这章最值钱的地方，是它把“外设建模”从一堆 QEMU API 还原成工程动作。你以后看到任何 SoC 外设，都可以先问 7 个问题。

```text
它是什么类型的设备？
它有哪些寄存器？
哪些寄存器是真状态，哪些是派生视图？
guest 写寄存器会触发什么副作用？
有没有 IRQ，IRQ 原因和使能怎么组合？
有没有 timer / clock / FIFO / 下游 bus？
它在 machine 里怎么被创建、映射、连接？
```

🧩 补充知识：如果你想把这章变成代码能力，最重要的不是背 `memory_region_init_io`，而是形成“寄存器规格 -> state 字段 -> read/write 语义 -> update_irq/update_timer -> qtest 验证”的肌肉记忆。

🔗 和 SoC 训练营的关系：G233 的 SoC 模块其实就是把这章拆成多轮考试。GPIO 考 MMIO 和 IRQ，PWM 考多 channel 和 timer，WDT 考 timeout 和 magic key，SPI 考总线协议、片选、下游 flash 和 overrun。

✅ 你现在就做：下一步学习“主板建模流程”时，重点看 machine 如何把这些设备接起来。等你能同时说清“设备内部逻辑”和“machine 外部连线”，SoC 模块就不再是一团雾了。

