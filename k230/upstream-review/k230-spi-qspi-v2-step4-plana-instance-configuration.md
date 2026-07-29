# K230 SPI/QSPI V2 Step 4 Plan A：实例配置与 Capability 参数化

首次记录：2026-07-29
修订：2026-07-29（v2，Review 后拆分 4A/4B，修正 capability 门控为字段级 mask）

> **历史方案：禁止作为实施依据。** 本文已由 [Step 4 Plan Final](k230-spi-qspi-v2-step4-plan-final-instance-configuration.md) 取代。文中“SDK 的 IDMA 1-4-4 路径复用 `XIP_MODE_BITS`”判断已被 TRM 与 RT-Smart/U-Boot/Linux SDK 源码复核推翻：普通 enhanced/IDMA 不读取 XIP mode fields；`XIP_MODE_BITS`、`XIP_INCR_INST`、`XIP_WRAP_INST` 均按 XIP capability 门控。本文的 `dma-mode` 枚举和字段级 mask 方案也不再采用。

本文是 [V2 实施路线](k230-spi-qspi-v2-implementation-plan.md) 第四步"引入实例配置"的执行细化。所有文件路径与行号基于实际代码探索（第三步已完成，分支 `k230-spiv3.4`），代码仓库位于 `my-qemu-camp-2026-k230/`。

> **命名约定**：第二步重命名已完成（`dw_ssi.*`、`DwSsiState`），第三步 GPIO 解耦已完成。本文基于解耦后的代码状态。

> **当时的 v2 修订说明（已失效）**：以下章节保留当时推演过程，不代表最终事实。最终裁决见 Plan Final。

---

## 摘要

将 SSI 控制器中剩余的硬编码实例配置改为 QOM 属性，使通用 DesignWare SSI 模型可被任意 SoC 通过属性配置实例化。分两个阶段：

- **Step 4A**：纯参数化（复位值、FIFO 深度、XIP window size），不改寄存器可见性
- **Step 4B**：capability 门控（enhanced SPI、DMA mode、XIP engine），需要字段级 writable/read mask

---

## 当前状态分析

### 已有 QOM 属性（第二步已引入）

`hw/ssi/dw_ssi.c:1722-1725`：

```c
static const Property dw_ssi_properties[] = {
    DEFINE_PROP_UINT32("num-cs", DwSsiState, num_cs, 1),
    DEFINE_PROP_UINT32("max-lines", DwSsiState, max_lines, 1),
};
```

三实例在 `hw/riscv/k230.c:246-251` 设置：

| 实例 | num-cs | max-lines |
|------|--------|-----------|
| dw_ssi[0] (QSPI0) | 5 | 4 |
| dw_ssi[1] (QSPI1) | 5 | 4 |
| dw_ssi[2] (SPI-OPI) | 1 | 8 |

### 待属性化的硬编码清单

| # | 属性名 | 类型 | 当前位置 | 当前值 | 阶段 |
|---|--------|------|----------|--------|------|
| 1 | `component-id` | uint32 | dw_ssi.c:32, reset:1581 | `0xa1b2c3d5` | 4A |
| 2 | `version-id` | uint32 | dw_ssi.c:37, reset:1582 | `0x3130332a` | 4A |
| 3 | `spi-ctrlr0-reset` | uint32 | dw_ssi.c:33-34, reset:1583-1585 | `0x04000200`/`0x28000200` | 4A |
| 4 | `fifo-depth` | uint32 | dw_ssi.c:27 | 256 | 4A |
| 5 | `xip-window-size` | uint64 | dw_ssi.h:28 | `0x08000000` (128 MiB) | 4A |
| 6 | `has-enhanced-spi` | bool | 隐式：max_lines+SPI_FRF | — | 4B |
| 7 | `dma-mode` | enum | 隐式：始终可用（internal DMA） | — | 4B |
| 8 | `has-xip` | bool | 隐式：GPIO+始终创建 region | — | 4B |

### 当前 reset handler 中的硬编码

`hw/ssi/dw_ssi.c:1562-1588` `dw_ssi_enter_reset`：

```c
s->regs[R_IDR] = DW_SSI_IDR_RESET;                  // IDR 复位值
s->regs[R_SSIC_VERSION_ID] = DW_SSI_VERSION;        // VERSION_ID 复位值
s->regs[R_SPI_CTRLR0] = s->max_lines == 8 ?         // 二选一
    DW_SSI_SPI_CTRLR0_FMC_RESET :
    DW_SSI_SPI_CTRLR0_SPI_RESET;
```

**关键耦合**：`R_SPI_CTRLR0` 复位值通过 `max_lines == 8` 判断"是否 OPI 模式"，把"最大数据线数"和"SPI_CTRLR0 复位配置"耦合在一起。属性化后解耦：`spi-ctrlr0-reset` 直接给出复位值。

