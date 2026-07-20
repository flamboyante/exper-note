# K230 SSI Patch 5：Dual/Quad QSPI 学习工作簿

更新时间：2026-07-17

当前状态：P5-1、P5-2、P5-3 已完成；Dual/Quad SDR PIO 的增强读取与增强
写入均已接入。Patch 5 范围内的 reg 8、PIO 10、Standard Flash 7 和 QSPI 6
共 31 项测试已通过。本文记录设计边界、实现分工和复习要点，不替代代码中的
最终行为。

## 目标

Patch 5 在 Patch 3 的 FIFO/TMOD 状态机和 Patch 4 的真实 W25Q256 接线上，
加入 K230 DWC SSI 的增强 SPI 事务生成：

```text
Guest 写 DR 中的 instruction/address
        + CTRLR0.SPI_FRF
        + CTRLR0.TMOD
        + SPI_CTRLR0
        + CTRLR1.NDF
                    |
                    v
instruction -> address -> mode -> dummy -> data
                    |
                    v
               同一 SSIBus/CS0/W25Q256
```

本 Patch 的核心不是模拟四根物理线的每个电平，而是把 Guest 配置翻译成
Flash 能观察到的正确字节阶段，并继续复用 `ssi_transfer()`。

建议提交标题：

```text
hw/ssi: Implement K230 enhanced QSPI transfers
```

## 本 Patch 做与不做

必须完成：

- 支持 Dual/Quad SDR PIO 的增强读取和增强写入。
- 实现 `instruction -> address -> mode -> dummy -> data` 五阶段。
- 让 `SPI_FRF/TRANS_TYPE/INST_L/ADDR_L/XIP_MBL/XIP_MD_BIT_EN/
  WAIT_CYCLES/TMOD/NDF` 真正参与事务。
- 增强 `TMOD=RO` 在 DATA 阶段生成接收时钟并写 RX FIFO。
- 增强 `TMOD=TO` 在 DATA 阶段消费 TX FIFO，并支持 Guest 分批填入 payload。
- 使用 Patch 3 的 TX/RX FIFO、SER、BUSY、`remaining_frames` 和恢复入口。
- 为 Patch 9 XIP 保留可复用的增强命令描述。
- 明确拒绝 Octal、DDR/DTR、RXDS、`TRANS_TYPE=3` 和超过实例线宽的配置。

本 Patch 不实现：

- Octal/OPI。
- DDR/DTR、RXDS、HyperBus、Data Mask。
- 内部 IDMA。
- IRQ/PLIC。
- `0xC0000000` XIP window。
- Flash opcode 特判或第二套 Flash 状态机。
- 增强 `TMOD=TR/EEPROM_READ`；K230 spi-mem 的明确增强路径使用 RO/TO。

## 已搭好的学习脚手架

生产代码已经提供：

- `K230DwSsiEnhancedCommand`：保存增强事务的纯描述。
- 五个 `K230_DW_SSI_PHASE_ENHANCED_*` 阶段。
- reset、abort 和 VMState 中的增强状态清理/迁移字段。
- Standard 与 Enhanced 共用的原始 TX FIFO。
- `SPI_FRF != Standard` 时进入增强 pump 的分流入口。

qtest 已整理为独立文件：

```text
tests/qtest/k230-dw-ssi-qspi-test.c
```

三个核心实现位置位于：

```text
hw/ssi/k230_dw_ssi.c

P5-1  校验增强配置是否受支持
P5-2  从寄存器和 TX FIFO 构造增强命令描述
P5-3  推进 instruction/address/mode/dummy/data，并按 TMOD 分流 DATA
```

建议严格按 P5-1 → P5-2 → P5-3 实现。不要先改 XIP 或 IDMA 来绕过当前阶段。

## 1. Patch 3 与 Patch 5 的关系

Patch 5 不是重写 `k230_dw_ssi_run_transfer()`，而是在 Standard 分支旁增加一种
“控制字段自动展开”方式：

