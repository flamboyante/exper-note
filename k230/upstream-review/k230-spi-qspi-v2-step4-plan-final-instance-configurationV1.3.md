# K230 V2 第四步 Plan Final V1.3：Standard PIO 第一批范围

首次记录：2026-07-30

V1.3 修订：2026-07-30（按上游 review 面收缩第一批，不预留无消费者的 future capability）

适用代码检查点：`my-qemu-camp-2026-k230/` 分支 `k230-V2-patch-spi`，commit `c689ac865f`

文档状态：Step 4.0 已完成；Step 4.1 至 Step 4.4 待实施

本文取代 [Plan Final V1.2](k230-spi-qspi-v2-step4-plan-final-instance-configurationV1.2.md)，是 Step 4 的唯一执行计划。V1.2 及更早版本只保留为历史推演材料。

> 第一批上游 series 只实现具有当前消费者的 Standard SPI PIO 基线：通用控制器、FIFO、四种 Standard TMOD、七路基础 IRQ、K230 三实例、PLIC 路由和 Standard 1-1-1 SPI NOR。enhanced SPI、external DMA、internal DMA、DONE/AXIE、HI_SYS/XIP 接口和额外 MMIO 资源均随各自首个真实功能 series 后送。

---

## 摘要

V1.3 的裁剪原则是：**第一批只提交当前能够独立工作、独立测试的接口，不提前提交未来功能的配置骨架。**

第一批保留：

- 单一通用 `TYPE_DW_SSI` / `DwSsiState`；
- 控制器 MMIO region 0；
- Standard single-line SPI PIO；
- TX/RX FIFO 和四种 Standard TMOD；
- TXE、TXO、RXF、RXO、TXU、RXU、MST 七路基础 IRQ；
- SSI bus 和动态 CS outputs；
- reset、基础 VMState、realize/finalize；
- K230 QSPI0、QSPI1、SPI-OPI/FMC 三实例；
- K230 PLIC 路由；
- Standard 1-1-1 SPI NOR 挂接；
- 独立通用 `dw-ssi-test` 测试机。

第一批不保留：

- enhanced/IDMA/XIP 内部位图或判断 helper；
- `has-enhanced-spi`、`has-idma`、`has-xip`、`xip-window-size` property；
- `max-lines`、`spi-ctrlr0-reset`；
- DMA register layout、AXI burst reset；
- DONE/AXIE IRQ output；
- `xip-enable` GPIO、第二个 MMIO region 和 `0xc0000000` 映射；
- enhanced、IDMA、XIP 运行时状态和 VMState 字段。

未实现的 enhanced、DMA/IDMA、XIP offset 使用统一 RAZ/WI 或明确 unsupported 语义，不通过不可配置的内部状态表示“关闭”。

---

## 1. 当前检查点

### 1.1 已完成内容

Step 4.0 已在 `c689ac865f` 完成：

- ordinary enhanced/IDMA transaction 固定为 instruction → address → dummy → data；
- 普通事务不再读取 `XIP_MD_BIT_EN`、`XIP_MBL`、`XIP_MODE_BITS`；
- 删除普通 enhanced mode phase；
- 删除 IDMA 1-4-4 的 XIP mode-byte 特判；
- K230 SSI qtest 12/12 PASS。

Step 4.0 不因第一批范围收缩而回退。它是后续 enhanced/IDMA series 的正确行为基线，但对应功能和测试不进入第一批最终 patch。

### 1.2 当前代码需要收缩的资源

当前中间态仍具有第一批不需要的资源和状态：

- 固定 128 MiB XIP region；
- 9 路 IRQ；
- `xip-enable` GPIO；
- enhanced、IDMA、XIP 寄存器行为和运行时状态；
- K230 FMC `0xc0000000` XIP 映射。

重组第一批时必须删除这些接口，而不是让它们以恒低、恒零或无消费者形式保留。

---

## 2. 第一批证据与消费者边界

### 2.1 K230 实例编号

