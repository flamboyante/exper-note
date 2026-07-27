# QEMU Camp SoC 方向学习计划

## 当前执行版

这份文档是早期学习计划。当前按更激进的 7 天完成版执行：

- `soc-7day-plan.md`: SoC 方向 7 天完成计划
- `soc-day1-whitepaper.md`: Day 1 白皮书，明天优先学习这份

## 背景

当前目标不是系统性学完整个 QEMU，而是在较短时间内形成一个能讲清楚、能验证、能写进简历的嵌入式项目经验。

个人基础：

- 有 Linux 驱动基础
- 有 MCU 外设和寄存器经验
- 需要重点补 QEMU 设备模型、MMIO 建模、QTest 自动化验证

推荐主线：

```text
环境和测试框架
-> board / memory region
-> GPIO
-> GPIO IRQ
-> WDT
-> SPI controller
-> SPI Flash
-> 项目总结和简历表达
```

## 总目标

最终要能清楚表达：

> 基于 QEMU 实现 RISC-V G233 虚拟 SoC 的部分外设模型，完成 GPIO、WDT、SPI Flash 等 MMIO 设备建模，支持寄存器读写、中断、虚拟定时器、SPI 片选和 Flash 状态机，并通过 QTest 自动化验证设备行为。

## 14 天最小有效计划

### 第 1 天：环境和仓库地图

目标：

- 跑通 configure/build/test-soc 的基本流程
- 找到 SoC 方向核心文件

重点文件：

- [Makefile.camp](../Makefile.camp)
- [hw/riscv/g233.c](../hw/riscv/g233.c)
- [tests/gevico/qtest/test-board-g233.c](../tests/gevico/qtest/test-board-g233.c)
- [tests/gevico/qtest/test-gpio-basic.c](../tests/gevico/qtest/test-gpio-basic.c)
- [tests/gevico/qtest/test-gpio-int.c](../tests/gevico/qtest/test-gpio-int.c)
- [tests/gevico/qtest/test-wdt-timeout.c](../tests/gevico/qtest/test-wdt-timeout.c)
- [tests/gevico/qtest/test-spi-jedec.c](../tests/gevico/qtest/test-spi-jedec.c)

需要理解：

- QTest 是宿主机测试，不需要先写 guest 程序
- 测试通过 MMIO 地址直接读写虚拟设备寄存器
- 不优先改测试，先让设备模型符合测试期望

### 第 2 天：QEMU MMIO 设备模型

目标：

- 搞懂 QTest 写 MMIO 地址后，如何进入设备回调

核心链路：

```text
qtest_writel(addr, value)
-> QEMU address space
-> MemoryRegion
-> MemoryRegionOps.write
-> 设备状态结构体
```

类比 Linux 驱动：

```text
Linux driver: ioremap + readl/writel
QEMU device: MemoryRegionOps.read/write
```

当天产出：

- 能画出 G233 的简化地址空间
- 能指出某个寄存器地址对应哪个设备
- 能说明 `read/write` 回调为什么会被调用

### 第 3-5 天：GPIO basic

目标：

- 实现 GPIO 基础寄存器行为
- 先不处理复杂中断

重点能力：

- 方向寄存器
- 输入寄存器
- 输出寄存器
- bit 独立控制
- reset 默认值
- 无效 bit 屏蔽

建议做法：

1. 先读 `test-gpio-basic.c`
2. 把每个断言整理成寄存器行为表
3. 再实现设备状态结构体和 read/write 逻辑
4. 每通过一个小测试就记录一次行为

当天产出：

- GPIO basic 测试通过
- 能讲清楚 GPIO 寄存器和内部状态变量的对应关系

### 第 6-7 天：GPIO interrupt

目标：

- 完成 GPIO 中断行为
- 理解 GPIO 到 PLIC 的中断线连接

重点能力：

- interrupt enable
- interrupt pending/status
- edge trigger
- level trigger
- W1C，也就是 write 1 clear
- IRQ line assert/deassert

需要特别注意：

- 状态位是否应该在中断关闭时继续记录
- W1C 只清写 1 的 bit
- 电平中断和边沿中断的状态更新时机不同
- 中断线最终是否正确汇聚到 PLIC

