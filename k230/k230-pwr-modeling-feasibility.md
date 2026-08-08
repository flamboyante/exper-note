# K230 PWR 建模可行性与难度分析

记录时间：2026-08-05
分析依据：K230 TRM V0.3.1 第 2.3 章 + SDK 实际代码 + QEMU 现有模型

## 1. 摘要与结论速览

### 一句话结论

**可行性 = 高，难度 = 中**（低于训练营笔记 `k230-qemu-camp-notes.md` §11 给出的"中高"评估）。

### 为什么难度从中高下调到中

训练营笔记给出"中高"评估时（2026-07-01），尚未动态验证 SPL 冷启动路径，也没有 `k230_rmu.c` 作为现成架构模板。本次分析基于以下新证据下调难度：

1. **RMU 模型已落地**：`hw/misc/k230_rmu.c`（425 行）实现了几乎完全相同的寄存器模式（wen strobe + done 位 + 表驱动 + ResettableClass），可直接作为 PWR 的架构模板。
2. **SPL 冷启动 gap 已动态验证**：`k230/spi/coldboot/k230-cold-boot-gap-investigation.md` §4.5 已确认 PWR 空轮询是性能阻塞而非硬阻塞，最小修复只需 latch 一个状态位。
3. **Linux 驱动证据完整**：`k230_pm_domains.c` 第 219-225 行的 `k230_offsets` 表明确列出了 5 个电源域的 4 个寄存器偏移，与 SPL 访问地址完全吻合。
4. **启动路径实际访问寄存器数量有限**：SPL + DDR init 实际只访问 ~10 个寄存器，远少于 TRM 第 2.3 章列出的 30+ 寄存器。

### 三档实施路线速查

| 档位 | 目标 | 文件规模 | 寄存器数 | 副作用 | 验证手段 |
|---|---|---|---|---|---|
| MVP | 解开 SPL 空轮询 | ~300 行 | ~10 | 无 | qtest + SPL 冷启动日志 |
| 标准 | 兼容 Linux pm_domains | ~500 行 | ~30 | 可选 cold-reset | qtest + Linux probe |
| 完整 | 含电源模式状态机 | ~1000 行 | 50+ | 全 | Linux suspend/wakeup |

## 2. PWR 模块在 K230 SoC 中的位置

### 2.1 地址映射

K230 SoC 的"系统控制块"集中在 `0x9100_0000 ~ 0x9110_8000` 区域，由 6 个相邻模块组成（来自 `hw/riscv/k230.c` 第 64-71 行的 `memmap[]`）：

| 模块 | 物理地址 | 大小 | QEMU 当前状态 | QOM 类型（建议） |
|---|---|---|---|---|
| PMU | 0x91000000 | 3KB (0xC00) | `create_unimplemented_device` | `riscv.k230.pmu` |
| RTC | 0x91000C00 | 1KB (0x400) | `create_unimplemented_device` | `riscv.k230.rtc` |
| CMU | 0x91100000 | 1KB (0x1000) | `create_unimplemented_device` | `riscv.k230.cmu` |
| RMU | 0x91101000 | 1KB (0x1000) | **已建模** `TYPE_K230_RMU` | `riscv.k230.rmu` |
| BOOT | 0x91102000 | 1KB (0x1000) | `create_unimplemented_device` | `riscv.k230.boot` |
| PWR | 0x91103000 | 1KB (0x1000) | `create_unimplemented_device` | `riscv.k230.pwr` |
| MAILBOX | 0x91104000 | 1KB (0x1000) | `create_unimplemented_device` | — |

### 2.2 跨模块依赖陷阱

**重要发现**：TRM 第 2.3 章标题是 "Power Management and Low-Power Mode"，但其寄存器实际分布在 PMU、BOOT、PWR 三个模块中，TRM 没有显式标注每个寄存器属于哪个模块，必须交叉验证 SDK 代码才能确定：

| TRM 2.3 寄存器 | TRM 标注 offset | SDK 代码中的实际地址 | 实际所属模块 |
|---|---|---|---|
| SOC_SLEEP_CTL | 0x6c | `0x9110206c`（`BOOT_REG_BASE + 0x6c`） | BOOT |
| SOC_WAKUP_SRC | 0x78 | `0x91102078`（`BOOT_REG_BASE + 0x78`） | BOOT |
| SOC_SLEEP_MASK | 0x118 | `0x91102118`（`BOOT_REG_BASE + 0x118`） | BOOT |
| SYS_CTL_INT0_RAW | 0x90 | `0x91102090`（推断） | BOOT |
| PMU_INT0_TO_CTRL | 0x40 | `0x91000040`（`PMU_REG_BASE + 0x40`） | PMU |
| PMU_INT_DETECT_EN | 0x4c | `0x9100004c`（`PMU_REG_BASE + 0x4c`） | PMU |
| MCTL_PWR_LPI_CTL | 0x9c | `0x9110309c`（`PWR_REG_BASE + 0x9c`） | PWR |
| dpu_PWR_LPI_CTL | 0x108 | `0x91103108`（`PWR_REG_BASE + 0x108`） | PWR |
| SSYS_CTL_GPIO_CTL | 0x168 | `0x91103168`（`PWR_REG_BASE + 0x168`） | PWR |

