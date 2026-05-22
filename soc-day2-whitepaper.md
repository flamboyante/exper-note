# Day 2 白皮书：GPIO basic + GPIO interrupt，从 MMIO 寄存器到 PLIC IRQ

## 🎯 Day 2 总目标

Day 2 不再只看 `test-gpio-basic`。如果只完成 basic，一周内推进到 PWM/WDT/SPI 会偏慢。今天把 GPIO 当成一个完整模块铺开：

```text
test-gpio-basic
-> 验证 GPIO 设备能被 QTest 打中
-> 验证 reset / DIR / OUT / IN / bit mask

test-gpio-int
-> 验证 IE / IS / TRIG / POL
-> 验证 edge / level / W1C
-> 验证 qemu_set_irq() 能进入 PLIC IRQ 2
```

今天的目标是建立一个能继续扩展到 PWM/WDT/SPI 的外设模板：

```text
SysBusDevice
-> MemoryRegionOps
-> 设备 state
-> reset
-> read/write 寄存器语义
-> update_irq()
-> board map
-> qtest 单项验证
```

今天结束时，你应该直接掌握这些结论：

| 模块 | 结论 |
| --- | --- |
| GPIO base | `0x10012000` |
| basic 测试 | `test-gpio-basic` 验证 reset、DIR、OUT、IN、多 bit |
| interrupt 测试 | `test-gpio-int` 验证 IE、IS、TRIG、POL、W1C、PLIC IRQ 2 |
| `GPIO_IN` | basic 阶段返回 `out & dir` |
| IRQ 输出 | `IS & IE` 有效时拉 `qemu_irq` |
| PLIC IRQ | `GPIO_PLIC_IRQ=2` |

---

## 🧠 Day 2 核心心智模型

Day 1 建立的是“QTest 怎么打到设备”。Day 2 要把这条链扩展到 GPIO 中断：

```text
QTest
  |
  | qtest_writel(qts, 0x10012004, value)
  v
QEMU system address space
  |
  | 0x10012004 命中 GPIO MemoryRegion
  | offset = 0x10012004 - 0x10012000 = 0x04
  v
GPIO MemoryRegionOps.write(offset=0x04)
  |
  | offset 0x04 是 GPIO_OUT
  v
G233GPIOState
  |
  | s->out = value
  | 根据 TRIG/POL/IE 判断是否置 IS
  v
g233_gpio_update_irq()
  |
  | (s->is & s->ie) != 0
  v
qemu_set_irq(s->irq, 1)
  |
  v
PLIC pending bit for IRQ 2
  |
  | qtest_readl(PLIC_PENDING)
  | qtest_readl(PLIC_CLAIM)
  v
test-gpio-int assert
```

关键分层：

| 层 | 负责什么 |
| --- | --- |
| QTest | 发起 guest 物理地址读写，观察寄存器和 PLIC |
| board / machine | 创建 GPIO 设备，把 MMIO 挂到 `0x10012000`，把 IRQ 接到 PLIC 2 |
| GPIO 设备 | 解释 offset，维护寄存器 state，产生 IRQ side effect |
| PLIC | 接收 GPIO IRQ 线，暴露 pending/claim/complete 行为 |

不要把这四层混在一起。设备文件不应该 switch 绝对地址，board 文件不应该写寄存器语义，测试文件不应该为实现让路。

---

## 📚 Day 2 资料清单

### 必看本地文件

- [test-gpio-basic.c](../tests/gevico/qtest/test-gpio-basic.c)
- [test-gpio-int.c](../tests/gevico/qtest/test-gpio-int.c)
- [hw/riscv/g233.c](../hw/riscv/g233.c)
- [include/hw/riscv/g233.h](../include/hw/riscv/g233.h)
- [soc-day1-whitepaper.md](./soc-day1-whitepaper.md)

### 参考代码怎么用

