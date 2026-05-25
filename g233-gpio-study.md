# G233 GPIO 学习与实现笔记

这份笔记用于实现 G233 SoC 的 GPIO 控制器，目标是通过：

```sh
cd build
pyvenv/bin/meson test --no-rebuild --print-errorlogs qtest-riscv64/test-gpio-basic
pyvenv/bin/meson test --no-rebuild --print-errorlogs qtest-riscv64/test-gpio-int
```

## 1. 这题要做什么

你要在 QEMU 里补一个固定地址的 MMIO 外设：

```text
G233 GPIO
  base: 0x10012000
  size: 0x100
  irq : PLIC IRQ 2
```

guest 或 qtest 访问 `0x10012000` 附近的寄存器时，QEMU 应该进入你的 GPIO 读写函数，读写你结构体里的状态字段。

## 2. 验收测试看什么

`test-gpio-basic.c` 主要看：

```text
1. reset 后所有寄存器读 0
2. GPIO_DIR 写什么，读回什么
3. GPIO_OUT 写高/低后，输出模式 pin 的 GPIO_IN 能读到对应电平
4. 多个 pin 的 bit 行为正确
```

`test-gpio-int.c` 主要看：

```text
1. edge + rising: 0 -> 1 时置 GPIO_IS
2. level + high : 高电平时置 GPIO_IS，低电平时清 GPIO_IS
3. GPIO_IS 支持 write-1-to-clear
4. GPIO_IE 为 0 时不应该置 GPIO_IS
5. GPIO 中断能进 PLIC IRQ 2
```

## 3. 寄存器表

G233 GPIO 只需要 7 个寄存器：

```text
offset  name       access  meaning
0x00    GPIO_DIR   R/W     方向，0=input，1=output
0x04    GPIO_OUT   R/W     输出值
0x08    GPIO_IN    R       当前 pin 电平；输出模式下由 OUT 驱动
0x0C    GPIO_IE    R/W     中断使能
0x10    GPIO_IS    R/W1C   中断状态，写 1 清除
0x14    GPIO_TRIG  R/W     触发类型，0=edge，1=level
0x18    GPIO_POL   R/W     极性，0=low/falling，1=high/rising
```

每个寄存器是 32 位，每一位对应一个 GPIO pin。

访问权限含义：

```text
R/W    普通可读写：写入值保存，读回保存值。
R      只读：guest 写入应忽略。
R/W1C  可读，写 1 清对应状态位，写 0 不影响对应状态位。
```

`GPIO_IN` 不是 guest 直接写出来的值，也不是 `GPIO_OUT` 的简单备份。
它表示 GPIO 控制器采样到的“当前 pin 电平”。

按 GPIO 语义，每个 pin 可以这样理解：

```text
DIR = 1 输出模式
  GPIO_OUT 是输出锁存值，会驱动 pin；
  GPIO_IN 读当前 pin 电平。当前训练营不模拟短路、开漏、总线争用等电气细节，
  所以输出模式下当前 pin 电平等于 GPIO_OUT。

DIR = 0 输入模式
  GPIO_OUT 仍然保存 guest 写入的输出锁存值，但不驱动 pin；
  GPIO_IN 应该来自外部设备驱动的输入电平。
```

当前 qtest 没有外部设备连接到 GPIO 输入 pin，所以最小模型里可以把外部输入看成 0：

```c
external_in = 0;
s->in = (s->out & s->dir) | (external_in & ~s->dir);
```

因为 `external_in` 暂时恒为 0，上式可简化为：

```c
s->in = s->out & s->dir;
```

`prev_in` 只是设备内部状态，用来判断边沿变化，不是手册 MMIO 寄存器。
不要把 `prev_in` 暴露成 `0x1c` 寄存器。

例子：

```text
GPIO_DIR = 0x00000001
```

表示 pin 0 是输出，其余 pin 是输入。

```text
GPIO_OUT = 0x00008081
```

表示 pin 0、pin 7、pin 15 输出高。

## 4. 需要哪些文件

你现在的拆分方式可以继续用：

```text
hw/riscv/g233_feat/g233_gpio.h
hw/riscv/g233_feat/g233_gpio.c
```

`hw/riscv/meson.build` 里把 `g233_gpio.c` 加进 `CONFIG_GEVICO_G233` 构建，这一步已经做对了。