**SDK 代码证据**（`k230_sdk/src/little/uboot/arch/riscv/cpu/k230/dram.c:66-75`）：

```c
#define PWR_REG_BASE    (0x91103000)
#define PMU_PWR_LPI_CTL                     (PWR_REG_BASE + 0x158)
#define SSYS_CTL_GPIO_CTL                   (PWR_REG_BASE + 0x168)
#define SHRM_PWR_LPI_CTL                    (PWR_REG_BASE + 0x68 )

#define BOOT_REG_BASE   (0x91102000)
#define SOC_SLEEP_CTL                       (BOOT_REG_BASE + 0x6C )
#define SOC_SLEEP_MASK                      (BOOT_REG_BASE + 0x118)

#define PMU_REG_BASE                        (0x91000000)
#define PMU_INT0_TO_CTRL                    (PMU_REG_BASE + 0x0040)
#define PMU_INT_TO_CPU                      (PMU_REG_BASE + 0x0048)
```

**结论**：用户问的"PWR 建模"在狭义上只指 `0x91103000` 的 1KB，但启动路径实际访问的寄存器分布在 PMU/BOOT/PWR 三个模块。本报告聚焦 PWR 本身，PMU/BOOT 作为"跨模块依赖"在第 9 节讨论。

## 3. TRM 第 2.3 章寄存器清单与分类

### 3.1 PWR 模块（0x91103000）完整寄存器表

下表按 TRM 第 2.3 章列出的所有 PWR 寄存器（offset 相对于 0x91103000）：

| Offset | 寄存器名 | 描述 | Reset Value | Write Enable | 启动路径访问 |
|---|---|---|---|---|---|
| 0x68 | SHRM_PWR_LPI_CTL | shrm power control | — | — | ✅ U-Boot change_pll0 |
| 0x9c | MCTL_PWR_LPI_CTL | MCTRL power control | — | — | ✅ DDR init (read-modify-write) |
| 0xa0 | MCTL_CLOCK_SWITCH | MCTRL clock switch | — | — | ✅ DDR init (lpddr4 变体) |
| 0x28 | AI_PWR_LPI_CTL (=PWR_EN) | AI power control | — | yes (wen bit16) | ✅ SPL device_disable |
| 0x2c | AI_PWR_LPI_STAT (=PWR_STAT) | AI power status | 0x0 | no | ✅ SPL device_disable 轮询 |
| 0x3c | DISP_PWR_LPI_CTL | DISP power control | — | yes | ✅ SPL device_disable |
| 0x40 | DISP_PWR_LPI_STAT | DISP power status | 0x0 | no | ✅ SPL device_disable 轮询 |
| 0x7c | VPU_PWR_LPI_CTL | VPU power control | — | yes | ✅ SPL device_disable |
| 0x80 | VPU_PWR_LPI_STAT | VPU power status | 0x0 | no | ✅ SPL device_disable 轮询 |
| 0x108 | DPU_PWR_LPI_CTL | DPU power control | — | yes | ✅ SPL device_disable |
| 0x10c | DPU_PWR_LPI_STAT | DPU power status | 0x0 | no | ✅ SPL device_disable 轮询 |
| 0x18 | CPU1_PWR_LPI_CTL | CPU1 power control | — | yes | ✅ Linux pm_domains |
| 0x1c | CPU1_PWR_LPI_STAT | CPU1 power status | 0x0 | no | ✅ Linux pm_domains |
| 0x158 | PMU_PWR_LPI_CTL | PMU power control | — | yes | ✅ U-Boot change_pll0 |
| 0x168 | SSYS_CTL_GPIO_CTL | gpio wakeup control | — | yes | ✅ U-Boot change_pll0 |
| 0x170 | SSYS_CTL_GPIO_EN0 | gpio wakeup enable | — | yes | ✅ U-Boot change_pll0 |
| 0x174 | SSYS_CTL_GPIO_EN1 | gpio wakeup enable | — | yes | ✅ U-Boot change_pll0 |
| 0x160 | SRAM0_REPAIR_TIM | sram repair timer / REPAIR_STAT | — | no | ✅ Linux repair 共用 |
| 0x60 | MCTL_PWR_TIM | MCTRL power timer | — | no | ❌ 未访问 |
| 0x98 | MCTL_AXI_LPI_TIM | MCTRL ACI LPI timer | — | no | ❌ 未访问 |
| 0x100-0x15c | 其余 *_PWR_TIM/LPI_TIM/CTL/STAT | 各子模块 timer/ctl/stat | — | mixed | ❌ 未访问 |

**统计**：PWR 模块共 ~40 个寄存器（TRM 列出），其中启动路径实际访问 **~14 个**（标记 ✅），其余主要是 timer 参数和未使用子模块的状态寄存器。

### 3.2 PWR_LPI_STAT 位语义

根据 SDK 代码（`k230_pm_domains.c` 第 88-115 行 + SPL `k230_spl.c` 第 97-100 行）和 TRM 第 2.3 章描述，每个子模块的 PWR_LPI_STAT 寄存器位语义为：

| Bit | 名称 | 含义 | 访问 | 谁轮询 |
|---|---|---|---|---|
| 0 | power_off_done | 关电源完成 | RO | SPL device_disable 轮询此位 |
| 1 | power_on_done | 开电源完成 | RO | Linux k230_power_on 轮询此位 |

