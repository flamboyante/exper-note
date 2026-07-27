# K230 SSI Patch 4：Standard SPI NOR 学习工作簿

更新时间：2026-07-17

## 目标

Patch 4 在 Patch 3 已完成的 Standard SPI PIO 数据路径上，为 K230 `spi0`
连接一个真实的 QEMU SPI NOR 从设备，并用 Standard `spi-mem` 事务验证：

```text
K230 Guest 驱动
  -> K230 DWC SSI 寄存器/FIFO/TMOD
  -> QEMU SSIBus
  -> QEMU w25q256
  -> 可选的 -drive if=mtd raw backend
```

本 Patch 的核心不是重新实现 Flash 协议，而是完成板级连接，并证明 Patch 3
产生的 Standard SPI 帧序列能被现有 `m25p80` 模型正确解释。

## Patch 4 回归补充：增强 reset 失败后的 Standard 回退

真实 K230 U-Boot 在 `sf probe` 前会尝试 Octal DTR software reset。该尝试失败
不是致命错误，驱动随后回退到 Standard `EEPROM_READ` 读取 6 字节 JEDEC ID。

新增回归：

```text
/k230-dw-ssi/flash/enhanced-reset-fallback-read-id
```

它覆盖：

```text
U-Boot 在 SSIENR=1 时通过 TXFTLR 回读探测 256 项 FIFO
  -> 不支持的增强 reset 留下未发送 TX 项
  -> SER=0、SSIENR=0 撤销 CS 并清理传输状态
  -> Standard EEPROM_READ，CTRLR1.NDF=5
  -> RX FIFO 获得 6 字节，前三字节为 ef 40 19
```

排障发现原模型错误地把 `TXFTLR/RXFTLR` 放入 disabled-only 配置寄存器列表。
但 TRM 将二者定义为运行期 FIFO/中断阈值，没有 `SSIENR=0` 才可写的限制；K230
U-Boot 的 `spi_hw_init()` 也明确先写 `SSIENR=1`，再逐值写读 `TXFTLR` 探测深度。

被错误拒绝时，U-Boot 得到 `fifo_len=0`，于是：

```text
dw_writer():
    tx_room = fifo_len - TXFLR = 0
    count = 0
```

因此 Read-ID opcode 从未写入 DR，外部现象就是“SER 已选中 CS0，但 TXFLR/RXFLR
均为 0，`ssi_transfer()` 没有被调用”。最小修复首先允许 SSI enabled 时更新
`TXFTLR/RXFTLR`，随后把 disabled-only 写锁收缩到有充分依据的最小集合：

```text
CTRLR0 / CTRLR1 / MWCR / BAUDR / SPI_CTRLR0
```

这些寄存器直接定义基本帧格式、传输模式、帧数、串行时钟或增强 SPI 事务 phase。
`DMACR`、AXI burst/address、`SPIDR/SPIAR`、RX/DDR 时序和 XIP opcode 等寄存器，
TRM 只给出了推荐的“先配置、后 enable”编程流程，没有逐项说明 SSIENR=1 时写入
必然无效。因此模型允许它们在 enabled 状态写入，由 guest 驱动保证不在活动事务中
修改；等 IDMA/XIP 真正实现时，再根据各自的 active 状态和新增回归收紧行为。

这里必须区分：

```text
SSIENR=1            控制器逻辑已启用
phase != IDLE       当前确实存在由模型推进的事务
IDMA/XIP active     对应专用引擎正在使用配置
```

三者不能互相替代。仅凭 `SSIENR=1` 扩大写锁，可能再次制造 FIFO 深度探测同类问题。

对应回归为：

```text
/k230-dw-ssi/reg/enabled-write-contract
```

它在 `SSIENR=1、SER=0` 的 enabled-idle 状态下同时验证：最小集合写入保持原值，
DMA、AXI、RX/DDR 时序、XIP opcode 和 FIFO 阈值写入则按各自 writable mask 生效。

真实预构建 U-Boot 的 Octal reset 会因为当前 DT/设备线宽在 `supports_op()` 层返回
`-ENOTSUPP`，通常还没有把 `0x6666` 写进 DR。因此实际现场并不存在增强 TX 项污染；
回归测试仍故意注入一个未发送的增强 TX 项，以覆盖比真实现场更严格的 abort 清理
边界。两层结论不能混淆：reset 失败是预期回退条件，真正阻止 Read-ID 的 bug 是
FIFO 深度探测被写锁破坏。

本阶段必须完成：

- 提供显式 `-machine k230,spi-flash=w25q256`，按需把该器件挂到 SDK
  `spi0 @ 0x91584000` 的 CS0；默认不创建 Flash。
- 支持 `-drive if=mtd` 作为 32 MiB Flash backend。
- 没有 backend 时仍创建一个内容全为 `0xff` 的空 Flash。
- 使用 TO 完成 WREN、Page Program 和 Sector Erase。
- 使用 EEPROM_READ 完成 JEDEC ID、状态寄存器和普通读取。
- 证明 3-byte read、4-byte read、program、erase 和 CS 事务重启。

本阶段不实现：

- Dual/Quad/Octal、DTR、RXDS。
- K230 内部 IDMA。
- IRQ 或 PLIC。
- `0xC0000000` XIP window。
- BootROM 冷启动。
- 在 K230 SSI 控制器中硬编码任何 Flash opcode。

建议提交标题：

```text
hw/riscv/k230: Attach SPI NOR flash to K230 spi0
```

## 学习脚手架状态

当前工作树已经完成不需要猜硬件语义的外围脚手架：

- K230 Kconfig 选择 `SSI_M25P80`；
- `K230MachineState` 保存可选的 `spi_flash_model` 字符串；
- 注册 `-machine k230,spi-flash=<model>` 属性并处理字符串生命周期；
- machine realize 后固定把可选 Flash 交给 SDK `spi0` 对应的 `dw_ssi[2]`；
- Flash qtest 使用显式 `spi-flash=w25q256` 和临时 MTD 镜像；
- 七项 Standard Flash 测试已改为调用 TO/EEPROM_READ 学习 helper，旧 TR
  helper 不再承担 Patch 4 的 Flash 读写。

本工作簿最初用以下三个 TODO 标记学习实现位置；当前代码已经全部完成：

```text
P4-3         machine 层创建 M25P80、绑定 backend、挂 SSIBus、连接 CS
P4-6A        用 TX_ONLY 组织 Standard Data-OUT 事务
P4-6B        用 EEPROM_READ/NDF 组织 Standard Data-IN 事务
```

这三个位置是 Patch 4 的核心。其余七个测试负责给出 opcode、地址、数据和预期值，
不需要在控制器中实现第二套 Flash 状态机。

学习脚手架曾经使用以下 RED 顺序定位问题：

