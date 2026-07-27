# K230 QEMU 对象模型与外设接线学习笔记

本文整理前面关于 K230 QEMU 代码结构的讨论，重点解释这些概念之间的关系：

- QOM 的 `Object` / `ObjectClass` / `TypeInfo`
- `MachineState` / `MachineClass` / `K230MachineState`
- `K230MachineState` / `K230SoCState` / 外设对象
- `object_initialize_child()` / `qdev_realize()` / `sysbus_realize()`
- `sysbus_init_mmio()` / `sysbus_mmio_map()`
- `sysbus_init_irq()` / `sysbus_connect_irq()`
- `qdev_get_gpio_in()` 和 PLIC 中断接线
- UART 快捷创建和 WDT 标准 SysBus 设备建模的区别

目标不是背宏，而是建立一套读 QEMU 板级代码时稳定可复用的判断方法。

## 1. 先抓住一条主线

K230 当前代码可以按三层看：

```text
Machine 层：
  QEMU 里的一台虚拟机，负责 -machine k230 的整体入口。

SoC 层：
  K230 这颗芯片，负责 CPU、PLIC、CLINT、UART、WDT、SRAM、BootROM 等内部资源。

外设层：
  具体设备模型，例如 WDT、UART、PLIC。
```

对应到当前代码：

```text
K230MachineState
  └── MachineState parent_obj
  └── K230SoCState soc
        └── DeviceState parent_obj
        └── RISCVHartArrayState c908_cpu
        └── K230WdtState wdt[2]
        └── MemoryRegion sram
        └── MemoryRegion bootrom
        └── DeviceState *c908_plic
```

读代码时可以先问：

```text
这个函数是在配置“机器类型”？
还是在初始化“一台机器实例”？
还是在 realize “SoC 设备”？
还是在实现“某个外设自己的寄存器行为”？
```

这个问题能避免把 `MachineClass`、`MachineState`、`K230MachineState` 混成一团。

## 2. QOM 基础：Object、ObjectClass、TypeInfo

QEMU 用 C 语言实现了一套对象系统，叫 QOM。它没有 C++ 那种语言级继承，所以用两个手段模拟：

```text
结构体第一个成员放父类结构体
运行时保存 ObjectClass 指针
```

### 2.1 Object 是实例对象的根

`Object` 是所有 QOM 实例对象的根。关键字段是第一个：

```c
struct Object
{
    ObjectClass *class;
    ObjectFree *free;
    GHashTable *properties;
    uint32_t ref;
    Object *parent;
};
```

重点：

```text
Object 里保存了 ObjectClass *class。
所以每个 QOM 实例都知道自己属于哪个 class。
```

`object_get_class()` 实际上就是返回这个字段：

```c
ObjectClass *object_get_class(Object *obj)
{
    return obj->class;
}
```

这就是前面说的关键点：

```c
object_get_class(OBJECT(machine))
```

不是重新扫描注册表，而是从当前实例里取出 `obj->class`。

### 2.2 ObjectClass 是类对象的根

`ObjectClass` 是所有 class 对象的根。`MachineClass`、`DeviceClass`、`SysBusDeviceClass` 最终都能追溯到它。

实例对象和类对象要分开：

```text
Object / MachineState / K230MachineState
  是运行时实例状态。

ObjectClass / MachineClass / DeviceClass
  是类型级配置和回调。
```

如果用 C 结构体视角看：

```text
MachineState 开头是 Object 实体。
MachineClass 开头是 ObjectClass 实体。

Object 里面有 ObjectClass *class 指针。
```

所以不是：

```text
MachineState 开头就是 ObjectClass
```

而是：

```text
MachineState
  └── Object parent_obj
        └── ObjectClass *class
```

### 2.3 TypeInfo 是类型注册说明书

`TypeInfo` 描述一个 QOM 类型：

```text
.name
  类型名。

.parent
  父类型名。

.instance_size
  实例对象分配多大。

.instance_init
  实例初始化时调用。

.class_init
  class 初始化时调用。
```

`TypeInfo` 本身不是实例，也不是设备。它是注册类型时用的说明书。

对象创建的大致流程是：

```text
type_register_static(&type_info)
  -> QOM 记录这个类型
  -> 后续 type_initialize() 创建 type->class
  -> object_initialize_with_type() 初始化实例
  -> obj->class = type->class
```

