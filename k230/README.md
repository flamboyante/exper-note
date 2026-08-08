# K230 QEMU 专题笔记索引

这个目录集中放 K230 QEMU 训练营相关笔记，包括当前 QEMU 支持现状、SDK/TRM 证据链、外设建模计划、启动链路分析和具体 issue 的实现准备。

建议阅读顺序：

```text
先理解当前 K230 machine 支持程度
  -> 再看 QOM / SysBus / MMIO 外设骨架
  -> 再看已完成或准备实现的具体外设
  -> 最后看启动链路和 U-Boot/Linux 验证路线
```

## 子目录结构

- `iomux/`           IOMUX/Pinctrl 主题(含 U-Boot 启动验证)
- `spi/`             SPI / QSPI / Flash Window / patch 学习手册
  - `spi/learning/`  patch3~8 + IDMA 学习手册
  - `spi/reference/`  TRM 第 12.3 章 SPI 译本(大文件)
  - `spi/PLAN.md`     SPI patch 重构计划
- `upstream-review/` 上游 review 资料
  - `current/`       最新版本(当前 v3.4)的 cover letter / commit messages
  - `archive/`       历史版本(v3.02 / v3.1 / v3.2 / v3.3)
- `reference/`       K230 TRM 原始 PDF 文本
- `assets/`          TRM 截图

## 总览与背景

- [K230 SPI NOR 冷启动专题](./spi/coldboot/)
  - [冷启动知识手册](./spi/coldboot/k230-cold-boot-handbook.md)
  - [v4 JFFS2 复现与验收](./spi/coldboot/k230-coldboot-canmv-qemu-v4-reproduction.md)
  - 稳定概念、启动层次、Guest 固件职责、镜像格式、模块边界和验证术语。
- [K230 SPI NOR 冷启动 Gap 调查与实施计划](./spi/coldboot/k230-cold-boot-gap-investigation.md)
  - 动态实验、当前阻塞、证据等级、路线选择和下一步实施计划。

- [K230 QEMU 训练营交流笔记](./k230-qemu-camp-notes.md)
  - 记录 K230 当前 QEMU 支持程度、SDK/Linux/U-Boot 现实状态、启动介质、RAM 执行和 OpenSBI 位置。
  - 第一次了解 K230 训练营任务时优先读这一篇。

- [K230 完整启动与 IOMUX 验证](./iomux/uboot-iomux-unimplemented-repro.md)
  - 以 `k230-boot-assets` 为默认镜像来源，给出 Yocto direct boot、Buildroot direct boot、U-Boot boot 和 IOMUX 验证命令。

- [K230 QEMU 对象模型与外设接线学习笔记](../camp/k230-qemu-object-model-study.md)
  - 解释 K230 machine、SoC、SysBusDevice、MemoryRegion、PLIC/IRQ 接线等基础结构。

- [QEMU 通用基础 MMIO 外设骨架模板](../camp/qemu-basic-mmio-device-template.md)
  - 总结最小 MMIO 外设的文件布局、State、MemoryRegionOps、read/write/reset、Kconfig/meson/qtest 接入方式。

## 已推进外设

### IOMUX

- [K230 IOMUX/Pinctrl 三阶段入门指南](./iomux/k230-iomux-three-stage-guide.md)
  - 记录 IOMUX/Pinctrl 的第一版实现边界、证据链、qtest 和上游提交思路。

### SPI/QSPI

- [SPI patch 重构计划](./spi/PLAN.md)

- [K230 U-Boot 从 SPI Flash 启动 Linux：最小复现](./spi/k230-spi-flash-uboot-linux-quickstart.md)
  - 面向当前 `k230-spiv2` 分支的最短复现路径。

- [K230 SPI/QSPI 最终处理报告](./spi/k230-spi-qspi-final-report.md)
  - 面向对外展示的能力清单、TRM/SDK/源码/qtest 证据、审阅结论和复现命令。

- [K230 SPI/QSPI 实施与 qtest Study](./spi/k230-spi-qspi-flash-window-study-plan.md)
  - 历史学习材料；正式系列已收敛为 9 个提交。

- [K230 SPI/QSPI 寄存器审计](./spi/k230-spi-qspi-register-audit.md)

- [旧 Patch 计划迁移说明](./spi/k230-spi-qspi-patch-plan.md)
  - 仅用于兼容历史链接，不再维护独立 Patch 状态或 qtest 表。

