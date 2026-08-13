# GPIO, Pin Multiplexing and I²C

This file covers the GPIO controllers and the peripherals attached to the
bit-banged I²C bus. The per-pin function tables live in
[19-pinmux-map.md](19-pinmux-map.md).

**Bit-banged I²C runs on SDA = GPIO12_4 and SCL = GPIO12_5.**

## GPIO controllers

The SoC provides **19 GPIO groups**, each a separate register block:

```
GPIO group n base = 0x20150000 + n * 0x10000     for n = 0 .. 18
```

| Group | Base | Group | Base |
|---|---|---|---|
| GPIO0 | `0x20150000` | GPIO10 | `0x201F0000` |
| GPIO1 | `0x20160000` | GPIO11 | `0x20200000` |
| GPIO2 | `0x20170000` | GPIO12 | `0x20210000` |
| GPIO3 | `0x20180000` | GPIO13 | `0x20220000` |
| GPIO4 | `0x20190000` | GPIO14 | `0x20230000` |
| GPIO5 | `0x201A0000` | GPIO15 | `0x20240000` |
| GPIO6 | `0x201B0000` | GPIO16 | `0x20250000` |
| GPIO7 | `0x201C0000` | GPIO17 | `0x20260000` |
| GPIO8 | `0x201D0000` | GPIO18 | `0x20270000` |
| GPIO9 | `0x201E0000` | | |

Each group appears to follow the ARM PL061 convention, based on the offsets the
SDK drivers use:

| Offset | Register |
|---|---|
| `+0x000`–`0x3FC` | Data registers, address-masked by bit (PL061 style) |
| `+0x400` | Direction (`GPIO_DIR`) |

The SDK's bit-banged I²C driver reads and writes a pin by addressing
`base + (1 << pin) * 4` — the PL061 masked-access scheme — which strongly
suggests these are PL061 or a close clone. Mainline `gpio-pl061` may work
directly, but this has **not been verified against the Hi3531 datasheet**.

## Pin multiplexing

Pin mux control (`IO_CONFIG`) is a flat array of 32-bit registers at
**`0x200F0000`**, one register per pin, holding a small integer function
selector.

The block runs from `+0x000` to `+0x258` — 151 registers. Every one of them is
described in [19-pinmux-map.md](19-pinmux-map.md), which pairs the chip
datasheet's function map with the values read from this board in U-Boot and
under the running kernel. That document, not this one, is where pinmux
questions get answered.

Note the Ethernet driver also claims `0x200F0000`–`0x200F01FF`, so it
reconfigures some of these pins at probe.

## I²C — there is no hardware I²C bus in use

**No I²C bus is registered with the kernel.** `/sys/bus/i2c/devices` does not
exist. Every I²C peripheral on this board is driven by **bit-banging GPIO** via
the vendor `gpioi2c.ko` module, which exposes `/dev/gpioi2c` and a set of
per-chip helper functions rather than a standard I²C adapter.

Helper routines present in the device's `gpioi2c.ko`:

| Routine family | Target chip |
|---|---|
| `gpio_sil9024_i2c_{read,write}` | Silicon Image SiI9024 HDMI transmitter |
| `gpio_ds1307_i2c_{read,write}` | DS1307-family RTC |
| `gpio_fpga_i2c_{read,write}` | Lattice ECP3 FPGA |
| `gpio_cx838_i2c_*` | Conexant CX838 (not populated) |
| `gpio_ANX7150_i2c_*` | Analogix ANX7150 (not populated) |
| `gpio_md240_i2c_*` | Unidentified (not populated) |

A separate `i2c.ko` module (version `201212270927`) is also loaded and reports
`current chip numbers = 0`.

### Known I²C addresses

| Address | Device | Evidence |
|---|---|---|
| `0x60` | Nextchip video decoder, chip ID `0x77` | `nvp1108 0x60 get chip id 77 init ok!` |
| `0x62`, `0x64`, `0x66` | Probed, no response | `warning: nvp1108 0x6x i2c_read err !!!` |

The decoder driver probes four addresses because the family supports up to four
cascaded decoders; only one is populated. See [11-video-input.md](11-video-input.md).

The RTC and FPGA addresses on this bus are not known.

### Bus pins

```
I2C_SDA -- GPIO12_4     (pinmux register 0x200f0198)
I2C_SCL -- GPIO12_5     (pinmux register 0x200f019c)
GPIO12 base address: 0x20210000
```

Both pinmux registers hold 0 — GPIO function, not the hardware I²C controller —
in the bootloader and under Linux alike, which is what a bit-banged driver
requires. This agrees with the SDK reference comment in
`mpp/extdrv/gpio_i2c_8b/gpio_i2c.c`. Supporting detail in
[19-pinmux-map.md](19-pinmux-map.md).

The bit-banged driver addresses a pin as `base + (1 << pin) * 4`, the PL061
masked-access scheme, with direction at `base + 0x400`.

> Note for anyone extending this: **byte-scanning stripped ARM modules for
> register base addresses does not work.** The constants sit in literal pools
> at unaligned file offsets and a naive scan returns mostly false positives.
> Use a real disassembler, or — as here — the vendor's own shell scripts.

## Other GPIO consumers

| Module | Purpose |
|---|---|
| `fgpio.ko` | Front panel keypad — "keybaord gpio init version : 201207021137" (vendor's typo) |
| `fpga_jtag.ko` | JTAG bit-banging to the Lattice FPGA — see [11-video-input.md](11-video-input.md) |
| `crypto_memory.ko` | Atmel CryptoMemory, likely a two-wire protocol on GPIO |
| `hi_ir.ko` | IR receiver (though the SoC has a dedicated IR block at `0x20070000`) |

## Mainline strategy

For a server build, most of this can simply be skipped — none of the bit-banged
peripherals are needed to boot and run.

If you do want them:

- Try `gpio-pl061` for the GPIO groups; verify against the datasheet.
- Use mainline `i2c-gpio` on GPIO12_4/GPIO12_5 to recreate the bit-banged bus,
  then attach `rtc-ds1307` for the RTC. This replaces the whole vendor
  `gpioi2c` stack with mainline code. (Do **not** plan on `sii902x` for HDMI —
  the transmitter is inside the SoC, not an external bridge. See
  [12-video-output.md](12-video-output.md).)
- The pinmux will need a small `pinctrl` driver, or can be left as-is if U-Boot
  configures it and the kernel does not disturb it. **Leaving the bootloader's
  pinmux untouched is the pragmatic choice for early bring-up** — with the
  caveat that U-Boot does not set everything. SPI, UART1, UART2, the audio SIO
  port and the HDMI pins are all configured by the Linux-side pinctrl script,
  not by the bootloader, so a kernel that skips that step will find those pins
  still in GPIO mode. The exact values are in
  [19-pinmux-map.md](19-pinmux-map.md).
