# USB

Two USB 2.0 host controllers — an EHCI for high-speed and a companion OHCI for
full/low-speed. Both are HiSilicon wrappers around standard IP.

## Controllers

| Property | EHCI | OHCI |
|---|---|---|
| Platform device | `hiusb-ehci.0` | `hiusb-ohci.0` |
| Register base | `0x100B0000`–`0x100BFFFF` | `0x100A0000`–`0x100AFFFF` |
| IRQ | 63 | 64 |
| USB bus number | 1 | 2 |
| Root hub ports | 2 | 2 |
| Driver | `ehci_hcd` | `ohci_hcd` |

Boot log:

```
ehci_hcd: USB 2.0 'Enhanced' Host Controller (EHCI) Driver
hiusb-ehci hiusb-ehci.0: HIUSB EHCI
hiusb-ehci hiusb-ehci.0: new USB bus registered, assigned bus number 1
hiusb-ehci hiusb-ehci.0: irq 63, io mem 0x100b0000
hiusb-ehci hiusb-ehci.0: USB 0.0 started, EHCI 1.00
hub 1-0:1.0: USB hub found / 2 ports detected

ohci_hcd: USB 1.1 'Open' Host Controller (OHCI) Driver
hiusb-ohci hiusb-ohci.0: HIUSB OHCI
hiusb-ohci hiusb-ohci.0: new USB bus registered, assigned bus number 2
hiusb-ohci hiusb-ohci.0: irq 64, io mem 0x100a0000
hub 2-0:1.0: USB hub found / 2 ports detected
```

The two controllers share the same two physical ports in the usual
EHCI/companion arrangement: a high-speed device binds to EHCI, a
full/low-speed device is handed off to OHCI.

U-Boot also initialises USB and reports `1 USB Device(s) found`,
`0 Storage Device(s)`. **That device is the root hub itself, not anything
attached.** Running `usb tree` at the U-Boot prompt gives:

```
Device Tree:
  1  Hub (12 Mb/s, 0mA)
      OHCI Root Hub
```

Nothing external was connected. Nothing was enumerated under Linux either.

### Over-current and port-power pins

Both ports have dedicated over-current and power-enable pins, each multiplexed
against a GPIO:

| Register | Pin | 0 | 1 | This board |
|---|---|---|---|---|
| `0x200f0234` | USB0_OVRCUR | GPIO17_3 | USB0_OVRCUR | 0 |
| `0x200f0238` | USB0_PWREN | GPIO17_4 | USB0_PWREN | 0 |
| `0x200f023c` | USB1_OVRCUR | GPIO17_5 | USB1_OVRCUR | 0 |
| `0x200f0240` | USB1_PWREN | GPIO17_6 | USB1_PWREN | 1 |

Only `USB1_PWREN` is routed to the controller; the other three are left as
GPIO. So port power for USB1 is under hardware control and the rest is either
hard-wired on the board or handled in software through those GPIOs. Nothing
here blocks a port — the EHCI and OHCI drivers work without them — but a
mainline tree wanting per-port power control has the pins available. See
[19-pinmux-map.md](19-pinmux-map.md).

## Class drivers in the vendor kernel

The vendor kernel is built with a broad set of USB class drivers, which is a
useful signal about what the DVR was expected to work with:

- **Mass storage**: `usb-storage` plus `ums-alauda`, `ums-datafab`,
  `ums-freecom`, `ums-isd200`, `ums-jumpshot`, `ums-sddr09`, `ums-sddr55`
- **Serial**: `usbserial`, `pl2303`, `option` (GSM modems), `cdc_acm`, `cdc_wdm`
- **HID**: `usbhid`, `mousedev` (PS/2 mouse emulation)
- **Other**: `mdc800` (Mustek digital camera)

USB mass storage and HID are how the DVR supports firmware updates from a USB
stick and a USB mouse for its on-screen interface.

`usbmon` is compiled in but unusable — `usbmon: debugfs is not available`.

## Mainline support

Mainline's `ehci-platform` and `ohci-platform` drivers handle the two host
controllers without Hi3531-specific host glue. The SoC-specific work belongs
in one shared PHY provider referenced by both controller nodes.

```dts
usbphy: usb-phy@20030000 {
    compatible = "hisilicon,hi3531-usb-phy";
    reg = <0x20030000 0x100>,
          <0x20050000 0x100>;
    reg-names = "crg", "sysctrl";
    #phy-cells = <0>;
};

usb_ehci: usb@100b0000 {
    compatible = "generic-ehci";
    reg = <0x100b0000 0x10000>;
    interrupts = <0 31 4>;          /* SPI 31, level-high — vendor IRQ 63 */
    phys = <&usbphy>;
    phy-names = "usb";
};

usb_ohci: usb@100a0000 {
    compatible = "generic-ohci";
    reg = <0x100a0000 0x10000>;
    interrupts = <0 32 4>;          /* SPI 32, level-high — vendor IRQ 64 */
    phys = <&usbphy>;
    phy-names = "usb";
};
```

The PHY provider implements the vendor 3.0.8 `hiusb_start_hcd()` sequence. It
enables the USB clock and releases the resets at `CRG + 0xB8`, then configures
the 8-bit UTMI interface and ULPI bypass at `SYS_CTRL + 0x80`. The generic PHY
core refcounts the shared provider across EHCI and OHCI, replacing the vendor
wrapper's manual open count.

The reconciled Linux 6.18.42 port validated this arrangement. Both two-port
root hubs enumerated. A high-speed Flash Voyager sustained about 33 MB/s in
both the front and rear sockets, and a full-speed FTDI serial adapter exercised
the OHCI companion path. The flash drive also completed 512 MiB of dispersed
write/readback testing without a reset, transport error or I/O error. No
`hisilicon,hi3531-ehci` or `hisilicon,hi3531-ohci` compatible is needed.

The source for the sequence is:

- SDK kernel source: `osdrv/kernel/linux-3.0.y/drivers/usb/host/` — look for
  `hiusb` platform glue
- The Hi3531 datasheet's USB and CRG chapters

Mainline's `phy-hisi-inno-usb2` and similar drivers cover later HiSilicon
parts, but none targets this Hi3531 register layout.

## Assessment

Validated. The host controllers are standard; only the shared PHY bring-up
needs local code. USB mass storage is loaded during normal Buildroot startup.