| 物理模块 | SDK 逻辑名 | QEMU `dw_ssi[]` 下标 | 控制器地址 |
|---|---|---:|---:|
| SPI-OPI/FMC | `spi0` | 2 | `0x91584000` |
| QSPI0 | `spi1` | 0 | `0x91582000` |
| QSPI1 | `spi2` | 1 | `0x91583000` |

代码、测试和文档不得用 SDK `spi0` 指代 `dw_ssi[0]`。

### 2.2 第一批 K230 profile

| 配置项 | QSPI0 | QSPI1 | SPI-OPI/FMC | 当前消费者 |
|---|---:|---:|---:|---|
| `num-cs` | 5 | 5 | 1 | CS output 数量和 `SER` mask |
| `fifo-depth` | 256 | 256 | 256 | FIFO 资源、level/status、threshold |
| `imr-reset` | `0x1f` | `0x1f` | `0x3f` | 基础 IRQ reset 契约 |
| region 0 | `0x91582000` | `0x91583000` | `0x91584000` | K230 MMIO 映射 |

以下已知硬件差异不进入第一批 profile，因为 Standard PIO 没有消费者：

- `max-lines=4/4/8`；
- `SPI_CTRLR0` reset；
- internal-AXI DMA 布局和 burst reset；
- FMC XIP window。

### 2.3 DMA 消费者核对

本轮定向检查了 K230 SDK U-Boot、Linux 和 RT-Smart：

- U-Boot 对 `DMACR`、`SPIDR`、`SPIAR`、AXI 地址寄存器的有效写入位于 IDMA memory-op 路径；
- Linux K230 驱动读取 `DMACR` 判断是否进入 DMA 分支，RAZ/WI 返回 0 时自然选择 PIO；写入和 DONE 清理属于 IDMA 路径；
- Linux debugfs 枚举 DMA offset 不是启动或 Standard PIO 的功能消费者；
- RT-Smart 的非零 `DMACR` 写入用于 internal DMA。

结论：第一批没有证据要求 DMA 寄存器提供存储 readback。`DMACR`、`0x050/0x054` 和 K230 internal-AXI 扩展统一后移；第一批读取返回 0、写入忽略。

### 2.4 QEMU 主线先例

第一批更接近 `hw/i2c/designware_i2c.c`：

- 只实现当前消费者需要的控制器能力；
- 未实现 DMA 寄存器使用 unsupported callback/语义；
- 不为没有消费者的能力创建内部位图或 public property；
- SoC 可以创建多个通用实例，而不设置一组恒 false 字段。

DesignWare I3C 的集中配置仅用于已有 property 和消费者，不能作为预留未来字段的依据。

---

## 3. 第一批通用配置

### 3.1 `DwSsiConfig`

第一批结构收缩为：

```c
typedef struct DwSsiConfig {
    uint32_t num_cs;
    uint32_t fifo_depth;
    uint32_t imr_reset;
} DwSsiConfig;
```

每个字段都有当前消费者：

| 字段 | property | 消费位置 |
|---|---|---|
| `num_cs` | `num-cs` | CS 数组、GPIO outputs、`SER` mask、active-CS 校验 |
| `fifo_depth` | `fifo-depth` | FIFO 创建、level/status、threshold、迁移解释 |
| `imr_reset` | `imr-reset` | `IMR` reset 和七路基础 IRQ mask |

IDR 和 `SSIC_VERSION_ID` 使用当前建模版本的通用常量。若以后出现第二种已确认 component/version，再以真实消费者为依据增加配置。

### 3.2 第一批 properties

```text
num-cs       default 1
fifo-depth   default 256
imr-reset    default 0x3f
```

不暴露任何只能取单一值、没有当前正路径的 property。

### 3.3 realize 校验

第一批只需要验证：

- `num-cs` 在 `1..8`；
- `fifo-depth` 在 `2..256`；
- `imr-reset` 不包含七路基础 IRQ 之外的位。

配置校验全部完成后再创建 CS 和 FIFO，避免 realize 失败留下部分资源。

### 3.4 资源生命周期

`instance_init`：

- 创建 SSI bus；
- 创建控制器 MMIO region 0；
- 注册七路基础 IRQ。

`realize`：

