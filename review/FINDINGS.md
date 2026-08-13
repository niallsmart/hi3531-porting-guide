# Review Findings

Responses to the investigation prompts in this directory. Each item is
reported as **confirmed**, **disproved**, or **uncertain**, with the exact
source used, per the request in [README.md](README.md).

Where a change was warranted, every affected occurrence was located rather
than only the cited example; the "Changed" row lists them all.

## Status

| # | Topic | Verdict | Commit |
|---|---|---|---|
| 1 | [U-Boot SATA loading](01-uboot-sata-loading.md) | **Confirmed** | `5835778` |
| 2 | [Device-tree interrupt numbers](02-device-tree-interrupts.md) | Not yet investigated | |
| 3 | [SP804 timer topology](03-sp804-timer-topology.md) | Not yet investigated | |
| 4 | [Device-tree handoff from U-Boot](04-device-tree-handoff.md) | Not yet investigated | |
| 5 | [Access to the upper DRAM bank](05-upper-dram-bank.md) | Not yet investigated | |
| 6 | [Secondary CPU startup](06-secondary-cpu-startup.md) | Not yet investigated | |
| 7 | [Pinmux provenance and completeness](07-pinmux-map.md) | Not yet investigated | |
| 8 | [Scope of the register documentation](08-register-documentation.md) | Not yet investigated | |
| 9 | [GPIO and watchdog confidence](09-gpio-watchdog.md) | Not yet investigated | |
| 10 | [DDR0 MMZ arithmetic](10-ddr0-mmz-arithmetic.md) | Not yet investigated | |
| 11 | [Ethernet DTS details](11-ethernet-dts.md) | Not yet investigated | |
| 12 | [Internal contradictions](12-internal-contradictions.md) | Not yet investigated | |
| 13 | [Register-map completeness](13-register-map-completeness.md) | Not yet investigated | |

---

## 1. U-Boot SATA loading — **Confirmed**

The installed U-Boot cannot load a kernel from the SATA disk. The observation
was correct, and the inference from `godnet.h` to `common/Makefile` holds.

### Evidence

**Live command list.** `help` at the U-Boot prompt on this board returns no
`sata`, `scsi` or `ide`. The storage-capable commands are `ext2load`, `ext2ls`,
`fatload`, `fatls`, `fatinfo`, `usb`, `usbboot`, `nand`, `nboot`, `sf`, with
`tftp` and `bootm`. Captured to `dvr-uboot.log` on the Raspberry Pi.

**Configuration.**
`osdrv/uboot/u-boot-2010.06/include/configs/godnet.h` defines
`CONFIG_CMD_EXT2`, `CONFIG_CMD_FAT`, `CONFIG_CMD_USB`, `CONFIG_USB_STORAGE`,
`CONFIG_DOS_PARTITION`. It defines none of `CONFIG_CMD_SATA`,
`CONFIG_CMD_SCSI`, `CONFIG_CMD_IDE`, and contains no occurrence of "sata",
"scsi", "ide" or "ahci".

**Interface registration.** `osdrv/uboot/u-boot-2010.06/disk/part.c`, lines
50–73, gates each entry of `block_drvr[]` on its command's config symbol:

```c
#if defined(CONFIG_CMD_SATA)
	{.name = "sata", .get_dev = sata_get_dev, },
#endif
#if defined(CONFIG_CMD_USB) && defined(CONFIG_USB_STORAGE)
	{ .name = "usb", .get_dev = usb_stor_get_dev, },
#endif
```

`usb` is the only surviving entry. `CONFIG_MMC` appears at `godnet.h:272` but
inside `#ifdef CONFIG_AUTO_SD_UPDATE`, and the shipped build has no `mmc`
command, so that branch is off.

So `ext2load` and `fatload` can address exactly one interface, `usb`, and
`ext2load sata …` fails inside `get_dev()`.

**The `SATA` string in the flash image is not evidence of support.**
`backups/2026-08-03/spi-nor/dhb-ax-spi-nor-cold-a.bin` contains the sequence
`IDE / SATA / ATAPI / USB / DOC / MMC` at offset `0x3db63`. That is
`if_typename[]` from `disk/part.c`, a static label table compiled in
unconditionally.

### Answers to the specific questions

| Question | Answer |
|---|---|
| Unlisted mechanism registering SATA as a block device? | No. `block_drvr[]` is the only registration path and SATA is compiled out |
| Can `ext2load`/`fatload` name a SATA interface? | No. `usb` is the only registered interface |
| Should the kernel come from TFTP or USB, with SATA as rootfs only? | Yes. That is now the documented strategy |
| Would direct SATA loading require rebuilding U-Boot? | More than that — see below |

### Rebuilding U-Boot would not be sufficient

`drivers/block/ahci.c` in this tree is PCI-only: `ahci_init_one(pci_dev_t
pdev)` reached through `pci_find_devices`. The Hi3531's AHCI is a
memory-mapped platform device, and U-Boot 2010.06 predates the
`CONFIG_SCSI_AHCI_PLAT` platform path that later releases use for this case.
The other SATA drivers present — `fsl_sata`, `sata_dwc`, `sata_sil3114` — target
other vendors' controllers. Enabling SATA in the bootloader means writing a
platform AHCI driver.

### Consequence beyond the wording

The strategy itself survives: the kernel arrives over TFTP and SATA carries the
root filesystem. `doc/16`'s Phase 1 step 4 already said TFTP, so only the
headline sentence disagreed with the plan beneath it.

But correcting it exposed a gap that was not previously stated anywhere:
**there is no way to boot this board unattended without either writing flash or
leaving USB media attached**, and the USB route runs into the `do_auto_update`
risk already listed in `doc/16`. Recorded as
[the standalone-boot gap](../doc/16-porting-roadmap.md#the-standalone-boot-gap).

### Changed

| File | What |
|---|---|
| `doc/16-porting-roadmap.md` | Strategy restated as TFTP + SATA rootfs; added a warning box and the standalone-boot gap section |
| `doc/07-sata-storage.md` | Rewrote the porting rationale; added "U-Boot cannot read the SATA disk" with the full evidence |
| `doc/04-flash-storage.md` | Option 1 no longer claims SATA loading |
| `doc/03-boot-chain.md` | "boot from USB or disk" → USB storage, with a note that `usb` is the only registered interface |
| `doc/README.md` | Finding 2 restated |
