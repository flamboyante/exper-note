# K230 SPI NOR 冷启动知识手册

记录时间：2026-08-05

## 文档定位

本文保存 K230 SPI NOR 冷启动中相对稳定的知识：启动层次、Guest 固件职责、镜像格式、
地址布局、QEMU 与真实硬件的差异，以及常用术语。动态实验、当前阻塞和实施顺序见
[冷启动 Gap 调查与实施计划](./k230-cold-boot-gap-investigation.md)。

本文不把任何 QEMU qtest 或单阶段启动结果写成“真实 BootROM 等价”。

## 1. 完整启动链

K230 SPI NOR 的目标启动链为：

```text
CPU reset
  -> BootROM
  -> SPI NOR 0x000000 的 SPL
  -> DDR 初始化完成后的完整 U-Boot
  -> OpenSBI
  -> Linux
```

各级职责不同：

| 阶段 | 运行位置 | 主要职责 |
|---|---|---|
| BootROM | 片内 ROM，QEMU 地址窗口为 `0x91200000` | 采样启动模式、选择介质、加载首级镜像 |
| SPL | SRAM，SDK 常用链接地址 `0x80300000` | pinmux/时钟/复位、DDR 初始化、加载完整 U-Boot |
| 完整 U-Boot | DDR，常用系统地址从 `0x08000000` 开始 | 读取 Linux system package、准备 DTB、启动 OpenSBI |
| OpenSBI | DDR | 提供 SBI 服务，完成 M-mode 到 S-mode 的启动交接 |
| Linux | DDR | 建立页表、初始化驱动、启动 init/rootfs |

真实 BootROM 的具体指令、SPI opcode、异常处理和安全策略没有公开完整规范。本文涉及
BootROM 的装载动作时，默认指“功能型 BootROM”的目标行为，而不是 ROM 指令级复现。

## 2. SoC 启动基础

### 2.1 reset vector

reset vector 是 CPU 复位后第一条指令所在的地址。它不是 Linux 入口，也不是任意
`-bios` 文件的物理地址。

真实芯片通常让 reset vector 指向片内 BootROM：

```text
reset vector -> BootROM -> 选择启动介质 -> 加载 SPL -> 跳转
```

QEMU 的 `-bios` 启动通常由 machine 代码把 firmware 放入 Guest 内存，再在 ROM 地址
生成一段 reset stub 跳转到 firmware。这会绕过真实 BootROM 的介质读取。

### 2.2 boot strap 与 boot mode

真实硬件在上电时采样 BOOT 引脚，QEMU 则用寄存器或 machine 属性表达相同选择。
K230 常见的 `BOOT1:BOOT0 = 00` 表示 SPI NOR，但最终 Guest 代码可能还会读取 SoC
boot-mode 寄存器或使用板级 hardcode。

必须区分三层选择：

```text
BootROM 选择 SPI NOR
SPL      选择 SPI NOR
U-Boot   environment 选择 SPI NOR
```

某一层选择错误时，其他层的 SPI 模型正确也不能补救。

### 2.3 SRAM 与 DDR

上电时外部 DDR 尚不可用，SPL 必须在片内 SRAM 中运行。SPL 不能假设完整 U-Boot 那样
拥有很大的可用内存；它只应完成必要初始化和下一阶段装载。

DDR training 不是简单的“打开一个开关”，物理芯片需要根据内存颗粒、走线和温度执行
延迟、读写均衡、ZQ 等校准。QEMU 可以选择：

```text
功能型：让 Guest 看到 DDR 已 ready
逼真型：建模 DDRC/PHY 寄存器、训练 mailbox 和 DFI 握手
```

两者都应明确记录，不能把虚拟 ready 位写成真实 PHY training 已复现。

### 2.4 多级加载的原因

多级加载是资源分层：

```text
BootROM：容量小，适合介质选择和首级复制
SPL：    在 SRAM 中初始化 DDR
U-Boot：在 DDR 中处理复杂镜像、命令和文件系统
Linux：  接管完整系统资源
```

这也是为什么“把完整 U-Boot 直接放进 RAM”不能替代冷启动：它跳过了最需要验证的
BootROM、SPL 和 DDR 阶段。

