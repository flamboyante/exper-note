# K230 DWC SSI V3: five-patch cover letter

```text
Subject: [PATCH v3 0/5] hw/riscv: Add K230 DWC SSI Standard PIO support

Hi,

This series adds a reusable Standard PIO model for the Synopsys DWC SSI
controller. The K230 machine has three such controllers, and the later
patches add their machine integration.

The first patch adds the Standard SPI register subset, configurable FIFO
and chip-select resources, four transfer modes, reset and migration state,
and RAZ/WI handling for unsupported enhanced SPI, DMA, and XIP registers.
It uses the DWC CTRLR0 layout, where TMOD is at bits [11:10]. The second
patch adds the Standard PIO interrupt support. The remaining patches add
the K230 instances, PLIC routing, and an optional SPI NOR on spi0 CS0.

Changes since v2:

* Rename the controller model and integration from DesignWare/DW to DWC.
* Clarify the DWC CTRLR0.TMOD layout and keep DW APB SSI variants out of
  scope.
* Fix Standard PIO transfer progress for TX-only, RX FIFO overrun in all
  PIO modes, and RX-only dummy frames.
* Merge the associated regression assertions into the existing qtests,
  including a non-zero SPI NOR EEPROM-read address.

Enhanced SPI, internal IDMA, HI_SYS, and XIP remain separate follow-up
series. The older DW APB SSI TMOD encoding is not part of this series.

Testing:

* Built qemu-system-riscv64 and the K230 DWC SSI qtest target.
* Ran all 14 K230 DWC SSI qtests, including Standard PIO modes, register
  contracts, interrupts and PLIC routing, and Standard SPI NOR reads.
* Standard 1-1-1 transfers were also exercised manually through the K230
  SDK U-Boot and Linux SPI paths against the attached flash.
* git diff --check passed.
* checkpatch.pl reported no code errors; its MAINTAINERS coverage warnings
  are expected because this series does not modify MAINTAINERS.

Kangjie Huang (5):
  hw/ssi: Add Synopsys DWC SSI standard PIO controller
  hw/ssi: Add DWC SSI standard interrupt support
  hw/riscv/k230: Instantiate DWC SSI controllers
  hw/riscv/k230: Route SSI interrupts to the PLIC
  hw/riscv/k230: Attach a standard SPI NOR flash

base-commit: b428fe036233cbd15d37e3c027ab6ca4d3661a80
--
2.43.0
```
