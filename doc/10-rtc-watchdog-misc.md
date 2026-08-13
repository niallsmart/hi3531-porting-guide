# RTC, Watchdog, IR and Alarm I/O

Small peripherals, grouped. None is on the critical path for booting, but the
watchdog can actively break a port if ignored.

The front-panel microcontroller, which also gates the alarm I/O described here,
has its own file: [20-front-panel-mcu.md](20-front-panel-mcu.md).

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

### There is a second watchdog

The SP805 above is not the only one. The AT89S52 front-panel microcontroller
holds its own, and unlike the SP805 **it is not disabled at boot** — the vendor
application keeps it satisfied by sending `A0 07 00 00 A7` over `/dev/ttyAMA1`
every 30 seconds, for as long as it runs.

Stopping those kicks while the watchdog is armed **hard-resets the SoC about 60
seconds later** — measured directly, with a full U-Boot banner on the console
confirming a reset rather than a hang.

**A port is not exposed to it.** The MCU shares the SoC's reset, so any reset
clears the watchdog, and command 7 is *arm-or-kick*, so a kernel that never
sends it never arms one. Neither a cold boot nor a reboot out of the vendor
firmware can carry an armed watchdog into your kernel, and a mainline kernel has
been run on this board without ever being reset.

The case that does bite is killing the vendor application while leaving the
kernel running — its `SIGTERM` handler exits cleanly but does not disarm the
watchdog, so the board resets a minute later. That is a hazard for poking at a
live board, not for porting. See
[20-front-panel-mcu.md](20-front-panel-mcu.md#the-mcu-watchdog) for the
measurements and the frames to send.

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

### Alarm inputs are dry contact, active low

Measured on the terminal block, and consistent from three directions:

| Property | Value |
|---|---|
| Open-circuit voltage | 5.2 V (pull-up) |
| Trigger | Short to ground |
| Resting state | Pulled high |
| Reported as | Active-low bitmap, bit 0 = input 1 |

There is **no common or ground terminal anywhere on the block** — all 16
positions are alarm-out, RS485 or alarm-in. Four terminals for four channels
with no shared return means each input must be referenced to the DVR's own
ground, which rules out isolated voltage-sensing inputs. The chassis label
independently specifies the inputs as "NO or NC", and a contact carries no
voltage of its own. The 5.2 V measurement completes the picture.

So a sensor triggers an input by closing a contact to chassis ground. Nothing
should apply a voltage to these pins; if they connect straight to the MCU, more
than Vcc+0.5 V risks damaging it. Whether a series resistor, clamp or opto sits
between the terminals and the MCU is unverified — that needs the traces around
`U32` followed, and only the top surface is photographed.

The NO/NC setting in the DVR's configuration does not change the electrical
behaviour, only the interpretation. An input declared NC rests shorted to
ground and alarms when the contact opens, which is also why a short cannot
harm the input: it is a supported steady state.

### How the relays and inputs are reached

**Not by SoC GPIOs.** The alarm I/O goes through the AT89S52 microcontroller,
over the serial link on `/dev/ttyAMA1`:
`keyboard_realmcu_alarm_output_set` drives the relays (command 4) and
`keyboard_realmcu_alarm_status_get` reads the inputs. The relay bank sits
directly beside the MCU on the board, which fits. The buzzer is on the same
path.

Reading the inputs is not a wire transaction. The MCU broadcasts their state
twice a second unprompted, and `alarm_status_get` returns what the library's
reader thread last cached — see
[the status broadcast](20-front-panel-mcu.md#the-mcu-status-broadcast).

That changes the cost of using them from a mainline kernel: not `gpio-pl061`
plus pin numbers, but a userspace implementation of the MCU protocol. Four relay
outputs remain an attractive thing to have on a home server — the protocol is
documented in
[20-front-panel-mcu.md](20-front-panel-mcu.md#wire-protocol), including the
`0x0F` relay mask that confirms the four-relay count from the software side.

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