| 参考文件 | 看什么 | 不要照抄什么 |
| --- | --- | --- |
| `hw/gpio/pl061.c` | GPIO 设备结构、IRQ 更新思路、QOM/SysBusDevice 形态 | 不要照抄复杂输入线、多引脚外部注入、完整 PL061 硬件细节 |
| `hw/gpio/mpc8xxx.c` | 简单 MMIO GPIO 的读写和 reset 风格 | 不要照搬寄存器布局，它不是 G233 |
| `hw/riscv/g233.c` | 既有设备如何创建、realize、map、接 IRQ | 不要把 GPIO 寄存器逻辑塞进 board 文件 |
| `hw/gpio/meson.build` | 新 GPIO 源文件怎么进入构建 | 不要只新建 `.c` 文件却忘记 Meson |

### 四门课在 Day2 的用法

| 课程笔记 | Day2 用法 |
| --- | --- |
| [qemu-hw-study.md](./qemu-hw-study.md) | 提供 `SysBusDevice + MemoryRegionOps + reset + read/write` 骨架 |
| [qemu-board-study.md](./qemu-board-study.md) | 提供 `qdev_new + sysbus_realize_and_unref + sysbus_mmio_map + sysbus_connect_irq` 思路 |
| [qemu-clock-study.md](./qemu-clock-study.md) | 提醒 GPIO 不需要 timer，别把 PWM/WDT 的时间问题提前拉进来 |
| Kernel 运行机制笔记 | 用来理解 QTest MMIO 就是未来 guest driver `readl/writel` 的 host 侧替身 |

---

## 🗺️ 今日路线图

```text
阶段 1：读 test-gpio-basic，确认 reset / DIR / OUT / IN / multi pin
阶段 2：读 test-gpio-int，确认 IE / IS / TRIG / POL / W1C / PLIC
阶段 3：设计 G233GPIOState
阶段 4：写 read/write/reset 伪代码
阶段 5：明确 board map 和 IRQ 连接
阶段 6：明确构建接入
阶段 7：跑 basic 单测
阶段 8：跑 interrupt 单测
```

期望输出：

```text
qtest-riscv64/test-gpio-basic PASS
qtest-riscv64/test-gpio-int PASS
```

我的实际输出：

```text
<你运行后粘贴 3-10 行关键日志>
```

如果没有一次通过，不是坏事。按本文失败定位表修，不要同时改很多地方。

---

## 1. 任务边界：今天做完整 GPIO，不碰 PWM/WDT/SPI

今天做：

| 目标 | 说明 |
| --- | --- |
| GPIO basic | `DIR/OUT/IN/reset/multi_pin` |
| GPIO interrupt | `IE/IS/TRIG/POL/W1C/PLIC IRQ 2` |
| GPIO 设备骨架 | `SysBusDevice + MemoryRegion + qemu_irq + state` |
| board 接入 | map 到 `0x10012000`，IRQ 接 PLIC 2 |

今天不做：

| 不做 | 原因 |
| --- | --- |
| PWM | 它引入 counter/timer/done flag，不属于 GPIO |
| WDT | 它引入虚拟时间、feed、lock、timeout IRQ |
| SPI | 它引入 TX/RX、协议状态机、flash 存储 |
| guest driver | QTest 已经能直接读写 MMIO |
| 完整真实 GPIO 硬件模型 | 当前测试只要求可验证语义 |

一句话结论：

```text
今天把 GPIO 作为一个完整 QEMU 外设做通；后面的 PWM/WDT/SPI 复用这套“MMIO + state + side effect + qtest”的方法。
```

---

## 2. `test-gpio-basic` 标准行为

寄存器表：

