# Video Input Chain

This documents **how the video input hardware is wired and addressed**. It is
not a guide to making the capture path work on a modern kernel — see
[14-media-codec.md](14-media-codec.md) for why that is impractical.

## Signal path

```
4x analog CVBS camera inputs (BNC)
        |
        v
  Nextchip NVP1104B  (U19, I2C 0x60, chip ID 0x77)
        |  4x ITU-R BT.656, 8-bit 4:2:2, 27/54/108 MHz
        v
  Lattice ECP3 FPGA  (U91, LFE3-17EA)
        |  BT.1120, byte-lane multiplexed
        v
  Hi3531 VIU  (video input unit, IRQ 90)
        |
        v
  VPSS -> VENC (H.264) -> disk
```

**The format changes across the FPGA.** The decoder emits BT.656; the SoC is
configured for BT.1120. Converting and aggregating between the two is the
FPGA's job — see below.

## Video decoder

| Property | Value |
|---|---|
| Reference designator | U19 |
| Marking on package | **Nextchip NVP1104B** (from `pcb/U19 nextchip NVP 1104B.jpeg`) |
| Function | Combined 4-channel video decoder **and** 4-channel PCM voice codec |
| I²C address | `0x60` |
| Chip ID read back | `0x77` |
| Driver module | `ncdecoder.ko`, version `201301231903` |
| Device node | `/dev/nvp1114adev` |
| Channels | 4 video in, 4 audio in, 1 audio out |
| Supply | 3.3 V I/O, 1.8 V core |
| Power | ~0.4 W typical |

The audio half is documented in [13-audio.md](13-audio.md).

### Video capabilities

From `datasheets/NVP1104B Overview.pdf`:

| Property | Value |
|---|---|
| Standards | NTSC-M/J/4.43, PAL-B/G/H/I/D/K/L/M/N/Nc/60 |
| Inputs | 4 × CVBS, 10-bit 4-channel ADC |
| Output | 4 × 4:2:2 8-bit **ITU-R BT.656** |
| Output clocks | 27 MHz / 54 MHz / 108 MHz |
| Processing | On-chip analog AGC and clamp; 3H/5H 2-D adaptive comb filter and notch filter; programmable vertical peaking, brightness, contrast, saturation, hue; white peak detection and peak AGC; colour transient improvement; PAL colour compensation |

Four output multiplexer modes are available:

| Port | Mode |
|---|---|
| `VOD1` | 27 MHz CCIR656, single channel |
| `VOD2` | 54 MHz, 2-channel mux |
| `VOD3` | 108 MHz D1, 4-channel mux — **requires an external 108 MHz oscillator** |
| `VOD4` | 54 MHz CIF, 4-channel mux |

Which mode this board uses has not been determined; it is set by
`ncdecoder.ko` over I²C. The presence or absence of a 108 MHz oscillator near
U19 would settle whether `VOD3` is available.

### Naming discrepancy

Three different part numbers refer to this one chip:

| Source | Name |
|---|---|
| Package marking (photo) | `NVP1104B` |
| Driver runtime messages | `nvp1108` |
| Device node and symbols | `nvp1114a` |
| Module banner | `nc1108` |

This is normal for Nextchip's NVP11xx family, which shares one driver across
several pin- and register-compatible variants differing in channel count and
resolution. The decisive runtime evidence:

```
Current 2704XD_P
current decode chip number = 1
open /dev/nvp1114adev success
nvp1108 0x60 get chip id 77 init ok !
warning: nvp1108 0x62 i2c_read err !!!
warning: nvp1108 0x64 i2c_read err !!!
warning: nvp1108 0x66 i2c_read err !!!
```

The driver probes four I²C addresses because the part supports **cascade mode,
up to 4 chips**, giving 16-channel recording, mixing output and playback — the
datasheet states this explicitly. **Only one chip responds**, consistent with a
4-channel unit, with the model number LTD**2704**XE-P, and with the vendor's
internal product string `2704XD_P`.

Take the package marking (NVP1104B) as authoritative for the hardware, and
treat `nvp1108`/`nvp1114a` as driver-family names.

> **Only an overview document is available**, in `datasheets/`. It gives
> features, interfaces, block and application diagrams, but **no register
> map**. Programming the decoder still requires either the full datasheet,
> which Nextchip does not publish openly, or reverse-engineering
> `ncdecoder.ko`.

