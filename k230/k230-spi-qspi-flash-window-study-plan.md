# K230 SPI/QSPI QEMU 实施与 Phase 10 qtest 验收

更新时间：2026-07-14

## 文档定位

本文是 K230 SPI/QSPI 模型的实施入口和测试记录表，回答五个问题：

```text
当前 10-Patch 主系列实现什么？
每个 Patch 由哪些 qtest 验收？
当前模型已经通过哪些用例？
后续写 C 代码时应该先修哪一项？
怎样证明 Phase 10 已完成？
```

寄存器位域、复位值、TRM 5.3/12.3 冲突和 SDK 证据统一查阅：

- [K230 TRM 12.3 SPI 中文学习版（含 5.3 FMC 对照）](k230-trm-12.3-spi-cn.md)

行为裁决优先级固定为：

```text
K230 TRM + K230 SDK 有效代码/设备树
  > 已完成的交叉裁决
  > QEMU 同类设备实现习惯
  > 当前 k230_dw_ssi.c 原型行为
```

当前模型只能用于判断“还缺什么”，不能反向定义测试预期。

## 1. 本轮能力边界

### 1.1 必须完成

- 三个 K230 SSI 实例及正确 profile。
- 4 KiB MMIO、复位值、writable mask、RO、RC 和 RAZ/WI。
- 256 项、每项最长 32 位的 TX/RX FIFO。
- Standard SPI 的 TR、TO、RO、EEPROM_READ PIO。
- Dual/Quad QSPI 的指令、地址、mode、dummy 和数据阶段。
- 九路独立中断及 PLIC 146–172 连线。
- spi0 CS0 的 W25Q256 MTD 后端。
- 独立 `HI_SYS.SSI_CTRL @ 0x91585068`。
- 受 `ssi0_xip_en` 门控的 `0xC0000000–0xC7ffffff` XIP window。

### 1.2 本轮明确不实现

- 内部 AXI DMA guest memory 搬运。
- Octal/OPI 数据事务。
- DDR、DTR、RXDS 精确时序。
- concurrent XIP、prefetch、XIP write。
- BootROM 到 U-Boot/Linux 的完整冷启动 functional test。

不实现不等于静默降级：

- DMA 配置寄存器必须按 SDK 布局保存读回，DONE/AXIE 不得虚假触发。
- Octal/DDR/RXDS 组合必须拒绝启动，不能悄悄按 Standard SPI 发送。
- XIP write 必须保持只读窗口语义。

## 2. K230 实例和固定地址

SDK 逻辑编号与 QEMU `dw_ssi[]` 数组顺序不同，测试始终使用 SDK 逻辑编号：

| SDK 实例 | 控制器基址 | `num-cs` | 最大线宽 | `SPI_CTRLR0` 复位 | PLIC 范围 | XIP |
|---|---:|---:|---:|---:|---:|---|
| `spi0` | `0x91584000` | 1 | 8 | `0x28000200` | 146–154 | 有 |
| `spi1` | `0x91582000` | 5 | 4 | `0x04000200` | 155–163 | 无 |
| `spi2` | `0x91583000` | 5 | 4 | `0x04000200` | 164–172 | 无 |

其他地址：

| 对象 | 地址 | 说明 |
|---|---:|---|
| `HI_SYS_CONFIG` | `0x91585000–0x915853ff` | SoC 包装寄存器块 |
| `SSI_CTRL` | `0x91585068` | 不属于任一 SSI 内部 MMIO |
| `DR2` | `spiN_base + 0x068` | 三个控制器的第三个 FIFO 数据口 |
| Flash window | `0xC0000000–0xC7ffffff` | 只属于 spi0，且受 bit0 门控 |
| PLIC | `0xF00000000` | pending 从 `+0x1000` 开始 |

九路输出顺序固定为：

| 相对序号 | 外部 IRQ | `spi0` | `spi1` | `spi2` |
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

注意：该顺序不是 `RISR` 的 bit 顺序，machine 接线和 qtest 都必须显式映射。

