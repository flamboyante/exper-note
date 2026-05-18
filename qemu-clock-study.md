# QEMU 时钟系统学习资料

## 来源

- 讲义：<https://qemu.gevico.online/tutorial/2026/ch2/qemu-clock/>
- 视频：<https://www.bilibili.com/video/BV1p99cBCEzV/>
- 视频标题校验：`QEMU 训练营 | 专业阶段 QEMU 时钟系统 [SOC 建模]`
- 作者 / UP：`绝对是泽文啦`
- 讲义作者：`@zevorn`
- 本地处理：已抓取讲义正文，已下载视频音频并用本地 `faster-whisper` 转写，视频时长约 `13:08`

本章没有放 PPT 截图。核心内容是概念边界和 API 使用链路，讲义已经提供关键代码片段；相比截图，把“什么时候用 QEMUClock，什么时候用 Clock QOM”讲清楚更重要。

---

## 阅读说明

- `🎥 视频/作者`：讲者在视频里强调的理解方式。
- `📘 讲义`：网页文档里可当手册反复查的稳定内容。
- `🧠 我的理解`：我把概念翻译成你写工程代码时的判断方式。
- `🧩 补充知识`：讲义和视频没完全展开，但做 SoC 必须知道的背景。
- `🔗 和 SoC 训练营的关系`：落到 PWM、WDT、SPI、I2C 这类 qtest 时间推进。
- `✅ 你现在就做`：看完这一节后立刻动手的小任务。
- `⚠️ 容易踩坑`：QEMU timer/clock 初学最容易混的地方。

---

## 一句话结论

QEMU 时钟系统有两套容易混的机制：`QEMUClock/QEMUTimer` 解决“虚拟时间怎么走、事件什么时候发生”，`Clock` QOM 对象解决“硬件频率怎么从板级时钟树分发到设备”。

🧠 我的理解：做 SoC 模块时，`qtest_clock_step` 能不能让 PWM counter 动、WDT timeout、SPI busy 结束，主要看你有没有正确使用 `QEMU_CLOCK_VIRTUAL + QEMUTimer`；而 baudrate、PWM 输入频率、PL011 clock 输入这类“频率来源”，才属于 `Clock` QOM 的领域。

---

## 本章结构

```text
QEMU 时间系统
  -> QEMUClock: 时间基准
    -> VIRTUAL / REALTIME / HOST / VIRTUAL_RT
  -> QEMUTimer: 未来某个时间点触发回调
    -> timer_init_ns()
    -> timer_mod_ns()
    -> timer_del()
  -> main loop: 计算最近到期时间并运行 timer
  -> qtest_clock_step: 测试里手动推进虚拟时间

硬件时钟树
  -> Clock QOM 对象
    -> qdev_init_clock_in()
    -> qdev_init_clock_out()
    -> qdev_connect_clock_in()
    -> clock_update_hz()
```

🎥 视频/作者：视频开头就给出核心结论：QEMU 里“时钟”不是一个东西，而是两套机制。第一套偏时间推进和调度，第二套偏硬件建模和频率传播。

📘 讲义：讲义一句话总结得很准：`QEMUClock` 决定“时间怎么走”，`Clock` 决定“频率怎么分发”。

🧠 我的理解：你可以把 `QEMUClock/QEMUTimer` 想成闹钟系统，把 `Clock` QOM 想成晶振/PLL/分频/门控构成的硬件时钟树。闹钟决定“什么时候叫醒我”，时钟树决定“这个设备以多少 Hz 工作”。

---

## 1. 为什么 QEMU 里有两套“时钟”

🎥 视频/作者：讲者强调，如果这两层边界没弄清楚，后面看设备建模、主循环、初始化代码会到处都看到 clock，但说不清它们各自干什么。

📘 讲义：QEMU 常见的“时钟”分两层：`QEMUClock` 是软件时钟，用于调度定时器和控制虚拟时间推进；`Clock` 是硬件时钟，用于设备/SoC 建模输入输出时钟和频率树。

🧠 我的理解：你写一个设备时，先问自己两个问题。

```text
我要安排未来某个时间点发生事件吗？
  -> 用 QEMUTimer，绑定某个 QEMUClock。

我要表达这个设备输入频率是多少、父时钟是谁、频率变化怎么通知设备吗？
  -> 用 Clock QOM。
```

