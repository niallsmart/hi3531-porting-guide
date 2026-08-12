# Product Identity and PCIe

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
   platform would contain the actual board file with real GPIO and pinmux
   assignments — which is the single largest gap in this documentation
   (see [09-gpio-pinmux-i2c.md](09-gpio-pinmux-i2c.md)).
2. **Other people's work** on TVT Hi3531 boards is more likely to be findable
   and applicable than work on "LTS LTD2704XE-P" specifically.
3. **FCC filings** under the OEM's name typically include internal photographs,
   which would partly substitute for the missing underside PCB photos.

Search terms worth trying: `TVT TD3531`, `TVT DHB_AX`, `td3531 GPL`,
`Hi3531 TVT source`, `2704XD`.

## Component inventory

From the labelled photographs in `pcb/` (top surface only):

Board silkscreen: **`DHB_AX V1.2`** on the main board, `DHB_AX_B V1.0` on the
rear I/O strip, with a manufacturing date code of `20130921` and UL marks
`E343438` / `94V-0` / `3813` (week 38, 2013). The `V1.2` matches the `v1.2`
string stored in the SPI-NOR parameter block — see
[04-flash-storage.md](04-flash-storage.md).

Also visible on the overview photograph: the RTC coin cell (BT1), a buzzer
(BZ1), three "HUI KE" relays (K1/K3/K4) for alarm outputs, the Ethernet
magnetics module (`PM6C-1001A 1000BASE-T`, U6), a 2x10 header (CON1), the
serial console header (J3), and **ten SATA connector footprints**
(`SATA1`–`SATA10`) of which only two are populated.

| Ref | Part | Function | Documented in |
|---|---|---|---|
| U1, U2 | Nanya (part number not decoded) | DDR SDRAM, 512 MB each | [02-memory-map.md](02-memory-map.md) |
| U16 | TI, marked `PN521` / `35KG4` / `AL2R`, 56-pin | **Unidentified** — sited beside the VGA and HDMI connectors | — |
| U19 | Nextchip NVP1104B | 4-channel analog video decoder | [11-video-input.md](11-video-input.md) |
| U32 | Atmel AT89S52 | 8051 MCU — front panel | [10-rtc-watchdog-misc.md](10-rtc-watchdog-misc.md) |
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

## PCIe

Two PCIe root complexes are present and probed, and **neither links up**.

```
controller0, config base:0x20800000, mem size:0x800000
controller1, config base:0x20810000, mem size:0x800000
pci 0000:00:00.0: [19e5:3531] type 1 class 0x000604
pcie_read_from_device->501,pcie0 not link up!
pci 0000:02:00.0: [19e5:3531] type 1 class 0x000604
pcie_read_from_device->501,pcie1 not link up!
```

| Controller | Registers | Memory window | Config space | State |
|---|---|---|---|---|
| PCIe0 | `0x20800000` | `0x30000000`–`0x377FFFFF` | `0x40000000` | Not linked |
| PCIe1 | `0x20810000` | `0x60000000`–`0x677FFFFF` | `0x70000000` | Not linked |

The kernel command line carries `pcieclkext=0`, selecting the internal
reference clock.

The vendor filesystem includes PCIe cascade drivers (`pcit_dma_host.ko`,
`pcit_dma_slv.ko`, `mcc_drv_host.ko`, `mcc_usrdev_host.ko`) and the SDK
documents a multi-chip cascade application — the Hi3531 supports chaining
several SoCs over PCIe to build 16- and 32-channel DVRs. **This board does not
use that feature**; the root complexes are enumerated but nothing is attached.

The runtime log line `current system has pci device cnt 1, sdkv 0` is the
vendor application counting SoCs in the cascade, and finding one.

For the port: **PCIe is unused and there is nothing physically connected.**
Unless a PCIe slot or an unpopulated footprint exists on the board — which
cannot be determined from top-surface photos alone — this can be left disabled.
Enabling it would need a `pcie-hisi`-style controller driver that does not
exist for this SoC.

## Product model summary

| Attribute | Value |
|---|---|
| Retail brand and model | LTS LTD2704XE-P |
| ODM | TVT Digital (Shenzhen) |
| Internal product code | `2704XD_P` |
| Board family | `DHB_AX` |
| Application | `td3531` |
| Channels | 4 analog inputs |
| Firmware base | HiSilicon Hi3531 SDK, MPP V1.0.7.3 (Aug 2012) |
| Kernel | Linux 3.0.8, built 2013-03-11 |
| Bootloader | U-Boot 2010.06, built 2012-11-01 |
