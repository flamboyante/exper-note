# QEMU Camp 2026 工作区状态

最后更新：2026-07-30

本页是工作区当前主题、实施阶段和下一步的唯一状态入口。新的 agent 先读根目录 `AGENTS.md`，再读本页；不要用目录名、历史分支或聊天记录推断当前优先级。

## 当前默认主题：K230 SPI/QSPI V2

| 项目 | 当前状态 |
|---|---|
| 阶段 | V2 Step 1 至 Step 3 已完成；Step 4.0 数据路径纠错已完成并通过 12 项 K230 SSI qtest；Step 4.1 至 Step 4.4 尚未实施。 |
| V2 定义 | 新一轮大版本：将当前 K230 SSI 模型重构为通用 `DW_SSI` 与 K230 SoC 集成。 |
| V1 基线 | V1 基线分支为 `k230-spiv3.4`；该分支和 `upstream-review/current/` 的 v3.4 材料均属于 V1。 |
| V2 源码检查点 | 分支 `k230-V2-patch-spi`，HEAD `c689ac865f`；Step 4.0 已删除普通 enhanced mode phase 和 IDMA 1-4-4 XIP 特判。 |
| 架构决策 | [K230 SPI/QSPI 上游 Review 与 V2 重构决策](./k230/upstream-review/k230-spi-qspi-review-v2-decision-notes.md)；这是 V2 唯一决策入口。 |
| 当前计划 | [Step 4 Plan Final：实例配置与 capability 门控](./k230/upstream-review/k230-spi-qspi-v2-step4-plan-final-instance-configuration.md)。 |
| 当前源码 | [通用 DW SSI 模型](../qemu-camp-2026-k230/hw/ssi/dw_ssi.c)、[K230 machine](../qemu-camp-2026-k230/hw/riscv/k230.c)。 |
| 当前验证 | [K230 SSI qtest](../qemu-camp-2026-k230/tests/qtest/k230-dw-ssi-test.c)。 |
| 关键证据 | [寄存器审计](./k230/spi/k230-spi-qspi-register-audit.md)、[V1 Flash Window study](./k230/spi/k230-spi-qspi-flash-window-study-plan.md)、K230 TRM 与 SDK 驱动（由决策记录按需链接）。 |

### 下一步

下一步从 Step 4.1 开始：建立 `DwSsiConfig` 的完整本地终态、properties、配置校验、动态 FIFO、通用测试机和最小迁移 equality；随后按 enhanced SPI → IDMA → XIP 的顺序逐项门控。上游投稿与本地完整实施分开组织：第一批 series 只覆盖通用 DW SSI Standard SPI PIO、基础 IRQ、K230 三实例和 PLIC 集成，扩展寄存器在对应功能 series 前统一 RAZ/WI。commit 按单一职责逐步完成，发送前再重组；未经用户明确要求，不创建分支、不提交、不推送。

### 证据与范围闸门

- V2 只实现已有多源证据支持的 DWC SSI 子集；不宣称完整覆盖全部 DWC SSI 变体。
- 尚未确认的 capability、实例参数或 K230 quirk 不凭猜测固化；遵循决策记录的证据分类和 Databook 闸门。
- 当前没有因缺少 Databook 而阻塞的架构问题；获得资料后用于验证具体寄存器与 capability 语义。

## 可切换主题

| 主题 | 入口 | 何时切换 |
|---|---|---|
| K230 IOMUX/Pinctrl | [三阶段入门指南](./k230/iomux/k230-iomux-three-stage-guide.md)、[U-Boot 验证](./k230/iomux/uboot-iomux-unimplemented-repro.md) | 任务涉及 pinctrl、IOMUX 上游 review 或启动验证。 |
| K230 启动链与镜像验证 | [K230 专题索引](./k230/README.md)、[`k230-boot-assets/` README](../k230-boot-assets/README.md) | 任务涉及 direct boot、U-Boot、Linux 或镜像选择。 |
| 训练营通用学习 | [课程地图](./camp/qemu-camp-2026-doc-video-map.md) | 任务是课程笔记、QEMU 基础模型或非 K230 专题。 |
| G233 GPIO | [G233 GPIO 笔记](./g233-gpio-study.md) | 任务明确指向 G233 板级 GPIO。 |

切换默认主题时，修改本页的标题、默认主题表和“下一步”，并更新最后更新时间；保留其他主题作为路由入口。

## 维护边界

- 状态、优先级、下一步、阻塞项：更新本页。
- 架构结论、patch 拆分、证据分类：更新对应专题决策记录。
- 实际实现和测试结果：更新对应源码仓库、测试或实验记录，并在本页只保留简短状态链接。
- 不把 V1 的历史 cover letter、patch 学习手册或分支名复制为 V2 进度。