```text
P4-3 未完成：显式 MTD 没有被 M25P80 消费，QEMU 在 machine 启动时拒绝配置。
P4-3 完成后：Flash 已接线，但 Data-IN helper 暂时填 0，JEDEC 断言立即失败。
P4-6B 完成后：JEDEC/read 可以通过，再填写 P4-6A 验证 program/erase。
```

这样不会因为缺少 RX 数据而随机超时。完成一个 TODO 后只运行对应的最小测试。

## 1. Patch 4 的功能、流程与证据层次

Patch 4 必须忠于 TRM 和 SDK，但二者证明的对象不同，不能混写。

### Patch 4 功能与调用流程总览

Patch 4 可以拆成两条相互配合、但职责不同的链路：

```text
配置链路：
-machine k230,spi-flash=w25q256
  -> QOM Machine 属性 "spi-flash"
  -> k230_machine_set_spi_flash()
  -> K230MachineState.spi_flash_model
  -> k230_machine_init()

设备链路：
spi_flash_model
  -> qdev_new("w25q256")
  -> 绑定可选 MTD BlockBackend
  -> realize 到 K230 dw_ssi[2].spi
  -> 连接 dw_ssi[2]."cs"[0] 到 Flash 的 SSI_GPIO_CS
  -> Guest 通过 SSI 寄存器访问真实 m25p80 模型
```

#### Machine、SoC 和 Flash 的职责

```text
K230MachineState
  保存板级选择：spi_flash_model
  决定是否安装 Flash、安装哪种 Flash、使用哪个 backend

K230SoCState
  保存 SoC 集成硬件：CPU、SRAM、PLIC、dw_ssi[0..2]
  不保存外部 Flash 型号，也不实现 Flash 协议

K230DwSsiState
  提供 MMIO、TX/RX FIFO、TMOD、SSIBus 和 CS 输出
  不解析 0x9f、0x03、0x02 等 Flash opcode

m25p80/w25q256
  处理 JEDEC ID、WEL、读、编程、擦除、SFDP 和 backend 同步
```

因此 `spi_flash_model` 放在 Machine 层是有意的：它描述的是当前虚拟板是否
焊接了某种外部器件，而不是 K230 SoC 内部固有的硬件资源。

#### `object_class_property_add_str()` 如何接入命令行

Machine class 初始化时注册：

```c
object_class_property_add_str(oc, "spi-flash",
                              k230_machine_get_spi_flash,
                              k230_machine_set_spi_flash);
```

这里的 `oc` 是 `TYPE_RISCV_K230_MACHINE` 的 `ObjectClass *`，表示 K230 Machine
类型本身，不是某一个运行中的 Machine 实例。注册后，所有 K230 Machine 实例都
拥有名为 `spi-flash` 的字符串属性。

```text
oc（K230 Machine 类）
  `- "spi-flash"
       |- getter: k230_machine_get_spi_flash()
       `- setter: k230_machine_set_spi_flash()
```

当 QEMU 解析：

```bash
-machine k230,spi-flash=w25q256
```

会调用 setter，把用户输入保存为 Machine 实例自己的配置状态：

```c
s->spi_flash_model = g_strdup(value);
```

setter 只保存配置字符串，不创建设备。原因是属性解析时 SoC 和 SSIBus 可能还
没有完成初始化；Flash 创建要延迟到 `k230_machine_init()`，此时目标总线已经存在。

getter 返回 `g_strdup()` 副本，因为 QOM 字符串属性约定调用者负责释放返回值。
Machine finalize 再释放 `spi_flash_model` 自己拥有的字符串。实际 Flash 设备由
QOM 总线/对象树管理，不需要在 MachineState 中额外保存一个 Flash 指针。

#### P4-3 的设备连接流程

`k230_machine_init()` 的顺序必须是：

```text
1. object_initialize_child(machine, "soc", &s->soc, TYPE_RISCV_K230_SOC)
2. qdev_realize(&s->soc)
3. s->spi_flash_model 非空时调用 k230_connect_spi_flash()
4. 绑定 RAM、注册 machine-done notifier
```

第 2 步完成后，`K230DwSsiState` 已经在 init/realize 中准备好：

```c
/* init 阶段：创建数据总线 */
s->spi = ssi_create_bus(dev, "spi");

/* realize 阶段：创建控制器片选输出数组 */
s->cs_lines = g_new0(qemu_irq, s->num_cs);
qdev_init_gpio_out_named(dev, s->cs_lines, "cs", s->num_cs);
```

随后 `k230_connect_spi_flash()` 按以下顺序操作：

```text
flash_type
  -> module_object_class_by_name()
  -> 检查存在、非 abstract、属于 TYPE_M25P80
  -> qdev_new(flash_type)
  -> realize 前设置 drive 属性
  -> qdev_realize_and_unref(flash, BUS(ssi->spi), ...)
  -> qdev_get_gpio_in_named(flash, SSI_GPIO_CS, 0)
  -> qdev_connect_gpio_out_named(DEVICE(ssi), "cs", cs, flash_cs)
```

三个容易混淆的 API 分工如下：

| API | 作用 | 在本流程中的含义 |
|---|---|---|
| `qdev_realize_and_unref()` | 初始化设备并加入指定 QEMU bus，释放调用方临时引用 | 把 `w25q256` 加入 `ssi->spi`，让 SSIBus 管理它 |
| `qdev_get_gpio_in_named()` | 获取设备的命名 GPIO 输入句柄 | 获取 Flash 的 `SSI_GPIO_CS` 输入端 |
| `qdev_connect_gpio_out_named()` | 把一个设备的命名 GPIO 输出接到目标输入 | 把 SSI 的 `"cs"[0]` 接到 Flash 的片选输入 |

`qdev_realize_and_unref()` 只负责“设备入总线”和 realize，不会自动完成 CS
GPIO 接线；CS GPIO 连接是后两步单独完成的。SSI 也提供语义更明确的封装：

```c
ssi_realize_and_unref(flash, ssi->spi, &error_fatal);
```

它本质上等价于把 `SSIBus` 转换为 `BusState` 后调用通用的
`qdev_realize_and_unref()`。

#### 数据线和片选线是两条不同路径

```text
SPI 数据路径：
Guest 写 DR
  -> K230 SSI TX FIFO
  -> k230_dw_ssi_send_frame()
  -> ssi_transfer(s->spi, tx)
  -> SSIBus
  -> w25q256

CS 控制路径：
Guest 写 SER/SSIENR
  -> k230_dw_ssi_update_cs()
  -> qemu_irq_lower/raise(s->cs_lines[cs])
  -> Flash 的 SSI_GPIO_CS 输入
  -> ssi_cs_default()
  -> m25p80 的 set_cs()/事务边界
```

当前 K230 的 `spi0` 映射是 `dw_ssi[2]`，地址为 `0x91584000`，Patch 4 只连接
CS0，因此最终拓扑为：

