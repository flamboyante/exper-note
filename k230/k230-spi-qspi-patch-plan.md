# K230 SPI/QSPI QEMU 上游 Patch 实施计划

> **For agentic workers:** 实施本文时按 Patch 顺序逐项执行；每个步骤完成后复核边界，再进入下一项。
>
> **执行说明：** 本文是 K230 SPI/QSPI QEMU 模型唯一的 Patch 规划与完成状态来源。所有实现、测试、限制和后续新增 Patch 都在本文维护；其他学习文档只解释 TRM/SDK 依据，不重复维护完成状态。

**目标：** 按 QEMU 上游可评审的能力边界，实现 K230 三个 SSI 控制器的寄存器契约、标准 SPI、Dual/Quad QSPI、九路中断、SPI NOR、HI_SYS `SSI_CTRL` 和 spi0 XIP。

**架构：** 控制器内部只处理寄存器、FIFO、事务和事件；K230 machine 负责实例、物理地址、PLIC 和板级 Flash 接线；HI_SYS 包装寄存器独立建模。每个 Patch 独立编译，功能实现与对应 qtest 最终位于同一 Patch。

**技术栈：** QEMU QOM、SysBusDevice、MemoryRegion、SSIBus、Fifo32、PLIC、qtest、VMState、K230 TRM 5.3/12.3 与 K230 SDK。

---

## 使用与勾选规则

- `[ ]` 表示尚未达到该项完成条件，`[x]` 表示已经实现并验证。
- 当前已有原型代码不等于 Patch 完成；必须按本文边界重新核对后才能勾选。
- 当前阶段允许暂缓 qtest，但不能因此勾选 Patch 总完成状态。
- 除尚未接入 machine 的 Patch 1 外，每个 Patch 的“功能编码”“qtest”“构建/检查”分别勾选，三者完成后才能勾选总表中的“完成”；Patch 1 的 MMIO 契约由 Patch 2 首次实例化后统一验证。
- 不在实现过程中顺手修改无关设备，不把重构、功能、测试和文档混入同一 Patch。
- 不在本文记录 Git commit/push 操作结果，除非用户明确要求执行。
- 新增 Patch 时使用文末模板追加，不在其他文档创建第二份状态表。

## 本轮支持边界

| 能力 | 本轮裁决 | 实现方式或限制 |
|---|---|---|
| 标准 SPI | 实现 | 通过 Fifo32、DR 别名和 SSIBus 建模软件可见事务 |
| Dual/Quad QSPI | 实现 | 模拟命令阶段和结果，不模拟 2/4 根物理线逐周期电平 |
| Octal/OPI | 不实现 | `SPI_FRF=Octal` 启动时记录 guest error并拒绝事务 |
| DDR/DTR/RXDS | 不实现行为 | 寄存器按裁决保存/读回，不产生精确采样和时序副作用 |
| 内部 AXI DMA | 不实现搬运 | 保留 MMIO 兼容层；`IDMAE` 不启动搬运，DONE/AXIE 不产生 |
| concurrent XIP | 不实现 | `0x108–0x11c` 按当前 profile RAZ/WI |
| XIP write | 不实现 | `0x140–0x148` RAZ/WI |
| spi0 读取 XIP | 实现 | `0xC0000000–0xC7FFFFFF`，受 `SSI_CTRL.ssi0_xip_en` 门控 |
| 冷启动 functional test | 后续系列 | 依赖 BootROM、DDRC 和可用镜像链路，不阻塞主系列 |

## K230 实例与中断映射

| 逻辑实例 | 控制器地址 | Profile | 最大线宽 | PLIC IRQ |
|---|---:|---|---:|---:|
| spi0 | `0x91584000` | FMC/XIP | 8 | 146–154 |
| spi1 | `0x91582000` | QSPI | 4 | 155–163 |
| spi2 | `0x91583000` | QSPI | 4 | 164–172 |

每个实例的外部中断顺序固定为：

```text
TXE, TXO, RXF, RXO, TXU, RXU, MST, DONE, AXIE
```

寄存器 bit 位置与外部 IRQ 下标不同，不能直接以 RISR bit 号作为输出数组下标。

## Patch 依赖关系