| 内容 | Patch 3 Standard | Patch 5 Enhanced |
|---|---|---|
| TX/RX FIFO | 使用 | 继续使用 |
| SSIENR/SER/CS | 使用 | 继续使用 |
| `remaining_frames` | RO/EEPROM 数据阶段 | Enhanced RO/TO 数据阶段复用 |
| 单个 TX FIFO 项 | 按 DFS 解释为普通帧 | instruction/address 可是完整逻辑字段 |
| 命令组织者 | Guest 逐帧写出 | 控制器按增强寄存器拆阶段 |
| Flash 协议 | M25P80 处理 | 仍由 M25P80 处理 |

最重要的变化是 TX FIFO 的保存语义。K230 U-Boot 的增强 PIO 路径会执行：

```c
writel(op->cmd.opcode, DR);
writel(op->addr.val, DR);
```

第二个 DR 项是一个完整的 Enhanced 逻辑字段，不一定是一个待发送的 SPI 帧。
例如配置 `ADDR_L=24` 时，DR 可以保存 `0x00123456`，P5-2 再把它拆成三个地址
字节。`DFS` 只描述实际在线上的帧宽度，不描述 FIFO 项的保存宽度；因此不能在
写 DR 时提前执行 `tx &= frame_mask`，否则地址高位会在 P5-2 解析前永久丢失。

Standard 和 Enhanced 对 FIFO 的解释不同：Standard 8-bit SPI 没有地址字段，
24-bit 地址可能由三个普通 FIFO 项表示：

```text
FIFO[0] = 0x12
FIFO[1] = 0x34
FIFO[2] = 0x56
```

Enhanced/QSPI PIO 则可能把完整地址放在一个逻辑项中：

```text
FIFO[0] = 0x0000006b   // instruction
FIFO[1] = 0x00123456   // address
```

这里的“一个 FIFO 项”不等于“一次 SPI 传输”：Enhanced 控制器会把第二项展开
成 `0x12 -> 0x34 -> 0x56`。当前模型的 descriptor 只有 `uint32_t address`，因此
只支持常见的 24/32-bit 地址；更长地址暂时拒绝。

当前脚手架的分工是：

```text
写 DR/TX FIFO       保存完整的 uint32_t 项
Standard 实际发送   k230_dw_ssi_send_frame() 按 DFS 截断
Enhanced P5-2       按 INST_L/ADDR_L 解释并拆分逻辑字段
```

这也是为什么不需要另建 `qspi_tx_fifo`：控制器只有一条 TX FIFO 和一条 RX FIFO，
`DR0..DR35` 是同一个 FIFO 的别名。两种路径共享同一条 Guest 可见的 DR/TX FIFO，
区别只在于消费 FIFO 时如何解释其中的项。Standard 路径仍在真正发送前按 DFS
截断，因此保存完整值不会把高 24 位发到线上。

注意：本模型中 `CTRLR0.DFS` 的寄存器值是“实际位数减一”，所以寄存器值 7
表示 8-bit frame。文档中若写“DFS=8-bit”，指的是实际帧宽度，而不是寄存器
字段值 8。

### 1.5 Enhanced Read 与 Write 的完整数据流

两种操作共用同一控制前缀：

```text
1. SSIENR=1，SER 暂不选择 CS
2. Guest 写 instruction 到 DR
3. Guest 写完整 address 到 DR
4. Guest 写 SER，选择 CS
5. pump 检查配置并构造 descriptor
6. instruction -> address -> mode -> dummy
7. 根据 TMOD 进入 DATA：
   RO：dummy 产生接收时钟，返回值进入 RX FIFO
   TO：从 TX FIFO 取得 payload 并发送
8. FIFO 暂时无法推进时保留 phase/remaining_frames
9. 后续 DR 读写重新调用 pump
10. remaining_frames=0 后 phase 回到 IDLE
```

`pump` 可以在数据尚未完整时被调用，但不能提前开始发送：

```text
只写 instruction
    -> pump 发现 address 不足
    -> 保持 IDLE，不 pop，不发送

再写 address
    -> 再次 pump
    -> instruction/address 齐全后一次性构造并开始事务
```

因此“调用 pump”和“真正开始 Enhanced 传输”是两个概念。

