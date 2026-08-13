# SoC Overview — HiSilicon Hi3531 V100

## Identification

| Property | Value | Source |
|---|---|---|
| SoC | HiSilicon Hi3531 V100 | SDK, PCB silkscreen |
| Marketing name | H.264 Codec Processor | SDK datasheet |
| CPU | Dual-core ARM Cortex-A9 MPCore | `/proc/cpuinfo` |
| CPU ID | `0x413fc090` — implementer 0x41 (ARM), part 0xc09, variant 0x3, rev 0 | dmesg |
| Architecture | ARMv7-A (`armv7l`) | `uname -a` |
| FPU | VFPv3-D16 (`vfp vfpv3 vfpv3d16`) | `/proc/cpuinfo` |
| Other features | `swp half thumb fastmult edsp` | `/proc/cpuinfo` |
| BogoMIPS | 1849.75 / 1856.30 (per core), `lpj=9248768` | dmesg |
| Cache | VIPT non-aliasing D-cache, VIPT aliasing I-cache | dmesg |
| L2 cache | HiSilicon proprietary "L2 Cache V200" at `0x20700000` — **not** an ARM PL310 | SDK, live register reads |
| Machine ID | `godnet` / `MACH_TYPE_GODNET` | dmesg, U-Boot `board.c` |
| PCI vendor:device | `19e5:3531` (PCIe root complexes) | dmesg |

`godnet` is HiSilicon's internal board name for the Hi3531 reference design. It is
used consistently across the SDK's U-Boot (`board/godnet/`), the kernel
(`arch/arm/mach-godnet/`), and the defconfig (`godnet_defconfig`). It is *not*
specific to this DVR.

## Mainline status

**Hi3531 has no mainline Linux support.** There is no `mach-godnet`, no
`hi3531` device tree, and no SoC entry in `arch/arm/mach-hisi/`. Mainline
carries other HiSilicon parts (`hix5hd2`, `hi3620`, `hip04`) but none share
this SoC's register layout.

A port is therefore a from-scratch device tree plus driver bring-up. The
mitigating factor is that most of the SoC's non-media blocks are licensed
third-party IP with existing mainline drivers:

| Block | IP | Mainline driver |
|---|---|---|
| UART | ARM PL011 | `amba-pl011` |
| Interrupt controller | ARM GIC (Cortex-A9) | `irq-gic` |
| System timers | Two ARM SP804 (part `0x804`, designer ARM) at `0x20000000` and `0x20010000`; only the first is used | `arm,sp804` |
| SMP | Cortex-A9 MPCore SCU at `0x20300000` | `arm,cortex-a9-scu` |
| Ethernet | Synopsys DesignWare MAC 1000 (ID 0x36) | `stmmac` |
| SATA | AHCI 1.2 | `ahci_platform` |
| USB | EHCI + OHCI | `ehci-platform`, `ohci-platform` |
| SD/MMC | Synopsys DesignWare (`hi_mci`) | `dw_mmc` |
| Watchdog | ARM SP805 (part `0x805`, designer ARM) at `0x20040000` | `arm,sp805` |
| On-chip RTC | ARM PL031 (part `0x031`, designer ARM) at `0x20060000` | `arm,pl031` |

Every row above was checked against the SDK sources *and* the running device,
rather than inferred from the IP pairing. The evidence:

| Block | Hardware evidence | Vendor software evidence |
|---|---|---|
| UART | AMBA peripheral ID at `0x20080FE0` = `0x11`, `0x10`, `0x24`, `0x00` → part `0x011`, designer `0x41` (ARM), rev 2 | `CONFIG_SERIAL_AMBA_PL011=y`; dmesg `ttyAMA0 at MMIO 0x20080000 (irq = 40) is a PL011 rev2` |
| GIC | Distributor at `0x20301000`, CPU interface at `0x20300100` — the standard Cortex-A9 PERIPHBASE layout, see [the private peripheral region](#the-cortex-a9-private-peripheral-region) | `gic_init()` in `mach-godnet/core.c`; `/proc/interrupts` shows `GIC` for every line |
| Timers | Peripheral ID at `0x20000FE0` **and** `0x20010FE0` → part `0x804` (SP804), designer ARM, rev 1 | `CONFIG_LOCAL_TIMERS` **not** set; all SP804 register use in `core.c` is through `CFG_TIMER_VABASE`, the first block |
| Watchdog | Peripheral ID at `0x20040FE0` = `0x05`, `0x18`, `0x14`, `0x00` → part `0x805` (SP805), designer `0x41` (ARM), rev 1 | Vendor `wdt.ko`; U-Boot closes it before boot |
| On-chip RTC | Peripheral ID at `0x20060FE0` = `0x31`, `0x10`, `0x04`, `0x00` → part `0x031` (PL031), designer `0x41` (ARM), rev 0 | None — the vendor system does not use it |
| IR receiver | Peripheral ID at `0x20070FE0` reads all zeros — **not a PrimeCell**, no ARM identity | Vendor `hi_ir.ko`, banner `HISI_IRDA-MF` |
| Ethernet | DWMAC version register `0x101C0020` = `0x00001036` → Synopsys ID **0x36** (DWMAC 3.6) | `CONFIG_STMMAC_ETH=m` with the upstream `drivers/net/stmmac/` tree incl. `dwmac1000_core.c` |
| SATA | AHCI version register `0x10080010` = `0x00010200` → **AHCI 1.2** | `CONFIG_SATA_AHCI_PLATFORM=y`; dmesg `AHCI 0001.0200 32 slots 2 ports`, `scsi0 : ahci_platform` |
| USB | dmesg `EHCI 1.00`, standard OHCI | `CONFIG_USB_EHCI_HCD`/`OHCI_HCD` cores wrapped by `hiusb-ehci.c` / `hiusb-ohci.c` |
| SD/MMC | Controller present at `0x10020000`, IRQ 67, zero interrupts taken | `hi_mci_reg.h` register map is byte-for-byte dw_mmc (`CTRL` 0x00 … `BUFADDR` 0x98, including the IDMAC block) |

Two caveats on otherwise-sound rows. The Ethernet and USB blocks need HiSilicon
glue that mainline does not have: `stmmac` here is a vendor fork configured by
Kconfig constants rather than device tree, and the USB cores need PHY and CRG
setup from `hiusb-godnet.c`. In both cases the IP core itself is genuine and the
mainline driver applies; the missing piece is init glue. The SD/MMC controller
is a real dw_mmc instance but has never enumerated a card on this board.

The blocks with *no* mainline path are the HiSilicon-proprietary media
pipeline (VIU, VPSS, VOU, VEDU/H.264, VDEC, TDE, IVE, JPEG), the
HiSilicon NAND and SPI flash controllers, and the L2 cache controller. See
[14-media-codec.md](14-media-codec.md) and [04-flash-storage.md](04-flash-storage.md).

## L2 cache controller

The L2 controller at `0x20700000` is a HiSilicon in-house design, **not** an ARM
PL310, despite the Cortex-A9 MPCore pairing that usually implies one. Nothing in
the SDK or on the running device supports a PL310 identification.

Evidence:

- The vendor defconfig selects `CONFIG_CACHE_HIL2V200=y` ("Enable the Hisilicon
  L2 Cache V200"), driven by `arch/arm/mm/cache-hil2v200.c` and
  `cache-hil2v200.h`. `CONFIG_CACHE_L2X0` is not set, and its `depends on` list
  in `arch/arm/mm/Kconfig` does not include `ARCH_GODNET` — the l2x0 driver
  cannot even be selected for this SoC.
- The register layout is incompatible with PL310 (see table below). PL310 places
  a read-only Cache ID register at offset `0x000` and the control register at
  `0x100`; this block puts a writable control register at `0x000`.
- The kernel banner is `L2cache cache controller enabled`, printed by
  `cache-hil2v200.c`. The l2x0 driver's `l2x0: N ways, CACHE_ID 0x…` line never
  appears.
- Live reads on the running DVR match the HiL2V200 driver's init sequence
  exactly:

  | Address | Value | HiL2V200 meaning |
  |---|---|---|
  | `0x20700000` | `0x00000001` | `L2_CTRL` — cache enabled |
  | `0x20700004` | `0x01803000` | `L2_AUCTRL` — bits 12/13 = `EVENT_BUS_EN`, `MONITOR_EN`, as written by the driver |
  | `0x20700008` | `0x00000002` | `L2_STATUS` |
  | `0x20700100` | `0x00003FFF` | `L2_INTMASK` — the literal `0x3fff` the driver writes |
  | `0x20700108` | `0x00000000` | `L2_RINT` — no errors latched |

  Under a PL310 mapping, `0x000` would read `0x410000C…` and `0x100` would be a
  control register with only bit 0 defined; `0x3FFF` there is meaningless.

### Register map

Offsets from `0x20700000`, per `arch/arm/mm/cache-hil2v200.h`:

| Offset | Register |
|---|---|
| `0x000` | `L2_CTRL` — bit 0 enables the cache |
| `0x004` | `L2_AUCTRL` — auxiliary control |
| `0x008` | `L2_STATUS` — bit 0 idle, bit 1 SPNIDEN |
| `0x100` – `0x12C` | Interrupt mask / masked / raw / clear, for the core, internal monitor, and external monitor |
| `0x200` | `L2_SYNC` |
| `0x20C` | `L2_MAINT_AUTO` |
| `0x210` | `L2_INVALID` |
| `0x214` | `L2_CLEAN` |
| `0x300`, `0x304` | D-side / I-side way lock |
| `0x400` – `0x410` | Test mode and interrupt/event test |
| `0x500` – `0x510` | Region 0–4 configuration |
| `0x600` – `0x628` | Internal monitor counters 0–10 |
| `0x700` – `0x76C` | External monitor counters 0–27 |
| `0x800` – `0x808` | Special control and check registers 0–1 |

The register file ends at `0x80C`, so a 4 KB window covers it.

Cleans and invalidates are range-based: write start and end addresses, then poll
`L2_RINT` for completion. There is no PL310-style per-line
`clean_line`/`inv_line` register.

### Interrupts

Three GIC lines, unlike PL310's single combined output:

| IRQ | Name | SDK symbol |
|---|---|---|
| 69 | `L2 chk0` | `INTNR_L2CACHE_CHK0_INT` (`GODNET_IRQ_START + 37`) |
| 70 | `L2 chk1` | `INTNR_L2CACHE_CHK1_INT` (`GODNET_IRQ_START + 38`) |
| 71 | `L2 com` | `INTNR_L2CACHE_INT_COMB` (`GODNET_IRQ_START + 39`) |

All three are confirmed present in `/proc/interrupts` on the running device.

### Porting implications

U-Boot leaves the L2 controller alone; the kernel brings it up. A mainline port
therefore boots fine with the outer cache disabled, at a performance cost.

To use the L2, the vendor driver has to be forward-ported: it is a self-contained
`outer_cache` provider (`inv_range`, `clean_range`, `flush_range`, `sync`,
`flush_all`, `inv_all`, `disable`) and the `outer_cache` hooks it fills still
exist in current kernels. The work is adapting it to device tree probing and
current locking conventions rather than reverse-engineering the hardware.
Do **not** point a `arm,pl310-cache` compatible at this address — the register
writes would land on unrelated registers.

## Register base map

**This is the complete SoC address map**, from section 1 of the Hi3531
datasheet, cross-checked against the SDK U-Boot header
`arch/arm/include/asm/arch-godnet/platform.h` and `/proc/iomem` on the running
device. Reserved ranges are omitted; everything the chip decodes is here.

### Reading the size column

The datasheet gives every peripheral a **64 KB decode window** regardless of how
many registers it actually has, so the windows say nothing about the register
count. The column below gives the **register extent** — how much a device-tree
`reg` property should actually cover — with the window noted only where it
differs usefully. Where the extent is not established the window is given and
marked.

| Base | Extent | Block | Notes |
|---|---|---|---|
| `0x00000000` | 64 MB | Address-remap region | Where U-Boot copies the CPU1 holding code. Not DRAM — see [Secondary CPU startup](#secondary-cpu-startup) |
| `0x04000000` | 64 KB | Boot ROM | |
| `0x04010000` | 64 KB | On-chip SRAM | 64 KB. Not used by the vendor firmware |
| `0x10000000` | 64 KB (window) | NAND flash controller (NANDC) | No mainline driver |
| `0x10010000` | 64 KB (window) | SPI flash controller (SFC) | No mainline driver |
| `0x10020000` | 0x1000 | SD/MMC/SDIO | `dw_mmc` may fit; pins clash with VIU3 |
| `0x10030000` | 64 KB (window) | SIO0 — audio serial port | |
| `0x10040000` | 64 KB (window) | SIO1 | |
| `0x10050000` | 64 KB (window) | SIO2 | |
| `0x10060000` | 64 KB (window) | SIO3 | |
| `0x10080000` | 0x10000 | SATA / AHCI | `ahci_platform` |
| `0x100A0000` | 0x10000 | USB 2.0 OHCI | |
| `0x100B0000` | 0x10000 | USB 2.0 EHCI | |
| `0x100C0000` | 64 KB (window) | CIPHER — crypto engine | HiSilicon block, no AMBA ID. Datasheet chapter 3.6 |
| `0x100D0000` | 0x1000 | DMA controller — **ARM PL080** | IRQ 61/62. Read as a Samsung PL080S by mainline — see below |
| `0x100E0000` | 64 KB (window) | **SIO4** | The audio port this board uses — see [13-audio.md](13-audio.md) |
| `0x100F0000` | 64 KB (window) | SIO5 | |
| `0x10150000` | 64 KB (window) | SCD0 — start-code detect | Media path |
| `0x10160000` | 64 KB (window) | SCD1 | Media path |
| `0x10170000` | 64 KB (window) | JPGD — JPEG decoder | Media path, undocumented |
| `0x101C0000` | 0x20000 | Ethernet — two DWMAC1000 instances | GMAC0 at `+0x0000`, **GMAC1 at `+0x4000` is the live port** |
| `0x20000000` | 0x1000 | SP804 dual timer 0 | The one the vendor kernel uses. IRQ 35 |
| `0x20010000` | 0x1000 | SP804 dual timer 1 | Unused. IRQ 36 |
| `0x20030000` | 0x100 | CRG — clock and reset generator | |
| `0x20040000` | 0x1000 | Watchdog — ARM SP805 | IRQ 34 |
| `0x20050000` | 0x1000 | System controller (SYS_CTRL) | |
| `0x20060000` | 0x1000 | On-chip RTC — ARM PL031 | IRQ 39 |
| `0x20070000` | 64 KB (window) | IR receiver | HiSilicon block, not a PrimeCell. IRQ 48 |
| `0x20080000` | 0x1000 | UART0 — console | IRQ 40 |
| `0x20090000` | 0x1000 | UART1 | IRQ 41 |
| `0x200A0000` | 0x1000 | UART2 — rear RS485 | IRQ 42 |
| `0x200B0000` | 0x1000 | UART3 | IRQ 43 |
| `0x200C0000` | 0x1000 | **Hardware SPI — ARM PL022** | IRQ 44. Mainline `spi-pl022`, binds cleanly |
| `0x200D0000` | 64 KB (window) | **Hardware I²C** | IRQ 45. Unused — this board bit-bangs I²C on GPIO |
| `0x200F0000` | **0x25C** | IO_CONFIG — pin multiplexing | 151 registers, `0x000`–`0x258`. See below |
| `0x20110000` | 64 KB (window) | DDR controller 0 | |
| `0x20120000` | 64 KB (window) | DDR controller 1 | |
| `0x20130000` | 0x1000 | **SP804 dual timer 2** | IRQ 37. Genuine SP804, absent from the SDK headers |
| `0x20140000` | 0x1000 | **SP804 dual timer 3** | IRQ 38. Genuine SP804, absent from the SDK headers |
| `0x20150000` | 0x1000 each | GPIO0 … GPIO18 at `0x20150000 + n*0x10000` | IRQs 105–117, paired above GPIO6 |
| `0x20300000` | 0x2000 | Cortex-A9 PERIPHBASE | See the offsets below |
| `0x20400000` | 1152 KB | ARM debug (CoreSight) | |
| `0x20580000` | 256 KB | VICAP — video capture | Documented; blocked by the external decoder |
| `0x205C0000` | 64 KB (window) | VDP — video display | Documented |
| `0x205D0000` | 64 KB (window) | HDMI transmitter | **No datasheet chapter** |
| `0x205E0000` | 64 KB (window) | IVE — intelligent video engine | Documented |
| `0x20600000` | 64 KB (window) | VPSS0 — scaler | Undocumented |
| `0x20610000` | 64 KB (window) | TDE — 2D engine | Undocumented |
| `0x20620000` | 64 KB (window) | VENC0 — H.264 encoder | Undocumented |
| `0x20630000` | 64 KB (window) | VDH0 — H.264 decoder | Undocumented |
| `0x20640000` | 64 KB (window) | VOIE — voice encoder | Documented. IRQ 102 |
| `0x20660000` | 64 KB (window) | JPGE — JPEG encoder | Undocumented |
| `0x20680000` | 64 KB (window) | VPSS1 | Undocumented |
| `0x20690000` | 64 KB (window) | VENC1 | Undocumented |
| `0x206A0000` | 64 KB (window) | VDH1 | Undocumented |
| `0x206B0000` | 64 KB (window) | VCMP | Undocumented. IRQ 101 |
| `0x206C0000` | 64 KB (window) | MD — motion detection | Documented |
| `0x206D0000` | 64 KB (window) | BPD — bit-plane decoder | Undocumented |
| `0x20700000` | 1 MB | L2 cache controller | HiSilicon L2 Cache V200, no mainline driver |
| `0x20800000` | 64 KB (window) | PCIe0 controller registers | |
| `0x20810000` | 64 KB (window) | PCIe1 controller registers | |
| `0x30000000` | 256 MB | PCIe0 memory window | |
| `0x40000000` | 256 MB | PCIe0 configuration space | |
| `0x50000000` | 64 MB | NAND memory-mapped window | |
| `0x58000000` | 64 MB | SPI flash memory-mapped window | |
| `0x60000000` | 256 MB | PCIe1 memory window | |
| `0x70000000` | 256 MB | PCIe1 configuration space | |
| `0x80000000` | 1 GB | DDR0 | 512 MB fitted — see [02-memory-map.md](02-memory-map.md) |
| `0xC0000000` | 1 GB | DDR1 | 512 MB fitted |

### Three blocks identified from their PrimeCell IDs

The datasheet names these but does not say whose IP they are. Reading
`base+0xFE0`–`0xFF0` on the live device settles it, and two of the three have
mainline drivers:

| Base | PID0–3 | periphid | Part |
|---|---|---|---|
| `0x200C0000` | `22 10 04 00` | `0x00041022` | **ARM PL022** SPI |
| `0x100D0000` | `80 10 14 0A` | `0x0A141080` | **ARM PL080** DMA, customer-modified |
| `0x20130000`, `0x20140000` | `04 18 14 00` | `0x00141804` | **ARM SP804**, same as timers 0 and 1 |

`0x200D0000` (hardware I²C), `0x100C0000` (CIPHER) and `0x100E0000` (SIO4) all
read zero — HiSilicon blocks with no AMBA identity, like the GPIO controllers.

**The SPI controller is free.** `0x00041022` matches `spi-pl022`'s plain ARM
entry exactly, so `compatible = "arm,pl022", "arm,primecell"` on SPI 12 binds
with no override. This board leaves the pins in GPIO mode and bit-bangs
instead, so nothing currently uses it.

> **The DMA controller is a trap.** Its periphid is `0x0A141080`, and
> `amba-pl08x` lists that value with mask `0xffffffff` as **`vendor_pl080s` —
> the Samsung PL080S**, ahead of the generic `0x00041080` entry. So an
> `arm,pl080` node binds, but the driver applies Samsung's variant behaviour,
> which differs in channel register layout. The `0x0A` in PID3 is the
> customer-modified field, and HiSilicon's modifications having the same value
> as Samsung's does not mean they are the same block. Verify before using DMA,
> and expect to need either `arm,primecell-periphid = <0x00041080>` to force
> the generic entry or a new table entry upstream. Nothing in a server build
> needs DMA, so this can be left alone.

There is **no eFuse register block**. The datasheet has eFUSE *pins* (section
2.1.3.21) but nothing in the address map, so there is no OTP region to read.

> **`0x200F0000` needs at least `0x25C`.** A `0x200` size stops at register 128
> and excludes the NAND, SPI-NOR, USB, HDMI and SATA-LED pins. A `syscon` node
> should use `reg = <0x200f0000 0x1000>`.

### Vendor kernel IO mapping

The vendor kernel statically maps two IO windows
(`arch/arm/mach-godnet/include/mach/io.h`):

```
IO_ADDRESS(x) = x + 0xDE000000   for x >= 0x20000000   (0x20000000 -> 0xFE000000)
IO_ADDRESS(x) = x + 0xEE900000   for x <  0x20000000   (0x10000000 -> 0xFE900000)
```

Window 1: phys `0x20000000`, size `0x820000`, virt `0xFE000000`.
Window 2: phys `0x10000000`, size `0x1D0000`, virt `0xFE900000`.

This matters when reverse-engineering the vendor `.ko` files: register
addresses appear in the binaries as *virtual* addresses, so a GPIO group 5 base
of `0x201A0000` appears as the constant `0xFE1A0000`.

A modern port should use device tree and `ioremap` rather than reproducing
this static mapping.

## Interrupt map

**The complete SPI assignment**, from the datasheet's interrupt chapter (3.3).
The "Vendor name" column maps the lines visible in `/proc/interrupts` back to
the SoC sources.

DT SPI = IRQ − 32 throughout.

| IRQ | SPI | Source | Vendor name in `/proc/interrupts` |
|---|---|---|---|
| 32, 33 | 0, 1 | COMMTX[0], COMMRX[0] | |
| 34 | 2 | Watchdog (SP805) | |
| 35 | 3 | **Timer0** (SP804) | System Timer Tick / Free Timer |
| 36 | 4 | Timer1 | |
| 37 | 5 | Timer2 | |
| 38 | 6 | Timer3 | |
| 39 | 7 | RTC (PL031) | |
| 40–43 | 8–11 | **UART0–UART3** | `uart-pl011` |
| 44 | 12 | SSP — hardware SPI (PL022) | |
| 45 | 13 | Hardware I²C | |
| 46 | 14 | NAND controller | |
| 47 | 15 | SPI flash controller | |
| 48 | 16 | IR receiver | `Hi_IR` |
| 49–54 | 17–22 | SIO0–SIO5 — audio | |
| 58 | 26 | Software interrupt | |
| 59 | 27 | Cortex-A9 PMU 1 | |
| 60 | 28 | CIPHER | |
| 61, 62 | 29, 30 | **DMAC0, DMAC1** (PL080) | HiSilicon DMAC |
| 63 | 31 | **USB EHCI** | `ehci_hcd:usb1` |
| 64 | 32 | **USB OHCI** | `ohci_hcd:usb2` |
| 67 | 35 | **SD/MMC/SDIO** | `hi_mci` |
| 68 | 36 | **SATA** | |
| 69–71 | 37–39 | L2 cache — chk0, chk1, combined | |
| 72 | 40 | Cortex-A9 PMU 0 | |
| 73–78 | 41–46 | PCIe0 — INTA–INTD, DMA0, DMA1 | |
| 79, 80 | 47, 48 | VPSS0, VPSS1 | |
| 81–86 | 49–54 | PCIe1 — INTA–INTD, DMA0, DMA1 | |
| 87 | 55 | COMMRX[1] | |
| 88 | 56 | IVE | |
| 89 | 57 | HDMI | |
| 90 | 58 | VICAP — video capture | VIU |
| 91 | 59 | VDP — video display | VOU |
| 92, 93 | 60, 61 | VEDU0, VEDU1 — H.264 encoders | |
| 94 | 62 | JPGE — JPEG encoder | JPEGU_0 |
| 95 | 63 | JPGD — JPEG decoder | x5_jpeg |
| 96, 97 | 64, 65 | VDH0, VDH1 — H.264 decoders | VDEC, VDEC_1 |
| 98 | 66 | TDE — 2D engine | |
| 99 | 67 | BPD — bit-plane decoder | |
| 100 | 68 | MD — motion detection | VDA |
| 101 | 69 | VCMP | |
| 102 | 70 | VOIE — voice encoder | |
| 103, 104 | 71, 72 | SCD0, SCD1 | SCD |
| 105–111 | 73–79 | **GPIO0 … GPIO6** | |
| 112–117 | 80–85 | **GPIO7/8, 9/10, 11/12, 13/14, 15/16, 17/18** — paired | |
| 118 | 86 | TOE low-power interrupt | |
| 119 | 87 | **Ethernet** | `stmmaceth` |
| 127 | 95 | COMMTX[1] | |

IRQs 55–57, 65, 66 and 120–126 are reserved.

**IRQ 100 is motion detection**, which the vendor driver calls VDA. IRQs 103
and 104 are separate start-code detectors; the vendor registers only one as
"SCD".

Inter-processor interrupts IPI0–IPI5 are the standard ARM SMP set.

The SoC declares `NR_IRQS:128`, matching `GICD_TYPER`. Only CPU0 services
peripheral interrupts in the vendor configuration — every peripheral IRQ shows
a zero count on CPU1.

### The Cortex-A9 private peripheral region

PERIPHBASE is `0x20300000`, with the standard MPCore layout. From
`arch/arm/mach-godnet/include/mach/platform.h` in the SDK kernel:

| Address | Block | Mainline compatible |
|---|---|---|
| `0x20300000` | SCU | `arm,cortex-a9-scu` |
| `0x20300100` | GIC CPU interface | part of the `arm,cortex-a9-gic` node |
| `0x20300200` | Global timer | `arm,cortex-a9-global-timer` |
| `0x20300600` | Private timer and watchdog (TWD) | `arm,cortex-a9-twd-timer`, PPI 29 |
| `0x20301000` | GIC distributor | `arm,cortex-a9-gic` |

These are definite, not inferred from the PERIPHBASE convention: `core.c`
passes `CFG_GIC_DIST_BASE` and `CFG_GIC_CPU_BASE` to `gic_init()`, and
`platsmp.c` reads the SCU at `REG_BASE_A9_PERI + REG_A9_PERI_SCU`.

### Converting to device-tree SPI numbers

From `arch/arm/mach-godnet/include/mach/irqs.h` in the SDK kernel:

```c
#define GODNET_IRQ_START   (32)
#define TIMER01_IRQ        (GODNET_IRQ_START + 3)
#define UART0_IRQ          (GODNET_IRQ_START + 8)
#define UART1_IRQ          (GODNET_IRQ_START + 9)
#define INTNR_L2CACHE_CHK0_INT  (GODNET_IRQ_START + 37)
#define IRQ_LOCALTIMER     29
#define NR_IRQS            (GODNET_IRQ_START + 96)
```

So the rule is simply:

> **device-tree SPI number = the `/proc/interrupts` number − 32**

Cross-checked three ways: UART0 = 32+8 = 40 ✓, the timer = 32+3 = 35 ✓, and
L2 chk0 = 32+37 = 69 ✓ — all matching the table above.

The Cortex-A9 private timer (TWD) sits at PPI 29 (`IRQ_LOCALTIMER`), the
standard `arm,cortex-a9-twd-timer` binding. **The vendor kernel does not use
it** — see [Timers](#timers) below.

Applying the rule to the peripherals that matter for a server port:

| Peripheral | Linux IRQ | DT SPI |
|---|---|---|
| Timer | 35 | 3 |
| UART0 (console) | 40 | 8 |
| UART1 | 41 | 9 |
| UART2 | 42 | 10 |
| UART3 | 43 | 11 |
| IR | 48 | 16 |
| DMAC | 61 | 29 |
| EHCI | 63 | 31 |
| OHCI | 64 | 32 |
| SD/MMC | 67 | 35 |
| SATA / AHCI | 68 | 36 |
| L2 chk0 / chk1 / comb | 69 / 70 / 71 | 37 / 38 / 39 |
| Ethernet | 119 | 87 |

In device-tree syntax these become `interrupts = <0 SPI 4>` — GIC type 0 (SPI),
level-high (4).

> **Use the DT SPI column, not the Linux IRQ column.** The `arm,gic` binding's
> second cell is the SPI index, so it is the `/proc/interrupts` number minus 32.
> Getting it wrong is silent: the tree compiles, `request_irq` succeeds on the
> wrong line, and the peripheral simply never interrupts.

**Every SPI on this SoC is level-sensitive**, so `4` (`IRQ_TYPE_LEVEL_HIGH`) is
right for all of them. Read from the live GIC rather than assumed: the
distributor's configuration registers `GICD_ICFGR2`–`GICD_ICFGR7` at
`0x20301C08`–`0x20301C1C`, covering IDs 32–127, all read `0x55555555`, which is
`0b01` in every field — bit 1 clear, level. Only the SGIs and three of the A9's
own PPIs are edge-triggered:

| Register | Covers | Value | Meaning |
|---|---|---|---|
| `GICD_ICFGR0` | IDs 0–15 (SGIs) | `0xAAAAAAAA` | All edge — architecturally fixed |
| `GICD_ICFGR1` | IDs 16–31 (PPIs) | `0x7DC00000` | Edge on 27 (global timer), 29 (TWD), 30 (private watchdog); level elsewhere |
| `GICD_ICFGR2`–`7` | IDs 32–127 (SPIs) | `0x55555555` | All level |

The same read confirms the distributor's identity and geometry:
`GICD_TYPER` at `0x20301004` = `0x0000FC23` — 128 interrupt lines, 2 CPU
interfaces, security extensions present — and `GICD_IIDR` at `0x20301008` =
`0x0102043B`, implementer `0x43B` (ARM), revision 2. `GICC_CTLR` at
`0x20300100` reads `1` and `GICC_PMR` at `0x20300104` reads `0xF0`, the CPU
interface enabled with Linux's usual priority mask.

## Timers

Timekeeping is done by an **ARM SP804** dual timer, not by the Cortex-A9 private
timer. `CONFIG_LOCAL_TIMERS` is *not* set in `godnet_defconfig`, so the TWD is
left idle even though the hardware has it.

**There are four SP804 blocks, and the vendor kernel uses only the first.** Both
of its internal timers are in play, at their two register windows inside the one
1 KB block:

| Property | Value |
|---|---|
| Block base | `0x20000000` |
| Timer 1 → clockevent | offsets `0x00`–`0x18` within that block |
| Timer 2 → clocksource and `sched_clock` | offsets `0x20`–`0x38` within that block |
| IRQ | 35 (SPI 3), `TIMER01_IRQ`, one line for both timers |
| Clock | 155 MHz (310 MHz bus ÷ 2) |

`CFG_TIMER_VABASE` is `IO_ADDRESS(TIMER0_BASE)` and every timer access in
`mach-godnet/core.c` goes through it: the clockevent at `REG_TIMER_*`, the
clocksource and `sched_clock` at `REG_TIMER1_*`, which `platform.h` defines as
`0x020`–`0x038`. `CFG_TIMER2_VABASE` — the second block — is defined and never
referenced.

Identification is from the AMBA peripheral ID registers, read live. Both blocks
return `0x04`, `0x18`, `0x14`, `0x00` → part number `0x804`, designer `0x41`
(ARM), revision 1, with the PrimeCell signature `0D F0 05 B1` following. That is
an SP804, which mainline drives with `arm,sp804`
(`drivers/clocksource/timer-sp804.c`). `0x20020000` bus-errors, so there is no
third block.

The control registers confirm which one is running. On the live device
`0x20000008` and `0x20000028` both read `0xE2` — enable (bit 7), periodic
(bit 6), interrupt enable (bit 5), 32-bit (bit 1) — while `0x20010008` and
`0x20010028` read `0x20`, interrupt enable alone with the timer disabled, and
both of that block's value registers sit at the `0xFFFFFFFF` reset state. Timer
1's load value is `0x0017A6B0` (1,550,000), which at HZ=100 gives the 155 MHz
figure.

Note that the ÷2 is external to the block: the SP804's own prescale field
(control bits 3:2) is zero, so 155 MHz is the frequency arriving at the timer
and the rate a `clocks` property has to advertise.

### The other three SP804 blocks

| Base | IRQ | DT SPI | In the SDK headers? |
|---|---|---|---|
| `0x20010000` | 36 | 4 | Yes, as `TIMER23_IRQ` |
| `0x20130000` | 37 | 5 | **No** |
| `0x20140000` | 38 | 6 | **No** |

Timers 2 and 3 are named only in the datasheet's address map and interrupt
table; `platform.h` and `irqs.h` stop at timer 1. All three read the same
PrimeCell ID as timer 0 — `04 18 14 00`, part `0x804`, designer ARM — so they
take the same `arm,sp804` node with different `reg` and `interrupts` values.

The vendor kernel requests none of them; IRQs 36, 37 and 38 do not appear in
`/proc/interrupts`. That leaves **six spare 32-bit timers** for a port that
wants them, and none of them needs to be described at all.

### Device tree

One node covers the working block. The mainline driver takes a single interrupt
per node and, with `arm,sp804-has-irq` absent, makes timer 1 the clockevent and
timer 2 the clocksource plus `sched_clock` — the same split the vendor kernel
uses:

```dts
timer0: timer@20000000 {
    compatible = "arm,sp804", "arm,primecell";
    reg = <0x20000000 0x1000>;
    interrupts = <0 3 4>;           /* SPI 3, level-high — vendor IRQ 35 */
    clocks = <&timclk>, <&timclk>, <&apb_clk>;
    clock-names = "timer1", "timer2", "apb_pclk";
};
```

`timclk` is the 155 MHz timer clock. The driver resolves clocks by index, not by
name — `of_clk_get(np, 0)` for timer 1, and `of_clk_get(np, 1)` for timer 2 only
when the node declares three — so a single `clocks` entry feeding both timers is
also valid. The names are carried because the binding asks for them.

The other three blocks are the same node with `reg` and `interrupts` changed:
`0x20010000` on SPI 4, `0x20130000` on SPI 5, `0x20140000` on SPI 6.

Because there is no per-CPU local timer, CPU1 receives its ticks by IPI: on the
running device `IPI0 Timer broadcast interrupts` has ~2.58 M counts on CPU1 and
zero on CPU0. A mainline port can either keep this arrangement with `arm,sp804`,
or enable the TWD at PPI 29 — the TWD path is untested on this hardware, and
the A9 TWD clock scales with the CPU clock, which is why vendors often avoid it.

## Secondary CPU startup

Both cores run under the vendor firmware. `/proc/cpuinfo` lists `processor 0`
and `processor 1`, and the SCU configuration register at `0x20300004` reads
`0x00000531`: bits [1:0] = `01`, two CPUs; bits [7:4] = `0011`, both taking part
in coherency. `SCU_CTLR` at `0x20300000` reads `1`.

**U-Boot has already done the hard part by the time the kernel runs.** It takes
CPU1 out of reset and leaves it spinning on a single system-controller word,
ready to branch to whatever address is written there. A kernel starts the
second core with one 32-bit store. It never touches the reset register.

### How the vendor firmware does it

| Step | Who | What |
|---|---|---|
| 1 | CPU0, U-Boot | Asserts CPU1 reset — bits 14–17 of `CRG + 0x28` (`0x20030028`) |
| 2 | CPU0, U-Boot | Copies its own early code from `0x80800000` to **physical address 0**, so CPU1 has something to run |
| 3 | CPU0, U-Boot | Clears `SYS_CTRL + 0x134` (`0x20050134`) |
| 4 | CPU0, U-Boot | Releases CPU1 reset, then boots the kernel normally |
| 5 | CPU1 | Starts at address 0, reads `MPIDR`, takes the non-zero-core branch, polls `0x20050134` |
| 6 | CPU0, kernel | `platform_smp_prepare_cpus()` enables the SCU and writes `virt_to_phys(godnet_secondary_startup)` to `0x20050134` |
| 7 | CPU1 | Sees the non-zero value, branches to it, lands in the kernel's holding pen |
| 8 | CPU0, kernel | `boot_secondary()` sets `pen_release = 1`; CPU1 leaves the pen for `secondary_startup` |

Steps 1–5 are `arch/arm/cpu/godnet/start.S` in the SDK U-Boot:

```asm
        /* check cpuid */
        mrc     p15, 0, r0, c0, c0, 5
        and     r0, r0, #0xf
        cmp     r0, #0
        bne     core_x_flow
        b       main_core

core_x_flow:
        ldr     r3, =SMP_COREX_START_ADDR_REG
        /* checking if cpu0 run to kernel, if that, we go */
        ldr     r0, [r3]
        cmp     r0, #0
        beq     core_x_flow
        /* we got the address, let's go */
core_x_jump:
        mov     pc, r0
```

with `SMP_COREX_START_ADDR_REG` = `SYS_CTRL_REG_BASE + 0x0134` and
`COREX_RST_REG` = `CRG_REG_BASE + 0x28` in
`arch/arm/include/asm/arch-godnet/platform.h`. The kernel side is
`arch/arm/mach-godnet/{platsmp.c,platsmp.h,headsmp.S}`, whose copy of the
constant is commented `/* see bootloader */`.

The installed U-Boot matches the SDK source here: the same instruction sequence
appears at file offset `0x114c`–`0x1170` of
`backups/2026-08-03/spi-nor/dhb-ax-spi-nor-cold-a.bin`, and `0x20050134` occurs
nowhere else in the image.

Three live reads on the running device confirm the whole chain:

| Read | Value | Meaning |
|---|---|---|
| `0x20050134` | `0x8000ECCC` | A physical address inside the loaded kernel — what CPU0 wrote at step 6 |
| `0x20030028` | `0x00000023` | Bits 14–17 clear: CPU1 reset released |
| `0x0` … `0x1170` | U-Boot's vectors, then `E59F36B4 E5930000 E3500000 0AFFFFFB E1A0F000` | The copied poll loop, still resident |

Physical address 0 is **not** an alias of DDR0. `0x0` reads U-Boot's copied
code while `0x80000000` reads kernel data, so the region U-Boot scribbles on at
step 2 lies outside anything a device tree would describe as memory. A kernel
cannot accidentally overwrite CPU1's holding code.

**No IPI is involved.** The vendor's `boot_secondary()` carries the stock
Realview comment about sending a soft interrupt but has no `smp_cross_call()`
call; it only sets `pen_release` and waits. It does not need one, because CPU1
is already spinning rather than sitting in WFI.

### If you skip SMP

CPU1 is harmless if a mainline kernel never writes `0x20050134`: U-Boot cleared
it at boot, so CPU1 stays in the poll loop at physical address 0. The loop only
reads — one register load, a compare and a branch — and that address is outside
DDR, so it cannot disturb the kernel. A uniprocessor first boot costs nothing
but a core spinning.

One caveat: the register is *not* cleared by a warm restart that skips U-Boot.
Reset through U-Boot (`reset`, or power cycling) before loading a test kernel.

### What a mainline kernel needs

Mainline ARM32 has no generic "write the entry point to a register" enable
method — `spin-table` is arm64-only. Hi3531 needs a `struct smp_operations`,
bound either through the machine descriptor's `.smp` field or through
`CPU_METHOD_OF_DECLARE` plus an `enable-method` in the `cpus` node. It is
around thirty lines:

```c
#define HI3531_SYSCTRL_CPU1_JUMP        0x134

static void __iomem *sysctrl_base;

static void __init hi3531_smp_prepare_cpus(unsigned int max_cpus)
{
        struct device_node *np;

        np = of_find_compatible_node(NULL, NULL, "hisilicon,hi3531-sysctrl");
        sysctrl_base = of_iomap(np, 0);

        scu_enable(scu_a9_get_base());
}

static int hi3531_boot_secondary(unsigned int cpu, struct task_struct *idle)
{
        if (!sysctrl_base)
                return -ENODEV;

        writel_relaxed(__pa_symbol(secondary_startup),
                       sysctrl_base + HI3531_SYSCTRL_CPU1_JUMP);
        return 0;
}

static const struct smp_operations hi3531_smp_ops __initconst = {
        .smp_prepare_cpus       = hi3531_smp_prepare_cpus,
        .smp_boot_secondary     = hi3531_boot_secondary,
};
CPU_METHOD_OF_DECLARE(hi3531_smp, "hisilicon,hi3531-smp", &hi3531_smp_ops);
```

Three differences from the vendor code matter:

- **Write the address in `.smp_boot_secondary`, not in `.smp_prepare_cpus`.**
  Mainline fills in `secondary_data` before calling `smp_boot_secondary()`, so
  by then CPU1 can go straight into `secondary_startup`. The vendor writes it
  early, which releases CPU1 before the stack and page tables exist — which is
  exactly why the vendor needs the holding pen.
- **Drop `pen_release` and `headsmp.S`.** With the write moved, mainline's own
  `secondary_startup` handles the handshake.
- **Leave `.cpu_die` and `.cpu_kill` unimplemented** unless CPU hotplug is
  actually wanted. Once CPU1 has left the poll loop there is nothing to put it
  back, so re-onlining would need a WFI-and-wakeup-IPI path written from
  scratch. Without those ops, `CONFIG_HOTPLUG_CPU` simply refuses to offline it.

`scu_a9_get_base()` reads PERIPHBASE from CP15, so the SCU node is optional for
the code above; declare it anyway for completeness. `.smp_init_cpus` is not
needed — `arm_dt_init_cpu_maps()` populates the possible map from the `cpus`
node.

The closest mainline template is `hi3xxx_smp_ops` in
`arch/arm/mach-hisi/platsmp.c`, which writes `__pa_symbol(secondary_startup)`
to `ctrl_base + ((cpu - 1) << 2)` — the same shape as this single register. **It
is not reusable as-is**: it also calls `hi3xxx_set_cpu()` to de-assert the
core's reset and then `arch_send_wakeup_ipi_mask()`. Hi3531 needs neither, and
those writes would land on unrelated registers.

The device-tree side:

```dts
cpus {
    #address-cells = <1>;
    #size-cells = <0>;
    enable-method = "hisilicon,hi3531-smp";

    cpu@0 {
        device_type = "cpu";
        compatible = "arm,cortex-a9";
        reg = <0>;
    };
    cpu@1 {
        device_type = "cpu";
        compatible = "arm,cortex-a9";
        reg = <1>;
    };
};

scu@20300000 {
    compatible = "arm,cortex-a9-scu";
    reg = <0x20300000 0x100>;
};

sysctrl: system-controller@20050000 {
    compatible = "hisilicon,hi3531-sysctrl", "syscon";
    reg = <0x20050000 0x1000>;
};
```

### Errata

The cores report `CPU variant 0x3` / `CPU revision 0` — Cortex-A9 **r3p0**. For
that revision in an SMP build:

| Erratum | Applies | Vendor sets it |
|---|---|---|
| `ARM_ERRATA_764369` — cache line maintenance by MVA may not succeed | Yes — all revisions, `depends on CPU_V7 && SMP` | No |
| `ARM_ERRATA_775420` — aborting cache maintenance may deadlock | Yes — listed for r3p0 | Not offered by 3.0 |
| `ARM_ERRATA_754322` — faulty MMU translations after an ASID switch | Yes — r2p\*, r3p\* | Yes |
| `ARM_ERRATA_720789` — TLBIASIDIS may broadcast a faulty ASID | No — fixed before r2p0 | No |
| `ARM_ERRATA_751472` — interrupted ICIALLUIS | No — fixed in r3p0 | No |

`764369` is the one to watch: it is SMP-only, the vendor kernel does not enable
it, and nothing in the vendor's single-core-affine interrupt setup would have
exposed it.

## Clocks

Clock and reset control lives in the CRG block at `0x20030000`. The vendor
U-Boot's `board_init()` clears `UART_CKSEL_APB` in `CRG + 0xE4` to source the
UART clock from the APB bus.

`sched_clock` reports the free-running timer as **32 bits at 155 MHz**, which
implies a 155 MHz bus/timer clock.

The Ethernet driver writes `0x003F003F` to `CRG + 0xEC` at probe
("Set system config register 0x200300ec with value 0x003f003f"), confirmed by
a live register read. See [06-ethernet.md](06-ethernet.md).

A live dump of the CRG block is in
[17-register-dumps.md](17-register-dumps.md). There is no public clock-tree
documentation for this SoC; the CRG register meanings must come from the
Hi3531 datasheet in `Hi3531_V100R001C01SPC0D1/00.hardware/chip/`.

## Boot-mode strap

`board/godnet/board.c` reads `SYS_CTRL + 0x8C` (`REG_SYSSTAT`), shifts right 4
and masks 2 bits:

| Value | Boot media |
|---|---|
| 0 | SPI flash |
| 1 | DDR (debug/recovery load) |
| 2, 3 | NAND |

On this board `SYS_CTRL + 0x8C` reads **`0xA0001D00`**, giving
`(0xA0001D00 >> 4) & 0x3 = 0` — **SPI flash**. U-Boot's own
`getinfo bootmode` independently reports `spi`.

To re-read it: `md 0x2005008c 1` at the U-Boot prompt, or `getinfo bootmode`
for the decoded answer. See [03-boot-chain.md](03-boot-chain.md).
