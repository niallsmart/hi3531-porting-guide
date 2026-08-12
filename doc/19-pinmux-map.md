# Pin Multiplexing Map

The complete IO_CONFIG function map for this board: which physical pin each
mux register controls, and what each function value selects.

## Where this comes from

The vendor filesystem contains a set of shell scripts that program the SoC's
IO_CONFIG block at boot:

```
rootfs/mtd/modules/pinctrl_16D1_hi3531.sh
rootfs/mtd/modules/pinctrl_16X960H_hi3531.sh
rootfs/mtd/modules/pinctrl_4HD_hi3531.sh        <- the variant this board uses
rootfs/mtd/modules/pinctrl_8X720P_hi3531.sh
rootfs/mtd/modules/pinctrl_8BT656_hi3531.sh
rootfs/mtd/modules/pinctrl_1HD+12D1_hi3531.sh
rootfs/mtd/modules/pinctrl_cas_hi3531.sh
rootfs/mtd/modules/pinctrl_cas_slave_hi3531.sh
```

Every line has the form:

```sh
himm 0x200f00ec 0x00000000 #VIU3_CLK / GPIO7_3 / VOU6_CLK / VOU1120_CLK / SDIO_CCLK_OUT
```

The comment lists the pin's alternate functions **in function-value order**, so
value 0 selects `VIU3_CLK`, value 1 selects `GPIO7_3`, value 3 selects
`VOU1120_CLK`, and so on. Together the scripts document **108 IO_CONFIG
registers**, which is enough to decode the live register dump in
[17-register-dumps.md](17-register-dumps.md) without needing the datasheet.

## Which variant this board uses

`rootfs/mtd/dep2.sh` selects the variant from the product type and calls:

```sh
cd /mnt/mtd/modules && ./load3531 -i 4hd      # 4-channel HD
```

`load3531` then runs `pinctrl_4HD_hi3531.sh`. This is consistent with the
runtime video configuration (four BT.1120 inputs at 1920x1080) and with the
product code `2704XD_P`.

The "4hd sets" column below gives the value that script writes. A dash means
the script does not touch that register, leaving whatever the bootloader set.

## Confirmed peripheral assignments

These are the ones a porter actually needs.

| Function | Pins | Registers | Value |
|---|---|---|---|
| **Bit-banged I²C SDA** | **GPIO12_4** | `0x200f0198` | 0 (GPIO) |
| **Bit-banged I²C SCL** | **GPIO12_5** | `0x200f019c` | 0 (GPIO) |
| SPI (`ssp.ko`) | GPIO12_0..12_3 | `0x200f0180`–`0x18c` | 1 |
| Audio I²S (SIO4) | GPIO11_4..11_7 | `0x200f0170`–`0x17c` | 1 |
| UART1 | GPIO12_7 (RXD), GPIO13_0 (TXD) | `0x200f01a4`, `0x1a8` | 1 |
| UART2 | GPIO2_4 (RXD), GPIO0_1 (TXD) | `0x200f0050`, `0x0004` | 2 |
| HDMI hotplug | GPIO17_7 | `0x200f0244` | 1 |
| HDMI CEC | GPIO18_0 | `0x200f0248` | 1 |
| HDMI DDC SDA | GPIO18_1 | `0x200f024c` | 1 |
| HDMI DDC SCL | GPIO18_2 | `0x200f0250` | 1 |
| Buzzer control | GPIO2_3 | `0x200f004c` | 1 in U-Boot; `dep2.sh` writes 0 |
| VGA sync | VGA_HS, VGA_VS | `0x200f00e4`, `0x0e8` | 1 |
| Video input | VIU0/VIU1/VIU2 buses | `0x200f0000`–`0x00e0` | 0 |
| BT.1120 video **output** | VOU1120 bus, 19 pins | `0x200f00ec`–`0x0134` | 3 |

### The I²C pins

**SDA is GPIO12_4 and SCL is GPIO12_5**, matching the SDK reference comment in
`mpp/extdrv/gpio_i2c_8b/gpio_i2c.c`.

Three independent things confirm it:

1. The pinctrl scripts name `0x200f0198` as `GPIO12_4 / I2C_SDA` and
   `0x200f019c` as `.../I2C_SCL`.
2. The live U-Boot dump reads **0** at both registers — GPIO function, not the
   hardware I²C controller. That is exactly what a bit-banged driver requires.
3. The `himm` lines that would switch them are **commented out in every
   pinctrl variant**, so they stay GPIO under Linux too.

Note the vendor's comment on `0x200f019c` says `GPIO12_4` — a copy-paste typo.
The register is one word further along and belongs to bit 5, and the SDK source
comment independently states `I2C_SCL -- GPIO12_5`.

With this, mainline `i2c-gpio` can replace the entire vendor `gpioi2c` stack:

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

## Caveats