### 共享寄存器问题（4B 需处理的）

以下寄存器包含多个 capability 的字段，不能整个寄存器 RAZ/WI：

**SPI_CTRLR0**（dw_ssi.c:158-174）包含：
- Enhanced SPI：TRANS_TYPE、ADDR_L、INST_L、WAIT_CYCLES、XIP_MD_BIT_EN
- DDR/RXDS：SPI_DDR_EN、INST_DDR_EN、SPI_RXDS_EN、SPI_RXDS_SIG_EN
- XIP：XIP_INST_EN、XIP_MBL、XIP_PREFETCH_EN、XIP_DFS_HC、SSIC_XIP_CONT_XFER_EN
- DMA：SPI_DM_EN
- Clock stretching：CLK_STRETCH_EN

**已确认的源码缺陷**：当时 `dw_ssi.c:948-953` 错误地让 IDMA 1-4-4 路径读取 `XIP_MODE_BITS`：
```c
/* The SDK's 1-4-4 read supplies its mode byte through XIP_MODE_BITS
 * without XIP_MD_BIT_EN */
dw_ssi_send_enhanced_field(s, s->regs[R_XIP_MODE_BITS], 8);
```

**IMR/ISR/RISR** 混合基础 IRQ + IDMA IRQ（DONEM bit 11、AXIEM bit 8）+ XIP IRQ（XRXOIM bit 6）。

**结论**：4B 的 capability 门控必须是字段级的，不能按整个寄存器地址隐藏。

### 现有 RAZ/WI 机制

`hw/ssi/dw_ssi.c:1201-1220` 已有 `dw_ssi_is_razwi()`，在 read/write 中调用。4B 扩展此机制为字段级 mask。

### 当前三实例配置差异

| 字段 | dw_ssi[0] (QSPI0) | dw_ssi[1] (QSPI1) | dw_ssi[2] (SPI-OPI) |
|------|-------|-------|-------|
| `num-cs` | 5 | 5 | 1 |
| `max-lines` | 4 | 4 | 8 |
| SPI_CTRLR0 reset | `0x04000200` (SPI) | `0x04000200` (SPI) | `0x28000200` (FMC) |
| XIP region | 创建但未映射 | 创建但未映射 | 创建且映射到 0xC0000000 |
| `xip-enable` GPIO | 未连接 | 未连接 | 连接到 HI_SYS |
| IDR | `0xa1b2c3d5` | `0xa1b2c3d5` | `0xa1b2c3d5` |
| VERSION_ID | `0x3130332a` | `0x3130332a` | `0x3130332a` |
| FIFO depth | 256 | 256 | 256 |
| enhanced-spi | 隐式 true | 隐式 true | 隐式 true |
| DMA mode | 隐式 internal | 隐式 internal | 隐式 internal |
| XIP engine | 隐式 false | 隐式 false | 隐式 true |

---

## 讲解：QOM 属性机制

QEMU 的 QOM (QEMU Object Model) 属性系统是设备配置的标准接口。属性在 `class_init` 阶段通过 `DEFINE_PROP_*` 宏注册到 `Property[]` 数组，由 `device_class_set_props()` 绑定到设备类。

**生命周期**：
1. `instance_init`：设备对象创建，属性字段已有默认值
2. **machine 设置属性**：`qdev_prop_set_uint32()` / `qdev_prop_set_bool()` 在 realize 前设置
3. `realize`：属性值固定，可在此校验合法性

**关键点**：属性设置发生在 instance_init 之后、realize 之前。因此 `instance_init` 中不能依赖属性值做条件初始化（如条件创建 memory region）——此时属性还是默认值。需要在 `realize` 中做条件创建。

---

## Step 4A：安全实例参数（纯参数化，不改寄存器可见性）

Step 4A 只做参数替换：硬编码值改为可配置属性，但不改变任何寄存器的存在性或可见性。所有寄存器保持当前行为，三实例行为完全不变。

### 改动 4A-1：复位值属性（component-id, version-id, spi-ctrlr0-reset）

**风险：最低**（纯复位值，不影响运行时逻辑）

#### 讲解：spi-ctrlr0-reset 解耦

当前 `s->max_lines == 8` 选择 SPI_CTRLR0 复位值，把两个独立概念耦合：
- `max-lines`：控制器支持的最大数据线数（影响 SPI_FRF 校验）
- `SPI_CTRLR0` 复位值：增强 SPI 控制寄存器的初始配置（instruction length、address length、wait cycles 等）

解耦后通用模型不需要知道"8 线意味着什么复位值"——由 machine 负责按 TRM 设置。

#### 代码改动