增强写入还有一个额外边界：SER 拉起时 TX FIFO 可能只有 instruction/address，
payload 由 Guest 随后分批写入。此时前缀发送完后进入 `ENHANCED_DATA`，TX FIFO
暂时为空只表示等待，不代表事务结束：

```text
remaining_frames > 0 && TX FIFO empty
    -> 保持 ENHANCED_DATA/BUSY
    -> Guest 下次写 DR 时恢复

remaining_frames == 0
    -> 写入数据阶段完成，phase 回到 IDLE
```

reset 或 `SSIENR` 从 1 变 0 时，必须释放 CS、清空 TX/RX FIFO、将 `phase` 和
`remaining_frames` 清零，并清理 Enhanced descriptor。VMState 也需要保存 FIFO、
phase、remaining_frames 以及 descriptor 的全部字段，避免迁移或 abort 后从旧阶段
继续执行。

## 2. P5-1：支持能力校验

函数：

```c
k230_dw_ssi_enhanced_config_supported()
```

先读取：

```text
spi_frf       = CTRLR0.SPI_FRF
TMOD          = CTRLR0.TMOD
trans_type    = SPI_CTRLR0.TRANS_TYPE
spi_ctrlr0    = regs[SPI_CTRLR0]
required_lines = spi_frf == Dual ? 2 :
                 spi_frf == Quad ? 4 : 0
```

代码中应使用 `FIELD_EX32()` 读取字段。不要用 `1 << spi_frf` 代替上述映射后
再直接判断，因为 Standard=0 和 Octal=3 也会得到数值，容易掩盖“先判断格式、
再判断实例线宽”的语义。

本 Patch 的接受表：

| 条件 | 结果 | 原因 |
|---|---|---|
| `SPI_FRF=Dual` | 可继续校验 | Patch 5 范围 |
| `SPI_FRF=Quad` | 可继续校验 | Patch 5 范围 |
| `SPI_FRF=Octal` | 拒绝 | 留给后续 OPI，不在当前范围 |
| `required_lines > max_lines` | 拒绝 | 实例综合能力不足 |
| `TMOD=RO` | 接受 | Enhanced Data-IN |
| `TMOD=TO` | 接受 | Enhanced Data-OUT |
| `TMOD=TR/EEPROM_READ` | 拒绝 | K230 spi-mem 增强路径没有当前需求 |
| `TRANS_TYPE=0/1/2` | 接受 | TRM 定义的 TT0/TT1/TT2 |
| `TRANS_TYPE=3` | 拒绝 | TRM 保留编码 |
| `SPI_DDR_EN/INST_DDR_EN` | 拒绝 | Patch 5 只做 SDR |
| `SPI_RXDS_EN/SPI_RXDS_SIG_EN` | 拒绝 | 不建立 RXDS/HyperBus 行为 |

P5-1 的职责到这里为止。它不需要在本阶段校验或修改 `DFS`、`INST_L`、`ADDR_L`、
`XIP_MBL` 和 `NDF` 的具体含义；这些字段属于 P5-2 的命令描述解析。尤其不能
在 P5-1 或 DR 写入路径中按 DFS 裁剪 TX FIFO 项。descriptor 是否能表示超过 32 bit
地址，是 P5-2 的可表示性检查，不是 P5-1 的能力检查。

拒绝契约必须是：

```text
不 pop TX FIFO
不调用 ssi_transfer()
不写 RX FIFO
phase 保持 IDLE，因此 BUSY=0
```

不要通过清空 FIFO 表示拒绝，因为 Guest 仍应能在禁用控制器时统一 abort。

### P5-1 自检

- [x] 我没有把 `max_lines=8` 误解为本 Patch 必须支持 Octal。
- [x] 我检查了 `TRANS_TYPE=3`。
- [x] 我只接受增强 `TMOD=RO/TO`，没有把所有 TMOD 都送入同一 DATA 逻辑。
- [x] 我检查了 DDR、instruction DDR 和两种 RXDS 位。
- [x] Dual/Quad 的所需线宽分别按 2/4 判断，并与 `max_lines` 比较。
- [x] 返回 false 前没有改变 FIFO/phase。
- [x] 返回 false 前没有调用 `ssi_transfer()` 或伪造 RX 数据。

