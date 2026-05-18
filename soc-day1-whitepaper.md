# Day 1 白皮书：从会用 QEMU 到能改 G233 SoC

## 🎯 Day 1 总目标

Day 1 不是为了写完任何外设，也不是为了把 QEMU 源码通读一遍。

Day 1 的目标是建立后面 6 天持续推进 SoC 实验的“开发地图”：

```text
知道训练营 SoC 要做什么
知道 G233 machine 从哪里启动
知道外设如何挂到 QEMU 地址空间
知道 QTest 如何验证 MMIO 寄存器
知道 10 个 SoC 测试分别在测什么
知道 Day 2 从 GPIO basic 的哪几个寄存器下手
```

今天结束时，你不需要会写完整外设，但必须能回答：

1. `-machine g233` 最终会进入哪个 machine init 函数？
2. `qtest_writel(qts, 0x10012000, value)` 为什么应该进入 GPIO 的 `write` 回调？
3. 一个最小 QEMU `SysBusDevice` 外设需要哪些部件？
4. 为什么 SoC 方向不需要先写 guest driver？
5. SoC 10 个基础测试分别验证哪些外设行为？
6. Day 2 写 GPIO basic 时，哪些寄存器必须先实现？

✅ 你现在就做：

- 先把这份文档完整扫一遍。
- 然后严格按“今日路线图”执行。
- 遇到失败，先填日志，不要立刻改代码。

---

## 🧭 你的背景对应的学习策略

你有 Linux 驱动和 MCU 基础，所以 Day 1 不需要重新学：

- GPIO 是什么。
- SPI 是什么。
- WDT 是什么。
- 什么是寄存器。
- 为什么驱动里会用 `readl/writel`。

Day 1 真正要学的是 **把真实 SoC 经验翻译成 QEMU 设备模型语言**：

```text
Linux driver: ioremap + readl/writel
QEMU device: MemoryRegion + MemoryRegionOps.read/write

硬件 IRQ 输出脚
QEMU qemu_irq + qemu_set_irq()

真实硬件定时器
QEMU virtual clock + timer

真实板级地址图
QEMU machine init + memory_region_add_subregion()

驱动测试外设
QTest 直接访问设备模型地址空间
```

🧠 我的理解：

你今天最值钱的收获不是“知道 QEMU 有很多 API”，而是建立这个映射关系：

```text
真实硬件寄存器行为
-> QEMU 设备状态结构体
-> MemoryRegionOps read/write
-> IRQ / timer / bus 副作用
-> QTest 自动化验证
```

这条线一旦打通，GPIO、PWM、WDT、SPI Flash 都只是不同复杂度的同一类问题。

⚠️ 容易踩坑：

- 不要把 QEMU 外设当成 Linux 驱动写。QEMU 里你写的是“虚拟硬件本体”，不是驱动。
- 不要一上来想“完整模拟硬件手册”。训练营测试先要求可验证的最小语义。
- 不要先读完整 QEMU 源码。今天只读 G233 machine、QTest 和 GPIO basic 相关入口。

---

## 📚 Day 1 资料清单

只看下面这些，不要扩散。

### 官方资料

- Machine model: <https://qemu.gevico.online/tutorial/2025/ch2/sec4/c-model/machine-model/>
- QEMU 外设建模流程: <https://qemu.gevico.online/tutorial/2026/ch2/qemu-hw/>
- QTest: <https://qemu.gevico.online/tutorial/2025/ch2/sec6/qtest/>
- SoC 实验手册: <https://qemu.gevico.online/exercise/2026/stage1/soc/g233-exper-manual/>
- G233 SoC 硬件手册: <https://qemu.gevico.online/exercise/2026/stage1/soc/g233-datasheet/>

### 本地文件

