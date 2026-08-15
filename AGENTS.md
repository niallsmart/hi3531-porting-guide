
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

* Live telnet access to the DVR at `dvr` with user root and default password `1001chin`

* Live UART console access to the DVR, proxied through `raspberrypi` (use: `ssh -t raspberrypi "picocom -b 115200 --omap crcrlf --logfile dvr.log /dev/serial0"`).  This is helpful if you need to inspect the U-Boot console (press any key to interrupt the default boot)

# Evidence and Inference

Most errors in this documentation came from treating a plausible label as a fact. Know which source a claim rests on:

1. **The chip datasheet** — `00.hardware/chip/documents_en/Hi3531 H.264 Codec Processor Data Sheet.pdf`, the full register manual. **Check it before reconstructing an answer from anything below.** It sat unused in the SDK while the whole pinmux map was rebuilt, badly, from shell-script comments.
2. **A live read** — `md` at the U-Boot prompt, `devmem` under Linux.
3. **SDK and U-Boot source** — real code, but for HiSilicon's demo board.
4. **Vendor scripts under `rootfs/`** — board-specific, and unreliable.
5. **Inference** — always usable, never unlabelled.

Three rules follow:

* **Read the write, not the comment.** Vendor comments here have been systematically shifted (37 wrong GPIO numbers), copy-pasted, and flatly contradicted by the line they annotate — `dep2.sh` writes `0x200f004c` under `#set default buzzer gpio control`, and the value it writes takes the pin *out* of GPIO mode.
* **A dump is a timestamp, not a configuration.** 44 of the 128 pinmux registers captured in U-Boot differ under the running kernel. Say where every dump was taken.
* **Label anything inferred**, and check whether another file in `doc/` already says the opposite.

# Permissions

DO NOT WRITE TO SPI OR NAND STORAGE ON THE DVR.  DO NOT WRITE TO ANY yaffs2 FILESYSTEMS.

You may reboot the DVR as necessary.

You may reconfigure the raspberrypi as necessary. This includes installing packages and running network services. (Please keep a log of all configuration changes to the raspberrypi in doc/raspberrypi-changes.md)

# Technical Language

You can assume the audience for the documentation you're creating in doc/ is a sophisticated and experienced kernel developer who is deeply familiar with the Linux kernel, the ARM processor architecture, porting concerns and supporting tooling.

The user driving this session interactively is a Staff Backend Engineer. They have experience with system programming and operating systems, but are relatively new to embedded systems and kernel internals. Calibrate every interactive reply to that by default, not only when asked to explain: assume the systems knowledge, explain the embedded concepts, and build up from any conceptual pre-reqs. Keep it proportionate — a short aside where a term or assumption is unfamiliar, not a tutorial on every answer.
