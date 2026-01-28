---
layout: post
title:  Recover from Linux update disaster
date:   2022-09-02 01:08:00 +0800
categories: document
tag: Tutorial
topping: false
---

* content
{:toc}

Disaster today. My Arch Linux suddenly shut down for no reason during upgrade. I’m 100% sure it’s not a battery issue, I still have 23% left when I started the upgrade - In any case. When I rebooted the system and got pass GRUB’s boot screen. It just says *“can’t find linux-zen”* and *“kernel needs to be loaded”*. Shouldn’t be a big deal as all my data are backed up. But still, pain in the ass.

Fixing Linux with broken system files                                                    {#Fixing}
====================================
I installed multiple kernels, and one of my kernels is missing. Probably there are a lot of writes that aren’t synced to disk. Fortunately, the situation is only seeming scary. The solution is easy. Get an Arch Live USB. `arch-chroot` into it. Reinstall all packages and pray documents are intact.

So I got another computer and made myself a Live USB. Booted my laptop with it. And arch-chroot into it. pacman complains libzstd is missing. Ohhh… Fine. pacman has a `-r` parameter to specify an alternative root directory for this exact situation. Repairing a broken pacman in my case is a simple command from the live system.

```bash
pacman -S pacman zstd -r /mnt --overwrite="*"
```

Then `arch-chroot` again. Nice, now pacman works again. Now I’m ready to reinstall everything, and replace all broken system files.

```bash
pacman -Qqn | pacman -Sy - --overwrite="*"
```

Done and reboot. I saw GRUB successfully loading the kernel and… Display Manager loaded! My system is back to life.

This kind of control is exactly why I love Linux. There’s no chance I can fix a similar problem with Mac or Windows.
