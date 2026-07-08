# K230 QEMU 训练营交流笔记

记录时间：2026-07-01

本文整理前面对 `gevico/qemu-camp-2026-k230`、K230 SDK/Linux/U-Boot、QEMU `-machine k230` 当前状态，以及 issue 选题策略的关键讨论结论。

## 1. 项目与资料入口

核心仓库：

- 训练营仓库：<https://github.com/gevico/qemu-camp-2026-k230>
- K230 建模母 issue：<https://github.com/gevico/qemu-camp-2026-k230/issues/1>
- QEMU K230 文档：<https://www.qemu.org/docs/master/system/riscv/k230.html>

Kendryte 官方资料：

- K230 SDK：<https://github.com/kendryte/k230_sdk>
- K230 文档：<https://github.com/kendryte/k230_docs>
- SDK 关键路径：
  - `src/little/linux`
  - `src/little/uboot`
  - `src/common/opensbi`
- SDK 常见构建产物路径：
  - `output/k230_canmv_defconfig/images/little-core/`

TRM 参考：

- K230 Technical Reference Manual V0.3.1：<https://github.com/revyos/external-docs/blob/master/K230/en-us/K230_Technical_Reference_Manual_V0.3.1_20241118.pdf>

## 2. 当前本地工程状态

仓库已克隆到：

```text
/home/flamboy/qemu-camp-2026-k230
```

当前本地 `master` 与 QEMU upstream master 对齐：

```text
HEAD: 30e8a06b64
```

关键源码位置：

- K230 machine：`hw/riscv/k230.c`
- K230 SoC 头文件：`include/hw/riscv/k230.h`
- K230 文档：`docs/system/riscv/k230.rst`
- K230 WDT qtest：`tests/qtest/k230-wdt-test.c`
- qtest 注册：`tests/qtest/meson.build`

## 3. QEMU 当前 K230 支持程度

当前 `-machine k230` 是一个“能支撑最小启动实验的板级框架”，不是完整 K230 SoC 模拟。

已支持内容：

- 1 个 C908 little core
- DDR/SRAM/BootROM 基础映射
- PLIC
- CLINT/ACLINT
- 5 个 UART
- 2 个 K230 WDT

仍大量使用 `create_unimplemented_device()` 占位的外设：

- DDRC_CFG
- SD/SDHCI
- QSPI
- Flash memory window
- CMU
- RMU
- IOMUX/Pinctrl
- Timer
- GPIO
- I2C
- SPI
- RTC
- Mailbox/IPCM
- DMA/GSDMA
- PMU/PWR/HI_SYS_CFG 等

`create_unimplemented_device()` 的行为要点：

- 读通常返回 `0`
- 写通常忽略
- 会记录 `LOG_UNIMP`
- 对驱动 probe 来说，有时足够绕过简单访问，有时会因为状态位永远为 `0` 卡死在轮询逻辑

## 4. K230 内存图中几个关键地址

来自当前 `hw/riscv/k230.c`：

```text
DDR RAM:       0x00000000, size 0x80000000
DDRC_CFG:      0x98000000, size 0x02000000
QSPI0:         0x91582000, size 0x00001000
QSPI1:         0x91583000, size 0x00001000
Flash window:  0xC0000000, size 0x08000000
CMU:           0x91100000, size 0x00001000
RMU:           0x91101000, size 0x00001000
IOMUX:         0x91105000, size 0x00000800
Timer:         0x91105800, size 0x00000800
GPIO0:         0x9140B000, size 0x00001000
GPIO1:         0x9140C000, size 0x00001000
```

## 5. 裸机、Linux、OpenSBI/U-Boot/Linux 的现实状态

### 5.1 裸机能不能跑

结论：可以做裸机级实验，但不是完整芯片环境。

适合做的实验：

- MMIO 读写行为验证
- UART 输出
- WDT 寄存器和中断行为
- PLIC/ACLINT 基础中断路径
- 自写小程序访问某个待建模外设的寄存器窗口
- qtest 驱动的寄存器级测试

