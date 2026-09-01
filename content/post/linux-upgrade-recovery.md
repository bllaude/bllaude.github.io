+++
date = '2022-09-01T21:51:17+08:00'
draft = false
title = 'Linux Upgrade Recovery'
tag = 'linux'
+++

*“Can’t find linux-zen”* and *“kernel needs to be loaded”*.
<!--more-->

## Fixing Linux with broken system files
I installed multiple kernels, and one of them is missing. There are probably a lot of writes that aren’t synced to disk. Luckily the situation only looks scary but the solution is easy: 
Get an Arch Live USB, `arch-chroot` into it, reinstall all packages, and pray documents are still intact.

So I got another computer and made myself a live USB. Booted my laptop and `arch-chroot` into it. `pacman` complains `libzstd` is missing. pacman has a `-r` parameter to specify an alternative root directory for this exact situation. Repairing a broken pacman in my case is a simple command from the live system.
```bash
pacman -S pacman zstd -r /mnt --overwrite="*"
```

`arch-chroot` again, now pacman works. Now reinstall everything, and replace all broken system files.

```bash
pacman -Qqn | pacman -Sy - --overwrite="*"
```

Done and reboot. `GRUB successfully loading the kernel and… **Display Manager loaded!** My system is back to life.

This kind of control is exactly why I love Linux. There’s no chance I can fix a similar problem with Mac or Windows.