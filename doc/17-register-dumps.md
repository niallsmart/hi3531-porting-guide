# Live Register Dumps

Raw register values read from the running board. These reflect **this DVR's
actual hardware configuration**, not the HiSilicon reference design, which makes
them the most direct evidence available for reconstructing the pinmux and clock
setup.

## How these were taken

Two read-only methods, and it matters which one a dump came from — the values
differ.

| Method | Where | Command |
|---|---|---|
| `md` | U-Boot prompt, before Linux runs | `hisilicon # md <address> <word_count>` |
| `devmem` | Running vendor kernel, over telnet | `devmem <address>` |
| `himd.l` | Running vendor kernel, over telnet | `himd.l <address> [words]` |

Each dump below says which it is. The blocks in this document are U-Boot
captures unless stated otherwise.

> **Never use `himm`.** It is HiSilicon's memory *modify* tool. Given one
> argument it reads the address, prints the value, then blocks on a `NewValue:`
> prompt and writes whatever is typed; given two it writes immediately without
> confirming. Leaving a session at that prompt on this board risks an
> unintended register write.

The safe Linux-side options are `devmem` and **`himd.l`**, both read-only.
Note the suffix: plain `himd` is the 8-bit variant and returns `Bus error` on
the pinmux block, which only accepts 32-bit accesses. `himd.l` does 32-bit
reads and dumps the block without complaint, printing offsets relative to the
address given. All of `himm`, `himd`, `himd.l` and `himc` are symlinks to one
multi-call binary, `/bin/btools`; source is in the SDK at
`osdrv/tools/board_tools/reg-tools-1.0.0/source/tools/`.

To retake or extend these dumps, see
[18-reference-assets.md](18-reference-assets.md) for the capture script, and
remember to resume the board with **`reset`** — neither `boot` nor `run` exists
in this U-Boot build, so `run bootcmd` fails silently-looking too.

## Pin multiplexing — IO_CONFIG at `0x200F0000`

One 32-bit register per pin, holding a function-select integer. This is the
board's shipped pin configuration.

```
200f0000: 00000000 00000000 00000000 00000000
200f0010: 00000000 00000000 00000000 00000000
200f0020: 00000000 00000000 00000000 00000000
200f0030: 00000000 00000000 00000000 00000000
200f0040: 00000000 00000000 00000000 00000001
200f0050: 00000000 00000000 00000000 00000000
200f0060: 00000000 00000000 00000000 00000000
200f0070: 00000000 00000000 00000000 00000000
200f0080: 00000000 00000000 00000000 00000000
200f0090: 00000000 00000000 00000000 00000000
200f00a0: 00000000 00000000 00000000 00000000
200f00b0: 00000000 00000000 00000000 00000000
200f00c0: 00000000 00000000 00000000 00000000
200f00d0: 00000000 00000000 00000000 00000000
200f00e0: 00000000 00000001 00000001 00000003
200f00f0: 00000003 00000003 00000003 00000003
200f0100: 00000003 00000003 00000003 00000003
200f0110: 00000003 00000003 00000003 00000003
200f0120: 00000003 00000003 00000003 00000003
200f0130: 00000003 00000003 00000000 00000000
200f0140: 00000000 00000000 00000000 00000000
200f0150: 00000000 00000000 00000000 00000000
200f0160: 00000000 00000000 00000000 00000000
200f0170: 00000000 00000000 00000000 00000000
200f0180: 00000000 00000000 00000000 00000000
200f0190: 00000000 00000000 00000000 00000000
200f01a0: 00000000 00000000 00000000 00000000
200f01b0: 00000000 00000002 00000002 00000001
200f01c0: 00000001 00000001 00000001 00000001
200f01d0: 00000001 00000001 00000001 00000001
200f01e0: 00000001 00000001 00000001 00000001
200f01f0: 00000002 00000002 00000000 00000000
```

Decoded against the chip datasheet — see
[19-pinmux-map.md](19-pinmux-map.md) for the full map and for the values the
running kernel has:

| Offset | Value | Selected function |
|---|---|---|
| `+0x000`–`+0x0E0` | 0 | VIU0 / VIU1 / VIU2 video input buses |
| `+0x04C` | 1 | `GPIO2_3` — but Linux writes 0, making it `VIU1_CLK`. Not the buzzer, despite the vendor comment; see [19-pinmux-map.md](19-pinmux-map.md#what-this-board-selects) |
| `+0x0E4`, `+0x0E8` | 1 | `VGA_HS`, `VGA_VS` |
| `+0x0EC`–`+0x134` | 3 | `VOU1120` — the BT.1120 video *output* bus, 19 pins |
| `+0x138`–`+0x1A8` | 0 | GPIO: the audio SIO ports, hardware SPI, hardware I²C and UART1 alternates are all deselected |
| `+0x1B0` | 0 | `RGMII0_TXCK` |
| `+0x1B4`, `+0x1B8` | 2 | `RGMII0_RXER`, `RGMII0_TXER` |
| `+0x1BC`–`+0x1EC` | 1 | **RGMII1 — RXDV, RXD3–RXD0, RXCK, TXEN, TXD3–TXD0, TXCK, TXCKOUT** |
| `+0x1F0`, `+0x1F4` | 2 | `RGMII1_RXER`, `RGMII1_TXER` |
| `+0x1F8`–`+0x1FC` | 0 | `IR_IN`, `NF_DQ0` |

The 13-pin run is Ethernet, confirmed from the datasheet rather than guessed:
`0x1BC` is the `RGMII1_RXDV` pin, and function 1 runs consecutively through the
receive bus, the transmit bus and both clocks. It is set here in U-Boot and left
alone by Linux.

The GPIO-mode block at `+0x138`–`+0x1A8` is what makes the bit-banged I²C work:
`0x200f0198` and `0x200f019c` — SDA and SCL — are left as GPIO rather than
routed to the hardware I²C controller, in U-Boot and afterwards.

**This dump is U-Boot's state, not the operating configuration.** Forty-four
of the 128 registers dumped here hold a different value under the running
kernel. The largest change is the 19-pin bus
at `+0x0EC`–`+0x134`: the pinctrl script rewrites it from 3 to 0, turning it
from the `VOU1120` output bus into the `VIU3` input bus, so all four video-input
channels are active. SPI, UART1, UART2, the audio SIO ports and the HDMI pins
are also set later. `stmmac` claims `0x200F0000`–`0x200F01FF` and reconfigures
pins at probe. The side-by-side comparison is in
[19-pinmux-map.md](19-pinmux-map.md#what-this-board-selects).

The dump above stops at `+0x1FC`; the block continues to `+0x258`.

## Clock and reset generator — CRG at `0x20030000`

```
20030000: 09000000 006c209b 14000000 006c2048
20030010: 14000000 006c2063 09000000 007c2087
20030020: 0b000000 007c207d 00000023 00000000
20030030: 000aaaa0 61e03fc0 02037cfe 0000004c
20030040: 00000001 00000001 00000001 00000001
20030050: 00000001 00000001 00000001 00000000
20030060: 00000001 00000001 00000001 00000001
20030070: 00000005 00000001 000001e6 00000001
20030080: 00000000 00000000 00000000 00000001
20030090: 00000001 00000001 00000001 00000001
200300a0: 00000001 00000000 00000000 00000000
200300b0: 00000000 00000000 00000080 00000000
200300c0: 00000006 00000002 00000000 0000000a
200300d0: 00000006 00000003 00000002 00000000
200300e0: 00000001 0000e060 0000001f 003f003f
200300f0: 003d0030 00000000 00000000 00000000
```

Known register meanings:

| Offset | Value | Meaning |
|---|---|---|
| `+0xE4` | `0x0000E060` | UART clock select — U-Boot clears `UART_CKSEL_APB` here |
| `+0xEC` | `0x003F003F` | Ethernet clock/reset — written by `stmmac` at probe, matching the boot message `Set system config register 0x200300ec with value 0x003f003f` |

The register pairs at `+0x00`–`+0x28` with values like `0x006C209B` and
`0x007C2087` have the shape of PLL configuration words (multiplier, divider,
fraction fields). The many `0x00000001` values from `+0x40` onward look like
individual clock-enable or reset-deassert bits.

The rest requires the datasheet's CRG chapter.

## System controller — SYS_CTRL at `0x20050000`

```
20050000: 00150124 00000002 00000000 00000000
20050010: 00000000 0fff8000 00000909 00003020
20050020: 00000000 00000000 00000000 ffffffff
20050030: 000007e0 00155500 5d75f000 00000000
20050040: 00000000 00000000 00000000 00000000
20050050: 00000000 00123456 00000000 00000000
20050060: 00000000 00123456 01123456 00000000
20050070: 00000000 00000000 00000000 00123456
```

`+0x00` is `REG_SC_CTRL` = `0x00150124`.

The repeated `0x00123456` values are HiSilicon's conventional lock/magic key
registers — writing that value unlocks an associated control register.

### Boot-mode strap — `SYS_CTRL + 0x8C`

`REG_SYSSTAT` sits beyond the block dumped above and was read separately:

```
2005008c: a0001d00
```

Bits [5:4] decode as 0 = SPI flash, 1 = DDR, 2/3 = NAND. Here
`(0xA0001D00 >> 4) & 0x3 = 0`, so the board boots from **SPI flash**, agreeing
with `getinfo bootmode`. See [01-soc-overview.md](01-soc-overview.md).

Only the strap bits are stable. The upper bits are live status and change
between reads — a later read of the same register returned `0xE8001D04`, with
`[5:4]` still `0`.

### Secondary CPU entry point — `SYS_CTRL + 0x134`

Also beyond the block above, read from the running vendor kernel:

```
20050130: 00000000 8000eccc 00000003 00000000
```

`+0x134` is the address CPU1 jumps to when it leaves U-Boot's poll loop;
`0x8000ECCC` is a physical address inside the loaded kernel. U-Boot zeroes this
register on every boot. `+0x138` reads `0x00000003` and is unidentified —
nothing in the SDK or the flash image references it. See
[Secondary CPU startup](01-soc-overview.md#secondary-cpu-startup).

## DDR controller 0 — `0x20110000`

```
20110000: 00000001 00000000 00000000 0000000f
20110010: 00000001 00061b50 00000010 83f10610
20110020: 00010785 00000000 00000000 00000132
20110030: 00000132 00000132 00000132 00000132
20110040: 80000000 80000000 80000000 80000000
20110050: c455120c ff527932 83510096 ffdff6f4
20110060: 000f2028 000f2028 000f2028 000f2028
20110070: c455120c ff527932 83510096 ffdff6f4
```

## DDR controller 1 — `0x20120000`

```
20120000: 00000000 00000000 00000000 0000000f
20120010: 00000001 00061b50 00000010 83f10610
20120020: 00010785 00000000 00000000 00000132
20120030: 00000132 00000132 00000132 00000132
20120040: c0000000 c0000000 c0000000 c0000000
20120050: c455120c ff527932 83510096 ffdff6f4
20120060: 000f2028 000f2028 000f2028 000f2028
20120070: c455120c ff527932 83510096 ffdff6f4
```

**This pair is the direct evidence for the two-controller memory layout.** The
registers at `+0x40`–`+0x4C` hold the base address each controller decodes:
`0x80000000` for DDRC0 and `0xC0000000` for DDRC1. Every other register is
identical between the two, indicating two identically configured banks of the
same DRAM. See [02-memory-map.md](02-memory-map.md).

The four repeated values at `+0x50` and `+0x70` (`0xC455120C`, `0xFF527932`,
`0x83510096`, `0xFFDFF6F4`) are per-lane DDR PHY training results. The
`0x000F2028` values at `+0x60` are likely timing parameters.

## Other useful U-Boot output

```
hisilicon # nand info
Device 0: NAND 128MiB 3,3V 8-bit, sector size 128 KiB

hisilicon # mii device
MII devices:
```

`mii device` is empty because U-Boot does not initialise the MDIO bus until a
network command runs. Run `ping` or `tftp` first if PHY register access is
needed.

`getinfo` requires an argument; invoking it bare only prints
`getinfo - print hardware information`. The valid arguments were not determined.
