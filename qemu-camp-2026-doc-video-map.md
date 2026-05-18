# QEMU 训练营 2026 讲义与视频对应表

更新时间：2026-05-18

这份表用于后续学习时直接查用。本次重新校验了两边来源：

- 讲义目录：[QEMU 训练营 2026 讲义](https://qemu.gevico.online/tutorial/2026/)
- 视频来源：B 站 UP [绝对是泽文啦](https://space.bilibili.com/483048140) 的公开视频
- 校验方法：逐条使用 B 站公开视频元信息核对 `BV`、标题和阶段标签

## 本次纠错说明

之前那版表里存在“顺位错位”，不是只有一条错。主要修正如下：

- 基础阶段已修正：`虚拟技术历史 / 启动参数 / 启动流程 / QOM / MemoryRegion / 调试手段` 的 BV 号重新对齐。
- 专业阶段已修正：`CPU 建模 / TCG / SoftMMU / 中断异常 / 多核 / 主板 / 外设 / 时钟` 的 BV 号重新对齐。
- `BV1Ae9wBjEiD` 确认是 `专业阶段 QEMU 外设建模流程 [SOC 建模]`。
- `BV1p99cBCEzV` 确认是 `专业阶段 QEMU 时钟系统 [SOC 建模]`，不是外设建模。

状态说明：

- `已匹配`：讲义标题和视频标题可以直接对应。
- `相关辅助`：主题明显相关，但不是严格同名章节视频。
- `待发布 / 待核验`：截至 2026-05-18，在已核到的公开视频里未找到同名视频。

---

## 总览入口

| 用途 | 视频 | 状态 | 备注 |
| --- | --- | --- | --- |
| 训练营总介绍 | [BV1CSSQByEDB](https://www.bilibili.com/video/BV1CSSQByEDB/) `QEMU 训练营 \| 2026 年课程内容介绍：四个阶段（六个实验、四个项目）- 泽文` | 相关辅助 | 开营前先看，适合建立整体地图 |
| 主办方介绍 | [BV1ASSQByE4Q](https://www.bilibili.com/video/BV1ASSQByE4Q/) `QEMU 训练营 \| 主办方华科开放原子开源俱乐部介绍 - 慕冬亮` | 相关辅助 | 了解训练营背景 |
| 在线讲义升级说明 | [BV151FxzmEYq](https://www.bilibili.com/video/BV151FxzmEYq/) `QEMU 训练营 \| 2026 在线讲义，全新升级！` | 相关辅助 | 适合先熟悉讲义站怎么用 |
| C / Rust 入门说明 | [BV151FxzmELv](https://www.bilibili.com/video/BV151FxzmELv/) `QEMU 训练营 \| 最后一次入门 C 和 Rust！` | 相关辅助 | 对补基础有帮助 |

---

## 导学阶段

| 讲义章节 | 文档 | 对应视频 | 状态 | 备注 |
| --- | --- | --- | --- | --- |
| a. QEMU 开发环境搭建 | [qemu-dev-env](https://qemu.gevico.online/tutorial/2026/ch0/qemu-dev-env/) | [BV15yFxzEEMB](https://www.bilibili.com/video/BV15yFxzEEMB/) `QEMU 训练营 \| 一键启动 QEMU 云原生开发环境！` | 相关辅助 | 主题高度一致，适合作为环境搭建入口 |
| b. GPU 发展历史 | [gpu-history](https://qemu.gevico.online/tutorial/2026/ch0/gpu-history/) | [BV1vyFxzEEer](https://www.bilibili.com/video/BV1vyFxzEEer/) `QEMU 训练营 \| 轻松了解 GPU 和 RISC-V！` | 相关辅助 | 更偏导学背景，GPU 部分相关 |
| c. GPGPU 学习资料 | [gpgpu-resources](https://qemu.gevico.online/tutorial/2026/ch0/gpgpu-resources/) | [BV1NkSSBrEqM](https://www.bilibili.com/video/BV1NkSSBrEqM/) `QEMU 训练营 \| 从零开始的 GPGPU 模拟器与 LLM 自动化建模...` | 相关辅助 | 长视频，适合建立 GPGPU 项目背景 |
| d. 训练营常见问答 | [qemu-camp-faq](https://qemu.gevico.online/tutorial/2026/ch0/qemu-camp-faq/) | [BV1eMdvBFEex](https://www.bilibili.com/video/BV1eMdvBFEex/) `导学阶段 训练营在线答疑回放 - 第一周`；[BV1SEdvBpE43](https://www.bilibili.com/video/BV1SEdvBpE43/) `导学阶段 训练营在线答疑回放 - 第二周` | 相关辅助 | FAQ 是静态整理，答疑回放是动态补充 |

---

## 基础阶段

| 讲义章节 | 文档 | 对应视频 | 状态 | 备注 |
| --- | --- | --- | --- | --- |
| a. 虚拟技术历史 | [vm-history](https://qemu.gevico.online/tutorial/2026/ch1/vm-history/) | [BV19xwNzoEMK](https://www.bilibili.com/video/BV19xwNzoEMK/) `QEMU 训练营 \| 基础阶段 虚拟化技术的发展历史` | 已匹配 | 先看这个，建立虚拟化背景 |
| b. 启动参数分析 | [qemu-startup-param](https://qemu.gevico.online/tutorial/2026/ch1/qemu-startup-param/) | [BV1aVwMzYE8a](https://www.bilibili.com/video/BV1aVwMzYE8a/) `QEMU 训练营 \| 基础阶段 QEMU 启动参数分析` | 已匹配 | 讲命令行参数如何影响机器 |
| c. 启动流程分析 | [qemu-init](https://qemu.gevico.online/tutorial/2026/ch1/qemu-init/) | [BV1kQwTzmEGc](https://www.bilibili.com/video/BV1kQwTzmEGc/) `QEMU 训练营 \| 基础阶段 QEMU 启动流程分析` | 已匹配 | 基础阶段主线之一 |
| d. 面向对象建模 | [qemu-qom](https://qemu.gevico.online/tutorial/2026/ch1/qemu-qom/) | [BV1wGwTzLE7q](https://www.bilibili.com/video/BV1wGwTzLE7q/) `QEMU 训练营 \| 基础阶段 QOM 面向对象的建模思想` | 已匹配 | 后面所有设备建模的基础 |
| e. 地址空间抽象 | [qemu-mr](https://qemu.gevico.online/tutorial/2026/ch1/qemu-mr/) | [BV1sSwuzZEXr](https://www.bilibili.com/video/BV1sSwuzZEXr/) `QEMU 训练营 \| 基础阶段 MemoryRegion 面向地址空间抽象` | 已匹配 | SoC MMIO 外设必须懂 |
| f. 常用调试方法 | [qemu-debug](https://qemu.gevico.online/tutorial/2026/ch1/qemu-debug/) | [BV1fwwgzUEt8](https://www.bilibili.com/video/BV1fwwgzUEt8/) `QEMU 训练营 \| 基础阶段 QEMU 常用调试手段` | 已匹配 | 建议和实际 QEMU 调试一起练 |

---

## 专业阶段

| 讲义章节 | 文档 | 对应视频 | 状态 | 备注 |
| --- | --- | --- | --- | --- |
| a. CPU 建模流程 | [qemu-cpu-model](https://qemu.gevico.online/tutorial/2026/ch2/qemu-cpu-model/) | [BV1J4AJz7Eda](https://www.bilibili.com/video/BV1J4AJz7Eda/) `QEMU 训练营 \| 专业阶段 QEMU CPU 建模流程：以 RISC-V 为例 [CPU 建模]` | 已匹配 | CPU 建模线第一课 |
| b. TCG 工作原理 | [qemu-tcg](https://qemu.gevico.online/tutorial/2026/ch2/qemu-tcg/) | [BV1QbApzkEmJ](https://www.bilibili.com/video/BV1QbApzkEmJ/) `QEMU TCG 介绍：二进制动态翻译原理与运行流程（一）`；[BV1beApzuEmt](https://www.bilibili.com/video/BV1beApzuEmt/) `...（二）` | 已匹配 | 这一章对应两条视频，建议连着看 |
| c. 模拟客户机指令 | [qemu-insn](https://qemu.gevico.online/tutorial/2026/ch2/qemu-insn/) | 暂未找到同名公开视频 | 待发布 / 待核验 | 可能包含在 TCG 系列中，也可能后续单独发布 |
| d. 模拟硬件 MMU | [qemu-softmmu](https://qemu.gevico.online/tutorial/2026/ch2/qemu-softmmu/) | [BV1AXQCBtEiC](https://www.bilibili.com/video/BV1AXQCBtEiC/) `QEMU SoftMMU：系统模式下的地址转换与内存访问（一）`；[BV1fnQCBEEW2](https://www.bilibili.com/video/BV1fnQCBEEW2/) `...（二）` | 已匹配 | 这一章也拆成两条视频 |
| e. 模拟中断和异常 | [qemu-intr](https://qemu.gevico.online/tutorial/2026/ch2/qemu-intr/) | [BV1zEX4BzEGR](https://www.bilibili.com/video/BV1zEX4BzEGR/) `QEMU 训练营 \| 专业阶段 QEMU TCG 中断与异常 [CPU 建模]` | 已匹配 | 标题多了 TCG 语境，但主题一致 |
| f. 模拟多核场景 | [qemu-multi-core](https://qemu.gevico.online/tutorial/2026/ch2/qemu-multi-core/) | [BV1CMX4BjE6w](https://www.bilibili.com/video/BV1CMX4BjE6w/) `QEMU 训练营 \| 专业阶段 QEMU 多核模拟 [CPU 建模]` | 已匹配 | CPU 建模线后段 |
| g. 主板建模流程 | [qemu-machine](https://qemu.gevico.online/tutorial/2026/ch2/qemu-machine/) | [BV1v4X4BPEP3](https://www.bilibili.com/video/BV1v4X4BPEP3/) `QEMU 训练营 \| 专业阶段 QEMU 主板建模流程 [SOC 建模]` | 已匹配 | SoC 模块建议优先看 |
| h. 外设建模流程 | [qemu-hw](https://qemu.gevico.online/tutorial/2026/ch2/qemu-hw/) | [BV1Ae9wBjEiD](https://www.bilibili.com/video/BV1Ae9wBjEiD/) `QEMU 训练营 \| 专业阶段 QEMU 外设建模流程 [SOC 建模]` | 已匹配 | 已生成学习资料：`notes/qemu-hw-study.md` |
| i. 时钟系统介绍 | [qemu-clock](https://qemu.gevico.online/tutorial/2026/ch2/qemu-clock/) | [BV1p99cBCEzV](https://www.bilibili.com/video/BV1p99cBCEzV/) `QEMU 训练营 \| 专业阶段 QEMU 时钟系统 [SOC 建模]` | 已匹配 | PWM/WDT 计时相关，建议接在外设后 |
| j. GPGPU 原理 | [qemu-gpgpu](https://qemu.gevico.online/tutorial/2026/ch2/qemu-gpgpu/) | [BV1NkSSBrEqM](https://www.bilibili.com/video/BV1NkSSBrEqM/) `从零开始的 GPGPU 模拟器与 LLM 自动化建模...` | 相关辅助 | 不是严格同名短课，但主题高度相关 |
| k. PCIe 模拟方法 | [qemu-pcie](https://qemu.gevico.online/tutorial/2026/ch2/qemu-pcie/) | 暂未找到同名公开视频 | 待发布 / 待核验 | 目前未在已核公开视频中看到同名课 |
| l. kernel 运行机制 | [qemu-gpgpu-kernel](https://qemu.gevico.online/tutorial/2026/ch2/qemu-gpgpu-kernel/) | [BV14NAPzjEPW](https://www.bilibili.com/video/BV14NAPzjEPW/) `超详细干货分享，手把手教你基于 QEMU 调试 Linux 内核` | 相关辅助 | 强相关辅助视频，但不是同名讲义课 |
| m. Rust FFI 介绍 | [qemu-rust-ffi](https://qemu.gevico.online/tutorial/2026/ch2/qemu-rust-ffi/) | 暂未找到同名公开视频 | 待发布 / 待核验 | 目前未核到同名公开视频 |
| n. Rust 建模流程 | [qemu-rust-model](https://qemu.gevico.online/tutorial/2026/ch2/qemu-rust-model/) | 暂未找到同名公开视频 | 待发布 / 待核验 | 目前未核到同名公开视频 |
| o. Rust 测题开发 | [qemu-rust-test](https://qemu.gevico.online/tutorial/2026/ch2/qemu-rust-test/) | 暂未找到同名公开视频 | 待发布 / 待核验 | 目前未核到同名公开视频 |

---

## 项目阶段

| 讲义章节 | 文档 | 对应视频 | 状态 | 备注 |
| --- | --- | --- | --- | --- |
| a. 数字星务计算机 | [qemu-k230](https://qemu.gevico.online/tutorial/2026/ch3/qemu-k230/) | [BV1PUSQBLEkW](https://www.bilibili.com/video/BV1PUSQBLEkW/) `QEMU 训练营 \| 数字星务计算机项目与 RustSBI 介绍 - 洛佳` | 相关辅助 | 主题非常接近，适合作为项目导入 |
| b. 建模 AI 加速卡 | [qemu-cxlemu](https://qemu.gevico.online/tutorial/2026/ch3/qemu-cxlemu/) | 暂未找到同名公开视频 | 待发布 / 待核验 | 项目阶段通常更晚，可能尚未公开放出 |
| c. 大模型建模外设 | [qemu-agent](https://qemu.gevico.online/tutorial/2026/ch3/qemu-agent/) | [BV1PPQTBsE9Q](https://www.bilibili.com/video/BV1PPQTBsE9Q/) `导学阶段 基于 Agent 驱动的 RISC-V 模拟器开发` | 相关辅助 | 和 Agent 建模方法相关，但不是同名项目课 |
| d. 应用跨平台转译 | [qemu-wine-ce](https://qemu.gevico.online/tutorial/2026/ch3/qemu-wine-ce/) | [BV1yUSQBLEPj](https://www.bilibili.com/video/BV1yUSQBLEPj/) `Box64 在龙架构上的 x86 兼容性实践 - 刘阳` | 相关辅助 | 方向接近，但不是讲义章节同名视频 |

---

## 当前公开视频清单

下面这部分是本次核验过的 UP 主公开视频标题，方便以后发现新视频时做增量更新。

| BV | 标题 |
| --- | --- |
| [BV1MrdwBbEUZ](https://www.bilibili.com/video/BV1MrdwBbEUZ/) | 教你一键搭建 Linux 内核开发环境 |
| [BV1eMdvBFEex](https://www.bilibili.com/video/BV1eMdvBFEex/) | QEMU 训练营 \| 导学阶段 训练营在线答疑回放 - 第一周 |
| [BV1SEdvBpE43](https://www.bilibili.com/video/BV1SEdvBpE43/) | QEMU 训练营 \| 导学阶段 训练营在线答疑回放 - 第二周 |
| [BV1LYDDBhEiq](https://www.bilibili.com/video/BV1LYDDBhEiq/) | 源来喻见你 \| 基于 Agent 范式 x 多人协作的系统软件开发介绍 |
| [BV1PPQTBsE9Q](https://www.bilibili.com/video/BV1PPQTBsE9Q/) | QEMU 训练营 \| 导学阶段 基于 Agent 驱动的 RISC-V 模拟器开发 |
| [BV1M6SQBDEpg](https://www.bilibili.com/video/BV1M6SQBDEpg/) | QEMU 训练营 \| 甲辰计划：这是最坏的时代，这是最好的时代 - 吴伟 |
| [BV1yUSQBLEPj](https://www.bilibili.com/video/BV1yUSQBLEPj/) | QEMU 训练营 \| Box64 在龙架构上的 x86 兼容性实践 - 刘阳 |
| [BV1PUSQBLEkW](https://www.bilibili.com/video/BV1PUSQBLEkW/) | QEMU 训练营 \| 数字星务计算机项目与 RustSBI 介绍 - 洛佳 |
| [BV1CSSQByEDB](https://www.bilibili.com/video/BV1CSSQByEDB/) | QEMU 训练营 \| 2026 年课程内容介绍：四个阶段（六个实验、四个项目）- 泽文 |
| [BV1ASSQByE4Q](https://www.bilibili.com/video/BV1ASSQByE4Q/) | QEMU 训练营 \| 主办方华科开放原子开源俱乐部介绍 - 慕冬亮 |
| [BV1NkSSBrEqM](https://www.bilibili.com/video/BV1NkSSBrEqM/) | QEMU 训练营 \| 从零开始的 GPGPU 模拟器与 LLM 自动化建模，762人报名！352 所高校和 165 家企业参与，18 周时长 |
| [BV1p99cBCEzV](https://www.bilibili.com/video/BV1p99cBCEzV/) | QEMU 训练营 \| 专业阶段 QEMU 时钟系统 [SOC 建模] |
| [BV1Ae9wBjEiD](https://www.bilibili.com/video/BV1Ae9wBjEiD/) | QEMU 训练营 \| 专业阶段 QEMU 外设建模流程 [SOC 建模] |
| [BV1CMX4BjE6w](https://www.bilibili.com/video/BV1CMX4BjE6w/) | QEMU 训练营 \| 专业阶段 QEMU 多核模拟 [CPU 建模] |
| [BV1zEX4BzEGR](https://www.bilibili.com/video/BV1zEX4BzEGR/) | QEMU 训练营 \| 专业阶段 QEMU TCG 中断与异常 [CPU 建模] |
| [BV1v4X4BPEP3](https://www.bilibili.com/video/BV1v4X4BPEP3/) | QEMU 训练营 \| 专业阶段 QEMU 主板建模流程 [SOC 建模] |
| [BV1fnQCBEEW2](https://www.bilibili.com/video/BV1fnQCBEEW2/) | QEMU 训练营 \| 专业阶段 QEMU SoftMMU：系统模式下的地址转换与内存访问（二） [CPU 建模] |
| [BV1AXQCBtEiC](https://www.bilibili.com/video/BV1AXQCBtEiC/) | QEMU 训练营 \| 专业阶段 QEMU SoftMMU：系统模式下的地址转换与内存访问（一） [CPU 建模] |
| [BV1MRAgzTECw](https://www.bilibili.com/video/BV1MRAgzTECw/) | QEMU 训练营 \| 从零 Vibe 龙虾之“流水线” — 用 Agent 跑大型重构和自动化 |
| [BV1eZATzqET4](https://www.bilibili.com/video/BV1eZATzqET4/) | QEMU 训练营 \| 从零 Vibe 龙虾之“对话” — 如何驱动 Agent 做设计和编码 |
| [BV1rpAMznESa](https://www.bilibili.com/video/BV1rpAMznESa/) | QEMU 训练营 \| 从零 Vibe 龙虾之“工作台” — 搭建你的 Agent 开发环境 |
| [BV14NAPzjEPW](https://www.bilibili.com/video/BV14NAPzjEPW/) | QEMU 训练营 \| 超详细干货分享，手把手教你基于 QEMU 调试 Linux 内核 |
| [BV1beApzuEmt](https://www.bilibili.com/video/BV1beApzuEmt/) | QEMU 训练营 \| 专业阶段 QEMU TCG 介绍：二进制动态翻译原理与运行流程（二） [CPU 建模] |
| [BV1QbApzkEmJ](https://www.bilibili.com/video/BV1QbApzkEmJ/) | QEMU 训练营 \| 专业阶段 QEMU TCG 介绍：二进制动态翻译原理与运行流程（一） [CPU 建模] |
| [BV1J4AJz7Eda](https://www.bilibili.com/video/BV1J4AJz7Eda/) | QEMU 训练营 \| 专业阶段 QEMU CPU 建模流程：以 RISC-V 为例 [CPU 建模] |
| [BV1TMw3zeEkM](https://www.bilibili.com/video/BV1TMw3zeEkM/) | RT-Claw 只需一美元硬件成本，快速部署你的专属龙虾！ |
| [BV1fwwgzUEt8](https://www.bilibili.com/video/BV1fwwgzUEt8/) | QEMU 训练营 \| 基础阶段 QEMU 常用调试手段 |
| [BV1sSwuzZEXr](https://www.bilibili.com/video/BV1sSwuzZEXr/) | QEMU 训练营 \| 基础阶段 MemoryRegion 面向地址空间抽象 |
| [BV1wGwTzLE7q](https://www.bilibili.com/video/BV1wGwTzLE7q/) | QEMU 训练营 \| 基础阶段 QOM 面向对象的建模思想 |
| [BV1kQwTzmEGc](https://www.bilibili.com/video/BV1kQwTzmEGc/) | QEMU 训练营 \| 基础阶段 QEMU 启动流程分析 |
| [BV1aVwMzYE8a](https://www.bilibili.com/video/BV1aVwMzYE8a/) | QEMU 训练营 \| 基础阶段 QEMU 启动参数分析 |
| [BV19xwNzoEMK](https://www.bilibili.com/video/BV19xwNzoEMK/) | QEMU 训练营 \| 基础阶段 虚拟化技术的发展历史 |
| [BV1kzZ6BcEeg](https://www.bilibili.com/video/BV1kzZ6BcEeg/) | 用 Claude + Codex 零代码写出 4.5 万行 Rust，性能超 QEMU 30% |
| [BV1VmFxzjE8v](https://www.bilibili.com/video/BV1VmFxzjE8v/) | QEMU 训练营 \| 轻松参与 QEMU 开源贡献！ |
| [BV1VUFxz1ENn](https://www.bilibili.com/video/BV1VUFxz1ENn/) | QEMU 训练营 \| ima 知识库助力系统软件学习！ |
| [BV15yFxzEEMB](https://www.bilibili.com/video/BV15yFxzEEMB/) | QEMU 训练营 \| 一键启动 QEMU 云原生开发环境！ |
| [BV1vyFxzEEer](https://www.bilibili.com/video/BV1vyFxzEEer/) | QEMU 训练营 \| 轻松了解 GPU 和 RISC-V！ |
| [BV151FxzmELv](https://www.bilibili.com/video/BV151FxzmELv/) | QEMU 训练营 \| 最后一次入门 C 和 Rust！ |
| [BV151FxzmEYq](https://www.bilibili.com/video/BV151FxzmEYq/) | QEMU 训练营 \| 2026 在线讲义，全新升级！ |

---

## 我的使用建议

后面你开始专业阶段 SoC 模块，优先顺序建议这样走：

1. `主板建模流程`：先知道 G233 machine 怎么把设备挂起来。
2. `外设建模流程`：再学 GPIO/PWM/WDT/SPI 这种 MMIO 设备怎么写。
3. `时钟系统介绍`：接上 PWM/WDT 的 timer、clock、虚拟时间。
4. `kernel 运行机制`：最后理解 guest driver 怎么通过 MMIO/IRQ 使用这些外设。

基础阶段如果要补，建议按 `启动参数 -> 启动流程 -> QOM -> MemoryRegion -> 调试手段` 的顺序快速复盘。
