# RTC, Watchdog, IR and Front Panel

Small peripherals, grouped. None is on the critical path for booting, but the
watchdog can actively break a port if ignored.

## Watchdog

| Property | Value |
|---|---|
| Register base | `0x20040000` |
| Driver | `wdt.ko`, vendor version `201206151658` |
| Kernel driver banner | `Hisilicon Watchdog Timer: 0.01 initialized` |
| Default margin | 60 seconds |
| `nowayout` | 0 |
| `nodeamon` | 0 |

**U-Boot explicitly disables the watchdog before booting Linux:**

```
close watch dog begin...............
test wdg 0
test wdg 1
dog_close
```

This matters. If you replace the bootloader, or if something re-enables the
watchdog and your kernel has no driver to pet it, the board will reset roughly
60 seconds into boot — a failure mode that looks like a kernel hang and is easy
to misdiagnose.

The IP is likely an ARM SP805, which mainline supports (`wdt-sp805`), but this
has not been confirmed against the datasheet.

## Real-time clock

There are **two** RTCs in play.

### On-chip RTC

The SoC has an RTC block at `0x20060000`. The vendor system does not appear to
use it as the system clock source.

### External RTC

| Property | Value |
|---|---|
| Modules | `hi_rtc.ko`, `ds1307.ko` |
| Device node | `/dev/ds1307`, char major 50 |
| Bus | Bit-banged GPIO I²C (`gpio_ds1307_i2c_read` / `_write`) |
| Driver banner | `2408  rtc Device Driver 201203081801 v1.0.0` |

The driver is named for the DS1307 but its banner says "2408", and the vendor
maintains a separate `hi_rtc.ko` as well. The actual part has not been
identified from the PCB photos. Candidates in this family are the DS1307,
DS1338, or an Intersil/Ricoh equivalent — several are register-compatible.

### Reading the boot-time debug line

At startup the vendor application prints:

```
get time is:38-08-18 4 19:06:38
```

**This is not a year of 38.** The line prints the RTC's **BCD registers as raw
decimal**, without converting. Decoded:

| Printed | Register byte | Actual value |
|---|---|---|
| `38` | `0x26` | year 26 → **2026** |
| `08` | `0x08` | month 8 → August |
| `18` | `0x12` | day **12** |
| `4` | — | weekday, Sunday = 1 → **Wednesday** |
| `19:06:38` | `0x13:0x06:0x26` | **13:06:26** |

Single-digit values such as the month are identical in both encodings, which is
why August reads correctly while the year and day do not. 2026-08-12 was indeed
a Wednesday, giving three independent confirmations of the decode.

**The RTC keeps the correct date, and its backup battery is fine.** Any tool
that consumes this driver's output must convert from BCD.

### The clock is fast by a fixed offset

There is a real fault, but it is not the one the debug line suggests. Measured
against a known-good host:

| | Value |
|---|---|
| DVR system time | 1774 s (29 m 34 s) **ahead** of true UTC |
| DVR uptime at measurement | 1584 s |

Because the offset exceeds the uptime, it cannot have accumulated as drift since
boot — the clock was already ~30 minutes fast when it was set. This is a
constant error inherited from the RTC chip, which was presumably never set
accurately.

### How system time actually gets set

Two things are worth knowing before porting:

- **The kernel never touches the RTC.** There is no `/sys/class/rtc` and no
  `/dev/rtc*` — only the vendor char device `/dev/ds1307` (major 50). The
  standard `hctosys` path does not exist on this system.
- **Userspace sets the clock.** The vendor application reads the chip through
  that char device and calls `settimeofday()` itself.
- **Nothing corrects it afterwards.** `/usr/sbin/ntpd` is present in the
  filesystem but no NTP process is running.