```text
K230SoCState.dw_ssi[2]
  MMIO: 0x91584000
  SSIBus: spi
    `- w25q256（cs_index=0）
  "cs"[0] ----------------> SSI_GPIO_CS
```

Flash 的 `SSI_GPIO_CS` 输入索引始终是 `0`，因为一个 Flash 设备只有一个片选
输入；`cs` 参数 `0` 表示控制器输出数组中的第 0 路，二者不是同一个编号概念。

#### 从 Guest 寄存器操作到 Flash 命令的完整时序

以一条 Standard read 为例：

```text
1. Guest 禁用 SSI，设置 CTRLR0.TMOD=EEPROM_READ、DFS=8
2. Guest 设置 CTRLR1.NDF=read_len-1
3. Guest 使能 SSI，但保持 SER=0
4. Guest 将 opcode/address/dummy 写入 DR，进入 TX FIFO
5. Guest 写 SER=BIT(0)，控制器拉低 CS0
6. 控制器消费命令 TX FIFO，调用 ssi_transfer()，丢弃命令阶段 RX
7. 命令 FIFO 为空后，控制器自动发送 dummy 帧
8. 控制器把 NDF+1 个数据帧放入 RX FIFO
9. Guest 从 DR 读取数据
10. Guest 等待 BUSY=0，写 SSIENR=0，拉高 CS0 并清理事务状态
11. m25p80 根据 CS 边界完成本条 Flash 命令
```

关键边界是：

- `TMOD` 描述控制器数据路径，不应由 K230 SSI 解析 Flash opcode。
- `TO` 的线路返回值被控制器丢弃，不进入 RX FIFO。
- `EEPROM_READ` 的命令阶段返回值无效，只有数据阶段的 `NDF+1` 帧进入 RX FIFO。
- `SSIENR=0` 不只是关闭一个寄存器位，还负责撤销 CS、清空 FIFO 和清理阶段状态。
- Flash 的 WREN、Page Program、Sector Erase 必须使用独立的 CS 事务。

#### Patch 4 的边界

Patch 4 证明的是：

```text
现有 K230 Standard SPI 控制器路径
  + 正确的 SSIBus/CS 接线
  + QEMU m25p80 设备模型
  = 可验证的 Standard SPI NOR PIO 访问
```

Patch 4 不证明也不实现：

- QSPI Dual/Quad/Octal 线宽协议。
- DTR、RXDS、IDMA、IRQ/PLIC。
- `0xC0000000` XIP MemoryRegion。
- BootROM 冷启动和真实量产镜像启动。

### 1.1 TRM 证明控制器行为

K230 TRM 12.3 能证明：

- 控制器具有 TX/RX FIFO。
- `TMOD=TX_ONLY` 时发送 TX FIFO，但不保存线路返回值。
- `TMOD=EEPROM_READ` 时先发送 opcode/address 等控制帧，再把数据阶段写入
  RX FIFO。
- `CTRLR1.NDF` 编码接收帧数减一，即实际接收 `NDF+1` 帧。
- DR 写入 TX FIFO，DR 读取 RX FIFO。
- SER 选择外部从设备，SSIENR 控制控制器启停。

对应学习资料：

- [K230 TRM 12.3 SPI 中文学习版](k230-trm-12.3-spi-cn.md)
- 重点阅读“12.3.3.1 外设总线时序”和 `CTRLR0/CTRLR1/DRx` 位域说明。

TRM 不证明：

- K230 板上一定焊接了 W25Q256。
- Flash 的 JEDEC ID、page size 或 erase opcode。
- QEMU 应怎样绑定块后端。

这些属于板级软件和 QEMU 设备模型证据。

### 1.2 SDK 证明板级连接和软件访问方式

K230 U-Boot DTS：

```text
k230_sdk/src/little/uboot/arch/riscv/dts/k230.dtsi
k230_sdk/src/little/uboot/arch/riscv/dts/k230_evb.dtsi
```

提供以下事实：

```text
spi0 base       = 0x91584000
spi0 num-cs     = 1
Flash reg       = 0，即 CS0
compatible      = "jedec,spi-nor"
max frequency   = 100 MHz
```

K230 Linux DTS 也把 SPI NOR 放在 `spi0` 的 `reg = <0>`，并为该控制器列出
IRQ 146–154。

但量产板 DTS 通常声明：

```text
spi-tx-bus-width = <8>
spi-rx-bus-width = <8>
```

这属于后续 Octal/IDMA 能力，不应提前进入 Standard Flash Patch。SDK 同时存在
K230 FPGA 的 1-line SPI NOR 配置，且 U-Boot/Linux 的 DesignWare 驱动明确支持
Standard `spi-mem` 分支。因此 Patch 4 验证 Standard 路径是 SDK 支持范围内的
分层实验，不是伪造一种驱动协议。

### 1.3 QEMU 证明可复用的 Flash 设备

QEMU 已有：

```text
hw/block/m25p80.c
hw/block/m25p80_sfdp.c
```

其中 `w25q256` 的模型参数是：

```text
JEDEC ID   = ef 40 19
容量       = 64 KiB * 512 = 32 MiB
支持       = 4 KiB erase
SFDP       = 已提供
```

该模型已经处理 opcode、地址收集、WEL、NOR 1→0 编程、擦除、SFDP、CS 事务边界
和 backend 同步。K230 控制器只需要按正确顺序调用 `ssi_transfer()`，不能复制
这套状态机。

### 1.4 关于 W25Q256 的准确表述

SDK DTS 使用的是通用 `jedec,spi-nor`，没有在 compatible 中冻结具体厂商型号。
因此本文选择 `w25q256` 的依据是：

1. 项目 10-Patch 计划已经将测试 Flash 固定为 W25Q256。
2. QEMU 已有成熟的 `w25q256` SSIPeripheral。
3. 32 MiB 容量能够同时覆盖 3-byte 和大于 16 MiB 的 4-byte 地址测试。
4. K230 SDK 的 SPI NOR 表包含 `w25q256`，能够识别 JEDEC ID `ef 40 19`。

所以应写：

```text
Patch 4 使用 W25Q256 作为 K230 spi0 的 QEMU 测试模型。
```

不能写：

```text
K230 TRM 规定板载 Flash 必须是 W25Q256。
```

## 2. Patch 3 是 Patch 4 的前置门禁

在连接 Flash 之前，Patch 3 至少必须满足：

- Standard TR/TO/RO/EEPROM_READ 已实现。
- `SSIENR=1、SER=0` 时可预填 TX FIFO。
- 写 SER 后能够选择 CS 并推进预填事务。
- EEPROM_READ 命令阶段不写 RX FIFO。
- EEPROM_READ 数据阶段自动产生 `NDF+1` 个接收帧。
- RX FIFO 满时暂停，Guest 读取 DR 后恢复。
- SSIENR 清零撤销 CS并清理 FIFO/阶段状态。
- SR.BUSY、TXFLR、RXFLR 从 FIFO和传输阶段动态计算。

如果这些条件不成立，Patch 4 不应在 Flash helper 中绕过控制器修复。例如禁止：

```text
检测 opcode 0x9f 后直接返回 ef 40 19
直接读写 MTD backend 内存
所有 Flash 事务强制改成 TR
绕过 TX/RX FIFO 直接调用 m25p80
```

Flash 暴露出的基础控制器缺陷，应回修 Patch 3，而不是在 Patch 4 建立旁路。

## 3. 最容易混淆的实例映射

SDK 命名和 QEMU 内部数组顺序不同：

| SDK 名称 | 地址 | QEMU memmap 名称 | `dw_ssi[]` | 最大线宽 | Patch 4 |
|---|---:|---|---:|---:|---|
| `spi1` | `0x91582000` | `K230_DEV_QSPI0` | 0 | 4 | 不接 Flash |
| `spi2` | `0x91583000` | `K230_DEV_QSPI1` | 1 | 4 | 不接 Flash |
| `spi0` | `0x91584000` | `K230_DEV_SPI` | 2 | 8 | CS0 接 W25Q256 |

因此代码中必须连接：

```c
&s->dw_ssi[2]
```

不能因为数组下标而误接到 `dw_ssi[0]`。判断依据始终是 SDK 地址
`0x91584000`，不是局部变量命名。

## 4. 生产代码的最小文件边界

Patch 4 原则上只修改：

```text
hw/riscv/Kconfig
hw/riscv/k230.c
include/hw/riscv/k230.h
tests/qtest/k230-dw-ssi-flash-test.c
tests/qtest/k230-dw-ssi-test-common.c
tests/qtest/k230-dw-ssi-test.h
```

如果 Patch 3 的 EEPROM_READ 尚未完成，应先完成并回归 Patch 3；不要把大量
控制器传输实现混入“attach Flash”的 Patch 4。

Patch 4 不需要修改：

```text
hw/block/m25p80.c
hw/ssi/k230_dw_ssi.c       # 除非发现并回修 Patch 3 缺陷
K230 SDK
Guest U-Boot/Linux 产物
```

## 5. P4-1：加入构建依赖

位置：`hw/riscv/Kconfig` 的 `config K230`。

加入：

```text
select SSI_M25P80
```

依据：

- `hw/block/meson.build` 只在 `CONFIG_SSI_M25P80` 时编译 `m25p80.c` 和 SFDP。
- K230 machine 将直接实例化 `w25q256`，因此这不是可选的运行时猜测，而是
  machine 的链接依赖。

不要通过在 C 文件中条件编译来规避 Kconfig；machine 声明依赖更直接。

完成检查：

```bash
ninja -C "my-qemu-camp-2026-k230/build" qemu-system-riscv64
```

## 6. P4-2：增加显式 machine 属性

命令行接口固定为：

```bash
qemu-system-riscv64 \
    -machine k230,spi-flash=w25q256 \
    -drive file=flash.bin,format=raw,if=mtd
