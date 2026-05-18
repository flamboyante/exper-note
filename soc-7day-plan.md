# QEMU Camp SoC 七天完成计划

## 目标

七天内完成 QEMU Camp 2026 专业阶段 SoC 方向 10 个基础测题，并形成一套能在面试和工作中讲清楚的 QEMU 设备模型开发方法。

这不是泛学 QEMU。主线是把已有的外设和寄存器经验翻译成 QEMU 的实现语言：

```text
真实 SoC 外设寄存器
-> QEMU 设备状态结构体
-> MemoryRegionOps read/write
-> qemu_set_irq / PLIC
-> QEMU virtual clock / timer
-> QTest 自动化验证
```

## 官方资料入口

- QEMU Camp Tutorial: https://qemu.gevico.online/
- Machine model: https://qemu.gevico.online/tutorial/2025/ch2/sec4/c-model/machine-model/
- QEMU 外设建模流程: https://qemu.gevico.online/tutorial/2026/ch2/qemu-hw/
- QTest: https://qemu.gevico.online/tutorial/2025/ch2/sec6/qtest/
- SoC 实验手册: https://qemu.gevico.online/exercise/2026/stage1/soc/g233-exper-manual/
- G233 SoC 硬件手册: https://qemu.gevico.online/exercise/2026/stage1/soc/g233-datasheet/

公开站点能确认训练营有讲义、练习和录播课程形式，但目前没有找到稳定公开视频回放入口。执行时以官方讲义、本地测试和本地代码为准。

## 每日固定流程

每天都按同一套流程走：

```text
1. 看官方资料 30-60 分钟，只看当天需要的概念
2. 读当天测试文件，逐条提取断言
3. 写寄存器行为表
4. 实现最小语义
5. 跑单项测试
6. 根据 error log 修正
7. 晚上记录 10-20 行复盘
```

原则：

- KISS: 只实现测试能证明的行为。
- YAGNI: 不提前做完整硬件仿真。
- DRY: 相同的 MMIO/IRQ/timer 更新路径要收敛成函数。
- SOLID: 外设模型尽量独立文件，板级文件只做实例化、地址映射和中断连线。

## Day 1: 环境、板卡、QEMU 设备模型入口

目标：

- 跑通 configure/build/test-soc。
- 建立 SoC 10 题失败矩阵。
- 理解 `-machine g233` 如何进入 `hw/riscv/g233.c`。
- 理解 QTest 写 MMIO 后如何进入设备 `read/write` 回调。

看：

- Machine model
- QEMU 外设建模流程
- QTest

读：

- `README_zh.md`
- `Makefile.camp`
- `hw/riscv/g233.c`
- `include/hw/riscv/g233.h`
- `tests/gevico/qtest/meson.build`
- `tests/gevico/qtest/test-board-g233.c`

做：

```bash
make -f Makefile.camp configure
make -f Makefile.camp build
make -f Makefile.camp test-soc
```

产出：

- `test-board-g233` 到 `test-spi-overrun` 的失败矩阵。
- G233 简化地址空间表。
- 外设接入方案：新增外设文件 + Meson/Kconfig + `g233.c` 实例化。

验收：

- 能解释 `qtest_writel(addr, value)` 到 `MemoryRegionOps.write` 的路径。
- 能指出 DRAM、PLIC、CLINT、UART 在 `g233.c` 中如何创建或映射。
- 能说明后续 GPIO/WDT/SPI 应该挂在哪个地址、接哪个 PLIC IRQ。

详细执行见 `soc-day1-whitepaper.md`。

## Day 2: GPIO basic

目标：通过 `qtest-riscv64/test-gpio-basic`。

看：

- QEMU 外设建模流程里的 MMIO 部分。
- 可参考 `hw/gpio/mpc8xxx.c`、`hw/gpio/pl061.c`。

读：

- `tests/gevico/qtest/test-gpio-basic.c`

实现：

```text
GPIO_BASE = 0x10012000

0x00 GPIO_DIR
0x04 GPIO_OUT
0x08 GPIO_IN
0x0C GPIO_IE
0x10 GPIO_IS
0x14 GPIO_TRIG
0x18 GPIO_POL
```