🧩 补充知识：这两套机制可以同时出现在同一个设备里。比如 PWM 可能有一个输入 `Clock *clk` 表示频率来源，同时用 `QEMUTimer` 在虚拟时间到期时更新 counter 和 DONE flag。

🔗 和 SoC 训练营的关系：`test-pwm-basic`、`test-wdt-timeout`、`test-spi-jedec/cs/overrun` 都用 `qtest_clock_step`。这些测试优先考的是软件时间推进，不是完整硬件时钟树。

✅ 你现在就做：看到 `clock` 这个词，先判断它属于哪一类。

```text
QEMUClock/QEMUTimer: 时间推进、定时事件、qtest_clock_step
Clock QOM: 硬件频率、时钟输入输出、clock_update_hz
```

⚠️ 容易踩坑：不要看到“PWM 有时钟”就立刻上 Clock QOM。训练营最小实现可能只需要 `QEMUTimer` 让 `qtest_clock_step` 后状态变化。频率树可以作为后续增强。

---

## 2. QEMUClock：时间从哪里来

🎥 视频/作者：视频里重点讲了 `QEMU_CLOCK_VIRTUAL`：它只在虚拟机运行时前进，虚拟机暂停时也暂停。设备行为如果和 guest 执行强相关，优先考虑 virtual。

📘 讲义：常见时钟类型包括 `QEMU_CLOCK_REALTIME`、`QEMU_CLOCK_VIRTUAL`、`QEMU_CLOCK_HOST`、`QEMU_CLOCK_VIRTUAL_RT`。它们最终映射到不同时间源。

🧠 我的理解：四种时钟可以先这样记。

| 类型 | 直觉 | 设备建模建议 |
| --- | --- | --- |
| `QEMU_CLOCK_VIRTUAL` | guest 视角时间 | 设备定时行为优先用它 |
| `QEMU_CLOCK_REALTIME` | 宿主真实墙上时间 | 日志、统计、宿主辅助逻辑 |
| `QEMU_CLOCK_HOST` | 宿主系统时间源变化 | 较少用于普通设备 |
| `QEMU_CLOCK_VIRTUAL_RT` | icount 场景补虚拟时间 | 了解即可，先别乱用 |

🧩 补充知识：设备模型里最常见的坑，是把 guest 设备行为绑定到宿主真实时间。这样暂停 VM、单步调试、qtest 确定性都会变差。

🔗 和 SoC 训练营的关系：qtest 里的 `qtest_clock_step(qts, ns)` 推进的是测试视角下的虚拟时间。你的 PWM/WDT/SPI 如果没有挂到合适的虚拟时间机制上，测试里 step 了半天，设备状态也不会变。

✅ 你现在就做：做设备 timer 时先写下这句决策。

```text
这个事件应该随 guest 虚拟时间走，所以 timer 绑定 QEMU_CLOCK_VIRTUAL。
```

⚠️ 容易踩坑：用 `gettimeofday()`、Windows/Linux host time 或 REALTIME 去驱动设备寄存器。这样测试会不稳定，调试时也很诡异。

---

## 3. QEMUTimer：未来某个时间点做事

🎥 视频/作者：视频把 `QEMUTimer` 讲成设备定时行为的承担者：当设备要在未来发生某个事件时，用 timer 安排回调。

📘 讲义：常见 API 是 `timer_init_ns()`、`timer_mod_ns()`、`timer_del()`。`QEMUTimer` 绑定到某个 `QEMUClockType`，到期后执行回调。

🧠 我的理解：设备 timer 的最小生命周期是这条线。

```text
realize/init:
  timer_init_ns(&s->timer, QEMU_CLOCK_VIRTUAL, cb, s)

write enable/config:
  expire = qemu_clock_get_ns(QEMU_CLOCK_VIRTUAL) + delay_ns
  timer_mod_ns(&s->timer, expire)

timer callback:
  更新设备状态
  设置 DONE/TIMEOUT/RXNE 等状态位
  update_irq()
  如需周期性事件，再 timer_mod_ns()

reset/disable:
  timer_del(&s->timer)
```

🧩 补充知识：`timer_mod_ns()` 接受的是“绝对到期时间”，不是“相对延迟”。所以通常要用当前虚拟时间加上 delay。

