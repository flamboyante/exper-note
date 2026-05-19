# Day 2 白皮书：GPIO basic，从寄存器表到第一个 G233 外设

## 🎯 Day 2 总目标

Day 2 不是“写一个完整 GPIO 控制器”，而是先拿下 SoC 模块里最小、最干净的一条闭环：

```text
读 test-gpio-basic
-> 提取寄存器语义
-> 写 G233 GPIO SysBusDevice
-> 挂到 0x10012000
-> 让 qtest-riscv64/test-gpio-basic 通过
```

今天结束时，你要能回答：

1. `GPIO_DIR / GPIO_OUT / GPIO_IN` 分别对应设备里的哪些状态。
2. 为什么 `GPIO_IN` 在 basic 阶段可以用 `OUT & DIR` 派生。
3. `qtest_writel(qts, 0x10012004, value)` 怎么一路打到 `MemoryRegionOps.write`。
4. 一个最小 `SysBusDevice` 需要哪些字段和初始化步骤。
5. GPIO 设备为什么要放到 G233 machine 的 `0x10012000`。
6. 哪些 IRQ / timer / 边沿触发逻辑必须留到 Day 3，不要今天强行做。

✅ 你现在就做：

- 今天只追一个目标：`test-gpio-basic` 通过。
- 先读测试，再写设备；不要凭真实硬件经验自由发挥。
- 每改完一个阶段就跑单项测试，不要攒一大坨再调。

---

## 🧭 你的背景对应的学习策略

你有 Linux 驱动和一定内核经验，所以 Day 2 最适合用“驱动反过来写硬件”的方式理解：

```text
Linux driver 里：
    writel(value, base + GPIO_OUT)
    readl(base + GPIO_IN)

QEMU device 里：
    qtest_writel(addr, value)
    -> MemoryRegionOps.write(offset, value)
    -> 修改设备 state

    qtest_readl(addr)
    -> MemoryRegionOps.read(offset)
    -> 返回设备 state 或派生值
```

🧠 我的理解：

今天的核心不是 GPIO，而是“寄存器语义怎么落成 QEMU 设备模型”。GPIO basic 只是最适合练手，因为它暂时没有 timer、没有协议状态机、没有复杂 IRQ，能把注意力集中到 `DeviceState + MemoryRegion + read/write + board map` 这四件事上。

🧩 补充知识：

把真实硬件想成“寄存器后面有电路”，把 QEMU 设备想成“寄存器后面有 C 结构体和回调”。所以你写的不是 driver，而是 driver 未来会访问到的那块虚拟硬件。

⚠️ 容易踩坑：

- 不要把 `GPIO_IN` 写成一个普通可写寄存器；basic 测试里它应该由 `OUT & DIR` 派生。
- 不要今天就把 GPIO 中断完整写完；Day 3 的 `test-gpio-int` 才会严格要求边沿、极性、W1C、PLIC。
- 不要一边写代码一边改测试；测试是规格，不是敌人。

---

## 📚 Day 2 资料清单

### 本地必读

- [soc-day1-whitepaper.md](./soc-day1-whitepaper.md)
- [soc-7day-plan.md](./soc-7day-plan.md)
- [test-gpio-basic.c](../tests/gevico/qtest/test-gpio-basic.c)
- [hw/riscv/g233.c](../hw/riscv/g233.c)
- [include/hw/riscv/g233.h](../include/hw/riscv/g233.h)
- [tests/gevico/qtest/meson.build](../tests/gevico/qtest/meson.build)

### 四门课怎么用

- [qemu-hw-study.md](./qemu-hw-study.md)：用来抄“外设建模骨架”，重点看 `SysBusDevice`、`MemoryRegionOps`、reset、read/write。
- [qemu-board-study.md](./qemu-board-study.md)：用来理解 GPIO 怎么挂进 G233 board，重点看 `qdev_new`、`sysbus_realize_and_unref`、`sysbus_mmio_map`。
- [qemu-clock-study.md](./qemu-clock-study.md)：用来提醒自己 Day 2 不要引入 timer；GPIO basic 没有时间推进语义。
- Kernel 运行机制笔记：今天只吸收一个观点，QTest 的 MMIO 读写就是未来 guest driver `readl/writel` 的 host 侧替身。

