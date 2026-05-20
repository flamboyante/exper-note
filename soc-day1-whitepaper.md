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

## 🧠 Day 1 核心心智模型

Day 1 最重要的不是把表格填满，而是在脑子里装下一张“QTest 如何打到 QEMU 外设”的地图。后面每个 `rg`、每个 `make`、每个测试失败，都是在验证这张图上的某一根线。

```text
测试端（Host）                         QEMU 虚拟硬件端（同一个 QEMU 进程内）
-------------------------              --------------------------------------
qtest_writel(0x10012004, val)
        |
        | 1. QTest 协议把“写物理地址”请求交给 QEMU
        v
QEMU 系统地址空间
        |
        | 2. 地址路由：0x10012004 落在 0x10012000 这个 GPIO MMIO 区间
        v
GPIO MemoryRegion
        |
        | 3. QEMU 把绝对地址换算成设备内 offset
        |    0x10012004 - 0x10012000 = 0x04
        v
GPIO 的 MemoryRegionOps.write
        |
        | 4. 设备模型按 offset 更新自己的 state
        v
s->out = val
```

你今天要建立四条连线：

```text
-machine g233
-> G233 machine init

G233 machine init
-> 创建 CPU/内存/中断控制器/外设地址空间

qtest_writel / qtest_readl
-> MemoryRegionOps.write / read

测试断言
-> 外设寄存器语义
```

📌 读这份文档时的规则：

- 看到命令，先问“它要验证哪条连线”。
- 看到输出，先找“应该圈出的关键符号”。
- 看到表格，别只抄答案，要写一句因果解释。
- 看到失败，别急着修，先判断是哪条连线断了。

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

✅ 标准答案：

```text
SoC 测试入口：
tests/gevico/qtest/

SoC 基础测试数量：
当前 Day 1 主线按 10 个基础 SoC 测试理解。

今天最先要跑的命令：
make -f Makefile.camp configure
make -f Makefile.camp build
make -f Makefile.camp test-soc

今天不做的事情：
不改测试，不写外设实现，不追完整 QEMU 源码，不提前展开 SPI/WDT/PWM 的细节。
```

---

## 2. 看 Machine model：`-machine g233` 怎么进代码

### 📘 你要看什么

- Machine model: <https://qemu.gevico.online/tutorial/2025/ch2/sec4/c-model/machine-model/>
- [hw/riscv/g233.c](../hw/riscv/g233.c)
- [include/hw/riscv/g233.h](../include/hw/riscv/g233.h)

### 🔍 本地搜索

这一步要回答的问题：敲了 `-machine g233` 之后，QEMU 到底调了哪个 C 函数来初始化这块板？

核心概念（30 秒）：QEMU 用 `MachineClass.init` 把命令行里的 machine 名字映射到一个 board init 函数。我们搜的不是普通字符串，而是“命令行参数 -> C 函数入口”的绑定关系。

在外层实验仓库执行：

```bash
cd "/home/flamboy/qemu-camp-2026-exper-flamboyante"
rg -n "TYPE_RISCV_G233_MACHINE|virt_machine_init|machine_class_init|mc->init" \
  "hw/riscv/g233.c" "include/hw/riscv/g233.h"
```

预期画面：

```text
你应该看到：
1. G233 machine 的类型名或宏。
2. class_init 函数。
3. 类似 mc->init = xxx 的赋值。

请圈出：
mc->init 右边那个函数名。

它意味着：
这个函数就是 -machine g233 之后真正开始搭虚拟板子的入口。
```

### 🧠 你要理解什么

QEMU 不是看到 `-machine g233` 就立刻调用 G233 的 board init。更准确地说，这里分成两件事：先“选中并创建 machine 对象”，再“退出 preconfig 后真正初始化板子”。

```text
main()
-> qemu_init(argc, argv)
   -> 解析命令行
      -> 处理 -machine g233
      -> machine_opts_dict["type"] = "g233"

   -> qemu_create_machine(machine_opts_dict)
      -> select_machine(qdict)
         -> machine_type = "g233"
         -> 枚举所有 MachineClass
            -> 必要时触发 type_initialize()
               -> 调用 G233 的 machine class init
               -> mc->init = virt_machine_init
         -> find_machine("g233")
            -> 找到 G233 对应的 MachineClass

      -> object_new_with_class(OBJECT_CLASS(machine_class))
         -> 创建 current_machine 实例
         -> 调用 G233 的 instance init

-> qmp_x_exit_preconfig()
   -> qemu_init_board()
      -> machine_run_board_init(current_machine, ...)
         -> machine_class->init(machine)
            -> virt_machine_init(machine)
```

这里最容易混淆的是：`mc->init = virt_machine_init` 是在 machine class 初始化时把函数指针登记好；`machine_class->init(machine)` 才是后面真正开始搭 G233 板级硬件时调用它。

如果你想把这段看得更“贴源码”，可以记成下面这个深入版：