```text
Patch 1  寄存器模型
  └─ Patch 2  machine 实例化
       └─ Patch 3  Fifo32 与标准 PIO
            ├─ Patch 4  控制器九路中断
            │    └─ Patch 5  PLIC 接线
            └─ Patch 6  SPI NOR
                 └─ Patch 7  Dual/Quad QSPI
                      ├─ Patch 8  HI_SYS SSI_CTRL
                      │    └─ Patch 9  XIP window
                      └─ Patch 10 上游英文文档
```

## 主系列总进度

| Patch | 建议标题 | 功能编码 | qtest | 构建/检查 | 完成 | 可行性 | 核心局限 |
|---:|---|:---:|:---:|:---:|:---:|---|---|
| 1 | `hw/ssi: Add a K230 DesignWare SSI register model` | [ ] | 不适用 | [ ] | [ ] | 高 | 只建立访问契约，不形成完整 FIFO/IRQ/XIP 行为 |
| 2 | `hw/riscv/k230: Instantiate the K230 SSI controllers` | [ ] | [ ] | [ ] | [ ] | 高 | 不接 IRQ、Flash 和 XIP window |
| 3 | `hw/ssi: Implement K230 SSI FIFO and standard PIO transfers` | [ ] | [ ] | [ ] | [ ] | 高 | 只实现 Standard SPI；错误锁存留给 Patch 4 |
| 4 | `hw/ssi: Implement K230 SSI interrupt handling` | [ ] | [ ] | [ ] | [ ] | 高 | DONE/AXIE 输出存在但保持低电平 |
| 5 | `hw/riscv/k230: Connect K230 SSI interrupts to the PLIC` | [ ] | [ ] | [ ] | [ ] | 高 | 只负责接线，不修改事件算法 |
| 6 | `hw/riscv/k230: Attach SPI NOR flash to K230 spi0` | [ ] | [ ] | [ ] | [ ] | 高 | 仅 spi0 CS0 接板级 Flash |
| 7 | `hw/ssi: Implement K230 enhanced QSPI transfers` | [ ] | [ ] | [ ] | [ ] | 中 | 不实现 Octal、DDR/DTR、RXDS 和物理多线时序 |
| 8 | `hw/misc: Model the K230 HI_SYS SSI control register` | [ ] | [ ] | [ ] | [ ] | 中 | RXDS 包装位只保存/读回，不产生线路行为 |
| 9 | `hw/ssi: Add the K230 SPI flash XIP window` | [ ] | [ ] | [ ] | [ ] | 中 | 只读 XIP，不实现 prefetch、continuous transfer 和 XIP write |
| 10 | `docs: Document K230 SPI and QSPI support` | [ ] | [ ] | [ ] | [ ] | 高 | 只能声明前九个 Patch 已验证的能力 |

---

## Patch 1：寄存器模型与访问契约

### 文件范围

- `hw/ssi/k230_dw_ssi.c`
- `include/hw/ssi/k230_dw_ssi.h`
- `hw/ssi/Kconfig`
- `hw/ssi/meson.build`

### 实现可行性

高。TRM/SDK 已能确定寄存器地址、主要 writable mask、固定只读值、复位值和条件地址。该 Patch 不依赖 SPI NOR、PLIC 或复杂事务状态机。

### 已知局限

- 不实现完整 FIFO 数据路径。
- 不建立临时单 IRQ 接口作为最终方案。
- 不映射 XIP window。
- DMA 只提供寄存器兼容性，不执行 guest memory 搬运。
- 若迁移格式尚未稳定，只建立当前字段需要的最小 VMState，不承诺未上游版本间兼容。

### 功能编码清单

- [ ] MMIO 大小为 4 KiB，已裁决寄存器覆盖 `0x000–0x148`。
- [ ] `K230_DW_SSI_REGS_SIZE` 等于最后一个寄存器末尾 `0x14c`。
- [ ] 普通 R/W 寄存器写入全部经过 writable mask。
- [ ] reset 只写入已确认的合法复位值，保留位保持 0。
- [ ] 固定只读 `IDR=0xa1b2c3d5`、`SSIC_VERSION_ID=0x3130332a`。
- [ ] spi0 使用 `SPI_CTRLR0=0x28000200`，spi1/spi2 使用 `0x04000200`。
- [ ] `IMR` 复位为 `0x3f`，有效 mask 为 `0x9bf`。
- [ ] `AXIAWLEN/AXIARLEN` 复位为 `0x700`。
- [ ] `0x108–0x11c`、`0x138–0x13c`、`0x140–0x148` 按 RAZ/WI。
- [ ] `DMACR/AXIAWLEN/AXIARLEN/SPIDR/SPIAR/AXIAR0/1` 保存并读回掩码值。
- [ ] `AXIECR/DONECR` 读取为 0、写入忽略。
- [ ] 需要禁用期配置的寄存器在 `SSIENR=1` 时拒绝写入。
- [ ] `SSIENR` 仅在 `0→1`、`1→0` 状态转换时产生副作用。
- [ ] `SSIENR 1→0` 撤销 CS、清空 TX/RX FIFO并终止当前状态。
- [ ] `SER` 只保存下一事务片选集合，不在写寄存器时立即切 CS。
- [ ] 多个 SER 位有效时不静默选择最低位。
- [ ] MMIO 范围外和非法访问不会污染 `regs[]`。
- [ ] VMState 只保存本 Patch 已存在的状态，并设置明确版本。

