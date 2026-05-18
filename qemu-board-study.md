# QEMU 主板建模流程学习资料

## 来源

- 讲义：<https://qemu.gevico.online/tutorial/2026/ch2/qemu-machine/>
- 视频：<https://www.bilibili.com/video/BV1v4X4BPEP3/>
- 视频标题校验：`QEMU 训练营 | 专业阶段 QEMU 主板建模流程 [SOC 建模]`
- 作者 / UP：`绝对是泽文啦`
- 讲义作者：`@zevorn`
- 本地处理：已抓取讲义正文，已下载视频音频并用本地 `faster-whisper` 转写，视频时长约 `15:00`

本章没有放 PPT 截图。原因和上一章一样：讲义已经给出核心代码路径，视频主要是在讲“主板模型的分层和总装线”。普通标题页、过渡页放进来只会让笔记变胖，不利于你复习。

---

## 阅读说明

- `🎥 视频/作者`：讲者在视频里强调的理解方式。
- `📘 讲义`：网页文档里可当手册反复查的稳定内容。
- `🧠 我的理解`：我把概念翻译成你写工程代码时的判断方式。
- `🧩 补充知识`：讲义和视频没完全展开，但做 SoC 必须知道的背景。
- `🔗 和 SoC 训练营的关系`：落到 G233 的 machine、地址、中断和 qtest。
- `✅ 你现在就做`：看完这一节后立刻动手的小任务。
- `⚠️ 容易踩坑`：QEMU 主板建模里最容易混的地方。

---

## 一句话结论

QEMU 的主板模型就是整机“总装线”：它不是 CPU，也不是某一个外设，而是负责选择 CPU、分配内存、建立地址空间、创建中断控制器、挂载外设、连接 IRQ、准备固件入口和 FDT/ACPI 的那一层。

🧠 我的理解：上一章“外设建模流程”教你写一个设备的内部行为；这一章“主板建模流程”教你把这些设备真正装进一台可启动的机器里。设备没挂到 machine 上，qtest 和 guest 都看不见。

---

## 本章结构

```text
用户命令
  -> -machine g233 / virt
  -> select_machine() 选择 MachineClass
  -> object_new_with_class() 创建 MachineState
  -> machine_run_board_init()
  -> mc->init(machine)
    -> 创建 CPU / hart / socket
    -> 注册 RAM / ROM / MMIO 地址空间
    -> 创建 PLIC / CLINT / UART / RTC / virtio / 自定义外设
    -> memory_region_add_subregion()
    -> sysbus_create_simple() / qdev_new() / sysbus_realize()
    -> sysbus_connect_irq() / qdev_get_gpio_in()
    -> 生成 FDT / 固件入口
```

🎥 视频/作者：视频开头强调，不要把 QEMU system emulation 只理解成 CPU。真正能启动的虚拟机还需要内存布局、地址空间、中断控制器、串口、timer、总线连接、固件入口和设备树/ACPI。

📘 讲义：讲义明确说，`-machine` 决定“整机框架”，`-cpu` 决定“CPU 细节”。主板模型是 `TYPE_MACHINE` 的子类，通过 `MachineClass` 挂载自己的 `init` 回调。

🧠 我的理解：主板建模最重要的不是背函数名，而是形成一个判断：我现在看的代码是在写 CPU 行为、设备行为，还是整机拓扑？三层分开，QEMU 源码会清爽很多。

---

## 1. 主板模型在 QEMU 里是什么

🎥 视频/作者：讲者强调，CPU 只是整台机器的一部分。Machine 解决的是“整机级别的拓扑和组装问题”，Device 解决的是“某个外设自身行为”，CPU model 解决的是“处理器本身”。

📘 讲义：QEMU 里的主板对应 machine model。它把 CPU 拓扑、内存布局、总线、外设、中断路由、固件入口统一编排起来。