```text
命令行 -M g233：
    你只知道用户输入的是短名字 g233。
    所以 QEMU 必须先枚举 MachineClass 列表，再按 mc->name / mc->alias 找匹配项。

object_class_get_list(target_machine_typename(), false)
-> 遍历已注册的 TypeImpl
-> 对候选 type 做 type_initialize(type)
   -> 如果 class 已经存在，直接 return
   -> 否则分配 class 空间
   -> 初始化父类 class
   -> 复制父类 class 内容到子类 class
   -> 调用父类 class_base_init
      -> 对 Machine 来说，会从 g233-machine 生成 mc->name = g233
   -> 调用 G233 的 class_init
      -> virt_machine_class_init(ObjectClass *oc, ...)
         -> MachineClass *mc = MACHINE_CLASS(oc)
            这里只是类型检查 + 指针转换，不分配空间
         -> mc->desc = "RISC-V VirtIO board"
         -> mc->default_cpu_type = TYPE_RISCV_CPU_GEVICO_CV1
         -> mc->init = virt_machine_init

select_machine()
-> find_machine("g233", machines)
-> 找到 G233 的 MachineClass

object_new_with_class(OBJECT_CLASS(machine_class))
-> 已经知道精确 QOM 类型
-> object_new_with_type(klass->type)
-> type_initialize(g233-machine)
   -> class 已经初始化过，直接 return
-> 分配实例空间
   -> 实际对象类型是 RISCVG233State
   -> 里面包含 MachineState parent
-> 先跑父类 instance_init
-> 再跑 G233 的 virt_machine_instance_init()
   -> virt_flash_create(s)
      注意：这里只是创建 flash 对象，还没映射到地址空间
   -> 初始化 acpi / iommu_sys 等默认属性

后面退出 preconfig：
qmp_x_exit_preconfig()
-> qemu_init_board()
-> machine_run_board_init(current_machine, ...)
   -> MACHINE_GET_CLASS(machine)
      从 current_machine 实例拿回 G233 的 MachineClass
   -> 做 RAM / NUMA / CPU type / accelerator 等通用检查
   -> machine_class->init(machine)
      -> virt_machine_init(machine)
         这里才真正创建 CPU array、中断控制器、RAM、ROM、virtio、PCIe、UART、RTC、flash map、FDT 等板级硬件。
```

对比一下就更清楚了：

```text
直接 object_new("g233-machine")：
    已经知道精确 QOM 类型，只需要初始化父类链 + g233，然后创建实例。

命令行 -M g233：
    只知道用户输入的短名字 g233。
    必须枚举 MachineClass，触发很多 machine class 的初始化，再按 mc->name / mc->alias 找到 G233。
    但只有 G233 会创建 current_machine 实例，也只有 G233 后面会跑 virt_machine_init。
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

✅ 标准答案：

```text
G233 machine 类型名：
TYPE_RISCV_G233_MACHINE，也就是 MACHINE_TYPE_NAME("g233")。

G233 machine init 函数：
virt_machine_init(MachineState *machine)。

MachineClass.init 设置位置：
hw/riscv/g233.c 的 virt_machine_class_init() 中：
mc->init = virt_machine_init;

后续 GPIO 应该在哪里实例化：
设备本体放在独立 GPIO 设备文件中；板级实例化和 MMIO map 放在 hw/riscv/g233.c 的 G233 board init 路径中。
```

---

## 3. 看 G233 板级代码：地址空间和已有外设

### 📘 你要看什么

- [include/hw/riscv/g233.h](../include/hw/riscv/g233.h)
- [hw/riscv/g233.c](../hw/riscv/g233.c)

### 🔍 本地搜索

这一步要回答的问题：G233 这块虚拟板把 CPU、内存、PLIC、UART、未来的 GPIO/WDT/PWM/SPI 都放到了哪些地址？

核心概念（30 秒）：QEMU machine init 就像在搭一张“虚拟电路板地址图”。测试写的是物理地址，QEMU 必须先知道这个地址属于哪个 `MemoryRegion`，才能把访问转给对应设备。

```bash
rg -n "virt_memmap|VIRT_PLIC|VIRT_CLINT|VIRT_UART0|VIRT_DRAM|memory_region_add_subregion|sysbus_create_simple|qdev_get_gpio_in|qemu_set_irq|plic" \
  "hw/riscv/g233.c" "include/hw/riscv/g233.h"
```

预期画面：

```text
你应该看到：
1. 一张类似 memmap / virt_memmap 的地址表。
2. CLINT、PLIC、UART、DRAM 等模块的 base 地址。
3. memory_region_add_subregion 或 sysbus_mmio_map 之类的挂载代码。

请圈出：
GPIO/WDT/PWM/SPI 的 base 地址，以及把设备挂进地址空间的 API。

它意味着：
测试里的地址不是凭空来的，它必须和 board 代码里的地址表对上。
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

这里要特别小心：当前 `g233.c` 的 `virt_memmap` 里已经能直接看到 CLINT、PLIC、UART0、VirtIO、pflash、DRAM 等既有区域；`WDT/GPIO/PWM/SPI` 这几个地址主要来自训练营测试和硬件手册，是后续 SoC 实验要你补进去的目标地址。不要因为现在在 `virt_memmap` 里找不到 `VIRT_GPIO` 就以为测试写错了，恰恰相反，这说明 Day 2 之后你要把设备挂上去。

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

这一步要回答的问题：为什么我们不先写 guest driver，也能测试 GPIO/WDT/SPI 这些外设？

核心概念（30 秒）：QTest 是 host 侧的“遥控器”。它不需要在 guest 里跑 C 程序，而是直接通过 QTest 协议让 QEMU 对 guest 物理地址做读写。

```bash
sed -n '1,160p' "tests/gevico/qtest/test-gpio-basic.c"
sed -n '1,180p' "tests/gevico/qtest/test-wdt-timeout.c"
```

预期画面：