```c
int64_t now = qemu_clock_get_ns(QEMU_CLOCK_VIRTUAL);
timer_mod_ns(&s->timer, now + delay_ns);
```

🔗 和 SoC 训练营的关系：PWM 周期完成、WDT timeout、SPI byte transfer completion、flash busy 结束，都可以用这个模式建模。测试里 `qtest_clock_step` 到达 expire 后，回调应该把状态位改掉。

✅ 你现在就做：给 WDT 写一个伪代码骨架。

```c
static void g233_wdt_timeout(void *opaque)
{
    G233WDTState *s = opaque;
    s->sr |= WDT_SR_TIMEOUT;
    if (s->ctrl & WDT_CTRL_INTEN) {
        qemu_set_irq(s->irq, 1);
    }
}
```

⚠️ 容易踩坑：enable 后忘记 `timer_mod_ns()`，或者 reset/disable 后忘记 `timer_del()`。前者表现为时间永远不动，后者表现为设备明明关了，未来回调又把状态改回来。

---

## 4. 主循环和 qtest_clock_step 怎么触发 timer

🎥 视频/作者：视频讲到定时器注册好了以后，还要有东西驱动时间推进并执行到期事件，这个中枢就是 QEMU 主循环。

📘 讲义：主循环会根据所有 timer 的最近到期时间计算 poll 超时，返回后执行到期 timer。相关主线是 `main_loop_wait()`、`timerlistgroup_deadline_ns()`、`qemu_clock_run_all_timers()`。

🧠 我的理解：在正常运行时，主循环像一个调度器：看最近哪个 timer 快到了，就决定睡多久，醒来后跑到期回调。在 qtest 里，测试通过 `qtest_clock_step` 主动推进虚拟时间，让 timer 更可控。

```text
test:
  qtest_clock_step(qts, 100000000)

QEMU:
  虚拟时间前进 100ms
  找到已到期 QEMUTimer
  执行 callback
  设备状态变化

test:
  qtest_readl() 看到 DONE/TIMEOUT/RXNE/OVERRUN
```

🧩 补充知识：qtest 的时间推进是学习设备模型的宝贝。它让你不用真的睡 100ms，也不用靠宿主时间碰运气，而是确定性地推进虚拟时间。

🔗 和 SoC 训练营的关系：`test-pwm-basic` 里 `qtest_clock_step(qts, 100000000)` 后期望 `PWM_GLB_CH_DONE` 置位；`test-wdt-timeout` 里 step 500ms 后期望 `WDT_SR_TIMEOUT` 置位；SPI 相关测试用小步 step 等待 `TXE/RXNE` 或 flash busy 清除。

✅ 你现在就做：以后看到 qtest 里有 `qtest_clock_step`，立刻标注一句。

```text
这个测试不是只检查寄存器保存值，它在检查设备有没有随虚拟时间推进。
```

⚠️ 容易踩坑：用循环在 read 回调里“临时计算状态”，却没有 timer callback 和状态位。短期可能骗过一个测试，但遇到 IRQ、W1C、pending、busy 这类副作用时会崩。

---

## 5. Clock QOM：硬件频率怎么分发

🎥 视频/作者：视频里把第二套机制叫做硬件线模和频率传播。它关心的是设备的输入输出 clock 怎么连接、频率变化怎么通知到设备。

📘 讲义：`Clock` 用于建模硬件时钟树。设备可以用 `qdev_init_clock_in()` 创建输入时钟端口，用 `qdev_init_clock_out()` 创建输出时钟端口，用 `qdev_connect_clock_in()` 连接来源，频率通过 `clock_update_hz()` 或 `clock_update_ns()` 传播。

🧠 我的理解：`Clock` QOM 更像硬件原理图上的时钟线。它不是“等 10ms 后做某事”，而是“这个设备现在的输入频率是 50MHz，频率变了要通知设备重新计算 baudrate/周期”。

```text
板级晶振 / PLL
  -> Clock out
    -> UART clk in
    -> PWM clk in
    -> WDT clk in
```

🧩 补充知识：PL011 讲义里有 `qdev_init_clock_in(..., "clk", pl011_clock_update, ...)`。这就是设备声明自己需要一个输入时钟，频率变化时执行回调。它和 `QEMUTimer` 是不同层次，但可以配合使用。

