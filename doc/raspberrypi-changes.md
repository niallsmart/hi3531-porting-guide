# Raspberry Pi Configuration

This describes the host's current state and the changes made to reach it.

## Host

| Property | Value |
|---|---|
| Hostname | `raspberrypi` |
| Address | `192.168.4.34` (from `~/.ssh/config` on the workstation) |
| OS | Raspbian, Linux 6.18.34+rpt-rpi-v6, `armv6l` |
| User | `niallsmart` |
| Groups | `adm dialout cdrom sudo audio video plugdev games users netdev gpio i2c spi render input` |
| Passwordless sudo | Yes |
| Serial port | `/dev/serial0` → `/dev/ttyAMA0`, connected to the DVR console |

## Services

Both were already installed and running; only the exports line was changed.

| Service | State |
|---|---|
| `tftpd-hpa` | 5.2+20240610-3, active. Root `/srv/tftp`, listening on `:69`, `--secure` |
| `nfs-kernel-server` | 1:2.8.3-1, active |
| `picocom` | Installed |

TFTP is usable for netbooting test kernels: place an image in `/srv/tftp` and
fetch it from U-Boot with `serverip=192.168.4.34`. See
[03-boot-chain.md](03-boot-chain.md).

## Changes made (2026-08-12)

### Installed `python3-serial` (3.5-2)

```sh
sudo apt-get install -y python3-serial
```

Needed for scripted control of the DVR serial console; `picocom` is interactive
and unsuitable for automation.

Reverse with `sudo apt-get remove python3-serial`.

### Added the DVR to the NFS exports

`/etc/exports` already exported `/srv/dhb-ax` to `192.168.7.240`. The DVR's
current address, `192.168.4.77`, was added alongside the existing client rather
than replacing it:

```
/srv/dhb-ax 192.168.7.240(rw,sync,no_subtree_check,no_root_squash) 192.168.4.77(rw,sync,no_subtree_check,no_root_squash)
```

Applied with `sudo exportfs -ra`. The original was backed up to
`/etc/exports.bak.<YYYYmmdd-HHMMSS>`; restore it and re-run `exportfs -ra` to
reverse.

### Added serial capture scripts

Four Python scripts in the home directory, with their logs:

| Script | Purpose | Output |
|---|---|---|
| `uboot_capture.py` | Interrupts U-Boot autoboot, runs `version` / `printenv` / `help` | `dvr-uboot.log`, `uboot_capture.out` |
| `uboot_capture2.py` | Dumps pinmux / CRG / SYS_CTRL / DDR registers with `md` | `dvr-uboot2.log`, `uboot_capture2.out` |
| `uboot_capture3.py` | `getinfo` subcommands, `usb tree`, boot-strap register | `dvr-uboot3.log`, `uboot_capture3.out` |
| `uboot_capture4.py` | DRAM aliasing test | `dvr-uboot4.log`, `uboot_capture4.out` |

All four try to resume the board with `run bootcmd`, which is **not a command
in this U-Boot** — use `reset` instead if reusing them. See the pitfalls table
in [18-reference-assets.md](18-reference-assets.md).

Reverse by deleting the files. Nothing was installed system-wide and no service
was created.

## Changes made (2026-08-13)

### Serial console logging with `cat`

The watchdog work needed the DVR console captured across resets, which
`picocom` cannot do unattended. No package was installed; `cat` was used
directly:

```sh
stty -F /dev/serial0 115200 raw -echo
setsid nohup cat /dev/serial0 > /tmp/wd.log 2>/dev/null < /dev/null &
```

This is worth reusing. It survives the ssh session ending, and because each
`read()` is written straight through there is no buffering to lose the last
seconds before a reset — which matters when the event you are trying to catch
*is* the reset. Stop it with `pkill -f 'cat /dev/serial0'`.

**Both loggers were stopped at the end of the session.** Two log files remain:

| File | Contents |
|---|---|
| `/tmp/wd.log` | The MCU watchdog reset — kicks stopped with `SIGSTOP`, 60-second timeout, U-Boot banner |
| `/tmp/wd2.log` | The `SIGTERM` reset and the subsequent clean `reboot` |

Both are in `/tmp` and will not survive a reboot of the Pi. Nothing else was
changed; no packages, services or configuration files were touched.

## Untouched

- Network configuration, firewall, and `systemd` units.
- TFTP and NFS server configuration, beyond the single exports line above.
- Serial port configuration — `/dev/serial0` was already enabled and accessible
  to the `dialout` group.
