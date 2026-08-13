# Ethernet

Gigabit Ethernet via a Synopsys DesignWare MAC and a Realtek PHY. This is one
of the best-supported subsystems on the board — `stmmac` is mainline and mature.

## MAC

| Property | Value |
|---|---|
| IP | Synopsys DesignWare MAC 1000 (DWMAC1000) |
| Synopsys ID | `0x36` (version 3.6) |
| User ID | `0x10` |
| Descriptors | Enhanced descriptor structure |
| Register base | `0x101C0000`, region `0x101C0000`–`0x101DFFFF` |
| IRQ | 119 (shared by both MAC instances) |
| Driver | `stmmac`, vendor version `201206191703` |
| Checksum offload | Rx Checksum Offload Engine supported |
| Platform device | `stmmaceth.0` |

**Two MAC instances are present.** From the boot log, their mapped bases differ
by `0x4000`:

| Interface | Virtual base | Inferred physical base | State |
|---|---|---|---|
| `eth1` | `0xCE9C0000` | `0x101C0000` | No PHY, MAC `FF:FF:FF:FF:FF:FF`, down |
| `eth0` | `0xCE9C4000` | `0x101C4000` | Active, link 1000/Full |

Only `eth0` is wired to a connector. `eth1` exists in the SoC and is probed,
but has no PHY and never comes up.

The pinmux corroborates this. The whole **RGMII1** bus — RXDV, RXD3–RXD0,
RXCK, TXEN, TXD3–TXD0, TXCK, TXCKOUT, plus RXER and TXER — is muxed to its
Ethernet function at `0x200f01bc`–`0x1f4`, set by U-Boot and left alone by
Linux. RGMII0 has mux registers only for `TXCK`, `CRS` and `COL`; the rest of
its bus is on dedicated pins, so the pinmux cannot say whether RGMII0 is routed
to anything. See [19-pinmux-map.md](19-pinmux-map.md).

The MAC also claims two register windows outside its own block:

- `0x200F0000`–`0x200F01FF` — pin multiplexing (IO_CONFIG)
- `0x20030000`–`0x200300FF` — CRG (clocks)

The boot message `Set system config register 0x200300ec with value 0x003f003f`
is a hard-coded `pr_info` at probe, and a live U-Boot read confirms
`CRG + 0xEC` holds `0x003F003F` before Linux starts.

### CRG + 0xEC bit layout

Recovered from `drivers/net/stmmac/stmmac_main.c` in the SDK kernel
(`stmmac_syscfg_phy_cfg`). The register is split into two 16-bit fields — GMAC0
at shift 0, GMAC1 at shift 16 — and is rewritten by the driver on every link
state change:

| Bits | Meaning |
|---|---|
| `0x0003` | Speed: `0x0` = 1000 Mbps, `0x3` = 100 Mbps, `0x1` = 10 Mbps |
| `0x000C` | Link up **and** TX enable |
| `0x0010` | Full duplex |
| `0x0020` | RGMII mode — the driver comments this as "Always RGMII mode" |

So a 1000 Mbps full-duplex link with TX enabled yields `0x3C` in the low field.

A port must reproduce this — most likely as a `syscon` phandle plus a
`hisilicon,*` glue driver implementing `fix_mac_speed`, which is how mainline
handles other DWMAC integrations (see
`drivers/net/ethernet/stmicro/stmmac/dwmac-*.c` for the pattern).

## PHY

| Property | Value |
|---|---|
| Part | Realtek **RTL8211CL** (U67, confirmed from PCB photo) |
| PHY ID | `0x001CC912` |
| MDIO address | 1 |
| Link at capture | 1000 Mbps, full duplex |
| MDIO buses | Two registered (`/sys/class/mdio_bus/0`, `/1`) |
| Platform device | `stmmacphy.1` |

A `Fixed MDIO Bus` is also registered by the kernel but is not used for this PHY.

The U-Boot environment expects different addresses from what the kernel finds:

| Variable | Value | Meaning |
|---|---|---|
| `phyaddr0` | 2 | U-Boot's MAC0 PHY address |
| `phyaddr1` | 1 | U-Boot's MAC1 PHY address |
| `phyintfx` | 0 | Interface mode selector |

The kernel reports **both** `eth0` and `eth1` finding "PHY ID 001cc912 at 1",
which is a quirk of the vendor driver scanning the same bus twice rather than
evidence of two PHYs. Only one RTL8211CL is visible on the board.

### Interface mode

The SDK's platform data settles this. From `stmmac_main.c`:

```c
static struct plat_stmmacphy_data stmmac_phy_private_data[] = {
    [0] = {
        .bus_id    = 1,
        .phy_addr  = CONFIG_STMMAC_PHY0_ID,
        .phy_mask  = 0,
        .interface = PHY_INTERFACE_MODE_RGMII,
    },
    ...
```

**Plain `PHY_INTERFACE_MODE_RGMII`** — no `-id`, `-txid` or `-rxid` variant.
In modern kernels that means neither MAC nor PHY adds delay, so the required
RGMII clock skew must come from board trace lengths or PHY strapping. The
RTL8211CL takes its delay configuration from hardware strap pins.

Start with `phy-mode = "rgmii"` to match the vendor. If the link comes up but
no traffic passes — the classic RGMII delay symptom — try `rgmii-id` next.

The driver is loaded by `rootfs/mtd/boot.sh` with the PHY addresses as module
parameters, and picks the speed from a config file:

```sh
NET_CHIP_TYPE=`cat "/etc/init.d/NetChip.dat"`
if [ "$NET_CHIP_TYPE" = "1" ] ; then
    insmod stmmac.ko macsorts=1 phyid0=2 phyid1=1 phytype=1    # 100 Mbps
else
    insmod stmmac.ko macsorts=1 phyid0=2 phyid1=1              # gigabit
fi
```

`NetChip.dat` on this unit contains `0`, so the gigabit path is taken.

## MAC address

The MAC address is **not** stored in the U-Boot environment in usable form.

| Source | Value |
|---|---|
| U-Boot `ethaddr` | `00:00:23:34:45:66` — placeholder |
| Kernel at probe | `00:00:00:00:00:00` — "no valid MAC address for MAC 0/1" |
| After vendor app starts | `00:18:AE:3C:A2:49` |

The vendor application sets it late in boot (`MACADDR in set is 0:18:ae:3c:a2:49`).
OUI `00:18:AE` belongs to **TVT Co., Ltd.** — see
[15-product-identity.md](15-product-identity.md).

**The address is stored in two places**, both found:

| Location | Form |
|---|---|
| `/etc/init.d/mac.dat` in the rootfs | 17-byte ASCII string, no terminator |
| SPI-NOR offset `0xBFC20` (mirrored at `0xFFC20`) | ASCII, NUL-padded, inside the board parameter block |

The vendor application reads the file directly — the string `/etc/init.d/mac.dat`
appears in the `td3531` binary — and applies it with `settimeofday`-style
runtime configuration rather than anything the kernel does.

The SPI-NOR copy is the factory master: it sits alongside a `v1.2` board
revision tag matching the `DHB_AX V1.2` PCB silkscreen, and survives a rootfs
reflash. See [04-flash-storage.md](04-flash-storage.md) for the full parameter
block layout.

For a port, the simplest approach is to read `mac.dat`, or hard-code the
address via a `mac-address` property or `ethaddr`. Preserving it matters — a
stable, unique MAC is worth keeping on a server.

## TOE / TNK

The vendor kernel includes a TCP offload engine driver:

```
**************************************************
*  TNK driver built on Jun 19 2012 at 17:11:50
*  TNK driver mode is BYPASS
**************************************************
```

It runs in **BYPASS** mode, meaning offload is not active and the MAC behaves as
a plain DWMAC1000. This is good news: there is nothing to reimplement, and
mainline `stmmac` should be functionally equivalent. Ignore TNK entirely.

## Device tree sketch

```dts
gmac0: ethernet@101c4000 {
    compatible = "hisilicon,hi3531-dwmac", "snps,dwmac-3.60a";
    reg = <0x101c4000 0x2000>;
    interrupts = <0 87 4>;          /* SPI 87, level-high — vendor IRQ 119 */
    interrupt-names = "macirq";
    phy-mode = "rgmii-id";          /* UNVERIFIED */
    phy-handle = <&phy1>;
    snps,pbl = <32>;                /* UNVERIFIED */

    mdio {
        compatible = "snps,dwmac-mdio";
        #address-cells = <1>;
        #size-cells = <0>;
        phy1: ethernet-phy@1 {
            reg = <1>;              /* RTL8211CL */
        };
    };
};
```

The Realtek PHY is handled by mainline `drivers/net/phy/realtek.c`, which
supports RTL8211C, so no PHY driver work is needed.

## Assessment

Low risk. The MAC IP is standard and well supported, the PHY is standard and
well supported, and the offload engine is disabled. The two things to get right
are the CRG/pinmux glue at probe time and the RGMII mode and delays. Expect
this to work early in the port.