- 校验三项配置；
- 创建 `num-cs` 个 CS outputs；
- 创建 `fifo-depth` 深度的 TX/RX FIFO。

`finalize`：

- 销毁 FIFO；
- 释放动态 CS 数组。

第一批不注册 XIP GPIO，不创建 region 1。

---

## 4. Standard PIO 寄存器与数据路径

### 4.1 已实现寄存器组

第一批只承诺 Standard PIO、FIFO、状态和基础 IRQ 所需语义：

- `CTRLR0`、`CTRLR1`；
- `SSIENR`、`MWCR`、`SER`、`BAUDR`；
- `TXFTLR`、`RXFTLR`、`TXFLR`、`RXFLR`；
- `SR`；
- `IMR`、`ISR`、`RISR`；
- Standard IRQ clear registers；
- `IDR`、`SSIC_VERSION_ID`；
- DR aperture/aliases。

`CTRLR0.SPI_FRF` 第一批只接受 Standard 值。非零写入忽略或清零，不进入 enhanced dispatch。

### 4.2 未实现寄存器

以下寄存器和字段不建立第一批功能契约：

- `SPI_CTRLR0`、`DDR_DRIVE_EDGE` 和 enhanced-only 字段；
- `DMACR`、`DMATDLR/DMARDLR`、`AXIAWLEN/AXIARLEN`；
- `SPIDR`、`SPIAR`、`AXIAR0/1`、`AXIECR`、`DONECR`；
- XIP-only register group；
- DDR、RXDS、Octal 扩展。

它们统一 RAZ/WI，或通过集中 unsupported helper 记录 guest error。不得为这些 offset 增加 future property、状态字段或测试专用入口。

### 4.3 PIO 行为

第一批实现：

- 4～32 bit frame；
- Transmit & Receive；
- Transmit Only；
- Receive Only；
- EEPROM Read；
- loopback；
- FIFO overflow/underflow；
- enable/disable/reset；
- CS assert/deassert 生命周期。

数据路径只调用 Standard `ssi_transfer()`，不存在 enhanced command decoder、IDMA engine 或 XIP transaction。

---

## 5. 基础 IRQ

第一批只注册和实现：

```text
TXE TXO RXF RXO TXU RXU MST
```

要求：

- raw status、mask、masked status 和 clear 语义完整；
- TXE/RXF threshold level IRQ 正确；
- TXO/RXO/TXU/RXU/MST latch/clear 按基础模型实现；
- TXU 始终属于 TX FIFO underflow；
- `IMR/ISR/RISR` 中未来 DONE/AXIE/XIP 位读零、写忽略；
- device 不注册 DONE、AXIE 或 XIP IRQ output。

K230 每个实例只连接七路 IRQ 到对应 PLIC source。后续 IDMA series 增加 DONE/AXIE 时，同时增加 device outputs、PLIC 路由和测试。

---

## 6. K230 集成

### 6.1 profile 与实例化

K230 profile 只包含 `num_cs`、`fifo_depth`、`imr_reset`。三个实例 realize 前显式设置三项 property，然后映射 region 0。

不设置 `max-lines`、DMA、enhanced 或 XIP 参数。第一批三个实例均没有 region 1。

### 6.2 删除 XIP 接口

第一批 K230 machine：

- 删除 HI_SYS 到 SSI 的 `xip-enable` 连接；
- 不映射 `0xc0000000`；
- 不查询或假设 SSI region 1；
- HI_SYS 自身可以保留为 K230 设备，但不作为第一批 DW SSI 依赖。

### 6.3 Standard SPI NOR

第一批保留 `spi-flash` machine property，并把 M25P80-compatible NOR 挂到选定 SSI bus/CS。

测试只验证 Standard 1-1-1：

- JEDEC ID 或固定地址读取；
- 不设置 Dual/Quad；
- 不使用 DMA；
- 不访问 XIP aperture；
- 不以完整 U-Boot 启动作为合入条件。

Flash 挂接必须是独立 patch，不能混入通用控制器实现。

---

## 7. VMState 与迁移边界

第一批 VMState 只保存：