- [README_zh.md](../README_zh.md)
- [Makefile.camp](../Makefile.camp)
- [exper-note/soc-7day-plan.md](./soc-7day-plan.md)
- [hw/riscv/g233.c](../hw/riscv/g233.c)
- [include/hw/riscv/g233.h](../include/hw/riscv/g233.h)
- [tests/gevico/qtest/meson.build](../tests/gevico/qtest/meson.build)
- [tests/gevico/qtest/test-board-g233.c](../tests/gevico/qtest/test-board-g233.c)
- [tests/gevico/qtest/test-gpio-basic.c](../tests/gevico/qtest/test-gpio-basic.c)
- [tests/gevico/qtest/test-gpio-int.c](../tests/gevico/qtest/test-gpio-int.c)
- [tests/gevico/qtest/test-wdt-timeout.c](../tests/gevico/qtest/test-wdt-timeout.c)
- [tests/gevico/qtest/test-spi-jedec.c](../tests/gevico/qtest/test-spi-jedec.c)
- [exper-note/qemu-hw-study.md](./qemu-hw-study.md)

📘 看资料的原则：

- 白天以本地代码和测试为准。
- 晚上再看 `qemu-hw-study.md` 补概念。
- 看到不懂的 QEMU API，先记下来，不要中断主线。

---

## 🗺️ 今日路线图

```text
09:00-09:30  建立任务边界
09:30-10:30  理解 machine model
10:30-11:30  读 G233 板级代码
11:30-12:00  理解 QTest 如何打 MMIO
13:30-15:00  configure/build
15:00-16:00  跑 test-soc 基线
16:00-17:30  读 SoC 测试总览
19:00-20:30  看 qemu-hw-study 的外设骨架
20:30-21:30  提取 GPIO basic 行为表
```

🎯 今日最小产出：

```text
构建结果
SoC 10 题失败矩阵
G233 地址空间表
QTest -> MemoryRegionOps 调用链
SoC 测试对照表
GPIO basic 行为表
Day 2 开工检查表
```

如果今天时间不够，优先级如下：

```text
必须完成：configure/build/test-soc + 失败矩阵
尽量完成：G233 地址图 + GPIO basic 行为表
可以推迟：完整阅读 qemu-hw-study
```

---

## 1. 先建立任务边界

### 📘 你要看什么

- [README_zh.md](../README_zh.md)
- [Makefile.camp](../Makefile.camp)
- [exper-note/soc-7day-plan.md](./soc-7day-plan.md)

### 🎯 你要提取什么

今天只提取 3 件事：

```text
SoC 测试在哪里：
tests/gevico/qtest/

SoC 测试怎么跑：
make -f Makefile.camp test-soc

SoC 基础测试有哪些：
test-board-g233
test-gpio-basic
test-gpio-int
test-pwm-basic
test-wdt-timeout
test-spi-jedec
test-flash-read
test-flash-read-interrupt
test-spi-cs
test-spi-overrun
```

🧠 我的理解：

SoC 方向不是让你写 Linux 驱动，而是给 QEMU 的 G233 machine 补虚拟外设模型。

测试不需要 guest Linux，不需要先写 driver。QTest 会在 host 侧启动 QEMU，然后直接对 guest 物理地址空间读写。

```text
QTest 进程
-> qtest_writel / qtest_readl
-> QEMU 地址空间
-> 设备 MemoryRegionOps
-> 断言寄存器行为是否正确
```

⚠️ 容易踩坑：

- 不要把 SoC 实验理解成“写驱动让 Linux 能用 GPIO”。
- Day 1 不改测试，也不改设备代码。
- 第一轮 `test-soc` 失败是正常的，失败矩阵本身就是今天的产出。

✍️ 你要填写：

```text
SoC 测试入口：
SoC 测试数量：
今天最先要跑的命令：
今天不做的事情：
```

---

## 2. 看 Machine model：`-machine g233` 怎么进代码

### 📘 你要看什么

- Machine model: <https://qemu.gevico.online/tutorial/2025/ch2/sec4/c-model/machine-model/>
- [hw/riscv/g233.c](../hw/riscv/g233.c)
- [include/hw/riscv/g233.h](../include/hw/riscv/g233.h)

### 🔍 本地搜索

在外层实验仓库执行：

```bash
cd "/home/flamboy/qemu-camp-2026-exper-flamboyante"
rg -n "TYPE_RISCV_G233_MACHINE|virt_machine_init|machine_class_init|mc->init" \
  "hw/riscv/g233.c" "include/hw/riscv/g233.h"
```

### 🧠 你要理解什么

QEMU 不是看到 `-machine g233` 就随便调用一个函数。它大致走这条线：