### qtest 归属

本 Patch 尚未接入 machine，不创建临时测试机或虚假 MMIO 入口。寄存器访问契约在 Patch 2 实例化三个控制器后统一验证。

### 完成条件

- [ ] `ninja -C build qemu-system-riscv64` 通过。
- [ ] `git diff --check` 对本 Patch 文件无新增问题。
- [ ] 代码不携带 Patch 3/4/9 才需要的最终 FIFO、九路 IRQ 或 XIP 行为。

---

## Patch 2：三个 SSI 实例与地址映射

### 文件范围

- `hw/riscv/k230.c`
- `include/hw/riscv/k230.h`
- `hw/riscv/Kconfig`
- `tests/qtest/k230-dw-ssi-test.c`
- `tests/qtest/meson.build`

### 实现可行性

高。地址和 profile 已由 TRM 地址图、RT-Smart 与设备树共同确认。

### 已知局限

- 本 Patch 不连接 PLIC。
- 不接 `w25q256` 和 MTD。
- 不映射 `0xC0000000` XIP window。
- 不创建临时聚合 IRQ。

### 功能编码清单

- [ ] SoC 状态中按逻辑顺序组织 `spi0/spi1/spi2`，避免按地址顺序造成编号混乱。
- [ ] spi0 映射到 `0x91584000`，设置 FMC/XIP profile 和最大线宽 8。
- [ ] spi1 映射到 `0x91582000`，设置 QSPI profile 和最大线宽 4。
- [ ] spi2 映射到 `0x91583000`，设置 QSPI profile 和最大线宽 4。
- [ ] 三个实例的 `num-cs` 属性来自当前 board/profile 选择。
- [ ] machine 只完成 QOM child、realize 和 MMIO map。
- [ ] `0x91585000` HI_SYS 仍由占位设备覆盖，等待 Patch 8。

### qtest 清单（当前可暂缓）

- [ ] `reset_values`
- [ ] `profile_reset_values`
- [ ] `register_write_masks`
- [ ] `read_only_and_reserved_registers`
- [ ] `disabled_register_write`
- [ ] `ssienr_state_transitions`
- [ ] `dr2_is_not_ssi_ctrl`
- [ ] 三个 MMIO 基址可访问且实例状态隔离。
- [ ] 当前阶段没有误映射 PLIC、Flash 或 XIP window。

### 完成条件

- [ ] machine 启动不产生地址重叠。
- [ ] `qemu-system-riscv64` 构建通过。
- [ ] Patch 只包含实例化和基础可达性验证。

---

## Patch 3：Fifo32 与标准 SPI PIO

### 文件范围

- `hw/ssi/k230_dw_ssi.c`
- `include/hw/ssi/k230_dw_ssi.h`
- `tests/qtest/k230-dw-ssi-pio-test.c`
- 公共 qtest helper 文件（若已在 Patch 2 创建）

### 实现可行性

高。QEMU 已提供 `Fifo32`、`VMSTATE_FIFO32` 和 `SSIBus`。实现难点主要是区分“FIFO 项”与“字节”，并让 `DFS/TMOD` 真正进入传输路径。

### 已知局限

- 只实现 Standard SPI，即 `SPI_FRF=0`。
- TXO/RXO/RXU/TXU 的完整锁存和九路输出留给 Patch 4。
- 不实现 Dual/Quad、XIP、DMA、DDR/DTR。
- `SSIBus` 是 word 级抽象，不模拟串行时钟边沿。

### 功能编码清单

