# K230 SPI/QSPI V2 第一批上游 Series：Cover Letter（中英文草案）

最后更新：2026-08-02

本文是 V2 第一批 `0/5` 的 cover letter 草案。英文版本可作为邮件正文使用，
中文版用于确认内容。

基线：`b428fe036233cbd15d37e3c027ab6ca4d3661a80`

建议邮件主题：

```text
[PATCH v2 0/5] hw/ssi: Add DesignWare SSI standard PIO support for K230
```

## English Cover Letter

```text
Hello,

This series adds a reusable Synopsys DesignWare SSI controller model and
uses it to model the three SSI instances on the Kendryte K230 machine.

This series supersedes the previous K230-specific SSI series:

  https://lore.kernel.org/qemu-devel/cover.1785064312.git.flamboyant.h.01@gmail.com/

Following review of v1, the controller has been split into a reusable
DesignWare SSI model and a K230 integration layer. The controller no longer
depends on K230 types, addresses, or HI_SYS state. K230-specific instance
profiles, MMIO mapping, PLIC routing, and flash attachment now live in the
K230 machine code.

The v1 implementation combined DesignWare SSI behaviour and K230-specific
SoC integration in one device. That made the reuse boundary unclear for
other SoCs using the same IP. This revision makes that boundary explicit:
the generic model implements the documented controller behaviour, while the
K230 machine supplies only its instance configuration and SoC connections.

The implemented controller scope is deliberately limited to Standard
single-line SPI PIO: configurable chip selects and FIFO depth, four TMOD
modes, FIFO and status handling, Standard interrupt state, and migration
state for the implemented data path. Enhanced SPI, internal DMA/IDMA, and
XIP transactions are deferred to follow-up series.

K230 has nine SSI interrupt source lines per instance. This series wires all
nine lines to preserve the documented PLIC topology. TXU, DONE, and AXIE
remain deasserted because their transfer engines are outside the Standard
PIO scope modeled here.

The final patch adds an optional M25P80-compatible SPI NOR flash on logical
spi0 CS0. It supports and tests Standard 1-1-1 accesses only; the enhanced
SPI and IDMA boot paths are outside the scope of this series.

Changes since v1:

* Replace the K230-specific controller type with a reusable DesignWare SSI
  model configured through num-cs, fifo-depth, and imr-reset properties.
* Keep K230 instance profiles, MMIO mapping, PLIC routing, and flash
  attachment in the K230 machine code.
* Limit this revision to Standard PIO, FIFO, interrupt, and 1-1-1 SPI NOR
  support; defer enhanced SPI, internal DMA/IDMA, and XIP to follow-up
  series.

Testing on the current V2 development tree:

* riscv64-softmmu build succeeds.
* K230 SSI qtest: 14/14 pass, covering register/reset contracts, Standard
  PIO TMOD modes, FIFO behaviour, interrupt state, PLIC routing, Standard
  1-1-1 flash identification and fixed-address reads, and
  unsupported-register semantics.
* U-Boot Standard SPI payload boot: U-Boot loads OpenSBI, Linux, initramfs,
  and the DTB from a W25Q256 flash and reaches a Linux initramfs shell.
  Standard sf probe, erase, write, readback, and compare also pass.
* Linux 5.10.4 Standard SPI PIO: dw_spi_mmio and spi-nor probe successfully;
  the MTD test partition completes erase, write, readback, and byte-wise
  comparison (K230_MTD_TEST_PASS mode=standard).

Kangjie Huang (5):
  hw/ssi: Add Synopsys DesignWare SSI standard PIO controller
  hw/ssi: Add DesignWare SSI standard interrupt support
  hw/riscv/k230: Instantiate DesignWare SSI controllers
  hw/riscv/k230: Route SSI interrupts to the PLIC
  hw/riscv/k230: Attach a standard SPI NOR flash

base-commit: b428fe036233cbd15d37e3c027ab6ca4d3661a80
--
2.43.0
```

## 中文版（仅供阅读）

