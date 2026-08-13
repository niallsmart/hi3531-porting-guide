# Secondary CPU Startup

## Observation

The roadmap includes CPU, GIC, timer, memory, and UART bring-up but does not
describe how CPU1 is released. The SDK's `arch/arm/mach-godnet/platsmp.c`:

- Enables the Cortex-A9 SCU.
- Writes the physical address of `godnet_secondary_startup` to system-controller
  register `0x20050134` (`SMP_COREX_START_ADDR_REG`).
- Uses a GIC software interrupt and holding-pen protocol.

## Questions to investigate

- What does a current kernel need to reproduce this sequence?
- Does Hi3531 need SoC-specific `smp_operations`, a new DT `enable-method`, or
  another mechanism?
- Is the initial port intended to use CPU0 only or to bring up both cores?
- Which parts of the vendor holding-pen sequence remain necessary under the
  current ARM SMP framework?