🧠 我的理解：可以把 machine 想成硬件工程里的“板级原理图 + 装配脚本”。它不应该把 GPIO 的每个寄存器语义写在自己里面，但它必须知道 GPIO 放在哪个地址、接到哪个 IRQ、是否出现在设备树里。

🧩 补充知识：主板模型常见职责可以分成 6 类。

| 职责 | 典型内容 |
| --- | --- |
| CPU 拓扑 | socket、hart、cpu type、smp |
| 内存地图 | DRAM、ROM、flash、MMIO base/size |
| 中断系统 | PLIC、CLINT/ACLINT、AIA、IRQ source 编号 |
| 外设挂载 | UART、RTC、virtio、GPIO、PWM、WDT、SPI |
| 固件与启动 | reset vector、OpenSBI、kernel/initrd/FDT 地址 |
| 硬件描述 | FDT/ACPI，告诉 guest 设备在哪里 |

🔗 和 SoC 训练营的关系：`qtest_init("-machine g233 -m 2G")` 这句话不是“启动一个普通 QEMU”，而是要求 QEMU 找到 `g233` 这个 machine type，创建 G233 board，并把 DRAM、PLIC、CLINT、UART 以及后续 GPIO/PWM/WDT/SPI 都装好。

✅ 你现在就做：把这句话背成主板建模的第一性原理。

```text
device 决定“外设怎么响应读写”；
machine 决定“这个外设是否存在、在哪里、接到谁”。
```

⚠️ 容易踩坑：不要把“设备寄存器逻辑”塞进 machine。machine 里应该做装配，不应该变成 GPIO/PWM/WDT/SPI 的大杂烩。

---

## 2. QOM、TypeInfo 和 MachineClass 怎么理解

🎥 视频/作者：视频里把 machine 分成两层：类型层面决定“它是什么”，行为层面决定“它被创建以后怎么初始化整块板子”。最关键的入口是 `MachineClass->init`。

📘 讲义：RISC-V `virt` 的主板通过 `TypeInfo` 注册，父类是 `TYPE_MACHINE`，`class_init` 设置 `MachineClass`，`instance_init` 初始化实例字段，`instance_size` 指向自己的状态结构体大小。

🧠 我的理解：外设有 `DeviceClass`，主板有 `MachineClass`。你可以把 `MachineClass` 当成“这类板子的说明书和默认参数”，把 `MachineState` 当成“这次运行真的创建出来的这块板子”。

```c
static void virt_machine_class_init(ObjectClass *oc, const void *data)
{
    MachineClass *mc = MACHINE_CLASS(oc);

    mc->desc = "RISC-V VirtIO board";
    mc->init = virt_machine_init;
    mc->max_cpus = VIRT_CPUS_MAX;
    mc->default_cpu_type = TYPE_RISCV_CPU_BASE;
    mc->default_ram_id = "riscv_virt_board.ram";
}
```

🧩 补充知识：`class_init` 和 `instance_init` 不要混。

| 阶段 | 作用 |
| --- | --- |
| `class_init` | 设置这一类 machine 的能力、默认 CPU、最大 CPU 数、`mc->init` |
| `instance_init` | 初始化这次 machine 实例里的字段，比如 flash、属性默认值 |
| `mc->init` | 真正把整台板子装起来，创建 CPU、内存、中断、外设 |

🔗 和 SoC 训练营的关系：G233 在头文件里有 `TYPE_RISCV_G233_MACHINE MACHINE_TYPE_NAME("g233")`。这就是 `-machine g233` 能被 QEMU 识别的根。后续你找入口时，不要只搜 `g233_init`，也要看 `TypeInfo` 和 `MachineClass` 怎么连到 init。

✅ 你现在就做：在 G233 代码里搜这几个符号。

```bash
rg -n "TYPE_RISCV_G233_MACHINE|MachineClass|class_init|instance_init|mc->init" hw/riscv/g233.c include/hw/riscv/g233.h
```

⚠️ 容易踩坑：看到 `instance_init` 里创建了 flash，不代表整块板已经初始化完成。真正 CPU/内存/中断/外设的大装配通常在 `mc->init` 指向的函数里。