不适合直接假设的实验：

- 完整 BootROM 启动链路
- 真实 QSPI flash 启动
- 真实 DDR training
- 大核/小核完整异构协作
- 复杂外设数据通路

### 5.2 Linux 能不能跑

结论：可以走裁剪过的 direct Linux boot 或手动 loader/U-Boot 流程，但不是常规真实板级启动链。

当前文档说明的 Linux 限制：

- SDK Linux 使用 T-HEAD C9xx 私有 MAEE 页表属性。
- QEMU 通用 RISC-V MMU 不实现 MAEE。
- 因此 Linux kernel 需要改成标准 RISC-V PTE bits 后才能在 QEMU 下跑。
- DTB 需要从 SDK 产物派生，并禁用当前未模拟的设备，例如 SDHCI。
- bootargs 里通常需要 `console=ttyS0,115200 earlycon=sbi cma=0`。

Linux 当前水准可以理解为：

- 可以验证 little-core Linux boot 的最小路径。
- 依赖派生/裁剪 DTB。
- 依赖绕过或禁用未建模外设。
- 不是拿真实 SDK 镜像原样就完整启动。

### 5.3 常规 OpenSBI + U-Boot + Linux 还差多少

常规真实链路大致是：

```text
BootROM -> U-Boot/SPL 或 SDK boot flow -> storage -> OpenSBI -> Linux
```

当前还差的关键环节：

- DDRC/DDRC_CFG：U-Boot 早期 DDR 初始化可能卡在训练/ready/status 轮询。
- CMU/RMU：时钟、复位状态会影响大量外设初始化。
- IOMUX：板级 pinmux 配置需要能写入/读回。
- QSPI + Flash window：真实 flash 启动或 memory-mapped flash 访问需要。
- SD/SDHCI：如果从 SD 卡路径启动，需要控制器模型。
- DMA/IRQ 等：取决于具体驱动是否启用。

因此“常规 OpenSBI + U-Boot + Linux”不是差一个小寄存器，而是差一组启动关键外设的最小兼容模型。

### 5.4 K230 是否存在 Linux + RTOS 异构

是。K230 是异构 SoC，存在大小核/多运行环境的实际使用场景。SDK 中 little core 走 Linux，另一个核心/运行域可涉及 RTOS 或其他固件协作。

但当前 QEMU `-machine k230` 只建模了 1 个 C908 little core。也就是说，当前工程阶段主要围绕 little-core Linux/SDK 最小启动路径推进，还没有完整覆盖 Linux + RTOS 异构协作。

### 5.5 启动介质、RAM 执行和 OpenSBI 的位置

这里容易把三件事混在一起：

```text
芯片/板子从哪里取启动镜像
QEMU 当前怎么把镜像放进 guest 地址空间
代码最终在哪里执行
```

K230 这类应用 SoC 和常见 MCU 的启动模型不完全一样。

MCU 常见模型是：

```text
内部 Flash 映射到 CPU 地址空间
reset vector 在 Flash 中
CPU 上电后可以直接从 Flash 取指
代码长期在 Flash 中 XIP 执行
数据、栈、堆放 SRAM
```

这里的关键机制是 XIP：

```text
XIP = Execute In Place
```

也就是 Flash 不只是存储介质，也可以是取指执行介质。

K230 / Linux SoC 的典型模型更接近：

```text
上电
  -> 片内 BootROM 执行
  -> BootROM 根据 boot pin/boot 配置选择 SD/eMMC/SPI NOR/SPI NAND/USB 等启动介质
  -> 从启动介质读取下一阶段 loader / SPL / U-Boot
  -> loader 初始化 SRAM/DDR 等运行环境
  -> 后续固件和系统镜像被搬运/解包到 DDR
  -> CPU 跳到 DDR 中执行
```

所以 SD/eMMC/SPI NOR/SPI NAND 主要表示：

