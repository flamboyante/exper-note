# K230 SPI NOR 完整冷启动 Gap 探究报告

记录时间：2026-07-28

## 结论摘要

当前已经验证成功的路径是：

```text
QEMU reset stub
  -> -bios 直接把完整 U-Boot 放入 DDR
  -> U-Boot 通过 Standard SPI 读取 Flash
  -> 启动未压缩 Linux
```

这条路径证明了 **完整 U-Boot 之后的 Standard SPI NOR 数据通路可用**，但它还不是
K230 的完整冷启动。真实冷启动前半段至少还包括 BootROM、SPL、DDR 初始化、启动介质
选择、固件头完整性校验，以及默认镜像的硬件 gzip 解压。

本次调查的核心结论是：

1. **完整冷启动目前仍有实质性 gap，不是只缺一段 BootROM 跳转代码。**
2. **直接启动现有 CanMV SPL 已动态跑到 DDR 初始化，并在 `0x9a04017c` 发生
   Store/AMO access fault。** 这是当前最早的硬阻塞。
3. 运行时跳过 DDR 后，CanMV SPL 会选择 SDIO1；强制改成 NOR 后，又依次暴露
   SPI bus disabled 和缺少 `spi-flash@0` 两个问题。这三项均已动态验证。
4. 即使使用非安全固件，SDK 仍会读 OTP 并调用 PUFS SHA-256；当前 QEMU 的
   `security@0x91210000` 只是 unimplemented 占位。这是源码确定的后续依赖，
   但本次尚未动态走到该阶段。
5. SDK 默认把完整 U-Boot 和 Linux 都打成 K230 私有 gzip，依赖 GSDMA、
   decomp-gzip 和 RMU；这些设备当前也只是占位。用户目前使用未压缩 Linux 的方向
   是正确的，但完整冷启动时还必须把 **SPL 加载的完整 U-Boot** 也改成不压缩。
6. 最短可行路线不是完整仿真 DDR PHY、PUFS 和硬件 gzip，而是实现一条明确标注为
   **功能型 BootROM** 的加载链，再为 SPL 提供极小的 DDR/PWR 兼容状态，并用少量
   SDK 配置/打包修改避开 PUFS 与硬件 gzip。

当前状态可简化为：

| 能力 | 当前结论 |
|---|---|
| `-bios U-Boot -> sf read -> 未压缩 Linux` | **支持，已验证** |
| `-bios SPL` 执行到 DDR 初始化 | **支持，已验证** |
| 现有 CanMV SPL 自然选择 SPI NOR | **不支持，已验证** |
| 现有 CanMV SPL DT 枚举 SPI NOR | **不支持，已验证** |
| QEMU 从 Flash 冷启动加载 SPL | **不支持** |
| 原样执行 SDK DDR training | **不支持，已验证为硬阻塞** |
| 原样执行 SDK PUFS OTP/SHA | **不支持，静态证据确定** |
| 原样解压 SDK 私有 gzip U-Boot/Linux | **不支持，静态证据确定** |
| 使用官方 EVB 配置减少 SDK 源码修改 | **有依据的候选路线，尚未动态验证** |

## 1. 调查范围与证据等级

### 1.1 版本基线

本次调查基于：

| 组件 | 版本 |
|---|---|
| QEMU | 分支 `k230-spiv3.4`，提交 `1342f13a6e` |
| K230 SDK | tag `v2.0`，提交 `7e302f7333` |
| 启动资产 | 提交 `c3c32fb46e` |

调查期间没有修改 QEMU 和 SDK 工作树；动态诊断只使用 `/tmp` 下的临时文件和 GDB
运行时改值。

### 1.2 证据等级

为避免把推断写成已完成能力，本文使用三种证据等级：

- **动态已验证**：实际执行到相应阶段，并观察到明确日志、异常或返回值。
- **静态确定**：源码调用关系和当前 QEMU 设备实现足以确定存在依赖，但尚未实际跑到。
- **待验证**：有源码依据的候选方案，尚未构建或启动成功。

“功能型冷启动”与“真实 BootROM 等价”也必须分开：

- **功能型冷启动**：CPU 从 reset vector 开始，执行放在 BootROM 地址空间内的 guest ROM
  stub；stub 通过现有 SPI 控制器和 Flash 模型读取镜像、处理固件头、加载 SPL 并跳转，
  最终进入 Linux。
- **真实 BootROM 等价**：执行合法的 K230 BootROM binary，或者按照公开且完整的 ROM
  规范复现其指令与外设行为。

本地仓库和 SDK 中没有找到可合法使用的真实 K230 BootROM binary；TRM 公开了启动模式、
地址窗口和 Flash 控制器能力，但没有公开完整 BootROM 指令和首级加载协议。因此本文只对
“功能型冷启动”给出可实施方案，**不声称可以做到真实 ROM 的指令级或周期级等价**。

### 1.3 EVB、CanMV 与“改 SDK”的边界