### 官方/训练营资料

- QEMU 外设建模流程：<https://qemu.gevico.online/tutorial/2026/ch2/qemu-hw/>
- QEMU 主板建模流程：<https://qemu.gevico.online/tutorial/2026/ch2/qemu-board/>
- G233 SoC 实验手册：<https://qemu.gevico.online/exercise/2026/stage1/soc/g233-exper-manual/>
- G233 SoC 硬件手册：<https://qemu.gevico.online/exercise/2026/stage1/soc/g233-datasheet/>

📘 讲义：

训练营讲义给的是方法论：外设先做 state，MMIO 用 `MemoryRegionOps` 承接，板级代码负责把设备挂到地址空间。Day 2 要把这套方法压缩成一个最小 GPIO。

---

## 🗺️ 今日路线图

```text
09:00-09:30  回看 Day 1：确认 G233 地址空间和 QTest 调用链
09:30-10:30  读 test-gpio-basic.c，提取断言表
10:30-11:30  设计 GPIO state 和寄存器行为表
13:30-14:30  写 G233 GPIO 设备骨架
14:30-15:30  实现 MemoryRegionOps read/write/reset
15:30-16:30  挂到 G233 machine 的 0x10012000
16:30-17:30  跑 test-gpio-basic，按失败日志修
19:00-20:00  整理执行记录和 Day 3 IRQ 预备问题
```

🎯 今日最小产出：

```text
GPIO basic 行为表
G233 GPIO state 设计
read/write offset 映射表
board 挂载位置记录
test-gpio-basic 运行日志
Day 3 GPIO IRQ 待办清单
```

如果今天时间不够，优先级如下：

```text
必须完成：读测试 + 行为表 + 设备骨架
尽量完成：挂载到 0x10012000 + 单项测试
可以推迟：GPIO IRQ、PLIC、复杂边沿触发、调试美化
```

---

## 1. 先建立任务边界

### 🎥 视频/作者

专业阶段的节奏不是“听懂一个 API 就算学完”，而是“每个 API 都要服务一个训练营测试”。今天作者真正想让你跨过去的坎是：从阅读文档，切到写出一个能被 QTest 打中的设备。

### 📘 讲义

外设建模的基础路径是：

```text
定义设备类型
-> 定义设备 state
-> 初始化 MMIO MemoryRegion
-> 提供 MemoryRegionOps read/write
-> reset 设置默认值
-> board 侧实例化并映射地址
```

### 🧠 我的理解

Day 2 是你后面 PWM、WDT、SPI 的模板日。GPIO basic 写顺以后，后面只是给模板增加不同复杂度：

```text
GPIO basic：state + MMIO
GPIO int：state + MMIO + IRQ
PWM：state + MMIO + timer/clock
WDT：state + MMIO + timer + IRQ + lock/feed
SPI：state + MMIO + FIFO/协议状态机 + flash 设备
```

### 🧩 补充知识

`SysBusDevice` 可以粗略理解为“挂在系统总线上的 MMIO 外设”。它没有 PCI 那样的枚举流程，通常由 board/machine 代码主动创建、realize、映射 MMIO、连接 IRQ。

### 🔗 和 SoC 训练营的关系

今天只对准这一题：

```text
qtest-riscv64/test-gpio-basic
```

不要被后面的 `test-gpio-int`、`test-pwm-basic`、`test-wdt-timeout`、`test-spi-*` 分散。它们今天只作为“为什么不能过度设计”的参照。

### ✅ 你现在就做

先写下今天不做什么：