```text
你应该看到：
1. qtest_init("-machine g233 ...")。
2. qtest_writel / qtest_readl。
3. g_assert_cmpuint 之类的断言。

请圈出：
测试写入的地址、写入的值、随后断言期待读到什么。

它意味着：
测试文件就是外设行为规格书。后面写设备时，不是靠猜，而是让 read/write 满足这些断言。
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

### 🔬 显微镜：`0x10012004` 怎么变成 `offset = 0x04`

这一步要回答的问题：测试写的是绝对物理地址，为什么设备代码里的 `.write` 回调只看到了一个小小的 offset？

核心概念（30 秒）：设备模型不关心“我在整块 SoC 的绝对地址是多少”，它只关心“这次访问落在我自己的 MMIO 区域里的第几个寄存器”。

```text
测试代码：
qtest_writel(qts, 0x10012004, value)

QEMU 地址路由：
0x10012004 落在 GPIO MMIO 区间
GPIO base = 0x10012000

QEMU 自动换算：
0x10012004 - 0x10012000 = 0x04

设备回调看到：
g233_gpio_write(opaque, offset = 0x04, value, size = 4)

GPIO 设备解释：
0x04 对应 GPIO_OUT
所以应该更新 s->out
```

所以，设备代码里通常不会看到 `0x10012004` 这个绝对地址。它看到的是 `0x04`，因为 QEMU 已经帮它完成了“系统地址空间 -> 设备内部寄存器偏移”的翻译。

✅ 推演答案：

```text
如果测试写 0x10012000，回调 offset 是：
0x00。

它对应的 GPIO 寄存器是：
GPIO_DIR。

如果测试写 0x10012008，回调 offset 是：
0x08。

它对应的 GPIO 寄存器是：
GPIO_IN。

如果 write 回调完全没进，最可能断在哪三处：
1. GPIO 设备没有编译进 QEMU 二进制。
2. G233 machine 没有创建/realize GPIO 设备。
3. GPIO 没有通过 sysbus_mmio_map() 映射到 0x10012000。
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

✅ 标准答案：

```text
addr 是：
QTest 侧传入的 guest 物理地址，比如 0x10012004。

offset 是：
QEMU 命中某个 MemoryRegion 后，换算出来的设备内部寄存器偏移，比如 0x04。

GPIO_OUT 的 guest 物理地址：
0x10012004。

GPIO_OUT 在回调里的 offset：
0x04。

QTest 为什么不需要 guest driver：
因为 QTest 在 host 侧控制 QEMU，直接对 guest 物理地址空间发起读写请求，不需要 guest Linux 里跑驱动。
```

---

## 5. 配置、编译和基线测试

### ✅ 你现在就做

这一步要回答的问题：你的本地环境能不能把 QEMU 编出来、把 SoC 测试拉起来？

核心概念（30 秒）：`configure/build/test-soc` 不是打卡，而是在确认“工具链 -> QEMU 二进制 -> QTest 测试入口”这条工程链路没断。只有这条链通了，后面外设失败才有调试意义。

在外层实验仓库执行：

```bash
cd "/home/flamboy/qemu-camp-2026-exper-flamboyante"
make -f Makefile.camp configure
make -f Makefile.camp build
make -f Makefile.camp test-soc
```

预期画面：

```text
configure：
    你希望看到配置完成，关键依赖没有缺。

build：
    你希望看到 qemu-system-riscv64 等目标能被编译出来。

test-soc：
    你可能会看到多个 SoC 测试失败，这是正常的。
    今天要圈出“失败在哪个测试、第一条关键 assert 是什么”。
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

✅ 这一节的结论：

```text
构建与测试命令记录：
只需要保留 configure/build/test-soc 三条命令的实际结果和第一条关键错误，不需要贴全量日志。

SoC 10 题失败矩阵：
不是为了打分焦虑，而是为了把失败归类到 board/GPIO/PWM/WDT/SPI/环境。

当前得分：
Day 1 不以通过数量为目标。Day 1 的真正目标是知道第一处失败属于哪条链，Day 2 从 GPIO basic 开始闭环。
```

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

这一步要回答的问题：训练营 10 个 SoC 测试分别在测哪类外设能力，为什么 Day 2 要从 GPIO basic 开始？

核心概念（30 秒）：测试顺序本身就是学习路线。它从最简单的 MMIO 读写开始，逐步加上 IRQ、timer、协议状态机和错误处理。

```bash
rg -n "qtest_add_func|#define .*BASE|PLIC IRQ|register map|GPIO_BASE|WDT_BASE|PWM_BASE|SPI_BASE" \
  "tests/gevico/qtest"
```

预期画面：

```text
你应该看到：
1. 每个 test_xxx.c 注册了哪些 qtest case。
2. GPIO_BASE / WDT_BASE / PWM_BASE / SPI_BASE。
3. 某些测试里出现 IRQ 编号、状态位、寄存器表。

请圈出：
每个测试的 base 地址和它最核心的寄存器行为。

它意味着：
Day 2 从 GPIO basic 开始不是因为它“简单无聊”，而是因为它最适合验证 MemoryRegionOps + state + mmio map 这条主链。
```

如果目录里还出现 Rust/I2C 相关测试，先不要被它们带跑。Day 1 这里说的“基础 10 题”是 SoC 主线的第一组目标，先把 GPIO/PWM/WDT/SPI 这条路线打通，再回头看扩展题。

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

✅ 标准答案：

```text
一个最小 SysBusDevice 外设需要哪些字段：
SysBusDevice parent_obj;
MemoryRegion mmio;
必要时 qemu_irq irq;
寄存器/内部状态字段，比如 uint32_t dir/out/ie/is/trig/pol。