| offset | 寄存器 | basic 语义 |
| --- | --- | --- |
| `0x00` | `GPIO_DIR` | R/W，方向位，reset 0 |
| `0x04` | `GPIO_OUT` | R/W，输出锁存，reset 0 |
| `0x08` | `GPIO_IN` | R/O 视图，返回 `out & dir` |
| `0x0C` | `GPIO_IE` | basic 只检查 reset 0 |
| `0x10` | `GPIO_IS` | basic 只检查 reset 0 |
| `0x14` | `GPIO_TRIG` | basic 只检查 reset 0 |
| `0x18` | `GPIO_POL` | basic 只检查 reset 0 |

四个测试场景：

| 场景 | 测试动作 | 期望结果 | 实现含义 |
| --- | --- | --- | --- |
| reset | 读 `DIR/OUT/IN/IE/IS/TRIG/POL` | 全部 0 | reset 必须清所有 state |
| direction | 写 `DIR=0x1` 和 `0xFFFFFFFF` | 读回一致 | `dir` 是普通 R/W 状态 |
| output | `DIR=1`，写 `OUT=1/0`，读 `IN & 1` | 跟随 OUT | `IN = OUT & DIR` |
| multi_pin | 操作 bit0/7/15/31，再清 bit7 | bit 独立 | 不能只处理 bit0 |

`GPIO_IN = OUT & DIR` 的理由：

```text
Day2 没有外部输入引脚模型。
测试要验证的是“输出方向下，OUT 能从 IN 读回”。
因此最小可验证语义是：
GPIO_IN = GPIO_OUT & GPIO_DIR
```

例子：

```text
DIR = 0x00000001
OUT = 0x00000003
IN  = OUT & DIR = 0x00000001
```

期望输出：

```text
g233/gpio/reset_value PASS
g233/gpio/direction PASS
g233/gpio/output PASS
g233/gpio/multi_pin PASS
```

我的实际输出：

```text
<你运行 test-gpio-basic 后粘贴关键日志>
```

---

## 3. `test-gpio-int` 标准行为

`test-gpio-int` 使用同一组 GPIO 寄存器，但开始验证中断语义：

| 寄存器 | interrupt 语义 |
| --- | --- |
| `GPIO_IE` | interrupt enable，bit 为 1 才允许置 `IS` |
| `GPIO_IS` | interrupt status，pending 位，write-1-to-clear |
| `GPIO_TRIG` | `0=edge`，`1=level` |
| `GPIO_POL` | edge 模式下 `1=rising`；level 模式下 `1=high` |
| `GPIO_OUT` | 测试通过写 OUT 模拟引脚电平变化 |

测试场景：

| 场景 | 配置 | 动作 | 期望 |
| --- | --- | --- | --- |
| edge rising | `DIR=1, TRIG=0, POL=1, IE=1` | `OUT: 0 -> 1` | `IS bit0 = 1` |
| level high | `DIR=1, TRIG=1, POL=1, IE=1` | `OUT=1` 后再 `OUT=0` | 高电平时置位，低电平时清除 |
| W1C | edge 触发后写 `GPIO_IS=1` | 清 bit0 | `IS bit0 = 0` |
| IE mask | `IE=0` | `OUT: 0 -> 1` | 不置 `IS` |
| PLIC | 使能 PLIC IRQ 2 后触发 GPIO | PLIC pending/claim 能看到 IRQ 2 | claim 返回 2，complete 后 pending 清除 |

PLIC 相关地址：

| 名称 | 地址/含义 |
| --- | --- |
| `PLIC_BASE` | `0x0C000000` |
| `PLIC_PRIORITY` | `PLIC_BASE + 0x000000` |
| `PLIC_PENDING` | `PLIC_BASE + 0x001000` |
| `PLIC_ENABLE` | `PLIC_BASE + 0x002000`，context 0 |
| `PLIC_CONTEXT` | `PLIC_BASE + 0x200000`，context 0 |
| `PLIC_THRESHOLD` | `PLIC_CONTEXT + 0x0` |
| `PLIC_CLAIM` | `PLIC_CONTEXT + 0x4` |
| `GPIO_PLIC_IRQ=2` | GPIO 接到 PLIC 的 IRQ 号 |

