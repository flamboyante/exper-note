# Day 3 白皮书：PWM + WDT，从驱动机理到 QEMU 虚拟时间

## 来源

这份 Day3 以真实测题为最高优先级，手册作为背景和编程模型参考。

| 来源 | 用法 |
| --- | --- |
| `/home/flamboy/qemu-camp/qemu-camp-2026-exper-flamboyante/tests/gevico/qtest/test-pwm-basic.c` | PWM 行为规格，以这里的断言为准 |
| `/home/flamboy/qemu-camp/qemu-camp-2026-exper-flamboyante/tests/gevico/qtest/test-wdt-timeout.c` | WDT 行为规格，以这里的 offset 和断言为准 |
| [G233 SoC 硬件手册](https://github.com/gevico/qemu-camp-tutorial/blob/main/docs/exercise/2026/stage1/soc/g233-datasheet.md) | PWM/WDT 的硬件背景、驱动编程模型、PLIC IRQ 编号 |
| [G233 SoC 实验手册](https://github.com/gevico/qemu-camp-tutorial/blob/main/docs/exercise/2026/stage1/soc/g233-exper-manual.md) | SoC 方向测题顺序和验收方式 |
| [qemu-clock-study.md](./qemu-clock-study.md) | `QEMUTimer`、`ptimer`、`QEMU_CLOCK_VIRTUAL`、`qtest_clock_step` 的知识补充 |
| [qemu-hw-study.md](./qemu-hw-study.md) | `SysBusDevice`、`MemoryRegionOps`、设备 state 的建模方法 |
| [qemu-board-study.md](./qemu-board-study.md) | board map、PLIC IRQ 接线、`sysbus_*` 接入方法 |

⚠️ 重要冲突说明：WDT 手册写的是 `WDT_SR=0x0C`、`WDT_KEY=0x10`，但真实测题写的是 `WDT_KEY=0x0C`、`WDT_SR=0x10`。训练营手册自己也说明“若手册与实际实现不一致，以测题/参考实现为准”。所以 Day3 的实现建议固定按测题：`KEY=0x0C`，`SR=0x10`。

## 阅读说明

Day2 的 GPIO 已经解决了“QTest 怎么打到一个 MMIO 设备”以及“设备怎么把状态投影成 IRQ”。Day3 的新增难点不是 read/write switch，而是“未来事件”：

```text
GPIO:
  写寄存器后，结果基本立刻出现

PWM / WDT:
  写寄存器只是启动一个未来事件
  测试通过 qtest_clock_step 推进虚拟时间
  到期后 timer callback 才改变 CNT / DONE / TIMEOUT / IRQ
```

这份文档按“驱动为什么这么写 -> 测题到底断言什么 -> QEMU 里怎么建模”的顺序读。你不需要先懂完整 Linux PWM/watchdog 子系统，但要能说清楚：驱动写的每个寄存器，最后会改变 QEMU 设备 state 里的哪一块。

## 一句话结论

Day3 的本质是把 Day2 的 `MMIO + state + side effect` 扩展成 `MMIO + state + virtual time + side effect`。

```text
PWM:
  用 QEMUTimer + elapsed time 让 CNT 随虚拟时间变大
  到一个周期后置 DONE
  DONE 用 W1C 清除

WDT:
  用 ptimer 让 VAL 随虚拟时间变小
  feed 后重装 VAL
  超时后置 TIMEOUT
  INTEN 打开时拉 PLIC IRQ 4
```

## Day3 核心心智模型

```text
driver/qtest writes MMIO
  |
  | qtest_writel(PWM_CH_CTRL(0), EN)
  | qtest_writel(WDT_CTRL, EN | INTEN)
  v
QEMU MemoryRegionOps.write()
  |
  | offset 被翻译成设备内部寄存器
  v
G233PWMState / G233WDTState
  |
  | 保存 ctrl / period / duty / load 等配置
  | PWM: 记录 start/last_update，并安排 QEMUTimer 到周期完成点
  | WDT: 配置 ptimer limit/count/frequency，然后 run
  v
QEMUTimer / ptimer on virtual time
  |
  | qtest_clock_step(qts, ns) 推进虚拟时间
  v
timer callback fires
  |
  | PWM: 主要置 DONE，CNT 由 elapsed time 推导
  | WDT: ptimer 到 0 后置 TIMEOUT
  v
optional qemu_set_irq()
  |
  | WDT INTEN 时进入 PLIC IRQ 4
  | PWM IRQ 3 手册有定义，但当前 test-pwm-basic 不检查
  v
PLIC pending
```

关键分层：

| 层 | 负责什么 | Day3 关注点 |
| --- | --- | --- |
| driver / qtest | 写 MMIO，读状态，推进虚拟时间 | `qtest_clock_step` 不是 sleep，是推进 QEMU 虚拟时钟 |
| 设备 read/write | 保存配置，启动/停止 timer，处理 W1C | 不要在 write 里假装时间已经过去 |
| timer / ptimer callback | 到期后改变状态位 | `DONE`、`TIMEOUT`、IRQ 都应该在这里或由这里触发 |
| board | 把设备映射到 base，IRQ 接到 PLIC | WDT 必须接 IRQ 4，PWM 当前测试先不依赖 IRQ |

---

## 1. 今天的任务边界

今天做：

| 目标 | 覆盖测题 | 必须掌握 |
| --- | --- | --- |
| PWM basic | `test-pwm-basic` | period、duty、polarity、enable、CNT、DONE、W1C、4 通道 |
| WDT timeout | `test-wdt-timeout` | LOAD、VAL、FEED、LOCK、TIMEOUT、W1C、PLIC IRQ 4 |
| 虚拟时间 | 两个测题都用 | PWM 用 `QEMUTimer + elapsed time`，WDT 用 `ptimer` |
| board 接入 | 两个设备都需要 | base 地址、MemoryRegion、IRQ 接线 |

今天不做：

| 不做 | 为什么 |
| --- | --- |
| PWM 真实波形输出 | 当前测题不观察引脚波形，只观察寄存器状态 |
| PWM IRQ 3 必过 | 手册定义了 IRQ 3，但 `test-pwm-basic` 没有检查 PLIC |
| WDT reset 机器 | 手册提到 reset，但测题只检查 timeout flag 和 IRQ |
| Linux driver 完整适配 | 这份文档只补驱动机理，不要求写 guest driver |
| SPI | SPI 是下一阶段，涉及协议状态机和 Flash，不要混在 Day3 |

今天结束时，你应该能口述：

```text
PWM 是“周期性计数 + 周期完成标志”。
WDT 是“递减倒计时 + 喂狗重装 + 超时报警”。
QEMU 里这两者都不是靠宿主 sleep，而是靠虚拟时间和 qtest_clock_step。默认选择是：PWM 用 `QEMUTimer + elapsed time`，WDT 用 `ptimer`。
```

---

## 2. PWM 驱动机理

### 2.1 PWM 到底在干什么

PWM 全称是 Pulse Width Modulation，脉冲宽度调制。它的核心不是“输出一个模拟电压”，而是在固定周期里控制高低电平持续时间。外部设备看到这个波形后，会把它理解成亮度、转速、占空比或控制信号。

一个 PWM 周期可以这样看：

```text
period = 10
duty   = 4
polarity = normal

计数:  0 1 2 3 4 5 6 7 8 9 | 0 1 2 3 ...
输出:  H H H H L L L L L L | H H H H ...
```

如果 `duty=4`，`period=10`，那么有效电平占 40%。这个比例就是占空比：

```text
duty_cycle = duty / period = 4 / 10 = 40%
```

如果 `polarity=1` 表示反相，那么“有效电平”可能从高电平变成低电平。驱动仍然配置同样的 `period/duty`，只是硬件输出的极性反过来。

### 2.2 驱动为什么要写 period / duty / polarity / enable

真实驱动面对的通常不是“写 1000 和 500”，而是上层要求：

```text
我要 1 kHz 的 PWM
占空比 50%
极性 normal
请启动
```

驱动会根据输入时钟频率换算寄存器值：

```text
period_ticks = input_clock_hz / target_frequency_hz
duty_ticks   = period_ticks * duty_percent / 100
```

然后写寄存器：

```text
write PWM_CHn_PERIOD = period_ticks
write PWM_CHn_DUTY   = duty_ticks
write PWM_CHn_CTRL   = POL | EN
```

在训练营测题里，换算过程被省略了。测题直接写 `PERIOD=1000`、`DUTY=500`，目的是验证你的设备模型能保存配置、启动计数、更新状态。

### 2.3 QEMU 里为什么不必真的输出波形

当前 `test-pwm-basic` 没有接一个虚拟 LED，也没有观察某根 GPIO 引脚的电平，所以你不需要模拟每个 tick 的高低电平。测试真正观察的是：

| 测题观察 | 对 QEMU 模型的要求 |
| --- | --- |
| `PERIOD/DUTY` 写后读回 | 保存寄存器影子值 |
| `CTRL.EN` 写后读回 | 保存 enable bit |
| `PWM_GLB.CHn_EN` | 根据 `CTRL.EN` 计算 mirror bit |
| `qtest_clock_step` 后 `CNT > 0` | 虚拟时间推进会改变 counter |
| 足够久后 `DONE=1` | 周期完成事件会置位状态 |
| 写 `DONE=1` 后清除 | W1C 语义正确 |

🧠 我的理解：PWM 在 Day3 是“虚拟时间驱动的状态机”，不是“波形渲染器”。先把测试可见状态建准，后面如果要接 Linux PWM framework 或虚拟外设，再补真实频率和输出线。

---

## 3. WDT 驱动机理

### 3.1 WDT 到底在解决什么问题

WDT 是 Watchdog Timer，看门狗定时器。它的设计很朴素：系统必须周期性证明自己还活着。

```text
系统正常:
  driver 周期性 feed
  WDT_VAL 被重新装载
  不会 timeout

系统卡死:
  driver 不再 feed
  WDT_VAL 一路倒数到 0
  WDT_SR.TIMEOUT = 1
  可触发 IRQ 或 reset
```

在真实系统里，WDT 常用于防死机。比如内核主循环、用户态守护进程或关键服务定期喂狗。如果系统锁死，喂狗动作停了，硬件就发出超时信号。

### 3.2 驱动为什么按 load -> enable -> feed 写

WDT 驱动典型流程：

```text
1. 写 WDT_LOAD
   设置超时窗口，比如 500 ms 或 10 s

2. 写 WDT_CTRL.EN
   启动倒计时

3. 周期性写 WDT_KEY_FEED
   把当前 VAL 重装回 LOAD

4. 如果需要中断，写 WDT_CTRL.INTEN
   超时后拉 PLIC IRQ

5. 如果配置不希望被误改，写 WDT_KEY_LOCK
   后续 CTRL 写入被忽略，直到复位
```

训练营测题把这个流程拆成七个小断言。你实现时不要一上来写“完整看门狗”，先让每个断言都能解释。

### 3.3 LOCK 和 TIMEOUT 的工程意义

`LOCK` 不是为了难为你，它的真实意义是防止系统运行到关键阶段后被误关看门狗。例如 bootloader 配好 WDT 后锁住，后面的软件如果 bug 写了 `CTRL=0`，硬件会忽略。

`TIMEOUT` 是 sticky flag。发生过超时后，它保持为 1，直到软件写 1 清除。这让软件即使晚一点检查，也能知道“刚才发生过超时”。

```text
TIMEOUT W1C:
  read WDT_SR -> 1
  write WDT_SR = 1
  read WDT_SR -> 0
```

🧠 我的理解：WDT 是“时间 + 责任链”的硬件化。驱动负责按时 feed；设备负责倒计时；超时后通过状态位和 IRQ 告诉系统“你没按时证明自己活着”。

---

## 4. PWM 标准行为

### 4.1 寄存器表

`test-pwm-basic.c` 里的地址定义如下：

| 寄存器 | 地址 / offset | 语义 |
| --- | --- | --- |
| `PWM_BASE` | `0x10015000` | PWM 控制器基地址 |
| `PWM_GLB` | `0x00` | `CHn_EN` mirror，`CHn_DONE` W1C |
| `PWM_CH_CTRL(n)` | `0x10 + n*0x10 + 0x00` | bit0 `EN`，bit1 `POL` |
| `PWM_CH_PERIOD(n)` | `0x10 + n*0x10 + 0x04` | 周期值 |
| `PWM_CH_DUTY(n)` | `0x10 + n*0x10 + 0x08` | 占空值 |
| `PWM_CH_CNT(n)` | `0x10 + n*0x10 + 0x0C` | 当前 counter，只读 |

`PWM_GLB` bit 定义：

| bit | 名称 | 访问语义 |
| --- | --- | --- |
| `[3:0]` | `CHn_EN` | 只读 mirror，来自各通道 `CTRL.EN` |
| `[7:4]` | `CHn_DONE` | 周期完成标志，写 1 清除 |

`PWM_CHn_CTRL` bit 定义：

| bit | 名称 | 当前测题是否检查 |
| --- | --- | --- |
| `0` | `EN` | 检查 |
| `1` | `POL` | 检查 |
| `2` | `INTIE` | 手册定义，当前 `test-pwm-basic` 不检查 |

### 4.2 七个测试函数对应的规格

| 测试函数 | 测试动作 | 期望结果 | 实现含义 |
| --- | --- | --- | --- |
| `g233/pwm/config` | 写 CH0 `PERIOD=1000`、`DUTY=500` | 读回一致 | `period/duty` 是普通 R/W 影子寄存器 |
| `g233/pwm/enable` | 写 CH0 `CTRL.EN` | `CTRL.EN` 和 `PWM_GLB.CH0_EN` 都为 1 | `GLB.EN` 是 mirror，不是独立保存 |
| `g233/pwm/counter` | 启动 CH0 后 step 1ms | `CH0_CNT > 0` | `CNT` 必须随虚拟时间推进 |
| `g233/pwm/done_flag` | `PERIOD=10` 后 step 100ms | `PWM_GLB.CH0_DONE=1` | 周期完成后置 DONE |
| `g233/pwm/done_clear` | 向 `PWM_GLB.CH0_DONE` 写 1 | DONE 清 0 | W1C 语义 |
| `g233/pwm/multi_channel` | 配置 CH0-CH3 不同 period/duty | 每个通道读回独立 | state 要数组化，offset 要算对 |
| `g233/pwm/polarity` | 写 `CTRL.POL` | 读回 `POL=1` | 当前只测配置保存，不测输出波形 |

### 4.3 PWM 最小可通过语义

```text
reset:
  ctrl/period/duty/cnt/done 全 0

write PERIOD/DUTY:
  保存到对应 channel

write CTRL:
  保存 EN/POL
  如果 EN 从 0 变 1:
    记录当前虚拟时间
    启动或重排 channel timer
  如果 EN 被清 0:
    停止 channel timer

read GLB:
  bits[3:0] = 每个 channel 的 CTRL.EN
  bits[7:4] = 每个 channel 的 done flag

write GLB:
  只处理 bits[7:4] 的 W1C
  不要把 bits[3:0] 当成可写 enable

timer callback:
  更新 CNT
  如果完成周期:
    done = true
```

⚠️ 容易踩坑：把 `PWM_GLB` 做成一个普通 `uint32_t glb`，然后写什么存什么。这样 `CHn_EN` mirror 很容易错，因为测试期望 `GLB.CH0_EN` 跟着 `CH0_CTRL.EN` 变。

---

## 5. WDT 标准行为

### 5.1 寄存器表，以测题为准

真实 `test-wdt-timeout.c` 的地址定义如下：

| 寄存器 | 地址 / offset | 语义 |
| --- | --- | --- |
| `WDT_BASE` | `0x10010000` | WDT 基地址 |
| `WDT_CTRL` | `0x00` | bit0 `EN`，bit1 `INTEN` |
| `WDT_LOAD` | `0x04` | reload value |
| `WDT_VAL` | `0x08` | 当前 counter，只读 |
| `WDT_KEY` | `0x0C` | magic write，feed 或 lock |
| `WDT_SR` | `0x10` | bit0 `TIMEOUT`，W1C |

magic key：

| 值 | 名称 | 行为 |
| --- | --- | --- |
| `0x5A5A5A5A` | `WDT_KEY_FEED` | 喂狗，`VAL` 重新装载为 `LOAD` |
| `0x1ACCE551` | `WDT_KEY_LOCK` | 锁定，后续 `CTRL` 写入无效 |

PLIC：

| 项 | 值 |
| --- | --- |
| `PLIC_BASE` | `0x0C000000` |
| `PLIC_PENDING` | `PLIC_BASE + 0x1000` |
| `WDT_PLIC_IRQ` | `4` |

⚠️ 再提醒一次：手册中的 WDT `SR/KEY` offset 和测题相反。本项目实现优先满足测题，所以写代码时不要照手册写错。

### 5.2 七个测试函数对应的规格

| 测试函数 | 测试动作 | 期望结果 | 实现含义 |
| --- | --- | --- | --- |
| `g233/wdt/config` | 写 `LOAD=0x100`，写 `CTRL.EN` | `LOAD` 读回，`CTRL.EN` 读回 | `LOAD/CTRL` 是 R/W |
| `g233/wdt/countdown` | `LOAD=0xFFFF`，enable，step 10ms | `VAL` 变小 | `VAL` 必须随虚拟时间递减 |
| `g233/wdt/feed` | 倒计时一段后写 feed key | `VAL` 变大 | feed 要 reload，并重排 timer |
| `g233/wdt/timeout_flag` | `LOAD=0x10`，enable，step 500ms | `SR.TIMEOUT=1` | timeout callback 置 sticky flag |
| `g233/wdt/timeout_clear` | timeout 后写 `SR.TIMEOUT=1` | `SR.TIMEOUT=0` | W1C 语义 |
| `g233/wdt/lock` | lock 后写 `CTRL=0` | `CTRL.EN` 仍为 1 | locked 后忽略 CTRL 写 |
| `g233/wdt/interrupt` | `EN|INTEN`，timeout | `SR.TIMEOUT=1` 且 PLIC IRQ 4 pending | timeout 时 `qemu_set_irq(irq, 1)` |

### 5.3 WDT 最小可通过语义

```text
reset:
  ctrl = 0
  load = 可以按测试友好值初始化，重点是写后读回
  val = load 或默认值
  sr = 0
  locked = false
  irq = low
  timer stopped

write LOAD:
  如果未 locked:
    load = value
    如果 WDT 正在运行:
      val = load
      重新安排 timeout

write CTRL:
  如果 locked:
    忽略
  否则:
    ctrl = value & (EN | INTEN)
    如果 EN:
      启动或重排 timer
    否则:
      停止 timer

write KEY:
  如果 FEED:
    val = load
    sr 清 TIMEOUT 或至少保持测试可接受
    IRQ 拉低或按 sr/inten 更新
    如果 EN:
      重新安排 timeout
  如果 LOCK:
    locked = true

write SR:
  sr &= ~(value & TIMEOUT)
  更新 IRQ

timer callback:
  val = 0
  sr |= TIMEOUT
  如果 CTRL.INTEN:
    qemu_set_irq(irq, 1)
```

🧠 我的理解：WDT 的 `VAL` 不是普通影子寄存器。它是“上次启动/喂狗时间、load、当前虚拟时间”共同决定的派生状态。为了测试简单，你可以用 timer callback 和读 `VAL` 时的时间差计算来实现，但不能永远返回 `load`。

---

## 6. 虚拟时间怎么承接 qtest_clock_step

### 6.1 `qtest_clock_step` 不是宿主 sleep

`qtest_clock_step(qts, 10000000)` 的意思不是让 Windows/WSL 真的等 10ms，而是告诉 QEMU：把虚拟时钟往前推进 10ms。

如果你的设备 timer 绑定在 `QEMU_CLOCK_VIRTUAL` 上，推进后到期的 timer callback 会被执行。

```text
设备写 CTRL.EN
  -> timer_mod(timer, now + timeout_ns)

测试 qtest_clock_step(500ms)
  -> QEMU virtual clock 前进
  -> timer 到期
  -> callback 执行
  -> WDT_SR.TIMEOUT = 1
```

### 6.2 `QEMUTimer` 和 `ptimer` 怎么选

更正式的说法不是“PWM 必须用哪个、WDT 必须用哪个”，而是看设备寄存器暴露的硬件语义。

| 机制 | 更像什么 | 适合场景 |
| --- | --- | --- |
| `QEMUTimer` | 到某个虚拟时间点触发一次 callback | 你自己根据 `qemu_clock_get_ns()` 推导 counter、状态或比较事件 |
| `ptimer` | QEMU 已封装好的 periodic down-counter | 设备有 `LOAD/VALUE/COUNT`，硬件语义就是按频率递减到 0 |

本项目默认选择：

| 设备 | 默认选择 | 为什么 |
| --- | --- | --- |
| PWM | `QEMUTimer + elapsed time` | G233 的 `CNT` 是递增 counter，用 elapsed time 推导最直观 |
| WDT | `ptimer` | G233 的 `LOAD/VAL` 是典型递减计数器，和 `ptimer` 完全贴合 |

🧠 我的理解：`ptimer` 本质上仍然基于 QEMU timer 体系，但它多封装了一层“递减计数器”。所以 WDT 用它会更正式；PWM 如果硬套 `ptimer`，也能做，但读 `CNT` 时要把“剩余 count”转换成“已经走过的 count”。

PWM 的两种合法写法：

| 写法 | 读 `CNT` | 优点 | 缺点 |
| --- | --- | --- | --- |
| `QEMUTimer + elapsed time` | `CNT = elapsed_ns / tick_ns` | 和递增 counter 直觉一致，容易解释 | 需要自己维护 `start_ns/last_update_ns` |
| `ptimer` | `CNT = PERIOD - ptimer_get_count()` | 更像硬件 counter 组件 | G233 `CNT` 是递增，语义要反向换算 |

WDT 的推荐写法：

```text
LOAD -> ptimer_set_limit(timer, load, reload=1)
VAL  -> ptimer_get_count(timer)
EN   -> ptimer_run(timer, oneshot=1)
FEED -> ptimer_set_count(timer, ptimer_get_limit(timer))
timeout callback -> SR.TIMEOUT=1, update_irq()
```

### 6.3 为什么不能用 sleep

设备模型里不要用宿主 `sleep` 或忙等：

| 错误做法 | 问题 |
| --- | --- |
| `sleep(1)` | 阻塞 QEMU 主循环，qtest 也不能正确推进 |
| read 时随便 `cnt++` | 可能骗过一次读取，但和 DONE/timeout 不自洽 |
| write 时立刻置 DONE/TIMEOUT | 没有体现虚拟时间，`qtest_clock_step` 失去意义 |
| 用真实墙钟时间 | 测试不稳定，机器快慢会影响结果 |

### 6.4 推荐的时间建模套路

PWM 默认用 `QEMUTimer + elapsed time`：

```c
static void pwm_channel_rearm(G233PWMState *s, int ch)
{
    int64_t now = qemu_clock_get_ns(QEMU_CLOCK_VIRTUAL);
    int64_t period_ns = pwm_period_to_ns(s->ch[ch].period);

    timer_mod(s->ch[ch].timer, now + period_ns);
}
```

WDT 默认用 `ptimer`：

```c
static void g233_wdt_reload_and_run(G233WDTState *s)
{
    ptimer_transaction_begin(s->timer);
    ptimer_set_limit(s->timer, s->load, 1);
    ptimer_run(s->timer, 1);
    ptimer_transaction_commit(s->timer);
}
```

这里的 `period_to_ns` 和 `ptimer_set_period/freq` 不需要一开始追求真实硬件频率。训练营测题只要求：

| 测题 | 时间要求 |
| --- | --- |
| PWM step 1ms 后 `CNT > 0` | counter 要能增长 |
| PWM step 100ms 后 `DONE=1` | 短 period 要能完成 |
| WDT step 10ms 后 `VAL` 变小 | countdown 要能下降 |
| WDT step 500ms 后 timeout | 小 load 要能超时 |

⚠️ 容易踩坑：`timer_mod` 用的是绝对到期时间，不是“从现在开始延迟多少”的相对值。通常写法是 `now + delay_ns`。

---

## 7. PWM 实现手册

### 7.1 推荐 state

```c
#define TYPE_G233_PWM "g233-pwm"
#define G233_PWM_SIZE 0x100
#define G233_PWM_NR_CH 4

typedef struct G233PWMChannel {
    uint32_t ctrl;
    uint32_t period;
    uint32_t duty;
    uint32_t cnt;
    bool done;
    QEMUTimer *timer;
    int64_t start_ns;
    int64_t last_update_ns;
} G233PWMChannel;

typedef struct G233PWMState {
    SysBusDevice parent_obj;
    MemoryRegion mmio;
    G233PWMChannel ch[G233_PWM_NR_CH];
} G233PWMState;
```

为什么数组化：

| 原因 | 说明 |
| --- | --- |
| 测题有 CH0-CH3 | `test_pwm_multi_channel` 会验证通道独立 |
| offset 有规律 | `0x10 + n*0x10 + reg` 适合统一解析 |
| 后续扩展方便 | 如果加 PWM IRQ 3 或输出线，不需要复制四份逻辑 |

### 7.2 offset 解析

```c
if (offset == PWM_GLB) {
    ...
} else if (offset >= 0x10 && offset < 0x50) {
    unsigned ch = (offset - 0x10) / 0x10;
    unsigned reg = (offset - 0x10) % 0x10;
}
```

通道寄存器：

| `reg` | 含义 |
| --- | --- |
| `0x00` | `CHn_CTRL` |
| `0x04` | `CHn_PERIOD` |
| `0x08` | `CHn_DUTY` |
| `0x0C` | `CHn_CNT` |

⚠️ 容易踩坑：`PWM_CH_CTRL(3)` 的 offset 是 `0x10 + 3*0x10 = 0x40`，不是 `0x30`。通道寄存器从 `0x10` 开始，每个通道占 `0x10`。

### 7.3 `PWM_GLB` read/write

`PWM_GLB` 不建议作为完整普通寄存器保存。它更像一个“状态视图”：

```c
static uint32_t g233_pwm_read_glb(G233PWMState *s)
{
    uint32_t v = 0;

    for (int i = 0; i < 4; i++) {
        if (s->ch[i].ctrl & PWM_CTRL_EN) {
            v |= 1u << i;
        }
        if (s->ch[i].done) {
            v |= 1u << (4 + i);
        }
    }

    return v;
}
```

`PWM_GLB` write 只处理 DONE W1C：

```c
static void g233_pwm_write_glb(G233PWMState *s, uint32_t value)
{
    for (int i = 0; i < 4; i++) {
        if (value & (1u << (4 + i))) {
            s->ch[i].done = false;
        }
    }
}
```

⚠️ 容易踩坑：写 `PWM_GLB=0` 不应该关掉 channel enable。enable 的源头是 `CHn_CTRL.EN`。

### 7.4 counter / done 的最小模型

PWM 默认用 `QEMUTimer + elapsed time`。原因是 G233 暴露出来的 `CHn_CNT` 是递增 counter，读寄存器时直接按虚拟时间差推导最顺。

```text
enable channel:
  cnt = 0
  done = false
  start_ns = now
  last_update_ns = now
  timer_mod(timer, now + period_delay)

read CNT:
  如果 EN:
    根据 now - last_update_ns 更新 cnt
  返回 cnt

timer callback:
  cnt 至少更新到非 0
  done = true
  如果仍 EN:
    可以继续 rearm 下一周期
```

如果你想顺手学习 `ptimer`，PWM 也能用，但要做反向换算：

```text
ptimer count = 距离本周期结束还剩多少 tick
PWM CNT      = PERIOD - ptimer_get_count()
```

两种方案对比：

| 方案 | 优点 | 注意 |
| --- | --- | --- |
| `QEMUTimer + elapsed time` | 和 `CNT` 递增语义一致，默认推荐 | 要自己维护 `start_ns/last_update_ns` |
| `ptimer` | 可以练习正式 down-counter 抽象 | `CNT = PERIOD - count`，语义要转换 |

🧠 我的理解：当前测题对 PWM 的要求是“有时间感”，不是“精确频率”。只要 1ms 后 `CNT > 0`，100ms 后短周期能 `DONE=1`，你的方向就是对的。

### 7.5 PWM 接入点

建议文件布局延续你现在的 GPIO 方式：

| 项 | 建议 |
| --- | --- |
| 头文件 | `hw/riscv/g233_feat/g233_pwm.h` |
| 源文件 | `hw/riscv/g233_feat/g233_pwm.c` |
| meson | `hw/riscv/meson.build` 里 `CONFIG_GEVICO_G233` 增加源文件 |
| board include | `hw/riscv/g233.c` include PWM 头 |
| memmap | `VIRT_PWM = 0x10015000, size 0x100` |
| create helper | 类似 `g233_gpio_create()` |

PWM 当前测题不要求 IRQ，所以最小接入可以先不连 IRQ 3。但手册定义了 PWM PLIC IRQ 3，后续如果你补 `INTIE`，应该接到 PLIC source 3。

---

## 8. WDT 实现手册

### 8.1 推荐 state

```c
#include "hw/core/ptimer.h"

#define TYPE_G233_WDT "g233-wdt"
#define G233_WDT_SIZE 0x100

typedef struct G233WDTState {
    SysBusDevice parent_obj;
    MemoryRegion mmio;
    qemu_irq irq;

    uint32_t ctrl;
    uint32_t load;
    uint32_t val;
    uint32_t sr;
    bool locked;

    ptimer_state *timer;
} G233WDTState;
```

`locked` 单独保存，不建议只塞进 `ctrl` 的某个未测 bit。测题只要求 lock 后 `CTRL` 写入无效，但你自己读代码时，单独字段更清楚。

WDT 默认用 `ptimer`。这是 QEMU 里更正式的 down-counter 抽象：它自己维护 limit/count，`ptimer_get_count()` 正好对应 `WDT_VAL`，到 0 后调用 callback。

realize/init 阶段至少要完成两件事：创建 `ptimer`，并设置 tick 频率或周期。教学版可以先用 `1000 Hz`，也就是 1 tick = 1 ms。

```c
static void g233_wdt_realize(DeviceState *dev, Error **errp)
{
    G233WDTState *s = G233_WDT(dev);

    s->timer = ptimer_init(g233_wdt_timeout, s,
                           PTIMER_POLICY_NO_COUNTER_ROUND_DOWN |
                           PTIMER_POLICY_TRIGGER_ONLY_ON_DECREMENT);

    ptimer_transaction_begin(s->timer);
    ptimer_set_freq(s->timer, 1000); /* 1 tick = 1 ms */
    ptimer_set_limit(s->timer, 0xffff, 1);
    ptimer_transaction_commit(s->timer);
}
```

如果后面接入 QEMU Clock QOM，也可以改成 `ptimer_set_period_from_clock()`；Day3 先不用把时钟树做复杂。

### 8.2 read/write 语义

read：

| offset | 返回 |
| --- | --- |
| `0x00 CTRL` | `s->ctrl` |
| `0x04 LOAD` | `s->load` |
| `0x08 VAL` | 当前倒计时值 |
| `0x0C KEY` | 只写寄存器，读可以返回 0 |
| `0x10 SR` | `s->sr` |

write：

| offset | 行为 |
| --- | --- |
| `0x00 CTRL` | 未 locked 时保存 `EN/INTEN`，按 EN 启停 timer |
| `0x04 LOAD` | 保存 load，必要时重装 val / rearm timer |
| `0x08 VAL` | 只读，忽略写 |
| `0x0C KEY` | `FEED` reload，`LOCK` 锁定 |
| `0x10 SR` | W1C 清 `TIMEOUT` |

### 8.3 countdown 的实现方式

为了让 `VAL` 在 `qtest_clock_step(10ms)` 后变小，使用 `ptimer_get_count()` 最直接：

```c
case WDT_VAL:
    return ptimer_get_count(s->timer);
```

启动和 reload 时统一走 transaction：

```c
static void g233_wdt_reload(G233WDTState *s, bool run)
{
    ptimer_transaction_begin(s->timer);
    ptimer_set_limit(s->timer, s->load, 1);
    if (run) {
        ptimer_run(s->timer, 1);
    }
    ptimer_transaction_commit(s->timer);
}
```

这里的 `ptimer_run(timer, 1)` 表示 one-shot：倒到 0 后触发一次 callback。对 WDT 测题来说，这比 periodic 更贴合，因为 timeout 后测试只检查 `TIMEOUT` 和 IRQ，不要求自动重装继续跑。

关键是测试里：

```text
val1 = read WDT_VAL
qtest_clock_step(10ms)
val2 = read WDT_VAL
assert val2 < val1
```

所以 `VAL` 不能永远等于 `LOAD`。用 `ptimer` 后，这件事由 QEMU 的 down-counter 帮你维护。

### 8.4 feed / lock / timeout

feed：

```c
static void g233_wdt_feed(G233WDTState *s)
{
    s->sr &= ~WDT_SR_TIMEOUT;
    g233_wdt_update_irq(s);

    if (s->ctrl & WDT_CTRL_EN) {
        g233_wdt_reload(s, true);
    } else {
        g233_wdt_reload(s, false);
    }
}
```

lock：

```c
case WDT_KEY:
    if (value == WDT_KEY_FEED) {
        g233_wdt_feed(s);
    } else if (value == WDT_KEY_LOCK) {
        s->locked = true;
    }
    break;
```

timeout callback：

```c
static void g233_wdt_timeout(void *opaque)
{
    G233WDTState *s = opaque;

    s->sr |= WDT_SR_TIMEOUT;
    g233_wdt_update_irq(s);
}
```

IRQ 更新：

```c
static void g233_wdt_update_irq(G233WDTState *s)
{
    bool level = (s->ctrl & WDT_CTRL_INTEN) &&
                 (s->sr & WDT_SR_TIMEOUT);

    qemu_set_irq(s->irq, level);
}
```

⚠️ 容易踩坑：`test_wdt_interrupt` 只检查 PLIC pending，不会帮你检查 `sysbus_connect_irq()` 写对没。如果 `WDT_SR.TIMEOUT` 对了但 PLIC pending 没有，优先看 IRQ 线有没有接到 source 4。

### 8.5 WDT 接入点

| 项 | 建议 |
| --- | --- |
| 头文件 | `hw/riscv/g233_feat/g233_wdt.h` |
| 源文件 | `hw/riscv/g233_feat/g233_wdt.c` |
| meson | `hw/riscv/meson.build` 里 `CONFIG_GEVICO_G233` 增加源文件 |
| board include | `hw/riscv/g233.c` include WDT 头 |
| memmap | `VIRT_WDT = 0x10010000, size 0x100` |
| IRQ | `WDT_PLIC_IRQ = 4` |
| create helper | 类似 `g233_gpio_create(addr, irq)` |

board 接线伪代码：

```c
static void g233_wdt_create(hwaddr addr, qemu_irq irq)
{
    DeviceState *dev = qdev_new(TYPE_G233_WDT);
    SysBusDevice *s = SYS_BUS_DEVICE(dev);

    sysbus_realize_and_unref(s, &error_fatal);
    sysbus_mmio_map(s, 0, addr);
    sysbus_connect_irq(s, 0, irq);
}
```

---

## 9. board / meson / 头文件接入提示

Day2 的 GPIO 已经给了一个模板。Day3 不要重新发明接入方式，直接复用这条链：

```text
设备头文件:
  TYPE_G233_PWM / TYPE_G233_WDT
  offset / bit 定义
  state 结构声明

设备源文件:
  TypeInfo
  realize
  reset
  MemoryRegionOps
  timer callback

meson:
  把 g233_pwm.c / g233_wdt.c 加进 CONFIG_GEVICO_G233

g233.h:
  enum memmap 增加 VIRT_WDT / VIRT_PWM
  enum irq 增加 WDT_PLIC_IRQ = 4
  PWM_PLIC_IRQ = 3 可以先预留，但当前测题不依赖

g233.c:
  virt_memmap 增加 base/size
  machine init 里 create 设备
  WDT 连接 qdev_get_gpio_in(mmio_irqchip, WDT_PLIC_IRQ)
```

建议顺序：

| 顺序 | 动作 | 为什么 |
| --- | --- | --- |
| 1 | 先加 PWM/WDT 的空设备和 map | 先确认 QTest 能打到设备 |
| 2 | 实现普通寄存器读写 | 先过 config 类断言 |
| 3 | 加 timer 和虚拟时间 | 再过 counter/countdown/done/timeout |
| 4 | 加 W1C | 清状态测试单独验证 |
| 5 | 加 WDT IRQ | 最后接 PLIC，避免状态机和接线混在一起 debug |

⚠️ 容易踩坑：如果 read/write 根本没进设备，不要先改 timer。先查 base 地址、memmap enum、`sysbus_mmio_map()`、Meson 是否把源文件编进去了。

---

## 10. 分阶段执行记录

### 阶段 1：PWM 配置寄存器

你要回答的问题：QTest 写 `PWM_CH_PERIOD/DUTY/CTRL` 后，是否真的进入 PWM 设备并保存到对应 channel？

期望输出：

```text
g233/pwm/config PASS
g233/pwm/polarity PASS
```

我的实际输出：

```text
<运行后贴 3-10 行关键日志>
```

如果不符合预期，优先查：

| 现象 | 优先排查 |
| --- | --- |
| 读回全是 0 | 设备没有 map 到 `0x10015000`，或 read offset 没实现 |
| CH0 正常，CH1-CH3 错 | channel offset 计算错 |
| POL 不生效 | `CTRL` bit mask 只保存了 EN |

### 阶段 2：PWM enable mirror

你要回答的问题：`PWM_GLB.CH0_EN` 是不是由 `CH0_CTRL.EN` 派生出来，而不是手动写 `GLB`？

期望输出：

```text
g233/pwm/enable PASS
```

我的实际输出：

```text
<运行后贴 3-10 行关键日志>
```

如果不符合预期，优先查：

| 现象 | 优先排查 |
| --- | --- |
| `CTRL.EN` 对，`GLB.EN` 不对 | `PWM_GLB` read 没计算 mirror |
| 写 `GLB` 影响 EN | 把 GLB 当普通可写寄存器了 |

### 阶段 3：PWM 虚拟时间

你要回答的问题：`qtest_clock_step` 后，PWM 的 counter 和 done flag 是否由 timer 推动？

期望输出：

```text
g233/pwm/counter PASS
g233/pwm/done_flag PASS
g233/pwm/done_clear PASS
```

我的实际输出：

```text
<运行后贴 3-10 行关键日志>
```

如果不符合预期，优先查：

| 现象 | 优先排查 |
| --- | --- |
| `CNT` 一直 0 | timer 没启动，或 read CNT 没更新时间 |
| `DONE` 不置位 | callback 没执行，period 到 ns 的映射太大，或没 rearm |
| `DONE` 清不掉 | W1C 写反了，应该 `done &= ~written_ones` |

### 阶段 4：PWM 多通道

你要回答的问题：CH0-CH3 是否各自保存 `period/duty`，互不污染？

期望输出：

```text
g233/pwm/multi_channel PASS
qtest-riscv64/test-pwm-basic PASS
```

我的实际输出：

```text
<运行后贴 3-10 行关键日志>
```

如果不符合预期，优先查：

| 现象 | 优先排查 |
| --- | --- |
| 只有 CH0 对 | state 没数组化 |
| CHn 读到相邻通道值 | `(offset - 0x10) / 0x10` 或 `% 0x10` 算错 |

### 阶段 5：WDT 配置和倒计时

你要回答的问题：WDT 是否能保存 `LOAD/CTRL`，并让 `VAL` 随虚拟时间下降？

期望输出：

```text
g233/wdt/config PASS
g233/wdt/countdown PASS
```

我的实际输出：

```text
<运行后贴 3-10 行关键日志>
```

如果不符合预期，优先查：

| 现象 | 优先排查 |
| --- | --- |
| `LOAD` 读不回 | offset 错或 read/write 没实现 |
| `VAL` 不变 | 没有根据虚拟时间更新 |
| 写 `CTRL.EN` 不生效 | locked 初始值错，或 ctrl mask 错 |

### 阶段 6：WDT feed / timeout / clear

你要回答的问题：WDT 是否能“喂狗重装、超时置位、W1C 清除”？

期望输出：

```text
g233/wdt/feed PASS
g233/wdt/timeout_flag PASS
g233/wdt/timeout_clear PASS
```

我的实际输出：

```text
<运行后贴 3-10 行关键日志>
```

如果不符合预期，优先查：

| 现象 | 优先排查 |
| --- | --- |
| feed 后 `VAL` 没变大 | `WDT_KEY=0x0C` 是否写对，feed 是否 reload |
| 500ms 后没有 TIMEOUT | `ptimer_run()` 没执行、`ptimer_set_freq/period` 没设置，或 load 到 timeout 的映射太长 |
| TIMEOUT 清不掉 | `WDT_SR=0x10` 是否按测题写，W1C 是否写反 |

### 阶段 7：WDT lock 和 IRQ

你要回答的问题：lock 后能否阻止 `CTRL` 写入，timeout 后能否进入 PLIC IRQ 4？

期望输出：

```text
g233/wdt/lock PASS
g233/wdt/interrupt PASS
qtest-riscv64/test-wdt-timeout PASS
```

我的实际输出：

```text
<运行后贴 3-10 行关键日志>
```

如果不符合预期，优先查：

| 现象 | 优先排查 |
| --- | --- |
| lock 后还能关 EN | `locked` 后没有忽略 CTRL 写 |
| `TIMEOUT` 有但 PLIC pending 没有 | `sysbus_init_irq` 或 `sysbus_connect_irq` 漏了 |
| PLIC pending 是别的 IRQ | IRQ 号不是 4 |
| 清 TIMEOUT 后 IRQ 仍高 | `g233_wdt_update_irq()` 没在 W1C 后调用 |

---

## 11. Day3 结论速查

### PWM

| 问题 | 答案 |
| --- | --- |
| PWM base 是多少 | `0x10015000` |
| PWM 有几个 channel | 4 个 |
| `period` 是什么 | 一个完整 PWM 周期长度 |
| `duty` 是什么 | 有效电平持续长度，`duty / period` 是占空比 |
| `polarity` 是什么 | 决定有效电平是高还是低 |
| `enable` 做什么 | 启动 channel 计数 |
| `PWM_GLB.CHn_EN` 怎么来 | 由 `CHn_CTRL.EN` 计算出来，是 mirror |
| `PWM_GLB.CHn_DONE` 怎么清 | 写 1 清除，W1C |
| 当前 PWM 是否必须做 IRQ 3 | 不必须，`test-pwm-basic` 不检查 |

### WDT

| 问题 | 答案 |
| --- | --- |
| WDT base 是多少 | `0x10010000` |
| WDT IRQ 是几号 | `WDT_PLIC_IRQ=4` |
| `LOAD` 做什么 | 设置倒计时 reload 值 |
| `VAL` 做什么 | 表示当前倒计时，只读 |
| `FEED` 做什么 | 写 `0x5A5A5A5A`，把 `VAL` 重装回 `LOAD` |
| `LOCK` 做什么 | 写 `0x1ACCE551`，后续 `CTRL` 写入无效 |
| `TIMEOUT` 怎么来 | 倒计时到期后置位 |
| `TIMEOUT` 怎么清 | 写 1 清除，W1C |
| 以哪个 WDT offset 为准 | 以测题为准：`KEY=0x0C`，`SR=0x10` |

### 虚拟时间选择

| 问题 | 答案 |
| --- | --- |
| `qtest_clock_step` 推进什么 | QEMU 虚拟时间，不是宿主 sleep |
| PWM 默认用什么 | `QEMUTimer + elapsed time` |
| WDT 默认用什么 | `ptimer` |
| 底层时间源 | 都应该跟虚拟时间走，避免宿主真实时间影响 qtest |
| timer callback 改什么 | PWM 主要置 `DONE`，`CNT` 由 elapsed time 推导；WDT callback 置 `TIMEOUT/IRQ` |
| reset/disable 时要做什么 | PWM `timer_del`，WDT `ptimer_stop`，并清状态 |

### ptimer

| 问题 | 答案 |
| --- | --- |
| `ptimer` 更像什么 | 一个 QEMU 封装好的 periodic down-counter |
| 为什么 WDT 适合 | `LOAD/VAL` 正好对应 `limit/count` |
| WDT `VAL` 怎么读 | `ptimer_get_count(timer)` |
| WDT feed 怎么做 | `ptimer_set_count()` 或重新 `ptimer_set_limit(..., reload=1)` |
| PWM 能不能用 | 能，但要把递减 count 转成递增 `CNT = PERIOD - count` |

### 实现顺序

```text
PWM:
  map -> config -> enable mirror -> counter -> done -> W1C -> multi_channel -> polarity

WDT:
  map -> config -> countdown -> feed -> timeout -> W1C -> lock -> IRQ 4
```

### 常见失败定位

| 失败点 | 先看哪里 |
| --- | --- |
| qtest 读写全错 | board map / memmap / Meson |
| 普通寄存器读回错 | read/write offset |
| 时间推进无效 | timer 是否绑定虚拟时钟，是否 rearm |
| 状态位清不掉 | W1C |
| PLIC 没 pending | `sysbus_init_irq`、`sysbus_connect_irq`、IRQ 号 |
| WDT offset 奇怪 | 记住测题 `KEY=0x0C`，`SR=0x10` |

---

## 12. 下一步怎么接 SPI

PWM/WDT 之后，下一步是 SPI。它会比 Day3 多一个维度：协议状态机。

```text
GPIO:
  MMIO + state + IRQ

PWM/WDT:
  MMIO + state + timer + IRQ

SPI:
  MMIO + state + timer + IRQ + command/phase/storage
```

Day3 给 SPI 铺好的能力：

| Day3 能力 | SPI 里怎么复用 |
| --- | --- |
| `QEMUTimer` | transfer 完成、flash busy 清除、PWM elapsed-time 模型 |
| `ptimer` | WDT 这类 down-counter，或者后续更正式的 timer 模型 |
| W1C | `SPI_SR.OVERRUN` |
| IRQ 投影 | `TXE/RXNE/ERRIE -> SPI IRQ 5` |
| state 拆分 | controller state + flash state |
| 分阶段调试 | JEDEC -> read/program/erase -> CS -> overrun |

下一章开始前，你只要带着这句话：

```text
SPI 不是一个更大的 GPIO，而是一个带协议记忆的外设。
每写一次 SPI_DR，不只是保存一个字节，而是在推进一次 command transaction。
```

Day3 到这里就够了。先把 PWM/WDT 的“虚拟时间”打通，再进 SPI 的“协议状态机”，学习路线会稳很多。
