# K230 IOMUX/Pinctrl 三阶段入门指南

本文面向第一次在 QEMU 里做 SoC 基础模块建模的同学。目标不是一次性做出完整 K230 IOMUX，而是按可验证、可解释、可提交 PR 的方式，完成 `#12 qemu-k230: model iomux/pinctrl` 的第一版最小模型。

当前建议路线：

```text
阶段 1：定范围和证据链。先确认 IOMUX 是什么、SDK/U-Boot/Linux 哪里会访问它。
阶段 2：实现最小模型。把 0x91105000 这段窗口做成 32-bit MMIO register bank。
阶段 3：验证和收尾。补 qtest、跑回归、更新 QEMU 文档，准备 PR 说明。
```

如果目标是 4 天内完成第一版，可以按这个节奏压缩：

```text
第 1 天：阶段 1，读代码、读 SDK 证据、定第一版边界。
第 2 天：阶段 2，写 k230_iomux.c/h，接入 Kconfig/meson/k230.c。
第 3 天：阶段 3，写 qtest，跑构建和测试。
第 4 天：补文档、补 trace/注释、清理 PR 描述和验证日志。
```

## 0. 先给结论

`IOMUX/Pinctrl` 不是 UART、SPI、GPIO 那种直接收发数据的外设。它更像 SoC 的“引脚配置寄存器块”。

真实硬件里，一个物理引脚可能可以复用成多种功能：

```text
同一个 pin 可以被配置成：
GPIO
UART TX/RX
SPI CLK/MOSI/MISO/CS
I2C SCL/SDA
SDIO
PWM
```

IOMUX/Pinctrl 的任务是告诉芯片：

```text
这个 pin 当前选择哪个功能？
是否上拉？
是否下拉？
驱动能力多大？
输入输出属性是什么？
```

在 QEMU 第一版里，最稳妥的目标是寄存器兼容：

```text
把 0x91105000..0x911057ff 这段 IOMUX MMIO 窗口从 unimplemented device
推进成一个可读写、可 reset、可 qtest 验证的寄存器 bank。
```

也就是说，第一版重点是：

- MMIO 地址范围正确。
- 32-bit 读写行为明确。
- 写入寄存器后能读回。
- reset 后回到明确默认值。
- qtest 能证明这些行为。

这已经是一个合理的入门 PR。

## 阶段 1：先看懂模块，定清楚第一版边界

建议用半天到一天完成。这个阶段不是为了把 TRM 全读懂，而是为了避免后面写代码时范围失控。

### 1.1 IOMUX 在启动链路中的位置

一个外设能工作，通常不只靠外设自己的寄存器。

以 UART/SPI 为例，真实 SoC 上常见路径是：

```text
RMU：解除外设 reset
CMU：打开外设 clock
IOMUX：把相关 pin 配成 UART/SPI 功能
UART/SPI controller：配置波特率、模式、FIFO，然后收发数据
```

所以 IOMUX 是很多外设 probe 的前置配置之一。

但是要注意：

```text
IOMUX 不负责 UART 发送字符。
IOMUX 不负责 SPI 传输 command。
IOMUX 不负责 SD 卡读写。
IOMUX 只负责 pin 配置寄存器。
```

QEMU 第一版只做寄存器兼容，是合理的。

### 1.2 为什么它仍然是 QEMU 设备

在 QEMU 里，一个“设备”不一定必须有复杂数据通路。只要 guest 会访问一段 MMIO 地址，这段地址就需要一个模型来响应读写。

当前 K230 machine 里，IOMUX 还是占位：

```c
create_unimplemented_device("iomux", memmap[K230_DEV_IOMUX].base,
                            memmap[K230_DEV_IOMUX].size);
```

对应地址来自 `hw/riscv/k230.c`：

```text
IOMUX base: 0x91105000
IOMUX size: 0x00000800
```

`create_unimplemented_device()` 的问题是：

- 读通常返回 `0`。
- 写通常忽略。
- 会记录未实现访问。
- 如果驱动只是写配置，可能暂时没事。
- 如果驱动读回确认或者等待某些位，就可能行为不对。

IOMUX 第一版模型要解决的是：这段地址不再只是“黑洞”，而是有基本寄存器语义。

### 1.3 第一版边界

建议第一版明确写成：