**文件：`include/hw/ssi/dw_ssi.h`** — DwSsiState 新增字段：

```c
struct DwSsiState {
    ...
    uint32_t component_id;        /* IDR 复位值 */
    uint32_t version_id;          /* SSIC_VERSION_ID 复位值 */
    uint32_t spi_ctrlr0_reset;    /* SPI_CTRLR0 复位值 */
    ...
};
```

**文件：`hw/ssi/dw_ssi.c`** — 属性数组新增：

```c
static const Property dw_ssi_properties[] = {
    DEFINE_PROP_UINT32("num-cs", DwSsiState, num_cs, 1),
    DEFINE_PROP_UINT32("max-lines", DwSsiState, max_lines, 1),
    DEFINE_PROP_UINT32("component-id", DwSsiState, component_id, 0),
    DEFINE_PROP_UINT32("version-id", DwSsiState, version_id, 0),
    DEFINE_PROP_UINT32("spi-ctrlr0-reset", DwSsiState, spi_ctrlr0_reset, 0),
};
```

> **默认值 0**：通用模型的默认值为中性值。当前未上游，无兼容性包袱。K230 machine 必须显式设置全部 K230 值。

reset handler 改为读属性：

```c
s->regs[R_IDR] = s->component_id;
s->regs[R_SSIC_VERSION_ID] = s->version_id;
s->regs[R_SPI_CTRLR0] = s->spi_ctrlr0_reset;       /* 不再依赖 max_lines */
```

删除不再使用的宏（`DW_SSI_IDR_RESET`、`DW_SSI_VERSION`、`DW_SSI_SPI_CTRLR0_SPI_RESET`、`DW_SSI_SPI_CTRLR0_FMC_RESET`），或保留为注释引用。

**文件：`hw/riscv/k230.c`** — machine 显式设置全部三项：

```c
for (int i = 0; i < ARRAY_SIZE(s->dw_ssi); i++) {
    qdev_prop_set_uint32(DEVICE(&s->dw_ssi[i]), "component-id", 0xa1b2c3d5);
    qdev_prop_set_uint32(DEVICE(&s->dw_ssi[i]), "version-id", 0x3130332a);
}
qdev_prop_set_uint32(DEVICE(&s->dw_ssi[0]), "spi-ctrlr0-reset", 0x04000200);
qdev_prop_set_uint32(DEVICE(&s->dw_ssi[1]), "spi-ctrlr0-reset", 0x04000200);
qdev_prop_set_uint32(DEVICE(&s->dw_ssi[2]), "spi-ctrlr0-reset", 0x28000200);
```

> **commit message 必须注明**这些值来自 K230 TRM 实例配置。

---

### 改动 4A-2：fifo-depth

**风险：低**（影响 FIFO 行为，含合法性校验）

#### 讲解：FIFO 容量的完整影响面

`DW_SSI_FIFO_CAPACITY` (dw_ssi.c:27) 在 4 处使用：
- `dw_ssi_init` (dw_ssi.c:1688-1689)：`fifo32_create()` 创建 TX/RX FIFO
- `dw_ssi_status` (dw_ssi.c:426)：计算 `SR.TFNF`（TX FIFO not full）
- `dw_ssi_status` (dw_ssi.c:430)：计算 `SR.RFF`（RX FIFO full）

属性化后还需考虑：FIFO 深度决定 TXFTLR.TFT / RXFTLR.RFT 的合法范围。当前按 8 位 mask 保存，未校验——属性化时一并修复。

#### 代码改动

**文件：`include/hw/ssi/dw_ssi.h`** — 新增字段：

```c
uint32_t fifo_depth;    /* 新增 */
```

**文件：`hw/ssi/dw_ssi.c`**：

属性数组新增：

```c
DEFINE_PROP_UINT32("fifo-depth", DwSsiState, fifo_depth, 256),
```

将 `fifo32_create()` 从 `dw_ssi_init` 移到 `dw_ssi_realize`：

```c
/* dw_ssi_realize() 中，在属性校验之后 */
fifo32_create(&s->tx_fifo, s->fifo_depth);
fifo32_create(&s->rx_fifo, s->fifo_depth);
```

`dw_ssi_finalize` 中的 `fifo32_destroy` 保留不变。

4 个使用点 `DW_SSI_FIFO_CAPACITY` 改为 `s->fifo_depth`。

**新增合法性校验**（`dw_ssi_realize` 中）：

```c
if (s->fifo_depth < 8 || s->fifo_depth > 256) {
    error_setg(errp, "%s: fifo-depth must be in range 8..256",
               dev->canonical_path);
    return;
}
```

> 范围 8..256 基于当前 IP 已知参数。如果 TRM 只允许特定离散值（8/16/32/64/128/256），可追加离散值校验。

