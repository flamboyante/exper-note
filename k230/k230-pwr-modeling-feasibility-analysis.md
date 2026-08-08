# K230 PWR 建模可行性与难度分析 — 实施计划

## 任务目标

根据 K230 TRM V0.3.1 第 2.3 章 "Power Management and Low-Power Mode"，结合 SDK 实际
使用证据（U-Boot SPL `device_disable()`、Linux `k230_pm_domains.c` / `k230-pmu.c`）和
QEMU 现有 `k230_rmu.c` 架构模板，输出一份**可行性与难度分析报告**，回答以下问题：

1. PWR 模块需要建模哪些寄存器？哪些是启动路径实际访问的？
2. 现有 RMU 模型能否直接作为架构模板？哪些地方需要差异化？
3. 难度的真实来源是什么（寄存器数量？状态机？跨模块依赖？副作用传播？）
4. 最小可行模型 vs 完整模型的边界在哪里？
5. 推荐的实施路线（含文件清单、规模估算、验证手段）

## 当前状态分析（Phase 1 探索结论）

### 资料与代码基础

| 资料/代码 | 路径 | 价值 |
|---|---|---|
| TRM V0.3.1 文本版 | `exper-note/k230/reference/K230_Technical_Reference_Manual_V0.3.1_20241118.txt` | 第 2.3 章（行 3677+）是权威寄存器表 |
| TRM V0.3.1 PDF | `docs/learning-guides/qemu-startup-param/K230_Technical_Reference_Manual_V0.3.1_20241118.pdf` | 原始 PDF，1220 页 |
| U-Boot SPL device_disable | `k230_sdk/src/little/uboot/board/canaan/common/k230_spl.c:80-110` | 实际轮询 PWR 状态位的代码 |
| U-Boot DDR init PWR 访问 | `k230_sdk/src/little/uboot/board/canaan/k230_evb/lpddr3_swap_1600.c:298-300` 等 | DDR training 中读写 `0x9110309c/0xa0` |
| U-Boot dram.c PWR 宏 | `k230_sdk/src/little/uboot/arch/riscv/cpu/k230/dram.c:66-75` | PWR_REG_BASE、PMU_PWR_LPI_CTL 等定义 |
| Linux PMU 中断驱动 | `k230_sdk/src/little/linux/drivers/soc/kendryte/k230-pmu.c` | PMU 模块（0x91000000）的 Linux 驱动 |
| Linux 电源域驱动 | `k230_sdk/src/little/linux/drivers/soc/canaan/k230_pm_domains.c` | PWR 模块的 Linux 驱动，含 power on/off 轮询 |
| QEMU RMU 模板 | `my-qemu-camp-2026-k230/hw/misc/k230_rmu.c` (~425 行) | 最接近的架构模板 |
| QEMU SoC 集成 | `my-qemu-camp-2026-k230/hw/riscv/k230.c:444-445` | 当前 `create_unimplemented_device("pwr", 0x91103000, 0x1000)` |
| 冷启动 gap 报告 | `exper-note/k230/spi/coldboot/k230-cold-boot-gap-investigation.md` §4.5 | 已动态验证 PWR 空轮询问题 |
| 训练营笔记 | `exper-note/k230/k230-qemu-camp-notes.md` §11 | PMU/PWR 评为"中高"难度/"高"优先级 |

### 关键技术发现

#### 发现 1：TRM 第 2.3 章跨三个 SoC 模块

TRM 第 2.3 章标题是 "Power Management and Low-Power Mode"，但其寄存器实际分布在
三个不同的 SoC 模块中（这一点在 TRM 中没有显式标注，需要交叉验证 SDK 代码才能发现）：

