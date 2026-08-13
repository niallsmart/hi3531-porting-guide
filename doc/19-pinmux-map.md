# Pin Multiplexing Map

The complete IO_CONFIG function map for the Hi3531, and what this board selects
at each pin.

## Where this comes from

**The chip datasheet, section 2.1.5 "Description of Multiplexing Registers".**
The SDK ships it as
`00.hardware/chip/documents_en/Hi3531 H.264 Codec Processor Data Sheet.pdf`
(Issue 09, 2015-02-09, 1794 pages). Section 2.1.5 gives one page per register,
naming the pin and listing every function value:

```
muxctrl_reg103 is the multiplexing control register for the I2C_SCL pin.
  Offset Address 0x19C     Register Name muxctrl_reg103

  Bits   Access  Name             Description
  [0]    RW      muxctrl_reg103   Multiplexing information about the I2C_SCL pin.
                                  0: GPIO12_5
                                  1: I2C_SCL
```

It covers **151 registers, `0x000` through `0x258`, with no gaps.**

The vendor's `rootfs/mtd/modules/pinctrl_*_hi3531.sh` scripts carry the same
information as trailing comments, and an earlier version of this document was
built from them. Their **peripheral names are reliable** — every signal name in
the function columns survived the comparison unchanged. Their **GPIO numbers
are not**: they run three too high across the VIU0 block and three too low
across the VIU2 block, giving 37 wrong GPIO labels in the 108 registers the
scripts cover. Two more registers, the VGA pair at `0x0e4`/`0x0e8`, had their
values transposed. The scripts remain the authority for one thing: which
*value* the vendor writes.

The "This board" column below is better evidence than either, being read from
the running device.

## What this board selects

Two captures: `md` at the U-Boot prompt, before Linux runs anything, and
`devmem` under the running vendor kernel. Every register not listed reads 0 in
both, selecting the function in the "0" column of the full map.

