# Access to the Upper DRAM Bank

## Observation

The memory documentation says that describing both banks and omitting MMZ should
make approximately 1 GiB available. The second bank begins at physical
`0xc0000000`, while the vendor ARM kernel uses:

- `PHYS_OFFSET=0x80000000`
- `CONFIG_VMSPLIT_3G=y`
- `PAGE_OFFSET=0xc0000000`
- `CONFIG_HIGHMEM` disabled

## Questions to investigate

- How much of the two non-contiguous banks can be represented as ARM low memory?
- Is `CONFIG_HIGHMEM` required to use the second bank?
- Are any devices or DMA implementations unable to address the upper bank?
- Does the physical hole require additional memblock, zone, or DMA precautions
  beyond two DT `reg` entries?
- Should statements that memory recovery requires "nothing more" than changing
  the command line or DT be qualified?