---

## 3. `-machine` 怎么一路走到主板 init

🎥 视频/作者：视频强调，很多同学知道有 `xxx_machine_init`，但不知道 QEMU 是什么时候、通过什么路径走到这里的。本章就是把这条路径讲清楚。

📘 讲义：创建流程大致是：解析 `-machine`，`select_machine()` 找到机型类，`qemu_create_machine()` 实例化 `MachineState`，最后 `machine_run_board_init()` 调用 `machine_class->init(machine)`。

🧠 我的理解：这条链路就是从“命令行参数”到“板子真的被装起来”的桥。

```text
-machine g233
  -> select_machine()
  -> find_machine("g233")
  -> object_new_with_class()
  -> current_machine = MACHINE(...)
  -> machine_run_board_init()
  -> machine_class->init(machine)
```

🧩 补充知识：`info qom-tree` 能看到 machine 对象挂在 QOM tree 里。也就是说 machine 不是一个抽象配置名，而是运行时真的存在的 QOM 对象。

🔗 和 SoC 训练营的关系：如果 `test-board-g233` 一启动就挂，可能不是 GPIO/WDT/SPI 的问题，而是 `-machine g233` 没被注册、machine init 里某个基础对象创建失败、或者 board init 阶段就 assert 了。

✅ 你现在就做：把 `qtest_init("-machine g233 -m 2G")` 展开理解。

```text
启动一个 QEMU 子进程
  -> 选择 g233 machine
  -> 分配 2G RAM
  -> 执行 G233 board init
  -> qtest 开始通过地址读写这台虚拟机器
```

⚠️ 容易踩坑：`-cpu` 不会帮你创建板上外设。`-cpu` 只影响 CPU 细节；外设地址、中断控制器、DRAM 布局这些都在 machine。

---

## 4. 主板 init 是一条“总装线”

🎥 视频/作者：视频里讲到，machine init 不是写描述信息就完了，它是真的在创建对象、设置属性、realize 对象，让它们进入能工作的状态。

📘 讲义：以 RISC-V `virt_machine_init()` 为例，init 会按 socket 创建 CPU 集群，配置 `cpu-type`、`hartid-base`、`num-harts`，再 `sysbus_realize()`。完整实现还会分配 RAM/ROM/MMIO、初始化中断控制器、挂载串口/RTC/PCIe/virtio-mmio、生成 FDT 或 ACPI。

🧠 我的理解：machine init 可以按“从底座到外设”的顺序读。

```text
1. 拿到 MachineState 和 machine 配置
2. 根据 memmap 准备地址布局
3. 创建 CPU/hart array
4. 创建 interrupt controller
5. 把 RAM 挂进 system_memory
6. 创建 ROM/flash/fw_cfg/test device
7. 创建 UART/RTC/virtio/platform bus
8. 创建自定义 SoC 外设
9. 连接 IRQ 到 PLIC/ACLINT/AIA
10. 生成 FDT/ACPI，准备固件入口
```

🧩 补充知识：`realize` 是 QEMU 设备从“对象”进入“可用设备”的关键阶段。machine 里经常出现 `qdev_new()`、`sysbus_realize_and_unref()`、`sysbus_create_simple()`，它们背后都在完成“创建设备实例 + 设置属性 + realize + 挂线”的不同组合。

🔗 和 SoC 训练营的关系：G233 现有代码里能看到基础 memmap 和装配动作，例如 DRAM base `0x80000000`、PLIC base `0x0c000000`、CLINT base `0x02000000`、UART base `0x10000000`。后面新增 GPIO/PWM/WDT/SPI，本质就是在这条总装线上继续加设备。

✅ 你现在就做：读 `hw/riscv/g233.c` 时，不要从第一行硬啃。先找这三类东西。

```text
memmap: 地址布局
MachineClass init: 入口函数
memory_region_add_subregion / sysbus_create_simple / qdev_new: 装配动作
```