对应的 PWR_LPI_CTL（写入请求）位语义：

| Bit | 名称 | 含义 | 访问 |
|---|---|---|---|
| 0 | pwr_off_req | 关电源请求（自清除） | W1T |
| 1 | pwr_on_req | 开电源请求（自清除） | W1T |
| 16 | pwr_off_wen | 关电源请求的 write-enable | W1T |
| 17 | pwr_on_wen | 开电源请求的 write-enable | W1T |

**SPL device_disable 写入模式**（`k230_spl.c` 第 84-92 行）：

```c
// disable ai power: 写 0x30001 = bit0(pwr_off_req) | bit16(wen) | bit17(wen)
if (readl(0x9110302c) & 0x2)        // 检查 AI 是否已开电（bit1=on_done）
    writel(0x30001, 0x91103028);    // 写 AI_PWR_LPI_CTL = 0x30001
```

**Linux k230_power_on 写入模式**（`k230_pm_domains.c` 第 91-92 行）：

```c
// 写 PWR_EN = BIT(pwr_on_bit=1) | BIT(pwr_on_wen_bit=17) | BIT(pwr_off_wen_bit=16)
val = BIT(pd->pwr_on_bit) | BIT(pd->pwr_on_wen_bit) | BIT(pd->pwr_off_wen_bit);
writel(val, sysctl_power_base + PM_PWR_EN(pd));
```

**关键观察**：SPL 写 `0x30001`，Linux 写 `0x30002`（bit1 而非 bit0）。两者使用完全相同的 wen strobe 模式，只是请求位不同（off vs on）。

## 4. SDK 实际使用证据

### 4.1 U-Boot SPL `device_disable()` 调用链

**文件**：`k230_sdk/src/little/uboot/board/canaan/common/k230_spl.c:80-110`

```c
static void device_disable(void)
{
    uint32_t value;
    // disable ai power: 检查 STAT bit1(已开电)，写 CTL 关电请求
    if (readl(0x9110302c) & 0x2)
        writel(0x30001, 0x91103028);
    // disable vpu power
    if (readl(0x91103080) & 0x2)
        writel(0x30001, 0x9110307c);
    // disable dpu power
    if (readl(0x9110310c) & 0x2)
        writel(0x30001, 0x91103108);
    // disable disp power
    if (readl(0x91103040) & 0x2)
        writel(0x30001, 0x9110303c);
    // check disable status: 轮询 4 个域的 STAT bit0(关电完成)，最多 1,000,000 次
    value = 1000000;
    while ((!(readl(0x9110302c) & 0x1) || !(readl(0x91103080) & 0x1) ||
        !(readl(0x9110310c) & 0x1) || !(readl(0x91103040) & 0x1)) && value)
        value--;
    // 后续关闭 AI/VPU/DPU/mclk 时钟（在 CMU 模块 0x91100000）
}
```

**调用点**：`spl_board_init_f()` 第 129 行调用 `device_disable()`，发生在 DDR init 之前，是 SPL 最早期的外设初始化之一。

**当前 QEMU 行为**：PWR 是 `create_unimplemented_device`，所有 readl 返回 0。
- 第一个 `if (readl(...) & 0x2)` 全部为假（STAT bit1=0），所以 4 个 writel 不执行
- while 循环条件 `!(0 & 0x1) || ...` = `true || ...`，但 `value` 从 1,000,000 递减
- 循环执行 1,000,000 次后退出（不是死循环，但严重延迟 + 1M 行 unimp 日志）

**最小修复**：让 PWR 模型在 readl(STAT) 时返回 bit0=1（off_done 已 latch），则 while 条件立即为 false，循环 0 次退出。

### 4.2 U-Boot DDR init 中 MCTL_PWR_LPI_CTL 的访问

**文件**：`k230_sdk/src/little/uboot/board/canaan/k230_evb/lpddr3_swap_1600.c:298-300`（同样模式出现在所有板级变体）

```c
reg_read ( 0x9110309c, data  );       // 读 MCTL_PWR_LPI_CTL
 data=data|0x00020000;                 // 设置 bit17（mctl_ddrc_init_done 相关）
reg_write ( 0x9110309c, data  );      // 写回
```

部分 lpddr4 变体还访问 `0x911030a0`（MCTL_CLOCK_SWITCH）。

**TRM 第 2.3 章 application note（行 613）**明确说明：
> When configuration of DDRC is finished, configure MCTL_PWR_LPI_CTL.mctl_ddrc_init_done to 1.

这证实 `0x9110309c` bit17 是 `mctl_ddrc_init_done` 标志位，由 DDR init 完成后写入。

**最小修复**：MCTL_PWR_LPI_CTL 实现为可读可写的存储寄存器（read-modify-write 透传），不需要副作用。MCTL_CLOCK_SWITCH 同理。

### 4.3 U-Boot `change_pll0()` 中的低功耗配置序列

**文件**：`k230_sdk/src/little/uboot/arch/riscv/cpu/k230/dram.c:171-200`

