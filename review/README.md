# Documentation Review: Independent Investigation Areas

## Purpose

This directory contains observations from a review of the Hi3531 hardware
documentation. Each topic is deliberately framed as an investigation prompt,
not a final finding. Please check it independently against the supplied SDK,
Hi3531 datasheet, flash backups, runtime captures, and current upstream Linux
interfaces before deciding whether the main documentation should change.

For each topic, please report whether the observation is confirmed, disproved,
or remains uncertain. Cite the exact SDK source, datasheet chapter or table,
runtime capture, or upstream binding used. If a change is appropriate, please
look for all affected occurrences rather than correcting only the cited example.

## Investigation index

1. [U-Boot SATA loading](01-uboot-sata-loading.md)
2. [Device-tree interrupt numbers](02-device-tree-interrupts.md)
3. [SP804 timer topology](03-sp804-timer-topology.md)
4. [Device-tree handoff from U-Boot](04-device-tree-handoff.md)
5. [Access to the upper DRAM bank](05-upper-dram-bank.md)
6. [Secondary CPU startup](06-secondary-cpu-startup.md)
7. [Pinmux provenance and completeness](07-pinmux-map.md)
8. [Scope of the Hi3531 register documentation](08-register-documentation.md)
9. [GPIO and watchdog confidence](09-gpio-watchdog.md)
10. [DDR0 MMZ arithmetic](10-ddr0-mmz-arithmetic.md)
11. [Ethernet DTS details](11-ethernet-dts.md)
12. [Internal contradictions](12-internal-contradictions.md)
13. [Register-map completeness](13-register-map-completeness.md)

## Checks that did not expose an issue

- Local Markdown links resolved successfully during the review.
- The flash backup SHA-256 values matched
  `backups/2026-08-03/MANIFEST.md`.
- Read-only SSH connectivity to `raspberrypi` was confirmed.

No DVR, Raspberry Pi, flash, NAND, or SATA state was changed during the review.