⚠️ 容易踩坑：设备对象创建了但没 realize，或者 realize 了但没映射 MMIO，或者映射了 MMIO 但 IRQ 没接。三种都会表现成“测试看不到正确行为”，但根因完全不同。

---

## 5. 内存地图和地址空间怎么读

🎥 视频/作者：视频强调，能启动的虚拟机必须有内存布局和地址空间。主板模型负责把 RAM、ROM、MMIO 这些区域放到同一个系统地址空间里。

📘 讲义：主板职责包括定义内存地图、外设挂载、IRQ 连接、固件/设备树生成等。`virt` 会分配内存区域，包括 RAM、ROM、MMIO。

🧠 我的理解：memmap 是 machine 的骨架图。外设模型只关心自己的 offset，machine 决定这个外设最终出现在 guest 物理地址的哪里。

```text
guest physical address
  -> system_memory address space
  -> 命中某个 MemoryRegion
  -> RAM: 普通内存读写
  -> MMIO: 调用设备 MemoryRegionOps
```

🧩 补充知识：G233 当前基础布局可以先记住这些。

| 区域 | 地址 | 测试/用途 |
| --- | --- | --- |
| CLINT | `0x02000000` | `test-board-g233` 读 `mtime` |
| PLIC | `0x0c000000` | `test-board-g233` 写 priority |
| UART0 | `0x10000000` | 串口输出/默认 console |
| DRAM | `0x80000000` | `test-board-g233` 写 `0xDEADBEEF` |
| GPIO | `0x10012000` | `test-gpio-basic/int` |
| WDT | `0x10010000` | `test-wdt-timeout` |
| PWM | `0x10015000` | `test-pwm-basic` |
| SPI | `0x10018000` | `test-spi-jedec/cs/overrun` |

🔗 和 SoC 训练营的关系：外设测试里写死的 base 地址，其实是在给你“验收 machine memmap”。如果 GPIO 设备内部逻辑写对了，但 machine 把它映射到 `0x10013000`，`test-gpio-basic` 仍然会失败。

✅ 你现在就做：每加一个 SoC 外设，先更新一张 machine 接线表。

| 设备 | base | size | IRQ | 下游连接 | FDT |
| --- | --- | --- | --- | --- | --- |
| GPIO | `0x10012000` | 待定 | PLIC `2` | pins/内部回读 | 需要 |
| WDT | `0x10010000` | 待定 | PLIC `4` | timer | 需要 |
| PWM | `0x10015000` | 待定 | 可选/按题目 | timer | 需要 |
| SPI | `0x10018000` | 待定 | PLIC `5` | flash CS0/CS1 | 需要 |

⚠️ 容易踩坑：memmap 不是注释。测试访问的是物理地址，QEMU 路由也看物理地址；base/size 错一位，设备 read/write 根本不会被调用。

---

## 6. 中断、外设和 FDT 怎么接起来

🎥 视频/作者：视频里提到，一台板子还要有中断控制器、串口、timer、总线连接，并且准备设备树或 ACPI，让 guest 把它当成一块板来识别。

📘 讲义：主板 init 里会初始化中断控制器，挂载串口、RTC、PCIe、virtio-mmio，并生成 FDT 或 ACPI 表。

🧠 我的理解：外设的 IRQ 线不是自己飘在空气里的。设备内部调用 `qemu_set_irq(s->irq, level)`，但这条线必须在 machine 里接到 PLIC 的某个输入上。否则设备“喊了”，中断控制器“听不见”。

```text
设备内部事件
  -> qemu_set_irq(device_irq, 1)
  -> machine 里 sysbus_connect_irq(...)
  -> qdev_get_gpio_in(plic, irq_number)
  -> PLIC pending bit 变化
  -> guest / qtest 观察到中断
```

🧩 补充知识：FDT/设备树主要服务 guest OS 和驱动。qtest 直接用固定地址读写时，不一定依赖 FDT；但如果要跑 Linux driver，设备树就很关键。没有 FDT 节点，Linux 可能根本不会 probe 你的设备。