后面还要在 `hw/riscv/g233.c` 里创建设备并挂载。

## 5. 设备状态结构该放什么

一个最小 GPIO 状态结构可以这样设计：

```c
typedef struct G233GPIOState {
    SysBusDevice parent_obj;
    MemoryRegion mmio;
    qemu_irq irq;

    uint32_t dir;
    uint32_t out;
    uint32_t in;
    uint32_t ie;
    uint32_t is;
    uint32_t trig;
    uint32_t pol;
    uint32_t prev_in;
} G233GPIOState;
```

字段含义：

```text
dir      保存 GPIO_DIR
out      保存 GPIO_OUT
in       保存 GPIO_IN 的当前值
ie       保存 GPIO_IE
is       保存 GPIO_IS
trig     保存 GPIO_TRIG
pol      保存 GPIO_POL
prev_in  保存上一次输入值，用来判断边沿
irq      GPIO 汇总中断输出线，接到 PLIC IRQ 2
```

`prev_in` 很关键。没有它，你无法知道这次是不是 `0 -> 1` 或 `1 -> 0`。

## 6. QOM 类型注册

你需要定义一个设备类型名：

```c
#define TYPE_G233_GPIO "g233-gpio"
```

然后用：

```c
OBJECT_DECLARE_SIMPLE_TYPE(G233GPIOState, G233_GPIO)
```

或使用仓库里相同风格的 `DECLARE_INSTANCE_CHECKER`。

它的作用和你看过的 `SIFIVE_GPIO(obj)` 一样：

```text
把通用 Object 指针转换成 G233GPIOState *
```

## 7. MMIO read 怎么写

read 的工作是：

```text
根据 offset 返回结构体里的对应字段
```

读权限表：

```text
GPIO_DIR   读 s->dir
GPIO_OUT   读 s->out
GPIO_IN    读 s->in
GPIO_IE    读 s->ie
GPIO_IS    读 s->is
GPIO_TRIG  读 s->trig
GPIO_POL   读 s->pol
其他 offset 返回 0，并打印 LOG_UNIMP 或 LOG_GUEST_ERROR
```

伪代码：

```c
switch (offset) {
case GPIO_DIR:
    return s->dir;
case GPIO_OUT:
    return s->out;
case GPIO_IN:
    return s->in;
case GPIO_IE:
    return s->ie;
case GPIO_IS:
    return s->is;
case GPIO_TRIG:
    return s->trig;
case GPIO_POL:
    return s->pol;
default:
    return 0;
}
```

`GPIO_IN` 是只读寄存器，但 read 逻辑只是返回 `s->in`。只读的限制要在 write 里体现：写 `GPIO_IN` 时忽略。
`prev_in` 不应该出现在 read 的 switch 里。

## 8. MMIO write 怎么写

write 的工作是：

```text
根据 offset 修改结构体字段，然后重新计算状态
```

写权限表：

```text
GPIO_DIR   R/W    s->dir = value
GPIO_OUT   R/W    s->out = value
GPIO_IN    R      忽略写入
GPIO_IE    R/W    s->ie = value
GPIO_IS    R/W1C  s->is &= ~value
GPIO_TRIG  R/W    s->trig = value
GPIO_POL   R/W    s->pol = value
其他 offset 忽略，并打印 LOG_UNIMP 或 LOG_GUEST_ERROR
```

伪代码：

```c
switch (offset) {
case GPIO_DIR:
    s->dir = value;
    break;
case GPIO_OUT:
    s->out = value;
    break;
case GPIO_IE:
    s->ie = value;
    break;
case GPIO_IS:
    s->is &= ~value;    /* write 1 to clear */
    break;
case GPIO_TRIG:
    s->trig = value;
    break;
case GPIO_POL:
    s->pol = value;
    break;
case GPIO_IN:
    /* read-only, ignore writes */
    break;
default:
    break;
}

g233_gpio_update(s);
```

注意：`GPIO_IS` 是写 1 清除，不是直接赋值。

```text
原 is = 0b1011
写入  = 0b0011
结果  = 0b1000
```

所以是：

```c
s->is &= ~value;
```

`GPIO_IN` 不保存 guest 写入值。它读的是当前 pin 电平。
当前最小实现里，输入模式没有外部驱动，默认读 0；输出模式由 `GPIO_OUT` 驱动，
所以可以暂时用 `s->out & s->dir` 得到 `GPIO_IN`。