```text
支持：
  - K230 IOMUX MMIO 地址窗口。
  - 32-bit 寄存器读写。
  - 写入值保存，读回返回保存值。
  - reset 默认值。
  - qtest 覆盖基本读写和 reset 行为。

不支持：
  - 不模拟 Linux pinctrl framework 的完整语义。
  - 不把 pin function 配置传播到 UART/SPI/I2C/SD/QSPI 等其他设备。
  - 不实现 TRM 中没有证据支撑的复杂副作用。
```

这符合 KISS 和 YAGNI：只做当前 issue 第一版真正需要的部分。

### 1.4 四天版本的判断

如果只做上面的第一版，4 天内完成是可行的。原因是 IOMUX 没有 FIFO、IRQ、DMA、timer，也不需要像 SPI 那样模拟协议。

真正要控制的是边界：

```text
要做：
  - QEMU 设备模型。
  - 0x91105000 起始的 0x800 MMIO 窗口。
  - 32-bit 寄存器保存和读回。
  - reset 默认值。
  - qtest。
  - 文档说明限制。

不要做：
  - 不要实现完整 Linux pinctrl framework。
  - 不要为了“看起来完整”编造没有证据的寄存器副作用。
```

如果 reviewer 后续要求更精细的 writable mask 或 reset value，再基于 TRM 补。第一版先让地址窗口从 unimplemented 变成可验证设备。

## 阶段 2：查资料和整理证据

建议用第 1 天剩余时间完成。这一阶段不要急着写代码。先把资料和最小行为表整理出来。设备模型不是凭感觉写的，应该能回答：

```text
这个寄存器在哪里？
默认值是什么？
guest 写它时我们保存什么？
guest 读它时应该返回什么？
为什么这个行为足够支持当前目标？
```

### 2.1 本仓库必读文件

先读这些本地文件：

```text
hw/riscv/k230.c
include/hw/riscv/k230.h
tests/qtest/k230-wdt-test.c
hw/watchdog/k230_wdt.c
include/hw/watchdog/k230_wdt.h
docs/system/riscv/k230.rst
tests/qtest/meson.build
hw/watchdog/Kconfig
hw/watchdog/meson.build
hw/riscv/Kconfig
```

每个文件看什么：

| 文件 | 看什么 |
| --- | --- |
| `hw/riscv/k230.c` | K230 内存图、IOMUX base/size、当前 unimplemented device 怎么创建。 |
| `include/hw/riscv/k230.h` | `K230SoCState` 怎么保存子设备，K230 设备枚举怎么组织。 |
| `hw/watchdog/k230_wdt.c` | 一个 K230 专属设备模型怎么写 read/write/reset/realize。 |
| `include/hw/watchdog/k230_wdt.h` | 设备 state、寄存器 offset、bit mask 怎么放头文件。 |
| `tests/qtest/k230-wdt-test.c` | K230 qtest 怎么启动 machine、怎么读写 MMIO。 |
| `tests/qtest/meson.build` | 新 qtest 怎么加入构建。 |
| `docs/system/riscv/k230.rst` | 新增设备支持后文档怎么描述。 |

不需要一开始完全看懂 WDT 的 timer/interrupt 逻辑。做 IOMUX 时最重要的是学这几个模式：

```text
read 函数怎么按 offset 返回值
write 函数怎么按 offset 保存值
reset 函数怎么恢复默认值
MemoryRegionOps 怎么注册
SysBusDevice 怎么映射 MMIO
qtest 怎么验证寄存器
```

### 2.2 SDK/Linux/U-Boot 里已经确认的证据

本地 SDK 路径：

```text
/home/flamboy/k230_sdk
```

目前已经确认，IOMUX 不是完全没人用。

U-Boot 证据最强：

```text
src/little/uboot/arch/riscv/dts/k230.dtsi
  iomux@91105000
  compatible = "pinctrl-single"
  reg = <0x0 0x91105000 0x0 0x10000>
  pinctrl-single,register-width = <32>

src/little/uboot/arch/riscv/dts/k230_evb.dtsi
  &iomux
  pinctrl-single,pins = <offset value ...>

src/little/uboot/arch/riscv/dts/k230_fpga.dts
  uart0_pins / mmc0_pins / spi0_pins / spi1_pins
```

这说明 U-Boot 的 `pinctrl-single` 驱动会按 DTS 里的 `<offset, value>` 写 IOMUX 寄存器。对 QEMU 来说，这是一条很好的后续集成验证路径：

```text
U-Boot DTS
  -> pinctrl-single driver
  -> 写 0x91105000 + offset
  -> QEMU IOMUX 模型保存寄存器值
```

