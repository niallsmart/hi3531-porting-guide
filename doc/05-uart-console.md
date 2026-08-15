# UART and Serial Console

## Hardware

Four ARM PL011 UARTs, all probed by the vendor kernel.

| Device | Base | IRQ | Revision | Role |
|---|---|---|---|---|
| `ttyAMA0` | `0x20080000` | 40 | PL011 rev2 | System console — U-Boot and Linux, 115200 |
| `ttyAMA1` | `0x20090000` | 41 | PL011 rev2 | Front-panel AT89S52 MCU, 9600 |
| `ttyAMA2` | `0x200A0000` | 42 | PL011 rev2 | **Rear-panel RS485**, 9600 |
| `ttyAMA3` | `0x200B0000` | 43 | PL011 rev2 | Registered, never opened, no IRQ line active |

Console parameters: **115200 8N1**, set by `console=ttyAMA0,115200` on the
kernel command line and `baudrate=115200` in the U-Boot environment.

The application process (`td3531`, PID 1033 at capture) holds both `ttyAMA1`
(fd 7) and `ttyAMA2` (fd 119) open, each at **9600 8N1**. `ttyAMA1` carries
steady traffic — over 52,000 interrupts — while `ttyAMA2` sits at zero, which
is what an idle RS485 bus with nothing attached looks like.

`ttyAMA1` is the front-panel MCU link. `libhi3531.so` references `/dev/ttyAMA1`
alongside its `keyboard_realmcu_*` functions, and `LocalDevice.cpp` in `td3531`
logs `Initial MCU fail` next to that path. The peer is the Atmel AT89S52 (U32),
and the 5-byte binary protocol it speaks is documented in
[20-front-panel-mcu.md](20-front-panel-mcu.md).

## RS485 (rear panel)

The rear panel carries an RS485 interface on the green screw-terminal block it
shares with the alarm I/O. It is **UART2 / `/dev/ttyAMA2`**.

| Property | Value |
|---|---|
| Device | `/dev/ttyAMA2` (major 204, minor 66) |
| Base / IRQ | `0x200A0000`, IRQ 42 (DT SPI 10) |
| Line settings | 9600 8N1 as configured by the vendor app |
| TXD pin | `GPIO0_1`, mux register `0x200f0004` = 2 |
| RXD pin | `GPIO2_4`, mux register `0x200f0050` = 2 |

The pinmux script `pinctrl_4HD_hi3531.sh` has an explicit `#UART2` block setting
both registers, and a live read of `0x200f0004` returns `0x00000002`, so the
pins are muxed to UART2 on the running device. Neither `UART2_RTS` nor
`UART2_CTS` is muxed — only TXD and RXD.

### What uses it

Two functions in the vendor application, both over the same bus:

- **PTZ camera control.** `td3531` carries protocol classes for Pelco-P,
  Visca and Minking, a `_ptz_serial_info` structure, and a "Serial Port"
  configuration page in the web UI.
- **An external RS485 keyboard.** `ExternalKeyboard.cpp` logs
  `The external 485 keyboard is %s`, with a `CKeyTWOEM485` handler and a
  `CProduct::External485KeyboardType()` accessor.

`LocalDevice.cpp` initialises the two ports in sequence: the MCU port first
(`Initial MCU fail`), then this one, logged as `485CS` followed by
`Initial serial fail` on error.

### Transceiver

`U34`, an 8-pin SOIC between the alarm relays and the rear terminal
block, marked `SP490EE` / `1249L` / `C23819`.

| Property | Value |
|---|---|
| Part | MaxLinear (originally Sipex/Exar) **SP490E**, 8-pin NSOIC |
| Type | **Full-duplex** RS-485/RS-422 transceiver |
| Supply | 5 V only |
| Max data rate | 10 Mbps |
| Enable pins | **None** |
| Pin compatible with | LTC490, SN75179 |

Pinout: 1 `VCC`, 2 `R` (receiver output), 3 `D` (driver input), 4 `GND`,
5 `Y` and 6 `Z` (driver outputs), 7 `B` and 8 `A` (receiver inputs). `D` takes
UART2 TXD and `R` feeds UART2 RXD.

