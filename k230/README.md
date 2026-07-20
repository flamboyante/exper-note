# K230 QEMU 专题笔记索引

这个目录集中放 K230 QEMU 训练营相关笔记，包括当前 QEMU 支持现状、SDK/TRM 证据链、外设建模计划、启动链路分析和具体 issue 的实现准备。

建议阅读顺序：

```text
先理解当前 K230 machine 支持程度
  -> 再看 QOM / SysBus / MMIO 外设骨架
  -> 再看已完成或准备实现的具体外设
  -> 最后看启动链路和 U-Boot/Linux 验证路线
```

## 总览与背景

- [K230 QEMU 训练营交流笔记](./k230-qemu-camp-notes.md)
  - 记录 K230 当前 QEMU 支持程度、SDK/Linux/U-Boot 现实状态、启动介质、RAM 执行和 OpenSBI 位置。
  - 第一次了解 K230 训练营任务时优先读这一篇。

- [K230 完整启动与 IOMUX 验证](./uboot-iomux-unimplemented-repro.md)
  - 以 `k230-boot-assets` 为默认镜像来源，给出 Yocto direct boot、Buildroot direct boot、U-Boot boot 和 IOMUX 验证命令。
  - 说明本地 `k230_sdk` 与预构建镜像的版本关系、QEMU 适配差异和当前存储启动边界。

- [K230 QEMU 对象模型与外设接线学习笔记](./k230-qemu-object-model-study.md)
  - 解释 K230 machine、SoC、SysBusDevice、MemoryRegion、PLIC/IRQ 接线等基础结构。
  - 适合作为写新 K230 外设前的 QOM/QEMU 代码地图。

- [QEMU 通用基础 MMIO 外设骨架模板](./qemu-basic-mmio-device-template.md)
  - 总结最小 MMIO 外设的文件布局、State、MemoryRegionOps、read/write/reset、Kconfig/meson/qtest 接入方式。
  - 适合 IOMUX、CMU、RMU、简单 SYSCTRL 类寄存器块参考。

## 已推进外设

- [K230 IOMUX/Pinctrl 三阶段入门指南](./k230-iomux-three-stage-guide.md)
  - 记录 IOMUX/Pinctrl 的第一版实现边界、证据链、qtest 和上游提交思路。
  - 适合复盘“从 unimplemented 到可验证寄存器模型”的完整路径。

## SPI/QSPI 当前实施入口

- [K230 SPI/QSPI 最终处理报告](./k230-spi-qspi-final-report.md)
  - 面向对外展示的能力清单、TRM/SDK/源码/qtest 证据、审阅结论和复现命令。

- [K230 SPI/QSPI 实施与 qtest Study](./k230-spi-qspi-flash-window-study-plan.md)
  - 历史学习材料；正式系列已收敛为 9 个提交，顺序为寄存器模型、实例、PIO、IRQ、PLIC、QSPI、SPI NOR、HI_SYS、XIP。
  - qtest 只保留 `k230-dw-ssi-test.c`，最终为 7 个场景、TAP `1..7`、7/7 通过；以最终处理报告为准。

- [K230 SSI Patch 3 学习工作簿](./k230-spi-patch3-learning-workbook.md)
  - 逐步讲解 Standard TR/TO/RO/EEPROM_READ、FIFO 背压、NDF 和动态 BUSY。

- [K230 SSI Patch 4：Standard SPI NOR 学习工作簿](./k230-spi-patch4-learning-workbook.md)
  - 按 TRM、K230 SDK 和 QEMU M25P80 三层证据说明 W25Q256 板级接线。
  - 讲解 TO/EEPROM_READ Flash 事务、MTD backend、六项 qtest 和真实 U-Boot 验证边界。

- [K230 SSI Patch 5：Dual/Quad QSPI 学习工作簿](./k230-spi-patch5-learning-workbook.md)
  - 讲解 Enhanced instruction/address/mode/dummy/data 阶段、RO/TO 数据路径和六项 qtest。

- [K230 SSI Patch 6：控制器内部 IRQ 学习工作簿](./k230-spi-patch6-learning-workbook.md)
  - 讲解 RISR/ISR/IMR、动态水位、锁存错误、RC 清除、九路输出和 Patch 7 边界。

- [K230 SSI Patch 7：PLIC 接线学习工作簿](./k230-spi-patch7-learning-workbook.md)
  - 讲解三个逻辑 SSI 实例、27 路 PLIC source、显式路由表和五项 routing qtest。

- [旧 Patch 计划迁移说明](./k230-spi-qspi-patch-plan.md)
  - 仅用于兼容历史链接，不再维护独立 Patch 状态或 qtest 表。

## 常用源码跳转

K230 QEMU 仓库：

- [K230 machine](../../qemu-camp-2026-k230/hw/riscv/k230.c)
- [K230 SoC 头文件](../../qemu-camp-2026-k230/include/hw/riscv/k230.h)
- [K230 文档](../../qemu-camp-2026-k230/docs/system/riscv/k230.rst)
- [K230 IOMUX 模型](../../qemu-camp-2026-k230/hw/misc/k230_iomux.c)

K230 SDK：

- [Linux K230 DTS](../../k230_sdk/src/little/linux/arch/riscv/boot/dts/kendryte/k230.dtsi)
- [U-Boot K230 DTS](../../k230_sdk/src/little/uboot/arch/riscv/dts/k230.dtsi)
- [RT-Smart SPI 驱动](../../k230_sdk/src/big/rt-smart/kernel/bsp/maix3/board/interdrv/spi/drv_spi.c)
- [RT-Smart K230 board 地址定义](../../k230_sdk/src/big/rt-smart/kernel/bsp/maix3/board/board.h)

## 维护规则

- 新增 K230 相关笔记默认放在本目录。
- 文件名使用 `k230-主题-用途.md`，例如 `k230-spi-qspi-flash-window-study-plan.md`。
- 每篇笔记开头说明记录时间、问题背景、最终结论和适用范围。
- 引用源码时优先使用从本目录出发的相对链接，保证 Markdown 预览可以直接跳转。
- 对尚未实现的方案要明确写“支持”和“不支持”，避免把调研结论误读成已完成能力。
