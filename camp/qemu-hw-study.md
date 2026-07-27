# QEMU 外设建模流程学习资料（自包含教材版）

## 来源

- 讲义：<https://qemu.gevico.online/tutorial/2026/ch2/qemu-hw/>
- 视频：<https://www.bilibili.com/video/BV1Ae9wBjEiD/>
- 视频标题校验：`QEMU 训练营 | 专业阶段 QEMU 外设建模流程 [SOC 建模]`
- 作者 / UP：`绝对是泽文啦`
- 讲义作者：`@zevorn`
- 讲义版本：基于 QEMU `v10.2.0`
- 本地素材：视频音频已下载并用本地 `faster-whisper` 转写，时长约 `11:51`

⚠️ 容易踩坑：`BV1p99cBCEzV` 不是本章视频，它是“专业阶段 QEMU 时钟系统 [SOC 建模]”。本章正确视频是 `BV1Ae9wBjEiD`。计划里提到的 `V:/CodexHome/tmp/video-study/BV1p99cBCEzV/doc.extracted.txt` 当前也对应时钟系统，不应再作为本章讲义缓存使用。

本教材不放 PPT 截图。讲义和视频的有效信息主要是代码结构、调用链和工程解释，用 Markdown 表格和 ASCII 流程图表达更稳定，也更适合后面查阅。

---

## 阅读说明

这份文档的目标不是“帮你回忆视频讲了什么”，而是替代视频和讲义，成为你学这一章的主教材。

- `🎥 视频/作者`：讲者强调的理解方法，尤其是哪些地方初学者容易乱。
- `📘 讲义`：讲义里的稳定知识点和必要源码短片段。
- `🧠 我的理解`：把 QEMU 概念翻译成写设备模型时的工程动作。
- `🧩 补充知识`：讲义和视频没有完整展开，但做 SoC 设备必须知道的背景。
- `🔗 和 SoC 训练营的关系`：对应到 G233 的 qtest、寄存器、设备模型设计。
- `✅ 你现在就做`：看完一节后马上能做的小练习。
- `⚠️ 容易踩坑`：实现 GPIO/PWM/WDT/SPI 时最常见的错误。

阅读建议：

1. 先通读 `0` 到 `5` 节，建立“一个设备从类型到 MMIO 再到 IRQ”的完整图。
2. 再重点读第 `6` 节，用 qtest 反推 G233 外设实现。
3. 最后做第 `7` 节练习和附录 B 检查题。

---

## 一句话结论

QEMU 外设建模就是把“硬件手册里的一个外设”翻译成 QEMU 里的一个对象：它有类型、有状态、有 MMIO 寄存器协议、有 IRQ/timer/bus 副作用，并且必须被 machine 创建、映射地址、连接中断后，guest 才能真正访问到。

```text
硬件外设
  -> QOM 类型：这个设备叫什么、继承谁、怎么创建
  -> State：设备当前有哪些寄存器和内部状态
  -> MemoryRegionOps：guest 读写 MMIO 时调用什么函数
  -> read/write：把寄存器访问翻译成状态变化
  -> update_irq/update_timer/update_bus：处理副作用
  -> machine 集成：把设备放进 SoC 地址空间并接到中断控制器
```

🧠 我的理解：你以后写 G233 的 GPIO/PWM/WDT/SPI，不要先问“这个 switch 怎么写”。先问：“这个外设有哪些状态？guest 能通过哪些寄存器观察或改变这些状态？哪些改变会引发 IRQ、timer 或下游设备行为？”

---

## 本章结构

```text
0. mental model
   先知道：QEMU 外设不是一堆函数，而是一个可创建、可连接、可访问的对象。

1. 外设模型是什么
   TypeInfo / DeviceState / SysBusDevice / machine 集成

2. 状态结构体怎么设计
   寄存器影子、派生状态、FIFO、IRQ、timer、下游设备

3. MMIO 和 MemoryRegionOps
   guest 物理地址访问如何进入 read/write 回调

4. read/write 怎么写
   reset 默认值、只读、可写、W1C、mirror bit、write trigger

5. IRQ / timer / bus 副作用
   状态位不是 IRQ 线；timer 不是普通变量；SPI controller 不是 flash 本体

6. SoC 测题映射
   用 qtest 反推 GPIO/PWM/WDT/SPI 实现

7. 最小练习
   从 PL011 到 G233 GPIO，完整走一遍

8. 下一章
   主板建模流程、时钟系统、kernel 运行机制
```

🎥 视频/作者：视频开头指出，很多人刚开始写 QEMU 设备时，会把 QOM、MMIO、IRQ、设备树、machine 集成混在一起。本章最重要的目标，是建立一条可复用的“外设建模主线”。

📘 讲义：讲义用 `PL011` UART 做样板。它是典型 `SysBusDevice`，有寄存器、FIFO、IRQ、clock、chardev、迁移状态，足够展示一个标准外设的骨架。

🧠 我的理解：PL011 不是让你背串口，而是让你学一套模板。GPIO 是简化版 PL011，PWM/WDT 是加 timer 的版本，SPI 是加下游 bus 和协议状态机的版本。

---

## 0. 学这章前先建立的 mental model

先把三个角色分清楚。

```text
guest 软件
  运行在虚拟 CPU 里，执行 load/store，认为自己在访问真实硬件寄存器。

QEMU 地址空间
  接住 guest 的物理地址访问，判断这个地址属于 RAM、ROM，还是某个 MMIO 设备。

设备模型
  一个 C 结构体加一组回调函数。它把 guest 的 read/write 翻译成设备状态变化。
```

举一个最小例子：

```c
qtest_writel(qts, GPIO_OUT, 0x1);
```

这行测试代码不是直接调用 `gpio_write()`。真实链路应该理解成：

```text
qtest_writel(qts, 0x10012004, 0x1)
  -> QTest 协议请求 QEMU 写 guest 物理地址
  -> QEMU 地址空间查找 0x10012004 属于哪个 MemoryRegion
  -> 命中 G233 GPIO 的 MMIO 区域
  -> 调用 gpio_write(opaque, offset=0x04, value=0x1, size=4)
  -> gpio_write 更新 s->out
  -> 重新计算 GPIO_IN / GPIO_IS / IRQ
```

🎥 视频/作者：视频里说，MMIO 表面像访存，本质是设备协议。guest 读写一个地址，触发的是设备逻辑，不是普通内存读写。

📘 讲义：PL011 的核心流程是：QOM 注册类型、状态结构描述寄存器/FIFO/IRQ/clock、MMIO 回调响应访问、IRQ 连接到中断控制器、machine 侧完成地址映射。

🧠 我的理解：本章所有 API 都服务于同一个问题：让 guest 的“物理地址读写”能够稳定地落到“你的设备状态机”上。

🧩 补充知识：MMIO 的关键不是“地址”，而是“地址背后的语义”。同样是写 4 字节，写 RAM 表示保存数据；写 `GPIO_OUT` 表示驱动引脚；写 `WDT_KEY` 可能表示喂狗；写 `SPI_DR` 可能表示发起一次 SPI 传输。

🔗 和 SoC 训练营的关系：G233 qtest 里的所有 `qtest_readl/qtest_writel` 都是这种链路。测试表面上读写地址，本质上是在验证你是否正确实现了设备协议。

