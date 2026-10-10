---
title: "Install Linux on External SSD"
date: 2022-08-03T18:03:48+08:00
draft: false

categories: ['Tutorial']
tags: ['Linux', 'Tutorial']
toc: false
author: ""
---
Things I might forget about later since my last memory swap is sinking (if I may call it that way)

<!--more-->

1. Install [Virtualbox](https://www.virtualbox.org/wiki/Downloads)
2. Install *Oracle VM VirtualBox Extension Pack*. This is needed to connect `USB2.0` and `USB3.0` drives to the virtual machine.
3. If host os = linux, make sure your user is in the `vboxusers` group using `sudo usermod -aG vboxusers $USER`,then reboot to refresh the groups.
4. Download linux distro of choice. Make sure its 64-bit for UEFI support.
5. Create a new virtual machine in VirtualBox.
6. Don't add a virtual hard disk, as we will be mounting the external SSD into the VM to act as the disk.
7. Go into the VM settings and enable EFI support (`Under System > Enable EFI`).
8. Under USB, select the USB controller, try `USB3.0` first and if it doesn’t work, try `USB2.0`.
9. Boot up the virtual machine, when asked, supply the ISO image to use.
10. Mount the USB hard drive by clicking the USB icon at the bottom of the virtualbox window and selecting the USB drive (if you don’t see the drive, try changing your USB controller in the options, see step 8).
11. Start the installer as normal.
12. During disk partitioning, ensure you have at least 3 partitions: `EFI Boot Partition` (`FAT32`, at least 200MB), / (`Root mountpoint`, `EXT4`), Swap (Should be around the size of computer’s RAM capacity).
13. Install as normal.
14. Shutdown the VM and the computer, try to boot the external SSD on the computer.

## Result
- Bootable system with UEFI support that works on different computers.
- Makes no changes to the computers internal drive during installation.
- Tested with Ubuntu and Kali.


