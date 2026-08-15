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
4. **The identity of `U16`.** A 56-pin TI part beside the VGA and HDMI
   connectors, marked `PN521` / `35KG4` / `AL2R`. The marking does not resolve
   to a confirmed part number; identifying it needs a TI marking
   cross-reference. Not on any path a server build depends on.
5. **The FPGA's I²C address and register map**, and what its bitstream
   implements. Only matters if the video capture path is ever revived.
   Recoverable by tracing the bit-banged I²C bus — see
   [11-video-input.md](11-video-input.md#a-possible-route-to-capture-on-a-modern-kernel).

## Out of scope

Not documented beyond what appears elsewhere in this set: the internals of the
H.264 codec path, and the auto-update mechanism's image format.

Note that "media" and "undocumented" are not the same set. The datasheet has
register descriptions for every block on this SoC except the codecs (VDH, VEDU,
JPGD, JPGE), the graphics blocks (TDE, VPSS, VCMP) and the HDMI transmitter.
Coverage map in
[18-reference-assets.md](18-reference-assets.md#the-hi3531-datasheet--what-it-does-and-does-not-document).