MemoryRegionOps 的 read/write 参数分别代表什么：
opaque 是设备 state 指针；
offset 是相对该设备 MMIO base 的寄存器偏移；
value 是写入值；
size 是访问宽度。

Day 2 GPIO 需要先实现哪些寄存器：
GPIO_DIR、GPIO_OUT、GPIO_IN；
GPIO_IE/GPIO_IS/GPIO_TRIG/GPIO_POL 至少 reset 后读 0，并为 Day 3 预留状态字段。

Day 3 GPIO IRQ 需要新增哪些状态：
IE mask、IS pending、TRIG edge/level、POL polarity、上一拍输入/输出状态、qemu_irq irq，以及统一的 update_irq()。
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
GPIO_IE   basic 只检查 reset 为 0，Day 3 才验证写入和中断使能语义
GPIO_IS   basic 只检查 reset 为 0，Day 3 才验证 W1C 和中断状态语义
GPIO_TRIG basic 只检查 reset 为 0，Day 3 才验证 edge/level 语义
GPIO_POL  basic 只检查 reset 为 0，Day 3 才验证 polarity 语义
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

上面这套写法是“为 Day 3 预留”的实现策略；如果只从 `test-gpio-basic.c` 的断言看，它真正强制验证的是 reset 值、`DIR` 可读写、`OUT` 影响 `IN`、多 bit 不互相污染。`IE/IS/TRIG/POL` 在 basic 里只出现在 reset_value 测试中。

⚠️ 容易踩坑：

- `GPIO_IN` 不要简单等于 `GPIO_OUT`，basic 测试里需要考虑方向位。
- `GPIO_IS` 在 basic 阶段不会被写入测试，但后续中断测试需要 W1C，所以你可以提前按 W1C 预留，不要误以为 basic 已经验证了它。
- Day 2 先让 `test-gpio-basic` 通过，不要把 GPIO IRQ 一起写进去。

✅ 标准答案：

- `GPIO basic 行为表`
- Day 2 实现文件清单
- Day 2 第一条单项测试命令

```text
GPIO basic 行为表：
reset 全 0；DIR 可读写；OUT 可读写；IN = OUT & DIR；多 bit 独立。

Day 2 实现文件清单：
include/hw/gpio/g233_gpio.h
hw/gpio/g233_gpio.c
hw/gpio/meson.build
hw/riscv/g233.c
必要时 include/hw/riscv/g233.h 或相关 Kconfig/Meson 配置。

Day 2 第一条单项测试命令：
cd build
./pyvenv/bin/meson test --no-rebuild --print-errorlogs qtest-riscv64/test-gpio-basic
```

---

## 9. Day 1 执行记录

下面这部分不再让你从零填表。概念类答案我已经直接写好；只有“你的机器实际跑出来的日志”需要你执行时补一两行关键输出。

### 9.1 构建与测试命令记录

| 时间 | 命令 | 结果 | 关键输出 / 错误摘要 | 处理方式 |
| --- | --- | --- | --- | --- |
| 执行时记录 | `make -f Makefile.camp configure` | 以本机输出为准 | 通过时说明依赖和配置基本可用；失败时重点看缺包、Meson、Python、工具链错误 | configure 失败就先修环境，不继续猜外设问题 |
| 执行时记录 | `make -f Makefile.camp build` | 以本机输出为准 | 通过时说明 QEMU 二进制能编出来；失败时重点看编译错误和新增文件是否进构建 | build 失败先修编译，不跑 qtest |
| 执行时记录 | `make -f Makefile.camp test-soc` | 以本机输出为准 | 早期多个 SoC 测试失败是正常现象；重点看第一个失败测试和断言 | 把失败归类到 board / GPIO / PWM / WDT / SPI / 环境 |

标准判读：

```text
configure 是否通过：
这是环境和配置链路是否成立。

build 是否通过：
这是 QEMU 二进制和新增代码是否能进入编译链路。

test-soc 是否跑完：
这是 QTest 能否启动 G233 machine 并访问测试入口。

最先失败的命令：
如果是 configure/build，优先当环境或编译问题处理。
如果是 test-soc，优先当设备模型或 board map 问题处理。

最关键的错误信息：
只保留第一条失败断言、缺包名、编译错误位置或 qtest 失败用例名。

下一步处理：
不要立刻大改代码，先判断断在哪条链：环境、构建、machine、MMIO map、read/write、IRQ/timer/协议。
```

标准理解：

```text
我跑 configure 是为了确认：
依赖、Meson/Python、交叉工具链、项目配置入口是否可用。

我跑 build 是为了得到：
能被 QTest 启动的 qemu-system-riscv64，以及确认新增源文件会不会参与编译。

我跑 test-soc 不是为了立刻全过，而是为了看到：
当前 10 个 SoC 基础测试分别失败在哪个模块，形成后续实现顺序。

如果 configure/build 失败，说明断在：
工程环境或编译链路，还没到设备模型行为验证。

如果只有 qtest 失败，说明工程链路大概率已经走到：
QEMU 能启动，测试能发起访问，问题更可能在 machine 挂载、MMIO 回调、寄存器语义或 IRQ/timer/协议模型。
```

### 9.2 SoC 10 题失败矩阵