```c
void change_pll0(void)
{
    writel(0x0, PMU_PWR_LPI_CTL);              // 0x91103158: 关闭 PMU 低功耗
    udelay(500000);
    writel(0x0, SSYS_CTL_GPIO_EN0);            // 0x91103170: 禁用 GPIO 唤醒 0
    writel(0x0, SSYS_CTL_GPIO_EN1);            // 0x91103174: 禁用 GPIO 唤醒 1
    writel(1<<3, SSYS_CTL_GPIO_CTL);           // 0x91103168: GPIO 唤醒控制
    writel(0x100000, SHRM_PWR_LPI_CTL);        // 0x91103068: shrm 不 repair
    // 后续配置 RTC alarm、写 SOC_SLEEP_CTL 进入 sleep（BOOT 模块）
}
```

**调用场景**：这是一个 PLL 变频 + sleep 演示函数，**不在标准启动路径上**。标准启动只调用 `device_disable()` 和 DDR init，不会执行 `change_pll0()`。

### 4.4 Linux `k230_pm_domains.c` 的 power on/off 状态机

**文件**：`k230_sdk/src/little/linux/drivers/soc/canaan/k230_pm_domains.c`

#### 4.4.1 寄存器偏移表（关键证据）

第 219-225 行定义了 5 个电源域的寄存器偏移：

```c
static u16 k230_offsets[K230_PM_DOMAIN_MAX][REG_PM_ARRAY_SIZE] = {
    {0x18,  0x1c,  0x18,  0x160},   // CPU1:    PWR_EN, PWR_STAT, REPAIR_EN, REPAIR_STAT
    {0x28,  0x2c,  0x28,  0x160},   // AI:      PWR_EN, PWR_STAT, REPAIR_EN, REPAIR_STAT
    {0x3c,  0x40,  0x3c,  0x160},   // DISP:    PWR_EN, PWR_STAT, REPAIR_EN, REPAIR_STAT
    {0x7c,  0x80,  0x7c,  0x160},   // VPU:     PWR_EN, PWR_STAT, REPAIR_EN, REPAIR_STAT
    {0x108, 0x10c, 0x108, 0x160},   // DPU:     PWR_EN, PWR_STAT, REPAIR_EN, REPAIR_STAT
};
```

**与 SPL 访问地址完全吻合**：
- SPL 读 `0x9110302c`（AI STAT）= `k230_offsets[K230_PM_DOMAIN_AI][1] + 0x91103000`
- SPL 读 `0x91103080`（VPU STAT）= `k230_offsets[K230_PM_DOMAIN_VPU][1] + 0x91103000`
- SPL 读 `0x9110310c`（DPU STAT）= `k230_offsets[K230_PM_DOMAIN_DPU][1] + 0x91103000`
- SPL 读 `0x91103040`（DISP STAT）= `k230_offsets[K230_PM_DOMAIN_DISP][1] + 0x91103000`

这证实了 TRM、U-Boot、Linux 三方对 PWR 寄存器布局的理解一致。

#### 4.4.2 power on/off 流程

Linux `k230_power_on`（第 85-125 行）：
1. 检查 PWR_STAT bit1 是否已开电，是则返回
2. 写 PWR_EN = `BIT(1) | BIT(17) | BIT(16) = 0x30002` 触发开电请求
3. 可选 repair 流程（AI 域）：写 REPAIR_EN bit4=1，轮询共享 REPAIR_STAT(0x160) bit0-2
4. 轮询 PWR_STAT bit1 直到开电完成（最多 1000 次，每次 udelay(1)）

Linux `k230_power_off`（第 145-165 行）：
1. 检查 PWR_STAT bit0 是否已关电，是则返回
2. 写 PWR_EN = `BIT(0) | BIT(17) | BIT(16) = 0x30001` 触发关电请求
3. 轮询 PWR_STAT bit0 直到关电完成

### 4.5 Linux `k230-pmu.c` 的中断/唤醒驱动（PMU 模块）

**文件**：`k230_sdk/src/little/linux/drivers/soc/kendryte/k230-pmu.c`

这个驱动绑定的是 **PMU 模块（0x91000000）**，不是 PWR 模块。它管理 PMU 中断（INT0_0 ~ INT8、RTC_ALARM、RTC_TICK）、唤醒源配置、IO64-IO71 配置。

**对 PWR 建模的影响**：PMU 驱动在 Linux boot 时会 probe，但主要功能是中断/唤醒，不涉及电源域 on/off。如果只做 PWR MVP，PMU 可以保持 `create_unimplemented_device`，PMU 驱动 probe 时会读到 0，但不会卡死。

## 5. QEMU 现有模型可复用性分析

### 5.1 RMU 作为架构模板的可复用部分

**文件**：`my-qemu-camp-2026-k230/hw/misc/k230_rmu.c`（425 行）

