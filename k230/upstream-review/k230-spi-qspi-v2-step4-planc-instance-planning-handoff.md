# K230 SPI/QSPI V2 Step 4 Plan C（实例配置）计划编写任务单

记录日期：2026-07-29

> **历史状态：已完成并停止使用。** 本文只记录 Plan C 的计划编写过程，其中关于 XIP 寄存器共享、TXU 归属和迁移边界的中间判断不得继续执行。Step 4 唯一入口是 [Plan Final](k230-spi-qspi-v2-step4-plan-final-instance-configuration.md)。

本文不是 Step 4 的最终实施计划，而是当时编写 Plan C 时的任务入口。最终计划现已统一为：

`k230-spi-qspi-v2-step4-plan-final-instance-configuration.md`

文档格式参考：

`k230-spi-qspi-v2-step3-hi-sys-decoupling.md`

## 1. 当前检查点

- QEMU 仓库：`qemu-camp-2026-k230/`
- 当前分支：`k230-V2-patch-spi`
- 当前 HEAD：`189638cdf4`（格式修改）
- Step 1、Step 2 已完成并通过编译与 qtest。
- Step 3 已完成：通用 DW SSI 与 K230 HI_SYS 已通过 `xip-enable` GPIO 解耦，用户明确要求本轮不再处理 Step 3。
- 当前任务仅规划 Step 4：把控制器硬编码参数改为通用实例配置和 capability。
- 不修改源码，不创建分支，不提交，不推送。

## 2. 已确认的 Step 4 方向

### 2.1 架构边界

- 保留单一通用模型 `TYPE_DW_SSI` / `DwSsiState`。
- 默认不增加 `TYPE_K230_DW_SSI` wrapper。
- K230 machine 负责为三个实例设置配置、映射地址和连接外围设备。
- 通用模型负责解释配置、创建内部资源、实现寄存器和数据通路行为。

### 2.2 配置表达

推荐在 `DwSsiState` 内增加集中配置结构，例如 `DwSsiConfig`，并通过独立 QOM properties 填充。最终字段至少覆盖：

```c
typedef struct DwSsiConfig {
    uint32_t num_cs;
    uint32_t fifo_depth;
    uint32_t max_lines;
    uint32_t component_id;
    uint32_t version_id;
    uint32_t imr_reset;
    uint32_t axiawlen_reset;
    uint32_t axiarlen_reset;
    uint32_t spi_ctrlr0_reset;
    bool has_enhanced_spi;
    bool has_idma;
    bool has_xip;
    uint64_t xip_window_size;
} DwSsiConfig;
```

字段名和具体类型在最终计划中应按当前 QEMU 代码风格复核，不把上述草案直接当成已定代码。

### 2.3 capability 范围

用户已确认三项 capability 全部纳入 Step 4：

- `has-enhanced-spi`
- `has-idma`
- `has-xip`

采用“小步实现、随时调整”的方式，不在一个改动中同时接入全部门控。每一项 capability 都必须明确：

1. 控制哪些寄存器、状态、IRQ 和数据路径；
2. capability 关闭时哪些访问为 RAZ/WI；
3. capability 关闭时哪些 IRQ 必须保持低电平；
4. 是否影响 `realize()` 阶段的资源创建；
5. 正路径与负路径如何分别测试。

## 3. 下一会话的最小读取顺序

1. `AGENTS.md`
2. `exper-note/workspace-state.md`
3. 本任务单
4. `k230-spi-qspi-review-v2-decision-notes.md`
5. `k230-spi-qspi-v2-step3-hi-sys-decoupling.md`
6. `qemu-camp-2026-k230/include/hw/ssi/dw_ssi.h`
7. `qemu-camp-2026-k230/hw/ssi/dw_ssi.c`
8. `qemu-camp-2026-k230/hw/riscv/k230.c`
9. `qemu-camp-2026-k230/tests/qtest/k230-dw-ssi-test.c`

源码仓库存在 `.codegraph/`。理解符号、调用关系和资源生命周期时先使用：

```bash
codegraph explore "DwSsiState dw_ssi_realize dw_ssi_reset k230_soc_realize"
```

TRM 和 SDK 属于大文件或大目录，只按最终计划所需的寄存器名、综合参数名和 DTS 节点定向搜索。

## 4. 最终计划必须核实的证据

### 4.1 当前代码

- `num-cs`、`max-lines` 已有 property 的当前实现和校验位置。
- FIFO 深度在哪些数组、level 计数、阈值和循环中硬编码为 256。
- `IDR`、`VERSION`、`IMR`、`AXIAWLEN`、`AXIARLEN`、`SPI_CTRLR0` 的当前 reset 来源。
- enhanced SPI、IDMA、XIP 分别涉及的寄存器读写分支、状态字段、IRQ 和数据路径。
- XIP `MemoryRegion` 当前在何时创建、为何只有 capability 确定后才能决定第二个 SysBus MMIO region。
- VMState 是否包含将被 capability 关闭的状态，以及属性变化对迁移兼容性的影响。

### 4.2 K230 TRM

