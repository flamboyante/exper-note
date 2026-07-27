# K230 SSI Patch 7：PLIC 接线学习工作簿

更新时间：2026-07-18

当前状态：Patch 6 的控制器内部九路 IRQ 已完成，reg 8、PIO 10、Standard
Flash 7、QSPI 6 和内部 IRQ 6 共 37 项测试已经通过。Patch 7 开始搭建
K230 SoC 到 PLIC 的 27 路接线脚手架；路由表和五项 qtest 已准备，实际
`sysbus_connect_irq()` 仍保持 RED，等待按 P7-1 → P7-2 完成。

## 目标

Patch 7 不再修改 SSI 中断算法，只把三个控制器各自的九路输出接到 K230 PLIC：

```text
logical spi0 = dw_ssi[2] -- IRQ output[0..8] --> PLIC 146..154
logical spi1 = dw_ssi[0] -- IRQ output[0..8] --> PLIC 155..163
logical spi2 = dw_ssi[1] -- IRQ output[0..8] --> PLIC 164..172
```

建议提交标题：

```text
hw/riscv/k230: Connect K230 SSI interrupts to the PLIC
```

Patch 7 完成后的累计门禁：

| 分组 | 数量 | 累计 |
|---|---:|---:|
| reg | 8 | 8 |
| pio | 10 | 18 |
| flash | 7 | 25 |
| qspi | 6 | 31 |
| irq/internal | 6 | 37 |
| irq/plic | 5 | 42 |

## 本 Patch 做与不做

必须完成：

- 在 K230 SoC 层定义 spi0/spi1/spi2 的 PLIC IRQ base。
- 使用显式逻辑实例路由表，不能从 `dw_ssi[]` 下标或 MMIO 地址推导。
- 三个实例各连接九路 SysBus IRQ，共 27 路。
- IRQ output index 直接复用 Patch 6 的固定顺序。
- 所有 source ID 都在 PLIC 的 208 路范围内。
- 注册 I07–I11，并保持 Patch 2–6 的 37 项测试无回归。

本 Patch 不实现：

- 不修改 `RISR/ISR/IMR`、水位、锁存和 RC 清除算法。
- 不在 SSI 控制器中加入 PLIC source ID。
- 不用 OR gate 聚合九路中断。
- 不修改 PLIC 设备模型、优先级、enable 或 claim/complete 语义。
- 不实现 HI_SYS、XIP、IDMA、DONE/AXIE 的新事件来源。
- 不重排 `dw_ssi[]`，避免扩大 Patch 范围。

## 已搭好的学习脚手架

生产代码目标：

```text
include/hw/riscv/k230.h
    K230_SPI0_IRQ_BASE = 146
    K230_SPI1_IRQ_BASE = 155
    K230_SPI2_IRQ_BASE = 164

hw/riscv/k230.c
    K230SsiIrqRoute
    k230_ssi_irq_routes[]
    k230_connect_ssi_irqs()
    TODO(P7-1)：九路逐一 sysbus_connect_irq()
```

测试入口：

```text
tests/qtest/k230-dw-ssi-plic-test.c
    I07–I11

tests/qtest/k230-dw-ssi-test.c
    k230_ssi_register_plic_tests()
```

定位脚手架：

```bash
rg -n "TODO\(P7|LEARNING\(P7" \
    "hw/riscv" "include/hw/riscv" "tests/qtest"
```

## 1. 最危险的实例编号陷阱

当前 `dw_ssi[]` 数组沿用 QEMU 初始设备创建顺序，而 SDK 的逻辑 spi 编号按
外设用途命名，两者不一致：

| 逻辑实例 | `dw_ssi[]` | QOM child | MMIO | profile | IRQ base |
|---|---:|---|---:|---|---:|
| spi0 | 2 | `k230-spi-opi` | `0x91584000` | 1 CS / 8 lines | 146 |
| spi1 | 0 | `k230-qspi0` | `0x91582000` | 5 CS / 4 lines | 155 |
| spi2 | 1 | `k230-qspi1` | `0x91583000` | 5 CS / 4 lines | 164 |

因此下面的实现是错误的：

```c
for (int i = 0; i < ARRAY_SIZE(s->dw_ssi); i++) {
    irq_base = K230_SPI0_IRQ_BASE + i * K230_DW_SSI_IRQ_COUNT;
    connect(&s->dw_ssi[i], irq_base);
}
```

