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

Groups 0–17 have eight pins each; **GPIO18 has only six**, so there are 150
pins in total, not 152.

Linux's PL061 driver nevertheless hard-codes eight lines per bank, and the
PL061 binding has no property for reducing that width. GPIO18 therefore appears
as an eight-line gpiochip even though only offsets 0–5 exist in the hardware.
Do not assign offsets 6 or 7 to a consumer. A Hi3531-specific driver quirk would
be needed to hide them from the userspace ABI.

### The register layout is PL061

Confirmed against chapter 14.5.5 of the Hi3531 datasheet, which gives the
offsets and reset values in full:

| Offset | Register | PL061 equivalent |
|---|---|---|
| `+0x000`–`0x3FC` | `GPIO_DATA` — address-masked by `PADDR[9:2]` | `GPIODATA` |
| `+0x400` | `GPIO_DIR` — direction | `GPIODIR` |
| `+0x404` | `GPIO_IS` — level or edge | `GPIOIS` |
| `+0x408` | `GPIO_IBE` — both edges | `GPIOIBE` |
| `+0x40C` | `GPIO_IEV` — polarity | `GPIOIEV` |
| `+0x410` | `GPIO_IE` — mask | `GPIOIE` |
| `+0x414` | `GPIO_RIS` — raw status | `GPIORIS` |
| `+0x418` | `GPIO_MIS` — masked status | `GPIOMIS` |
| `+0x41C` | `GPIO_IC` — clear | `GPIOIC` |

Byte-for-byte the PL061 map, including the masked-data scheme the SDK's
bit-banged I²C driver relies on. Out of reset all pins are inputs, all
registers zero, and the trigger mode is edge-sensitive — again standard.

### The blocks have native AMBA identities

Target validation with Linux 6.18.42 shows that `gpio-pl061` works without an
identity override. All nineteen devices, from `20150000.gpio` through
`20270000.gpio`, bind to `pl061_gpio` and report:

```text
AMBA_ID=00041061
```

The live device tree used for that test contains no
`arm,primecell-periphid`. Individual 32-bit reads on GPIO0, GPIO12 and GPIO18
all returned the same valid identity:

```text
PID0..PID3 (+0xFE0..+0xFEC): 61 10 04 00
CID0..CID3 (+0xFF0..+0xFFC): 0D F0 05 B1
```

The PID encodes PL061 peripheral ID `0x00041061`, and the CID is the standard
PrimeCell signature. The earlier invalid values published here could not be
reproduced and are withdrawn; their cause is unknown. They must not be used as
evidence that the registers are absent.

Use normal AMBA discovery. An override is neither needed nor desirable because
it would conceal a genuine identity-read failure:

```dts
gpio12: gpio@20210000 {
    compatible = "arm,pl061", "arm,primecell";
    reg = <0x20210000 0x1000>;
    gpio-controller;
    #gpio-cells = <2>;
    clocks = <&apb_clk>;
    clock-names = "apb_pclk";
};
```

### Interrupts, and a complication above GPIO6

From the datasheet's interrupt table. Subtract 32 for the device-tree SPI
number:

| Groups | Linux IRQ | DT SPI |
|---|---|---|
| GPIO0 … GPIO6 | 105 … 111 | 73 … 79 |
| GPIO7 **and** GPIO8 | 112 | 80 |
| GPIO9 **and** GPIO10 | 113 | 81 |
| GPIO11 **and** GPIO12 | 114 | 82 |
| GPIO13 **and** GPIO14 | 115 | 83 |
| GPIO15 **and** GPIO16 | 116 | 84 |
| GPIO17 **and** GPIO18 | 117 | 85 |

Each block ORs its eight pins into one line, which is normal PL061. What is not
normal is that **the twelve blocks above GPIO6 share six GIC lines between
them.** `gpio-pl061` installs a *chained* parent handler
(`girq->parent_handler = pl061_irq_handler`), and a chained handler owns its
parent line outright — declaring two nodes on SPI 82 means the second
`irq_set_chained_handler_and_data()` displaces the first, and one of the two
blocks silently stops delivering interrupts.

Nothing in a server build needs GPIO interrupts, so the simple course is to
omit `interrupts` from the paired nodes and use them for output and polled
input. If a paired block genuinely needs interrupts, give the line to one node
of the pair only, or write a small demultiplexing irqchip.

GPIO0–GPIO6 have dedicated lines and are unaffected.

Trigger types are the full PL061 set — level or edge via `GPIO_IS`, polarity
via `GPIO_IEV`, both-edges via `GPIO_IBE` — so `#interrupt-cells = <2>` with
the usual `IRQ_TYPE_*` flags behaves as expected.

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
| `gpio_sil9024_i2c_{read,write}` | Silicon Image SiI9024 HDMI transmitter (not populated) |
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

- Use `gpio-pl061` for the GPIO groups and let the AMBA core read their valid
  native identities. No `arm,primecell-periphid` override is needed. Watch the
  shared interrupts above GPIO6, and do not use GPIO18 offsets 6 or 7.
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