```text
main()
-> qemu_init()
-> 解析 -machine g233
-> 创建 MachineState
-> machine_run_board_init()
-> MachineClass.init()
-> G233 machine init
```

这里的关键是：

```text
machine init 负责创建这块板：
CPU
DRAM
中断控制器
CLINT / APLIC / PLIC
UART
VirtIO
Flash
后续你新增的 GPIO / WDT / PWM / SPI
```

🔗 和 SoC 实验的关系：

Day 2 之后你写的 GPIO/WDT/SPI 不是孤立文件。你最终必须把设备接到 G233 machine 里：

```text
qdev_new()
-> sysbus_realize_and_unref()
-> sysbus_mmio_map()
-> sysbus_connect_irq()
```

也就是说：

```text
设备文件负责“这个外设内部怎么工作”
g233.c 负责“这个外设挂到板上哪里”
```

⚠️ 容易踩坑：

- `g233.c` 很大，不要逐行读。
- 今天只找 machine init、地址表、中断连接、已有外设创建方式。
- 不要被 RISC-V AIA/PLIC/IMSIC 等分支细节拖走。今天只需要知道 IRQ 最终能接到中断控制器。

✍️ 你要填写：

```text
G233 machine 类型名：
G233 machine init 函数：
MachineClass.init 设置位置：
后续 GPIO 应该在哪里实例化：
```

---

## 3. 看 G233 板级代码：地址空间和已有外设

### 📘 你要看什么

- [include/hw/riscv/g233.h](../include/hw/riscv/g233.h)
- [hw/riscv/g233.c](../hw/riscv/g233.c)

### 🔍 本地搜索

```bash
rg -n "virt_memmap|VIRT_PLIC|VIRT_CLINT|VIRT_UART0|VIRT_DRAM|memory_region_add_subregion|sysbus_create_simple|qdev_get_gpio_in|qemu_set_irq|plic" \
  "hw/riscv/g233.c" "include/hw/riscv/g233.h"
```

### 🧠 你要理解什么

QEMU machine 本质上是在搭一张虚拟板级地址图。guest 或 QTest 访问某个物理地址时，QEMU 会根据地址空间找到对应的 `MemoryRegion`。

Day 1 你先建立这张简化图：

```text
0x02000000 CLINT
0x0C000000 PLIC
0x10000000 UART0
0x10001000 VirtIO MMIO slots
0x10010000 WDT      SoC 实验待实现
0x10012000 GPIO     SoC 实验待实现
0x10015000 PWM      SoC 实验待实现
0x10018000 SPI      SoC 实验待实现
0x20000000 pflash   已有并行 Flash 区域
0x80000000 DRAM
```

🔗 和 SoC 实验的关系：

SoC 测试里直接写固定地址，例如：

```text
GPIO_BASE = 0x10012000
WDT_BASE  = 0x10010000
PWM_BASE  = 0x10015000
SPI_BASE  = 0x10018000
```

如果你没有把设备映射到这些地址，测试访问时就不会进入你的外设回调。

⚠️ 容易踩坑：

- `0x20000000` 的 pflash 不是 SPI Flash 测试里的 W25X Flash。
- SPI Flash 测试要通过 `0x10018000` 的 SPI controller 间接访问 flash 状态机。
- 地址表里已有的 `VIRT_FLASH` 不等于你要实现的 SPI Flash 协议设备。

✅ 你现在就做：

- 在执行记录里填 `G233 简化地址空间表`。
- 每个地址都写一个证据位置，比如 `test-xxx.c define` 或 `g233.c virt_memmap`。

---

## 4. 看 QTest：为什么不用 guest driver

### 📘 你要看什么

- QTest: <https://qemu.gevico.online/tutorial/2025/ch2/sec6/qtest/>
- [tests/gevico/qtest/test-gpio-basic.c](../tests/gevico/qtest/test-gpio-basic.c)
- [tests/gevico/qtest/test-wdt-timeout.c](../tests/gevico/qtest/test-wdt-timeout.c)

### 🔍 本地命令

```bash
sed -n '1,160p' "tests/gevico/qtest/test-gpio-basic.c"
sed -n '1,180p' "tests/gevico/qtest/test-wdt-timeout.c"
```

### 🧠 你要理解什么