## 3. P5-2：构造增强命令描述

函数：

```c
k230_dw_ssi_prepare_enhanced_command()
```

这个函数只做“读取配置并形成描述”，不应调用 `ssi_transfer()`。

### 3.1 字段解码

| 寄存器字段 | 描述字段 | 解码 |
|---|---|---|
| `CTRLR0.SPI_FRF` | `spi_frf` | 1=Dual，2=Quad |
| `CTRLR0.TMOD` | `tmod` | RO=增强读，TO=增强写 |
| `SPI_CTRLR0.TRANS_TYPE` | `trans_type` | TT0/TT1/TT2 |
| `SPI_CTRLR0.INST_L` | `instruction_bits` | 0/4/8/16 |
| `SPI_CTRLR0.ADDR_L` | `address_bits` | 字段值 × 4 |
| `SPI_CTRLR0.XIP_MBL` | `mode_bits` | 2/4/8/16 |
| `XIP_MODE_BITS` | `mode` | 低 16 位 |
| `SPI_CTRLR0.XIP_MD_BIT_EN` | `mode_bits_enabled` | 是否存在 mode 阶段 |
| `SPI_CTRLR0.WAIT_CYCLES` | `wait_cycles` | 原值保存 |
| `CTRLR1.NDF` | `data_frames` | `NDF + 1` |

`INST_L` 是编码值，不能直接乘四：

```text
INST_L=0/1/2/3 -> 0/4/8/16 bit
```

可以写成：

```c
instruction_bits = inst_l ? (1U << (inst_l + 1)) : 0;
```

`ADDR_L` 和 `XIP_MBL` 的编码是连续的，可以分别写成：

```c
address_bits = addr_l << 2;
mode_bits = 1U << (mode_length_encoding + 1);
```

但 `mode_bits` 表示长度，不表示 mode 的值；descriptor 中的 `mode` 才是实际值。
`mode_bits_enabled` 不能用 `mode == 0` 替代，因为“未启用”和“启用但值为 0”
是两种不同情况。

### 3.2 原子消费 TX FIFO

增强 RO/TO 的 SDK PIO 控制前缀通常是：

```text
TX FIFO item 0 = instruction
TX FIFO item 1 = address（若 ADDR_L != 0）
```

对于增强 TO，item 2 之后还可能保存 data payload；P5-2 只原子消费
instruction/address，不能提前 pop payload。payload 已存在时由 DATA 阶段立即消费；
尚未存在时则在 `ENHANCED_DATA` 中等待后续 DR 写入。

先计算需要几个 FIFO 项，再检查 `fifo32_num_used()`。只有数量足够时才能统一 pop。
否则会出现“instruction 被吃掉、address 还没来”的半命令状态。

这里必须从 FIFO 取出完整的 `uint32_t` 原值，再根据 `INST_L` 和 `ADDR_L` 解释。
例如 `ADDR_L=24` 时，第二项的 `0x00123456` 应保留为地址 `0x123456`，而不是
先按 8-bit DFS 变成 `0x56`。DFS 的帧裁剪只适用于 Standard 的 `send_frame()`；
Enhanced 阶段要按逻辑字段长度生成后续的字节传输。

`MAKE_64BIT_MASK(0, bits)` 生成从 bit 0 开始、长度为 `bits` 的低位掩码。例如：

```text
bits=8  -> 0x000000ff
bits=24 -> 0x00ffffff
```

因此：

```c
value & (uint32_t)MAKE_64BIT_MASK(0, bits)
```

表示只保留逻辑字段的有效低位；这里的 `&` 是按位与。

建议逻辑：

```text
required_items = (instruction_bits != 0) + (address_bits != 0)
if TX FIFO used < required_items:
    return false

按顺序 pop instruction/address
填写 descriptor
记录 TMOD
remaining_frames = data_frames
phase = ENHANCED_INSTRUCTION
return true
```

可以先在局部变量中构造完整 descriptor，最后执行：

```c
s->enhanced = command;
```

