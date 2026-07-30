# K230 SPI/QSPI V2 Step 4 Plan Final V1.2 上游 Review 边界修订 Handoff

记录日期：2026-07-30

> 本文是下一会话修改 Plan Final V1.2 的任务入口，不是新的架构决策文档。完成修订后，以更新后的 [Plan Final V1.2](k230-spi-qspi-v2-step4-plan-final-instance-configurationV1.2.md) 为唯一执行计划，并将本文标记为已完成。

## 1. 下一会话直接执行的任务

修订以下活动文档，使第一批上游 series 只提交具有当前消费者的 Standard SPI PIO 基线，不再预留恒为零的 enhanced SPI、IDMA 或 XIP capability 骨架：

1. `k230-spi-qspi-v2-step4-plan-final-instance-configurationV1.2.md`；
2. `k230-spi-qspi-review-v2-decision-notes.md`；
3. `k230-spi-qspi-v2-implementation-plan.md`；
4. `workspace-state.md`；
5. 原 Plan Final 顶部的历史替代提示。

只修改文档。不要修改 QEMU 源码，不创建分支，不提交，不推送。保留工作区已有的无关修改和未跟踪文件。

## 2. 下一会话的读取顺序

1. 工作区根目录 `AGENTS.md`；
2. `exper-note/workspace-state.md`；
3. 本 handoff；
4. `k230-spi-qspi-v2-step4-plan-final-instance-configurationV1.2.md`；
5. `k230-spi-qspi-review-v2-decision-notes.md`；
6. `k230-spi-qspi-v2-implementation-plan.md`。

修改前确认 `qemu-camp-2026-k230/.codegraph/` 是否存在。若存在，理解当前 `DwSsiState`、IRQ、DMA 寄存器和 K230 实例消费者时先使用 CodeGraph，不先扫描整个源码仓库。

## 3. 已确认的核心裁决

### 3.1 “骨架全留、肌肉后置”的精确定义

第一批保留的是当前可独立工作和测试的架构骨架：

- 单一通用 `TYPE_DW_SSI` / `DwSsiState`；
- 控制器 MMIO region 0；
- Standard single-line SPI PIO；
- TX/RX FIFO 和四种 Standard TMOD；
- Standard PIO 所需的基础 IRQ；
- SSI bus 和 CS outputs；
- reset、VMState、realize/finalize；
- K230 三实例、PLIC 路由和 Standard 1-1-1 SPI NOR 挂接；
- 独立通用 `dw-ssi-test` 测试机。

“骨架全留”不表示提前保留未来功能的配置字段、capability bit、helper、IRQ output、运行时状态或额外资源。enhanced SPI、DMA/IDMA 和 XIP 是后续功能 series 的行为与接口，应随各自第一个真实消费者一起引入。

### 3.2 第一批删除内部 capability 骨架

从 V1.2 第一批设计中删除：

- `DwSsiConfig.capabilities`；
- `DwSsiConfig.xip_window_size`；
- `DW_SSI_CAP_ENHANCED_SPI`、`DW_SSI_CAP_IDMA`、`DW_SSI_CAP_XIP`；
- `DW_SSI_CAP_VALID_MASK`；
- `dw_ssi_has_capability()`；
- 对恒零 capability 位图的 realize 校验；
- K230 profile 中的 capability 行；
- migration equality 或测试中对内部 capability 值的断言。

第一批仍不暴露以下 public property：

- `has-enhanced-spi`；
- `has-idma`；
- `has-xip`；
- `xip-window-size`。

这些 property 必须分别与 enhanced SPI、IDMA、XIP 的正路径、资源、IRQ、迁移约束和 qtest 在同一后续 series 引入。

### 3.3 主线先例结论

DesignWare I2C 比 DesignWare I3C 更接近第一批 DW SSI 的边界：

- `DesignWareI2CState` 没有 `cfg` capability 位图；
- FIFO 深度和 component 参数按当前实现固定；
- DMA 寄存器地址存在，但未实现功能通过 unsupported callback 表达；
- `DW_IC_COMP_PARAM_1` 直接报告 `HAS_DMA=0`；
- Atlantis 创建多个实例时不设置恒 false property 或 profile 字段。

DesignWare I3C 可以支持集中 `cfg`、property 驱动资源创建和 realize 校验，但其中每个配置字段都有当前 property 和消费者。I2C/I3C 均不能为“第一批保留无入口、无消费者、永远为零的内部 capability 位图”提供先例。

## 4. 第一批配置边界

采用投稿面最小的方案：第一批 `DwSsiConfig` 只保留具有当前行为消费者或真实实例差异的字段。目标结构优先收缩为：

```c
typedef struct DwSsiConfig {
    uint32_t num_cs;
    uint32_t fifo_depth;
    uint32_t imr_reset;
} DwSsiConfig;
```

字段处理规则如下：

