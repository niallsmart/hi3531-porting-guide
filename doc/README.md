# Hi3531 DVR — Hardware Documentation

Hardware documentation for an **LTS LTD2704XE-P** digital video recorder, built
on a HiSilicon Hi3531 SoC, prepared to support porting a modern Linux kernel so
the unit can be repurposed as a general-purpose server.

This is reference material for a kernel developer. It documents what the
hardware *is* and how the vendor firmware drives it. It does not attempt the
port.

## The short version

| | |
|---|---|
| **SoC** | HiSilicon Hi3531 V100 — dual-core Cortex-A9, ARMv7, ~1 GHz |
| **RAM** | ~1 GB across two DDR controllers — but Linux is given only 224 MB |
| **Storage** | 2 MB SPI-NOR (U-Boot), 128 MB NAND (kernel + rootfs), 1 TB SATA disk |
| **Network** | Gigabit — Synopsys DWMAC1000 + Realtek RTL8211CL |
| **Firmware** | U-Boot 2010.06 (2012), Linux 3.0.8 (2013), HiSilicon MPP V1.0.7.3 |
| **ODM** | TVT Digital — LTS is a rebadger |

**The project looks viable.** The two subsystems that matter for a server —
Ethernet and SATA — are standard licensed IP with mature mainline drivers. The
CPU, memory and UART are equally conventional.

**Three findings shape the work:**

1. **~790 MB of RAM is reserved for the video pipeline.** Not loading
   HiSilicon's MMZ allocator takes the machine from 224 MB to roughly 1 GB.
   This is the single biggest win available.
2. **Neither flash controller has a mainline driver**, but this does not block
   anything — loading a modern kernel over TFTP with the existing vendor U-Boot
   and putting the root filesystem on the SATA disk sidesteps both, satisfies
   the no-flash-writes constraint, and leaves the original firmware intact as a
   fallback. The kernel cannot be loaded from SATA itself: this U-Boot has no
   `sata` command.
3. **The media hardware is a genuine dead end** — proprietary binary modules, no
   public register documentation, a 6.6 MB firmware blob, and a custom FPGA
   bitstream. For a server this costs nothing, since none of it is needed.

## Table of contents

### Core subsystems — needed for a bootable server

| # | Document | Contents |
|---|---|---|
| 01 | [SoC Overview](01-soc-overview.md) | CPU, complete register base map, interrupt map, clocks, mainline status |
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
| 11 | [Video Input](11-video-input.md) | Nextchip decoder, Lattice FPGA, BT.1120 into the VIU |
| 12 | [Video Output](12-video-output.md) | VGA, HDMI via SiI9024, dual CVBS, framebuffer layers |
| 13 | [Audio](13-audio.md) | TLV320AIC31, SIO/I²S, what is unconfirmed |
| 14 | [Media Codec — Why This Is a Dead End](14-media-codec.md) | The MPP stack and why it cannot be ported |

### Reference

| # | Document | Contents |
|---|---|---|
| 15 | [Product Identity and PCIe](15-product-identity.md) | TVT as the ODM, component inventory, unused PCIe |
| 16 | [Porting Roadmap and Open Questions](16-porting-roadmap.md) | Phased plan, resolved and open questions, risks |
| 17 | [Live Register Dumps](17-register-dumps.md) | Pinmux, CRG, SYS_CTRL and DDR controller dumps from the running board |
| 18 | [Reference Assets and Capture Methods](18-reference-assets.md) | SDK layout, SDK verification, live access, pitfalls |
| 19 | [Pin Multiplexing Map](19-pinmux-map.md) | All 108 IO_CONFIG registers with their function tables |
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
| **The SDK's board documentation is for HiSilicon's demo board** | Board-level detail comes from the device itself — chiefly the vendor `pinctrl_*.sh` scripts. A TVT GPL source release would corroborate it. |
| **`U16` is unidentified** | A 56-pin TI part beside the VGA/HDMI connectors. Not on any path a server build depends on. |
| **No NVP1104B datasheet** | Only affects the video capture path, which is out of scope anyway. |

[Porting Roadmap](16-porting-roadmap.md) carries a quick-reference table
pointing to the answers most often needed.

`PLAN.md` constraints observed throughout: **nothing was written to SPI or
NAND.** The device was rebooted twice, with permission, to capture U-Boot state.
Register reads were done from U-Boot's read-only `md` rather than the Linux
`himm` tool, which prompts for a write after every read.

Full detail on gaps and next steps is in
[Porting Roadmap and Open Questions](16-porting-roadmap.md).
