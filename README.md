# QEMU Camp 2026 学习笔记

这个仓库放在训练营实验仓库的 `exper-note/` 目录下。文档里的源码链接默认用相对路径跳到外层 `exper` 仓库，例如：

- [G233 board](../hw/riscv/g233.c)
- [G233 header](../include/hw/riscv/g233.h)
- [SoC qtest](../tests/gevico/qtest)
- [Makefile.camp](../Makefile.camp)
- [README_zh.md](../README_zh.md)

## 当前主线

优先执行 SoC 方向，不继续横向扩散资料。

1. [SoC 七天完成计划](./soc-7day-plan.md)
2. [Day 1 白皮书](./soc-day1-whitepaper.md)
3. [早期 SoC 学习计划](./soc-study-plan.md)

## 课程与视频地图

- [QEMU 训练营 2026 讲义与视频对应表](./qemu-camp-2026-doc-video-map.md)

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
- 需要引用实验仓库源码时，优先使用从 `exper-note/` 出发的相对链接，例如 `[hw/riscv/g233.c](../hw/riscv/g233.c)`。
- 每篇笔记只保留对当前学习和 SoC 实验有用的结论，避免继续堆课堂转写。
