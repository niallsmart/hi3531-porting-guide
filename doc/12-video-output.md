# Video Output

Three output devices: an HD path carrying VGA and HDMI, and two
standard-definition CVBS paths.

## Output devices

From `/proc/umap/vo` on the running device:

```
-----DEV CONFIG--------------------------------------------------
 DevId   DevEn    Mux1    Mux2       InfSync   BkClr  DevFrt DispFrt
     1       Y     VGA    HDMI    800x600@60       0      60      30
     2       Y    CVBS                   NTSC      0      30      30
     3       Y    CVBS                   NTSC      0      30      30
```

| Device | Outputs | Timing | Frame rate | Notes |
|---|---|---|---|---|
| 1 | VGA + HDMI (simultaneous) | 800x600@60 | 60 Hz dev / 30 Hz display | Main monitor output |
| 2 | CVBS | NTSC (720x480) | 30 Hz | Composite |
| 3 | CVBS | NTSC (720x480) | 30 Hz | Composite (spot monitor) |

Device 1 drives VGA and HDMI from the same timing generator, which is why both
run at the same 800x600@60. That is the vendor's choice of mode, not a hardware
limit — the SoC does 1080p60 on HDMI and up to 2560x1600@60 on VGA. Device 1 was showing a 4-channel view with
picture-in-picture enabled (`EnChNum 4`, a second entry with `PiP Y` at
704x480).

Video parameters at capture — device 1: luma 37, contrast 57, hue 50,
saturation 47; devices 2 and 3 at the default 50 across the board.

IRQ 91 (`VOU Interrupt`) had the highest count of any peripheral (681,012),
consistent with a continuously refreshing display.

## HDMI — the SoC's integrated transmitter

HDMI comes from the **Hi3531's own integrated HDMI transmitter**, handled by
`hi3531_hdmi.ko`. There is no external Silicon Image SiI9024 in the signal
path, despite a driver for one shipping in the filesystem.

The evidence:

- Every `insmod extdrv/sil9024.ko` line in the active load script
  `rootfs/mtd/modules/load3531` is **commented out**, for all eleven video
  modes. The module ships but is never loaded.
- `sil9024` does not appear in `/proc/modules` on the running device.
- The pinctrl script muxes the SoC's own HDMI pins:

  | Signal | Pin | Register | Value |
  |---|---|---|---|
  | `HDMI_HOTPLUG` | GPIO17_7 | `0x200f0244` | 1 |
  | `HDMI_CEC` | GPIO18_0 | `0x200f0248` | 1 |
  | `HDMI_SDA` (DDC) | GPIO18_1 | `0x200f024c` | 1 |
  | `HDMI_SCL` (DDC) | GPIO18_2 | `0x200f0250` | 1 |

  A dedicated DDC pair and hotplug pin on the SoC is exactly what an integrated
  transmitter needs, and would be redundant alongside an external one.
- `/proc/umap/hdmi` reports a `Hisi HDMI Dev Stat` block, not a third-party
  device.

`gpioi2c.ko` does still export `gpio_sil9024_i2c_read`/`_write` helpers — those
are dead code retained for sibling board variants that do fit the part.

See [19-pinmux-map.md](19-pinmux-map.md).

State at capture, from `/proc/umap/hdmi`:

```
[HDMI] Version: [Hi3531_MPP_V1.0.0.0 Debug], Build Time[Aug 29 2012, 11:57:03]

 DevId  Open Start    Event
     0     y     n  NO PLUG

HPD Status:                Out
HDMI do not Start!
```

The HDMI device was open but not started, with hot-plug detect showing nothing
connected. No monitor was attached.

Because the transmitter is inside the SoC, mainline's `sii902x` bridge driver
is **not** applicable here. Reviving HDMI under a modern kernel would mean
writing a driver for the Hi3531's own HDMI block as well as the VOU — both
undocumented. That is a considerably worse position than an external bridge
chip would have been.

## CVBS encoder

The SDK ships `adv_7179.ko` — a driver for the Analog Devices ADV7179 video
encoder — and the module is present in the DVR's filesystem, but **it is not
used on this board**.

`insmod extdrv/adv_7179.ko` appears only in `load3531_cascade`, the script for
the PCIe-cascaded multi-SoC variants. The script this board actually runs,
`load3531` (selected as `-i 4hd` by `dep2.sh`), never loads it, and
`adv_7179` does not appear in `/proc/modules`.

Since both CVBS outputs were enabled and running with no encoder driver loaded,
they are driven by the **Hi3531's integrated CVBS DACs**.