```

默认行为：

```text
-machine k230
  -> 不创建任何 SPI NOR
```

建议在 `K230MachineState` 保存：

```c
char *spi_flash_model;
```

并在 machine class 注册字符串属性：

```c
object_class_property_add_str(oc, "spi-flash",
                              k230_machine_get_spi_flash,
                              k230_machine_set_spi_flash);
```

setter 只保存字符串；实际创建设备必须等到 `k230_machine_init()` 中 K230 SoC
完成 realize、spi0 的 SSIBus 和 CS GPIO 已经存在之后。

属性值必须验证：

- QOM type 存在。
- 不是 abstract type。
- 是 `TYPE_M25P80` 的子类型。

错误型号应直接报告配置错误并退出，不能静默回退到 W25Q256。该验证可参考
`xlnx-versal-virt` 的 `ospi-flash` machine 属性。

machine finalize 必须释放 `spi_flash_model`。本地若暂不维护 machine ABI，可将
该属性保留在本地 Patch 4；未来尝试上游时再讨论命名和兼容承诺。

## 7. P4-3：建立通用的板级 Flash 连接 helper

位置：`hw/riscv/k230.c`。

需要的头文件：

```c
#include "system/blockdev.h"
#include "hw/block/flash.h"
#include "hw/ssi/ssi.h"
```

建议 helper 契约：

```c
static void k230_connect_spi_flash(K230DwSsiState *ssi,
                                   unsigned int cs,
                                   const char *flash_type,
                                   DriveInfo *dinfo);
```

虽然 Patch 4 只连接 CS0，保留显式 `cs` 参数可以避免 helper 把 CS0 隐藏成
不透明常量；但不需要为了未来多 Flash 建立复杂数组或配置对象。

实现步骤：

```text
1. qdev_new("w25q256")
2. 若 dinfo 非空，将 blk_by_legacy_dinfo(dinfo) 绑定到 drive 属性
3. 在 K230 SSI 的 SSIBus 上 realize Flash
4. 取得 Flash 的 SSI_GPIO_CS 输入
5. 将控制器命名 CS 输出的指定 cs 连接到 Flash CS 输入
```

参考形态：

```c
DeviceState *flash = qdev_new(flash_type);

if (dinfo) {
    qdev_prop_set_drive(flash, "drive",
                        blk_by_legacy_dinfo(dinfo));
}

qdev_realize_and_unref(flash, BUS(ssi->spi), &error_fatal);

qemu_irq flash_cs = qdev_get_gpio_in_named(flash, SSI_GPIO_CS, 0);
qdev_connect_gpio_out_named(DEVICE(ssi), "cs", cs, flash_cs);
```

注意所有权：

- `qdev_new()` 返回未 realize 的 DeviceState。
- backend 属性必须在 realize 前设置。
- `qdev_realize_and_unref()` 将设备交给 QOM/bus 管理，调用方不再持有引用。
- Flash 必须挂在 `ssi->spi` 上，不能挂到 system bus。
- 数据通过 SSIBus，CS 通过 GPIO 单独连接；二者缺一不可。

## 8. P4-4：在正确的时机连接 spi0 CS0

在 `k230_machine_init()` 中 SoC realize 完成后，仅当 `spi_flash_model` 非空时
调用：

```c
k230_connect_spi_flash(&s->soc.dw_ssi[2], 0, s->spi_flash_model,
                       drive_get(IF_MTD, 0, 0));
```

此时三个 SSI 已在 SoC realize 中完成属性设置、realize 和 MMIO map。把可选
板载器件放在 machine 层创建，可以让 `K230SoCState` 保持只描述 SoC 集成，
同时让 machine 属性决定当前虚拟板是否实际安装 Flash。

连接后的结构：

```text
K230SoCState.dw_ssi[2]
  MMIO: 0x91584000
  SSIBus: spi
    `- w25q256
  cs[0] GPIO
    `- w25q256 SSI_GPIO_CS