因此：

```text
注册 TypeInfo
  让 QEMU 知道“这个类型存在”

创建实例
  才会分配具体的 MachineState、K230SoCState、K230WdtState
```

## 3. MachineClass、MachineState、K230MachineState

### 3.1 MachineClass 是机器类型配置

`MachineClass` 开头是：

```c
struct MachineClass {
    ObjectClass parent_class;

    const char *desc;
    void (*init)(MachineState *state);
    int default_cpus;
    ram_addr_t default_ram_size;
    const char *default_ram_id;
    ...
};
```

它表达的是：

```text
这种 machine 类型应该怎么初始化？
默认 RAM 多大？
默认 CPU 几个？
描述字符串是什么？
```

K230 里对应：

```c
static void k230_machine_class_init(ObjectClass *oc, const void *data)
{
    MachineClass *mc = MACHINE_CLASS(oc);

    mc->desc = "RISC-V Board compatible with Kendryte K230 SDK";
    mc->init = k230_machine_init;
    mc->default_cpus = 1;
    mc->default_ram_id = "riscv.K230.ram";
    mc->default_ram_size = memmap[K230_DEV_DDRC].size;
}
```

什么时候用 `MachineClass *mc`：

```text
需要读取或设置 machine 类型级配置时。
例如 default_ram_size、default_cpus、init 回调。
```

### 3.2 MachineState 是一台机器的通用实例状态

`MachineState` 开头是：

```c
struct MachineState {
    Object parent_obj;

    void *fdt;
    char *dtb;
    char *firmware;
    MemoryRegion *ram;
    ram_addr_t ram_size;
    char *kernel_filename;
    char *kernel_cmdline;
    ...
};
```

它表达的是：

```text
当前这一台虚拟机运行时的通用状态。
```

什么时候用 `MachineState *machine`：

```text
需要访问所有 machine 都有的字段时。
例如 RAM、firmware、kernel、dtb、cmdline、fdt。
```

K230 的 `k230_machine_init()` 参数就是：

```c
static void k230_machine_init(MachineState *machine)
```

这是因为 QEMU 通用 machine 框架只认识父类接口。不同机器都通过 `MachineClass->init(MachineState *)` 进入。

### 3.3 K230MachineState 是 K230 的实例扩展

K230 自己扩展了 `MachineState`：

```c
typedef struct K230MachineState {
    MachineState parent_obj;

    K230SoCState soc;
    Notifier machine_done;
} K230MachineState;
```

关键是 `MachineState parent_obj` 放在第一个成员。

这表示：

```text
K230MachineState 可以当 MachineState 用。
MachineState 指针也可以在确认真实类型后转回 K230MachineState。
```

K230 里常见写法：

```c
MachineClass *mc = MACHINE_GET_CLASS(machine);
K230MachineState *s = RISCV_K230_MACHINE(machine);
```

含义：

```text
MACHINE_GET_CLASS(machine)
  从 machine 实例中取 class，再按 MachineClass 使用。

RISCV_K230_MACHINE(machine)
  确认 machine 真实类型是 k230-machine，再按 K230MachineState 使用。
```

什么时候用 `K230MachineState *s`：

```text
需要访问 K230 自己加的字段时。
例如 s->soc、s->machine_done。
```

## 4. MACHINE_GET_CLASS 到底做了什么

`MACHINE_GET_CLASS(machine)` 不是魔法。它来自：

```c
#define TYPE_MACHINE "machine"
OBJECT_DECLARE_TYPE(MachineState, MachineClass, MACHINE)
```

宏展开后会生成类似：

```c
static inline MachineClass *MACHINE_GET_CLASS(const void *obj)
{
    return OBJECT_GET_CLASS(MachineClass, obj, TYPE_MACHINE);
}
```

再继续看：

```c
#define OBJECT_GET_CLASS(class, obj, name) \
    OBJECT_CLASS_CHECK(class, object_get_class(OBJECT(obj)), name)
```

因此逻辑是：

```text
MACHINE_GET_CLASS(machine)
  -> OBJECT(machine)
  -> object_get_class()
  -> 取 machine->parent_obj.class
  -> 检查这个 class 是 TYPE_MACHINE 或其子类
  -> 转成 MachineClass *
```

