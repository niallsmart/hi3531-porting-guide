# DHB_AX Hardware Guide

Hardware reference for the Shenzhen TVT Digital **`DHB_AX V1.2`** board, built
on the HiSilicon Hi3531 and sold as the LTS LTD2704XE-P four-channel DVR. The
board is being repurposed as a general-purpose Linux server.

The guide covers the SoC, memory layout, boot chain, device-tree requirements,
Ethernet, SATA, USB, display hardware, and other board peripherals. It
documents what the hardware is and how the vendor firmware drives it.

The port itself lives in [dhb-ax-buildroot](https://github.com/niallsmart/dhb-ax-buildroot):
Buildroot config, device trees and the kernel patch queue.

Start with the [documentation index](doc/README.md).
