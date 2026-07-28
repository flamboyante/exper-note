# 在 QEMU 里给 K230 把 SPI 控制器跑通:从零到 U-Boot 启动 Linux

记录时间：2026-07-28（扩展版）
分支：`k230-spiv3.4`　基线：`f893c46c3931b3684d235d221bf8b7844ddbf1d7`（upstream/master）
补丁数：11　主实现：`hw/ssi/k230_dw_ssi.c`（1754 行）　qtest：`tests/qtest/k230-dw-ssi-test.c`（917 行）

这篇博客记录的是我在 QEMU Camp 里做的事：给 K230 这个 RISC-V SoC 加一个 SPI 控制器模型，让它能真的从 SPI NOR Flash 启动 Linux。不是事后总结的漂亮话，是边做边踩的坑、做的决策、还有一堆"当时怎么也没想到"的瞬间。写下来主要是想给老师一个能看的全过程，也顺便给后面想给 QEMU 加设备的人留个参考。

中间会穿插一些 QEMU 的机制（QOM、MemoryRegion、SSI 总线、qtest 这些）和少量的 SPI 协议、Linux 启动链知识——这些是我做的时候现学的，写在这里方便和我一样刚接触 QEMU 设备建模的人。

## 1. 为什么是这么个题目

QEMU Camp 是开源贡献训练营，有 mentor 带，目标是真的往 QEMU 上游提补丁。我选 K230 SPI 这个题，原因其实挺朴素的：在我之前已经有人把 K230 的 C908 内核、CLINT、PLIC、看门狗和 UART 建好了，但这个机器是个"半残"——它能跑，但没法从真实启动路径起来，因为缺 SPI 控制器。K230 的真实启动是从 SPI NOR Flash 加载 U-Boot，再由 U-Boot 拉 OpenSBI 和 Linux。SPI 这块不补上，整个机器就永远只能靠 `-bios` 直接喂内核，离"模拟真实硬件"差一口气。

这里多说一句 QEMU 的启动模型。QEMU 里 `-bios` 参数把一个镜像直接加载到 CPU 复位后开始取指的地址（RISC-V 上一般是 `0x80000000`），跳过 BootROM 和真实外设交互；而真实硬件上电后是 BootROM 从 SPI NOR 把 U-Boot 读进 RAM 再跳过去。我做的事就是让后一条路径也能在 QEMU 里跑通——这要求 SPI 控制器、SPI NOR Flash、地址映射都得建模，不然 U-Boot 的 `sf probe` 一执行就找不到设备。

所以我给自己定的目标就是：**把 K230 的 DesignWare SSI 控制器建出来，让 U-Boot 能从 SPI NOR 把 Linux 拉起来**。具体要覆盖三条启动路径——标准 SPI PIO、QSPI IDMA、还有 XIP 直接读窗口。最后提了 11 个补丁到上游，这就是 v3.4。

这篇博客不讲上游邮件来回的细节（其实也不多），重点放在：我怎么从读 TRM 开始，一步步把模型搭起来，中间踩了什么坑，怎么做的设计权衡。

## 2. K230 和它的 SPI 控制器长什么样

K230 是个 RISC-V SoC，C908 内核。它有三个 SSI 控制器实例，SDK 按地址映射给它们编号：

| 实例 | MMIO 地址 | 能力 | num-cs | 有 XIP |
|---|---:|---|---:|:---:|
| spi0 | `0x91584000` | SPI-OPI（支持 8 线） | 1 | 是 |
| spi1 | `0x91582000` | QSPI0 | 5 | 否 |
| spi2 | `0x91583000` | QSPI1 | 5 | 否 |

这三个实例能力不一样，但寄存器布局是同一套——都是 Synopsys 的 DesignWare SSI IP。这一点后面讲拆分的时候还会提到。

启动路径主要用 spi0。它的 CS0 上挂一片 W25Q256（32 MiB），Flash 里烧着 OpenSBI、Linux 内核、initrd 和 DTB。U-Boot 通过 `sf probe` / `sf read` 把这些镜像读到 RAM 里，最后 `bootm` 启动。XIP 路径更直接一点：OpenSBI 不往 RAM 里搬了，U-Boot 直接从 `0xc0000000` 这个映射窗口读 OpenSBI 的 uImage 头然后 bootm。

除了三个 SSI 实例，还有一块 `HI_SYS` 寄存器（`0x91585000` 附近），里面有个 `SSI_CTRL`（`0x91585068`）控制 XIP 的使能和三实例的模式/休眠状态。这块不算 SSI 内部寄存器，是 SoC 级的包装，所以我把它单独放到了 `hw/misc/k230_hi_sys.c`。

**QEMU 里 SoC 是怎么建模的**：QEMU 把每个设备做成一个 `TYPE_SYS_BUS_DEVICE`（系统总线设备），设备在 `realize` 阶段通过 `sysbus_init_mmio()` 把自己的寄存器 MemoryRegion 注册到系统总线上，机器板级代码（`hw/riscv/k230.c`）再用 `sysbus_mmio_map()` 把它映射到 SoC 地址空间的具体偏移。CPU 访问 `0x91584000` 时，QEMU 的 MemoryRegion 查找会路由到 spi0 设备的 `.read`/`.write` 回调。XIP 窗口 `0xc0000000` 同理，是 spi0 设备额外注册的第二个 MemoryRegion。这套机制让你不用关心物理地址怎么接线的细节，只管"我这个设备有几个 MMIO 窗口、每个窗口的 ops 是什么"。