**TXFTLR/RXFTLR 阈值校验修复**（顺带修正现有问题，write handler 中）：

```c
case A_TXFTLR:
    /* 阈值不能超过 FIFO 深度 */
    if (FIELD_EX32(value, TXFTLR, TFT) >= s->fifo_depth) {
        return;  /* 非法值，保留旧值 */
    }
    break;
case A_RXFTLR:
    if (FIELD_EX32(value, RXFTLR, RFT) >= s->fifo_depth) {
        return;
    }
    break;
```

**文件：`tests/qtest/k230-dw-ssi-test.c`** — `K230_SSI_FIFO_DEPTH` 宏保持 256。

---

### 改动 4A-3：xip-window-size（条件 region 创建）

**风险：低**（当前 QSPI0/1 的 region 从未被映射，移除无影响）

#### 讲解：为什么条件创建

当前三个实例都创建了 128 MiB 的 XIP MMIO region，但 machine 只映射了 dw_ssi[2] 的。前两个实例暴露了一个从不被使用的 sysbus MMIO region——这不符合设备能力模型。

注意：`memory_region_init_io()` 创建的是 I/O 地址范围描述和回调对象，不会分配 128 MiB RAM。条件创建的价值在于**避免暴露不存在的 sysbus MMIO region**，使设备能力表达更准确。

#### 代码改动

**文件：`include/hw/ssi/dw_ssi.h`** — 新增字段：

```c
uint64_t xip_window_size;    /* 新增 */
```

**文件：`hw/ssi/dw_ssi.c`**：

属性数组新增：

```c
DEFINE_PROP_UINT64("xip-window-size", DwSsiState, xip_window_size, 0),
```

**默认值 0**：表示不创建 XIP region。只有 machine 显式设置非零值的实例才有 XIP aperture。

`dw_ssi_init` 中删除 XIP region 创建，移到 `dw_ssi_realize`：

```c
/* dw_ssi_realize() 中 */
if (s->xip_window_size > 0) {
    memory_region_init_io(&s->xip, OBJECT(dev), &dw_ssi_xip_ops, s,
                          TYPE_DW_SSI ".xip", s->xip_window_size);
    sysbus_init_mmio(SYS_BUS_DEVICE(dev), &s->xip);
}
```

**历史说明**：这一结论已被 Plan Final 推翻。正确顺序是先在 Step 4.0 删除 IDMA 对 `XIP_MODE_BITS` 的错误依赖，再由 `has-xip` 统一门控三个 XIP 专用寄存器；QSPI profile 对它们 RAZ/WI。

**文件：`hw/riscv/k230.c`** — machine 设置：

```c
qdev_prop_set_uint64(DEVICE(&s->dw_ssi[2]), "xip-window-size", 0x08000000);
/* QSPI0/1 不设置，默认 0，不创建 XIP region */
```

---

## Step 4B：capability 门控（字段级 mask）

Step 4B 引入 capability 属性，需要**字段级**而非寄存器级门控。实施前必须建立寄存器/字段 → capability 矩阵。

### 改动 4B-1：has-enhanced-spi（字段级 writable mask）

**风险：中高**（影响 CTRLR0.SPI_FRF 和 SPI_CTRLR0 部分字段）

#### 讲解：为什么 need 字段级 mask

`has-enhanced-spi=false` 不意味着"整个 SPI_CTRLR0 寄存器消失"。该寄存器还包含 DDR/RXDS、XIP、DMA、clock stretching 字段。只能 mask 增强 SPI 相关字段：

- CTRLR0.SPI_FRF（bit 22-23）：写入忽略、读返回 0
- SPI_CTRLR0.TRANS_TYPE/ADDR_L/INST_L/WAIT_CYCLES/XIP_MD_BIT_EN：写入忽略、读返回 0

传输分发中，guest 写入非零 SPI_FRF 时不应静默退化到标准单线传输——那是语义错误。应在寄存器写入阶段阻止该配置。

#### 代码改动

**文件：`include/hw/ssi/dw_ssi.h`** — 新增字段：

```c
bool has_enhanced_spi;    /* 新增 */
```

**文件：`hw/ssi/dw_ssi.c`**：

属性：

```c
DEFINE_PROP_BOOL("has-enhanced-spi", DwSsiState, has_enhanced_spi, false),
```

字段级 mask 辅助函数：