这是结构体值拷贝，不是保存局部变量指针。函数返回后，`command` 的字段内容已经
复制到设备状态中；数据不足时则不会修改 `s->enhanced`。这也是“先检查并完成
构造，再一次性提交状态”的原因。

### 字段推导例子

给定：

```text
SPI_FRF=Quad
TRANS_TYPE=1
INST_L=2
ADDR_L=6
XIP_MBL=2
XIP_MD_BIT_EN=1
WAIT_CYCLES=4
CTRLR1.NDF=3
```

结果是：

```text
instruction_bits = 8
address_bits     = 24
mode_bits         = 8
data_frames       = 4
instruction 线宽  = 1 线
address 线宽      = 4 线
data 线宽          = 4 线
```

也就是 Quad 下的 `1-4-4` 事务。

## 4. P5-3：五阶段执行器

函数：

```c
k230_dw_ssi_run_enhanced_transfer()
```

推荐阶段责任：

| phase | 动作 | 下一个 phase |
|---|---|---|
| `INSTRUCTION` | 按 `instruction_bits` 大端发送 | `ADDRESS` |
| `ADDRESS` | 按 `address_bits` 大端发送 | `MODE` |
| `MODE` | 未启用则跳过，否则按 `mode_bits` 发送 `mode` | `DUMMY` |
| `DUMMY` | 发送 `wait_cycles` 个事务级 dummy | `DATA` |
| `DATA + RO` | 每次 dummy 获得一个 RX frame | 完成后 `IDLE` |
| `DATA + TO` | 从 TX FIFO 消费一个 payload frame | 完成后 `IDLE` |

P5-3 需要复用前缀执行器，但不能直接复用 Standard 的 `TMOD=TO/RO` 分支：

```text
Standard TO：每个 TX FIFO 项就是一个普通帧
Standard RO：一个 dummy DR 后自动生成 NDF 个接收帧
Enhanced：instruction -> address -> mode -> dummy -> data
```

可以复用的是 FIFO、`phase`、`remaining_frames`、FIFO 满/空暂停和 DR 访问后的恢复
这些状态管理机制；不能直接把完整 address 当成 Standard DFS 帧发送。

可以新增一个很小的 helper，例如“按大端顺序发送一个 N-bit 值”，避免 instruction、
address、mode 三处复制移位循环。这个抽象符合 DRY；不要把 FIFO、phase 或 RX push
也塞入该 helper，否则职责会过大。

### 4.1 大端发送

地址 `0x00123456`、`address_bits=24` 应让 Flash 依次看到：

```text
0x12 -> 0x34 -> 0x56
```

不是主机内存字节序，也不是 `0x56 -> 0x34 -> 0x12`。

### 4.2 mode bits

`XIP_MD_BIT_EN=0` 时必须完全跳过 MODE 阶段。置 1 时，长度来自 `XIP_MBL`，
内容来自 `XIP_MODE_BITS`。当前 `qspi/mode-bits-dummy` 使用 8-bit mode，避免把 QEMU 字节总线误装成
位级总线；2/4-bit 值可右对齐后通过一次事务级字节发送表达。

### 4.3 dummy：硬件周期与 QEMU 字节接口

TRM 中 `WAIT_CYCLES` 是 SPI 时钟周期；但 QEMU `m25p80` 明确采用：

```text
dummy cycles modeled with bytes writes instead of bits
```

因此本模型不根据 opcode 硬编码 dummy，而是在事务适配层按 `WAIT_CYCLES` 次数调用
`ssi_transfer(..., 0)`：

- Q01 `0x6b` 配置 `WAIT_CYCLES=8`，m25p80 收到 8 个 dummy 写入。
- Q02 `0xeb` 先收到一个 mode byte，再按 `WAIT_CYCLES=4` 收到 4 个 dummy 写入。

这是 QEMU 外设接口的抽象适配，不代表真实 Quad 总线上每周期传输了一个字节。

### 4.4 DATA 与暂停恢复

RO 数据阶段沿用 Patch 3 的规则：

```text
while RX FIFO 未满 && remaining_frames > 0:
    rx = ssi_transfer(dummy)
    push RX FIFO
    remaining_frames--
```