## 3. SDK SPI NOR 镜像

SDK `genimage-spinor.cfg` 常见的 32 MiB SPI NOR 布局是：

```text
0x000000  swap_fn_u-boot-spl.bin
0x080000  fn_ug_u-boot.bin
0x1e0000  environment
0xfc0000  linux_system.bin
```

实际分区大小、rootfs 和业务区域以所选 board 的 `genimage-spinor.cfg` 为准。

### 3.1 SPL 文件名中的 swap

SDK 打包脚本先给 SPL 加 firmware header，再调用 `endian-swap.py` 生成
`swap_fn_u-boot-spl.bin`。这说明某一级加载逻辑需要将其还原为 CPU 可执行字节序；
但公开资料尚不能确认真实 BootROM 的还原算法、粒度和 SPI 读取细节。

因此 QEMU 功能型 BootROM 可以实现一种明确记录的还原流程，但不能声称与真实 ROM 完全
相同。

### 3.2 `k230-boot-assets` 与 SDK

`k230-boot-assets` 主要保存 SDK 或 QEMU 适配后的后段载荷：

| 文件 | 用途 |
|---|---|
| `common/u-boot` | SDK 完整 U-Boot ELF，直接用 `-bios` 启动 |
| `common/fw_jump.uImage` | SDK OpenSBI 包装成 U-Boot `bootm` 可识别的 image |
| `buildroot/*/Image` | SDK Linux 5.10.4 的 QEMU 适配内核 |
| `buildroot/*/rootfs.cpio.gz` | SDK Buildroot initramfs |
| `yocto/*` | mainline Linux 6.18.28 测试载荷 |

这些文件可以验证完整 U-Boot 之后的路径，但不包含真实 Flash 0 的 SPL 分区。

## 4. 两层 firmware 格式

K230 文件通常包含两层 header：

```text
K230 firmware header
  -> 保护整个 payload，包含 magic、length、crypto type、hash/签名

U-Boot legacy uImage header
  -> 描述 image name、压缩类型、load 地址和 entry 地址
```

### 4.1 K230 firmware header

SDK 公共头文件定义的魔数是：

```text
0x3033324b = ASCII "K230"
```

header 总大小为 528 字节，包含 payload 长度、`crypto_type` 和校验区域。
`firmware_gen.py -n` 生成 `NONE_SECURITY` 形式，通常携带 payload SHA-256。

SPL 加载完整 U-Boot 时会检查 header magic、长度和完整性结果；magic 错误、读取截断或
hash 不匹配都会拒绝下一阶段。

### 4.2 firmware header 不是 PUFS 专用格式

非安全 header 本身只表达 SHA-256 摘要。SDK 默认 SPL 选择 PUFS 硬件作为计算后端，
但同一个 header 可以使用软件 `sha256_csum_wd()` 验证。

因此：

```text
原始 fn_ug-u-boot.bin 可以保留
SDK 默认 SPL 可能因缺少 PUFS 而失败
QEMU 专用 SPL 可切换到软件 SHA，仍保留真实 hash 比较
```

安全镜像 `fa_*`、`fs_*` 还涉及 OTP、RSA/AES-GCM 或 SM2/SM3/SM4，不能用非安全路径
推导其可启动性。

### 4.3 uImage 的压缩类型

SDK loader 同时有两条分支：

```text
IH_COMP_GZIP -> gunzip()/K230 私有 gzip 路径
IH_COMP_NONE -> 直接 memmove()
```

第一轮调试可先将完整 U-Boot 和 Linux payload 设为 `IH_COMP_NONE`，隔离“读取/校验/跳转”
与“硬件解压”两个问题。header 和 SHA 校验不应因此删除。

## 5. Guest 固件职责

### 5.1 SPL

SPL 的典型顺序是：

```text
关闭不需要的设备/时钟
  -> 读取 boot mode
  -> pinmux 和板级初始化
  -> DDR training 或检查 DDR ready
  -> 初始化 SPI 控制器
  -> probe Flash
  -> 从 0x80000 读取完整 U-Boot
  -> 校验 K230 firmware header
  -> 解压/复制并跳转
```