- 已实现寄存器状态；
- TX/RX FIFO；
- Standard PIO phase/remaining frames；
- 基础 IRQ latch；
- active CS 和必要基础状态。

只增加一项配置 equality：

```c
VMSTATE_UINT32_EQUAL(cfg.fifo_depth, DwSsiState),
```

FIFO 深度会改变 `VMSTATE_FIFO32` 的解释，必须一致。第一批没有 DMA layout 或内部 capability，因此不存在对应 equality。

enhanced command、IDMA engine、DONE/AXIE 和 XIP 状态不提前进入 schema。后续 series 通过版本或 subsection 增加真实运行时字段。

---

## 8. 通用测试载体

保留独立 `dw-ssi-test` machine：

- 一个 `TYPE_DW_SSI` child；
- 无 CPU、PLIC、Flash 和 K230 常量；
- 只映射 region 0；
- 使用 `-preconfig` 设置第一批 properties 并测试 realize error。

测试 guest-visible ABI，不暴露不存在的内部实现位。

### 8.1 TDD 矩阵

| 测试路径 | 核心断言 |
|---|---|
| `/dw-ssi/config/defaults` | 三项 property 默认值、仅 region 0、七路 IRQ |
| `/dw-ssi/config/fifo-depth` | 动态深度、满/空、threshold、reset |
| `/dw-ssi/config/invalid/*` | CS、FIFO、IMR 非法值在 realize 返回清晰错误 |
| `/dw-ssi/pio/tr` | Transmit & Receive |
| `/dw-ssi/pio/to` | Transmit Only |
| `/dw-ssi/pio/ro` | Receive Only |
| `/dw-ssi/pio/eeprom-read` | EEPROM Read |
| `/dw-ssi/irq/*` | 七路基础 IRQ 的 raw/mask/clear/output |
| `/dw-ssi/register/unsupported-enhanced` | enhanced offset RAZ/WI，Standard PIO 不受影响 |
| `/dw-ssi/register/unsupported-dma` | DMA offset RAZ/WI，无 guest memory 访问、无 DMA IRQ output |
| `/dw-ssi/register/unsupported-xip` | XIP offset RAZ/WI，无 GPIO、无 region 1 |
| `/dw-ssi/migration/same-profile` | FIFO、寄存器、IRQ 状态恢复 |
| `/dw-ssi/migration/fifo-depth-mismatch` | equality 拒绝迁移 |
| `/k230-dw-ssi/register-contract` | 三实例 `num-cs/fifo-depth/imr-reset` |
| `/k230-dw-ssi/no-xip-aperture` | 三实例无 region 1，`0xc0000000` 未映射 |
| `/k230-dw-ssi/plic-isolation` | 三实例七路 IRQ 路由和隔离 |
| `/k230-dw-ssi/standard-flash` | Standard 1-1-1 NOR 读取 |

Step 4.0 的 enhanced/XIP 隔离和 IDMA `0xeb` 测试属于当前中间态历史回归，不进入第一批最终 patch。对应功能后续重新引入时必须恢复这些测试。

---

## 9. Step 4.1 至 Step 4.4

### Step 4.1：通用 Standard PIO 控制器

1. 新增最小 `DwSsiConfig` 和三项 properties；
2. 增加 `dw-ssi-test` machine；
3. 动态创建 FIFO/CS；
4. 实现 Standard 寄存器、四种 TMOD、PIO、reset；
5. 添加基础 VMState 和 FIFO equality；
6. 对 enhanced/DMA/XIP offset 提供 RAZ/WI；
7. 完成通用 config/PIO/unsupported qtest。

完成标准：单独一个通用 patch 即可实例化、执行 Standard PIO 并通过测试，不依赖 K230 patch 补基本功能。

### Step 4.2：基础 IRQ

1. 注册七路 IRQ；
2. 实现 raw/masked/clear/threshold；
3. 确认 TXU 基础归属；
4. 增加通用 IRQ qtest。

完成标准：无 DONE/AXIE/XIP output 或状态残留。

### Step 4.3：K230 三实例与 PLIC