启动 QEMU 跑这条路径的命令大概长这样（摘自我自己的复现笔记 [k230-spi-flash-uboot-linux-quickstart.md](k230-spi-flash-uboot-linux-quickstart.md)）：

```bash
QEMU=./qemu-camp-2026-k230/build/qemu-system-riscv64
UBOOT_SPI=/tmp/u-boot-spi-enabled

"$QEMU" \
    -M k230 -nographic \
    -bios "$UBOOT_SPI" \
    -drive if=none,file=flash.bin,format=raw,id=flash0 \
    ... # -device 把 flash0 挂到 spi0 CS0
    -serial stdio -monitor none -display none -no-reboot
```

## 3. v3.4 这 11 个补丁在干什么

一开始我想着一个大补丁搞定，后来 mentor 反馈说必须拆——设备模型、SoC 接线、qtest 要分开，每个补丁都能独立编译独立测试。这个思路其实是从上游 IOMUX review 里 Alistair 那里学来的。最后拆成了 11 个，按功能依赖排：

| # | Commit | 标题 | 一句话职责 |
|---|---|---|---|
| 1 | `12e325190a` | hw/ssi: Add K230 DesignWare SSI register model | 寄存器壳子：布局、复位值、writable mask、MMIO |
| 2 | `41750ee3b9` | hw/riscv/k230: Instantiate K230 SSI controllers | 机器里实例化三个 SSI + qtest 入口 |
| 3 | `6fc734d599` | hw/ssi: Implement K230 SSI FIFO and standard PIO transfers | Fifo32 + 四种 TMOD 的标准 PIO |
| 4 | `087a0aba5b` | hw/ssi: Add K230 SSI interrupt controller | 9 路 IRQ、TXE/RXF 动态、RC 锁存 |
| 5 | `662158aeeb` | hw/riscv: Route K230 SSI IRQs to the PLIC | 9 路 PLIC source 接通 |
| 6 | `db4d5698e1` | hw/ssi: Implement K230 enhanced QSPI transfers | Dual/Quad SDR 五阶段事务 |
| 7 | `d42e5d4f87` | hw/riscv/k230: Attach SPI NOR flash to spi0 | `spi-flash` 机器属性，m25p80 挂 CS0 |
| 8 | `8f3b84c222` | hw/ssi: Implement K230 SSI internal DMA transfers | 同步内部 DMA（8 位 SDR Dual/Quad） |
| 9 | `4b46245962` | hw/misc: Add K230 HI_SYS SSI control | SSI_CTRL 包装寄存器 |
| 10 | `3556bbff1b` | hw/ssi: Add K230 SSI XIP read window | 128 MiB spi0 XIP MMIO |
| 11 | `1342f13a6e` | hw/ssi: Add trace events for K230 DesignWare SSI | trace 事件（排除 DR） |

拆分顺序的逻辑是：先把寄存器壳搭起来（1），再实例化进机器（2），然后从最简单的标准 PIO 开始一层层加功能（3→4→5→6），中间挂上 Flash（7），再做 IDMA（8）和 HI_SYS（9），最后 XIP（10），trace 单独放最后（11）。每个补丁只加一层，qtest 跟着扩。这样做的好处是 review 的时候每个补丁聚焦一件事，坏处是前几个补丁的代码很"空"——比如补丁 1 只有寄存器读写，没有任何传输逻辑，看着像个空壳。但这是上游要求的拆法。

下面挑五个有故事的补丁展开讲：1、3、6、8、10。

## 4. 关键补丁深入

### 4.1 补丁 1：寄存器壳子怎么搭

这一步的目标很单纯：把 K230 DW SSI 的寄存器空间映射到 QEMU 的 MemoryRegion 上，能读能写，复位值对，writable mask 对。传输逻辑先不管。

寄存器列表是从 K230 TRM 12.3 一条条抄下来的：`CTRLR0`、`CTRLR1`、`SSIENR`、`SER`、`BAUDR`、`TXFTLR`/`RXFTLR`、`TXFLR`/`RXFLR`（动态）、`SR`（动态）、`IMR`/`ISR`/`RISR`、`*ICR`（RC）、`IDR`（RO，复位 `0xa1b2c3d5`）、`VERSION`（RO，复位 `0x3130332a`）、还有 `DR0`..`DR35` 一堆别名。

这里得先讲一下 QEMU 设备建模的核心——**QOM（QEMU Object Model）**。QOM 是 QEMU 自己搞的一套面向对象类型系统，每个设备类型由一个 `TypeInfo` 静态描述，运行时由 `ObjectClass`/`Object` 实例表示。给一个 SoC 外设建模，基本上是固定套路：

- `TYPE_K230_DW_SSI` 继承自 `TYPE_SYS_BUS_DEVICE`
- `instance_init`：分配子对象（比如 SSI 总线）、初始化 MemoryRegion（但还没映射）
- `realize`：设备"真正可用"的钩子，这里挂从设备、完成和外部世界的连接
- `class_init`：注册 QOM 属性、`dc->realize`、`dc->reset`、`dc->vmsd`（迁移状态）
- `exit_reset`：复位时填寄存器复位值