✅ 你现在就做：记住一句话：`qtest_writel` 不是函数单测，它是在模拟 guest 软件访问 MMIO 寄存器。

⚠️ 容易踩坑：如果 qtest 失败，不要只盯着测试本身。失败可能来自 4 层：地址没映射、offset 算错、read/write 语义错、副作用没更新。

你应该能回答：

- 为什么 `qtest_writel` 最终会进入设备的 write 回调？
- 为什么 MMIO write 不等于普通内存写？
- 为什么设备模型必须被 machine 映射后 guest 才能看到？

答不上来回看：本节的 qtest 链路图。

---

## 1. 外设模型在 QEMU 里是什么

### 1.1 QEMU 外设不是裸函数，而是 QOM 对象

QEMU 有自己的对象系统，叫 QOM。你可以先把它理解成 C 语言写出来的“类和对象”机制。

一个外设模型至少要回答这些问题：

| 问题 | QEMU 里的答案 |
| --- | --- |
| 这个设备叫什么？ | `TYPE_XXX` |
| 它继承谁？ | `.parent = TYPE_SYS_BUS_DEVICE` |
| 一个实例占多大内存？ | `.instance_size = sizeof(XXXState)` |
| 创建实例时做什么？ | `.instance_init = xxx_init` |
| 这个类型有哪些类级行为？ | `.class_init = xxx_class_init` |

📘 讲义：PL011 的类型注册核心结构是：

```c
static const TypeInfo pl011_arm_info = {
    .name          = TYPE_PL011,
    .parent        = TYPE_SYS_BUS_DEVICE,
    .instance_size = sizeof(PL011State),
    .instance_init = pl011_init,
    .class_init    = pl011_class_init,
};
```

逐项解释：

| 字段 | 含义 | 你写 G233 设备时怎么想 |
| --- | --- | --- |
| `.name` | QEMU 识别这个类型的名字 | 例如 `TYPE_G233_GPIO` |
| `.parent` | 继承哪个父类 | SoC 内部 MMIO 外设通常是 `TYPE_SYS_BUS_DEVICE` |
| `.instance_size` | 每个设备实例的 state 多大 | 你的 `G233GPIOState` 或 `G233SPIState` |
| `.instance_init` | 对象刚创建时搭骨架 | 初始化 MMIO region、IRQ、clock |
| `.class_init` | 设置这个类型的默认能力 | realize、reset、properties、vmstate |

🎥 视频/作者：视频强调，如果没有类型注册，后面根本没法创建这个设备。`SysBusDevice` 也很关键，它说明这是 SoC 内部平台设备，不是 PCIe 那类可枚举总线设备。

🧠 我的理解：`TypeInfo` 是设备的“出生证明”。没有它，QEMU 不知道这个设备类型存在；没有正确 parent，QEMU 也不知道它能不能被当作 sysbus 设备去映射 MMIO、连接 IRQ。

### 1.2 `DeviceState` 和 `SysBusDevice` 是什么

可以先看继承关系：

```text
Object
  -> DeviceState
    -> SysBusDevice
      -> PL011State / G233GPIOState / G233PWMState / G233WDTState / G233SPIState
```

`DeviceState` 表示“这是一个 QEMU 设备”。它让设备进入 QEMU 设备生命周期，比如 realize、reset、属性设置、迁移等。

`SysBusDevice` 表示“这是挂在系统总线上的设备”。它给设备提供 sysbus 相关能力，最重要的是：

- 可以注册 MMIO region。
- 可以注册 IRQ 输出线。
- 可以被 machine 用 `sysbus_realize_and_unref()` 激活。
- 可以被 machine 用 `sysbus_connect_irq()` 接到中断控制器。

🧩 补充知识：SoC 内部外设一般不是“自动发现”的。真实 SoC 里，GPIO、UART、WDT 的地址通常写死在 datasheet 里；QEMU 也类似，machine 代码负责把这些设备放到固定地址。

### 1.3 设备代码和设备存在是两回事

一个常见误区是：我写了 `hw/gpio/g233_gpio.c`，设备就存在了。不是。

设备真正被 guest 看见，需要 machine 做这些事：

```c
DeviceState *dev = qdev_new(TYPE_PL011);
SysBusDevice *s = SYS_BUS_DEVICE(dev);

sysbus_realize_and_unref(s, &error_fatal);
memory_region_add_subregion(mem, base, sysbus_mmio_get_region(s, 0));
sysbus_connect_irq(s, 0, irq_line);
```

这段表达的是：

1. 创建一个设备对象。
2. 把它当作 sysbus 设备。
3. realize，让设备正式激活。
4. 把设备 MMIO region 放进系统地址空间。
5. 把设备 IRQ 输出接到中断控制器输入。

🔗 和 SoC 训练营的关系：G233 测试用固定地址访问外设：

| 设备 | base |
| --- | --- |
| WDT | `0x10010000` |
| GPIO | `0x10012000` |
| PWM | `0x10015000` |
| SPI | `0x10018000` |

这些地址只有在 machine 中映射过，qtest 才能访问到。否则 read/write 可能打到 unmapped 区域，或者根本不会进入你的设备回调。

✅ 你现在就做：看一个设备时，分两步问：

```text
设备内部：TypeInfo / State / MMIO / IRQ 实现了吗？
machine 外部：创建 / realize / map / connect IRQ 做了吗？
```

⚠️ 容易踩坑：设备内部写得再对，如果 machine 没接进去，qtest 仍然失败。反过来，machine 地址映射对了，但 read/write 语义错，qtest 也失败。

你应该能回答：

- `DeviceState` 和 `SysBusDevice` 的区别是什么？
- 为什么 SoC 内部 GPIO/WDT/SPI 更适合 `SysBusDevice`？
- 为什么“写了设备文件”不等于“guest 能看到设备”？

答不上来回看：`1.2` 和 `1.3`。

---

## 2. 设备状态结构体怎么设计

### 2.1 设备建模本质是状态建模

🎥 视频/作者：视频里强调，设备建模本身就是状态建模。guest 看到的所有行为，本质都是设备内部状态变化后的结果。

这句话非常重要。你写外设时，不要先写 `switch (offset)`，而要先设计 state。

📘 讲义：PL011 的状态结构体节选：

```c
struct PL011State {
    SysBusDevice parent_obj;
    MemoryRegion iomem;
    uint32_t flags;
    uint32_t lcr;
    uint32_t cr;
    uint32_t int_enabled;
    uint32_t int_level;
    uint32_t read_fifo[PL011_FIFO_DEPTH];
    int read_pos;
    int read_count;
    CharFrontend chr;
    qemu_irq irq[6];
    Clock *clk;
};
```

这不是随便堆字段。每类字段都有职责：

| 字段类型 | 例子 | 职责 |
| --- | --- | --- |
| 父类对象 | `SysBusDevice parent_obj` | 让这个 struct 能作为 QOM/sysbus 设备 |
| MMIO 入口 | `MemoryRegion iomem` | guest 地址访问进入设备的入口 |
| 寄存器影子 | `flags/lcr/cr` | 保存 guest 可读写的寄存器状态 |
| 中断状态 | `int_enabled/int_level` | 分离中断使能和中断原因 |
| FIFO 状态 | `read_fifo/read_pos/read_count` | 表达 UART 收发缓冲行为 |
| 外部连接 | `qemu_irq irq[]`、`Clock *clk` | 连接中断控制器和时钟输入 |
| 后端资源 | `CharFrontend chr` | 连接终端、socket、文件等串口后端 |

