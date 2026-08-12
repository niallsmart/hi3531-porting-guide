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

### Key files in the DVR's own filesystem

The vendor `rootfs` carries better *board-level* documentation than the SDK
does, because the SDK's board material describes HiSilicon's demo board.

| File | Why |
|---|---|
| `rootfs/mtd/modules/pinctrl_*.sh` | **108 IO_CONFIG registers with their full function tables.** Closed the pinmux and I²C pin questions outright — see [19-pinmux-map.md](19-pinmux-map.md) |
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

The device runs BusyBox. `himm`, `himd` and `msh` are in `/bin`.

A scripted wrapper lives in the working scratchpad as `dvr.exp`:

```sh
./dvr.exp "uname -a" "cat /proc/mtd" "dmesg"
```

Two details make this fiddly and are worth knowing before rewriting it:

1. **Marker strings must not appear in the echoed command line.** The terminal
   echoes what is typed, so a naive `echo END` marker matches on the echo rather
   than the output. The script splits markers with shell concatenation
   (`"EN""D"`) so the literal never appears in the typed line.
2. **`himm` is interactive.** It prints the value then waits at `NewValue:` for
   a write. A script that sends a bare newline there may write to the register.
   Use U-Boot `md` for register reads instead — see
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

## Raspberry Pi

Configuration changes made to the Pi during this work are logged separately in
[raspberrypi-changes.md](raspberrypi-changes.md), as required by `PLAN.md`.
