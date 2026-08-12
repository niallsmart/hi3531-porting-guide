# Audio

Documented at wiring level per the agreed scope.

## Codec

| Property | Value |
|---|---|
| Part | **Texas Instruments TLV320AIC31** |
| Driver module | `tlv_320aic31.ko` (present in both the SDK and the DVR filesystem) |
| Interface | I²S / PCM via the SoC's SIO block |
| Control bus | I²C (bit-banged, presumed) |

The TLV320AIC31 is a stereo audio codec with an integrated headphone/speaker
amplifier, commonly paired with HiSilicon DVR SoCs.

> **Not used on this board.** In the active load script
> `rootfs/mtd/modules/load3531`, both the insertion and removal lines are
> commented out:
>
> ```sh
> #insmod extdrv/tlv_320aic31.ko > /dev/null
> ...
> #rmmod tlv_320aic31
> ```
>
> and `tlv_320aic31` does not appear in `/proc/modules`. No TI audio codec was
> identified in the PCB photos either. The audio path was nonetheless active
> (see below), so the codec function is coming from somewhere else — most
> likely the Nextchip decoder's integrated audio ADCs for capture, and the
> SoC's own DACs for playback.

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

## Channel count

The chassis label specifies **4 audio inputs** alongside the four video
channels (`4CH 1080P Realtime H.264 DVR, HDMI, 4 Audio, 4 Alarm, DC 12V`).

That is a useful constraint: a single bidirectional I²S port (SIO4) carries
four capture channels, so the four inputs are multiplexed onto it rather than
each having its own port. Nextchip NVP11xx decoders integrate audio ADCs with
exactly this arrangement — audio sampled alongside the video channels and
serialised out on one bus — which fits the absence of any separate audio codec
on this board.

## What is not known

- **The I²S configuration** — sample rate, bit depth, master/slave, frame
  format. The vendor MPP stack configures this internally and it was not
  dumped.
- **Whether the four audio inputs come through the NVP1104B** or from some
  other part. The reasoning above is inference from the channel count and the
  single SIO port, not a traced signal path. The `NVP1104B Overview.pdf` in
  `datasheets/` would settle it.
- **The physical connections** — how many audio inputs the DVR exposes, whether
  the analog audio for each camera channel goes through the codec or through
  the video decoder (Nextchip NVP11xx parts often include audio ADCs, which
  would make a separate codec unnecessary for capture).
- **Whether audio capture is via the codec or the video decoder.** Given
  `ncdecoder.ko` handles a 4-channel decoder and the driver includes
  `drv:set playback audio channel`, the decoder may well carry the audio
  inputs, with the TLV320AIC31 (if fitted) handling only output.

Resolving these would need the NVP1104B datasheet, underside PCB photos, and
tracing.

## Mainline support

Mixed:

- **The codec has a mainline driver.** `sound/soc/codecs/tlv320aic31xx.c`
  supports the AIC31 family via ASoC.
- **The SoC side does not.** There is no mainline ASoC platform driver for the
  Hi3531 SIO/AIO blocks. One would have to be written against the datasheet,
  plus a machine driver to bind the two together.

That is a real but bounded piece of work — an ASoC platform driver for a
straightforward I²S controller is a well-trodden pattern. It is, however,
entirely optional for a server.

## Assessment

Skip for the server use case. Nothing depends on audio, and the modules simply
will not be loaded. The information here is recorded so the path could be
picked up later.

If audio is ever wanted, the order of work is: confirm the codec is physically
present, find its I²C address, write the ASoC platform driver for the SIO
block, then bind mainline `tlv320aic31xx` to it.