```text
今天不做 GPIO IRQ。
今天不做 PLIC 调试。
今天不做 PWM/WDT/SPI。
今天不做 guest Linux driver。
今天不改测试。
```

### ⚠️ 容易踩坑

最大坑是“你懂真实 GPIO，所以写得比测试要求复杂”。训练营早期最重要的是可验证语义，不是硬件手册级完整度。

---

## 2. 读 test-gpio-basic：把断言翻译成寄存器语义

### 🎥 视频/作者

作者在这类训练营里反复强调的思路是：测试文件就是实验规格。不要先猜外设该怎么写，而是先把 `qtest_readl/qtest_writel` 的断言翻译成寄存器行为。

### 📘 讲义

你要读：

- [test-gpio-basic.c](../tests/gevico/qtest/test-gpio-basic.c)

核心寄存器：

| offset | 名称 | Day 2 语义 |
| --- | --- | --- |
| `0x00` | `GPIO_DIR` | 方向寄存器，R/W，reset 为 0 |
| `0x04` | `GPIO_OUT` | 输出寄存器，R/W，reset 为 0 |
| `0x08` | `GPIO_IN` | 输入视图，read 返回 `out & dir` |
| `0x0C` | `GPIO_IE` | 中断使能，basic 阶段先保存值 |
| `0x10` | `GPIO_IS` | 中断状态，Day 3 完善 W1C |
| `0x14` | `GPIO_TRIG` | 触发方式，basic 阶段先保存值 |
| `0x18` | `GPIO_POL` | 触发极性，basic 阶段先保存值 |

### 🧠 我的理解

这份测试可以拆成四个场景：

| 场景 | 测试动作 | 你要实现的行为 |
| --- | --- | --- |
| reset | 启动 machine 后读所有寄存器 | 都是 0 |
| direction | 写 `GPIO_DIR` 再读回来 | `dir` 能保存 32 位值 |
| output bit0 | `DIR=1`，写 `OUT=1/0`，读 `IN` | `IN` 跟随 `OUT & DIR` |
| multi pin | 同时操作 bit0/7/15/31 | 位操作互不污染 |

`GPIO_IN` 最关键。它不是“外部真实引脚电平”，因为 Day 2 没有外部输入模型。测试用它来验证输出方向下的回读，所以最小正确语义是：

```c
GPIO_IN = GPIO_OUT & GPIO_DIR;
```

### 🧩 补充知识

真实 GPIO 通常有两种视角：

```text
output latch：软件写出去的输出锁存值
input sample：从引脚采样回来的电平
```

训练营 Day 2 没有外部世界，所以用 `OUT & DIR` 把“输出引脚读回”这件事简化掉。这是教学用语义，不等于所有真实芯片都这样。

### 🔗 和 SoC 训练营的关系

`test-gpio-basic` 是后续题的地基：

```text
test-gpio-basic
-> 证明 MMIO map 正确
-> 证明 read/write offset 正确
-> 证明 reset 默认值正确
-> 证明 32 个 pin 的 bit 操作正确
```

如果这个不过，Day 3 的 GPIO IRQ、后面的 PWM/WDT/SPI 都会变成“你不知道是挂载错了，还是设备语义错了”。

### ✅ 你现在就做

在执行记录里填：

```text
GPIO_BASE:
GPIO_DIR offset:
GPIO_OUT offset:
GPIO_IN offset:
test_gpio_reset_value 检查什么:
test_gpio_direction 检查什么:
test_gpio_output 检查什么:
test_gpio_multi_pin 检查什么:
```

### ⚠️ 容易踩坑

- `offset` 是相对 GPIO base 的偏移，不是绝对物理地址。
- `qtest_readl(GPIO_BASE + 0x08)` 命中 read 回调时，回调看到的是 `0x08`。
- 多 pin 测试会暴露“只处理 bit0”的偷懒实现。

---

## 3. 设计 GPIO state：哪些字段是真状态，哪些是派生视图

### 🎥 视频/作者