这里的 EVB 和 CanMV 都是 **K230 SDK 的板级构建目标**，不是两颗不同的 SoC，也不是
QEMU 的两种 CPU 模式：

```text
K230 SoC
├── EVB   ：官方 Evaluation Board（评估板）板级配置
└── CanMV ：面向机器视觉开发板/平台的板级配置
```

对应的 U-Boot 入口分别是：

| 板级目标 | defconfig | DTS | 板级 C 代码 |
|---|---|---|---|
| EVB | `configs/k230_evb_defconfig` | `arch/riscv/dts/k230_evb.dts/.dtsi` | `board/canaan/k230_evb/` |
| CanMV | `configs/k230_canmv_defconfig` | `arch/riscv/dts/k230_canmv.dts` | `board/canaan/k230_canmv/` |

两者共享 U-Boot 公共框架、K230 公共 SPL、镜像加载器和通用驱动，但会选择不同的：

- 板级 `board.c`；
- pinmux 和外设 DT；
- DDR 初始化对象；
- 默认启动介质和板载外设初始化。

最容易混淆的是“改配置”和“改代码”的边界：

| 修改方式 | 是否改变 SDK 内容 | 是否改变 C/汇编运行时代码 | 本报告中的分类 |
|---|---:|---:|---|
| 直接执行 `make k230_evb_defconfig` | 否 | 否 | 选择 SDK 已有官方配置 |
| 修改本次构建生成的 `.config` | 通常不提交 | 否 | 构建实例参数 |
| 修改 `*_defconfig` | 是 | 否 | SDK 配置修改 |
| 修改 `Kconfig` 菜单/默认值 | 是 | 否 | SDK 构建系统修改 |
| 修改 DTS/DTSI | 是 | 否 | SDK 板级硬件描述修改 |
| 修改 `board.c`、SPL 或 loader C 代码 | 是 | 是 | SDK 运行时代码修改 |
| 修改镜像生成脚本 | 是 | 否 | SDK 打包流程修改 |

因此，本文后面使用三个更精确的说法：

```text
SDK 零 C 代码修改
  = 不改 board.c、SPL、loader 等运行时代码；可以只选择已有 defconfig。

SDK 配置/板级描述修改
  = 修改 defconfig、Kconfig、DTS 或镜像脚本；仍然属于 SDK 改动，但不改 C 逻辑。

SDK 运行时代码修改
  = 修改 board.c、公共 SPL、镜像校验/加载逻辑或驱动实现。
```

EVB 之所以在本报告中作为候选，是因为其现有 DTS 已经启用 SPI0 并声明 `spi-flash@0`，
而且 EVB 没有覆盖启动模式函数，会使用公共实现读取 `SOC_BOOT_CTL[1:0]`。CanMV 则有
两个已动态暴露的问题：

1. `board/canaan/k230_canmv/board.c` 把启动模式硬编码为 `SYSCTL_BOOT_SDIO1`；
2. CanMV SPL 控制 DT 没有启用 `spi0`，也没有 CS0 的 `spi-flash@0` child。

所以“换 EVB”不是把 QEMU 换成另一颗芯片，而是选择 SDK 已存在的另一套板级 U-Boot
构建目标；“改 CanMV DTS”也不是改 U-Boot 核心代码，但确实属于 SDK 板级描述修改。

## 2. SDK SPI NOR 镜像所表达的真实启动链

SDK 的 SPI NOR 镜像布局来自
[`genimage-spinor.cfg`](../../../k230_sdk/board/common/gen_image_cfg/genimage-spinor.cfg)：

| 内容 | Flash 偏移 | 文件 |
|---|---:|---|
| SPL | `0x000000` | `swap_fn_u-boot-spl.bin` |
| 完整 U-Boot | `0x080000` | `fn_ug_u-boot.bin` |
| environment | `0x1e0000` | `env.env` / `jffs2.env` |
| Linux system | `0xfc0000` | `linux_system.bin` |

SDK 的 [`gen_image_comm_func.sh`](../../../k230_sdk/board/common/gen_image_script/gen_image_comm_func.sh)
还表明：

- SPL 先加 K230 firmware header，再做 endian swap；
- 完整 U-Boot 默认经过 `k230_priv_gzip`，再封装 legacy uImage 和 firmware header；
- Linux payload 同样默认经过 `k230_priv_gzip`。

K230 firmware header 定义在
[`k230_board_common.h`](../../../k230_sdk/src/little/uboot/board/canaan/common/k230_board_common.h)，
魔数为 `0x3033324b`（`"K230"`），结构大小为 528 字节，包含 payload 长度、crypto type
以及 hash/签名区域。生成逻辑见
[`firmware_gen.py`](../../../k230_sdk/tools/firmware_gen.py)。

结合 SPL 链接地址、镜像布局和 loader 代码，SDK 表达的 SPI NOR 启动链是：