```c
/* CTRLR0 的 writable mask：has-enhanced-spi=false 时屏蔽 SPI_FRF */
static uint32_t dw_ssi_ctrlr0_writable_mask(const DwSsiState *s)
{
    uint32_t mask = R_CTRLR0_DFS_MASK | R_CTRLR0_CFS_MASK |
                    R_CTRLR0_SRL_MASK | R_CTRLR0_SSTE_MASK |
                    R_CTRLR0_TMOD_MASK | R_CTRLR0_SCPHA_MASK |
                    R_CTRLR0_SCPOL_MASK;
    if (s->has_enhanced_spi) {
        mask |= R_CTRLR0_SPI_FRF_MASK;
    }
    return mask;
}

/* SPI_CTRLR0 的 writable mask：has-enhanced-spi=false 时屏蔽增强字段 */
static uint32_t dw_ssi_spi_ctrlr0_writable_mask(const DwSsiState *s)
{
    uint32_t mask = R_SPI_CTRLR0_CLK_STRETCH_EN_MASK;
    /* DDR/RXDS 字段暂时全开放（K230 三实例都用） */
    if (s->has_enhanced_spi) {
        mask |= R_SPI_CTRLR0_TRANS_TYPE_MASK |
                R_SPI_CTRLR0_ADDR_L_MASK |
                R_SPI_CTRLR0_INST_L_MASK |
                R_SPI_CTRLR0_WAIT_CYCLES_MASK |
                R_SPI_CTRLR0_XIP_MD_BIT_EN_MASK;
    }
    /* XIP/DMA 相关字段由各自 capability 控制（4B-2/4B-3） */
    return mask;
}
```

> 具体 mask 值需核对实际寄存器位布局。以上为示意，使用 R_*_MASK 宏或显式 `0x...` 值。

read 中写入时应用 mask：

```c
value &= dw_ssi_ctrlr0_writable_mask(s);   /* ctrlr0 */
value &= dw_ssi_spi_ctrlr0_writable_mask(s); /* spi_ctrlr0 */
s->regs[addr / sizeof(uint32_t)] = value;
```

传输分发门控：

```c
if (s->has_enhanced_spi &&
    FIELD_EX32(s->regs[R_CTRLR0], CTRLR0, SPI_FRF) != 0) {
    /* 增强传输 */
} else {
    /* 标准传输 */
}
```

**文件：`hw/riscv/k230.c`** — machine 设置：

```c
for (int i = 0; i < ARRAY_SIZE(s->dw_ssi); i++) {
    qdev_prop_set_bool(DEVICE(&s->dw_ssi[i]), "has-enhanced-spi", true);
}
```

---

### 改动 4B-2：dma-mode（枚举：none / external / internal）

**风险：高**（影响 IDMA 寄存器可见性、IRQ 行为、SR.CMPLTD_DF）

#### 讲解：为什么用枚举而非 bool

Linux 通用驱动本身就用 `SSIC_HAS_DMA` 三态（0/1/2）区分无 DMA / external DMA / internal DMA。当前 K230 三实例都是 internal DMA，但通用模型应为其他配置留接口。

**枚举三个值**：
- `none`：无 DMA。所有 IDMA 寄存器 RAZ/WI，IDMA IRQ 保持低
- `external`：外部 DMA。当前未实现，行为等同 `none`（预留接口）
- `internal`：内部 AXI DMA。当前实现

#### IDMA 相关寄存器/字段/IRQ 完整清单

| 项目 | 位置 | has-idma=false 时行为 |
|------|------|----------------------|
| DMACR | 寄存器 | RAZ/WI |
| AXIAWLEN | 寄存器 | RAZ/WI |
| AXIARLEN | 寄存器 | RAZ/WI |
| SPIDR | 寄存器 | RAZ/WI |
| SPIAR | 寄存器 | RAZ/WI |
| AXIAR0/1 | 寄存器 | RAZ/WI |
| AXIECR | 寄存器 | RAZ/WI |
| DONECR | 寄存器 | RAZ/WI |
| IMR.DONEM (bit 11) | 字段 | 写入忽略、读 0 |
| IMR.AXIEM (bit 8) | 字段 | 写入忽略、读 0 |
| ISR/RISR 对应位 | 字段 | 读 0 |
| SR.CMPLTD_DF | 字段 | 必须为 0 |
| irq_latched (IDMA 位) | 状态 | capability mask 清除 |
| DONE/AXIE IRQ | 输出 | 保持低电平 |

#### 代码改动

**文件：`include/hw/ssi/dw_ssi.h`** — 新增枚举和字段：

```c
typedef enum {
    DW_SSI_DMA_NONE,
    DW_SSI_DMA_EXTERNAL,    /* 预留，当前未实现 */
    DW_SSI_DMA_INTERNAL,
} DwSsiDmaMode;

struct DwSsiState {
    ...
    DwSsiDmaMode dma_mode;    /* 新增 */
    ...
};
```

**文件：`hw/ssi/dw_ssi.c`**：

属性（使用 string 类型映射枚举）：

