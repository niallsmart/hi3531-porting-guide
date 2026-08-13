# Reference Assets and Capture Methods

Where everything came from, and how to reproduce or extend it.

## Asset inventory

| Path | Contents |
|---|---|
| `Hi3531_V100R001C01SPC0D1/` | HiSilicon SDK, 2.5 GB |
| `rootfs/` | Export of the DVR filesystem, 49 MB |
| `backups/2026-08-03/` | SPI-NOR, NAND and filesystem images |
| `pcb/` | Photographs of the board and rear connectors |
| `datasheets/` | Local copies of datasheets for identified parts |
| `doc/` | This documentation |

### SDK layout

```
Hi3531_V100R001C01SPC0D1/
├── 00.hardware/
│   ├── chip/
│   │   ├── documents_en/Hi3531 H.264 Codec Processor Data Sheet.pdf   (17 MB — the register manual)
│   │   ├── documents_cn/Hi3531 H.264编解码处理器用户指南.pdf           (16 MB — Chinese equivalent)
│   │   ├── hi3531v100.ibs                                            (IBIS models)
│   │   └── HI3531V100.bsdl                                           (JTAG boundary scan)
│   └── board/                     ← HI3531DMO demo board, NOT this DVR
│       ├── hi3531dmo.pdf          (schematic)
│       ├── HI3531DMO.DSN          (OrCAD)
│       ├── HI3531DMO_VER_B_PCB.rar (Allegro)
│       └── HI3531DMO VER.B_2 BOM.txt
└── 01.software/board/Hi3531_SDK_V1.0.D.1/
    ├── osdrv/
    │   ├── kernel/linux-3.0.y/    (extracted; also linux-3.0.y.tgz)
    │   ├── uboot/u-boot-2010.06/  (extracted; also u-boot-2010.06.tgz)
    │   └── toolchain/             (arm-hisiv100nptl-linux, arm-hisiv200-linux)
    ├── mpp/                       (media libraries, prebuilt .ko, extdrv sources)
    ├── drv/                       (rtc, wtdg, cipher, hidmac, mmz, irda)
    └── package/                   (prebuilt images)
```

**The `00.hardware/board/` material is for HiSilicon's HI3531DMO reference
board, not this DVR.** The chip-level documents apply; the board-level ones do
not. This is the root cause of most gaps in this documentation.

### Unpacking the SDK archives

Several parts ship as tarballs. The kernel is already extracted; U-Boot is not:

```sh
cd Hi3531_V100R001C01SPC0D1/01.software/board/Hi3531_SDK_V1.0.D.1
tar xzf osdrv/uboot/u-boot-2010.06.tgz          # -> u-boot-2010.06/board/godnet/
```

Other archives: `package/{rootfs_uclibc,osdrv,drv,mpp}.tgz`,
`osdrv/busybox/busybox-1.16.1.tgz`, `osdrv/rootfs_scripts/rootfs.tgz`,
plus toolchain runtime libraries. The `.rar` files under `01.software/pc/` are
Windows-side decoder SDKs and are not relevant to the port.

Key files for a porter:

| File | Why |
|---|---|
| `u-boot-2010.06/arch/arm/include/asm/arch-godnet/platform.h` | The complete SoC register base map |
| `u-boot-2010.06/board/godnet/board.c` | Boot media strap, UART clock setup |
| `u-boot-2010.06/include/configs/godnet.h` | Build-time configuration |
| `u-boot-2010.06/common/cmd_getinfo.c` | The `getinfo` subcommands, incl. undocumented `bootmode` |
| `linux-3.0.y/arch/arm/mach-godnet/include/mach/irqs.h` | `GODNET_IRQ_START` — the GIC SPI offset |
| `linux-3.0.y/arch/arm/mach-godnet/` | Machine code, clocks, SMP, IO mapping |
| `linux-3.0.y/arch/arm/configs/godnet_defconfig` | Reference kernel config |
| `linux-3.0.y/drivers/net/stmmac/stmmac_main.c` | RGMII mode, CRG+0xEC bit layout |
| `mpp/extdrv/gpio_i2c_8b/gpio_i2c.c` | Bit-banged I²C reference implementation |
| `00.hardware/chip/documents_en/Hi3531 H.264 Codec Processor Data Sheet.pdf` | The register manual. Section 2.1.5 is the complete pinmux function map — the source for [19-pinmux-map.md](19-pinmux-map.md) |

### Key files in the DVR's own filesystem

The vendor `rootfs` carries better *board-level* documentation than the SDK
does, because the SDK's board material describes HiSilicon's demo board.