## 3. 10-Patch 主系列

| Patch | 单一职责 | qtest 组 | 完成条件 |
|---:|---|---|---|
| 1 | 寄存器模型和 QOM 框架 | 编译门禁 | 设备可独立编译，未接 machine |
| 2 | 三个实例、地址和 profile | `reg/*` | 地址、复位、mask、CS 数量和保留区正确 |
| 3 | Fifo32、Standard PIO、TMOD | `pio/*` | 256 项 FIFO 和四种 TMOD 全部通过 |
| 4 | 内部九路中断 | `irq/*` 内部状态 | 水位、错误锁存、RC 清除正确 |
| 5 | PLIC 146–172 接线 | `irq/plic-*` | 路由和实例隔离正确 |
| 6 | spi0 W25Q256 | `flash/*` | JEDEC、读、写、擦除和 CS 重启通过 |
| 7 | Dual/Quad QSPI | `qspi/*` | 阶段格式正确，不支持模式明确拒绝 |
| 8 | HI_SYS `SSI_CTRL` | `hi-sys/*` | 地址、mask、mode/sleep 和 DR2 隔离正确 |
| 9 | 受控 XIP window | `xip/*` | 门控、宽度、地址、mode/dummy、CS 协调通过 |
| 10 | 上游英文文档 | 文档构建 + 本表 | 文档只声明前九个 Patch 已验证能力 |

上游 Patch 10 仍然是文档 Patch。本文的“Phase 10”是本地最终验收阶段，不改变上游 Patch 编号和单一职责。

## 4. qtest 源码结构

所有源码最终链接为一个测试程序：

```text
tests/qtest/k230-dw-ssi-test
```

| 文件 | 职责 |
|---|---|
| `k230-dw-ssi-test.c` | 只注册五个能力组 |
| `k230-dw-ssi-test.h` | SDK/TRM 常量、实例表、helper 接口 |
| `k230-dw-ssi-test-common.c` | QEMU 启动、Flash 镜像、PIO、轮询、PLIC helper |
| `k230-dw-ssi-reg-test.c` | 实例、复位、mask、RO/RC/RAZ-WI、DMA 兼容层 |
| `k230-dw-ssi-pio-test.c` | DR 别名、Fifo32、DFS、TMOD、动态状态 |
| `k230-dw-ssi-irq-test.c` | RISR/ISR/IMR、错误锁存、RC、PLIC |
| `k230-dw-ssi-flash-test.c` | W25Q256 和 Dual/Quad QSPI |
| `k230-dw-ssi-xip-test.c` | HI_SYS、DR2 隔离、XIP window |

测试代码可以使用较多中文注释，但注释必须解释证据或硬件语义。例如：

```text
好：IMR 复位为 0x3f，空 TX FIFO 满足 TXE 水位，所以复位后 TXE 线可拉高。
差：读取寄存器，然后判断寄存器。
```

运行时测试名保持英文和稳定路径，便于 Meson、CI 和失败日志检索。

## 5. Phase 10：完整 qtest 验收

### 5.1 Phase 10 目标

Phase 10 不是“再补几个测试”，而是把前九个功能 Patch 合并成一份最终可执行规格：

```text
SDK/TRM 行为裁决
  -> 46 个独立 qtest
  -> 每个失败对应一个明确能力缺口
  -> 后续 C 实现逐项 RED -> GREEN
  -> 46/46 PASS 后才能声明 Phase 10 完成
```

### 5.2 当前实测汇总

2026-07-14 在 `spi` 分支重新生成 Meson、编译测试并逐项执行：

| 指标 | 结果 |
|---|---:|
| 注册用例 | 46 |
| PASS | 16 |
| FAIL | 30 |
| TIMEOUT | 0 |
| 测试编译 | PASS |
| 完整功能验收 | 未完成 |

各组统计：

