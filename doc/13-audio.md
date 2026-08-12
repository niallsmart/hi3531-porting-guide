# Audio

Documented at wiring level per the agreed scope.

## The codec is inside the video decoder

**There is no separate audio codec on this board.** Audio is handled by the
**Nextchip NVP1104B at U19** — the same part that decodes the video. The
NVP1104 is a combined *4-channel video decoder and 4-channel PCM voice codec*,
and the vendor uses both halves.

From `datasheets/NVP1104B Overview.pdf`:

| Property | Value |
|---|---|
| Codec | 4-channel PCM voice codec, integrated |
| Capture | **4-channel voice record** |
| Playback | **1-channel playback** |
| Encodings | Linear PCM (8/16 kHz, 8/16-bit); G.711 A-law / µ-law compand/expand (8/16 kHz, 8-bit) |
| Voice band | 300 Hz – 3400 Hz |
| Input gain | 0 – 21 dB, 3 dB steps |
| Output gain | −6 – +6 dB, 0.75 dB steps |
| Other | Input mixing, digital volume, mute detection |
| Digital interface | **SSP / DSP / I²S**, master or slave |
| Control | I²C — address `0x60` on this board |

The signal path, per the datasheet's own 4-channel DVR application diagram,
matches this board exactly:

```
AUDIO1..4 -> AIN_01..04 -> ADC -> decimation filter -> PCM engine
                                                        |
                              ADATA_REC, ACLK_REC, ASYNC_REC  (record)
                              ADATA_PB,  ACLK_PB,  ASYNC_PB   (playback)
                                                        |
                                                     Hi3531 SIO4
VOICE_OUT <- AOUT <- DAC <- interpolation filter
```

Four record channels and one playback channel is exactly what the runtime log
reports:

```
drv:set playback audio channel ok
```

and matches the chassis label's `4 Audio`.

### The TLV320AIC31 is not fitted

The SDK ships `tlv_320aic31.ko` for a TI stereo codec and the module is present
in the filesystem, but in the active load script
`rootfs/mtd/modules/load3531` both the insertion and removal lines are
commented out:

```sh
#insmod extdrv/tlv_320aic31.ko > /dev/null
...
#rmmod tlv_320aic31
```

It does not appear in `/proc/modules`, and no TI audio codec is visible in the
PCB photographs. It is dead code retained for sibling board variants.

## SoC audio blocks

These modules **were** loaded and active:

| Module | Role | Size | Refs |
|---|---|---|---|
| `hi3531_ai.ko` | Audio input (capture) | 176 KB | 6 |
| `hi3531_ao.ko` | Audio output (playback) | 178 KB | 4 |
| `hi3531_aenc.ko` | Audio encode | 43 KB | 7 |
| `hi3531_adec.ko` | Audio decode | 15 KB | 4 |
| `hi3531_sio.ko` | Serial audio interface (I²S/PCM) | 12 KB | 3 |

`hi3531_sio.ko` is the I²S controller and is a dependency of both `ai` and `ao`,
confirming the codec is attached over a serial audio interface.

`/proc/umap/` exposes `ai`, `ao`, `aenc` and `adec` nodes.

DMA buffers were allocated in MMZ at capture time:

| Block | Size |
|---|---|
| `AODMA` | 780 KB |
| `AODMA` | 132 KB |
| `AIDMA` | 20 KB (×2) |

And the runtime log shows the audio path being configured:

```
drv:set playback audio channel ok
```

So audio playback was set up and running under the vendor firmware, using DMA
into the SIO block.

## I²S pin assignment

The active pinctrl script muxes **SIO4** for audio:

| Signal | Pin | Register | Value |
|---|---|---|---|
| `SIO4_RCLK` | GPIO11_4 | `0x200f0170` | 1 |
| `SIO4_RFS` | GPIO11_5 | `0x200f0174` | 1 |
| `SIO4_DOUT` | GPIO11_6 | `0x200f0178` | 1 |
| `SIO4_DIN` | GPIO11_7 | `0x200f017c` | 1 |

The SoC provides five serial audio ports (SIO0–SIO4) across
`0x200f0138`–`0x200f017c`; only SIO4 is enabled, and it is the only one with a
`DOUT` as well as a `DIN`, i.e. the only bidirectional port. See
[19-pinmux-map.md](19-pinmux-map.md).

Four record channels sharing one bidirectional port is consistent: the codec
multiplexes them onto a single serial stream rather than giving each its own
bus.

## What is not known

- **The I²S framing actually used** — sample rate, bit depth, master or slave,
  and which of the SSP/DSP/I²S modes the decoder is configured for. The part
  supports 8 or 16 kHz at 8 or 16 bits, linear or G.711; which combination the
  vendor selects is set by the MPP stack at runtime and was not dumped.
- **The decoder's audio register map.** The overview document describes
  features and interfaces, not registers. Programming the codec still requires
  either the full NVP1104B datasheet or reverse-engineering `ncdecoder.ko`.
- **Where the analog audio enters the chassis** and how the single playback
  channel is routed to the outputs.

## Mainline support

Worse than it first appears, because the codec is not a standalone part:

- **There is no mainline driver for the NVP1104B's codec**, and none is likely
  — it is a surveillance-specific voice codec welded to a video decoder. The
  `tlv320aic31xx` ASoC driver in mainline is irrelevant here, since that part
  is not fitted.
- **The SoC side has no mainline driver either.** No ASoC platform driver
  exists for the Hi3531 SIO/AIO blocks; one would have to be written against
  the datasheet, plus a machine driver to bind the two ends together.

So enabling audio means writing *both* ends: an ASoC platform driver for the
SoC's SIO4 port, and a codec driver for the NVP1104B's PCM engine. The first is
a well-trodden pattern; the second needs documentation that is not public.
Neither is on the critical path for a server.

## Assessment

Skip for the server use case. Nothing depends on audio, and the modules simply
will not be loaded. The information here is recorded so the path could be
picked up later.

If audio is ever wanted, the order of work is: confirm the codec is physically
present, find its I²C address, write the ASoC platform driver for the SIO
block, then bind mainline `tlv320aic31xx` to it.