```c
/* 简单方案：先用 uint32（0=none, 1=external, 2=internal） */
DEFINE_PROP_UINT32("dma-mode", DwSsiState, dma_mode, DW_SSI_DMA_NONE),
```

或使用 QEMU 的 `DEFINE_PROP_UNSIGNED` + 手动校验。

IDMA 相关的 writable mask：

```c
static bool dw_ssi_is_idma_reg(hwaddr addr)
{
    switch (addr) {
    case A_DMACR:
    case A_AXIAWLEN:
    case A_AXIARLEN:
    case A_SPIDR:
    case A_SPIAR:
    case A_AXIAR0:
    case A_AXIAR1:
    case A_AXIECR:
    case A_DONECR:
        return true;
    default:
        return false;
    }
}
```

IMR 字段级 mask：

```c
static uint32_t dw_ssi_imr_writable_mask(const DwSsiState *s)
{
    uint32_t mask = R_IMR_TXEIM_MASK | R_IMR_TXOIM_MASK |
                    R_IMR_RXUIM_MASK | R_IMR_RXOIM_MASK |
                    R_IMR_MSTIM_MASK | R_IMR_RXFIM_MASK;
    if (s->dma_mode == DW_SSI_DMA_INTERNAL) {
        mask |= R_IMR_DONEM_MASK | R_IMR_AXIEM_MASK;
    }
    return mask;
}
```

IRQ valid mask：

```c
static uint32_t dw_ssi_irq_valid_mask(const DwSsiState *s)
{
    uint32_t mask = DW_SSI_IRQ_BASE_MASK;
    if (s->dma_mode == DW_SSI_DMA_INTERNAL) {
        mask |= DW_SSI_IRQ_IDMA_MASK;
    }
    return mask;
}
```

IDMA 触发门控：

```c
static bool dw_ssi_idma_enabled(const DwSsiState *s)
{
    return s->dma_mode == DW_SSI_DMA_INTERNAL &&
           FIELD_EX32(s->regs[R_DMACR], DMACR, IDMAE);
}
```

SR.CMPLTD_DF 清零（reset 和 irq_raw_status 中 cap mask）：

```c
if (s->dma_mode != DW_SSI_DMA_INTERNAL) {
    s->irq_latched &= ~(R_RISR_DONER_MASK | R_RISR_AXIER_MASK);
}
```

**文件：`hw/riscv/k230.c`** — machine 设置：

```c
for (int i = 0; i < ARRAY_SIZE(s->dw_ssi); i++) {
    qdev_prop_set_uint32(DEVICE(&s->dw_ssi[i]), "dma-mode",
                         DW_SSI_DMA_INTERNAL);
}
```
---

### 改动 4B-3：has-xip（字段级 mask + region 条件创建）

**风险：高**（影响 XIP 寄存器字段、XIP engine 行为）

#### 讲解：XIP 字段的精确分类

必须区分三类 XIP 相关寄存器：
1. **纯 XIP 寄存器**：XIP_CTRL、XIP_SER、XRXOICR、XIP_CNT_TIME_OUT、XIP_INCR_INST、XIP_WRAP_INST、XIP_WRITE_INCR_INST、XIP_WRITE_WRAP_INST、XIP_WRITE_CTRL——这些**可以**整个寄存器 RAZ/WI
2. **共享寄存器中的 XIP 字段**：SPI_CTRLR0（XIP_INST_EN、XIP_MBL、XIP_PREFETCH_EN 等）、IMR/ISR/RISR（XRXOIM 等）——这些需要字段级 mask
3. **历史误判**：曾把 `XIP_MODE_BITS` 当作 IDMA 共享寄存器；最终证据确认它与 `XIP_INCR_INST/XIP_WRAP_INST` 一样属于 XIP-only，应由 `has-xip` 隐藏

#### 代码改动

**文件：`include/hw/ssi/dw_ssi.h`** — 新增字段：

```c
bool has_xip;                /* 新增 */
```

**文件：`hw/ssi/dw_ssi.c`**：

属性：

```c
DEFINE_PROP_BOOL("has-xip", DwSsiState, has_xip, false),
```

纯 XIP 寄存器（可整个隐藏）：

```c
static bool dw_ssi_is_xip_only_reg(hwaddr addr)
{
    switch (addr) {
    case A_XIP_CTRL:
    case A_XIP_SER:
    case A_XIP_INCR_INST:
    case A_XIP_WRAP_INST:
    case A_XIP_CNT_TIME_OUT:
    case A_XIP_WRITE_INCR_INST:
    case A_XIP_WRITE_WRAP_INST:
    case A_XIP_WRITE_CTRL:
    case A_XRXOICR:
        return true;
    default:
        return false;
    }
}
/* Plan Final：A_XIP_MODE_BITS 也在 XIP-only 列表 */
```

