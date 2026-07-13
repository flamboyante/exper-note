# QEMU Camp 2026 公共学习笔记

这个目录现在作为 `qemu-camp-2026` 工作区的公共文档区使用，集中记录训练营实验、K230 QEMU、SDK 证据链和学习笔记。

推荐工作区结构：

```text
qemu-camp-2026/
  exper-note/
  qemu-camp-2026-k230/
  k230_sdk/
  qemu-camp-2026-exper-flamboyante/
  qemu-camp-2026-c-flamboyante/
```

文档引用源码时，优先使用从 `exper-note/` 出发的相对路径。

常用跳转：

- [K230 QEMU 源码仓库](../qemu-camp-2026-k230)
- [K230 SDK](../k230_sdk)
- [训练营实验仓库](../qemu-camp-2026-exper-flamboyante)
- [C 基础练习仓库](../qemu-camp-2026-c-flamboyante)

## 上游沟通

- [上游邮件沟通日志](./upstream-mail-log.md)

## 当前主线

优先执行 SoC 方向，不继续横向扩散资料。

1. [SoC 七天完成计划](./soc-7day-plan.md)
2. [Day 1 白皮书](./soc-day1-whitepaper.md)
3. [早期 SoC 学习计划](./soc-study-plan.md)

## 课程与视频地图

- [QEMU 训练营 2026 讲义与视频对应表](./qemu-camp-2026-doc-video-map.md)

## K230 专题

- [K230 QEMU 专题笔记索引](./k230/README.md)
  - 集中跳转 K230 machine 现状、对象模型、IOMUX、QSPI/SPI/Flash Window 调研和 SDK 证据链。

## 基础阶段笔记

- [课程介绍学习笔记](./qemu-training-camp-2026-course-intro-study.md)
- [启动参数分析学习资料](./qemu-startup-param-study.md)
- [启动流程分析学习资料](./qemu-init-study.md)

## 专业阶段与专题笔记

- [QEMU 外设建模流程学习资料](./qemu-hw-study.md)
- [基于 QEMU 调试 Linux 内核学习笔记](./qemu-gdb-kernel-debug-study.md)

## 维护规则

- 文本文件统一使用 UTF-8 + LF，避免 Windows/Linux 交替编辑造成整文件 diff。
- 原始视频、音频、转写 JSON、网页抓取文件不入库，只保留整理后的 Markdown 和必要截图。
- 需要引用实验仓库源码时，优先使用从 `exper-note/` 出发的相对链接，例如 `[G233 board](../qemu-camp-2026-exper-flamboyante/hw/riscv/g233.c)`。
- 需要引用 K230 QEMU 源码时，优先使用从 `exper-note/` 出发的相对链接，例如 `[K230 machine](../qemu-camp-2026-k230/hw/riscv/k230.c)`。
- 每篇笔记只保留对当前学习和 SoC 实验有用的结论，避免继续堆课堂转写。
