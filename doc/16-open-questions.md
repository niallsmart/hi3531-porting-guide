# Open Questions

What remains unknown about this board, and what this documentation set does not
cover. For the state of the Linux port and the peripherals it does not yet
drive, see
[dhb-ax-buildroot](https://github.com/niallsmart/dhb-ax-buildroot).

## Unresolved

Ranked by how much they would change the work.

1. **A TVT GPL source release.** Would supply the vendor's own board file,
   confirming the pinmux and GPIO detail derived here from the `pinctrl_*.sh`
   scripts. See [15-product-identity.md](15-product-identity.md) for search
   terms.
2. **Underside PCB photographs.** The SPI-NOR, NAND, RTC and regulators are
   unlocated. These have to be taken directly — the unit has no FCC ID, so there
   is no FCC filing with internal photographs to fall back on (see
   [15-product-identity.md](15-product-identity.md)).
3. **A flash programmer (CH341A + SOIC-8 clip) for recovery.** Not needed while
   the no-flash-writes rule holds, but essential before the first flash write.
4. **The serial console pin header location.** The UART pinout is known and the
   console works; where the header sits on the board is not recorded. See
   [05-uart-console.md](05-uart-console.md).
5. **The identity of `U16`.** A 56-pin TI part beside the VGA and HDMI
   connectors, marked `PN521` / `35KG4` / `AL2R`. The marking does not resolve
   to a confirmed part number; identifying it needs a TI marking
   cross-reference. Not on any path a server build depends on.
6. **The FPGA's I²C address and register map**, and what its bitstream
   implements. Only matters if the video capture path is ever revived.
   Recoverable by tracing the bit-banged I²C bus — see
   [11-video-input.md](11-video-input.md#a-possible-route-to-capture-on-a-modern-kernel).

## No unattended boot path

This board cannot boot without either writing flash or leaving USB media
attached. The constraint is the bootloader's, not the hardware's:

| Route | Status |
|---|---|
| USB (`usbboot`, `fatload usb`) | Works. Weigh the `do_auto_update` risk before leaving media attached — see [03-boot-chain.md](03-boot-chain.md) |
| TFTP | Works, and needs a host reachable at boot |
| SATA | Not possible. This U-Boot has no `sata`, `scsi` or `ide` command, and `ext2load`/`fatload` address only `usb` — see [07-sata-storage.md](07-sata-storage.md#u-boot-cannot-read-the-sata-disk) |
| Kernel in NAND or SPI-NOR | Forbidden under the no-flash-writes rule; needs a programmer on hand first |

## Out of scope

Not documented beyond what appears elsewhere in this set: the internals of the
H.264 codec path, and the auto-update mechanism's image format.

Note that "media" and "undocumented" are not the same set. The datasheet has
register descriptions for every block on this SoC except the codecs (VDH, VEDU,
JPGD, JPGE), the graphics blocks (TDE, VPSS, VCMP) and the HDMI transmitter.
Coverage map in
[18-reference-assets.md](18-reference-assets.md#the-hi3531-datasheet--what-it-does-and-does-not-document).