外设建模流程里很重要的一点是：先设计 state，再写 read/write。否则你会把寄存器读写写成一堆散乱分支，后面加 IRQ 时非常难改。

### 📘 讲义

最小设备状态可以先长这样：

```c
typedef struct G233GPIOState {
    SysBusDevice parent_obj;
    MemoryRegion mmio;
    qemu_irq irq;

    uint32_t dir;
    uint32_t out;
    uint32_t ie;
    uint32_t is;
    uint32_t trig;
    uint32_t pol;
} G233GPIOState;
```

### 🧠 我的理解

Day 2 要分清两类东西：

```text
真状态：
dir / out / ie / is / trig / pol

派生视图：
in = out & dir
```

为什么 `in` 不建议单独存？因为测试希望它随 `out` 和 `dir` 变化。你如果存成独立字段，就要到处同步，很容易出现 `OUT` 改了但 `IN` 忘了更新。

### 🧩 补充知识

设备模型里常见原则是：能从其他状态稳定推导出来的值，优先不要重复存。否则后面引入 reset、migration、IRQ side effect 时，会出现“两个状态谁才是真的”的问题。

### 🔗 和 SoC 训练营的关系

这个设计会直接影响后面：

```text
test-gpio-int:
    会继续使用 IE/IS/TRIG/POL
    还会要求状态变化后 update_irq()

test-pwm-basic:
    也会遇到“哪些寄存器保存，哪些状态派生”的问题

test-wdt-timeout:
    会要求 reset/feed/timeout 状态严格分清
```

### ✅ 你现在就做

写出 Day 2 寄存器状态表：

| 寄存器 | 是否保存字段 | 字段名 | reset | read | write |
| --- | --- | --- | --- | --- | --- |
| `GPIO_DIR` | 是 | `dir` | 0 | `dir` | 保存 |
| `GPIO_OUT` | 是 | `out` | 0 | `out` | 保存 |
| `GPIO_IN` | 否 | - | 0 | `out & dir` | 忽略 |
| `GPIO_IE` | 是 | `ie` | 0 | `ie` | 保存 |
| `GPIO_IS` | 是 | `is` | 0 | `is` | basic 可先保存，Day 3 改 W1C |
| `GPIO_TRIG` | 是 | `trig` | 0 | `trig` | 保存 |
| `GPIO_POL` | 是 | `pol` | 0 | `pol` | 保存 |

### ⚠️ 容易踩坑

- `GPIO_IN` 不要单独作为 `uint32_t in` 持久字段，除非你后面真的引入外部输入注入机制。
- Day 2 可以保留 `qemu_irq irq` 字段，但不需要把中断逻辑写完。
- reset 必须清掉所有寄存器，否则第一条 reset 测试就会失败。

---

## 4. 搭最小 SysBusDevice 骨架

### 🎥 视频/作者

视频/讲义里的外设建模流程可以理解成“三件套”：类型注册、MMIO 区域、读写回调。GPIO basic 正好把这三件套练完整。

### 📘 讲义

一个最小外设通常需要：

```text
头文件：
    TYPE_G233_GPIO
    OBJECT_DECLARE_SIMPLE_TYPE
    G233GPIOState

C 文件：
    read/write/reset
    MemoryRegionOps
    instance_init 或 realize
    class_init
    type_init

构建系统：
    hw/gpio/meson.build
    必要时 Kconfig

板级代码：
    g233.c 创建设备并映射到 0x10012000
```

### 🧠 我的理解

你可以把 GPIO 设备分成两层：

```text
设备文件负责：
    我是谁
    我有多少 MMIO 空间
    每个 offset 怎么读写
    reset 后是什么值

board 文件负责：
    我要不要创建这个设备
    它挂在哪个物理地址
    它的 IRQ 接到哪里
```

这和 Linux driver 的设备树思维有点像：设备自己描述行为，板级代码/设备树描述它在系统里的位置。

### 🧩 补充知识