为什么 `test-gpio-int` 要操作 PLIC：

```text
GPIO 设备只能拉高自己的 qemu_irq 线。
PLIC 是否能观察到这个中断，还取决于：
1. IRQ 2 priority 是否非 0
2. context enable bit 是否打开
3. threshold 是否允许该 priority
4. claim/complete 流程是否正确
```

期望输出：

```text
g233/gpio-int/edge_rising PASS
g233/gpio-int/level_high PASS
g233/gpio-int/is_clear PASS
g233/gpio-int/ie_mask PASS
g233/gpio-int/plic PASS
```

我的实际输出：

```text
<你运行 test-gpio-int 后粘贴关键日志>
```

---

## 4. GPIO state 推荐设计

推荐字段：

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
    uint32_t last_level;
} G233GPIOState;
```

字段解释：

| 字段 | 用途 |
| --- | --- |
| `mmio` | 暴露给系统地址空间的 GPIO MMIO 区域 |
| `irq` | GPIO 输出到 PLIC 的中断线 |
| `dir` | `GPIO_DIR` |
| `out` | `GPIO_OUT` |
| `ie` | `GPIO_IE` |
| `is` | `GPIO_IS` |
| `trig` | `GPIO_TRIG` |
| `pol` | `GPIO_POL` |
| `last_level` | edge 检测需要上一拍电平 |

为什么需要 `last_level`：

```text
edge rising 不是看当前是否为 1，而是看是否从 0 变成 1。
所以 write OUT 时需要比较：
old_level = last_level
new_level = out & dir
```

为什么 `GPIO_IN` 不存字段：

```text
GPIO_IN 是派生视图：
read(GPIO_IN) = out & dir

如果单独存 in，会出现 out/dir 改了但 in 忘记同步的问题。
```

---

## 5. read/write/reset 伪代码骨架

reset：

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
    s->last_level = 0;
    g233_gpio_update_irq(s);
}
```

read：

```c
static uint64_t g233_gpio_read(void *opaque, hwaddr offset, unsigned size)
{
    G233GPIOState *s = opaque;

    switch (offset) {
    case 0x00: return s->dir;
    case 0x04: return s->out;
    case 0x08: return s->out & s->dir;
    case 0x0c: return s->ie;
    case 0x10: return s->is;
    case 0x14: return s->trig;
    case 0x18: return s->pol;
    default: return 0;
    }
}
```

write 主体：

```c
static void g233_gpio_write(void *opaque, hwaddr offset,
                            uint64_t value, unsigned size)
{
    G233GPIOState *s = opaque;

    switch (offset) {
    case 0x00:
        s->dir = value;
        g233_gpio_eval_interrupt(s);
        break;
    case 0x04:
        s->out = value;
        g233_gpio_eval_interrupt(s);
        break;
    case 0x08:
        /* GPIO_IN is read-only. Ignore writes. */
        break;
    case 0x0c:
        s->ie = value;
        g233_gpio_eval_interrupt(s);
        break;
    case 0x10:
        s->is &= ~value;       /* W1C */
        g233_gpio_update_irq(s);
        break;
    case 0x14:
        s->trig = value;
        g233_gpio_eval_interrupt(s);
        break;
    case 0x18:
        s->pol = value;
        g233_gpio_eval_interrupt(s);
        break;
    default:
        break;
    }
}
```

说明：

```text
这不是完整代码，只是实现方向。
真正写代码时要按 QEMU 当前风格补宏、类型声明、MemoryRegionOps、class_init、instance_init。
```

---

## 6. interrupt 伪代码骨架

GPIO 电平视图：

```c
static uint32_t g233_gpio_level(G233GPIOState *s)
{
    return s->out & s->dir;
}
```

更新 IRQ 线：

```c
static void g233_gpio_update_irq(G233GPIOState *s)
{
    qemu_set_irq(s->irq, (s->is & s->ie) != 0);
}
```

评估中断状态：

