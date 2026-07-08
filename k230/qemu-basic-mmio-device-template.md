# QEMU 通用基础 MMIO 外设骨架模板

本文是一份“以后写新外设时可以照着改”的基础模板。

它适合这类外设：

```text
IOMUX / Pinctrl
RMU / Reset 控制器
CMU / Clock 控制器
简单 SYSCTRL / MISC 寄存器块
只有寄存器读写、暂时没有复杂数据通路的外设
```

它不适合直接完整覆盖这些外设：

```text
UART：还需要 chardev、FIFO、IRQ。
SPI：还需要 SPI bus、片选、flash 下游设备。
SD/MMC：还需要 block backend、命令状态机、DMA/IRQ。
网卡/USB：协议状态机更复杂。
```

但这些复杂外设的第一层仍然一样：

```text
QOM 类型
  -> State 结构体
  -> MemoryRegionOps
  -> read/write/reset
  -> SysBus MMIO
  -> machine 映射
  -> meson/Kconfig
  -> qtest 验证
```

## 0. 先给结论

一个最基础的 QEMU MMIO 外设，本质是：

```text
guest 访问某个物理地址
  -> QEMU 地址空间命中这个外设的 MemoryRegion
  -> 调用设备自己的 read/write 回调
  -> read/write 修改或返回设备 State 里的寄存器状态
```

所以最小模型要解决三个问题：

```text
这个设备叫什么？              TYPE_XXX / TypeInfo
它保存什么状态？              XXXState
guest 读写地址时怎么响应？     MemoryRegionOps + read/write
```

再往外一层，还要解决：

```text
谁创建这个设备？              SoC / board 代码
映射到哪个地址？              sysbus_mmio_map()
是否参与构建？                meson.build / Kconfig
如何证明它能工作？            qtest 或真实启动日志
```

## 1. 文件布局

最常见的基础外设至少有两个文件：

```text
include/hw/misc/xxx.h
hw/misc/xxx.c
```

如果是某类已有目录，也可以放到更专门的位置：

```text
hw/watchdog/xxx_wdt.c
include/hw/watchdog/xxx_wdt.h

hw/gpio/xxx_gpio.c
include/hw/gpio/xxx_gpio.h
```

对 K230 这种 SoC 内部控制寄存器，`hw/misc/` 通常够用。

## 2. 头文件模板：`include/hw/misc/xxx.h`

这个文件只放外部需要知道的东西：

```c
/*
 * Vendor XXX basic MMIO device
 *
 * Copyright (c) 2026
 *
 * SPDX-License-Identifier: GPL-2.0-or-later
 */

#ifndef HW_MISC_XXX_H
#define HW_MISC_XXX_H

#include "hw/core/sysbus.h"
#include "qom/object.h"

#define TYPE_XXX "vendor.xxx"
OBJECT_DECLARE_SIMPLE_TYPE(XXXState, XXX)

#define XXX_MMIO_SIZE 0x1000
#define XXX_NUM_REGS (XXX_MMIO_SIZE / sizeof(uint32_t))

struct XXXState {
    SysBusDevice parent_obj;

    MemoryRegion mmio;
    uint32_t regs[XXX_NUM_REGS];
};

#endif /* HW_MISC_XXX_H */
```

### 2.1 include guard

```c
#ifndef HW_MISC_XXX_H
#define HW_MISC_XXX_H
...
#endif
```

作用是防止头文件被重复 include。

命名习惯：

```text
路径 include/hw/misc/xxx.h
  -> HW_MISC_XXX_H
```

### 2.2 `TYPE_XXX`

```c
#define TYPE_XXX "vendor.xxx"
```

这是 QOM 类型名。

它不是 C 类型名，而是 QEMU 运行时对象系统里的类型字符串。后面 `TypeInfo` 会用它注册设备。

K230 IOMUX 可以类似写成：

```c
#define TYPE_K230_IOMUX "riscv.k230.iomux"
```

### 2.3 `OBJECT_DECLARE_SIMPLE_TYPE`

```c
OBJECT_DECLARE_SIMPLE_TYPE(XXXState, XXX)
```

它会帮你声明常用的类型转换宏。

后面代码里可以写：

```c
XXXState *s = XXX(obj);
```

这比手写强转更符合 QEMU 风格。

### 2.4 `SysBusDevice parent_obj`

```c
struct XXXState {
    SysBusDevice parent_obj;
    ...
};
```

QEMU 用结构体第一个成员模拟继承。

这个设备要挂在 SoC 系统总线上，有 MMIO 地址，所以继承 `SysBusDevice`。

不要把它写成：