🧠 我的理解：state 结构体就是你的“硬件抽象”。你不需要模拟晶体管，但必须把 guest 能观察到的行为所依赖的状态保存下来。

### 2.2 state 字段分 5 类

写 G233 外设时，建议把字段分成 5 类。

| 类别 | 解释 | G233 例子 |
| --- | --- | --- |
| 寄存器影子 | guest 写了以后可再读回 | `GPIO_DIR`、`SPI_CR1`、`WDT_LOAD` |
| 派生状态 | 由其他状态算出来 | `GPIO_IN`、`PWM_GLB.CH_EN`、`WDT_VAL` |
| sticky 状态 | 事件发生后保持，直到软件清除 | `GPIO_IS`、`WDT_SR.TIMEOUT`、`SPI_SR.OVERRUN` |
| 外部连线 | 连接到 QEMU 其他对象 | `qemu_irq irq`、flash child、clock |
| 时间状态 | 需要虚拟时间推进 | PWM counter、WDT countdown、flash busy |

### 2.3 不要把每个寄存器都机械变成字段

有些寄存器适合有字段，有些不适合。

例子 1：`GPIO_DIR` 适合有字段。

```c
s->dir = value;
```

它是 guest 写入的配置，读回来也应该看到同样的方向位。

例子 2：`GPIO_IN` 不一定适合直接可写字段。

```text
GPIO_IN = 外部输入位 + 输出模式下的 OUT 回读
```

如果 pin 配成 output，测试期望写 `GPIO_OUT` 后读 `GPIO_IN` 能反映输出。这时 `GPIO_IN` 更像派生视图。

例子 3：`PWM_GLB.CH_EN` 是 mirror bit。

```text
PWM_GLB.CH_EN(n) = channels[n].ctrl.EN
```

它不是独立状态，而是 channel ctrl 的镜像。

🧩 补充知识：硬件寄存器表是软件接口，不等于内部实现。QEMU 设备模型应该模拟接口语义，不是机械地给每个 offset 一个变量。

### 2.4 reset 默认值放在哪里

`reset` 是设备回到硬件复位状态的地方。reset 应该集中设置：

- 寄存器默认值。
- FIFO 清空。
- sticky 状态清除。
- timer 停止或重新装载。
- IRQ 线拉低。

伪代码：

```c
static void g233_gpio_reset(DeviceState *dev)
{
    G233GPIOState *s = G233_GPIO(dev);

    s->dir = 0;
    s->out = 0;
    s->ie = 0;
    s->is = 0;
    s->trig = 0;
    s->pol = 0;
    g233_gpio_update_irq(s);
}
```

🔗 和 SoC 训练营的关系：`test-gpio-basic` 第一项就是 reset 默认值。它会读 `GPIO_DIR/OUT/IN/IE/IS/TRIG/POL`，全部期望为 0。这个测试本质是在问：你的 reset 语义集中且完整吗？

✅ 你现在就做：实现任何外设前，先写 state 设计表。

| 字段 | 来源 | reset 默认值 | 是否迁移 | 是否派生 |
| --- | --- | --- | --- | --- |
| `dir` | `GPIO_DIR` | `0` | 是 | 否 |
| `out` | `GPIO_OUT` | `0` | 是 | 否 |
| `is` | `GPIO_IS` | `0` | 是 | 否，sticky |
| `irq` | machine 连接 | 拉低 | 否 | 外部线 |

⚠️ 容易踩坑：把 reset 默认值散落在 `instance_init`、`realize`、`write` 里。这样后面系统 reset、qtest 重启、迁移恢复时很容易状态不一致。

你应该能回答：

- state 结构体为什么不是“寄存器表照抄”？
- 哪些状态应该保存，哪些状态应该计算？
- reset 应该清哪些东西？

答不上来回看：`2.2` 到 `2.4`。

---

## 3. MMIO 和 MemoryRegionOps

### 3.1 MMIO 是设备协议，不是普通内存

MMIO 全称是 memory-mapped I/O。它的形式像内存访问：

```c
qtest_writel(qts, 0x10012004, 0x1);
```

但语义不是“把 1 存到内存地址 `0x10012004`”。它表示：

```text
guest 写 GPIO_OUT 寄存器
  -> 设备输出值改变
  -> 可能影响 GPIO_IN 回读
  -> 可能触发 edge/level interrupt
```

🎥 视频/作者：视频强调，读写 MMIO 的重点不是地址本身，而是寄存器语义和副作用。读一个寄存器可能清状态位，写一个寄存器可能触发中断或传输。

### 3.2 MemoryRegion 是地址空间里的“门牌”

设备要让 guest 访问，必须创建一块 `MemoryRegion`。

📘 讲义：PL011 在 `instance_init` 里初始化 MMIO：

```c
memory_region_init_io(&s->iomem, OBJECT(s), &pl011_ops, s, "pl011", 0x1000);
sysbus_init_mmio(sbd, &s->iomem);
```

逐项解释：

| 参数/调用 | 含义 |
| --- | --- |
| `&s->iomem` | 设备自己的 MMIO 区域 |
| `OBJECT(s)` | 这个 region 属于哪个 QOM 对象 |
| `&pl011_ops` | guest 读写时调用哪些函数 |
| `s` | 传给 read/write 的 opaque，通常就是 state |
| `"pl011"` | region 名字，方便调试 |
| `0x1000` | 这个设备 MMIO 窗口大小 |
| `sysbus_init_mmio` | 告诉 SysBusDevice：我有一个 MMIO region |

🧠 我的理解：`MemoryRegion` 是设备在 QEMU 地址空间里的入口，但它此时还没有具体物理地址。真正放到 `0x10012000` 这种地址，是 machine 做的。

### 3.3 MemoryRegionOps 是 read/write 函数表

📘 讲义：PL011 的 `MemoryRegionOps`：

```c
static const MemoryRegionOps pl011_ops = {
    .read = pl011_read,
    .write = pl011_write,
    .endianness = DEVICE_LITTLE_ENDIAN,
    .impl.min_access_size = 4,
    .impl.max_access_size = 4,
};
```

这表示：

- guest 读这个 region 时，调用 `pl011_read`。
- guest 写这个 region 时，调用 `pl011_write`。
- 设备按小端解释数据。
- 最小/最大访问粒度是 4 字节。

G233 外设大多数测试都用 `qtest_readl/qtest_writel`，也就是 32-bit little-endian 访问。你的设备可以先支持 4 字节访问，其他 size 返回默认值或按 QEMU 习惯处理。

### 3.4 offset 是相对地址

这是初学者最容易写错的地方。

如果测试写：

```c
#define GPIO_BASE 0x10012000ULL
#define GPIO_OUT  (GPIO_BASE + 0x04)

qtest_writel(qts, GPIO_OUT, 0x1);
```

你的 write 回调里应该看到：

```text
addr/offset = 0x04
value       = 0x1
size        = 4
```

不是 `0x10012004`。

所以设备内部 switch 应该写：

```c
switch (offset) {
case 0x04:
    s->out = value;
    break;
}
```

不要写：

```c
case 0x10012004:
```

🔗 和 SoC 训练营的关系：GPIO、WDT、PWM、SPI 都在测试里定义了 base。base 是 machine 映射地址；设备 read/write 只管 base 内部 offset。