> **Package discrepancy.** The overview specifies the NVP1104 as 100-TQFP,
> 12 × 12 mm, 0.4 mm pitch. The PCB silkscreen around U19 numbers pins at 32,
> 33, 64, 65, 96 and 97, which indicates a **128-pin** package with 32 pins per
> side. The fitted part is the `B` variant, so it likely differs from the base
> NVP1104 the overview describes. Treat pin-level detail from that document
> with caution; the functional description is unaffected.

### Observed configuration

```
=============> nvp1108 set video mode [1] clk 0
```

Video mode 1 with clock setting 0. The mapping of these values to standards
(NTSC/PAL) is not documented, but the video output is configured for NTSC, so
mode 1 is likely NTSC.

The driver also carries a `nvp1108_init_pal` symbol and a
`g_chip_has_nvp1700_flag`, indicating PAL support and awareness of another
family member.

## FPGA

| Property | Value |
|---|---|
| Reference designator | U91 |
| Part | **Lattice ECP3, LFE3-17EA** (from `pcb/U91 Lattice LFE3-17EA.jpeg`) |
| Family | LatticeECP3, 17K LUT, low-cost FPGA with SERDES |
| Control interfaces | Bit-banged I²C (`gpio_fpga_i2c_read`/`_write` in `gpioi2c.ko`) and JTAG (`fpga_jtag.ko`) |

### Its role

The FPGA bridges a genuine format mismatch between the two chips either side of
it:

| | Produces / expects |
|---|---|
| NVP1104B output | 4 × ITU-R **BT.656**, 8-bit 4:2:2, 27/54/108 MHz |
| Hi3531 VIU input | **BT1120S**, 1Mux, 1920x1080, `sp420` |

BT.656 is an 8-bit standard-definition interface; BT.1120 is the 16-bit
high-definition one. The decoder cannot emit BT.1120, and the VIU is not
configured to accept BT.656, so the FPGA must convert between them — and the
byte-lane component masks in the VIU configuration show how the result is
packed:

```
 Dev   ComMsk0   ComMsk1
   0  ff000000    ff0000
   2      ff00        ff
```

Devices 0 and 2 take different byte lanes of the same wider bus, which is what
carrying two 8-bit BT.656 streams inside one 16-bit BT.1120 link looks like.
The same pattern repeats for devices 4 and 6.

So the FPGA is doing format conversion and channel aggregation. Whether it also
scales is unclear — the VIU capture geometry is 1920x1080 while the decoder's
sources are D1 or CIF, so either the FPGA upscales, or the 1920x1080 window is
simply a container for multiplexed lower-resolution streams. The latter is more
likely for a DVR of this class, but this has not been confirmed.

Two control paths exist:

- **I²C**, via the shared bit-banged bus. The FPGA's I²C address was not
  captured at runtime. The configuration code lives in userspace, in TVT's
  `libhi3531.so`, which exports `EXDRV_FPGA_Device_Open`,
  `EXDRV_FPGA_I2C_RW_Data`, `EXDRV_ConfigFpgaForLiveVi`,
  `SDVR_GetVideoLossForFPGA` and `xvfunc_gpioi2c_config_fpga_live_vi`, and
  carries the error strings `FPGA i2c read fail` / `FPGA i2c write fail`. So
  the FPGA is configured for live-view channel routing and queried for video
  loss, both over I²C, from the application rather than from a kernel driver.
- **JTAG**, via `fpga_jtag.ko`, which bit-bangs TCK/TMS/TDI/TDO on GPIO pins.
  The presence of a JTAG bit-bang driver in the shipping firmware strongly
  suggests the **bitstream is loaded at runtime by the host**, rather than from
  a dedicated configuration flash.

> **`fpga_jtag.ko` is never loaded on this board.** `load3531` inserts it only
> under `SDK_TYPE=8720p` (line 155), and `dep2.sh` selects `4hd` here because
> `bootargs` contains the partition name `32M(hdr000000)` and the script greps
> `/proc/cmdline` for `(hdr`. `load3531` is the only file in the entire rootfs
> that references `fpga_jtag`, and `td3531` contains no reference to
> `/dev/fpgasdi`. The misc device that `fpga_jtag.ko` registers therefore never
> exists on this unit, and `libhi3531.so`'s FPGA path — which opens
> `/dev/fpgasdi` — is dead code here.
>
> The bitstream is therefore not host-loaded on this board, and the JTAG
> inference above applies only to the 8720p variant. Two possibilities remain
> open: the FPGA self-configures from a dedicated configuration source at
> power-up and needs no runtime setup on a 4HD board, or the live-view routing
> calls fail silently and the FPGA runs in a fixed default mode.

