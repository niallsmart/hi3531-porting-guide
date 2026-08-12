# Flash Storage — SPI-NOR and NAND

The board has two flash devices: a small SPI-NOR holding the bootloader, and a
128 MB parallel NAND holding the kernel and filesystems. Both use
HiSilicon-proprietary controllers.

## SPI-NOR

| Property | Value |
|---|---|
| Part | Spansion **S25FL216K** |
| Capacity | 2 MB |
| Erase block | 64 KB |
| JEDEC ID | `0x01 0x40 0x15` |
| Chip select | CS1 |
| Controller | HiSilicon SFC "V300", driver version 1.10 |
| Controller registers | `0x10010000` |
| Memory window | `0x58000000` |
| MTD name | `hi_sfc` |

### Layout

Mapped by scanning the recovered image for non-erased regions:

| Offset | Size | Contents |
|---|---|---|
| `0x000000`–`0x045000` | 277 KB | U-Boot 2010.06 (contains an embedded JPEG at `0x0364DB`) |
| `0x080000`–`0x081000` | 4 KB used | U-Boot environment (629 bytes of a 128 KB region) |
| `0x0A0000` | — | Auto-update parameter block — **currently erased (all `0xFF`)** |
| `0x0BF000`–`0x0C0000` | 64 KB | **Board parameter block** (see below) |
| `0x0C0000`–`0x0D2000` | 71 KB | Boot splash JPEG (SOI at `0x0C0000`, size `0x1156F`) |
| `0x0FF000`–`0x100000` | 64 KB | **Mirror of the board parameter block** — byte-identical |

The whole 2 MB is presented to Linux as one partition, `mtd0` / `mtdblock0`,
named `boot`. Nothing mounts it.

The U-Boot environment references the splash as `jpeg_addr=0xc0000000`,
`jpeg_size=0x1156f` — note `0xC0000000` is the *RAM* address it is decoded to
(in DDR1), while `0x0C0000` is where it lives in flash.

### Board parameter block

The two identical 64 KB blocks at `0x0BF000` and `0x0FF000` hold per-unit
factory data as NUL-padded ASCII fields at fixed offsets:

| Offset (within block) | Absolute | Contents |
|---|---|---|
| `+0xC20` | `0x0BFC20` | **MAC address** — `00:18:ae:3c:a2:49` |
| `+0xC40` | `0x0BFC40` | `v1.2` — board revision, matching the `DHB_AX V1.2` PCB silkscreen |
| `+0xD20` | `0x0BFD20` | `0008` |
| `+0xE40` | `0x0BFE40` | `1156f` — the splash JPEG size |
| `+0xE60` | `0x0BFE60` | `a0224Mhdr000000` — boot argument fragments |
| `+0xE80` | `0x0BFE80` | `5091eb04513d503c5091ef22` — the `auversion*` values concatenated |
| `+0xFF8` | `0x0BFFF8` | `ZZZZ` — end marker |

This is the factory master copy of the MAC address; the working copy the vendor
application actually reads is `/etc/init.d/mac.dat` in the rootfs. See
[06-ethernet.md](06-ethernet.md).

> The auto-update parameter area at `0xA0000` being fully erased is reassuring
> — the `do_auto_update` path that runs on every boot has nothing queued.

## NAND