✅ 你现在就做：手工算三个 offset。

```text
WDT_SR  = 0x10010000 + 0x10 -> write 回调 offset 0x10
GPIO_IE = 0x10012000 + 0x0C -> write 回调 offset 0x0C
SPI_DR  = 0x10018000 + 0x0C -> write 回调 offset 0x0C
```

⚠️ 容易踩坑：把 machine base 和设备内部 offset 混在一起。结果是设备换个 base 就坏，或者 qtest 永远打不到正确 case。

你应该能回答：

- `memory_region_init_io` 做了什么？
- `sysbus_init_mmio` 和 `memory_region_add_subregion` 区别是什么？
- read/write 回调里的 offset 为什么不是绝对地址？

答不上来回看：`3.2` 到 `3.4`。

---

## 4. 寄存器行为怎么写成 read/write

### 4.1 read/write 的基本模板

设备 read/write 的工作不是“保存值”，而是实现寄存器协议。

一个靠谱的 write 模板：

```c
static void g233_xxx_write(void *opaque, hwaddr offset,
                           uint64_t value, unsigned size)
{
    G233XXXState *s = opaque;

    if (size != 4) {
        return;
    }

    switch (offset) {
    case REG_CTRL:
        s->ctrl = value & CTRL_WRITABLE_MASK;
        xxx_update_after_ctrl_write(s);
        break;
    case REG_STATUS:
        s->status &= ~(value & STATUS_W1C_MASK);
        xxx_update_irq(s);
        break;
    default:
        qemu_log_mask(LOG_GUEST_ERROR,
                      "bad write offset 0x%" HWADDR_PRIx "\n", offset);
        break;
    }
}
```

一个靠谱的 read 模板：

```c
static uint64_t g233_xxx_read(void *opaque, hwaddr offset, unsigned size)
{
    G233XXXState *s = opaque;

    if (size != 4) {
        return 0;
    }

    switch (offset) {
    case REG_CTRL:
        return s->ctrl;
    case REG_STATUS:
        return xxx_compute_status(s);
    default:
        qemu_log_mask(LOG_GUEST_ERROR,
                      "bad read offset 0x%" HWADDR_PRIx "\n", offset);
        return 0;
    }
}
```

🧠 我的理解：这个模板里最重要的是两件事：mask 和 update。mask 控制哪些 bit 真的可写；update 把寄存器变化传播到 IRQ、timer、下游 bus 等副作用。

### 4.2 常见寄存器语义

| 语义 | 含义 | 实现方式 |
| --- | --- | --- |
| RW | 可读可写 | 写入 state，读回 state |
| RO | 只读 | write 忽略，read 返回 state 或计算值 |
| WO | 只写 | write 触发动作，read 返回 0 或固定值 |
| W1C | 写 1 清除 | `reg &= ~(value & mask)` |
| mirror bit | 镜像其他状态 | read 时计算，不独立保存 |
| write trigger | 写入触发动作 | write case 中调用动作函数 |
| reserved bit | 保留位 | write mask 掉，read 返回 0 或固定值 |

### 4.3 W1C 是什么

W1C 是 write-one-to-clear。意思是：软件写 1 的 bit 被清掉，写 0 的 bit 不变。

正确写法：

```c
s->status &= ~(value & STATUS_W1C_MASK);
```

错误写法：

```c
s->status = value;
```

为什么硬件喜欢 W1C？因为状态位可能由硬件异步置位。软件清某个 bit 时，不应该误清其他同时发生的事件。

🔗 和 SoC 训练营的关系：

- `GPIO_IS`：edge interrupt status，测试会写 `1` 清除。
- `PWM_GLB.CH_DONE`：周期完成标志，测试会写 `1` 清除。
- `WDT_SR.TIMEOUT`：超时标志，测试会写 `1` 清除。
- `SPI_SR.OVERRUN`：溢出错误，测试会写 `1` 清除。

### 4.4 read 也可能有副作用

不是所有 read 都是无副作用的。SPI 是典型例子。

`SPI_DR` 的读语义通常是：

```text
如果 RX buffer 有数据
  -> 返回这个 byte
  -> 清 RXNE
  -> 可能允许下一次传输不 overrun
```

所以 `test-spi-jedec` 中：

```c
qtest_writel(qts, SPI_DR, tx);
...
return (uint8_t)qtest_readl(qts, SPI_DR);
```

这不是普通读回，而是从 SPI receive data register 取走一个接收字节。

### 4.5 write trigger 是什么

write trigger 指写某个寄存器不是为了保存值，而是为了触发动作。

| 设备 | 触发写 | 动作 |
| --- | --- | --- |
| WDT | `WDT_KEY = 0x5A5A5A5A` | feed，重新装载计数器 |
| WDT | `WDT_KEY = 0x1ACCE551` | lock，后续控制写可能被忽略 |
| SPI | `SPI_DR = byte` | 发起一次 SPI byte transfer |
| GPIO | `GPIO_OUT` 变化 | 可能触发 edge/level interrupt |

🧩 补充知识：触发写通常不要简单保存 `value`。比如 `WDT_KEY` 不需要读回 magic value，它的意义是“执行喂狗或锁定动作”。

### 4.6 read/write 写完必须更新副作用

很多寄存器写入后，需要立即更新相关状态。

```text
写 GPIO_IE
  -> 中断使能变化
  -> 需要 gpio_update_irq()

写 GPIO_OUT
  -> pin 电平变化
  -> 需要判断 edge/level
  -> 可能设置 GPIO_IS
  -> 需要 gpio_update_irq()

写 PWM_CH_CTRL.EN
  -> channel 启停变化
  -> 需要更新 PWM_GLB mirror
  -> 需要启动或停止 timer

写 WDT_CTRL.EN
  -> watchdog 启停变化
  -> 需要启动或停止 timer
```

✅ 你现在就做：看到任何寄存器写入，都问三个问题。

```text
这个值要不要保存？
这个值哪些 bit 允许保存？
保存后会不会影响 IRQ/timer/bus/派生寄存器？
```

⚠️ 容易踩坑：只写 `s->reg = value`，没有 mask、没有 W1C、没有 update。这样的模型可能过最简单的读回测试，但会在 interrupt、timer、overrun 这种测试里失败。

你应该能回答：

- W1C 为什么不能写成普通赋值？
- `SPI_DR` 为什么既是 data register 又是传输触发点？
- `GPIO_IN` 和 `PWM_GLB.CH_EN` 为什么可能是派生值？

答不上来回看：`4.2` 到 `4.6`。

---

## 5. IRQ / timer / bus 这些副作用怎么理解

### 5.1 IRQ：中断线是内部状态的投影

🎥 视频/作者：视频把中断拆成三层：事件发生、更新状态位、把状态位映射到 IRQ 线。IRQ 线只看最终结果，真正原因在设备内部。

📘 讲义：PL011 维护 `int_level` 与 `int_enabled`，再计算最终输出：

```c
flags = s->int_level & s->int_enabled;
qemu_set_irq(s->irq[i], (flags & irqmask[i]) != 0);
```

这段代码的关键思想是：

```text
中断原因 int_level
  & 中断使能 int_enabled
  -> 最终是否拉高 IRQ 线
```

