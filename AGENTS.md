# Agent Guide

本文件给 Codex / TRAE / Claude Code 等 agent 自动读取，避免每次重新探索项目结构。读这一篇就能拿到项目地图、常用路径和不变事实，不用再 ls + grep + read 多轮搜索。

## 项目导航

`exper-note/` 是 QEMU 训练营 2026 的公共文档区，按主题分子目录。进入本目录前先读取上级工作区的 `AGENTS.md` 与 [工作区状态页](./workspace-state.md)：状态页决定当前默认主题和最小阅读路径，本文件只提供稳定的文档地图。

- `camp/`           训练营课程笔记(SoC 白皮书、QEMU 各主题 study、通用 MMIO 模板、assets)
- `k230/`           K230 实战主项目
  - `iomux/`         IOMUX/Pinctrl 主题(含 U-Boot 启动验证)
  - `spi/`           SPI / QSPI / Flash Window
    - `spi/learning/` patch3~8 + IDMA 学习手册
    - `spi/reference/` TRM 第 12.3 章 SPI 译本(大文件,2.3k 行)
    - `spi/PLAN.md`   SPI patch 重构计划
  - `upstream-review/` 上游 review 资料
    - `current/`     最新版本(当前 v3.4)cover letter / commit messages
    - `archive/`     历史版本(v3.02 / v3.1 / v3.2 / v3.3)
  - `reference/`     K230 TRM 原始 PDF 文本
  - `assets/`        TRM 截图
- `g233-gpio-study.md`  G233 板子 GPIO 笔记(独立小主题)

## 常用路径速查

| 任务 | 路径 |
|---|---|
| 工作区当前状态与主题切换 | workspace-state.md |
| V2 架构决策（唯一入口） | k230/upstream-review/k230-spi-qspi-review-v2-decision-notes.md |
| 当前最新 cover letter | k230/upstream-review/current/k230-spiv3.4-cover-letter.md |
| 当前最新 commit messages | k230/upstream-review/current/k230-spiv3.4-commit-messages-bilingual.md |
| 上游 review 决策 | k230/upstream-review/k230-spi-qspi-review-v2-decision-notes.md |
| 上游邮件时间线 | k230/upstream-review/upstream-mail-log.md |
| DW SSI 拆分分析 | k230/upstream-review/k230-spi-qspi-dwssi-split-analysis.md |
| IOMUX 上游回复 | k230/upstream-review/k230-iomux-upstream-review-reply.md |
| patch 学习手册 | k230/spi/learning/k230-spi-patch{N}-learning-workbook.md (N=3..8) |
| SPI Flash 启动复现 | k230/spi/k230-spi-flash-uboot-linux-quickstart.md |
| U-Boot 启动验证 | k230/iomux/uboot-iomux-unimplemented-repro.md |
| IOMUX 入门 | k230/iomux/k230-iomux-three-stage-guide.md |
| K230 TRM SPI 译本 | k230/spi/reference/k230-trm-12.3-spi-cn.md (大文件,2.3k 行) |
| K230 TRM 原始文本 | k230/reference/K230_Technical_Reference_Manual_V0.3.1_20241118.txt |
| K230 项目总览 | k230/k230-qemu-camp-notes.md |
| 通用 MMIO 外设模板 | camp/qemu-basic-mmio-device-template.md |
| SPI patch 重构计划 | k230/spi/PLAN.md |
| 历史版本 cover letter | k230/upstream-review/archive/v3.2/k230-spiv3.2-cover-letter.md |

## 不变事实(无需重新发现)

- 工作区根:`/home/flamboy/qemu-camp-2026/`
- K230 QEMU 仓库:`/home/flamboy/qemu-camp-2026/qemu-camp-2026-k230/`
- K230 SDK:`/home/flamboy/qemu-camp-2026/k230_sdk/`
- exper-note 仓库:`/home/flamboy/qemu-camp-2026/exper-note/`
  - origin: `git@github.com:flamboyante/exper-note.git`
- V1 当前代码分支:`k230-spiv3.4`；`upstream-review/current/` 的 v3.4 资料同属 V1。V2 是下一代大版本，阶段以 `workspace-state.md` 为准。
- 文件名约定:`k230-主题-用途.md`，保留完整身份不简写
- 发新版本时:`current/` 整体挪到 `archive/vX.Y/`，再放新版本进 `current/`

## 大文件警告

读这些文件前先用 Grep 找关键词，不要整文件 Read：

| 文件 | 行数 | 内容 |
|---|---|---|
| k230/spi/reference/k230-trm-12.3-spi-cn.md | 2303 | TRM 第 12.3 章 SPI 译本 |
| camp/soc-day4-spi-flash-whitepaper.md | 1740 | Day 4 SPI Flash 白皮书 |
| camp/soc-day1-whitepaper.md | 1532 | Day 1 白皮书 |
| camp/qemu-hw-study.md | 1527 | QEMU 外设建模流程 |
| k230/spi/learning/k230-spi-patch4-learning-workbook.md | 1443 | Patch 4 学习手册 |
| camp/qemu-basic-mmio-device-template.md | 1428 | MMIO 外设模板 |
| k230/spi/k230-spi-qspi-flash-window-study-plan.md | 1081 | QSPI flash window study |
| k230/spi/learning/k230-spi-idma-patch-learning-workbook.md | 994 | IDMA patch 手册 |
| camp/soc-day3-whitepaper.md | 1213 | Day 3 白皮书 |
| k230/k230-qemu-camp-notes.md | 521 | K230 项目总览 |

## 调研委托建议

主 agent 做调研时，优先用 Task subagent 翻文件，只回收结论：

- 主 agent 说："查 dwssi 拆分边界，只要结论"
- subagent 读 `k230/upstream-review/k230-spi-qspi-dwssi-split-analysis.md`，只返回 200 字结论
- 主 context 只增加 200 字，而不是文件全文

## 维护约定

- 文本统一 UTF-8 + LF
- 引用源码用从 `exper-note/` 出发的相对路径
- 新增 K230 笔记按主题归入 `iomux/` / `spi/` / `upstream-review/` 等子目录
- 对尚未实现的方案要明确写"支持"和"不支持"，避免把调研结论误读成已完成能力