```text
CPU reset
  -> 片内 BootROM，地址 0x91200000
  -> 根据 BOOT1:BOOT0 选择 SPI NOR
  -> 从 Flash 0x000000 读取并处理 swap_fn_u-boot-spl.bin
  -> SPL 在 SRAM 0x80300000 执行
  -> SPL 做板级初始化和 DDR training
  -> SPL 从 Flash 0x080000 读取 fn_ug_u-boot.bin
  -> 检查 K230 firmware header、OTP 策略和 payload hash
  -> 解压/复制完整 U-Boot到 0x08000000
  -> 完整 U-Boot 从 Flash 读取 linux_system.bin
  -> 再次校验、解压/复制并启动 OpenSBI/Linux
```

TRM 证据见
[`K230_Technical_Reference_Manual_V0.3.1_20241118.txt`](../reference/K230_Technical_Reference_Manual_V0.3.1_20241118.txt)：

- `BOOT1:BOOT0 = 00` 表示 SPI NOR 启动；
- BootROM 地址窗口为 `0x9120_0000..0x9121_0000`，大小 64 KiB；
- Security 窗口紧随其后，为 `0x9121_0000..0x9121_8000`。

## 2.5 SoC 多级启动与 QEMU 跳过机制速览

如果对 SoC 启动不熟悉，本节先解释为什么 K230 要分这么多级、每一级做什么、QEMU 当前的"成功路径"具体跳过了哪些环节，以及"跳过"在 QEMU 内部是怎么实现的。第 3 节的"绕过列表"可以对照本节理解。

### 2.5.1 为什么 SoC 启动要分多级

SoC 上电瞬间，CPU 从一个固定地址（reset vector）开始取指，该地址通常指向片内 ROM（BootROM），ROM 内容出厂固化、不可改。此时面临三个限制：

- 外部 DDR 还没初始化，CPU 只能用片内 SRAM，而 SRAM 容量很小（K230 SPL 链接地址 `0x80300000` 所在的 SRAM 仅够放 SPL 量级的代码）；
- BootROM 容量也很小（K230 是 64 KiB），不足以容纳完整系统加载逻辑；
- 启动介质（SPI NOR、SDIO、NAND…）的访问协议、镜像格式、安全策略因板而异。

因此行业通用做法是分多级加载：每一级做有限初始化，然后加载并跳转到下一级。K230 的 SPI NOR 链路（见第 2 节图示）是：

```text
BootROM  ->  SPL  ->  完整 U-Boot  ->  OpenSBI  ->  Linux
```

每一级都比上一级拥有更多资源（更大的 RAM、更完整的外设），可以承担更复杂的工作。

### 2.5.2 每一级做什么

| 阶段 | 运行位置 | 主要职责 |
|---|---|---|
| **BootROM** | 片内 ROM `0x91200000`，64 KiB | 读 BOOT strap 选介质；从 Flash `0x0` 读 SPL；校验 K230 firmware header（528 字节，魔数 `"K230"`）；处理 `swap_fn_u-boot-spl.bin` 的字节交换；把 SPL 放到 SRAM `0x80300000` 并跳转 |
| **SPL** (Secondary Program Loader) | SRAM `0x80300000` | 板级初始化（clock/reset/pinmux）；**DDR training**（训练 DRAM 控制器，让外部 DDR 可用）；从 Flash `0x080000` 读完整 U-Boot；再次校验 firmware header 和 payload hash；解压/复制到 DDR `0x08000000` 并跳转 |
| **完整 U-Boot** | DDR `0x08000000` | 资源充足，提供命令行（`sf probe`/`sf read`/`bootm`）；从 Flash `0xfc0000` 读 Linux；校验、解压；启动 OpenSBI |
| **OpenSBI** | DDR | RISC-V 的 M-mode -> S-mode 切换层，为 Linux 提供 SBI 服务 |
| **Linux** | DDR | 最终操作系统 |

注意 BootROM 和 SPL 都会校验 K230 firmware header——这意味着 header 不是只验一次，而是**每一级加载下一级时都重新校验**。这是 K230 的安全启动设计。

### 2.5.3 几个常被问到的概念

- **BOOT strap**：芯片上专门的启动模式引脚，上电时的电平组合决定从哪种介质启动。K230 用 `BOOT1:BOOT0` 两位，`00` = SPI NOR。真实硬件是物理引脚，QEMU 用 `SOC_BOOT_CTL[1:0]` 寄存器模拟。但 CanMV 的 board 代码直接 hardcode 返回 `SYSCTL_BOOT_SDIO1`，根本不读这个寄存器（见 4.2 节），所以 QEMU 把 strap 设成 NOR 也没用，必须改 board 配置或换 EVB。

- **K230 firmware header**：K230 私有的 528 字节镜像头，魔数 `0x3033324b`（ASCII `"K230"`），包含 payload 长度、crypto type、hash/签名区域。作用类似 U-Boot 的 uImage header，但是 K230 私有，且 BootROM 和 SPL 都会读它。生成逻辑见 SDK 的 `firmware_gen.py`。