最终计划需要形成 QSPI0/1 与 SPI-OPI/FMC 的实例配置表，至少核对：

- FIFO 深度；
- CS 数和最大线宽；
- component/version ID；
- `IMR` reset；
- `AXIAWLEN` / `AXIARLEN` reset；
- `SPI_CTRLR0` reset；
- enhanced SPI、IDMA、XIP 的综合能力；
- XIP 寄存器组和 XIP window 是否仅属于 FMC。

当前已发现的候选差异为：QSPI 的 `IMR=0x1f`、AXI burst length reset 为 `0x700`、`SPI_CTRLR0=0x04000200`；FMC 的候选值分别为 `0x3f`、`0`、`0x28000200`。最终文档必须重新定位原文后再落表。

### 4.3 K230 SDK

- 核对 U-Boot 实际启动路径使用的三个节点及 `num-cs`。
- 记录 Linux DTS 与 U-Boot DTS 对 `num-cs` 的冲突。
- 若仍采用物理 QSPI0/QSPI1/SPI-OPI 为 `5/5/1`，必须在计划中写明证据优先级：当前 firmware 路径、U-Boot DTS、现有模型和 qtest 共同支持该选择。
- 核对 SDK 对 enhanced SPI、内部 DMA、XIP 寄存器的真实访问路径，区分“硬件 capability”与“当前软件恰好未使用”。

### 4.4 QEMU 主线先例

重点参考 `hw/i3c/dw-i3c.c`：

- 内部 `cfg` 结构与 QOM property 的组织；
- 依赖 property 的 FIFO/MMIO 资源在 `realize()` 创建；
- reset 从实例配置填充寄存器；
- 非法 property 组合通过 `error_setg()` 阻止 realize。

最终计划只借鉴组织方式，不机械复制 I3C 模型。

## 5. Step 4 最终计划的建议分解

所有小目标写在同一个最终文档中。每个小目标遵循“先增加失败断言，再实现最小改动，再运行局部与完整回归”。

### Step 4.1：建立配置骨架，保持当前行为

- 增加集中配置结构和完整 property 集合。
- 为 property 设置与当前模型一致的默认值，保证未修改 machine 时行为不变。
- 在 `realize()` 校验取值和组合，但暂不改变寄存器可见性。
- 将固定大小 FIFO 改为由 `fifo-depth` 决定的资源，或在最终计划中明确采用 QEMU 合适的数据结构。
- 本小目标结束时，现有 10 项 K230 SSI qtest 应保持不变。

### Step 4.2：应用 K230 三实例 profile

- 在 `hw/riscv/k230.c` 为 QSPI0、QSPI1、SPI-OPI/FMC 显式设置所有配置。
- reset 从配置读取，不再由通用模型硬编码 K230 值。
- 增加三实例 reset profile、SER 位宽、FIFO 深度和最大线宽测试。
- 这一阶段先证明“同一通用模型可以表达不同实例”，再进入 capability 负路径。

### Step 4.3：逐项接入 capability 门控

按以下顺序逐项完成；每一项形成独立、可验证的小改动：

1. `has-enhanced-spi`：门控 `SPI_CTRLR0`、增强传输格式和相关校验。
2. `has-idma`：门控内部 AXI DMA 寄存器、状态、传输入口和 IDMA IRQ。
3. `has-xip`：门控 XIP 寄存器组，并只在 capability 开启且 window size 合法时创建第二个 MMIO region。

XIP 放在最后，因为它同时影响寄存器、运行时 GPIO、MemoryRegion 生命周期和 machine 映射，风险最高。

### Step 4.4：收敛验证与 patch 边界检查

- 完整构建和 qtest 回归。
- 检查公共头文件可独立包含。
- 检查通用 SSI 目录无 K230 依赖。
- 检查每个小改动的测试归属和未来 patch 归属，避免后续 patch 修复前一 patch 的行为。
- 只记录建议的 commit 边界，不在本任务中执行 commit。

## 6. 最终计划必须解决的关键决策

### 6.1 capability 负路径测试载体

K230 的三个实例可能无法同时提供三项 capability 的 `false` 配置。最终计划必须二选一并说明理由：

- 增加一个面向通用 `TYPE_DW_SSI` 的独立 qtest 测试载体，用不同 property 组合验证 RAZ/WI 和 IRQ 低电平；
- 若 QEMU 当前测试基础设施无法低成本映射独立 SysBus 设备，则在 K230 测试机中增加仅测试使用的配置入口。

推荐优先调查独立通用 qtest，避免为了测试污染 K230 machine 的产品配置。

### 6.2 capability 与资源生命周期

- `fifo-depth` 和 `xip-window-size` 必须在 realize 前确定。
- `has-xip=false` 时不应要求 machine 映射不存在的第二个 MMIO region。
- machine 侧 `sysbus_mmio_map()` 必须与实例实际暴露的 region 数量一致。
- 不允许设备 realize 后动态改变 capability。

### 6.3 property 默认值

默认值服务于通用模型的兼容和可用性，不应伪装成“所有 DW SSI 的硬件默认值”。最终计划需明确：

