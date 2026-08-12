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

Straightforward. Mainline has `ehci-platform` and `ohci-platform`
(`drivers/usb/host/`), which handle generic memory-mapped controllers.

```dts
usb_ehci: usb@100b0000 {
    compatible = "hisilicon,hi3531-ehci", "generic-ehci";
    reg = <0x100b0000 0x10000>;
    interrupts = <0 63 4>;          /* verify SPI numbering */
};

usb_ohci: usb@100a0000 {
    compatible = "hisilicon,hi3531-ohci", "generic-ohci";
    reg = <0x100a0000 0x10000>;
    interrupts = <0 64 4>;          /* verify SPI numbering */
};
```

The likely complication is the **USB PHY**. HiSilicon SoCs typically require a
vendor-specific PHY init sequence — clock enables, resets, and PHY tuning
values written through CRG — before the controllers respond. The `hiusb-`
prefix on the platform device names indicates exactly such a wrapper.

Where to find the sequence:

- SDK kernel source: `osdrv/kernel/linux-3.0.y/drivers/usb/host/` — look for
  `hiusb` platform glue
- The Hi3531 datasheet's USB and CRG chapters

Mainline's `phy-hisi-inno-usb2` and similar drivers cover later HiSilicon
parts and may be a useful template, but none targets Hi3531 specifically.

## Assessment

Low-to-moderate risk. The host controllers themselves are standard; the work is
in the PHY bring-up glue. For a server, USB is useful but not on the critical
path — Ethernet and SATA matter more. Defer it until after the system boots.
