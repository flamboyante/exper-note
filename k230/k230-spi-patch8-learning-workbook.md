# K230 SSI Patch 8：HI_SYS `SSI_CTRL` 学习工作簿

更新时间：2026-07-19

当前状态：Patch 7 负责三个 SSI 到 PLIC 的 27 路接线；Patch 8 转向 SoC
包装层寄存器 `HI_SYS.SSI_CTRL`。独立 QOM 设备、三个逻辑 SSI 状态连接和
H01–H04 qtest 脚手架已经就位，等待按门禁逐项收口。

## 目标

Patch 8 建模独立的 HI_SYS_CONFIG 设备，不修改 DWC SSI 控制器的寄存器地址
语义，不把 `SSI_CTRL` 错并到任一 SSI 的 `base + 0x068`：

```text
HI_SYS_CONFIG base       = 0x91585000
HI_SYS.SSI_CTRL          = 0x91585068
spi0/spi1/spi2 DR2       = base + 0x068（各自控制器内部）
```

建议标题：

```text
hw/misc: Model the K230 HI_SYS SSI control register
```

## 与 P7 的边界

| Patch | 负责内容 |
|---:|---|
| P7 | SSI 九路 IRQ 输出接入 PLIC 146–172，以及 reset 后 IRQ 重驱动 |
| P8 | HI_SYS `SSI_CTRL` 的寄存器、动态状态和 `spi0` XIP gate 接口 |

P8 不修复 P7 的 PLIC/reset 问题，也不修改 SSI 内部 IRQ 算法。P8 依赖三个
SSI 实例已经稳定存在，但不依赖 PLIC 接线；后续 P9 的 XIP window 才依赖
P8 导出的 `ssi0_xip_en`。

## 寄存器契约

### `SSI_CTRL @ 0x068`

```c
#define K230_SSI_CTRL_RESET             0x00004000U
#define K230_SSI_CTRL_IMPLEMENTED_MASK 0x0003fff1U
#define K230_SSI_CTRL_WRITABLE_MASK    0x0003e001U
```

| 位 | 字段 | 访问 | 来源/行为 |
|---:|---|---|---|
| 0 | `ssi0_xip_en` | R/W | 包装层保存；后续控制 spi0 XIP window |
| 3:1 | 保留 | R | 读 0，写忽略 |
| 4 | `ssi0_ssi_sleep` | RO | spi0 对外 sleep 信号 |
| 6:5 | `ssi0_spi_mode` | RO | 动态反映 spi0 `SPI_FRF` |
| 7 | `ssi1_ssi_sleep` | RO | spi1 对外 sleep 信号 |
| 9:8 | `ssi1_spi_mode` | RO | 动态反映 spi1 `SPI_FRF` |
| 10 | `ssi2_ssi_sleep` | RO | spi2 对外 sleep 信号 |
| 12:11 | `ssi2_spi_mode` | RO | 动态反映 spi2 `SPI_FRF` |
| 13 | `rxds_sampling_edge` | R/W | 保存读回，不产生精确 RXDS 时序 |
| 17:14 | `rxds_delay_num` | R/W | 保存读回，不产生精确延迟行为 |
| 31:18 | 保留 | R | 读 0，写忽略 |

复位值 `0x00004000` 表示 `rxds_delay_num=1`，其余可见字段均为 0。
其中 `implemented_mask=0x0003fff1` 表示所有非保留字段均可读；这不代表它们都可写。
实际写入只能改变 `writable_mask=0x0003e001` 中的 RXDS 配置位和 `ssi0_xip_en`。

## TRM 与 SDK：为什么需要裁决

TRM 的 `SSI_CTRL` 表同时出现两个无法同时成立的描述：它把 `SSI_CTRL` 列为
offset `0x068`，而同一份 DWC SSI 寄存器表又把 `DR0–DR35` 覆盖为
`0x060–0x0ec`，其中 `0x068` 正好是 `DR2`。TRM 位表还把
`rxds_delay_num` 和 `ssi2_spi_mode` 标成了重叠范围。

SDK 的 U-Boot `drivers/spi/designware_spi.c` 给出了两个决定性证据：

```c
#define SSI_CTRL (0x91585000UL + 0x68)

typedef struct {
    uint32_t ssi0_xip_en:1;
    uint32_t rsvd0:3;
    uint32_t ssi0_ssi_sleep:1;
    uint32_t ssi0_spi_mode:2;
    uint32_t ssi1_ssi_sleep:1;
    uint32_t ssi1_spi_mode:2;
    uint32_t ssi2_ssi_sleep:1;
    uint32_t ssi2_spi_mode:2;
    uint32_t rxds_sampling_edge:1;
    uint32_t rxds_delay_num:4;
    uint32_t rsvd1:14;
} ssi_ctrl_t;
```