| 模块 | 物理地址 | 大小 | TRM 2.3 中的寄存器 | QEMU 当前状态 |
|---|---|---|---|---|
| PMU | 0x91000000 | 3KB | PMU_INT0_TO_CTRL(0x40)、PMU_INT1_TO_CTRL(0x44)、PMU_INT_TO_CPU(0x48)、PMU_INT_DETECT_EN(0x4C)、PMU_INT_DETECT_CLR(0x54)、PMU_SYSCTRL_REG(0x78)、PMU_NORMAL_TIMER_VAL(0xA0) | unimplemented |
| BOOT | 0x91102000 | 1KB | SOC_SLEEP_CTL(0x6c)、SOC_WAKUP_SRC(0x78)、SOC_SLEEP_MASK(0x118)、SYS_CTL_INT0_RAW(0x90)、SYS_CTL_INT0_EN(0x94)、SYS_CTL_INT0_STAT(0x98)、SYS_CTL_INT1_RAW(0xA0)、SYS_CTL_INT1_EN(0xA4) | unimplemented |
| PWR | 0x91103000 | 1KB | 所有 *_PWR_LPI_CTL/STAT/TIM/LPI_TIM（MCTL/DPU/hi/ls/sec/ISP/PMU 各一组，每组 4 个寄存器）、SSYS_CTL_GPIO_CTL(0x168)、SSYS_CTL_GPIO_EN0(0x170)、SSYS_CTL_GPIO_EN1(0x174) | unimplemented |

**结论**：用户问的"PWR 建模"在狭义上只指 0x91103000 的 1KB，但启动路径实际访问
的寄存器分布在 PMU/BOOT/PWR 三个模块。最小可行模型应聚焦 PWR 本身，但分析报告
必须说明跨模块依赖。

#### 发现 2：PWR 模块的实际寄存器使用模式（来自 SDK 代码）

**PWR 模块（0x91103000）的实际访问点**（按调用路径分类）：

| 调用路径 | 访问的 PWR 偏移 | 寄存器名（按 TRM 2.3） | 行为 |
|---|---|---|---|
| SPL `device_disable()` | 0x28, 0x2c, 0x3c, 0x40, 0x7c, 0x80, 0x108, 0x10c | AI/VPU/DISP/DPU 的 PWR_LPI_CTL 与 PWR_LPI_STAT | 写 0x30001 关电源，轮询 STAT bit0 等关断完成 |
| DDR init（所有板级变体） | 0x9c, 0xa0 | MCTL_PWR_LPI_CTL、MCTL_CLOCK_SWITCH | read-modify-write 设置 bit17（mctl_ddrc_init_done 相关） |
| U-Boot `change_pll0()` | 0x68, 0x158, 0x168, 0x170, 0x174 | SHRM_PWR_LPI_CTL、PMU_PWR_LPI_CTL、SSYS_CTL_GPIO_CTL/EN0/EN1 | 写 0 关闭低功耗，配置 GPIO 唤醒 |
| Linux `k230_pm_domains.c` | 动态（由 DTS reg_offset 决定） | 各域的 PWR_EN/PWR_STAT/PWR_REPAIR_EN/PWR_REPAIR_STAT | 写 PWR_EN 触发上下电，轮询 PWR_STAT bit1(ON)/bit0(OFF) |

**PWR_STAT 位语义**（来自 `k230_pm_domains.c` 第 88-115 行 + SPL 第 97-100 行）：
- bit 0 = power-off done（关闭完成，SPL 轮询此位）
- bit 1 = power-on done（开启完成，Linux 轮询此位）
- 高 16 位 = write-enable strobe（与 RMU 完全一致的 wen 模式）

#### 发现 3：RMU 模板几乎完美匹配 PWR 需求

RMU 的 `K230RmuPair` 结构（reset_bit + done_bit + self_clear）和 `K230RmuReg` 表驱动
设计，可以直接映射到 PWR 的 (pwr_off_bit → pwr_off_done_bit) 配对关系：

| RMU 概念 | PWR 对应 |
|---|---|
| reset_bit (请求位) | pwr_off_bit / pwr_on_bit |
| done_bit (完成位) | PWR_STAT bit0 (off done) / bit1 (on done) |
| self_clear (自动清除) | off 请求自清除（参照 RMU CPU0 行为） |
| write-enable (高 16 位) | 完全一致（PWR 也用 wen strobe） |
| W1C done | 完全一致 |
| `k230_rmu_propagate()` 副作用 | 类比：PWR off 可触发 `device_cold_reset()` |

**差异点**：
1. PWR 每个域有 4 个寄存器（PWR_TIM + LPI_TIM + PWR_LPI_CTL + PWR_LPI_STAT），RMU 每组只有 1-2 个
2. PWR_STAT 同时承载 off-done (bit0) 和 on-done (bit1)，需要双向配对
3. PWR 涉及约 7 个子模块 × 4 寄存器 = ~28 个寄存器，比 RMU 的 21 个略多但量级相当
4. PWR 的 `MCTL_CLOCK_SWITCH` 等寄存器有 DDR 频率切换副作用，语义比 RMU 复杂