QTest 是 QEMU 的设备模型测试框架。它不是在 guest 里运行测试程序，而是在 host 侧控制 QEMU。

典型模式：

```c
QTestState *qts = qtest_init("-machine g233 -m 2G");
qtest_writel(qts, GPIO_DIR, 0x1);
uint32_t val = qtest_readl(qts, GPIO_DIR);
g_assert_cmpuint(val, ==, 0x1);
qtest_quit(qts);
```

这条调用链可以先这样记：

```text
qtest_writel(qts, addr, value)
-> QTest 协议把写请求发给 QEMU
-> QEMU 在系统地址空间里查找 addr
-> 命中某个 MemoryRegion
-> 调用 MemoryRegionOps.write(opaque, offset, value, size)
-> 设备状态结构体发生变化
```

🔗 和 SoC 实验的关系：

这就是为什么 Day 1 一定要读测试。测试文件就是硬件行为的“可执行规格”。

你后面实现设备时，不是凭感觉写寄存器，而是照测试提取：

```text
测试写了什么
测试读了什么
测试期待什么
这个期待对应设备 state 怎么变化
```

⚠️ 容易踩坑：

- `qtest_writel` 的 `addr` 是 guest 物理地址。
- `MemoryRegionOps.write` 收到的 `offset` 是相对设备 MMIO 基地址的偏移。
- 如果 GPIO 映射到 `0x10012000`，测试写 `0x10012004`，回调里看到的 offset 应该是 `0x04`。

✍️ 你要填写：

```text
addr 是：
offset 是：
GPIO_OUT 的 guest 物理地址：
GPIO_OUT 在回调里的 offset：
QTest 为什么不需要 guest driver：
```

---

## 5. 配置、编译和基线测试

### ✅ 你现在就做

在外层实验仓库执行：

```bash
cd "/home/flamboy/qemu-camp-2026-exper-flamboyante"
make -f Makefile.camp configure
make -f Makefile.camp build
make -f Makefile.camp test-soc
```

### 🧠 你要理解什么

第一轮基线不是为了证明你已经会了，而是为了确认：

```text
环境是否能配置
QEMU 是否能编译
SoC 测试能不能启动
现在 10 个测试分别失败在哪里
```

如果 `configure` 或 `build` 就失败，今天的重点先变成环境问题。不要跳过构建失败去读后面的测试。

### 常见失败分类

| 类型 | 典型现象 | 处理策略 |
| --- | --- | --- |
| 缺包 | `not found` / `dependency` | 记录缺的包名，集中安装 |
| Python/Meson | `meson` / `pyvenv` 错误 | 优先使用 `build/pyvenv` |
| Rust 工具链 | `cargo` / `bindgen` | 先记录，SoC 基础题不一定第一时间阻塞 |
| RISC-V 交叉工具链 | `riscv64-unknown-elf-gcc` | CPU 测试更依赖，先确认是否影响 SoC |
| QTest 失败 | `assert` / `abort` | 填失败矩阵，先不修 |

⚠️ 容易踩坑：

- 不要为了过 configure 乱改 `Makefile.camp`。
- 不要看到 10 个测试失败就慌。外设没实现时失败是正常现象。
- 日志不要全贴，记录最关键的 5-10 行即可。

✍️ 你要填写：

- `构建与测试命令记录`
- `SoC 10 题失败矩阵`
- `当前得分`

---

## 6. 读 SoC 测试总览

### 📘 你要看什么

- [tests/gevico/qtest/meson.build](../tests/gevico/qtest/meson.build)
- [tests/gevico/qtest/test-board-g233.c](../tests/gevico/qtest/test-board-g233.c)
- [tests/gevico/qtest/test-gpio-basic.c](../tests/gevico/qtest/test-gpio-basic.c)
- [tests/gevico/qtest/test-gpio-int.c](../tests/gevico/qtest/test-gpio-int.c)
- [tests/gevico/qtest/test-pwm-basic.c](../tests/gevico/qtest/test-pwm-basic.c)
- [tests/gevico/qtest/test-wdt-timeout.c](../tests/gevico/qtest/test-wdt-timeout.c)
- [tests/gevico/qtest/test-spi-jedec.c](../tests/gevico/qtest/test-spi-jedec.c)

今天不需要细读 SPI Flash 全部测试。先建立总览。