```text
您好：

本系列新增可复用的 Synopsys DesignWare SSI 控制器模型，并用其建模
Kendryte K230 机器上的三个 SSI 实例。

本系列是 K230 SSI 工作的 v2，并取代上一版 K230 专有 SSI series：

  https://lore.kernel.org/qemu-devel/cover.1785064312.git.flamboyant.h.01@gmail.com/

根据 v1 的 review 意见，控制器现已拆分为可复用的 DesignWare SSI 模型与
K230 集成层。通用控制器不再依赖 K230 类型、地址或 HI_SYS 状态；
K230 专有的实例 profile、MMIO 映射、PLIC 路由和 Flash 挂接均位于
K230 machine 代码。

v1 将 DesignWare SSI 的通用行为与 K230 的 SoC 集成写在同一个设备中，
使得其他使用同一 IP 的 SoC 难以直接复用，也不易判断哪些行为属于
IP 本身、哪些行为属于 SoC 集成。本次修订明确了这个边界：通用模型
实现有文档证据支持的控制器行为，K230 machine 只提供实例配置和 SoC 连接。

控制器实现范围有意限制为 Standard 单线 SPI PIO：可配置片选和 FIFO 深度、
四种 TMOD 模式、FIFO/状态处理、Standard 中断状态，以及已实现数据路径所需的
迁移状态。enhanced SPI、internal DMA/IDMA 和 XIP transaction 留给后续 series。

K230 的每个 SSI 实例有 9 路中断源。本系列连接全部 9 路，以保持文档规定的
PLIC 拓扑。TXU、DONE 和 AXIE 对应的传输引擎不在当前 Standard PIO 范围内，
因此保持低电平。

最后一个补丁在逻辑 spi0 CS0 挂接可选的 M25P80 兼容 SPI NOR Flash。它只支持
并测试 Standard 1-1-1 访问；enhanced SPI 和 IDMA 启动路径不在本 series 范围内。

相较 v1 的主要变更：

* 用可复用的 DesignWare SSI 模型替换 K230 专有控制器类型，并通过
  num-cs、fifo-depth 和 imr-reset 属性配置实例。
* 将 K230 实例 profile、MMIO 映射、PLIC 路由和 Flash 挂接保留在
  K230 machine 代码中。
* 本次 revision 仅包含 Standard PIO、FIFO、中断和 1-1-1 SPI NOR 支持；
  enhanced SPI、internal DMA/IDMA 和 XIP 推迟到后续 series。

当前 V2 开发树上的测试：

* riscv64-softmmu 构建成功。
* K230 SSI qtest：14/14 通过，覆盖寄存器/复位契约、Standard PIO TMOD 模式、
  FIFO 行为、中断状态、PLIC 路由、Standard 1-1-1 Flash 识别与固定地址读取，
  以及未支持寄存器语义。
* U-Boot Standard SPI payload boot：U-Boot 从 W25Q256 Flash 加载 OpenSBI、
  Linux、initramfs 和 DTB，并进入 Linux initramfs shell。Standard sf probe、
  erase、write、readback 和 compare 也全部通过。
* Linux 5.10.4 Standard SPI PIO：dw_spi_mmio 和 spi-nor 成功 probe；MTD 测试
  分区完成 erase、write、readback 和逐字节比较
  （K230_MTD_TEST_PASS mode=standard）。

Kangjie Huang（5 个补丁）：
  hw/ssi：新增 Synopsys DesignWare SSI 标准 PIO 控制器
  hw/ssi：新增 DesignWare SSI 标准中断支持
  hw/riscv/k230：实例化 DesignWare SSI 控制器
  hw/riscv/k230：将 SSI 中断路由至 PLIC
  hw/riscv/k230：挂接标准 SPI NOR Flash

base-commit: b428fe036233cbd15d37e3c027ab6ca4d3661a80
--
2.43.0
```

## 发送前替换项

正式发送前，将英文邮件正文中的以下内容更新为最终事实：

1. `v2`、补丁数、基线 commit 和 Git 版本；
2. 最终 qtest 数量和结果；
3. `scripts/checkpatch.pl`、`git diff --check` 和 Flash qtest 结果；
4. `git format-patch --cover-letter` 自动生成的 diffstat。

不要在 cover letter 中加入以下宣称：

- ROM/SPL 从 Flash 加载完整 U-Boot；
- enhanced SPI、Quad SPI、IDMA 或 XIP 已可用；
- SDK 默认 U-Boot 或 Linux 内核无需测试 DT/镜像即可直接使用。