| Register | Pin | U-Boot | Linux | Selected under Linux |
|---|---|---|---|---|
| `0x200f0004` | VIU0_VS | 0 — VIU0_VS | 2 | **UART2_TXD** |
| `0x200f004c` | VIU1_CLK | 1 — GPIO2_3 | 0 | **VIU1_CLK** |
| `0x200f0050` | VIU1_VS | 0 — VIU1_VS | 2 | **UART2_RXD** |
| `0x200f00e4` | VGA_HS | 1 — VGA_HS | 1 | **VGA_HS** |
| `0x200f00e8` | VGA_VS | 1 — VGA_VS | 1 | **VGA_VS** |
| `0x200f00ec` | VOU1120_CLK | 3 — VOU1120_CLK | 0 | **VIU3_CLK** |
| `0x200f00f0` | VOU1120_VS | 3 — VOU1120_VS | 0 | **VIU3_VS** |
| `0x200f00f4` | VOU1120_HS | 3 — VOU1120_HS | 0 | **VIU3_HS** |
| `0x200f00f8` | VOU1120_DATA15 | 3 — VOU1120_DATA15 | 0 | **VIU3_DAT15** |
| `0x200f00fc` | VOU1120_DATA14 | 3 — VOU1120_DATA14 | 0 | **VIU3_DAT14** |
| `0x200f0100` | VOU1120_DATA13 | 3 — VOU1120_DATA13 | 0 | **VIU3_DAT13** |
| `0x200f0104` | VOU1120_DATA12 | 3 — VOU1120_DATA12 | 0 | **VIU3_DAT12** |
| `0x200f0108` | VOU1120_DATA11 | 3 — VOU1120_DATA11 | 0 | **VIU3_DAT11** |
| `0x200f010c` | VOU1120_DATA10 | 3 — VOU1120_DATA10 | 0 | **VIU3_DAT10** |
| `0x200f0110` | VOU1120_DATA9 | 3 — VOU1120_DATA9 | 0 | **VIU3_DAT9** |
| `0x200f0114` | VOU1120_DATA8 | 3 — VOU1120_DATA8 | 0 | **VIU3_DAT8** |
| `0x200f0118` | VOU1120_DATA7 | 3 — VOU1120_DATA7 | 0 | **VIU3_DAT7** |
| `0x200f011c` | VOU1120_DATA6 | 3 — VOU1120_DATA6 | 0 | **VIU3_DAT6** |
| `0x200f0120` | VOU1120_DATA5 | 3 — VOU1120_DATA5 | 0 | **VIU3_DAT5** |
| `0x200f0124` | VOU1120_DATA4 | 3 — VOU1120_DATA4 | 0 | **VIU3_DAT4** |
| `0x200f0128` | VOU1120_DATA3 | 3 — VOU1120_DATA3 | 0 | **VIU3_DAT3** |
| `0x200f012c` | VOU1120_DATA2 | 3 — VOU1120_DATA2 | 0 | **VIU3_DAT2** |
| `0x200f0130` | VOU1120_DATA1 | 3 — VOU1120_DATA1 | 0 | **VIU3_DAT1** |
| `0x200f0134` | VOU1120_DATA0 | 3 — VOU1120_DATA0 | 0 | **VIU3_DAT0** |
| `0x200f0140` | SIO0_DIN | 0 — GPIO10_0 | 1 | **SIO0_DIN** |
| `0x200f0144` | SIO1_RCLK | 0 — GPIO10_1 | 1 | **SIO1_RCLK** |
| `0x200f0148` | SIO1_RFS | 0 — GPIO10_2 | 1 | **SIO1_RFS** |
| `0x200f014c` | SIO1_DIN | 0 — GPIO10_3 | 1 | **SIO1_DIN** |
| `0x200f0150` | SIO2_RCLK | 0 — GPIO10_4 | 1 | **SIO2_RCLK** |
| `0x200f0154` | SIO2_RFS | 0 — GPIO10_5 | 1 | **SIO2_RFS** |
| `0x200f0158` | SIO2_DIN | 0 — GPIO10_6 | 1 | **SIO2_DIN** |
| `0x200f015c` | SIO3_RCLK | 0 — GPIO10_7 | 1 | **SIO3_RCLK** |
| `0x200f0160` | SIO3_RFS | 0 — GPIO11_0 | 1 | **SIO3_RFS** |
| `0x200f0164` | SIO3_DIN | 0 — GPIO11_1 | 1 | **SIO3_DIN** |
| `0x200f0168` | SIO4_XCLK | 0 — GPIO11_2 | 1 | **SIO4_XCLK** |
| `0x200f016c` | SIO4_XFS | 0 — GPIO11_3 | 1 | **SIO4_XFS** |
| `0x200f0170` | SIO4_RCLK | 0 — GPIO11_4 | 1 | **SIO4_RCLK** |
| `0x200f0174` | SIO4_RFS | 0 — GPIO11_5 | 1 | **SIO4_RFS** |
| `0x200f0178` | SIO4_DOUT | 0 — GPIO11_6 | 1 | **SIO4_DOUT** |
| `0x200f017c` | SIO4_DIN | 0 — GPIO11_7 | 1 | **SIO4_DIN** |
| `0x200f0180` | SPI_SCLK | 0 — GPIO12_0 | 1 | **SPI_SCLK** |
| `0x200f0184` | SPI_SDO | 0 — GPIO12_1 | 1 | **SPI_SDO** |
| `0x200f0188` | SPI_SDI | 0 — GPIO12_2 | 1 | **SPI_SDI** |
| `0x200f018c` | SPI_CSN0 | 0 — GPIO12_3 | 1 | **SPI_CSN0** |
| `0x200f01a4` | UART1_RXD | 0 — GPIO12_7 | 1 | **UART1_RXD** |
| `0x200f01a8` | UART1_TXD | 0 — GPIO13_0 | 1 | **UART1_TXD** |
| `0x200f01b4` | RGMII0_CRS | 2 — RGMII0_RXER | 2 | **RGMII0_RXER** |
| `0x200f01b8` | RGMII0_COL | 2 — RGMII0_TXER | 2 | **RGMII0_TXER** |
| `0x200f01bc` | RGMII1_RXDV | 1 — RGMII1_RXDV | 1 | **RGMII1_RXDV** |
| `0x200f01c0` | RGMII1_RXD3 | 1 — RGMII1_RXD3 | 1 | **RGMII1_RXD3** |
| `0x200f01c4` | RGMII1_RXD2 | 1 — RGMII1_RXD2 | 1 | **RGMII1_RXD2** |
| `0x200f01c8` | RGMII1_RXD1 | 1 — RGMII1_RXD1 | 1 | **RGMII1_RXD1** |
| `0x200f01cc` | RGMII1_RXD0 | 1 — RGMII1_RXD0 | 1 | **RGMII1_RXD0** |
| `0x200f01d0` | RGMII1_RXCK | 1 — RGMII1_RXCK | 1 | **RGMII1_RXCK** |
| `0x200f01d4` | RGMII1_TXEN | 1 — RGMII1_TXEN | 1 | **RGMII1_TXEN** |
| `0x200f01d8` | RGMII1_TXD3 | 1 — RGMII1_TXD3 | 1 | **RGMII1_TXD3** |
| `0x200f01dc` | RGMII1_TXD2 | 1 — RGMII1_TXD2 | 1 | **RGMII1_TXD2** |
| `0x200f01e0` | RGMII1_TXD1 | 1 — RGMII1_TXD1 | 1 | **RGMII1_TXD1** |
| `0x200f01e4` | RGMII1_TXD0 | 1 — RGMII1_TXD0 | 1 | **RGMII1_TXD0** |
| `0x200f01e8` | RGMII1_TXCK | 1 — RGMII1_TXCK | 1 | **RGMII1_TXCK** |
| `0x200f01ec` | RGMII1_TXCKOUT | 1 — RGMII1_TXCKOUT | 1 | **RGMII1_TXCKOUT** |
| `0x200f01f0` | RGMII1_CRS | 2 — RGMII1_RXER | 2 | **RGMII1_RXER** |
| `0x200f01f4` | RGMII1_COL | 2 — RGMII1_TXER | 2 | **RGMII1_TXER** |
| `0x200f0240` | USB1_PWREN | not dumped | 1 | **USB1_PWREN** |
| `0x200f0244` | HDMI_HOTPLUG | not dumped | 1 | **HDMI_HOTPLUG** |
| `0x200f0248` | HDMI_CEC | not dumped | 1 | **HDMI_CEC** |
| `0x200f024c` | HDMI_SDA | not dumped | 1 | **HDMI_SDA** |
| `0x200f0250` | HDMI_SCL | not dumped | 1 | **HDMI_SCL** |