### Analog output stage

The DACs feed an external buffer: `U17`, an **SG Micro SGM9119** marked
`SGM9119YS8` / `1323C`. It is a 3-channel, 5th-order standard-definition video
reconstruction filter driver in SOIC-8 — 3.3–5.5 V, 8 MHz −3 dB bandwidth,
transparent input clamps, 6 dB gain, rail-to-rail output, about 21 mA.

Three channels covers the two CVBS outputs in use with one spare. Three
electrolytic capacitors sit immediately beside the part, consistent with
per-channel output coupling, though the connections have not been traced.

This is a passive analog stage with no register interface, no I²C address and
no clock: nothing for a device tree to describe and no driver to port. It
matters only as confirmation that the CVBS path is DAC-plus-buffer rather than
an external encoder — a mainline port that brings up the VOU's DACs gets working
composite output with no additional support.

## Framebuffer layers

The vendor stack exposes HiSilicon's `hifb` framebuffer driver with seven
layers, backed by MMZ allocations:

| Device | MMZ block | Size |
|---|---|---|
| `/dev/fb0` | `hifb_layer0` | 1024 KB |
| `/dev/fb1` | `hifb_layer1` | 1024 KB |
| `/dev/fb2` | `hifb_layer2` | 1024 KB |
| `/dev/fb3` | `hifb_layer3` | 4 KB |
| `/dev/fb4` | `hifb_layer4` | 8100 KB |
| `/dev/fb5` | `hifb_layer5` | 128 KB |
| `/dev/fb6` | `hifb_layer6` | 128 KB |

`hifb` is proprietary HiSilicon code with no mainline equivalent. The 8100 KB
layer 4 is large enough for a 1920x1080 32bpp surface.

The SoC also has a 2D acceleration engine (TDE, IRQ 98) with its own memory
pools (`TDE_MEMPOOL_MMB`, coefficient buffers), driven by `hi3531_tde.ko`.

### U-Boot handoff

A live register read at the U-Boot prompt, before Linux changed the VOU,
confirmed this scanout configuration:

| Layer | Output channel | Address | Stride | Input/output size |
|---|---|---|---|---|
| G0 | DHD0 — main HD path at 1280x1024@60 | `0xC1000000` | 960 bytes | 480x300 |
| G2 | DSD0 — first NTSC CVBS output | `0xC1000000` | 960 bytes | 480x300 |
| G3 | DSD1 — second NTSC CVBS output | `0xC1000000` | 960 bytes | 480x300 |

All three enabled layers use ARGB1555 and read the same 288,000-byte buffer;
none scales it. The 480x300 plane is positioned within each larger output.
U-Boot's 1280x1024 main-output mode is distinct from the vendor Linux stack's
800x600 mode documented above.

There are two valid kernel handoffs:

- For a headless system, run `stopgx` for layers 0, 2 and 3, then `stopvo` for
  devices 0, 2 and 3 before `bootm`.
- To retain a diagnostic display, reserve the page-aligned range
  `0xC1000000`–`0xC1046FFF` and describe one `simple-framebuffer` with width
  480, height 300, stride 960 and format `a1r5g5b5`. It is one logical display
  mirrored to all three outputs, with no modesetting or acceleration.

See [03-boot-chain.md](03-boot-chain.md) for the boot sequence.

## Assessment

There is no complete mainline path, but the VDP itself is documented.

| Block | Base | Documented? |
|---|---|---|
| **VDP** — the display controller, what MPP calls the VOU | `0x205C0000` | **Yes** — chapter 11.2, register summary and descriptions |
| **HDMI transmitter** | `0x205D0000` | **No.** The address map lists the block; the datasheet has no chapter for it |
| **TDE** — 2D graphics engine | `0x20610000` | **No** — chapter 8.1 stops at the functional description |
| **VPSS** — scaler | `0x20600000` | **No** — chapter 8.2, likewise |

So a DRM/KMS driver for VGA output is *possible* from documentation: VDP is the
scanout engine and it is fully described. HDMI is not, which matters because
the transmitter is inside the SoC rather than an external bridge — there is no
`sii902x` to fall back on. TDE and VPSS are unavailable, so acceleration and
hardware scaling are out; a driver would be dumb-framebuffer scanout.

For a headless server, stop the inherited output and use serial and SSH. The
firmware framebuffer above is a low-effort diagnostic console; a native VDP
driver is a much larger project. If one is ever justified, VGA is the tractable
path and HDMI is not.
