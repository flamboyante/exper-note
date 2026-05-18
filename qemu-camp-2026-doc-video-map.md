# QEMU 训练营 2026 讲义与视频对应表

更新时间：2026-05-18

这份表是给后面学习时直接查用的。我这次对了两边来源：

- 讲义目录：[QEMU 训练营 2026 讲义](https://qemu.gevico.online/tutorial/2026/)
- 视频来源：B 站 UP [绝对是泽文啦](https://space.bilibili.com/483048140) 的公开视频页

状态说明：

- `已匹配`：讲义标题和视频标题能直接对上
- `相关辅助`：主题明显相关，但不是严格一一同名
- `待发布/待核验`：截至 2026-05-18，我在已核到的公开视频页里还没看到能直接对应的公开视频

## 总览入口

| 用途 | 视频 | 备注 |
| --- | --- | --- |
| 训练营总介绍 | [BV1PUSQBLEkW](https://www.bilibili.com/video/BV1PUSQBLEkW/) `QEMU 训练营 \| 2026 年课程内容介绍：四个阶段（六个实验、四个项目）- 泽文` | 开营前先看，适合建立整体地图 |
| 主办方介绍 | [BV1CSSQByEDB](https://www.bilibili.com/video/BV1CSSQByEDB/) `QEMU 训练营 \| 主办方华科开放原子开源俱乐部介绍 - 慕冬亮` | 辅助了解训练营背景 |
| 在线讲义升级说明 | [BV151FxzmELv](https://www.bilibili.com/video/BV151FxzmELv/) `QEMU 训练营 \| 2026 在线讲义，全新升级！` | 适合先熟悉讲义站怎么用 |
| 筹备讨论会 | [BV151FxzmEYq](https://www.bilibili.com/video/BV151FxzmEYq/) `QEMU 训练营 \| 2026 筹备启动讨论会` | 适合了解训练营最初设计思路 |

## 导学阶段

| 讲义章节 | 文档 | 对应视频 | 状态 | 备注 |
| --- | --- | --- | --- | --- |
| a. QEMU 开发环境搭建 | [qemu-dev-env](https://qemu.gevico.online/tutorial/2026/ch0/qemu-dev-env/) | [BV15yFxzEEMB](https://www.bilibili.com/video/BV15yFxzEEMB/) `QEMU 训练营 \| 一键启动 QEMU 云原生开发环境！` | 相关辅助 | 主题高度一致，适合作为环境搭建入口 |
| b. GPU 发展历史 | [gpu-history](https://qemu.gevico.online/tutorial/2026/ch0/gpu-history/) | [BV1vyFxzEEer](https://www.bilibili.com/video/BV1vyFxzEEer/) `QEMU 训练营 \| 轻松了解 GPU 和 RISC-V！` | 相关辅助 | 更偏导学背景，GPU 部分是直接相关的 |
| c. GPGPU 学习资料 | [gpgpu-resources](https://qemu.gevico.online/tutorial/2026/ch0/gpgpu-resources/) | [BV1ASSQByE4Q](https://www.bilibili.com/video/BV1ASSQByE4Q/) `QEMU 训练营 \| 从零开始的 GPGPU 模拟器与 LLM 自动化建模...` | 相关辅助 | 适合补导学视角和方向感 |
| d. 训练营常见问答 | [qemu-camp-faq](https://qemu.gevico.online/tutorial/2026/ch0/qemu-camp-faq/) | [BV1MrdwBbEUZ](https://www.bilibili.com/video/BV1MrdwBbEUZ/) `QEMU 训练营 \| 导学阶段 训练营在线答疑回放 - 第一周`；[BV1eMdvBFEex](https://www.bilibili.com/video/BV1eMdvBFEex/) `QEMU 训练营 \| 导学阶段 训练营在线答疑回放 - 第二周` | 相关辅助 | FAQ 更像静态整理，答疑回放更像动态补充 |

## 基础阶段

| 讲义章节 | 文档 | 对应视频 | 状态 | 备注 |
| --- | --- | --- | --- | --- |
| a. 虚拟技术历史 | [vm-history](https://qemu.gevico.online/tutorial/2026/ch1/vm-history/) | [BV1aVwMzYE8a](https://www.bilibili.com/video/BV1aVwMzYE8a/) `QEMU 训练营 \| 基础阶段 虚拟化技术的发展历史` | 已匹配 | 讲义和视频是直接对应关系 |
| b. 启动参数分析 | [qemu-startup-param](https://qemu.gevico.online/tutorial/2026/ch1/qemu-startup-param/) | [BV1kQwTzmEGc](https://www.bilibili.com/video/BV1kQwTzmEGc/) `QEMU 训练营 \| 基础阶段 QEMU 启动参数分析` | 已匹配 | 建议和讲义一起看 |
| c. 启动流程分析 | [qemu-init](https://qemu.gevico.online/tutorial/2026/ch1/qemu-init/) | [BV1wGwTzLE7q](https://www.bilibili.com/video/BV1wGwTzLE7q/) `QEMU 训练营 \| 基础阶段 QEMU 启动流程分析` | 已匹配 | 是基础阶段最关键的主线之一 |
| d. 面向对象建模 | [qemu-qom](https://qemu.gevico.online/tutorial/2026/ch1/qemu-qom/) | [BV1sSwuzZEXr](https://www.bilibili.com/video/BV1sSwuzZEXr/) `QEMU 训练营 \| 基础阶段 QOM 面向对象的建模思想` | 已匹配 | 标题表达稍有差异，但内容是一一对应 |
| e. 地址空间抽象 | [qemu-mr](https://qemu.gevico.online/tutorial/2026/ch1/qemu-mr/) | [BV1fwwgzUEt8](https://www.bilibili.com/video/BV1fwwgzUEt8/) `QEMU 训练营 \| 基础阶段 MemoryRegion 面向地址空间抽象` | 已匹配 | MemoryRegion 就是这章的核心 |
| f. 常用调试方法 | [qemu-debug](https://qemu.gevico.online/tutorial/2026/ch1/qemu-debug/) | [BV1TMw3zeEkM](https://www.bilibili.com/video/BV1TMw3zeEkM/) `QEMU 训练营 \| 基础阶段 QEMU 常用调试手段` | 已匹配 | 标题差异只在“方法/手段”措辞上 |

## 专业阶段

| 讲义章节 | 文档 | 对应视频 | 状态 | 备注 |
| --- | --- | --- | --- | --- |
| a. CPU 建模流程 | [qemu-cpu-model](https://qemu.gevico.online/tutorial/2026/ch2/qemu-cpu-model/) | [BV1QbApzkEmJ](https://www.bilibili.com/video/BV1QbApzkEmJ/) `QEMU 训练营 \| 专业阶段 QEMU CPU 建模流程：以 RISC-V 为例 [CPU 建模]` | 已匹配 | 直接对上 |
| b. TCG 工作原理 | [qemu-tcg](https://qemu.gevico.online/tutorial/2026/ch2/qemu-tcg/) | [BV1beApzuEmt](https://www.bilibili.com/video/BV1beApzuEmt/) `QEMU 训练营 \| 专业阶段 QEMU TCG 介绍：二进制动态翻译原理与运行流程（一） [CPU 建模]`；[BV14NAPzjEPW](https://www.bilibili.com/video/BV14NAPzjEPW/) `...（二）` | 已匹配 | 这一章对应两条视频，建议连着看 |
| c. 模拟客户机指令 | [qemu-insn](https://qemu.gevico.online/tutorial/2026/ch2/qemu-insn/) | 暂未找到同名公开视频 | 待发布/待核验 | 可能并入 TCG 系列，也可能后续单独发布 |
| d. 模拟硬件 MMU | [qemu-softmmu](https://qemu.gevico.online/tutorial/2026/ch2/qemu-softmmu/) | [BV1fnQCBEEW2](https://www.bilibili.com/video/BV1fnQCBEEW2/) `QEMU 训练营 \| 专业阶段 QEMU SoftMMU：系统模式下的地址转换与内存访问（一） [CPU 建模]`；[BV1v4X4BPEP3](https://www.bilibili.com/video/BV1v4X4BPEP3/) `...（二）` | 已匹配 | 也是一章拆成两条视频 |
| e. 模拟中断和异常 | [qemu-intr](https://qemu.gevico.online/tutorial/2026/ch2/qemu-intr/) | [BV1CMX4BjE6w](https://www.bilibili.com/video/BV1CMX4BjE6w/) `QEMU 训练营 \| 专业阶段 QEMU TCG 中断与异常 [CPU 建模]` | 已匹配 | 标题里多了 TCG 语境，但主题一致 |
| f. 模拟多核场景 | [qemu-multi-core](https://qemu.gevico.online/tutorial/2026/ch2/qemu-multi-core/) | [BV1Ae9wBjEiD](https://www.bilibili.com/video/BV1Ae9wBjEiD/) `QEMU 训练营 \| 专业阶段 QEMU 多核模拟 [CPU 建模]` | 已匹配 | 直接对上 |
| g. 主板建模流程 | [qemu-machine](https://qemu.gevico.online/tutorial/2026/ch2/qemu-machine/) | [BV1zEX4BzEGR](https://www.bilibili.com/video/BV1zEX4BzEGR/) `QEMU 训练营 \| 专业阶段 QEMU 主板建模流程 [SOC 建模]` | 已匹配 | 直接对上 |
| h. 外设建模流程 | [qemu-hw](https://qemu.gevico.online/tutorial/2026/ch2/qemu-hw/) | [BV1p99cBCEzV](https://www.bilibili.com/video/BV1p99cBCEzV/) `QEMU 训练营 \| 专业阶段 QEMU 外设建模流程 [SOC 建模]` | 已匹配 | 直接对上 |
| i. 时钟系统介绍 | [qemu-clock](https://qemu.gevico.online/tutorial/2026/ch2/qemu-clock/) | [BV1NkSSBrEqM](https://www.bilibili.com/video/BV1NkSSBrEqM/) `QEMU 训练营 \| 专业阶段 QEMU 时钟系统 [SOC 建模]` | 已匹配 | 直接对上 |
| j. GPGPU 原理 | [qemu-gpgpu](https://qemu.gevico.online/tutorial/2026/ch2/qemu-gpgpu/) | 暂未找到同名公开视频 | 待发布/待核验 | 截至今天未在已核到的公开视频里看到同名课 |
| k. PCIe 模拟方法 | [qemu-pcie](https://qemu.gevico.online/tutorial/2026/ch2/qemu-pcie/) | 暂未找到同名公开视频 | 待发布/待核验 | 很可能在专业阶段后续周次发布 |
| l. kernel 运行机制 | [qemu-gpgpu-kernel](https://qemu.gevico.online/tutorial/2026/ch2/qemu-gpgpu-kernel/) | [BV1rpAMznESa](https://www.bilibili.com/video/BV1rpAMznESa/) `QEMU 训练营 \| 超详细干货分享，手把手教你基于 QEMU 调试 Linux 内核` | 相关辅助 | 这是强相关辅助视频，但不是同名讲义课 |
| m. Rust FFI 介绍 | [qemu-rust-ffi](https://qemu.gevico.online/tutorial/2026/ch2/qemu-rust-ffi/) | 暂未找到同名公开视频 | 待发布/待核验 | 目前没核到同名公开视频 |
| n. Rust 建模流程 | [qemu-rust-model](https://qemu.gevico.online/tutorial/2026/ch2/qemu-rust-model/) | 暂未找到同名公开视频 | 待发布/待核验 | 目前没核到同名公开视频 |
| o. Rust 测题开发 | [qemu-rust-test](https://qemu.gevico.online/tutorial/2026/ch2/qemu-rust-test/) | 暂未找到同名公开视频 | 待发布/待核验 | 目前没核到同名公开视频 |

## 项目阶段

| 讲义章节 | 文档 | 对应视频 | 状态 | 备注 |
| --- | --- | --- | --- | --- |
| a. 数字星务计算机 | [qemu-k230](https://qemu.gevico.online/tutorial/2026/ch3/qemu-k230/) | [BV1yUSQBLEPj](https://www.bilibili.com/video/BV1yUSQBLEPj/) `QEMU 训练营 \| 数字星务计算机项目与 RustSBI 介绍 - 洛佳` | 相关辅助 | 主题非常接近，适合作为项目导入 |
| b. 建模 AI 加速卡 | [qemu-cxlemu](https://qemu.gevico.online/tutorial/2026/ch3/qemu-cxlemu/) | 暂未找到同名公开视频 | 待发布/待核验 | 项目阶段通常更晚，可能尚未公开放出 |
| c. 大模型建模外设 | [qemu-agent](https://qemu.gevico.online/tutorial/2026/ch3/qemu-agent/) | 暂未找到同名公开视频 | 待发布/待核验 | 暂未核到 |
| d. 应用跨平台转译 | [qemu-wine-ce](https://qemu.gevico.online/tutorial/2026/ch3/qemu-wine-ce/) | [BV1M6SQBDEpg](https://www.bilibili.com/video/BV1M6SQBDEpg/) `QEMU 训练营 \| Box64 在龙架构上的 x86 兼容性实践 - 刘阳` | 相关辅助 | 方向接近，但不是讲义章节同名视频 |

## 我给你的使用建议

后面你真开始学时，最省脑子的顺序是：

1. 先看“总览入口”里的课程内容介绍
2. 基础阶段严格按表一章一章走
3. 专业阶段优先把 `CPU 建模 / TCG / SoftMMU / 中断异常 / 主板 / 外设 / 时钟` 串起来
4. 遇到 `待发布/待核验` 的章节，就先只看讲义，我后面再帮你补视频更新表

如果你愿意，我下一步可以继续把这份表升级成“学习导航版”：

- 每章后面再补一列 `建议先看讲义还是先看视频`
- 再补一列 `你现在最该关注的源码目录`