```text
非易失存储介质
BootROM 或 U-Boot 从哪里读取镜像
```

不表示 Linux 一直在 SD/SPI/eMMC 上直接取指运行。到 Linux 阶段，kernel 基本一定在 DDR 中执行。

当前 QEMU K230 的简化点正是在这里：

```text
少建模了：外部存储介质 -> BootROM/U-Boot 读取 -> 搬运/解包 -> DDR
```

现在更多是：

```text
QEMU 直接用 -bios / -device loader 把 U-Boot、OpenSBI、Linux、DTB、initrd
放到 guest RAM 的指定地址
  -> QEMU 生成的 BootROM reset vector 跳过去
  -> U-Boot / OpenSBI / Linux 在 RAM/DDR 中执行
```

这没有否认真实硬件的存储启动路径，只是当前还没有完整模拟 SD/QSPI/eMMC/Flash 控制器读取镜像、分区解析、解包搬运等过程。

OpenSBI 在这条链里处在 U-Boot 和 Linux 之间：

```text
BootROM
  -> U-Boot
  -> OpenSBI
  -> Linux kernel
```

OpenSBI 是 RISC-V 的 M-mode 固件。它的作用不是替代 U-Boot，也不是 Linux 本体，而是给 S-mode Linux 提供 SBI 服务，例如：

```text
timer
IPI
hart start/stop
system reset/shutdown
console 或调试相关服务
```

在 K230 QEMU 文档的 U-Boot boot 流程中，OpenSBI 通常以 `fw_jump.bin` 形式存在，并被打成 U-Boot 能 `bootm` 的 uImage：

```text
fw_jump.bin
  -> mkimage
  -> k230-fw-jump.uImage
```

然后 QEMU 用 loader 把它预放到 RAM：

```text
0x0c100000  OpenSBI uImage
0x08200000  Linux Image
0x0a000000  DTB
0x0a100000  initrd
```

U-Boot 执行：

```text
bootm 0xc100000 - 0xa000000
```

大致含义是：

```text
先进入 OpenSBI
OpenSBI 完成 M-mode 固件职责后跳到 Linux
Linux 以 S-mode 运行，并通过 SBI 调用请求 M-mode 服务
```

对当前建模任务来说，重要结论是：

```text
即使 QEMU 暂时绕过了真实 SD/SPI/eMMC 读取过程，
U-Boot 仍然会真实执行早期初始化，
因此仍会访问 IOMUX、CMU、RMU、WDT、UART 等 MMIO 外设。
```

这也是为什么用 RAM 预加载方式依然能验证 IOMUX：被省略的是“从存储介质搬运镜像到 RAM”的路径，不是 U-Boot 的设备初始化逻辑。

## 6. 关于“U-Boot 卡在 DDRC”的理解

老师说“U-Boot 卡在 DDRC”，应理解为：

- U-Boot 早期初始化访问 DDRC/DDRC_CFG 寄存器。
- 它可能写入初始化参数、启动 DDR training/calibration。
- 然后轮询某些 status/ready/done bit。
- 当前 QEMU 对 DDRC_CFG 只是 unimplemented device，读值通常为 `0`。
- 如果 U-Boot 等待某个位变成 `1`，就会无限卡住。

是否容易解决取决于卡点是否单一：

- 如果只是少量状态位轮询，可以做最小 stub，返回 ready/done，比较快。
- 如果 U-Boot 初始化流程依赖大量 DDR PHY/控制器寄存器副作用，就会变成较大的建模/调试任务。

正确调试方式：

```text
1. 复现 U-Boot 卡点。
2. 开启 -d unimp,guest_errors 或加 trace。
3. 找出最后访问的 DDRC offset。
4. 对照 U-Boot 源码和 TRM 判断该 offset 的语义。
5. 只实现当前启动路径真正依赖的最小行为。
```

不要在没有日志和 offset 的情况下猜 DDRC 行为。