🧠 我的理解：不要把“状态位”和“IRQ 线”混成一个变量。状态位是设备内部原因；IRQ 线是向外部中断控制器报告的电平。

G233 GPIO 可以这样设计：

```text
pin 变化
  -> 根据 TRIG/POL 判断是否满足触发条件
  -> 如果 IE 允许，设置 IS
  -> active = (IS & IE) != 0
  -> qemu_set_irq(s->irq, active)
```

### 5.2 PLIC pending 不是设备自己直接写出来的

在 RISC-V SoC 里，外设一般把 IRQ 线接到 PLIC。设备调用 `qemu_set_irq()` 拉高线，PLIC 模型看到输入变化后设置 pending。

```text
G233 GPIO
  -> qemu_set_irq(gpio_irq, 1)
  -> PLIC input IRQ 2 active
  -> PLIC_PENDING 对应 bit 变成 1
  -> qtest 读 PLIC_PENDING 看到 pending
```

🔗 和 SoC 训练营的关系：`test-gpio-int` 检查 GPIO PLIC IRQ 2，`test-wdt-timeout` 检查 WDT PLIC IRQ 4，`test-spi-overrun` 检查 SPI PLIC IRQ 5。

⚠️ 容易踩坑：在 GPIO 设备里直接改 PLIC 内部状态。这是错误方向。设备应该输出 IRQ 线；PLIC 负责 pending/claim/complete。

### 5.3 timer：虚拟时间驱动设备状态

PWM 和 WDT 不能只靠寄存器保存值。它们的行为依赖时间。

qtest 中的时间推进：

```c
qtest_clock_step(qts, 100000000);
```

含义是推进 QEMU 虚拟时钟。你的设备如果用 `QEMUTimer` 挂在正确时钟上，timer callback 就会在虚拟时间到期时运行。

WDT 的典型流程：

```text
write WDT_LOAD
write WDT_CTRL.EN
  -> timer_mod() 安排 timeout
qtest_clock_step()
  -> timer callback 到期
  -> s->sr |= WDT_SR_TIMEOUT
  -> 如果 INTEN，qemu_set_irq(s->irq, 1)
```

PWM 的典型流程：

```text
write PERIOD/DUTY
write CTRL.EN
  -> 启动 channel timer
qtest_clock_step()
  -> counter 增长
  -> period 完成后设置 DONE
```

🧩 补充知识：`QEMUTimer` 是软件调度机制；`Clock` QOM 对象是硬件时钟树建模。下一章“时钟系统”会更系统地讲。本章先掌握：需要时间推进的外设，不能只用普通变量。

### 5.4 bus side effect：SPI controller 不是 flash

SPI 题最容易混淆。SPI controller 和 SPI flash 是两个东西。

```text
guest
  -> 写 SPI controller 的 SPI_DR
  -> SPI controller 根据 CR2 选择 CS0/CS1
  -> 把 byte 发送给对应 flash 模型
  -> flash 返回一个 byte
  -> controller 更新 RXNE/TXE/OVERRUN
```

SPI controller 负责：

- `CR1`：enable、master、interrupt enable。
- `CR2`：chip select。
- `SR`：RXNE、TXE、OVERRUN。
- `DR`：发送/接收数据寄存器。

flash 负责：

- JEDEC ID。
- READ STATUS。
- READ DATA。
- PAGE PROGRAM。
- SECTOR ERASE。
- busy 状态。
- 每片 flash 的独立存储容量和内容。

🔗 和 SoC 训练营的关系：`test-spi-cs` 会验证 CS0/CS1 两片 flash 的 JEDEC ID、容量、数据隔离。如果你只在 SPI controller 里用一个全局 buffer 假装 flash，很可能过不了这个测试。

✅ 你现在就做：每次写副作用逻辑时，把它归类。

```text
IRQ side effect：状态变化后拉高/拉低中断线
timer side effect：状态变化后启动/停止/重排 timer
bus side effect：状态变化后访问下游设备
```

⚠️ 容易踩坑：把副作用直接散落在每个 read/write case 里。更好的方式是集中到 `update_irq()`、`update_timer()`、`do_transfer()` 这类函数。

你应该能回答：

- 为什么 IRQ 线不是中断原因本身？
- 为什么 `qtest_clock_step` 能测 PWM/WDT？
- SPI controller 和 SPI flash 的职责边界是什么？

答不上来回看：`5.1` 到 `5.4`。

---

## 6. 对 SoC 测题的直接帮助

这一节按 qtest 反推设备模型。你后面做题时，先读测试，再回到这里找对应能力。

### 6.1 `test-gpio-basic`

测试覆盖：

- `g233/gpio/reset_value`
- `g233/gpio/direction`
- `g233/gpio/output`
- `g233/gpio/multi_pin`

寄存器：

| offset | name | 语义 |
| --- | --- | --- |
| `0x00` | `GPIO_DIR` | 方向，`0=input`，`1=output` |
| `0x04` | `GPIO_OUT` | 输出数据 |
| `0x08` | `GPIO_IN` | 输入数据，只读视图 |
| `0x0C` | `GPIO_IE` | 中断使能 |
| `0x10` | `GPIO_IS` | 中断状态，W1C |
| `0x14` | `GPIO_TRIG` | `0=edge`，`1=level` |
| `0x18` | `GPIO_POL` | `0=low/falling`，`1=high/rising` |

🎥 视频/作者：视频强调“状态结构就是设备本体”。GPIO basic 就是在检查最基础的状态是否可靠。

🧠 我的理解：这题不是考复杂中断，而是考你能否把最基本的寄存器状态和派生读写做对。

实现思路：

```c
typedef struct G233GPIOState {
    SysBusDevice parent_obj;
    MemoryRegion iomem;
    uint32_t dir;
    uint32_t out;
    uint32_t ie;
    uint32_t is;
    uint32_t trig;
    uint32_t pol;
    qemu_irq irq;
} G233GPIOState;
```

read 关键点：

```c
case GPIO_IN:
    return gpio_compute_input(s);
```

`GPIO_IN` 应该能反映 output pin 的 `OUT` 值。简单模型里可以先做：

```c
static uint32_t gpio_compute_input(G233GPIOState *s)
{
    return s->out & s->dir;
}
```

🔗 和 SoC 训练营的关系：`test_gpio_output` 会先写 `DIR=1`，再写 `OUT=1`，最后读 `IN & 1` 期望为 `1`。这要求 `GPIO_IN` 不是独立死变量。

✅ 你现在就做：先保证 reset 和 `OUT -> IN` 回读过，再碰 interrupt。

⚠️ 容易踩坑：把 `GPIO_IN` 当普通 RW 寄存器。测试没有写 `GPIO_IN`，它必须由设备内部状态计算出来。

### 6.2 `test-gpio-int`

测试覆盖：

- `g233/gpio-int/edge_rising`
- `g233/gpio-int/level_high`
- `g233/gpio-int/is_clear`
- `g233/gpio-int/ie_mask`
- `g233/gpio-int/plic`

核心要求：

| 场景 | 期望 |
| --- | --- |
| edge + rising | `OUT` 从 0 到 1 后，`IS` bit 置位 |
| level + high | pin 高时 `IS` 置位，pin 低后清除 |
| W1C | 写 `GPIO_IS=1` 清状态 |
| IE mask | `IE=0` 时不设置或不输出 interrupt |
| PLIC | GPIO IRQ 2 出现在 PLIC pending |