## 9. update_state 怎么写

G233 GPIO 的状态更新比 `sifive_gpio.c` 简单。

第一步：计算当前 pin 电平，也就是 `GPIO_IN` 要读到的值。

完整语义可以写成：

```c
external_in = 0; /* 当前训练营 GPIO 测试没有外部输入源 */
new_in = (s->out & s->dir) | (external_in & ~s->dir);
```

由于 `external_in` 暂时恒为 0，实际实现可以简化成：

```c
new_in = s->out & s->dir;
```

第二步：算中断状态。

只允许 `GPIO_IE` 为 1 的 pin 置中断：

```c
enabled = s->ie;
```

### edge 模式

`GPIO_TRIG` bit 为 0 表示边沿触发。

```text
POL = 1: rising，0 -> 1
POL = 0: falling，1 -> 0
```

伪代码：

```c
edge_mask = ~s->trig;
rising = ~s->prev_in & new_in;
falling = s->prev_in & ~new_in;

edge_hit = ((rising & s->pol) | (falling & ~s->pol)) & edge_mask & enabled;
s->is |= edge_hit;
```

### level 模式

`GPIO_TRIG` bit 为 1 表示电平触发。

```text
POL = 1: high level
POL = 0: low level
```

电平触发不是 sticky：电平不满足后，对应 `GPIO_IS` 要清掉。

伪代码：

```c
level_mask = s->trig;
level_active = ((new_in & s->pol) | (~new_in & ~s->pol)) & level_mask & enabled;

s->is &= ~level_mask;
s->is |= level_active;
```

这里有一个细节：只清 level 模式对应的 bit，不要清 edge 模式已经置位的 bit。

第三步：更新输入和历史值。

```c
s->in = new_in;
s->prev_in = new_in;
```

第四步：更新汇总 IRQ。

```c
qemu_set_irq(s->irq, (s->is & s->ie) != 0);
```

这表示只要任意一个启用的 pin 有中断状态，就拉高 GPIO 总中断线。

## 10. realize 里做什么

`realize` 是设备实例初始化阶段。需要做三件事：

```text
1. 初始化 MMIO MemoryRegion
2. 注册到 SysBusDevice
3. 初始化一根 IRQ 输出线
```

伪代码：

```c
memory_region_init_io(&s->mmio, OBJECT(dev), &g233_gpio_ops,
                      s, TYPE_G233_GPIO, 0x100);

sysbus_init_mmio(SYS_BUS_DEVICE(dev), &s->mmio);
sysbus_init_irq(SYS_BUS_DEVICE(dev), &s->irq);
```

## 11. reset 里做什么

reset 后测试期望所有寄存器为 0：

```c
s->dir = 0;
s->out = 0;
s->in = 0;
s->ie = 0;
s->is = 0;
s->trig = 0;
s->pol = 0;
s->prev_in = 0;
qemu_set_irq(s->irq, 0);
```

## 12. 板级 g233.c 里还缺什么

只写设备文件还不够。你必须把设备挂到 G233 机器里。

需要在合适位置：

```text
1. qdev_new(TYPE_G233_GPIO)
2. sysbus_realize_and_unref()
3. sysbus_mmio_map(..., 0x10012000)
4. sysbus_connect_irq(..., qdev_get_gpio_in(mmio_irqchip, 2))
```

伪代码：

```c
DeviceState *gpio = qdev_new(TYPE_G233_GPIO);
SysBusDevice *gpio_sbd = SYS_BUS_DEVICE(gpio);

sysbus_realize_and_unref(gpio_sbd, &error_fatal);
sysbus_mmio_map(gpio_sbd, 0, 0x10012000);
sysbus_connect_irq(gpio_sbd, 0, qdev_get_gpio_in(mmio_irqchip, 2));
```

这个逻辑应该放在 PLIC 已经创建好之后，因为你需要 `mmio_irqchip`。

## 13. 为什么现在还不能过

当前 `g233_gpio.c` 已经开始有 QOM 类型和 read 框架，但还没有完整实现：

```text
完整 write 逻辑
update_state
reset
realize
板级挂载
IRQ 连接
```

`test-gpio-basic` 失败在：

```text
GPIO_DIR 写 1 后读回 0
```