| 当前 V1.2 字段 | 第一批处理 | 理由 |
|---|---|---|
| `num_cs` | 保留 property | 决定 CS output 数量，K230 三实例有真实差异 |
| `fifo_depth` | 保留 property | 决定 FIFO 资源和 guest-visible level |
| `imr_reset` | 保留 property | QSPI/FMC reset 有真实差异，基础 IRQ 消费该值 |
| `component_id` | 固定在通用模型 | 当前建模实例取值相同，没有 profile 差异 |
| `version_id` | 固定在通用模型 | 当前只建模一个已确认版本 |
| `max_lines` | 随 enhanced SPI series 后移 | Standard 1-1-1 数据路径不消费物理多线能力 |
| `spi_ctrlr0_reset` | 随 enhanced SPI series 后移 | 第一批不建立 enhanced 寄存器契约 |
| `dma_register_layout` | 随 DMA/IDMA series 后移 | 第一批没有 DMA request 或 DMA engine |
| `axiawlen_reset` / `axiarlen_reset` | 随 IDMA series 后移 | 第一批没有行为消费者 |
| `capabilities` | 删除 | 恒零且无入口、无消费者 |
| `xip_window_size` | 删除 | 无 region、property 或 transaction 消费者 |

修改时必须重新核对当前源码消费者。如果某字段确实被 Standard PIO、基础 IRQ或第一批资源生命周期使用，可以保留，但必须在 V1.2 中写出具体消费者；不能仅以“以后会需要”作为理由。

## 5. 寄存器、IRQ 与资源边界

### 5.1 未实现寄存器

第一批保留稳定的 region 0 aperture，但不承诺“完整 DW SSI 标准寄存器全组”。文档改写为：

- 已实现的 Standard PIO、FIFO、状态和基础 IRQ 寄存器具有真实 reset、mask 和副作用；
- 未实现的 enhanced、IDMA、XIP offset 统一 RAZ/WI 或采用明确的 unsupported 语义；
- 不为未实现寄存器增加 capability 分发表、未来状态字段或纯粹为测试存在的配置入口；
- 如果 firmware 枚举必须读取某个扩展寄存器 reset，单独给出证据、最小只读语义和测试，不把整个未来功能骨架带入第一批。

### 5.2 DMA 边界

第一批不以“寄存器纯存储”暗示 DMA 能力。`dma-register-layout`、external DMA 配置寄存器、K230 internal-AXI 寄存器、DONE/AXIE 行为均后移到 DMA/IDMA series。

如果下一会话通过当前 firmware 或 Linux 通用驱动路径证明 external `DMACR/DMATDLR/DMARDLR` 是 Standard PIO 第一批启动或枚举的必要兼容面，可以保留最小寄存器语义；此时必须：

1. 明确它们没有 DMA request 数据通路；
2. 不引入 IDMA capability；
3. 不连带引入 K230 internal-AXI 寄存器；
4. 在提交说明中单独解释兼容性消费者。

没有上述证据时按后移处理。

### 5.3 IRQ 边界

第一批只注册和实现 Standard PIO 基础 IRQ：

- TXE；
- TXO；
- RXF；
- RXO；
- TXU；
- RXU；
- MST。

TXU 是 TX FIFO underflow，属于第一批基线。DONE、AXIE 随 IDMA series 引入，不在第一批注册后恒低。XIP 相关 IRQ 随 XIP series 引入。

### 5.4 XIP 边界

第一批：

- 不创建 `xip-enable` GPIO；
- 不创建第二个 MMIO region；
- K230 不映射 `0xc0000000` XIP aperture；
- XIP-only offset RAZ/WI；
- 不保留 `xip_window_size` 或内部 XIP bit。

XIP property、GPIO、region、K230 映射、transaction 和迁移状态必须在同一 XIP series 引入。

## 6. V1.2 全文修改清单

不能只改 `DwSsiConfig` 代码片段。当前“内部 capability 全零”结论贯穿全文，必须同步清理：

1. 标题、修订说明、投稿边界说明和摘要；
2. §1 硬编码点与资源生命周期；
3. §2.3 K230 profile 和表后说明；
4. §2.4 QEMU 主线先例，增加 DW I2C 对照；
5. §2.5 已确认决策；
6. §3.1 配置结构、capability bit 和 helper；
7. §3.2 property 集合和默认值；
8. §3.3 realize 校验；
9. §4 layout/capability 双门控叙事；
10. §5 K230 profile 和 property helper；
11. §6 通用测试机定位；
12. §8 至 §10 的 Step 4.1/4.2/4.3 任务划分；
13. §11 migration equality；
14. §12 TDD 测试矩阵；
15. §14 实施清单；
16. §15 第一批上游 series 边界和 commit 顺序；
17. §16 与最终 V2 patch 系列的关系；
18. 其他所有包含 `capability`、`xip_window_size`、`max-lines`、DMA layout、DONE/AXIE 第一批语义的段落。