```c
static void g233_gpio_eval_interrupt(G233GPIOState *s)
{
    uint32_t old_level = s->last_level;
    uint32_t new_level = g233_gpio_level(s);
    uint32_t changed = old_level ^ new_level;
    uint32_t pending = 0;

    uint32_t edge = ~s->trig;
    uint32_t level = s->trig;

    uint32_t rising = changed & new_level & s->pol;
    uint32_t falling = changed & old_level & ~s->pol;
    uint32_t high = new_level & s->pol;
    uint32_t low = ~new_level & ~s->pol;

    pending |= edge & (rising | falling);
    pending |= level & (high | low);

    s->is |= pending & s->ie;

    /*
     * Level 模式下，条件不满足时测试期望 IS 清掉。
     * Edge 模式下，IS 通常保持到 W1C。
     */
    s->is = (s->is & ~level) | (pending & level & s->ie);

    s->last_level = new_level;
    g233_gpio_update_irq(s);
}
```

实现时的重点：

| 语义 | 规则 |
| --- | --- |
| edge rising | `old=0,new=1,POL=1,TRIG=0` 时置 `IS` |
| level high | `new=1,POL=1,TRIG=1` 时置 `IS`，`new=0` 时清 |
| IE mask | `IE=0` 时不置 `IS` |
| W1C | 写 `GPIO_IS=1` 清对应 bit |
| IRQ 输出 | `(IS & IE) != 0` 时 `qemu_set_irq(irq, 1)` |

注意：

```text
这里的伪代码表达实现方向，实际 C 代码需要处理 uint32_t mask、类型转换和 QEMU 编码风格。
如果你写完 basic 后先跑 basic，basic 应该不受 IRQ 逻辑影响。
```

---

## 7. SysBusDevice / board / build 接入

### 7.1 设备文件

建议文件：

```text
include/hw/gpio/g233_gpio.h
hw/gpio/g233_gpio.c
```

头文件放：

```c
#define TYPE_G233_GPIO "g233-gpio"
OBJECT_DECLARE_SIMPLE_TYPE(G233GPIOState, G233_GPIO)
```

C 文件放：

```text
G233GPIOState
read/write/reset
MemoryRegionOps
instance_init 或 realize
class_init
TypeInfo
type_init
```

### 7.2 MMIO 初始化

伪代码：

```c
memory_region_init_io(&s->mmio, OBJECT(dev), &g233_gpio_ops,
                      s, "g233-gpio", 0x1000);
sysbus_init_mmio(SYS_BUS_DEVICE(dev), &s->mmio);
sysbus_init_irq(SYS_BUS_DEVICE(dev), &s->irq);
```

MMIO size 用 `0x1000` 足够覆盖 `0x00-0x18`，也符合常见 4 KiB 外设页习惯。

### 7.3 board map

目标：

```text
GPIO_BASE = 0x10012000
GPIO_PLIC_IRQ=2
```

伪代码：

```c
DeviceState *dev = qdev_new(TYPE_G233_GPIO);
SysBusDevice *sbd = SYS_BUS_DEVICE(dev);

sysbus_realize_and_unref(sbd, &error_fatal);
sysbus_mmio_map(sbd, 0, 0x10012000);
sysbus_connect_irq(sbd, 0, gpio_irq_line_for_plic_irq_2);
```

`gpio_irq_line_for_plic_irq_2` 需要参考 `g233.c` 里已有 PLIC/IRQ 连接方式。不要猜 API 名；在 board 文件里找现有 `qdev_get_gpio_in()`、`sysbus_connect_irq()`、PLIC 创建相关代码。

### 7.4 构建接入

需要检查：

```text
hw/gpio/meson.build
可能的 hw/gpio/Kconfig
hw/riscv/meson.build 是否已经包含 g233.c
头文件 include 路径是否符合 QEMU 风格
```

期望输出：