```c
DeviceState parent_obj;
```

除非你非常确定这个设备不需要 `sysbus_init_mmio()`、`sysbus_mmio_map()`、`sysbus_init_irq()` 这些 SysBus 能力。

### 2.5 `MemoryRegion mmio`

```c
MemoryRegion mmio;
```

这是设备暴露给 guest 的 MMIO 窗口。

guest 不是直接访问 `regs[]`，而是访问某段物理地址。QEMU 地址空间命中这个 `MemoryRegion` 后，才会进入 read/write。

### 2.6 `regs[]`

```c
uint32_t regs[XXX_NUM_REGS];
```

这是最简单的“寄存器影子”。

第一版设备如果只是寄存器读写，可以先用数组保存全部 32-bit 寄存器：

```text
offset 0x00 -> regs[0]
offset 0x04 -> regs[1]
offset 0x08 -> regs[2]
```

后续如果某些寄存器有特殊语义，再逐个从数组模型改成 switch 模型。

## 3. C 文件模板：`hw/misc/xxx.c`

这是最小可工作的设备模型。

```c
/*
 * Vendor XXX basic MMIO device
 *
 * Copyright (c) 2026
 *
 * SPDX-License-Identifier: GPL-2.0-or-later
 */

#include "qemu/osdep.h"
#include "hw/misc/xxx.h"
#include "migration/vmstate.h"
#include "qemu/log.h"
#include "qemu/module.h"

static void xxx_reset(DeviceState *dev)
{
    XXXState *s = XXX(dev);

    memset(s->regs, 0, sizeof(s->regs));
}

static uint64_t xxx_read(void *opaque, hwaddr offset, unsigned size)
{
    XXXState *s = XXX(opaque);

    if (offset >= XXX_MMIO_SIZE) {
        qemu_log_mask(LOG_GUEST_ERROR,
                      "%s: bad offset 0x%" HWADDR_PRIx "\n",
                      __func__, offset);
        return 0;
    }

    return s->regs[offset >> 2];
}

static void xxx_write(void *opaque, hwaddr offset,
                      uint64_t value, unsigned size)
{
    XXXState *s = XXX(opaque);

    if (offset >= XXX_MMIO_SIZE) {
        qemu_log_mask(LOG_GUEST_ERROR,
                      "%s: bad offset 0x%" HWADDR_PRIx "\n",
                      __func__, offset);
        return;
    }

    s->regs[offset >> 2] = value;
}

static const MemoryRegionOps xxx_ops = {
    .read = xxx_read,
    .write = xxx_write,
    .endianness = DEVICE_LITTLE_ENDIAN,
    .impl = {
        .min_access_size = 4,
        .max_access_size = 4,
        .unaligned = false,
    },
};

static void xxx_init(Object *obj)
{
    XXXState *s = XXX(obj);

    memory_region_init_io(&s->mmio, obj, &xxx_ops, s,
                          TYPE_XXX, XXX_MMIO_SIZE);
    sysbus_init_mmio(SYS_BUS_DEVICE(obj), &s->mmio);
}

static const VMStateDescription vmstate_xxx = {
    .name = TYPE_XXX,
    .version_id = 1,
    .minimum_version_id = 1,
    .fields = (const VMStateField[]) {
        VMSTATE_UINT32_ARRAY(regs, XXXState, XXX_NUM_REGS),
        VMSTATE_END_OF_LIST()
    }
};

static void xxx_class_init(ObjectClass *klass, const void *data)
{
    DeviceClass *dc = DEVICE_CLASS(klass);

    dc->desc = "Vendor XXX basic MMIO device";
    dc->vmsd = &vmstate_xxx;
    device_class_set_legacy_reset(dc, xxx_reset);
}

static const TypeInfo xxx_info = {
    .name          = TYPE_XXX,
    .parent        = TYPE_SYS_BUS_DEVICE,
    .instance_size = sizeof(XXXState),
    .instance_init = xxx_init,
    .class_init    = xxx_class_init,
};

static void xxx_register_types(void)
{
    type_register_static(&xxx_info);
}

type_init(xxx_register_types)
```

## 4. C 文件逐段解释

### 4.1 include 顺序

```c
#include "qemu/osdep.h"
```

QEMU C 文件通常第一行 include `qemu/osdep.h`。

它会处理平台差异、基础宏、常用系统头。不要把自己的头文件放在它前面。

```c
#include "hw/misc/xxx.h"
#include "migration/vmstate.h"
#include "qemu/log.h"
#include "qemu/module.h"
```

分别对应：

