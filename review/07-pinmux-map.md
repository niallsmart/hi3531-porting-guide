# Pinmux Map Provenance and Completeness

## Observation

`doc/19-pinmux-map.md` presents a complete IO_CONFIG map derived primarily from
vendor shell-script comments. It also notes that offsets `0x1ac` through
`0x240` are absent from those scripts and that some GPIO comments conflict.

The supplied Hi3531 datasheet appears to contain the complete mux descriptions.
Examples noticed during this review include:

- `0x1bc` is documented as GPIO13_5 / RGMII1_RXDV.
- The function-1 run through `0x1ec` covers the other RGMII1 signals.
- `0x1f0` and `0x1f4` include RGMII1 error-signal alternatives.
- At `0x0004`, function 1 appears to be GPIO0_1 rather than GPIO0_4.
- At `0x000c`, function 1 appears to be GPIO0_3 rather than GPIO0_6.

## Questions to investigate

- Can every IO_CONFIG row be reconciled against the chip datasheet?
- Should the shell scripts be used only to establish this board's selected
  values, rather than as the authoritative function map?
- Can the RGMII run currently described as undocumented or unconfirmed in
  `doc/17-register-dumps.md` be identified definitively?
- Are there other shifted or duplicated GPIO labels inherited from script
  comments?