它会把：

```text
dw_ssi[0] 错接到 spi0 146
dw_ssi[1] 错接到 spi1 155
dw_ssi[2] 错接到 spi2 164
```

而正确映射必须显式写出：

```c
static const K230SsiIrqRoute k230_ssi_irq_routes[] = {
    { .ssi_index = 2, .irq_base = K230_SPI0_IRQ_BASE },
    { .ssi_index = 0, .irq_base = K230_SPI1_IRQ_BASE },
    { .ssi_index = 1, .irq_base = K230_SPI2_IRQ_BASE },
};
```

这个三项表不是多余抽象，而是防止逻辑编号、数组编号和地址排序再次混淆的唯一事实源。

## 2. 九路输出顺序

Patch 6 已固定 SysBus IRQ output index：

| output index | 名称 | spi0 | spi1 | spi2 |
|---:|---|---:|---:|---:|
| 0 | TXE | 146 | 155 | 164 |
| 1 | TXO | 147 | 156 | 165 |
| 2 | RXF | 148 | 157 | 166 |
| 3 | RXO | 149 | 158 | 167 |
| 4 | TXU | 150 | 159 | 168 |
| 5 | RXU | 151 | 160 | 169 |
| 6 | MST | 152 | 161 | 170 |
| 7 | DONE | 153 | 162 | 171 |
| 8 | AXIE | 154 | 163 | 172 |

注意：output index 不是 RISR bit。Patch 6 已在控制器内部完成映射，Patch 7 只处理
`output index -> PLIC source`，不能再次按 RISR 位号重排。

## 3. P7-1：IRQ base 与路由表

IRQ base 应定义在 K230 SoC 头文件，而不是控制器头文件：

```c
K230_SPI0_IRQ_BASE = 146,
K230_SPI1_IRQ_BASE = 155,
K230_SPI2_IRQ_BASE = 164,
```

原因：

- 146–172 是 K230 SoC 集成结果，不是 DWC SSI IP 自身属性。
- 同一个 SSI 控制器模型可以被其他 machine 复用。
- Patch 6 的控制器不应依赖 K230 PLIC。

路由表至少保存：

```text
ssi_index
irq_base
```

不需要把 MMIO、num-cs、max-lines 再复制一遍。那些信息已经由现有 machine 配置负责；
重复保存会形成两个可能漂移的配置源，违反 DRY。

脚手架中的边界断言：

```c
g_assert(route->ssi_index < ARRAY_SIZE(s->dw_ssi));
g_assert(route->irq_base + K230_DW_SSI_IRQ_COUNT <=
         K230_PLIC_NUM_SOURCES);
```

第二个条件使用开区间上界：spi2 base 164 加 9 等于 173，实际最后一路是 172，
仍小于 208。

## 4. P7-2：逐路接线

推荐实现：

```c
static void k230_connect_ssi_irqs(K230SoCState *s)
{
    for (int route_index = 0;
         route_index < ARRAY_SIZE(k230_ssi_irq_routes);
         route_index++) {
        const K230SsiIrqRoute *route = &k230_ssi_irq_routes[route_index];
        SysBusDevice *ssi = SYS_BUS_DEVICE(&s->dw_ssi[route->ssi_index]);

        for (int irq = 0; irq < K230_DW_SSI_IRQ_COUNT; irq++) {
            sysbus_connect_irq(
                ssi, irq,
                qdev_get_gpio_in(DEVICE(s->c908_plic),
                                 route->irq_base + irq));
        }
    }
}
```

调用时机：

```text
创建 PLIC
  -> 设置三个 SSI profile
  -> realize 三个 SSI
  -> 连接 27 路 IRQ
  -> map 三个 MMIO
```

PLIC 必须先创建，才能取得 PLIC input GPIO。
SSI 的九路 SysBus output 在 instance_init 阶段已经由 sysbus_init_irq() 声明，
连接本身不要求 SSI realize 之后。由于 SSI 复位过程中可能早于 IRQ 连接产生
初始 TXE，控制器在 reset exit 阶段必须重新调用 update_irq()，将当前 IRQ
状态重新驱动到 PLIC。

MMIO map 与 IRQ connect 没有数据依赖，但建议保持“realize → connect → map”的清晰顺序。

## 5. 为什么禁止 OR gate

如果把九路中断 OR 成一根线：

