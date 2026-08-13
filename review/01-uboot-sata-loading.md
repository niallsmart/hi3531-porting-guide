# Can the Installed U-Boot Load a Kernel from SATA?

## Observation

Several documents recommend loading a modern kernel from the SATA disk with the
existing U-Boot's `ext2load` or `fatload` commands:

- `doc/04-flash-storage.md`, lines 130–134
- `doc/07-sata-storage.md`, lines 126–137
- `doc/16-porting-roadmap.md`, lines 6–23

The captured U-Boot command list contains `ext2load`, `fatload`, and USB
commands, but no `sata`, `scsi`, or `ide` command. The Godnet configuration in
`osdrv/uboot/u-boot-2010.06/include/configs/godnet.h` enables FAT, ext2, and USB
storage, but does not appear to enable `CONFIG_CMD_SATA`, `CONFIG_CMD_SCSI`, or
`CONFIG_CMD_IDE`. U-Boot's `common/Makefile` builds the corresponding commands
only when those options are selected.

## Questions to investigate

- Does the installed U-Boot have an unlisted mechanism that registers SATA as a
  block device?
- Can `ext2load` or `fatload` name a SATA-backed interface in this build?
- Should the recommended path instead load the kernel by TFTP or USB and use
  SATA only as the Linux root filesystem after AHCI initializes?
- Would direct SATA loading require rebuilding or replacing U-Boot?