`DeviceState` 是 QOM 设备对象的通用基类，`SysBusDevice` 是系统总线设备基类。GPIO 这种 MMIO 外设一般继承 `SysBusDevice`，再通过 `sysbus_init_mmio()` 暴露 MMIO 区域。

### 🔗 和 SoC 训练营的关系

后面你会反复复制这个骨架：

```text
GPIO -> G233GPIOState
PWM  -> G233PWMState
WDT  -> G233WDTState
SPI  -> G233SPIState
```

区别只在寄存器行为、timer、IRQ、副作用越来越复杂。

### ✅ 你现在就做

实现前先检查或规划文件清单：

```text
include/hw/gpio/g233_gpio.h
hw/gpio/g233_gpio.c
hw/gpio/meson.build
hw/riscv/g233.c
```

### ⚠️ 容易踩坑

- 不要把 GPIO 设备代码直接塞进 `g233.c`；这会让 board 文件越来越乱。
- 不要忘记把新 C 文件加入构建系统，否则代码写了也不会编译。
- 不要只实现 read/write，不实现 reset；QTest 会从 reset 默认值开始查。

---

## 5. 写 MemoryRegionOps：offset 到寄存器行为

### 🎥 视频/作者

外设建模视频里最应该反复看的就是 MMIO 回调。这里的 `offset` 就是寄存器表的 offset，`value` 就是 guest/QTest 写进来的值。

### 📘 讲义

Day 2 的 read 逻辑可以按这个心智模型写：

```text
offset 0x00 -> return dir
offset 0x04 -> return out
offset 0x08 -> return out & dir
offset 0x0C -> return ie
offset 0x10 -> return is
offset 0x14 -> return trig
offset 0x18 -> return pol
其他 offset -> 返回 0 或打印 guest error
```

write 逻辑：

```text
offset 0x00 -> dir = value
offset 0x04 -> out = value
offset 0x08 -> ignore
offset 0x0C -> ie = value
offset 0x10 -> Day 2 先保存或按 W1C 预留
offset 0x14 -> trig = value
offset 0x18 -> pol = value
其他 offset -> 忽略或打印 guest error
```

### 🧠 我的理解

今天最值得你盯住的是这个转换：

```text
qtest_writel(qts, GPIO_BASE + GPIO_OUT, value)
-> addr = 0x10012004
-> QEMU 找到 GPIO MemoryRegion
-> write(opaque, offset = 0x04, value, size = 4)
-> s->out = value
```

一旦这条链路通了，你就真正从“看 QEMU”走到了“能改 QEMU”。

### 🧩 补充知识

`MemoryRegionOps` 里还会涉及访问宽度、endianness、valid/impl size 等细节。Day 2 先按训练营测试的 32 位 `qtest_readl/qtest_writel` 做，不要提前处理所有 8/16/64 位访问。

### 🔗 和 SoC 训练营的关系

这一步是所有外设测试的共同入口：

```text
test-gpio-basic     -> GPIO read/write
test-gpio-int       -> GPIO read/write + IRQ side effect
test-pwm-basic      -> PWM read/write + timer side effect
test-wdt-timeout    -> WDT read/write + virtual clock side effect
test-spi-jedec      -> SPI read/write + FIFO/protocol side effect
test-spi-cs         -> SPI read/write + CS 选择 side effect
test-spi-overrun    -> SPI read/write + overrun/error IRQ side effect
```

### ✅ 你现在就做

先不要写完整代码，先在笔记里画 offset 表：

```text
0x00 GPIO_DIR  -> s->dir
0x04 GPIO_OUT  -> s->out
0x08 GPIO_IN   -> s->out & s->dir
0x0C GPIO_IE   -> s->ie
0x10 GPIO_IS   -> s->is
0x14 GPIO_TRIG -> s->trig
0x18 GPIO_POL  -> s->pol
```

### ⚠️ 容易踩坑