混合寄存器中的 XIP 字段 mask（追加到 4B-1 的 SPI_CTRLR0 mask 中）：

```c
/* SPI_CTRLR0 writable mask 中追加 */
if (s->has_xip) {
    mask |= R_SPI_CTRLR0_XIP_INST_EN_MASK |
            R_SPI_CTRLR0_XIP_MBL_MASK |
            R_SPI_CTRLR0_XIP_PREFETCH_EN_MASK |
            R_SPI_CTRLR0_XIP_DFS_HC_MASK |
            R_SPI_CTRLR0_SSIC_XIP_CONT_XFER_EN_MASK;
}
```

```c
/* IMR writable mask 中追加 */
if (s->has_xip) {
    mask |= R_IMR_XRXOIM_MASK;
}
```

统一 read/write 门控（替代现有 `dw_ssi_is_razwi`）：

```c
static bool dw_ssi_reg_is_readable(const DwSsiState *s, hwaddr addr)
{
    /* 原有未实现寄存器 */
    if (dw_ssi_is_razwi(addr)) {
        return false;
    }
    /* 纯 XIP 寄存器：has-xip=false 时不可读 */
    if (!s->has_xip && dw_ssi_is_xip_only_reg(addr)) {
        return false;
    }
    /* 纯 IDMA 寄存器：dma-mode != internal 时不可读 */
    if (s->dma_mode != DW_SSI_DMA_INTERNAL && dw_ssi_is_idma_reg(addr)) {
        return false;
    }
    return true;
}
```

read handler 中：

```c
if (!dw_ssi_reg_is_readable(s, addr)) {
    trace_dw_ssi_reg_read(s, addr, 0);
    return 0;
}
```

write handler 中同样检查 + 应用字段级 mask。

XIP engine 行为门控：

```c
static uint64_t dw_ssi_xip_read(void *opaque, hwaddr address, ...)
{
    DwSsiState *s = DW_SSI(opaque);
    if (!s->has_xip || !s->xip_enabled) {
        return 0;
    }
    ...
}
```

**文件：`hw/riscv/k230.c`** — machine 设置：

```c
for (int i = 0; i < 2; i++) {
    qdev_prop_set_bool(DEVICE(&s->dw_ssi[i]), "has-xip", false);
}
qdev_prop_set_bool(DEVICE(&s->dw_ssi[2]), "has-xip", true);
```

---

## "通用化"设计原则

### 1. 属性名是功能名

`fifo-depth`（不是 `k230-fifo-depth`）、`dma-mode`（不是 `k230-dma`）。任何 SoC 都可配置。

### 2. 默认值是安全降级

capability 默认 `false`/`none`。只有 machine 显式设置才启用。复位值的通用默认值为中性值（0），不隐含任何 SoC profile。

### 3. 字段级 mask 而非寄存器级隐藏

共享寄存器只有 `SPI_CTRLR0`、`CTRLR0`、`IMR/ISR/RISR`；`XIP_MODE_BITS` 不是共享寄存器，QSPI profile 下整个寄存器 RAZ/WI。

### 4. 属性设置时机

```
instance_init -> machine 设置属性 -> realize -> reset
     ^                              ^          ^
  注册 GPIO/IRQ               校验 + 条件创建   用属性值初始化
```

FIFO 创建和 XIP region 创建从 `instance_init` 移到 `realize`。

---

## 假设与决策

1. **三实例 capability 取值**：`has-enhanced-spi=true` / `dma-mode=internal`（全实例），`has-xip=false`（QSPI0/1）/ `true`（SPI-OPI）。
2. **`spi-ctrlr0-reset` 解耦**：单一 uint32 属性，替代 `max_lines==8` 条件。
3. **XIP region 条件创建**：`xip-window-size=0` 时不创建 XIP region。QSPI0/1 不再暴露不存在的 sysbus MMIO region。
4. **FIFO 创建移到 realize**：`fifo32_create()` 从 instance_init 移到 realize，因需要读 `fifo_depth` 属性。
5. **fifo-depth 校验**：范围 8..256。TXFTLR/RXFTLR 阈值写入校验（阈值 < fifo_depth）。
6. **VMState**：properties 属于不可变设备配置，由目标端 machine 在加载迁移状态前设置；迁移要求源端与目标端配置一致。`fifo_depth` 影响 FIFO VMState buffer 长度，建议用 `VMSTATE_UINT32_EQUAL()` 检查一致性。
7. **XIP_MODE_BITS 由 has-xip 隐藏**：普通 enhanced/IDMA 不读取它，只有真正 XIP transaction 使用。
8. **SPI_CTRLR0 的 DDR/RXDS 字段**暂不门控（K230 三实例都使用），后续可追加 capability。
9. **dma-mode 枚举**：当前 `external` 未实现（行为等同 `none`），但接口已预留。commit message 说明接口语义。
10. **通用默认值**：所有属性默认值为中性值（0/false/none）。K230 machine 必须显式设置全部 K230 值。commit message 注明值来源。