### The bitstream

**The FPGA bitstream is compiled into `fpga_jtag.ko` itself.**

Parsing the module's ELF section headers:

| Section | Offset | Size |
|---|---|---|
| `.text` | `0x34` | 8,048 |
| **`.rodata`** | **`0x2004`** | **509,676** |
| everything else | — | < 4 KB each |
| **module total** | | **526,628** |

97% of the driver is a single `.rodata` blob, and it begins at offset `0x2004`
with the ASCII signature **`_SVME1.3`**.

`_SVME` is Lattice's **VME** (ispVM Embedded) format. The driver is a port of
Lattice's reference embedded programmer — its symbol table contains the whole
ispVME engine:

```
ispEntryPoint  ispProcessVME  ispVMStateMachine  ispVMShift  ispVMSend
ispVMRead      ispVMDelay     ispVMClocks        ispVMBypass ispVMLDELAY
g_pucAlgoArray g_pucDataArray g_iAlgoSize        g_iDataSize
readPort writePort initPort sclock EnableHardware DisableHardware
```

with source filenames `slim_vme.c`, `slim_pro.c` and `hardware.c`, and a
supported-version table of `_SVME1.1`, `_SVME1.2`, `_SVME1.3`.

The JTAG pins are driven through `hd_gpio_write_val`, `hd_gpio_read_val` and
`hd_gpio_set_dir` against a `g_fpga_gpio_ctrl` descriptor and `g_siIspPins`.

The driver registers a misc device named **`fpgasdi`** with an ioctl entry
point (`fpga_sdi_ioctl`), reports its version as `fpga sdi ver 201306171002
driver`, and prints `fpga sdi firware down success` when programming completes.
It also counts devices (`g_fpga_chips_count`, `cur cpu type %d, fpga cnt %d`),
so the same driver supports boards with more than one FPGA.

Practical consequence: **the bitstream can be recovered as a file** by
extracting `.rodata` from `rootfs/mtd/modules/extdrv/fpga_jtag.ko` at offset
`0x2004`. Anyone reviving the video path can replay it with Lattice's own
tooling rather than reverse-engineering it.

### What is still not known

- **The FPGA's I²C address and register map.** Entirely undocumented. The
  `gpio_fpga_i2c_read`/`_write` helpers exist in `gpioi2c.ko` but the address
  was not captured at runtime.
- **Whether the FPGA is required for the board to boot.** Probably not, but
  unverified.
- **The specific GPIO pins used for JTAG.** The pinctrl scripts cover
  `0x200f0000`–`0x200f01a8` and `0x200f0244`–`0x0250`; the JTAG pins are not
  among the named functions, so they are ordinary GPIOs whose bit numbers live
  only in the driver's `g_siIspPins` table.
- **What the bitstream actually implements.** Recoverable as a blob, but VME is
  a programming stream, not readable RTL.

This is a genuinely closed component: a custom bitstream for a custom board,
with no source and no documentation. Reviving the video capture path means
either using the vendor bitstream as an opaque blob loaded exactly as the
vendor driver loads it, or replacing it entirely — which would mean writing new
FPGA RTL.

## Hi3531 VIU

From `/proc/umap/vi` on the running device:

```
[VIU] Version: [Hi3531_MPP_V1.0.7.3 Debug], Build Time: [Aug 29 2012, 11:56:57]

-----VI DEV ATTR------------------------------------------
 Dev   IntfM  WkM  ComMsk0  ComMsk1 ScanM   Seq
   0 BT1120S 1Mux ff000000   ff0000     P  UVUV
   2 BT1120S 1Mux     ff00       ff     P  UVUV
   4 BT1120S 1Mux ff000000   ff0000     P  UVUV
   6 BT1120S 1Mux     ff00       ff     P  UVUV

-----VI PHYCHN ATTR---------------------------------------
 PhyChn CapW  CapH  DstW  DstH CapSel PixFom Stride
      0 1920  1080  1920  1080   both  sp420   1920
      4 1920  1080  1920  1080   both  sp420   1920
      8 1920  1080  1920  1080   both  sp420   1920
     12 1920  1080  1920  1080   both  sp420   1920
```

