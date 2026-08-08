# QEMU Camp 2026 工作区状态

最后更新：2026-08-08

本页是工作区当前主题、实施阶段和下一步的唯一状态入口。新的 agent 先读根目录 `AGENTS.md`，再读本页；不要用目录名、历史分支或聊天记录推断当前优先级。

## 当前默认主题：K230 SPI/QSPI V2（Standard PIO series V3 待发送）

| 项目 | 当前状态 |
|---|---|
| 阶段 | V2 重构完成；Standard PIO 5-patch 系列 V2 已发送并收齐上游意见，V3 已重写完成（14 项 qtest），尚未发送。 |
| V2 定义 | 新一轮大版本：将当前 K230 SSI 模型重构为通用 `DWC_SSI` 与 K230 SoC 集成。 |
| V1 基线 | V1 基线分支为 `k230-spiv3.4`；该分支和 `upstream-review/current/` 的 v3.4 材料均属于 V1。 |
| 已发送 V2 | 分支 `origin/T_v2-5patch`，tip `b0fc2d4893`，base `b428fe0362`；`[PATCH v2 0/5]` 于 2026-08-02 03:28 (+0800) 发送，[lore 线程](https://lore.kernel.org/qemu-devel/20260801192848.30606-1-flamboyant.h.01@gmail.com/T/)。 |
| 当前 V3 源码 | 分支 `T_v3-5patch`（当前 checkout），HEAD `fef8beff7e`，base 与 V2 相同；五个 commit 已就地重写完成，尚未推送、尚未发送。 |
| 架构决策 | [K230 SPI/QSPI 上游 Review 与 V2 重构决策](./k230/upstream-review/k230-spi-qspi-review-v2-decision-notes.md)；这是 V2 唯一决策入口。 |
| 当前计划 | [V3 改动计划（上游评审意见落实）](./k230/spi/k230-spi-v3-review-plan.md)；V2 批次范围见 [Step 4 Plan Final V1.3](./k230/upstream-review/k230-spi-qspi-v2-step4-plan-final-instance-configurationV1.3.md)。 |
| 当前源码 | [通用 DWC SSI 模型](../qemu-camp-2026-k230/hw/ssi/dwc_ssi.c)、[K230 machine](../qemu-camp-2026-k230/hw/riscv/k230.c)。 |
| 当前验证 | [K230 DWC SSI qtest](../qemu-camp-2026-k230/tests/qtest/k230-dwc-ssi-test.c)，14 项；V3 cover letter 声明构建、qtest、checkpatch 与 `git diff --check` 均通过。 |
| 待提交文档 | V3 review plan、[V3 cover letter](./k230/upstream-review/k230-spi-qspi-v2-5patch-cover-letter.md)、[V2/V3 双语对比稿](./k230/upstream-review/k230-spi-qspi-v3-5patch-commit-cover-letter-bilingual-comparison.md)、[b4 发送指南](./k230/upstream-review/k230-upstream-b4-email-send-guide.md)（exper-note 内均未跟踪）。 |
| 关键证据 | [寄存器审计](./k230/spi/k230-spi-qspi-register-audit.md)、[V1 Flash Window study](./k230/spi/k230-spi-qspi-flash-window-study-plan.md)、K230 TRM 与 SDK 驱动（由决策记录按需链接）。 |

### 下一步

1. 按 [V3 review plan](./k230/spi/k230-spi-v3-review-plan.md) 验证步骤复核 V3：逐 patch 构建、checkpatch、残留搜索、参考链接可达性。
2. 用 b4 发送 `[PATCH v3 0/5]`（新顶层线程，不 `In-Reply-To` V2；抄送 Bin Meng、Chao Liu、Anirudh Srinivasan）。
3. 发送后回到 V2 后续批次：enhanced、IDMA、XIP 为独立 follow-up series，不进入本轮。

### 主要时间节点

| 时间 (+0800) | 事件 |
|---|---|
| 07-29 ~ 07-31 | V2 Step 4 计划 V1.1 → V1.3 迭代；07-31 最后更新状态页（本页此前的内容）。 |
| 08-02 02:25 ~ 02:36 | V2 五个 commit 完成于 `T_v2-5patch`（base `b428fe0362`）。 |
| 08-02 03:28 | `[PATCH v2 0/5] hw/ssi: Add DesignWare SSI standard PIO support for K230` 发送至 qemu-devel。 |
| 08-03 | Chao Liu（02:10 UTC）要求 databook 公开链接；当日回复公开资料清单（K230 TRM、Intel Arria 10 TRM、Linux 驱动与 binding）。 |
| 08-05 | Anirudh（8/4 18:38 UTC）指出 dw-apb-ssi 与 dwc-ssi 的 TMOD 位布局不同，建议命名统一 `DWC_SSI`/`dwc_ssi.c`；当天完成双方确认，定下 V3 必改项。 |
| 08-07 | 检查 b4 0.13.0，撰写 [b4 发送指南](./k230/upstream-review/k230-upstream-b4-email-send-guide.md)。 |
| 08-08 10:43 ~ 10:48 | V3 五个 commit 重写完成于 `T_v3-5patch`（HEAD `fef8beff7e`）：DWC 命名统一、补充公开参考链接与三个 `Suggested-by`、修正 EEPROM-read / TX-only / RX FIFO overrun / RX-only dummy、`SPI_FRF` capability suppression、九路 IRQ 且 DONE/AXIE 恒低。 |
| 08-08 | V3 review plan、V3 cover letter、V2/V3 双语对比稿备好（exper-note 未跟踪、未提交）。 |
| 下一步 | V3 校验 → b4 发送 → 等待上游 review。 |

## 最近完成：全 canmv 冷启动验证（2026-08-07）

| 项 | 状态 |
|---|---|
| 目标 | U-Boot 与 Linux 统一 canmv 板级（独立 dts 源文件、同一板级语义），QEMU 零改动，独立于上次 EVB 混搭验证 |
| 结果 | **全流程通过**：BootROM → SPL（SPI NOR）→ U-Boot（Model `kendryte k230 canmv (qemu cold boot)`）→ OpenSBI → Linux 5.10.4 → initramfs shell `~ #`；门禁三项全过 |
| SDK 分支 | `k230-coldboot-canmv-qemu`（v2.0 `7e302f733` 派生） |
| 关键改动 | canmv `board.c` 删除 boot-mode 覆盖（weak 读 `SOC_BOOT_CTL`=0 → NOR）；`CONFIG_K230_PUFS` 注释（走软件 SHA）；新增 uboot/linux 两侧 `k230_canmv_qemu.dts` + uboot `k230_canmv_qemu_defconfig`；`k230.dtsi`/`k230_evb.dtsi` 还原 SDK 原始 |
| 已知问题 | **Linux 侧 spi0 曾因 clock 挂死（已修复，非 QEMU 缺陷）**：SDK 原始 dts 的 spi0 clock（`canaan,k230-clk-composite`）在 QEMU 上 PLL 未建模 → rate=0 → `BAUDR.SCKDV=0` → V2 `dw_ssi` 按 TRM 拒传 → probe 挂死；已给 spi0 换 fixed-clock(50MHz)+`interrupt-names` 修复，Linux 下 dw_spi/spi-nor/MTD 正常（与 V2 cover letter 一致）。V1 模型无 SCKDV 门控掩盖了同问题。其余：U-Boot SPI NOR soft reset 报 -524（容错继续）；GUEST_ERROR/hardlock/pll unlock 为未建模域预期噪音（观察项） |
| 脚本 | `toolchains/mkcanmv_qemu_flash.py`（装配+逐段大小校验）、`toolchains/run_canmv_qemu.sh`（端到端）、`toolchains/pkg_uboot.sh`（打包） |
| 文档 | [冷启动专题](./k230/spi/coldboot/)、[博客记录](./k230/spi/coldboot/k230-coldboot-canmv-qemu-blog.md)、[resource README](../resource/README.md) |
| 计划 | [k230-coldboot-canmv-qemu-validation-plan.md（v4）](./k230/spi/coldboot/k230-coldboot-canmv-qemu-validation-plan.md) |
| 遗留 | Linux dts 恢复 `&spi0` okay 需等 QEMU `dw_ssi` 中断模型补全（V2 follow-up）；linux Makefile `_zicsr_zifencei` hack、`cache.c`/`start.S` 不入本分支，仍在工作区 |

## 最近完成：K230 SPI Flash 冷启动验证（2026-08-05）

| 项 | 状态 |
|---|---|
| 目标 | SPL 从 SPI Flash 冷启动（CPU reset → BootROM → SPL → U-Boot），范围收缩到"能读 U-Boot 镜像"即达成 |
| 结果 | 已完成：U-Boot 2022.10 启动到 `K230#` shell，`SF: Detected w25q256 ... 32 MiB` |
| 全流程 | **已完成：BootROM → SPL → U-Boot → OpenSBI → Linux initramfs shell**（官方 Xuantie 工具链重建 SPL/U-Boot；内核两条已验证路径：yocto 6.18.28、SDK 5.10.4 标准 PTE 重编版，见 [k230-cold-boot-from-zero.md](./k230/spi/coldboot/k230-cold-boot-from-zero.md)） |
| 官方工具链 | `toolchains/`（SDK 外）+ `/opt/toolchain` 软链接；Xuantie V2.6.0 不支持 `_zicsr_zifencei` march（linux Makefile hack 需回退）；canmv SPL 固定走 MMC，冷启动用 **k230_evb** defconfig |
| QEMU 改动 | 分支 `k230-coldboot-validation`，commit `7730876ca5`（`k230_coldboot_boot()`：解 32-bit swap → 验 K230 magic → 拆 528B 头 → SPL 载 0x80300000；dw_ssi CTRLR0 SPI_FRF 可写掩码） |
| SDK 侧临时改动 | `CONFIG_K230_PUFS` 注释（走软件 sha256）、DTS bus-width 8→1、spi0 okay、`mhcr`/`mcor` 数字 CSR（本地 binutils 权宜） |
| 复现命令 | `qemu-system-riscv64 -M k230 -nographic -mtdblock /tmp/k230-coldboot-spi.img`（镜像与 U-Boot 构建产物在 `/tmp`，WSL 重启即失） |
| 计划文档 | [k230-coldboot-merge-plan.md（v3）](./k230/spi/coldboot/k230-coldboot-merge-plan.md) |
| 遗留 | 三处调试打印未清理（k230.c L649/699、designware_spi.c L1179、k230_img.c L472）；官方 linux_system.bin 布局（7MB 分区）压缩包大小待验证（boot-assets 已有 SDK 5.10.4 标准 PTE 重编版，但 26.7MB gzip 大概率超 7MB）；canmv_qemu dts 如需实际启动需改 canmv boot mode |

### 证据与范围闸门

- V2/V3 只实现已有多源证据支持的 DWC SSI 子集；不宣称完整覆盖全部 DWC SSI 变体。
- V3 保持 Standard-only：不加入 enhanced、IDMA、DONE/AXIE 事件源、XIP 正路径、trace event 和 `MAINTAINERS` 修改；dw-apb-ssi 变体有意不实现。
- 尚未确认的 capability、实例参数或 K230 quirk 不凭猜测固化；遵循决策记录的证据分类和 Databook 闸门。Synopsys databook 非公开，只作内部核对，不进上游引用。

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