Linux 证据弱一些，但也存在：

```text
src/little/linux/drivers/i2c/busses/i2c-designware-master.c
  #define IOMUX_BASE_ADDR (0x91105000U)
  ioremap(IOMUX_BASE_ADDR, 0x1000)
  readl/writel(iomux_regs + gpio * 4)
```

这条路径用于 I2C bus recovery，会临时把 SCL/SDA 对应 pin 切到 GPIO 再恢复。不过当前 K230 Linux DTS 里没有看到标准 `iomux@91105000` / `pinctrl-single` 节点，也没有明显的 `scl-gpios` / `sda-gpios` 默认配置，所以它可能是“代码存在，但默认板级 DTS 不一定触发”。

因此第一版验证优先级是：

```text
qtest 最高优先级。
tiny guest MMIO 测试可选。
U-Boot pinctrl-single 访问作为后续加分验证。
Linux boot 不是第一版必需条件。
```

### 2.3 外部资料应该看什么

优先级从高到低：

1. K230 Technical Reference Manual 的 IOMUX/Pinctrl 章节。
   - [K230 Technical Reference Manual V0.3.1 PDF](https://github.com/revyos/external-docs/blob/master/K230/en-us/K230_Technical_Reference_Manual_V0.3.1_20241118.pdf)
   - 本地 QEMU 内存图对应：`hw/riscv/k230.c` 里的 `K230_DEV_IOMUX = 0x91105000, size 0x800`。
2. K230 SDK 里的 DTS 和 board 初始化代码。
   - [Linux `k230.dtsi`](https://github.com/kendryte/k230_sdk/blob/main/src/little/linux/arch/riscv/boot/dts/kendryte/k230.dtsi)
   - [U-Boot `k230.dtsi`](https://github.com/kendryte/k230_sdk/blob/main/src/little/uboot/arch/riscv/dts/k230.dtsi)
   - [U-Boot `k230_evb.dtsi`](https://github.com/kendryte/k230_sdk/blob/main/src/little/uboot/arch/riscv/dts/k230_evb.dtsi)
   - [U-Boot K230 EVB board init](https://github.com/kendryte/k230_sdk/blob/main/src/little/uboot/board/canaan/k230_evb/board.c)
   - [U-Boot CanMV board init](https://github.com/kendryte/k230_sdk/blob/main/src/little/uboot/board/canaan/k230_canmv/board.c)
3. K230 SDK Linux pinctrl 相关驱动。
   - [Linux generic `pinctrl-single.c`](https://github.com/kendryte/k230_sdk/blob/main/src/little/linux/drivers/pinctrl/pinctrl-single.c)
   - 当前 SDK 里没有看到专门命名为 `pinctrl-k230.c` 的 Linux 驱动；K230 相关证据主要来自 DTS、通用 `pinctrl-single` binding，以及个别驱动直接访问 `0x91105000` 的代码路径。
4. U-Boot 里和 pinmux、board init 有关的代码。
   - [U-Boot generic `pinctrl-single.c`](https://github.com/kendryte/k230_sdk/blob/main/src/little/uboot/drivers/pinctrl/pinctrl-single.c)
   - [K230 EVB pinctrl binding header](https://github.com/kendryte/k230_sdk/blob/main/src/little/uboot/include/dt-bindings/pinctrl/k230_evb.h)
   - [CanMV 01Studio board init](https://github.com/kendryte/k230_sdk/blob/main/src/little/uboot/board/canaan/k230_canmv_01studio/board.c)
   - [CanMV DongshanPI board init](https://github.com/kendryte/k230_sdk/blob/main/src/little/uboot/board/canaan/k230_canmv_dongshanpi/board.c)

搜索关键词：

```text
0x91105000
91105000
iomux
pinctrl
pinmux
pad
pull
drive
schmitt
slew
func
fsel
```

如果在 SDK/Linux/U-Boot 里找不到更多明确 IOMUX driver，也不代表不能做第一版。第一版仍然可以基于 TRM、QEMU 内存图和当前 SDK 证据做“寄存器 bank 模型”，但文档里要诚实说明：

```text
当前模型提供 IOMUX 配置寄存器兼容，pin function 设置只保存在寄存器状态中。
```

### 2.4 要整理的行为表

开始写代码前，先建一张表。可以先放在笔记里，不一定提交。

```text
offset | name | reset value | read behavior | write behavior | why enough
```

示例：

```text
0x000 | PIN0_CFG | 0x00000000 | return regs[0] | save 32-bit value | pin config register, readback enough
0x004 | PIN1_CFG | 0x00000000 | return regs[1] | save 32-bit value | pin config register, readback enough
...
```

如果 TRM 明确某些 bit 是 reserved，第一版可以先保存整个 32-bit 值，也可以按 TRM mask 掉 reserved bit。怎么选取决于证据：

- 如果 TRM 清楚写了 writable bits，优先 mask。
- 如果资料不清楚，先保存 32-bit，并在注释或文档里说明这是兼容性模型。

新手第一版更推荐：

```text
先做简单 register bank。
不要过早精细化每个 bit。
后续如果 reviewers 要求，再根据 TRM 增加 mask。
```

### 2.5 怎么判断资料够不够

满足这些条件，就可以进入阶段 3：

```text
1. 知道 IOMUX base 是 0x91105000。
2. 知道 size 是 0x800。
3. 知道第一版只做配置寄存器读写保存。
4. 知道 reset 默认值怎么处理，至少能说明为什么用 0。
5. 知道 qtest 要测哪些行为。
6. 知道文档要声明哪些不支持行为。
```

如果 TRM 暂时没看完，也可以先写代码骨架。但不要在没有证据的情况下编造复杂寄存器含义。

## 阶段 3：实现和验证

建议用第 2 到第 4 天完成。这一阶段才开始写代码。

### 3.1 推荐文件布局

IOMUX/Pinctrl 更接近 SoC misc/config block，不是 UART/SPI/SD 那样的数据外设。建议放在 `hw/misc`：

```text
hw/misc/k230_iomux.c
include/hw/misc/k230_iomux.h
```

需要改的构建文件：

```text
hw/misc/Kconfig
hw/misc/meson.build
hw/riscv/Kconfig
```

需要改的 machine 文件：

```text
hw/riscv/k230.c
include/hw/riscv/k230.h
```

需要新增测试：

```text
tests/qtest/k230-iomux-test.c
tests/qtest/meson.build
```

需要更新文档：

```text
docs/system/riscv/k230.rst
```

### 3.2 设备模型应该长什么样

第一版 state 可以非常简单：

```c
typedef struct K230IomuxState {
    SysBusDevice parent_obj;
    MemoryRegion mmio;
    uint32_t regs[K230_IOMUX_MMIO_SIZE / 4];
} K230IomuxState;
```

核心思路：

```text
read:
  检查 size 是否是 4。
  检查 offset 是否在范围内。
  返回 regs[offset / 4]。

write:
  检查 size 是否是 4。
  检查 offset 是否在范围内。
  保存 value 到 regs[offset / 4]。

reset:
  把 regs 清零，或者按 TRM 默认值初始化。
```

这就是最小寄存器 bank。

### 3.3 为什么不是直接在 k230.c 里写 MemoryRegion

可以直接在 `k230.c` 里写，但不推荐。

原因：

- `k230.c` 应该负责板级组装：CPU、PLIC、UART、WDT、IOMUX 怎么接起来。
- IOMUX 自己的寄存器行为应该放到独立设备文件。
- 后续如果要增强 IOMUX，不会让 `k230.c` 越来越大。

这符合单一职责：

```text
k230.c：负责把设备放到 K230 地址空间。
k230_iomux.c：负责 IOMUX 寄存器语义。
```

### 3.4 Kconfig 和 meson 怎么接

大致思路：

在 `hw/misc/Kconfig` 新增：

```text
config K230_IOMUX
    bool
```

在 `hw/misc/meson.build` 新增：

```meson
system_ss.add(when: 'CONFIG_K230_IOMUX', if_true: files('k230_iomux.c'))
```

在 `hw/riscv/Kconfig` 的 `config K230` 里新增：

```text
select K230_IOMUX
```

这样启用 K230 machine 时，IOMUX 设备模型会被编进来。

### 3.5 k230.c 怎么接

当前是：

```c
create_unimplemented_device("iomux", memmap[K230_DEV_IOMUX].base,
                            memmap[K230_DEV_IOMUX].size);
```

目标是替换成：

```text
在 K230SoCState 里加 K230IomuxState iomux;
在 k230_soc_init() 里 object_initialize_child(...)
在 k230_soc_realize() 里 sysbus_realize(...)
在 k230_soc_realize() 里 sysbus_mmio_map(..., 0x91105000)
删除原来的 create_unimplemented_device("iomux", ...)
```

可以参考 WDT 在 K230 里的接法：

```text
include/hw/riscv/k230.h 里有 K230WdtState wdt[2]
hw/riscv/k230.c 里 object_initialize_child(...)
hw/riscv/k230.c 里 sysbus_realize(...)
hw/riscv/k230.c 里 sysbus_mmio_map(...)
```

IOMUX 没有中断，所以比 WDT 简单，不需要 `sysbus_connect_irq()`。

### 3.6 qtest 应该测什么

新增 `tests/qtest/k230-iomux-test.c`。

建议测试项：

```text
1. reset_default
   读几个代表 offset，确认默认值。

2. register_read_write
   写 0x91105000 + 0x00，读回相同值。
   写 0x91105000 + 0x04，读回相同值。

3. register_isolation
   写 offset 0x00 不影响 offset 0x04。

4. high_offset
   写窗口末尾附近的合法 offset，例如 0x7fc。

5. reset_after_write
   写寄存器后触发 qtest reset，再读回默认值。
```

测试不需要 Linux，也不需要 SDK 镜像。

qtest 重点是证明：

```text
MMIO 地址正确。
寄存器可写可读。
不同 offset 独立。
reset 行为明确。
```

### 3.7 第一版验证命令

构建：

```bash
ninja -C "build" "qemu-system-riscv64"
```

跑现有 K230 WDT 测试，确认没破坏已有行为：

```bash
"build/pyvenv/bin/meson" test -C "build" "qtest-riscv64/k230-wdt-test" --print-errorlogs
```

跑新 IOMUX 测试：

```bash
"build/pyvenv/bin/meson" test -C "build" "qtest-riscv64/k230-iomux-test" --print-errorlogs
```

确认机器仍存在：

```bash
"build/qemu-system-riscv64" -machine help | rg "^k230"
```

如果需要看未实现访问：

```bash
timeout 3s "build/qemu-system-riscv64" \
  -machine "k230" \
  -bios "none" \
  -display "none" \
  -serial "null" \
  -monitor "none" \
  -d "unimp,guest_errors" \
  -D "k230-unimp.log" \
  -S
```

`-S` 表示 CPU 创建后暂停，不真正运行 guest。这个命令只能证明 machine 能创建，不能证明 Linux 能 boot。

### 3.8 文档要写什么

更新 `docs/system/riscv/k230.rst` 时，不要夸大支持范围。

建议描述：

```text
* IOMUX/Pinctrl configuration register window
```

并在限制说明里写清楚：

```text
The IOMUX model provides a configuration register bank for software read/write
compatibility. Pin function settings are retained in the IOMUX registers but
are not propagated to other emulated peripherals.
```

中文理解就是：

```text
当前支持配置寄存器读写。
pin function 设置会保存在 IOMUX 寄存器里。
不会进一步改变 UART/SPI/GPIO 等外设模型的连接关系。
```

### 3.9 第一版 PR 自查清单

提交 PR 前至少满足：

```text
代码：
  - 有独立 k230_iomux.c。
  - 有独立 k230_iomux.h。
  - Kconfig/meson 接入正确。
  - k230.c 不再对 IOMUX 创建 unimplemented device。
  - K230SoCState 保存 IOMUX 子设备。

测试：
  - 新增 k230-iomux-test。
  - 覆盖 reset 默认值。
  - 覆盖读写保存。
  - 覆盖不同寄存器独立。
  - 覆盖末尾合法 offset。
  - 现有 k230-wdt-test 仍通过。

文档：
  - k230.rst 把 IOMUX 加到 supported devices 或 supported register blocks。
  - 明确这是寄存器兼容模型。

证据：
  - 记录构建命令。
  - 记录 qtest 结果。
  - 记录当前没有跑 Linux，说明第一版以 qtest 为主验证。
```

## 新手常见误区

### 误区 1：必须先跑 Linux 才能做 IOMUX

不需要。

IOMUX 第一版是寄存器 bank，可以先用 qtest 验证。Linux boot 是加分项，不是第一版必须项。

如果做 CMU、DDRC、QSPI、SDHCI，Linux/U-Boot 证据会更重要。但 IOMUX 适合先用 qtest 完成入门。

### 误区 2：必须完整描述所有 pin function

不需要。

完整 pinctrl 资料整理会涉及：

```text
每个 pin 的功能枚举
每个 bit 的精确含义
Linux pinctrl binding
```

这些可以作为后续增强资料，不是第一版必需条件。

### 误区 3：读 TRM 时必须全懂

不需要。

第一遍只抓三件事：

```text
base/size
寄存器 offset
reset value 和 writable bits
```

不懂的复杂字段先记录，不要硬实现。

### 误区 4：看到 WDT 很复杂，就觉得 IOMUX 也很复杂

WDT 复杂是因为它有 timer、interrupt、reset action。

IOMUX 第一版没有 timer，没有 IRQ，没有 DMA，没有 FIFO。它更像：

```text
一段 MMIO-backed uint32_t regs[]
```

所以它适合作为入门题。

## 推荐 4 天节奏

### 第 1 天：读代码、定范围、写行为表

任务：

```text
1. 读 hw/riscv/k230.c，标出 IOMUX base/size 和 unimplemented 位置。
2. 读 tests/qtest/k230-wdt-test.c，理解 qtest 怎么读写 MMIO。
3. 读 hw/watchdog/k230_wdt.c 的 read/write/reset/type init。
4. 读 SDK 里的 U-Boot pinctrl-single DTS 证据。
5. 搜 TRM 的 IOMUX/Pinctrl 章节。
6. 建第一版 offset 行为表。
```

产出：

```text
一张 IOMUX 行为表。
一段第一版范围说明。
一段 SDK/U-Boot/Linux 使用证据说明。
```

第 1 天结束时应该能回答：

```text
IOMUX base/size 是多少？
第一版支持什么、不支持什么？
为什么 U-Boot 会访问这个窗口？
qtest 要测哪些点？
```

### 第 2 天：写最小模型并接入 K230 machine

任务：

```text
1. 新增 k230_iomux.c/h。
2. 接 Kconfig/meson。
3. 修改 K230SoCState。
4. 替换 k230.c 里的 unimplemented IOMUX。
5. 确保 QEMU 能编译。
6. 确认 k230 machine 还能创建。
```

产出：

```text
build/qemu-system-riscv64 能编译。
k230 machine 能创建。
```

第 2 天结束时不要求 qtest 已完成，但至少要能编译过。如果编译不过，先修编译问题，不要继续堆测试。

### 第 3 天：写 qtest，跑验证

任务：

```text
1. 新增 k230-iomux-test.c。
2. 加入 tests/qtest/meson.build。
3. 跑新测试。
4. 跑现有 k230-wdt-test。
5. 修掉测试暴露的问题。
```

产出：

```text
k230-iomux-test 通过。
k230-wdt-test 仍通过。
```

第 3 天结束时，代码应该已经基本可提交。不要在这一天临时扩大范围。

### 第 4 天：补文档、整理证据、收敛 PR

任务：

```text
1. 更新 docs/system/riscv/k230.rst。
2. 在本学习文档或 PR 描述里写清楚 SDK/U-Boot/Linux 证据。
3. 对照 TRM 补必要 reset value 或 writable mask；没有证据就保持 register bank。
4. 收敛代码风格，不做无关重构。
5. 重新跑完整验证命令。
6. 整理 PR 描述和验证日志。
```

产出：

```text
一个范围小、证据清楚、qtest 通过的 PR。
```

4 天版本的最低合格线：

```text
必须有：
  - QEMU 能编译。
  - K230 IOMUX 不再是 unimplemented device。
  - qtest 覆盖 reset/read/write/isolation/high offset。
  - k230-wdt-test 仍通过。
  - 文档说明支持和限制。

可以没有：
  - Linux boot。
  - U-Boot 完整启动验证。
  - 完整 TRM bit 级精确建模。
```

## 当前你下一步该做什么

不要先写代码。先完成这三个动作：

```text
1. 打开 hw/riscv/k230.c，找到 K230_DEV_IOMUX 和 create_unimplemented_device("iomux")。
2. 打开 tests/qtest/k230-wdt-test.c，照着它写出 k230-iomux-test 的测试清单。
3. 查 TRM 或 SDK，确认 IOMUX 寄存器是否就是 0x800 大小的连续配置寄存器窗口。
```

完成后，再进入实现。

第一版成功的标志不是“模拟得像真芯片”，而是：

```text
这个地址窗口有明确行为。
行为可以被 qtest 证明。
文档诚实说明支持和不支持什么。
代码结构能被后续增强。
```
