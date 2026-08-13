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
| 0 | Start byte — `0xA0` from the SoC, `0x0A` for MCU-originated frames |
| 1 | Command |
| 2 | Data byte 1 |
| 3 | Data byte 2 (`0x00` for single-argument commands) |
| 4 | Checksum — sum of bytes 0–3, modulo 256 |

**The start byte encodes direction.** `0xA0` marks a frame the SoC sent, and the
MCU acknowledges by echoing that frame back byte-for-byte, so `0xA0` appears in
both directions. Frames the MCU raises on its own — the idle heartbeat, and
presumably key events — start with `0x0A` instead.

`keyboard_realmcu_serial_write(ctx, cmd, d1)` emits a frame with byte 3 zero;
`serial_write_ex(ctx, cmd, d1, d2)` fills both data bytes. Both compute the
checksum as `cmd + d1 + d2 - 0x60`, which is the plain additive sum because
`0xA0 ≡ -0x60 (mod 256)`.

**The vendor sends every command twice, but the protocol does not require it.**
Nothing in the frame format expresses repetition, and the MCU acknowledges each
copy independently. The duplication is an artefact of how `serial_write` waits
for the ack, and a clean implementation should not copy it — see
[Why every command appears twice](#why-every-command-appears-twice).

Each write is followed by a 2 ms sleep, then a 20 ms sleep after releasing the
port mutex — `nanosleep({0, 2000000})` and `{0, 20000000}` as observed on the
wire. The library then checks whether the MCU has echoed the command back, and
**retries the frame up to four times** before returning failure.

### Why every command appears twice

The ack is not read by the thread that sends. `serial_write` writes the frame,
sleeps, then tests a context field that a separate reader thread fills in when it
recognises the echo. Critically, the sentinel that field is compared against is
set **once before the retry loop**, not per attempt:

```
mov  r2, #255
str  r2, [r4, #0x128]     ; ctx[0x128] = 0xFF, before any attempt
...
ldr  r1, [r4, #0x128]
cmp  r1, r6               ; == command?  -> success
```

Traced against a live watchdog frame, with the sleeps visible:

```
1148  04:08:11.073986  write(7, "\xa0\x07\x00\x00\xa7", 5) = 5
1036  04:08:11.107266  read(7,  "\xa0\x07\x00\x00\xa7", 5) = 5   echo, +33 ms
1148  04:08:11.112973  <... nanosleep resumed>                   ack check, +5.7 ms
1148  04:08:11.137277  write(7, "\xa0\x07\x00\x00\xa7", 5) = 5   retry
1036  04:08:11.208541  read(7,  "\xa0\x07\x00\x00\xa7", 5) = 5
```

The echo arrives before the check, but only barely — the reader thread has done
the raw `read()` and not yet parsed the frame and recorded the ack. So the first
attempt is judged failed and the loop retries. By the next pass the ack is
recorded and the loop exits.

**The repeat count varies**, which is the clearest evidence that this is the
retry loop rather than a deliberate double-send: two copies is typical, but a
command 2 frame was observed sent three times when the ack took longer to be
recorded. The ceiling is the loop's four attempts.

The vendor loses this race on nearly every frame, on a box that idles at a load
average around 20. It is a bug of the harmless kind: the commands are
idempotent, so the extra copies cost nothing but bus time.

**Do not reproduce it.** Send once and wait for the echo; allow ~50 ms.

### Command bytes

The command byte is the same value as the `keyboard_realmcu_operation()` opcode:

| Command | Meaning | Status |
|---|---|---|
| 2 | Front-panel LEDs (5-bit field, mask `0x7C`) | **Observed** |
| 4 | Alarm relay outputs (4-bit field, mask `0x0F`) | **Observed** |
| 5 | Buzzer (0 = off, 1 = on) | **Observed** |
| 6 | Unused by any named wrapper | From disassembly |
| 7 | Sent every 30 s unprompted; the watchdog kick | **Observed** |
| 8 | Watchdog — the other branch of `wdg_set` | From disassembly |
| 9 | Boolean flag | From disassembly |
| 10 | MCU firmware version query | From disassembly |
| 12 | LEDs, two data bytes | From disassembly |
| 13 | Two data bytes taken from the upper half of the argument | From disassembly |

Commands 2 and 4 both read-modify-write a cached state word in the context
(`+0x130` for LEDs, `+0x12c` for relays): bit 7 of the argument set means clear
the named bits, clear means set them. **The relay mask is `0x0F` — four bits,
one per relay**, which is an independent confirmation of the four-relay count
from the software side.

### Captured traffic

Traced with `strace` on the running application (see
[18-reference-assets.md](18-reference-assets.md#building-binaries-for-the-dvr)),
while an operator triggered alarm 1 from the web UI and then cancelled it. Every
frame below is verbatim from the capture.

Alarm asserted — relay 1 closes and the buzzer sounds:

```
04:00:56.333604 write(7, "\xa0\x04\x01\x00\xa5", 5)    relay bitmap = 0x01
04:00:56.393554 write(7, "\xa0\x04\x01\x00\xa5", 5)
04:00:56.463614 write(7, "\xa0\x05\x01\x00\xa6", 5)    buzzer on
04:00:56.534260 write(7, "\xa0\x05\x01\x00\xa6", 5)
```

Alarm cancelled, two seconds later:

```
04:00:58.454461 write(7, "\xa0\x04\x00\x00\xa4", 5)    relay bitmap = 0x00
04:00:58.513075 write(7, "\xa0\x04\x00\x00\xa4", 5)
04:00:58.584071 write(7, "\xa0\x05\x00\x00\xa5", 5)    buzzer off
04:00:58.644285 write(7, "\xa0\x05\x00\x00\xa5", 5)
```

Each is echoed back by the MCU roughly 30 ms later, identical bytes.

This confirms the two commands that matter for a port. **Command 4 carries a
relay bitmap** — relay 1 is bit 0, and clearing the byte releases every relay,
consistent with the `0x0F` four-bit mask in the library. **Command 5 is the
buzzer**, a plain 0/1. Byte 3 is `0x00` for both, and all four checksums are the
additive sum: `0xA0 + 0x04 + 0x01 = 0xA5`, `0xA0 + 0x05 + 0x01 = 0xA6`, and so
on.

### Front-panel LEDs

Starting the web UI and then quitting it, with the panel LEDs changing colour in
between:

```
04:24:52.836056 write(7, "\xa0\x02\x68\x00\x0a", 5)   web UI up   -> 0x68
04:24:56.390222 write(7, "\xa0\x02\x60\x00\x02", 5)   web UI gone -> 0x60
```

Command 2 carries the **whole LED bitmap**, not a delta — the library
read-modify-writes a cached state word at `ctx+0x130` and transmits the merged
result. Checksums hold: `0xA0 + 0x02 + 0x68 = 0x10A → 0x0A`, and
`0xA0 + 0x02 + 0x60 = 0x102 → 0x02`.

The full bitmap, established by driving one bit at a time and observing the
panel:

| Bit | Mask | LED |
|---|---|---|
| 2 | `0x04` | `BACKUP` |
| 3 | `0x08` | `NETWORK` |
| 4 | `0x10` | `PLAY` |
| 5 | `0x20` | `REC` |
| 6 | `0x40` | `HDD` |
| 7 | `0x80` | Not an LED — set/clear modifier on the argument |

So `0x60` is `REC` + `HDD`, the idle state of a recording DVR, and `0x68` adds
`NETWORK` while the web UI is connected. `BACKUP` and `PLAY` never appeared in
captured traffic because neither operation was running.

The panel carries **six** LEDs but the mask addresses only five. The sixth is
`POWER`, and it stays lit even when the bitmap is `0x00` — it is wired to the
supply rather than given a bit. Bit order does not follow physical order: left
to right the panel reads `REC` `HDD` `BACKUP` `NETWORK` `PLAY` `POWER`, which is
bits 5, 6, 2, 3, 4 and then the hardwired one.

`cs485_led_set` also has a branch issuing command 12 through `serial_write_ex`,
which carries two data bytes and could address up to sixteen. This board never
uses it — presumably for panel variants with more indicators.

### Driving the LEDs directly

Writing bitmaps to `/dev/ttyAMA1` at 9600 8N1 sets the panel immediately:

```
A0 02 00 00 A2      all off (POWER stays lit)
A0 02 08 00 AA      NETWORK only
A0 02 7C 00 1E      all five on
A0 02 60 00 02      REC + HDD, the normal idle state
```

Two cautions. The vendor application owns the port and will overwrite the panel
on its next update, so changes are transient while it runs. And writing
concurrently puts two writers on one bus, so the ack matching in `serial_write`
may see frames it did not send.

> **Watch the byte encoding.** busybox `printf` parses `\0240` as `\024`
> (`0x14`) followed by a literal `0`, silently producing a malformed 6-byte
> frame. Use `\xa0` hex escapes, or octal without the leading zero (`\240`).
> Verify with `printf '...' | od -An -tx1` before trusting a result, and check
> that the TX counter in `/proc/tty/driver/ttyAMA` advances by exactly 5 per
> frame — a mismatch there is the quickest sign the bytes are wrong.

### The MCU watchdog

The one thing the SoC sends without being asked is this, every 30 seconds:

```
04:00:45.542174 write(7, "\xa0\x07\x00\x00\xa7", 5)
04:00:45.614311 write(7, "\xa0\x07\x00\x00\xa7", 5)
```

Command 7 is `keyboard_realmcu_wdg_set(1)`. The function branches on its
argument and nothing else:

```
cmp  r1, #1
beq  ...            ; arg == 1  ->  operation(ctx, 7, 0)
                    ; otherwise ->  operation(ctx, 8, 0)
```

So 7 is arm-or-kick and 8 is disable. The vendor application issues 7 on a
30-second cycle for as long as it runs, and command 8 was never observed — it
never disables the watchdog.

**This is a second watchdog, separate from the SoC's own.** The SP805 at
`0x20040000` is switched off by U-Boot before Linux starts
([10-rtc-watchdog-misc.md](10-rtc-watchdog-misc.md#watchdog)). This one is not:
it lives in the AT89S52, survives anything happening on the SoC, and is kept
satisfied purely by the vendor application writing to a serial port.

> **A porting hazard.** A mainline kernel that boots without servicing this will
> stop the kicks. If the MCU responds by resetting the SoC — which is the usual
> reason for putting a watchdog in a companion microcontroller, and the reason
> its watchdog is worth more than one inside the part being watched — the board
> will reset some time after boot, looking exactly like an unexplained hang.
> Either send `A0 07 00 00 A7` every 30 seconds, or send `A0 08 00 00 A8` once
> to disable it.

Two things are not established: the MCU's timeout (only that it exceeds 30
seconds), and what it actually does when the timeout expires. Testing that means
stopping the vendor application and waiting to see whether the board resets.

And the MCU's idle heartbeat, twice a second:

```
read(7, "\x0a\x05\xff\xff\x0d", 5)
```

`0x0A + 0x05 + 0xFF + 0xFF = 0x20D`, low byte `0x0D` — the same checksum rule
with the MCU's own start byte. The `0xFF 0xFF` payload is stable while nothing
is happening, which is what an idle key-state report would look like, though
that has not been confirmed by pressing anything.

### Driving it from a port

To close relay 1 from your own code: open `/dev/ttyAMA1` at 9600 8N1 and write
`A0 04 01 00 A5`. To sound the buzzer, `A0 05 01 00 A6`. Release with the same
frames carrying `0x00`. Send each **once** and wait up to ~50 ms for the MCU to
echo the frame back — the vendor's double-send is a race it loses, not a
requirement.

Note that the vendor application holds the port open and kicks the watchdog on
it every 30 seconds, so a replacement userspace daemon has to take over that
duty too rather than merely sharing the port.

## Live observations

`/proc/tty/driver/ttyAMA` exposes per-port TX and RX byte counters, which gives
a non-invasive view of the link without opening the port or disturbing the
vendor application:

```
1: uart:PL011 rev2 mmio:0x20090000 irq:41 tx:5590 rx:137570
```

Sampling those counters once a second for 150 seconds, across an operator-
triggered alarm that closed relay 1 and sounded the buzzer:

| Observation | Value |
|---|---|
| TX at idle | Completely static — no traffic unless something happens |
| RX at idle | Exactly 10 bytes/s |
| Every TX delta | An exact multiple of 5 |
| Background TX | 2 frames every 30 s |
| The alarm event | 8 frames in ~2 s (4 + 4), with 3–4 extra RX frames alongside |

Two things are confirmed from live traffic rather than from code. **The 5-byte
frame size holds**: eight separate TX events over the run, 85 bytes total, every
delta a multiple of 5. And the **idle RX rate of 10 bytes/s is exactly two
frames per second**, so the MCU heartbeats at 2 Hz unprompted.

The 30-second background exchange is 2 frames each time, at 8 s, 37 s, 67 s,
96 s and 126 s. The later byte-level capture identified it as command 7 sent
twice — the watchdog kick. It is *not* commands 7 and 8, which is what I had
guessed from the two constants in `wdg_set`; those are separate branches.

The eight-frame alarm burst also resolves cleanly against the captured bytes:
four frames on assert (relay and buzzer, each sent twice) and four on release,
two seconds apart, matching the 86 s / 88 s split exactly.

The extra RX frames accompanying each TX burst are the command echoes.

## What is not yet established

- **The key-event encoding.** The idle heartbeat is `0A 05 FF FF 0D` at 2 Hz.
  What a keypress looks like has not been captured — nobody has pressed a
  front-panel button during a trace. This is the obvious next capture, and it
  is cheap now that the tooling is in place.
- **Commands 6, 8, 9, 10, 12, 13.** Named in the library but never seen on the
  wire, so their data encodings rest on static analysis alone.
- **Alarm inputs.** `alarm_status_get` exists, but no alarm input was asserted
  during a trace, so the poll or report frame for it is unidentified.
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
