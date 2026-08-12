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
| System timers | ARM SP804 (part `0x804`, designer ARM) at `0x20000000`, `0x20010000` | `arm,sp804` |
| SMP | Cortex-A9 MPCore SCU at `0x20300000` | `arm,cortex-a9-scu` |
| Ethernet | Synopsys DesignWare MAC 1000 (ID 0x36) | `stmmac` |
| SATA | AHCI 1.2 | `ahci_platform` |
| USB | EHCI + OHCI | `ehci-platform`, `ohci-platform` |
| SD/MMC | Synopsys DesignWare (`hi_mci`) | `dw_mmc` |

Every row above was checked against the SDK sources *and* the running device,
rather than inferred from the IP pairing. The evidence:

| Block | Hardware evidence | Vendor software evidence |
|---|---|---|
| UART | AMBA peripheral ID at `0x20080FE0` = `0x11`, `0x10`, `0x24`, `0x00` → part `0x011`, designer `0x41` (ARM), rev 2 | `CONFIG_SERIAL_AMBA_PL011=y`; dmesg `ttyAMA0 at MMIO 0x20080000 (irq = 40) is a PL011 rev2` |
| GIC | Distributor at `0x20301000`, CPU interface at `0x20300100` — the standard Cortex-A9 PERIPHBASE layout | `gic_init()` in `mach-godnet/core.c`; `/proc/interrupts` shows `GIC` for every line |
| Timers | Peripheral ID at `0x20000FE0` → part `0x804` (SP804), designer ARM | `CONFIG_LOCAL_TIMERS` **not** set; SP804 register use in `core.c` |
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

Taken from the SDK U-Boot header
`arch/arm/include/asm/arch-godnet/platform.h`, cross-checked against
`/proc/iomem` on the running device. This is the authoritative address map for
writing a device tree.

| Base | Size | Block |
|---|---|---|
| `0x10000000` | 0x100 | NAND flash controller (NANDC) |
| `0x10010000` | 0x100 | SPI flash controller (SFC) |
| `0x10020000` | 0x1000 | SD/MMC controller (`hi_mci`) |
| `0x10080000` | 0x10000 | SATA / AHCI |
| `0x100A0000` | 0x10000 | USB 2.0 OHCI |
| `0x100B0000` | 0x10000 | USB 2.0 EHCI |
| `0x101C0000` | 0x20000 | Ethernet (two DWMAC1000 instances) |
| `0x20000000` | 0x1000 | Timer 0 — ARM SP804 dual timer |
| `0x20010000` | 0x1000 | Timer 1 — ARM SP804 dual timer |
| `0x20030000` | 0x100 | CRG — clock and reset generator |
| `0x20040000` | — | Watchdog |
| `0x20050000` | — | System controller (SYS_CTRL) |
| `0x20060000` | — | On-chip RTC |
| `0x20070000` | — | IR receiver |
| `0x20080000` | 0x1000 | UART0 (console) |
| `0x20090000` | 0x1000 | UART1 |
| `0x200A0000` | 0x1000 | UART2 |
| `0x200B0000` | 0x1000 | UART3 |
| `0x200F0000` | 0x200 | IO_CONFIG — pin multiplexing |
| `0x20110000` | — | DDR controller 0 |
| `0x20120000` | — | DDR controller 1 |
| `0x20150000` | 0x10000 each | GPIO group 0 … GPIO group 18 (`0x20150000 + n*0x10000`) |
| `0x20300000` | — | Cortex-A9 private peripherals (SCU, GIC, TWD) |
| `0x20400000` | — | ARM debug |
| `0x20700000` | 0x1000 | L2 cache controller (HiSilicon L2 Cache V200) |
| `0x20800000` | — | PCIe0 controller registers |
| `0x20810000` | — | PCIe1 controller registers |
| `0x30000000` | 0x7800000 | PCIe0 memory window |
| `0x40000000` | — | PCIe0 configuration space |
| `0x50000000` | 0x880 | NAND memory-mapped window |
| `0x58000000` | 0x4000000 | SPI flash memory-mapped window |
| `0x60000000` | 0x7800000 | PCIe1 memory window |
| `0x70000000` | — | PCIe1 configuration space |
| `0x80000000` | — | DDR0 (see [02-memory-map.md](02-memory-map.md)) |
| `0xC0000000` | — | DDR1 |

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

From `/proc/interrupts` on the running device. All are GIC SPIs.

| IRQ | Source |
|---|---|
| 35 | System Timer Tick / Free Timer |
| 40 | UART0 (`ttyAMA0`) |
| 41 | UART1 |
| 42 | UART2 |
| 43 | UART3 (registered, idle) |
| 48 | IR receiver (`Hi_IR`) |
| 61 | HiSilicon DMAC |
| 63 | EHCI (`ehci_hcd:usb1`) |
| 64 | OHCI (`ohci_hcd:usb2`) |
| 67 | SD/MMC (`hi_mci`) |
| 68 | SATA / AHCI |
| 69, 70, 71 | L2 cache — chk0, chk1, com |
| 79, 80 | VPSS0, VPSS1 |
| 88 | IVE |
| 90 | VIU (video input) |
| 91 | VOU (video output) |
| 92, 93 | VEDU_0, VEDU_1 (H.264 encoders) |
| 94 | JPEGU_0 |
| 95 | x5_jpeg |
| 96, 97 | VDEC, VDEC_1 |
| 98 | TDE (2D engine) |
| 100 | VDA (video detect/analysis) |
| 101 | vcmp |
| 102 | VOIE |
| 103 | SCD |
| 119 | Ethernet (`stmmaceth`) — shared by both MACs |

Inter-processor interrupts IPI0–IPI5 are the standard ARM SMP set.

Note the SoC declares `NR_IRQS:128`. Only CPU0 services peripheral interrupts
in the vendor configuration — every peripheral IRQ shows a zero count on CPU1.

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
level-high (4). Verify the trigger type per peripheral before relying on it.

## Timers

Timekeeping is done by two **ARM SP804** dual-timer blocks, not by the Cortex-A9
private timer. `CONFIG_LOCAL_TIMERS` is *not* set in `godnet_defconfig`, so the
TWD is left idle even though the hardware has it.

| Property | Value |
|---|---|
| Timer 0 (clockevent) | `0x20000000` |
| Timer 1 (clocksource / `sched_clock`) | `0x20010000` |
| IRQ | 35 (SPI 3), `TIMER01_IRQ`, shared by both halves |
| Clock | 155 MHz (310 MHz bus ÷ 2 fixed prescale) |

Identification is from the AMBA peripheral ID registers at `0x20000FE0`, read
live: `0x04`, `0x18`, `0x14`, `0x00` → part number `0x804`, designer `0x41`
(ARM), revision 1. That is an SP804, which mainline drives with `arm,sp804`
(`drivers/clocksource/timer-sp804.c`).

The control register at `0x20000008` reads `0xE2` on the running device —
enable (bit 7), periodic (bit 6), interrupt enable (bit 5), 32-bit (bit 1) —
matching the standard SP804 `TimerXControl` layout. Timer 0's load value is
`0x0017A6B0` (1,550,000), which at HZ=100 gives the 155 MHz figure.

Because there is no per-CPU local timer, CPU1 receives its ticks by IPI: on the
running device `IPI0 Timer broadcast interrupts` has ~2.58 M counts on CPU1 and
zero on CPU0. A mainline port can either keep this arrangement with `arm,sp804`,
or enable the TWD at PPI 29 — the TWD path is untested on this hardware, and
the A9 TWD clock scales with the CPU clock, which is why vendors often avoid it.

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
