# Memory Map and DRAM

## Summary

The board has **two DDR controllers** with roughly **1 GB of DRAM total**, but
the vendor firmware gives Linux only **224 MB**. The remaining ~790 MB is
carved out for HiSilicon's Media Memory Zone (MMZ) allocator, which serves the
video pipeline.

**Recovering that memory is the single largest win in repurposing this board as
a general-purpose server.** No new drivers are needed — MMZ simply is not
loaded. But it is not a one-line change either: the second bank sits at
`0xC0000000`, beyond what a default 3G/1G kernel can map as low memory, so the
kernel configuration has to be chosen for it. See
[making both banks usable](#making-both-banks-usable).

## Physical layout

| Range | Size | Owner |
|---|---|---|
| `0x80000000` – `0x8DFFFFFF` | 224 MB | Linux `System RAM` (set by `mem=224M`) |
| `0x8E000000` – `0x9F9FFFFF` | 282 MB | MMZ zone `anonymous` |
| `0x9FA00000` – `0x9FEFFFFF` | 5 MB | MMZ zone `jpeg` |
| `0x9FF00000` – `0x9FFFFFFF` | 1 MB | DDR0, claimed by nothing |
| `0xA0000000` – `0xBFFFFFFF` | — | Not mapped (hole between the two controllers) |
| `0xC0000000` – `0xDF7FFFFF` | 504 MB | MMZ zone `ddr1` |
| `0xDF800000` – `0xDFFFFFFF` | 8 MB | DDR1, claimed by nothing |

DDR controller 0 backs `0x80000000`; DDR controller 1 backs `0xC0000000`. This
was confirmed by a live U-Boot register read: DDRC1 at `0x20120000` holds
`0xC0000000` in four consecutive registers at offset `+0x40`, while the
equivalent DDRC0 registers are zero (base `0x80000000` being the default).

Adding up the claimed regions gives 224 + 282 + 5 = 511 MB on DDR0 and 504 MB
on DDR1 — consistent with **512 MB per controller, 1 GB total**, with 1 MB
spare at the top of DDR0 and 8 MB at the top of DDR1.

Neither tail is reserved for anything. Nothing in `/proc/iomem` covers them,
and reading `0x9FF00000` and `0x9FFFFFF0` through `/dev/mem` returns the same
uninitialised-DRAM pattern as unallocated addresses inside the MMZ zones. They
are simply left over: HiSilicon's own reference `load3531` uses
`anonymous,0,0x84000000,447M:ddr1,0,0xC0000000,511M`, which stops exactly 1 MB
short of the top of each bank, and every MMZ variant on this board ends the
`jpeg` zone at `0x9FF00000`. A port that describes the full 512 MB in the
device tree gets that memory back along with everything else.

## What the firmware reports

Three different numbers appear, all correct in their own context:

| Reported | Where | Meaning |
|---|---|---|
| `DRAM: 256 MiB` | U-Boot banner | Compile-time constant `CFG_DDR_SIZE` in `include/configs/godnet.h`. Not a probe. Wrong for this board. |
| `Memory: 224MB` | Kernel | Result of the `mem=224M` command-line clamp. |
| `total size=809984KB (791MB)` | `/proc/media-mem` | Sum of the three MMZ zones. |

U-Boot's `CONFIG_NR_DRAM_BANKS` is 1 and `CFG_DDR_SIZE` is `256*1024*1024` in
the *SDK* source. The device's U-Boot was built from a modified tree — it
reports the same 256 MiB, so the vendor apparently never corrected it. Treat
the U-Boot DRAM figure as meaningless.

## MMZ (Media Memory Zone)

MMZ is HiSilicon's carve-out physical allocator, implemented in `mmz.ko` and
exposed at `/proc/media-mem`. It is not CMA and has no mainline equivalent.

At runtime the vendor application had allocated ~358 MB of the 791 MB across
139 blocks with names like `vb` (video buffers), `Vdec0_Vdh`, `h264e0_Str`,
`hifb_layer0`, `TDE_MEMPOOL_MMB`.

For a server port, MMZ is simply not loaded and the whole address space becomes
available to Linux.

## Reclaiming the memory

The vendor command line is:

```
mem=224M console=ttyAMA0,115200 root=/dev/mtdblock2 rootfstype=yaffs2 \
  mtdparts=hi_sfc:2M(boot);hinand:8M(kernel),16M(rootfs),64M(user),32M(hdr000000) \
  pcieclkext=0
```

Because the two banks are non-contiguous with a large hole between them, a
modern kernel should describe them as two separate `memory` nodes in the device
tree rather than using a single `mem=` argument:

```dts
memory@80000000 {
    device_type = "memory";
    reg = <0x80000000 0x20000000>,   /* DDR0: 512 MB — confirmed */
          <0xc0000000 0x20000000>;   /* DDR1: 512 MB — confirmed */
};
```

> **This node only survives if `CONFIG_ARM_ATAG_DTB_COMPAT` is off.** With it on,
> the decompressor overwrites `/memory` with U-Boot's `ATAG_MEM` — one bank of
> 256 MB, a compile-time constant rather than a probe — and applies the vendor
> `bootargs`, which start `mem=224M`. Either alone puts you back where you
> started. See
> [getting a device tree into a modern kernel](03-boot-chain.md#getting-a-device-tree-into-a-modern-kernel).

### Evidence for the bank sizes

The 512 MB figure is measured. A write/read-back aliasing test from the U-Boot
prompt, using addresses spread across each bank:

```
mw 0x81000000 aaaa0000 ... mw 0x9f000000 aaaa0004     (DDR0, 5 addresses over 496 MB)
mw 0xc1000000 bbbb0000 ... mw 0xdf000000 bbbb0004     (DDR1, 5 addresses over 496 MB)
```

All ten read back their own distinct value:

```
81000000: aaaa0000     c1000000: bbbb0000
91000000: aaaa0001     d1000000: bbbb0001
99000000: aaaa0002     d9000000: bbbb0002
9d000000: aaaa0003     dd000000: bbbb0003
9f000000: aaaa0004     df000000: bbbb0004
```

No aliasing anywhere in either bank, so each is at least 496 MB — and since
DRAM comes in powers of two, exactly **512 MB**. Re-reading `0x81000000` after
the DDR1 writes still returned `aaaa0000`, confirming the two banks are
genuinely independent rather than two windows onto the same memory.

This is corroborated by the vendor's own alternate load script,
`rootfs/mtd/modules/load3531_fpga`, which reserves nearly all of both banks:

```
mmz=anonymous,0,0x84000000,447M:ddr1,0,0xC0000000,511M
```

`0x84000000 + 447 MB = 0x9FF00000` and `0xC0000000 + 511 MB = 0xDFF00000` —
each exactly 1 MB short of a full 512 MB bank.

Two further notes:

1. **The DRAM parts have not been decoded.** U1 and U2 are Nanya devices
   (photos in `pcb/`), and only the top surface has been photographed. Two
   packages totalling 1 GB implies 512 MB each, consistent with the measurement
   above.
2. **DDR init is done before U-Boot proper.** DDR training/timing is set up by
   the boot ROM and the SPI-NOR preloader using a register table, not by code
   in `board/godnet/board.c` (whose `dram_init()` only fills in
   `gd->bd->bi_dram[0]` from constants). A port that keeps the existing U-Boot
   does not need to touch this; a port that replaces the bootloader does, and
   the register table would have to be extracted from the SPI-NOR image.

## Making both banks usable

Two `reg` entries describe the memory. They do not, on their own, make it
reachable. On 32-bit ARM the low-memory linear map starts at `PHYS_OFFSET` and
runs for a fixed span, and **`0xC0000000` is outside that span under the default
3G/1G split** — so a kernel that is otherwise correct will boot with 512 MB and
print a notice about the rest.

### Why the second bank falls outside

`adjust_lowmem_bounds()` in `arch/arm/mm/mmu.c` computes the physical ceiling of
low memory as

```
vmalloc_limit = VMALLOC_END − vmalloc_size − VMALLOC_OFFSET − PAGE_OFFSET + PHYS_OFFSET
```

with `VMALLOC_END` = `0xFF800000`, `VMALLOC_OFFSET` = 8 MB and `vmalloc_size`
defaulting to 240 MB. `PHYS_OFFSET` is `0x80000000` here. That gives:

| Split | `PAGE_OFFSET` | Low-memory window, physical | DDR0 | DDR1 |
|---|---|---|---|---|
| `VMSPLIT_3G` (default) | `0xC0000000` | `0x80000000`–`0xAFFFFFFF` (768 MB) | Low | **High** |
| `VMSPLIT_3G_OPT` | `0xB0000000` | `0x80000000`–`0xBFFFFFFF` (1024 MB) | Low | **High** — the window ends exactly where DDR1 begins |
| `VMSPLIT_2G` | `0x80000000` | `0x80000000`–`0xEFFFFFFF` (1792 MB) | Low | **Low** |

Shrinking the vmalloc reservation does not rescue the 3G split: even at the
16 MB floor the window only reaches `0xBE000000`, still short of `0xC0000000`.
`VMSPLIT_3G_OPT` is a particular trap — its help text offers "full 1G low
memory", and the 1 GB it grants stops one byte below the bank you want.

### What happens if you get it wrong

`adjust_lowmem_bounds()` ends with:

```c
if (!IS_ENABLED(CONFIG_HIGHMEM) || cache_is_vipt_aliasing()) {
        if (memblock_end_of_DRAM() > arm_lowmem_limit) {
                ...
                pr_notice("Ignoring RAM at %pa-%pa\n", &memblock_limit, &end);
                pr_notice("Consider using a HIGHMEM enabled kernel.\n");
                memblock_remove(memblock_limit, end - memblock_limit);
```

So a 3G-split kernel without `CONFIG_HIGHMEM` discards DDR1 outright and says
so. **If a port reports ~512 MB, search the boot log for `Ignoring RAM`** — that
is this, and not a device-tree mistake.

### Two ways to get all of it

| Approach | Config | Trade-off |
|---|---|---|
| **`CONFIG_VMSPLIT_2G`** | `PAGE_OFFSET` becomes `0x80000000` | Both banks become low memory, no highmem machinery at all. User address space drops from 3 GB to 2 GB — irrelevant on a 1 GB machine. **Recommended** |
| `CONFIG_HIGHMEM` with the 3G split | Keep `PAGE_OFFSET` at `0xC0000000` | Works, and keeps 3 GB of user space, but every kernel access to DDR1 goes through `kmap`, and kernel allocations stay confined to DDR0 |

Either gives the full ~1 GB. The 2G split is simpler and faster; nothing about
this workload needs 3 GB of user virtual address space.

### The hole

The 512 MB gap at `0xA0000000`–`0xBFFFFFFF` needs no special handling. Both
banks and the gap are 256 MB-aligned, which is exactly ARM's
`SECTION_SIZE_BITS` of 28, so `CONFIG_SPARSEMEM` represents the layout without
waste — and `ARCH_SPARSEMEM_ENABLE` is `def_bool !ARCH_FOOTBRIDGE`, so it is
available here. `CONFIG_FLATMEM` also works, at the cost of a few megabytes of
`struct page` covering pages that do not exist. The vendor kernel uses FLATMEM,
but it only ever saw one bank.

No memblock or zone precautions are needed beyond the two `reg` entries. The
linear map is built per memblock region, so the gap simply has no page tables.

### DMA

No restriction applies. ARM creates `ZONE_DMA` only when the machine descriptor
sets `dma_zone_size`; with it unset, `setup_dma_zone()` leaves `arm_dma_limit`
at `0xFFFFFFFF` and every address is DMA-able. A DT-only platform sets nothing,
so this is the default.

The hardware side is partly evidenced rather than proven: the vendor firmware
runs 504 MB of MMZ video buffers in DDR1 at `0xC0000000`, so the media engines
demonstrably master to that range. The same has not been shown for AHCI,
`stmmac` or EHCI, which never see those addresses under the vendor
configuration. All three are 32-bit AXI masters on the same interconnect and
there is no reason to expect trouble, but if `CONFIG_HIGHMEM` is used rather
than the 2G split, DMA to and from DDR1 becomes routine and is worth watching
during Phase 4.

## Kernel virtual layout (vendor kernel, for reference)

```
vector  : 0xffff0000 - 0xffff1000   (   4 kB)
fixmap  : 0xfff00000 - 0xfffe0000   ( 896 kB)
DMA     : 0xffc00000 - 0xffe00000   (   2 MB)
vmalloc : 0xce800000 - 0xfe000000   ( 760 MB)
lowmem  : 0xc0000000 - 0xce000000   ( 224 MB)
modules : 0xbf000000 - 0xc0000000   (  16 MB)
```

Note the vendor kernel's lowmem ends at `0xCE000000` and vmalloc starts there —
the static IO windows at `0xFE000000` / `0xFE900000` sit inside vmalloc space.
