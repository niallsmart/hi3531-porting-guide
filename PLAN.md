
# Goal

The user owns a Hi3531 powered DVR (TVT Digital LTD2704XE-P, distributed by LTS) that they want to repurpose as a general purpose Linux server. To support this, they intend to port a modern Linux kernel to the DVR, and enable as much of the on-device hardware as reasonably technically feasible.

Your goal is to support the porting effort by documenting the hardware, such that a kernel developer or another agent can complete the porting work independently.  You should not attempt the kernel port yourself.  Just provide the documentation required for that work.

# Documentation

Please create the required documentation as separate markdown files in a doc/ subfolder.  Include an overall README.md with a summary and table of contents.  Create a separate markdown file to document each subsystem.

# Reference Assets

You have access to these assets. 

* A HiSilicon SDK (l)ocated in Hi3531_V100R001C01SPC0D1/)  This is believed to be close or identical to the original SDK for this device. Note that there are compressed tgz files in the SDK that you may need to unpack to access sources. Please verify this before starting.

* An export of the DVR filesystem (located in rootfs/)

* An export of the SPI and NAND memory (located in backups/)

* Photos of the top-surface of the PCB (located in pcb/)

* Live telnet access to the DVR on 192.168.4.77 with user root default password 1001chin

* Live UART console access to the DVR, proxied through `raspberrypi` (use: `ssh -t raspberrypi "picocom -b 115200 --omap crcrlf --logfile dvr.log /dev/serial0"`).  This is helpful if you need to inspect the U-Boot console (press any key to interrupt the default boot)

# Process

First inventory the assets and live DVR.

Once you have completed the inventory, let's pause and review which subsystems are worth documenting.

You don't need to record the history of your investigation in the documentation, just state what you know at the end.

Then you will create the documentation.

# Permissions

DO NOT WRITE TO SPI OR NAND STORAGE ON THE DVR.

You may reboot the DVR as necessary.

You may reconfigure the raspberrypi as necessary. This includes installing packages and running network services. (Please keep a log of all configuration changes to the raspberrypi in a separate markdown file)