🔗 和 SoC 训练营的关系：当前 qtest 直接访问地址，所以第一阶段重点是“machine 里确实映射和连线”。但你学习 Linux 驱动时，要把 FDT 加进思考：compatible、reg、interrupts、clocks 这些都会影响驱动能否绑定。

✅ 你现在就做：看到一个中断测试时，按两段查。

```text
设备内部：状态位是否设置，update_irq 是否调用 qemu_set_irq
machine 外部：设备 IRQ 是否接到 PLIC 正确 source number
```

⚠️ 容易踩坑：只看设备内部 `qemu_set_irq`，忘了 machine 外部连线。`test_gpio_plic`、`test_wdt_interrupt`、`test_spi_overrun` 都会抓这种问题。

---

## 7. 对 SoC 测题的直接帮助

这一章对 SoC 模块的价值，是帮你把“单个设备实现”升级成“整块 G233 板子可被测试和 guest 发现”。

### `test-board-g233`

🎥 视频/作者：视频强调 machine 决定整机骨架，不是 CPU 本身。

📘 讲义：主板模型负责 CPU、内存、中断、外设和固件入口的统一编排。

🧠 我的理解：这个测试不是在考某个新外设，而是在考 G233 machine 的底座有没有装好。

🔗 和 SoC 训练营的关系：`test-board-g233` 检查 `-machine g233` 能启动、DRAM `0x80000000` 可读写、PLIC `0x0c000000` 存在、CLINT `0x02000000` 的 `mtime` 可读。

✅ 你现在就做：先保证 board 基础测试过，再做 GPIO/PWM/WDT/SPI。不然你会在外设 bug 和 machine 基础 bug 之间来回迷路。

⚠️ 容易踩坑：`test-board-g233` 过了，只说明底座在；不代表后续自定义外设已经挂好。

### `test-gpio-basic`

🎥 视频/作者：Machine init 是整板总装入口。GPIO 设备写好以后，必须在这里被创建和映射。

📘 讲义：主板 init 里要挂载外设，并把 MMIO 区域放进地址空间。

🧠 我的理解：`test-gpio-basic` 从 machine 视角看，核心是 `0x10012000` 这段地址必须命中 GPIO 的 `MemoryRegionOps`。

🔗 和 SoC 训练营的关系：如果 `GPIO_DIR/OUT/IN` 的设备逻辑都写对了，但 machine 没有把 GPIO 映射到 `0x10012000`，qtest 仍然读不到正确值。

✅ 你现在就做：查 machine 里是否有类似这条路径。

```text
qdev_new(TYPE_G233_GPIO)
  -> sysbus_realize_and_unref()
  -> memory_region_add_subregion(system_memory, 0x10012000, gpio_mmio)
```

⚠️ 容易踩坑：base 地址和测试文件不一致。先看测试定义，再写 machine memmap。

### `test-gpio-int`

🎥 视频/作者：主板必须把中断控制器和外设连起来。

📘 讲义：主板职责包括中断路由。

🧠 我的理解：GPIO interrupt 从 board 角度看，就是“GPIO 这个 SysBusDevice 的 IRQ 输出接到 PLIC source 2”。

🔗 和 SoC 训练营的关系：测试里明确 `GPIO_PLIC_IRQ = 2`，会读 PLIC pending、claim、complete。设备内部只负责拉线，PLIC pending 能不能变，要看 machine 接线。

✅ 你现在就做：把 GPIO 的 IRQ 号写进接线表，不要散落在代码魔法数字里。

⚠️ 容易踩坑：IRQ source number off-by-one。PLIC source 0 通常不用，测试用的是 source 2，就别接到 1 或 3。

### `test-pwm-basic`

🎥 视频/作者：主板 init 会把平台设备真正创建出来，让它们进入工作状态。

📘 讲义：machine 会挂载基础外设，完整平台可以继续挂载更多 MMIO 设备。

