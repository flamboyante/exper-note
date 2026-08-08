# 计划：编写 K230 V2 第四步实施文档

## Summary

为 V2 第四步"引入实例配置（属性化）"编写实施细化文档，风格与第三步文档一致，讲解直接融入文档。文档产出位置：`exper-note/k230/upstream-review/k230-spi-qspi-v2-step4-instance-properties.md`。

## Phase 1 探索结论

### 已有 QOM 属性（无需再做）
- `num-cs`（dw_ssi.c:1723，DEFINE_PROP_UINT32，三实例 5/5/1）
- `max-lines`（dw_ssi.c:1724，DEFINE_PROP_UINT32，三实例 4/4/8）

### 待属性化的 8 个硬编码（决策文档 line 228-236 确认）

| # | 属性名 | 类型 | 当前位置 | 当前值 | 三实例差异 |
|---|--------|------|----------|--------|-----------|
| 1 | `component-id` | uint32 | dw_ssi.c:32,1581 | `0xa1b2c3d5` | 无 |
| 2 | `version-id` | uint32 | dw_ssi.c:37,1582 | `0x3130332a` | 无 |
| 3 | `spi-ctrlr0-reset` | uint32 | dw_ssi.c:33,34,1583-1585 | `0x04000200`(QSPI) / `0x28000200`(OPI) | 有，当前由 max_lines==8 二选一 |
| 4 | `fifo-depth` | uint32 | dw_ssi.c:27 | 256 | 无 |
| 5 | `xip-window-size` | uint64 | dw_ssi.h:28 | `0x08000000` | 无（但前两实例浪费） |
| 6 | `has-enhanced-spi` | bool | 隐式：max_lines+SPI_FRF | — | 无（全 true） |
| 7 | `has-idma` | bool | 隐式：始终可用 | — | 无（全 true） |
| 8 | `has-xip` | bool | 隐式：GPIO+始终创建region | — | 有（仅 SPI-OPI true） |

### 决策文档的关键规则
- capability 关闭时寄存器 RAZ/WI（line 238, 362-368）
- `has-xip=false` 不创建 XIP region（line 322-323, 367）
- `spi-ctrlr0-reset` 是单一 uint32，替代 max_lines==8 条件（line 235）
- 用辅助函数统一判断寄存器组可见性（line 371）
- 可选 IRQ 在 capability 关闭时保持低电平（line 375）

### 实施路线建议的顺序（section 3.4）
1. component-id / version-id / spi-ctrlr0-reset（纯复位值，最安全）
2. fifo-depth
3. num-cs（已完成）
4. max-lines（已完成）
5. has-enhanced-spi / has-idma / has-xip（capability 门控，风险最高）
6. xip-window-size（只有 has-xip=true 时生效）

## Phase 3 文档结构计划

文档将写入 `exper-note/k230/upstream-review/k230-spi-qspi-v2-step4-instance-properties.md`，包含以下章节：

### 1. 摘要
把 SSI 的硬编码实例配置改为 QOM 属性，使通用模型可被任意 SoC 配置。

### 2. 当前状态分析
- 硬编码清单（上述 8 项表格，带文件:行号）
- 已有属性（num-cs, max-lines）
- 三实例配置差异表（来自搜索报告）
- 当前 capability 的隐式判断机制（enhanced-spi 由 max_lines 推导、idma 始终可用、xip 由 GPIO+region 始终创建）

### 3. 提议改动（按风险分 4 组，讲解融入）

**改动 1：复位值属性**（component-id, version-id, spi-ctrlr0-reset）
- 讲解：QOM property 机制（DEFINE_PROP_* 宏、device_class_set_props、realize 前可设置）
- 讲解：spi-ctrlr0-reset 为何从二选一改为单一属性（解耦复位值与 max_lines 语义）
- 代码：属性数组新增 3 项、reset handler 改读属性、machine 设置三实例值

**改动 2：fifo-depth**
- 讲解：FIFO 容量影响 SR.TFNF/SR.RFF 的计算
- 代码：宏改属性、4 个使用点改读属性、测试文件 K230_SSI_FIFO_DEPTH 对齐

**改动 3：XIP 属性**（has-xip, xip-window-size）
- 讲解：has-xip 门控 region 创建（不只是读返回 0）——解释为何 QSPI0/1 不应创建 128MB region
- 讲解：xip-window-size 仅 has-xip=true 时生效
- 讲解：has-xip=false 时 XIP 寄存器 RAZ/WI 的实现
- 代码：属性新增、realize 中条件创建 region、read/write 中 RAZ/WI 门控、machine 设置值

**改动 4：Capability 布尔**（has-enhanced-spi, has-idma）
- 讲解：capability 门控的 RAZ/WI 模式（QEMU 标准做法）
- 讲解：辅助函数 dw_ssi_reg_is_visible() 统一判断
- 讲解：capability 关闭时可选 IRQ 保持低电平
- 代码：属性新增、辅助函数、read/write 门控、transfer 分发门控、machine 设置值

### 4. "通用化"设计原则
- 属性名是功能名（fifo-depth）而非来源名（k230-fifo-depth）
- 默认值是 DWC SSI 的典型值，不是 K230 特定值
- capability 关闭时安全降级（RAZ/WI + IRQ 低电平 + 不创建 region）
- 属性设置时机：instance_init 后、realize 前（machine 中设置）

### 5. 假设与决策
- 三实例 capability 取值（保守假设：全 true 除 has-xip 区分）
- spi-ctrlr0-reset 替代 max_lines==8 条件
- has-xip=false 不创建 region（行为变更，QSPI0/1 不再有第二个 sysbus MMIO）
- VMState 不需要新增字段（属性值不随迁移变化，由 machine 重新设置）

### 6. 验证步骤
- 每组属性后 ninja + qtest
- 属性值核对（三实例 vs TRM/当前代码）
- capability 门控验证（has-xip=false 时读 XIP 寄存器返回 0）
- rg 残留检查（硬编码宏无残留）

### 7. 实施清单
收敛为按顺序的 N 个动作。

## Assumptions & Decisions

1. **文档位置**：exper-note/k230/upstream-review/（git 可同步，与 step3 同目录）
2. **文件名**：`k230-spi-qspi-v2-step4-instance-properties.md`（与 step3 命名风格一致）
3. **三实例 capability**：保守假设 has-enhanced-spi=true / has-idma=true（全实例），has-xip=false（QSPI0/1）/ true（SPI-OPI）。如用户有 TRM 数据可修正。
4. **讲解融入方式**：在相关改动小节内嵌"讲解"子段，解释 QOM 机制/RAZ/WI/capability 门控等，不单独成章
5. **文档内链接**：相对链接（同目录直接文件名），git/邮件通用
6. **代码基线**：第三步已完成（dw_ssi.c/h 已重命名，GPIO 解耦已完成）

## Verification

文档完成后检查：
- 所有文件路径和行号基于实际代码（搜索报告已确认）
- 8 个属性全部覆盖
- 三实例配置值与搜索报告一致
- 讲解融入但不冗余
- 验证命令用 rg 不用 grep