🧠 我的理解：GPIO interrupt 至少有四层，不要压成一个 bool。

```text
pin_value：当前引脚电平
prev_pin_value：上一拍引脚电平，用来判断 edge
IS：设备内部中断状态
IE：中断使能
IRQ line：IS & IE 后输出给 PLIC
```

推荐流程：

```c
static void gpio_update_pin_state(G233GPIOState *s)
{
    uint32_t old = s->prev_in;
    uint32_t now = gpio_compute_input(s);
    uint32_t changed = old ^ now;
    uint32_t active = 0;

    uint32_t edge_mode = ~s->trig;
    uint32_t level_mode = s->trig;

    uint32_t rising = changed & now & s->pol;
    uint32_t falling = changed & ~now & ~s->pol;
    active |= edge_mode & (rising | falling);

    uint32_t high = now & s->pol;
    uint32_t low = ~now & ~s->pol;
    active |= level_mode & (high | low);

    s->is |= active & s->ie;

    if (level_mode) {
        s->is = (s->is & ~level_mode) | (active & level_mode & s->ie);
    }

    s->prev_in = now;
    gpio_update_irq(s);
}
```

这段是思路模板，不是要求逐字照抄。重点是分清 edge 和 level。

PLIC 链路：

```text
qtest_writel(GPIO_OUT, 0x1)
  -> gpio_write offset 0x04
  -> gpio_update_pin_state()
  -> s->is |= 0x1
  -> gpio_update_irq()
  -> qemu_set_irq(s->irq, 1)
  -> PLIC IRQ 2 pending
  -> qtest_readl(PLIC_PENDING) 看到 bit
```

🔗 和 SoC 训练营的关系：`test_gpio_plic` 还会 claim/complete PLIC。设备侧只负责 IRQ 线，PLIC 的 claim/complete 由 PLIC 模型负责。

✅ 你现在就做：先让 `edge_rising`、`is_clear` 过，再处理 `level_high`，最后接 PLIC。

⚠️ 容易踩坑：IE mask 的语义要按测试实现。当前测试要求 IE 关闭时不设置 `IS`。不要按另一种硬件规格写成“IS 总是置位，只是不输出 IRQ”，否则会和测试冲突。

### 6.3 `test-pwm-basic`

测试覆盖：

- `g233/pwm/config`
- `g233/pwm/enable`
- `g233/pwm/counter`
- `g233/pwm/done_flag`
- `g233/pwm/done_clear`
- `g233/pwm/multi_channel`
- `g233/pwm/polarity`

寄存器：

| offset | name | 语义 |
| --- | --- | --- |
| `0x00` | `PWM_GLB` | `CHn_EN` mirror，`CHn_DONE` W1C |
| `0x10+n*0x10+0x00` | `CHn_CTRL` | `EN`、`POL` |
| `0x10+n*0x10+0x04` | `CHn_PERIOD` | 周期 |
| `0x10+n*0x10+0x08` | `CHn_DUTY` | 占空 |
| `0x10+n*0x10+0x0C` | `CHn_CNT` | counter，只读 |

🧠 我的理解：PWM 是“多 channel state + timer”的练习。不要写 4 份重复字段，应该数组化。

```c
typedef struct G233PWMChannel {
    uint32_t ctrl;
    uint32_t period;
    uint32_t duty;
    uint32_t cnt;
    bool done;
    QEMUTimer timer;
} G233PWMChannel;

typedef struct G233PWMState {
    SysBusDevice parent_obj;
    MemoryRegion iomem;
    G233PWMChannel ch[4];
} G233PWMState;
```

`PWM_GLB` read 应该计算：

```c
static uint32_t pwm_read_glb(G233PWMState *s)
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

`PWM_GLB` write 只清 DONE：

```c
for (int i = 0; i < 4; i++) {
    if (value & (1u << (4 + i))) {
        s->ch[i].done = false;
    }
}
```

🔗 和 SoC 训练营的关系：`test_pwm_counter` 和 `test_pwm_done_flag` 都会 `qtest_clock_step`。如果没有 timer，counter 不会变，DONE 也不会置位。

✅ 你现在就做：先实现 config、enable、multi_channel，再加 timer 支持 counter/done。

⚠️ 容易踩坑：把 `PWM_GLB.CH_EN` 当可写独立 bit。测试期望它反映 `CHn_CTRL.EN`，它是 mirror，不是独立配置。

### 6.4 `test-wdt-timeout`

测试覆盖：

- `g233/wdt/config`
- `g233/wdt/countdown`
- `g233/wdt/feed`
- `g233/wdt/timeout_flag`
- `g233/wdt/timeout_clear`
- `g233/wdt/lock`
- `g233/wdt/interrupt`

寄存器：

| offset | name | 语义 |
| --- | --- | --- |
| `0x00` | `WDT_CTRL` | `EN`、`INTEN` |
| `0x04` | `WDT_LOAD` | reload value |
| `0x08` | `WDT_VAL` | 当前 counter，只读 |
| `0x0C` | `WDT_KEY` | magic write：feed / lock |
| `0x10` | `WDT_SR` | `TIMEOUT`，W1C |

🧠 我的理解：WDT 是“小状态机 + timer + magic key”。

```text
disabled
  -> write LOAD
  -> write CTRL.EN
running
  -> timer 推进，VAL 下降
  -> KEY_FEED，VAL 重装
  -> timeout，SR.TIMEOUT 置位
locked
  -> 后续 CTRL 写入被限制
```

关键行为：

```c
case WDT_KEY:
    if (value == WDT_KEY_FEED) {
        wdt_reload(s);
    } else if (value == WDT_KEY_LOCK) {
        s->locked = true;
    }
    break;
