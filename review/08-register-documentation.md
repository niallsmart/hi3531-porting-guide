# Scope of the Hi3531 Register Documentation

## Observation

Several documents characterize video capture and display blocks as having no
public register-level documentation:

- `doc/11-video-input.md`, lines 290–294
- `doc/12-video-output.md`, lines 142–148
- `doc/14-media-codec.md`, lines 49–54
- `doc/16-porting-roadmap.md`, lines 90–93

The supplied `Hi3531 H.264 Codec Processor Data Sheet.pdf` appears to contain
detailed register summaries and descriptions for VICAP at `0x20580000`, VDP at
`0x205c0000`, GPIO, watchdog, I2C, SATA PHY setup, USB setup, IVE, motion
detection, and other media-adjacent blocks.

The document also advertises HDMI up to 1080p60 and VGA up to 2560x1600 at
60 Hz. `doc/12-video-output.md` describes the observed 800x600 mode as a limit.

## Questions to investigate

- Which blocks genuinely lack a usable programming model, and which are covered
  by the supplied register manual?
- Should conclusions about the proprietary H.264 firmware and MPP stack be kept
  separate from assessments of VICAP, VDP, GPIO, IVE, and similar blocks?
- Is 800x600 only the board's current configuration rather than its hardware
  limit?
- How should the capture and display effort estimates change, if at all, after
  examining the documented initialization and register sequences?