等价理解：

```c
Object *obj = OBJECT(machine);
ObjectClass *oc = obj->class;
MachineClass *mc = MACHINE_CLASS(oc);
```

为什么能转？

因为 K230 machine 的 TypeInfo 写了：

```c
static const TypeInfo k230_machine_typeinfo = {
    .name       = TYPE_RISCV_K230_MACHINE,
    .parent     = TYPE_MACHINE,
    .class_init = k230_machine_class_init,
    .instance_init = k230_machine_instance_init,
    .instance_size = sizeof(K230MachineState),
};
```

真实类型是 `k230-machine`，父类是 `machine`。所以它的 class 至少可以按 `MachineClass` 使用。

## 5. 为什么 K230 有两个 TypeInfo

K230 当前有两个核心 QOM 类型：

```text
k230-machine
  parent = TYPE_MACHINE
  instance = K230MachineState

riscv.k230.soc
  parent = TYPE_DEVICE
  instance = K230SoCState
```

### 5.1 k230-machine TypeInfo

```c
static const TypeInfo k230_machine_typeinfo = {
    .name       = TYPE_RISCV_K230_MACHINE,
    .parent     = TYPE_MACHINE,
    .class_init = k230_machine_class_init,
    .instance_init = k230_machine_instance_init,
    .instance_size = sizeof(K230MachineState),
    .interfaces = riscv64_machine_interfaces,
};
```

它注册的是：

```text
QEMU 有一种 machine 叫 k230。
它的实例是 K230MachineState。
它的 machine init 是 k230_machine_init。
```

### 5.2 riscv.k230.soc TypeInfo

```c
static const TypeInfo k230_soc_type_info = {
    .name = TYPE_RISCV_K230_SOC,
    .parent = TYPE_DEVICE,
    .instance_size = sizeof(K230SoCState),
    .instance_init = k230_soc_init,
    .class_init = k230_soc_class_init,
};
```

它注册的是：

```text
K230 SoC 是一个 QEMU device。
它的实例是 K230SoCState。
它被 realize 时会创建/映射 SoC 内部资源。
```

为什么不只用一个 TypeInfo？

因为职责不同：

```text
Machine：
  表示整台虚拟机。
  负责 machine 参数、RAM、启动路径、创建 SoC。

SoC：
  表示芯片。
  负责 CPU、PLIC、CLINT、SRAM、BootROM、UART、WDT 等内部组件。
```

这是标准分层。Machine 不应该直接塞满所有外设细节；SoC 内部固定外设应该归 SoC 管。

## 6. K230 当前初始化流程

可以按调用链看：

```text
QEMU 启动
  -> type_init() 注册 k230-machine
  -> type_init() 注册 riscv.k230.soc

用户选择 -machine k230
  -> QOM 创建 K230MachineState 实例
  -> MachineClass->init = k230_machine_init
  -> 调用 k230_machine_init(machine)

k230_machine_init()
  -> 检查 RAM
  -> object_initialize_child(machine, "soc", &s->soc, TYPE_RISCV_K230_SOC)
  -> qdev_realize(&s->soc)
  -> 把 machine->ram 映射到 DDR 起始地址
  -> 注册 machine_done notifier

k230_soc_realize()
  -> realize CPU hart array
  -> 创建 SRAM / BootROM memory region
  -> 创建 PLIC
  -> 创建 CLINT
  -> 创建 UART
  -> realize/map/connect WDT
  -> 创建大量 unimplemented device 占位
```

注意：

```text
object_initialize_child()
  初始化子对象。

qdev_realize()
  让设备进入可用状态，调用 DeviceClass->realize。
```

这两个阶段不要混为一谈。

## 7. SoC 里为什么 WDT 用 object_initialize_child

K230 SoC 结构里有：

```c
K230WdtState wdt[2];
```

`k230_soc_init()` 里：

```c
object_initialize_child(obj, "k230-wdt0", &s->wdt[0], TYPE_K230_WDT);
object_initialize_child(obj, "k230-wdt1", &s->wdt[1], TYPE_K230_WDT);
```

参数含义：