- **字节交换（endian swap）**：SPL 的最终镜像名 `swap_fn_u-boot-spl.bin` 中"swap"表示字节顺序被做了交换，通常是为了配合 SPI NOR 的 bit order 或 K230 内部总线要求。BootROM 加载后会做反向 swap 还原成可执行代码。完整 U-Boot 和 Linux 不做 swap，它们走的是 `k230_priv_gzip` 压缩路径。

- **DDR training**：DRAM 控制器需要根据 PCB 走线、温度、颗粒参数做信号校准（延迟、ZQ、读写均衡等），外部 DDR 才能稳定访问。SPL 里的 `ddr_init_training()` 就是做这件事。QEMU 没有真实 DRAM PHY，所以这一步要么跳过、要么用"DDR already ready"的兼容位（见 5.2 节）。
### 2.5.4 哪些阶段和真实冷启动相同

QEMU 当前已验证的成功路径是：

```text
QEMU reset stub  ->  -bios 直接把完整 U-Boot 放入 DDR  ->  U-Boot sf read  ->  OpenSBI/Linux
```

**相同的部分**：从"完整 U-Boot 在 DDR 中开始执行"之后的所有事情都和真实冷启动一致——

- U-Boot 通过 Standard SPI 控制器读 Flash（`sf probe`/`sf read`）；
- 启动 OpenSBI；
- 启动 Linux。

这部分能跑通，说明 SPI NOR 数据通路本身是可用的。

**不同的部分**：U-Boot 进入 DDR 之前的所有事情都被跳过了——没有 BootROM 行为、没有 SPL、没有 DDR training、没有 firmware header 校验、没有启动介质选择。

换句话说，QEMU 当前路径等价于"假装 BootROM 和 SPL 已经完成了它们的工作，直接把完整 U-Boot 摆在 DDR 里"。这能验证后段数据通路，但**不能宣称完成了完整冷启动**。

### 2.5.5 "跳过"在 QEMU 内部是怎么实现的

QEMU 的 `-bios <file>` 参数做两件事：

1. 把 `file` 内容加载到 guest 物理内存（对 K230 是 DDR）；
2. 在 reset vector 生成一段小程序（reset stub），CPU 上电后先执行这段 stub，然后跳转到 `-bios` 文件的入口地址。

K230 的具体实现见 `hw/riscv/k230.c`：它调用 `riscv_setup_rom_reset_vec()` 生成 stub，stub 直接跳到 `-bios` firmware。**stub 不读 MTD Flash、不解析 K230 firmware header、不加载 `swap_fn_u-boot-spl.bin`**——这就是"跳过 BootROM"的本质。

第 4.2 节里 GDB 运行时跳过 DDR training 是另一种"跳过"：不修改 QEMU/SDK 文件，只在运行时把 `ddr_init_training()` 的返回值改成 0，让 SPL 以为 DDR 已经训练好了。这种跳过是临时诊断手段，不是正式方案。

### 2.5.6 为什么要分阶段"跳过"

完整冷启动需要实现：BootROM 行为 + SPL 依赖的所有硬件（DDR PHY、PUFS、GSDMA、PWR…）。一次性全做完工作量大、调试也难。所以本报告采用**逐段验证**策略：

1. 先用 `-bios 完整 U-Boot` 验证后段（已成功）；
2. 再用 `-bios SPL` 验证 SPL 能不能跑（跑到 DDR training 才挂，见 4.1 节）；
3. 再补 DDR-ready、PWR 等最小兼容模型；
4. 最后才补 functional BootROM。

这也是为什么第 3 节列出的"绕过列表"会随着每一步推进而缩短——每补一块，就少跳过一段。

## 3. 当前成功路径绕过了什么

现有成功记录见
[`k230-spi-flash-uboot-linux-quickstart.md`](k230-spi-flash-uboot-linux-quickstart.md)。
该验证明确覆盖：

```text
-bios 完整 U-Boot
  -> sf probe / sf read
  -> OpenSBI
  -> Linux
```

它绕过了：

1. 真实 BootROM；
2. BOOT strap 与启动介质选择；
3. BootROM 从 Flash 读取、校验、字节交换 SPL；
4. SPL；
5. DDR training；
6. SPL 从 Flash 加载完整 U-Boot；
7. K230 firmware header、OTP 策略和 payload hash 校验；
8. 完整 U-Boot 的 K230 私有 gzip 解压。

QEMU 当前在 BootROM 地址建立 64 KiB ROM region，但
[`hw/riscv/k230.c`](../../../qemu-camp-2026-k230/hw/riscv/k230.c) 最终调用
`riscv_setup_rom_reset_vec()` 生成的是跳转到 `-bios` firmware 的 reset stub。它不会读取
MTD Flash，不解析 K230 firmware header，也不会加载 `swap_fn_u-boot-spl.bin`。

## 4. 动态实验记录

### 4.1 实验一：直接以现有 CanMV SPL 作为 `-bios`