| 测试 | 预期阶段 | 失败时优先怀疑 | 初步判断 |
| --- | --- | --- | --- |
| `test-board-g233` | Day 1 基线 | machine 是否能启动，DRAM/PLIC/CLINT 是否存在 | board 框架问题 |
| `test-gpio-basic` | Day 2 | GPIO 是否编译、创建、映射到 `0x10012000`，DIR/OUT/IN/reset 是否正确 | 最小 MMIO 外设问题 |
| `test-gpio-int` | Day 3 | IE/IS/TRIG/POL、W1C、`qemu_set_irq()`、PLIC IRQ 2 | GPIO 中断问题 |
| `test-pwm-basic` | 后续 PWM | channel、counter、enable、done flag、polarity、时间推进 | PWM/timer 问题 |
| `test-wdt-timeout` | 后续 WDT | LOAD/VAL/CTRL/KEY/SR、feed/lock、timeout、PLIC IRQ 4 | WDT/timer/IRQ 问题 |
| `test-spi-jedec` | 后续 SPI | SPI CR/SR/DR、TX/RX 状态、JEDEC ID 状态机 | SPI 协议问题 |
| `test-flash-read` | 后续 SPI Flash | flash status、read/program/erase、数据持久状态 | flash 状态机问题 |
| `test-flash-read-interrupt` | 后续 SPI IRQ | TXE/RXNE/ERR 中断、PLIC IRQ 5 | SPI 中断问题 |
| `test-spi-cs` | 后续 SPI CS | CS0/CS1 选择、多个 flash 独立状态 | 片选/多设备问题 |
| `test-spi-overrun` | 后续 SPI 错误 | RX overrun、ERRIE、W1C、PLIC IRQ 5 | 错误状态/中断问题 |

当前得分：

```text
通过数量：
以你运行结果为准。Day 1 重点不是分数，而是知道失败分布。

失败数量：
以你运行结果为准。外设未实现时失败是预期现象。

阻塞点：
如果 board 都不过，先查 machine 基线；如果 board 过而 GPIO basic 不过，Day 2 从 GPIO 设备开始。
```

标准理解：

```text
第一个失败的测试是：
它决定你先修哪条链；不要越过第一个基础失败去写后面复杂外设。

它属于哪类问题：board / GPIO / PWM / WDT / SPI / 环境
这决定排查入口。

这个失败说明哪条链可能没接上：
环境 -> 构建 -> machine -> MMIO map -> MemoryRegionOps -> state/IRQ/timer。

我下一步不应该乱改，而应该先确认：
失败地址、失败断言、对应设备是否已创建、对应 read/write 是否会被打到。
```

### 9.3 G233 简化地址空间表

| 模块 | 地址 | 当前状态 | 证据位置 |
| --- | --- | --- | --- |
| CLINT | `0x02000000` | 既有基础设备区域 | `hw/riscv/g233.c` 的 `virt_memmap` / `VIRT_CLINT` |
| PLIC | `0x0C000000` | 既有中断控制器区域 | `hw/riscv/g233.c` 的 `virt_memmap` / `VIRT_PLIC` |
| UART0 | `0x10000000` | 既有串口区域 | `hw/riscv/g233.c` 的 `virt_memmap` / `VIRT_UART0` |
| VirtIO MMIO | `0x10001000` | 既有 VirtIO MMIO 起始区域 | `hw/riscv/g233.c` 的 `virt_memmap` / `VIRT_VIRTIO` |
| WDT | `0x10010000` | SoC 实验目标设备 | `tests/gevico/qtest/test-wdt-timeout.c` |
| GPIO | `0x10012000` | SoC 实验目标设备 | `tests/gevico/qtest/test-gpio-basic.c` / `test-gpio-int.c` |
| PWM | `0x10015000` | SoC 实验目标设备 | `tests/gevico/qtest/test-pwm-basic.c` |
| SPI | `0x10018000` | SoC 实验目标设备 | `tests/gevico/qtest/test-spi-*.c` / `test-flash-*.c` |
| pflash | `0x20000000` | 既有并行 flash 区域 | `hw/riscv/g233.c` 的 `VIRT_FLASH` / `virt_flash_map()` |
| DRAM | `0x80000000` | 既有内存区域 | `hw/riscv/g233.c` 的 `VIRT_DRAM` |

把这张表再升级成“逻辑连线”：

| 模块 | 基地址 | 谁把它挂上去的 API | 谁会访问它 |
| --- | --- | --- | --- |
| GPIO | `0x10012000` | 需要你后续在 `g233.c` 里通过 `sysbus_mmio_map()` 挂上 | `test-gpio-basic.c` / `test-gpio-int.c` |
| WDT | `0x10010000` | 需要你后续在 `g233.c` 里通过 `sysbus_mmio_map()` 挂上 | `test-wdt-timeout.c` |
| PWM | `0x10015000` | 需要你后续在 `g233.c` 里通过 `sysbus_mmio_map()` 挂上 | `test-pwm-basic.c` |
| SPI | `0x10018000` | 需要你后续在 `g233.c` 里通过 `sysbus_mmio_map()` 挂上 | `test-spi-jedec.c` / `test-spi-cs.c` / `test-spi-overrun.c` |

标准理解：

```text
因为 GPIO 被挂到：
0x10012000。

所以测试写：
0x10012000 + offset，例如 GPIO_OUT 是 0x10012004。

会命中：
GPIO 设备的 MemoryRegionOps.read/write。

如果测试地址和 board 里 map 的地址不一致，结果会是：
QTest 访问不会命中目标设备，read/write 回调不会按预期触发。

我验证一个地址是否可信，至少要同时看这两个地方：
1. tests/gevico/qtest/test-xxx.c 里的 BASE 定义。
2. hw/riscv/g233.c 里的设备创建和 sysbus_mmio_map()。
```