1. 增加三项 profile；
2. 创建和映射三个 region 0；
3. 删除 XIP GPIO 和 aperture 映射；
4. 连接七路基础 IRQ 到 PLIC；
5. 增加 profile、无 XIP aperture 和 PLIC 隔离测试。

### Step 4.4：Standard Flash 与收敛

1. 独立挂接 Standard 1-1-1 NOR；
2. 增加 Flash 读取测试；
3. 完成构建、qtest、迁移、公共头文件和依赖残留检查；
4. 重组最终第一批 patch series。

---

## 10. 第一批提交顺序

第一批为 5 个功能 patch：

1. `hw/ssi: Add a Synopsys DesignWare SSI standard PIO controller`
   - 通用类型、Standard 寄存器、FIFO、四种 TMOD、PIO、reset、VMState 和通用 qtest；
2. `hw/ssi: Add DesignWare SSI standard interrupt support`
   - 七路基础 IRQ 及 qtest；
3. `hw/riscv/k230: Instantiate DesignWare SSI controllers`
   - 三实例、最小 profile、region 0 和实例测试；
4. `hw/riscv/k230: Route SSI interrupts to the PLIC`
   - 七路基础 IRQ 的 PLIC 接线和隔离测试；
5. `hw/riscv/k230: Attach a standard SPI flash to the K230 SSI`
   - Standard 1-1-1 NOR 挂接和读取测试。

每个 patch 自己可编译、可用、可测试。测试跟随首次引入相应行为的 patch，不集中到最后。

预计总新增约 1400～1800 行，包含通用/K230 qtest、测试 machine 和 Meson 注册，不包含 cover letter。测试预计占 30%～40%。

---

## 11. 后续 series 的引入规则

### 11.1 Enhanced SPI

与真实 Dual/Quad SDR 数据路径同批引入：

- `max-lines`；
- `SPI_CTRLR0` reset/profile；
- enhanced 寄存器和 command state；
- 正负路径 qtest；
- 必要 VMState 扩展。

### 11.2 DMA/IDMA

与真实 DMA consumer 同批引入：

- external/internal register layout；
- DMA 配置寄存器语义；
- request 或 guest-memory engine；
- DONE/AXIE output 和 K230 PLIC 路由；
- 正负路径和迁移测试。

普通 enhanced/IDMA 1-4-4 必须保持 instruction → address → dummy → data。`XIP_MODE_BITS` 继续属于 XIP-only，不能重新用于 IDMA mode byte。

### 11.3 XIP

同一 series 同时引入：

- XIP properties；
- XIP-only registers；
- `xip-enable` GPIO；
- 第二个 MMIO region；
- K230 `0xc0000000` 映射；
- transaction、迁移状态和 qtest。

任何后续 series 都不得修复第一批已经引入的不可编译、不可 realize 或虚假接口中间态。

---

## 12. 验证命令

在 QEMU 源码根目录执行：

```bash
ninja -C build qemu-system-riscv64
ninja -C build tests/qtest/dw-ssi-test
ninja -C build tests/qtest/k230-dw-ssi-test
ninja -C build tests/qtest/k230-wdt-test

TMPDIR=/tmp/qemu-k230-qtest-v2-step4 \
QTEST_QEMU_BINARY=$PWD/build/qemu-system-riscv64 \
build/tests/qtest/dw-ssi-test -v

TMPDIR=/tmp/qemu-k230-qtest-v2-step4 \
QTEST_QEMU_BINARY=$PWD/build/qemu-system-riscv64 \
build/tests/qtest/k230-dw-ssi-test -v

git diff --check
scripts/checkpatch.pl -f include/hw/ssi/dw_ssi.h
scripts/checkpatch.pl -f hw/ssi/dw_ssi.c
scripts/checkpatch.pl -f hw/ssi/dw_ssi-test.c
scripts/checkpatch.pl -f hw/riscv/k230.c
scripts/checkpatch.pl -f tests/qtest/dw-ssi-test.c
scripts/checkpatch.pl -f tests/qtest/k230-dw-ssi-test.c
```

定向残留检查：