🧠 我的理解：PWM 从 board 角度看，重点是 `0x10015000` 映射和虚拟时钟/timer 能正常推进。qtest 的 `qtest_clock_step` 会驱动 QEMU 虚拟时间。

🔗 和 SoC 训练营的关系：`test-pwm-basic` 不只读写配置，还检查 clock step 后 counter 和 DONE。machine 至少要保证 PWM 设备已 realize、timer 属于 QEMU 虚拟时钟体系，不是一个死对象。

✅ 你现在就做：实现 PWM 前，先确认设备 state 里的 timer 在 realize/init 时创建，machine 里负责实例化和映射。

⚠️ 容易踩坑：PWM MMIO 能读写，但 `qtest_clock_step` 后状态不动。通常说明 timer 没接好或 enable 后没 rearm。

### `test-wdt-timeout`

🎥 视频/作者：主板模型把 timer、中断控制器、外设组织成一个可运行系统。

📘 讲义：主板负责中断控制器和外设挂载，设备内部负责自己的行为。

🧠 我的理解：WDT 从 board 角度看要满足三件事：base `0x10010000`，timer 能随 qtest clock 走，IRQ 接到 PLIC source 4。

🔗 和 SoC 训练营的关系：`test_wdt_interrupt` 会检查 timeout 后 PLIC source `4` pending。如果 `WDT_SR.TIMEOUT` 已经置位但 PLIC 没 pending，大概率是 machine IRQ 连接错。

✅ 你现在就做：把 WDT 的设备内部测试和 board 接线测试分开排查。

⚠️ 容易踩坑：WDT timeout flag 对了，但 interrupt 不对。这个时候不要一直改 WDT 状态机，先看 `sysbus_connect_irq`。

### `test-spi-jedec / test-spi-cs / test-spi-overrun`

🎥 视频/作者：Machine 不是描述板子，而是在运行时真的把板子组起来。

📘 讲义：主板会挂载外设和总线。SPI 这种设备还可能有下游 flash 芯片，需要 machine 一并装配。

🧠 我的理解：SPI 从 board 角度比 GPIO/WDT 更复杂，因为它不只是一个 MMIO controller。它还要有 CS0/CS1 下游 flash，而且两片 flash 状态要隔离。

🔗 和 SoC 训练营的关系：`test-spi-jedec` 要读 CS0 的 JEDEC ID，`test-spi-cs` 要验证 CS0/CS1 两片 flash 的 ID、容量和数据隔离，`test-spi-overrun` 要把 SPI IRQ 接到 PLIC source `5`。

✅ 你现在就做：SPI 的 machine 接线表至少写出。

```text
SPI controller: base 0x10018000, IRQ 5
CS0: W25X16, JEDEC 0xEF3015, 2MB
CS1: W25X32, JEDEC 0xEF3016, 4MB
```

⚠️ 容易踩坑：只创建 SPI controller，不创建/绑定 flash。这样 `SPI_CR1/SR/DR` 可能有反应，但 JEDEC 和 CS 测试一定会出问题。

---

## 8. 最小练习

✅ 你现在就做：练习一，追一次 `-machine g233`。

```text
入口：qtest_init("-machine g233 -m 2G")
  -> select_machine()
  -> qemu_create_machine()
  -> machine_run_board_init()
  -> G233 的 mc->init
```

✅ 你现在就做：练习二，读 G233 memmap。

```bash
rg -n "MemMapEntry|VIRT_DRAM|VIRT_PLIC|VIRT_CLINT|VIRT_UART0" hw/riscv/g233.c include/hw/riscv/g233.h
```

✅ 你现在就做：练习三，按 qtest 反推 board 责任。

