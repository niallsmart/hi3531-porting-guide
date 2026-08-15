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
| Checksum offload | Vendor reports Rx COE; disabled in the validated mainline port because the hardware feature register is unusable |
| Platform device | `stmmaceth.0` |

**Two MAC instances are present.** From the boot log, their mapped bases differ
by `0x4000`:

| Interface | Virtual base | Inferred physical base | State |
|---|---|---|---|
| `eth1` | `0xCE9C0000` | `0x101C0000` | No PHY, MAC `FF:FF:FF:FF:FF:FF`, down |
| `eth0` | `0xCE9C4000` | `0x101C4000` | Active, link 1000/Full |

Only `eth0` is wired to a connector. `eth1` exists in the SoC and is probed,
but has no PHY and never comes up.

The integration is not two independent DWMAC register windows. It is one
shared block whose layout was recovered from the vendor driver and validated
by the Linux 6.18 port:

| Offset from `0x101C0000` | Function |
|---|---|
| `+0x0000` | GMAC0 control and shared MDIO registers |
| `+0x4000` | GMAC1 control registers |
| `+0x1000 + n * 0x100` | DMA channel `n`; GMAC1 uses channel 1 |
| `+0x9000` | TNK interrupt aggregator |

Generic `stmmac` therefore cannot drive the wired port merely by mapping
`0x101C4000`: it expects its MAC, MDIO and DMA registers at fixed relative
offsets. The Hi3531 glue keeps the shared base for MDIO, redirects MAC accesses
to `+0x4000` and DMA accesses to channel 1, resets all three DMA channels after
a warm boot, and enables only the GMAC1/DMA1 aggregator interrupt bits.

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

So a 1000 Mbps full-duplex link with TX enabled yields `0x3C`.

**Which half belongs to which interface is settled by a live read.** With
`eth0` up at 1000/full and `eth1` down, the register holds:

```
0x200300EC = 0x003C003F
             ^^^^ ^^^^
             |    +--- bits [15:0]  GMAC0 = eth1 = 0x101C0000
             |         0x3F: still the 100 Mbps probe default, never updated
             +-------- bits [31:16] GMAC1 = eth0 = 0x101C4000
                       0x3C: 1000 Mbps, link up, TX enabled, full duplex, RGMII

```

That both confirms the bit layout and pins down the numbering:

| Physical base | Kernel name | CRG+0xEC field | State |
|---|---|---|---|
| `0x101C0000` | `eth1` | bits [15:0] | No PHY, never comes up |
| `0x101C4000` | `eth0` | bits [31:16] | **The live port** |

This agrees with the pinmux, where the bus muxed out to the PHY is **RGMII1**,
not RGMII0 — see [19-pinmux-map.md](19-pinmux-map.md). GMAC1, RGMII1 and `eth0`
are the same path, and a port only needs to describe that one.

A port must reproduce the register write in a `hisilicon,*` glue driver's
`fix_mac_speed`, in addition to handling the shared-block integration above.
This follows the pattern of other mainline DWMAC integrations (see
`drivers/net/ethernet/stmicro/stmmac/dwmac-*.c` for the pattern). Note the
shift: for `eth0` the fields are at bit 16, not bit 0.

### Core version

The MAC's version register reports the Synopsys core revision directly, so
there is no need to infer it:

```
0x101C0020 = 0x00001036      synopsys_id = 0x36 -> DWMAC 3.60
0x101C4020 = 0x00001036      user-defined version 0x10
```