### 9.4 QTest 到 MMIO 回调调用链

标准调用链：

```text
qtest_writel(qts, addr, value)
-> QTest 协议把 guest 物理地址写请求发给 QEMU
-> QEMU 在系统地址空间中查找 addr 命中的 MemoryRegion
-> QEMU 把 addr 换算成该 MemoryRegion 内部 offset
-> 调用设备的 MemoryRegionOps.write(opaque, offset, value, size)
-> 设备根据 offset 修改自己的 state 字段
```

需要确认的问题：

```text
addr 是 guest 物理地址还是 offset：
guest 物理地址。

MemoryRegionOps.write 里的 offset 是什么：
相对设备 MMIO base 的寄存器偏移。

为什么 SoC 测试不需要 guest driver：
因为 QTest 在 host 侧直接驱动 QEMU 的 guest 物理地址空间，不需要 guest 内核参与。
```

标准理解：

```text
qtest_writel 写的是：
guest 物理地址。

MemoryRegionOps.write 收到的是：
设备内部 offset。

二者中间由 QEMU 的哪一层完成翻译：
系统地址空间 / MemoryRegion 路由。

设备代码里看不到绝对地址，是因为：
绝对地址已经在地址空间层被消费掉了，设备只负责解释自己的寄存器偏移。
```

### 9.5 SoC 测试对照表

| 测试文件 | 外设 | 基地址 | IRQ | 核心行为 |
| --- | --- | --- | --- | --- |
| `test-board-g233.c` | board | - | - | machine 启动、DRAM、PLIC、CLINT |
| `test-gpio-basic.c` | GPIO | `0x10012000` | - | reset、DIR、OUT、IN、多 pin |
| `test-gpio-int.c` | GPIO | `0x10012000` | `2` | edge/level、IE、IS、W1C、PLIC |
| `test-pwm-basic.c` | PWM | `0x10015000` | - | config、enable、counter、done flag、多 channel、polarity |
| `test-wdt-timeout.c` | WDT | `0x10010000` | `4` | LOAD/VAL/CTRL/KEY/SR、feed、lock、timeout、IRQ |
| `test-spi-jedec.c` | SPI | `0x10018000` | - | SPI init、transfer byte、JEDEC ID |
| `test-flash-read.c` | SPI Flash | `0x10018000` | - | flash status、sector erase、page program、read data |
| `test-flash-read-interrupt.c` | SPI Flash | `0x10018000` | `5` | TXE/RXNE/JEDEC/write data interrupt |
| `test-spi-cs.c` | SPI Flash | `0x10018000` | - | CS0/CS1、独立 flash、cross/alternating/capacity |
| `test-spi-overrun.c` | SPI | `0x10018000` | `5` | RX overrun、polling、ERRIE、PLIC IRQ |

标准理解：

```text
这些测试不是随机顺序，它们的复杂度递进是：
board -> GPIO basic -> GPIO interrupt -> PWM/WDT -> SPI/Flash/CS/overrun。

GPIO basic 最适合先做，是因为它暂时不需要：
IRQ、timer、复杂协议状态机。

WDT/PWM 比 GPIO basic 多出来的关键机制是：
虚拟时间、计数推进、状态 flag、必要时 IRQ。

SPI 比 GPIO basic 多出来的关键机制是：
TX/RX 状态、FIFO/数据寄存器、flash 协议状态机、片选和错误状态。
```

### 9.6 GPIO basic 行为表

从 [test-gpio-basic.c](../tests/gevico/qtest/test-gpio-basic.c) 里逐条提取，不要凭感觉写。

| 场景 | 测试动作 | 期望结果 | 实现提示 |
| --- | --- | --- | --- |
| reset | 启动 `-machine g233` 后读 `DIR/OUT/IN/IE/IS/TRIG/POL` | 全部为 0 | reset 清空所有 GPIO 状态字段 |
| direction | 写 `GPIO_DIR=0x1` 和 `0xFFFFFFFF` 后读回 | 读回值与写入一致 | `s->dir = value` |
| output bit0 | `DIR=0x1`，写 `OUT=1/0`，读 `IN & 1` | `OUT=1` 时为 1，`OUT=0` 时为 0 | `GPIO_IN` 返回 `out & dir` |
| multi pin | `DIR/OUT` 写 bit0/7/15/31，再清 bit7 | bit7 清掉，其他 bit 保持 | 按 32 位 mask 处理，不要写死 bit0 |
| interrupt registers reset only | reset 后读 `IE/IS/TRIG/POL` | 全部为 0 | basic 只测 reset，写语义留给 Day 3 |

标准理解：

```text
GPIO_DIR 表示：
每个 pin 的方向，0=input，1=output。

GPIO_OUT 表示：
软件写出的输出锁存值。

GPIO_IN 为什么可以用 OUT & DIR：
basic 阶段没有外部输入模型，测试只要求输出方向下能从 IN 读回 OUT。

如果 DIR=0x1 且 OUT=0x3，IN 应该是：
0x1。

这个结果说明 read 回调里不能简单返回：
不能简单返回 out，也不能无视 dir。
```

### 9.7 Day 2 开工前检查

