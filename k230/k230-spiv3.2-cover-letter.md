# K230 SPI v3.2 Cover Letter

分支：`k230-spiv3.2`

基线：`f893c46c3931b3684d235d221bf8b7844ddbf1d7`（`upstream/master`）

提交数：10

## English cover letter

~~~text
Subject: [PATCH v3 00/10] hw/riscv: Add K230 SPI, QSPI, IDMA and XIP support

Hi,

This series adds the Kendryte K230 DesignWare SSI controllers and their
machine integration.

K230 contains three SSI instances with different capabilities. The SDK
numbers them according to the address map: spi0 is the SPI-OPI instance,
while spi1 and spi2 are QSPI0 and QSPI1. The series models the documented
register contract, standard FIFO-backed SPI transfers, interrupt routing,
Dual/Quad SDR transfers, SPI NOR attachment, synchronous internal DMA,
the HI_SYS SSI_CTRL wrapper, and the spi0 XIP read window.

The implementation is split so that the register model and machine
integration come first, followed by the PIO data path, interrupt support,
enhanced transfers, flash integration, IDMA, HI_SYS, and XIP. A single
K230 SSI qtest binary grows with the series and exercises each layer when
it becomes available.

Changes in v3.2:

* Drop all MAINTAINERS changes from the series.
* Use complete copyright, SPDX, TRM, and product-reference headers for
  the new SSI and HI_SYS source/header files.
* Add the copyright header to the K230 SSI qtest.
* Explain the SDK spi0/spi1/spi2 numbering relationship directly beside
  the machine routing table.
* Document why IDMA completes synchronously in the model.
* Document the SDK-specific 1-4-4 mode-byte handling.
* Clarify the TRM address-space conflict between HI_SYS SSI_CTRL and the
  controller-local DR2 offset.
* Refresh all ten commit messages and verification statements.

Testing:

* Built qemu-system-riscv64 at every commit with the riscv64-softmmu
  configuration.
* Ran the cumulative K230 SSI qtest after every commit that provides the
  test target. The final revision passes all ten scenarios:
  register-contract, pio-data-path, interrupt-controller, plic-routing,
  qspi-config, spi-nor, qspi-sdr, idma, hi-sys, and xip-read-window.
* Standard SPI: U-Boot detected W25Q256, loaded OpenSBI, Linux, initrd,
  and DTB with sf read, and reached the Linux initramfs shell.
* Quad SPI: with spi0 configured for 4-bit transfers, U-Boot erased a
  64 KiB sector, wrote and read back 256 bytes successfully, loaded all
  boot payloads from QSPI flash, and reached the Linux initramfs shell.
* XIP: U-Boot read the OpenSBI uImage header from 0xc0000000, verified
  its checksum, and reached the Linux initramfs shell through bootm.
* git diff --check reports no whitespace errors.
* checkpatch reports no errors. The file-addition warnings for patches
  1, 2, and 9 are expected because this revision intentionally contains
  no MAINTAINERS changes.

Kangjie Huang (10):
  hw/ssi: Add K230 DesignWare SSI register model
  hw/riscv/k230: Instantiate K230 SSI controllers
  hw/ssi: Implement K230 SSI FIFO and standard PIO transfers
  hw/ssi: Add K230 SSI interrupt controller
  hw/riscv: Route K230 SSI IRQs to the PLIC
  hw/ssi: Implement K230 enhanced QSPI transfers
  hw/riscv/k230: Attach SPI NOR flash to spi0
  hw/ssi: Implement K230 SSI internal DMA transfers
  hw/misc: Add K230 HI_SYS SSI control
  hw/ssi: Add K230 SSI XIP read window

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

## 中文审阅要点

- 系列保持 10 个补丁，不修改 `MAINTAINERS`。
- 1/10 引入 SSI 模型文件及完整文件头；2/10 引入机器实例化和 qtest；
  9/10 引入 HI_SYS 模型及完整文件头。
- 5/10 直接解释 SDK 编号与物理实例关系，不再只描述“路由表”。
- 8/10 明确同步 IDMA 是建模取舍，并解释 1-4-4 mode byte 的来源和
  wait-cycle 消耗。
- 9/10 明确 TRM 中 `HI_SYS_CONFIG + 0x068` 的 `SSI_CTRL` 与 SSI 控制器
  相对偏移 `0x068` 的 `DR2` 属于不同地址空间。
- 集成验证覆盖 Standard SPI、Quad SPI/IDMA 和 XIP 三条启动路径，均进入
  Linux initramfs shell。