| 头文件 | 用途 |
| --- | --- |
| `hw/misc/xxx.h` | 自己的 State、TYPE、常量。 |
| `migration/vmstate.h` | VMState 迁移保存。 |
| `qemu/log.h` | `qemu_log_mask()`、`LOG_GUEST_ERROR`。 |
| `qemu/module.h` | `type_init()`。 |

### 4.2 reset

```c
static void xxx_reset(DeviceState *dev)
{
    XXXState *s = XXX(dev);

    memset(s->regs, 0, sizeof(s->regs));
}
```

reset 表示设备回到硬件复位状态。

最基础版本可以全部清零。

如果手册写了默认值，后面可以改成：

```c
memset(s->regs, 0, sizeof(s->regs));
s->regs[REG_A >> 2] = 0x00000001;
s->regs[REG_B >> 2] = 0x00000100;
```

注意：默认值不要凭感觉写。没有证据时，第一版写清零更容易解释。

### 4.3 read

```c
static uint64_t xxx_read(void *opaque, hwaddr offset, unsigned size)
```

参数含义：

| 参数 | 含义 |
| --- | --- |
| `opaque` | `memory_region_init_io()` 传进来的设备对象。 |
| `offset` | guest 访问地址相对本设备 MMIO base 的偏移。 |
| `size` | 访问宽度，比如 1、2、4、8。 |

例如设备映射在 `0x91105000`：

```text
guest 读 0x91105000 -> offset = 0x00
guest 读 0x91105004 -> offset = 0x04
guest 读 0x91105020 -> offset = 0x20
```

最小模型只支持 32-bit 访问，所以：

```c
return s->regs[offset >> 2];
```

`offset >> 2` 等价于 `offset / 4`。

### 4.4 write

```c
static void xxx_write(void *opaque, hwaddr offset,
                      uint64_t value, unsigned size)
```

最小模型就是写入保存：

```c
s->regs[offset >> 2] = value;
```

这类模型适合 IOMUX/RMU/CMU 的第一版，因为很多控制寄存器至少应该支持：

```text
guest 写一个配置值
guest 再读回来
```

后续如果某些寄存器不是普通读写，再逐个处理：

```text
只读寄存器：忽略写入。
写 1 清零：W1C。
写触发动作：write 之后调用 update 函数。
保留位：用 writable mask 过滤。
```

### 4.5 `MemoryRegionOps`

```c
static const MemoryRegionOps xxx_ops = {
    .read = xxx_read,
    .write = xxx_write,
    .endianness = DEVICE_LITTLE_ENDIAN,
    .impl = {
        .min_access_size = 4,
        .max_access_size = 4,
        .unaligned = false,
    },
};
```

这里告诉 QEMU：

```text
这个 MMIO 区域读时调用 xxx_read
写时调用 xxx_write
设备是小端
最小/最大访问宽度都是 4 字节
不支持非对齐访问
```

对 RISC-V SoC 上的 32-bit 控制寄存器来说，这是一个常见起点。

如果 guest 真的有 8-bit/16-bit 访问需求，再放开访问宽度。第一版不要过早支持没有证据的访问形式。

### 4.6 init

```c
static void xxx_init(Object *obj)
{
    XXXState *s = XXX(obj);

    memory_region_init_io(&s->mmio, obj, &xxx_ops, s,
                          TYPE_XXX, XXX_MMIO_SIZE);
    sysbus_init_mmio(SYS_BUS_DEVICE(obj), &s->mmio);
}
```

这里做两件事：

```text
memory_region_init_io()
  创建一段 MMIO 区域，并绑定 read/write 回调。

sysbus_init_mmio()
  告诉 SysBusDevice：这个设备有一个 MMIO BAR 可以被 machine 映射。
```

注意：这里还没有决定映射到哪个物理地址。

真正的地址映射在 SoC / board 代码里做：

```c
sysbus_mmio_map(SYS_BUS_DEVICE(&s->xxx), 0, base);
```

### 4.7 为什么这个模板不写 `dc->realize`

这个基础模板的 `class_init` 里没有写：

```c
dc->realize = xxx_realize;
```

原因是：这个设备的基础结构已经在 `instance_init` 里建好了。

```c
static void xxx_init(Object *obj)
{
    XXXState *s = XXX(obj);

    memory_region_init_io(&s->mmio, obj, &xxx_ops, s,
                          TYPE_XXX, XXX_MMIO_SIZE);
    sysbus_init_mmio(SYS_BUS_DEVICE(obj), &s->mmio);
}
```

QOM/qdev 生命周期可以先这样记：

```text
object_new() / object_initialize_child()
  -> instance_init

qdev_realize() / sysbus_realize()
  -> dc->realize，如果 class 设置了 realize 回调

reset
  -> reset callback
```