| 组 | PASS | FAIL | 当前结论 |
|---|---:|---:|---|
| `reg` | 7 | 1 | 基础寄存器已接近冻结，`spi1/spi2 num-cs=5` 未实现 |
| `pio` | 0 | 9 | TX FIFO 尚未进入真实路径，TMOD/DFS/Fifo32 未实现 |
| `irq` | 2 | 9 | 只有 TXE/RXF 简化水位，错误锁存、RC 和 PLIC 未完成 |
| `flash` | 6 | 0 | 当前字节直传已能完成基础 W25Q256 命令 |
| `qspi` | 1 | 2 | 仅配置读回存在，增强事务和拒绝策略未完成 |
| `hi-sys` | 0 | 4 | `SSI_CTRL` 仍是未实现设备 |
| `xip` | 0 | 5 | 当前窗口始终可读，尚无 HI_SYS 门控和寄存器驱动阶段 |

### 5.3 完整用例矩阵

状态列是 2026-07-14 当前源码实测结果；最终目标全部为 PASS。

| ID | 测试路径 | 验证重点 | 依据 | Patch | 当前 |
|---:|---|---|---|---:|---|
| R01 | `reg/reset-values` | CTRLR0、IMR、AXILEN、ID、SR 复位 | TRM/RT/SDK | 2 | PASS |
| R02 | `reg/profile-reset-values` | FMC 与 QSPI 的 `SPI_CTRLR0` 差异 | TRM 5.3/12.3 | 2 | PASS |
| R03 | `reg/write-masks` | 全部已裁决 RW mask | TRM 寄存器表、U-Boot 探测 | 2 | PASS |
| R04 | `reg/ser-num-cs` | spi0=1，spi1/spi2=5 | U-Boot `k230.dtsi` | 2 | FAIL |
| R05 | `reg/read-only-and-razwi` | 动态/固定 RO 与条件保留区 | TRM/SDK profile | 2 | PASS |
| R06 | `reg/enabled-write-lock` | 使能期配置寄存器写入忽略 | TRM `SSIENR` 约束 | 2 | PASS |
| R07 | `reg/internal-dma-passive` | DMA 寄存器可见但无虚假 DONE/AXIE | SDK `SSIC_HAS_DMA=2` | 2 | PASS |
| R08 | `reg/system-reset` | 系统复位恢复 profile 契约 | QEMU reset + TRM | 2 | PASS |
| P01 | `pio/dr-aliases` | DR0–DR35 共用一个 TX FIFO | RT `dr[36]` | 3 | FAIL |
| P02 | `pio/fifo-depth-256` | 256 项 Fifo32、TFNF、TXO | TRM/RT FIFO 深度 | 3 | FAIL |
| P03 | `pio/dfs-frame-mask` | 4/8/16/32 位帧截断 | TRM DFS | 3 | FAIL |
| P04 | `pio/tmod-tr` | 同时收发 | TRM TMOD | 3 | FAIL |
| P05 | `pio/tmod-to` | TX-only 不写无效 RX 数据 | TRM TMOD | 3 | FAIL |
| P06 | `pio/tmod-ro` | RX-only 按 NDF+1 产生 dummy clock | TRM/U-Boot | 3 | FAIL |
| P07 | `pio/tmod-eeprom-read` | 控制阶段后独立接收 NDF+1 | TRM/U-Boot | 3 | FAIL |
| P08 | `pio/disable-clears-fifo` | 禁用停止事务、撤销 CS、清 FIFO | TRM `SSIENR` | 3 | FAIL |
| P09 | `pio/dynamic-watermark` | TXE 随 FIFO 水位实时变化 | TRM TXFTLR/RISR | 3 | FAIL |
| I01 | `irq/watermark-mask` | `ISR = RISR & IMR` | TRM | 4 | PASS |
| I02 | `irq/rxu-read-clear` | 空读锁存 RXU，RXUICR 读取清除 | TRM RC | 4 | FAIL |
| I03 | `irq/txo-read-clear` | 满写锁存 TXO，TXEICR 读取清除 | TRM RC | 4 | FAIL |
| I04 | `irq/rxo-read-clear` | RX 满后锁存 RXO，RXOICR 读取清除 | TRM RC | 4 | FAIL |
| I05 | `irq/icr-clear-scope` | ICR 只清规定错误，不影响 DMA 事件 | TRM RC | 4 | FAIL |
| I06 | `irq/inactive-causes` | 无原因时 MST/TXU/DONE/AXIE 为 0 | TRM + 本轮边界 | 4 | PASS |
| I07 | `irq/plic-txe-reset-routing` | 三个 TXE 起始号和 DONE/AXIE 静默 | DTS/TRM IRQ | 5 | FAIL |
| I08 | `irq/plic-rxu-isolation` | RXU 路由和实例隔离 | DTS/TRM IRQ | 5 | FAIL |
| I09 | `irq/plic-rxf-routing` | RXF 相对序号 2 | DTS/TRM IRQ | 5 | FAIL |
| I10 | `irq/plic-txo-routing` | TXO 相对序号 1 | DTS/TRM IRQ | 5 | FAIL |
| I11 | `irq/plic-rxo-routing` | RXO 相对序号 3 | DTS/TRM IRQ | 5 | FAIL |
| F01 | `flash/jedec-id` | W25Q256 `ef 40 19` | m25p80 + SDK 板级连接 | 6 | PASS |
| F02 | `flash/read-3byte` | `0x03` 三字节地址读取 | SPI NOR 协议 | 6 | PASS |
| F03 | `flash/read-4byte` | `0x13` 四字节地址读取 | SPI NOR 协议 | 6 | PASS |
| F04 | `flash/page-program` | WREN、PP、busy、readback | SPI NOR 协议 | 6 | PASS |
| F05 | `flash/sector-erase` | WREN、4 KiB erase、readback | SPI NOR 协议 | 6 | PASS |
| F06 | `flash/cs-restarts-command` | CS 撤销后命令状态重新开始 | SSI/SPI NOR | 6 | PASS |
| Q01 | `qspi/quad-output-read` | Quad 指令/地址/dummy/data 阶段 | TRM/U-Boot exec_op | 7 | FAIL |
| Q02 | `qspi/mode-bits-dummy` | `XIP_MD_BIT_EN` 和 WAIT 字段读回 | TRM/SPI_CTRLR0 | 7 | PASS |
| Q03 | `qspi/unsupported-octal-ddr-rxds` | 不支持组合拒绝，不产生伪 RX | 本轮明确边界 | 7 | FAIL |
| H01 | `hi-sys/reset-mask` | reset=`0x4000`，mask=`0x3e001` | U-Boot `ssi_ctrl_t` | 8 | FAIL |
| H02 | `hi-sys/mode-status` | 三实例 mode 动态反映 SPI_FRF | TRM/U-Boot | 8 | FAIL |
| H03 | `hi-sys/sleep-status` | enable 清零，disable+idle 后置位 | TRM `ssi_sleep` | 8 | FAIL |
| H04 | `hi-sys/dr2-independent` | `0x91585068` 与三个 DR2 隔离 | SDK 地址裁决 | 8 | FAIL |
| X01 | `xip/enable-gate` | bit0=0 返回 0，bit0=1 才访问 Flash | TRM/U-Boot | 9 | FAIL |
| X02 | `xip/read-widths` | 1/2/4/8 字节 little-endian 读取 | QEMU MMIO + TRM | 9 | FAIL |
| X03 | `xip/address-width` | 地址长度来自 SPI_CTRLR0，不按 opcode 猜 | TRM | 9 | FAIL |
| X04 | `xip/mode-bits-dummy` | mode/dummy 来自配置寄存器 | TRM 5.3/12.3 | 9 | FAIL |
| X05 | `xip/pio-coordination` | PIO/XIP 共用 Flash 和 CS，无陈旧事务 | TRM/QEMU SSI | 9 | FAIL |