| File | Why |
|---|---|
| `rootfs/mtd/modules/pinctrl_*.sh` | Which IO_CONFIG value the vendor writes per board variant. **Use them for values only** — their function-name comments are unreliable, see [19-pinmux-map.md](19-pinmux-map.md) |
| `rootfs/mtd/modules/load3531` | The active module load script: MMZ zone layout, which drivers are and are not used |
| `rootfs/mtd/dep2.sh` | Selects the board variant (`load3531 -i 4hd`) |
| `rootfs/mtd/boot.sh` | `stmmac` module parameters, PHY addresses |
| `rootfs/etc/init.d/mac.dat` | The MAC address the vendor app actually reads |
| `rootfs/mtd/modules/extdrv/fpga_jtag.ko` | Contains the FPGA bitstream in `.rodata` |

**Grep the vendor filesystem's shell scripts before attempting to
reverse-engineer its binaries.** Questions that look like disassembly problems
are often answered by a commented-out `himm` line.

## SDK verification

`PLAN.md` asks that the SDK be confirmed as close to the device's original.
It is — same generation, slightly newer release:

| | Device | SDK |
|---|---|---|
| Kernel | 3.0.8, machine `godnet` | `linux-3.0.y` = 3.0.8, `godnet_defconfig` |
| U-Boot | 2010.06 (2012-11-01) | `u-boot-2010.06.tgz`, `board/godnet/` |
| Module vermagic | `3.0.8 SMP mod_unload ARMv7` | `3.0.8` |
| Toolchain | gcc 4.4.1 `Hisilicon_v100 … uclibc_0.9.32.1` | `arm-hisiv100nptl-linux` |
| MPP userspace | V1.0.7.3, built Aug 2012 | V1.0.D.1, built Apr 2015 |

The MPP release differs — the SDK is newer than the shipped firmware — so the
binaries are not identical (`hi3531_viu.ko` differs by MD5). This does not
affect its usefulness: the kernel and bootloader sources match the device's
generation, and those are what a port needs.

## Live access

### Telnet

```
host: 192.168.4.77
user: root
pass: 1001chin
```

The device runs BusyBox. `devmem` is available, as are `himm`, `himd`, `himd.l` and `himc` — all symlinks to `/bin/btools`.

A scripted wrapper lives in the working scratchpad as `dvr.exp`:

```sh
./dvr.exp "uname -a" "cat /proc/mtd" "dmesg"
```

Two details make this fiddly and are worth knowing before rewriting it:

1. **Marker strings must not appear in the echoed command line.** The terminal
   echoes what is typed, so a naive `echo END` marker matches on the echo rather
   than the output. The script splits markers with shell concatenation
   (`"EN""D"`) so the literal never appears in the typed line.
2. **Never let a script run `himm`.** It is a memory *modify* tool: with one
   argument it prints the value then waits at `NewValue:` for a write, so a
   bare newline can write to the register. Use `devmem` or `himd.l` for
   read-only access under Linux — see
   [17-register-dumps.md](17-register-dumps.md).

### Serial console

Proxied through the Raspberry Pi at `192.168.4.34`:

```sh
ssh -t raspberrypi "picocom -b 115200 --omap crcrlf --logfile dvr.log /dev/serial0"
```

For scripted capture, two Python scripts live on the Pi:

| Script | Purpose |
|---|---|
| `/home/niallsmart/uboot_capture.py` | Interrupts autoboot, runs `version`/`printenv`/`help` |
| `/home/niallsmart/uboot_capture2.py` | Interrupts autoboot, dumps pinmux/CRG/SYS_CTRL/DDR registers with `md` |
| `/home/niallsmart/uboot_capture3.py` | `getinfo` subcommands, `usb tree`, boot strap register |
| `/home/niallsmart/uboot_capture4.py` | DRAM aliasing test |

All four end with `run bootcmd`, **which does not work** — see the pitfalls
table. Fix them to send `reset` instead, or drive the prompt directly as below.

Logs: `dvr-uboot*.log` and the stdout copies `uboot_capture*.out`.

Usage pattern:

```sh
ssh raspberrypi "cd /home/niallsmart && nohup python3 -u uboot_capture2.py > uboot_capture2.out 2>&1 &"
# then, from the host:
./dvr.exp "sync; reboot"
```

The scripts send a key repeatedly from before the reset until `hisilicon #`
appears, because `bootdelay` is only 1 second. **This is the part most likely
to misbehave** — see the two spam-related entries in the pitfalls table.

If the board is *already* at the prompt, skip the interrupt logic entirely and
drive it directly, which is far more reliable:

```sh
ssh raspberrypi "python3 -c \"
import serial,time,sys
s=serial.Serial('/dev/serial0',115200,timeout=0.2)
s.write(b'\r\n'); time.sleep(0.5); s.read(4000)
out=b''
for c in [b'getinfo bootmode', b'md 0x200f0000 0x80']:
    s.write(c+b'\r\n'); time.sleep(0.7); out += s.read(8000)
sys.stdout.write(out.decode('ascii','replace'))
\""
```