RX FIFO 满时不能清 phase，也不能忙等。Guest 读取任意 DR 别名后，已有读路径会再次
调用 `k230_dw_ssi_run_transfer()`，从 `ENHANCED_DATA` 恢复。

TO 数据阶段方向相反：

```text
while TX FIFO 非空 && remaining_frames > 0:
    tx = pop TX FIFO
    ssi_transfer(tx)
    remaining_frames--
```

TX FIFO 空时不能清 phase。Guest 后续写 DR 会再次调用 pump，从同一个
`ENHANCED_DATA` 阶段继续。只有 `remaining_frames == 0` 才能回到 IDLE 并清 BUSY。

### 4.5 为什么读写共用一个增强执行器

RO 与 TO 的差异只出现在 DATA：

```text
k230_dw_ssi_run_enhanced_transfer()
    instruction
    -> address
    -> mode
    -> dummy
    -> DATA
         RO: k230_dw_ssi_run_enhanced_rx_data()
         TO: k230_dw_ssi_run_enhanced_tx_data()
```

不要复制一套 `run_enhanced_write_transfer()`。两套函数会重复前缀阶段，随后很容易在
mode、dummy、abort 或迁移逻辑上发生偏差。descriptor 保存 `tmod`，统一执行器只在
DATA 分流，符合 DRY 和单一职责。

### P5-3 自检

- [x] 我没有在增强路径再次 pop instruction/address。
- [x] 每个阶段只执行一次并明确进入下一阶段。
- [x] 地址按大端总线顺序发送。
- [x] mode 禁用时没有发送占位字节。
- [x] dummy 没有按 opcode 做 switch。
- [x] DATA 在 RX FIFO 满时能暂停，并在 DR read 后恢复。
- [x] DATA 在 TX FIFO 空时能暂停，并在后续 DR write 后恢复。
- [x] 完成后 `phase=IDLE` 且 `remaining_frames=0`。

## 5. 事务级模型边界

当前 QEMU SSIBus 不携带“这一个字节用了几根线”的参数，因此 `TRANS_TYPE`/`SPI_FRF`
只用于能力校验、descriptor 保存和阶段语义，不模拟四根 GPIO 的逐线波形。

`TRANS_TYPE=1` 的 `0xeb` 仍可以通过普通 `ssi_transfer()` 交给 M25P80，是因为当前
测试关心 Flash 能观察到的字节顺序和阶段；M25P80 不需要控制器额外实现四根 GPIO。
这不是忽略寄存器，而是事务级接口的边界。Patch 9 XIP 可以复用同一 descriptor；
若未来总线接口携带线宽元数据，再扩展逐线语义。

## 6. 六项 qtest 的证明范围

| 编号 | 用例 | 证明什么 |
|---|---|---|
| Q01 | `qspi/dual-quad-output-read` | Dual 1-1-2 与 Quad 1-1-4 共用增强读取路径 |
| Q02 | `qspi/mode-bits-dummy` | 1-4-4，mode 与 dummy 真正影响最终数据，不只是寄存器可读回 |
| Q03 | `qspi/quad-page-program` | Quad TO、payload 分批写入、BUSY 等待和真实回读 |
| Q04 | `qspi/rx-fifo-resume` | RO 在 RX FIFO 满时暂停，DR read 后补齐最后一帧 |
| Q05 | `qspi/prefix-atomic` | instruction/address 不完整时不半途 pop 或发送 |
| Q06 | `qspi/unsupported-configs` | 独立拒绝 Octal、DDR/RXDS、TT3、TR 和 EEPROM_READ |

Q02 读取 Flash 固定模式 `a5 5a 3c c3`。如果只保存寄存器、不执行 mode/dummy，
测试会失败。Q03 使用 Standard WREN、Quad Page Program 和增强回读，证明 TO 不只是
消费 FIFO，而是真正改变同一个 W25Q256 backend。

## 7. 实现顺序与验证结果

本次实现顺序和结果：