## 7. Issue 模板的真实含义

这些 issue 的描述大多是统一模板。真正要做的不是“完整模拟硬件”，而是：

- MMIO 地址必须和 K230 内存图一致。
- 驱动 probe 不能因为该外设卡死。
- 当前启动路径访问到的寄存器要有明确行为。
- 未实现的重要寄存器要有注释说明。
- 可行时添加 qtest 或启动 smoke test。
- 文档记录已支持和未支持行为。

换句话说，训练营目标是把外设从：

```text
地址占位
```

推进到：

```text
足够支持 SDK/Linux boot 和 driver probe 的可测试模型
```

## 8. 当前判断：先打通前置环境

结论：有必要先把基础环境、最小启动、qtest、日志抓取流程打通，再认领复杂外设。

原因很直接：

- 多数 issue 的验收都不是“能编译”即可，而是要证明 SDK/Linux driver probe 不会卡死。
- QEMU 设备模型的正确性主要来自实际访问证据：访问了哪个 base、哪个 offset、读写什么值、是否轮询 status bit。
- 如果没有可重复的 Linux/U-Boot/qtest 验证环境，容易写出“看起来像寄存器模型、但没有启动路径价值”的代码。
- 对 DDRC、CMU、RMU、QSPI、SDHCI、DMA 这类任务，脱离启动日志和驱动源码做静态实现，风险很高。

因此前置目标应该先定义成：

```text
1. 能本地构建 qemu-system-riscv64，并跑 K230 qtest。
2. 能用 -machine k230 启动到最小固件路径，至少能看到串口输出。
3. 能复现 direct Linux boot 或 U-Boot loader boot 的一个路径。
4. 能打开 -d unimp,guest_errors 或 trace，收集未实现 MMIO 访问。
5. 能把每个卡点整理成 offset -> 源码位置 -> 需要的最小行为。
```

这一步本身不是“绕路”，而是后续所有高质量 PR 的共同基础。

## 9. 当前主线 K230 支持边界

本地源码显示，当前 `-machine k230` 的有效支持范围仍然比较窄。

已经有真实模型或复用 QEMU 通用模型的部分：

- 1 个 T-HEAD C908 little core，`hartid-base = 0`。
- DDR RAM 映射到 `0x00000000`，默认大小 `0x80000000`。
- SRAM `0x80200000`，用 RAM region 建模。
- BootROM `0x91200000`，用 ROM region 承载 reset vector。
- PLIC，使用 `sifive_plic_create()`。
- ACLINT/CLINT，使用 QEMU RISC-V ACLINT helper。
- 5 个 UART，核心串口行为用 `serial_mm_init()`，外层 0x1000 窗口仍有 unimplemented 覆盖。
- 2 个 K230 WDT，有独立模型和 qtest。

仍然主要是 `create_unimplemented_device()` 的部分：

- CMU、RMU、IOMUX、Timer、RTC、GPIO、I2C、SPI、QSPI、SD/SDHCI、Flash window。
- DDRC_CFG、PMU、PWR、HI_SYS_CFG、Mailbox/IPCM、DMA、GSDMA。
- ISP、KPU、视频/图像相关大块外设。

需要注意：`hw/riscv/Kconfig` 里 `CONFIG_K230` 选入了 `RISCV_ACLINT`、`SERIAL_MM`、`UNIMP`、`K230_WDT` 等配置，但这不等于所有 K230 外设都已建模。真实边界应以 `hw/riscv/k230.c` 中实例化了什么设备为准。

## 10. Linux 验证怎么分层

不是每个 issue 都必须一开始就完整跑通 Linux，但多数有价值的 issue 最终都应该至少能解释它和 Linux/U-Boot 启动路径的关系。

建议把验证分成三层：

```text
L1: qtest
    验证 MMIO base、寄存器读写、reset default、关键 status bit。

L2: 裸机或固件 smoke test
    验证设备不会破坏 -machine k230 基础启动，必要时验证中断路径。

L3: U-Boot/Linux boot/probe
    验证真实 SDK/U-Boot/Linux 访问路径不会在该外设卡住。
```

