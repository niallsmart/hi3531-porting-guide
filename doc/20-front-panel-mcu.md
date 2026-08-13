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
both directions. Frames the MCU raises on its own — the 2 Hz status broadcast
and key events — start with `0x0A` instead.

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
| 6 | Spot-monitor channel select; data byte 1 is the channel | From disassembly |
| 7 | Sent every 30 s unprompted; the watchdog kick | **Observed** |
| 8 | Watchdog — the other branch of `wdg_set` | From disassembly |
| 9 | Boolean flag | From disassembly |
| 10 | MCU firmware version query | **Observed** |
| 12 | LEDs, two data bytes | From disassembly |
| 13 | Two data bytes taken from the upper half of the argument | From disassembly |

The MCU originates its own frames, and **the command byte is scoped to the
direction** — it is not a shared opcode space. Read the start byte first:

| Command | Direction | Meaning |
|---|---|---|
| 1 | MCU→SoC | Key event — see [Key events](#key-events) |
| 5 | MCU→SoC | Status broadcast at 2 Hz — see [The MCU status broadcast](#the-mcu-status-broadcast) |
| 5 | SoC→MCU | Buzzer |
| 6 | SoC→MCU | Spot-monitor channel select |
| 6 | MCU→SoC | Firmware version reply — see [Firmware version](#firmware-version) |

Commands 5 and 6 are both reused this way, so two of the ten opcodes are
direction-dependent. Treat the start byte as part of the message identity.

**Spot channel select is probably dead code on this board.** The spot monitor
is VOU device 3, driven by the Hi3531's own integrated CVBS DAC
([12-video-output.md](12-video-output.md#cvbs-encoder)), so the SoC composes
that output itself and has no reason to ask the MCU to switch it. The command
has never appeared in any capture. `libhi3531.so` is shared across a family of
boards, and on variants that route the spot output through an external analog
mux the MCU would drive the select lines — which is the arrangement this
command is shaped for.

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

> **Not a porting hazard, on the evidence.** Command 7 is *arm-or-kick*. Once
> armed, stopping the kicks makes the MCU **hard-reset the SoC about 60 seconds
> later** — measured. But a port is not exposed to it: a kernel that never sends
> command 7 never arms the watchdog, and any SoC reset clears the MCU anyway, so
> the arm cannot carry over from a vendor boot. A mainline kernel has been run on
> this board without ever being reset. The sections below give the measurements;
> the one situation that does bite is
> [killing the vendor application without resetting](#the-one-way-to-get-bitten).

#### The timeout, measured

This was tested directly rather than assumed. The vendor application was frozen
with `SIGSTOP` immediately after a kick went out — verified by watching the
UART1 TX counter advance by exactly 10 bytes, the two frames of one kick — and
the serial console was logged from an external host.

| | |
|---|---|
| Last kick | t0 |
| First U-Boot output on the console | t0 + 61 s … t0 + 67 s |
| Timeout | **≈ 60 s**, two missed kicks |
| Board back up and running | t0 + ~2.5 min, normally, no intervention |

The bound is the console poll interval; U-Boot prints its banner a second or
two after reset, so the underlying timeout is essentially 60 seconds.

**It is a hard reset, not a hang.** The console produced a full
`U-Boot 2010.06` banner with DRAM and NAND re-initialisation, so the MCU
asserts SoC reset rather than merely signalling. Nothing else could have caused
it: `/dev/watchdog` was open by no process at the time, so the SP805 was not
armed, and `inittab` has no respawn entry for the application.

#### No reset loop

The same capture rules out a loop. After the reset the board took about 150
seconds to come back — far longer than the 60-second timeout — and the vendor
application cannot have resumed kicking until late in that boot. Yet the
console shows **exactly one** `U-Boot 2010.06` banner and **one** `Linux
version` line.

The reason is the next section: the reset it asserted also cleared the MCU.

#### A SoC reset clears the MCU

The MCU shares the SoC's reset. Two observations establish it.

**The LEDs.** Setting all five with `A0 02 7C 00 1E` and then resetting the SoC
blanks them at the instant of reset — the MCU drops its output state — and they
stay dark until the vendor application restarts and sets `0x60`. `POWER` stays
lit throughout, being wired to the supply rather than driven.

**A software reboot does not trigger the watchdog.** Issuing `reboot` from the
running vendor system produced exactly one U-Boot banner, and the board then ran
for five and a half minutes without another.

The timing makes that decisive. The countdown runs from the last kick, which
precedes the reboot by up to 30 seconds, so an armed watchdog would have fired
roughly 30–60 seconds after the reboot began — comfortably inside a boot that
takes 70 seconds or more to reach the network, and longer to reach the point
where the vendor application resumes kicking. A second banner should have been
unmissable. There was none.

So **the arm does not survive a reboot**, and there is no dual-boot hazard: a
developer who reboots from the vendor firmware into their own kernel gets a
disarmed MCU.

#### The one way to get bitten

Killing the vendor application *without* resetting the SoC. The application has
a clean `SIGTERM` handler — it prints `DVRService.cpp, 141 system exit!` and
exits deliberately — but **that path does not disable the watchdog**. Sending
`SIGTERM` and leaving the kernel running resets the board about 60 seconds
later.

That is a real trap for anyone exploring a live board, which is how it was
found here. It is not a trap for a port: a port either never arms the watchdog,
or reaches its own kernel through a reset that clears it.

### The MCU status broadcast

Twice a second, unprompted, the MCU sends:

```
read(7, "\x0a\x05\xff\xff\x0d", 5)
```

`0x0A + 0x05 + 0xFF + 0xFF = 0x20D`, low byte `0x0D` — the same checksum rule
with the MCU's own start byte.

**This is not a heartbeat. It is a status report, and the two data bytes are
live state.** Shorting alarm input 1 to ground changes byte 3 for exactly as
long as the input is held:

```
1036  05:10:57.446161 read(7, "\x0a\x05\xff\xfe\x0c", 5) = 5
1036  05:10:57.988225 read(7, "\x0a\x05\xff\xfe\x0c", 5) = 5
...   11 frames at ~540 ms, then back to \xff\xff
```

`0x0A + 0x05 + 0xFF + 0xFE = 0x20C → 0x0C`, so the checksum tracks the change.

| Byte | Field | Convention |
|---|---|---|
| 2 | Unknown — constant `0xFF` across every frame observed | — |
| 3 | Alarm inputs, bit 0 = input 1 | Active low, `0xFF` = all clear |

Byte 3 is active low, matching the electrical arrangement of the alarm inputs
(see [Alarm inputs](10-rtc-watchdog-misc.md#alarm-inputs-are-dry-contact-active-low)).
All four were shorted to ground in turn under capture:

| Input | Byte 3 | Frame |
|---|---|---|
| 1 | `0xFE` | `0a 05 ff fe 0c` |
| 2 | `0xFD` | `0a 05 ff fd 0b` |
| 3 | `0xFB` | `0a 05 ff fb 09` |
| 4 | `0xF7` | `0a 05 ff f7 05` |

A plain `0x0F` mask, one bit per input, the same width as the relay mask.

Byte 2 is not the key state — keys use a different frame entirely, see below.
It held `0xFF` across every frame in both captures, including during key
presses, so what it reports is undetermined. A second bank of inputs the board
does not populate is a plausible guess and nothing more.

Two consequences for a port.

**Nothing polls the MCU.** Over a four-minute capture spanning an alarm event,
the SoC sent nothing but the 30-second watchdog kick. `alarm_status_get` is not
a wire transaction — it reads state the reader thread has already cached from
this broadcast. A replacement implementation should do the same: parse `0x0A`
frames as they arrive and keep the last known state, rather than trying to
query.

**Command 5 means different things in each direction.** SoC→MCU it is the
buzzer; MCU→SoC it is this status frame. The command byte alone does not
identify a message — the start byte has to be read first.

The complete frame inventory from that capture, which is the whole vocabulary
of the link at idle:

| Count | Direction | Frame | Meaning |
|---|---|---|---|
| 423 | MCU→SoC | `0a 05 ff ff 0d` | Status, nothing asserted |
| 11 | MCU→SoC | `0a 05 ff fe 0c` | Status, alarm input 1 asserted |
| 17 | SoC→MCU | `a0 07 00 00 a7` | Watchdog kick |
| 16 | MCU→SoC | `a0 07 00 00 a7` | Echo of the kick |

### Key events

Front-panel keys do **not** appear in the status broadcast. They are a separate
MCU-originated frame, command 1, sent once per press:

```
05:14:29.260223  read(7, "\x0a\x01\x01\x01\x0d", 5)    front-panel "1"
05:14:33.935878  read(7, "\x0a\x01\x04\x01\x10", 5)    front-panel "2"
```

Checksums hold: `0x0A + 0x01 + 0x01 + 0x01 = 0x0D` and
`0x0A + 0x01 + 0x04 + 0x01 = 0x10`.

| Byte | Field |
|---|---|
| 1 | Command 1 — key event |
| 2 | Key code |
| 3 | `0x00` short press, `0x01` sustained press |

These are the frames behind `keyboard_realmcu_get_key_value` and `get_event`.

**One frame per press, and no release event.** The MCU sent nothing when a
button was let go, so this is an edge-triggered event rather than the level
report the alarm inputs get. A driver should treat it as a keypress
notification, not poll it for held state.

#### Key codes

All 23 front-panel buttons, pressed one at a time under capture:

| Code | Button | Code | Button |
|---|---|---|---|
| `0x01` | `1` | `0x0D` | `REC` |
| `0x02` | `4` | `0x0E` | `SEARCH` |
| `0x03` | `7` | `0x0F` | `PLAY/PAUSE` |
| `0x04` | `2` | `0x10` | `REW` |
| `0x05` | `5` | `0x11` | `FF` |
| `0x06` | `8` | `0x12` | `EXIT` |
| `0x07` | `3` | `0x13` | `OK` (D-pad centre) |
| `0x08` | `6` | `0x14` | `DOWN` |
| `0x09` | `9` | `0x15` | `UP` |
| `0x0A` | `MENU/+` | `0x16` | `LEFT` |
| `0x0B` | `BACKUP/-` | `0x17` | `RIGHT` |
| `0x0C` | `0/10+` | | |

**The codes are a matrix scan position, three rows by eight columns, scanned
column-major.** The numeric keypad proves it: labels `1 2 3` / `4 5 6` /
`7 8 9` are laid out in rows, but the codes run `1 4 7` down the first column,
`2 5 8` down the second, `3 6 9` down the third. Every group of three that
follows behaves the same way, so the whole panel is one matrix:

| Column | Codes | Buttons |
|---|---|---|
| 1 | `01`–`03` | `1` `4` `7` |
| 2 | `04`–`06` | `2` `5` `8` |
| 3 | `07`–`09` | `3` `6` `9` |
| 4 | `0A`–`0C` | `MENU/+` `BACKUP/-` `0/10+` |
| 5 | `0D`–`0F` | `REC` `SEARCH` `PLAY/PAUSE` |
| 6 | `10`–`12` | `REW` `FF` `EXIT` |
| 7 | `13`–`15` | `OK` `DOWN` `UP` |
| 8 | `16`–`17` | `LEFT` `RIGHT` |

So `code = (column − 1) × 3 + row`. The eighth column has only two buttons,
which predicts an unused position at `0x18`. That code was never seen and
nothing on the panel produces it.

Byte 3 distinguishes a tap from a hold. In a capture where each button was
tapped once, 22 of 23 events carried `0x00`; the one exception was a button
held slightly longer. In an earlier capture where two buttons were deliberately
held for about a second, both carried `0x01`. Same key codes, different byte 3
across the two runs, so it is not part of the code.

**There is no auto-repeat.** A button held for two seconds produced exactly one
frame, with byte 3 set. So `0x01` is emitted once, when the press crosses a
hold threshold — the MCU does not stream events while a key is down. The
threshold itself is under a second, since a slightly-longer-than-normal tap was
enough to set the bit.

For a driver this means a held key is indistinguishable from a tap in duration:
you get one event either way, and byte 3 is the only signal that the press was
long. Anything wanting key-repeat behaviour has to synthesise it, and nothing
reports the release.

The map above was validated blind: three buttons pressed without telling the
capture side which, decoded from the frames alone as `REC` (held), `MENU/+` and
`BACKUP/-`, and confirmed correct including the hold flag.

### Firmware version

Command 10 is the one true request/response exchange in the protocol —
everything else is either fire-and-forget with an echo, or unsolicited
broadcast. Sending `A0 0A 00 00 AA` to `/dev/ttyAMA1` produces:

```
05:25:30.016646  read(7, "\xa0\x0a\x00\x00\xaa", 5)    echo of the query
05:25:30.065846  read(7, "\x0a\x06\x97\x14\xbb", 5)    the reply, 49 ms later
```

So **the reply comes back as command 6**, in the MCU direction — the same
opcode that selects the spot-monitor channel when the SoC sends it.

The two data bytes are a packed date, not a version triple.
`keyboard_realmcu_version_get` stores the payload and unpacks it as:

```
orr  r6, r7, r6, lsl #8   ; value = d1 | (d2 << 8)
str  r6, [r5, #0x134]
...
asr  r2, r3, #9  ; and #127   -> field 1, 7 bits
asr  r1, r3, #5  ; and #15    -> field 2, 4 bits
and  r12, r3, #31             -> field 3, 5 bits
sprintf(out, "%02d.%02d.%02d", ...)
```

Widths of 7, 4 and 5 bits are the FAT/DOS packed-date layout — year, month,
day. With `d1 = 0x97` and `d2 = 0x14` the value is `0x1497`, giving **10.04.23**
— a firmware build date of 2010-04-23. The opposite byte order would yield
"75.08.20", which is not a date, and the store instruction settles it
independently.

To read the version from your own code, send the query and wait for a `0x0A`
frame with command 6; allow ~100 ms.

### Driving it from a port

To close relay 1 from your own code: open `/dev/ttyAMA1` at 9600 8N1 and write
`A0 04 01 00 A5`. To sound the buzzer, `A0 05 01 00 A6`. Release with the same
frames carrying `0x00`. Send each **once** and wait up to ~50 ms for the MCU to
echo the frame back — the vendor's double-send is a race it loses, not a
requirement.

Whether you also have to kick the watchdog depends on how you got there, and the
two cases are opposite:

| Situation | Watchdog state | What your code must do |
|---|---|---|
| Clean boot into your own kernel | **Disarmed.** Nothing sent command 7, and the preceding reset cleared the MCU | Nothing. Never send command 7 and it stays disarmed |
| Taking over UART1 on the running vendor system | **Armed**, and the vendor application has been kicking it every 30 s | Kick it yourself every 30 s from the moment you displace the vendor application, or reset the board within 60 s |

The second case is the one to watch on a live board: the vendor application
holds the port open, so taking it over means inheriting the kick, not merely
sharing the port. See [the MCU watchdog](#the-mcu-watchdog).

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

- **Byte 2 of the status broadcast.** Constant `0xFF` in every frame captured,
  including during key presses and all four alarm inputs asserted.
- **Commands 8, 9, 12, 13.** Named in the library but never seen on the wire, so
  their encodings rest on static analysis alone. Command 12 is the two-byte LED
  variant this board never uses; 9 and 13 have no named wrapper at all.
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