```text
obj
  父对象，也就是 K230SoCState 对应的 Object。

"k230-wdt0"
  子对象名字。

&s->wdt[0]
  子对象内存位置。这个对象嵌在 SoCState 里，不是 qdev_new 动态分配出来的。

TYPE_K230_WDT
  要初始化成哪种 QOM 类型。
```

这一步只是把 `s->wdt[0]` 这块内存初始化成一个 QOM 对象。它还没有 MMIO 地址，也还没有连 IRQ。

后面在 `k230_soc_realize()` 里：

```c
for (int i = 0; i < 2; i++) {
    if (!sysbus_realize(SYS_BUS_DEVICE(&s->wdt[i]), errp)) {
        return;
    }
}
```

这一步才会进入 WDT 自己的 `k230_wdt_realize()`。

## 8. WDT 作为 SysBusDevice 的标准步骤

WDT 是 K230 自己实现的外设模型。它的状态结构是：

```c
struct K230WdtState {
    SysBusDevice parent_obj;

    MemoryRegion mmio;
    qemu_irq irq;
    struct ptimer_state *timer;

    uint32_t cr;
    uint32_t torr;
    uint32_t ccvr;
    uint32_t stat;
    ...
};
```

核心点：

```text
SysBusDevice parent_obj
  表示它是 sysbus 外设，可以暴露 MMIO region 和 IRQ output。

MemoryRegion mmio
  表示设备寄存器窗口。

qemu_irq irq
  表示设备向外输出的一根中断线。
```

### 8.1 设备自己声明 MMIO 和 IRQ

WDT realize 里：

```c
memory_region_init_io(&s->mmio, OBJECT(dev),
                      &k230_wdt_ops, s,
                      TYPE_K230_WDT,
                      K230_WDT_MMIO_SIZE);
sysbus_init_mmio(sbd, &s->mmio);
sysbus_init_irq(sbd, &s->irq);
```

含义：

```text
memory_region_init_io()
  创建一段 MMIO 区域，绑定 read/write 回调。

sysbus_init_mmio()
  告诉 SysBusDevice：我有一个 MMIO region。

sysbus_init_irq()
  告诉 SysBusDevice：我有一根 IRQ 输出线。
```

这里还没有映射到 K230 的物理地址。

### 8.2 SoC 负责把 MMIO 放到地址空间

SoC 里：

```c
sysbus_mmio_map(SYS_BUS_DEVICE(&s->wdt[0]), 0, memmap[K230_DEV_WDT0].base);
```

参数含义：

```text
SYS_BUS_DEVICE(&s->wdt[0])
  要映射的设备。

0
  第 0 个 MMIO region。

memmap[K230_DEV_WDT0].base
  guest 物理地址。
```

为什么是第 0 个？

因为 WDT 设备里只调用了一次：

```c
sysbus_init_mmio(sbd, &s->mmio);
```

第一次注册的是 region 0。如果一个设备注册多个 region：

```c
sysbus_init_mmio(sbd, &s->ctrl_mmio);  // region 0
sysbus_init_mmio(sbd, &s->data_mmio);  // region 1
```

板级代码就要分别映射：

```c
sysbus_mmio_map(sbd, 0, CTRL_BASE);
sysbus_mmio_map(sbd, 1, DATA_BASE);
```

### 8.3 SoC 负责把 IRQ 接到中断控制器

SoC 里：

```c
sysbus_connect_irq(SYS_BUS_DEVICE(&s->wdt[0]), 0,
                   qdev_get_gpio_in(DEVICE(s->c908_plic), K230_WDT0_IRQ));
```

参数含义：

```text
SYS_BUS_DEVICE(&s->wdt[0])
  发出中断的设备。

0
  该设备第 0 根 IRQ 输出线。

qdev_get_gpio_in(...)
  中断接收端，这里是 PLIC 的某个输入线。
```

为什么 IRQ index 也是 0？

因为 WDT 设备里只调用了一次：

```c
sysbus_init_irq(sbd, &s->irq);
```

第一次注册的是 IRQ output 0。如果一个设备有多根 IRQ：

```c
sysbus_init_irq(sbd, &s->rx_irq);  // irq output 0
sysbus_init_irq(sbd, &s->tx_irq);  // irq output 1
```

连接时就应该写：