🔗 和 SoC 训练营的关系：如果训练营后续要求更真实的 PWM 频率、WDT tick rate、SPI baudrate，就要把 Clock QOM 纳入设计。但当前 qtest 多数先看虚拟时间推进和状态变化，最小实现先用固定 tick/period 也能建立主线。

✅ 你现在就做：区分两个问题。

```text
PWM 输入频率是多少？ -> Clock QOM / 固定频率配置
PWM 什么时候 period 到期？ -> QEMUTimer
```

⚠️ 容易踩坑：把 `Clock` 当 timer 用。`clock_update_hz()` 不会自动帮你在未来触发 DONE，它只是传播频率变化。

---

## 6. 迁移、reset 和 icount 要怎么想

🎥 视频/作者：视频后面提到 `icount`，直觉是让虚拟时间和指令执行建立确定性关系。普通设备建模先知道它存在，不要过早陷进去。

📘 讲义：Clock 对象提供 `vmstate_clock`，设备可用 `VMSTATE_CLOCK` 写入迁移描述。`QEMU_CLOCK_VIRTUAL_RT` 在 icount 模式下用于推进虚拟时间，避免 vCPU 休眠时虚拟时间停滞。

🧠 我的理解：做一个可靠设备，reset 和迁移都要考虑 timer 状态。

```text
reset:
  删除 timer
  恢复寄存器默认值
  清状态位
  拉低 IRQ

migration:
  保存足够状态，load 后能恢复设备行为
  如果保存 Clock，需要 VMSTATE_CLOCK
  如果有 active timer，需要考虑剩余时间或重建 timer
```

🧩 补充知识：训练营早期 qtest 可能不测迁移，但你建立习惯会很赚。WDT/PWM/SPI 这种带时间的设备，迁移时如果只保存寄存器、不恢复 timer，迁移后设备会“失去未来事件”。

🔗 和 SoC 训练营的关系：当前 SoC 模块的核心还是 qtest 通过；迁移可以先作为加分项。但 reset 默认值一定会测，比如 `test-gpio-basic`、`test-board-g233`、SPI init、WDT config 都隐含要求 reset 状态干净。

✅ 你现在就做：所有带 timer 的设备都写一条 reset 规则。

```text
reset 时必须 timer_del，并清理由 timer 产生的状态位和 IRQ。
```

⚠️ 容易踩坑：reset 只清寄存器，不删 timer。结果 reset 后旧 timer 到期，又把 DONE/TIMEOUT 置回来了。这个 bug 很阴，像半夜自己响的闹钟。

---

## 7. 对 SoC 测题的直接帮助

这一章对 SoC 模块的价值非常直接：凡是测试里出现 `qtest_clock_step`，都应该回到本章这条线检查。

### `test-pwm-basic`

🎥 视频/作者：设备行为如果和 guest 执行过程强相关，优先用 `QEMU_CLOCK_VIRTUAL`。

📘 讲义：`QEMUTimer` 用于安排未来某个时间点的设备事件。

🧠 我的理解：PWM 的 counter 和 DONE 不是普通寄存器保存值。enable 后，timer 应该随虚拟时间推进；周期到达后更新 CNT/DONE，必要时继续安排下一周期。

🔗 和 SoC 训练营的关系：`test_pwm_counter` step 1ms 后要求 `CHn_CNT > 0`；`test_pwm_done_flag` step 100ms 后要求 `PWM_GLB_CH_DONE(0)` 置位；`test_pwm_done_clear` 还要求 DONE 是 W1C。

✅ 你现在就做：PWM 的实现先拆成两部分。

```text
write CTRL.EN:
  启动或停止 channel timer

timer callback:
  增加/更新 CNT
  到 period 后设置 DONE
  更新 GLB mirror
```

⚠️ 容易踩坑：把 `CNT` 写成每次 read 时加一。这样简单 counter 测试可能过，但 DONE、W1C、周期性行为会很难自洽。

### `test-wdt-timeout`

🎥 视频/作者：Virtual time 适合虚拟定时器、DMA 完成延迟、网络设备内部等待等 guest 视角行为。

