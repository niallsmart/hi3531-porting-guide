# RTC, Watchdog, IR, Alarm I/O and Other Small Peripherals

Small peripherals, grouped. None is on the critical path for booting, and none
needs to be handled to get a kernel up — both watchdogs are inert unless
something arms them.

AMBA peripheral IDs identify the watchdog as an **SP805** and the on-chip RTC
as a **PL031**, both with mainline drivers.

The front-panel microcontroller, which also gates the alarm I/O described here,
has its own file: [20-front-panel-mcu.md](20-front-panel-mcu.md).

## Watchdog

| Property | Value |
|---|---|
| Register base | `0x20040000` |
| IP | **ARM SP805** — confirmed, see below |
| Mainline driver | `sp805-wdt`, compatible `arm,sp805` |
| Vendor driver | `wdt.ko`, version `201206151658` |
| Kernel driver banner | `Hisilicon Watchdog Timer: 0.01 initialized` |
| Vendor driver default margin | 60 seconds (a module parameter, not a hardware property) |
| `nowayout` | 0 |
| `nodeamon` | 0 |

### The IP is an SP805

Read from the AMBA peripheral ID registers at the U-Boot prompt:

```
20040fe0: 00000005 00000018 00000014 00000000
20040ff0: 0000000d 000000f0 00000005 000000b1
```

Part number is `0x05 | (0x8 << 8)` = **`0x805`**, designer `0x41` (ARM),
revision 1, with the standard PrimeCell signature `0D F0 05 B1` at `0xFF0`. The
same read against UART0 and Timer0 returned `0x011` (PL011) and `0x804`
(SP804), matching what [01-soc-overview.md](01-soc-overview.md) already
established, which validates the method.

Mainline `sp805-wdt` matches the register interface. A device-tree node with
`compatible = "arm,sp805"` and the APB clock is sufficient to bind and operate
the counter, but the SoC reset output needs additional integration described
below.