| 测试 | board 必须保证 |
| --- | --- |
| `test-board-g233` | `g233` 注册、DRAM、PLIC、CLINT 存在 |
| `test-gpio-basic` | GPIO 映射到 `0x10012000` |
| `test-gpio-int` | GPIO IRQ 接到 PLIC source `2` |
| `test-pwm-basic` | PWM 映射到 `0x10015000`，timer 可推进 |
| `test-wdt-timeout` | WDT 映射到 `0x10010000`，IRQ source `4` |
| `test-spi-jedec/cs/overrun` | SPI 映射到 `0x10018000`，flash CS0/CS1，IRQ source `5` |

✅ 你现在就做：练习四，每新增一个外设前，先写这 5 行。

```text
设备类型名：
MMIO base/size：
IRQ source：
machine 创建位置：
qtest 验证路径：
```

🧠 我的理解：这份练习的目的不是让你背 G233，而是训练一种工程习惯：先把 board 接线关系画清楚，再写设备内部逻辑。这样 debug 时你能判断问题是在 device 里，还是在 machine 装配层。

⚠️ 容易踩坑：不要等所有外设写完才跑测试。每接一个外设，就用最小 qtest 验证一次“能被看见”。

---

## 9. 下一章怎么接

🎥 视频/作者：这一章把主板装配线讲完，但 timer/clock 相关内容只是点到。视频顺序里下一节就是“时钟系统”，正好补 PWM/WDT 会用到的虚拟时间和 clock。

📘 讲义：主板模型负责把外设组织起来；clock/timer 则会影响这些外设如何随虚拟时间变化。

🧠 我的理解：建议下一章优先看“时钟系统”，因为 SoC 模块里 PWM/WDT/SPI busy/timeout 都绕不开虚拟时间。你现在已经有了两块拼图：

```text
外设建模：设备内部怎么响应寄存器
主板建模：设备怎么被挂到整机
下一章时钟：设备状态怎么随虚拟时间变化
```

🔗 和 SoC 训练营的关系：做 GPIO 时，本章已经够用了；做 PWM/WDT 时，下一章 clock/timer 会直接影响测试是否过；做 SPI flash 时，也会遇到 busy、delay、qtest_clock_step 这类问题。

✅ 你现在就做：学下一章前，先确认自己能回答这三个问题。

```text
1. -machine g233 怎么找到 G233 board init？
2. GPIO/WDT/SPI 的 IRQ 是在哪里接到 PLIC 的？
3. qtest 写 0x10012000 为什么会进入 GPIO 设备，而不是普通内存？
```

⚠️ 容易踩坑：如果这三个问题还模糊，先别急着写复杂 SPI。先用 `test-board-g233` 和一个最小 GPIO 设备把 machine 装配链路跑通。

---

## 我的点评和延伸

🧠 我的理解：这章是 SoC 模块真正的“地图课”。外设建模告诉你每个零件怎么工作，主板建模告诉你零件怎么装到板子上。你以后 debug 时可以先问：这是“零件坏了”，还是“线没接上”？

🧩 补充知识：我建议你以后每个 G233 外设都维护一张接线表。它看起来很朴素，但能极大减少上下文爆炸。

```text
Device: G233 GPIO
Type: SysBusDevice
Base: 0x10012000
IRQ: PLIC source 2
State: dir/out/in/ie/is/trig/pol
Tests: test-gpio-basic, test-gpio-int
Machine hook: hw/riscv/g233.c
```

🔗 和 SoC 训练营的关系：等你真正开始做大项目，我会按这种粒度拆 commit：一个 commit 只完成一个设备或一条接线，例如“注册 GPIO 设备类型”“把 GPIO 映射到 G233 machine”“接 GPIO IRQ 到 PLIC”“补 GPIO qtest 通过”。这样你按 commit 学，脑子不会炸。

✅ 你现在就做：读完这章后，回到上一章 `qemu-hw-study.md`，把两章合起来看一遍。你的主线应该变成：

```text
设备内部：state + MemoryRegionOps + IRQ/timer update
主板外部：memmap + realize + MMIO map + IRQ route + FDT
测试入口：qtest_init + qtest_readl/writel + qtest_clock_step
```