```c
sysbus_connect_irq(sbd, 0, rx_target);
sysbus_connect_irq(sbd, 1, tx_target);
```

判断规则：

```text
看设备模型里调用了几次 sysbus_init_mmio()
  第一次对应 sysbus_mmio_map(..., 0, ...)
  第二次对应 sysbus_mmio_map(..., 1, ...)

看设备模型里调用了几次 sysbus_init_irq()
  第一次对应 sysbus_connect_irq(..., 0, ...)
  第二次对应 sysbus_connect_irq(..., 1, ...)
```

## 9. qdev_get_gpio_in 为什么传 PLIC

K230 的外设中断通常要接到 PLIC。PLIC 是 RISC-V 外部中断控制器。

代码：

```c
qdev_get_gpio_in(DEVICE(s->c908_plic), K230_WDT0_IRQ)
```

意思是：

```text
从 PLIC 设备上取第 K230_WDT0_IRQ 根输入线。
```

`qdev_get_gpio_in()` 实现很直接：

```c
qemu_irq qdev_get_gpio_in(DeviceState *dev, int n)
{
    return qdev_get_gpio_in_named(dev, NULL, n);
}
```

它返回的是一个 `qemu_irq` 句柄。这个句柄代表：

```text
某个设备的某根输入线。
```

PLIC 为什么有这些输入线？

PLIC 设备内部会注册输入 GPIO：

```c
qdev_init_gpio_in(dev, sifive_plic_irq_request, s->num_sources);
```

K230 创建 PLIC 时传了 `K230_PLIC_NUM_SOURCES`，所以 PLIC 有很多输入线：

```text
PLIC input[16]   <- UART0
PLIC input[17]   <- UART1
...
PLIC input[107]  <- WDT0
PLIC input[108]  <- WDT1
```

中断路径可以这样理解：

```text
WDT 超时
  -> WDT 内部 raise s->irq
  -> 通过 sysbus_connect_irq 接到 PLIC input[107]
  -> PLIC 记录外部中断源
  -> PLIC 通知 CPU external interrupt
```

所以：

```text
qdev_get_gpio_in()
  不是触发中断。
  它是在拿“接收端线头”。

sysbus_connect_irq()
  才是在把外设输出线接到这个接收端。
```

## 10. UART 和 WDT 为什么创建方式不同

K230 里 UART 和 WDT 都是外设，但当前建模方式不同。

### 10.1 UART：复用现成 serial-mm

UART 创建函数：

```c
static void k230_create_uart(MemoryRegion *sys_mem, DeviceState *plic,
                             int index)
{
    int uart_dev = K230_DEV_UART0 + index;
    g_autofree char *name = g_strdup_printf("uart%d", index);

    create_unimplemented_device(name, memmap[uart_dev].base,
                                memmap[uart_dev].size);

    serial_mm_init(sys_mem, memmap[uart_dev].base, 2,
                   qdev_get_gpio_in(plic, K230_UART0_IRQ + index),
                   399193, serial_hd(index), DEVICE_LITTLE_ENDIAN);
}
```

这里没有把 UART 放进 `K230SoCState`。原因是：

```text
QEMU 已经有现成的 memory-mapped 16550/serial-mm 模型。
K230 这里直接复用它。
```

`serial_mm_init()` 内部已经完成：

```text
qdev_new(TYPE_SERIAL_MM)
设置属性
realize
connect irq
map mmio
```

所以 K230 代码只传：

```text
地址
regshift
PLIC 输入线
baudbase
chardev
大小端
```

前面的 `create_unimplemented_device()` 是为了覆盖 K230 UART 的 0x1000 窗口中 serial-mm 没实现的部分。

### 10.2 WDT：自己实现完整设备模型

WDT 是 K230 自己实现的：

```text
include/hw/watchdog/k230_wdt.h
hw/watchdog/k230_wdt.c
```

它有：

```text
自己的 TypeInfo
自己的 K230WdtState
自己的 MemoryRegionOps
自己的 read/write
自己的 timer
自己的 IRQ
自己的 reset
自己的 vmstate
```

所以 WDT 适合用标准 SysBusDevice 方式：

```text
SoCState 里嵌入 K230WdtState
SoC instance_init 里 object_initialize_child()
SoC realize 里 sysbus_realize()
SoC realize 里 sysbus_mmio_map()
SoC realize 里 sysbus_connect_irq()
```