```

不要连接到：

- `dw_ssi[0]` 或 `0x91582000`。
- `dw_ssi[1]` 或 `0x91583000`。
- XIP window `0xC0000000`；那是 Patch 9 的 MemoryRegion。

Patch 4 必须保留 `k230_soc_realize()` 中现有的
`create_unimplemented_device("flash", ...)`。挂接 SPI NOR 只增加 SSIBus 从设备，
不替换、删除或重映射 `0xC0000000`；Patch 9 才接管该 XIP aperture。

## 9. P4-5：定义 backend 语义

### 9.1 同时提供 machine 属性和 `-drive if=mtd`

启动参数：

```bash
-drive file=flash.bin,format=raw,if=mtd
```

machine 使用：

```c
drive_get(IF_MTD, 0, 0)
```

把第一个 MTD backend 绑定给 `spi-flash` 属性选择的 spi0 CS0 从设备。

测试镜像大小应为 32 MiB，与 QEMU `w25q256` 容量一致。测试应使用临时文件，
退出时删除，不能改动 SDK 的真实镜像或工作区中的构建产物。

### 9.2 指定 `spi-flash`，但没有 `-drive if=mtd`

仍然创建属性指定的 Flash，但不设置 `drive`。对于 `w25q256`，QEMU
`m25p80_realize()` 会分配 32 MiB 内部存储并初始化为 `0xff`。

该行为的好处：

- `-machine k230` 不会因为缺少 Flash 镜像而启动失败。
- Guest 仍可读取 JEDEC ID。
- 未写入区域符合擦除态 Flash 的 `0xff` 语义。

不要在 machine 中额外复制一份 Flash RAM；无 backend 的生命周期已经由
`m25p80` 负责。

### 9.3 只有 `-drive if=mtd`，但没有 `spi-flash`

不自动猜测器件型号，也不因为存在 MTD backend 就隐式创建 Flash。推荐把这种
组合视为配置错误并提示：

```text
MTD backend requires -machine k230,spi-flash=<model>
```

这样 `-drive` 只提供存储内容，machine 属性明确决定板上安装什么器件，两种
职责不会混在一起。

### 9.4 两者都没有

不创建 Flash。普通 `-machine k230`、direct boot 和不涉及存储的 qtest 不增加
任何 W25Q256 假设或 M25P80 状态。

### 9.5 写入持久性

如果 raw backend 可写，QEMU M25P80 会把 program/erase 同步回 backend。
测试必须使用临时可写镜像，不能把生产启动镜像直接用于破坏性 qtest。

## 10. P4-6：重新定义 Standard Flash 测试 helper

旧原型常把所有 Flash 操作都展开为 TR：

```text
tx = opcode + address + dummy bytes
TR 逐字节交换
从 rx[prefix] 取数据
```

这种写法能驱动 M25P80，但不能证明 K230 SDK 的 DesignWare 模式选择。Patch 4
必须按 SDK 实际方向拆分 helper。

建议提供两个核心 helper。

### 10.1 Standard write/no-data helper：TO

适用于：

- WREN `0x06`
- Page Program `0x02 + address + data`
- Sector Erase `0x20 + address`
- 其他纯命令或 Data-OUT 操作

序列：

```text
SSIENR = 0
CTRLR0.SPI_FRF = Standard
CTRLR0.TMOD = TO
DFS = 8 bit
SER = 0
SSIENR = 1
按顺序写 command/address/data 到 DR
SER = BIT(0)              # 启动预填事务
等待 TXFLR=0 且 BUSY=0
SSIENR = 0                # 撤销 CS，结束 Flash 命令
```

控制帧的返回值不能进入 RX FIFO。

### 10.2 Standard Data-IN helper：EEPROM_READ

适用于：

- JEDEC ID `0x9f`
- Read Status `0x05`
- Read `0x03 + address`
- Read4 `0x13 + address`
- SFDP `0x5a + address + dummy`

输入应明确区分：

```text
command[]     opcode、address 和协议要求的 dummy command bytes
command_len
rx_len        只统计数据阶段
```

序列：

```text
SSIENR = 0
CTRLR0.SPI_FRF = Standard
CTRLR0.TMOD = EEPROM_READ
CTRLR1.NDF = rx_len - 1
DFS = 8 bit
SER = 0
SSIENR = 1
把 command[] 全部写入 TX FIFO
SER = BIT(0)              # 发送命令并自动进入 NDF+1 接收
从 RX FIFO读取恰好 rx_len 帧
等待 BUSY=0
SSIENR = 0                # 撤销 CS
```

RX 数组只包含数据阶段，不需要再跳过 opcode/address 对应的“假 RX”。这正是
EEPROM_READ 与 TR helper 的核心区别。

### 10.3 qtest helper 与测试用例的分工

事务 helper 只负责控制器寄存器和 FIFO 时序，测试用例负责提供 Flash 协议
字段和断言。两者不要互相越权：

```text
flash_write_transaction()
  接收完整的 command[]：opcode + address + data
  只组织 TO 发送，不解释 opcode

flash_read_transaction()
  接收 command[] 和 data_len
  只组织 EEPROM_READ/NDF 接收，不解释返回数据

flash_read()
  把 opcode、big-endian address、dummy bytes 组装成 command[]
  再调用 flash_read_transaction()

flash_read_status()
  用 RDSR + 1 字节 Data-IN 事务读取 WIP/WEL 等状态

flash_write_enable()
  用独立 TO 事务发送 WREN，依赖 CS 结束边界锁存 WEL
