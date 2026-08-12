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

Supporting evidence:

- `fgpio.ko` logs `keybaord gpio init version : 201207021137` — a GPIO-driven
  keyboard/keypad driver.
- `/dev/boardgpio` exists.
- `ttyAMA1` had accumulated 11,268 interrupts while `ttyAMA2`/`ttyAMA3` had
  none, implying an active serial peer. UART1 is muxed onto GPIO12_7 (RXD) and
  GPIO13_0 (TXD) by the pinctrl script, so it is a real serial port rather than
  a pin left in GPIO mode — see [19-pinmux-map.md](19-pinmux-map.md).

> **The link between the AT89S52 and the SoC has not been traced, and the
> protocol is unknown.** The `ttyAMA1` hypothesis is inference from interrupt
> counts and the pinmux, not a confirmed connection. Establishing this would
> need either PCB tracing or capturing `ttyAMA1` traffic on the running device.

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

Alarm inputs and relay outputs on this class of hardware are ordinary GPIOs.
Neither the specific pins nor the input/output split has been established — the
`pinctrl_*.sh` scripts name peripheral functions but not board-level roles for
pins left in GPIO mode. `/dev/boardgpio` is the likely control interface.

For a repurposed server this is unclaimed, directly usable I/O: four relay
outputs are a genuinely useful thing to have on a home server, and reaching
them needs only `gpio-pl061` plus the pin numbers.

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