Reading across it:

- **`0x1bc`–`0x1ec` is the RGMII1 bus** — 13 pins at function 1: RXDV,
  RXD3–RXD0, RXCK, TXEN, TXD3–TXD0, TXCK and TXCKOUT, plus RXER and TXER at
  `0x1f0`/`0x1f4`. Set in U-Boot and left alone by Linux. This is the run
  previously recorded as an unidentified 13-pin bus. Only RGMII1's signals are
  multiplexed against GPIO; RGMII0 has mux registers for just `TXCK`, `CRS` and
  `COL` at `0x1b0`–`0x1b8`, with the rest of its bus on dedicated pins — so
  these registers show RGMII1 is wired out but say nothing either way about
  RGMII0.
- **The 19-pin bus at `0x0ec`–`0x134` is video *input*, not output.** U-Boot
  leaves it at 3 (`VOU1120`, the BT.1120 output bus), and the pinctrl script
  writes 0, selecting `VIU3`. With VIU0, VIU1 and VIU2 also at 0, all four
  video-input channels are active — which is what a 4-channel HD recorder needs.
- **The bit-banged I²C pins stay GPIO.** `0x198` and `0x19c` read 0 in both
  captures, so `I2C_SDA`/`I2C_SCL` are never routed to the hardware controller.
- **UART2 is switched on by Linux, not U-Boot** — `0x004` and `0x050` to
  function 2. That is the rear-panel RS485 port.
- **`0x04c` is not a buzzer pin on this board**, despite the vendor calling it
  one. `rootfs/mtd/dep2.sh` writes 0 to it under the comment
  `#set default buzzer gpio control`, and 0 is `VIU1_CLK` — the write takes the
  pin *out* of GPIO mode. U-Boot leaves it at 1 (`GPIO2_3`); every pinctrl
  variant that would also set 1 has the line commented out. Under Linux it
  reads 0 and is the VIU1 clock. The buzzer BZ1 is driven by the AT89S52
  microcontroller over `ttyAMA1`, observed on the wire — see
  [20-front-panel-mcu.md](20-front-panel-mcu.md). Whether GPIO2_3 is physically
  wired to anything on this board is unknown; the vendor comment is the only
  suggestion that it is, and the vendor disables it.
- `0x138` and `0x13c` (`SIO0_RCLK`, `SIO0_RFS`) read 0 under Linux even though
  the 4HD script writes 1 to both. Something after the script clears them;
  `SIO0_DIN` at `0x140` stays at 1.

## Pins a server port needs

| Function | Pin | Register | Value |
|---|---|---|---|
| **Bit-banged I²C SDA** | GPIO12_4 | `0x200f0198` | 0 |
| **Bit-banged I²C SCL** | GPIO12_5 | `0x200f019c` | 0 |
| Ethernet RGMII1 | 13 signals + RXER/TXER | `0x200f01bc`–`0x1f4` | 1, or 2 for the error pair |
| UART1 (front-panel MCU) | GPIO12_7, GPIO13_0 | `0x200f01a4`, `0x1a8` | 1 |
| UART1 flow control | GPIO12_6, GPIO13_1 | `0x200f01a0`, `0x1ac` | 0 — unused |
| UART2 (rear RS485) | VIU0_VS, VIU1_VS pins | `0x200f0004`, `0x0050` | 2 |
| SPI (`ssp.ko`) | GPIO12_0–12_3 | `0x200f0180`–`0x18c` | 1 |
| Audio I²S (SIO4) | GPIO11_2–11_7 | `0x200f0168`–`0x17c` | 1 |
| IR receiver | IR_IN | `0x200f01f8` | 0 |
| NAND data and ready | NF_DQ0–7, NF_RDY0/1 | `0x200f01fc`–`0x220` | 0 |
| SPI-NOR (SFC) | SFC_DIO, WP, DOI, HOLD | `0x200f0224`–`0x230` | 0 |
| USB over-current and power | GPIO17_3–17_6 | `0x200f0234`–`0x240` | 0, except `0x240` = 1 |
| HDMI hotplug, CEC, DDC | GPIO17_7, GPIO18_0–18_2 | `0x200f0244`–`0x250` | 1 |
| SATA activity LEDs | GPIO18_3, GPIO18_4 | `0x200f0254`, `0x0258` | 0 — LED function not selected |
| VGA sync | VGA_HS, VGA_VS | `0x200f00e4`, `0x0e8` | 1 |