```

timeout callback：

```c
static void wdt_timeout(void *opaque)
{
    G233WDTState *s = opaque;

    s->sr |= WDT_SR_TIMEOUT;
    if (s->ctrl & WDT_CTRL_INTEN) {
        qemu_set_irq(s->irq, 1);
    }
}
```

🔗 和 SoC 训练营的关系：`test_wdt_lock` 会写 lock key 后尝试 `WDT_CTRL=0`，期望 `EN` 仍然保持。这说明 lock 改变的是后续 write 行为。

✅ 你现在就做：实现顺序建议：config -> countdown -> feed -> timeout -> W1C -> lock -> interrupt。

⚠️ 容易踩坑：`WDT_VAL` 不能只是 load 的副本。`qtest_clock_step` 后它必须下降；feed 后它必须变大。

### 6.5 `test-spi-jedec`

测试覆盖：

- `g233/spi/init`
- `g233/spi/jedec_id`
- `g233/spi/transfer_byte`

寄存器：

| offset | name | 语义 |
| --- | --- | --- |
| `0x00` | `SPI_CR1` | `SPE`、`MSTR`、interrupt enable |
| `0x04` | `SPI_CR2` | CS select |
| `0x08` | `SPI_SR` | `RXNE`、`TXE`、`OVERRUN` |
| `0x0C` | `SPI_DR` | data register，写触发传输，读取接收数据 |

测试中的 transfer：

```text
wait TXE
write SPI_DR = tx
wait RXNE
read SPI_DR -> rx
```

🧠 我的理解：`SPI_DR` 是 SPI controller 的动作入口。写它表示“发出一个 byte”；读它表示“取走收到的 byte”。

最小 transfer 模型：

```c
static void spi_transfer_byte(G233SPIState *s, uint8_t tx)
{
    uint8_t rx;

    if (!(s->cr1 & SPI_CR1_SPE) || !(s->cr1 & SPI_CR1_MSTR)) {
        return;
    }

    if (s->sr & SPI_SR_RXNE) {
        s->sr |= SPI_SR_OVERRUN;
    }

    rx = g233_spi_flash_xfer(s, s->cr2 & 0x3, tx);
    s->rx = rx;
    s->sr |= SPI_SR_RXNE | SPI_SR_TXE;
    spi_update_irq(s);
}
```

🔗 和 SoC 训练营的关系：JEDEC ID 命令是 `0x9F`，CS0 上 W25X16 期望返回 `{0xEF, 0x30, 0x15}`。

✅ 你现在就做：先只支持 CS0 和 JEDEC ID，把 `test-spi-jedec` 跑通，再扩 CS。

⚠️ 容易踩坑：写 `SPI_DR` 后只保存 `tx`，没有生成 `rx` 和设置 `RXNE`。这样 `spi_wait_rxne` 会超时。

### 6.6 `test-spi-cs`

测试覆盖：

- `g233/spi-cs/identification`
- `g233/spi-cs/individual`
- `g233/spi-cs/cross`
- `g233/spi-cs/alternating`
- `g233/spi-cs/capacity`
- `g233/spi-cs/concurrent_status`

核心要求：

| CS | Flash | JEDEC | 容量 |
| --- | --- | --- | --- |
| CS0 | W25X16 | `0xEF3015` | 2 MiB |
| CS1 | W25X32 | `0xEF3016` | 4 MiB |

🧠 我的理解：这题验证 SPI controller 是否真正把 CS 路由到不同 flash，而不是所有数据都进一个全局状态。

flash 子状态应该独立：

```c
typedef struct G233SPIFlash {
    uint8_t jedec[3];
    uint8_t *storage;
    uint32_t size;
    uint8_t status;
    bool write_enable;
    /* command parser state */
} G233SPIFlash;

typedef struct G233SPIState {
    SysBusDevice parent_obj;
    MemoryRegion iomem;
    uint32_t cr1;
    uint32_t cr2;
    uint32_t sr;
    uint8_t rx;
    G233SPIFlash flash[2];
    qemu_irq irq;
} G233SPIState;
```

CS 路由：

```c
static G233SPIFlash *spi_selected_flash(G233SPIState *s)
{
    return &s->flash[s->cr2 & 0x1];
}
```

🔗 和 SoC 训练营的关系：`cross` 和 `alternating` 会在两片 flash 之间切换访问。如果 CS 切换不重置或不隔离 flash command parser，很容易串状态。

✅ 你现在就做：给每片 flash 独立维护 command parser、地址、busy/status、storage。

⚠️ 容易踩坑：CS 变化时没有结束上一片 flash 的 command transaction。测试里会通过切换 CS 模拟片选释放，flash 状态机应能回到合理边界。

### 6.7 `test-spi-overrun`

测试覆盖：

- `g233/spi-overrun/interrupt`
- `g233/spi-overrun/polling`

核心语义：

```text
如果 RXNE=1，说明上一个接收字节还没被软件读走。
这时新 byte 又完成接收，应该设置 OVERRUN。
如果 ERRIE 打开，还应该触发 SPI IRQ 5。
```

流程：

```text
write SPI_DR = 0xAA
  -> RXNE = 1
不读 SPI_DR
write SPI_DR = 0xBB
  -> 发现 RXNE 仍为 1
  -> SR.OVERRUN = 1
  -> 如果 CR1.ERRIE，qemu_set_irq(irq, 1)
```

W1C 清除：

```c
case SPI_SR:
    s->sr &= ~(value & SPI_SR_OVERRUN);
    spi_update_irq(s);
    break;
```

🔗 和 SoC 训练营的关系：`test_interrupt_overrun_detection` 会打开 `SPI_CR1_ERRIE` 并检查 PLIC IRQ 5；`test_polling_overrun_detection` 不开 interrupt，只轮询 `SPI_SR.OVERRUN`，再 W1C 清除。

✅ 你现在就做：先用一个 byte 的 RX buffer 建模。只有当 `read SPI_DR` 时清 `RXNE`，这样 overrun 条件才自然出现。

⚠️ 容易踩坑：每次写 `SPI_DR` 都覆盖 `rx`，但不检查旧 `RXNE`。这样永远不会产生 overrun。

---

## 7. 最小练习

### 练习 1：手工追 PL011 外设建模链路

✅ 你现在就做：按这个顺序在源码里找。

```text
TYPE_PL011
  -> TypeInfo
  -> PL011State
  -> pl011_init
  -> memory_region_init_io
  -> pl011_ops
  -> pl011_read / pl011_write
  -> pl011_update
  -> machine 中 qdev_new / sysbus_realize / map / connect irq
```

参考思路：

| 找到的东西 | 说明 |
| --- | --- |
| `TypeInfo` | 设备类型如何注册 |
| `PL011State` | 设备状态有哪些 |
| `pl011_init` | MMIO/IRQ/clock 骨架如何建立 |
| `pl011_ops` | guest 访问如何进入回调 |
| `pl011_update` | 中断状态如何投影到 IRQ 线 |
| machine 集成 | 设备如何出现在地址空间 |

### 练习 2：给 GPIO 写寄存器语义表

✅ 你现在就做：不要写代码，先填表。

| 寄存器 | reset | read | write | 副作用 |
| --- | --- | --- | --- | --- |
| `DIR` | `0` | 返回方向 | 保存 32-bit | 影响 `IN` 派生 |
| `OUT` | `0` | 返回输出 | 保存 32-bit | 可能触发中断 |
| `IN` | `0` | 计算输入视图 | 忽略 | 无 |
| `IE` | `0` | 返回使能 | 保存 32-bit | 更新 IRQ |
| `IS` | `0` | 返回状态 | W1C | 更新 IRQ |
| `TRIG` | `0` | 返回模式 | 保存 32-bit | 影响后续触发判断 |
| `POL` | `0` | 返回极性 | 保存 32-bit | 影响后续触发判断 |

### 练习 3：从 qtest 追一次 `test-gpio-int`

✅ 你现在就做：把下面链路默写出来。

```text
qtest_writel(GPIO_DIR, 1)
qtest_writel(GPIO_TRIG, 0)
qtest_writel(GPIO_POL, 1)
qtest_writel(GPIO_IE, 1)
qtest_writel(GPIO_OUT, 0)
qtest_writel(GPIO_OUT, 1)
  -> output 发生 0->1
  -> rising edge 条件成立
  -> IS bit0 置位
  -> IE bit0 允许
  -> GPIO IRQ line 拉高
  -> PLIC IRQ 2 pending
```

### 练习 4：从 qtest 追一次 `test-spi-overrun`

✅ 你现在就做：检查自己能否说清每个状态位。

```text
初始：TXE=1，RXNE=0，OVERRUN=0
write DR 0xAA
  -> RXNE=1
不 read DR
write DR 0xBB
  -> 发现 RXNE 仍为 1
  -> OVERRUN=1
  -> ERRIE=1 时 IRQ 5 pending
write SR OVERRUN
  -> W1C 清 OVERRUN
