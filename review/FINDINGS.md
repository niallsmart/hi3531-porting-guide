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
| 10 | [DDR0 MMZ arithmetic](10-ddr0-mmz-arithmetic.md) | **Confirmed** | |
| 11 | [Ethernet DTS details](11-ethernet-dts.md) | Not yet investigated | |
| 12 | [Internal contradictions](12-internal-contradictions.md) | **Confirmed** (all four) | |
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

---

## 10. DDR0 MMZ arithmetic — **Confirmed**

The table said 288 MB where the zone is 282 MiB. All four sub-questions
resolve yes.

### Evidence

The running kernel's own zone list settles it:

```
+---ZONE: PHYS(0x8E000000, 0x9F9FFFFF), GFP=0, nBYTES=288768KB, NAME="anonymous"
+---ZONE: PHYS(0x9FA00000, 0x9FEFFFFF), GFP=0, nBYTES=5120KB,   NAME="jpeg"
+---ZONE: PHYS(0xC0000000, 0xDF7FFFFF), GFP=0, nBYTES=516096KB, NAME="ddr1"
 total size=809984KB(791MB),...
```

`288768 KB = 282 MiB`. The `288` in the table was the KB figure read as MB.
The address span agrees independently: `0x9FA00000 − 0x8E000000 = 0x11A00000`
= 295,698,432 bytes = 282 MiB. So does the total the same file prints —
282 + 5 + 504 = 791 MiB = 809,984 KB, which is exact; 288 + 5 + 504 would be
797 MiB and would not match.

### Answers to the specific questions

| Question | Answer |
|---|---|
| 282 MiB rather than 288 MiB? | Yes |
| Is DDR0 224 + 282 + 5 = 511 MiB? | Yes |
| ~1 MiB at the end of DDR0 outside the documented regions? | Yes — `0x9FF00000`–`0x9FFFFFFF` exactly |
| Reserved by firmware, or unused? | **Unused.** Nothing in `/proc/iomem` covers it, and `/dev/mem` reads at `0x9FF00000` and `0x9FFFFFF0` return the same uninitialised-DRAM pattern as unallocated addresses inside the MMZ zones |

The 1 MiB gap is a vendor convention rather than this board's choice.
HiSilicon's own reference script,
`Hi3531_SDK_V1.0.D.1/mpp/ko/load3531:90`, uses
`anonymous,0,0x84000000,447M:ddr1,0,0xC0000000,511M` — stopping exactly 1 MB
short of the top of *each* bank. Every MMZ variant in this board's
`rootfs/mtd/modules/load3531` ends the `jpeg` zone at `0x9FF00000` regardless
of where `anonymous` starts. A port that describes the full 512 MB in the
device tree reclaims it along with everything else.

### A second arithmetic error found alongside

`doc/02-memory-map.md` also stated `0x84000000 + 447 MB = 0x9FFC0000`. It is
`0x9FF00000`. The conclusion drawn from it — "within 1 MB of a full bank" —
was right, so nothing downstream changed.

### Changed

| File | What |
|---|---|
| `doc/02-memory-map.md` | 288 → 282 MB; sum corrected to 511; the two bank tails given their own rows and explained; `0x9FFC0000` → `0x9FF00000` |

The `~790 MB` figure used in `doc/README.md`, `doc/14-media-codec.md` and the
summary of `doc/02` is unaffected — it rounds 791 MiB, not the DDR0 subtotal.

---

## 12. Internal contradictions — **Confirmed**, all four

Each of the four was a stale passage surviving next to a later, better-evidenced
one. In every case the later passage was the correct one.

### Audio codec

**Confirmed.** `doc/13-audio.md` establishes across three sections that the
TLV320AIC31 is not fitted and that the codec is the NVP1104B's PCM engine, then
closes with "bind mainline `tlv320aic31xx` to it". The closing paragraph also
told the reader to "confirm the codec is physically present, find its I²C
address" — both already answered earlier in the same file (U19, I²C `0x60`).

The intended path is indeed a new NVP1104B codec driver, and it is not the only
piece missing: the SoC side needs an ASoC platform driver for SIO4 as well, as
the *Mainline support* section already said. Rewrote the recommendation to
state the full order of work and to say plainly that `tlv320aic31xx` is not a
shortcut.

### MCU watchdog experiment

**Confirmed — the passage predates the experiment.** "Two things are not
established: the MCU's timeout (only that it exceeds 30 seconds), and what it
actually does when the timeout expires" sat about sixty lines below the
measured result (≈ 60 s, hard reset, one-shot) and described the very
experiment that had since been run. Deleted.

### MCU watchdog takeover

**Confirmed, and the two cases genuinely differ.** Both statements were true of
different situations, and the document never distinguished them. Replaced the
bare "a replacement userspace daemon has to take over that duty" with a table:

| Situation | Watchdog state | What your code must do |
|---|---|---|
| Clean boot into your own kernel | Disarmed — nothing sent command 7, and the preceding reset cleared the MCU | Nothing |
| Taking over UART1 on the running vendor system | Armed and being kicked every 30 s | Kick it yourself, or reset within 60 s |

### GIC confidence

**Confirmed — `doc/01` was right and `doc/16` was stale.** The SDK gives the
addresses outright in
`osdrv/kernel/linux-3.0.y/arch/arm/mach-godnet/include/mach/platform.h`:

```c
#define ARM_INTNL_BASE      0x20300000
#define A9_GIC_OFFSET       0x100
#define A9_GIC_DIST         0x1000
#define CFG_GIC_CPU_BASE    (IO_ADDRESS(ARM_INTNL_BASE) + A9_GIC_OFFSET)
#define CFG_GIC_DIST_BASE   (IO_ADDRESS(ARM_INTNL_BASE) + A9_GIC_DIST)
```

`core.c:123` passes both to `gic_init()`, and `platsmp.c:50` reads the SCU at
`REG_BASE_A9_PERI + REG_A9_PERI_SCU`. The same header gives the global timer at
`+0x200` and the private timer/TWD at `+0x600`. SPI numbering is equally
settled: `gic_init(0, GODNET_IRQ_START, …)` with `GODNET_IRQ_START == 32`,
which is exactly the −32 rule `doc/01` documents.

Nothing here is inferred from the PERIPHBASE convention, so `doc/16`'s
"unknowns" was wrong on both counts. What does remain open is the per-peripheral
**trigger type** in the third interrupt cell, which `doc/01` already flags;
`doc/16` now says that instead.

### Changed

| File | What |
|---|---|
| `doc/13-audio.md` | Assessment rewritten: NVP1104B codec driver + SIO4 ASoC platform driver, `tlv320aic31xx` ruled out |
| `doc/20-front-panel-mcu.md` | Stale "two things are not established" paragraph deleted; watchdog-takeover advice split into the two cases |
| `doc/16-porting-roadmap.md` | GIC/SPI "unknowns" replaced with the settled values and a pointer; trigger type named as the remaining check |
| `doc/01-soc-overview.md` | New [private peripheral region](../doc/01-soc-overview.md#the-cortex-a9-private-peripheral-region) table (SCU, GIC CPU, global timer, TWD, distributor) with the SDK citation; register-map row and GIC evidence row point to it |