**Only one process may hold `/dev/serial0`.** Stop picocom before running the
scripts.

### Pitfalls encountered

Recorded so they are not rediscovered:

| Symptom | Cause |
|---|---|
| `Unknown command 'boot'` | This U-Boot has no `boot`. |
| `Unknown command 'run'` | It has no `run` either, so `run bootcmd` fails too. **Use `reset`** to resume booting. |
| `Unknown command 'bdinfo'` / `'flinfo'` | Not built in. |
| `## Command too long!`, console returns only BEL (`0x07`) | Continuous key-spam overflowed U-Boot's line buffer. Send a short burst to interrupt autoboot, then stop. |
| Interrupt script never detects the prompt | If the script matches on the tail of its buffer, unbroken spam pushes `hisilicon #` out of the window. Stop spamming as soon as any output arrives. |
| Board resets on its own after ~1 min at the prompt | Observed repeatedly, despite U-Boot disabling the watchdog. Batch commands and keep sessions short. |
| `himm` appears to hang | It is waiting at `NewValue:` for a write. |
| `himd 0x200f0000` → `Bus error` | Use U-Boot `md` instead. |
| DVR unreachable for ~30–60 s after reboot | The network comes up late, after the vendor app starts. |

## Backups

See [04-flash-storage.md](04-flash-storage.md) for contents and geometry.
`backups/2026-08-03/MANIFEST.md` carries SHA-256 for every file; each layer was
read twice and compared, with a second copy on the Pi at
`/home/niallsmart/dhb_ax/backups/2026-08-03/`.

## PCB photographs

`pcb/` contains a 27 MB overview (`PCB.png`, also `PCB.heic`) plus labelled
close-ups and connector shots:

```
U1 Nanya.jpg                    U2 Nanya.jpeg
U16 TI PN521.jpeg               U19 nextchip NVP 1104B.jpeg
U32 Atmel AT89S52.jpeg          U67 RealTek RTL8211CL.jpeg
U88 JMB321.jpeg                 U91 Lattice LFE3-17EA.jpeg
connector_block.png             label.png
```