```

### 练习 5：实现前检查清单

每做一个新外设，先填这 10 项。

```text
1. TYPE 名字是什么？
2. parent 是否是 TYPE_SYS_BUS_DEVICE？
3. State 里有哪些寄存器影子？
4. 哪些寄存器是派生值？
5. reset 默认值是什么？
6. MemoryRegion 大小是多少？
7. read/write 支持哪些 offset？
8. 哪些寄存器有 W1C / RO / mirror / trigger？
9. 有没有 IRQ/timer/bus side effect？
10. machine 里 base 和 IRQ 号是多少？
```

⚠️ 容易踩坑：直接开写通常会快 10 分钟，然后 debug 慢 3 小时。先做表，尤其是 W1C、timer、IRQ 这三类。

---

## 8. 下一章怎么接

本章解决的是“单个外设内部怎么建模”。后面还要接三块知识。

| 下一步 | 解决什么问题 | 和本章关系 |
| --- | --- | --- |
| 主板建模流程 | 怎么把 CPU、RAM、PLIC、外设组成一块 board | 本章设备必须由 machine 创建和连线 |
| 时钟系统 | QEMUClock/QEMUTimer/Clock 怎么工作 | PWM/WDT/SPI busy 需要时间推进 |
| kernel 运行机制 | guest driver 如何发现和访问设备 | 设备树、MMIO、IRQ 最终给 Linux driver 用 |

🎥 视频/作者：视频最后强调，写完设备内部逻辑还不够，必须把设备集成到 machine：创建、设置属性、realize、映射 MMIO、连接 IRQ、写设备树。

📘 讲义：PL011 在 ARM virt 机型里会被创建、映射地址、连接中断，并写入设备树，包括 `compatible`、`reg`、`interrupts`、`clock-names`、`stdout-path`。

🧠 我的理解：设备模型和主板模型是两半。设备模型回答“这个外设如何响应访问”；主板模型回答“这个外设在哪里、接到谁、guest 如何发现它”。

🔗 和 SoC 训练营的关系：如果某个 qtest 读写地址完全没进入设备回调，优先查 machine 映射；如果进入回调但返回错，查设备寄存器语义；如果寄存器对但 PLIC 不 pending，查 IRQ 连接和 `qemu_set_irq`。

✅ 你现在就做：下一章学习主板建模时，只盯这 4 个动作：

```text
qdev_new
sysbus_realize_and_unref
memory_region_add_subregion
sysbus_connect_irq
```

⚠️ 容易踩坑：不要把所有失败都归因于设备 read/write。SoC 题里至少一半问题是“设备没被正确放进 board”。

---

## 附录 A：术语表

| 术语 | 解释 |
| --- | --- |
| QOM | QEMU Object Model，QEMU 的对象系统，用 C 实现类似类/对象/继承的机制 |
| `TypeInfo` | 描述一个 QOM 类型如何注册、继承、初始化 |
| `DeviceState` | QEMU 设备基类，负责设备生命周期、属性、reset、realize 等 |
| `SysBusDevice` | 系统总线设备，适合 SoC 内部 MMIO 外设 |
| state struct | 设备实例的内部状态，通常第一个字段是父类对象 |
| `instance_init` | 对象实例创建时调用，适合搭 MMIO/IRQ/clock 骨架 |
| `class_init` | 类型初始化时调用，适合绑定 realize/reset/vmstate/properties |
| `realize` | 设备正式激活阶段，适合检查属性、绑定后端资源 |
| `reset` | 设备回到硬件复位状态 |
| `MemoryRegion` | QEMU 地址空间中的一段区域，可以是 RAM、ROM、MMIO |
| `MemoryRegionOps` | MMIO 读写函数表 |
| `opaque` | read/write 回调里的用户指针，通常指向设备 state |
| `offset` | guest 访问地址相对于设备 MMIO base 的偏移 |
| `qemu_irq` | QEMU 表示中断线的对象 |
| `qemu_set_irq` | 拉高或拉低 IRQ 线 |
| PLIC | RISC-V Platform-Level Interrupt Controller，平台级中断控制器 |
| `QEMUTimer` | QEMU 定时器，用虚拟时间或宿主时间驱动未来事件 |
| W1C | write-one-to-clear，写 1 清除对应状态位 |
| mirror bit | 镜像其他状态的 bit，通常 read 时计算 |
| write trigger | 写寄存器触发动作，而不是简单保存值 |
| qtest | QEMU 的测试框架，可通过协议读写 guest 地址和推进虚拟时间 |

---

## 附录 B：本章检查题

### 基础检查

1. `DeviceState` 和 `SysBusDevice` 的区别是什么？
2. 为什么 SoC 内部 GPIO 通常建模成 `SysBusDevice`，而不是 PCI 设备？
3. `memory_region_init_io` 和 `memory_region_add_subregion` 分别解决什么问题？
4. write 回调里的 `offset` 为什么不是绝对物理地址？
5. 为什么设备 state 不应该机械照抄寄存器表？

### 寄存器语义检查

1. W1C 应该怎么写？为什么不能 `reg = value`？
2. `GPIO_IN` 为什么可能是派生值？
3. `PWM_GLB.CH_EN` 为什么是 mirror bit？
4. `WDT_KEY` 为什么是 write trigger？
5. `SPI_DR` 的 read 和 write 分别有什么副作用？

### IRQ/timer/bus 检查

1. 中断原因、中断使能、IRQ 线三者是什么关系？
2. GPIO edge interrupt 为什么需要保存上一拍 pin 值？
3. `qtest_clock_step` 为什么能验证 PWM/WDT？
4. WDT timeout 后应该更新哪些状态？
5. SPI controller 和 SPI flash 的状态为什么要分开？

### SoC 测试检查

1. `test-gpio-basic` 真正在验证什么？
2. `test-gpio-int` 中，从 `GPIO_OUT 0->1` 到 PLIC pending 的完整链路是什么？
3. `test-pwm-basic` 为什么要求 `PWM_GLB` 既有 mirror bit 又有 W1C bit？
4. `test-wdt-timeout` 中 lock key 改变了哪个行为？
5. `test-spi-jedec` 为什么写 `SPI_DR` 后必须设置 `RXNE`？
6. `test-spi-cs` 为什么能抓出 CS0/CS1 状态没有隔离的问题？
7. `test-spi-overrun` 为什么必须在读 `SPI_DR` 时清 `RXNE`？

### 参考答案方向

如果答不上来，按下面回看：

| 问题类型 | 回看章节 |
| --- | --- |
| QOM / 类型 / 生命周期 | 第 1 节 |
| state 设计 / reset | 第 2 节 |
| MMIO / offset / qtest 链路 | 第 3 节 |
| W1C / mirror / trigger | 第 4 节 |
| IRQ / timer / bus side effect | 第 5 节 |
| G233 qtest 具体映射 | 第 6 节 |

---

## 最后总评

这章的核心不是 PL011，也不是某个 API 名字，而是这条工程路线：

```text
寄存器规格
  -> state 设计
  -> MemoryRegionOps
  -> read/write 语义
  -> IRQ/timer/bus 副作用
  -> machine 映射和连线
  -> qtest 验证
```

你只要能用这条路线解释 GPIO/PWM/WDT/SPI 的每个测试，就已经不是在“看 QEMU 源码”，而是在按 QEMU 的模型写硬件。