| 检查项 | 是否完成 | 备注 |
| --- | --- | --- |
| 环境能 configure/build | 执行后确认 | 这是工程链路，不是概念题 |
| 已跑过 `test-soc` 基线 | 执行后确认 | 目的是看到当前失败分布 |
| 已知道 GPIO 基地址 | 是 | `0x10012000` |
| 已知道 GPIO 寄存器 offset | 是 | `DIR=0x00`、`OUT=0x04`、`IN=0x08`、`IE=0x0C`、`IS=0x10`、`TRIG=0x14`、`POL=0x18` |
| 已知道 GPIO basic 测什么 | 是 | reset、DIR、OUT/IN、多 pin |
| 已知道新增设备文件大概放哪里 | 是 | `include/hw/gpio/g233_gpio.h`、`hw/gpio/g233_gpio.c`、`hw/gpio/meson.build`、`hw/riscv/g233.c` |

标准理解：

```text
我可以开始 Day 2，不是因为表填完了，而是因为我能解释：
qtest_writel(0x10012004, 1)
-> QTest 写 guest 物理地址 0x10012004
-> QEMU 地址空间命中 GPIO base 0x10012000
-> QEMU 计算 offset = 0x04
-> 调用 GPIO MemoryRegionOps.write
最终应该修改：
s->out
```

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

### 10.1 分模块结论速查：Machine / QOM

这部分你不用背问题，直接记住下面这些结论：

| 模块 | 你应该直接掌握的答案 |
| --- | --- |
| `-M g233` 的含义 | 命令行只给了短名字 `g233`，QEMU 需要枚举 `MachineClass`，按 `mc->name` 或 alias 找到匹配项。它不等价于直接 `object_new("g233-machine")`。 |
| `mc->init = virt_machine_init` | 这一步发生在 `virt_machine_class_init()`，只是把真正的板级初始化入口登记到 `MachineClass` 里，不会立刻搭板子。 |
| `virt_machine_init(machine)` | 真正执行时间在退出 preconfig 之后：`qmp_x_exit_preconfig()` -> `qemu_init_board()` -> `machine_run_board_init()` -> `machine_class->init(machine)`。 |
| `MACHINE_CLASS(oc)` | 它主要是类型检查和指针转换，把 `ObjectClass *` 当成 `MachineClass *` 使用，不分配新对象。 |
| `instance_init` vs board init | `virt_machine_instance_init()` 是对象实例初始化，可以创建属性或子对象占位，比如 `virt_flash_create(s)`；`virt_machine_init()` 才真正搭 CPU、内存、中断控制器、flash map 等板级硬件。 |

一句话版本：

```text
class_init 负责登记“这类 machine 怎么初始化”，instance_init 负责创建 machine 实例的初始对象状态，machine_class->init 才真正把 G233 板子搭起来。
```

### 10.2 分模块结论速查：G233 地址空间

| 模块 | 你应该直接掌握的答案 |
| --- | --- |
| 既有地址图 | 当前 `g233.c` 的 `virt_memmap` 主要能看到 CLINT、PLIC、UART0、VirtIO、pflash、DRAM 等既有区域。 |
| SoC 实验目标地址 | `GPIO=0x10012000`、`WDT=0x10010000`、`PWM=0x10015000`、`SPI=0x10018000` 主要来自测试和手册，是后续要挂进 G233 machine 的目标区域。 |
| 地址的两端 | 测试告诉你“谁会访问这个地址”，board 代码告诉你“谁把设备挂到这个地址”。两端对上，MMIO 链路才闭合。 |
| pflash vs SPI flash | `0x20000000` 是已有并行 flash 地址区；SPI flash 测试要通过 `0x10018000` 的 SPI controller 间接访问 flash 状态机。 |
| 访问不到设备时的定位顺序 | 先查设备是否编译进来，再查 board 是否创建设备，最后查 `sysbus_mmio_map()` 的 base 是否和测试里的 base 一致。 |

一句话版本：

```text
地址不是“记数字”，而是要把测试里的访问地址和 board 里的设备映射钉在一起。
```

### 10.3 分模块结论速查：QTest / MMIO

| 模块 | 你应该直接掌握的答案 |
| --- | --- |
| `qtest_writel` 的地址视角 | `qtest_writel(qts, 0x10012004, 1)` 写的是 guest 物理地址，不是设备内部 offset。 |
| 绝对地址到 offset | QEMU 地址空间先匹配 GPIO 的 `MemoryRegion`，再用访问地址减 GPIO base：`0x10012004 - 0x10012000 = 0x04`。 |
| 设备代码看什么 | 设备代码应该 switch 自己的寄存器 offset，比如 `0x04`，不应该 switch `0x10012004` 这种绝对地址。 |
| 为什么不用 guest driver | QTest 在 host 侧控制 QEMU，直接通过协议读写 guest 物理地址，所以可以绕过 guest 内核和驱动。 |
| 测试文件的价值 | 测试文件就是可执行规格：它写清楚了访问哪些寄存器、期望什么值、哪些副作用必须发生。 |

一句话版本：

```text
QTest 写绝对物理地址，QEMU 地址空间负责路由，设备 read/write 只处理相对 offset。
```

### 10.4 分模块结论速查：GPIO basic