- `offset` 不要和 `GPIO_BASE` 相加后再 switch。
- `GPIO_IN` 的 write 最好忽略，不要污染状态。
- 不要在 read/write 里散落太多中断逻辑；Day 3 可以统一成 `g233_gpio_update_irq()`。

---

## 6. 接入 G233 machine：base 地址和 MMIO 区域

### 🎥 视频/作者

主板建模那一章的重点是：设备不只是写出来，还要被 board 创建并挂到地址空间。否则测试写 `0x10012000` 永远不会进入你的 GPIO 回调。

### 📘 讲义

G233 GPIO 的目标地址：

```text
GPIO_BASE = 0x10012000
```

典型挂载流程：

```text
DeviceState *dev = qdev_new(TYPE_G233_GPIO);
sysbus_realize_and_unref(SYS_BUS_DEVICE(dev), &error_fatal);
sysbus_mmio_map(SYS_BUS_DEVICE(dev), 0, 0x10012000);
```

如果 Day 3 要接 IRQ，再补：

```text
sysbus_connect_irq(SYS_BUS_DEVICE(dev), 0, irq_line);
```

### 🧠 我的理解

设备文件回答“GPIO 怎么工作”，`g233.c` 回答“GPIO 在这块 SoC 上在哪里”。这两个概念分开，后面你才不会把板级地址、IRQ 编号、寄存器语义混成一锅。

### 🧩 补充知识

真实 SoC 里地址空间通常来自硬件手册；QEMU 里则由 machine 代码建立 `MemoryRegion` 树。QTest 写的是 guest 物理地址，QEMU 根据这棵树找到对应设备。

### 🔗 和 SoC 训练营的关系

Day 2 挂载错地址时，现象通常是：

```text
readl 读不到你期望的值
write 没有改变设备 state
测试可能读到 0、触发 unassigned access，或者直接 assert 失败
```

所以要把 `GPIO_BASE` 和 `sysbus_mmio_map()` 对齐到同一个值。

### ✅ 你现在就做

在执行记录里填：

```text
GPIO base:
GPIO MMIO size:
挂载函数位置:
是否需要 IRQ:
Day 3 IRQ 预留方式:
```

### ⚠️ 容易踩坑

- `0x10012000` 是 GPIO base，不是 `GPIO_DIR` offset。
- `sysbus_mmio_map(..., 0, base)` 里的 `0` 是第几个 MMIO region，不是 offset。
- Day 2 可以不接 IRQ，但不要把设备结构写死成以后没法接 IRQ。

---

## 7. 配置构建系统：让代码真的进编译

### 🎥 视频/作者

外设建模流程里容易被忽视的是构建系统。QEMU 代码不是你新建一个 `.c` 文件就自动编译，Meson/Kconfig 没接上时，后面调半天其实跑的不是你的代码。

### 📘 讲义

重点检查：

```text
hw/gpio/meson.build
hw/gpio/Kconfig 或相关配置
hw/riscv/g233.c 是否 include 了需要的头
目标 riscv64-softmmu 是否能编译到这个设备
```

### 🧠 我的理解

如果你发现：

```text
代码里故意写错也能 build 通过
```

那通常说明新文件根本没进编译。这个检查非常土，但很救命。

### 🧩 补充知识

QEMU 的构建系统会按 target、Kconfig、Meson source set 组合决定哪些文件进最终二进制。训练营项目可能已经替你简化了一部分，但新增外设时仍要尊重它的构建路径。

### 🔗 和 SoC 训练营的关系

后面 PWM/WDT/SPI 都会重复这个动作。你今天把构建系统摸顺，后面每加一个设备就能少踩一次坑。

### ✅ 你现在就做

写完骨架后先跑：

```bash
cd ~/qemu-camp/qemu-camp-2026-exper-flamboyante
make -f Makefile.camp build
```

如果 build 没有编译你的新文件，先修构建系统，不要急着跑测试。

### ⚠️ 容易踩坑