1. P5-1 拒绝非法 Dual/Quad 组合，Q06 证明不发送、不产 RX、不进入 BUSY。
2. P5-2 原子构造 descriptor，Q05 证明前缀不足时不半途消费 FIFO。
3. P5-3 共用前缀执行器，RO/TO 在 DATA 阶段分流。
4. Q01/Q02/Q04 验证 Dual/Quad 读取、mode/dummy 和 RX FIFO 恢复。
5. Q03 验证流式 Quad Page Program；reg/pio/flash 全部无回归。

Patch 5 的累计门禁为：

| 分组 | 数量 | 结果 |
|---|---:|---|
| reg | 8 | PASS |
| pio | 10 | PASS |
| flash | 7 | PASS |
| qspi | 6 | PASS |
| 合计 | 31 | PASS |

不要在 Patch 5 阶段把整个 51 项测试集当作必须通过的门禁。后续 IRQ/PLIC、HiSys
和 XIP 用例已经注册在同一个 qtest 二进制中，但分别属于 Patch 6/7、Patch 8 和
Patch 9。
例如当前 Patch 6 尚未实现时，IRQ `watermark-mask` 会因为 `RISR.TXE=0` 失败；
这不属于 Patch 5 QSPI 数据路径回归。

运行：

```bash
ninja -C "build" \
    "qemu-system-riscv64" \
    "tests/qtest/k230-dw-ssi-test"

QTEST_QEMU_BINARY="./build/qemu-system-riscv64" \
    ./build/tests/qtest/k230-dw-ssi-test \
    -p /riscv64/k230-dw-ssi/qspi

QTEST_QEMU_BINARY="./build/qemu-system-riscv64" \
    ./build/tests/qtest/k230-dw-ssi-test \
    -p /riscv64/k230-dw-ssi/flash
```

## 8. 完成标准

- [x] P5-1、P5-2、P5-3 已按顺序完成。
- [x] Q01–Q06 全部通过。
- [x] Patch 2 寄存器 8 项持续通过。
- [x] Patch 3 PIO 10 项持续通过。
- [x] Standard Flash 7 项持续通过。
- [x] Patch 5 范围累计 31/31 PASS。
- [x] 未修改 `hw/block/m25p80.c`。
- [x] 未加入 XIP window、IRQ、PLIC 或 IDMA 搬运。
- [x] 没有按 Flash opcode 硬编码控制器行为。
- [x] reset、disable/abort、迁移状态包含增强事务进度。

达到以上条件后，Patch 5 才算告一段落，可以开始 Patch 6 的控制器内部 IRQ。

## 9. Patch 6 集成验证报告：U-Boot、Linux 与回归比较

本节记录 2026-07-17 对提交 `88f1062118d0889b4ce833905592daed61a53c02`
（`qspi write`）的独立复测。测试在 `/tmp/k230-p6-verify` detached worktree 中
进行；主工作树及其中正在学习的后续改动没有参与构建或修改。

### 9.1 与 Patch 4 final 的进展

比较基线为 `fd8caab4afe95872d903d9eacd0d4d651d0b76fc`（Patch 4 final）。当前
提交新增/整理了增强 QSPI 的事务状态机和 6 项专用测试：

```text
instruction -> address -> mode -> dummy -> data
```

| 项目 | Patch 4 final | 当前提交 | 结论 |
|---|---:|---:|---|
| reg | 8/8 | 8/8 | 无回归 |
| PIO | 10/10 | 10/10 | 无回归 |
| Standard Flash | 7/7 | 7/7 | 无回归 |
| QSPI | 仅旧的部分/脚手架 | 6/6 | Dual/Quad SDR PIO 读写完成 |
| 全量 qtest | 28 PASS / 19 FAIL / 47 | 33 PASS / 18 FAIL / 51 | 新增 QSPI 覆盖；后续范围失败少 1 项 |

当前 6 项 QSPI 用例覆盖 Dual/Quad 输出读、mode/dummy、Quad Page Program、RX FIFO
满后的恢复、前缀原子性，以及 Octal/DDR/RXDS/非法 TMOD 的拒绝路径。全量仍失败的
18 项全部属于尚未实现的 IRQ/PLIC、HiSys 或 XIP；它们不是本提交引入的回归。

