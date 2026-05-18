# Day 1 白皮书：从会用 QEMU 到能改 G233 SoC

## 读这份文档的目标

Day 1 不是为了写完所有外设。Day 1 的目标是建立后面 6 天能持续推进的开发地图：

```text
知道训练营 SoC 要做什么
知道 QEMU machine 如何启动
知道外设如何挂到地址空间
知道 QTest 如何验证 MMIO
知道本仓库每个测试对应哪个外设
知道明天从 GPIO basic 怎么下手
```

学完 Day 1，你应该能回答：

1. `-machine g233` 最终进入哪个 C 函数？
2. `qtest_writel(0x10012000, value)` 为什么应该进入 GPIO 的 `write` 回调？
3. 一个 QEMU 外设最小需要哪些东西？
4. 为什么 SoC 方向不需要先写 guest driver？
5. SoC 10 个测试分别验证什么？
6. 后续新增 GPIO/WDT/SPI 文件时，应该怎么接入构建和板级文件？

## 你的背景对应的学习策略

你已经有三年外设开发经验，普通外设基本都做过，RISC-V 有接触，C 能写，短板是 QEMU 开发而不是 QEMU 使用。

所以 Day 1 不学这些：

- GPIO 是什么。
- SPI 是什么。
- WDT 是什么。
- 寄存器读写为什么要 `readl/writel`。

Day 1 只学 QEMU 的翻译关系：

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

## 资料清单

只看下面这些，不要扩散：

- Machine model: https://qemu.gevico.online/tutorial/2025/ch2/sec4/c-model/machine-model/
- QEMU 外设建模流程: https://qemu.gevico.online/tutorial/2026/ch2/qemu-hw/
- QTest: https://qemu.gevico.online/tutorial/2025/ch2/sec6/qtest/
- SoC 实验手册: https://qemu.gevico.online/exercise/2026/stage1/soc/g233-exper-manual/
- G233 SoC 硬件手册: https://qemu.gevico.online/exercise/2026/stage1/soc/g233-datasheet/

公开站点暂时没有稳定的视频回放入口。训练营新闻能看到录播课程和直播答疑形式，但执行上不要等视频；官方讲义 + 本地测试已经足够推进。

## Day 1 时间安排

### 09:00-09:30 先建立任务边界

读：

```text
README_zh.md
Makefile.camp
exper-note/soc-7day-plan.md
```

你要提取三件事：

1. SoC 方向测试位置：`tests/gevico/qtest/`
2. SoC 方向执行入口：`make -f Makefile.camp test-soc`
3. SoC 方向 10 个基础测试：