也就是：

```text
instance_init
  适合做对象创建时就能确定的基础结构初始化。

realize
  适合做设备真正启用时才应该做、可能失败、依赖外部资源或依赖属性的初始化。
```

对 IOMUX / RMU / CMU / SYSCTRL 这类第一版寄存器块来说，通常满足：

```text
MMIO size 固定
没有 IRQ
没有 qdev property
没有子设备需要 realize
没有 chardev/block/net backend
没有 timer 或 DMA 等运行资源
初始化不会失败
```

所以把 `memory_region_init_io()` 和 `sysbus_init_mmio()` 放在 `instance_init` 里是合理的，`class_init` 不需要额外设置 `dc->realize`。

此时 SoC 侧仍然要调用：

```c
if (!sysbus_realize(SYS_BUS_DEVICE(&s->xxx), errp)) {
    return;
}
```

只是因为设备没有自定义 `dc->realize`，所以这个 realize 阶段不会进入设备自己的 `xxx_realize()`。

#### 什么时候应该写 `dc->realize`

如果设备有下面任意一种情况，就应该认真考虑写 `xxx_realize()`：

```text
需要根据 qdev property 决定设备规模
初始化可能失败，需要通过 Error **errp 报错
需要 realize 子设备
需要连接外部 backend，比如 chardev、blockdev、netdev
需要创建或启动 timer、DMA、bus 等运行资源
需要在 realize 阶段校验外部连接是否完整
```

这时结构通常变成：

```c
static void xxx_realize(DeviceState *dev, Error **errp)
{
    XXXState *s = XXX(dev);

    memory_region_init_io(&s->mmio, OBJECT(dev), &xxx_ops, s,
                          TYPE_XXX, XXX_MMIO_SIZE);
    sysbus_init_mmio(SYS_BUS_DEVICE(dev), &s->mmio);

    /* 如果有 IRQ，才需要这一句。 */
    sysbus_init_irq(SYS_BUS_DEVICE(dev), &s->irq);
}

static void xxx_class_init(ObjectClass *klass, const void *data)
{
    DeviceClass *dc = DEVICE_CLASS(klass);

    dc->realize = xxx_realize;
    device_class_set_legacy_reset(dc, xxx_reset);
}
```

注意：同一个 MMIO region 不要同时在 `instance_init` 和 `realize` 里初始化。二选一。

基础寄存器块推荐优先使用本文模板的写法：

```text
instance_init:
  memory_region_init_io()
  sysbus_init_mmio()

class_init:
  desc / vmsd / reset
```

等设备真的需要属性、子设备、IRQ、timer 或外部 backend 时，再引入 `dc->realize`。

### 4.8 VMState

```c
static const VMStateDescription vmstate_xxx = {
    .name = TYPE_XXX,
    .version_id = 1,
    .minimum_version_id = 1,
    .fields = (const VMStateField[]) {
        VMSTATE_UINT32_ARRAY(regs, XXXState, XXX_NUM_REGS),
        VMSTATE_END_OF_LIST()
    }
};
```

VMState 用于迁移和保存虚拟机状态。

最小寄存器块只要保存 `regs[]`。如果后面加了 IRQ 状态、FIFO、timer，也要考虑迁移状态。

### 4.9 class init

```c
static void xxx_class_init(ObjectClass *klass, const void *data)
{
    DeviceClass *dc = DEVICE_CLASS(klass);

    dc->desc = "Vendor XXX basic MMIO device";
    dc->vmsd = &vmstate_xxx;
    device_class_set_legacy_reset(dc, xxx_reset);
}
```

这里设置设备类级别的信息：

| 字段 | 含义 |
| --- | --- |
| `dc->desc` | 设备描述。 |
| `dc->vmsd` | 迁移状态描述。 |
| reset 回调 | 设备 reset 时调用。 |

### 4.10 TypeInfo

```c
static const TypeInfo xxx_info = {
    .name          = TYPE_XXX,
    .parent        = TYPE_SYS_BUS_DEVICE,
    .instance_size = sizeof(XXXState),
    .instance_init = xxx_init,
    .class_init    = xxx_class_init,
};
```

这是 QOM 类型注册说明书。

它回答：

```text
这个类型叫什么？
继承哪个父类型？
实例对象多大？
创建实例时调用哪个函数？
初始化 class 时调用哪个函数？
```

### 4.11 type_init

```c
static void xxx_register_types(void)
{
    type_register_static(&xxx_info);
}

type_init(xxx_register_types)
```

作用是让 QEMU 启动时注册这个类型。