使用 SDK 已构建的 `spl/u-boot-spl` 启动，实际输出为：

```text
U-Boot SPL 2022.10 ...
Unhandled exception: Store/AMO access fault
EPC:  0x80305c64
TVAL: 0x9a04017c
```

`addr2line` 将 `0x80305c64` 定位到：

```text
ddr_init_board
board/canaan/k230_canmv/gen_pilpddr3_2133.c:345
```

QEMU 的 DDRC_CFG 占位窗口是 `0x98000000`、大小 `0x02000000`，结束地址为
`0x9a000000`。SPL 访问的 `0x9a04017c` 已越过该窗口，因此产生 access fault。

结论：

- **SPL 本身可以在当前 QEMU 执行，并真实进入 DDR 初始化。**
- **当前 DDRC/PHY 地址覆盖和寄存器行为不足以运行 SDK 原始 DDR training。**

### 4.2 实验二：仅在 GDB 中跳过 DDR training

不修改 SDK/QEMU 文件，只在运行时让 `ddr_init_training()` 返回 0，输出继续到：

```text
Loading Environment from MMC...
MMC: no card present
...
uboot boot failed
```

原因不是 QEMU BOOT 寄存器默认值，而是 CanMV board 文件
[`board.c`](../../../k230_sdk/src/little/uboot/board/canaan/k230_canmv/board.c)
直接覆盖了 weak 实现：

```c
sysctl_boot_mode_e sysctl_boot_get_boot_mode(void)
{
    return SYSCTL_BOOT_SDIO1;
}
```

结论：**当前 CanMV defconfig 不会因为 BOOT strap 为 `00` 而选择 SPI NOR。**

### 4.3 实验三：运行时强制 `g_bootmod = SYSCTL_BOOT_NORFLASH`

强制 SPL 选择 NOR 后，输出为：

```text
Loading Environment from SPIFlash...
Invalid bus 0 (err=-19)
...
spl_board_init_f() failed: -19
```

原因是 CanMV 控制 DT 没有启用 `spi0@91584000`。公共
[`k230.dtsi`](../../../k230_sdk/src/little/uboot/arch/riscv/dts/k230.dtsi)
默认把该节点标为 `status = "disabled"`，CanMV DTS 没有覆盖它。

### 4.4 实验四：运行时把 SPL DT 中的 `disabled` 改成 `okay`

SPI bus 被 U-Boot DM 识别后，错误推进为：

```text
dw_spi spi@91584000: Invalid chip select 0:0 (err=-19)
```

这是因为仅启用 controller 仍不够；SPL 内嵌 DT 中还缺少 CS0 的 `spi-flash@0` child。
`spi_flash_probe_bus_cs()` 需要已有的从设备节点。

结论：完整冷启动需要修改或更换 **SPL 自己的控制 DT**，只修改完整 U-Boot 的 DT
不能解决问题。

### 4.5 PWR 空轮询

在进入 DDR 前，SPL 的
[`k230_spl.c`](../../../k230_sdk/src/little/uboot/board/canaan/common/k230_spl.c)
会调用 `device_disable()`，检查 PWR 状态寄存器的 bit 0。当前 QEMU 的 PWR 是
unimplemented device，读取为 0，因此循环会一直执行到计数器从 1,000,000 递减到 0。

对应的 `/tmp/k230-spl-unimp.log` 有 1,000,399 行。它最终能退出，所以不是绝对硬阻塞，
但会制造严重的启动延迟和无意义日志。

### 4.6 EVB 配置尝试

SDK 官方 `k230_evb` 配置具有两个重要差异：

- [`k230_evb.dtsi`](../../../k230_sdk/src/little/uboot/arch/riscv/dts/k230_evb.dtsi)
  原生启用 `spi0`，并声明 `spi-flash@0` 和 `u-boot,dm-pre-reloc`；
- [`k230_evb/board.c`](../../../k230_sdk/src/little/uboot/board/canaan/k230_evb/board.c)
  没有覆盖 `sysctl_boot_get_boot_mode()`，会使用
  [`sysctl.c`](../../../k230_sdk/src/little/uboot/arch/riscv/cpu/k230/sysctl.c)
  中读取 `SOC_BOOT_CTL[1:0]` 的 weak 实现。

这说明“改用官方 EVB defconfig”可能同时解决启动模式和 SPL SPI DT 问题，而且不需要
修改这两处 SDK 语义代码。

本次已在 `/tmp/k230-uboot-evb` 生成 EVB `.config`，但系统 GCC 13 需要显式加入
`-march=rv64imac_zicsr_zifencei`，继续构建后仍受该 SDK U-Boot 构建/链接环境影响，
没有得到可运行的 EVB SPL。因此这一项只能写为 **待验证候选方案**，不能写成已通过。

## 5. Gap 矩阵