### 10.3 和 qdev_new 写法的区别

之前类似 G233 的写法：

```c
static void g233_wdt_create(hwaddr addr, qemu_irq irq)
{
    DeviceState *dev;
    SysBusDevice *s;

    dev = qdev_new(TYPE_G233_WDT);
    s = SYS_BUS_DEVICE(dev);
    sysbus_realize_and_unref(s, &error_fatal);
    sysbus_mmio_map(s, 0, addr);
    sysbus_connect_irq(s, 0, irq);
}
```

这也是正确写法。它表达的是：

```text
板子直接动态创建一个外设。
```

K230 当前写法表达的是：

```text
Machine 创建 SoC。
SoC 持有内部外设。
SoC 统一 realize/map/connect 内部外设。
```

两者不是谁绝对正确，而是建模粒度不同：

```text
没有 SoC 模型，或者设备只是板级挂件：
  qdev_new() + sysbus_realize_and_unref() 很直接。

有明确 SoC 模型，外设是芯片内部固定资源：
  object_initialize_child() + SoCState 持有子设备更清晰。
```

K230 已经有 `K230SoCState`，所以 WDT 当前方式更一致。

## 11. 几个常见宏的读法

### 11.1 MACHINE(s)

```c
MachineState *machine = MACHINE(s);
```

含义：

```text
把某个对象按 MachineState 使用。
常用于从 K230MachineState 上转到 MachineState。
```

因为：

```text
K230MachineState 第一个成员是 MachineState parent_obj。
```

### 11.2 RISCV_K230_MACHINE(machine)

```c
K230MachineState *s = RISCV_K230_MACHINE(machine);
```

含义：

```text
确认 machine 的真实 QOM 类型是 k230-machine，然后按 K230MachineState 使用。
```

### 11.3 DEVICE(obj)

```c
qdev_realize(DEVICE(&s->soc), NULL, &error_fatal);
```

含义：

```text
把对象按 DeviceState 使用。
```

K230 SoC 的 TypeInfo 里：

```c
.parent = TYPE_DEVICE
```

所以 `K230SoCState` 可以按 `DeviceState` 使用。

### 11.4 SYS_BUS_DEVICE(obj)

```c
sysbus_realize(SYS_BUS_DEVICE(&s->wdt[i]), errp);
```

含义：

```text
把设备按 SysBusDevice 使用。
```

WDT 的 TypeInfo 里：

```c
.parent = TYPE_SYS_BUS_DEVICE
```

所以 `K230WdtState` 可以按 `SysBusDevice` 使用。

## 12. 读 K230 外设代码的标准流程

以后看一个外设，例如 IOMUX、GPIO、Timer，可以按这个顺序检查。

### 12.1 先找 TypeInfo

看：

```text
.name
.parent
.instance_size
.class_init
.instance_init
```

判断：

```text
它是 Machine？
是 Device？
是 SysBusDevice？
有没有 realize？
```

### 12.2 再看状态结构

看结构体第一个成员：

```text
DeviceState parent_obj
SysBusDevice parent_obj
MachineState parent_obj
```

判断它继承哪类对象。

看它有哪些状态字段：

```text
MemoryRegion
qemu_irq
timer
寄存器变量
子设备
```

这能判断设备复杂度。

### 12.3 再看 class_init

重点看：

```text
dc->realize
reset
vmsd
desc
categories
```

如果是 machine，则看：

```text
mc->init
mc->default_ram_size
mc->default_cpus
```

### 12.4 再看 realize

对于 SysBusDevice，重点看：

```text
memory_region_init_io()
sysbus_init_mmio()
sysbus_init_irq()
timer 初始化
子设备 realize
```

### 12.5 最后看板级/SoC 接线

在 SoC 或 board 代码里找：

```text
object_initialize_child()
qdev_new()
qdev_realize()
sysbus_realize()
sysbus_mmio_map()
sysbus_connect_irq()
qdev_get_gpio_in()
```

判断：

```text
这个设备在哪里创建？
MMIO 映射到哪个 guest 物理地址？
IRQ 接到哪个中断控制器输入？
```

## 13. 一张总表