### 5.4 qtest 无法单独证明的内容

| 内容 | 原因 | 验证方式 |
|---|---|---|
| 未触发状态下 DONE/AXIE GPIO 是否接到正确 PLIC ID | 本轮没有 DMA 状态机，黑盒 guest 无法主动拉高这两根线 | machine 代码审查；后续 DMA 系列产生真实事件后补 qtest |
| MST 的真实多主机争用 | 当前 board 没有第二主机或外部争用输入 | 保持无原因时为 0；后续若建模争用输入再补 |
| IO0–IO7 逐周期线宽和 DDR 边沿 | QEMU `SSIBus` 是事务级接口 | 只验证软件可见事务阶段，不声称波形正确 |
| 真机 `ssi_sleep` 延迟周期 | TRM 未给出已裁决的精确周期 | 测试最终状态，不锁死具体纳秒数 |

## 6. 运行方法

### 6.1 编译

```bash
ninja -C "build" "tests/qtest/k230-dw-ssi-test"
```

完成标准：链接成功，无新增编译错误。

### 6.2 列出全部用例

```bash
QTEST_QEMU_BINARY="./build/qemu-system-riscv64" \
    "./build/tests/qtest/k230-dw-ssi-test" -l
```

预期看到 `1..46`。

### 6.3 跑单项