#### Patch 学习手册(`spi/learning/`)

- [Patch 3 - Standard TR/TO/RO/EEPROM_READ](./spi/learning/k230-spi-patch3-learning-workbook.md)
- [Patch 4 - Standard SPI NOR](./spi/learning/k230-spi-patch4-learning-workbook.md)
- [Patch 5 - Dual/Quad QSPI](./spi/learning/k230-spi-patch5-learning-workbook.md)
- [Patch 6 - 控制器内部 IRQ](./spi/learning/k230-spi-patch6-learning-workbook.md)
- [Patch 7 - PLIC 接线](./spi/learning/k230-spi-patch7-learning-workbook.md)
- [Patch 8 - 后续扩展](./spi/learning/k230-spi-patch8-learning-workbook.md)
- [IDMA Patch](./spi/learning/k230-spi-idma-patch-learning-workbook.md)

#### 参考资料(`spi/reference/`)

- [K230 TRM 12.3 SPI 中文学习版](./spi/reference/k230-trm-12.3-spi-cn.md) — TRM SPI 章节译本(2.3k 行,大文件)

## 上游 Review 资料(`upstream-review/`)

- [上游邮件沟通日志](./upstream-review/upstream-mail-log.md)
- [IOMUX 上游 review 回复](./upstream-review/k230-iomux-upstream-review-reply.md)
- [DW SSI 拆分分析](./upstream-review/k230-spi-qspi-dwssi-split-analysis.md)
- [Review v2 决策记录](./upstream-review/k230-spi-qspi-review-v2-decision-notes.md)

### 当前版本(v3.4，`upstream-review/current/`)

- [Cover Letter](./upstream-review/current/k230-spiv3.4-cover-letter.md)
- [Cover Letter 中文](./upstream-review/current/k230-spiv3.4-cover-letter-cn.md)
- [Commit Messages Bilingual](./upstream-review/current/k230-spiv3.4-commit-messages-bilingual.md)

### 历史版本(`upstream-review/archive/`)

- v3.3：[Commit Messages](./upstream-review/archive/v3.3/k230-spiv3.3-commit-messages-bilingual.md)
- v3.2：[Cover Letter](./upstream-review/archive/v3.2/k230-spiv3.2-cover-letter.md) / [中文](./upstream-review/archive/v3.2/k230-spiv3.2-cover-letter-cn.md) / [v3.1→v3.2 转换说明](./upstream-review/archive/v3.2/基于k230-spiv3.1做这样的修改，输出k230-spiv3.2，包含10)
- v3.1：[Commit Messages](./upstream-review/archive/v3.1/k230-spiv3.1-commit-messages-bilingual.md)
- v3.02：[Commit Messages](./upstream-review/archive/v3.02/k230-spiv3.02-commit-messages-bilingual.md)

## 常用源码跳转

K230 QEMU 仓库：

- [K230 machine](../../my-qemu-camp-2026-k230/hw/riscv/k230.c)
- [K230 SoC 头文件](../../my-qemu-camp-2026-k230/include/hw/riscv/k230.h)
- [K230 文档](../../my-qemu-camp-2026-k230/docs/system/riscv/k230.rst)

K230 SDK：

- [Linux K230 DTS](../../k230_sdk/src/little/linux/arch/riscv/boot/dts/kendryte/k230.dtsi)
- [U-Boot K230 DTS](../../k230_sdk/src/little/uboot/arch/riscv/dts/k230.dtsi)
- [RT-Smart SPI 驱动](../../k230_sdk/src/big/rt-smart/kernel/bsp/maix3/board/interdrv/spi/drv_spi.c)
- [RT-Smart K230 board 地址定义](../../k230_sdk/src/big/rt-smart/kernel/bsp/maix3/board/board.h)

## 维护规则

- 新增 K230 相关笔记按主题归入 `iomux/`、`spi/`、`upstream-review/` 等子目录。
- 文件名使用 `k230-主题-用途.md`，例如 `k230-spi-qspi-flash-window-study-plan.md`。
- 每篇笔记开头说明记录时间、问题背景、最终结论和适用范围。
- 引用源码时优先使用从本目录出发的相对链接，保证 Markdown 预览可以直接跳转。
- 发新版本时，`upstream-review/current/` 整体挪到 `upstream-review/archive/vX.Y/`，再把新版本放进 `current/`。
- 对尚未实现的方案要明确写“支持”和“不支持”，避免把调研结论误读成已完成能力。
