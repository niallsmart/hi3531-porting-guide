# Video Input Chain

Per the agreed scope, this documents **how the video input hardware is wired and
addressed**, so it could be revived later. It is not a guide to making the
capture path work on a modern kernel — see
[14-media-codec.md](14-media-codec.md) for why that is impractical.

## Signal path

```
4x analog camera inputs (BNC)
        |
        v
  Nextchip decoder  (U19, I2C 0x60, chip ID 0x77)
        |  ITU-R BT.656 / BT.1120 digital video
        v
  Lattice ECP3 FPGA  (U91, LFE3-17EA)
        |  4x BT.1120 streams, 1920x1080
        v
  Hi3531 VIU  (video input unit, IRQ 90)
        |
        v
  VPSS -> VENC (H.264) -> disk
```

## Video decoder

| Property | Value |
|---|---|
| Reference designator | U19 |
| Marking on package | **Nextchip NVP1104B** (from `pcb/U19 nextchip NVP 1104B.jpeg`) |
| I²C address | `0x60` |
| Chip ID read back | `0x77` |
| Driver module | `ncdecoder.ko`, version `201301231903` |
| Device node | `/dev/nvp1114adev` |
| Channels | 4 |

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

The driver probes four I²C addresses because the family supports up to four
cascaded decoders for 16 channels. **Only one chip responds**, consistent with
a 4-channel unit — and with the model number LTD**2704**XE-P and the vendor's
internal product string `2704XD_P`.

Take the package marking (NVP1104B) as authoritative for the hardware, and
treat `nvp1108`/`nvp1114a` as driver-family names.

> **No public datasheet for the NVP1104B was located.** Nextchip does not
> publish these openly. Without it, the register-level programming of the
> decoder is only recoverable by reverse-engineering `ncdecoder.ko`.

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

### What is observable

The FPGA sits between the decoder and the SoC. Its role is inferred from the
VIU configuration rather than documented: the SoC receives **four independent
BT.1120 streams at 1920x1080**, which a 4-channel D1/960H analog decoder cannot
produce directly. The FPGA is therefore performing format conversion, scaling
and/or multiplexing to present the decoder's output in the form the VIU expects.

Two control paths exist:

- **I²C**, via the shared bit-banged bus. The FPGA's I²C address was not
  captured at runtime.
- **JTAG**, via `fpga_jtag.ko`, which bit-bangs TCK/TMS/TDI/TDO on GPIO pins.
  The presence of a JTAG bit-bang driver in the shipping firmware strongly
  suggests the **bitstream is loaded at runtime by the host**, rather than from
  a dedicated configuration flash.

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

Not portable in any practical sense. It requires: an undocumented Nextchip
decoder, an undocumented FPGA bitstream, and the proprietary HiSilicon VIU
block with no public register documentation and no mainline driver.

What *is* recorded here — the I²C address, the chip identity, the BT.1120
configuration, the channel mapping — is enough for someone to pick up the
thread later, which is the purpose of this file.

For the server use case, the correct action is to leave the video input
hardware unconfigured. It consumes no resources if its drivers are not loaded.