```

当前 helper 会在拉起 CS 前一次性预填命令 FIFO，因此对命令长度做 FIFO 容量
断言；Data-IN helper 会先等待本次数据全部进入 RX FIFO，再读取数据，因此测试
数据长度必须小于 FIFO 深度。Patch 4 的 JEDEC、3/4-byte read、Page Program、
Sector Erase 和 CS restart 都远小于该限制。

### 10.4 为什么每次事务最后必须撤销 CS

QEMU `m25p80` 使用 CS 上升沿结束或提交事务：

- 结束可变长度 Page Program。
- 把状态恢复到可解析下一条 opcode 的 IDLE。
- 清理当前 read loop。
- 将脏页同步到 backend。

若两条命令之间 CS 始终保持有效，第二个 opcode 可能被解释为上一条命令的
地址、dummy 或数据。因此 helper 必须形成清晰的 CS 边界。

在当前 K230 模型中，写 `SSIENR=0` 会统一撤销 CS 并清理控制器事务，适合作为
每条 qtest Flash 命令的结束动作。

## 11. P4-7：七项 Standard Flash 验证

### F01：JEDEC ID

命令：

```text
command = [0x9f]
TMOD    = EEPROM_READ
NDF     = 5
rx_len  = 6
```

采用 6 字节是为了贴近 K230 U-Boot `SPI_NOR_MAX_ID_LEN` 的 Read-ID 请求；至少
检查前三字节：

```text
ef 40 19
```

该测试直接覆盖本次真实 `sf probe` 曾阻塞的位置。

验证点：

- 命令阶段的线路返回值没有进入 RX FIFO。
- RXFLR 最终得到 6 个数据帧，或被 Guest 边读边补充到总计 6 帧。
- JEDEC ID 与 QEMU `w25q256` 一致。

测试路径：

```text
/k230-dw-ssi/flash/jedec-id
```

### F02：3-byte address read

预先在 backend 的低地址写入固定 pattern，例如：

```text
offset 0x00000100
data   a5 5a 3c c3 11 22 33 44
```

事务：

```text
command = [0x03, 0x00, 0x01, 0x00]
TMOD    = EEPROM_READ
NDF     = 7
rx_len  = 8
```

验证读回与 backend pattern 完全相同。

测试路径：

```text
/k230-dw-ssi/flash/read-3byte
```

### F03：4-byte address read

W25Q256 容量为 32 MiB。选择大于 16 MiB 的地址，例如：

```text
offset = 0x01000100
```

使用专用 4-byte read opcode：

```text
command = [0x13, 0x01, 0x00, 0x01, 0x00]
TMOD    = EEPROM_READ
```

该测试不是 XIP，也不是进入全局 4-byte address mode；它只验证 opcode `0x13`
携带四字节地址能够访问 Flash 高半区。

测试路径：

```text
/k230-dw-ssi/flash/read-4byte
```

### F04：Page Program

Flash 编程前必须执行 WREN：

```text
事务 1：TO [0x06]
CS 拉高

事务 2：TO [0x02, A23, A15, A7, payload...]
CS 拉高