QOM 有个细节我一开始没搞明白：`instance_init` 和 `realize` 的区别。`instance_init` 在对象一创建就跑，那时候属性还没设置、parent 设备还没 realize，所以不能依赖任何"外部世界"的东西；`realize` 才是"设备真正接上系统"的时刻，这时候才能去访问 QOM 属性、挂子设备。我把 SSI 总线放在 `instance_init`（因为它是个子对象，不依赖外部），把挂 SPI NOR 这种事放到机器板级代码里（因为要根据 `-device` 参数决定挂不挂）。

寄存器访问走 `MemoryRegionOps`：

```c
/* 伪代码：展示类型声明和 ops 骨架，省略错误处理 */
static const TypeInfo k230_dw_ssi_info = {
    .name = TYPE_K230_DW_SSI,
    .parent = TYPE_SYS_BUS_DEVICE,
    .instance_size = sizeof(K230DwSsiState),
    .instance_init = k230_dw_ssi_init,
    .class_init = k230_dw_ssi_class_init,
};

static void k230_dw_ssi_init(Object *obj)
{
    K230DwSsiState *s = K230_DW_SSI(obj);
    /* SSI 总线子对象，挂 SPI NOR 这种 SSI 从设备 */
    object_initialize_child(obj, "ssi-bus", &s->ssi_bus, TYPE_SSI_BUS);
    /* 主 MMIO：4 KiB 寄存器窗口 */
    memory_region_init_io(&s->mmio, obj, &k230_dw_ssi_ops, s,
                          TYPE_K230_DW_SSI, K230_DW_SSI_MMIO_SIZE);
    sysbus_init_mmio(SYS_BUS_DEVICE(obj), &s->mmio);
    /* XIP 窗口：128 MiB，单独的 MemoryRegionOps */
    memory_region_init_io(&s->xip, obj, &k230_dw_ssi_xip_ops, s,
                          TYPE_K230_DW_SSI ".xip", K230_DW_SSI_XIP_SIZE);
}

static const MemoryRegionOps k230_dw_ssi_ops = {
    .read  = k230_dw_ssi_read,
    .write = k230_dw_ssi_write,
    .valid.min_access_size = 1,
    .valid.max_access_size = 4,
    .impl.min_access_size  = 4,   /* 寄存器按 32 位对齐 */
    .impl.max_access_size  = 4,
    .endianness = DEVICE_LITTLE_ENDIAN,
};
```

`MemoryRegionOps` 里 `.valid` 和 `.impl` 是两套不同的约束：`.valid` 是"QEMU 接受的访问范围"（超出会被拒），`.impl` 是"设备真正能处理的访问范围"（超出时 QEMU 会帮你拆分/合并）。对 SSI 这种 32 位寄存器块，把 `.impl` 钉死成 4 字节，QEMU 就会自动把 guest 的字节/半字访问拆成多次 word 访问，省得自己处理 size。这个 trick 是我读 `designware_i2c.c` 学的。

这里有个细节挺有意思。`VERSION` 寄存器复位值是 `0x3130332a`，乍看像个魔数。后来翻 TRM 17376 行，原文写着 "Contains the hex representation of the **Synopsys** component version"，把 `0x3130332a` 按 ASCII 解出来就是字符串 `1.03*`。也就是说 K230 TRM 自己承认这是 Synopsys 的 IP，不是自研。这个发现后来成了"该不该拆通用模型"的关键证据，先按下不表，第 6 章再说。

补丁 1 还做了一件不太显眼但很重要的事：`writable mask`。DW SSI 的寄存器很多字段是保留位（RAZ/WI）或者只读（RO），不能直接 `s->regs[offset] = value`。我写了个 `k230_dw_ssi_write_masked()`，每个寄存器配一个 mask，写入时按位与。这个在补丁 1 只做了一半，后面补丁加新字段时还得回来补 mask——这是拆分补丁的代价，同一份寄存器表被碰了好几次。QEMU 里有些设备用宏生成 mask 表（看 `pl011.c` 那种），我这里图省事用了手写数组，后面 v2 重构时可能会换成宏。

### 4.2 补丁 3：PIO 数据路径和 FIFO

补丁 1 只能读写寄存器，不能传数据。补丁 3 开始加真正的传输逻辑：标准 SPI 的 PIO，也就是 `DR` 写入 → TX FIFO → 传输泵 → RX FIFO → `DR` 读出。

这里得先讲 QEMU 的 **SSI 总线抽象**。QEMU 在 `include/hw/ssi.h` 里定义了一套 SSI（Synchronous Serial Interface）总线，专门给 SPI 这种主从结构用：

- `TYPE_SSI_BUS` 是总线，挂在 master 设备下
- `TYPE_SSI_SLAVE` 是从设备接口，m25p80（SPI NOR Flash 模型）就继承它
- master 用 `ssi_transfer(SSI_SLAVE(cs), value)` 给某个片选发一帧，slave 在自己的 `transfer` 回调里返回 RX 值

一次 `ssi_transfer()` 就是 SPI 总线上的一个时钟周期——master 推一个值出去，同时收到一个值回来。这是 QEMU 对 SPI 的"事务级"抽象：不模拟时钟沿、不模拟波形，只关心"一帧数据进出"。这对 SPI NOR 这种设备完全够用，因为它只关心字节流。