| RMU 概念 | RMU 实现 | PWR 对应 | 可复用性 |
|---|---|---|---|
| 寄存器描述结构 | `K230RmuReg { offset, has_we, pairs, n_pairs, reset_val, writable_mask }` | 完全一致 | ✅ 直接复制 |
| 请求/完成配对 | `K230RmuPair { reset_bit, done_bit, self_clear }` | 扩展为双向 (off_req → off_done, on_req → on_done) | ✅ 扩展为双向 |
| write-enable strobe | 高 16 位 wen | 完全一致 | ✅ 直接复制 |
| W1C done 位 | `done_mask` 计算 + `s->regs |= done_mask` | 完全一致 | ✅ 直接复制 |
| 自清除请求位 | `reset_mask` + `new_val &= ~reset_mask` | 完全一致 | ✅ 直接复制 |
| `MemoryRegionOps` | `k230_rmu_read`/`k230_rmu_write` | 完全一致 | ✅ 直接复制 |
| ResettableClass | `rc->phases.hold` | 完全一致 | ✅ 直接复制 |
| VMState | `vmstate_k230_rmu` | 完全一致 | ✅ 直接复制 |
| QOM 类型注册 | `TYPE_K230_RMU = "riscv.k230.rmu"` | `TYPE_K230_PWR = "riscv.k230.pwr"` | ✅ 改名即可 |
| SoC 集成 | `object_initialize_child` + `sysbus_mmio_map` | 完全一致 | ✅ 直接复制 |

### 5.2 PWR 需要差异化的部分

| 差异点 | RMU 做法 | PWR 需求 | 工作量 |
|---|---|---|---|
| 配对方向 | 单向（reset_req → done） | 双向（off_req → off_done, on_req → on_done） | 低：扩展结构体加 on 字段 |
| 每域寄存器数 | 1-2 个（CTRL + 可选 TIM） | 4 个（PWR_TIM + LPI_TIM + PWR_LPI_CTL + PWR_LPI_STAT） | 低：TIM 用存储模式 |
| repair 流程 | 无 | AI 域有 repair，轮询共享 0x160 | 低：MVP 不实现 |
| 副作用传播 | `k230_rmu_propagate` 对 WDT 调 `device_cold_reset` | 类比：PWR off 可触发子设备 cold-reset | 中：MVP 不实现 |
| 寄存器总数 | 21 个 CTRL + 21 个 TIM = 42 | ~40 个 | 相当 |
| DDR 频率切换 | 无 | MCTL_CLOCK_SWITCH 有副作用 | 高：MVP 不实现，透传 |

### 5.3 `k230_rmu_propagate()` 模式如何映射到 PWR

RMU 的副作用传播（`k230_rmu.c` 第 167-180 行）对已建模的 WDT 调 `device_cold_reset()`，对未建模的忽略。PWR 的子设备（AI/VPU/DPU/DISP）当前都未建模，所以即使实现传播也是 no-op。MVP 阶段不实现传播，与 RMU 对未建模外设的处理一致。

## 6. 难度真实来源分析

按"对实施难度的贡献度"从高到低排序：

### 6.1 MCTL_CLOCK_SWITCH / DDR 频率切换语义（贡献度：高，但 MVP 可回避）

- TRM 行 1769 暗示写入会触发 DDR 频率切换硬件序列
- 完整建模需要理解 DDR PHY 变频流程，远超 PWR 本身
- **MVP 策略**：作为存储寄存器透传，不实现副作用

### 6.2 Sleep0/Sleep1/Standby/Powerdown 模式状态机（贡献度：高，但 MVP 可回避）

- TRM 第 2.3.1 节定义 5 种电源模式，模式间转换涉及 PMU/BOOT/PWR 三模块联动
- 完整建模需要模拟电源域状态机、PLL 关闭/重启、唤醒源检测
- **MVP 策略**：完全不实现 sleep 模式。SOC_SLEEP_CTL 在 BOOT 模块，不在 PWR MVP 范围内

### 6.3 副作用传播决策（贡献度：中）

- PWR off 请求是否真的 cold-reset 子设备？
- 当前子设备（AI/VPU/DPU/DISP）都未建模，即使实现传播也是 no-op
- **决策**：MVP 不实现传播；标准模型实现框架但 `power_domains[]` 全为 NULL

### 6.4 跨模块寄存器分布（贡献度：中，主要是认知陷阱）

- TRM 第 2.3 章把 PMU/BOOT/PWR 寄存器混在一起，容易建模错位
- **缓解**：本报告第 2.2 节已给出交叉验证表，建模时严格按 SDK `#define` 确定地址

### 6.5 Linux pm_domains 驱动的兼容性期望（贡献度：中）

- Linux `k230_power_on()` 轮询 PWR_STAT bit1，循环 1000 次
- 如果 PWR 模型不 latch on_done 位，Linux 会超时但通常不卡死
- **缓解**：标准模型实现双向 done latch

### 6.6 双向 done 位状态机（贡献度：低中）

- PWR_STAT 同时承载 off_done (bit0) 和 on_done (bit1)
- 需要扩展 RMU 的 `K230RmuPair` 为双向配对
- 工作量小：只需加一个 `on_done_bit` 字段

### 6.7 寄存器数量（贡献度：低）

- PWR 约 40 个寄存器，RMU 42 个，量级相当
- 表驱动设计让寄存器数量不影响代码复杂度

## 7. 三档实施路线

### 7.1 最小可行模型（MVP）— 解开 SPL 冷启动空轮询

#### 目标
让 SPL `device_disable()` 的 while 循环立即退出，消除 1M 行 unimp 日志和启动延迟。

#### 文件清单