### 🔍 快速提取命令

```bash
rg -n "qtest_add_func|#define .*BASE|PLIC IRQ|register map|GPIO_BASE|WDT_BASE|PWM_BASE|SPI_BASE" \
  "tests/gevico/qtest"
```

### 🧠 你要理解什么

SoC 10 题不是随机排列。它们基本按外设复杂度递增：

```text
board
-> GPIO basic
-> GPIO interrupt
-> PWM
-> WDT
-> SPI JEDEC
-> Flash read/program/erase
-> SPI interrupt
-> multi-CS
-> overrun
```

建议整理成这张表：

| 测试文件 | 外设 | 基地址 | IRQ | 核心行为 |
| --- | --- | --- | --- | --- |
| `test-board-g233.c` | board | - | - | machine/DRAM/PLIC/CLINT |
| `test-gpio-basic.c` | GPIO | `0x10012000` | - | DIR/OUT/IN/reset |
| `test-gpio-int.c` | GPIO | `0x10012000` | `2` | edge/level/IE/IS/W1C/PLIC |
| `test-pwm-basic.c` | PWM | `0x10015000` | - | channel/timer/DONE |
| `test-wdt-timeout.c` | WDT | `0x10010000` | `4` | countdown/feed/lock/timeout IRQ |
| `test-spi-jedec.c` | SPI | `0x10018000` | - | CR/SR/DR/JEDEC |
| `test-flash-read.c` | SPI Flash | `0x10018000` | - | status/read/program/erase |
| `test-flash-read-interrupt.c` | SPI Flash | `0x10018000` | `5` | TXE/RXNE/ERR interrupt |
| `test-spi-cs.c` | SPI Flash | `0x10018000` | - | CS0/CS1 independent flash |
| `test-spi-overrun.c` | SPI | `0x10018000` | `5` | RX overrun/W1C/ERRIE |

🔗 和 Day 2 的关系：

Day 2 不从 SPI 开始，不从 WDT 开始，而是从 `test-gpio-basic.c` 开始。

原因：

```text
GPIO basic 不需要 timer
GPIO basic 不需要 IRQ
GPIO basic 不需要外部协议状态机
GPIO basic 能最快验证 MemoryRegionOps + state + mmio map 的闭环
```

---

## 7. 晚上补外设建模骨架

### 📘 你要看什么

- [exper-note/qemu-hw-study.md](./qemu-hw-study.md)
- QEMU 外设建模流程: <https://qemu.gevico.online/tutorial/2026/ch2/qemu-hw/>

### 🎯 只看这些内容

不要整篇精读。Day 1 晚上只看这几个关键词：

```text
SysBusDevice
MemoryRegionOps
read/write callback
memory_region_init_io
sysbus_init_mmio
sysbus_init_irq
sysbus_mmio_map
sysbus_connect_irq
reset
update_irq
```

### 🧠 最小外设骨架

你可以先把所有 SoC 外设都想成这个模板：

```c
typedef struct G233XXXState {
    SysBusDevice parent_obj;
    MemoryRegion mmio;
    qemu_irq irq;

    /* registers / internal state */
} G233XXXState;

static uint64_t g233_xxx_read(void *opaque, hwaddr offset, unsigned size);
static void g233_xxx_write(void *opaque, hwaddr offset, uint64_t value,
                           unsigned size);

static const MemoryRegionOps g233_xxx_ops = {
    .read = g233_xxx_read,
    .write = g233_xxx_write,
    .endianness = DEVICE_LITTLE_ENDIAN,
};
```

然后在 realize/init 里：

```text
memory_region_init_io()
sysbus_init_mmio()
sysbus_init_irq()
```

最后在 board 里：

```text
qdev_new()
sysbus_realize_and_unref()
sysbus_mmio_map()
sysbus_connect_irq()
```

⚠️ 容易踩坑：

- `read/write` 里处理的是寄存器协议，不是 Linux driver 逻辑。
- `update_irq()` 应该是状态变化后的统一出口，不要到处散写 `qemu_set_irq()`。
- Day 2 的 GPIO basic 可以先不处理 IRQ，但结构里可以预留 `qemu_irq irq`，后面 Day 3 会用。

✍️ 你要填写：