### The I²C pins

**SDA is GPIO12_4 and SCL is GPIO12_5.** The datasheet settles this: `0x198` is
the `I2C_SDA` pin with `0: GPIO12_4`, and `0x19c` is the `I2C_SCL` pin with
`0: GPIO12_5`. The vendor script comments label both as `GPIO12_4`, which is one
of the errors described above; the SDK source comment in
`mpp/extdrv/gpio_i2c_8b/gpio_i2c.c` agrees with the datasheet.

Both registers read 0 in U-Boot and under Linux, which is what a bit-banged
driver needs, so mainline `i2c-gpio` can replace the vendor `gpioi2c` stack:

```dts
i2c-gpio {
    compatible = "i2c-gpio";
    sda-gpios = <&gpio12 4 (GPIO_ACTIVE_HIGH|GPIO_OPEN_DRAIN)>;
    scl-gpios = <&gpio12 5 (GPIO_ACTIVE_HIGH|GPIO_OPEN_DRAIN)>;
    #address-cells = <1>;
    #size-cells = <0>;

    rtc@68 { compatible = "dallas,ds1307"; reg = <0x68>; };   /* address unverified */
};
```

## Full function map

One 32-bit register per pin at `0x200F0000 + offset`, holding a function-select
integer. The field is one, two or three bits wide depending on how many
functions the pin has; the datasheet gives the width per register, and writing a
value the pin does not define is reserved.

"Pin" is the datasheet's name for the physical ball, which is not always the
function selected by 0 — `0x0ec` is the `VOU1120_CLK` pin but selects `VIU3_CLK`
at 0. "This board" is the value read from the running vendor kernel.