这通常说明两个问题之一：

```text
1. 0x10012000 还没有真正接到你的 GPIO 设备模型
2. GPIO write 没有保存 DIR/OUT 等寄存器状态
```

## 14. 推荐实现顺序

不要一次性写完所有逻辑。按这个顺序来：

```text
第 1 步：补 QOM 类型和空设备
第 2 步：补 MemoryRegionOps
第 3 步：实现 read/write 七个寄存器
第 4 步：在 g233.c 里 map 到 0x10012000
第 5 步：跑 test-gpio-basic
第 6 步：补 edge/level 中断逻辑
第 7 步：接 PLIC IRQ 2
第 8 步：跑 test-gpio-int
```

每完成一步就构建一次：

```sh
make -f Makefile.camp build
```

## 15. 常见错误

### 错误 1：只加 meson，不挂设备

编译进来了不等于机器上存在这个外设。

### 错误 2：GPIO_IN 当成可写寄存器

测试期望输出模式下 `GPIO_IN` 能读到当前 pin 电平；在这个最小模型里，
当前 pin 电平由 `GPIO_OUT` 驱动。输入模式下没有外部驱动，读 0。

### 错误 3：GPIO_IS 写入时直接赋值

错误：

```c
s->is = value;
```

正确：

```c
s->is &= ~value;
```

### 错误 4：level 中断做成 sticky

level 模式下电平不满足后，`GPIO_IS` 要清掉。

### 错误 5：清 GPIO_IS 后期待 PLIC pending 自动消失

测试里也说明了：PLIC pending 需要 claim/complete。GPIO 侧只负责拉高/拉低 IRQ 线。

## 16. 和 SiFive GPIO 的关系

`sifive_gpio.c` 可以学这些：

```text
QOM 类型注册
SysBusDevice 结构
MemoryRegionOps read/write
sysbus_init_mmio
sysbus_init_irq
reset/realize 写法
```

不要照抄这些：

```text
SiFive 的寄存器表
input_en/output_en/port/pue/out_xor
rise/fall/high/low 四套中断寄存器
每个 pin 一根 IRQ
```

G233 GPIO 的寄存器更简单，中断是 32 个 pin 汇总成 PLIC IRQ 2。

## 17. 暂未实现但可扩展

当前 GPIO 只实现 qtest 需要的最小模型。下面这些不是当前必需项，但可以作为后续扩展线索：

```text
trace events
  用于稳定记录 GPIO read/write 调试信息。
  如果要启用，需要在对应 trace-events 文件里定义 g233_gpio_read/write。

QOM properties
  用于让外部配置设备属性，例如 pin 数。
  当前 G233 GPIO 固定 32 pin、固定地址、固定 IRQ，不需要属性表。

VMStateDescription
  用于 savevm/loadvm/live migration 保存和恢复 GPIO 寄存器状态。
  当前训练营 qtest 不覆盖迁移，可后续补 dir/out/in/ie/is/trig/pol/prev_in。

external GPIO lines
  用于让其他虚拟设备驱动输入 pin，或观察输出 pin。
  当前没有外部输入源，所以输入模式读 0，输出模式用 OUT 驱动 IN。
```

代码里可以保留这些 TODO 注释，但不要只留下没有解释的注释代码；否则后续很难判断它是模板残留还是明确计划。

## 18. 最终检查清单

实现完成后逐项确认：

```text
[ ] g233_gpio.h 定义 TYPE_G233_GPIO 和 G233GPIOState
[ ] g233_gpio.c 注册 TypeInfo
[ ] realize 初始化 MMIO 和 IRQ
[ ] reset 清空全部寄存器
[ ] read 支持 7 个寄存器
[ ] read 不暴露 prev_in
[ ] write 支持 DIR/OUT/IE/IS/TRIG/POL
[ ] write 忽略 GPIO_IN
[ ] GPIO_IN 表示当前 pin 电平；最小模型中等价于 GPIO_OUT & GPIO_DIR
[ ] GPIO_IS write-1-to-clear
[ ] edge rising/falling 正确
[ ] level high/low 正确
[ ] qemu_set_irq 汇总 is & ie
[ ] g233.c map 到 0x10012000
[ ] g233.c IRQ 接到 PLIC IRQ 2
[ ] test-gpio-basic 通过
[ ] test-gpio-int 通过
```