#### 发现 4：PWR 空轮询是已验证的冷启动阻塞点

`exper-note/k230/spi/coldboot/k230-cold-boot-gap-investigation.md` §4.5 已动态验证：
- SPL `device_disable()` 在 PWR_STAT bit0 恒为 0 时循环 1,000,000 次
- 不是硬阻塞（计数器会耗尽），但产生 1,000,399 行 unimp 日志和严重启动延迟
- 最小修复：让 PWR 模型在收到 off 请求后立即在 STAT 中 latch bit0=1

#### 发现 5：上游 patch 现状

`exper-note/k230-new-modules-guide.md` 列出的 11 个上游 pending patch series 中，
**没有 PMU/PWR 模块**。最近的模块是 #9 RMU（reset only，TRM §2.1）。
**目前无人向上游提交 K230 PMU/PWR 模型**，存在空白窗口。

## 报告结构与待写入内容

分析报告将写入 `exper-note/k230/k230-pwr-modeling-feasibility.md`，结构如下：

### 1. 摘要与结论速览
- 一句话结论：可行性 = 高，难度 = 中（低于训练营笔记给出的"中高"评估）
- 三档实施路线速查表（最小 / 标准 / 完整）

### 2. PWR 模块在 K230 SoC 中的位置
- 地址映射表（PMU/BOOT/PWR 三模块对比）
- 与 RMU/CMU 的邻接关系
- 跨模块依赖图（TRM 2.3 寄存器分布的"陷阱"）

### 3. TRM 第 2.3 章寄存器清单与分类
- 完整寄存器 offset 表（按模块分组）
- 每个寄存器的 access type / reset value / write_enable 标注
- 标记"启动路径实际访问"的寄存器（来自 SDK 代码证据）

### 4. SDK 实际使用证据
- 4.1 U-Boot SPL `device_disable()` 调用链与轮询模式
- 4.2 U-Boot DDR init 中 MCTL_PWR_LPI_CTL 的 read-modify-write
- 4.3 U-Boot `change_pll0()` 中的低功耗配置序列
- 4.4 Linux `k230_pm_domains.c` 的 power on/off 状态机
- 4.5 Linux `k230-pmu.c` 的中断/唤醒驱动（PMU 模块，非 PWR）

### 5. QEMU 现有模型可复用性分析
- 5.1 RMU 作为架构模板的可复用部分（K230RmuReg/K230RmuPair/wen/W1C/ResettableClass）
- 5.2 PWR 需要差异化的部分（双向 done、4 寄存器/域、DDR 频率切换副作用）
- 5.3 `k230_rmu_propagate()` 模式如何映射到 PWR 的 device_cold_reset 副作用

### 6. 难度真实来源分析
按"难度贡献度"排序：
1. **跨模块寄存器分布**（PMU/BOOT/PWR 在 TRM 2.3 中混合，容易建模错位）— 中
2. **双向 done 位状态机**（off-done 和 on-done 共存于 PWR_STAT）— 低中
3. **副作用传播决策**（PWR off 是否真的 cold-reset 子设备）— 中
4. **MCTL_CLOCK_SWITCH / DDR 频率切换语义**— 高（但启动路径可不实现）
5. **Linux pm_domains 驱动的兼容性期望**— 中（轮询模式与 RMU 一致）
6. **Sleep0/Sleep1/Standby/Powerdown 模式状态机**— 高（但启动路径不需要）

### 7. 三档实施路线

#### 7.1 最小可行模型（MVP，解开 SPL 冷启动空轮询）
- 文件：`hw/misc/k230_pwr.c` + `include/hw/misc/k230_pwr.h`
- 范围：仅 PWR 模块（0x91103000，1KB），不涉及 PMU/BOOT
- 寄存器：AI/VPU/DPU/DISP 的 PWR_LPI_CTL + PWR_LPI_STAT（8 个）+ MCTL_PWR_LPI_CTL(0x9c) + MCTL_CLOCK_SWITCH(0xa0)
- 行为：收到 off 请求后零延迟 latch STAT bit0=1，让 SPL 轮询立即退出
- 副作用：无（不真的 cold-reset 子设备，与 RMU 对未建模外设的处理一致）
- 规模估算：~250-350 行 C（参照 RMU 425 行的简化版）
- 验证：qtest 验证 reset default + 读写语义；SPL 冷启动日志验证空轮询消失

