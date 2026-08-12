# Media Codec and MPP — Why This Is a Dead End

Per the agreed scope, this is a short note explaining why the H.264 hardware is
not a realistic porting target, rather than a deep dive.

## What is there

The Hi3531 is a video codec SoC, and the majority of its silicon is dedicated
media hardware. The vendor kernel loads 30+ proprietary modules for it:

| Block | Module | Purpose | IRQ |
|---|---|---|---|
| VEDU | `hi3531_h264e.ko` | H.264 encode | 92, 93 |
| VDEC / VFMW | `hi3531_vdec.ko`, `hi3531_vfmw.ko` (6.6 MB) | Video decode | 96, 97 |
| VPSS | `hi3531_vpss.ko` | Video pre/post-processing | 79, 80 |
| VOU | `hi3531_vou.ko` | Video output | 91 |
| VIU | `hi3531_viu.ko` | Video input | 90 |
| TDE | `hi3531_tde.ko` | 2D graphics engine | 98 |
| IVE | `hi3531_ive.ko` | Intelligent video engine | 88 |
| VDA | `hi3531_vda.ko` | Video detection/analysis | 100 |
| JPEG | `hi3531_jpege.ko`, `jpeg.ko` | JPEG encode | 94, 95 |
| MPEG4 | `hi3531_mpeg4e.ko` | MPEG-4 encode | — |
| RC | `hi3531_rc.ko` | Rate control | — |
| SCD | — | Stream cut detect | 103 |

Plus the supporting infrastructure: `hi3531_base.ko`, `hi3531_sys.ko`,
`hi3531_group.ko`, `hi3531_chnl.ko`, `hi3531_region.ko`, `mmz.ko`, `hidmac.ko`.

Collectively this is HiSilicon's **MPP** (Media Processing Platform). Userspace
talks to it through `/dev/` nodes and `/proc/umap/*`, using the `libmpi` API
documented in the SDK's *HiMPP Media Processing Software Development Reference*.

Version on this device: `Hi3531_MPP_V1.0.7.3 Debug`, built Aug 2012.

## Why it cannot be ported

**1. The modules are proprietary binaries.**

```
hi3531_base: module license 'Proprietary' taints kernel.
Disabling lock debugging due to kernel taint
```

Every MPP module is marked `(P)` in `/proc/modules`. The SDK ships them as
prebuilt `.ko` files and prebuilt userspace libraries — there is no source for
the media blocks, only headers and samples. They are compiled against kernel
3.0.8 with a specific vermagic and will not load on any other kernel.

**2. There is no public register documentation.**

The Hi3531 datasheet in `00.hardware/chip/` documents the SoC at the level
needed for board design. It does not contain the register-level programming
model for the codec blocks — that was only ever available to HiSilicon
customers under NDA, if at all.

**3. `hi3531_vfmw.ko` is 6.6 MB.**

The video firmware module is larger than the entire kernel image. It contains
microcode for the decoder's internal processor. Reimplementing it is not a
reverse-engineering task, it is a re-implementation of a video codec on
undocumented hardware.

**4. It depends on MMZ.**

The whole pipeline is built on HiSilicon's carve-out allocator, which reserves
~790 MB of the board's 1 GB. Keeping MPP means keeping MMZ means leaving Linux
with 224 MB — the exact trade-off a server port wants to reverse. See
[02-memory-map.md](02-memory-map.md).

**5. No one has done it.**

There is no open-source Hi3531 media driver, in mainline or out of tree. Other
HiSilicon SoCs have partial community support for their non-media blocks; none
has a working open codec driver.

## What this means in practice

For a general-purpose server, this is not a loss. The intended workloads —
file serving, networking, general Linux — use the CPU, RAM, Ethernet and SATA,
all of which port cleanly.

**The correct action is to not load any MPP module.** Doing so:

- frees ~790 MB of RAM,
- removes the proprietary taint,
- eliminates 20+ interrupt sources,
- and removes the dependency on a 2012 kernel ABI.

The codec silicon simply sits idle. It draws some static power but is otherwise
inert.

## If you need video encoding anyway

Realistic options, none of which involve this hardware:

- **Software encoding on the CPU.** Two Cortex-A9 cores at ~1 GHz with NEON
  absent (this part has VFPv3-D16 but `/proc/cpuinfo` does not list `neon`) will
  manage low-resolution, low-framerate H.264 at best. Adequate for a webcam
  stream, not for four 1080p channels.
- **Keep the vendor firmware.** If the DVR function matters more than the server
  function, the original firmware is intact in NAND and fully backed up. The
  board can be returned to it at any time.
- **Dual-boot.** Since the port is planned to live on the SATA disk with NAND
  untouched (see [16-porting-roadmap.md](16-porting-roadmap.md)), both remain
  available by changing U-Boot's boot command.

## Assessment

Documented as a dead end. Do not spend time here. Everything else in this
documentation set is a better use of effort.