For a port, mainline `rtc-ds1307` over `i2c-gpio` should work once the SDA/SCL
pins are known (see [09-gpio-pinmux-i2c.md](09-gpio-pinmux-i2c.md)). That gives
a proper `/dev/rtc0` with standard `hctosys` behaviour and correct BCD handling
— strictly better than the vendor arrangement. Set the chip once to a correct
time, then run NTP.

## Infrared receiver

| Property | Value |
|---|---|
| Register base | `0x20070000` |
| IRQ | 48 (`Hi_IR`) |
| Device node | `/dev/Hi_IR` |
| Module | `hi_ir.ko`, version `201206151658`, built Jun 16 2012 |
| Driver banner | `HISI_IRDA-MF @Hi3520v100R001_C_0_2_0 2011-04-29` |

The SoC has a dedicated IR block. The banner references Hi3520, so the driver
is shared across the vendor's SoC family.

Zero interrupts had been taken at capture time — the remote had not been used.

Mainline has no Hi3531 IR driver. For a server this is dispensable; if wanted,
it would be a small `rc-core` driver, and the register layout would come from
the datasheet.

## Front panel and MCU

An **Atmel AT89S52** (U32) is on the board — an 8051-family microcontroller with
8 KB flash. On DVRs this part typically handles the front-panel buttons, LEDs,
the IR remote, and sometimes power sequencing, communicating with the SoC over
a UART or a simple GPIO protocol.

On this board it does exactly that, over `/dev/ttyAMA1` at 9600 8N1.

`libhi3531.so` is shared across a family of boards, and carries two front-panel
back-ends behind one interface. The application calls `ext_buzzer_set`,
`ext_alarm_output_set` and so on, and the library routes to whichever back-end
the board is built for. Despite its name, `mcusim` emulates nothing — it is the
implementation for variants that omit the MCU and wire the panel to SoC pins
instead, expanding them through a shift register (`alarm_set_clk`,
`alarm_set_shift`).

| Back-end | Transport | Representative symbols |
|---|---|---|
| `keyboard_realmcu_*` | Serial to a real MCU | `serial_read`, `serial_write`, `serial_set_speed`, `serial_set_parity`, `dev_open` |
| `keyboard_mcusim_*` | SoC GPIOs driven directly | `get_gpio`, `set_gpio`, `send_high`, `send_low`, `alarm_set_clk`, `alarm_set_shift`, `scan_key` |

**This board uses the `realmcu` path.** `libhi3531.so` references
`/dev/ttyAMA1` alongside the `keyboard_realmcu_*` symbols; the application holds
that port open with steady traffic; and no process holds `/dev/boardgpio` or the
`fgpio` device, so the GPIO back-end is not in play. The only GPIO-style node
open at runtime is `/dev/gpioi2c`, which is the bit-banged I²C bus.
UART1 is muxed onto GPIO12_7 (RXD) and GPIO13_0 (TXD) by the pinctrl script —
see [19-pinmux-map.md](19-pinmux-map.md).

The MCU does considerably more than read buttons. The `realmcu` HAL exposes:

| Function | What it does |
|---|---|
| `keyboard_realmcu_get_key_value`, `get_event` | Front-panel keys |
| `keyboard_realmcu_buzzer_set` | **Drives the buzzer (BZ1)** |
| `keyboard_realmcu_alarm_output_set` | **Drives the alarm relays** |
| `keyboard_realmcu_alarm_status_get` | **Reads the alarm inputs** |
| `keyboard_realmcu_cs485_led_set` | Front-panel LEDs |
| `keyboard_realmcu_spot_channel` | Spot-monitor channel select |
| `keyboard_realmcu_wdg_set` | A watchdog, separate from the SoC's |
| `keyboard_realmcu_version_get` | MCU firmware version |

### Wire protocol

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

Command bytes are the same values as the `keyboard_realmcu_operation()` opcodes:

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

> Recovered by static analysis, and not yet checked against captured traffic.
> The MCU→SoC direction shares the framing, but the encoding of key events in
> `keyboard_realmcu_serial_read` has not been worked through.