📘 讲义：设备行为和客户机时间绑定时，优先用 `QEMU_CLOCK_VIRTUAL`。

🧠 我的理解：WDT 是最典型的 QEMUTimer 题。`LOAD` 决定超时长度，`CTRL.EN` 启动 timer，`KEY.FEED` 重新安排 timer，timeout callback 设置 `SR.TIMEOUT` 并按 `INTEN` 拉 IRQ。

🔗 和 SoC 训练营的关系：`test_wdt_countdown` step 10ms 后要求 `WDT_VAL` 变小；`test_wdt_feed` 要求 feed 后 value 变大；`test_wdt_timeout_flag` step 500ms 后要求 timeout；`test_wdt_interrupt` 要求 PLIC IRQ 4 pending。

✅ 你现在就做：WDT 的 feed 不要只改 `VAL`，还要重新安排到期时间。

```text
KEY_FEED:
  s->val = s->load
  timer_mod_ns(&s->timer, now + load_to_ns(s->load))
```

⚠️ 容易踩坑：timeout 后只置状态位，不更新 IRQ。测试 `WDT_SR` 可能过，但 PLIC pending 会失败。

### `test-spi-jedec / test-spi-cs / test-spi-overrun`

🎥 视频/作者：设备里的等待逻辑、DMA 完成延迟这类行为，也应该跟虚拟机推进绑定。

📘 讲义：定时器解决“未来某个时刻做某件事”。主循环和 qtest clock step 会触发到期回调。

🧠 我的理解：SPI 可以做成同步模型，也可以做成带短延迟的异步模型。训练营测试里 `spi_wait_txe()`、`spi_wait_rxne()` 通过小步 `qtest_clock_step(qts, 1000)` 等待状态位，说明你的实现至少要让 TXE/RXNE 在虚拟时间推进后稳定出现。

🔗 和 SoC 训练营的关系：`test-spi-jedec` 等 RXNE 后读 JEDEC；`test-spi-cs` 等 flash busy 清除；`test-spi-overrun` 第二次写 DR 后 step 100000，再检查 OVERRUN 和 PLIC IRQ 5。

✅ 你现在就做：SPI transfer 可以先写成这条状态机。

```text
write SPI_DR:
  如果 RXNE 已经为 1，设置 OVERRUN
  清/更新 TXE
  安排 transfer_done timer

transfer_done callback:
  根据 CS 和 tx byte 产生 rx byte
  设置 RXNE/TXE
  update_irq()
```

⚠️ 容易踩坑：把 TXE/RXNE 永远置 1。短测试可能混过去，但 overrun、busy、interrupt-driven transfer 会出问题。

### `test-flash-read` / `test-flash-read-interrupt`

🎥 视频/作者：设备内部等待逻辑也应该用虚拟时间，不要和宿主时间绑死。

📘 讲义：主循环会运行到期 timer，timer 精确与否影响设备超时行为。

🧠 我的理解：flash 的 erase/program/read status 很适合用 timer 建模 busy。写入 erase/program 命令后置 `BUSY`，安排一个虚拟时间到期回调，到期后清 busy。

🔗 和 SoC 训练营的关系：测试里会循环读 status，并用 `qtest_clock_step(qts, 100000)` 等 busy 清除。如果你没有用虚拟时间清 busy，测试会卡到 timeout。

✅ 你现在就做：flash busy 行为别写成宿主 `sleep`。用 timer。

⚠️ 容易踩坑：在 QEMU 设备模型里 sleep。它会阻塞主循环，让整台虚拟机卡住，qtest 也会很难看。

---

## 8. 最小练习

✅ 你现在就做：练习一，给 PWM 画时间线。

```text
t=0:
  write PERIOD=10
  write DUTY=5
  write CTRL.EN=1
  timer_mod_ns(done_timer, now + period_ns)

t=100ms:
  qtest_clock_step
  timer callback fires
  DONE=1
  qtest_readl(PWM_GLB) sees DONE
```

✅ 你现在就做：练习二，给 WDT 画时间线。

```text
LOAD=0x10
CTRL.EN=1
  -> 安排 timeout timer

qtest_clock_step(500ms)
  -> timeout callback
  -> SR.TIMEOUT=1
  -> if INTEN: IRQ=1
```

✅ 你现在就做：练习三，看到以下 API 时立刻归类。