```text
一个最小 SysBusDevice 外设需要哪些字段：
MemoryRegionOps 的 read/write 参数分别代表什么：
Day 2 GPIO 需要先实现哪些寄存器：
Day 3 GPIO IRQ 需要新增哪些状态：
```

---

## 8. 为 Day 2 准备 GPIO basic 行为表

### 📘 你要看什么

- [tests/gevico/qtest/test-gpio-basic.c](../tests/gevico/qtest/test-gpio-basic.c)

### 🎯 只提取测试断言

不要凭硬件经验猜，直接从测试里抄行为。

先提取这些场景：

```text
reset
direction
output bit0
multi pin
interrupt registers in basic test
```

### 🧠 GPIO basic 的最小语义

根据测试，Day 2 的第一版 GPIO 可以先关注：

```text
GPIO_DIR  可读写
GPIO_OUT  可读写
GPIO_IN   派生自 OUT & DIR
GPIO_IE   可读写，复杂中断 Day 3 再做
GPIO_IS   basic 阶段先能读写/清理，Day 3 完善中断语义
GPIO_TRIG 可读写
GPIO_POL  可读写
```

最小 state 可以先长这样：

```text
struct G233GPIOState:
    SysBusDevice parent_obj;
    MemoryRegion mmio;
    qemu_irq irq;
    uint32_t dir;
    uint32_t out;
    uint32_t ie;
    uint32_t is;
    uint32_t trig;
    uint32_t pol;
```

读寄存器：

```text
DIR  -> dir
OUT  -> out
IN   -> out & dir
IE   -> ie
IS   -> is
TRIG -> trig
POL  -> pol
```

写寄存器：

```text
DIR/OUT/IE/TRIG/POL 保存
IS 做 write 1 clear
```

⚠️ 容易踩坑：

- `GPIO_IN` 不要简单等于 `GPIO_OUT`，basic 测试里需要考虑方向位。
- `GPIO_IS` 不要写成普通赋值，后续中断测试需要 W1C。
- Day 2 先让 `test-gpio-basic` 通过，不要把 GPIO IRQ 一起写进去。

✍️ 你要填写：

- `GPIO basic 行为表`
- Day 2 实现文件清单
- Day 2 第一条单项测试命令

---

## 9. Day 1 执行记录

下面这部分是当天必须填写的日志。不要追求好看，先保证信息真实、可复盘。

### 9.1 构建与测试命令记录

| 时间 | 命令 | 结果 | 关键输出 / 错误摘要 | 处理方式 |
| --- | --- | --- | --- | --- |
|  | `make -f Makefile.camp configure` |  |  |  |
|  | `make -f Makefile.camp build` |  |  |  |
|  | `make -f Makefile.camp test-soc` |  |  |  |

补充记录：

```text
configure 是否通过：
build 是否通过：
test-soc 是否跑完：
最先失败的命令：
最关键的错误信息：
下一步处理：
```

### 9.2 SoC 10 题失败矩阵

| 测试 | 结果 | 失败现象 / 关键日志 | 初步判断 |
| --- | --- | --- | --- |
| `test-board-g233` |  |  |  |
| `test-gpio-basic` |  |  |  |
| `test-gpio-int` |  |  |  |
| `test-pwm-basic` |  |  |  |
| `test-wdt-timeout` |  |  |  |
| `test-spi-jedec` |  |  |  |
| `test-flash-read` |  |  |  |
| `test-flash-read-interrupt` |  |  |  |
| `test-spi-cs` |  |  |  |
| `test-spi-overrun` |  |  |  |

当前得分：

```text
通过数量：
失败数量：
阻塞点：
```

### 9.3 G233 简化地址空间表

| 模块 | 地址 | 当前状态 | 证据位置 |
| --- | --- | --- | --- |
| CLINT | `0x02000000` |  |  |
| PLIC | `0x0C000000` |  |  |
| UART0 | `0x10000000` |  |  |
| VirtIO MMIO | `0x10001000` |  |  |
| WDT | `0x10010000` |  |  |
| GPIO | `0x10012000` |  |  |
| PWM | `0x10015000` |  |  |
| SPI | `0x10018000` |  |  |
| pflash | `0x20000000` |  |  |
| DRAM | `0x80000000` |  |  |

