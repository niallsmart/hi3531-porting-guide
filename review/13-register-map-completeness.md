# Register-Map Completeness

## Observation

The README and SoC overview describe the register base map as complete or
authoritative for device-tree work. The SDK and datasheet contain additional
potentially useful blocks that are absent from the table, including I2C, SPI,
SIO, DMAC, cipher, eFuse, internal SRAM and boot ROM, additional timer blocks,
and several media register ranges.

## Questions to investigate

- Is the current table intended to be exhaustive or only a port-focused subset?
- Which omitted blocks matter to the stated goal of enabling as much usable
  hardware as practical?
- Should all known blocks be added, or should the wording explicitly define the
  table's narrower scope?
- Are the stated sizes for existing entries register-window sizes, mapped
  resource sizes, or only the portions observed in use?

