# PCI Express

Two PCIe root complexes are present and probed, and **neither links up**.

```
controller0, config base:0x20800000, mem size:0x800000
controller1, config base:0x20810000, mem size:0x800000
pci 0000:00:00.0: [19e5:3531] type 1 class 0x000604
pcie_read_from_device->501,pcie0 not link up!
pci 0000:02:00.0: [19e5:3531] type 1 class 0x000604
pcie_read_from_device->501,pcie1 not link up!
```

| Controller | Registers | Memory window | Config space | State |
|---|---|---|---|---|
| PCIe0 | `0x20800000` | `0x30000000`–`0x377FFFFF` | `0x40000000` | Not linked |
| PCIe1 | `0x20810000` | `0x60000000`–`0x677FFFFF` | `0x70000000` | Not linked |

The kernel command line carries `pcieclkext=0`, selecting the internal
reference clock.

The vendor filesystem includes PCIe cascade drivers (`pcit_dma_host.ko`,
`pcit_dma_slv.ko`, `mcc_drv_host.ko`, `mcc_usrdev_host.ko`) and the SDK
documents a multi-chip cascade application. The Hi3531 supports chaining
several SoCs over PCIe to build 16- and 32-channel DVRs. **This board does not
use that feature**; the root complexes are enumerated but nothing is attached.
The runtime log line `current system has pci device cnt 1, sdkv 0` is the
vendor application counting one SoC in the cascade.

PCIe can be left disabled for the port. Enabling it would require a
`pcie-hisi`-style controller driver that does not exist for this SoC.
