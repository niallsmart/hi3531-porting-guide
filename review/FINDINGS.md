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
| 2 | [Device-tree interrupt numbers](02-device-tree-interrupts.md) | **Confirmed** | `0b2bdad` |
| 3 | [SP804 timer topology](03-sp804-timer-topology.md) | **Confirmed** | `c17240e` |
| 4 | [Device-tree handoff from U-Boot](04-device-tree-handoff.md) | **Confirmed** | `296668b` |
| 5 | [Access to the upper DRAM bank](05-upper-dram-bank.md) | **Confirmed** | `6193589` |
| 6 | [Secondary CPU startup](06-secondary-cpu-startup.md) | **Confirmed** | |
| 7 | [Pinmux provenance and completeness](07-pinmux-map.md) | Not yet investigated | |
| 8 | [Scope of the register documentation](08-register-documentation.md) | Not yet investigated | |
| 9 | [GPIO and watchdog confidence](09-gpio-watchdog.md) | Not yet investigated | |
| 10 | [DDR0 MMZ arithmetic](10-ddr0-mmz-arithmetic.md) | **Confirmed** | `a6ac3ed` |
| 11 | [Ethernet DTS details](11-ethernet-dts.md) | Not yet investigated | |
| 12 | [Internal contradictions](12-internal-contradictions.md) | **Confirmed** (all four) | `a6ac3ed` |
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
"unknowns" was wrong on both counts. The one thing genuinely still open at this
point was the per-peripheral **trigger type**, and [item 2](#2-device-tree-interrupt-numbers--confirmed)
closed that too by reading the GIC's configuration registers.

### Changed

| File | What |
|---|---|
| `doc/13-audio.md` | Assessment rewritten: NVP1104B codec driver + SIO4 ASoC platform driver, `tlv320aic31xx` ruled out |
| `doc/20-front-panel-mcu.md` | Stale "two things are not established" paragraph deleted; watchdog-takeover advice split into the two cases |
| `doc/16-porting-roadmap.md` | GIC/SPI "unknowns" replaced with the settled values and a pointer |
| `doc/01-soc-overview.md` | New [private peripheral region](../doc/01-soc-overview.md#the-cortex-a9-private-peripheral-region) table (SCU, GIC CPU, global timer, TWD, distributor) with the SDK citation; register-map row and GIC evidence row point to it |

---

## 2. Device-tree interrupt numbers — **Confirmed**

Every DTS example put the vendor Linux IRQ number in the SPI cell, contradicting
the rule the same document set establishes. All five need the SPI value the
reviewer suggests.

### The encoding a current kernel requires

`Documentation/devicetree/bindings/interrupt-controller/arm,gic.yaml` — three
cells: `<type, number, flags>`. For `type = 0` (SPI) the second cell is the
**SPI index**, which the kernel offsets by 32 internally to reach the GIC
interrupt ID. It is neither the GIC ID nor the number `/proc/interrupts` prints.

This is a silent failure mode, which is why it is worth stating in the document
rather than only fixing the examples. The tree compiles, `request_irq` succeeds
on the wrong line, and the peripheral never interrupts — for a console UART,
output that works and input that does not.

### The corrections

| Peripheral | Was | Now | File |
|---|---|---|---|
| UART0 | `<0 40 4>` | `<0 8 4>` | `doc/05-uart-console.md` |
| Ethernet | `<0 119 4>` | `<0 87 4>` | `doc/06-ethernet.md` |
| SATA | `<0 68 4>` | `<0 36 4>` | `doc/07-sata-storage.md` |
| EHCI | `<0 63 4>` | `<0 31 4>` | `doc/08-usb.md` |
| OHCI | `<0 64 4>` | `<0 32 4>` | `doc/08-usb.md` |

Exactly the five the reviewer predicted. These are all the `interrupts =`
properties in `doc/`; the sixth DTS block, the `i2c-gpio` node in
`doc/19-pinmux-map.md`, has none.

### Trigger types — correct, and now measured

The `4` in every example is right. Read from the live GIC distributor rather
than assumed:

| Register | Address | Value | Covers | Meaning |
|---|---|---|---|---|
| `GICD_ICFGR0` | `0x20301C00` | `0xAAAAAAAA` | IDs 0–15, SGIs | All edge — architecturally fixed |
| `GICD_ICFGR1` | `0x20301C04` | `0x7DC00000` | IDs 16–31, PPIs | Edge on 27, 29, 30; level elsewhere |
| `GICD_ICFGR2`–`7` | `0x20301C08`–`1C1C` | `0x55555555` | IDs 32–127, SPIs | `0b01` in every field — bit 1 clear, **level** |

So every SPI on this SoC is level-sensitive and `IRQ_TYPE_LEVEL_HIGH` applies
throughout. The GIC only supports level-high and edge-rising in any case, so
there is no active-low variant to get wrong.

The three edge PPIs decode as ID 27 global timer, ID 29 private timer (TWD) and
ID 30 private watchdog — textbook Cortex-A9, and independent corroboration that
PERIPHBASE is where [item 12](#gic-confidence) puts it.

### Distributor identity

The same read confirms the block itself, which had rested on the SDK header
alone:

| Register | Address | Value | Decode |
|---|---|---|---|
| `GICD_CTLR` | `0x20301000` | `0x00000001` | Enabled |
| `GICD_TYPER` | `0x20301004` | `0x0000FC23` | 128 interrupt lines, 2 CPU interfaces, security extensions present |
| `GICD_IIDR` | `0x20301008` | `0x0102043B` | Implementer `0x43B` = ARM, revision 2 |
| `GICC_CTLR` | `0x20300100` | `0x00000001` | CPU interface enabled |
| `GICC_PMR` | `0x20300104` | `0x000000F0` | Linux's usual priority mask |

128 lines matches the `NR_IRQS:128` the vendor kernel reports, and 2 CPU
interfaces matches the dual Cortex-A9.

### Validation against a boot

Not done, and it cannot be from here — this is a documentation exercise and
booting a mainline kernel is the port itself. What has been done is stronger
than a desk check: the numbers come from the SDK's `irqs.h`, the trigger types
and the distributor's geometry come from reading the live GIC, and the two
agree. A minimal boot remains the final confirmation, as Phase 1 of
[the roadmap](../doc/16-porting-roadmap.md) already frames it.

### Changed

| File | What |
|---|---|
| `doc/05-uart-console.md`, `doc/06-ethernet.md`, `doc/07-sata-storage.md`, `doc/08-usb.md` | SPI values corrected in all five nodes; the "verify SPI numbering" comments replaced with the settled value and its vendor-IRQ cross-reference |
| `doc/01-soc-overview.md` | Warning that the second cell is the SPI index, not the Linux IRQ; the ICFGR/TYPER/IIDR evidence for level-high throughout |
| `doc/16-porting-roadmap.md` | Trigger type no longer listed as an open per-peripheral check |

---

## 3. SP804 timer topology — **Confirmed**

The reviewer's reading of the SDK is correct in every particular. The vendor
kernel uses **both internal timers of the single block at `0x20000000`**, and
`doc/01` had them as one timer in each of two blocks.

### What the SDK actually does

`arch/arm/mach-godnet/include/mach/time.h`:

```c
#define CFG_TIMER_VABASE        IO_ADDRESS(TIMER0_BASE)   /* 0x20000000 */
#define CFG_TIMER2_VABASE       IO_ADDRESS(TIMER1_BASE)   /* 0x20010000 */
```

Every timer access in `core.c` goes through `CFG_TIMER_VABASE`. The clockevent
uses `REG_TIMER_*` (`core.c:182`–`210`), and the clocksource and `sched_clock`
use `REG_TIMER1_*` (`core.c:146`, `158`, `258`, `280`–`283`) — which
`platform.h:112`–`118` defines as offsets `0x020`–`0x038`, the *second timer of
the same block*, not a second block. `CFG_TIMER2_VABASE` appears nowhere else in
the tree.

Both handlers are installed on the one line, `core.c:301`–`302`:

```c
setup_irq(TIMER01_IRQ, &godnet_timer_irq);
setup_irq(TIMER01_IRQ, &godnet_freetimer_irq);
```

Each checks its own raw-interrupt-status register — `REG_TIMER_RIS` at
`core.c:228`, `REG_TIMER1_RIS` at `core.c:238` — which is the usual arrangement
for an SP804's combined output. `/proc/interrupts` shows the pair sharing one
line: `35: 405833 0 GIC System Timer Tick, Free Timer`.

### Answers to the specific questions

**Should the initial DT describe one SP804 at `0x20000000` using its two
internal timers on SPI 3?** Yes, and that is now what `doc/01` and the roadmap
say.

**Is `0x20010000` a second SP804 on DT SPI 4?** Yes, on both counts.

| Address | `0xFE0`–`0xFEC` | `0xFF0`–`0xFFC` | Decode |
|---|---|---|---|
| `0x20000FE0` | `04 18 14 00` | `0D F0 05 B1` | Part `0x804`, designer `0x41` (ARM), rev 1 |
| `0x20010FE0` | `04 18 14 00` | `0D F0 05 B1` | Identical — a second SP804 |
| `0x20020FE0` | Bus error | Bus error | No third block |

`irqs.h:7` gives `TIMER23_IRQ = GODNET_IRQ_START + 4` = Linux IRQ 36 = DT SPI 4.
It is never requested — IRQ 36 does not appear in `/proc/interrupts`.

The control registers confirm which block is live:

| Register | Value | State |
|---|---|---|
| `0x20000008`, `0x20000028` | `0xE2` | Enabled, periodic, IRQ enabled, 32-bit |
| `0x20010008`, `0x20010028` | `0x20` | IRQ enable only, timers **disabled** |
| `0x20010004`, `0x20010024` | `0xFFFFFFFF` | Reset state, never loaded |

**What does the current binding expect?**
`Documentation/devicetree/bindings/timer/arm,sp804.yaml` requires `arm,sp804`
plus `arm,primecell`, one `reg`, one or two interrupts, and either one clock or
three (timer 1, timer 2, APB) with names `timer1`/`timer2`/`apb_pclk`.

`drivers/clocksource/timer-sp804.c` resolves clocks **by index, not by name** —
`of_clk_get(np, 0)` for timer 1, and `of_clk_get(np, 1)` for timer 2 only when
the node declares three — so one clock feeding both is equally valid. It takes a
single IRQ per node via `irq_of_parse_and_map(np, 0)`, and with
`arm,sp804-has-irq` absent it makes **timer 1 the clockevent and timer 2 the
clocksource plus `sched_clock`**. That is exactly the vendor's split, so a
single node reproduces the working configuration with no property gymnastics.

### One detail worth stating

The ÷2 that gives 155 MHz is external to the block. `CFG_TIMER_PRESCALE` is a
macro applied to the bus clock in `time.h:20`–`22`, while the SP804's own
prescale field (control bits 3:2) is zero in the live `0xE2`. So 155 MHz is the
frequency arriving at the timer, and that is the rate a `clocks` property must
advertise — not 310 MHz with a divider.

### Changed

| File | What |
|---|---|
| `doc/01-soc-overview.md` | Timers section rewritten: one block with two internal timers, the second block identified and marked unused, live control-register evidence, a DTS node, and the prescale note. Register-map and driver/evidence rows updated to match |
| `doc/16-porting-roadmap.md` | Phase 1 step 2 now says one `arm,sp804` node at `0x20000000` on SPI 3, rather than both bases on IRQ 35 |

---

## 4. Device-tree handoff from U-Boot — **Confirmed**

The installed U-Boot has no FDT support of any kind, so an appended DTB is the
only route. Investigating it turned up a trap that would have cost most of the
board's RAM.

### No FDT support, confirmed three ways

`include/configs/godnet.h` defines no `CONFIG_OF_LIBFDT`, and `boot_get_fdt()`
is reachable only from `common/cmd_bootm.c:303` inside
`#if defined(CONFIG_OF_LIBFDT)` — so `bootm <kernel> - <dtb>` ignores its third
argument rather than failing.

The handoff itself is decisive. `arch/arm/lib/bootm.c:149`:

```c
theKernel (0, machid, bd->bi_boot_params);
```

`bi_boot_params` is assigned once, at `board/godnet/board.c:101`, to
`CFG_BOOT_PARAMS` = `0x80000100`. No path in this build puts a DTB address in
r2. The live `help` output has no `fdt` command.

**Two red-herring strings.** The SPI backup contains `Device Tree:` at `0x3BE96`
and `Flat Device Tree` at `0x3C8B0`. The first is the header `usb tree` prints
(`common/cmd_usb.c:578`); the second is a label in `uimage_type[]`
(`common/image.c:140`), unconditional. Recorded in `doc/03` because a grep of
the flash image is exactly how someone would talk themselves out of this
conclusion — the same shape as the `SATA` string in item 1.

### Answers to the specific questions

| Question | Answer |
|---|---|
| How does the first modern kernel receive its DTB? | Appended to a zImage. There is no alternative short of replacing the bootloader |
| Is `CONFIG_ARM_APPENDED_DTB` appropriate? | Yes — it is the only option. Note it lives in the decompressor, so the uImage must wrap a **zImage**, not the uncompressed `Image` the vendor ships |
| Is `CONFIG_ARM_ATAG_DTB_COMPAT` needed to retain the U-Boot command line? | **No, and it must be left off** — see below |
| Does this `bootm` understand a usable multi-image format? | `IH_TYPE_MULTI` is compiled in, but it is no help: without libfdt, r2 is always the ATAG pointer regardless of what the image contains |
| Would rebuilding U-Boot with libfdt be preferable? | Technically nicer, practically no. The rebuilt bootloader has to be written to SPI-NOR to be used, which the project constraints forbid and which is the only real brick risk on the board |

### The ATAG trap — it would have cost 800 MB

`ARM_ATAG_DTB_COMPAT` reads as the obviously right option for an un-upgradable
bootloader. On this board it silently reimposes the vendor's memory limit, by
two independent routes:

| Route | What U-Boot supplies | Effect |
|---|---|---|
| `ATAG_MEM` | One bank, `0x80000000`, 256 MB | `atags_to_fdt()` **overwrites** `/memory` `reg` wholesale. DDR1 vanishes, DDR0 halves |
| `ATAG_CMDLINE` | `bootargs` beginning `mem=224M` | `early_mem()` clamps to 224 MB whatever the DT says |

Both U-Boot figures are constants, not measurements: `dram_init()` at
`board/godnet/board.c:130` assigns `bi_dram[0]` from `CFG_DDR_PHYS_OFFSET` and
`CFG_DDR_SIZE` with no probe, `CONFIG_NR_DRAM_BANKS` is 1, and
`setup_memory_tags()` emits that single bank. The board has 512 MB in each of
two banks, measured.

The overwrite is unconditional in
`arch/arm/boot/compressed/atags_to_fdt.c` — `if (memcount) setprop(fdt,
"/memory", "reg", …)` — so choosing `CMDLINE_EXTEND` over
`CMDLINE_FROM_BOOTLOADER` does not rescue it. Those options govern only how
`/chosen/bootargs` is merged, and `mem=224M` survives an extend just as it
survives a replace. The net result of enabling `COMPAT` would be a kernel
reporting 224 MB — the vendor's exact limit, arrived at silently, with the
device tree's own `/memory` node discarded.

With it off, the decompressor puts the appended DTB's address in r2 and the ATAG
list is never read. The cost is that `setenv bootargs` stops having any effect,
so the command line moves into the DTB's `/chosen`.

`machid` is a non-issue either way: U-Boot passes `MACH_TYPE_GODNET` in r1, and
a DT kernel matches on the root node's `compatible` and ignores it.

### Changed

| File | What |
|---|---|
| `doc/03-boot-chain.md` | New "Getting a device tree into a modern kernel" section: the no-FDT evidence, the two red-herring strings, the appended-DTB recipe, the `ATAG_DTB_COMPAT` analysis, `machid`, and why not to rebuild U-Boot |
| `doc/02-memory-map.md` | Warning on the two-bank `memory` node that it only survives with `ARM_ATAG_DTB_COMPAT` off |
| `doc/16-porting-roadmap.md` | New Phase 1 step 4 for packaging; Phase 2 names the option as the first thing to check if memory is still 224 MB; quick-reference row; risks-table entry |

The `mkimage` recipe and the `0x82000000` load address are marked untested —
nothing here has booted a mainline kernel, and that remains Phase 1's job.

---

## 5. Access to the upper DRAM bank — **Confirmed**

The premises are all correct — the vendor `godnet_defconfig` has
`CONFIG_VMSPLIT_3G=y`, `CONFIG_PAGE_OFFSET=0xC0000000`, `CONFIG_HIGHMEM` unset,
`CONFIG_FLATMEM=y` — and the concern behind them is real. **Two `reg` entries
are not enough.** A kernel that is otherwise correct will boot with 512 MB.

### The arithmetic

`adjust_lowmem_bounds()` in `arch/arm/mm/mmu.c` sets the physical ceiling of low
memory to

```
vmalloc_limit = VMALLOC_END − vmalloc_size − VMALLOC_OFFSET − PAGE_OFFSET + PHYS_OFFSET
```

`VMALLOC_END` is `0xFF800000` (`arch/arm/include/asm/pgtable.h`),
`VMALLOC_OFFSET` is 8 MB, `vmalloc_size` defaults to 240 MB, and `PHYS_OFFSET`
is `0x80000000` on this board:

| Split | `PAGE_OFFSET` | Low-memory window | DDR0 | DDR1 |
|---|---|---|---|---|
| `VMSPLIT_3G` | `0xC0000000` | `0x80000000`–`0xAFFFFFFF` (768 MB) | Low | **High** |
| `VMSPLIT_3G_OPT` | `0xB0000000` | `0x80000000`–`0xBFFFFFFF` (1024 MB) | Low | **High** |
| `VMSPLIT_2G` | `0x80000000` | `0x80000000`–`0xEFFFFFFF` (1792 MB) | Low | Low |

`VMSPLIT_3G_OPT` deserves a warning of its own: its help text advertises "full
1G low memory", and the 1 GB window it produces ends at `0xC0000000` — the exact
address DDR1 starts at, so it captures none of it. Shrinking vmalloc does not
save the 3G split either; at the 16 MB floor the window reaches only
`0xBE000000`.

### Answers to the specific questions

**How much can be low memory?** Under the 3G split, DDR0's 512 MB and none of
DDR1. Under the 2G split, all 1 GB.

**Is `CONFIG_HIGHMEM` required?** Under the 3G split, yes — and without it the
kernel does not fail, it discards the bank:

```c
if (!IS_ENABLED(CONFIG_HIGHMEM) || cache_is_vipt_aliasing()) {
        if (memblock_end_of_DRAM() > arm_lowmem_limit) {
                pr_notice("Ignoring RAM at %pa-%pa\n", &memblock_limit, &end);
                pr_notice("Consider using a HIGHMEM enabled kernel.\n");
                memblock_remove(memblock_limit, end - memblock_limit);
```

Under `VMSPLIT_2G`, `HIGHMEM` is not required at all. That is the better answer
here: no `kmap` on the hot paths, kernel allocations free to use either bank,
and the only cost is user address space falling from 3 GB to 2 GB, which no
workload on a 1 GB machine will notice.

**Can any device not address the upper bank?** Nothing indicates so. ARM creates
`ZONE_DMA` only when a machine descriptor sets `dma_zone_size`; with it unset —
which is what a DT-only platform does — `setup_dma_zone()` leaves
`arm_dma_limit` at `0xFFFFFFFF`. On the hardware side the evidence is partial
but favourable: the vendor firmware runs 504 MB of MMZ video buffers at
`0xC0000000`, so the media engines master to that range. AHCI, `stmmac` and EHCI
have not been shown to, only because they never see those addresses under the
vendor configuration. Recorded as a thing to watch in Phase 4 rather than a
known problem.

**Does the hole need extra precautions?** No. The linear map is built per
memblock region, so the gap simply gets no page tables. Both banks and the
512 MB gap are 256 MB-aligned, exactly ARM's `SECTION_SIZE_BITS` of 28, so
`CONFIG_SPARSEMEM` fits the layout with no waste — and `ARCH_SPARSEMEM_ENABLE`
is `def_bool !ARCH_FOOTBRIDGE`, so it is available. `FLATMEM` also works, paying
a few MB of `struct page` for pages that do not exist.

**Should the "nothing more than" claim be qualified?** Yes, and it has been. No
new drivers are needed, which was the point being made, but the kernel
configuration is not incidental: `VMSPLIT_2G` to reach the bank and
`ARM_ATAG_DTB_COMPAT=n` (item 4) to keep the device tree's own memory node.

### Changed

| File | What |
|---|---|
| `doc/02-memory-map.md` | Summary claim qualified; new "Making both banks usable" section — the `vmalloc_limit` arithmetic and per-split table, the `Ignoring RAM` failure mode, `VMSPLIT_2G` vs `HIGHMEM`, the hole and memory model, and DMA |
| `doc/16-porting-roadmap.md` | Phase 2 now requires `VMSPLIT_2G` and tabulates the two failure modes by what the kernel reports; quick-reference row; risks-table entry |
| `doc/README.md` | Finding 1 notes both configuration traps |

---

## 6. Secondary CPU startup — **Confirmed**

The roadmap did not describe how CPU1 is released, and the review's reading of
`arch/arm/mach-godnet/platsmp.c` is accurate: it enables the SCU, writes the
physical address of `godnet_secondary_startup` to `0x20050134`, and uses a
holding pen. One detail in the observation needs correcting — the sequence uses
**no GIC software interrupt**. It does not need one.

### Evidence

**The bootloader half of the mechanism.** `SMP_COREX_START_ADDR_REG` is
commented `/* see bootloader */`, and the bootloader is where the interesting
part is. `arch/arm/cpu/godnet/start.S` in the SDK U-Boot, with
`SMP_COREX_START_ADDR_REG` = `SYS_CTRL_REG_BASE + 0x0134` and `COREX_RST_REG` =
`CRG_REG_BASE + 0x28` from `arch/arm/include/asm/arch-godnet/platform.h`:

```asm
        /* check cpuid */
        mrc     p15, 0, r0, c0, c0, 5
        and     r0, r0, #0xf
        cmp     r0, #0
        bne     core_x_flow
        b       main_core

core_x_flow:
        ldr     r3, =SMP_COREX_START_ADDR_REG
        /* checking if cpu0 run to kernel, if that, we go */
        ldr     r0, [r3]
        cmp     r0, #0
        beq     core_x_flow
core_x_jump:
        mov     pc, r0
```

`main_core` asserts bits 14–17 of `CRG + 0x28` to hold CPU1 in reset, copies
U-Boot's early code from `0x80800000` to physical address 0 (`/* slave run code
start position */`), clears `0x20050134`, and releases the reset. CPU1 then runs
the copied code from address 0 and parks in `core_x_flow`.

**The installed image matches the SDK source.** The same instruction sequence
is at file offset `0x114c`–`0x1170` of
`backups/2026-08-03/spi-nor/dhb-ax-spi-nor-cold-a.bin`, and the constant
`0x20050134` appears exactly once in the whole 2 MB image, in the literal pool
at `0x181c`.

**Three live reads on the running board tie it together:**

| Read | Value | Meaning |
|---|---|---|
| `0x20050134` | `0x8000ECCC` | Physical address inside the loaded kernel — what `platform_smp_prepare_cpus()` wrote |
| `0x20030028` | `0x00000023` | Bits 14–17 clear: CPU1 reset released by U-Boot |
| `0x0`–`0x1170` | U-Boot vectors, then `E59F36B4 E5930000 E3500000 0AFFFFFB E1A0F000` | The copied poll loop, still resident |
| `0x20300004` | `0x00000531` | SCU: two CPUs, both in coherency |

Physical address 0 is not an alias of DDR0 — `0x0` reads U-Boot's copied code
while `0x80000000` reads kernel data — so the holding code sits outside anything
a device tree describes as memory.

**No software interrupt.** The vendor's `boot_secondary()` keeps the Realview
comment about sending one but has no `smp_cross_call()`. Upstream Realview of
that vintage did call it; the vendor removed it because CPU1 is spinning on a
register rather than waiting in WFI.

### Answers to the four questions

**What does a current kernel need to reproduce this?** One 32-bit store. U-Boot
has already released CPU1 and left it polling, so a mainline port never touches
the reset register. The `smp_operations` is about thirty lines: ioremap the
system controller, `scu_enable(scu_a9_get_base())` in `.smp_prepare_cpus`, and
`writel_relaxed(__pa_symbol(secondary_startup), base + 0x134)` in
`.smp_boot_secondary`.

**SoC-specific `smp_operations`, a new `enable-method`, or something else?**
SoC-specific ops. Mainline ARM32 has no generic "write the entry point to a
register" method — `spin-table` is arm64-only. They can be bound through the
machine descriptor's `.smp` field or through `CPU_METHOD_OF_DECLARE` with an
`enable-method` in the `cpus` node. The closest existing template is
`hi3xxx_smp_ops` in `arch/arm/mach-hisi/platsmp.c`, which writes
`__pa_symbol(secondary_startup)` to `ctrl_base + ((cpu - 1) << 2)` — the same
shape — but it is not reusable unmodified, because it also calls
`hi3xxx_set_cpu()` to de-assert a reset and `arch_send_wakeup_ipi_mask()`,
neither of which applies here.

**CPU0 only, or both cores?** CPU0 only for the first boot, both later. This is
now explicit in Phase 1 and Phase 5 of the roadmap. Leaving CPU1 alone is free
and safe: U-Boot zeroed the register, so CPU1 stays in a loop that performs one
register read, a compare and a branch, at an address outside DDR.

**Which parts of the vendor sequence remain necessary?** The SCU enable and the
register write. The holding pen, `pen_release` and `headsmp.S` all go, provided
the register write moves from `.smp_prepare_cpus` into `.smp_boot_secondary` —
mainline populates `secondary_data` before calling the latter, so CPU1 can enter
`secondary_startup` directly. The pen exists only because the vendor releases
CPU1 before the stack and page tables are ready. `.cpu_die`/`.cpu_kill` should
be left unimplemented unless hotplug is wanted: once CPU1 leaves the poll loop
nothing puts it back, so re-onlining needs a WFI-and-wakeup-IPI path written
from scratch.

### Also found

The cores are Cortex-A9 **r3p0** (`CPU variant 0x3`, `CPU revision 0`). Two
errata that apply to an SMP build on r3p0 are not enabled in
`godnet_defconfig`: `ARM_ERRATA_764369` (`depends on CPU_V7 && SMP`, all
revisions) and `ARM_ERRATA_775420` (listed for r3p0; not offered by 3.0). The
vendor does set `ARM_ERRATA_754322`, which is correct for r3p0. `720789` and
`751472` do not apply — both are fixed by r3p0.

`SYS_CTRL + 0x138` reads `0x00000003` and is unidentified; nothing in the SDK
or the flash image references it.

### Changed

| File | What |
|---|---|
| `doc/01-soc-overview.md` | New "Secondary CPU startup" section — the eight-step firmware sequence, the U-Boot source and live evidence, the skip-SMP case, a worked `smp_operations` and device-tree fragment, and the errata table |
| `doc/03-boot-chain.md` | U-Boot section points at the CPU1 release path and names `arch/arm/cpu/godnet/` |
| `doc/16-porting-roadmap.md` | Phase 1 says to boot CPU0 only and why that is safe; Phase 5 gains a "second CPU" row; quick-reference row |
| `doc/17-register-dumps.md` | New `SYS_CTRL + 0x134` dump; note that only the strap bits of `+0x8C` are stable |