- include 路径不对时，不要用相对路径硬凑。
- Meson 文件改了以后，必要时重新 configure。
- 不要在 Windows 工作区和 WSL 工作区之间来回改同一份代码，先确认你实际编译的是哪份。

---

## 8. 跑单项测试并按失败修正

### 🎥 视频/作者

训练营的工程节奏应该是“小步跑测试”。GPIO basic 的失败日志很有价值，它会告诉你当前是 reset、方向、输出还是多 pin 行为没对。

### 📘 讲义

推荐命令：

```bash
cd ~/qemu-camp/qemu-camp-2026-exper-flamboyante
make -f Makefile.camp build
cd build
./pyvenv/bin/meson test --no-rebuild --print-errorlogs qtest-riscv64/test-gpio-basic
```

如果训练营 Makefile 提供了更短的单项命令，优先用项目内命令。

### 🧠 我的理解

按失败类型定位：

| 失败位置 | 优先怀疑 |
| --- | --- |
| reset 全部不为 0 | reset 没接、字段没清、设备复用旧值 |
| `DIR` 写后读不回 | offset 映射错、write 没保存 |
| `OUT` 写后 `IN` 不对 | `GPIO_IN` 没用 `out & dir` |
| multi pin 不对 | 位操作写死 bit0、mask 处理错误 |
| 访问不到设备 | board 没挂载、base 地址错、构建没进 |

### 🧩 补充知识

QTest 是 host 侧测试，不需要等 guest Linux 启动，所以单项测试非常快。你应该利用这个特点，把 GPIO basic 调成“写一小步，跑一次”的节奏。

### 🔗 和 SoC 训练营的关系

今天跑通后，Day 3 可以从同一个 GPIO 设备继续演进，而不是重写：

```text
Day 2:
    dir/out/in/ie/is/trig/pol 基础读写

Day 3:
    外部输入注入或模拟
    edge/level 检测
    W1C status
    qemu_set_irq
    PLIC 侧观测
```

### ✅ 你现在就做

每次测试失败都记录：

```text
失败测试名：
失败断言：
期望值：
实际值：
我判断的问题层级：测试理解 / read/write / reset / board map / build
下一步最小修改：
```

### ⚠️ 容易踩坑

- 不要一次修五个地方；你会失去因果关系。
- 不要看到 `test-gpio-int` 失败就改 Day 2 目标。
- 不要为了让测试过而在 QTest 里特殊判断地址；应该修设备模型。

---

## 9. Day 2 执行记录

下面这部分是你当天边做边填的，不追求漂亮，追求真实。

### 9.1 GPIO basic 行为表

| 场景 | 测试动作 | 期望结果 | 实现提示 | 状态 |
| --- | --- | --- | --- | --- |
| reset | 读 `DIR/OUT/IN/IE/IS/TRIG/POL` | 全部 0 | reset 清所有字段，`IN=out&dir` |  |
| direction | 写 `DIR=0x1/0xFFFFFFFF` | 读回一致 | `s->dir = value` |  |
| output bit0 | `DIR=1`，写 `OUT=1/0` | `IN&1` 跟随 | `read IN` 返回 `out & dir` |  |
| multi pin | 操作 bit0/7/15/31 | 各 bit 独立 | 不要写死 bit0 |  |
| interrupt regs | basic 阶段读写相关寄存器 | reset 为 0，值可保存 | Day 3 再补 IRQ 语义 |  |

### 9.2 实现文件清单

| 文件 | 作用 | 是否完成 | 备注 |
| --- | --- | --- | --- |
| `include/hw/gpio/g233_gpio.h` | 类型和 state 声明 |  |  |
| `hw/gpio/g233_gpio.c` | GPIO 设备实现 |  |  |
| `hw/gpio/meson.build` | 把设备加入编译 |  |  |
| `hw/riscv/g233.c` | 创建并映射 GPIO |  |  |
| `include/hw/riscv/g233.h` | 地址枚举或声明 |  |  |

### 9.3 命令记录