| Gap | 阻塞级别 | 证据 | 归属 | 最小处理方向 |
|---|---|---|---|---|
| BootROM 不从 Flash 加载 SPL | 硬阻塞 | QEMU 静态确定 | QEMU | 实现功能型 BootROM loader |
| 真实 BootROM binary/完整协议缺失 | 目标边界 | 仓库与 TRM 调查 | 外部资料 | 不能宣称真实 ROM 等价 |
| DDR training 越界访问 `0x9a04017c` | 硬阻塞 | 动态已验证 | QEMU | 返回 DDR 已初始化状态，或完整建模 DDRC/PHY |
| CanMV 固定选择 SDIO1 | 硬阻塞 | 动态已验证 | SDK board config | 改用 EVB，或最小修改 CanMV boot mode |
| CanMV SPL DT 禁用 SPI0 | 硬阻塞 | 动态已验证 | SDK DTS | 改用 EVB，或启用 SPI0 |
| CanMV SPL DT 缺 `spi-flash@0` | 硬阻塞 | 动态已验证 | SDK DTS | 增加 CS0 SPI NOR child |
| PWR 状态位恒为 0 | 性能/兼容阻塞 | 动态已验证 | QEMU | 提供最小 power-off complete 状态 |
| 非安全镜像仍读 OTP | 后续硬依赖 | 静态确定 | QEMU/SDK | 最小 OTP 模型，或 SDK 兼容路径 |
| 非安全镜像使用 PUFS SHA-256 | 后续硬阻塞 | 静态确定 | QEMU/SDK | 建模 PUFS DMA/HMAC，或改用软件 SHA-256 |
| 完整 U-Boot 默认私有 gzip | 后续硬阻塞 | 静态确定 | QEMU/SDK 打包 | 建模硬件解压，或打包为 `IH_COMP_NONE` |
| Linux 默认私有 gzip | 后续硬阻塞 | 静态确定 | QEMU/SDK 打包 | 同上；当前未压缩 Linux 已规避 |
| RMU/CMU/IOMUX 多为占位 | 兼容性风险 | QEMU 静态确定 | QEMU | 按实际轮询点补最小状态，不追求无关全模型 |

### 5.1 BootROM：根本缺口

QEMU 目前有 BootROM 地址空间，但没有 K230 BootROM 行为。为了让路径尽量接近真实启动，
推荐在 `0x91200000` 放置一段由 C/汇编构建的 RISC-V guest ROM stub，而不是在 QEMU
machine 初始化阶段直接把 SPL payload 注入 SRAM。该 stub 至少需要：

1. 从 BOOT 配置确定 SPI NOR；
2. 从 MTD Flash 偏移 `0x0` 读取首级镜像；
3. 识别 K230 528 字节 firmware header；
4. 处理 `swap_fn_u-boot-spl.bin` 的字节交换格式；
5. 做长度、魔数和至少非安全 SHA-256 完整性检查；
6. 把 SPL payload 放到 `0x80300000` 并跳转。

ROM stub 应通过现有 SPI0/DW SSI 寄存器访问已经连接到 MTD backend 的 SPI NOR，从而让
首级加载也经过 controller、CS0 和 Flash 命令路径。不建议让 host-side machine 代码直接
读取 MTD backend 后复制 SPL；后者可以作为早期调试台阶，但它会绕过 BootROM -> SPI
controller -> Flash 这一段，不能作为本文所说的“尽量真实冷启动”最终结论。

由于真实 BootROM 的首级 SPI 命令序列没有公开，guest stub 使用的是“当前模型可表达的
标准 SPI NOR 读取协议”。它能验证逐级加载和现有 SPI 数据通路，但仍不能证明与芯片 ROM
使用完全相同的 opcode、时序或错误恢复策略。

但必须在设备说明和日志中明确写成 **functional BootROM**。没有真实 ROM binary 或完整
公开协议时，不能声称复现了芯片 ROM 的 SPI 命令序列、异常策略、secure boot 策略或时序。

### 5.2 DDR：优先提供“已经初始化”的虚拟平台契约

CanMV board 的 `ddr_init_training()` 已经留有兼容入口：

```c
if (readl(0x980001bc) & 0x1)
    return 0;
```

QEMU 的 DDR RAM 在 CPU 执行 SPL 前已经可访问，虚拟平台没有真实 DRAM PHY 需要训练。
因此最小且语义清楚的方案，是把当前 `ddrc_cfg` 占位替换为极小兼容模型，使
`0x980001bc bit0` 返回 1。

这比修改 SDK 直接跳过函数更少侵入，也符合 YAGNI：只表达虚拟 DDR 已 ready，不伪造
数千个 training 寄存器。

如果目标改成“原样执行 DDR training”，工作量会明显扩大：当前不仅缺少状态位，而且
SDK 会访问 `0x9a000000` 之外的 PHY/训练区域，并对大量寄存器进行写入和轮询。只扩大
MMIO 窗口虽然能消除首个 access fault，却不能保证训练流程完成。

### 5.3 启动模式与 SPL DT

