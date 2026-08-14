# SATA and Disk Storage

Standard AHCI behind a port multiplier. Libata handles the controller and
multiplier after SoC-specific glue has enabled the clocks and programmed the
Hi3531 PHY.

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

The SoC has two dedicated activity-LED outputs, `SATA_LED_N0` and `SATA_LED_N1`,
multiplexed against GPIO18_3 and GPIO18_4 at `0x200f0254` and `0x200f0258`. Both
registers read 0 on this board, so the LED function is not selected and the
pins are plain GPIO. See [19-pinmux-map.md](19-pinmux-map.md).

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

The controller topology supports five drives on `ata2` plus one on `ata1`.
The PCB has ten SATA footprints, but only two connectors are populated; usable
expansion also depends on chassis power and cabling.

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

`ahci_platform` and libata supply the generic host and port-multiplier support,
but the generic driver alone is not sufficient. With its clocks gated, the
Hi3531 AHCI register window reads as zero. A small `ahci_hi3531` glue driver is
required to enable the three clocks at `CRG + 0xB4`, sequence the host, PHY and
lane resets, program both PHYs and apply the two vendor port workarounds before
calling `ahci_platform_init_host()`.

```dts
sata: sata@10080000 {
    compatible = "hisilicon,hi3531-ahci";
    reg = <0x10080000 0x10000>,
          <0x20030000 0x100>;
    reg-names = "ahci", "syscfg";
    interrupts = <0 36 4>;          /* SPI 36, level-high — vendor IRQ 68 */
};
```

The controller reports its two-port implemented mask itself once initialized,
so the validated node does not override `ports-implemented`.

The reconciled Linux 6.18.42 port has exercised this path. It enumerated AHCI
1.2 with 32 command slots, the five-port JMicron multiplier and the attached
1 TB disk, and completed raw read I/O. `CONFIG_SATA_PMP` is required. The disk
was kept unmounted and no write was issued.

The initialization sequence came from the vendor 3.0.8
`hi_sata_init()` and its Godnet configuration. Two details remain worth
remembering:

1. The vendor defaults configure both PHYs for 1.5 Gbps, even though the AHCI
   capability register advertises 3 Gbps.
2. `CRG + 0xB4` is shared SoC state, so the local glue maps that second window
   without claiming exclusive ownership.

## Role in the port

The recommended TFTP-kernel/SATA-root workflow is in
[16-porting-roadmap.md](16-porting-roadmap.md).

Neither flash controller has a mainline driver. A SATA root would avoid both,
but it is not required for bring-up:

- Keep the vendor U-Boot in SPI-NOR untouched.
- Keep the vendor kernel in NAND untouched, so the DVR firmware remains
  bootable as a fallback.
- Put the root filesystem on SATA only when the disk contents may be replaced,
  or use an NFS root while preserving the recordings.

This gives a reversible, low-risk path to a working general-purpose system.
See [16-porting-roadmap.md](16-porting-roadmap.md).

## U-Boot cannot read the SATA disk

The disk is not reachable until Linux has initialised `ahci_platform`. **The
kernel therefore cannot live on it** — it arrives over TFTP or from USB, and
SATA carries the root filesystem only.

Four independent checks agree:

| Check | Result |
|---|---|
| `help` at the U-Boot prompt | No `sata`, `scsi` or `ide` command |
| `include/configs/godnet.h` | No `CONFIG_CMD_SATA`, `CONFIG_CMD_SCSI` or `CONFIG_CMD_IDE`; no occurrence of "sata", "scsi", "ide" or "ahci" at all |
| `disk/part.c` | `block_drvr[]` registers an interface name only when its command is enabled. `usb` is the only one that survives — `CONFIG_MMC` exists but is inside `#ifdef CONFIG_AUTO_SD_UPDATE`, and the shipped build has no `mmc` command |
| The `SATA` string in the flash image | Part of `if_typename[]` in `disk/part.c`, a static label table compiled in unconditionally. Not evidence of support |

So `ext2load` and `fatload` can address exactly one interface, `usb`.
`ext2load sata …` fails inside `get_dev()`.

**Rebuilding U-Boot would not be enough either.** This tree's
`drivers/block/ahci.c` is PCI-only — `ahci_init_one(pci_dev_t pdev)`, reached
through `pci_find_devices` — while the Hi3531's AHCI is a memory-mapped
platform device. U-Boot 2010.06 predates the `CONFIG_SCSI_AHCI_PLAT` platform
path that later releases use for exactly this case. The other SATA drivers in
the tree (`fsl_sata`, `sata_dwc`, `sata_sil3114`) are for other vendors'
controllers. Enabling SATA in the bootloader means writing a platform AHCI
driver, not flipping a config switch.