| Property | Value |
|---|---|
| Interface mode | BT1120S (single-link BT.1120) |
| Working mode | 1Mux — one video stream per interface |
| Devices used | 0, 2, 4, 6 (four of eight) |
| Physical channels | 0, 4, 8, 12 |
| Capture resolution | 1920x1080 progressive |
| Pixel format | `sp420` — semi-planar YUV 4:2:0 |
| Component masks | `0xFF000000`/`0xFF0000` and `0xFF00`/`0xFF` — alternating byte lanes |
| IRQ | 90 (VIU) |

The alternating component masks across devices show the four streams are packed
into byte lanes of wider buses — two devices share each physical bus, each
taking a different byte lane. This is consistent with the FPGA aggregating
channels.

`LosInt` (loss-of-signal interrupts) was 2 on every channel with `IntCnt` 0,
meaning **no cameras were connected** at capture time.

## Assessment

The capture path has three elements, and they are blocked to different degrees.

| Element | Status |
|---|---|
| The SoC capture block | **Documented.** VICAP at `0x20580000`, chapter 11.1 of the Hi3531 datasheet, with a register summary and roughly 85 pages of register descriptions |
| The Nextchip NVP1104B decoder | **Undocumented, observable.** Only an overview document is public, with no register map. The initialisation sequence is recoverable from the I²C bus |
| The Lattice FPGA | **Undocumented, observable.** No I²C address or register map recovered, and its bitstream is not host-loaded on this board. Its configuration writes are recoverable from the same bus |

The SoC side is programmable from the datasheet, and a V4L2 driver for VICAP is
a normal, if large, driver-writing job. The two chips in front of it have no
public documentation, but both are configured over the bit-banged I²C bus by
firmware that runs correctly on every boot, so what they need can be captured
from a working system rather than derived from a datasheet. See
[A possible route to capture on a modern kernel](#a-possible-route-to-capture-on-a-modern-kernel).

Encoding is a separate matter: VEDU has no register documentation, so hardware
H.264 remains reachable only through MPP. See
[14-media-codec.md](14-media-codec.md).

See
[18-reference-assets.md](18-reference-assets.md#the-hi3531-datasheet--what-it-does-and-does-not-document)
for the datasheet coverage map.

For the server use case, the correct action is to leave the video input
hardware unconfigured. It consumes no resources if its drivers are not loaded.

## A possible route to capture on a modern kernel

**Unexplored. Recorded as a direction for future work.**

The two obstacles in the table above are gaps in documentation rather than in
what can be observed. Both chips are configured over the same bit-banged I²C
bus, and the stock firmware configures them correctly on every boot, so the
byte sequences that work on this board can be recovered by capturing them:

- Instrument `gpio_i2c_write` and `gpio_i2c_read` under the stock 3.0.8 kernel
  and log every transaction while the firmware brings video up. Source for the
  bus driver is in the SDK at `mpp/extdrv/gpio_i2c_8b/`, so it can be rebuilt
  with logging added.
- Or capture the bus with a logic analyser on the SCL/SDA pins.

That yields the NVP1104B initialisation sequence and, if the FPGA is addressed
at all on this board (see the note under [Its role](#its-role)), its
configuration writes.

With those in hand, a capture path on a mainline kernel becomes conceivable:

| Piece | Status |
|---|---|
| VICAP register programming | Documented — chapter 11.1, ~85 pages |
| NVP1104B setup | Recoverable by I²C tracing |
| FPGA setup | Recoverable by I²C tracing, if it is configured at runtime at all |
| Encoding | **Not available.** VEDU has no register documentation (chapter 7.2 is functional description only), so H.264 encode stays locked to MPP |

Encoding would have to be software on the Cortex-A9 pair — plausible for one or
two channels at D1, out of reach for 4x1080p.

Note the shape of the datasheet's coverage: the blocks at the edges of the
pipeline — VICAP in (11.1), VDP out (11.2) — have full register manuals, while
everything that transforms or compresses in between (VEDU 7.2, VDH 6.1, VPSS
8.2, TDE 8.1) has functional description only. Pixels in and pixels out are
documented; the parts that would let MPP be replaced are not.