| API / 概念 | 归类 | 用途 |
| --- | --- | --- |
| `timer_init_ns` | QEMUTimer | 初始化定时器 |
| `timer_mod_ns` | QEMUTimer | 安排绝对到期时间 |
| `timer_del` | QEMUTimer | 取消未来事件 |
| `qemu_clock_get_ns` | QEMUClock | 读取当前时间基准 |
| `qtest_clock_step` | qtest 虚拟时间 | 测试里推进时间 |
| `qdev_init_clock_in` | Clock QOM | 创建设备输入时钟 |
| `clock_update_hz` | Clock QOM | 传播频率变化 |

✅ 你现在就做：练习四，每个带时间的设备实现前写这 5 行。

```text
事件是什么：
绑定哪个 QEMUClock：
什么时候 timer_mod：
callback 改哪些状态位：
reset/disable 时是否 timer_del：
```

🧠 我的理解：这 5 行就是你防止“时间相关 bug”爆炸的护身符。PWM/WDT/SPI/I2C 一旦出问题，先按这 5 行排查。

⚠️ 容易踩坑：只在 write 回调里处理状态，不考虑“未来事件”。所有带 `qtest_clock_step` 的测试，本质都在逼你补上“未来事件”。

---

## 9. 下一章怎么接

🎥 视频/作者：这章把时间推进和硬件时钟树讲清楚后，SoC 设备的动态行为就有抓手了。后面再学 kernel 运行机制，就能从 guest 侧理解驱动为什么要等状态位、处理中断、访问 timer。

📘 讲义：本章结束时总结为两条脉络：前者解决“事件何时发生”，后者解决“频率如何分发”。

🧠 我的理解：下一步建议学 `kernel 运行机制`，因为你已经有了 SoC 建模三件套。

```text
外设建模：寄存器和设备行为
主板建模：地址和 IRQ 接线
时钟系统：虚拟时间和 timer 事件
```

🔗 和 SoC 训练营的关系：学完这三章后，你已经能系统拆 GPIO/PWM/WDT/SPI 题。再补 kernel 运行机制，是为了理解 Linux driver/guest 程序如何通过 MMIO、IRQ、timer 使用你建出来的设备。

✅ 你现在就做：开始实战前，用这张表自检。

| 设备 | 是否需要 QEMUTimer | 是否需要 Clock QOM | 关键测试 |
| --- | --- | --- | --- |
| GPIO | 通常不需要 | 通常不需要 | `test-gpio-basic/int` |
| PWM | 需要 | 可后续增强 | `test-pwm-basic` |
| WDT | 需要 | 可后续增强 | `test-wdt-timeout` |
| SPI | 可需要 | 可后续增强 | `test-spi-jedec/cs/overrun` |
| I2C | 可需要 | 可后续增强 | `test-i2c-*` |

⚠️ 容易踩坑：别因为 Clock QOM 很“硬件真实”就一上来全做。训练营最稳的路线是先用 QEMUTimer 让 qtest 过，再逐步把频率树补漂亮。

---

## 我的点评和延伸

🧠 我的理解：这章最有用的不是记住四种 `QEMUClockType`，而是形成一个工程判断：时间相关行为一定要可暂停、可单步、可 qtest 推进、可 reset 清理。做到这四点，你的设备模型就稳了很多。

🧩 补充知识：我建议你后面写大项目时，把时间逻辑单独拆出来，不要揉进 read/write 大 switch 里。

```text
寄存器写入：只负责改变配置和启动/停止 timer
timer callback：只负责时间到期后的状态变化
update_irq：只负责把状态投影成 IRQ
read 回调：只负责返回当前状态
```

🔗 和 SoC 训练营的关系：这会直接影响 commit 组织。比如 WDT 可以拆成：“寄存器和 reset 默认值”“QEMUTimer timeout”“feed/lock 语义”“IRQ 接 PLIC”。每个 commit 都能被一个测试或一组测试验证，你学习时也更容易跟上。

✅ 你现在就做：读完这章，回看 `test-pwm-basic.c` 和 `test-wdt-timeout.c`，把每个 `qtest_clock_step` 旁边标一句“它期待哪个 timer callback 已经发生”。这一步很小，但会让你突然看懂测试在催你实现什么。

