# GPIO and Watchdog Confidence Levels

## Observation

`doc/09-gpio-pinmux-i2c.md` describes the GPIO blocks as PL061-like and says the
layout has not been checked against the datasheet. The datasheet appears to
specify masked data, direction, and interrupt registers using the PL061 layout.

`doc/10-rtc-watchdog-misc.md` calls the SoC watchdog "likely" SP805. The
datasheet gives an SP805-style layout at `0x20040000`, including LOAD, VALUE,
CONTROL, INTCLR, RIS, MIS, LOCK, and the standard `0x1acce551` unlock value.

## Questions to investigate

- Which GPIO characteristics are confirmed by the register manual?
- Do AMBA peripheral IDs or integration differences affect direct use of
  `gpio-pl061`?
- What are the GPIO interrupt assignments and trigger capabilities?
- What are the watchdog interrupt, clock selection, and AMBA identification?
- Does the current `arm,sp805` binding and driver apply directly, or is Hi3531
  glue still required?

