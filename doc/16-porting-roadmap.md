# Porting Roadmap and Open Questions

A suggested order of work for a kernel developer taking this on, plus an honest
list of what this documentation does not establish.

## Recommended strategy

**Load a modern kernel over TFTP with the existing vendor U-Boot and put the
root filesystem on the SATA disk, leaving both flash devices untouched.**

This is the lowest-risk, highest-value path:

| Advantage | Why |
|---|---|
| No flash writes | Satisfies the `PLAN.md` constraint; no brick risk |
| Reversible | The DVR firmware stays intact in NAND and boots as before |
| Avoids the two unportable flash controllers | See [04-flash-storage.md](04-flash-storage.md) |
| DDR init already done | The hardest bring-up problem is solved by the existing bootloader |
| TFTP available for iteration | Test kernels never touch persistent storage |

> **The kernel cannot be loaded from the SATA disk.** The installed U-Boot has
> no `sata`, `scsi` or `ide` command, and `ext2load`/`fatload` can only address
> `usb`. SATA is reachable only after Linux brings up `ahci_platform`, so it
> serves as the root filesystem, never as the boot medium. See
> [07-sata-storage.md](07-sata-storage.md#u-boot-cannot-read-the-sata-disk).

### The standalone-boot gap

This plan depends on the Raspberry Pi being present at every boot, which is
fine for bring-up and awkward for a finished server. There is currently **no
way to boot this board unattended without either writing flash or leaving USB
media attached**:

| Route | Status |
|---|---|
| TFTP | Works today; needs the Pi reachable at boot |
| USB stick (`usbboot`, `fatload usb`) | Works, but see the `do_auto_update` risk below before leaving media attached |
| SATA | Not possible from this bootloader |
| Writing a kernel to NAND or SPI-NOR | Forbidden under the current constraint; needs a programmer on hand first |

Closing this gap is a decision for later in the port, not a blocker for any of
the phases below.

The two subsystems that matter for a server — Ethernet and SATA — are both
standard licensed IP with mature mainline drivers. That is the core bet of this
project, and it looks sound.

## Phase 1 — Boot a mainline kernel to a serial console

1. Set up cross-compilation for ARMv7 (`arm-linux-gnueabihf`).
2. Write a minimal device tree: CPU, GIC, timers, memory, PL011 UART0.
   Bases and IRQs are in [01-soc-overview.md](01-soc-overview.md). Use
   `arm,sp804` at `0x20000000`/`0x20010000` on IRQ 35 for timekeeping — that is
   the path the vendor kernel uses; the Cortex-A9 TWD is present but untested
   here. Leave the L2
   cache out — it is a HiSilicon block with no mainline driver, and the kernel
   boots without it. See
   [the L2 section](01-soc-overview.md#l2-cache-controller).
3. Build with `earlycon` on the PL011 at `0x20080000`.
4. Load over TFTP from the Raspberry Pi and boot with `bootm`, writing nothing.

Success criterion: kernel output on `ttyAMA0` reaching "Kernel panic - not
syncing: VFS: Unable to mount root filesystem", which means CPU, memory,
interrupts and console all work.

The unknowns here are the GIC base within the Cortex-A9 private region
(`0x20300000`) and the exact SPI numbering. Both come from the datasheet and
from `arch/arm/mach-godnet/` in the SDK kernel.

> **The MCU watchdog is not something a port has to handle.** Once armed, the
> AT89S52 hard-resets the SoC about 60 seconds after the kicks stop — measured —
> but it is only armed by a command 7 the vendor application sends, and the MCU
> shares the SoC's reset, so no armed state can survive into your kernel. A
> mainline kernel has been run on this board without incident. It only bites if
> you kill the vendor application while leaving the kernel up. See
> [20-front-panel-mcu.md](20-front-panel-mcu.md#the-mcu-watchdog).

## Phase 2 — Reclaim the memory

Describe both DRAM banks in the device tree and drop `mem=224M`. This should
take the machine from 224 MB to around 1 GB. Verify the upper bound of each
bank empirically — see the warning in [02-memory-map.md](02-memory-map.md).

## Phase 3 — Ethernet

`stmmac` with a small glue driver for the CRG and pinmux writes. See
[06-ethernet.md](06-ethernet.md). Expect the RGMII mode and delay settings to
need experimentation.

Success criterion: DHCP and SSH.

## Phase 4 — SATA and root filesystem

`ahci_platform` plus `CONFIG_SATA_PMP`. The likely obstacle is SATA PHY
initialisation via CRG. See [07-sata-storage.md](07-sata-storage.md).

Then repartition the disk — **after confirming with the owner that the recorded
video on the four FAT32 partitions can be destroyed** — and install a root
filesystem. At this point the machine is a usable server.

## Phase 5 — Optional extras

In rough order of value:

| Item | Effort | Notes |
|---|---|---|
| Watchdog | Low | Confirmed ARM SP805 — `arm,sp805` node and the APB clock |
| USB | Low–medium | Standard EHCI/OHCI, needs PHY glue |
| RTC | Low | On-chip PL031 (`arm,pl031`) is trivial but has no battery. Battery-backed external chip needs `i2c-gpio` + `rtc-ds1307`; pins are known |
| SD/MMC | Medium | `dw_mmc` may fit; socket may not exist |
| L2 cache | Medium | Forward-port the vendor `cache-hil2v200.c`; performance only, boots without it |
| Front panel, buzzer, alarm relays | Low–medium | All behind the AT89S52 on `ttyAMA1`. Protocol fully recovered and verified on the wire — userspace serial, no kernel driver needed |
| Audio | High | Needs an ASoC platform driver written from scratch |
| Video output | Very high | Proprietary VOU, no documentation |
| Video capture | Impractical | See [11-video-input.md](11-video-input.md) |
| H.264 codec | Impossible | See [14-media-codec.md](14-media-codec.md) |

## Open questions

Ranked by how much they would change the work.

1. **A TVT GPL source release.** Would supply the vendor's own board file,
   confirming the pinmux and GPIO detail derived here from the `pinctrl_*.sh`
   scripts. See [15-product-identity.md](15-product-identity.md) for search
   terms.
2. **Underside PCB photographs.** The SPI-NOR, NAND, RTC and regulators are
   unlocated. These have to be taken directly —
   the unit has no FCC ID, so there is no FCC filing with internal photographs
   to fall back on (see [15-product-identity.md](15-product-identity.md)).
3. **A flash programmer (CH341A + SOIC-8 clip) for recovery.** Not needed while
   the no-flash-writes rule holds, but essential before the first flash write.
4. **The identity of `U16`.** A 56-pin TI part beside the VGA and HDMI
   connectors, marked `PN521` / `35KG4` / `AL2R`. The marking does not resolve
   to a confirmed part number; identifying it needs a TI marking
   cross-reference. Not on any path a server build depends on.
5. **The FPGA's I²C address and register map**, and what its bitstream
   implements. Only matters if the video capture path is ever revived.
6. **~~The AT89S52 protocol.~~ Resolved.** Recovered and verified on the wire in
   both directions: the relay, buzzer and LED frames; the 2 Hz status broadcast
   carrying the alarm inputs; and the command 1 key event with all 23 buttons
   mapped. Enough to drive the front panel, buzzer and relays from a port
   today. What is left is detail — whether a held key repeats, and the constant
   byte 2 in the status broadcast. See
   [20-front-panel-mcu.md](20-front-panel-mcu.md#key-events).

## Quick reference

Where the answers to the most commonly needed questions live.

| Question | Answer | Detail in |
|---|---|---|
| How much RAM, and where? | 512 MB at `0x80000000`, 512 MB at `0xC0000000` | [02-memory-map.md](02-memory-map.md) |
| Device-tree interrupt numbers? | SPI = `/proc/interrupts` number − 32 | [01-soc-overview.md](01-soc-overview.md) |
| Ethernet `phy-mode`? | Plain `rgmii`, plus the CRG+0xEC bit layout | [06-ethernet.md](06-ethernet.md) |
| Where is the MAC address? | `/etc/init.d/mac.dat`; factory master at SPI-NOR `0xBFC20` | [04-flash-storage.md](04-flash-storage.md) |
| Bit-banged I²C pins? | SDA = GPIO12_4, SCL = GPIO12_5 | [19-pinmux-map.md](19-pinmux-map.md) |
| What does the board boot from? | SPI-NOR (`getinfo bootmode` → `spi`) | [03-boot-chain.md](03-boot-chain.md) |
| Is the L2 cache a PL310? | No — HiSilicon L2 Cache V200, no mainline driver | [01-soc-overview.md](01-soc-overview.md#l2-cache-controller) |
| Which timer drives the clock? | ARM SP804 at `0x20000000`, IRQ 35 — not the A9 TWD | [01-soc-overview.md](01-soc-overview.md#timers) |
| Is the SoC watchdog portable? | Yes — confirmed ARM SP805, use `arm,sp805` | [10-rtc-watchdog-misc.md](10-rtc-watchdog-misc.md#the-ip-is-an-sp805) |
| Is there an RTC with a mainline driver? | Yes — on-chip ARM PL031, but no battery backup | [10-rtc-watchdog-misc.md](10-rtc-watchdog-misc.md#on-chip-rtc--an-arm-pl031) |
| Where does the rear RS485 go? | UART2 / `ttyAMA2` at `0x200A0000`, IRQ 42, 9600 | [05-uart-console.md](05-uart-console.md#rs485-rear-panel) |
| How do I read the front-panel keys? | Parse `0A 01 <code> <hold>` on `ttyAMA1`; 23 codes mapped | [20-front-panel-mcu.md](20-front-panel-mcu.md#key-codes) |
| How do I read the alarm inputs? | Don't poll — the MCU broadcasts them at 2 Hz, active low | [20-front-panel-mcu.md](20-front-panel-mcu.md#the-mcu-status-broadcast) |
| What triggers an alarm input? | A dry contact shorting it to ground; 5.2 V pull-up | [10-rtc-watchdog-misc.md](10-rtc-watchdog-misc.md#alarm-inputs-are-dry-contact-active-low) |
| Where is the FPGA bitstream? | `.rodata` of `fpga_jtag.ko`, a Lattice VME file | [11-video-input.md](11-video-input.md) |
| Which external codecs are fitted? | None — no ADV7179, TLV320AIC31 or SiI9024 | [12-video-output.md](12-video-output.md) |

### Out of scope

Not documented beyond what appears elsewhere in this set: the internals of the
proprietary media pipeline, and the auto-update mechanism's image format.

## Risks

| Risk | Mitigation |
|---|---|
| SoC watchdog resets a kernel that does not service it | U-Boot disables it; keep it disabled or add the driver early |
| **MCU watchdog** resets the board ~60 s in | Not a porting risk — it only fires once armed by command 7, and the MCU shares the SoC reset so no arm survives into a new kernel. Only bites if you kill the vendor app without rebooting — see [20-front-panel-mcu.md](20-front-panel-mcu.md#the-mcu-watchdog) |
| U-Boot auto-update reflashes from attached media | Understand `do_auto_update` before leaving USB media attached |
| Repartitioning destroys recorded video | Confirm with the owner first; there is no disk backup |
| A bad flash write bricks the board | Do not write flash until a programmer is on hand; backups exist in `backups/2026-08-03/` |
| SATA PHY init missing, disk never appears | Extract the sequence from the SDK kernel sources |