Like the GPIO blocks, the watchdog has a valid native PrimeCell identity and
needs no `arm,primecell-periphid` override. See
[09-gpio-pinmux-i2c.md](09-gpio-pinmux-i2c.md#the-blocks-have-native-amba-identities).

Chapter 3.8 of the datasheet gives the full register map — `WdogLoad`,
`WdogValue`, `WdogControl`, `WdogIntClr`, `WdogRIS`, `WdogMIS`, `WdogLock` with
the standard `0x1ACCE551` unlock key — matching SP805 exactly.

The watchdog IRQ is **34**, so `interrupts = <0 2 4>`.

### The counting clock is 3 MHz

Measured on the running device, since the timeout depends on it and no clock
tree documentation covers this branch. `WdogValue` was read twice five seconds
apart:

```
0x08E91F7A  ->  0x08039BB2      15,041,480 ticks in 5 s  =  3.008 MHz
```

So **3 MHz**, and that is what the node's `clocks` property has to advertise
for the driver's timeout arithmetic to come out right. It also explains
`WdogLoad`, which reads `0x0ABA9500` = 180,000,000 = exactly **60 seconds**.

Note that the 60-second figure in the table above is the vendor Linux driver's
module parameter — but the driver programs the hardware to match it, so on this
system the two agree. Remember SP805 semantics when reasoning about the
consequences: the first expiry raises an interrupt, and the reset only happens
if a *second* expiry arrives with the first interrupt still outstanding. From
the last kick that is 120 seconds, not 60.

### Mainline reset routing remains unresolved

Linux 6.18 validation confirmed that `sp805-wdt` binds and counts at the
expected 3 MHz rate. Leaving it unserviced raises the first-stage raw interrupt
(`WdogRIS = 1`), but the second expiry does not reset the SoC. The standard
SP805 register interface is therefore correct; the missing piece is Hi3531
reset routing outside the watchdog block.

The vendor driver clears bit 23 through a separate mapping while configuring
the watchdog, but the target register has not been identified. Until that
integration is understood, do not rely on this watchdog to recover a hung
mainline kernel.

### U-Boot disables it before booting Linux

```
close watch dog begin...............
test wdg 0
test wdg 1
dog_close
```

The practical consequence is that Linux inherits a disabled watchdog and
nothing has to service it. **The vendor kernel then turns it back on.** Under
the running firmware:

```
WdogControl  0x20040008 = 0x00000003    INTEN | RESEN — armed, will reset
WdogLoad     0x20040000 = 0x0ABA9500    60 s at 3 MHz
WdogLock     0x20040C00 = 0x00000001    locked; registers write-protected
```

`wdt.ko` arms it and `/dev/watchdog` exists, so something in the vendor
application is kicking it. This does not reach a port — U-Boot runs before your
kernel and disables it — but it does mean a *live* takeover of the running
system inherits an armed 60-second watchdog, the same trap as the MCU watchdog
below.

Whether the block is *enabled coming out of reset* is still not established.
U-Boot closing it explicitly suggests something upstream — the boot ROM, most
likely — turns it on, but a stock SP805 is disabled after reset, and the close
happens during U-Boot init before the prompt is reachable, so there is no way
to observe the prior state from here. If you replace the bootloader, disable it
yourself early rather than assuming either way.

### There is a second watchdog

The AT89S52 front-panel MCU has a separate watchdog. Reset clears it, and the
vendor application arms and kicks it over UART1. It is therefore irrelevant to
a clean boot into a port, but killing the vendor application without resetting
leaves it armed. Protocol and measurements are in
[20-front-panel-mcu.md](20-front-panel-mcu.md#the-mcu-watchdog).

## Real-time clock

There are **two** RTCs in play.

### On-chip RTC — an ARM PL031

| Property | Value |
|---|---|
| Register base | `0x20060000` |
| IP | **ARM PL031** |
| Mainline driver | `rtc-pl031`, compatible `arm,pl031` `arm,primecell` |
| Used by the vendor system | No |

From the peripheral ID registers:

```
20060fe0: 00000031 00000010 00000004 00000000
20060ff0: 0000000d 000000f0 00000005 000000b1
```

Part `0x031`, designer `0x41` (ARM), revision 0 — a PL031, but not one that
works with an unmodified `rtc-pl031` driver. Hi3531 adds a write lock at
`RTC + 0x20`. The SDK and datasheet require writing `0x1ACCE551` there before
setting `RTC_CR` or `RTC_LR`; mainline does not perform that unlock.

Target validation reproduced the distinction. With `CRG + 0xE4` bit 2 already
clear (reset deasserted), the counter remained at zero and a plain driver write
to `RTC_CR` did not stick. A volatile write of the unlock value followed by
`RTC_CR = 1` started the counter, which then advanced at exactly one tick per
second. This is an integration quirk, not a clock-gated or absent block.

A new `hisilicon,hi3531-rtc` compatible and small driver quirk could perform
the unlock and ensure reset is deasserted. Until then, do not add a plain PL031
node expecting it to provide a working RTC. **It also has no battery backup**,
so the external clock below remains the more useful system RTC.

The vendor system ignores it entirely.

### External RTC

| Property | Value |
|---|---|
| Modules | `hi_rtc.ko`, `ds1307.ko` |
| Device node | `/dev/ds1307`, char major 50 |
| Bus | Bit-banged GPIO I²C (`gpio_ds1307_i2c_read` / `_write`) |
| Driver banner | `2408  rtc Device Driver 201203081801 v1.0.0` |

The exact package has not been identified from the PCB photos, but its required
software compatibility is now established. The vendor module defaults to its
DS1307 mode: I²C address byte base `0xD0` (7-bit `0x68`) and the DS1307 register
order. Its separate PCF8563-style mode uses address `0x51`. On the
mainline port, `rtc-ds1307` successfully reads the eight time registers,
registers the device at `0-0068`, and supplies `rtc0`.

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

Confirmed a day later, across several reboots: the offset measured 1766–1769 s,
against 1774 s originally. It is stable to within a few seconds and survives
power-cycling, which is what an error stored in a battery-backed chip looks
like. It is not drift, and it is not a boot-time accident.

### How system time actually gets set

Two things are worth knowing before porting:

- **The kernel never touches either RTC.** There is no `/sys/class/rtc` and no
  `/dev/rtc*` — only the vendor char device `/dev/ds1307` (major 50). The
  standard `hctosys` path does not exist on this system, and the on-chip PL031
  is not driven at all.
- **Userspace sets the clock.** The vendor application reads the chip through
  that char device and calls `settimeofday()` itself.
- **Nothing corrects it afterwards.** `/usr/sbin/ntpd` is present in the
  filesystem but no NTP process is running.

For a port there are two options, but only the external one works with
unmodified mainline drivers.

**The on-chip PL031** needs a Hi3531-specific unlock/reset quirk before it can
be represented as a working RTC. It has no battery backup and needs setting at
every cold boot.

**The external chip** via mainline `rtc-ds1307` on `i2c-gpio`. The pins are
known: **SDA = GPIO12_4, SCL = GPIO12_5**, established from the vendor pinctrl
scripts and corroborated by an SDK reference comment — see
[19-pinmux-map.md](19-pinmux-map.md#the-i²c-pins), which carries a
ready-made `i2c-gpio` fragment. The exact manufacturer is not known, but
`dallas,ds1307` register compatibility at address `0x68` is validated on the
target.

The external device gives a proper `/dev/rtc0` with standard `hctosys`
behaviour and correct BCD handling. A Linux 6.18.42 boot registered it as
`rtc0`, advanced by five seconds during a five-second observation, and logged
that it set system time from the RTC. Set the chip once to a correct time, then
run NTP.

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

**Unlike the watchdog and the on-chip RTC, this is not a licensed ARM
PrimeCell.** The peripheral ID registers read all zeros:

```
20070fe0: 00000000 00000000 00000000 00000000
20070ff0: 00000000 00000000 00000000 00000000
```

No part number, no designer, no PrimeCell signature — a HiSilicon block with no
standard identity to match a mainline driver against.

Mainline has no Hi3531 IR driver. For a server this is dispensable; if wanted,
it would be a small `rc-core` driver, and the register layout would come from
the datasheet.

The input pin is `IR_IN`, mux register `0x200f01f8`, function 0 — its reset
state, and what this board reads. Function 1 would make it GPIO15_4.

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
directly beside the MCU on the board, which fits. The buzzer (BZ1) is on the
same path, as command 5 — `A0 05 01 00 A6` sounds it and `A0 05 00 00 A5`
silences it.

There is a second, misleading trail worth knowing about. `rootfs/mtd/dep2.sh`
writes `0x200f004c` under the comment `#set default buzzer gpio control`,
which reads as though GPIO2_3 drives the buzzer. The value it writes is 0,
which is `VIU1_CLK` — the write takes that pin out of GPIO mode. Nothing on
this board sounds the buzzer through a SoC GPIO.

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

**The pins clash with video input.** The chip datasheet puts the whole SDIO
interface on function 4 of the `0x200f00ec`–`0x110` run: `SDIO_CCLK_OUT`,
`SDIO_CARD_POWER_EN`, `SDIO_CARD_DETECT`, `SDIO_CWPR`, `SDIO_CCMD` and
`SDIO_CDATA0`–`CDATA3`. Those same registers are what the vendor sets to 0 for
the `VIU3` video-input bus. An SD card and the fourth video-input channel
cannot both be wired at once, which is a plausible reason no socket appears in
the vendor configuration. For a server build the trade is free — video input is
out of scope anyway. See [19-pinmux-map.md](19-pinmux-map.md).

Whether an SD socket is physically fitted is unknown — the underside of the
board has not been photographed. If one exists, it would be a convenient boot
device.
