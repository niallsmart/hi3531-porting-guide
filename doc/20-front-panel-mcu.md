# Front Panel MCU and the SoC ↔ MCU Protocol

An **Atmel AT89S52** (U32), an 8051-family microcontroller with 8 KB of internal
flash, sits between the SoC and most of the board's low-speed I/O. It is not a
peripheral the SoC drives directly — it is a second processor running its own
firmware, reached over a serial link.

What is behind it: the front-panel buttons and LEDs, the buzzer (BZ1), the four
alarm relays, the four alarm inputs, spot-monitor channel select, and a watchdog
independent of the SoC's own. None of those are reachable from a mainline kernel
without speaking this protocol.

| Property | Value |
|---|---|
| Part | Atmel AT89S52 (U32), 8051 family, 8 KB flash |
| Link | `/dev/ttyAMA1` — UART1 at `0x20090000`, IRQ 41 |
| Line settings | 9600 8N1 |
| Pins | GPIO12_7 (RXD), GPIO13_0 (TXD) — see [19-pinmux-map.md](19-pinmux-map.md) |
| Frame | 5 bytes, binary, `0xA0` start byte |

## Why there is an MCU at all

The pattern is common on this class of hardware. The SoC's pins are consumed by
DDR, video, SATA and Ethernet, and a front panel wants a lot of them — a button
matrix, LEDs, a buzzer, four relay drivers. Scanning a key matrix and blinking
LEDs is constant low-value work that a general-purpose OS handles awkwardly. And
an MCU keeps running when Linux is not, which is what makes its watchdog useful:
a watchdog inside the thing it is meant to reset is worth less than one outside
it.

## Two back-ends, one interface

`libhi3531.so` is shared across a family of boards and carries two front-panel
implementations behind a single interface. The application calls
`ext_buzzer_set`, `ext_alarm_output_set` and so on, and the library routes to
whichever back-end the board is built for.

| Back-end | Transport | Representative symbols |
|---|---|---|
| `keyboard_realmcu_*` | Serial to a real MCU | `serial_read`, `serial_write`, `serial_set_speed`, `serial_set_parity`, `dev_open` |
| `keyboard_mcusim_*` | SoC GPIOs driven directly | `get_gpio`, `set_gpio`, `send_high`, `send_low`, `alarm_set_clk`, `alarm_set_shift`, `scan_key` |

Despite its name, `mcusim` emulates nothing. It is the implementation for
variants that omit the MCU and wire the panel straight to SoC pins, expanding
them through a shift register (`alarm_set_clk`, `alarm_set_shift`). The name
refers to presenting the same interface as an MCU-equipped board, not to
simulating the chip.

**This board uses the `realmcu` path.** `libhi3531.so` references
`/dev/ttyAMA1` alongside the `keyboard_realmcu_*` symbols; the application holds
that port open with steady traffic; and no process holds `/dev/boardgpio` or the
`fgpio` device, so the GPIO back-end is not in play. The only GPIO-style node
open at runtime is `/dev/gpioi2c`, which is the bit-banged I²C bus.

## What the MCU exposes

| Function | What it does |
|---|---|
| `keyboard_realmcu_get_key_value`, `get_event` | Front-panel keys |
| `keyboard_realmcu_buzzer_set` | Drives the buzzer (BZ1) |
| `keyboard_realmcu_alarm_output_set` | Drives the alarm relays |
| `keyboard_realmcu_alarm_status_get` | Reads the alarm inputs |
| `keyboard_realmcu_cs485_led_set` | Front-panel LEDs |
| `keyboard_realmcu_spot_channel` | Spot-monitor channel select |
| `keyboard_realmcu_wdg_set` | A watchdog, separate from the SoC's |
| `keyboard_realmcu_version_get` | MCU firmware version |

## Wire protocol

`libhi3531.so` is an unstripped ARM shared object with full symbols, so the
protocol comes straight out of the disassembly rather than needing a capture.

Frames are **binary and fixed at 5 bytes**, in both directions:

| Offset | Field |
|---|---|
| 0 | `0xA0` — start byte |
| 1 | Command |
| 2 | Data byte 1 |
| 3 | Data byte 2 (`0x00` for single-argument commands) |
| 4 | Checksum — sum of bytes 0–3, modulo 256 |

`keyboard_realmcu_serial_write(ctx, cmd, d1)` emits a frame with byte 3 zero;
`serial_write_ex(ctx, cmd, d1, d2)` fills both data bytes. Both compute the
checksum as `cmd + d1 + d2 - 0x60`, which is the plain additive sum because
`0xA0 ≡ -0x60 (mod 256)`. The receive path checks the same `0xA0` start byte and
the same 5-byte length, so the MCU answers in the same format.

Each write is followed by `usleep(2)`, then `usleep(20)` after releasing the
port mutex. The library waits for the MCU to echo the command back, and
**retries the frame up to four times** before returning failure.

### Command bytes

The command byte is the same value as the `keyboard_realmcu_operation()` opcode:

| Command | Meaning |
|---|---|
| 2 | Front-panel LEDs (5-bit field, mask `0x7C`) |
| 4 | Alarm relay outputs (4-bit field, mask `0x0F`) |
| 5 | Buzzer |
| 6 | Unused by any named wrapper |
| 7, 8 | Watchdog (`keyboard_realmcu_wdg_set` uses both) |
| 9 | Boolean flag |
| 10 | MCU firmware version query |
| 12 | LEDs, two data bytes |
| 13 | Two data bytes taken from the upper half of the argument |

Commands 2 and 4 both read-modify-write a cached state word in the context
(`+0x130` for LEDs, `+0x12c` for relays): bit 7 of the argument set means clear
the named bits, clear means set them. **The relay mask is `0x0F` — four bits,
one per relay**, which is an independent confirmation of the four-relay count
from the software side.

### Worked example

Sounding the buzzer is a single frame. `keyboard_realmcu_buzzer_set` calls
`keyboard_realmcu_operation(ctx, 5, value)`, which calls
`keyboard_realmcu_serial_write(ctx, 5, value)`, which writes:

```
A0 05 <value> 00 <checksum>
```

with the checksum `0xA0 + 0x05 + value` truncated to eight bits.

## What is not yet established

- **Verification on the wire.** All of the above is static analysis of
  `libhi3531.so`. Nothing has been checked against captured traffic.
- **The MCU→SoC event encoding.** The framing is shared, but how key presses and
  alarm-input changes are encoded inside it has not been worked through.
  `keyboard_realmcu_serial_read` is the function to read.
- **The MCU firmware.** It lives in the AT89S52's internal flash and is in no
  backup. Whether it can be read out depends on the part's lock bits.

## Capturing traffic

`ttyAMA1` is held open by the vendor application, so a second reader would steal
bytes from it and disturb the front panel. Either stop the application first, or
observe the line externally. A reboot recovers the board either way.

## Porting

For a plain server build this is all optional — the SoC boots and runs without
touching the MCU. It matters if you want the front panel, the buzzer, or the
four relay outputs, which are otherwise unreachable.

The work is a userspace serial protocol rather than a kernel driver: `ttyAMA1`
is an ordinary PL011 that mainline drives already, so nothing beyond a device
tree node is needed at the kernel level. See
[05-uart-console.md](05-uart-console.md).