`stmmac` reads this itself at probe and keys its behaviour off it, which is why
the device tree does not have to name the revision — see the compatible-string
discussion under [Device tree](#device-tree).

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

The PHY ID does not distinguish the B and C/CL variants. This RTL8211CL reports
`0x001cc912`, an ID also used by RTL8211B parts. Mainline therefore binds its
"RTL8211B Gigabit Ethernet" entry even though the package marking, confirmed in
`pcb/U67 RealTek RTL8211CL.jpeg`, says RTL8211CL. The driver name in the boot
log is not a hardware identification, and changing the device-tree compatible
cannot resolve the ambiguity.

This matters because mainline applies a gigabit slave-mode erratum only in its
RTL8211C entry: `rtl8211c_config_init()` forces the PHY to be the 1000BASE-T
master by setting `CTL1000_ENABLE_MASTER | CTL1000_AS_MASTER` in
`MII_CTRL1000`. The RTL8211B entry selected here does not perform that setup.
The link has been stable at 1 Gbps under sustained traffic, so no workaround is
currently needed.

If gigabit negotiation becomes unreliable or reports a master/slave resolution
failure, first test the same setup at runtime with:

```
ethtool -s eth0 master-slave forced-master
```

If that fixes the problem, make it persistent by adding
`timing-role = "forced-master";` to the PHY node. Phylib then programs the same
two `MII_CTRL1000` bits during autonegotiation without pretending the shared
PHY ID identifies a different chip.

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
RGMII clock skew must come from board trace lengths or PHY strapping.

Two things make strapping the likely source here. The RTL8211CL takes its delay
configuration from strap pins, and **mainline's driver for this PHY ID cannot
program delays at all** — the RTL8211B and RTL8211C entries in
`drivers/net/phy/realtek/` have no `config_init` for it. Only the RTL8211E entry
(`0x001cc915`) reaches extension page 164 to set RX/TX delay. So on this board
`phy-mode` selects MAC-side behaviour and nothing else; the PHY will do whatever
its straps say regardless of what the device tree asks for.

The practical consequence is that **`rgmii` and `rgmii-id` are likely to behave
identically here**, because neither end acts on the difference. If the link
comes up but no traffic passes — the classic delay symptom — the fix is not in
the device tree. It is a strap resistor on the board, and settling it needs
either underside PCB photographs or an MDIO read of the PHY's own registers.
Both are open items.

Use `phy-mode = "rgmii"`, matching the vendor.

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

It runs in **BYPASS** mode, meaning the TOE data path is not active and does not
need to be reimplemented. The TNK block cannot be ignored completely, however:
it is also the shared interrupt aggregator. The mainline glue writes `0x48` to
its interrupt-enable register (`+0x9004`), selecting GMAC1 and DMA channel 1
while leaving the TOE interrupt sources disabled.

## Device tree

### The compatible string

**Do not use `snps,dwmac-3.60a`.** It is not a value
`Documentation/devicetree/bindings/net/snps,dwmac.yaml` accepts, so
`dtbs_check` rejects it and no driver matches it. The binding's versioned
strings jump from `snps,dwmac-3.50a` to `snps,dwmac-3.610`, with nothing for
3.60.

The revision does not need to be named in the device tree. `stmmac` reads the
version register at probe, while the Hi3531 glue explicitly supplies the
undetectable capabilities established on this integration. Enhanced
descriptors are required. Checksum offload, jumbo frames and PMT/WOL remain
disabled because CSR58 is unusable and those capabilities have not been
validated. Forced TX store-and-forward must also remain disabled: with checksum
offload off, target testing found that it wedges the TX DMA after a short burst.

Two options, depending on how far the port has got:

```dts
/* Diagnostic bring-up only. This can identify the core, but does not describe
   the shared GMAC1/MDIO/DMA/TNK integration correctly. */
compatible = "snps,dwmac";

/* Validated port: Hi3531 glue with the unversioned Synopsys fallback. */
compatible = "hisilicon,hi3531-dwmac", "snps,dwmac";
```

`hisilicon,hi3531-dwmac` does not exist upstream. Adding it means a
`dwmac-hi3531.c` under `drivers/net/ethernet/stmicro/stmmac/` and a binding
document that references `snps,dwmac.yaml` — the pattern every other vendor
glue follows. The glue must implement the shared register layout, DMA reset and
TNK mask described above as well as rewriting the GMAC1 field of `CRG + 0xEC`
when the link changes. The generic fallback records the underlying IP identity;
it is not a promise that the wired port works without the Hi3531 glue.

### The node

```dts
gmac1: ethernet@101c0000 {
    compatible = "hisilicon,hi3531-dwmac", "snps,dwmac";
    reg = <0x101c0000 0x20000>,
          <0x20030000 0x100>;
    reg-names = "stmmaceth", "syscfg";
    interrupts = <0 87 4>;          /* SPI 87, level-high — vendor IRQ 119 */
    interrupt-names = "macirq";
    clocks = <&peripheral_clk>;
    clock-names = "stmmaceth";
    phy-mode = "rgmii";             /* matches the vendor; see above */
    phy-handle = <&phy1>;
    max-speed = <1000>;
    local-mac-address = [00 18 ae 3c a2 49];
    snps,pbl = <16>;
    snps,no-pbl-x8;

    mdio {
        compatible = "snps,dwmac-mdio";
        #address-cells = <1>;
        #size-cells = <0>;
        phy1: ethernet-phy@1 {
            reg = <1>;              /* marked RTL8211CL; IDs as 0x001cc912 */
        };
    };
};
```

Named `gmac1` deliberately — this is GMAC1, the second instance, and the one
wired to the connector. The unit address is nevertheless the shared block base,
because the glue redirects the MAC data path to GMAC1 while retaining the
shared MDIO window. GMAC0 has no PHY and is not exposed as a second netdev.

The Realtek PHY needs no driver work; `drivers/net/phy/realtek/` already matches
its ID.

### What is still unverified

| Item | Why it is open |
|---|---|
| `snps,pbl = <32>` | Carried over from the vendor's platform data. Not checked against the DMA bus-mode register |
| The RGMII delay source | Neither MAC nor PHY driver supplies it; assumed to be strapping or trace length, unconfirmed either way |
| Whether any `snps,dwmac-3.610`-style quirk applies | The 3.60 core is not described by any upstream compatible, so the quirk set is untested |
| Clocks and resets | The vendor driver touches CRG for speed but no separate clock or reset phandles have been identified |

## Assessment

Low risk. The MAC IP is standard and well supported, the PHY is standard and
well supported, the offload engine is disabled, and the pinmux is already
correct out of U-Boot and untouched by Linux.

The one piece of real work is a small glue driver whose only job is
`fix_mac_speed` against `CRG + 0xEC`. Without it the link is stuck at whatever
speed the register already holds, which is enough to prove the port and not
enough to ship.

The RGMII delay is the residual risk, and it is a board question rather than a
software one — no driver in the path can set it. Expect this to work early.