低风险寄存器块，例如 IOMUX、HI_SYS_CFG、部分 GPIO，可以先用 L1 站住。启动关键外设，例如 DDRC、CMU、RMU、QSPI、SDHCI、DMA，最好尽快补 L3 证据。

当前已有 qtest 参考：

```text
tests/qtest/k230-wdt-test.c
```

可复用的测试风格：

- `qtest_init("-machine k230")`
- 对固定 MMIO base 做 `qtest_readl()` / `qtest_writel()`
- 使用 `g_assert_cmphex()` / `g_assert_cmpuint()`
- 覆盖读写、reset default、关键状态位

## 11. 25 个子 issue 评价

记录时间：2026-07-01。通过 GitHub issue API 查看时，`#1` 是母 issue，`#2` 到 `#26` 是 25 个子 issue，均为 open。`#8 UART` 已有人认领，`#11 RMU` 已有人认领，其余未见 assignee。

评价维度：

- 含金量：对 K230 bring-up、SDK/Linux boot、QEMU 设备建模能力展示的价值。
- 难度：实现复杂度、资料依赖、调试不确定性。
- 环境依赖：是否强依赖 Linux/U-Boot 真实启动验证。

| Issue | 内容 | 当前状态判断 | 含金量 | 难度 | 环境依赖 | 评价 |
| --- | --- | --- | --- | --- | --- | --- |
| #2 | CPU/C908 hart | 已有 1 个 C908 little core，但 issue 仍 open | 高 | 高 | 中 | CPU 语义和 T-HEAD 特性水很深，不适合作为第一个外设任务。 |
| #3 | DDRC / DDR map | DDR RAM 已映射，DDRC_CFG 仍是占位 | 很高 | 很高 | 很高 | 能解决 U-Boot DDRC 卡点就很有价值，但必须先拿日志和 offset。 |
| #4 | SRAM | 已用 RAM region 建模 | 中 | 低 | 低 | 可能偏收尾和文档/qtest，含金量有限但容易闭环。 |
| #5 | BootROM | 有 BootROM region 和 reset vector | 中 | 中 | 中 | 若做真实 boot flow 会变复杂；若只完善 reset/文档则范围可控。 |
| #6 | PLIC | 已复用 SiFive PLIC | 高 | 中 | 中 | 基础中断路径重要，但可能更多是验证、参数校准和文档化。 |
| #7 | CLINT/ACLINT | 已复用 ACLINT helper | 高 | 中 | 中 | 和时钟、中断、Linux timer 路径相关，适合做验证补强。 |
| #8 | UART | 已有 serial-mm，已有人认领 | 高 | 中 | 中 | console 是所有验证基础；已有部分实现，后续多为兼容性补齐。 |
| #9 | WDT | 已有设备模型和 qtest | 中 | 中 | 低 | 适合学习 QEMU 设备模型，但当前可能更接近完善/修正。 |
| #10 | CMU | 当前占位 | 很高 | 中高 | 高 | 时钟是大量 probe 的前置条件，价值高，但必须用启动日志限定范围。 |
| #11 | RMU | 当前占位，已有人认领 | 高 | 中高 | 高 | reset/status 和 CMU 强相关，适合联动，但避免和认领者冲突。 |
| #12 | IOMUX/Pinctrl | 当前占位 | 高 | 中 | 中 | 很适合做第一个认真 PR：寄存器可读写模型清晰，启动价值也足够。 |
| #13 | SD/SDHCI | 当前占位 | 高 | 高 | 很高 | 存储启动价值高，但控制器兼容性、DMA、IRQ、clock/reset 都要确认。 |
| #14 | QSPI | 当前占位 | 很高 | 高 | 很高 | 启动链路价值高；需要搞清 controller、FIFO、XIP、SPI NOR 关系。 |
| #15 | Flash window | 当前占位 | 中高 | 中 | 中高 | 单做 aperture 含金量一般，和 QSPI/启动镜像联动后价值明显提高。 |
| #16 | GPIO | 当前占位 | 中 | 中 | 中 | 练设备模型不错；若只做寄存器读写，bring-up 推动力有限。 |
| #17 | I2C | 当前占位 | 中 | 中高 | 中 | probe 价值有，但外设从设备和中断行为可能拉大范围。 |
| #18 | SPI | 当前占位 | 中高 | 中高 | 中高 | 比 QSPI略小，但仍要处理 controller/FIFO/flash 或外设连接。 |
| #19 | Timer | 当前占位 | 中高 | 中高 | 高 | Linux 时间源通常已有 ACLINT，但 K230 timer 仍可能影响 SDK 驱动。 |
| #20 | RTC | 当前占位 | 中 | 中 | 中 | 可控度比 SD/QSPI 高，适合中等难度设备模型。 |
| #21 | PMU | 当前占位 | 中高 | 中高 | 高 | 电源管理可能影响启动状态位，但资料和真实行为需要谨慎。 |
| #22 | PWR | 当前占位 | 中高 | 中高 | 高 | 和 PMU/RMU/CMU 相关，容易变成状态兼容模型。 |
| #23 | HI_SYS_CFG | 当前占位 | 中高 | 中 | 中 | 系统配置寄存器块通常适合做最小读写/status 模型，范围可控。 |
| #24 | Mailbox/IPCM | 当前占位 | 高 | 高 | 高 | 和 Linux + RTOS/多核协作相关；当前单 C908 环境下验证难度较高。 |
| #25 | DMA | 当前占位 | 高 | 高 | 很高 | 含金量高，但真实数据搬运、IRQ、descriptor、memory transaction 都复杂。 |
| #26 | GSDMA | 当前占位 | 高 | 高 | 很高 | 和 DMA 类似，可能更依赖 SDK 实际使用路径，不建议无日志开做。 |