```text
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

记录到笔记：

```text
SoC 方向不是写 Linux 驱动，而是给 QEMU 的 G233 machine 补虚拟外设模型。
测试通过 QTest 直接访问 MMIO 地址，不依赖 guest 程序。
```

### 09:30-10:30 看 Machine model

看：

https://qemu.gevico.online/tutorial/2025/ch2/sec4/c-model/machine-model/

只抓这条主线：

```text
main()
-> qemu_init()
-> 解析 -machine
-> 创建 MachineState
-> machine_run_board_init()
-> MachineClass.init()
-> g233 的 machine init 函数
```

本地对照：

```bash
rg -n "TYPE_RISCV_G233_MACHINE|virt_machine_init|machine_class_init|mc->init" "hw/riscv/g233.c" "include/hw/riscv/g233.h"
```

你要理解：

- QEMU machine 是一个 QOM 对象。
- `g233` 是一种 machine type。
- machine init 负责创建 CPU、内存、中断控制器和板上外设。
- 后面新增 GPIO/WDT/SPI，本质是在 G233 machine init 阶段把设备实例化并映射到地址空间。

输出笔记：

```text
-machine g233 的核心入口：
TYPE_RISCV_G233_MACHINE
MachineClass.init
virt_machine_init(MachineState *machine)
```

### 10:30-11:30 看 G233 板级代码

读：

```text
include/hw/riscv/g233.h
hw/riscv/g233.c
```

重点搜索：

```bash
rg -n "virt_memmap|VIRT_PLIC|VIRT_CLINT|VIRT_UART0|VIRT_DRAM|memory_region_add_subregion|sysbus_create_simple|qdev_get_gpio_in|qemu_set_irq|plic" "hw/riscv/g233.c" "include/hw/riscv/g233.h"
```

你要画出简化地址图：

```text
0x02000000 CLINT
0x0C000000 PLIC
0x10000000 UART0
0x10001000 VirtIO MMIO slots
0x10010000 WDT      待实现
0x10012000 GPIO     待实现
0x10015000 PWM      待实现
0x10018000 SPI      待实现
0x20000000 Flash    已有 pflash 区域，不等同于 SPI Flash 测试模型
0x80000000 DRAM
```

注意：

- `0x20000000` 的 pflash 是 QEMU virt 类机器已有的并行 Flash 区域。
- SoC 测题里的 SPI Flash 是挂在 `0x10018000` SPI 控制器后的协议设备，不是直接 MMIO pflash。
- 后续 SPI 测试要实现 SPI 控制器内部连接的 W25X16/W25X32 状态机。

输出笔记：

```text
G233 现有基础设施：CPU、DRAM、PLIC、CLINT、UART、RTC、pflash。
SoC 实验缺口：WDT/GPIO/PWM/SPI MMIO 外设及其 IRQ/timer/状态机。
```

### 11:30-12:00 看 QTest

看：

https://qemu.gevico.online/tutorial/2025/ch2/sec6/qtest/

抓三句话：

```text
QTest 是 QEMU 设备模型测试框架。
测试进程通过 QMP/qtest 协议和 QEMU 交互。
测试可以直接访问设备模型地址空间，也能推进虚拟时钟。
```

本地对照：

```bash
sed -n '1,120p' "tests/gevico/qtest/test-gpio-basic.c"
sed -n '1,120p' "tests/gevico/qtest/test-wdt-timeout.c"
```

你要理解：

- `qtest_init("-machine g233 -m 2G")` 启动一个 QEMU。
- `qtest_writel(qts, addr, value)` 对 QEMU 的物理地址空间写 32 位。
- 如果 `addr` 落在某个 MemoryRegion 上，就进入该设备的 `write` 回调。
- `qtest_clock_step()` 可以推进虚拟时间，所以 WDT/PWM 可测试。

输出笔记：

```text
QTest 不需要 guest driver。它绕过 guest，直接测试 QEMU 设备模型。
这正适合训练营：先把虚拟硬件行为做对，再谈 guest 使用。
```

### 13:30-15:00 配置和编译

执行：

```bash
make -f Makefile.camp configure
make -f Makefile.camp build
```

如果失败，按类型处理：

```text
缺包：先补 build dependency
缺 Rust：安装 rustup/cargo/bindgen
缺 riscv64-unknown-elf-gcc：CPU 测试需要，SoC 本身主要用 riscv64-softmmu
Meson/Python 环境：优先用 build/pyvenv 里的 meson
```

注意：

- 不要一上来改测试。
- 不要为了过 configure 修改核心构建逻辑。
- 先记录错误，再判断是不是依赖问题。

输出笔记：

```text
构建状态：
configure: pass/fail
build: pass/fail
失败原因：
修复方式：
```

### 15:00-16:00 跑 SoC 基线

执行：

```bash
make -f Makefile.camp test-soc
```

如果要单独跑某个测试：

```bash
cd build
./pyvenv/bin/meson test --no-rebuild --print-errorlogs qtest-riscv64/test-board-g233
./pyvenv/bin/meson test --no-rebuild --print-errorlogs qtest-riscv64/test-gpio-basic
```

建立失败矩阵：

```text
test-board-g233: pass/fail
test-gpio-basic: pass/fail
test-gpio-int: pass/fail
test-pwm-basic: pass/fail
test-wdt-timeout: pass/fail
test-spi-jedec: pass/fail
test-flash-read: pass/fail
test-flash-read-interrupt: pass/fail
test-spi-cs: pass/fail
test-spi-overrun: pass/fail