```text
修改 g233_gpio.c 后，build 会重新编译这个文件。
如果故意写错 g233_gpio.c 仍然 build 通过，说明文件没进构建。
```

我的实际输出：

```text
<你 build 后粘贴和 g233_gpio.c 相关的 3-10 行日志>
```

---

## 8. 分阶段执行和失败定位

### 阶段 A：先让 `test-gpio-basic` 过

命令：

```bash
cd ~/qemu-camp/qemu-camp-2026-exper-flamboyante
make -f Makefile.camp build
cd build
./pyvenv/bin/meson test --no-rebuild --print-errorlogs qtest-riscv64/test-gpio-basic
```

期望输出：

```text
qtest-riscv64/test-gpio-basic OK
或
g233/gpio/reset_value PASS
g233/gpio/direction PASS
g233/gpio/output PASS
g233/gpio/multi_pin PASS
```

我的实际输出：

```text
<你运行后粘贴关键日志>
```

失败定位：

| 现象 | 优先查 |
| --- | --- |
| reset 不为 0 | reset 函数是否注册，state 是否清零 |
| `DIR` 写后读不回 | `write 0x00` 是否保存 `dir` |
| `OUT` 写后 `IN` 不对 | `read 0x08` 是否返回 `out & dir` |
| multi pin 不对 | 是否只处理 bit0，是否 mask 逻辑错误 |
| 所有读写都不对 | 设备是否 map 到 `0x10012000`，文件是否进构建 |

### 阶段 B：再让 `test-gpio-int` 过

命令：

```bash
cd ~/qemu-camp/qemu-camp-2026-exper-flamboyante/build
./pyvenv/bin/meson test --no-rebuild --print-errorlogs qtest-riscv64/test-gpio-int
```

期望输出：

```text
qtest-riscv64/test-gpio-int OK
或
g233/gpio-int/edge_rising PASS
g233/gpio-int/level_high PASS
g233/gpio-int/is_clear PASS
g233/gpio-int/ie_mask PASS
g233/gpio-int/plic PASS
```

我的实际输出：

```text
<你运行后粘贴关键日志>
```

失败定位：

| 现象 | 优先查 |
| --- | --- |
| edge_rising 失败 | `last_level`、changed、rising 检测 |
| level_high 失败 | level 模式下不满足条件时是否清 `IS` |
| is_clear 失败 | `GPIO_IS` 是否 W1C，而不是普通赋值 |
| ie_mask 失败 | `IE=0` 时是否仍然错误置 `IS` |
| plic 失败 | `sysbus_init_irq()`、`sysbus_connect_irq()`、PLIC IRQ 2、`qemu_set_irq()` |

### 阶段 C：回归 basic

命令：

```bash
cd ~/qemu-camp/qemu-camp-2026-exper-flamboyante/build
./pyvenv/bin/meson test --no-rebuild --print-errorlogs qtest-riscv64/test-gpio-basic
```

期望输出：

```text
test-gpio-basic 仍然 PASS
```

我的实际输出：

```text
<你运行后粘贴关键日志>
```

如果 interrupt 过了但 basic 失败，通常是 IRQ 逻辑污染了 basic 的 reset/read/write 基础语义。

---

## 9. Day2 实施检查单

| 阶段 | 完成标准 |
| --- | --- |
| 读 basic 测试 | 能说清 reset、direction、output、multi_pin 四个场景 |
| 读 int 测试 | 能说清 edge、level、W1C、IE mask、PLIC 五个场景 |
| state 设计 | 有 `dir/out/ie/is/trig/pol/last_level/mmio/irq` |
| read/write | `0x00-0x18` offset 都有明确行为 |
| reset | 所有寄存器和 `last_level` 清 0 |
| update_irq | `(is & ie) != 0` 时拉高 IRQ |
| board map | GPIO MMIO 到 `0x10012000`，IRQ 接 PLIC 2 |
| build | 新源文件进入编译 |
| basic test | `test-gpio-basic` PASS |
| int test | `test-gpio-int` PASS |