- [ ] 头文件改用 `#include "qemu/fifo32.h"`。
- [ ] TX/RX FIFO 类型改为 `Fifo32`。
- [ ] FIFO 容量固定为 256 项，而不是 256 字节。
- [ ] create/reset/destroy/push/pop/used/full/empty 全部改用 fifo32 API。
- [ ] VMState 使用 `VMSTATE_FIFO32`。
- [ ] 增加 `k230_dw_ssi_frame_mask()`，按 `DFS+1` 生成 4–32 位帧 mask。
- [ ] `DR0–DR35` 全部映射到同一 FIFO 数据口。
- [ ] DR 写入先压入 TX FIFO，不再直接等同于一次 `ssi_transfer()`。
- [ ] DR 读取从 RX FIFO pop 并右对齐。
- [ ] 建立统一 `k230_dw_ssi_run_transfer()` 或等价 transfer pump。
- [ ] transfer pump 检查 `SSIENR`、SER、Standard SPI profile 和 FIFO 条件。
- [ ] `TMOD=TX_AND_RX` 从 TX FIFO取帧并保存 RX。
- [ ] `TMOD=TX_ONLY` 从 TX FIFO取帧并丢弃 RX。
- [ ] `TMOD=RX_ONLY` 使用 dummy 帧并接收 `NDF+1` 帧。
- [ ] `TMOD=EEPROM_READ` 明确区分发送命令阶段和接收阶段。
- [ ] 增加 `busy` 或等价事务活动状态。
- [ ] `SR.BUSY/TFE/TFNF/RFNE/RFF` 从状态和 FIFO 动态计算。
- [ ] `TXFLR/RXFLR` 返回 FIFO 当前帧数，允许表示 0–256。
- [ ] `SSIENR 1→0` 将 busy、事务阶段、剩余帧数和 FIFO 统一清理。
- [ ] 配置期间写 `SER` 不立即切换正在活动的 CS。

### qtest 清单（当前可暂缓）

- [ ] `dr_aliases`
- [ ] `fifo_depth_256`
- [ ] `dfs_frame_mask`
- [ ] `tmod_tx_and_rx`
- [ ] `tmod_tx_only`
- [ ] `tmod_rx_only`
- [ ] `tmod_eeprom_read`
- [ ] `disable_clears_fifo`
- [ ] `dynamic_status`

### 完成条件

- [ ] TX FIFO 真正参与发送路径，不能长期保持 0。
- [ ] 所有 DR 地址共享同一对 FIFO。
- [ ] 标准 SPI PIO 不再固定只发送低 8 位。
- [ ] `qemu-system-riscv64` 构建通过。

---

## Patch 4：控制器内部九路中断

### 文件范围

- `hw/ssi/k230_dw_ssi.c`
- `include/hw/ssi/k230_dw_ssi.h`
- `tests/qtest/k230-dw-ssi-irq-test.c`

### 实现可行性

高。TRM 已给出 RISR/IMR/ISR 和 RC 寄存器语义，SDK 已确认九路外部顺序。

### 已知局限

- 当前无 DMA，DONE/AXIE 线路存在但始终保持低电平。
- 当前 profile 不产生 XRXO 和 SPITE。
- QEMU 单 Master 模型可能没有可信 MST 产生源，MST 可保留锁存接口但不伪造事件。

### 功能编码清单

- [ ] 用九元素 `qemu_irq` 数组替换单个聚合 IRQ。
- [ ] 输出顺序为 TXE/TXO/RXF/RXO/TXU/RXU/MST/DONE/AXIE。
- [ ] TXE、RXF 根据 FIFO 水位动态计算。
- [ ] TXO、RXO、TXU、RXU、MST 使用锁存状态。
- [ ] `RISR` 返回水位条件与锁存事件组合，不经过 IMR。
- [ ] `ISR = RISR & IMR & 0x9bf`。
- [ ] 写 IMR 后立即重新计算九路输出。
- [ ] TXEICR 读取清除 TXO/TXU。
- [ ] RXOICR、RXUICR、MSTICR 分别读取清除对应事件。
- [ ] ICR 读取清除 TXO/RXU/RXO/MST，不清 TXU。
- [ ] AXIECR/DONECR 当前读取为 0，写入忽略。
- [ ] reset 清锁存事件，post-load 从 FIFO/IMR/锁存状态重算输出。
- [ ] 外部 IRQ 使用 level，不使用 pulse。

### qtest 清单（当前可暂缓）