- 哪些默认值只是保持当前 V2 中间态行为；
- K230 machine 必须显式设置哪些值；
- 是否在最终上游版本中继续保留这些默认值。

### 6.4一点小提示
落地时的几个具体建议
DwSsiConfig 里建议放这些内容，对应 properties 用 kebab-case 命名：

fifo_depth（K230 是 256，其他 SoC 的 DW SSI 常见 16/32/64/128，必须可配）
max_lines（1/2/4/8，控制 Dual/Quad/Octal 能力的接受范围）
has_xip、has_idma（布尔能力开关，即使你暂时不实现 IDMA 分离，开关先留好）
关键寄存器的 reset 值——不用每个都做成 property，通用模型用 DW IP 标准默认值（参照 Linux spi-dw.h 的复位语义），K230 TRM 里与默认值不同的少数几个才暴露成 property

realize() 里做的事：按 cfg.fifo_depth 动态分配 TX/RX FIFO，校验 max_lines 合法性（非法值直接 error_setg 拒绝 realize），has_xip 为真才创建 XIP 子 region。K230 machine 侧就是三次 sysbus_create_simple 风格实例化加 object_property_set_* 配置，然后 sysbus_mmio_map、接 PLIC——这段代码本身就会成为 patch 里证明"通用模型够用"的最有力证据。
一个提醒：properties 只负责表达配置差异，不要顺手把运行时开关（比如 SSIENR 使能态）也做成 property，那是 guest 通过寄存器写的状态，属于 VMState 管辖，两边别串。

XIP window 尽量剥离出通用模型。​ XIP 的本质是把 SPI flash 内容映射进 CPU 地址空间，这个窗口的地址由 SoC 地址映射决定（K230 是 FMC/HI_SYS 控制的），它不属于 DW SSI IP 核本身。QEMU 里两种先例可对比：stm32f7xx_qspi 把 memory-mapped 模式做在控制器内部，zynq 的 LQSPI 干脆是独立设备。对 K230 更干净的做法是：通用 dw_ssi 只管控制器寄存器和传输，XIP 窗口由 K230 machine（或未来的 wrapper）创建 alias region 指向 flash 后端。这样 cfg 里连 has_xip 都可以省掉，通用模型一行 XIP 代码都没有。
IDMA 用配置开关条件装配，别用条件分支。​ 如果 K230 的 IDMA 寄存器就在 SSI 地址空间内，没法物理剥离，那就让 realize() 根据 cfg.has_idma 决定是否注册那段寄存器、是否初始化 DMA 状态机，运行时代码里不出现散落的 if (s->cfg.has_idma)。判断集中在装配点，逻辑集中在 IDMA 自己的函数里——这是"参数化装配"和"参数化行为"的区别，前者 reviewer 能接受，后者会被挑。

## 7. 最终计划的测试矩阵下限

- 三实例分别验证 `IMR`、`AXIAWLEN`、`AXIARLEN`、`SPI_CTRLR0`、`IDR` 和 `VERSION` reset。
- QSPI 验证 5 个 CS 位可用，FMC 验证只有 1 个 CS 位可用。
- 实际向 FIFO 填充 256 帧，验证第 257 帧不再增加 level。
- `has-enhanced-spi=false`：相关寄存器 RAZ/WI，增强格式不能进入有效数据路径。
- `has-idma=false`：IDMA 寄存器 RAZ/WI，IDMA 状态和 IRQ 保持无效。
- `has-xip=false`：XIP-only 寄存器 RAZ/WI，不暴露或不映射第二个 MMIO region。
- `has-xip=true`：FMC XIP 寄存器和 128 MiB XIP window 继续工作。
- system reset 后重新检查三个实例 profile 和 capability 行为。
- 完整运行 K230 SSI 10 项 qtest 和 K230 WDT qtest。
- 构建完成后运行 `git diff --check`、相关文件 checkpatch 和公共头文件独立包含检查。

## 8. 最终计划的文档结构

最终文档沿用 Step 3 的表达方式，至少包含：

1. 摘要；
2. 当前状态分析；
3. 证据与硬编码点清单；
4. K230 三实例配置矩阵；
5. 精确改动方案；
6. Step 4.1 至 Step 4.4 的小步实施任务；
7. 关键代码片段；
8. capability 关闭语义；
9. 假设与决策；
10. TDD 测试矩阵和精确命令；
11. 收敛后的实施清单；
12. 与 V2 总决策文档和总实施路线的关系。

最终文档不得保留占位标记、延期测试表述或没有验证方式的模糊任务。

## 9. 下一会话的完成标准

- 已创建 Plan C，并在后续复核中由 `k230-spi-qspi-v2-step4-plan-final-instance-configuration.md` 取代。
- 所有配置值和 capability 归属均能回指当前代码、TRM、SDK 或 QEMU 主线先例。
- 明确 capability 关闭时的寄存器、IRQ、数据路径和 MMIO 行为。
- 每个小目标都能独立编译、运行定向测试并执行完整回归。
- 同步更新 `k230-spi-qspi-v2-implementation-plan.md` 和 `workspace-state.md`。
- 仅修改文档，不执行 Git commit 或 push。
