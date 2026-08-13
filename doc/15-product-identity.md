# Product Identity

## The ODM is TVT, not LTS

The unit is branded **LTS LTD2704XE-P**, but LTS is a rebadger. The original
design manufacturer is **Shenzhen TVT Digital Technology Co., Ltd.**

Evidence, all from the running device:

| Signal | Value |
|---|---|
| MAC address OUI | `00:18:AE` — registered to TVT Co., Ltd. |
| Main application binary | `td3531` (7.5 MB, in `/mnt/mtd`) |
| Shared library | `libhi3531.so` (1.9 MB) |
| CryptoMemory driver banner | `TVT 35xx CryptoMemory Device Driver v1.0.0` |
| Application version string | `version:2012030908580+->TD3515` |
| Internal product code | `Current 2704XD_P` |
| Board/product family | `DHB_AX` (used in the backup naming) |

**This matters for the port.** TVT-manufactured DVRs are sold under many brands
(LTS, Q-See, Night Owl, Swann, and others), so:

1. **GPL source releases** are far more likely to be found under TVT's name, or
   under a larger rebadger's, than under LTS. A TVT GPL drop for a Hi3531
   platform would contain the vendor's own board file, corroborating the pinmux
   and GPIO detail derived here from the `pinctrl_*.sh` scripts
   (see [19-pinmux-map.md](19-pinmux-map.md)).
2. **Other people's work** on TVT Hi3531 boards is more likely to be findable
   and applicable than work on "LTS LTD2704XE-P" specifically.

Search terms worth trying: `TVT TD3531`, `TVT DHB_AX`, `td3531 GPL`,
`Hi3531 TVT source`, `2704XD`.

> **There is no FCC ID, so FCC internal photographs are not available.** The
> chassis label carries the FCC logo but no ID number, meaning the unit was
> authorised under Supplier's Declaration of Conformity rather than
> certification. A DVR is an unintentional radiator (Part 15B), which does not
> require an FCC ID and therefore generates no public filing. Underside board
> photographs have to be taken directly.

## Chassis label

```
Model: LTD2704XE-P
4CH 1080P Realtime H.264 DVR,
HDMI, 4 Audio, 4 Alarm, DC 12V
UPC 8 12009 01917 0          CE   FC
S/N: 812 1309 95 X 0058      MADE IN CHINA
Warranty void if removed.
```

| Field | Value | Corroborates |
|---|---|---|
| Channels | **4CH 1080P realtime** | The four BT.1120 inputs at 1920x1080 and the `-i 4hd` variant selection |
| Video out | **HDMI** | The SoC's integrated HDMI transmitter |
| Audio | **4 inputs** | The audio path in [13-audio.md](13-audio.md) |
| Alarm | **4 channels** | Four relays on the board and four `COM`/`NO` output pairs plus four inputs on the rear terminal block — see [10-rtc-watchdog-misc.md](10-rtc-watchdog-misc.md#alarm-io) |
| Power | **DC 12 V** | — |
| Serial | `812 1309 95 X 0058` | `1309` matches the PCB date code `20130921` and UL date code `3813` |
| UPC | `8 12009 01917 0` | — |

Photograph: `pcb/label.png`.

## Component inventory

From the labelled photographs in `pcb/` (top surface only):

Board silkscreen: **`DHB_AX V1.2`** on the main board, `DHB_AX_B V1.0` on the
rear I/O strip, with a manufacturing date code of `20130921` and UL marks
`E343438` / `94V-0` / `3813` (week 38, 2013). The `V1.2` matches the `v1.2`
string stored in the SPI-NOR parameter block — see
[04-flash-storage.md](04-flash-storage.md).

Also visible on the overview photograph: the RTC coin cell (BT1), a buzzer
(BZ1), four "HUI KE" relays for alarm outputs, the Ethernet
magnetics module (`PM6C-1001A 1000BASE-T`, U6), a 2x10 header (CON1), the
serial console header (J3), and **ten SATA connector footprints**
(`SATA1`–`SATA10`) of which only two are populated.

| Ref | Part | Function | Documented in |
|---|---|---|---|
| U1, U2 | Nanya (part number not decoded) | DDR SDRAM, 512 MB each | [02-memory-map.md](02-memory-map.md) |
| U16 | TI, marked `PN521` / `35KG4` / `AL2R`, 56-pin | **Unidentified** — sited beside the VGA and HDMI connectors | — |
| U17 | SG Micro SGM9119, marked `SGM9119YS8` / `1323C` | 3-channel 5th-order SD video reconstruction filter driver — analog video output stage | [12-video-output.md](12-video-output.md#cvbs-encoder) |
| U19 | Nextchip NVP1104B | 4-channel analog video decoder | [11-video-input.md](11-video-input.md) |
| U32 | Atmel AT89S52 | 8051 MCU — front panel, buzzer, alarm I/O | [20-front-panel-mcu.md](20-front-panel-mcu.md) |
| U34 | MaxLinear/Sipex SP490E, marked `SP490EE` / `1249L` / `C23819` | Full-duplex RS-485 transceiver — rear-panel RS485 | [05-uart-console.md](05-uart-console.md#rs485-rear-panel) |
| U67 | Realtek RTL8211CL | Gigabit Ethernet PHY | [06-ethernet.md](06-ethernet.md) |
| U88 | JMicron JMB321 | 5-port SATA port multiplier | [07-sata-storage.md](07-sata-storage.md) |
| U91 | Lattice LFE3-17EA | ECP3 FPGA — video aggregation | [11-video-input.md](11-video-input.md) |
| — | Spansion S25FL216K | 2 MB SPI-NOR | [04-flash-storage.md](04-flash-storage.md) |
| — | NAND, JEDEC `0x01 0xF1` | 128 MB NAND | [04-flash-storage.md](04-flash-storage.md) |

> **Only the top surface has been photographed.** The SPI-NOR, the NAND, the
> RTC and voltage regulators have not been located on the board. `U16` remains
> unidentified — its marking does not resolve to a confirmed TI part number,
> and the top line may be a symbolised marking rather than the orderable device
> name.

Note that no ADV7179 CVBS encoder, TLV320AIC31 audio codec or SiI9024 HDMI
transmitter is fitted — drivers for all three ship in the filesystem but none
is loaded, and the corresponding functions come from the SoC itself. See
[12-video-output.md](12-video-output.md) and [13-audio.md](13-audio.md).

## Product model summary

| Attribute | Value |
|---|---|
| Retail brand and model | LTS LTD2704XE-P |
| ODM | TVT Digital (Shenzhen) |
| Internal product code | `2704XD_P` |
| Board family | `DHB_AX`, revision V1.2 |
| Application | `td3531` |
| Channels | 4 video, 4 audio, 4 alarm |
| Supply | DC 12 V |
| Serial number | `812 1309 95 X 0058` |
| Firmware base | HiSilicon Hi3531 SDK, MPP V1.0.7.3 (Aug 2012) |
| Kernel | Linux 3.0.8, built 2013-03-11 |
| Bootloader | U-Boot 2010.06, built 2012-11-01 |
