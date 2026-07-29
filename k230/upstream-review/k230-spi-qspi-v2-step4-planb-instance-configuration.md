# K230 SPI/QSPI V2 Plan B：面向 QEMU 主线的完整 Step 4 方案

首次记录：2026-07-29

> **历史方案：禁止作为实施依据。** 本文已由 [Step 4 Plan Final](k230-spi-qspi-v2-step4-plan-final-instance-configuration.md) 取代。Plan B 关于“三实例均 `has-xip=true`”“`XIP_MODE_BITS` 被 enhanced/IDMA 共享”的结论不成立；QEMU profile 最终为 QSPI0/QSPI1 `has-xip=false`、FMC `has-xip=true`。本文的全量迁移 equality 和流程检查装置也不再采用。

适用开发分支：`qemu-camp-2026-k230/k230-V2-patch-spi`

代码基线：`b7702892e2`（在 `1342f13a6e` 的 v3.4 功能基线上完成通用命名和 HI_SYS GPIO 解耦）

文档定位：独立于 [Step 4 Plan A](k230-spi-qspi-v2-step4-plana-instance-configuration.md) 的主线导向替代方案。本文的目标是完整完成 Step 4：实例参数化、capability 定义、字段/行为/IRQ 门控、reset/migration 一致性和正反路径验证。工作可以拆成多个小步骤和小章节，但不能把 capability 推迟到 Step 5。

---

## 1. 执行结论

Plan B 的核心判断是：

> Bin Meng 要求的是清晰的通用 IP/SoC 集成边界，不是要求 v2 一次性实现完整的 DWC SSI CoreConsultant 参数框架。

但“无需覆盖全部 CoreConsultant 组合”不等于“只做非 capability 参数化”。因此，Step 4 应当：

1. 保留单一的通用 `TYPE_DW_SSI`，不增加 K230 SSI wrapper；
2. 把 K230 地址、PLIC、HI_SYS、Flash 和 XIP 映射留在 K230 machine；
3. 参数化当前模型已经依赖的固定配置和动态资源大小；
4. 为当前已经实现的 optional enhanced SPI、internal DMA 和 XIP engine 建立 capability；
5. capability 必须同时控制寄存器字段、数据路径、IRQ、reset 和 migration；
6. 先完成逐字段 capability matrix，再实现门控，不能按寄存器名字粗暴分类；
7. 提供可执行 false-profile，不能因为 K230 三实例都启用某能力就只测试 true-path；
8. external DMA、concurrent XIP、dynamic wait、DDR/RXDS 等尚未实现能力继续明确留在范围外。

Plan B 建议本轮新增或整理的配置只有：

| 配置 | 当前状态 | Plan B |
|---|---|---|
| `num-cs` | 已有 property | 保留，记录 DTS 冲突 |
| `max-lines` | 已有 property | 保留，作为事务线宽上限 |
| `fifo-depth` | 固定 256 | 新增 property |
| `component-id` | 固定 `0xa1b2c3d5` | 新增 property |
| `version-id` | 固定 `0x3130332a` | 新增 property |
| `spi-ctrlr0-reset` | 由 `max-lines == 8` 推导 | 新增 property，解除隐式耦合 |
| `xip-window-size` | 固定 128 MiB，所有实例创建 region | 新增 property；0 表示无 aperture |
| `has-enhanced-spi` | 隐式始终启用 | 新增 capability，并实现 false-path |
| `has-idma` | 隐式始终启用 | 新增 internal DMA capability，并实现 false-path |
| `has-xip` | engine、aperture、运行时 enable 混合 | 新增 XIP engine capability；与 window/GPIO 分离 |