The MCU's own firmware is in its internal flash and is not part of any backup.
Whether it is readable depends on its lock bits.

For a server port, the front panel is optional. If you want the buttons, the
practical approach is to snoop `/dev/ttyAMA1` under the vendor firmware and
reimplement the protocol.

## Alarm I/O

The chassis label specifies **4 alarm channels**. Four "HUI KE" HK4100F-DC5V-SHG
relays (3 A at 250 V AC / 30 V DC) are on the top surface of the board,
alongside a buzzer (BZ1).

### Rear terminal block

A 16-way green screw-terminal block on the rear panel carries the alarm I/O and
the RS485 bus together (`pcb/connector_block.png`). Silkscreen, left to right:

| Row | Terminals |
|---|---|
| Upper | `COM3` `NO3` `COM4` `NO4` — **ALARM OUT**, then `Y` `Z` (`P/Z`) and `A` `B` (`K/B`) — **RS485** |
| Lower | `COM1` `NO1` `COM2` `NO2` — **ALARM OUT**, then `1` `2` `3` `4` — **ALARM IN** |

So the alarm side is **four relay outputs** as common/normally-open pairs
(`COM1`–`NO1` … `COM4`–`NO4`) and **four alarm inputs**, one relay per output
pair.

The RS485 terminals on the upper row belong to UART2, not to the alarm
subsystem; see [05-uart-console.md](05-uart-console.md#terminal-block).

### How the relays and inputs are reached

**Not by SoC GPIOs.** The alarm I/O goes through the AT89S52, over the same
`/dev/ttyAMA1` link as the front panel: `keyboard_realmcu_alarm_output_set`
drives the relays and `keyboard_realmcu_alarm_status_get` reads the inputs. The
GPIO back-end that would have bit-banged them directly
(`keyboard_mcusim_alarm_out`, `alarm_set_clk`, `alarm_set_shift`) is the
alternative implementation for boards without an MCU, and is not used here — no
process holds `/dev/boardgpio` at runtime. The relay bank sits directly beside
the MCU on the board, which fits.

The buzzer (BZ1) is on the same path, via `keyboard_realmcu_buzzer_set`.

This changes what it takes to use them from a mainline kernel. Four relay
outputs and four inputs are still attractive on a home server, but reaching them
is not a matter of `gpio-pl061` plus pin numbers — it means implementing the
AT89S52's serial protocol on `ttyAMA1`. See
[Front panel and MCU](#front-panel-and-mcu).

## Atmel CryptoMemory

| Property | Value |
|---|---|
| Module | `crypto_memory.ko` |
| Device node | `/dev/cryptomemory` |
| Driver banner | `TVT 35xx CryptoMemory Device Driver v1.0.0: Mar  9 2012` |

An Atmel CryptoMemory device (AT88SC family) — a small authenticated EEPROM.
The "TVT" in the banner identifies the ODM; see
[15-product-identity.md](15-product-identity.md).

This is almost certainly **copy protection / anti-cloning**: the vendor
application checks the chip is present and authentic before running. It may
also store per-unit data — possibly including the MAC address, which is not
in the U-Boot environment (see [06-ethernet.md](06-ethernet.md)).

Irrelevant to a Linux port except as a possible location for the MAC address
and serial number. Do not erase it.

## SD/MMC

A controller is present but unused:

| Property | Value |
|---|---|
| Platform device | `hi_mci.0` |
| Register base | `0x10020000`–`0x10020FFF` |
| IRQ | 67 |
| Interrupts taken | 0 |
| `/sys/class/mmc_host` | Present but empty |

The SDK includes `drivers/mmc/himciv100_godnet.c` for U-Boot. The IP is
Synopsys DesignWare, so mainline `dw_mmc` is a plausible fit.

Whether an SD socket is physically fitted is unknown — the underside of the
board has not been photographed. If one exists, it would be a convenient boot
device.