| Register | Pin | 0 | 1 | 2 | 3 | 4 | This board |
|---|---|---|---|---|---|---|---|
| `0x200f0000` | VIU0_CLK | VIU0_CLK | GPIO0_0 | VOU0_CLK |  |  | **0** |
| `0x200f0004` | VIU0_VS | VIU0_VS | GPIO0_1 | UART2_TXD |  |  | **2** |
| `0x200f0008` | VIU0_HS | VIU0_HS | GPIO0_2 | VOU1_CLK | VIU0_CLKA |  | **0** |
| `0x200f000c` | VIU0_DAT15 | VIU0_DAT15 | GPIO0_3 | VOU0_DATA7 |  |  | **0** |
| `0x200f0010` | VIU0_DAT14 | VIU0_DAT14 | GPIO0_4 | VOU0_DATA6 |  |  | **0** |
| `0x200f0014` | VIU0_DAT13 | VIU0_DAT13 | GPIO0_5 | VOU0_DATA5 |  |  | **0** |
| `0x200f0018` | VIU0_DAT12 | VIU0_DAT12 | GPIO0_6 | VOU0_DATA4 |  |  | **0** |
| `0x200f001c` | VIU0_DAT11 | VIU0_DAT11 | GPIO0_7 | VOU0_DATA3 |  |  | **0** |
| `0x200f0020` | VIU0_DAT10 | VIU0_DAT10 | GPIO1_0 | VOU0_DATA2 |  |  | **0** |
| `0x200f0024` | VIU0_DAT9 | VIU0_DAT9 | GPIO1_1 | VOU0_DATA1 |  |  | **0** |
| `0x200f0028` | VIU0_DAT8 | VIU0_DAT8 | GPIO1_2 | VOU0_DATA0 |  |  | **0** |
| `0x200f002c` | VIU0_DAT7 | VIU0_DAT7 | GPIO1_3 | VOU1_DATA7 |  |  | **0** |
| `0x200f0030` | VIU0_DAT6 | VIU0_DAT6 | GPIO1_4 | VOU1_DATA6 |  |  | **0** |
| `0x200f0034` | VIU0_DAT5 | VIU0_DAT5 | GPIO1_5 | VOU1_DATA5 |  |  | **0** |
| `0x200f0038` | VIU0_DAT4 | VIU0_DAT4 | GPIO1_6 | VOU1_DATA4 |  |  | **0** |
| `0x200f003c` | VIU0_DAT3 | VIU0_DAT3 | GPIO1_7 | VOU1_DATA3 |  |  | **0** |
| `0x200f0040` | VIU0_DAT2 | VIU0_DAT2 | GPIO2_0 | VOU1_DATA2 |  |  | **0** |
| `0x200f0044` | VIU0_DAT1 | VIU0_DAT1 | GPIO2_1 | VOU1_DATA1 |  |  | **0** |
| `0x200f0048` | VIU0_DAT0 | VIU0_DAT0 | GPIO2_2 | VOU1_DATA0 |  |  | **0** |
| `0x200f004c` | VIU1_CLK | VIU1_CLK | GPIO2_3 | VOU2_CLK | VIU4_CLK |  | **0** |
| `0x200f0050` | VIU1_VS | VIU1_VS | GPIO2_4 | UART2_RXD |  |  | **2** |
| `0x200f0054` | VIU1_HS | VIU1_HS | GPIO2_5 | VOU3_CLK | VIU1_CLKA |  | **0** |
| `0x200f0058` | VIU1_DAT15 | VIU1_DAT15 | GPIO2_6 | VOU2_DATA7 | VIU4_DAT15 |  | **0** |
| `0x200f005c` | VIU1_DAT14 | VIU1_DAT14 | GPIO2_7 | VOU2_DATA6 | VIU4_DAT14 |  | **0** |
| `0x200f0060` | VIU1_DAT13 | VIU1_DAT13 | GPIO3_0 | VOU2_DATA5 | VIU4_DAT13 |  | **0** |
| `0x200f0064` | VIU1_DAT12 | VIU1_DAT12 | GPIO3_1 | VOU2_DATA4 | VIU4_DAT12 |  | **0** |
| `0x200f0068` | VIU1_DAT11 | VIU1_DAT11 | GPIO3_2 | VOU2_DATA3 | VIU4_DAT11 |  | **0** |
| `0x200f006c` | VIU1_DAT10 | VIU1_DAT10 | GPIO3_3 | VOU2_DATA2 | VIU4_DAT10 |  | **0** |
| `0x200f0070` | VIU1_DAT9 | VIU1_DAT9 | GPIO3_4 | VOU2_DATA1 | VIU4_DAT9 |  | **0** |
| `0x200f0074` | VIU1_DAT8 | VIU1_DAT8 | GPIO3_5 | VOU2_DATA0 | VIU4_DAT8 |  | **0** |
| `0x200f0078` | VIU1_DAT7 | VIU1_DAT7 | GPIO3_6 | VOU3_DATA7 | VIU4_DAT7 |  | **0** |
| `0x200f007c` | VIU1_DAT6 | VIU1_DAT6 | GPIO3_7 | VOU3_DATA6 | VIU4_DAT6 |  | **0** |
| `0x200f0080` | VIU1_DAT5 | VIU1_DAT5 | GPIO4_0 | VOU3_DATA5 | VIU4_DAT5 |  | **0** |
| `0x200f0084` | VIU1_DAT4 | VIU1_DAT4 | GPIO4_1 | VOU3_DATA4 | VIU4_DAT4 |  | **0** |
| `0x200f0088` | VIU1_DAT3 | VIU1_DAT3 | GPIO4_2 | VOU3_DATA3 | VIU4_DAT3 |  | **0** |
| `0x200f008c` | VIU1_DAT2 | VIU1_DAT2 | GPIO4_3 | VOU3_DATA2 | VIU4_DAT2 |  | **0** |
| `0x200f0090` | VIU1_DAT1 | VIU1_DAT1 | GPIO4_4 | VOU3_DATA1 | VIU4_DAT1 |  | **0** |
| `0x200f0094` | VIU1_DAT0 | VIU1_DAT0 | GPIO4_5 | VOU3_DATA0 | VIU4_DAT0 |  | **0** |
| `0x200f0098` | VIU2_CLK | VIU2_CLK | GPIO4_6 | VOU4_CLK |  |  | **0** |
| `0x200f009c` | VIU2_VS | VIU2_VS | GPIO4_7 | UART3_TXD |  |  | **0** |
| `0x200f00a0` | VIU2_HS | VIU2_HS | GPIO5_0 | VOU5_CLK | VIU2_CLKA |  | **0** |
| `0x200f00a4` | VIU2_DAT15 | VIU2_DAT15 | GPIO5_1 | VOU4_DATA7 |  |  | **0** |
| `0x200f00a8` | VIU2_DAT14 | VIU2_DAT14 | GPIO5_2 | VOU4_DATA6 |  |  | **0** |
| `0x200f00ac` | VIU2_DAT13 | VIU2_DAT13 | GPIO5_3 | VOU4_DATA5 |  |  | **0** |
| `0x200f00b0` | VIU2_DAT12 | VIU2_DAT12 | GPIO5_4 | VOU4_DATA4 |  |  | **0** |
| `0x200f00b4` | VIU2_DAT11 | VIU2_DAT11 | GPIO5_5 | VOU4_DATA3 |  |  | **0** |
| `0x200f00b8` | VIU2_DAT10 | VIU2_DAT10 | GPIO5_6 | VOU4_DATA2 |  |  | **0** |
| `0x200f00bc` | VIU2_DAT9 | VIU2_DAT9 | GPIO5_7 | VOU4_DATA1 |  |  | **0** |
| `0x200f00c0` | VIU2_DAT8 | VIU2_DAT8 | GPIO6_0 | VOU4_DATA0 |  |  | **0** |
| `0x200f00c4` | VIU2_DAT7 | VIU2_DAT7 | GPIO6_1 | VOU5_DATA7 |  |  | **0** |
| `0x200f00c8` | VIU2_DAT6 | VIU2_DAT6 | GPIO6_2 | VOU5_DATA6 |  |  | **0** |
| `0x200f00cc` | VIU2_DAT5 | VIU2_DAT5 | GPIO6_3 | VOU5_DATA5 |  |  | **0** |
| `0x200f00d0` | VIU2_DAT4 | VIU2_DAT4 | GPIO6_4 | VOU5_DATA4 |  |  | **0** |
| `0x200f00d4` | VIU2_DAT3 | VIU2_DAT3 | GPIO6_5 | VOU5_DATA3 |  |  | **0** |
| `0x200f00d8` | VIU2_DAT2 | VIU2_DAT2 | GPIO6_6 | VOU5_DATA2 |  |  | **0** |
| `0x200f00dc` | VIU2_DAT1 | VIU2_DAT1 | GPIO6_7 | VOU5_DATA1 |  |  | **0** |
| `0x200f00e0` | VIU2_DAT0 | VIU2_DAT0 | GPIO7_0 | VOU5_DATA0 |  |  | **0** |
| `0x200f00e4` | VGA_HS | GPIO7_1 | VGA_HS |  |  |  | **1** |
| `0x200f00e8` | VGA_VS | GPIO7_2 | VGA_VS |  |  |  | **1** |
| `0x200f00ec` | VOU1120_CLK | VIU3_CLK | GPIO7_3 | VOU6_CLK | VOU1120_CLK | SDIO_CCLK_OUT | **0** |
| `0x200f00f0` | VOU1120_VS | VIU3_VS | GPIO7_4 | UART3_RXD | VOU1120_VS | SDIO_CARD_POWER_EN | **0** |
| `0x200f00f4` | VOU1120_HS | VIU3_HS | GPIO7_5 | VOU7_CLK | VOU1120_HS | VIU3_CLKA | **0** |
| `0x200f00f8` | VOU1120_DATA15 | VIU3_DAT15 | GPIO7_6 | VOU6_DATA7 | VOU1120_DATA15 | SDIO_CARD_DETECT | **0** |
| `0x200f00fc` | VOU1120_DATA14 | VIU3_DAT14 | GPIO7_7 | VOU6_DATA6 | VOU1120_DATA14 | SDIO_CWPR | **0** |
| `0x200f0100` | VOU1120_DATA13 | VIU3_DAT13 | GPIO8_0 | VOU6_DATA5 | VOU1120_DATA13 | SDIO_CCMD | **0** |
| `0x200f0104` | VOU1120_DATA12 | VIU3_DAT12 | GPIO8_1 | VOU6_DATA4 | VOU1120_DATA12 | SDIO_CDATA0 | **0** |
| `0x200f0108` | VOU1120_DATA11 | VIU3_DAT11 | GPIO8_2 | VOU6_DATA3 | VOU1120_DATA11 | SDIO_CDATA1 | **0** |
| `0x200f010c` | VOU1120_DATA10 | VIU3_DAT10 | GPIO8_3 | VOU6_DATA2 | VOU1120_DATA10 | SDIO_CDATA2 | **0** |
| `0x200f0110` | VOU1120_DATA9 | VIU3_DAT9 | GPIO8_4 | VOU6_DATA1 | VOU1120_DATA9 | SDIO_CDATA3 | **0** |
| `0x200f0114` | VOU1120_DATA8 | VIU3_DAT8 | GPIO8_5 | VOU6_DATA0 | VOU1120_DATA8 |  | **0** |
| `0x200f0118` | VOU1120_DATA7 | VIU3_DAT7 | GPIO8_6 | VOU7_DATA7 | VOU1120_DATA7 |  | **0** |
| `0x200f011c` | VOU1120_DATA6 | VIU3_DAT6 | GPIO8_7 | VOU7_DATA6 | VOU1120_DATA6 |  | **0** |
| `0x200f0120` | VOU1120_DATA5 | VIU3_DAT5 | GPIO9_0 | VOU7_DATA5 | VOU1120_DATA5 |  | **0** |
| `0x200f0124` | VOU1120_DATA4 | VIU3_DAT4 | GPIO9_1 | VOU7_DATA4 | VOU1120_DATA4 |  | **0** |
| `0x200f0128` | VOU1120_DATA3 | VIU3_DAT3 | GPIO9_2 | VOU7_DATA3 | VOU1120_DATA3 |  | **0** |
| `0x200f012c` | VOU1120_DATA2 | VIU3_DAT2 | GPIO9_3 | VOU7_DATA2 | VOU1120_DATA2 |  | **0** |
| `0x200f0130` | VOU1120_DATA1 | VIU3_DAT1 | GPIO9_4 | VOU7_DATA1 | VOU1120_DATA1 |  | **0** |
| `0x200f0134` | VOU1120_DATA0 | VIU3_DAT0 | GPIO9_5 | VOU7_DATA0 | VOU1120_DATA0 |  | **0** |
| `0x200f0138` | SIO0_RCLK | GPIO9_6 | SIO0_RCLK |  |  |  | **0** |
| `0x200f013c` | SIO0_RFS | GPIO9_7 | SIO0_RFS |  |  |  | **0** |
| `0x200f0140` | SIO0_DIN | GPIO10_0 | SIO0_DIN |  |  |  | **1** |
| `0x200f0144` | SIO1_RCLK | GPIO10_1 | SIO1_RCLK |  |  |  | **1** |
| `0x200f0148` | SIO1_RFS | GPIO10_2 | SIO1_RFS |  |  |  | **1** |
| `0x200f014c` | SIO1_DIN | GPIO10_3 | SIO1_DIN |  |  |  | **1** |
| `0x200f0150` | SIO2_RCLK | GPIO10_4 | SIO2_RCLK |  |  |  | **1** |
| `0x200f0154` | SIO2_RFS | GPIO10_5 | SIO2_RFS |  |  |  | **1** |
| `0x200f0158` | SIO2_DIN | GPIO10_6 | SIO2_DIN |  |  |  | **1** |
| `0x200f015c` | SIO3_RCLK | GPIO10_7 | SIO3_RCLK |  |  |  | **1** |
| `0x200f0160` | SIO3_RFS | GPIO11_0 | SIO3_RFS |  |  |  | **1** |
| `0x200f0164` | SIO3_DIN | GPIO11_1 | SIO3_DIN |  |  |  | **1** |
| `0x200f0168` | SIO4_XCLK | GPIO11_2 | SIO4_XCLK |  |  |  | **1** |
| `0x200f016c` | SIO4_XFS | GPIO11_3 | SIO4_XFS |  |  |  | **1** |
| `0x200f0170` | SIO4_RCLK | GPIO11_4 | SIO4_RCLK |  |  |  | **1** |
| `0x200f0174` | SIO4_RFS | GPIO11_5 | SIO4_RFS |  |  |  | **1** |
| `0x200f0178` | SIO4_DOUT | GPIO11_6 | SIO4_DOUT |  |  |  | **1** |
| `0x200f017c` | SIO4_DIN | GPIO11_7 | SIO4_DIN |  |  |  | **1** |
| `0x200f0180` | SPI_SCLK | GPIO12_0 | SPI_SCLK |  |  |  | **1** |
| `0x200f0184` | SPI_SDO | GPIO12_1 | SPI_SDO |  |  |  | **1** |
| `0x200f0188` | SPI_SDI | GPIO12_2 | SPI_SDI |  |  |  | **1** |
| `0x200f018c` | SPI_CSN0 | GPIO12_3 | SPI_CSN0 |  |  |  | **1** |
| `0x200f0190` | SPI_CSN6 | SPI_CSN6 | NF_ECC_TYPE1 |  |  |  | **0** |
| `0x200f0194` | SPI_CSN7 \* | SPI_CSN7 | NF_ECC_TYPE2 | CLK_TEST_OUT0 | CLK_TEST_OUT1 | CLK_TEST_OUT2 | **0** |
| `0x200f0198` | I2C_SDA | GPIO12_4 | I2C_SDA |  |  |  | **0** |
| `0x200f019c` | I2C_SCL | GPIO12_5 | I2C_SCL |  |  |  | **0** |
| `0x200f01a0` | UART1_RTSN | GPIO12_6 | UART1_RTSN |  |  |  | **0** |
| `0x200f01a4` | UART1_RXD | GPIO12_7 | UART1_RXD |  |  |  | **1** |
| `0x200f01a8` | UART1_TXD | GPIO13_0 | UART1_TXD |  |  |  | **1** |
| `0x200f01ac` | UART1_CTSN | GPIO13_1 | UART1_CTSN |  |  |  | **0** |
| `0x200f01b0` | RGMII0_TXCK | RGMII0_TXCK | GPIO13_2 |  |  |  | **0** |
| `0x200f01b4` | RGMII0_CRS | GPIO13_3 | RGMII0_CRS | RGMII0_RXER |  |  | **2** |
| `0x200f01b8` | RGMII0_COL | GPIO13_4 | RGMII0_COL | RGMII0_TXER |  |  | **2** |
| `0x200f01bc` | RGMII1_RXDV | GPIO13_5 | RGMII1_RXDV |  |  |  | **1** |
| `0x200f01c0` | RGMII1_RXD3 | GPIO13_6 | RGMII1_RXD3 |  |  |  | **1** |
| `0x200f01c4` | RGMII1_RXD2 | GPIO13_7 | RGMII1_RXD2 |  |  |  | **1** |
| `0x200f01c8` | RGMII1_RXD1 | GPIO14_0 | RGMII1_RXD1 |  |  |  | **1** |
| `0x200f01cc` | RGMII1_RXD0 | GPIO14_1 | RGMII1_RXD0 |  |  |  | **1** |
| `0x200f01d0` | RGMII1_RXCK | GPIO14_2 | RGMII1_RXCK |  |  |  | **1** |
| `0x200f01d4` | RGMII1_TXEN | GPIO14_3 | RGMII1_TXEN |  |  |  | **1** |
| `0x200f01d8` | RGMII1_TXD3 | GPIO14_4 | RGMII1_TXD3 |  |  |  | **1** |
| `0x200f01dc` | RGMII1_TXD2 | GPIO14_5 | RGMII1_TXD2 |  |  |  | **1** |
| `0x200f01e0` | RGMII1_TXD1 | GPIO14_6 | RGMII1_TXD1 |  |  |  | **1** |
| `0x200f01e4` | RGMII1_TXD0 \* | PLL_TEST_OUT0 | RGMII1_TXD0 | PLL_TEST_OUT1 | PLL_TEST_OUT2 | PLL_TEST_OUT3 | **1** |
| `0x200f01e8` | RGMII1_TXCK | GPIO15_0 | RGMII1_TXCK |  |  |  | **1** |
| `0x200f01ec` | RGMII1_TXCKOUT | GPIO15_1 | RGMII1_TXCKOUT |  |  |  | **1** |
| `0x200f01f0` | RGMII1_CRS | GPIO15_2 | RGMII1_CRS | RGMII1_RXER |  |  | **2** |
| `0x200f01f4` | RGMII1_COL | GPIO15_3 | RGMII1_COL | RGMII1_TXER |  |  | **2** |
| `0x200f01f8` | IR_IN | IR_IN | GPIO15_4 |  |  |  | **0** |
| `0x200f01fc` | NF_DQ0 | NF_DQ0 | GPIO15_5 |  |  |  | **0** |
| `0x200f0200` | NF_DQ1 | NF_DQ1 | GPIO15_6 |  |  |  | **0** |
| `0x200f0204` | NF_DQ2 | NF_DQ2 | GPIO15_7 |  |  |  | **0** |
| `0x200f0208` | NF_DQ3 | NF_DQ3 | GPIO16_0 |  |  |  | **0** |
| `0x200f020c` | NF_DQ4 | NF_DQ4 | GPIO16_1 |  |  |  | **0** |
| `0x200f0210` | NF_DQ5 | NF_DQ5 | GPIO16_2 |  |  |  | **0** |
| `0x200f0214` | NF_DQ6 | NF_DQ6 | GPIO16_3 |  |  |  | **0** |
| `0x200f0218` | NF_DQ7 | NF_DQ7 | GPIO16_4 |  |  |  | **0** |
| `0x200f021c` | NF_RDY0 | NF_RDY0 | GPIO16_5 |  |  |  | **0** |
| `0x200f0220` | NF_RDY1 | NF_RDY1 | GPIO16_6 |  |  |  | **0** |
| `0x200f0224` | SFC_DIO | SFC_DIO | GPIO16_7 |  |  |  | **0** |
| `0x200f0228` | SFC_WP_IO2 | SFC_WP_IO2 | GPIO17_0 |  |  |  | **0** |
| `0x200f022c` | SFC_DOI | SFC_DOI | GPIO17_1 |  |  |  | **0** |
| `0x200f0230` | SFC_HOLD_IO3 | SFC_HOLD_IO3 | GPIO17_2 |  |  |  | **0** |
| `0x200f0234` | USB0_OVRCUR | GPIO17_3 | USB0_OVRCUR |  |  |  | **0** |
| `0x200f0238` | USB0_PWREN | GPIO17_4 | USB0_PWREN |  |  |  | **0** |
| `0x200f023c` | USB1_OVRCUR | GPIO17_5 | USB1_OVRCUR |  |  |  | **0** |
| `0x200f0240` | USB1_PWREN | GPIO17_6 | USB1_PWREN |  |  |  | **1** |
| `0x200f0244` | HDMI_HOTPLUG | GPIO17_7 | HDMI_HOTPLUG |  |  |  | **1** |
| `0x200f0248` | HDMI_CEC | GPIO18_0 | HDMI_CEC |  |  |  | **1** |
| `0x200f024c` | HDMI_SDA | GPIO18_1 | HDMI_SDA |  |  |  | **1** |
| `0x200f0250` | HDMI_SCL | GPIO18_2 | HDMI_SCL |  |  |  | **1** |
| `0x200f0254` | GPIO18_3 | GPIO18_3 | SATA_LED_N0 |  |  |  | **0** |
| `0x200f0258` | GPIO18_4 | GPIO18_4 | SATA_LED_N1 |  |  |  | **0** |

\* Two registers define functions past 4: `0x194` adds 5 = `CLK_TEST_OUT3`, and
`0x1e4` adds 6 = `PLL_TEST_OUT4`. Both are test outputs of no interest to a
port.

## Notes for a port

- Nothing here needs a pinctrl driver. The pins a server build cares about are
  already correct at reset or set by U-Boot: RGMII1, the NAND and SPI-NOR
  interfaces, the IR input and the I²C GPIOs. UART2, UART1 and the audio SIO
  pins are the only ones Linux changes, and a port can write those four groups
  directly.
- The `stmmac` glue claims `0x200F0000`–`0x200F01FF` and rewrites pins at probe,
  so a mainline `syscon` node for this block should expect to share it.
- The datasheet's section 2.1.6 and 2.1.7 give the same information organised by
  peripheral (software-multiplexed and hardware-multiplexed pins), which is
  easier to read when chasing one interface.
