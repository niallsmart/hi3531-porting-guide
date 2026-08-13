# Device-Tree Handoff from the Vendor U-Boot

## Observation

The roadmap proposes a modern DT-based kernel loaded with `bootm`, but the
Godnet U-Boot configuration does not appear to define `CONFIG_OF_LIBFDT`. No FDT
command is present in the captured command list, and no FDT-related string was
found in the SPI image during this review.

## Questions to investigate

- How is the first modern kernel expected to receive its DTB?
- Is an appended DTB with `CONFIG_ARM_APPENDED_DTB` appropriate?
- Is `CONFIG_ARM_ATAG_DTB_COMPAT` needed to retain the U-Boot command line or
  other ATAG data?
- Does this `bootm` understand a usable multi-image format?
- Would rebuilding U-Boot with libfdt support be preferable?

The exact kernel packaging steps and tested `bootm` command may belong in the
Phase 1 instructions.