对于当前 CanMV 配置，QEMU 把 BOOT strap 设为 SPI NOR 也无效，因为 board 代码根本不读
该寄存器。这里有两种最小方案：

- **优先候选：使用 SDK 官方 EVB defconfig。** 启动模式和 SPI child 已在官方配置中存在，
  但本次尚未完成动态验证。
- **保留 CanMV：做两个小改动。** 让 boot mode 使用公共 weak 实现或返回 NOR，并在
  CanMV DTS 中启用 SPI0、增加 `spi-flash@0`。该路径的缺口已通过运行时逐项证明。

不能把“由 QEMU 在加载时偷偷改 SPL DT”作为正式方案。那会隐藏 SDK 配置事实，也使最终
镜像无法脱离特定 QEMU hack 复现。

### 5.4 PUFS/OTP：非安全镜像也不能直接忽略

[`k230_img.c`](../../../k230_sdk/src/little/uboot/board/canaan/common/k230_img.c)
对 `NONE_SECURITY` 镜像仍会：

1. 调用 `cb_pufs_read_otp()` 读取 product misc，判断是否允许非安全启动；
2. 调用 `cb_pufs_hash()` 对 payload 做 SHA-256；
3. 与 firmware header 中的 hash 比较。

而 `CONFIG_K230_PUFS` 在 `k230_board_common.h` 中无条件定义。PUFS 驱动访问：

- 基址 `0x91210000`；
- DMA offset `0x000`；
- HMAC/hash offset `0x800`；
- OTP/RT offset `0x3000`。

SHA 路径会配置内部 DMA、启动运算并轮询 DMA busy/status。当前 QEMU 对整个 security 窗口
仅调用 `create_unimplemented_device()`，没有生成 digest 的能力。

最小选择：

- **SDK 零改方向**：QEMU 实现最小 OTP + SHA-256 兼容模型；
- **QEMU 少建模方向**：SDK 取消 `CONFIG_K230_PUFS`，使用代码中已有的
  `sha256_csum_wd()` 软件分支，同时保留 hash 比较。

第二种不是“取消完整性校验”，只是把计算后端从 PUFS 换成软件，语义变化较小。但 OTP
的“是否允许非安全启动”读取仍需单独处理，不能仅切换 SHA 后端就认为全部解决。

### 5.5 私有 gzip、GSDMA 与 RMU

SDK 的 `k230_gzip()` 把 gzip header 的 compression method 字节从 `0x08` 改成 `0x09`。
[`unzip.c`](../../../k230_sdk/src/little/uboot/arch/riscv/cpu/k230/unzip.c)
检测到 `0x09` 后进入 `k230_priv_unzip()`，访问：

- GSDMA：`0x80800000`；
- decomp-gzip：`0x80808000`；
- RMU reset/status。

当前 QEMU 的 GSDMA、decomp-gzip、RMU 都是 unimplemented device，因此 SDK v2.0 默认压缩
镜像不能原样通过该阶段。

已有 loader 并不强制 gzip。
[`k230_img.c`](../../../k230_sdk/src/little/uboot/board/canaan/common/k230_img.c)
同时支持：

```text
IH_COMP_GZIP -> gunzip()
IH_COMP_NONE -> memmove()
```

所以最小方案是只改镜像打包：

- 完整 U-Boot legacy uImage 使用 `-C none`，payload 不经过 `k230_priv_gzip`；
- Linux system 同样使用 `-C none`；
- firmware header、hash 校验和现有 loader 逻辑保持不变。

当前“未压缩 Linux”只解决了第二项；若 SPL 仍从 `0x080000` 读取默认
`fn_ug_u-boot.bin`，仍会在启动完整 U-Boot 前进入硬件 gzip 路径。

### 5.6 CMU、RMU、IOMUX 的边界

当前 CMU、RMU、IOMUX 多数写入由占位设备吞掉。它们暂时可能不阻塞，是因为 QEMU 的
SPI/UART/RAM 并不真正受 guest clock gate、reset tree 和 pinmux 控制。

这可以作为功能型虚拟平台的阶段性兼容策略，但不能解释成硬件等价。实施时应通过日志和
动态执行找到真正被轮询的状态位，再补最小模型；不应在没有消费者的情况下预先实现整套
clock/reset/pinmux。

## 6. 两条可实施路线

### 6.1 路线 A：功能型完整冷启动，尽量少改 SDK

推荐目标：

```text
CPU reset
  -> QEMU functional BootROM
  -> Flash 0x0 的 SPL
  -> SPL
  -> Flash 0x80000 的未压缩完整 U-Boot
  -> Flash 0xfc0000 的未压缩 Linux system
  -> OpenSBI/Linux
```

QEMU 侧最小实现：

1. 在 `0x91200000` 执行、并通过现有 SPI0 读取 Flash 的 functional BootROM stub；
2. BOOT strap/BOOT register 的 SPI NOR 默认值；
3. `0x980001bc bit0 = 1` 的 DDR-ready 兼容模型；
4. PWR power-off complete 最小状态；
5. 继续复用现有 Standard SPI NOR、HI_SYS、XIP 和 SSI IDMA 模型。