- The scripts are the *vendor's* description of the hardware, not the
  datasheet. A few comments are internally inconsistent — `GPIO2_3` and
  `GPIO4_3` each appear against two different registers, and the `I2C_SCL`
  line has the typo noted above. Treat individual GPIO bit numbers with mild
  suspicion; the peripheral names are reliable.
- Registers `0x200f01ac`–`0x200f0240` are **not documented** by any script.
  The live dump shows a 13-pin run of function 1 at `0x200f01bc`–`0x1ec` and
  function 2 at `0x1b4`/`0x1b8` and `0x1f0`/`0x1f4`. Ethernet RGMII is the
  obvious candidate for a 13-pin run, but this is unconfirmed.
- The live dump was taken from U-Boot, before Linux runs the pinctrl script.
  Some pins differ afterwards — SPI, for instance, reads 0 in U-Boot and is set
  to 1 by the script.

## Full register map

Functions are listed by value: column "0" is what the pin does when the
register reads 0, and so on.

| Register | 0 | 1 | 2 | 3 | 4 | 4hd sets |
|---|---|---|---|---|---|---|
| `0x200f0000` | VIU0_CLK | GPIO0_3 | VOU0_CLK |  |  | — |
| `0x200f0004` | VIU0_VS | GPIO0_4 | UART2_TXD |  |  | 2 |
| `0x200f0008` | VIU0_HS | GPIO0_2 | VOU1_CLK | VIU0_CLKA |  | — |
| `0x200f000c` | VIU0_DAT15 | GPIO0_6 | VOU0_DAT7 |  |  | — |
| `0x200f0010` | VIU0_DAT14 | GPIO0_7 | VOU0_DAT6 |  |  | — |
| `0x200f0014` | VIU0_DAT13 | GPIO1_0 | VOU0_DAT5 |  |  | — |
| `0x200f0018` | VIU0_DAT12 | GPIO1_1 | VOU0_DAT4 |  |  | — |
| `0x200f001c` | VIU0_DAT11 | GPIO1_2 | VOU0_DAT3 |  |  | — |
| `0x200f0020` | VIU0_DAT10 | GPIO1_3 | VOU0_DAT2 |  |  | — |
| `0x200f0024` | VIU0_DAT9 | GPIO1_4 | VOU0_DAT1 |  |  | — |
| `0x200f0028` | VIU0_DAT8 | GPIO1_5 | VOU0_DAT0 |  |  | — |
| `0x200f002c` | VIU0_DAT7 | GPIO1_6 | VOU1_DAT7 |  |  | — |
| `0x200f0030` | VIU0_DAT6 | GPIO1_7 | VOU1_DAT6 |  |  | — |
| `0x200f0034` | VIU0_DAT5 | GPIO2_0 | VOU1_DAT5 |  |  | — |
| `0x200f0038` | VIU0_DAT4 | GPIO2_1 | VOU1_DAT4 |  |  | — |
| `0x200f003c` | VIU0_DAT3 | GPIO2_2 | VOU1_DAT3 |  |  | — |
| `0x200f0040` | VIU0_DAT2 | GPIO2_3 | VOU1_DAT2 |  |  | — |
| `0x200f0044` | VIU0_DAT1 | GPIO2_4 | VOU1_DAT1 |  |  | — |
| `0x200f0048` | VIU0_DAT0 | GPIO2_5 | VOU1_DAT0 |  |  | — |
| `0x200f004c` | VIU1_CLK | GPIO2_3 | VOU2_CLK | VIU4_CLK |  | — |
| `0x200f0050` | VIU1_VS | GPIO2_4 | UART2_RXD |  |  | 2 |
| `0x200f0054` | VIU1_HS | GPIO2_5 | VOU3_CLK | VIU1_CLKA |  | — |
| `0x200f0058` | VIU1_DAT15 | GPIO2_6 | VOU2_DAT7 | VIU4_DAT15 |  | — |
| `0x200f005c` | VIU1_DAT14 | GPIO2_7 | VOU2_DAT6 | VIU4_DAT14 |  | — |
| `0x200f0060` | VIU1_DAT13 | GPIO3_0 | VOU2_DAT5 | VIU4_DAT13 |  | — |
| `0x200f0064` | VIU1_DAT12 | GPIO3_1 | VOU2_DAT4 | VIU4_DAT12 |  | — |
| `0x200f0068` | VIU1_DAT11 | GPIO3_2 | VOU2_DAT3 | VIU4_DAT11 |  | — |
| `0x200f006c` | VIU1_DAT10 | GPIO3_3 | VOU2_DAT2 | VIU4_DAT10 |  | — |
| `0x200f0070` | VIU1_DAT9 | GPIO3_4 | VOU2_DAT1 | VIU4_DAT9 |  | — |
| `0x200f0074` | VIU1_DAT8 | GPIO3_5 | VOU2_DAT0 | VIU4_DAT8 |  | — |
| `0x200f0078` | VIU1_DAT7 | GPIO3_6 | VOU3_DAT7 | VIU4_DAT7 |  | — |
| `0x200f007c` | VIU1_DAT6 | GPIO3_7 | VOU3_DAT6 | VIU4_DAT6 |  | — |
| `0x200f0080` | VIU1_DAT5 | GPIO4_0 | VOU3_DAT5 | VIU4_DAT5 |  | — |
| `0x200f0084` | VIU1_DAT4 | GPIO4_1 | VOU3_DAT4 | VIU4_DAT4 |  | — |
| `0x200f0088` | VIU1_DAT3 | GPIO4_2 | VOU3_DAT3 | VIU4_DAT3 |  | — |
| `0x200f008c` | VIU1_DAT2 | GPIO4_3 | VOU3_DAT2 | VIU4_DAT2 |  | — |
| `0x200f0090` | VIU1_DAT1 | GPIO4_4 | VOU3_DAT1 | VIU4_DAT1 |  | — |
| `0x200f0094` | VIU1_DAT0 | GPIO4_5 | VOU3_DAT0 | VIU4_DAT0 |  | — |
| `0x200f0098` | VIU2_CLK | GPIO4_3 | VOU4_CLK |  |  | — |
| `0x200f009c` | VIU2_VS | GPIO4_4 | UART3_TXD |  |  | — |
| `0x200f00a0` | VIU2_HS | GPIO5_0 | VOU5_CLK | VIU2_CLKA |  | — |
| `0x200f00a4` | VIU2_DAT15 | GPIO4_6 | VOU4_DAT7 |  |  | — |
| `0x200f00a8` | VIU2_DAT14 | GPIO4_7 | VOU4_DAT6 |  |  | — |
| `0x200f00ac` | VIU2_DAT13 | GPIO5_0 | VOU4_DAT5 |  |  | — |
| `0x200f00b0` | VIU2_DAT12 | GPIO5_1 | VOU4_DAT4 |  |  | — |
| `0x200f00b4` | VIU2_DAT11 | GPIO5_2 | VOU4_DAT3 |  |  | — |
| `0x200f00b8` | VIU2_DAT10 | GPIO5_3 | VOU4_DAT2 |  |  | — |
| `0x200f00bc` | VIU2_DAT9 | GPIO5_4 | VOU4_DAT1 |  |  | — |
| `0x200f00c0` | VIU2_DAT8 | GPIO5_5 | VOU4_DAT0 |  |  | — |
| `0x200f00c4` | VIU2_DAT7 | GPIO5_6 | VOU5_DAT7 |  |  | — |
| `0x200f00c8` | VIU2_DAT6 | GPIO5_7 | VOU5_DAT6 |  |  | — |
| `0x200f00cc` | VIU2_DAT5 | GPIO6_0 | VOU5_DAT5 |  |  | — |
| `0x200f00d0` | VIU2_DAT4 | GPIO6_1 | VOU5_DAT4 |  |  | — |
| `0x200f00d4` | VIU2_DAT3 | GPIO6_2 | VOU5_DAT3 |  |  | — |
| `0x200f00d8` | VIU2_DAT2 | GPIO6_3 | VOU5_DAT2 |  |  | — |
| `0x200f00dc` | VIU2_DAT1 | GPIO6_4 | VOU5_DAT1 |  |  | — |
| `0x200f00e0` | VIU2_DAT0 | GPIO6_5 | VOU5_DAT0 |  |  | — |
| `0x200f00e4` | VGA_HS |  |  |  |  | 1 |
| `0x200f00e8` | VGA_VS |  |  |  |  | 1 |
| `0x200f00ec` | VIU3_CLK | GPIO7_3 | VOU6_CLK | VOU1120_CLK | SDIO_CCLK_OUT | 0 |
| `0x200f00f0` | VIU3_VS | GPIO7_4 | UART3_RXD | VOU1120_VS | SDIOxx | 0 |
| `0x200f00f4` | VIU3_HS | GPIO7_5 | VOU7_CLK | VOU1120_HS | VIU3_CLKA | 0 |
| `0x200f00f8` | VIU3_DAT15 | GPIO7_6 | VOU6_DAT7 | VOU1120_DAT15 | SDIOxx | 0 |
| `0x200f00fc` | VIU3_DAT14 | GPIO7_7 | VOU6_DAT6 | VOU1120_DAT14 | SDIOxx | 0 |
| `0x200f0100` | VIU3_DAT13 | GPIO8_0 | VOU6_DAT5 | VOU1120_DAT13 | SDIOxx | 0 |
| `0x200f0104` | VIU3_DAT12 | GPIO8_1 | VOU6_DAT4 | VOU1120_DAT12 | SDIOxx | 0 |
| `0x200f0108` | VIU3_DAT11 | GPIO8_2 | VOU6_DAT3 | VOU1120_DAT11 | SDIOxx | 0 |
| `0x200f010c` | VIU3_DAT10 | GPIO8_3 | VOU6_DAT2 | VOU1120_DAT10 | SDIOxx | 0 |
| `0x200f0110` | VIU3_DAT9 | GPIO8_4 | VOU6_DAT1 | VOU1120_DAT9 | SDIOxx | 0 |
| `0x200f0114` | VIU3_DAT8 | GPIO8_5 | VOU6_DAT0 | VOU1120_DAT8 |  | 0 |
| `0x200f0118` | VIU3_DAT7 | GPIO8_6 | VOU7_DAT7 | VOU1120_DAT7 |  | 0 |
| `0x200f011c` | VIU3_DAT6 | GPIO8_7 | VOU7_DAT6 | VOU1120_DAT6 |  | 0 |
| `0x200f0120` | VIU3_DAT5 | GPIO9_0 | VOU7_DAT5 | VOU1120_DAT5 |  | 0 |
| `0x200f0124` | VIU3_DAT4 | GPIO9_1 | VOU7_DAT4 | VOU1120_DAT4 |  | 0 |
| `0x200f0128` | VIU3_DAT3 | GPIO9_2 | VOU7_DAT3 | VOU1120_DAT3 |  | 0 |
| `0x200f012c` | VIU3_DAT2 | GPIO9_3 | VOU7_DAT2 | VOU1120_DAT2 |  | 0 |
| `0x200f0130` | VIU3_DAT1 | GPIO9_4 | VOU7_DAT1 | VOU1120_DAT1 |  | 0 |
| `0x200f0134` | VIU3_DAT0 | GPIO9_5 | VOU7_DAT0 | VOU1120_DAT0 |  | 0 |
| `0x200f0138` | GPIO9_6 | SIO0_RCLK |  |  |  | 1 |
| `0x200f013c` | GPIO9_7 | SIO0_RFS |  |  |  | 1 |
| `0x200f0140` | GPIO10_0 | SIO0_DIN |  |  |  | 1 |
| `0x200f0144` | GPIO10_1 | SIO1_RCLK |  |  |  | 1 |
| `0x200f0148` | GPIO10_2 | SIO1_RFS |  |  |  | 1 |
| `0x200f014c` | GPIO10_3 | SIO1_DIN |  |  |  | 1 |
| `0x200f0150` | GPIO10_4 | SIO2_RCLK |  |  |  | 1 |
| `0x200f0154` | GPIO10_5 | SIO2_RFS |  |  |  | 1 |
| `0x200f0158` | GPIO10_6 | SIO2_DIN |  |  |  | 1 |
| `0x200f015c` | GPIO10_7 | SIO3_RCLK |  |  |  | 1 |
| `0x200f0160` | GPIO11_0 | SIO3_RFS |  |  |  | 1 |
| `0x200f0164` | GPIO11_1 | SIO3_DIN |  |  |  | 1 |
| `0x200f0168` | GPIO11_2 | SIO4_XCLK |  |  |  | 1 |
| `0x200f016c` | GPIO11_3 | SIO4_XFS |  |  |  | 1 |
| `0x200f0170` | GPIO11_4 | SIO4_RCLK |  |  |  | 1 |
| `0x200f0174` | GPIO11_5 | SIO4_RFS |  |  |  | 1 |
| `0x200f0178` | GPIO11_6 | SIO4_DOUT |  |  |  | 1 |
| `0x200f017c` | GPIO11_7 | SIO4_DIN |  |  |  | 1 |
| `0x200f0180` | GPIO12_0 | SPI_SCLK |  |  |  | 1 |
| `0x200f0184` | GPIO12_1 | SPI_SDO |  |  |  | 1 |
| `0x200f0188` | GPIO12_2 | SPI_SDI |  |  |  | 1 |
| `0x200f018c` | GPIO12_3 | SPI_CSN0 |  |  |  | 1 |
| `0x200f0198` | GPIO12_4 | I2C_SDA |  |  |  | — |
| `0x200f019c` | GPIO12_4 | I2C_SCL |  |  |  | — |
| `0x200f01a4` | GPIO12_7 | UART1_RXD |  |  |  | 1 |
| `0x200f01a8` | GPIO13_0 | UART1_TXD |  |  |  | 1 |
| `0x200f0244` | GPIO17_7 | HDMI_HOTPLUG |  |  |  | 1 |
| `0x200f0248` | GPIO18_0 | HDMI_CEC |  |  |  | 1 |
| `0x200f024c` | GPIO18_1 | HDMI_SDA |  |  |  | 1 |
| `0x200f0250` | GPIO18_2 | HDMI_SCL |  |  |  | 1 |
