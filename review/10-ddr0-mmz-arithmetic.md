# DDR0 MMZ Arithmetic

## Observation

`doc/02-memory-map.md` lists the DDR0 anonymous MMZ region as 288 MiB and totals
DDR0 as 224 + 288 + 5 = 517 MiB. The active command in
`rootfs/mtd/modules/load3531` uses:

```text
anonymous,0,0x8E000000,282M:jpeg,0,0x9fa00000,5M
```

The interval from `0x8e000000` through `0x9f9fffff` also appears to be 282 MiB.

## Questions to investigate

- Should the table say 282 MiB rather than 288 MiB?
- Is the corresponding accounting 224 + 282 + 5 = 511 MiB?
- Does that leave approximately 1 MiB at the end of DDR0 outside the documented
  Linux and MMZ regions?
- Is that final region intentionally reserved by firmware or simply unused?

