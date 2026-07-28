# K230 SPI v2 补丁集重构计划

## 总结

目标是基于 `f893c46c39` 重新生成 9 个可独立审查、构建和测试的补丁，最终设备行为与 `9a61b56a6f` 保持一致。

已确认：

- 当前最终生产代码与 `9a61b56a6f` 没有发现功能差异，主要差异为注释、格式和测试组织。
- 当前最终树可完整构建，现有 7 组 qtest 全部通过，51 个叶子测试场景均保留。
- 中间历史存在测试与实现冲突、语法错误、跨补丁修复、功能回退和未来脚手架提前引入。
- `hw/misc/k230_hi_sys.c` 中 TRM URL 返回 404。
- 测试保持单文件，最终调整为 9 个逻辑组，不恢复旧的本地彩色 runner。

## 补丁重构

1. `hw/ssi: Add K230 DesignWare SSI register model`
   - 引入完整寄存器布局、复位值、读写掩码、RAZ/WI、基础 QOM/VMState 和 Kconfig。
   - 从一开始采用最终 enabled-write 合约：仅锁定 `CTRLR0/1`、`MWCR`、`BAUDR`、`SPI_CTRLR0`。
   - 加入正确版权和可访问的 `/K230/en-us/...` TRM 链接。
   - 不包含 PIO、IRQ、QSPI、flash、HI_SYS 或 XIP 实现。

2. `hw/riscv/k230: Instantiate K230 SSI controllers`
   - 实例化三个控制器，设置 `num-cs/max-lines` 并映射寄存器 MMIO。
   - 引入单文件 qtest，仅包含当期需要的寄存器常量和辅助函数。
   - 注册 `/register-contract`；修正全局测试数据为 `static const`，清理连续空行和缩进。
   - 此补丁中的 enabled-write 测试必须立即与补丁 1 的实现一致。
   - 在 MAINTAINERS 中加入精确的测试文件路径。

3. `hw/ssi: Implement K230 SSI FIFO and standard PIO transfers`
   - 完整实现 FIFO、DR aliases、DFS 截断、SRL loopback、`ssi_transfer()` 和四种 TMOD。
   - `send_frame()` 在本补丁首次出现时就必须正确赋值 `rx`。
   - 使用规范缩进和明确的 fallthrough 注释，不留下空分支或空注释。
   - 增加 `/pio-data-path` 测试组。

4. `hw/ssi: Add K230 SSI interrupt controller`
   - 引入动态 TXE/RXF、锁存错误、中断清除寄存器和九路 GPIO。
   - 将 reset enter/exit、post-load 的 IRQ 重驱动全部放在本补丁。
   - 增加 `/interrupt-controller` 测试组。

5. `hw/riscv: Route K230 SSI IRQs to the PLIC`
   - 引入最终命名的通用 `K230SsiRoute`，避免补丁 8 再重命名。
   - 接通三个逻辑 SSI 的九路 PLIC source。
   - 增加 `/plic-routing` 测试组；此补丁不再修改 SSI 设备内部 reset 行为。

6. `hw/ssi: Implement K230 enhanced QSPI transfers`
   - 引入 enhanced command、phase、migration state 和 SDR Dual/Quad 状态机。
   - 保持补丁 3 的 standard PIO 代码不变，不夹带修复或格式重排。
   - 修正原提交中的 EEPROM_READ 多余大括号。
   - 增加 `/qspi-config`，只覆盖 unsupported/atomic-prefix 等控制器本地场景。

7. `hw/riscv/k230: Attach SPI NOR flash to spi0`
   - 只增加 `spi-flash` machine 属性、MTD backend 和 m25p80 CS0 接线。
   - 增加 flash image 辅助代码、`/spi-nor` 和成功路径 `/qspi-sdr`。
   - 不修改 enabled-write 合约，不删除 `send_frame()` 的赋值，不批量增删注释。

8. `hw/misc: Add K230 HI_SYS SSI control`
   - 引入 HI_SYS 设备、SSI mode/sleep 状态、machine 映射和共享路由表。
   - 加入正确 TRM URL；头文件通过前向声明引用 `K230DwSsiState`，实现文件显式包含 SSI 头，降低头文件耦合。
   - `get_spi_mode()`、`is_sleeping()` 和 `sleep_status` migration state 在此补丁引入。
   - 增加 `/hi-sys` 测试组；XIP 查询 helper 延后到补丁 9。

9. `hw/ssi: Add K230 SSI XIP read window`
   - 只加入 XIP MemoryRegion、HI_SYS XIP gate 查询、命令构造、flash window 映射和 XIP 测试。
   - 增加 `/xip-read-window`。
   - 不再修改版权、MAINTAINERS、PIO/QSPI 状态机、flash 属性或前序注释。

## 验证方案

- 在独立 worktree 中从 `f893c46c39` 重建系列，保留当前 `k230-spiv2` 不动。
- 对原提交执行 RED 验证：补丁 2 的 enabled-write 测试失败、补丁 3 的 PIO 数据错误、补丁 6 构建失败、补丁 7 再次破坏 `send_frame()`。
- 重写后从补丁 2 开始逐提交构建 `qemu-system-riscv64` 和 K230 qtest，并运行当期所有已注册测试组。
- 最终固定为 9 组：`register-contract`、`pio-data-path`、`interrupt-controller`、`plic-routing`、`qspi-config`、`spi-nor`、`qspi-sdr`、`hi-sys`、`xip-read-window`。
- 每个提交运行 `git diff --check` 和 QEMU `scripts/checkpatch.pl`。
- 搜索并禁止空注释、中文学习脚手架、`LEARNING(...)`、无关 TODO 和未来补丁未使用的测试定义。
- 最终分别构建并运行 `9a61b56a6f` 与重写后的 tip；生产代码差异只能是文档、注释、格式和头文件依赖收敛，51 个叶子场景必须全部被 9 组调用。
- 使用 `git range-diff` 和逐提交 stat 检查标题、说明与实际文件职责一致。

## 操作约束

- 不使用 `git reset --hard`，不直接覆盖当前分支。
- 新提交、替换 `k230-spiv2` 分支引用以及 force-push 分别执行前取得明确确认。
- 不新增功能，不改变 QOM machine 属性、寄存器语义、IRQ 编号、flash 型号选择或迁移状态的最终行为。