当前得分估算：
已通过数量 / 10
```

预期：

- `test-board-g233` 可能通过，因为基础 machine 已经存在。
- 外设相关测试如果设备没挂，通常会因为 MMIO 未实现、读值不符或 QEMU abort 失败。

输出笔记：

```text
第一轮基线不是为了好看，而是为了知道每个后续改动增加了多少通过数。
```

### 16:00-17:30 读 SoC 测试总览

读：

```text
tests/gevico/qtest/meson.build
tests/gevico/qtest/test-board-g233.c
tests/gevico/qtest/test-gpio-basic.c
tests/gevico/qtest/test-gpio-int.c
tests/gevico/qtest/test-pwm-basic.c
tests/gevico/qtest/test-wdt-timeout.c
tests/gevico/qtest/test-spi-jedec.c
```

不要细读 SPI Flash 全部测试，今天只做总览。用下面命令快速看每个测试注册了哪些 case：

```bash
rg -n "qtest_add_func|#define .*BASE|PLIC IRQ|register map|GPIO_BASE|WDT_BASE|PWM_BASE|SPI_BASE" "tests/gevico/qtest"
```

整理表：

```text
测试文件                         外设        基地址        IRQ       核心行为
test-board-g233.c                board       -             -         machine/DRAM/PLIC/CLINT
test-gpio-basic.c                GPIO        0x10012000    -         DIR/OUT/IN/reset
test-gpio-int.c                  GPIO        0x10012000    2         edge/level/IE/IS/W1C/PLIC
test-pwm-basic.c                 PWM         0x10015000    -         channel/timer/DONE
test-wdt-timeout.c               WDT         0x10010000    4         countdown/feed/lock/timeout IRQ
test-spi-jedec.c                 SPI         0x10018000    -         CR/SR/DR/JEDEC
test-flash-read.c                SPI Flash   0x10018000    -         status/read/program/erase
test-flash-read-interrupt.c      SPI Flash   0x10018000    5         TXE/RXNE/ERR interrupt
test-spi-cs.c                    SPI Flash   0x10018000    -         CS0/CS1 independent flash
test-spi-overrun.c               SPI         0x10018000    5         RX overrun/W1C/ERRIE
```

输出笔记：

```text
SoC 10 题的最短路径：
board -> GPIO basic -> GPIO IRQ -> PWM -> WDT -> SPI JEDEC -> Flash -> SPI IRQ -> CS -> overrun
```

### 19:00-20:30 看 QEMU 外设建模流程

看：

https://qemu.gevico.online/tutorial/2026/ch2/qemu-hw/

只抓外设最小结构：

```text
typedef struct DeviceState {
    SysBusDevice parent_obj;
    MemoryRegion mmio;
    qemu_irq irq;
    寄存器状态字段;
} DeviceState;

static uint64_t read(void *opaque, hwaddr offset, unsigned size)
static void write(void *opaque, hwaddr offset, uint64_t value, unsigned size)

static const MemoryRegionOps ops = {
    .read = read,
    .write = write,
    .endianness = DEVICE_LITTLE_ENDIAN,
    .impl.min_access_size = 4,
    .impl.max_access_size = 4,
};

realize:
    memory_region_init_io()
    sysbus_init_mmio()
    sysbus_init_irq()

board:
    qdev_new()
    sysbus_realize_and_unref()
    sysbus_mmio_map()
    sysbus_connect_irq()
```

Day 2 写 GPIO 时就按这个模板来。

输出笔记：

```text
QEMU 外设不是驱动，而是虚拟硬件本体。
read/write 回调是寄存器访问入口。
update_irq() 是所有状态变化后的统一出口。
reset() 负责寄存器默认值。
```

### 20:30-21:30 为 Day 2 准备 GPIO 行为表

读：

```text
tests/gevico/qtest/test-gpio-basic.c
```

抄出断言：

```text
reset:
DIR=0 OUT=0 IN=0 IE=0 IS=0 TRIG=0 POL=0

direction:
write DIR=0x1 -> read DIR=0x1
write DIR=0xFFFFFFFF -> read DIR=0xFFFFFFFF

output:
DIR bit0=1
OUT bit0=1 -> IN bit0=1
OUT bit0=0 -> IN bit0=0

multi_pin:
DIR pins = 0,7,15,31
OUT pins -> IN pins all set
clear bit7 -> only bit7 clear, others stay set
```

写成实现计划：

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

read:
    DIR -> dir
    OUT -> out
    IN  -> out & dir
    IE  -> ie
    IS  -> is
    TRIG -> trig
    POL -> pol

write:
    DIR/OUT/IE/TRIG/POL 保存
    IS 做 write 1 clear，Day 2 可先保存，Day 3 完善
```

Day 2 的第一个目标不是优雅，而是 `test-gpio-basic` 通过。

## Day 1 完成标准

今天结束前必须有这些东西：

```text
1. build 是否成功
2. SoC 10 题失败矩阵
3. G233 简化地址空间表
4. QTest -> MemoryRegionOps 调用链说明
5. SoC 10 题外设/地址/IRQ 对照表
6. GPIO basic 行为表
7. Day 2 实现文件清单
```

## Day 1 自测问题

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
2. hw/riscv/g233.c 的 machine init，即当前文件中的 virt_machine_init。
3. 0x80000000。
4. 0x0C000000。
5. 0x10012000。
6. 4。
7. 5。
8. 相对该 MemoryRegion 基地址的 offset。
9. QTest 进程通过 qtest/QMP 直接访问 QEMU 设备模型地址空间。
10. 基础测试下为 OUT & DIR。
```

## Day 1 之后不要做的事

- 不要改测试。
- 不要一次性实现 GPIO/PWM/WDT/SPI。
- 不要先读完整 QEMU 源码。
- 不要追求完整硬件规格。
- 不要把 SPI Flash 和 `0x20000000` pflash 混为一谈。
- 不要在没跑基线前开始写代码。

## Day 2 开工口令

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