---

## 验证步骤

### 1. 编译（每组属性后）

```bash
ninja -C build qemu-system-riscv64
ninja -C build tests/qtest/k230-dw-ssi-test
```

### 2. qtest 回归（每组属性后）

```bash
QTEST_QEMU_BINARY=build/qemu-system-riscv64 build/tests/qtest/k230-dw-ssi-test -v
```

主成功标准：退出码 0，所有用例 PASS。

### 3. 属性值核对

```bash
rg -n 'qdev_prop_set' hw/riscv/k230.c
```

三实例应设置：
- `component-id`: 0xa1b2c3d5 / 0xa1b2c3d5 / 0xa1b2c3d5
- `version-id`: 0x3130332a / 0x3130332a / 0x3130332a
- `spi-ctrlr0-reset`: 0x04000200 / 0x04000200 / 0x28000200
- `fifo-depth`: 256 / 256 / 256
- `xip-window-size`:（仅 dw_ssi[2]）0x08000000
- `has-xip`: false / false / true
- `has-enhanced-spi`: true / true / true
- `dma-mode`: internal / internal / internal

### 4. 复位值验证（qtest 追加）

用 qtest 读 R_IDR、R_SSIC_VERSION_ID、R_SPI_CTRLR0 寄存器，验证与 machine 设置值一致。

### 5. XIP region 验证

```bash
# 确认只有 dw_ssi[2] 有第二个 sysbus MMIO region
rg -n 'sysbus_mmio_map.*dw_ssi.*1,' hw/riscv/k230.c
```

### 6. 硬编码残留检查

```bash
rg -n 'DW_SSI_IDR_RESET|DW_SSI_VERSION|DW_SSI_SPI_CTRLR0' hw/ssi/
rg -n '\bDW_SSI_FIFO_CAPACITY\b' hw/ssi/ include/hw/ssi/
rg -n '\bDW_SSI_XIP_WINDOW_SIZE\b' hw/ssi/ include/hw/ssi/
```

### 7. 代码规范

```bash
git diff --check
scripts/checkpatch.pl -f hw/ssi/dw_ssi.c
scripts/checkpatch.pl -f include/hw/ssi/dw_ssi.h
scripts/checkpatch.pl -f hw/riscv/k230.c
```

---

## 实施清单

### Step 4A（安全参数化，不改寄存器可见性）

1. **复位值属性**：新增 `component-id`/`version-id`/`spi-ctrlr0-reset` 三个 uint32 属性，默认值 0；reset handler 改读属性；machine 显式设置全部 K230 值
2. **FIFO 深度属性**：新增 `fifo-depth` uint32 属性；`fifo32_create()` 从 instance_init 移到 realize；增加 8..256 范围校验；修复 TXFTLR/RXFTLR 阈值校验
3. **XIP window size**：新增 `xip-window-size` uint64 属性，默认值 0；XIP region 创建从 instance_init 移到 realize，`xip_window_size > 0` 时创建；machine 设置 dw_ssi[2] 为 0x08000000

### Step 4B（capability 字段级门控）

4. **寄存器/字段 capability 矩阵**：先建表（实施前），列出每个寄存器字段归属于 enhanced/DMA/XIP/shared
5. **has-enhanced-spi**：新增 bool 属性，默认 false；`dw_ssi_ctrlr0_writable_mask()`/`dw_ssi_spi_ctrlr0_writable_mask()` 字段级 mask；传输分发门控
6. **dma-mode 枚举**：新增 `DwSsiDmaMode` 枚举（none/external/internal）；`dw_ssi_is_idma_reg()` 完整寄存器清单（含 AXIECR/DONECR）；IMR/ISR/RISR 字段 mask；`dw_ssi_irq_valid_mask()`；SR.CMPLTD_DF 清零；IRQ 输出低电平
7. **has-xip**：新增 bool 属性，默认 false；`dw_ssi_is_xip_only_reg()` 包含 `XIP_MODE_BITS/XIP_INCR_INST/XIP_WRAP_INST`；XIP engine 行为门控

每组完成后编译 + qtest 回归。

---

## 与决策文档的关系

本文档是 V2 实施路线第四步的执行细化。属性清单、capability 门控规则以 [V2 决策记录](k230-spi-qspi-review-v2-decision-notes.md) 为准。如果实施中发现边界问题，先更新决策文档，再回头调整本文档。