## 12. 建议推进顺序

当前最务实的路线不是马上认领最高含金量 issue，而是先建立验证闭环。

建议顺序：

```text
阶段 0：环境与证据链
  - 构建 QEMU
  - 跑 k230-wdt-test
  - 跑 -machine k230 最小启动
  - 准备 SDK/U-Boot/Linux 产物或可替代 smoke test
  - 学会收集 -d unimp,guest_errors 日志

阶段 1：可控且有价值的寄存器块
  - #12 IOMUX/Pinctrl
  - #23 HI_SYS_CFG
  - #16 GPIO 或 #20 RTC

阶段 2：启动关键前置
  - #10 CMU
  - #11 RMU（已有人认领，需先协调）
  - #21 PMU / #22 PWR

阶段 3：存储与启动链路
  - #15 Flash window
  - #14 QSPI
  - #13 SD/SDHCI

阶段 4：高风险高价值
  - #3 DDRC
  - #25 DMA
  - #26 GSDMA
  - #24 Mailbox/IPCM
```

如果只选一个适合作为第一个完整 PR 的任务，当前更推荐 `#12 IOMUX/Pinctrl` 或 `#23 HI_SYS_CFG`。它们比 DDRC/QSPI/SDHCI 更容易控制范围，又比单纯做 SRAM/Flash aperture 更能体现板级 bring-up 价值。

## 13. 后续应补充的证据

后续无论选哪个 issue，都应该补齐以下证据，避免凭感觉建模：

- TRM 对应章节和寄存器表。
- SDK/U-Boot/Linux 中实际访问该外设的源码位置。
- 当前 QEMU 启动日志中的 unimplemented MMIO offset。
- 驱动 probe 或 U-Boot 初始化卡住的具体循环。
- 最小寄存器行为设计表：

```text
offset | name | reset value | read behavior | write behavior | why enough
```

这张表会直接决定 PR 是否站得住。
