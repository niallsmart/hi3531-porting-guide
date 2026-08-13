# Ethernet Device-Tree Details

## Observation

The Ethernet narrative recommends beginning with `phy-mode = "rgmii"` because
that matches the vendor platform data. The DTS sketch instead uses `rgmii-id`.

The sketch also uses `snps,dwmac-3.60a`. The current upstream
`snps,dwmac.yaml` binding lists `snps,dwmac`, `snps,dwmac-3.610`, and several
other revisions, but not `snps,dwmac-3.60a`.

## Questions to investigate

- Which PHY mode best matches the vendor configuration and board wiring?
- Where is the required RGMII delay supplied: RTL8211CL straps, trace length,
  MAC configuration, or PHY programming?
- Which upstream-compatible string matches the observed DWMAC version register?
- What Hi3531-specific clock, reset, syscon, or PHY glue is required?
- Would a new `hisilicon,hi3531-dwmac` binding be needed, and what should its
  generic fallback be?