- PLIC 只能看到一个 source，无法符合 DTS 的九路编号。
- TXE、RXF、RXU 等原因无法拥有独立 pending bit。
- spi0/spi1/spi2 的实例隔离无法验证。
- DONE/AXIE 等未来事件无法保持既定 ABI。

PLIC 本身已经负责多个 source 的仲裁，machine 不需要再次聚合。

## 6. reset 电平与 PLIC pending

Patch 6 的 IMR 复位值是 `0x3f`。空 TX FIFO 满足 TXE 水位，因此 reset 后三个
控制器的 TXE 输出都应为高：

```text
PLIC pending[146] = 1
PLIC pending[155] = 1
PLIC pending[164] = 1
```

DONE/AXIE 无事件来源，应保持：

```text
pending[153/154] = 0
pending[162/163] = 0
pending[171/172] = 0
```

I07 正是为了发现“线路连接成功但 reset 后未重新驱动”的问题。Patch 6 已在 reset 和
post-load 中调用统一 `update_irq()`，Patch 7 不应通过 machine 私自拉高 TXE 来补救。

## 7. 五项 PLIC qtest

| 编号 | 用例 | 证明范围 |
|---|---|---|
| I07 | `irq/plic-txe-reset-routing` | 三实例 TXE reset pending，DONE/AXIE inactive |
| I08 | `irq/plic-rxu-isolation` | RXU 到正确实例，其他实例同类 source 不受影响 |
| I09 | `irq/plic-rxf-routing` | 动态 RXF 的三实例 base 和 output index |
| I10 | `irq/plic-txo-routing` | 锁存 TXO 的三实例路由 |
| I11 | `irq/plic-rxo-routing` | 锁存 RXO 的三实例路由 |

测试直接读取 PLIC pending register，不需要设置 priority、enable 或 claim/complete。
这里验证的是外设输入线是否到达指定 source；PLIC 的 CPU 投递流程已经由现有 PLIC
模型负责，不属于 Patch 7。

### RED 预期

脚手架阶段已注册 I07–I11，但尚未调用 `sysbus_connect_irq()`：

```text
reg/pio/flash/qspi/irq-internal 37 项继续 PASS
plic 5 项 FAIL
无 TIMEOUT
```

如果内部 IRQ 用例也失败，说明 Patch 7 修改越过了 machine 接线边界，应先回退检查，
不要在控制器算法中为 PLIC 测试添加特判。

## 8. 推荐实现顺序

### P7-1：固定拓扑

1. 核对三个 IRQ base。
2. 核对 `{2, 0, 1}` 实例映射。
3. 核对每组九路、最后 source 172 未越界。
4. 运行测试列表，确认累计 42 项且 PLIC 五项 RED。

### P7-2：连接线路

1. 在双层循环中逐路 `sysbus_connect_irq()`。
2. 先通过 I07 reset TXE。
3. 再通过 I08 实例隔离。
4. 最后通过 I09–I11 动态/锁存事件路由。
5. 运行 42 项累计回归。

## 9. 验证命令

```bash
ninja -C "build" \
    "qemu-system-riscv64" \
    "tests/qtest/k230-dw-ssi-test"

QTEST_QEMU_BINARY="./build/qemu-system-riscv64" \
    ./build/tests/qtest/k230-dw-ssi-test \
    -p /riscv64/k230-dw-ssi/irq/plic
```

逐项：

```bash
bash "tests/qtest/run-k230-dw-ssi-tests.sh" \
    "irq/plic-txe-reset-routing"
```

累计门禁：

```text
reg       8/8
pio      10/10
flash     7/7
qspi      6/6
irq      11/11
累计     42/42
```

## 10. 完成标准

- [ ] P7-1 IRQ base 和逻辑实例路由表完成。
- [ ] P7-2 27 路逐一接入 PLIC。
- [ ] I07–I11 全部通过。
- [ ] Patch 2–6 的 37 项测试持续通过。
- [ ] 累计 42/42 PASS，无 TIMEOUT。
- [ ] 控制器源码无 PLIC source ID。
- [ ] machine 未使用 OR gate。
- [ ] `dw_ssi[]` 未因本 Patch 重排。
- [ ] spi0/spi1/spi2 无实例错位、source 重复或越界。
- [ ] Patch 7 未修改 SSI 事件算法。

达到以上条件后，可以进入 Patch 8 的 HI_SYS `SSI_CTRL`。