最小语义：

- reset 全 0。
- `DIR` 可读写。
- `OUT` 可读写。
- `IN = OUT & DIR`。
- 中断相关寄存器先保存值，复杂中断逻辑 Day 3 做。

建议文件：

```text
hw/gpio/g233_gpio.c
include/hw/gpio/g233_gpio.h
hw/gpio/meson.build
hw/riscv/g233.c
```

验收：

```bash
cd build
./pyvenv/bin/meson test --no-rebuild --print-errorlogs qtest-riscv64/test-gpio-basic
```

记录：

- GPIO 寄存器行为表。
- `read/write` 回调如何映射 offset。
- `GPIO_IN` 为什么不能简单等于 `GPIO_OUT`，而要考虑方向位。

## Day 3: GPIO IRQ + PLIC

目标：通过 `qtest-riscv64/test-gpio-int`。

看：

- QEMU 外设建模流程里的中断线部分。
- `hw/riscv/g233.c` 里 PLIC 创建和 IRQ 连接方式。

读：

- `tests/gevico/qtest/test-gpio-int.c`

实现：

```text
GPIO_IE
GPIO_IS
GPIO_TRIG
GPIO_POL
PLIC IRQ 2
```

关键语义：

- edge rising/falling: 跳变时置 `IS`。
- level high/low: 电平满足时置 `IS`，不满足时清。
- 测试要求 `IE=0` 时不置 `IS`。
- `GPIO_IS` write 1 clear。
- 有效 `IS & IE` 非 0 时 `qemu_set_irq(irq, 1)`，清空后拉低。

验收：

```bash
cd build
./pyvenv/bin/meson test --no-rebuild --print-errorlogs qtest-riscv64/test-gpio-int
```

记录：

- GPIO IRQ 到 PLIC pending 的路径。
- PLIC priority、enable、threshold、claim、complete 的测试观察。

## Day 4: PWM + WDT

目标：通过 `test-pwm-basic` 和 `test-wdt-timeout`。

看：

- QEMU 时钟系统。
- 可参考 `hw/watchdog/wdt_ib700.c`、`hw/timer/sifive_pwm.c`。

读：

- `tests/gevico/qtest/test-pwm-basic.c`
- `tests/gevico/qtest/test-wdt-timeout.c`

PWM 实现：

```text
PWM_BASE = 0x10015000
PWM_GLB
4 个 channel: CTRL / PERIOD / DUTY / CNT
```

PWM 最小语义：

- `CTRL.EN`、`CTRL.POL` 可读写。
- `GLB` 镜像 channel enable。
- `CNT` 随虚拟时间增长。
- 周期完成置 `DONE`。
- `DONE` write 1 clear。

WDT 实现：

```text
WDT_BASE = 0x10010000
CTRL / LOAD / VAL / KEY / SR
PLIC IRQ 4
```

WDT 最小语义：

- `EN` 后 `VAL` 随虚拟时间减少。
- feed key 后 `VAL` reload。
- timeout 后 `SR.TIMEOUT = 1`。
- `SR` write 1 clear。
- lock 后 `CTRL` 写入无效。
- `INTEN` 时 timeout 拉 IRQ 4。

验收：

```bash
cd build
./pyvenv/bin/meson test --no-rebuild --print-errorlogs qtest-riscv64/test-pwm-basic
./pyvenv/bin/meson test --no-rebuild --print-errorlogs qtest-riscv64/test-wdt-timeout
```

记录：

- QEMU timer 的创建、重装、取消。
- 为什么测试只需要可验证的虚拟计时行为，不要求真实硬件频率。

## Day 5: SPI 控制器基础 + JEDEC

目标：通过 `qtest-riscv64/test-spi-jedec`。

看：

- QEMU 外设建模流程里的状态机和 IRQ 更新方法。
- 可参考 `hw/ssi/sifive_spi.c`、`hw/ssi/stm32f2xx_spi.c`。

读：

- `tests/gevico/qtest/test-spi-jedec.c`

实现：