FIFO 用的是 QEMU 的 `Fifo32`，容量 256 项，每项代表一个 4–32 位的帧（不是字节）。`Fifo32` 是 QEMU 自带的环形缓冲，`fifo32_push`/`fifo32_pop`/`fifo32_is_full` 这套 API 简单到不用查文档。这里有个点容易绕进去：`DR0` 到 `DR35` 这 36 个寄存器地址**共享同一对 TX/RX FIFO**。SDK 驱动可能写 `DR0`，也可能写 `DR35`，行为完全一样。所以读写的入口得统一：

```c
/* 伪代码：DR 写入的入口，省略 SSIENR/IDMA 等门控 */
static void k230_dw_ssi_push_tx(K230DwSsiState *s, uint32_t tx)
{
    uint32_t mask = k230_dw_ssi_frame_masked(s);   /* 按 DFS+1 截断 */
    if (fifo32_is_full(&s->tx_fifo)) {
        return;                                    /* 满了就丢，不阻塞 */
    }
    fifo32_push(&s->tx_fifo, tx & mask);
    k230_dw_ssi_run_transfer(s);                   /* 尝试推进事务 */
}
```

四种 TMOD 模式（TR 收发、TO 只发、RO 只收、EEPROM_READ）走同一个 `k230_dw_ssi_run_transfer()` 传输泵，区别只在 RX 帧数怎么决定。这里稍微解释下 SPI 协议层面 TMOD 是什么：DW SSI 的 `CTRLR0[11:10]` 叫 TMOD，决定这次事务是"既发又收"（TR，标准 SPI 全双工）、"只发不收"（TO，写 Flash 用）、"只收不发"（RO，需要先发个 dummy 指令触发）、还是"EEPROM_READ"（先发指令+地址，再按 NDF 收数据）。EEPROM_READ 是 SPI NOR 读的标准姿势——发一个 `0x03` 加地址，然后 SPI NOR 自动往外吐数据，控制器按 `CTRLR1.NDF+1` 收这么多帧。

- TR/TO：TX FIFO 有多少发多少，RX 跟着回多少（TO 没有 RX）
- RO/EEPROM_READ：由 `CTRLR1.NDF+1` 决定接收帧数，先发一个指令/地址帧，然后纯收

这里踩了个坑，第 5 章会讲——一开始我没实现 EEPROM_READ，结果 U-Boot 卡在 `spi_nor_read_id()`，因为 SDK 的 `dw_spi_exec_op()` 标准 read 用的就是 EEPROM_READ 模式。

补丁 3 还处理了几个不变量，这些是从 TRM 和 SDK 行为里反推出来的：

- `0 <= TXFLR <= 256`，动态计算，不存状态
- `SSIENR` 写 0 会停止事务、撤销 CS、清 FIFO
- `SSIENR=0` 时 `DR` 写入被拒（保守的 SDK 兼容策略）

跑这个补丁的 qtest：

```bash
tests/qtest/run-k230-dw-ssi-tests.sh "pio-data-path"
```

qtest 是 QEMU 自家的设备测试框架，不走 CPU 仿真，直接在进程内调用设备的 MMIO read/write 回调，速度快、结果确定。我后面所有补丁都用它驱动。

### 4.3 补丁 6：QSPI 增强传输

标准 SPI 是一根线一位地传，QSPI 是 4 根线并行传数据。DW SSI 的增强模式通过 `SPI_CTRLR0` 寄存器配置，把一次事务拆成五个阶段：**指令 → 地址 → mode bits → dummy cycles → 数据**。

SPI 协议层面解释一下为什么是这五阶段。标准 SPI 读 Flash 是：主机拉低 CS，发 `0x03` 指令、发 3 字节地址，然后持续读时钟，Flash 在每个时钟沿吐一个字节回来——指令、地址、数据全走一根 MOSI/MISO。Quad SPI 的区别是数据阶段把 4 根 IO 线都拿来用，一个时钟传 4 位，吞吐量 4 倍。但指令和地址阶段要不要也走 4 线？这就得让控制器能分别配置每个阶段的线宽。所以 `SPI_CTRLR0` 把指令长度、地址长度、mode bits、数据线宽、dummy 周期数都做成独立字段，控制器按字段依次发。

`SPI_CTRLR0` 的关键字段：

| 字段 | 含义 |
|---|---|
| `INST_L` | 指令长度（0=不传，1~4 字节） |
| `ADDR_L` | 地址长度（0~32 字节） |
| `MODE_BITS` | mode bits 长度 |
| `DATA_WIDTH` | 数据宽度（1/2/4/8 位） |
| `WAIT` | dummy cycles 数 |

实现上我写了个状态机，按字段依次喂给 SSI 总线：

```c
/* 伪代码：五阶段事务骨架，省略错误路径 */
static void k230_dw_ssi_run_enhanced_transfer(K230DwSsiState *s)
{
    /* 1. 指令阶段 */
    k230_dw_ssi_send_enhanced_field(s, inst, inst_len, /*width=*/1);
    /* 2. 地址阶段 */
    k230_dw_ssi_send_enhanced_field(s, addr, addr_len, addr_width);
    /* 3. mode bits */
    k230_dw_ssi_send_enhanced_field(s, mode, mode_len, mode_width);
    /* 4. dummy：不发数据，只是等周期数 */
    for (i = 0; i < dummy_cycles; i++) { /* 空转 */ }
    /* 5. 数据阶段：TO 或 RO，复用标准 FIFO 逻辑 */
    k230_dw_ssi_run_transfer(s);
}
```