| 时间 | 命令 | 结果 | 关键输出 / 错误摘要 | 下一步 |
| --- | --- | --- | --- | --- |
|  | `make -f Makefile.camp build` |  |  |  |
|  | `./pyvenv/bin/meson test --no-rebuild --print-errorlogs qtest-riscv64/test-gpio-basic` |  |  |  |

### 9.4 失败日志复盘

```text
第一次失败：
失败断言：
我判断原因：
修改点：
再次运行结果：

第二次失败：
失败断言：
我判断原因：
修改点：
再次运行结果：
```

### 9.5 Day 3 IRQ 预留问题

今天不要实现，但要记下：

```text
GPIO IRQ 编号：
IE 的含义：
IS 是否 W1C：
TRIG 表示 edge 还是 level：
POL 表示 high/rising 还是 low/falling：
update_irq() 应该在什么状态变化后调用：
```

---

## 10. Day 2 自测问题

如果下面问题能答出来，Day 2 思维就过关了：

1. `GPIO_BASE` 是多少？
2. `GPIO_OUT` 的 offset 是多少？
3. QTest 写 `GPIO_BASE + 0x04` 时，write 回调里的 `offset` 是多少？
4. `GPIO_IN` 为什么不建议单独存字段？
5. `GPIO_DIR=0x1`、`GPIO_OUT=0x3` 时，`GPIO_IN` 应该读到多少？
6. reset 后 7 个寄存器应该是什么值？
7. `SysBusDevice` 和 `DeviceState` 的关系是什么？
8. `memory_region_init_io()` 和 `sysbus_mmio_map()` 分别解决什么问题？
9. 为什么 Day 2 不做 timer？
10. 为什么 Day 2 不做完整 IRQ？

参考答案：

```text
1. 0x10012000。
2. 0x04。
3. 0x04。
4. 因为 basic 阶段它是 OUT & DIR 的派生视图，重复存容易不同步。
5. 0x1。
6. 全部 0。
7. SysBusDevice 是系统总线设备基类，间接属于 QOM DeviceState 体系。
8. 前者创建设备自己的 MMIO 区域和回调，后者把该区域映射到 board 地址空间。
9. GPIO basic 没有时间推进语义，timer 是 PWM/WDT 的主题。
10. GPIO IRQ 属于 Day 3，需要 IE/IS/TRIG/POL/PLIC/W1C 等完整语义。
```

---

## 11. Day 2 之后不要做的事

- 不要把 `test-gpio-basic` 过了就立刻重构一大轮。
- 不要为了“像真实硬件”提前加入复杂输入引脚模型。
- 不要今天就写完 `test-gpio-int` 的所有逻辑，除非 Day 2 已经稳定通过并且你明确切到 Day 3。
- 不要把 PWM/WDT/SPI 的字段提前塞进 GPIO。
- 不要把 `GPIO_IS` 的最终语义随手写死；Day 3 要根据测试确认 W1C 和触发条件。
- 不要忽略执行记录；后面调 SPI/WDT 时，你会感谢今天留下的调用链笔记。

---

## 12. Day 3 开工口令

Day 3 只做一件事：

```text
在 Day 2 GPIO basic 通过的基础上，实现 GPIO interrupt，让 test-gpio-int 通过。
```

最小路径：

```text
确认 Day 2 test-gpio-basic 仍通过
-> 读 test-gpio-int.c
-> 提取 IE/IS/TRIG/POL 语义
-> 实现外部输入或测试需要的触发入口
-> 实现 W1C
-> 实现 update_irq()
-> 接 PLIC IRQ line
-> 跑 test-gpio-int
```

Day 3 的判断标准不是“我写了中断代码”，而是：

```text
状态位什么时候置位说得清
W1C 怎么清状态说得清
qemu_set_irq 什么时候拉高/拉低说得清
PLIC 为什么能看到 GPIO IRQ 说得清
test-gpio-basic 仍然通过
test-gpio-int 通过
```