当天产出：

- GPIO interrupt 测试通过
- 形成第一个完整外设模型经验

简历表达：

> 实现 GPIO 控制器 MMIO 模型，支持方向配置、输入输出锁存、多引脚独立控制、边沿/电平中断和 PLIC 中断汇聚。

### 第 8-9 天：WDT

目标：

- 掌握 QEMU timer 和虚拟时间
- 实现 watchdog timeout 行为

重点能力：

- load/current value
- enable
- interrupt enable
- feed key
- timeout flag
- lock 寄存器
- timer callback
- timeout 后触发 IRQ

当天产出：

- WDT 测试通过
- 能说明 QEMU 里定时器如何驱动外设状态变化

简历表达：

> 实现 WDT 虚拟外设，支持超时计数、喂狗、寄存器锁定、超时状态位和中断触发。

### 第 10-13 天：SPI Flash 最小闭环

目标：

- 完成一个控制器 + 外部器件协议的完整模型

重点能力：

- SPI controller 状态机
- TX/RX buffer
- TXE/RXNE 状态位
- chip select
- JEDEC ID
- read
- page program
- sector erase
- RX overrun

建议理解模型：

```text
QTest 写 SPI 数据寄存器
-> SPI 控制器收到一个 byte
-> 根据 CS 选择 Flash
-> Flash 根据当前命令状态返回数据
-> SPI 更新 RXNE/TXE/错误状态
-> QTest 读取结果
```

需要特别注意：

- Flash 命令通常是多阶段状态机
- 地址阶段和数据阶段要分开记录
- CS 切换可能意味着一次事务结束
- RXNE 未清时继续接收可能触发 overrun

当天产出：

- SPI JEDEC/read/CS/overrun 相关测试尽量通过
- 能讲清楚 SPI 控制器和 Flash 设备如何协作

简历表达：

> 实现 SPI 控制器与 Flash 设备互联模型，支持 JEDEC ID、数据读取、页编程、扇区擦除、多片片选、RX overrun 检测和中断驱动传输。

### 第 14 天：项目总结

目标：

- 把训练营实验整理成可复盘、可面试表达的项目

必须整理：

- 项目背景
- 你实现了哪些外设
- MMIO 如何建模
- IRQ 如何接到 PLIC
- QEMU timer 如何模拟 WDT
- SPI Flash 状态机如何设计
- 遇到的一个具体 bug 和调试过程
- 测试如何验证行为

建议最终项目描述：

> 基于 QEMU 的 RISC-V G233 虚拟 SoC 外设建模项目。实现 GPIO、WDT、SPI Flash 等外设的 MMIO 寄存器模型，支持中断、虚拟定时器、片选和外部 Flash 协议状态机，并使用 QTest 构造寄存器级自动化测试，验证设备行为和边界条件。

## 7 天压缩计划

如果时间非常紧，只做 GPIO + WDT：

```text
第 1 天：环境 + qtest + 仓库地图
第 2 天：MMIO 设备模型
第 3-4 天：GPIO basic
第 5 天：GPIO interrupt
第 6 天：WDT
第 7 天：整理项目说明
```

这个版本也能形成有效项目经验：

> 基于 QEMU 实现 RISC-V SoC 的 GPIO/WDT 外设模型，支持 MMIO 寄存器访问、中断触发、虚拟定时器和 QTest 自动化验证。

## 学习时的固定工作法

每个外设都按同一套流程走：

```text
读测试
-> 提取寄存器行为
-> 设计状态结构体
-> 实现 read/write
-> 接 timer/irq
-> 跑单项测试
-> 记录 bug 和修复原因
```

不要一开始通读 QEMU。优先跟着测试走，测试需要什么行为，就实现什么行为。

## 当前优先级

最高优先级：

- SoC
- GPIO
- WDT
- SPI Flash

暂缓：

- CPU 自定义指令
- GPGPU
- Rust 方向
- Linux boot 进阶

原因：

- SoC 与 Linux 驱动、MCU 外设基础最接近
- GPIO/WDT/SPI 能最快形成项目闭环
- QEMU 设备模型、MMIO、IRQ、timer、QTest 都是换工作时更容易讲清楚的经验