QEMU 的 SSI 总线抽象不暴露"线宽"概念——`ssi_transfer()` 永远是"一个值进一个值出"。所以多线事务在 QEMU 里没法真的并行传 4 位，我只能按"一个时钟周期传一个数据单元"来近似。这对功能正确性没影响（Flash 模型只关心字节流内容），但意味着我不能用 QEMU 来验证时序相关的 bug。这是 QEMU SSI 抽象的固有边界，`SCPOL/SCPH/SSTE` 这些线级时序寄存器我也只能做契约保存，不模拟波形。

这里有个"做减法"的决策值得说一下。spi0 在 TRM 5.3 里描述是支持 8 线 OPI 的，但当前模型**保留 8 线 profile 却拒绝 Octal 数据事务**。原因很实在：启动软件路径上根本没人用 Octal，U-Boot 和 RT-Smart 的 SPI NOR 驱动最高只到 Quad。把 Octal 也建了是纯粹的浪费，而且 TRM 对 Octal 的描述也不完整。所以我选择在 `k230_dw_ssi_enhanced_config_supported()` 里明确拒绝 Octal、DDR、DTR 这些，留个口子以后真要用了再加。

这个决策后来在上游 review 里没人挑刺——说明"按需实现、明确边界"在上游是可接受的。

### 4.4 补丁 8：IDMA 为什么做成同步

这个补丁的故事最多，也最能体现"设计权衡"四个字。

K230 的 SPI 控制器有个内部 DMA（IDMA），不是外接的通用 DMA，是控制器自己带的 AXI master。它有一组寄存器：`SPIDR`（SPI 侧地址，即 Flash 偏移）、`SPIAR`（占位）、`AXIAR0`/`AXIAR1`（内存侧地址）、`DMACR.IDMAE`/`AINC`/`ATW` 等、还有 `DONE`/`AXIE` 两个状态位。

一开始我想把它做成异步的——QEMU 里有现成的 stream 模型（Versal OSPI + CSU DMA 就是这么做的），控制器和 DMA 设备之间用 `stream_push()`/`stream_can_push()` 互动，能体现 backpressure 和分块搬运。但写之前我先把 SDK 的驱动看了一遍，发现一个关键事实：

**U-Boot 的 `designware_spi.c` 和 RT-Smart 的 `drv_spi.c` 都是轮询 `DONE` 位的。** SDK 的调用序列大概是这样：

```c
/* 伪代码：SDK 驱动的 IDMA 使用方式，摘自 RT-Smart drv_spi.c */
spi->dmacr = DMACR_IDMAE | DMACR_AINC | ...;
spi->spi_ar = flash_offset;
spi->axi_ar0 = dram_addr;
spi->axi_awlen = length;
spi->ser = 1 << cs;
spi->ssienr = 1;
/* 然后……就死等 DONE */
rt_event_recv(..., BIT(SSI_DONE) | BIT(SSI_AXIE), ...);
spi->ser = 0;
spi->ssienr = 0;
```

软件既然是轮询的，QEMU 的同步模型完全够用：写完 `SSIENR=1` 之后，模型一次性把数据从 Flash 搬到内存，置 `DONE`，软件读到 `DONE` 继续往下走。异步反而增加复杂度——得多一个 BH、多一套 backpressure，但软件根本感知不到这个异步性。

这里展开讲一下 QEMU 的 DMA 建模。QEMU 里 DMA 本质上是"设备代替 CPU 访问 Guest 物理内存"。直接用 `memcpy` 不行，因为 Guest 物理地址不等于 Host 虚拟地址——中间隔着 MemoryRegion 的翻译。QEMU 提供了两种方式让设备访问 Guest 内存：

1. **`dma_memory_read()`/`dma_memory_write()`**：给定一个 `AddressSpace` 和 Guest 物理地址，QEMU 自动走 MemoryRegion 查找，翻译成 Host 指针后搬运。这是最简单的 DMA 建模，一次调用搬一块，同步完成。
2. **Stream 模型**：master 和 slave 通过 `stream_push()`/`stream_can_push()` 互动，能表达 backpressure（下游满了就暂停）、分块搬运、异步推进（用 Bottom Half 延后）。Versal OSPI + CSU DMA 走这条路。

我选的是第一种。IDMA 触发时，模型拿到 `AXIAR0`（Guest RAM 地址）和 `SPIDR`（Flash 偏移），通过 SSI 总线从 m25p80 读出数据，再用 `dma_memory_write()` 写进 Guest RAM。一次事务同步完成，最后置 `DONE`。