修改后执行定向全文搜索，确保没有旧结论残留。

## 7. 测试叙事改写

测试只验证 guest-visible ABI，不验证不存在的内部 implementation bit。删除或改名：

```text
/dw-ssi/capability/enhanced-off
/dw-ssi/capability/idma-off
/dw-ssi/capability/xip-off
/k230-dw-ssi/profile-capabilities
```

建议替换为：

```text
/dw-ssi/register/unsupported-enhanced
/dw-ssi/register/unsupported-dma
/dw-ssi/register/unsupported-xip
/k230-dw-ssi/no-xip-aperture
```

测试断言关注：

- Standard PIO 正常工作；
- 未实现 offset 的固定 RAZ/WI 或 unsupported 语义；
- 未实现数据路径不会访问 guest memory；
- 不存在未实现功能的 IRQ、GPIO 和 region；
- system reset 后 guest-visible 契约不变。

不要通过 QMP 或测试专用 property 暴露内部 capability 值。

## 8. 第一批建议提交顺序

投稿顺序调整为每个 commit 自己可用、可测试：

1. `hw/ssi: Add a Synopsys DesignWare SSI standard PIO controller`
   - 通用类型、Standard 寄存器、FIFO、四种 TMOD、PIO、reset、VMState 和通用 qtest；
2. `hw/ssi: Add DesignWare SSI standard interrupt support`
   - 七路基础 IRQ 及 qtest；
3. `hw/riscv/k230: Instantiate DesignWare SSI controllers`
   - K230 三实例、必要 profile、region 0 和实例测试；
4. `hw/riscv/k230: Route SSI interrupts to the PLIC`
   - 七路基础 IRQ 的 PLIC 接线和隔离测试；
5. `hw/riscv/k230: Attach a standard SPI flash to the K230 SSI`
   - Standard 1-1-1 NOR 挂接和读取测试。

不要让 K230 实例化 patch 依赖后续 commit 才获得基本 PIO 功能。测试跟随首次引入相应行为的 commit。

## 9. 不得改变的既有结论

- V2 是大版本号；`k230-spiv3.4` 是 V1 分支名，不是 V2。
- Step 4.0 已完成，不因文档收缩而回退源码。
- ordinary enhanced/IDMA transaction 为 `instruction -> address -> dummy -> data`。
- `XIP_MODE_BITS`、`XIP_INCR_INST`、`XIP_WRAP_INST` 属于 XIP-only，不得重新用于普通 enhanced 或 IDMA mode byte。
- TXU 属于基础 FIFO IRQ，不属于 IDMA。
- 通用 `dw_ssi` 不重新依赖 K230 HI_SYS、K230 地址或 K230 类型。
- 不增加 `TYPE_K230_DW_SSI` wrapper。
- K230 machine 负责实例化、地址映射、PLIC 接线和外围设备挂接。

## 10. 文档一致性与验证

修改完成后至少执行：

```bash
git -C exper-note diff --check
git -C exper-note status --short

rg -n 'capabilities|DW_SSI_CAP_|dw_ssi_has_capability|xip_window_size|内部 capability|全零位图' \
  exper-note/k230/upstream-review/k230-spi-qspi-v2-step4-plan-final-instance-configurationV1.2.md \
  exper-note/k230/upstream-review/k230-spi-qspi-review-v2-decision-notes.md \
  exper-note/k230/upstream-review/k230-spi-qspi-v2-implementation-plan.md \
  exper-note/workspace-state.md

rg -n 'has-enhanced-spi|has-idma|has-xip|xip-window-size' \
  exper-note/k230/upstream-review/k230-spi-qspi-v2-step4-plan-final-instance-configurationV1.2.md
```

第一组搜索在活动文档中应无第一批内部位图残留；第二组只允许出现在“后续 series 才引入”的边界说明中。

另需检查：

- Markdown 相对链接均存在；
- fenced code block 成对；
- V1.2 内没有互相矛盾的第一批字段表、测试表和 patch 归属；
- `workspace-state.md` 的“下一步”与修订后的 V1.2 一致；
- 本轮没有修改 `qemu-camp-2026-k230/` 下任何源码或测试。

## 11. 下一会话完成标准

- V1.2 不再包含内部全零 capability 位图或无消费者的 future-proofing 字段；
- 第一批每个 property、状态字段、IRQ output 和 helper 都能指出当前消费者；
- 未实现功能通过 guest-visible RAZ/WI、unsupported 或资源不存在表达，而不是通过不可配置的内部 capability；
- enhanced SPI、DMA/IDMA、XIP 的 property 与功能分别归入后续 series；
- 第一批 commit 顺序不存在“先实例化不可用设备、后补基本数据路径”的中间态；
- 所有活动入口同步为同一结论；
- 文档验证通过，未执行 commit 或 push。

