# SATA and Disk Storage

Standard AHCI behind a port multiplier. Along with Ethernet, this is the other
subsystem that should work almost immediately on a modern kernel — and it is
the key to the whole project, since booting from disk avoids the unportable
flash controllers.

## AHCI controller

| Property | Value |
|---|---|
| Version | AHCI 1.2 (`0001.0200`) |
| Register base | `0x10080000`–`0x1008FFFF` |
| IRQ | 68 |
| Ports implemented | 2 (`0x3` implemented mask) |
| Command slots | 32 |
| Link speed | 3 Gbps (SATA II) |
| Driver | `ahci_platform` |
| Platform device | `ahci.0` |

Capability flags reported at probe:

```
ncq sntf stag pm led clo only pmp pio slum part ccc sxs boh
```

`pmp` (port multiplier support) is present and in use. `SSS` (staggered
spin-up) is set, which disables parallel bus scan.

| Port | Device path | State |
|---|---|---|
| `ata1` | `mmio 0x10080000 port 0x100` | SATA link down — nothing connected |
| `ata2` | `mmio 0x10080000 port 0x180` | SATA link up, 1.5 Gbps |

## Port multiplier

| Property | Value |
|---|---|
| Part | JMicron **JMB321** (U88, confirmed from PCB photo) |
| PCI-style ID | `0x197b:0x0325` rev 0 |
| Spec | Port Multiplier 1.2 |
| Ports | 5 |
| Features | `0x5/0xf` |
| Attached to | `ata2` |

Only one of the five multiplier ports has a drive:

| Multiplier port | State |
|---|---|
| `ata2.00` | Link down |
| `ata2.01` | Link down |
| `ata2.02` | **Link up, 1.5 Gbps — disk attached** |
| `ata2.03` | Link down |
| `ata2.04` | Link down |

**This means the board can host up to five drives on `ata2`, plus one directly
on `ata1`.** For a repurposed server that is a genuinely useful amount of
storage expansion, limited by SATA II bandwidth shared across the multiplier
and by whatever physical connectors and power the chassis provides.

> The number of physical SATA connectors on the board has not been established —
> only the top surface was photographed. Worth checking before planning a
> multi-drive build.

## Attached disk

| Property | Value |
|---|---|
| Model | `WDC WD10EURX-63C57Y0` (WD AV-GP, 1 TB) |
| Firmware | `01.01A01` |
| Capacity | 1,953,525,168 sectors = 1.00 TB / 931 GiB |
| Logical sector | 512 B |
| Physical sector | 4096 B (4Kn-emulated / 512e) |
| Transfer mode | UDMA/133 |
| NCQ | Yes, depth 31/32 |
| Link speed | 1.5 Gbps (SATA I rate, through the multiplier) |
| SCSI address | `1:2:0:0` → `/dev/sda` |

Note the disk negotiates only 1.5 Gbps even though the controller supports
3 Gbps. Whether that is a limitation of the JMB321, the cabling, or the drive
has not been investigated.

### Partitioning

The vendor firmware splits the disk into four equal FAT32 partitions:

| Partition | Size | Mount | Filesystem | Used |
|---|---|---|---|---|
| `/dev/sda1` | 232.8 GB | `/mnt/00` | vfat | 99% |
| `/dev/sda2` | 232.8 GB | `/mnt/01` | vfat | 99% |
| `/dev/sda3` | 232.8 GB | `/mnt/02` | vfat | 99% |
| `/dev/sda4` | 232.8 GB | `/mnt/03` | vfat | 99% |

All four are at 99% — this is normal for a DVR, which fills the disk and then
overwrites the oldest footage. **The partitions contain recorded video.** Confirm
with the owner before repartitioning; there is no backup of the disk contents,
only of the flash.

Mount options in use: `rw,relatime,fmask=0022,dmask=0022,codepage=cp437,iocharset=iso8859-1,shortname=mixed,errors=remount-ro`.

## Mainline support

`ahci_platform` (`drivers/ata/ahci_platform.c`) has supported generic
memory-mapped AHCI for many years, and libata's port-multiplier support handles
the JMB321 generically. No new drivers should be needed.

```dts
sata: sata@10080000 {
    compatible = "hisilicon,hi3531-ahci", "generic-ahci";
    reg = <0x10080000 0x10000>;
    interrupts = <0 68 4>;          /* verify SPI numbering */
    ports-implemented = <0x3>;
};
```

Two things to check:

1. **PHY and clock initialisation.** The SoC's SATA PHY almost certainly needs
   CRG setup before the AHCI block responds. The vendor kernel does this in
   platform code; the sequence would need extracting from the SDK sources under
   `osdrv/kernel/linux-3.0.y/drivers/ata/` or from the Hi3531 datasheet.
   This is the most likely failure point.
2. **Port multiplier enablement.** Mainline requires `CONFIG_SATA_PMP`. It is
   not enabled in all defconfigs.

## Why this matters for the port

`PLAN.md` forbids writing to SPI or NAND, and neither flash controller has a
mainline driver. Booting from SATA resolves both problems at once:

- Keep the vendor U-Boot in SPI-NOR untouched.
- Keep the vendor kernel in NAND untouched, so the DVR firmware remains
  bootable as a fallback.
- Put the modern kernel and root filesystem on the disk, and load it with
  U-Boot's existing `ext2load` or `fatload`.

This gives a reversible, low-risk path to a working general-purpose system.
See [16-porting-roadmap.md](16-porting-roadmap.md).