```c
/* 伪代码：IDMA 同步搬运骨架，省略错误处理和 trace */
static void k230_dw_ssi_try_idma(K230DwSsiState *s)
{
    if (!k230_dw_ssi_idma_triggered(s)) {
        return;    /* 门控条件没满足，不动 */
    }
    uint64_t address = k230_dw_ssi_idma_address(s);
    /* 从 Flash 读 length 字节到 Guest 内存 */
    for (i = 0; i < length; i += chunk) {
        /* 经过 SPI 事务读 Flash，再 dma_memory_write 进 Guest RAM */
        ...
    }
    k230_dw_ssi_idma_end(s, R_RISR_DONER_MASK);    /* 置 DONE */
}
```

为什么不用 stream？因为 SDK 是轮询的，软件感知不到异步性，stream 的 backpressure 和分块能力在这里纯属浪费。我在 IDMA 学习手册（[k230-spi-idma-patch-learning-workbook.md](learning/k230-spi-idma-patch-learning-workbook.md)）里把 QEMU 主线两种 DMA 实现方式做了对照：Versal OSPI 的 stream 连接 vs Aspeed SMC 的内部 AddressSpace。当前 K230 走的是后者那种"内部 DMA 直接访问 AddressSpace"的简化路线，但还没做到 Aspeed 那种分块、进度、backpressure 级别。这是有意为之的取舍——同步够用，异步是过度设计。

关于"QEMU 里什么时候该用 BH 异步"也多说一句。Bottom Half（`aio_bh_schedule`）是 QEMU 的延迟执行机制，用于把耗时事务切成小块、避免阻塞 vCPU 线程。如果你模拟的 DMA 在真实硬件上是后台跑的，软件会一边干别的、一边等 DMA 完成，那 QEMU 里就得用 BH+定时器模拟这种异步性，否则 vCPU 会被一次大搬运卡死。但我这里软件就是死等 `DONE`，vCPU 反正要停，同步反而更贴合软件行为。

还有一个限制：IDMA 只支持 8 位 SDR Dual/Quad，和 SDK 驱动实际使用的模式一致。Octal/DDR 这些不进 IDMA 路径。

### 4.5 补丁 10：XIP 读窗口

XIP（eXecute In Place）是让 CPU 直接把 SPI NOR 当内存读。spi0 上挂一个 128 MiB 的 MMIO region @ `0xc0000000`，CPU 读这个地址，模型就翻译成一次 SPI 读事务发给 m25p80，把结果返回给 CPU。

QEMU 层面，XIP 窗口是 spi0 设备注册的**第二个 MemoryRegion**。一个设备可以有多个 MemoryRegion，各自有独立的 `MemoryRegionOps`。寄存器窗口是 4 KiB（`k230_dw_ssi_ops`），XIP 窗口是 128 MiB（`k230_dw_ssi_xip_ops`），机器板级代码把它们分别映射到 `0x91584000` 和 `0xc0000000`。CPU 访问 `0xc0000000` 时，QEMU 的 MemoryRegion 查找会路由到 XIP ops 的 `.read` 回调，和访问寄存器是完全不同的路径。

实现上根据 `SPI_CTRLR0` 当前的 `DATA_WIDTH` 选 Standard/Dual/Quad SDR 读命令：

```c
/* 伪代码：XIP 读入口，省略门控和错误处理 */
static uint64_t k230_dw_ssi_xip_read(void *opaque, hwaddr address, unsigned int size)
{
    K230DwSsiState *s = opaque;
    if (!k230_dw_ssi_xip_config_supported(s)) {
        return 0;    /* XIP 没使能，读返回 0 */
    }
    /* 根据 DATA_WIDTH 组装读命令：opcode + 地址 + dummy */
    uint32_t opcode = k230_dw_ssi_xip_opcode(s);    /* 0x03/0x3b/0x6b */
    /* 发指令、地址、dummy，然后读 size 字节 */
    ...
    trace_k230_dw_ssi_xip_read(s, address, size, value);
    return value;
}
```

XIP 受 `HI_SYS.SSI_CTRL` 的 bit 0 门控，关闭时读返回 0。还有个细节：XIP 窗口的读命令要和 Flash 当前的地址模式匹配。W25Q256 在 `sf probe` 后会进 4-byte address mode，所以 XIP 寄存器必须在所有 `sf read` 完成后重新配置，opcode 用 `0x13`（4-byte Read），`XIP_CTRL` 配成 8-bit instruction + 32-bit address。如果还用 3-byte read 配置，XIP 读取会错位。这个是实机调试才发现的。

这里有个 QEMU 的坑值得提：XIP 窗口默认会被 QEMU 当成普通 RAM 区域，guest 的一次 `memcpy` 可能发一个超大的访问请求过来（比如 64 字节），但 SPI NOR 一次只能按事务读。所以 XIP ops 的 `.impl.max_access_size` 我设成了 4 字节，让 QEMU 自动把大访问拆成多次 word 读，每次走一次 SPI 事务。不设这个的话，Flash 会收到一个它看不懂的超大 size，直接出错。

XIP 验证的实际输出长这样（摘自 [quickstart](k230-spi-flash-uboot-linux-quickstart.md)）：

```text
c0000000: 56190527
## Booting kernel from Legacy Image at c0000000 ...
Verifying Checksum ... OK
Starting kernel ...

OpenSBI v0.9
[    0.000000] Linux version 6.18.28
meta-k230 initramfs starting...
~ #
```

`56190527` 是 OpenSBI uImage 的 magic，能读到这个并且 checksum OK，说明 XIP 窗口真的把 Flash 里的 OpenSBI 映射出来了。