#### 7.2 标准模型（兼容 Linux pm_domains 驱动）
- 在 MVP 基础上增加：
  - 全部 7 个子模块的 PWR_LPI_CTL/STAT/TIM/LPI_TIM（~28 个寄存器）
  - SSYS_CTL_GPIO_CTL/EN0/EN1（GPIO 唤醒控制）
  - 双向 done（on/off 都 latch）
  - 可选：通过 QOM link 让 PWR off 真的触发已建模子设备的 `device_cold_reset()`
- 规模估算：~450-600 行 C
- 验证：qtest 全寄存器覆盖；Linux boot 时 `k230_pm_domains` probe 不卡死

#### 7.3 完整模型（含电源模式状态机）
- 在标准模型基础上增加：
  - PMU 模块（0x91000000）中断/唤醒寄存器
  - BOOT 模块 SOC_SLEEP_CTL/SOC_WAKUP_SRC/SOC_SLEEP_MASK
  - Sleep0/Sleep1/Standby/Powerdown 模式转换状态机
  - SYS_CTL_INT0/INT1 中断路由到 PLIC
- 规模估算：~800-1200 行 C，可能拆分为 3 个文件（k230_pmu.c / k230_pwr.c / k230_boot.c）
- 验证：Linux suspend-to-ram / wakeup 测试（受限于 QEMU 单核环境，可能无法完整验证）

### 8. 推荐路线与决策点
- 推荐：先做 7.1 MVP，验证 SPL 冷启动，再按需扩展到 7.2
- 决策点 1：PWR off 是否传播到子设备？建议 MVP 阶段不传播，与 RMU 对未建模外设的处理一致
- 决策点 2：是否同时建模 PMU/BOOT？建议 MVP 阶段不建模，但在 PWR 头文件中预留注释
- 决策点 3：是否作为上游 patch series 提交？建议先在本地验证，待 RMU/CMU 等相邻模块上游定论后再决定

### 9. 风险与开放问题
- TRM V0.3.1 与实际硅片行为可能存在差异（SDK 代码是更可信的证据）
- `k230_pm_domains.c` 中 `reg_offset` 由 DTS 决定，需要确认 SDK DTS 中的实际 offset 值
- MCTL_CLOCK_SWITCH 的 DDR 频率切换副作用是否需要建模，取决于是否追求 Linux 完整 suspend
- PWR 模块是否产生中断？TRM 2.3 提到 SYS_CTL_INT0/INT1 但路由到 PLIC 的 IRQ 号未在 `k230.h` 中定义

### 10. 参考资料
- TRM 章节指针表
- SDK 代码位置索引
- QEMU 现有模型对照表

## 假设与决策

1. **分析范围**：以 PWR 模块（0x91103000）为核心，PMU/BOOT 作为"跨模块依赖"分析，不深入建模细节
2. **证据优先级**：SDK 实际代码 > TRM 文档 > 推断。当 SDK 与 TRM 矛盾时以 SDK 为准
3. **难度评估视角**：从"训练营 PR 可交付性"角度评估，不是"完整硅片仿真"角度
4. **不修改任何代码**：本任务是分析报告，唯一产出是 `exper-note/k230/k230-pwr-modeling-feasibility.md`
5. **报告语言**：中文（与 `k230-qemu-camp-notes.md` 等同仓文档一致）
6. **报告规模**：预计 600-900 行 markdown，含表格和代码引用

## 验证步骤

报告完成后通过以下方式验证质量：

1. **完整性检查**：报告是否覆盖了第 1-10 节所有结构
2. **证据链检查**：每个"实际访问"标注是否能在 SDK 代码中找到对应行号
3. **交叉验证**：TRM 寄存器 offset 是否与 SDK `#define` 完全一致
4. **可执行性检查**：三档路线的文件清单、规模估算、验证手段是否具体可执行
5. **与现有文档一致性**：与 `k230-qemu-camp-notes.md` §11 的难度评估是否吻合或给出合理的修订理由

## 执行步骤

1. 创建 `exper-note/k230/k230-pwr-modeling-feasibility.md`
2. 按第 1-10 节结构写入分析内容
3. 嵌入 TRM 寄存器表、SDK 代码引用、QEMU 模型对照表
4. 自检完整性、证据链、交叉验证
5. 返回最终报告路径给用户