完整 capability 实施细则见：
[第 16 章 Capability 门控实施细则](#step4-capability-gating)。

Step 5 才负责最终 commit 拆分、重排、逐提交构建和 11-patch series 收口；这些工作不再作为 Step 4 的设计内容。

---

## 2. 当前基线与开工前置修复

### 2.1 当前开发基线

```text
b7702892e2  使用 GPIO 解除 HI_SYS -> DWC SSI 的反向依赖
5d1e7c9241  将 k230_dw_ssi 重命名为通用 dw_ssi
1342f13a6e  v3.4 完整功能基线
```

当前核心结构已经满足正确方向：

```text
K230 HI_SYS --xip-enable GPIO--> TYPE_DW_SSI
K230 HI_SYS --generic getters--> DwSsiState mode/sleep
```

### 2.2 Step 4 前必须先修复的 Step 3 残留

开始属性化前先做一个小型清理提交或并入当前开发提交：

- 删除 `include/hw/ssi/dw_ssi.h` 中残留的 `K230HiSysState` 前置声明；
- 删除 `include/hw/misc/k230_hi_sys.h` 中已无实现的 `k230_hi_sys_xip_enabled()` 声明；
- 修复 `qdev_init_gpio_out_named()` 缺少空格和多余空行；
- 整理 `k230.c` GPIO 连接缩进；
- 最终上游提交使用英文标题和 `Signed-off-by`。

前置残留检查：

```bash
rg -n 'K230|k230_' hw/ssi/dw_ssi.c include/hw/ssi/dw_ssi.h
```

预期无结果。

---

## 3. QEMU 主线视角下的设计原则

### 3.1 通用模型不等于完整 IP 生成器

QEMU 现有 `hw/i2c/designware_i2c.c` 是通用 DesignWare I2C 模型，但它仍然模拟一个具体、有限的 DesignWare profile：FIFO 深度、component version/type 和未实现能力都由模型固定。

这说明主线可接受的“通用”含义是：

- 文件、类型和行为不依赖某个 SoC；
- SoC 集成在 machine 中完成；
- 已实现行为可以被其他同 profile 的 SoC 复用；
- 不要求第一次提交就覆盖所有综合变体。

因此 `TYPE_DW_SSI` 可以在 v2 中明确限定为：

> 支持 K230 所需且有多源证据的 DWC SSI 1.03* register subset，包括标准 PIO、enhanced SDR、内部 AXI DMA 和可选 XIP aperture。

### 3.2 综合参数与运行状态分离

建议参考 `hw/i3c/dw-i3c.c`，把不可变配置集中在 `cfg` 子结构中：

```c
enum {
    DW_SSI_CAP_ENHANCED_SPI,
    DW_SSI_CAP_IDMA,
    DW_SSI_CAP_XIP,
};
```

```c
struct DwSsiState {
    SysBusDevice parent_obj;

    MemoryRegion mmio;
    MemoryRegion xip;
    SSIBus *spi;
    qemu_irq *cs_lines;
    qemu_irq irqs[DW_SSI_IRQ_COUNT];

    Fifo32 tx_fifo;
    Fifo32 rx_fifo;

    struct {
        uint32_t num_cs;
        uint32_t max_lines;
        uint32_t fifo_depth;
        uint32_t component_id;
        uint32_t version_id;
        uint32_t spi_ctrlr0_reset;
        uint64_t xip_window_size;
        uint32_t capabilities;
    } cfg;

    uint32_t regs[DW_SSI_NUM_REGS];
    /* 其余运行状态 */
};
```

这样可以明确区分：

- `cfg.*`：综合/实例配置，realize 后不变化；
- `regs[]`、FIFO、phase、IRQ latch：guest 可见运行状态；
- `xip_enabled`：外部 GPIO 输入的当前电平。

### 3.3 主线抽象必须有有限边界，也必须闭合

一个普通配置 property 至少应满足下列条件之一：

- K230 三个实例之间确实不同；
- DWC 公开驱动明确证明这是互斥寄存器布局；
- 参数决定 QEMU 对象资源大小；
- 参数是 guest 可观察的固定寄存器值；
- 参数已经有可执行测试覆盖。

仅仅在 TRM 中看到 `SSIC_*` 综合参数名，不足以要求 v2 为每个参数增加 bool property。

但是，一旦把当前模型已经实现的 optional block 声明为 capability，Step 4 就必须把
它闭合：

- property 实际控制寄存器 readable/writable mask；
- capability=false 不进入对应数据路径；
- capability=false 不产生对应状态和 IRQ；
- reset 和 migration 不恢复矛盾状态；
- true/false 两条路径都有可执行验证。

Plan B 的有限边界是只参数化当前已实现的 enhanced SPI、internal DMA 和 XIP
engine，而不是推迟这三项 capability。

---

## 4. K230 实例证据矩阵

### 4.1 已确认的共同能力

| 项目 | K230 结论 | 主要证据 |
|---|---|---|
| IP 家族 | `snps,dwc-ssi-1.01a`，component version `1.03*` | Linux DTS、TRM VERSION_ID |
| FIFO 深度 | TX/RX 各 256 entries | RT `SSIC_TX_ABW/SSIC_RX_ABW=256`、TRM |
| DMA profile | 内部 AXI DMA，`SSIC_HAS_DMA=2` | RT/U-Boot 驱动 |
| enhanced SPI | 三实例均需 Dual/Quad，OPI 实例还暴露 8-line profile | RT `max_line` 和传输路径 |
| IRQ | 每实例 9 路 | Linux DTS、RT IRQ 枚举 |
| component ID | `0xa1b2c3d5` | TRM IDCODE |
| version ID | `0x3130332a` | TRM SSIC_COMP_VERSION |
| concurrent XIP | 当前 profile 未启用 | RT 条件布局、寄存器审阅表 |
| dynamic wait/SPITE | 当前 profile 未启用 | RT 明确保留、寄存器审阅表 |

这意味着 K230 machine 不需要伪造 `has-idma=false` 或
`has-enhanced-spi=false` 的物理实例，但通用模型仍需要可配置 false-profile 来证明
capability 门控完整。

### 4.2 三实例差异

按 QEMU 物理数组顺序：

| QEMU 实例 | 物理模块 | SDK 逻辑编号 | 基址 | 最大线宽 | SPI_CTRLR0 reset | XIP aperture |
|---|---|---:|---:|---:|---:|---|
| `dw_ssi[0]` | QSPI0 | spi1 | `0x91582000` | 4 | `0x04000200` | 无 |
| `dw_ssi[1]` | QSPI1 | spi2 | `0x91583000` | 4 | `0x04000200` | 无 |
| `dw_ssi[2]` | SPI-OPI/FMC | spi0 | `0x91584000` | 8 | `0x28000200` | `0xC0000000–0xC7FFFFFF` |

### 4.3 `num-cs` 证据冲突

当前资料不是完全一致：

- Linux DTS：三个实例均为 `num-cs = <1>`；
- U-Boot DTS：SPI-OPI 为 1，QSPI0/QSPI1 为 5；
- 当前 QEMU 基线和 qtest：物理顺序为 `5/5/1`；
- 当前启动验证只实际使用 CS0。

Plan B 的 v2 决策是：

> 保留当前 `5/5/1`，避免在架构重构中夹带 guest-visible 行为变化；在 commit message 中如实记录 Linux/U-Boot DTS 冲突，不把 5 宣称为已由硅上证据确认的固定宽度。

如果后续获得实机 `SER` implemented bits 或 CoreConsultant 报告，再单独决定是否收敛为 `1/1/1`。这不应阻塞当前通用层拆分。

---

## 5. Plan B 的明确边界

### 5.1 Step 4 必须完成

- 通用文件和 QOM 类型；
- 标准寄存器、FIFO/PIO、TMOD；
- 现有 enhanced SDR 行为；
- 现有内部 AXI DMA profile；
- 现有 IRQ 集合；
- K230 HI_SYS GPIO 解耦；
- 实例固定值 property；
- FIFO 深度 property；
- 可选 XIP aperture 大小 property；
- `has-enhanced-spi`、`has-idma`、`has-xip` capability；
- capability 字段级 readable/writable/reset mask；
- capability 对传输分发、guest memory、IRQ 和 GPIO 副作用的门控；
- capability 配置与运行状态的 migration 一致性；
- 可执行的 standard-only、enhanced-pio、enhanced-idma、full-xip profile；
- K230 machine 对三个实例的显式配置。

### 5.2 Step 4 明确不实现

- external DMA 请求和 `DMATDLR/DMARDLR` 数据通路；
- `dma-mode=external`；
- concurrent XIP；
- dynamic wait/SPITE；
- DDR/RXDS/Octal 实际传输；
- 按 DWC 版本自动切换整套寄存器布局；
- 完整覆盖全部 DWC SSI CoreConsultant 综合参数。

对于范围外能力，继续使用当前已有的 RAZ/WI 或显式拒绝行为，并在文档和最终 commit message 中声明限制。对于范围内三个 capability，false-path 是 Step 4 的硬性完成条件。

---

## 6. Property 设计

### 6.1 属性表

```c
static const Property dw_ssi_properties[] = {
    DEFINE_PROP_UINT32("num-cs", DwSsiState, cfg.num_cs, 1),
    DEFINE_PROP_UINT32("max-lines", DwSsiState, cfg.max_lines, 1),
    DEFINE_PROP_UINT32("fifo-depth", DwSsiState, cfg.fifo_depth, 256),
    DEFINE_PROP_UINT32("component-id", DwSsiState, cfg.component_id,
                       0xa1b2c3d5),
    DEFINE_PROP_UINT32("version-id", DwSsiState, cfg.version_id,
                       0x3130332a),
    DEFINE_PROP_UINT32("spi-ctrlr0-reset", DwSsiState,
                       cfg.spi_ctrlr0_reset, 0),
    DEFINE_PROP_UINT64("xip-window-size", DwSsiState,
                       cfg.xip_window_size, 0),
    DEFINE_PROP_BIT("has-enhanced-spi", DwSsiState, cfg.capabilities,
                    DW_SSI_CAP_ENHANCED_SPI, false),
    DEFINE_PROP_BIT("has-idma", DwSsiState, cfg.capabilities,
                    DW_SSI_CAP_IDMA, false),
    DEFINE_PROP_BIT("has-xip", DwSsiState, cfg.capabilities,
                    DW_SSI_CAP_XIP, false),
};
```

默认 capability=false、`max-lines=1`、`spi-ctrlr0-reset=0` 共同组成自洽的
standard-only profile。K230 machine 仍应显式设置全部 guest 可观察值和 capability，
使 SoC 集成可以在代码 review 中直接看到。

### 6.2 `has-xip`、window 和 GPIO 的三层语义

`has-xip` 容易把三种不同概念重新混在一起：

- DWC SSI 寄存器是否存在；
- SoC 是否把 XIP 接口映射成地址窗口；
- HI_SYS 是否在运行时允许窗口响应。

Plan B 使用：

```text
has-xip               -> XIP command engine 和专属字段是否存在
xip-window-size == 0  -> SoC 不暴露 XIP MMIO aperture
xip-window-size != 0  -> 创建第二个 sysbus MMIO region
xip-enable GPIO       -> 运行时控制已经存在的窗口是否响应
```

该段结论已失效。最终证据确认 `XIP_MODE_BITS/XIP_INCR_INST/XIP_WRAP_INST` 只在 FMC 5.3 XIP 寄存器表中定义，普通 enhanced/IDMA 不读取它们。K230 最终 profile 为 QSPI0/QSPI1 `has-xip=false`、SPI-OPI/FMC `has-xip=true`。

### 6.3 `has-idma` 的有限语义

K230 三实例均使用 `SSIC_HAS_DMA=2`。`has-idma` 只表达当前模型的 internal AXI DMA profile。实现 false-path 必须处理：

- `DMACR` 混合字段的动态 mask；
- `IMR/ISR/RISR` 中 DONE/AXIE 字段 mask；
- `AXIECR/DONECR` 的 RC 语义；
- `SR.CMPLTD_DF`；
- `0x50/0x54` 与 external DMA 布局的互斥解释；
- false-path qtest 基础设施。

这些工作全部属于 Step 4，不能只添加 bool 后留空。

本轮不把 `has-idma=false` 自动解释为 external DMA。未来需要支持 external DMA
时，再升级为：

```text
dma-mode = none / external / internal
```

而不是只能表达“内部 DMA 有/无”的 bool。

### 6.4 `has-enhanced-spi` 的闭合范围

三实例均使用 enhanced SPI，且现有 `max-lines` 承担事务线宽约束。加入 `has-enhanced-spi=false` 必须完成：

- `CTRLR0.SPI_FRF` RAZ/WI；
- `SPI_CTRLR0` 混合字段 mask；
- enhanced transfer dispatch；
- 相关寄存器复位值；
- 可执行 false-path 测试。

上述内容在 Step 4 内分小点完成。`has-enhanced-spi=false` 时要求
`max-lines=1`，且必须有 standard-only profile 验证。

---

## 7. 实施阶段

### B0：清理 Step 3 残留

只做依赖和风格清理，不改变行为。完成后：

```bash
rg -n 'K230|k230_' hw/ssi/dw_ssi.c include/hw/ssi/dw_ssi.h
git diff --check
```

均无异常。

### B1：建立 `cfg` 并属性化固定寄存器值

#### 状态结构

将现有 `num_cs`、`max_lines` 与新增配置集中到 `s->cfg`。

#### reset 逻辑

```c
s->regs[R_IDR] = s->cfg.component_id;
s->regs[R_SSIC_VERSION_ID] = s->cfg.version_id;
s->regs[R_SPI_CTRLR0] = s->cfg.spi_ctrlr0_reset;
```

删除：

```c
s->max_lines == 8 ? FMC_RESET : SPI_RESET
```

K230 machine 显式设置：

```text
component-id       = 0xa1b2c3d5
version-id         = 0x3130332a
spi-ctrlr0-reset   = 0x04000200 / 0x28000200
```

#### realize 校验

保留当前：

- `num-cs` 当前模型支持范围 `1..8`；
- `max-lines` 当前支持 `1/4/8`。

增加：

- `spi-ctrlr0-reset` 不得超出当前模型声明支持的
  `DW_SSI_SPI_CTRLR0_WRITABLE_MASK`；这只是配置合法性检查，不表示
  DDR/RXDS/HyperBus 等尚未执行的数据路径因此变成已实现；
- `has-enhanced-spi=false` 时 `max-lines` 必须为 1；
- `has-xip=true` 时必须同时启用 enhanced SPI；
- `xip-window-size != 0` 时必须 `has-xip=true`；
- reset 值不得包含与 capability=false 冲突的专属字段；
- 所有配置校验必须先于可变 FIFO、条件 XIP region 和动态 GPIO 数组等
  realize 阶段资源的创建。

本轮不顺带扩展 `max-lines=2`。Dual 事务已经可以由 `max-lines=4` 的实例使用；没有独立 dual-only 消费者时无需扩大配置面。

### B2：属性化 FIFO 深度

#### 合法范围

依据 K230 TRM 和当前寄存器宽度，Plan B 支持：

```text
fifo-depth = 8 / 16 / 32 / 64 / 128 / 256
```

realize 中验证：

```c
if (s->cfg.fifo_depth < 8 || s->cfg.fifo_depth > 256 ||
    !is_power_of_2(s->cfg.fifo_depth)) {
    error_setg(errp, ...);
    return;
}
```

#### FIFO 创建

参考 DWC I3C，将 FIFO 创建移到 realize，且在全部配置校验通过之后：

```c
fifo32_create(&s->tx_fifo, s->cfg.fifo_depth);
fifo32_create(&s->rx_fifo, s->cfg.fifo_depth);
```

状态计算改为使用 `s->cfg.fifo_depth`：

- `SR.TFNF`；
- `SR.RFF`。

#### 阈值寄存器

引入可变 FIFO 深度后，不能只替换容量常量。必须同步实现：

- `TXFTLR.TFT < fifo-depth`；
- `RXFTLR.RFT < fifo-depth`；
- 非法值写入保持旧字段值；
- 合法字段更新后重新计算 IRQ。

`TXFTLR.TXFTHR` 继续按当前 11 位字段保存；在没有更明确硬件限制证据前，不额外猜测它必须小于 FIFO 深度。

### B3：集中 K230 三实例配置

在 `hw/riscv/k230.c` 中建立物理实例配置表，避免分散的多组 `qdev_prop_set_*()`：

```c
typedef struct K230DwSsiConfig {
    uint32_t num_cs;
    uint32_t max_lines;
    uint32_t fifo_depth;
    uint32_t spi_ctrlr0_reset;
    uint64_t xip_window_size;
    bool has_enhanced_spi;
    bool has_idma;
    bool has_xip;
} K230DwSsiConfig;

static const K230DwSsiConfig k230_dw_ssi_cfg[] = {
    /* QSPI0 */ { 5, 4, 256, 0x04000200, 0, true, true, true },
    /* QSPI1 */ { 5, 4, 256, 0x04000200, 0, true, true, true },
    /* SPI-OPI */ { 1, 8, 256, 0x28000200, 128 * MiB,
                    true, true, true },
};
```

推荐使用指定初始化器而不是位置初始化器，最终代码例如：

```c
{
    .num_cs = 5,
    .max_lines = 4,
    .fifo_depth = 256,
    .spi_ctrlr0_reset = 0x04000200,
    .xip_window_size = 0,
    .has_enhanced_spi = true,
    .has_idma = true,
    .has_xip = true,
},
```

共同的 `component-id` 和 `version-id` 也应由 K230 machine 显式设置，不能依赖通用默认值表达 K230 集成。

配置表使用物理数组顺序；`k230_ssi_routes[]` 继续只负责 SDK 逻辑编号和 PLIC 路由，两个表职责不能混合。

### B4：属性化 XIP aperture

#### 对象生命周期

主寄存器 region 继续在 instance_init 注册为 sysbus MMIO index 0。

XIP aperture 根据配置在 realize 中注册为 index 1：

```c
if (s->cfg.xip_window_size != 0) {
    memory_region_init_io(&s->xip, OBJECT(dev), &dw_ssi_xip_ops, s,
                          TYPE_DW_SSI ".xip",
                          s->cfg.xip_window_size);
    sysbus_init_mmio(SYS_BUS_DEVICE(dev), &s->xip);
}
```

约束：

- `xip-window-size == 0` 合法，表示设备没有向 SoC 暴露 XIP aperture；
- 当前 XIP 地址生成只支持最多 32-bit address，因此非零大小不得超过 4 GiB；
- K230 SPI-OPI 设置 128 MiB；
- QSPI0/QSPI1 设置 0；
- K230 machine 只对 SPI-OPI 调用 `sysbus_mmio_map(..., 1, 0xC0000000)`；
- `xip-enable` GPIO 仍只连接 SPI-OPI。

这里的收益是准确表达设备拓扑，不是节省 128 MiB RAM。I/O MemoryRegion 不分配等大后备内存。

#### 寄存器策略

Plan Final 的寄存器策略为：

- QSPI profile 的 `XIP_MODE_BITS`、`XIP_INCR_INST`、`XIP_WRAP_INST` RAZ/WI；FMC/XIP profile 保持读写语义；
- concurrent XIP 相关寄存器继续按当前证据 RAZ/WI；
- `SPI_CTRLR0` 的 XIP/enhanced/IDMA 共享字段保持当前 mask；
- 不增加按名字分类的 `dw_ssi_is_xip_reg()`。

### B5：迁移配置一致性

properties 属于 immutable configuration：

- 源端和目标端 machine 都在加载 VMState 前完成属性设置和 realize；
- properties 不作为普通运行状态迁移；
- 迁移要求两端使用一致的 machine/device 配置。

所有会影响 guest 可见行为、VMState 形状或对象拓扑的配置都应增加一致性
检查，并放在对应动态状态字段之前：

```c
VMSTATE_UINT32_EQUAL(cfg.num_cs, DwSsiState),
VMSTATE_UINT32_EQUAL(cfg.max_lines, DwSsiState),
VMSTATE_UINT32_EQUAL(cfg.fifo_depth, DwSsiState),
VMSTATE_UINT32_EQUAL(cfg.component_id, DwSsiState),
VMSTATE_UINT32_EQUAL(cfg.version_id, DwSsiState),
VMSTATE_UINT32_EQUAL(cfg.spi_ctrlr0_reset, DwSsiState),
VMSTATE_UINT64_EQUAL(cfg.xip_window_size, DwSsiState),
VMSTATE_UINT32_EQUAL(cfg.capabilities, DwSsiState),
```

嵌套字段写法可直接用于 `VMSTATE_*_EQUAL`；QEMU 现有
`virtio-gpu` 迁移描述已经使用同类写法。`VMSTATE_FIFO32` 最终按目标对象
已创建 FIFO 的 capacity 装载动态缓冲区，因此 `fifo-depth` equality 必须先于
两个 FIFO 字段，不能只依赖装载过程偶然失败。

固定只读寄存器值可由 reset/profile 重建，不需要作为普通可变状态保存；
但即使现有 `regs[]` 已包含其迁移时快照，也必须检查
`component-id`/`version-id`/`spi-ctrlr0-reset` 一致，否则迁移后的下一次
system reset 会使用目标端配置，造成 guest-visible 行为漂移。

迁移策略需要在最终 series 重组时统一决定：

- 如果保留整个 `regs[]` 迁移，配置 equality 负责拒绝不一致目标；
- 为这组最终一次性引入的 VMState 字段明确设置 `.version_id` 和
  `.minimum_version_id`；由于设备尚未进入上游，不为当前开发分支中的临时
  格式保留无意义的兼容层；
- 不要在 post-load 中重新执行完整 reset；
- `xip_enabled` 继续作为运行状态迁移，HI_SYS post-load 再驱动 GPIO校准。

### B6：冻结 capability 字段矩阵

将 [第 16 章 Capability 门控实施细则](#step4-capability-gating)
中的字段矩阵落实到代码前的审阅表，至少覆盖：

- `CTRLR0.SPI_FRF`；
- `SPI_CTRLR0` 的 shared/XIP/enhanced 字段；
- `XIP_MODE_BITS`、`XIP_INCR_INST`、`XIP_WRAP_INST`；
- `DMACR`、`AXIAWLEN`、`AXIARLEN`、`SPIDR`、`SPIAR`、`AXIAR0/1`；
- `IMR/ISR/RISR` 的 DONE/AXIE；
- `SR.CMPLTD_DF`、`AXIECR`、`DONECR`；
- reset 值、post-load 状态和外部 IRQ 输出。

这一阶段的产物是“字段—能力—读写—行为—IRQ—证据”矩阵，而不是只列寄存器地址。

### B7：实现 capability 寄存器和行为门控

按小提交依次完成：

1. `has-enhanced-spi`：动态 `CTRLR0/SPI_CTRLR0` mask、标准路径回退、非法增强配置拒绝；
2. `has-idma`：internal DMA 专属寄存器、guest-memory 访问、完成状态和 DONE/AXIE IRQ；
3. `has-xip`：XIP 专属字段、XIP command builder、XIP GPIO 和 aperture 三层语义。

所有 capability 都必须满足“寄存器层、行为层、信号层”三层一致性。混合寄存器使用
动态字段 mask，不能通过地址范围把整个寄存器 RAZ/WI。

### B8：建立可执行 false-profile

K230 三个物理实例均为 enhanced+IDMA profile，不能作为 false-path 测试。必须提供
独立可配置的通用 DWC SSI 测试实例，至少验证：

```text
standard-only   enhanced=false idma=false xip=false window=0
enhanced-pio    enhanced=true  idma=false xip=false window=0
enhanced-idma   enhanced=true  idma=true  xip=false window=0
full-xip        enhanced=true  idma=true  xip=true  window>0
```

优先复用现有 qtest 容器；若无法映射独立 SysBus 设备，再增加最小 test machine/harness。
不得篡改 K230 某个真实实例的 capability 来冒充 false-profile。

### B9：Step 4 总回归

在进入 Step 5 前完成：

- K230 PIO、enhanced SPI、internal DMA、HI_SYS、XIP 原有覆盖不回归；
- false-profile 的寄存器、传输、IRQ、reset、migration 测试全部可执行；
- external DMA、concurrent XIP、DDR/RXDS 等范围外功能仍明确拒绝或 RAZ/WI；
- 主计划和第 16 章中的完成定义全部满足。

---

## 8. Capability 和寄存器可见性策略

### 8.1 Step 4 的 capability profile

Step 4 不追求覆盖所有 DWC SSI 综合组合，而是闭合当前模型已实现的四个可测试
profile：

```text
standard-only   enhanced=false idma=false xip=false window=0
enhanced-pio    enhanced=true  idma=false xip=false window=0
enhanced-idma   enhanced=true  idma=true  xip=false window=0
full-xip        enhanced=true  idma=true  xip=true  window>0
```

K230 三实例使用 `enhanced-idma` 或 `full-xip`；它们是通用 profile 的真实消费者，
不是 K230 wrapper 的私有分支。

范围外的 external DMA、concurrent XIP、dynamic wait、DDR/RXDS 和 Octal 实际事务
执行继续保持未实现状态。

### 8.2 字段级门控

以下寄存器必须按字段生成动态 mask，不能整体 RAZ/WI：

- `CTRLR0.SPI_FRF`；
- `SPI_CTRLR0` 的共享字段和 XIP 专属字段；
- `XIP_MODE_BITS`；
- `DMACR`；
- `IMR/ISR/RISR`；
- `SR`。

专属寄存器（例如 `AXIECR/DONECR`、`XIP_INCR_INST`）可以由集中 switch 判断
RAZ/WI，但不得通过连续地址范围推断 capability。完整字段矩阵见
[第 16 章 Capability 门控实施细则](#step4-capability-gating)。

### 8.3 三层一致性

每项 capability 关闭时必须同时保证：

1. 寄存器字段不可见或不可写；
2. 传输分发、DMA/XIP handler 和 guest-memory 访问不进入对应路径；
3. 对应 raw/masked IRQ、latch、完成状态和外部 IRQ 输出无效。

reset、post-load 和 GPIO handler 也必须遵循同一 capability 状态，不能出现入口不一致。

---

## 9. 测试方案

### 9.1 本轮必须保持的现有覆盖

- register contract；
- PIO/TMOD；
- IRQ 和 PLIC；
- Dual/Quad SDR；
- SPI NOR；
- IDMA success/error；
- HI_SYS；
- XIP read window。

### 9.2 Plan B 新增断言

#### 固定配置

- 三实例 IDR/version；
- 三实例 `SPI_CTRLR0` reset；
- `SER` 按当前 `5/5/1` mask；
- system reset 后固定值恢复。

#### FIFO 深度

- K230 FIFO 仍为 256；
- `TFT=255`、`RFT=255` 合法；
- 通用 test profile 覆盖 8/16/32/64/128/256 的 realize 和边界行为；
- 当前 K230 qtest 至少覆盖非法阈值保持旧值的规则。

#### XIP aperture

- 使能前读取窗口返回 0；
- 使能后读取 Flash；
- 再关闭后返回 0；
- system reset 后窗口恢复禁用；
- QSPI 实例的三个 XIP 专用寄存器 RAZ/WI；FMC XIP 正路径单独验证。

### 9.3 Capability false-path

Step 4 必须验证：

- `has-enhanced-spi=false` 时 `SPI_FRF` RAZ/WI，且不进入 enhanced dispatch；
- `has-idma=false` 时 internal DMA 寄存器/字段无效，不访问 guest memory，不产生 DONE/AXIE；
- `has-xip=false`、`has-xip=true/window=0`、`has-xip=true/window>0` 三种合法组合；
- capability 与 reset/profile 冲突时 realize 失败；
- migration 两端 capability 不一致时拒绝装载；
- capability=false 时 post-load 不接受矛盾的活动 phase 或 latch。

external DMA 不属于本轮 capability，不应为它编写“看似通过但没有数据路径”的测试。

### 9.4 通用测试实例是 Step 4 阻塞条件

首先调查现有 QEMU qtest 容器是否能映射独立 SysBus `TYPE_DW_SSI`。若不能，增加
最小、可复用的 test machine/harness。它只负责：

- 控制器 MMIO 映射；
- IRQ 拦截；
- 可选 SSI backend/Flash；
- property profile 配置。

K230 qtest 继续负责真实 SoC 地址、PLIC、HI_SYS、Flash 和 XIP aperture 集成。
通用 test profile 负责 capability 正反路径。两者不能互相替代。

---

## 10. Step 4 实施拆分

Step 4 可以拆成多次小改动、小章节和检查点，但总目标不变：所有参数化与 capability
门控全部完成后才能进入 Step 5。

建议实施拆分：

| 子章节/检查表 | 内容 |
|---|---|
| Step 4A | `cfg`、固定值 property、realize 校验和 K230 配置表 |
| Step 4B | FIFO 深度、阈值、动态资源和 migration shape |
| Step 4C | capability 字段归属矩阵 |
| Step 4D | enhanced SPI 门控 |
| Step 4E | internal DMA 门控 |
| Step 4F | XIP engine、aperture 和 GPIO 三层门控 |
| Step 4G | 通用 false-profile 测试环境 |
| Step 4H | reset、migration、K230 回归和完成审计 |

完整 capability 细则集中在
[第 16 章 Capability 门控实施细则](#step4-capability-gating)。
后续可以按实际实施进度把 C0–C7 拆成更小检查点，但仍在本文维护。

### Step 5 边界

以下内容统一留到 Step 5：

- 把开发提交重排到最终 series；
- 决定 11 个 patch 是否增减、合并或调整顺序；
- 逐提交 build/qtest；
- checkpatch、commit message、Signed-off-by 和 cover letter；
- 面向 mailing list 的最终 diff 范围控制。

Step 4 可以保留便于迭代的开发提交，不要求每一个中间提交已经符合最终投稿顺序。

---

## 11. 实施检查点

### B0 清理后

```bash
rg -n 'K230|k230_' hw/ssi/dw_ssi.c include/hw/ssi/dw_ssi.h
git diff --check
```

### B1 固定配置后

- reset 不再根据 `max-lines` 推导 `SPI_CTRLR0`；
- K230 machine 显式设置 ID/version/profile；
- 三实例 register-contract 不变。

### B2 FIFO 后

- `fifo-depth` 校验存在；
- FIFO 只在 realize 校验通过后创建；
- `SR.TFNF/RFF` 使用配置深度；
- TFT/RFT 非法写入保持旧值。

### B3/B4 XIP 后

- QSPI0/QSPI1 `xip-window-size=0`；
- SPI-OPI 为 128 MiB；
- 只有 SPI-OPI 注册和映射 sysbus MMIO index 1；
- QSPI profile 的 XIP 专用寄存器 RAZ/WI；FMC profile 保持 XIP 正路径；
- XIP GPIO enable/disable/reset 行为正确。

### B5 migration 后

- 仅 `fifo-depth` 和 capability profile 使用 equality；
- equality 位于动态 FIFO/寄存器/phase 之前；
- post-load 不恢复与 capability 冲突的状态；
- system reset 使用目标 profile，但迁移已保证两端 profile 一致。

### B6/B7 capability 后

- capability matrix 已冻结并能追溯到 TRM/SDK；
- enhanced、IDMA、XIP 的寄存器/行为/IRQ 三层门控完成；
- shared field 未被整个寄存器 RAZ/WI；
- K230 三实例显式设置 capability。

### B8 false-profile 后

- standard-only、enhanced-pio、enhanced-idma、full-xip 均可启动测试；
- property 冲突可验证 realize 失败；
- capability=false 的字段、路径和 IRQ 均有断言。

### Step 4 完成审计

- 主计划与第 16 章完成定义全部满足；
- K230 回归和通用 false-profile 同时通过；
- 文档准确记录范围外功能；
- 只在此后进入 Step 5 的最终 patch 拆分。

---

## 12. 风险与回退策略

| 风险 | 预防措施 | 回退点 |
|---|---|---|
| 配置属性改变 reset | K230 显式设置所有固定值 | 回退 B1 单组 |
| FIFO 深度导致阈值语义变化 | 同步实现 TFT/RFT 合法性 | 回退 B2 |
| 可选 XIP region 导致 MMIO index 错误 | 只在非零实例映射 index 1 | 回退 B4 |
| QSPI IDMA mode bits 被破坏 | XIP engine 与 aperture 分离，按字段门控 | 回退 B7 XIP 小点 |
| capability 只门控寄存器但仍执行数据路径 | 三层一致性检查 | 回退对应 B7 小点 |
| false-path 无真实 K230 消费者 | 建立独立通用测试 profile | 回退 B8 harness |
| migration 两端配置不一致 | `VMSTATE_*_EQUAL` | 暂时依赖同 machine config |
| `num-cs` 证据冲突 | v2 保持 5/5/1，不夹带行为变化 | 后续独立修正 |

每一阶段都应是独立、可撤销的开发小点；开发提交如何折叠到最终 series 属于 Step 5。

---

## 13. Step 4 之后的扩展条件

enhanced SPI、internal DMA 和 XIP engine capability 已属于本 Step 4。只有满足以下
任一条件，才扩展到本轮范围之外的综合变体：

- 出现第二个真实 SoC 使用不同 DWC SSI profile；
- reviewer 明确要求支持 none/external/internal DMA 切换；
- 获得 DWC SSI Databook/CoreConsultant 配置并能完成字段矩阵；
- K230 实机证明三个实例能力不同于当前软件证据。

届时优先设计：

```text
dma-mode = none / external / internal
concurrent-xip capability
dynamic-wait capability
DDR/RXDS capability
```

并按字段而不是寄存器名字生成 readable/writable/IRQ masks。

---

## 14. Plan B 完成定义

Plan B 完成时应满足：

- `dw_ssi.c/h` 不出现 K230 类型或常量；
- K230 machine 中能直接看到三个实例的固定配置；
- FIFO 深度、ID/version、SPI profile 和 XIP aperture 不再由通用代码隐式写死；
- `has-enhanced-spi`、`has-idma`、`has-xip` 全部完成字段/行为/IRQ 门控；
- K230 的 PIO、enhanced SPI、IDMA、HI_SYS 和 XIP 行为不回归；
- capability=false 分支全部可执行并有断言；
- XIP 专用寄存器只在 FMC/XIP profile 中可见；
- reset 和 migration 不允许 capability/configuration 不一致；
- standard-only、enhanced-pio、enhanced-idma、full-xip profile 均完成验证；
- external DMA 等范围外能力仍明确未实现；
- 已满足进入 Step 5 做最终 patch 拆分的条件。

---

## 15. 参考入口

- [V2 合并决策记录](k230-spi-qspi-review-v2-decision-notes.md)
- [V2 总实施路线](k230-spi-qspi-v2-implementation-plan.md)
- [Step 3 HI_SYS 解耦](k230-spi-qspi-v2-step3-hi-sys-decoupling.md)
- [Step 4 Plan A](k230-spi-qspi-v2-step4-plana-instance-configuration.md)
- [寄存器审阅表](../spi/k230-spi-qspi-register-audit.md)
- [TRM 12.3 中文证据](../spi/reference/k230-trm-12.3-spi-cn.md)
- [当前 DWC SSI 模型](../../../qemu-camp-2026-k230/hw/ssi/dw_ssi.c)
- [当前 K230 machine](../../../qemu-camp-2026-k230/hw/riscv/k230.c)
- [当前 K230 DWC SSI qtest](../../../qemu-camp-2026-k230/tests/qtest/k230-dw-ssi-test.c)
- [QEMU DesignWare I2C 参考](../../../qemu-camp-2026-k230/hw/i2c/designware_i2c.c)
- [QEMU DesignWare I3C 参考](../../../qemu-camp-2026-k230/hw/i3c/dw-i3c.c)
- [K230 Linux DTS](../../../k230_sdk/src/little/linux/arch/riscv/boot/dts/kendryte/k230.dtsi)
- [K230 U-Boot DTS](../../../k230_sdk/src/little/uboot/arch/riscv/dts/k230.dtsi)
- [K230 U-Boot DWC SSI 驱动](../../../k230_sdk/src/little/uboot/drivers/spi/designware_spi.c)
- [K230 RT-Smart SPI 驱动](../../../k230_sdk/src/big/rt-smart/kernel/bsp/maix3/board/interdrv/spi/drv_spi.c)

---

<a id="step4-capability-gating"></a>

## 16. Step 4 Capability 门控实施细则

本章是 Plan B 的组成部分，不是独立后续计划。只有本章定义的配置、寄存器字段、
数据路径、IRQ、reset、migration 和 false-path 验证全部完成，Step 4 才算完成。

### 16.1 完成目标

Step 4 需要同时解决两类问题：

1. **参数化**：把实例固定值、资源大小和综合配置从通用模型硬编码中移出；
2. **capability 门控**：当某项可选能力不存在时，模型必须在寄存器、传输、IRQ、
   reset 和迁移层面表现一致。

不能只增加 property 而不消费它，也不能只让寄存器 RAZ/WI，却仍从其他入口触发
对应数据路径或中断。

本计划的 capability 范围是当前模型已经实现、并且寄存器审阅表能够定义
false-path 的三项能力：

- enhanced SPI；
- internal AXI DMA；
- XIP command engine。

external DMA、concurrent XIP、dynamic wait/SPITE、XIP write、HyperBus、DDR/RXDS
和 Octal 事务执行不属于本次 capability 参数化；它们在 Step 4 完成后仍保持模型
当前明确声明的未实现行为。

---

### 16.2 Property 与语义

#### 16.2.1 配置结构

```c
enum {
    DW_SSI_CAP_ENHANCED_SPI,
    DW_SSI_CAP_IDMA,
    DW_SSI_CAP_XIP,
};
```

```c
typedef struct DwSsiConfig {
    uint32_t num_cs;
    uint32_t max_lines;
    uint32_t fifo_depth;
    uint32_t component_id;
    uint32_t version_id;
    uint32_t spi_ctrlr0_reset;
    uint64_t xip_window_size;
    uint32_t capabilities;
} DwSsiConfig;
```

三个 bool 的严格语义：

| Property | 表达内容 | 不表达内容 |
|---|---|---|
| `has-enhanced-spi` | `SPI_FRF`、增强命令格式和相关共享字段存在 | 最大线宽；由 `max-lines` 表达 |
| `has-idma` | 当前模型的 internal AXI DMA profile 存在 | external DMA；本轮不实现 |
| `has-xip` | XIP command engine 和 XIP 专属字段存在 | SoC 是否映射 aperture；由 `xip-window-size` 表达 |

`xip-enable` GPIO 仍然只是运行时开关，不是 capability。

#### 16.2.2 默认值

通用类型的 optional capability 默认 `false`：

```c
DEFINE_PROP_BIT("has-enhanced-spi", DwSsiState, cfg.capabilities,
                DW_SSI_CAP_ENHANCED_SPI, false),
DEFINE_PROP_BIT("has-idma", DwSsiState, cfg.capabilities,
                DW_SSI_CAP_IDMA, false),
DEFINE_PROP_BIT("has-xip", DwSsiState, cfg.capabilities,
                DW_SSI_CAP_XIP, false),
```

同时要求 `max-lines=1`、`spi-ctrlr0-reset=0`、`xip-window-size=0`，使默认对象是
可 realize 的 standard-only profile，不能让 capability 默认值与 reset 默认值互相
冲突。

K230 machine 必须显式设置真实 profile，不能依赖通用默认值。

#### 16.2.3 K230 三实例配置

现有 TRM/SDK 证据支持以下取值：

| 物理实例 | enhanced | internal DMA | XIP engine | XIP aperture |
|---|---:|---:|---:|---:|
| QSPI0 | true | true | false | 0 |
| QSPI1 | true | true | false | 0 |
| SPI-OPI/FMC | true | true | true | 128 MiB |

QSPI0/QSPI1 的最终裁决：

- `SPI_CTRLR0` 作为 enhanced 综合寄存器继续可见，其 reset 中的 XIP 命名字段不等于实例具备 XIP engine；
- 12.3 QSPI 寄存器表没有 `0x0fc/0x100/0x104`，也没有 XIP aperture；
- 因而两个 QSPI 必须 `has-xip=false`、`xip-window-size=0`。

`XIP_MODE_BITS` 不是共享字段；普通 enhanced/IDMA 只使用 `WAIT_CYCLES` 表达等待阶段。

---

### 16.3 Realize 依赖校验

以下组合必须在创建动态资源之前拒绝：

- `has-enhanced-spi=false` 时，`max-lines` 必须为 1；
- `has-xip=true` 时，必须同时 `has-enhanced-spi=true`；
- `xip-window-size != 0` 时，必须 `has-xip=true`；
- `xip-window-size` 不得超过 4 GiB；
- `spi-ctrlr0-reset` 不得设置当前模型不支持的位；
- reset 中 capability 专属字段不得与 capability=false 冲突；
- `has-idma=false` 时不得使用 internal-DMA-only reset/profile 值；
- FIFO、CS 和线宽继续执行主计划定义的范围校验。

不要自动修正矛盾配置。realize 失败比静默修改 machine 配置更容易审阅和定位。

---

### 16.4 门控原则

#### 16.4.1 以字段为单位，不以寄存器名字为单位

以下寄存器包含多个能力共享字段，禁止整个寄存器 RAZ/WI：

- `CTRLR0`；
- `SPI_CTRLR0`；
- `DMACR`；
- `IMR/ISR/RISR`；
- `SR`。

实现应集中生成动态 mask：

```c
static uint32_t dw_ssi_ctrlr0_writable_mask(const DwSsiState *s);
static uint32_t dw_ssi_spi_ctrlr0_writable_mask(const DwSsiState *s);
static uint32_t dw_ssi_imr_valid_mask(const DwSsiState *s);
static uint32_t dw_ssi_sr_valid_mask(const DwSsiState *s);
```

专属寄存器可以通过小型 switch 判断 RAZ/WI，但不能使用连续地址范围推断能力。
DWC SSI 的不同综合布局会复用相邻甚至相同偏移。

#### 16.4.2 三层一致性

每项 capability 关闭时都必须同时满足：

1. **寄存器层**：不存在字段 RAZ/WI，reset 不注入该字段；
2. **行为层**：传输分发、DMA/XIP handler 不进入对应路径；
3. **信号层**：相关 raw/masked IRQ 和外部 IRQ 输出保持无效。

只完成其中一层不算该 capability 门控完成。

---

### 16.5 Enhanced SPI 门控

#### 16.5.1 寄存器字段

`has-enhanced-spi=false` 时：

- `CTRLR0.SPI_FRF` RAZ/WI；
- `max-lines` 只允许 1；
- `SPI_CTRLR0` 中仅由 enhanced/XIP 使用的字段按 capability matrix 处理；
- `XIP_MODE_BITS` 不属于 ordinary enhanced；由 `has-xip` 整体门控；
- `SPI_CTRLR1` 继续按当前“未实现”策略处理。

不能把整个 `CTRLR0` 或 `SPI_CTRLR0` 按地址隐藏。

#### 16.5.2 行为

- 标准 SPI 路径保持可用；
- guest 尝试设置 Dual/Quad/Octal 编码时读回仍为 0；
- 传输 dispatch 不得进入 enhanced command builder；
- 已迁移或异常状态中若出现不可能的 enhanced phase，post-load 必须拒绝，不能
  回退成标准事务继续执行。

---

### 16.6 Internal DMA 门控

#### 16.6.1 专属寄存器

`has-idma=false` 时，当前 internal-DMA profile 的以下寄存器或专属字段 RAZ/WI：

- `DMACR` internal DMA 字段；
- `AXIAWLEN`；
- `AXIARLEN`；
- `SPIDR`；
- `SPIAR`；
- `AXIAR0/1`；
- `AXIECR`；
- `DONECR`。

`0x50/0x54` 只按当前 profile 解释为 `AXIAWLEN/AXIARLEN`。本轮不伪装成 external
DMA 的 `DMATDLR/DMARDLR`。未来支持 external DMA 时必须引入明确 DMA profile，
不能在 `has-idma=false` 下自动切换偏移语义。

#### 16.6.2 混合字段和 IRQ

`has-idma=false` 时：

- `IMR` 的 `AXIEM/DONEM` RAZ/WI；
- `RISR/ISR` 的 AXIE/DONE 位始终为 0；
- 对应外部 IRQ 线始终保持低；
- `SR.CMPLTD_DF` 返回 0；
- `irq_latched` 不允许积累 IDMA-only 事件；
- 读 `AXIECR/DONECR` 不改变其他标准 IRQ 状态。

#### 16.6.3 行为

- `dw_ssi_idma_enabled()` 首先检查 capability；
- SSI enable、SER 更新、FIFO 更新均不能触发 IDMA；
- 不访问 guest physical memory；
- 不更新 AXI 地址、完成帧计数或 DONE/AXIE latch；
- 标准 PIO 和 enhanced PIO 仍可独立工作。

---

### 16.7 XIP 门控

#### 16.7.1 XIP engine 与 aperture 分离

```text
has-xip=false, window=0     -> 无 XIP engine，无 aperture
has-xip=true,  window>0     -> 有 engine，并注册第二个 MMIO region
has-xip=false, window>0     -> realize 失败
```

#### 16.7.2 字段矩阵

至少区分：

| 寄存器/字段 | enhanced | IDMA | XIP | `has-xip=false` |
|---|---:|---:|---:|---|
| `SPI_CTRLR0.TRANS_TYPE/ADDR_L/INST_L/WAIT_CYCLES` | ✓ | ✓ | ✓ | 按其他能力保留 |
| `SPI_CTRLR0.XIP_INST_EN/XIP_DFS_HC/XIP_PREFETCH_EN` |  |  | ✓ | RAZ/WI |
| `SPI_CTRLR0.XIP_MD_BIT_EN/XIP_MBL` |  |  | ✓ | 寄存器值可见但普通 enhanced/IDMA 不消费 |
| `XIP_MODE_BITS` |  |  | ✓ | RAZ/WI |
| `XIP_INCR_INST/XIP_WRAP_INST` |  |  | ✓ | RAZ/WI |
| concurrent-XIP/XIP-write 寄存器 |  |  | 未实现 | 始终按当前 RAZ/WI |

最终 mask 必须以 TRM 寄存器审阅表和 SDK 实际读写为准；上表用于约束设计方向，
不能替代逐字段复核。

#### 16.7.3 行为

- `has-xip=false` 时不创建 aperture，GPIO 输入无 XIP 副作用；
- window 非零时才注册 sysbus MMIO index 1；
- aperture read 同时要求 `has-xip`、非零 window 和运行时 `xip_enabled`；
- 关闭/复位后 aperture 返回 0，不得意外启动 SPI transaction；
- 普通 enhanced/IDMA 不读取 `XIP_MODE_BITS`；FMC XIP transaction 才读取该寄存器。

---

### 16.8 Reset 与迁移

#### 16.8.1 Reset

reset handler 应按配置生成最终值：

- 固定 ID/version 来自 `cfg`；
- `SPI_CTRLR0` reset 先读取配置，再与 capability 有效 mask 做一致性检查；
- capability=false 的专属寄存器复位为 0；
- capability=false 的 IRQ latch、完成帧计数和活动 phase 清零；
- XIP enable 属于运行状态，system reset 后关闭。

#### 16.8.2 Migration

全部 immutable 配置使用 `VMSTATE_*_EQUAL`，并置于动态状态之前：

```c
VMSTATE_UINT32_EQUAL(cfg.num_cs, DwSsiState),
VMSTATE_UINT32_EQUAL(cfg.max_lines, DwSsiState),
VMSTATE_UINT32_EQUAL(cfg.fifo_depth, DwSsiState),
VMSTATE_UINT32_EQUAL(cfg.component_id, DwSsiState),
VMSTATE_UINT32_EQUAL(cfg.version_id, DwSsiState),
VMSTATE_UINT32_EQUAL(cfg.spi_ctrlr0_reset, DwSsiState),
VMSTATE_UINT64_EQUAL(cfg.xip_window_size, DwSsiState),
VMSTATE_UINT32_EQUAL(cfg.capabilities, DwSsiState),
```

post-load 还必须验证动态状态与 capability 相容：

- 无 IDMA capability 时不能恢复 IDMA phase、DONE/AXIE latch 或完成帧计数；
- 无 enhanced capability 时不能恢复 enhanced transaction phase；
- 无 XIP capability 时不能恢复 active XIP transaction；
- 不执行完整 reset 覆盖源端运行状态。

---

### 16.9 False-path 验证基础设施

K230 三实例的三个 capability 均为 true，不能只靠 K230 正常 profile 证明门控正确。
Step 4 必须提供可执行 false-profile。

优先顺序：

1. 先调查能否使用现有 QEMU 测试容器把独立 SysBus `TYPE_DW_SSI` 实例映射到
   qtest 地址空间；
2. 如果现有容器无法完成映射，增加最小、可复用的 DWC SSI test machine/harness；
3. test harness 只负责 MMIO、IRQ 和可选 SPI backend 接线，不包含 K230 行为；
4. 不通过篡改某个 K230 物理实例的真实 capability 来制造 false-path。

至少提供以下 profile：

| Profile | enhanced | IDMA | XIP | window |
|---|---:|---:|---:|---:|
| standard-only | false | false | false | 0 |
| enhanced-pio | true | false | false | 0 |
| enhanced-idma | true | true | false | 0 |
| full-xip | true | true | true | 非零 |

必须覆盖：

- property 组合 realize 成功/失败；
- capability=false 字段 RAZ/WI；
- capability=true 字段正常保存和读回；
- standard-only 不进入 enhanced/IDMA/XIP 数据路径；
- IDMA false 时 DONE/AXIE 和对应 IRQ 永不产生；
- XIP engine 与 aperture 的三种合法组合；
- reset 后 capability 专属状态恢复正确；
- migration 配置不一致时拒绝装载。

测试基础设施可以继续拆成更小的实施检查点，但执行结果不能推迟到 Step 5。

---

### 16.10 分步实施

为了控制每次修改范围，capability 工作拆为：

#### C0：冻结字段矩阵

- 逐字段标注 standard/enhanced/IDMA/XIP 归属；
- 标出共享字段；
- 对证据不足字段保持当前未实现策略。

#### C1：引入 capability properties 和依赖校验

- 只增加 `cfg` 字段、properties、K230 显式配置和 realize 校验；
- 暂不改变寄存器行为；
- 该中间状态只用于开发，不能作为 Step 4 完成状态。

#### C2：Enhanced SPI 门控

- 动态 `CTRLR0/SPI_CTRLR0` mask；
- dispatch/phase/reset/post-load 门控；
- standard-only false-path。

#### C3：Internal DMA 门控

- 专属寄存器和混合 IRQ 字段；
- 触发、guest memory、完成状态门控；
- enhanced-pio false-path。

#### C4：XIP engine 门控

- 专属与共享字段矩阵；
- engine/window/GPIO 三层分离；
- window=0 与 has-xip=false 的不同语义。

#### C5：统一 reset 和 migration

- capability-compatible reset；
- 全配置 equality；
- post-load 动态状态合法性。

#### C6：False-profile 测试环境

- 建立最小通用测试实例；
- 覆盖四类 profile；
- K230 qtest 继续验证真实 SoC 集成。

#### C7：Step 4 总回归和文档收口

- K230 原功能不回归；
- capability 正反路径全部可执行；
- 更新主计划、寄存器矩阵和实现状态；
- 不在此阶段进行最终 11-patch 拆分。

---

### 16.11 完成定义

以下条件全部满足，本实施细则才完成：

- 三个 capability property 都实际控制寄存器、行为和 IRQ；
- 没有按整个混合寄存器粗粒度隐藏字段；
- XIP 专用寄存器只在 `has-xip=true` 的 FMC profile 中可见；
- capability=false 不产生隐藏的数据搬运、状态变化或 IRQ；
- reset 和 migration 不允许恢复与 capability 冲突的状态；
- K230 machine 显式设置真实 profile；
- standard-only、enhanced-pio、enhanced-idma 和 full-xip 均有可执行测试；
- external DMA 等超出本轮范围的功能仍被明确拒绝或 RAZ/WI；
- Step 4 文档状态更新为完成后，才进入 Step 5 的 patch 拆分和重排。

---