## 5. 踩过的坑

这一章是最真实的部分。不是事后美化的"设计决策"，是当时真的卡住、真的调了一晚上才搞明白的东西。

**坑 1：不实现 EEPROM_READ，U-Boot 就卡死**

补丁 3 一开始我只实现了 TR 和 TO，觉得 RO 和 EEPROM_READ 后面再说。结果挂上 Flash 跑 U-Boot，卡在 `spi_nor_read_id()`。打开 trace 一看：

```text
CTRLR0 = 0x00000c07  -> TMOD[11:10] = 3，EEPROM_READ
CTRLR1 = 0x00000005  -> NDF + 1 = 6 个接收帧
SSIENR = 0x00000001  -> 控制器已启用
```

SDK 的 `dw_spi_exec_op()` 标准 read 用的就是 EEPROM_READ 模式，先发一个指令帧，然后按 `NDF` 收数据。我这边 TMOD=3 压根没生成接收时钟，U-Boot 死等 RX FIFO。后来老老实实把 EEPROM_READ 补上，read_id 才通。这个坑教会我：**实现顺序不能图省事，得跟着软件的实际调用路径走**。

**坑 2：补丁 6 构建失败，多余大括号**

补丁 6 加增强 QSPI 的时候，我在 `EEPROM_READ` 的一个分支里多写了一对大括号。本地用 gcc 编译没事，一上 CI（LLVM/Clang）直接报错。这种 gcc/clang 严格性差异以前只在书上见过，这次真碰上了。之后我养成了本地用 clang 也编一遍的习惯。

**坑 3：补丁 7 破坏了 send_frame()**

补丁 7 挂 SPI NOR 的时候，改了 CS 和事务结束的处理，结果把补丁 3 的 `k230_dw_ssi_send_frame()` 路径弄回退了。RED 测试（先写测试再写实现那套）抓到了——pio-data-path 用例红了。这个让我意识到补丁拆得越细，越容易在后面的补丁里把前面的不变量碰坏。qtest 必须每个补丁都跑全量，不能只跑新增的。

**坑 4：num-cs 的 SDK 内部冲突**

K230 的 `num-cs` 在 SDK 内部自己对不上：U-Boot DTS 写的是 `1/5/5`（spi0/spi1/spi2），Linux DTS 写的是 `1/1/1`。这个冲突不是 bug，是两边配置策略不同。我按当前启动软件路径选了 `1/5/5`，但没把它写成 TRM 的唯一结论，而是在上游说明里保留了证据差异。**遇到 SDK 内部不一致，记录证据、按用例决策、不替上游下结论**——这个是 mentor 教的。

**坑 5：版本号谜题**

K230 DTS 里 SSI 的兼容串是 `snps,dwc-ssi-1.01a`，但 `VERSION` 寄存器复位值 `0x3130332a` 解出来是 `1.03*`。1.01a 和 1.03* 对不上。这个我到现在没搞明白是 K230 的 DTS 写错了还是 Synopsys 的版本号体系和 compatible string 不是一回事。在寄存器审计里把它记成"待证据升级"的疑点，没有强行解释。这种"不知道就记下来"的态度，我觉得比硬编一个解释强。

## 6. 上游 review 和 DW SSI 拆分

v3.2 的时候 Bin Meng 在 patch 1 上给了一个反馈：**把模型拆成两层**——一个通用的 Synopsys DesignWare SSI 控制器模型，加一个可选的 K230 专有 wrapper。目的是让这个 SSI 模型以后能被其他用 DW SSI IP 的 SoC 复用。

我当时的第一反应是"凭什么是通用的"，结果一翻 TRM，证据全摆在那：

1. **TRM 自己承认是 Synopsys IP**：`VERSION` 寄存器那段 "Contains the hex representation of the **Synopsys** component version"，而且全文大量引用 `SSIC_HAS_*` / `SSIC_*` 这种 Synopsys IP 的 Verilog 参数化配置项。
2. **U-Boot 的 `designware_spi.c` 就是通用 driver**：文件头注释写着 Denx 维护，Copyright Stefan Roese / Sean Anderson，基于 Linux `drivers/spi/spi-dw.c`。K230 SDK 只在顶部改了一行 `#define SSIC_HAS_DMA 2`。
3. **QEMU 已有先例**：`hw/i2c/designware_i2c.c` 就是 Synopsys DW I2C 的通用模型，`TYPE_DESIGNWARE_I2C`，没挂任何 SoC wrapper——通用层加属性就够了。

三条证据链对齐，结论很清楚：**该拆**。QEMU 上游对"IP 通用模型 + SoC wrapper"这种分层是有惯例的，比如 `designware_i2c`、`pl011`、`cadence_gem` 都是这么组织的。我现在把所有东西塞进 `k230_dw_ssi.c` 是个临时形态。

但 v3.4 我没拆。原因也很实在：v3.4 的重点是收敛启动路径，让 U-Boot 真的能从 SPI NOR 起来。拆分是个大重构，得重新设计模型组织、把寄存器按"通用/K230 专有"分类、决定 XIP 窗口和 HI_SYS 联动怎么放。这些事我想放到 v2 去做，v3.4 先把功能跑通、把上游能 review 的东西稳住。