SPL 的控制 DT 与完整 U-Boot 使用的 DT 不一定相同。调试 SPL 时必须检查 SPL 自己内嵌的
DT，而不是只给 Linux 传一个正确的 DTB。

### 5.2 完整 U-Boot

完整 U-Boot 在 DDR 中运行，负责：

- 重新初始化或复用 SPI NOR 总线；
- 读取 `linux_system.bin`；
- 校验 firmware header 和内部 uImage；
- 解压或复制 OpenSBI、Linux、rootfs、DTB；
- 设置 `/chosen`、bootargs 和 initrd 地址；
- 通过 `bootm` 将控制权交给 OpenSBI。

U-Boot prompt 能出现，只证明 SPL 已成功跳转到 U-Boot，不代表 Linux payload 已经可用。

### 5.3 OpenSBI 与 Linux

OpenSBI 通常在 M-mode 运行，为 Linux 提供 SBI console、timer、IPI 等服务，然后把控制
权交给 S-mode Linux：

```text
U-Boot bootm -> OpenSBI -> Linux S-mode entry
```

`fw_jump.uImage` 成功并不意味着 BootROM/SPL 成功，它只覆盖 U-Boot 之后的启动阶段。

## 6. QEMU 模型边界

### 6.1 第一轮通常需要

| 模块 | 用途 |
|---|---|
| 功能型 BootROM | 从 Flash 0 加载 SPL 并跳转 |
| DDRC/PHY 或 DDR-ready 契约 | 让 SPL 越过 DDR 阶段 |
| Standard DW SSI | SPL/U-Boot 读取 SPI NOR |
| SPI NOR backend | 提供真实 Flash 内容 |
| UART | 观察 SPL/U-Boot/OpenSBI 输出 |
| GSDMA/decomp-gzip | 使用 SDK 私有 gzip 时需要 |

### 6.2 先不必实现

第一轮 Standard 1-1-1、未压缩 payload 验证通常不需要：

```text
Enhanced QSPI
SSI IDMA
XIP window
完整 PUFS 安全启动
SDHCI
完整 CMU 时钟树
```

IOMUX、RMU、PWR 可按 Guest 实际访问逐步补齐。PWR 状态位恒为零通常只会造成有限次数
的忙等，不等同于永久死循环；真正的无界等待应重点检查 GSDMA、gzip、SPI FIFO 和 DDR
训练状态位。

## 7. Linux 兼容性

“U-Boot 读取 Linux 成功”和“Linux 到达 shell”是两个判定点。SDK vendor Linux 可能使用
T-HEAD 私有页表属性，通用 QEMU RISC-V MMU 不一定支持。因此可以先使用 QEMU 适配过的
Linux Image 验证：

```text
U-Boot -> OpenSBI -> kernel entry -> initramfs shell
```

不要因为出现 `Starting kernel` 或 kernel entry 就声称原始 vendor Linux 完整支持。

## 8. 与实施计划的关系

所有以下内容放在
[k230-cold-boot-gap-investigation.md](./k230-cold-boot-gap-investigation.md)：

- 当前基线和证据等级；
- 动态失败现场；
- DDRC/PHY、GSDMA/decomp-gzip 等新模块的当前状态；
- Gap 矩阵；
- EVB/CanMV 选择；
- 分阶段实施顺序；
- “功能型冷启动”与“SDK 原样镜像”的路线差异。

## 9. 常用判定规则

```text
direct boot 成功       != BootROM 冷启动成功
-bios U-Boot 成功      != SPL 成功
U-Boot sf read 成功    != BootROM/SPL 读取成功
kernel entry 出现      != Linux shell 成功
单个 qtest 通过        != 多级 Guest 启动链通过
```

记录实验时，至少标明：

```text
QEMU 基线和模块组合
SDK board/defconfig
Flash 镜像来源和分区布局
SPL load address
boot mode
firmware crypto_type 和压缩类型
最终达到的阶段：P1 U-Boot / P2 OpenSBI / P3 Linux shell
```
