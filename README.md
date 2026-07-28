# QEMU Camp 2026 公共学习笔记

`qemu-camp-2026` 工作区的公共文档区，集中记录训练营实验、K230 QEMU、SDK 证据链和学习笔记。

工作区：`exper-note/` | `my-qemu-camp-2026-k230/` | `k230_sdk/` | `qemu-camp-2026-exper-flamboyante/` | `qemu-camp-2026-c-flamboyante/`

引用源码时优先用从 `exper-note/` 出发的相对路径。Agent 导航详见 [AGENTS.md](./AGENTS.md)。

## 目录结构

- `camp/`  训练营课程笔记(SoC 白皮书、QEMU 各主题 study、通用 MMIO 模板、assets)
- `k230/`   K230 实战主项目
  - `iomux/`         IOMUX/Pinctrl(含 U-Boot 启动验证)
  - `spi/`           SPI / QSPI / Flash Window(`spi/learning/` 放 patch 学习手册,`spi/reference/` 放 TRM 译本,`spi/PLAN.md` 是 SPI patch 重构计划)
  - `upstream-review/`  上游 review 资料(`current/` 最新 v3.4，`archive/` 历史版本)
  - `reference/`     K230 TRM 原始文本
  - `assets/`        K230 TRM 截图
- [G233 GPIO 独立小主题](./g233-gpio-study.md)

## 索引

### 上游沟通
- [上游邮件沟通日志](./k230/upstream-review/upstream-mail-log.md) — 上游 review 邮件时间线与结论

### 当前主线(SoC 方向)
- [SoC 七天完成计划](./camp/soc-7day-plan.md)
- [Day 1 白皮书](./camp/soc-day1-whitepaper.md)
- [早期 SoC 学习计划](./camp/soc-study-plan.md)

### K230 专题
- [K230 专题笔记索引](./k230/README.md) — 跳转 K230 machine 现状、IOMUX、QSPI/SPI 调研、上游 review

### 课程地图
- [讲义与视频对应表](./camp/qemu-camp-2026-doc-video-map.md)

### 基础阶段
- [课程介绍](./camp/qemu-training-camp-2026-course-intro-study.md)
- [启动参数分析](./camp/qemu-startup-param-study.md)
- [启动流程分析](./camp/qemu-init-study.md)

### 专业阶段
- [外设建模流程](./camp/qemu-hw-study.md)
- [QEMU 调试 Linux 内核](./camp/qemu-gdb-kernel-debug-study.md)
- [通用 MMIO 外设骨架模板](./camp/qemu-basic-mmio-device-template.md)

## 维护规则

- UTF-8 + LF，避免 Windows/Linux 交替编辑造成整文件 diff
- 原始视频/音频/转写 JSON/网页抓取不入库，只保留 Markdown 和必要截图
- 引用实验仓库源码：`[G233 board](../qemu-camp-2026-exper-flamboyante/hw/riscv/g233.c)`
- 引用 K230 QEMU 源码：`[K230 machine](../my-qemu-camp-2026-k230/hw/riscv/k230.c)`
- 每篇笔记只保留对当前学习和 SoC 实验有用的结论，避免堆课堂转写