**This settles the direction-control question: there isn't any.** The SP490E has
no driver-enable or receiver-enable pin — the tri-state enables are what
distinguish the 14-pin SP491E, which this board does not use. The driver is
permanently on. That is why no 485 direction GPIO appears in the pinmux scripts
or the loadable modules: nothing needs one.

`1249L` reads as a 2012 week-49 date code, consistent with the board's other
2013 markings. `C23819` is a lot code.

### Terminal block

Being full duplex, the part has a separate driver pair (`Y`/`Z`) and receiver
pair (`A`/`B`) — four signal wires, not two. The rear terminal block breaks out
both pairs, and the silkscreen labels them with the transceiver's own pin
names (`pcb/connector_block.png`):

| Terminal | Silkscreen | SP490E pin | Direction |
|---|---|---|---|
| `Y` | `P/Z` | 5 (driver out) | DVR → device |
| `Z` | `P/Z` | 6 (driver out) | DVR → device |
| `A` | `K/B` | 8 (receiver in) | device → DVR |
| `B` | `K/B` | 7 (receiver in) | device → DVR |

`P/Z` is PTZ and `K/B` is keyboard, matching the two firmware users exactly:
the DVR transmits camera commands on the driver pair and receives keypresses on
the receiver pair. Neither ground nor a termination pin is brought out.

The four RS485 terminals sit on the upper row of a 16-way block shared with the
alarm I/O — see [10-rtc-watchdog-misc.md](10-rtc-watchdog-misc.md#alarm-io) for
the alarm pins on the same connector.

### Porting

Straightforward: `ttyAMA2` is a plain PL011 node, identical to the console but
for the base and IRQ. Because the transceiver self-enables, **no RS485 support
is needed in software at all** — no `rs485` device-tree properties, no DE GPIO,
no `TIOCSRS485`. Open the port, set 9600 8N1, and read and write it as an
ordinary serial device.

## Clock source

The vendor U-Boot's `board_init()` clears the `UART_CKSEL_APB` bit in
`CRG + 0xE4` so the UARTs are clocked from the APB bus. A port must either
reproduce this or describe the resulting clock rate in the device tree.

## Mainline support

PL011 is fully supported by `drivers/tty/serial/amba-pl011.c`. This is one of
the easiest parts of the port:

```dts
uart0: serial@20080000 {
    compatible = "arm,pl011", "arm,primecell";
    reg = <0x20080000 0x1000>;
    interrupts = <0 8 4>;       /* SPI 8, level-high — vendor IRQ 40 */
    clocks = <&uart_clk>, <&apb_clk>;
    clock-names = "uartclk", "apb_pclk";
};
```

The vendor IRQs 40–43 map to device-tree SPIs 8–11. See the verified
[interrupt map](01-soc-overview.md#converting-to-device-tree-spi-numbers).

## Physical access

The console is exposed on a header and is proxied through a Raspberry Pi:

```sh
ssh -t raspberrypi "picocom -b 115200 --omap crcrlf --logfile dvr.log /dev/serial0"
```

The Pi is at `192.168.4.34`, its serial port is `/dev/serial0` → `ttyAMA0`, and
the user is in the `dialout` group.

### The J3 header

UART0 is brought out on **J3**, silkscreened `UART0`, immediately to the right
of the SoC heatsink. It ships unpopulated — this unit has header pins soldered
in.

It is a four-way header. Counting positions from the heatsink side:

| Position | Signal |
|---|---|
| 1 | `GND` |
| 2 | `TXD` |
| 3 | `RXD` |
| 4 | `VCC` |

Signal direction is from the DVR's point of view, and is inferred from the
working console wiring rather than from a silkscreen. Only the two data lines
and a ground are needed; leave `VCC` unconnected unless the far end has no
supply of its own.

Only one process may hold `/dev/serial0` at a time, so stop picocom before
running anything else against the port.

## Notes for the port

- The console is the only reliable channel during early bring-up. Get it up
  first; `earlycon` on the PL011 will pay for itself immediately.
- The vendor kernel prints via `console [ttyAMA0] enabled` before the colour
  dummy console is replaced, so early boot output is available.
- A getty runs on the console under the vendor firmware — stray input typed at
  the serial port lands at a login prompt, which is harmless but means the port
  cannot be assumed idle.