因此本 Patch 的裁决是：

| 问题 | 裁决 | 原因 |
|---|---|---|
| `SSI_CTRL` 的实际地址 | `0x91585068` | U-Boot 使用 HI_SYS 的绝对地址定义 |
| `spiN_base+0x068` | 始终为 `DR2` | SDK 将 `0x060–0x0ec` 作为 36 个 DR FIFO 别名 |
| RXDS 字段范围 | edge=`bit13`，delay=`bits17:14` | U-Boot 位域无重叠且与 reset `0x4000` 自洽 |
| mode/sleep 字段 | `bits12:4` 中的只读状态位 | U-Boot 给出三个实例的完整布局 |

TRM 仍然提供字段意图和复位语义；SDK 则用于裁决 K230 这颗 SoC 的实际集成地址和
无重叠位图。不能因为 TRM 的汇总表使用 offset `0x068`，就把 HI_SYS 寄存器塞进
任一 DWC SSI 控制器。

## 如何理解这一个寄存器

`SSI_CTRL` 是 SoC 包装层的“观察口 + 少量总控位”，不是第四个 SSI 控制器：

```text
                 可写：RXDS 配置、ssi0_xip_en
Guest <----> HI_SYS.SSI_CTRL
                    ^
                    |
      只读汇总：spi0 / spi1 / spi2 的 mode、sleep
```

### mode：控制器当前采用哪种线宽格式

`ssiN_spi_mode` 直接反映对应实例的 `CTRLR0.SPI_FRF[1:0]`：

| 值 | 含义 |
|---:|---|
| 0 | Standard SPI |
| 1 | Dual SPI |
| 2 | Quad SPI |
| 3 | Octal SPI |

它是只读状态，不是另一个配置入口。Guest 应先在对应 SSI 的 `CTRLR0` 中配置格式，
然后才能从 HI_SYS 看到 mode 改变。P8 不因 mode=3 自动实现 Octal、DDR 或 RXDS。

### sleep：SoC 是否可关闭该 SSI 时钟

TRM 对 `SSIENR` 的描述是：控制器禁用后停止传输、清 FIFO，随后通过
`ssi_sleep` 通知系统可以关闭 `ssi_clk`。在功能级模型中应保持以下可观察契约：

```text
reset                    -> sleep=0
SSIENR=1                 -> sleep=0
SSIENR=0 且 BUSY 已清除  -> sleep=1
```

真机中的 sleep 延迟长度属于硬件时序；P8 不需要模拟精确时钟周期。当前同步 pump
中 disable 会同步 abort，因此可以在 abort 完成后得到稳定 sleep 状态。未来若引入
异步传输 engine，sleep 必须等待 `BUSY=0` 和包装层延迟完成。

### RXDS：保留软件配置，不伪造物理时序

`rxds_sampling_edge` 与 `rxds_delay_num` 服务于 DDR/HyperBus/OPI 等场景的
读取采样。P8 的唯一要求是可写、可读回和可迁移：

```text
Guest 写入 -> HI_SYS 保存 mask 内的值 -> Guest 读取相同值
```

这些字段不得改变当前 Standard SPI 或 Dual/Quad SDR 的 `ssi_transfer()` 结果；
RXDS 精确采样边沿和延迟不是本系列的模拟目标。

### `ssi0_xip_en`：为 P9 准备的门锁

bit 0 只针对 spi0：

```text
P8: 保存并通过窄接口导出 ssi0_xip_en
P9: 读取 0xc0000000–0xc7ffffff 时检查该状态
    0 -> 不返回 Flash 数据
    1 -> 使用 XIP 配置发起 Flash read
```

它不应影响普通 PIO SPI/QSPI 传输，也不应在 P8 提前创建 XIP MemoryRegion、发送
opcode 或伪造 Flash 数据。

## Guest 可观察流程

一次典型配置可理解为：

```text
1. Guest 读 0x91585068，看到 reset=0x00004000
2. Guest 写 bit0 / RXDS 配置位；保留位和 mode/sleep 写入被忽略
3. Guest 配置某个 spiN.CTRLR0.SPI_FRF
4. Guest 再读 HI_SYS，看到对应 spiN_spi_mode 动态变化
5. Guest enable spiN，sleep 保持 0
6. Guest disable spiN，传输停止、FIFO 清空；空闲后 sleep 变为 1
```

