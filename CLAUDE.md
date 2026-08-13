
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

# Evidence and Inference

Most errors found in this documentation came from treating a plausible label as a fact. Know which source a claim rests on, and rank them:

1. **The chip datasheet** — `Hi3531_V100R001C01SPC0D1/00.hardware/chip/documents_en/Hi3531 H.264 Codec Processor Data Sheet.pdf`, 1794 pages, the full register manual with a section per peripheral. **Check whether it answers the question before building an answer out of anything below it.** It sat unused in the SDK while the entire pinmux map was reconstructed, badly, from shell-script comments.
2. **A live read from the device** — `md` at the U-Boot prompt, `devmem` under Linux.
3. **SDK and U-Boot source** — real code, but written for HiSilicon's demo board, not this one.
4. **Vendor scripts and binaries under `rootfs/`** — board-specific, which is what makes them valuable, and unreliable in the specific ways below.
5. **Inference from the shape of the data** — always usable, never unlabelled.

## Vendor comments are claims, not evidence

A comment in `rootfs/mtd/**` records what a TVT engineer believed, possibly about a different product variant. In this project they have been wrong in three distinct ways:

* **Systematically shifted.** GPIO numbers in `pinctrl_*_hi3531.sh` run three too high across the VIU0 block and three too low across the VIU2 block — 37 wrong labels, which also produced two duplicate GPIO names that looked like isolated typos.
* **Copy-pasted.** The `I2C_SCL` line carries the `GPIO12_4` label belonging to the line above it.
* **Contradicted by the code they annotate.** `dep2.sh` writes `0x200f004c` under `#set default buzzer gpio control`; the value it writes is 0, which takes that pin *out* of GPIO mode. The buzzer is on the front-panel MCU.

Read the write, not the comment. Where they disagree, the write wins and the documentation should record both.

## A register dump is a timestamp, not a configuration

Mux and control registers are mutable, and several actors write them in sequence: reset defaults, then U-Boot, then vendor init scripts, then individual drivers at probe. 44 of the 128 pinmux registers captured in U-Boot hold a different value under the running kernel. A dump taken at one stage described as "the board's configuration" is how `doc/17` came to state that the widest bus on the chip was video output when the running system has it as video input.

Every dump must say where it was taken. If a pin or clock matters, capture it at more than one stage.

## Before writing a claim

* **Does a label assert purpose?** "Buzzer control", "video output bus", "plausibly RGMII" are conclusions. Each needs a source or an explicit marker that it is inferred.
* **Does another document in `doc/` already say something different?** Resolve it before writing, and fix the loser. The buzzer was correctly documented in `doc/10` and `doc/20` while `doc/17` and `doc/19` contradicted them.
* **Is a number counted or estimated?** State the denominator, and count only what the claim actually describes. Two figures reported during the pinmux work — "65 wrong cells across 56 registers" and "68 registers differ" — were both inflated by sweeping in differences that were not errors, or not differences.
* **Would anything break if this were wrong?** If not, that is a reason for more suspicion rather than less. The shifted GPIO labels survived because nothing on the board used those pins as GPIO.

# Permissions

DO NOT WRITE TO SPI OR NAND STORAGE ON THE DVR.  DO NOT WRITE TO ANY yaffs2 FILESYSTEMS.

You may reboot the DVR as necessary.

You may reconfigure the raspberrypi as necessary. This includes installing packages and running network services. (Please keep a log of all configuration changes to the raspberrypi in doc/raspberrypi-changes.md)

# Technical Language

The can assume the audience for the documentation you're creating in doc/ is a sophisticated and experienced kernel developer who is deeply familiar with the Linux kernel, the ARM processor archecture, porting concerns and supporting tooling.

The user driving this session interactively is a Staff Backend Engineer. They have experience with system programming and operating systems, but are relatively new to embedded systems and kernel internals. If they ask you to explain something interactively, calibrate your explanation and jargon appropriately, and build up from any conceptual pre-reqs.

