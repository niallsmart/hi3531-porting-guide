# DHB_AX Hardware Documentation

Hardware documentation for the TVT Digital **`DHB_AX V1.2`** board, built on a
HiSilicon Hi3531 SoC and sold as the **LTS LTD2704XE-P** digital video
recorder, prepared to support porting a modern Linux kernel so the unit can be
repurposed as a general-purpose server.

This is reference material for a kernel developer. It documents what the
hardware *is* and how the vendor firmware drives it. It does not attempt the
port: that work lives in
[dhb-ax-buildroot](https://github.com/niallsmart/dhb-ax-buildroot), and
[the division between the two repositories](https://github.com/niallsmart/dhb-ax-buildroot/blob/main/doc/repository-split.md)
sets out where a given change belongs.

## The short version

| | |
|---|---|
| **Board** | TVT Digital `DHB_AX V1.2`, sold as the LTS LTD2704XE-P |
| **SoC** | HiSilicon Hi3531 V100 — dual-core Cortex-A9, ARMv7, ~1 GHz |
| **RAM** | ~1 GB across two DDR controllers — the vendor firmware gives Linux only 224 MB |
| **Storage** | 2 MB SPI-NOR (U-Boot), 128 MB NAND (kernel + rootfs), 1 TB SATA disk |
| **Network** | Gigabit — Synopsys DWMAC1000 + Realtek RTL8211CL |
| **Firmware** | U-Boot 2010.06 (2012), Linux 3.0.8 (2013), HiSilicon MPP V1.0.7.3 |

CPU, memory, UART, Ethernet and SATA use conventional IP, and mainline Linux
runs on the board today — see the
[port](https://github.com/niallsmart/dhb-ax-buildroot) for what is working.
The remaining constraints are the unusual DRAM layout, the lack of mainline
flash-controller drivers, and the closed media stack. See
[Memory Map](02-memory-map.md), [Porting Roadmap](16-porting-roadmap.md), and
[Media Codec](14-media-codec.md).

## Table of contents

### Core subsystems — needed for a bootable server

| # | Document | Contents |
|---|---|---|
| 01 | [SoC Overview](01-soc-overview.md) | CPU, the full SoC address map, all 96 interrupts, timers, SMP, clocks, mainline status |
| 02 | [Memory Map and DRAM](02-memory-map.md) | Two DDR controllers, the 224 MB/MMZ split, how to reclaim the memory |
| 03 | [Boot Chain and U-Boot](03-boot-chain.md) | Boot sequence, environment, available commands, TFTP workflow |
| 04 | [Flash Storage](04-flash-storage.md) | SPI-NOR and NAND, partition layout, kernel image, backups |
| 05 | [UART and Serial Console](05-uart-console.md) | Four PL011s, console setup, physical access, rear-panel RS485 |
| 06 | [Ethernet](06-ethernet.md) | DWMAC1000, RTL8211CL PHY, MAC address, TOE bypass |
| 07 | [SATA and Disk Storage](07-sata-storage.md) | AHCI, JMB321 port multiplier, the 1 TB disk, expansion capacity |
| 08 | [USB](08-usb.md) | EHCI and OHCI, class drivers, PHY glue |
| 09 | [GPIO, Pin Multiplexing and I²C](09-gpio-pinmux-i2c.md) | 19 GPIO groups, pinmux, bit-banged I²C — **read the confidence notes** |
| 10 | [RTC, Watchdog, IR, Alarm I/O and Other Small Peripherals](10-rtc-watchdog-misc.md) | SP805 watchdog, PL031 and external RTCs, rear alarm terminals, CryptoMemory, SD/MMC |
| 20 | [Front Panel MCU and Protocol](20-front-panel-mcu.md) | AT89S52 on `ttyAMA1`, the 5-byte binary protocol, command bytes |

### DVR-specific hardware — documented at wiring level

| # | Document | Contents |
|---|---|---|
| 11 | [Video Input](11-video-input.md) | Nextchip decoder, Lattice FPGA, BT.1120 into the VIU, a possible capture route |
| 12 | [Video Output](12-video-output.md) | Integrated VGA, HDMI and dual CVBS paths, framebuffer layers |
| 13 | [Audio](13-audio.md) | NVP1104B codec, SIO/I²S, what is unconfirmed |
| 14 | [Media Codec — Why This Is a Dead End](14-media-codec.md) | The MPP stack and why it cannot be ported |

### Reference

| # | Document | Contents |
|---|---|---|
| 15 | [Product Identity](15-product-identity.md) | TVT as the ODM, chassis label, component inventory, the `productID` chain, build provenance |
| 16 | [Porting Roadmap and Open Questions](16-porting-roadmap.md) | Phased plan, open questions, risks |
| 17 | [Live Register Dumps](17-register-dumps.md) | Pinmux, CRG, SYS_CTRL and DDR controller dumps from the running board |
| 18 | [Reference Assets and Capture Methods](18-reference-assets.md) | SDK layout, SDK verification, live access, boot console capture, pitfalls |
| 19 | [Pin Multiplexing Map](19-pinmux-map.md) | All 151 IO_CONFIG registers from the chip datasheet, with what this board selects |
| 21 | [PCI Express](21-pcie.md) | Two unused root complexes and the vendor cascade feature |
| — | [Raspberry Pi Configuration Log](raspberrypi-changes.md) | Changes made to the serial-proxy host |

## Where to start

A developer picking this up should read, in order:

1. [Porting Roadmap](16-porting-roadmap.md) — the plan and what is unknown
2. [SoC Overview](01-soc-overview.md) — the register map for the device tree
3. [Memory Map](02-memory-map.md) — the memory prize and how to claim it
4. [Boot Chain](03-boot-chain.md) — how to load a test kernel without risk

Then [Ethernet](06-ethernet.md) and [SATA](07-sata-storage.md), which are the
two drivers that make the machine useful.

## Confidence and gaps

This documentation distinguishes between what was **measured** and what is
**inferred**. Claims are sourced to a live capture, a register dump, an SDK
file, or a PCB photograph wherever possible, and inferences are labelled.

The remaining gaps, in order of impact:

| Gap | Effect |
|---|---|
| **Only the top PCB surface is photographed** | SPI-NOR, NAND, RTC and regulators are unlocated on the board. |
| **The SDK's board documentation is for HiSilicon's demo board** | Board-level detail comes from the device itself — live register reads and the vendor `pinctrl_*.sh` scripts. A TVT GPL source release would corroborate it. Chip-level detail does not have this problem: the SDK ships the full 1794-page Hi3531 datasheet. |
| **`U16` is unidentified** | A 56-pin TI part beside the VGA/HDMI connectors. Not on any path a server build depends on. |
| **No full NVP1104B datasheet or register map** | Blocks the external video decoder and its integrated audio codec. |

Full detail on gaps and next steps is in
[Porting Roadmap and Open Questions](16-porting-roadmap.md).