如果没有这段，QEMU 不知道 `TYPE_XXX` 这个设备类型存在。

## 5. SoC 侧接入模板

`.c/.h` 设备本体写完后，还不代表 guest 能访问它。

还需要 SoC 代码做三件事：

```text
在 SoC State 里放一个设备实例。
初始化并 realize 这个设备。
把设备 MMIO 映射到物理地址。
```

### 5.1 SoC 头文件加入设备成员

例如：

```c
#include "hw/misc/xxx.h"

struct SomeSoCState {
    DeviceState parent_obj;

    XXXState xxx;
};
```

含义：

```text
这颗 SoC 内部包含一个 XXX 外设实例。
```

### 5.2 SoC init 里初始化 child

```c
object_initialize_child(OBJECT(obj), "xxx", &s->xxx, TYPE_XXX);
```

含义：

```text
创建 SoC 的时候，也创建它内部的 xxx 子设备对象。
```

### 5.3 SoC realize 里 realize 和映射

```c
if (!sysbus_realize(SYS_BUS_DEVICE(&s->xxx), errp)) {
    return;
}

sysbus_mmio_map(SYS_BUS_DEVICE(&s->xxx), 0, XXX_BASE);
```

含义：

```text
sysbus_realize()
  让设备进入可用状态。

sysbus_mmio_map()
  把设备第 0 个 MMIO 区域映射到 SoC 物理地址。
```

如果原来有：

```c
create_unimplemented_device("xxx", XXX_BASE, XXX_SIZE);
```

接入真实模型后，应替换掉它。否则同一段地址会有重复映射风险。

### 5.4 有 IRQ 的设备才接 IRQ

基础寄存器块通常没有 IRQ。

不要因为别的设备有这段，就照抄：

```c
sysbus_connect_irq(...);
```

只有设备模型里调用了：

```c
sysbus_init_irq(SYS_BUS_DEVICE(obj), &s->irq);
```

并且硬件手册/驱动确实说明有中断输出，才需要在 SoC 侧连接 IRQ。

IOMUX 这类第一版寄存器块通常不需要 IRQ。

## 6. 构建接入模板

设备本体写完后，还要让 QEMU 构建系统编译它。

### 6.1 `hw/misc/meson.build`

```meson
system_ss.add(when: 'CONFIG_XXX', if_true: files('xxx.c'))
```

含义：

```text
当 CONFIG_XXX 打开时，把 hw/misc/xxx.c 编进 system emulator。
```

### 6.2 `hw/misc/Kconfig`

```text
config XXX
    bool
```

含义：

```text
定义一个可被 SoC 选择的构建配置项。
```

### 6.3 SoC 的 Kconfig 选择它

例如在 `hw/riscv/Kconfig`：

```text
config SOME_SOC
    bool
    select XXX
```

含义：

```text
只要这个 SoC 被启用，就自动编译 XXX 外设。
```

## 7. 最小 qtest 验证模板

qtest 是 QEMU 自己的设备级测试框架。

它不需要真的跑 guest 程序，而是在测试进程里启动一个 QEMU，然后通过 qtest 协议直接读写 guest 物理地址。

对基础 MMIO 寄存器块来说，可以先把 qtest 理解成：

```text
测试代码
  -> 启动 qemu-system-xxx -machine some-machine
  -> qtest_writel(addr, value)
  -> QEMU memory API 命中设备 MMIO
  -> 调用设备 write 回调
  -> qtest_readl(addr)
  -> 调用设备 read 回调
  -> g_assert_cmphex() 检查结果
```

### 7.1 基础寄存器块优先测什么

最小寄存器块优先验证四件事：

```text
设备地址已经被真实模型接管，不再是 unimplemented。
reset 默认值正确。
普通寄存器能写后读回。
不同 offset 不会互相覆盖。
```

如果设备只允许 32-bit 访问，还可以加访问宽度测试。但第一版可以先不测非法访问，先把正常路径打稳。

### 7.2 测试文件位置

通常放在：

```text
tests/qtest/xxx-test.c
```

K230 IOMUX 可以是：

```text
tests/qtest/k230-iomux-test.c
```

### 7.3 最小 qtest 文件模板