- [ ] `interrupt_watermark`
- [ ] `interrupt_mask`
- [ ] `interrupt_error_latch`
- [ ] `interrupt_read_clear`
- [ ] `interrupt_output_order`
- [ ] `dma_interrupts_inactive`

### 完成条件

- [ ] 控制器内部不包含任何 PLIC 中断号。
- [ ] 九路输出与 SDK 顺序一致。
- [ ] 读 ISR/RISR 不错误清除事件。

---

## Patch 5：九路 IRQ 接入 PLIC

### 文件范围

- `hw/riscv/k230.c`
- `include/hw/riscv/k230.h`
- `tests/qtest/k230-dw-ssi-irq-test.c`

### 实现可行性

高。K230 PLIC 支持 208 个源，最高 SSI IRQ 172 在有效范围内。

### 已知局限

- 只负责设备输出到 PLIC 的板级连接。
- 不修改 RISR/IMR/ISR 和错误事件算法。

### 功能编码清单

- [ ] 定义 spi0 IRQ 146–154。
- [ ] 定义 spi1 IRQ 155–163。
- [ ] 定义 spi2 IRQ 164–172。
- [ ] machine 以逻辑实例编号选择 IRQ base，不能按控制器地址递增推导。
- [ ] 每个实例九路输出逐一连接 PLIC。
- [ ] 不使用 OR gate 合并九路 IRQ。

### qtest 清单（当前可暂缓）

- [ ] `plic_routing`
- [ ] `plic_mask_and_clear`
- [ ] `plic_instance_isolation`
- [ ] 检查 spi0 RXU=151、spi1 RXU=160、spi2 RXU=169。

### 完成条件

- [ ] 三个实例共 27 路连接无重复、无越界、无编号错位。
- [ ] `qemu-system-riscv64` 构建通过。

---

## Patch 6：spi0 SPI NOR 与 MTD

### 文件范围

- `hw/riscv/k230.c`
- `tests/qtest/k230-dw-ssi-flash-test.c`
- qtest 公共 Flash 镜像 helper

### 实现可行性

高。QEMU `m25p80` 已支持 `w25q256` 和常用读命令，现有原型也已证明基础连接可行。

### 已知局限

- 只为 spi0 CS0 接入板级 Flash。
- spi1/spi2 不虚构从设备。
- 本 Patch 只验证标准 SPI 命令，不依赖 QSPI 阶段模型。

### 功能编码清单

- [ ] spi0 SSIBus 上实例化 `w25q256`。
- [ ] `-drive if=mtd` 连接 Flash backend。
- [ ] CS0 输出连接 Flash `SSI_GPIO_CS`。
- [ ] Flash helper 只负责 board 接线，不包含控制器事务逻辑。
- [ ] 没有 MTD 镜像时仍可创建空 Flash 设备。

### qtest 清单（当前可暂缓）

- [ ] `jedec_id`
- [ ] `flash_read_3byte_address`
- [ ] `flash_read_4byte_address`
- [ ] `chip_select_restarts_command`
- [ ] 测试结束清理临时 Flash 镜像。

### 完成条件

- [ ] 标准 SPI PIO 可以读取确定的 Flash 镜像内容。
- [ ] 不在控制器模型中硬编码 `w25q256`。

---

## Patch 7：Dual/Quad QSPI 阶段模型

### 文件范围

- `hw/ssi/k230_dw_ssi.c`
- `include/hw/ssi/k230_dw_ssi.h`
- `tests/qtest/k230-dw-ssi-pio-test.c`
- `tests/qtest/k230-dw-ssi-flash-test.c`

### 实现可行性

中。`m25p80` 能理解 Dual/Quad opcode，但 QEMU SSIBus 不表达真实多线电平，因此只能模拟软件可见命令阶段和结果。

### 已知局限

- 不模拟 2/4 根数据线的逐周期行为。
- 不实现 Octal/OPI。
- 不实现 DDR/DTR、RXDS 精确采样和 clock stretching 时序。
- unsupported 组合必须拒绝，不能静默降级成普通 SPI。

### 功能编码清单

