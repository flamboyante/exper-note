# K230 SPI/QSPI Patch 计划迁移说明

更新时间：2026-07-14

K230 SPI/QSPI 的 Patch 顺序、功能边界、qtest、当前差距和完成状态已统一迁移到：

- [K230 SPI/QSPI 10-Patch 实施与 qtest Study](k230-spi-qspi-flash-window-study-plan.md)

该 Study 是唯一实时实施入口。本文保留文件名只为兼容已有链接，不再维护：

- 第二份 Patch 总表；
- Patch 完成勾选；
- qtest 矩阵和运行结果；
- 当前实现差距；
- 新增 Patch 模板。

当前主系列顺序为：

```text
寄存器/QOM
  -> machine 实例
  -> Standard SPI/FIFO/TMOD
  -> Standard SPI NOR
  -> Dual/Quad QSPI
  -> 控制器内部 IRQ
  -> PLIC 接线
  -> HI_SYS SSI_CTRL
  -> XIP
  -> 上游文档
```

请勿在本文继续追加状态或清单，避免与 Study 再次产生漂移。