所以博客里这块我就点到"需要拆分"为止，不画 v2 的目标结构图——因为我还没想清楚，不想把未来结构写死。能说的是：v2 会把通用层摘出来，K230 wrapper 会很薄，主要做实例化参数和 XIP 窗口注册。

## 7. 实机验证：U-Boot 真的能启动 Linux

这是整个项目最有成就感的一步。三条启动路径都验通了。

先简单交代下 RISC-V 的启动链。RISC-V 上电后跑的是 SBI（Supervisor Binary Interface）固件，K230 这条路径里 SBI 实现是 OpenSBI，它负责 M-mode 的初始化和 SBI 调用转发。U-Boot 在这里既当 BootROM 之后的 loader（被 `-bios` 加载），又负责把 OpenSBI、Linux、initrd、DTB 读进 RAM，最后 `bootm` 跳进 OpenSBI，OpenSBI 再跳进 Linux 内核。Linux 起来后如果没有根文件系统挂载点，就用 initramfs（一个内存里的根文件系统镜像）跑 shell。所以"`~ #`"出来，意味着整条链 SPI→U-Boot→OpenSBI→Linux 全通了。

**标准 SPI**：U-Boot 用 `sf probe` 识别 W25Q256，`sf read` 把 OpenSBI、Linux、initrd、DTB 读到 RAM，`bootm` 启动到 initramfs shell。关键命令序列：

```text
sf probe 0:0
sf read 0x0c100000 0x0 0x14000        # OpenSBI
sf read 0x08200000 0x100000 0x1a1fe00 # Yocto Linux
sf read 0x0a100000 0x1c00000 0x1eec20 # initrd
sf read 0x0a000000 0x1f00000 0x1000   # DTB
bootm 0x0c100000 - 0x0a000000
```

`bootm` 是 U-Boot 启动 legacy uImage 的命令，格式是 `bootm <kernel> <ramdisk> <dtb>`，`-` 表示 ramdisk 用已加载的。它会校验 uImage 头的 checksum，然后跳到入口地址。

**Quad SPI**：spi0 配成 4 位传输，U-Boot 擦一个 64 KiB 扇区、写 256 字节、读回验证，再加载全部启动载荷，进 Linux initramfs shell。

**XIP**：OpenSBI 不往 RAM 搬了，U-Boot 直接从 `0xc0000000` 读 uImage 头、验 checksum、`bootm`。前面那节贴的 `c0000000: 56190527` 就是这条路径的输出。

最后跑完 `git diff --check` 没有空白错误，`checkpatch` 没有报错。qtest 收敛成 7 个端到端场景：`register-contract`、`pio-data-path`、`interrupt-routing`、`spi-nor`、`qspi-sdr`、`hi-sys`、`xip-read-window`，TAP `1..7 7/7` 通过。

## 8. 这次 Camp 我学到了什么

**技术上**：QEMU 的 QOM 设备建模、MemoryRegion、SSI 总线抽象、qtest 怎么写怎么跑、RED/GREEN 怎么用 qtest 驱动实现、trace 事件怎么加。还有 RISC-V SoC 的启动路径全貌——从 SPI NOR → U-Boot → OpenSBI → Linux 这条链以前是模糊的，现在亲手把它在 QEMU 里跑通了一遍，每个环节都摸过了。

最大的认知更新是 QEMU 的"事务级建模"哲学：它不模拟时钟沿、不模拟波形，只关心"一次访问、一次事务、一个状态变化"。这让我一开始很不适应——我老想着"这个时钟周期该干嘛"，但 QEMU 的 SSI 总线就一个 `ssi_transfer()`，一帧进一帧出，没有时钟概念。想通这点后，很多设计就顺了：`SCPOL/SCPH` 这些线级时序寄存器只存不动作，因为 QEMU 抽象层根本不暴露波形。

**工程方法上**：补丁拆分、commit message 怎么写、cover letter 怎么组织、怎么回应 review。这些以前看上游邮件列表觉得"至于分这么细吗"，自己走一遍才理解为什么上游要求每个补丁只做一件事——review 的人没时间通读 1700 行，只能按补丁逐个看，每个补丁越小、职责越单一，review 越快，回退风险也越小。QEMU 上游用 git send-email + lore 那套，patchwork 跟踪状态，checkpatch 做 stylistic 检查——这套工具链第一次用很别扭，但走完一遍就懂了为什么开源项目都用它。

**一个反思**：第一版把所有东西塞进 `k230_dw_ssi.c`（1754 行）是为了快速跑通，但代价是 v2 要重构。如果重来一次，我会在一开始就分层——先把通用寄存器壳搭好，再在 K230 wrapper 里填实例化参数。不过话说回来，没有 v1 的"一锅炖"，我可能也搞不清哪些是通用的、哪些是 K230 专有的——正是 v1 的混乱催生了 v2 的拆分需求。这个教训值。

最后，相关的笔记我都放在 [exper-note/k230/spi/](.) 下面了：最终报告在 [k230-spi-qspi-final-report.md](k230-spi-qspi-final-report.md)，启动复现在 [k230-spi-flash-uboot-linux-quickstart.md](k230-spi-flash-uboot-linux-quickstart.md)，各补丁的学习手册在 [learning/](learning/)。如果有人想复现或者接着做，从这些文件进去就行。