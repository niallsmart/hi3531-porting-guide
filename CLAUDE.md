
# Goal

The user owns a Hi3531 powered DVR (TVT Digital LTD2704XE-P, distributed by LTS) that they are repurposing as a general purpose Linux server. They intend to port a modern Linux kernel to the DVR, and enable as much of the on-device hardware as reasonably technically feasible.

This project supports the porting effort by documenting the hardware, such that a kernel developer or another agent can complete the porting work independently. Do not attempt the kernel port yourself. Just support creating the technical documentation required for that work.

# Documentation

Documentation should be stored as markdown in the doc/ subfolder. The README.md provides a summary and table of contents. Separate markdown files document each subsystem.

You don't need to record the history of your investigations and research in the documentation, or a Changelog as you refined your understanding. Just state the final conclusions and what you know at the end.

Store local copies of any relevant datasheets in datasheets/

# Reference Assets

You have access to these assets:

* A HiSilicon SDK (located in Hi3531_V100R001C01SPC0D1/)
  * This is similar or identical to the original SDK for this device
  * Note that compressed tgz files in the SDK have been unpacked in-situ.
  * The SDK contains deeply nested subdirectories. Depth-limited searches may limit your visibility.

* An export of the DVR filesystem (located in rootfs/)

* An export of the SPI and NAND memory (located in backups/)

* Photos of the top-surface of the PCB (located in pcb/)

* Live telnet access to the DVR on 192.168.4.77 with user root default password 1001chin

* Live UART console access to the DVR, proxied through `raspberrypi` (use: `ssh -t raspberrypi "picocom -b 115200 --omap crcrlf --logfile dvr.log /dev/serial0"`).  This is helpful if you need to inspect the U-Boot console (press any key to interrupt the default boot)

# Permissions

DO NOT WRITE TO SPI OR NAND STORAGE ON THE DVR.  DO NOT WRITE TO ANY yaffs2 FILESYSTEMS.

You may reboot the DVR as necessary.

You may reconfigure the raspberrypi as necessary. This includes installing packages and running network services. (Please keep a log of all configuration changes to the raspberrypi in doc/raspberrypi-changes.md)