| 文件 | 操作 | 行数估算 |
|---|---|---|
| `hw/misc/k230_pwr.c` | 新建 | ~250-300 |
| `include/hw/misc/k230_pwr.h` | 新建 | ~50 |
| `hw/misc/meson.build` | 编辑（加一行） | +1 |
| `hw/misc/Kconfig` | 编辑（加 config 项） | +3 |
| `hw/riscv/k230.c` | 编辑（替换 unimplemented） | +5/-2 |
| `include/hw/riscv/k230.h` | 编辑（加成员） | +2 |
| `tests/qtest/k230-pwr-test.c` | 新建（可选） | ~150 |

#### 寄存器范围（MVP）

| Offset | 名称 | 行为 |
|---|---|---|
| 0x28 | AI_PWR_LPI_CTL | 写入触发 off_done latch（bit0 → STAT） |
| 0x2c | AI_PWR_LPI_STAT | 读返回 bit0=1（off_done 已 latch） |
| 0x3c | DISP_PWR_LPI_CTL | 同上 |
| 0x40 | DISP_PWR_LPI_STAT | 同上 |
| 0x7c | VPU_PWR_LPI_CTL | 同上 |
| 0x80 | VPU_PWR_LPI_STAT | 同上 |
| 0x108 | DPU_PWR_LPI_CTL | 同上 |
| 0x10c | DPU_PWR_LPI_STAT | 同上 |
| 0x9c | MCTL_PWR_LPI_CTL | 存储寄存器（read-modify-write 透传） |
| 0xa0 | MCTL_CLOCK_SWITCH | 存储寄存器（透传） |

#### 行为设计

- **PWR_LPI_CTL 写入**：解析 wen strobe（bit16/17），如果是 off 请求（bit0=1），立即在对应的 PWR_LPI_STAT 中 latch bit0=1（off_done），并自清除 bit0 请求位
- **PWR_LPI_STAT 读取**：返回当前 latch 值（reset 后为 0，收到 off 请求后为 1）
- **MCTL_PWR_LPI_CTL / MCTL_CLOCK_SWITCH**：纯存储寄存器，read 返回最后写入值，write 透传，reset 值为 0
- **未建模偏移**：`qemu_log_mask(LOG_UNIMP, ...)` + 存储透传（与 RMU 一致）

#### 副作用
无。不调用 `device_cold_reset()`，不传播到子设备。

#### 验证手段

1. **qtest**：`tests/qtest/k230-pwr-test.c`
   - 验证 reset default（STAT=0, CTL=0）
   - 写 CTL=0x30001，验证 STAT 立即变为 0x1
   - 验证 MCTL_PWR_LPI_CTL read-modify-write 透传
2. **SPL 冷启动日志**：
   - 跑 `-machine k230 -bios <SPL>`，验证 `/tmp/k230-spl-unimp.log` 不再有 1M 行 PWR 空轮询
   - 验证 SPL 能继续走到 DDR init

### 7.2 标准模型 — 兼容 Linux pm_domains 驱动

#### 目标
在 MVP 基础上覆盖全部 5 个电源域 + repair 流程，让 Linux `k230_pm_domains.c` probe 和 power on/off 不超时。

#### 增量文件清单

| 文件 | 操作 | 增量行数 |
|---|---|---|
| `hw/misc/k230_pwr.c` | 扩展 | +150-200 |
| `include/hw/misc/k230_pwr.h` | 扩展 | +30 |
| `tests/qtest/k230-pwr-test.c` | 扩展 | +100 |

#### 增量寄存器范围

| Offset | 名称 | 行为 |
|---|---|---|
| 0x18 | CPU1_PWR_LPI_CTL | 双向（off + on）done latch |
| 0x1c | CPU1_PWR_LPI_STAT | bit0=off_done, bit1=on_done |
| 0x68 | SHRM_PWR_LPI_CTL | 存储寄存器 |
| 0x158 | PMU_PWR_LPI_CTL | 存储寄存器 |
| 0x160 | SRAM0_REPAIR_TIM / REPAIR_STAT | repair 请求触发 bit0-2 latch |
| 0x168 | SSYS_CTL_GPIO_CTL | 存储寄存器 |
| 0x170 | SSYS_CTL_GPIO_EN0 | 存储寄存器 |
| 0x174 | SSYS_CTL_GPIO_EN1 | 存储寄存器 |
| 其余 *_PWR_TIM / *_LPI_TIM | 存储寄存器（reset 值按 TRM） | 透传 |

#### 增量行为

- **双向 done latch**：PWR_LPI_CTL 写 on 请求（bit1=1）→ STAT latch bit1=1；写 off 请求（bit0=1）→ STAT latch bit0=1，同时清除 bit1
- **repair 流程**（AI 域）：写 REPAIR_EN bit4=1 → 共享 REPAIR_STAT(0x160) latch bit0-2
- **QOM link 框架**：`K230PwrState` 增加 `DeviceState *power_domains[5]`，通过 `object_property_add_link()` 在 SoC realize 时连接（当前全为 NULL）

#### 验证手段

1. **qtest**：覆盖全部 5 域的 on/off 流程 + repair
2. **Linux boot**：`k230_pm_domains` probe 不超时，dmesg 无 `power on timeout` 错误

### 7.3 完整模型 — 含电源模式状态机

#### 目标
实现 Sleep0/Sleep1/Standby/Powerdown 模式转换 + 中断路由 + 跨模块联动。

#### 增量文件清单