```c
#include "qemu/osdep.h"
#include "libqtest.h"

#define XXX_BASE 0x10000000
#define XXX_REG0 (XXX_BASE + 0x00)
#define XXX_REG4 (XXX_BASE + 0x04)

static QTestState *xxx_qtest_start(void)
{
    return qtest_init("-machine some-machine");
}

static void test_xxx_reset_values(void)
{
    QTestState *qts = xxx_qtest_start();

    g_assert_cmphex(qtest_readl(qts, XXX_REG0), ==, 0x00000000);
    g_assert_cmphex(qtest_readl(qts, XXX_REG4), ==, 0x00000000);

    qtest_quit(qts);
}

static void test_xxx_rw(void)
{
    QTestState *qts = xxx_qtest_start();

    qtest_writel(qts, XXX_REG0, 0x12345678);
    g_assert_cmphex(qtest_readl(qts, XXX_REG0), ==, 0x12345678);

    qtest_writel(qts, XXX_REG4, 0xa5a5a5a5);
    g_assert_cmphex(qtest_readl(qts, XXX_REG4), ==, 0xa5a5a5a5);

    qtest_quit(qts);
}

int main(int argc, char **argv)
{
    g_test_init(&argc, &argv, NULL);

    qtest_add_func("/xxx/reset-values", test_xxx_reset_values);
    qtest_add_func("/xxx/rw", test_xxx_rw);

    return g_test_run();
}
```

注意这里用的是：

```c
#include "libqtest.h"
```

以及：

```c
QTestState *qts = qtest_init("-machine some-machine");
qtest_writel(qts, addr, value);
qtest_readl(qts, addr);
qtest_quit(qts);
```

这种写法每个 test case 启动和退出自己的 QEMU，隔离性更强。K230 现有 WDT qtest 也是这个风格。

### 7.4 K230 IOMUX qtest 示例

K230 IOMUX 的地址窗口是：

```text
base = 0x91105000
size = 0x800
```

如果第一版模型是 32-bit 寄存器 bank，可以先写：

```c
#include "qemu/osdep.h"
#include "libqtest.h"

#define K230_IOMUX_BASE 0x91105000
#define K230_IOMUX_IO0  (K230_IOMUX_BASE + 0x00)
#define K230_IOMUX_IO1  (K230_IOMUX_BASE + 0x04)
#define K230_IOMUX_IO63 (K230_IOMUX_BASE + 0xfc)

static QTestState *k230_iomux_qtest_start(void)
{
    return qtest_init("-machine k230");
}

static void test_k230_iomux_reset_values(void)
{
    QTestState *qts = k230_iomux_qtest_start();

    g_assert_cmphex(qtest_readl(qts, K230_IOMUX_IO0), ==, 0x944);
    g_assert_cmphex(qtest_readl(qts, K230_IOMUX_IO1), ==, 0x944);

    qtest_quit(qts);
}

static void test_k230_iomux_rw(void)
{
    QTestState *qts = k230_iomux_qtest_start();

    qtest_writel(qts, K230_IOMUX_IO0, 0x12345678);
    g_assert_cmphex(qtest_readl(qts, K230_IOMUX_IO0), ==, 0x12345678);

    qtest_writel(qts, K230_IOMUX_IO1, 0xa5a5a5a5);
    g_assert_cmphex(qtest_readl(qts, K230_IOMUX_IO1), ==, 0xa5a5a5a5);

    g_assert_cmphex(qtest_readl(qts, K230_IOMUX_IO0), ==, 0x12345678);

    qtest_quit(qts);
}

static void test_k230_iomux_last_known_pin(void)
{
    QTestState *qts = k230_iomux_qtest_start();

    qtest_writel(qts, K230_IOMUX_IO63, 0x00000abc);
    g_assert_cmphex(qtest_readl(qts, K230_IOMUX_IO63), ==, 0x00000abc);

    qtest_quit(qts);
}

int main(int argc, char **argv)
{
    g_test_init(&argc, &argv, NULL);

    qtest_add_func("/k230-iomux/reset-values",
                   test_k230_iomux_reset_values);
    qtest_add_func("/k230-iomux/rw", test_k230_iomux_rw);
    qtest_add_func("/k230-iomux/last-known-pin",
                   test_k230_iomux_last_known_pin);

    return g_test_run();
}
```

这里先测到 `IO63`，是因为 K230 IOMUX 常见 pin 配置表是 64 个 pin。即使 MMIO region 是 `0x800`，也不要一上来假设所有 0x800 字节都有真实 pin 语义。

### 7.5 qtest 的 meson 接入

测试文件写完后，还要接到：

```text
tests/qtest/meson.build
```

K230 WDT 已经有类似接入：

```meson
qtests_riscv64 = ['riscv-csr-test'] + \
  (config_all_devices.has_key('CONFIG_K230') ? ['k230-wdt-test'] : [])
```

可以改成：

```meson
qtests_riscv64 = ['riscv-csr-test'] + \
  (config_all_devices.has_key('CONFIG_K230') ? [
    'k230-wdt-test',
    'k230-iomux-test',
  ] : [])
```