Patch 8 需要保证以下两个地址空间互不影响：

```text
write 0x91585068     -> 修改 HI_SYS 包装层配置
write spiN_base+0x68 -> 向该 spiN 的 DR2/TX FIFO 写入数据
```

这正是 H04 `hi-sys/dr2-independent` 的意义。

## 代码脚手架

已添加：

```text
include/hw/misc/k230_hi_sys.h
    QOM 类型、MMIO 大小、SSI_CTRL 位图和状态结构

hw/misc/k230_hi_sys.c
    SysBus MMIO 骨架
    reset = 0x00004000
    implemented/writable mask
    VMState
    动态 SSI mode/sleep 组合和 XIP enable 查询接口

hw/misc/meson.build
    CONFIG_K230 下编译 k230_hi_sys.c
```

machine 已实例化 `K230HiSysState`，替换原 `hi_sys_cfg` 占位区域，并通过
统一的 SoC SSI 拓扑表连接逻辑 `spi0/spi1/spi2`。

## 实施步骤

### P8-1：固定寄存器模型

1. 将 `hi_sys_cfg` 占位区域替换为 `TYPE_K230_HI_SYS`。
2. 保持 MMIO 窗口 `0x91585000–0x915853ff`，只在 `+0x68` 提供 `SSI_CTRL`。
3. 完成 reset、implemented mask、writable mask 和 RAZ/WI。
4. 不影响任一 `spiN_base + 0x068` 的 DR2 访问。

### P8-2：动态状态接口

通过窄接口读取三个 `K230DwSsiState`：

```text
spi0 = dw_ssi[2]
spi1 = dw_ssi[0]
spi2 = dw_ssi[1]
```

只读字段从 SSI 当前状态组合得到：

- mode：`k230_dw_ssi_get_spi_mode()` 返回 `CTRLR0.SPI_FRF[1:0]`
- sleep：`k230_dw_ssi_is_sleeping()` 返回控制器对外 sleep 信号

不要让 HI_SYS 直接访问 `regs[]` 或 machine 全局变量，也不要复制一份 SSI
寄存器数组。

### P8-3：XIP gate 接口

P8 通过只读查询接口导出 bit 0：

```c
bool k230_hi_sys_xip_enabled(const K230HiSysState *s);
```

P9 的 spi0 XIP window 消费该接口：

```text
ssi0_xip_en = 0 → XIP window 不响应 Flash 数据
ssi0_xip_en = 1 → P9 的只读 XIP window 可响应
```

P8 不实现 XIP 地址窗口、opcode、dummy 或 Flash 事务；这些属于 P9。

## qtest 门禁

现有测试入口：

```text
tests/qtest/k230-dw-ssi-hi-sys-test.c
    /k230-dw-ssi/hi-sys/reset-mask
    /k230-dw-ssi/hi-sys/mode-status
    /k230-dw-ssi/hi-sys/sleep-status
    /k230-dw-ssi/hi-sys/dr2-independent
```

证明范围：

| 用例 | 证明内容 |
|---|---|
| H01 | reset 值和 writable/implemented mask |
| H02 | 三个 SSI mode 状态动态映射 |
| H03 | sleep 状态随 enable/idle 变化 |
| H04 | HI_SYS `SSI_CTRL` 与三个 SSI 的 DR2 地址独立 |

P8 验证命令：

```bash
ninja -C "build" \
    "qemu-system-riscv64" \
    "tests/qtest/k230-dw-ssi-test"

QTEST_QEMU_BINARY="./build/qemu-system-riscv64" \
    ./build/tests/qtest/k230-dw-ssi-test \
    -p /k230-dw-ssi/hi-sys
```

P8 收口标准：

- [ ] H01–H04 全部通过。
- [ ] P7 的 IRQ/PLIC 门禁无回归。
- [ ] `SSI_CTRL` 不再由 `create_unimplemented_device()` 覆盖。
- [ ] `spiN_base + 0x068` 仍按 DR2 工作。
- [ ] HI_SYS 不持有 machine 全局变量，不复制 SSI 状态数组。
- [ ] bit 0 已通过窄接口供 P9 使用，但 P8 不实现 XIP window。

## 当前验证预期

H01–H04 用于确认寄存器、动态状态和地址独立性。不得为了让 qtest 变绿而把
`SSI_CTRL` 错映射到 SSI 内部 MMIO，或在 HI_SYS 中伪造 PLIC/XIP 状态。

达到上述条件后，进入 Patch 9 的只读 XIP window。