```bash
QTEST_QEMU_BINARY="./build/qemu-system-riscv64" \
    "./build/tests/qtest/k230-dw-ssi-test" \
    -p "/riscv64/k230-dw-ssi/pio/fifo-depth-256" --tap
```

写 FIFO C 代码时先跑 P02；不要每改一行就跑 46 项。

### 6.4 跑能力组

`-p` 可以接受组路径：

```bash
QTEST_QEMU_BINARY="./build/qemu-system-riscv64" \
    "./build/tests/qtest/k230-dw-ssi-test" \
    -p "/riscv64/k230-dw-ssi/pio" --tap
```

组内第一个断言失败后 GLib 会停止进程。需要统计整组所有结果时，按 `-l` 输出逐项执行。

### 6.5 跑 Meson 完整入口

```bash
./build/pyvenv/bin/meson test -C "build" \
    --no-rebuild --print-errorlogs \
    qtest-riscv64/k230-dw-ssi-test
```

开发期允许红灯；只有 Phase 10 最终验收要求完整入口为绿色。

### 6.6 彩色逐项运行并显示错误原因

人工开发时优先使用彩色 runner。它会先编译 QEMU 和 qtest，再逐项运行；即使某项 assertion 触发 `abort`，后续用例仍会继续执行。

运行全部 46 项：

```bash
bash "tests/qtest/run-k230-dw-ssi-tests.sh"
```

只运行一个能力组：

```bash
bash "tests/qtest/run-k230-dw-ssi-tests.sh" "pio"
bash "tests/qtest/run-k230-dw-ssi-tests.sh" "irq"
bash "tests/qtest/run-k230-dw-ssi-tests.sh" "xip"
```

只运行一个用例：

```bash
bash "tests/qtest/run-k230-dw-ssi-tests.sh" \
    "pio/fifo-depth-256"
```

输出含义：

| 颜色 | 标签 | 含义 |
|---|---|---|
| 绿色 | `PASS` | 用例通过 |
| 红色 | `FAIL` | 用例失败，并显示第一条 assertion/ERROR、expected/got 和复现命令 |
| 红色 | `BUILD FAIL` | 编译失败，并单独提取第一条编译 `error:` |
| 黄色 | `TIMEOUT` | 超过 `QTEST_TIMEOUT` 指定秒数 |
| 青色 | `BUILD/RUN/SUMMARY` | 编译、运行范围和最终统计 |

失败时展开完整原始日志：

```bash
QTEST_VERBOSE=1 \
    bash "tests/qtest/run-k230-dw-ssi-tests.sh" \
    "reg/ser-num-cs"
```

调整单项超时：