| 名称 | 它是什么 | 在 K230 中的例子 | 什么时候用 |
| --- | --- | --- | --- |
| `Object` | QOM 实例根类型 | 所有 QOM 对象开头都能转到它 | 获取 class、属性、父子对象关系 |
| `ObjectClass` | QOM class 根类型 | `MachineClass.parent_class` | class 类型检查和转换 |
| `TypeInfo` | 类型注册说明书 | `k230_machine_typeinfo`、`k230_soc_type_info`、`k230_wdt_info` | 注册 QOM 类型 |
| `MachineClass` | machine 类型配置 | `mc->init`、`mc->default_ram_size` | 设置或读取机器类型默认行为 |
| `MachineState` | machine 实例通用状态 | `machine->ram`、`machine->dtb` | 访问通用虚拟机状态 |
| `K230MachineState` | K230 machine 私有状态 | `s->soc`、`s->machine_done` | 访问 K230 特有字段 |
| `K230SoCState` | K230 SoC 设备状态 | `c908_cpu`、`wdt[2]`、`sram` | 管理 SoC 内部资源 |
| `DeviceState` | 设备实例通用状态 | `DEVICE(&s->soc)` | qdev realize、属性、设备通用接口 |
| `SysBusDevice` | 可挂 MMIO/IRQ 的系统总线设备 | `K230WdtState` | sysbus map/connect |
| `MemoryRegion` | 地址空间区域 | WDT MMIO、SRAM、BootROM | RAM/ROM/MMIO 映射 |
| `qemu_irq` | 中断线句柄 | `s->wdt[0].irq` | 连接设备输出和中断控制器输入 |

## 14. 最容易混淆的点

### 14.1 class 和 instance 不要混

```text
MachineClass
  类型配置，通常所有同类型实例共享。

MachineState
  一台虚拟机的运行时状态。
```

`mc->default_ram_size` 是默认配置。

`machine->ram_size` 是当前实例实际 RAM 大小。

### 14.2 init 和 realize 不要混

```text
instance_init
  初始化对象内存和子对象。

realize
  设备进入可用状态，创建 MMIO/IRQ/timer 等运行资源。

machine init
  QEMU machine 框架调用的板级初始化入口。
```

### 14.3 sysbus_init_mmio 和 sysbus_mmio_map 不要混

```text
sysbus_init_mmio()
  设备自己声明“我有一个 MMIO region”。

sysbus_mmio_map()
  SoC/board 把这个 region 放到 guest 物理地址。
```

### 14.4 sysbus_init_irq 和 sysbus_connect_irq 不要混

```text
sysbus_init_irq()
  设备自己声明“我有一根 IRQ 输出线”。

sysbus_connect_irq()
  SoC/board 把这根输出线接到目标输入线。
```

### 14.5 qdev_get_gpio_in 不是 GPIO 外设专用

QEMU 里 `gpio in/out` 常被用作通用信号线，包括中断线。

```text
qdev_get_gpio_in(PLIC, irq_id)
  取 PLIC 的某根中断输入线。
```

这里的 GPIO 不一定是 SoC 的 GPIO 控制器。

## 15. 对后续写 IOMUX 的启发

如果要把 K230 IOMUX 从 `create_unimplemented_device()` 推进成正式设备，可以按 WDT 的结构简化：

```text
1. 定义 K230IomuxState，父类用 SysBusDevice。
2. 状态里放 MemoryRegion mmio 和寄存器数组。
3. 实现 read/write/reset。
4. TypeInfo parent = TYPE_SYS_BUS_DEVICE。
5. class_init 里设置 realize/reset/vmsd。
6. SoCState 里嵌入 K230IomuxState iomux。
7. k230_soc_init() 里 object_initialize_child()。
8. k230_soc_realize() 里 sysbus_realize()。
9. sysbus_mmio_map() 到 0x91105000。
10. 如果没有 IRQ，就不要 sysbus_init_irq / sysbus_connect_irq。
```

IOMUX 第一版通常不需要 IRQ、timer、DMA，也不需要和 UART/SPI/GPIO 动态联动。它更像一个可读写的寄存器 bank。

这就是前面学习这些 QOM/SysBus 概念的直接用途：后面写新设备时，可以清楚知道每一步应该放在哪一层。