```bash
rg -n 'K230|k230' hw/ssi/dw_ssi.c include/hw/ssi/dw_ssi.h
rg -n 'capabilities|DW_SSI_CAP_|dw_ssi_has_capability|xip_window_size' \
  hw/ssi/dw_ssi.c include/hw/ssi/dw_ssi.h
rg -n 'DONE|AXIE|xip-enable|dma-register-layout|max-lines' \
  hw/ssi/dw_ssi.c include/hw/ssi/dw_ssi.h hw/riscv/k230.c
```

预期第一组没有 K230 类型、地址或 HI_SYS 依赖；第二组无结果；第三组只允许后续功能注释，不允许第一批接口或状态。

---

## 13. 实施清单

### Step 4.0

- [x] ordinary enhanced/IDMA 不读取 XIP-only fields
- [x] 删除普通 enhanced mode phase
- [x] 删除 IDMA 1-4-4 XIP mode-byte 特判
- [x] K230 SSI qtest 12/12 PASS

### Step 4.1

- [ ] `DwSsiConfig` 只含 `num_cs/fifo_depth/imr_reset`
- [ ] 只暴露三项有消费者的 property
- [ ] 动态 FIFO/CS 生命周期完整
- [ ] Standard PIO 四种 TMOD 通过
- [ ] unsupported enhanced/DMA/XIP offset 测试通过
- [ ] VMState 只含基础状态和 FIFO equality

### Step 4.2

- [ ] 只注册七路基础 IRQ
- [ ] TXU 归属和 raw/mask/clear/threshold 测试通过
- [ ] 无 DONE/AXIE/XIP output

### Step 4.3

- [ ] K230 三实例只设置三项 profile
- [ ] 三实例只映射 region 0
- [ ] 无 XIP GPIO、region 1、`0xc0000000` 映射
- [ ] 七路 IRQ PLIC 路由和隔离通过

### Step 4.4

- [ ] Standard 1-1-1 Flash 挂接和读取通过
- [ ] 通用/K230/WDT qtest 全过
- [ ] 公共头文件、依赖残留、diff/checkpatch 通过
- [ ] 最终 5 个 patch 各自可编译、可测试

---

## 14. Cover Letter 边界声明

建议主动说明：

> This series introduces the Standard PIO baseline of a reusable Synopsys DesignWare SSI controller, together with the seven standard interrupt outputs and K230 integration. Enhanced SPI, DMA/IDMA, and XIP interfaces are intentionally not exposed in this revision; each will be introduced with its first functional consumer in a follow-up series. Unsupported extension registers are read-as-zero/write-ignored and no unused IRQ, GPIO, or MMIO resources are pre-created.

同时说明 Standard 1-1-1 NOR 只证明真实 peripheral consumer，不宣称 enhanced、DMA 或 XIP 启动链已经完成。

---

## 15. 参考入口

- [V2 决策记录](k230-spi-qspi-review-v2-decision-notes.md)
- [V2 实施路线](k230-spi-qspi-v2-implementation-plan.md)
- [V1.2 scope revision handoff](k230-spi-qspi-v2-step4-v1.2-review-scope-revision-handoff.md)
- [K230 TRM SPI 中文对照](../spi/reference/k230-trm-12.3-spi-cn.md)
- [寄存器审计](../spi/k230-spi-qspi-register-audit.md)
- [当前 DW SSI 模型](../../../my-qemu-camp-2026-k230/hw/ssi/dw_ssi.c)
- [当前 K230 machine](../../../my-qemu-camp-2026-k230/hw/riscv/k230.c)
- [当前 K230 SSI qtest](../../../my-qemu-camp-2026-k230/tests/qtest/k230-dw-ssi-test.c)
- [QEMU DesignWare I2C 先例](../../../my-qemu-camp-2026-k230/hw/i2c/designware_i2c.c)

本文只修改实施计划，不代表 Step 4.1 至 Step 4.4 已经修改源码。实施中若发现第一批某个字段、IRQ 或寄存器没有当前消费者，应继续删除；若发现新的必要消费者，先记录证据，再扩大范围。
