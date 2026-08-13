# Boot Chain and U-Boot

## Sequence

```
Boot ROM  ->  U-Boot 2010.06 (SPI-NOR, 0x00000000)
          ->  reads boot-mode strap at SYS_CTRL+0x8C
          ->  runs bootcmd: nand read 0x82000000 0x0 0x500000; bootm 0x82000000
          ->  uImage (Linux 3.0.8) from NAND partition 'kernel'
          ->  root=/dev/mtdblock2, yaffs2
```

## U-Boot

| Property | Value |
|---|---|
| Version | `U-Boot 2010.06 (Nov 01 2012 - 11:21:03)` |
| Prompt | `hisilicon # ` |
| Location | SPI-NOR offset `0x00000000` |
| Environment | SPI-NOR offset `0x00080000`, 128 KB region, 629 bytes used |
| Boot delay | 1 second |
| Console | `ttyAMA0`, 115200 8N1 |
| Machine ID | `MACH_TYPE_GODNET` |
| Load address | `0x82000000` |

The SDK ships matching source: `osdrv/uboot/u-boot-2010.06.tgz`, board files in
`board/godnet/`, config in `include/configs/godnet.h`, CPU code in
`arch/arm/cpu/godnet/`. Its reset path is what releases the second CPU and
leaves it waiting for the kernel; see
[Secondary CPU startup](01-soc-overview.md#secondary-cpu-startup).

### Environment

Recovered from the SPI-NOR backup and confirmed with a live `printenv`:

```
bootdelay=1
baudrate=115200
ethaddr=00:00:23:34:45:66
ipaddr=192.168.1.10
serverip=192.168.1.1
netmask=255.255.255.0
bootfile="uImage"
bootcmd=nand read 0x82000000 0x0 0x500000;bootm 0x82000000
phyaddr0=2
phyaddr1=1
phyintfx=0
bootargs=mem=224M console=ttyAMA0,115200 root=/dev/mtdblock2 rootfstype=yaffs2 \
  mtdparts=hi_sfc:2M(boot);hinand:8M(kernel),16M(rootfs),64M(user),32M(hdr000000) pcieclkext=0
auversion0=513d503c
auversion1=5091ef22
auversion2=5192119e
auversion3=5091eb04
stdin=serial
stdout=serial
stderr=serial
verify=n
jpeg_addr=0xc0000000
jpeg_size=0x1156f
vobuf=0xc1000000
ver=U-Boot 2010.06 (Nov 01 2012 - 11:21:03)
```

`ethaddr` is a placeholder — the real MAC (`00:18:AE:3C:A2:49`) is set later by
the vendor application. See [06-ethernet.md](06-ethernet.md).

`phyaddr0=2` / `phyaddr1=1` are the MDIO addresses U-Boot expects. The kernel
finds the single populated PHY at address 1.

### Available commands

This is a **cut-down U-Boot**. Notable absences trip up the obvious workflow:

- **`boot` does not exist.**
- **`run` does not exist either** — so `run bootcmd` also fails. This is easy to
  miss, because the board's idle reset (below) makes it *look* like the command
  worked.
- **`bdinfo` does not exist.**
- **`flinfo` does not exist.**
- **`mtdparts` does not exist** — partitioning is passed to the kernel purely
  via the `mtdparts=` string in `bootargs`.

**To resume booting from the prompt, use `reset`.** It restarts the board,
which then autoboots normally. Alternatively, type the contents of `bootcmd`
by hand:

```
nand read 0x82000000 0x0 0x500000
bootm 0x82000000
```

Present and useful:

```
base bootm bootp cmp cp crc32 decjpg ext2load ext2ls fatinfo fatload fatls
getinfo go help loadb loady loop md mii mm mtest mw nand nboot nm ping
printenv rarpboot reset saveenv setenv setvobg sf startgx startvo stopgx
stopvo tftp usb usbboot version
```

- `md` / `mm` / `mw` — memory display and modify. **`md` is the safe way to read
  SoC registers**; see [17-register-dumps.md](17-register-dumps.md).
- `tftp`, `ping`, `bootp` — network boot is available. Essential for iterating
  on a new kernel without writing flash.
- `nand`, `sf` — NAND and SPI flash subsystems.
- `usb`, `usbboot`, `fatload`, `ext2load` — boot from USB storage. `usb` is the
  only block interface registered in this build, so these cannot reach the SATA
  disk; see [07-sata-storage.md](07-sata-storage.md#u-boot-cannot-read-the-sata-disk).
- `mii` — PHY register access. Note `mii device` reports no devices until the
  network is initialised.
- `getinfo` — takes one of four arguments (from `common/cmd_getinfo.c` in the
  SDK; only three are listed in its own usage text):

  | Argument | Output on this board |
  |---|---|
  | `getinfo bootmode` | `spi` — **the board boots from SPI-NOR** |
  | `getinfo version` | `version: 3.0.3` (bootloader version) |
  | `getinfo spi` | `Block:64KB Chip:2MB*1 / ID:0x01 0x40 0x15 / Name:"S25FL216K"` |
  | `getinfo nand` | `Block:128KB Chip:128MB*1 Page:2KB OOB:64B ECC:1bit` |

  `getinfo bootmode` is undocumented in the command's own help but is the
  cleanest way to read the boot-mode strap.
- `decjpg`, `setvobg`, `startvo`, `stopvo`, `startgx`, `stopgx` — vendor
  additions that render the boot splash. See below.

### Vendor additions

The bootloader brings up the video pipeline to display a JPEG splash screen
before Linux starts. From the boot log:

```
current auto update version : 201211011107
current used param offset 0xa0000
current do_check_flash_boot_param
close watch dog begin...............
test wdg 0 / test wdg 1 / dog_close
dev 0 / dev 2 / dev 3 set background color, opened
jpeg decoding ...
<<addr=0xc0000000, size=0x1156f, vobuf=0xc1000000>>
mmu_enable
<<imgwidth=480, imgheight=300, linebytes=960>>
decode success!!!!
vo hd 0 end / vo cvbs end
graphic layer 0, 2, 3 opened
USB:   scanning bus for devices... 1 USB Device(s) found
judge ddr init
user init finish.
```

Two things worth noting for a port:

1. **U-Boot disables the watchdog** (`dog_close`) before booting. If you replace
   the bootloader, make sure the watchdog is either disabled or serviced, or the
   board will reset mid-boot. See [10-rtc-watchdog-misc.md](10-rtc-watchdog-misc.md).
2. **There is an auto-update mechanism** reading parameters at SPI-NOR offset
   `0xA0000`. It runs on every boot (`do_auto_update` in `misc_init_r`) and can
   reflash the device from USB or disk. This is a hazard: an unexpected update
   image on an attached volume could overwrite your work. Understand it before
   leaving media attached.

## Interrupting the bootloader

Boot delay is 1 second, so key presses must already be arriving when the prompt
appears. A reliable approach is to spam a key continuously from before the
reset until `hisilicon #` appears.

Via the serial proxy on the Raspberry Pi:

```sh
ssh -t raspberrypi "picocom -b 115200 --omap crcrlf --logfile dvr.log /dev/serial0"
```

For scripted capture, `/home/niallsmart/uboot_capture.py` on the Pi does this
automatically — see [18-reference-assets.md](18-reference-assets.md) and
[raspberrypi-changes.md](raspberrypi-changes.md).

Two behaviours to plan around:

- **Do not spam keys continuously.** U-Boot's line buffer fills and it responds
  `## Command too long!` while echoing BEL, after which it ignores real
  commands. Send a short burst, then stop as soon as `hisilicon #` appears.
- **The prompt does not persist.** The board resets itself after roughly a
  minute sitting idle at the prompt, despite U-Boot having disabled the
  watchdog. Keep interactive sessions short and batch commands.

**Resume with `reset`.** Neither `boot` nor `run bootcmd` exists.

## Recommended porting workflow

Keep the vendor U-Boot initially. It already initialises DDR — which is the
hardest part to reproduce — and it supports TFTP.

```
setenv serverip 192.168.4.34        # the Raspberry Pi
setenv ipaddr 192.168.4.77
tftp 0x82000000 uImage-test
bootm 0x82000000
```

This loads and boots a test kernel entirely from RAM, writing nothing to flash.
A TFTP server is already running on the Pi at `192.168.4.34`, root `/srv/tftp`.
See [raspberrypi-changes.md](raspberrypi-changes.md).

Note `bootargs` is passed from the environment, so a test kernel can be given a
different command line with `setenv bootargs ...` without a permanent change
(avoid `saveenv`, which writes SPI-NOR). That applies to an ATAG boot; a
device-tree kernel configured as recommended below ignores it.

## Getting a device tree into a modern kernel

**This U-Boot cannot pass a DTB.** It has no FDT support at all, so the only
route is a device tree appended to the kernel image.

`include/configs/godnet.h` does not define `CONFIG_OF_LIBFDT`, and every path
that would handle a device tree is behind it — `boot_get_fdt()` is called only
from `common/cmd_bootm.c:303`, inside `#if defined(CONFIG_OF_LIBFDT)`. So
`bootm <kernel> - <dtb>` ignores its third argument. The handoff itself confirms
it: `arch/arm/lib/bootm.c:149` is

```c
theKernel (0, machid, bd->bi_boot_params);
```

`bi_boot_params` is set once, in `board/godnet/board.c:101`, to
`CFG_BOOT_PARAMS` = `0x80000100`. Nothing in this build ever puts a DTB address
in r2.

> **Two strings in the flash image look like FDT support and are not.** The SPI
> backup contains `Device Tree:` at `0x3BE96` and `Flat Device Tree` at
> `0x3C8B0`. The first is the header `usb tree` prints
> (`common/cmd_usb.c:578`); the second is a label in `uimage_type[]`
> (`common/image.c:140`), a table compiled in unconditionally. Neither implies
> libfdt, in the same way the `SATA` string does not imply SATA support — see
> [07-sata-storage.md](07-sata-storage.md#u-boot-cannot-read-the-sata-disk).

### Appended DTB

| Kernel option | Setting |
|---|---|
| `CONFIG_ARM_APPENDED_DTB` | **y** |
| `CONFIG_ARM_ATAG_DTB_COMPAT` | **n** — see below, this one matters |

`ARM_APPENDED_DTB` lives in the decompressor, so the image has to be a
**zImage**, not the uncompressed `Image` the vendor ships:

```
cat arch/arm/boot/zImage arch/arm/boot/dts/<board>.dtb > zImage-dtb
mkimage -A arm -O linux -T kernel -C none \
        -a 0x82000000 -e 0x82000000 \
        -n 'hi3531 test' -d zImage-dtb uImage-test
```

The load address has to sit clear of where the kernel decompresses to
(`0x80008000`); `0x82000000` is the address the vendor `bootcmd` already uses as
a staging area, so it is known good for `bootm`. Untested end to end — nothing
here has booted a mainline kernel.

### Leave `ARM_ATAG_DTB_COMPAT` off, or lose three quarters of the RAM

It looks like the helpful option — it is what the "old bootloader" case is for —
but on this board it silently reimposes the vendor's memory limit. It is the
only thing that makes the ATAG list matter, and switching it on hands the kernel
**two** independent clamps:

| Route | What U-Boot supplies | Effect |
|---|---|---|
| `ATAG_MEM` | One bank, `0x80000000`, 256 MB | `atags_to_fdt()` **overwrites** `/memory` `reg` wholesale with the ATAG values. DDR1 at `0xC0000000` disappears and DDR0 halves |
| `ATAG_CMDLINE` | `bootargs`, which begins `mem=224M` | `early_mem()` clamps to 224 MB whatever the device tree says |

Both figures are fabrications rather than measurements. `dram_init()` in
`board/godnet/board.c:130` assigns `bi_dram[0]` from the constants
`CFG_DDR_PHYS_OFFSET` and `CFG_DDR_SIZE` with no probe, `CONFIG_NR_DRAM_BANKS`
is 1, and `setup_memory_tags()` emits exactly that one bank. The board really
has 512 MB in each of two banks — see
[02-memory-map.md](02-memory-map.md#evidence-for-the-bank-sizes).

Choosing `CMDLINE_EXTEND` over `CMDLINE_FROM_BOOTLOADER` does not rescue this.
Those options only govern how `/chosen/bootargs` is merged; the `/memory`
overwrite happens unconditionally whenever any `ATAG_MEM` is present, and
`mem=224M` survives an extend just as it survives a replace.

With `ARM_ATAG_DTB_COMPAT` off, the decompressor replaces r2 with the appended
DTB's address and the ATAG list is never read. The device tree's `/memory` and
`/chosen/bootargs` are used exactly as written, and both banks are visible.

The cost is that `setenv bootargs` at the U-Boot prompt stops doing anything, so
the command line has to be edited in the device tree and the image rebuilt.
During bring-up that is a rebuild you are doing anyway.

### `machid` does not matter here

U-Boot passes `MACH_TYPE_GODNET` in r1 (`board/godnet/board.c:100`), and the
`machid` environment variable can override it. Neither is worth pursuing: once
r2 holds a DTB the kernel matches the machine on the root node's `compatible`
and ignores r1.

### Rebuilding U-Boot instead

Adding `CONFIG_OF_LIBFDT` and `CONFIG_CMD_FDT` would give the documented
`bootm <kernel> - <dtb>` handoff and a `fdt` command for editing the tree in
place, which is genuinely nicer to iterate against. It is not worth it here: the
new bootloader has to be written to SPI-NOR to be used, which the project
constraints forbid and which carries the only real brick risk on the board. The
appended DTB reaches the same place while writing nothing.

Only once a kernel boots reliably from TFTP should flashing be considered — and
at that point a hardware flash programmer for recovery becomes important, since
a bad SPI-NOR write bricks the board. Full flash backups exist in
`backups/2026-08-03/`.
