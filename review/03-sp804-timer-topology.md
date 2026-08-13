# SP804 Timer Topology

## Observation

The timer documentation describes a clockevent at `0x20000000` and a clocksource
at `0x20010000`, sharing Linux IRQ 35. The roadmap recommends both bases on that
interrupt.

The SDK kernel appears to use both halves of the first SP804 block:

- `CFG_TIMER_VABASE` is `IO_ADDRESS(TIMER0_BASE)`.
- The clockevent uses the registers at offsets `0x00` onward.
- The clocksource reads `REG_TIMER1_VALUE`, offset `0x24`, from that same base.
- Both handlers are registered on `TIMER01_IRQ`.
- `CFG_TIMER2_VABASE`, corresponding to `0x20010000`, is defined but does not
  appear to be used by the vendor timer implementation.
- `TIMER23_IRQ` is separately defined as Linux IRQ 36.

Relevant SDK files are `arch/arm/mach-godnet/include/mach/time.h`,
`platform.h`, `irqs.h`, and `arch/arm/mach-godnet/core.c`.

## Questions to investigate

- Should the initial DT describe one SP804 at `0x20000000`, using its two
  internal timers and DT SPI 3?
- Is `0x20010000` a second SP804 block associated with DT SPI 4?
- What clock declarations and interrupt arrangement does the current SP804
  binding expect?