含义是：

```text
只有当前构建包含 CONFIG_K230 时，才编译并运行 K230 相关 qtest。
```

### 7.6 编译并运行 qtest

训练营目录如果提供了 `Makefile.camp`，优先使用训练营封装命令。

常见完整流程是：

```bash
make -f Makefile.camp configure
make -f Makefile.camp build
make -f Makefile.camp test-xxx
```

含义是：

```text
configure
  配置 QEMU build 目录和目标架构。

build
  编译 QEMU 和测试程序。

test-xxx
  运行某一类训练营测试。
```

例如训练营 SoC 实验里常见的是：

```bash
make -f Makefile.camp configure
make -f Makefile.camp build
make -f Makefile.camp test-soc
```

如果 K230 仓库后续也放入 `Makefile.camp`，可以加专门目标：

```bash
make -f Makefile.camp test-k230-iomux
```

它内部可以封装成：

```make
.PHONY: test-k230-iomux
test-k230-iomux: build
	cd $(BUILD_DIR) && $(MESON) test --no-rebuild --print-errorlogs \
	    qtest-riscv64/k230-iomux-test
```

当前 `qemu-camp-2026-k230` 仓库根目录如果没有 `Makefile.camp`，就使用下面的 QEMU 原生命令。

设备代码、Kconfig、meson 和 qtest 都接好后，先编译 RISC-V system emulator：

```bash
ninja -C build qemu-system-riscv64
```

如果只想编译 K230 IOMUX qtest 二进制，可以跑：

```bash
ninja -C build tests/qtest/k230-iomux-test
```

如果不确定测试名，先列出来：

```bash
build/pyvenv/bin/meson test -C build --list | grep k230
```

K230 IOMUX 单项 qtest：

```bash
build/pyvenv/bin/meson test -C build qtest-riscv64/k230-iomux-test
```

K230 WDT 单项 qtest：

```bash
build/pyvenv/bin/meson test -C build qtest-riscv64/k230-wdt-test
```

一次跑 K230 相关 qtest：

```bash
build/pyvenv/bin/meson test -C build qtest-riscv64/k230-iomux-test qtest-riscv64/k230-wdt-test
```

如果只想直接跑测试二进制，通常也可以这样：

```bash
build/tests/qtest/k230-iomux-test
```

但推荐优先用 `meson test`，因为它会带上 QEMU 测试需要的环境变量。

日常最短闭环可以记这一组：

```bash
ninja -C build qemu-system-riscv64 tests/qtest/k230-iomux-test
build/pyvenv/bin/meson test -C build qtest-riscv64/k230-iomux-test
```

含义是：

```text
先确保 qemu-system-riscv64 和 k230-iomux-test 都能编译。
再通过 meson test 用标准 qtest 环境运行 K230 IOMUX 测试。
```

如果你的 shell 里已经能直接找到 `meson`，也可以把上面的 `build/pyvenv/bin/meson` 简写成 `meson`。

### 7.7 qtest 失败时先看什么

失败时按这个顺序查：

```text
1. QEMU 是否能启动 -machine k230？
2. 设备源文件是否真的被 meson 编译？
3. Kconfig 是否定义了 CONFIG_XXX？
4. SoC Kconfig 是否 select 了设备？
5. SoC realize 是否 sysbus_realize() 成功？
6. sysbus_mmio_map() 的 base 是否正确？
7. 原来的 create_unimplemented_device() 是否已经删除？
8. qtest 访问地址是否等于 memmap 里的 base + offset？
```

如果 read/write 完全没有进设备回调，优先怀疑：

```text
meson/Kconfig 没接上。
SoC 没 object_initialize_child()。
SoC 没 sysbus_realize()。
SoC 没 sysbus_mmio_map()。
同地址还残留 unimplemented region 或其他 region。
```

### 7.8 qtest 不应该测什么

第一版基础寄存器块不要把 qtest 写成硬件手册大全。

暂时不建议测：

```text
还没实现的 pin 复用真实副作用。
还没确认的保留位行为。
还没确认的非法访问异常细节。
依赖真实 SDK 启动完整流程的行为。
```

优先测已经承诺的模型语义：

```text
reset 默认值。
32-bit read/write。
offset 隔离。
SoC 地址接入正确。
```

## 8. 什么时候从数组模型升级成 switch 模型

第一版数组模型：

```c
s->regs[offset >> 2] = value;
```

适合：

```text
寄存器只是保存配置。
驱动写后可能读回。
暂时没有明确副作用。
```

当出现以下情况，就应该改成 switch：

