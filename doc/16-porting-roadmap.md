# Porting Roadmap and Open Questions

A suggested order of work for a kernel developer, plus the remaining gaps.

## Recommended strategy

**Load a modern kernel over TFTP with the existing vendor U-Boot and put the
root filesystem on the SATA disk, leaving both flash devices untouched.**

This is the lowest-risk, highest-value path:

| Advantage | Why |
|---|---|
| No flash writes | Satisfies the project constraint; no brick risk |
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
   Use the verified bases, IRQs and timer node in
   [01-soc-overview.md](01-soc-overview.md); leave the optional L2 cache out.
3. Build with `earlycon` on the PL011 at `0x20080000`.
4. Build a **zImage** with `CONFIG_ARM_APPENDED_DTB=y`, append the DTB, then
   wrap it with `mkimage`. Either disable ATAG compatibility or replace the
   U-Boot command line with the verified two-bank `mem=` override. Recipe and
   reasoning:
   [03-boot-chain.md](03-boot-chain.md#getting-a-device-tree-into-a-modern-kernel).
5. Load over TFTP from the Raspberry Pi and boot with `bootm`, writing nothing.

**Boot CPU0 only to start with.** Describe one `cpu@0` and add the second core
in Phase 5. U-Boot safely parks CPU1; see
[Secondary CPU startup](01-soc-overview.md#secondary-cpu-startup).

Success criterion: kernel output on `ttyAMA0` reaching "Kernel panic - not
syncing: VFS: Unable to mount root filesystem", which means CPU, memory,
interrupts and console all work.

## Phase 2 — Reclaim the memory

Describe both DRAM banks in the device tree, drop `mem=224M`, and **set
`CONFIG_VMSPLIT_2G`**. This should take the machine from 224 MB to around 1 GB.

Before exposing DDR1 to the page allocator, either stop U-Boot's VOU pipelines
or reserve and inherit its framebuffer. The measured configuration and both
handoff choices are in [12-video-output.md](12-video-output.md#u-boot-handoff).

DDR1 lies outside a default 3G/1G kernel's low-memory window.
`CONFIG_HIGHMEM` is the alternative; see
[making both banks usable](02-memory-map.md#making-both-banks-usable).

Two failure modes to recognise if the memory does not appear:

| Kernel reports | Cause |
|---|---|
| 224 MB | The vendor `ATAG_MEM` and `mem=224M` survived. Disable ATAG compatibility, or replace `bootargs` and supply both measured banks with `mem=` as described in the boot-chain chapter |
| ~512 MB, with `Ignoring RAM` in the log | 3G split without `CONFIG_HIGHMEM`; DDR1 was dropped |

## Phase 3 — Ethernet

`stmmac` on **GMAC1 at `0x101C4000`** — the second instance, which is the one
wired to the connector. GMAC0 has no PHY; leave it out of the tree.

Use `compatible = "hisilicon,hi3531-dwmac", "snps,dwmac"`, not
`snps,dwmac-3.60a`, which the upstream binding does not accept. The pinmux needs
nothing: RGMII1 is already muxed by U-Boot and Linux never touches it. The glue
must redirect generic stmmac to GMAC1 and DMA channel 1 within the shared block,
reset all three DMA channels after warm boot, mask the TNK aggregator to the
GMAC1/DMA1 sources, and implement `fix_mac_speed` against `CRG + 0xEC` bits
[31:16]. Use enhanced descriptors, but leave checksum offload, jumbo frames,
PMT/WOL and forced TX store-and-forward disabled unless independently proven.

See [06-ethernet.md](06-ethernet.md). If the link comes up but no traffic
passes, that is RGMII delay, and it is a board-level strap question — no driver
in this path can set it.

Success criterion: DHCP and SSH.

## Phase 4 — SATA and root filesystem

Use the Hi3531 AHCI glue plus `CONFIG_SATA_PMP`. The validated glue performs
the required clock, reset, PHY and port-workaround sequence before handing the
controller to libata. See [07-sata-storage.md](07-sata-storage.md).

Then repartition the disk — **after confirming with the owner that the recorded
video on the four FAT32 partitions can be destroyed** — and install a root
filesystem. At this point the machine is a usable server.

## Phase 5 — Optional extras

In descending order of value, then ascending effort:

| Item | Value | Effort | Notes |
|---|---|---|---|
| Watchdog | High | Medium | `wdt-sp805` binds and counts at the measured 3 MHz rate, but its second expiry does not reset the SoC; Hi3531 reset routing remains to be identified |
| The second CPU | High | Low–medium | ~30 lines of `smp_operations`: enable the SCU, write `__pa_symbol(secondary_startup)` to `SYS_CTRL + 0x134`. No reset or IPI needed — U-Boot leaves CPU1 running and polling. Also enable `ARM_ERRATA_764369` and `775420` for A9 r3p0 SMP. [Detail](01-soc-overview.md#secondary-cpu-startup) |
| USB | High | Low | Validated generic EHCI/OHCI hosts with one shared Hi3531 PHY provider |
| Front panel, buzzer, alarm relays | High | Low–medium | All behind the AT89S52 on `ttyAMA1`. Protocol fully recovered and verified on the wire — userspace serial, no kernel driver needed |
| RTC | Medium | Low | Use the validated battery-backed device at `0x68` with `i2c-gpio` + `rtc-ds1307`. The on-chip PL031 needs a Hi3531 unlock/reset quirk and has no battery |
| L2 cache | Medium | Medium | Forward-port the vendor `cache-hil2v200.c`; performance only, boots without it |
| Hardware SPI | Low | Low | An ARM PL022 at `0x200C0000`, SPI 12 — `spi-pl022` binds with no override. The board bit-bangs instead, so the pins need muxing to function 1 |
| Spare timers | Low | Low | Four SP804 blocks, not two. Timers 1–3 at `0x20010000`, `0x20130000`, `0x20140000` are unused: six spare 32-bit timers |
| Inherited firmware framebuffer | Low | Low | U-Boot leaves one 480x300 ARGB1555 buffer mirrored to its main HD path and both CVBS outputs; reserve it and use `simple-framebuffer` for a fixed diagnostic display |
| SD/MMC | Low | Medium | `dw_mmc` may fit; socket may not exist. Its pins are function 4 of the `VIU3` run, so it excludes the fourth video input |
| DMA | Low | Medium | An ARM PL080 at `0x100D0000`, but its periphid collides with mainline's Samsung PL080S entry — read the warning before enabling |
| Audio | Low | High | Needs an ASoC platform driver written from scratch |
| Native video output | Low | Very high | VDP is documented, so a VGA DRM/KMS driver is possible. The on-chip HDMI transmitter and TDE/VPSS are not documented. 800x600 is the vendor Linux mode, not a hardware limit |
| Video capture | Low | Impractical | VICAP itself is fully documented; the analogue decoder and FPGA feeding it are not. See [11-video-input.md](11-video-input.md) |
| H.264 codec | Low | Impossible | See [14-media-codec.md](14-media-codec.md) |

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

## Out of scope

Not documented beyond what appears elsewhere in this set: the internals of the
H.264 codec path, and the auto-update mechanism's image format.

Note that "media" and "undocumented" are not the same set. The datasheet has
register descriptions for every block on this SoC except the codecs (VDH, VEDU,
JPGD, JPGE), the graphics blocks (TDE, VPSS, VCMP) and the HDMI transmitter.
Coverage map in
[18-reference-assets.md](18-reference-assets.md#the-hi3531-datasheet--what-it-does-and-does-not-document).