```bash
QTEST_TIMEOUT=30 \
    bash "tests/qtest/run-k230-dw-ssi-tests.sh" "xip"
```

禁用 ANSI 颜色，便于重定向到文件：

```bash
K230_QTEST_NO_COLOR=1 \
    bash "tests/qtest/run-k230-dw-ssi-tests.sh" > "k230-ssi-qtest.log"
```

runner 返回值保持可用于自动化判断：全部通过返回 0，存在 FAIL/TIMEOUT 返回 1，编译失败或过滤条件无匹配返回 2。

## 7. 后续 C 实现顺序

当前失败不是随机分布，按依赖顺序修：

| 顺序 | 先跑用例 | 生产代码重点 | 进入下一步条件 |
|---:|---|---|---|
| 1 | R04 | machine 设置 spi1/spi2 `num-cs=5` | `reg` 8/8 PASS |
| 2 | P01/P02/P08 | Fifo32、DR push、禁用清 FIFO | TXFLR 能真实变化 |
| 3 | P03–P07/P09 | DFS、TMOD、传输状态机、水位 | `pio` 9/9 PASS |
| 4 | I02–I05 | 错误锁存和 RC 清除 | 内部 `irq` 状态通过 |
| 5 | I06–I08 | 九路 GPIO 和 PLIC 显式映射 | PLIC 路由通过 |
| 6 | Q01/Q03 | 增强阶段和不支持模式拒绝 | `qspi` 3/3 PASS |
| 7 | H01–H04 | 新建 HI_SYS 设备和状态连接 | `hi-sys` 4/4 PASS |
| 8 | X01–X05 | XIP 门控和寄存器驱动阶段 | `xip` 5/5 PASS |
| 9 | 全部 46 项 | 回归和文档同步 | Phase 10 完成 |

基础 Flash 6 项当前已通过。重构 FIFO/CS/XIP 时必须持续回归，不能默认它们以后仍会通过。

## 8. 测试记录模板

每次完成一个能力组后更新一行：

| 日期 | commit/工作树 | 组 | PASS/总数 | 第一失败 | 原因 | 下一步 |
|---|---|---|---:|---|---|---|
| 2026-07-14 | `spi` / qtest RED 基线 | 全部 | 16/46 | R04 `ser-num-cs` | spi1/spi2 仍为 1 CS | 先修 Patch 2 machine profile |
| YYYY-MM-DD | `<hash 或 dirty>` | `pio` | x/9 | `<路径>` | `<根因>` | `<下一动作>` |

失败记录只写第一条有信息量的断言，例如：

```text
P02 FAIL: TXFLR expected 256, got 0
结论：DR 写仍绕过 TX FIFO；不是阈值 mask 问题。
```

不要只写“qtest failed”。

## 9. Phase 10 完成标准

同时满足以下条件才能标记完成：

- [ ] `ninja -C build tests/qtest/k230-dw-ssi-test` 成功。
- [ ] 46/46 qtest PASS。
- [ ] 不存在 TIMEOUT、随机 sleep 或依赖宿主真实时间的测试。
- [ ] 三个实例地址、CS 数量、profile 和 PLIC 映射与 SDK 一致。
- [ ] Standard/Dual/Quad、Flash、HI_SYS 和 XIP 均有真实行为断言。
- [ ] DMA/Octal/DDR/RXDS 等未支持项有明确、稳定的拒绝或静默契约。
- [ ] 主 cnmd、study 文档和测试名/数量完全同步。
- [ ] Patch 10 英文文档只声明已由前九个 Patch 和 qtest 证明的能力。

最终结论应写成：

```text
K230 SPI/QSPI 10-Patch 主系列的 46 项 qtest 全部通过；
支持范围为 Standard/Dual/Quad、九路 IRQ、W25Q256、HI_SYS 和受控 XIP；
不包含内部 DMA 搬运、Octal/OPI、DTR/RXDS 和完整冷启动。
```

不得写成“完整支持 K230 OPI”或“已验证真实八线时序”。