```text
某个寄存器只读。
某个位写 1 清零。
某个位写入会触发 reset/clock/IRQ/timer。
读某个寄存器会清状态。
保留位必须固定为 0。
不同 offset 默认值不同。
```

形态类似：

```c
switch (offset) {
case XXX_CTRL:
    s->regs[XXX_CTRL >> 2] = value & XXX_CTRL_WRITABLE_MASK;
    xxx_update(s);
    break;
case XXX_STATUS:
    qemu_log_mask(LOG_GUEST_ERROR, "%s: write to read-only STATUS\n",
                  __func__);
    break;
default:
    s->regs[offset >> 2] = value;
    break;
}
```

核心原则：

```text
先让未知寄存器保持简单。
只给已经确认的寄存器加特殊语义。
```

## 9. 常见变体

### 9.1 有 IRQ 的外设

State 里增加：

```c
qemu_irq irq;
```

init 里增加：

```c
sysbus_init_irq(SYS_BUS_DEVICE(obj), &s->irq);
```

状态变化后调用：

```c
qemu_set_irq(s->irq, level);
```

SoC 侧再连接到 PLIC/GIC：

```c
sysbus_connect_irq(SYS_BUS_DEVICE(&s->xxx), 0,
                   qdev_get_gpio_in(interrupt_controller, irq_num));
```

注意：

```text
状态位不是 IRQ 线。
IRQ 线应该由状态位和使能位共同计算出来。
```

### 9.2 有 timer 的外设

State 里增加：

```c
QEMUTimer *timer;
```

init 里创建：

```c
s->timer = timer_new_ns(QEMU_CLOCK_VIRTUAL, xxx_timer_cb, s);
```

reset 或写寄存器时重新计算到期时间。

适合：

```text
WDT
timer
PWM
timeout 类型控制器
```

### 9.3 有下游 bus 的外设

SPI controller 这类设备不只保存寄存器，还要和下游设备通信。

常见结构：

```text
SPI controller
  -> SSI bus
  -> SPI flash
```

这时 write 某个 TX/data 寄存器，不只是保存值，还要发起一次 bus transaction。

这已经不是本文的最小模板范围，但它仍然从同一套 QOM/MMIO 骨架开始。

## 10. 写新外设时的检查清单

开始写之前先确认：

```text
1. base/size 来自哪里？
2. guest 是否真的会访问？
3. 访问宽度是 32-bit 还是也有 8/16-bit？
4. reset 默认值有没有资料？
5. 第一版是否只需要读写保存？
6. 是否有 IRQ？如果没有，不要接 IRQ。
7. 是否有 timer/bus/DMA？如果没有，不要提前设计。
```

写完 `.c/.h` 后检查：

```text
1. osdep.h 是否第一 include？
2. TYPE 名是否唯一？
3. State 第一个成员是否是 SysBusDevice parent_obj？
4. MemoryRegion 是否 init？
5. sysbus_init_mmio() 是否调用？
6. read/write 是否检查 offset？
7. reset 是否给出明确默认状态？
8. VMState 是否保存必要状态？
```

接入 SoC 后检查：

```text
1. 头文件是否 include 新设备头？
2. SoC State 是否有设备成员？
3. object_initialize_child() 是否调用？
4. sysbus_realize() 是否检查返回值？
5. sysbus_mmio_map() 地址是否正确？
6. 原来的 create_unimplemented_device() 是否去掉？
7. 没有 IRQ 的设备是否没有错误接 IRQ？
```

构建和验证检查：

```text
1. meson.build 是否加入源文件？
2. Kconfig 是否定义 config？
3. SoC Kconfig 是否 select？
4. ninja 是否能编译？
5. qtest 是否覆盖读写？
6. 真实启动日志里 unimplemented access 是否消失？
```

## 11. 对 K230 IOMUX 的映射

把本文模板套到 K230 IOMUX：

```text
XXX
  -> K230_IOMUX

XXXState
  -> K230IomuxState

TYPE_XXX
  -> TYPE_K230_IOMUX

XXX_MMIO_SIZE
  -> K230_IOMUX_MMIO_SIZE = 0x800

regs[]
  -> 保存 0x91105000..0x911057ff 的 32-bit 寄存器
```

第一版目标：

```text
把原来的 unimplemented iomux 地址窗口
替换成一个可读、可写、可 reset、可 qtest 验证的寄存器 bank。
```

不做：

```text
不模拟真实 pin 连线。
不把 UART/SPI/SD 的功能选择传播给其他设备。
不编造复杂 pinctrl 语义。
```

这就是最基础但可以解释清楚的 QEMU 外设模型。