| Property | Value |
|---|---|
| Manufacturer ID | `0x01` (AMD/Spansion per the kernel's table) |
| Device ID | `0xF1` |
| Full ID | `0x01 0xF1 0x00 0x1D` |
| Capacity | 128 MB |
| Page size | 2 KB |
| Block size | 128 KB |
| OOB size | 64 bytes |
| ECC | 1-bit, hardware |
| Bus width | 8-bit, 3.3 V |
| Controller | HiSilicon NANDC "V301", driver version 1.10 |
| Controller registers | `0x10000000` |
| Memory window | `0x50000000` |
| MTD name | `hinand` |

> The kernel prints `AMD NAND 128MiB 3,3V 8-bit` from its generic ID table.
> This is a decode of the JEDEC ID, **not** a reading of the package marking.
> The actual part has not been identified from the PCB photos. This matters
> only if you need exact timing parameters.

### Partitions

Defined entirely by the `mtdparts=` kernel argument — there is no on-flash
partition table.

| Device | Offset | Size | Name | Mount | Filesystem |
|---|---|---|---|---|---|
| `mtd1` | `0x0000000` | 8 MB | `kernel` | — | raw uImage |
| `mtd2` | `0x0800000` | 16 MB | `rootfs` | `/` | yaffs2 |
| `mtd3` | `0x1800000` | 64 MB | `user` | `/mnt/mtd` | yaffs2 |
| `mtd4` | `0x5800000` | 32 MB | `hdr000000` | `/mnt/bak` | yaffs2 |

Usage at capture time: rootfs 11.4 MB of 16 MB (71%), user 41.8 MB of 64 MB
(65%), hdr000000 1.3 MB of 32 MB (4%).

### Kernel image

Parsed from `backups/2026-08-03/nand-kernel/dhb-ax-nand-kernel-cold-a.bin`:

| Field | Value |
|---|---|
| Magic | `0x27051956` (valid uImage) |
| Name | `Linux-3.0.8` |
| Built | 2013-03-11 03:23:48 UTC |
| Data size | 3,629,636 bytes (3.46 MiB) |
| Load address | `0x80008000` |
| Entry point | `0x80008000` |
| OS / Arch | Linux / ARM |
| Type | Kernel image |
| Compression | **None** |

The build timestamp matches `uname -a` (`Mon Mar 11 11:23:32 CST 2013`),
confirming the backup is the running kernel.

Note `bootcmd` reads `0x500000` (5 MB) from NAND into RAM — comfortably more
than the 3.46 MB image plus header.

## Mainline support

**Neither controller has a mainline driver.** This is one of the two genuine
blockers for a modern kernel (the other being the media pipeline).

Options, roughly in order of effort:

1. **Boot from SATA instead.** The vendor U-Boot can already `ext2load` and
   `fatload`. Keep the vendor bootloader in SPI-NOR, keep the vendor kernel
   partition untouched, and put the modern kernel and root filesystem on the
   1 TB disk. This sidesteps both flash controllers entirely and is the
   fastest route to a working server. **Recommended.**
2. **Boot over TFTP/NFS.** Same idea, no local storage at all. Best during
   bring-up.
3. **Port the NAND driver.** The SDK ships source under
   `osdrv/kernel/linux-3.0.y/drivers/mtd/nand/` — a HiSilicon `hinand` driver
   that would need forward-porting across ~15 years of MTD API change. Only
   worth it if you must boot from NAND.

Since `PLAN.md` prohibits writing to SPI or NAND, options 1 and 2 are also the
only ones available without relaxing that constraint.

## Backups

`backups/2026-08-03/` holds a complete, verified set:

| File | Size | Contents |
|---|---|---|
| `spi-nor/dhb-ax-spi-nor-cold-a.bin` | 2 MB | Full SPI-NOR — U-Boot + environment |
| `nand-physical/dhb-ax-nand-raw-oob-a.bin` | 132 MB | Full NAND, raw with OOB |
| `nand-kernel/dhb-ax-nand-kernel-cold-a.bin` | 8 MB | `kernel` partition |
| `filesystems/dhb-ax-rootfs-files-a.tar` | 9.4 MB | rootfs contents |
| `filesystems/dhb-ax-user-files-a.tar` | 35 MB | user partition contents |
| `filesystems/dhb-ax-hdr000000-files-a.tar` | 2 KB | hdr000000 contents |

The raw NAND image is exactly 65,536 pages × (2048 + 64) bytes = 138,412,032
bytes, confirming it is a complete page+OOB dump at the stated geometry.

Each layer was read twice and compared, SHA-256 recorded in `MANIFEST.md`, with
a second copy on the Raspberry Pi at
`/home/niallsmart/dhb_ax/backups/2026-08-03/`.

> **Restoring these requires hardware.** The images are safe, but writing them
> back to a bricked board needs a SPI flash programmer (e.g. CH341A with a
> SOIC-8 clip) or a working HiSilicon serial boot/burn path. Confirm you have
> one before the first flash write.