真实执行记录只保留关键输出：

```text
build 输出：
<粘贴 3-10 行>

test-gpio-basic 输出：
<粘贴 3-10 行>

test-gpio-int 输出：
<粘贴 3-10 行>
```

---

## 10. Day2 结论速查

### GPIO basic

| 结论 | 答案 |
| --- | --- |
| `GPIO_BASE` | `0x10012000` |
| `GPIO_DIR` | offset `0x00`，R/W |
| `GPIO_OUT` | offset `0x04`，R/W |
| `GPIO_IN` | offset `0x08`，read 返回 `out & dir` |
| reset | `DIR/OUT/IN/IE/IS/TRIG/POL` 全 0 |
| multi pin | bit0/7/15/31 独立，不能写死 bit0 |

### GPIO interrupt

| 结论 | 答案 |
| --- | --- |
| `GPIO_IE` | offset `0x0C`，interrupt enable |
| `GPIO_IS` | offset `0x10`，interrupt status，W1C |
| `GPIO_TRIG` | offset `0x14`，`0=edge`，`1=level` |
| `GPIO_POL` | offset `0x18`，edge 下 `1=rising`，level 下 `1=high` |
| `GPIO_PLIC_IRQ=2` | GPIO 接 PLIC IRQ 2 |
| `update_irq` | `(is & ie) != 0` 时 `qemu_set_irq(irq, 1)` |

### QTest / MMIO

| 结论 | 答案 |
| --- | --- |
| `qtest_writel(0x10012004)` | 写 guest 物理地址 |
| write 回调 offset | `0x04` |
| 设备代码 switch | switch offset，不 switch 绝对地址 |
| QTest 不需要 driver | host 侧直接控制 QEMU MMIO |

### 实现顺序

```text
1. 写 GPIO 类型和 state
2. 写 reset
3. 写 read/write
4. 写 eval_interrupt/update_irq
5. 初始化 MMIO 和 IRQ
6. Meson 接入
7. g233.c map 到 0x10012000
8. g233.c connect IRQ 2
9. 跑 test-gpio-basic
10. 跑 test-gpio-int
11. 回归 test-gpio-basic
```

---

## 11. Day2 之后不要做的事

| 不做 | 原因 |
| --- | --- |
| 不立刻重构 GPIO | 先保证 basic 和 int 都稳定通过 |
| 不提前写 PWM/WDT/SPI | 它们需要 timer/协议状态机，和 GPIO 的问题层级不同 |
| 不把 PLIC 逻辑写进 GPIO | GPIO 只拉 IRQ 线，PLIC 是外部中断控制器 |
| 不把 QTest 特判写进设备 | 设备要实现通用寄存器语义，不为某个测试硬编码 |
| 不忽略 basic 回归 | interrupt 逻辑很容易破坏 reset 或 IN 行为 |

---

## 12. 下一步怎么接 PWM/WDT/SPI

GPIO 完成后，后面的外设不是重新学一套东西，而是在 GPIO 模板上增加复杂度：

| 后续模块 | 在 GPIO 基础上新增什么 |
| --- | --- |
| PWM | counter、period、duty、done flag、虚拟时间 |
| WDT | load/value/feed/lock/timeout、timer、IRQ 4 |
| SPI | TX/RX、状态位、flash 命令状态机、CS、overrun、IRQ 5 |

Day2 最终口述版本：

```text
我已经能把一个 MMIO 外设挂到 G233 machine。
QTest 写 0x10012004 会命中 GPIO 的 write offset 0x04。
basic 阶段 OUT/IN/DIR 验证 state 和 read/write。
interrupt 阶段 IE/IS/TRIG/POL 验证 side effect。
GPIO 通过 qemu_set_irq 拉 PLIC IRQ 2。
后面的 PWM/WDT/SPI 只是继续叠加 timer 或协议状态机。
```