- [ ] 事务阶段枚举包含 instruction/address/mode/dummy/data。
- [ ] `SPI_FRF` 支持 Standard/Dual/Quad。
- [ ] `TRANS_TYPE` 决定指令、地址阶段格式。
- [ ] `INST_L` 决定是否发送 0/4/8/16 bit 指令。
- [ ] `ADDR_L` 决定地址阶段长度。
- [ ] `XIP_MD_BIT_EN` 和 `XIP_MODE_BITS` 决定 mode bits 阶段。
- [ ] `WAIT_CYCLES` 进入 dummy 阶段，不再只按 opcode 硬编码。
- [ ] `NDF+1` 决定 RX-only/EEPROM read 数据帧数。
- [ ] `SPI_FRF=Octal` 记录 guest error并拒绝启动。
- [ ] DDR、DTR、RXDS 组合记录 unsupported，不产生虚假行为。
- [ ] CS 在完整命令阶段内保持有效，事务结束后撤销。

### qtest 清单（当前可暂缓）

- [ ] `qspi_trans_type`
- [ ] `qspi_instruction_and_address`
- [ ] `qspi_mode_bits_and_dummy`
- [ ] `qspi_rx_only_ndf`
- [ ] `qspi_flash_read`
- [ ] `octal_transfer_rejected`

### 完成条件

- [ ] K230 U-Boot 的 Dual/Quad 配置序列可以形成正确 Flash 命令。
- [ ] 文档和日志不声称完成物理多线时序建模。

---

## Patch 8：HI_SYS `SSI_CTRL`

### 文件范围

- 新增 `hw/misc/k230_hi_sys.c`
- 新增对应 `include/hw/misc/` 头文件
- `hw/misc/Kconfig`
- `hw/misc/meson.build`
- `hw/riscv/k230.c`
- `include/hw/riscv/k230.h`
- `tests/qtest/k230-dw-ssi-reg-test.c`

### 实现可行性

中。地址和位图已有 U-Boot/地址图证据，主要设计点是 HI_SYS 与三个 SSI 实例之间的状态接口。

### 已知局限

- RXDS 包装位只保存和读回。
- OPI/DTR 未实现时，RXDS 设置不产生串行线路副作用。
- 不允许 HI_SYS 直接访问 machine 全局变量。

### 功能编码清单

- [ ] HI_SYS MMIO 映射 `0x91585000–0x915853ff`。
- [ ] `SSI_CTRL` 只位于绝对地址 `0x91585068`。
- [ ] reset、implemented/writable mask 与 U-Boot 位图一致。
- [ ] `ssi0_xip_en` 保存并导出给 spi0。
- [ ] 三个实例 sleep/mode 状态动态返回。
- [ ] RXDS edge/delay 字段保存并读回。
- [ ] HI_SYS 与 SSI 使用明确 QOM link、GPIO 或窄接口连接。
- [ ] SPI `base+0x068` 始终继续作为 `DR2`。

### qtest 清单（当前可暂缓）

- [ ] `hi_sys_ssi_ctrl_reset`
- [ ] `hi_sys_ssi_ctrl_mask`
- [ ] `ssi_ctrl_and_dr2_are_independent`
- [ ] `hi_sys_sleep_and_mode_status`

### 完成条件

- [ ] 移除或缩小原 `hi_sys_cfg` 占位设备，不能与新 MMIO 重叠。
- [ ] `0x91585068` 与三个 SPI 控制器内部地址完全隔离。

---

## Patch 9：spi0 只读 XIP window

### 文件范围

- `hw/ssi/k230_dw_ssi.c`
- `include/hw/ssi/k230_dw_ssi.h`
- `hw/riscv/k230.c`
- `tests/qtest/k230-dw-ssi-xip-test.c`

### 实现可行性

中。现有原型已经可以读取 Flash，但必须改成寄存器驱动，并由 `SSI_CTRL.ssi0_xip_en` 门控。

### 已知局限

- 只实现读取，不实现 XIP write。
- 不实现 prefetch 和 continuous transfer。
- 不实现 concurrent XIP。
- 不实现 Octal/DTR/RXDS。

### 功能编码清单

- [ ] 仅 spi0 暴露第二个 XIP MemoryRegion。
- [ ] XIP window 映射 `0xC0000000–0xC7ffffff`。
- [ ] `ssi0_xip_en=0` 时 window 不返回 Flash 数据。
- [ ] `ssi0_xip_en=1` 时允许建立读取事务。
- [ ] opcode 来自 `XIP_INCR_INST/XIP_WRAP_INST`。
- [ ] 指令使能、地址长度、mode bits、dummy 来自控制寄存器。
- [ ] 不继续只按 opcode 硬编码地址长度和 dummy。
- [ ] 支持 1/2/4/8 字节 MMIO read。
- [ ] PIO 和 XIP 共享同一 Flash/CS 时保持事务互斥。
- [ ] XIP write 记录 guest error并忽略。