| 文件 | 操作 | 增量行数 |
|---|---|---|
| `hw/misc/k230_pwr.c` | 扩展 | +200-300 |
| `hw/misc/k230_pmu.c` | 新建 | ~300-400 |
| `include/hw/misc/k230_pmu.h` | 新建 | ~60 |
| `hw/misc/k230_boot.c` | 新建 | ~200-300 |
| `include/hw/misc/k230_boot.h` | 新建 | ~40 |

#### 增量功能

- **PMU 模块（0x91000000）**：中断控制器（INT0/INT1 路由到 PLIC）、唤醒源检测、IO64-IO71 配置
- **BOOT 模块（0x91102000）**：SOC_SLEEP_CTL 状态机、SOC_WAKUP_SRC、SYS_CTL_INT0/1
- **PWR 扩展**：与 PMU/BOOT 联动的 sleep 进入/退出流程、PLL 状态联动
- **中断路由**：SYS_CTL_INT0/INT1 → PLIC（需要确认 IRQ 号，当前 `k230.h` 未定义）

#### 验证手段

1. **qtest**：覆盖 sleep/wakeup 状态转换
2. **Linux suspend**：`echo mem > /sys/power/state` + 唤醒测试（受限于 QEMU 单 C908 核环境，可能无法完整验证多核唤醒）

#### 风险

- TRM 对 sleep 状态机的描述偏寄存器级，缺少时序细节
- 实际硅片行为可能与 TRM 有差异
- QEMU 单核环境无法验证多核电源域协作
- **建议**：除非有明确的 Linux suspend/wakeup 验证需求，否则不建议推进到完整模型

## 8. 推荐路线与决策点

### 8.1 推荐路线

**先做 7.1 MVP，验证 SPL 冷启动，再按需扩展到 7.2 标准。**

理由：
1. MVP 工作量小（~300 行），可在 RMU 模板基础上快速完成
2. MVP 直接解决已动态验证的 SPL 空轮询问题，价值明确
3. 标准模型的增量工作（+200 行）可以等 Linux boot 验证需求出现后再做
4. 完整模型的 sleep 状态机受限于 QEMU 单核环境，性价比低

### 8.2 关键决策点

#### 决策 1：PWR off 是否传播到子设备？

| 选项 | 优点 | 缺点 |
|---|---|---|
| MVP 不传播（推荐） | 简单，与 RMU 对未建模外设一致 | 未来子设备建模后需要补 |
| 标准模型传播 | 架构完整 | 当前无子设备可传播，代码是 no-op |

**推荐**：MVP 不传播，标准模型实现框架但 `power_domains[]` 全为 NULL。

#### 决策 2：是否同时建模 PMU/BOOT？

| 选项 | 优点 | 缺点 |
|---|---|---|
| MVP 只建模 PWR（推荐） | 范围可控，解决已验证问题 | Linux PMU 驱动 probe 时读到 0 |
| 标准模型同时建模 PMU/BOOT | 跨模块依赖完整 | 工作量翻倍，且 PMU/BOOT 启动路径非关键 |

**推荐**：MVP 只建模 PWR。PMU 驱动读 0 等于中断禁用，不会卡死。BOOT 模块的 SOC_SLEEP_CTL 只在 `change_pll0()` 演示代码中使用，标准启动路径不访问。

#### 决策 3：是否作为上游 patch series 提交？

| 选项 | 时机 | 考虑 |
|---|---|---|
| 暂不提交 | 现在 | RMU 模块（#9）还在上游 review 中，等它定论 |
| 跟随 RMU 提交 | RMU 合并后 | 作为 follow-up series，复用 RMU 的架构模式 |
| 独立提交 | 随时 | 需要完整的 cover letter + qtest + 文档 |

**推荐**：先在本地验证，待 RMU/CMU 等相邻模块上游定论后再决定。

## 9. 风险与开放问题

### 9.1 TRM 与实际硅片行为可能存在差异

- TRM V0.3.1 是 2024-11-18 版本，可能落后于实际硅片
- SDK 代码是更可信的证据（它直接对应实际运行的固件）
- **缓解**：建模时以 SDK 代码为准，TRM 用于补充寄存器 reset value 和位语义

### 9.2 Linux DTS 中的 `reg` 地址未确认

- `k230_pm_domains.c` 通过 `platform_get_resource(pdev, IORESOURCE_MEM, 0)` 获取 `sysctl_power_base`，具体地址由 DTS 的 `reg` 属性决定
- 本次未找到 K230 Linux DTS 中 `pm_domains` 节点的 `reg` 值
- **推断**：应为 `0x91103000`（与 U-Boot `PWR_REG_BASE` 一致），但需要确认
- **缓解**：MVP 阶段不运行 Linux，标准模型阶段再确认 DTS

### 9.3 MCTL_CLOCK_SWITCH 的 DDR 频率切换副作用

- TRM 行 1769 暗示写入会触发 DDR 变频序列
- DDR init 代码只 read-modify-write，不依赖副作用
- **缓解**：MVP 作为存储寄存器透传，不实现副作用

### 9.4 PWR 模块是否产生中断？

- TRM 2.3 提到 SYS_CTL_INT0/INT1（power domain power-off/up 中断）
- 但这些寄存器在 BOOT 模块（0x91102090/0x911020a0），不在 PWR
- `k230.h` 中 `K230_*_IRQ` 枚举没有为 PWR/PMU 中断定义 IRQ 号
- **结论**：PWR 模块本身不直接产生中断，中断由 BOOT 模块的 SYS_CTL_INTx 路由
- MVP 不需要实现中断