| 模块 | 你应该直接掌握的答案 |
| --- | --- |
| basic 真正验证什么 | reset 后 7 个寄存器为 0；`GPIO_DIR` 可读写；`GPIO_OUT` 通过 `GPIO_IN` 体现；多 bit 操作互不污染。 |
| basic 不验证什么 | `GPIO_IE/IS/TRIG/POL` 的写语义不在 basic 里验证；basic 只读它们 reset 后是否为 0。 |
| `GPIO_IN` 的最小语义 | basic 阶段用 `OUT & DIR`。只有配置成输出的 pin 才读回输出值。 |
| 一个例子 | `DIR=0x1`、`OUT=0x3` 时，`IN=0x1`，因为 bit1 没被配置为输出。 |
| multi_pin 的意义 | 它防止实现只处理 bit0；你的实现必须支持 32 位 bit mask，bit0/7/15/31 要独立工作。 |

一句话版本：

```text
GPIO basic 的核心不是中断，而是证明 MMIO map、state、read/write offset 和 bit mask 都正确。
```

### 10.5 分模块结论速查：GPIO interrupt

| 模块 | 你应该直接掌握的答案 |
| --- | --- |
| 相比 basic 多出的语义 | `test-gpio-int.c` 多了 edge/level 触发、polarity、IE mask、IS 状态、W1C 清除、PLIC IRQ 2。 |
| `GPIO_TRIG` | `0` 表示 edge，`1` 表示 level。 |
| `GPIO_POL` | edge 模式下 `1` 表示 rising；level 模式下 `1` 表示 high。 |
| `GPIO_IS` | 不能只是普通赋值寄存器；测试要求 write-1-to-clear，写 1 清对应 pending bit。 |
| PLIC 观察路径 | 测试会设置 priority、threshold、enable，然后通过 PLIC pending/claim 观察 GPIO IRQ 2。 |

一句话版本：

```text
GPIO interrupt 是在 GPIO basic 的 state/read/write 基础上，加触发条件、状态位、W1C 和 qemu_set_irq。
```

### 10.6 分模块结论速查：PWM / WDT / SPI

| 模块 | 你应该直接掌握的答案 |
| --- | --- |
| PWM | 比 GPIO basic 多 channel、counter、enable、polarity、done flag，通常要和虚拟时间或计数推进关联。 |
| WDT | 关键寄存器是 `WDT_CTRL`、`WDT_LOAD`、`WDT_VAL`、`WDT_KEY`、`WDT_SR`；`WDT_KEY` 负责 feed/lock，`WDT_SR` 有 TIMEOUT W1C。 |
| WDT IRQ | `test-wdt-timeout.c` 定义 `WDT_PLIC_IRQ = 4`。 |
| SPI | SPI JEDEC 不是简单读写寄存器，它要求 controller 处理 TX/RX、状态位，并通过协议状态机返回 flash JEDEC ID。 |
| SPI IRQ | `SPI_PLIC_IRQ = 5`，主要在 flash interrupt 和 overrun 相关测试里出现。 |

一句话版本：

```text
PWM/WDT/SPI 都复用 Day2 建立的 MMIO + state + board map 闭环，只是分别叠加 timer、IRQ、协议状态机。
```

---

## 11. Day 1 之后不要做的事

这一节不是为了压住好奇心，而是告诉你“为什么不准做”：每条禁令背后都在保护 Day 1 的主线，避免你在还没建立心智模型前就掉进细节沼泽。

- 不要改测试。因为测试是训练营给你的“可执行规格”，改了测试就等于把尺子锯短了，后面你会不知道自己到底实现对了没有。
- 不要一次性实现 GPIO/PWM/WDT/SPI。因为这四类外设复杂度不同，混在一起会让失败原因变得不可定位：你分不清是 MMIO 没挂上，还是 IRQ/timer/协议状态机写错了。
- 不要先读完整 QEMU 源码。因为 Day 1 只需要建立 `machine -> address space -> MemoryRegion -> QTest` 这条主线，横向扩散会让你很累但收获很散。
- 不要追求完整硬件规格。因为训练营早期考的是“最小可验证语义”，不是硬件手册级别的完美仿真。
- 不要把 SPI Flash 和 `0x20000000` pflash 混为一谈。因为前者是通过 SPI controller 访问的协议设备，后者是板上已有的并行 flash 地址区，它们在 QEMU 地址图里的角色不同。
- 不要在没跑基线前开始写代码。因为你需要先知道环境、构建和测试入口是否正常，否则后面失败时无法判断是你代码错了还是工程链路本来就断。
- 不要继续横向补很多视频笔记，先把 Day 1 产出填出来。因为 Day 1 的目标不是“收集更多资料”，而是把第一张可用的心智地图画出来。

🧠 换句话说：

```text
这些“不准做”不是为了压住你的好奇心，
而是为了保护今天唯一的主线：

QTest 地址访问
-> QEMU 地址空间
-> MemoryRegion
-> 外设 read/write
-> 设备 state
```

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

### 🎙️ Day 1 最终口述验收

真正能开始 Day 2 的标准不是“表格都填了”，而是你能不看答案讲清楚下面这段：

```text
当测试执行 qtest_writel(qts, 0x10012004, 1) 时：
1. 这个地址是 guest 物理地址。
2. QEMU 在系统地址空间里查找它。
3. 因为 GPIO 被映射到 0x10012000，所以这次访问命中 GPIO 的 MemoryRegion。
4. QEMU 把 0x10012004 换算成 GPIO 内部 offset 0x04。
5. offset 0x04 对应 GPIO_OUT。
6. GPIO 的 write 回调应该把 value 写进 s->out。
7. 后续 read GPIO_IN 时，basic 阶段可以返回 s->out & s->dir。
```

如果你能把这 7 句讲顺，Day 1 就不是机械执行了。你已经知道这台机器的第一组齿轮是怎么咬合的。