```text
SPI_BASE = 0x10018000
0x00 SPI_CR1
0x04 SPI_CR2
0x08 SPI_SR
0x0C SPI_DR
```

最小语义：

- reset 后 `TXE=1`。
- 写 `DR` 表示发送 1 byte。
- 写 `DR` 后立即产生接收 byte，置 `RXNE`。
- 读 `DR` 清 `RXNE`。
- `CR2` 选择 CS。
- CS0 支持 JEDEC: `0x9F -> 0xEF 0x30 0x15`。

建议：

- 先把 Flash 做成 SPI 控制器内部子结构，不急着拆成独立外设。
- 用 `cmd/addr/phase/storage` 表示 Flash 状态机。

验收：

```bash
cd build
./pyvenv/bin/meson test --no-rebuild --print-errorlogs qtest-riscv64/test-spi-jedec
```

记录：

- 一次 `spi_transfer_byte()` 的执行路径。
- `TXE/RXNE` 置位和清除规则。

## Day 6: SPI Flash read/program/erase + interrupt

目标：通过 `test-flash-read` 和 `test-flash-read-interrupt`。

读：

- `tests/gevico/qtest/test-flash-read.c`
- `tests/gevico/qtest/test-flash-read-interrupt.c`

实现命令：

```text
0x9F JEDEC ID
0x05 Read Status
0x06 Write Enable
0x03 Read Data
0x02 Page Program
0x20 Sector Erase
```

Flash 语义：

- storage 初始为 `0xFF`。
- Read Status 返回 BUSY/WEL。
- Write Enable 置 WEL。
- Page Program 需要 WEL，只能 `old & new`。
- Sector Erase 需要 WEL，擦成 `0xFF`。
- 事务结束后清理 `phase/cmd`。

SPI interrupt：

- `TXEIE && TXE -> IRQ`。
- `RXNEIE && RXNE -> IRQ`。
- `ERRIE && OVERRUN -> IRQ`。
- SPI PLIC IRQ 按测试和手册确认，通常为 IRQ 5。

验收：

```bash
cd build
./pyvenv/bin/meson test --no-rebuild --print-errorlogs qtest-riscv64/test-flash-read
./pyvenv/bin/meson test --no-rebuild --print-errorlogs qtest-riscv64/test-flash-read-interrupt
```

记录：

- Flash 命令状态机图。
- 一次 page program/read back 的完整过程。

## Day 7: 双 CS + overrun + 全量收敛

目标：通过剩余 SPI 边界测试，最后 SoC 10/10。

读：

- `tests/gevico/qtest/test-spi-cs.c`
- `tests/gevico/qtest/test-spi-overrun.c`

实现：

- CS0 / CS1 两片 Flash。
- 每片 Flash 独立 `jedec/status/storage/phase`。
- CS 切换结束当前 transaction。
- CS0 JEDEC `0xEF3015`，CS1 JEDEC `0xEF3016`。
- RXNE 未清又收到新 byte，置 `OVERRUN`。
- `OVERRUN` write 1 clear。
- `ERRIE` 开启时 overrun 拉 IRQ。

验收：

```bash
cd build
./pyvenv/bin/meson test --no-rebuild --print-errorlogs qtest-riscv64/test-spi-cs
./pyvenv/bin/meson test --no-rebuild --print-errorlogs qtest-riscv64/test-spi-overrun
cd ..
make -f Makefile.camp test-soc
```

最终产出：

```text
SoC 10/10 测试结果
一页项目复盘
一个具体 bug 的定位过程
```

复盘模板：

```text
项目背景：
基于 QEMU 实现 RISC-V G233 虚拟 SoC 的 GPIO/PWM/WDT/SPI Flash 外设模型。

核心技术：
MemoryRegionOps 实现 MMIO 寄存器读写；
SysBusDevice 接入 G233 machine；
qemu_set_irq 接入 RISC-V PLIC；
QEMU timer 模拟 WDT/PWM 虚拟时间；
SPI Flash 用命令状态机模拟 JEDEC/read/program/erase/CS/overrun。

验证方式：
使用 QTest 直接读写 MMIO 地址，验证寄存器、状态位、中断 pending、claim/complete 和边界条件。
```

