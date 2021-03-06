---
title: Samsung Galaxy Nexus
---


Introduction
====================

We're currently working on two projects

* Removing untrusted proprietary binaries from the android distribution
* Virtualizing Android on top of the Genode framework

The primary development device for this project is the Samsung Galaxy Nexus phone.

Virtualization
---------------------

Samsung Galaxy Nexus is based on the Texas Instruments OMAP4 CPU which is an ARMv7 core and thus incapable of hardware-assisted virtualization. We rely on paravirtualization therefore. A modified version of linux (L4Linux) is running on top of the Fiasco.OC L4 microkernel and the Genode Framework. Virtualization and moving drivers from linux to Genode gives us several advantages.

* Running different OS (Android, Ubuntu) simultaneosly without rebooting
* Putting untrusted code into containers like Qubes OS
* Reusing the same L4Linux and Android userland on different phones, reducing update time
* Using L4 IPC protects against stack smashing attacks against device drivers

A picture of the Galaxy Nexus phone running three instances of linux simultaneously.

2175944.jpg

Genode Support for Samsung Galaxy Nexus
---------------------

[repository](https://github.com/Ksys-labs/genode.git)

The tuna-hacks branch contains the WIP support for the Galaxy Nexus. Here is the list of our changes.

* OMAP4 Framebuffer support for Galaxy Nexus (currently just using the framebuffer preconfigured by the SBL bootloader and changing the memory address and color space)
* OMAP4 I2C support
* OMAP4 GPIO MUX support
* Melfas 144 Touchscreen Driver
* L4Android fixes for linux kernel 3.5

To build a demo, use the “astarasikov/run/tuna-l4android35.run”. Refer to the general Genode building guide and do a “make run/tuna-l4android35” to build the system. Make sure to prepare an initial ramdisk image and edit the path to it in the tuna-l4android35.run file.

Alternatively, use a prebuilt demo image [still uploading] and refer to the u-boot section on the details about booting this image.

U-boot port
====================

Detailed description is on the page [gnex_uboot](/gnex_uboot)

To be able to run custom OS without reflashing the phone, we have ported the u-boot bootloader to the Galaxy Nexus phone. It currently has the following features

* Acting as a replacement for Samsung SBL bootloader
* Booting instead of linux kernel from Samsung SBL bootloader
* Booting ANDROID! boot.img format images
* Booting custom kernels from emmc (/sdcard)

[U-boot source code](https://github.com/Ksys-labs/uboot-tuna.git)

Take a look at the video showing u-boot booting Android without Samsung SBL bootloader!

[U-boot demo](http://youtu.be/tcrNbwwPBkI)

To build an image that can be flashed to the kernel partition (as opposed to replacing the SBL bootloader), edit the **include/configs/omap4_tuna.h** and comment out the **#define TUNA_SPL_BUILD**

If you put the u-boot instead of the kernel partition, put put the kernel in either u-boot or android image format to **/boot/vmlinux.uimg** inside the android system partition.

U-boot also supports booting custom kernels from the emmc (/sdcard partition). The kernel has to be put to /boot/vmlinux.uimg on /sdcard or /data/media/boot/vmlinux.uimg in the device root. Hold down the volume down key to boot a custom kernel

Currently u-boot has issues with reading EXT4 file system, large files (>10MB) mostly fail to load. So if you need them, check out the **tuna_fosdem_hacks**, format the cache partition to ext2, and put the kernel to /media/boot/vmlinux.uimg on the cache partition.

RIL (Radio Information Layer)
====================

Overview
---------------------

For a number of reasons, we wanted to replace the proprietary modem binaries with an open source implementation

* It is impossible to put a closed binary into TCB (trusted computing base).
* Having source code would allow to split modem-specific code into a separate daemon shared by multiple virtualized linuxes.

The modem inside Samsung Galaxy Nexus and Samsung Galaxy S2 is the Infenion XMM6260 modem running a custom baseband firmware from Samsung. The physical transport is different on these devices (S2 uses USB EHCI, while Nexus is using HCI which is a special kind of a fast serial port). Software-wise, the modem and the RIL exchange the packets of a HDLC-encoded data. These packages do not contain standard GSM AT commands, but the proprietary binary commands of the Samsung IPC protocol. XMM6260 uses SIPCv4 protocol.

The [Replicant](http://replicant.us/) project already had a working implementation for the older phones (namely, the non-galaxy Nexus) so all we had to do was to write the firmware loader for the modem.

We're nice guys and have contributed our work back to Replicant so now there's full modem support with open-source drivers.

Since Replicant is under heavy development, we have a 'stable' mirror of the code in our repository which is tested to compile and work successfully.

Samsung IPC Library
---------------------

[libsamsung-ipc](https://github.com/Ksys-labs/libsamsung-ipc.git)

Here is our patched libsamsung-ipc library which is implementing the SIPC protocol support and is not tied to a particular RIL implementation (Android, Ofono etc)

Our contribution includes:

* Samsung Galaxy S2 support for XMM6260
* Samsung Galaxy Nexus support for XMM6260
* Fixing numerous bugs in EFS/RFS subsystem (IMEI, SIM contacts)

Samsung RIL
---------------------

[samsung-ril](https://github.com/Ksys-labs/samsung-ril.git)

Our additions include.

* USSD Support for Galaxy Nexus
* UTF8 (UCS2) support for USSD (tested with cyrillic russian messages)
* GPRS Fixes and support for Galaxy Nexus
* RSSI (signal strength) indicatior fixes

Samsung RIL Client
---------------------

[samsung ril client](https://github.com/Ksys-labs/samsung-ril-client.git)

RIL Client is a library which allows the GPS, NFC and Sound drivers to access the modem for certain power-management features. Our library is based on the Replicatn library, with the following additions.

* Sound support for Galaxy Nexus
* Stub routines to allow NFC and GPS drivers work
* Can serve a drop-in replacement for the proprietary RIL client, while being compatible with all the proprietary binaries which depend on RIL.

GPS Proxy
====================

As a part of the project to isolate trusted and untrusted stuff, we have implemented a daemon and the library that proxify the GPS interface for the Android OS. The rationale behind developing them was that reverse-engineering the Sirf Star 4 driver was unrealistic.

[RPC Library](https://github.com/Ksys-labs/libstc-rpc.git). THe RPC Library can exchange arbitrary binary messages in both directions via a single socket. It is similiar to MSGPack, but much simpler. It uses select() calls to avoid polling.

[GPS Proxy](https://github.com/Ksys-labs/gps-proxy.git)

The GPS proxy consists of a proxy library that implements the GPS interface and the daemon which loads the proprietary GPS binary. The proxy library and the daemon communicate via a socket using the RPC Library. The idea is that the untrusted binary driver can be run in an isolated l4linux container while being isolated from the trusted Android instance by an RPC transport. Using fixed-size RPC messages serves as a protection from stack smashing techiques.