### 9.4 QTest 到 MMIO 回调调用链

用自己的话写出来，不要只复制模板：

```text
qtest_writel(qts, addr, value)
->
->
->
```

需要确认的问题：

```text
addr 是 guest 物理地址还是 offset：
MemoryRegionOps.write 里的 offset 是什么：
为什么 SoC 测试不需要 guest driver：
```

### 9.5 SoC 测试对照表

| 测试文件 | 外设 | 基地址 | IRQ | 核心行为 |
| --- | --- | --- | --- | --- |
| `test-board-g233.c` |  |  |  |  |
| `test-gpio-basic.c` |  |  |  |  |
| `test-gpio-int.c` |  |  |  |  |
| `test-pwm-basic.c` |  |  |  |  |
| `test-wdt-timeout.c` |  |  |  |  |
| `test-spi-jedec.c` |  |  |  |  |
| `test-flash-read.c` |  |  |  |  |
| `test-flash-read-interrupt.c` |  |  |  |  |
| `test-spi-cs.c` |  |  |  |  |
| `test-spi-overrun.c` |  |  |  |  |

### 9.6 GPIO basic 行为表

从 [test-gpio-basic.c](../tests/gevico/qtest/test-gpio-basic.c) 里逐条提取，不要凭感觉写。

| 场景 | 测试动作 | 期望结果 | 实现提示 |
| --- | --- | --- | --- |
| reset |  |  |  |
| direction |  |  |  |
| output bit0 |  |  |  |
| multi pin |  |  |  |
| interrupt registers in basic test |  |  |  |

### 9.7 Day 2 开工前检查

| 检查项 | 是否完成 | 备注 |
| --- | --- | --- |
| 环境能 configure/build |  |  |
| 已跑过 `test-soc` 基线 |  |  |
| 已知道 GPIO 基地址 |  |  |
| 已知道 GPIO 寄存器 offset |  |  |
| 已知道 GPIO basic 测什么 |  |  |
| 已知道新增设备文件大概放哪里 |  |  |

---

## 10. Day 1 自测问题

如果下面 10 个问题能答出来，Day 1 就过关。

1. `qtest_init("-machine g233 -m 2G")` 做了什么？
2. `g233` machine 的初始化函数在哪？
3. G233 的 DRAM 起始地址是多少？
4. PLIC 基地址是多少？
5. GPIO 基地址是多少？
6. WDT IRQ 是几号？
7. SPI IRQ 是几号？
8. `MemoryRegionOps.write` 的 `offset` 是绝对地址还是相对设备基地址？
9. QTest 为什么可以不写 guest driver？
10. 明天实现 GPIO basic 时，`GPIO_IN` 应该怎么计算？

参考答案：

```text
1. 启动一个 QEMU 测试实例，machine 为 g233，内存 2G。
2. hw/riscv/g233.c 的 machine init，即当前文件中的 G233 machine init。
3. 0x80000000。
4. 0x0C000000。
5. 0x10012000。
6. 4。
7. 5。
8. 相对该 MemoryRegion 基地址的 offset。
9. QTest 进程通过 qtest/QMP 直接访问 QEMU 设备模型地址空间。
10. 基础测试下为 OUT & DIR。
```

---

## 11. Day 1 之后不要做的事

- 不要改测试。
- 不要一次性实现 GPIO/PWM/WDT/SPI。
- 不要先读完整 QEMU 源码。
- 不要追求完整硬件规格。
- 不要把 SPI Flash 和 `0x20000000` pflash 混为一谈。
- 不要在没跑基线前开始写代码。
- 不要继续横向补很多视频笔记，先把 Day 1 产出填出来。

---

## 12. Day 2 开工口令

Day 2 开始时只做一件事：

```text
实现 G233 GPIO basic，让 test-gpio-basic 通过。
```

最小路径：

```text
新增 g233_gpio 设备
-> 实现 MemoryRegionOps
-> 挂到 0x10012000
-> 跑 test-gpio-basic
-> 修到通过
```

Day 2 的判断标准不是“代码是否优雅”，而是：

```text
test-gpio-basic 是否通过
你是否能解释每个 GPIO 寄存器对应哪个 state 字段
你是否能解释 qtest_writel 为什么能打到你的 write 回调
```