事务 3：EEPROM_READ [0x05]，读取状态
事务 4：EEPROM_READ [0x03, address...]，读回验证
```

注意：

- 测试地址必须预先为 `0xff`。
- payload 不跨越 256-byte page 边界。
- NOR 编程只能可靠地把 bit 从 1 改成 0；不能用 program 验证 0→1。
- QEMU M25P80 当前 program/erase 是同步模型，RDSR.WIP 通常不会模拟真实耗时；
  状态轮询仍应保留以符合 Guest 流程，但不要写依赖真实毫秒延迟的测试。

测试路径：

```text
/k230-dw-ssi/flash/page-program
```

### F05：4 KiB Sector Erase

先在 4 KiB 对齐扇区准备非 `0xff` 内容，然后执行：

```text
事务 1：TO [0x06]
事务 2：TO [0x20, A23, A15, A7]
事务 3：EEPROM_READ [0x03, address...] 读回
```

验证选定范围恢复为 `0xff`。地址应 4 KiB 对齐；至少读取 16 字节确认，必要时
可检查整个扇区，但不应为了一个语义点引入过大的 qtest IO。

测试路径：

```text
/k230-dw-ssi/flash/sector-erase
```

### F06：CS 重启命令解析

连续执行两次独立 JEDEC ID 事务：

```text
CS low  -> 0x9f -> read ID -> CS high
CS low  -> 0x9f -> read ID -> CS high
```

两次都必须从 `ef` 开始。该测试用于捕获：

- 控制器禁用时没有撤销 CS。
- 下一事务沿用旧 EEPROM_DATA phase。
- M25P80 没有收到命令结束边界。
- TX/RX FIFO 没有在禁用时清理。

测试路径：

```text
/k230-dw-ssi/flash/cs-restarts-command
```

### F07：增强 reset 失败后回退 Read-ID

先按 U-Boot `spi_hw_init()` 的顺序，在 `SSIENR=1` 时通过 `TXFTLR` 写回探测
256 项 FIFO；再模拟一次未支持的 Octal DTR reset-enable，撤销 CS并禁用 SSI，
最后执行 Standard EEPROM_READ Read-ID。

验证点：

- enabled 状态下 `TXFTLR/RXFTLR` 仍可更新。
- 增强事务未发送的 TX 项在 `SSIENR=0` 时清理。
- Standard 回退重新使用 `SPI_FRF=0、TMOD=3、NDF=5`。
- RX FIFO 最终获得 6 字节，前三字节为 `ef 40 19`。

测试路径：

```text
/k230-dw-ssi/flash/enhanced-reset-fallback-read-id
```

所有 Flash qtest 的 QEMU 参数必须显式包含：

```text
-machine k230,spi-flash=w25q256
-drive file=<tmp>,format=raw,if=mtd
```

普通寄存器和 PIO qtest 继续只使用 `-machine k230`，以验证可选 Flash 不会改变
默认 machine 行为。

## 12. qtest 镜像的安全准备

建议临时镜像布局：

| 地址 | 初始内容 | 用途 |
|---:|---|---|
| `0x00000100` | 固定 8-byte pattern | 3-byte read |
| `0x01000100` | 固定 4-byte pattern | 4-byte read |
| `0x00022000` | 至少一个 page 为 `0xff` | Page Program |
| `0x00024000` | 一个 4 KiB sector 为非 `0xff` | Sector Erase |

流程：

```text
创建临时文件
调整为 32 MiB
写入测试所需的确定性区域
以 -machine k230,spi-flash=w25q256 和 -drive if=mtd 启动 QEMU
测试结束 qtest_quit()
删除临时文件
```

如果只写部分区域，稀疏文件的未写区域通常读为 0，不等于擦除态 `0xff`；因此
所有参与“空 Flash”断言的区域必须显式填充 `0xff`。最简单的办法是只初始化
测试会访问的 page/sector，避免为了少量断言机械写满 32 MiB。

不得使用：

- SDK 原始 Flash 镜像。
- U-Boot/Linux 编译输出作为写测试 backend。
- 工作区中需要保留的启动镜像。

## 13. Patch 4 与 Patch 5 的隔离

当前历史测试文件可能同时包含 `/flash/*` 和 `/qspi/*`。最终重拆时：

Patch 4 只包含：

```text
flash/jedec-id
flash/read-3byte
flash/read-4byte
flash/page-program
flash/sector-erase
flash/cs-restarts-command
flash/enhanced-reset-fallback-read-id
```

以下测试属于 Patch 5：

```text
qspi/dual-quad-output-read
qspi/mode-bits-dummy
qspi/quad-page-program
qspi/rx-fifo-resume
qspi/prefix-atomic
qspi/unsupported-configs
```

Patch 4 的生产代码也不得读取 `SPI_CTRLR0.TRANS_TYPE/WAIT_CYCLES` 来合成增强
事务；这些字段留给 Patch 5。

## 14. machine Flash 与 XIP window 的关系

显式选择 Flash 不会妨碍后续 XIP，前提是 XIP 采用协议访问，而不是把 backend
直接映射成 RAM。

正确关系：

```text
-machine k230,spi-flash=w25q256
  -> 在 spi0 SSIBus/CS0 创建唯一一颗 w25q256

PIO MMIO
  -> TX/RX FIFO/TMOD
  -> spi0 SSIBus/CS0
  -> 同一颗 w25q256

XIP MemoryRegion @ 0xC0000000
  -> spi0 XIP 事务生成器
  -> spi0 SSIBus/CS0
  -> 同一颗 w25q256
```

禁止以下设计：

```text
XIP MemoryRegion 直接映射 flash.bin
XIP callback 直接读取 BlockBackend
PIO 挂一颗 M25P80，XIP 再创建第二颗 Flash
XIP 保存并调用 W25Q256 私有状态
```

否则 PIO program/erase 和 XIP 会看到不同状态，opcode、地址模式、dummy、CS、
WEL 等 Flash 语义也会被绕过。

### 14.1 XIP 如何访问同一颗 Flash

XIP read callback 根据 `XIP_INCR_INST/XIP_WRAP_INST/SPI_CTRLR0` 等 Guest
配置生成一次完整 SPI 事务：

```text
assert CS0
发送 read opcode
发送 address
发送 mode bits / dummy clocks
调用 ssi_transfer(0) 接收请求字节
deassert CS0
```

它应复用 Patch 5 的增强阶段生成器和同一个 SSIBus 帧交换 helper，但不经过
Guest TX/RX FIFO。machine 属性只决定安装什么型号；XIP opcode、地址宽度和
dummy 由 Guest 的控制器配置决定。配置与器件不匹配时应按真实硬件失败，
machine 不能偷偷修改协议。

### 14.2 PIO 与 XIP 的互斥

当前计划不实现 concurrent XIP，因此两条路径不能交错占用同一个 CS0：

- XIP 开始前不得存在 active PIO 命令、待发送 TX FIFO 或未完成 phase。
- PIO 正忙时的 XIP 访问必须被拒绝或返回确定失败值，并记录 guest error。
- XIP 事务期间 MMIO pump 不能抢占 CS0。
- 每次 XIP 事务结束必须撤销 CS，使 Flash 回到下一命令边界。

具体使用全 1 返回值还是 `MemTxError`，在 Patch 9 统一裁决；绝不能把 XIP 帧
静默插入正在进行的 EEPROM_READ 或 Page Program。

### 14.3 128 MiB aperture 与 32 MiB W25Q256

K230 XIP aperture 为 `0xC0000000–0xC7ffffff`，共 128 MiB；W25Q256 只有
32 MiB。窗口表示控制器可解码的地址范围，不要求所挂器件必须同样大。

Patch 9 首先只承诺：

```text
0 <= XIP offset < 32 MiB
```

超过器件容量后的 wrap、alias 或空闲值不能仅凭窗口大小推断。高于 16 MiB 的
有效地址还要求 Guest 正确配置 4-byte opcode/address；`spi-flash` 属性不会自动
替 Guest 切换地址模式。

### 14.4 没有显式 Flash 时

默认 `-machine k230` 不安装 spi0 Flash。XIP window 仍可作为控制器能力在
Patch 9 实现，但 `ssi0_xip_en=0` 时保持禁用；Guest 强行开启却没有 CS0 从设备
时，应返回确定的空闲/错误结果，不能访问不存在的 backend。

因此二者并不矛盾：machine 属性决定“板上是否安装从设备”，XIP window决定
“CPU 是否通过控制器向该从设备发起读取事务”。

Flash 在 `k230_machine_init()` 中、vCPU 开始运行之前完成创建和接线，所以将来
BootROM 在启动早期访问 spi0/XIP 时器件已经存在。但 machine 属性本身不会让
CPU 自动从 `0xC0000000` 取指；reset vector、BootROM 分支、`ssi0_xip_en` 和 XIP
协议寄存器仍属于独立的启动链路实现与验证。

## 15. 与真实 U-Boot/Linux 的关系

### 15.1 U-Boot Standard 集成验证

Patch 4 qtest 通过后，可复用
[K230 完整启动与 IOMUX 验证](uboot-iomux-unimplemented-repro.md) 的方法，
用未修改的 U-Boot 二进制和 `/tmp` 派生 DTB启用：

```dts
&spi0 {
    status = "okay";

    flash@0 {
        compatible = "jedec,spi-nor";
        reg = <0>;
        spi-max-frequency = <100000000>;
    };
```

不写 `spi-tx-bus-width`/`spi-rx-bus-width` 时，spi-mem 默认按 1-line Standard
访问。这样可以验证：

```text
sf probe 0:0
sf read
sf write
sf erase
```

这不修改 SDK 源码或原始 U-Boot，只是在 Guest RAM 中覆盖派生 DTB。

`sf probe` 可能先打印 Octal DTR software reset 的 `-ENOTSUPP` 警告；SDK 源码
规定该失败非致命，随后仍尝试 Standard Read-ID。判断 Patch 4 是否成功，应以
最终能否识别 W25Q256并返回提示符为准，不能只看这条警告。

QEMU 同时显式提供：

```bash
-machine k230,spi-flash=w25q256 \
-drive file=flash.bin,format=raw,if=mtd
```

### 15.2 原始量产 DTS 仍不属于 Patch 4 完成条件

原始 K230 U-Boot/Linux DTS 声明 8-line，SDK 会进入增强/IDMA 分支。Patch 4
只证明 Standard Flash，因此以下现象不构成 Patch 4 回归：

```text
原始 Linux enh_mem_op 失败
Octal software reset 不支持
IDMA/DONE/AXIE 未完成
XIP window 不可用
```

这些分别属于后续 QSPI/IDMA/IRQ/XIP 范围。若目标改变为“不修改原始 8-line
DT 即支持 Linux Flash”，必须先扩大总体计划边界，不能把全部功能塞进 Patch 4。

## 16. 常见错误及定位方法

### 16.1 JEDEC ID 全为 0 或 RXFLR 永远为 0

优先检查：

```text
CTRLR0.TMOD 是否为 EEPROM_READ
CTRLR1 是否为 rx_len - 1
SSIENR 是否为 1
SER 是否选择 CS0
TX FIFO 是否发送了 0x9f
EEPROM_DATA 是否自动产生接收帧
```

若 PC 卡在 U-Boot `dw_reader()` 读取 RXFLR，通常仍是 Patch 3 EEPROM_READ 问题，
不是 board Flash 接线本身。

### 16.2 JEDEC ID 为 ff ff ff

可能原因：

- Flash 没有被 CS 选中，SSIBus 返回空闲值。
- CS GPIO 极性或连接目标错误。
- Flash 连接到了错误的 `dw_ssi[]`。

检查 QOM tree 和 controller `active_cs`，并确认连接的是 SDK `spi0`
`0x91584000`。

### 16.3 第一条命令成功，第二条失败

优先检查 CS 是否在两条命令间拉高，以及 SSIENR=0 是否真正调用 deselect。
F06 就是该问题的最小回归测试。

### 16.4 Program 后数据不变

检查：

- WREN 是否是独立 TO 事务并有 CS 结束边界。
- program 目标是否预先为 `0xff`。
- Page Program 是否跨 page。
- backend 是否只读。
- opcode/address/data 是否在同一次 CS low 期间发送。

### 16.5 Erase 后仍为 0

检查：

- 是否先 WREN。
- opcode 是否为 4 KiB erase `0x20`。
- 地址是否正确且位于测试 backend 范围。
- readback 是否仍使用错误的 TR prefix 偏移。

### 16.6 qtest 能过但 `sf probe` 失败

qtest 只覆盖六个固定命令，而 U-Boot probe 还可能执行：

- Octal DTR software reset 尝试。
- 读取 6-byte JEDEC ID。
- Read SFDP `0x5a`，包含地址和 dummy byte。
- 读取/写入状态或配置寄存器。

先确认失败事务仍是 Standard。若是 Standard SFDP，应检查 EEPROM_READ 是否能把
opcode、3-byte address 和 dummy byte全部作为命令阶段发送，再单独接收数据。
不要因此提前实现 Quad/Octal。

## 17. 实施顺序

建议严格按以下顺序工作：

1. 确认 Patch 3 的 RO/EEPROM_READ、动态 BUSY 和 CS 结束语义。
2. Kconfig 加入 `SSI_M25P80`。
3. 增加 `spi-flash` machine 字符串属性和 M25P80 型号校验。
4. 在 `k230.c` 建立最小 Flash 连接 helper。
5. 仅在显式属性存在时，将选择的型号连接到 `dw_ssi[2]` CS0。
6. 构建 QEMU，确认默认无 Flash和显式空 Flash两种 machine 都能启动。
7. 准备 32 MiB 临时 raw backend。
8. 将 Standard Flash helper 从 TR 改成 TO/EEPROM_READ。
9. 依次完成 JEDEC、3-byte read、4-byte read。
10. 完成 WREN、Page Program、RDSR 和 readback。
11. 完成 Sector Erase 和 CS restart。
12. 跑 `reg + pio + flash`，确认 Patch 4 没有破坏 Patch 1–3。
13. 最后用未修改 U-Boot + `/tmp` Standard DTB执行 `sf probe` 集成验证。

不要先做写/擦除。JEDEC 和只读测试能先隔离 bus、CS、EEPROM_READ 和 backend，
破坏性测试放在基础读路径稳定之后更容易定位问题。

## 18. 验证命令

构建：

```bash
ninja -C "my-qemu-camp-2026-k230/build" \
    qemu-system-riscv64 \
    tests/qtest/k230-dw-ssi-test
```

单项：

```bash
QTEST_QEMU_BINARY="./my-qemu-camp-2026-k230/build/qemu-system-riscv64" \
    "./my-qemu-camp-2026-k230/build/tests/qtest/k230-dw-ssi-test" \
    -p "/riscv64/k230-dw-ssi/flash/jedec-id" --tap
```

Flash 组：

```bash
"my-qemu-camp-2026-k230/tests/qtest/run-k230-dw-ssi-tests.sh" "flash"
```

前置组回归：

```bash
"my-qemu-camp-2026-k230/tests/qtest/run-k230-dw-ssi-tests.sh" "reg"
"my-qemu-camp-2026-k230/tests/qtest/run-k230-dw-ssi-tests.sh" "pio"
```

静态检查：

```bash
git -C "my-qemu-camp-2026-k230" diff --check
```

## 19. Patch 4 完成标准

- [ ] K230 Kconfig 选择 `SSI_M25P80`。
- [ ] `-machine k230,spi-flash=<m25p80-model>` 可显式选择器件。
- [ ] 默认 `-machine k230` 不创建 Flash。
- [ ] 显式 `w25q256` 连接到 SDK `spi0 @ 0x91584000` CS0。
- [ ] `-drive if=mtd` 绑定第一个 32 MiB raw backend。
- [ ] 指定型号但无 MTD backend 时，Flash 使用内部擦除态存储。
- [ ] 控制器中没有 Flash opcode 或 backend 访问旁路。
- [ ] Standard Data-IN 使用 EEPROM_READ，不使用 TR 假装读取。
- [ ] WREN/program/erase 使用 TO。
- [ ] 七项 `/flash/*` qtest 全部通过。
- [ ] `reg + pio` 持续通过。
- [ ] QSPI、IRQ、IDMA、XIP 没有提前混入本 Patch。
- [ ] 未修改 K230 SDK和原始 U-Boot/Linux 编译产物。

最终应能解释：

| 问题 | 答案 |
|---|---|
| 为什么 Flash 型号不由 TRM 决定？ | TRM 描述 SSI IP；板载器件由 DTS/板级设计决定，QEMU 型号是模型选择。 |
| 为什么不能在 SSI 中解析 `0x9f`？ | opcode 状态属于 SPI NOR；SSI 只负责寄存器、FIFO、CS和帧传输。 |
| 为什么 read 使用 EEPROM_READ？ | SDK只写命令和 NDF，硬件必须自动产生数据阶段时钟。 |
| 为什么 write 使用 TO？ | 返回值无效，不应污染 RX FIFO。 |
| 为什么 WREN 和 Program 必须分成两次 CS 事务？ | WREN 是独立命令；CS 边界让 Flash 提交命令并开始解析下一 opcode。 |
| 为什么 qtest 不能使用真实 SDK镜像？ | program/erase 会修改 backend，测试必须隔离且可重复。 |
| 为什么 Patch 4 不能宣称原始 Linux Flash 可用？ | 原始 DTS 是 8-line 增强/IDMA路径，超出 Standard Patch 4。 |
| machine 属性会不会妨碍 XIP？ | 不会；属性只安装从设备，PIO/XIP 都经同一 spi0 SSIBus 和 CS0 访问。 |

## 学习小结

Patch 4 的职责可以压缩为一句话：

```text
把现有 W25Q256 作为 K230 spi0 CS0 的真实 SSIPeripheral 接入，
并用 SDK 一致的 TO/EEPROM_READ 序列证明 Patch 3 的 Standard SPI 数据路径。
```

它不是新的 Flash 控制器实现，也不是 QSPI、IDMA 或 XIP Patch。保持这个边界，
后续 Patch 5 才能只扩展增强传输，而不重写已经验证过的 Flash、FIFO 和 CS。