SDK 侧最小改动，二选一：

- 尝试使用官方 EVB defconfig，动态验证其 SPL；或
- 保留 CanMV，仅修正 boot mode 和 SPL DT。

另外需要两项打包/兼容选择：

- 完整 U-Boot 和 Linux 都打成 `IH_COMP_NONE`；
- PUFS 选择“QEMU 最小模型”或“SDK 软件 SHA + 最小 OTP 策略兼容”。

这条路线可以做到“从 reset 开始、从同一张 SPI Flash 镜像逐级加载”的功能闭环，SDK 改动
可收敛在板级配置和镜像打包，不需要改 loader 主流程。

### 6.2 路线 B：SDK v2.0 官方 SPI-NOR 镜像原样启动

如果“原样”指 SDK 默认私有 gzip、PUFS hash 和板级初始化都不修改，那么 QEMU 还需要：

1. functional BootROM 及首级镜像格式处理；
2. DDRC/PHY training 的足够模型，或至少能无修改地满足 SDK ready 契约；
3. PWR/RMU 的状态和轮询语义；
4. PUFS OTP、内部 DMA、HMAC/SHA-256；
5. GSDMA 与 decomp-gzip；
6. 与所选官方板级配置一致的 SPI DT 和启动模式。

这里还要区分板型：

- **EVB 官方 SPI NOR 配置**：源码条件较完整，但本次尚未动态验证构建产物。
- **当前 CanMV defconfig 完全零改**：不可行。它硬编码 SDIO1，且 SPL DT 禁用 SPI0、
  缺少 SPI NOR child；这些不是通过正常 BOOT strap 能解决的 QEMU 外设缺口。

因此，“SDK 零改”若要成立，应定义为“选择 SDK 已提供的 EVB SPI NOR board profile，镜像
内容不打补丁”，不能把现有 CanMV profile 也算作零改可启动。

## 7. 推荐实施顺序

建议按每一步都能产生新证据的顺序推进：

1. **先定板型。** 优先完成 EVB SPL 的可重复构建；若工具链成本过高，再对 CanMV 做最小
   boot mode + DTS 修改。
2. **先用 `-bios SPL` 打通 SPL 到完整 U-Boot。** 暂时不写 BootROM，加入 DDR-ready、
   PWR 状态，并使用未压缩 U-Boot，逐项暴露 PUFS/OTP 后续行为。
3. **处理非安全 firmware header 校验。** 在“最小 PUFS 模型”和“软件 SHA”之间选一个，
   保留真实 hash 比较，不直接跳过校验。
4. **打通 SPL -> 完整 U-Boot -> 未压缩 Linux。** 这一步确认除 BootROM 外的整条链路。
5. **最后实现 functional BootROM。** 从同一张 SDK 布局 Flash 的 `0x0` 读取 SPL，替换
   `-bios SPL`，形成真正的 reset-to-Linux 功能闭环。
6. **再决定是否值得实现硬件 gzip/PUFS。** 只有“默认 SDK 镜像原样启动”是明确需求时，
   才投入 GSDMA、decomp-gzip 和 PUFS 的完整建模。

这个顺序能把复杂问题拆成可验证边界：先证明 SPL 后半段，再补最前端 BootROM，避免同时
调试 ROM、DDR、SPI、PUFS 和 gzip。

## 8. 最终判断

回答最初问题：**有 gap，而且有多个硬 gap。**

当前离“功能型完整冷启动”最近的现实方案是：

```text
functional BootROM
  + EVB 配置或最小 SPI-enabled CanMV SPL DT
  + QEMU DDR-ready bit
  + QEMU 最小 PWR 状态
  + 软件 SHA 或最小 PUFS/OTP
  + 未压缩完整 U-Boot
  + 未压缩 Linux
  + 现有 Standard SPI NOR 模型
```

其中已经通过动态实验确认的首要阻塞顺序是：

```text
DDR training 越界
  -> CanMV 固定 SDIO1
  -> SPL DT 禁用 SPI0
  -> SPL DT 缺少 spi-flash@0
```

后续由源码确定的阻塞是 PUFS/OTP 和私有 gzip。BootROM 则是整条冷启动入口的根本缺口。

如果允许少量 SDK 板级配置和打包修改，QEMU 侧工作可以收敛为 BootROM、DDR-ready、PWR
以及可选的最小 OTP/PUFS 兼容，不需要立即完整仿真 DDR PHY 和硬件解压器。

如果要求 SDK v2.0 默认 SPI-NOR 镜像原样启动，则工作量会扩大到 PUFS、GSDMA、gzip、
RMU/PWR 和更多 DDR 语义；同时必须选用本身支持 SPI NOR 的官方板级 profile。这个目标
不能被描述成“只差 BootROM”。
