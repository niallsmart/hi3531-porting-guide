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
| Internal product code | `2704XD_P` |
| PCB designation | `DHB_AX` — silkscreen only, absent from all firmware |

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

`DHB_AX` is TVT's designation for the PCB itself, not a product or model name.
The string appears on the silkscreen and nowhere else: it is absent from the
SPI-NOR image, the filesystem, `td3531`, `libhi3531.so` and the boot log. Only
the revision crosses into software, as `v1.2` in the board parameter block and
in the `hardwareVersion` string. The `_B` suffix on the rear I/O strip marks
the second board of the set, revised on its own numbering.

The product identity is carried separately, in flash — see
[Where the product identity lives](#where-the-product-identity-lives). One PCB
design would be expected to serve several SKUs, differentiated by component
population and by the parameter block rather than by the layout, which would
explain why the software never refers to the board. **That is inference from
the string's absence**; a `DHB_AX` board from another TVT model carrying a
different `productID` in the same parameter block layout would confirm it.

What `DHB` abbreviates is unknown. No sibling designation appears anywhere in
the available material.

Also visible on the overview photograph: the RTC coin cell (BT1), a buzzer
(BZ1), four "HUI KE" relays for alarm outputs, the Ethernet
magnetics module (`PM6C-1001A 1000BASE-T`, U6), a 2x10 header (CON1), the
serial console header (J3, silkscreened `UART0` — see
[05-uart-console.md](05-uart-console.md#the-j3-header)), and **ten SATA
connector footprints**
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

## Where the product identity lives

The internal code `2704XD_P` is a string compiled into **`libhi3531.so`**, in
the message `Current 2704XD_P DVR Product Param Init`, with a matching function
symbol `PCIV_ProgParam_DVR2704XD_P_Iniit` (TVT's typo). It is one of **29**
such per-model initialisation routines in the same library:

```
2704XD_P    2708XD_P    2708XE_M   2708XE_MH  2708XE_S   2716XD_P
NVR08LD     NVR08PE_M   NVR16LD    NVR16ND_M  NVR28XX Z
TD_4503D_A  TD2404MD_B  TD2408MD_B TD2416MD_B
TD2504HD_C  TD2508HD_C  TD2508HE_B TD2508HE_C TD2508HE_N100B
TD2516HD_C  TD2516HE_B  TD2516HE_C TD2516HE_N100B
TD2524HD_C  TD2524HE_B  TD2532HD_C TD2532HE_B  TD2616IPIN
```

One library carries parameter initialisation for 29 DVR and NVR products and
selects one at run time, through `PUBF_SysSetCurProductType` and
`xvfunc_get_product_info_change_param`, which pass a `product_type` integer.
`libhi3531.so` in `rootfs/mtd/` is byte-identical to the copy in the `user`
partition backup.

### What selects it: a flash parameter

The application reads a numeric product ID from a flash parameter block at
startup and resolves it to the model. From the boot console capture,
`doc/bootlog.txt`:

```
current flash param ==
------get 1------current param type == 4067
productID:10680,subProductID:0
...
vo device ... product_type= 2704
Current 2704XD_P
 DVR Product Param Init
```

So the chain is:

| Step | Value |
|---|---|
| Flash parameter type `4067` | `productID:10680`, `subProductID:0` |
| resolves to numeric type | `product_type = 2704` |
| selects the routine | `PCIV_ProgParam_DVR2704XD_P_Iniit` in `libhi3531.so` |
| which prints | `Current 2704XD_P DVR Product Param Init` |

The application reads roughly twenty parameter types in the range `4065`–`4084`
through the same interface. **Which flash region holds them is unknown.** The
SPI-NOR board parameter block at `0xBF000` (see
[04-flash-storage.md](04-flash-storage.md)) is the likely candidate; the
capture shows the read but not its source.

### The composite identity string

Printed by `Product.cpp` late in startup:

```
kernelVersion:CB13-D3B3-CB13,
hardwareVersion:10680.0.14.Q7-v1.2-31xx,
MCUVersion:10.04.23,
mac:018ae3ca249
```

`hardwareVersion` decomposes as:

| Field | Value | Also appears as |
|---|---|---|
| `10680` | product ID | flash parameter `4067`, above |
| `0` | sub-product ID | `subProductID:0` |
| `14` | — | **unidentified** |
| `Q7` | keyboard type | `GetKeyBoardNameFromFlash:Q7` |
| `v1.2` | board revision | `DHB_AX V1.2` silkscreen and the `v1.2` string in the SPI-NOR board parameter block |
| `31xx` | SoC family | Hi3531 |

The MAC is absent at kernel level — the boot log shows `no valid MAC address`
for both interfaces — and is applied later by the application from flash
(`macflash:00:18:ae:3c:a2:49`), which also derives `dvrId:0018ae3ca2490000`.

### Build provenance

From the same capture:

| Component | Evidence |
|---|---|
| Kernel builder | `lzg@localhost.localdomain` |
| Toolchain | gcc 4.4.1, `Hisilicon_v100(gcc4.4-290+uclibc_0.9.32.1+eabi+linuxpthread)` |
| Kernel build | `#20121101111407 SMP Mon Mar 11 11:23:32 CST 2013` |
| U-Boot | `Nov 01 2012 - 11:21:03`, matching `current auto update version : 201211011107` |
| MPP | `Hi3531_MPP_V1.0.7.3 Debug`, Aug 2012 |
| TVT application layer | `DVR SDK VERSION : *** Jun 15 2013 11:15:37 ***` / `*** 201306151103 ***` |

Three version timelines run separately: HiSilicon's MPP (Aug 2012), the kernel
and U-Boot (Nov 2012 build ID, kernel linked Mar 2013), and TVT's application
layer (Jun 2013).

### Source filenames

The application logs `file.cpp,line` on most operations, exposing its source
tree: `xdvr.cpp`, `LocalDevice.cpp`, `Product.cpp`, `MainFrame.cpp`,
`ConfigEx.cpp`, `ConfigSetMan.cpp`, `Mcu8952.cpp`, `StreamProc.cpp`,
`NetServer.cpp`, `webserver.cpp`, `SWL_ListenProcEx.cpp`,
`SWL_MultiNetComm.cpp`, `ExternalKeyboard.cpp`, `PUB_common.cpp`,
`ReclogManEx.cpp`, `CDOperationMan.cpp`, `DdnsManager.cpp`, `UpnpMan.cpp`,
`SmtpMan.cpp`, `AlarmMan.cpp`, `DisplayMan.cpp`, `FakeCurise.cpp`,
`DVRTimer.cpp`, `RecordMan.cpp`, `DataSerProc.cpp`, `NetReader.cpp`,
`LocalUIMan.cpp`, `pciv_stream.c`.

C++ with an MFC-influenced structure. `xdvr.cpp`, `SWL_MultiNetComm.cpp` and
`ReclogManEx.cpp` are distinctive enough to be worth searching directly when
hunting for leaked or GPL-released source.

`Mcu8952.cpp` names the front-panel MCU: 8952 is the AT89S52 documented in
[20-front-panel-mcu.md](20-front-panel-mcu.md).

### `/mnt/mtd/product/product.def` describes a different product

The text manifest declares:

```
PRODUCT_NAME       = "TD_2316ME"
VIDEO_INPUT_NUM    = "16"
VIDEO_OUT_DEVICE   = "VGA,CVBS"
VIDEO_SIZE_MASK    = "D1,CIF"
RESOLUTION_MASK    = "640X480,800X600,1024X768,1280X1024"
```

That is a 16-channel **standard definition** DVR with no HDMI. This unit is a
4-channel 1080p DVR with HDMI. `TD_2316ME` appears nowhere in `libhi3531.so`'s
product table, so the manifest and the compiled table are separate systems and
the manifest is not what selects the running identity. `td3531` does contain
the path string `./product/product.def`, so it references the file, but a
reference is not proof it is parsed. Treat the manifest as stale and the
`libhi3531.so` table as authoritative.

### Rebadging residue

All 27 language files under `mtd/WebSites/language/` direct users to a
different brand's support desk. From `english_us/string.js`:

> "Lost or forgotten passwords will require assistance from **Q-see**
> technical support to reset."

`mtd/WebSites/logo/logo.js` leaves the branding blank (`g_bUseLogo = true`,
`g_logoName = ""`). So this image was built for Q-See's product line and
shipped in an LTS-branded chassis with the support contact unchanged.

**Consequence for source hunting:** Q-See GPL releases and firmware archives
are a live lead, plausibly a better one than TVT's own, since US-market
rebadgers face more GPL compliance pressure than Shenzhen ODMs. Add `Q-See
GPL` and `Q-See DVR source` to the search terms above. Which Q-See model
corresponds to this board has not been established.

## Product model summary

| Attribute | Value |
|---|---|
| Retail brand and model | LTS LTD2704XE-P |
| ODM | TVT Digital (Shenzhen) |
| Internal product code | `2704XD_P` |
| PCB designation | `DHB_AX`, revision V1.2 |
| Application | `td3531` |
| Channels | 4 video, 4 audio, 4 alarm |
| Supply | DC 12 V |
| Serial number | `812 1309 95 X 0058` |
| Firmware base | HiSilicon Hi3531 SDK, MPP V1.0.7.3 (Aug 2012) |
| Kernel | Linux 3.0.8, built 2013-03-11 |
| Bootloader | U-Boot 2010.06, built 2012-11-01 |
