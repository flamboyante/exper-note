# 上游邮件沟通日志

本文记录训练营相关补丁与上游社区的关键邮件往来、已确认结论和后续行动。邮件正文只保留必要摘要，完整上下文以原始线程为准。

维护约定：

- 新进展按时间倒序追加。
- 明确区分上游反馈、当前判断和待办事项。
- 只记录会影响补丁设计、实现、评审或后续行动的信息。
- 不记录账号凭据、隐私数据等敏感信息。

## 2026-07-13：Chao Liu 补充 IOMUX 位级访问权限意见

### 线程信息

- 上游回复人：Chao Liu
- 回复时间：2026-07-13 13:48（UTC+8）
- 原始线程：[lore.kernel.org](https://lore.kernel.org/qemu-devel/20260710041845.67170-1-flamboyant.h.01@gmail.com/T/#u)
- 参考文档：[K230 Linux SDK IOMUX Registers](https://www.kendryte.com/k230_linux/en/main/app_develop_guide/driver/iomux.html#registers)
- Patchwork 状态：`new`
- 自动检查：截至记录时仍无检查结果

### 上游反馈

Chao Liu 指出，IOMUX 寄存器不能把所有 32 位都作为普通可写存储处理：

- bit 31：`DI`，只读，表示输入数据；
- bits 30–14：保留，只读；
- bits 13–0：配置字段，可读写；
- 实现时需要逐位遵循每个字段的读写权限。

### 对当前实现的影响

- 当前 `s->regs[offset >> 2] = value` 会错误地保存 bits 31–14 的写入值。
- v2 写路径需要应用可写位掩码；根据当前寄存器表，基础掩码为 `0x00003fff`。
- bit 31 对应真实引脚输入状态。当前模型没有物理 pin routing，因此其返回语义需要明确，至少不能被 guest 写入修改。
- 当前 qtest 使用 `0x12345678`、`0xa5a5a5a5` 并期待完整读回，这与位权限要求冲突，需要改为验证可写位更新、只读位保持不变。
- 参考文档说明 Main IOMUX 有 64 个 pin 寄存器，偏移范围为 `0x00..0xfc`。现有 `0x7fc` 末尾寄存器读写测试需要结合 TRM 重新确认，不能继续默认整个 `0x800` 窗口都是普通寄存器数组。

### 待办

- [ ] 回复 Chao Liu，确认 v2 会实现位级写掩码并补充权限测试。
- [ ] 将 bits 13–0 作为可写字段，禁止 guest 修改 bits 31–14。
- [ ] 为只读位、保留位和正常读改写增加 qtest。
- [ ] 结合 TRM 核实 `0x100..0x7ff` 区域的定义，再决定未定义偏移的读写行为。
- [x] 更新现有英文回复稿，使 writable mask 的说明与该评审意见直接对应。

## 2026-07-13：K230 IOMUX RFC 收到首轮评审

### 线程信息

- 提交时间：2026-07-10 12:18（UTC+8）
- 补丁主题：`[PATCH RFC 1/1] hw/riscv/k230: add IOMUX register block model`
- 提交人：Kangjie Huang
- 上游回复人：Alistair Francis
- 原始线程：[lore.kernel.org](https://lore.kernel.org/qemu-devel/20260710041845.67170-1-flamboyant.h.01@gmail.com/T/#u)
- Patchwork 状态：`new`
- 自动检查：截至记录时暂无检查结果

### 上游反馈

1. 补丁需要进一步 split（拆分）。建议分别提交：
   - IOMUX 设备模型；
   - K230 SoC 接线；
   - qtest 测试。
2. `k230_iomux_read()` 和 `k230_iomux_write()` 中对 `offset`、`offset + size` 及对齐的显式检查没有必要；访问大小和对齐已经由 `MemoryRegionOps.valid` 与 `MemoryRegionOps.impl` 约束。
3. 上游追问该设备是否始终只需要寄存器数组的读写存储语义，即是否不存在其他必须建模的功能或副作用。

### 当前判断

- 这是可执行的首轮评审，不是拒绝该建模方向。
- v2 应先调整补丁组织，并删除与 `MemoryRegionOps` 重复的访问检查。
- 对寄存器数组语义的回复需要说明当前建模边界：目标是满足 SDK U-Boot `pinctrl-single` 的 32 位读改写；物理引脚路由、pad 电气效果和独立 PMU IOMUX 暂不建模。
- 仍需确认是否有会影响现有启动路径的寄存器副作用，避免把“当前只观察到存储需求”表述成永久硬件结论。

### 待办

- [ ] 回复 Alistair Francis，解释当前采用寄存器存储模型的依据和边界。
- [ ] 将下一版拆为 model、SoC wiring、qtest 三个补丁。
- [ ] 删除读写回调中与 `MemoryRegionOps` 重复的范围和对齐检查。
- [ ] 重新运行 qtest 与 SDK U-Boot 启动冒烟测试后发送 v2。
