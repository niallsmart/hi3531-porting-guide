# Device-Tree Interrupt Numbers

## Observation

`doc/01-soc-overview.md`, lines 260–304, derives this rule:

> DT SPI number = vendor Linux IRQ number - 32

The device-tree sketches elsewhere use the vendor Linux IRQ values directly:

- UART0 uses 40 in `doc/05-uart-console.md`, line 133.
- Ethernet uses 119 in `doc/06-ethernet.md`, line 178.
- SATA uses 68 in `doc/07-sata-storage.md`, line 111.
- EHCI and OHCI use 63 and 64 in `doc/08-usb.md`, lines 75 and 81.

## Questions to investigate

- What GIC interrupt encoding does the intended current kernel require?
- If the central rule applies, should the examples use SPI values 8, 87, 36,
  31, and 32 respectively?
- Are the documented trigger types correct for each peripheral?
- Can every DTS example be reconciled with the central interrupt table and
  validated against the current GIC binding or a minimal boot?

