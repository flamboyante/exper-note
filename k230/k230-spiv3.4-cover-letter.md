# K230 SPI v3.4 Cover Letter

分支：`k230-spiv3.4`

基线：`f893c46c3931b3684d235d221bf8b7844ddbf1d7`（`upstream/master`）

提交数：11

## English cover letter

~~~text
Subject: [PATCH 00/11] hw/riscv: Add K230 SPI, QSPI, IDMA and XIP support

Hi,

The k230 machine currently models the C908 core, CLINT, PLIC, watchdogs
and UARTs, but cannot yet model the real boot path. This series adds the
Kendryte K230 DesignWare SSI controllers and their machine integration
as groundwork for full boot support, enabling U-Boot to boot from SPI
NOR through standard PIO, QSPI IDMA, and the XIP read window.

K230 contains three SSI instances with different capabilities. The SDK
numbers them according to the address map: spi0 is the SPI-OPI instance,
while spi1 and spi2 are QSPI0 and QSPI1. The series models the register
contract documented in the K230 Technical Reference Manual V0.3.1
(2024-11-18), standard FIFO-backed SPI transfers, interrupt routing,
Dual/Quad SDR transfers, SPI NOR attachment, synchronous internal DMA,
the HI_SYS SSI_CTRL wrapper, and the spi0 XIP read window.

The implementation is split so that the register model and machine
integration come first, followed by the PIO data path, interrupt support,
enhanced transfers, flash integration, IDMA, HI_SYS, and XIP. A single
K230 SSI qtest binary grows with the series and exercises each layer when
it becomes available.

The final patch adds trace events covering register accesses (excluding
DR, whose per-frame traffic would drown out everything else during U-Boot
and Linux SPI NOR transfers), transaction boundaries, interrupt levels,
IDMA start/completion/error, and XIP reads. The trace points are placed
in a dedicated patch at the end of the series so that the functional
patches stay focused on behaviour and the trace surface can be reviewed
in one place. The events only observe model behaviour and do not change
it.

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
* checkpatch reports no errors.

Kangjie Huang (11):
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
  hw/ssi: Add trace events for K230 DesignWare SSI

 docs/system/riscv/k230.rst     |    4 +
 hw/misc/k230_hi_sys.c          |  169 ++++
 hw/misc/meson.build            |    1 +
 hw/riscv/Kconfig               |    2 +
 hw/riscv/k230.c                |  165 ++-
 hw/ssi/Kconfig                 |    4 +
 hw/ssi/k230_dw_ssi.c           | 1754 ++++++++++++++++++++++++++++++++++++++++
 hw/ssi/meson.build             |    1 +
 hw/ssi/trace-events            |   11 +
 include/hw/misc/k230_hi_sys.h  |   55 ++
 include/hw/riscv/k230.h        |    9 +
 include/hw/ssi/k230_dw_ssi.h   |  111 +++
 tests/qtest/k230-dw-ssi-test.c |  917 +++++++++++++++++++++
 tests/qtest/meson.build        |    4 +-
 14 files changed, 3191 insertions(+), 16 deletions(-)

base-commit: f893c46c3931b3684d235d221bf8b7844ddbf1d7
--
2.43.0
~~~