`connector_block.png` is the rear alarm/RS485 terminal block, decoded in
[05-uart-console.md](05-uart-console.md#terminal-block) and
[10-rtc-watchdog-misc.md](10-rtc-watchdog-misc.md#alarm-io).

**Top surface only.** See [15-product-identity.md](15-product-identity.md) for
the component inventory and what remains unlocated.

## Datasheets

`datasheets/` holds local copies for parts identified on the board:

```
NVP1104B Overview.pdf                  4-channel analog video decoder (U19)
SP490E SP491E RS-485 Transceiver.pdf   RS485 transceiver (U34)
SGM9119 Video Filter Driver.pdf        SD video filter driver (U17)
```

### The Hi3531 datasheet — what it does and does not document

`00.hardware/chip/documents_en/Hi3531 H.264 Codec Processor Data Sheet.pdf`,
Issue 09 (2015-02-09), 1794 pages. It is the SoC register manual, not a
board-design summary, and it is the first place to look for anything
chip-level.

Most chapters follow the pattern Overview → Features → Function Description →
Operating Mode → **Register Summary** → **Register Description**, the last two
giving a table of offsets and then a page per register with bit fields. **A
chapter that stops before those two sections is a block you cannot program from
this document.** That distinction is what separates the genuinely closed
hardware on this SoC from the merely undriven.

| Block | Chapter | Registers documented |
|---|---|---|
| Reset, clock (CRG) | 3.1, 3.2 | Yes |
| Interrupt system | 3.3 | No — it is the ARM GIC, described by reference |
| System controller | 3.4 | Yes |
| DMA controller | 3.5 | Yes |
| CIPHER | 3.6 | Yes |
| Timer | 3.7 | Yes |
| Watchdog | 3.8 | Yes |
| RTC | 3.9 | Yes |
| Power management and low-power modes | 3.10 | No — description only |
| Cortex-A9 and L2 cache | 3.11 | No — description only |
| DDR controller | 4.1 | Yes |
| NAND controller | 4.2 | Yes |
| SPI flash controller | 4.3 | Yes |
| GMAC / TOE | 5 | Yes |
| **VDH — H.264/MPEG decoder** | 6.1 | **No** |
| **JPGD — JPEG decoder** | 6.2 | **No** |
| **VEDU — H.264 encoder** | 7.2 | **No** |
| **JPGE — JPEG encoder** | 7.3 | **No** |
| **TDE — 2D graphics engine** | 8.1 | **No** |
| **VPSS — scaler and pre-processor** | 8.2 | **No** |
| **VCMP** | 8.3 | **No** |
| Motion detection | 9 | Yes |
| IVE — intelligent video engine | 10 | Yes |
| **VICAP — video capture** | 11.1 | **Yes** — around 85 pages |
| **VDP — video display** | 11.2 | **Yes** |
| **HDMI transmitter** | — | **No — it has no chapter at all** |
| Audio encoding (VOIE) | 12 | Yes |
| SIO — audio interfaces | 13.1 | Yes |
| I²C controller | 14.1 | Yes |
| SPI controller | 14.2 | Yes |
| UART | 14.3 | Yes |
| IR receiver | 14.4 | Yes |
| GPIO | 14.5 | Yes |
| USB 2.0 host | 14.6 | Yes |
| MMC/SD/SDIO | 14.7 | Yes |
| PCI Express | 14.8 | Yes — section 14.8.5 |
| SATA | 14.9 | Yes |

Base addresses are in
[01-soc-overview.md](01-soc-overview.md#register-base-map). Section 1 of the
datasheet carries its own address map, which covers more blocks than that
table does.

Two quirks when searching it: the section headings for the watchdog and the
NAND controller lost a space in typesetting and read `WatchDogRegister Summary`
and `NANDCRegister Summary`, so a search for "Register Summary" misses them.
And the datasheet's block name is not always the name the vendor SDK uses —
what MPP calls the VOU is the **VDP** here, and what `doc/11` calls the VIU is
the **VICAP**.

Extract the text with `pdftotext -layout`; the tables survive well enough to
parse.

## Building binaries for the DVR

The DVR runs uClibc 0.9.32.1 with gcc 4.4.1, EABI, linuxthreads — as reported by
`/proc/version`. The SDK's `arm-hisiv100-linux` toolchain matches that exactly.
(`arm-hisiv100nptl-linux` is the NPTL variant and is *not* what this firmware
was built with.)

The toolchain binaries are x86-64 Linux, so on an Apple Silicon Mac they need a
container:

```sh
tar xjf .../osdrv/toolchain/arm-hisiv100-linux/arm-hisiv100-linux.tar.bz2
docker run --rm --platform linux/amd64 -v "$PWD:/w" -w /w debian:bullseye-slim \
  /w/arm-hisiv100-linux/bin/arm-hisiv100-linux-uclibcgnueabi-gcc -o hello hello.c
```

A correct result reports `ELF 32-bit LSB executable, ARM, EABI5 ... interpreter
/lib/ld-uClibc.so.0`. If the interpreter says anything else, the binary will not
run on the DVR.

Autotools projects cross-compile with
`./configure --host=arm-hisiv100-linux-uclibcgnueabi`. `strace` 4.7 builds clean
this way and was the tool used to recover the MCU protocol in
[20-front-panel-mcu.md](20-front-panel-mcu.md).

### Getting binaries onto the DVR

> **`/tmp` is not tmpfs on this device.** `df /tmp` reports `/dev/root`, the
> 16 MB yaffs2 rootfs on **NAND**. Writing there writes flash. Use **`/nfsdir`**,
> which is a 107 MB tmpfs, or `/dev`, which is also tmpfs.

There is no `scp` on the DVR, but busybox provides `wget`, `tftp`, `nc` and
`base64`. Serving over HTTP from the workstation is the least friction:

```sh
python3 -m http.server 8099 --bind 0.0.0.0          # on the workstation
# then, on the DVR:
wget -q http://<workstation-ip>:8099/strace -O /nfsdir/strace && chmod +x /nfsdir/strace
```

Everything in `/nfsdir` vanishes on reboot, which is the desired property.

### Tracing the vendor application

The application is a 56-thread process. Attach to the **thread-group leader**,
not to the `XDVRStart.hisi` launcher that also matches `td3531` in `ps`:

```
 1028 root  ./XDVRStart.hisi ./td3531     <- launcher, single thread, traces nothing
 1032 root  ./td3531                      <- the real process, 56 threads, owns the fds
```

Find it by looking for the process that actually holds the device open rather
than by name. Then:

```sh
/nfsdir/strace -p 1032 -f -tt -xx -s 64 -e trace=read,write -e read=7 -e write=7 \
  -o /nfsdir/a.log
```

`-xx` is what makes the buffers readable as hex. Traffic on that port is sparse,
so the overhead is negligible and the application is unaffected — but note that
a short trace window can easily catch nothing, since the only unprompted traffic
is a 30-second watchdog frame.

There is no `strace`, `ftrace`, `kprobes` or `debugfs` on the stock device, but
`CONFIG_MODULES=y` and the SDK ships the matching kernel source, so a loadable
module is also possible if something needs kernel-side visibility.

## Raspberry Pi

Configuration changes made to the Pi during this work are logged separately in
[raspberrypi-changes.md](raspberrypi-changes.md), as required by `PLAN.md`.
