# UART and Serial Console

## Hardware

Four ARM PL011 UARTs, all probed by the vendor kernel.

| Device | Base | IRQ | Revision | Role |
|---|---|---|---|---|
| `ttyAMA0` | `0x20080000` | 40 | PL011 rev2 | System console — U-Boot and Linux |
| `ttyAMA1` | `0x20090000` | 41 | PL011 rev2 | In use (11,268 interrupts at capture) |
| `ttyAMA2` | `0x200A0000` | 42 | PL011 rev2 | Registered, zero interrupts |
| `ttyAMA3` | `0x200B0000` | 43 | PL011 rev2 | Registered, no IRQ line active |

Console parameters: **115200 8N1**, set by `console=ttyAMA0,115200` on the
kernel command line and `baudrate=115200` in the U-Boot environment.

`ttyAMA1` is actively used — it had accumulated interrupts while `ttyAMA2` and
`ttyAMA3` had none. The most likely peer is the Atmel AT89S52 microcontroller
(U32) that handles the front panel; see
[10-rtc-watchdog-misc.md](10-rtc-watchdog-misc.md). This has not been
confirmed by tracing the PCB, and the protocol is unknown.

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
    interrupts = <0 40 4>;      /* verify GIC SPI numbering offset */
    clocks = <&uart_clk>, <&apb_clk>;
    clock-names = "uartclk", "apb_pclk";
};
```

> The IRQ numbers in `/proc/interrupts` (40–43) are the vendor kernel's Linux
> IRQ numbers. Mainline device trees express GIC SPIs with an offset (typically
> `SPI n` = Linux IRQ `n + 32` in the vendor numbering, but this depends on how
> `mach-godnet` maps them). **Verify the mapping against
> `arch/arm/mach-godnet/` before writing the device tree** rather than assuming
> the raw numbers transfer.

## Physical access

The console is exposed on a header and is proxied through a Raspberry Pi:

```sh
ssh -t raspberrypi "picocom -b 115200 --omap crcrlf --logfile dvr.log /dev/serial0"
```

The Pi is at `192.168.4.34`, its serial port is `/dev/serial0` → `ttyAMA0`, and
the user is in the `dialout` group. The exact pin header location on the DVR
board has not been documented — see
[16-porting-roadmap.md](16-porting-roadmap.md).

For scripted access, use the capture scripts described in
[18-reference-assets.md](18-reference-assets.md); only one process may hold
`/dev/serial0` at a time, so stop picocom before running them.

## Notes for the port

- The console is the only reliable channel during early bring-up. Get it up
  first; `earlycon` on the PL011 will pay for itself immediately.
- The vendor kernel prints via `console [ttyAMA0] enabled` before the colour
  dummy console is replaced, so early boot output is available.
- A getty runs on the console under the vendor firmware — stray input typed at
  the serial port lands at a login prompt, which is harmless but means the port
  cannot be assumed idle.