### qtest 清单（当前可暂缓）

- [ ] `xip_enable_gate`
- [ ] `xip_read_widths`
- [ ] `xip_address_width`
- [ ] `xip_mode_bits_and_dummy`
- [ ] `pio_xip_chip_select_coordination`
- [ ] `xip_write_ignored`

### 完成条件

- [ ] 默认禁用 XIP，不再永久暴露可读 Flash window。
- [ ] XIP 命令由 K230 寄存器配置生成。

---

## Patch 10：上游英文文档

### 文件范围

- `docs/devel/k230-spi-qspi.rst`
- `docs/devel/index-internals.rst`
- `docs/system/riscv/k230.rst`

### 实现可行性

高。文档内容由前九个 Patch 的实际通过结果决定。

### 已知局限

- 不能声明未验证的启动能力。
- 不能把寄存器可读回写成对应物理行为已实现。
- 不复制整份 TRM 位表到上游文档。

### 文档清单

- [ ] 三个控制器地址、profile 和最大线宽。
- [ ] 九路 IRQ 范围和事件顺序。
- [ ] spi0 CS0 Flash 与 `-drive if=mtd`。
- [ ] `SSI_CTRL @ 0x91585068` 与 `DR2` 区别。
- [ ] XIP window 地址和 enable 条件。
- [ ] Standard/Dual/Quad 支持范围。
- [ ] 明确不支持 Octal/OPI、DTR/RXDS、DMA 搬运、concurrent XIP 和 XIP write。
- [ ] 明确 QSPI 是软件可见命令建模，不是物理多线电平仿真。
- [ ] 启动参数示例与 machine 实际行为一致。

### 验证清单

- [ ] 文档构建通过。
- [ ] 支持矩阵与 qtest 覆盖一致。
- [ ] 所有功能声明都能指向具体实现 Patch 和测试。

---

## 主系列之外的后续能力

| 后续系列 | 前置条件 | 主要难点 | 当前决策 |
|---|---|---|---|
| Octal/OPI | Patch 7 | 16 bit 指令、8 线语义和 Flash backend 支持 | 后续独立系列 |
| DDR/DTR/RXDS | OPI 基础 | 双边沿数据、采样延迟和 strobe 抽象 | 后续独立系列 |
| 内部 AXI DMA | Patch 3/4/7 | guest physical memory、AINC/ATW、DONE/AXIE | 后续独立系列 |
| concurrent XIP | Patch 9 | PIO/XIP 并发、prefetch 和 XRXO | 当前不实现 |
| XIP write | Patch 9 | 写命令、权限、数据一致性 | 当前不实现 |
| 冷启动 functional test | BootROM、DDRC、镜像链 | 跨子系统集成和稳定资产 | 后续平台系列 |

## 每个 Patch 的统一完成门槛

- [ ] 功能边界与本文一致，没有夹带后续能力。
- [ ] 修改文件与该 Patch 单一职责一致。
- [ ] `ninja -C build qemu-system-riscv64` 通过。
- [ ] 对应 qtest 新鲜通过；当前暂缓时不得勾选 Patch 总完成。
- [ ] `git diff --check` 无本 Patch 新增问题。
- [ ] `scripts/checkpatch.pl --strict <patch-file>` 无需修正的新增问题。
- [ ] commit title 使用英文 QEMU 风格。
- [ ] commit message 说明硬件依据、行为边界和测试结果。
- [ ] 文档中“已实现”与实际代码、测试保持一致。

## 新增 Patch 模板

以后新增能力时，将下面模板复制到本文末尾：

```markdown
## Patch N：能力名称

### 文件范围

- `exact/path/to/file`

### 实现可行性

说明已有 QEMU 抽象、TRM/SDK 证据和实现依赖。

### 已知局限

- 明确本 Patch 不实现的行为。

### 功能编码清单

- [ ] 可独立验证的具体实现项。

### qtest 清单

- [ ] 具体测试名称与观察结果。

### 完成条件

- [ ] 构建、测试、格式和文档条件。
```

## 变更记录

| 日期 | 变更 |
|---|---|
| 2026-07-14 | 建立独立 10-Patch 主系列追踪文档；DMA 搬运、OPI/DTR 和完整启动测试移入后续系列 |