### 9.5 repair 流程的共享 REPAIR_STAT

- `k230_pm_domains.c` 中所有域的 `PWR_REPAIR_STAT` 偏移都是 `0x160`
- TRM 中 `0x160` 是 `SRAM0_REPAIR_TIM`（sram repair timer）
- 命名不一致：SDK 当作 REPAIR_STAT，TRM 当作 REPAIR_TIM
- **缓解**：标准模型实现时按 SDK 语义（STAT）处理，但保留 TRM 名称注释

## 10. 参考资料

### 10.1 TRM 章节指针

| 章节 | 行号（文本版） | 内容 |
|---|---|---|
| 1.3.11 PMU | 333 | PMU 简要介绍 |
| 1.5 Address Space map | 419 | PMU/RTC 地址映射 |
| 2.1 Reset | 556 | 复位（RMU 已建模） |
| 2.2 Clock | 1636 | 时钟（CMU 未建模） |
| 2.3 Power Management | 3677 | **本报告主要依据** |
| 2.3.1 overview | 3685 | 5 种电源模式定义 |
| 2.3.5 register description | 3700+ | 寄存器位域详情 |
| 5.1.3.4 SDRAM Power Saving | 15456+ | DDR 低功耗（不在 PWR 范围） |

### 10.2 SDK 代码位置索引

| 文件 | 行号 | 内容 |
|---|---|---|
| `k230_sdk/src/little/uboot/board/canaan/common/k230_spl.c` | 80-110 | `device_disable()` |
| `k230_sdk/src/little/uboot/board/canaan/common/k230_spl.c` | 129 | `spl_board_init_f()` 调用点 |
| `k230_sdk/src/little/uboot/arch/riscv/cpu/k230/dram.c` | 66-75 | PWR/BOOT/PMU 寄存器宏定义 |
| `k230_sdk/src/little/uboot/arch/riscv/cpu/k230/dram.c` | 171-200 | `change_pll0()` sleep 演示 |
| `k230_sdk/src/little/uboot/board/canaan/k230_evb/lpddr3_swap_1600.c` | 298-300 | DDR init 访问 MCTL_PWR_LPI_CTL |
| `k230_sdk/src/little/linux/drivers/soc/canaan/k230_pm_domains.c` | 64 | `sysctl_power_base` 定义 |
| `k230_sdk/src/little/linux/drivers/soc/canaan/k230_pm_domains.c` | 85-125 | `k230_power_on()` |
| `k230_sdk/src/little/linux/drivers/soc/canaan/k230_pm_domains.c` | 145-165 | `k230_power_off()` |
| `k230_sdk/src/little/linux/drivers/soc/canaan/k230_pm_domains.c` | 219-225 | `k230_offsets` 偏移表 |
| `k230_sdk/src/little/linux/drivers/soc/kendryte/k230-pmu.c` | 1-150 | PMU 中断驱动（PMU 模块） |

### 10.3 QEMU 现有模型对照表

| 文件 | 行号 | 可复用内容 |
|---|---|---|
| `my-qemu-camp-2026-k230/hw/misc/k230_rmu.c` | 1-100 | 头部注释 + 数据结构定义 |
| `my-qemu-camp-2026-k230/hw/misc/k230_rmu.c` | 100-165 | `K230RmuReg`/`K230RmuPair` 表 |
| `my-qemu-camp-2026-k230/hw/misc/k230_rmu.c` | 167-180 | `k230_rmu_propagate()` 副作用模式 |
| `my-qemu-camp-2026-k230/hw/misc/k230_rmu.c` | 182-260 | `k230_rmu_read`/`k230_rmu_write` |
| `my-qemu-camp-2026-k230/hw/misc/k230_rmu.c` | 260-340 | reset_hold + realize + init |
| `my-qemu-camp-2026-k230/hw/misc/k230_rmu.c` | 400-425 | class_init + type_init |
| `my-qemu-camp-2026-k230/include/hw/misc/k230_rmu.h` | 全文 | 状态结构模板 |
| `my-qemu-camp-2026-k230/hw/riscv/k230.c` | 160-197 | `object_initialize_child` 模式 |
| `my-qemu-camp-2026-k230/hw/riscv/k230.c` | 266-534 | SoC realize + MMIO map 模式 |
| `my-qemu-camp-2026-k230/include/hw/riscv/k230.h` | 64-71 | memmap 地址定义 |

### 10.4 相关文档

| 文档 | 路径 | 关系 |
|---|---|---|
| 训练营笔记 | `exper-note/k230/k230-qemu-camp-notes.md` §11 | PMU/PWR 评为"中高"难度（本报告下调到"中"） |
| 冷启动 gap 报告 | `exper-note/k230/spi/coldboot/k230-cold-boot-gap-investigation.md` §4.5 | PWR 空轮询动态验证 |
| 上游模块指南 | `exper-note/k230-new-modules-guide.md` | 11 个 pending patch series，无 PMU/PWR |
| 工作区状态 | `exper-note/workspace-state.md` | 当前主题是 SPI V2，PWR 不是活跃主题 |