### 9.2 U-Boot 实测流程与结果

使用未修改的 SDK U-Boot、派生的 1-line SPI NOR DTB，以及临时 32 MiB raw backend：

```bash
truncate -s 32M /tmp/k230-p6-uboot-w25q256.bin
build/qemu-system-riscv64 \
  -machine k230,spi-flash=w25q256 \
  -drive file=/tmp/k230-p6-uboot-w25q256.bin,format=raw,if=mtd \
  -bios /tmp/k230-p4-uboot -nographic -no-reboot
```

在 `K230#` 执行：

```text
sf probe 0:0
sf read ${loadaddr} 0 0x10
sf write ${loadaddr} 0 0x10
sf erase 0 0x10000
sf read ${loadaddr} 0 0x10
md.b ${loadaddr} 0x10
```

结果：识别 `w25q256`（32 MiB），读写均返回 `OK`；擦除后读回 16 字节均为 `ff`。
Probe 初始仍会出现 `Software reset enable failed: -524`，这是 SDK 先尝试不在当前
范围内的 Octal DTR reset、随后回退 Standard Read-ID 的非致命提示。

`sf erase 0 0x1000` 曾返回 `-22`，原因是 W25Q256 的命令擦除粒度为 64 KiB，而非
控制器或 Patch 6 问题；使用 `sf erase 0 0x10000` 后成功。

该 U-Boot 用例验证的是 Standard fallback 没有被增强状态机破坏；它没有触发
Dual/Quad 数据事务，不能替代 QSPI qtest。

### 9.3 Linux 实测流程与结果

先使用 1-line DTB + Buildroot initramfs，Linux 5.10.4 输出：

```text
spi-nor spi0.0: w25q256 (32768 Kbytes)
Creating 2 MTD partitions on "spi0.0"
```

再用镜像原始 `k230.dtb`（`spi-tx-bus-width = <8>`、`spi-rx-bus-width = <8>`）启动，
仍得到相同的 NOR 识别和分区创建结果。因此当前提交没有破坏原始 Linux DTS 下的
probe/fallback。

这不是 Linux Dual/Quad PIO 的端到端证明。Read-ID/probe 可以走 Standard 路径；且
启动日志仍有：

```text
dw_spi_mmio 91584000.spi: IRQ index 9 not found
spi spi0.0: setup: ignoring unsupported mode bits 6000
```

前者对应未实现的 SSI IRQ/PLIC 范围。后者来自 SDK 通用 `spi-dw-core.c` 仅声明
`SPI_CPOL | SPI_CPHA | SPI_LOOP` 为 controller `mode_bits`，而 DTS 请求 Octal
TX/RX 能力；SDK 保留了该设备 mode，因此 Standard fallback 能继续，但日志表明
当前启动本身不是干净的 QSPI 能力握手。

### 9.4 结论、边界与下一步

当前没有发现 Patch 6 的控制器模型回归：构建成功，Patch 4 的 Standard 路径、6 项
新增 Dual/Quad PIO QSPI 测试，以及 U-Boot/Linux 的基础 Flash probe 都通过。

仍不能宣称“原始 Linux 已端到端使用 Patch 6 QSPI”：原始 DTS 要求 8-line，而当前
模型按范围拒绝 Octal、DDR/RXDS 和 IDMA；Linux probe 成功只说明 fallback 有效。
要完成 Linux 侧的增强集成测试，下一步应：

1. 准备只声明 2-line 或 4-line 的派生 DTS，避免把 Octal 要求混入结果。
2. 让 SDK SPI controller 明确宣告 Dual/Quad `mode_bits`，或切换到适配的 K230
   驱动路径；不能只依赖 `spi-tx/rx-bus-width` 属性。
3. 用实际 `spi_mem_op`/MTD 读写触发 1-1-2、1-1-4 或 1-4-4，并开启 QEMU trace
   证明 `instruction/address/mode/dummy/data` 的完整路径。
4. 将 `IRQ index 9`、18 项 IRQ/HiSys/XIP qtest 失败保留给相应后续 Patch，避免把
   中断、IDMA、Octal 或 XIP 偷渡进当前范围。
