# K230 SPI v3.2 Cover Letter 中文版

分支：`k230-spiv3.2`

基线：`f893c46c3931b3684d235d221bf8b7844ddbf1d7`（`upstream/master`）

提交数：10

## 中文 cover letter

~~~text
主题：[PATCH 00/10] hw/riscv：新增 K230 SPI、QSPI、IDMA 和 XIP 支持

您好：

当前 k230 机器已建模 C908 内核、CLINT、PLIC、看门狗和 UART，但尚无法通过
SDK 存储启动。本系列新增 Kendryte K230 DesignWare SSI 控制器及其机器集成，
使 U-Boot 能够通过标准 PIO、QSPI IDMA 和 XIP 读取窗口从 SPI NOR 启动。

K230 包含三个能力不同的 SSI 实例。SDK 按照地址映射对它们编号：spi0 是
SPI-OPI 实例，spi1 和 spi2 分别是 QSPI0 和 QSPI1。本系列对已有文档说明的
寄存器访问约定、基于 FIFO 的标准 SPI 传输、中断路由、Dual/Quad SDR 传输、
SPI NOR 挂接、同步内部 DMA、HI_SYS SSI_CTRL 包装寄存器以及 spi0 XIP 只读
窗口进行建模。

本系列按照功能依赖拆分：首先加入寄存器模型和机器集成，随后依次加入 PIO
数据路径、中断支持、增强传输、Flash 集成、IDMA、HI_SYS 和 XIP。K230 SSI
qtest 使用一个测试程序随系列逐步扩展，并在每一层功能可用后对其进行验证。

测试：

* 在每个提交上使用 riscv64-softmmu 配置构建 qemu-system-riscv64。
* 在每个提供测试目标的提交上运行累积式 K230 SSI qtest。最终版本的十个场景
  全部通过：register-contract、pio-data-path、interrupt-controller、
  plic-routing、qspi-config、spi-nor、qspi-sdr、idma、hi-sys 和
  xip-read-window。
* 标准 SPI：U-Boot 识别到 W25Q256，通过 sf read 加载 OpenSBI、Linux、
  initrd 和 DTB，并进入 Linux initramfs shell。
* Linux：K230 SDK 内核已使用标准 RISC-V PTE 位重新构建；QEMU 的通用
  RISC-V MMU 未实现 T-HEAD C9xx 私有 MAEE 页表属性。
* Quad SPI：将 spi0 配置为 4 位传输后，U-Boot 成功擦除一个 64 KiB 扇区，
  写入并读回验证 256 字节数据，从 QSPI Flash 加载全部启动载荷，并进入
  Linux initramfs shell。
* XIP：U-Boot 从 0xc0000000 读取 OpenSBI uImage 头，验证其校验和，并通过
  bootm 进入 Linux initramfs shell。
* git diff --check 未报告空白符错误。
* checkpatch 未报告错误。

Kangjie Huang（10 个补丁）：
  hw/ssi：新增 K230 DesignWare SSI 寄存器模型
  hw/riscv/k230：实例化 K230 SSI 控制器
  hw/ssi：实现 K230 SSI FIFO 和标准 PIO 传输
  hw/ssi：新增 K230 SSI 中断控制器
  hw/riscv：将 K230 SSI IRQ 路由到 PLIC
  hw/ssi：实现 K230 增强 QSPI 传输
  hw/riscv/k230：将 SPI NOR Flash 挂接到 spi0
  hw/ssi：实现 K230 SSI 内部 DMA 传输
  hw/misc：新增 K230 HI_SYS SSI 控制
  hw/ssi：新增 K230 SSI XIP 读取窗口

 hw/misc/k230_hi_sys.c          |  169 ++++
 hw/misc/meson.build            |    1 +
 hw/riscv/Kconfig               |    2 +
 hw/riscv/k230.c                |  165 +++-
 hw/ssi/Kconfig                 |    4 +
 hw/ssi/k230_dw_ssi.c           | 1728 ++++++++++++++++++++++++++++++++++++++++
 hw/ssi/meson.build             |    1 +
 include/hw/misc/k230_hi_sys.h  |   55 ++
 include/hw/riscv/k230.h        |    9 +
 include/hw/ssi/k230_dw_ssi.h   |  111 +++
 tests/qtest/k230-dw-ssi-test.c |  917 +++++++++++++++++++++
 tests/qtest/meson.build        |    4 +-
 12 files changed, 3150 insertions(+), 16 deletions(-)

base-commit: f893c46c3931b3684d235d221bf8b7844ddbf1d7
-- 
2.43.0
~~~
