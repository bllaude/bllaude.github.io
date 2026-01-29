---
layout: post
title:  Kali upgrade crash fix
date:   2026-01-28 22:00:00 +0800
categories: document
tag: troubleshoot
topping: false
---

* content
{:toc}

There's a particular kind of dread that comes with a Linux upgrade gone wrong. This happened to me after Kali Linux upgrade crashed midway. 

Symptom                                                    {#Symptom}
====================================
After rebooting:
- Black screen
- Blinking cursor
- No graphical login
- No error messages

The desktop never appeared, even though the kernel booted. You've probably seen this at least once if you've used Kali extensively.

What went wrong (technically)                                                    {#Wrong}
====================================
Kali is a rolling distribution. During upgrades, a lot happens at once:
- Kernel updates
- Display manager updates (gdm3 / lightdm)
- Desktop environment changes
- Firmware and systemd updates

When the upgrade crashed, `dpkg` was left mid-transaction.

The result:
- Packages were half-configured
- The display manager failed to start
- X/Wayland never launched

The system was simply unable to launch a GUI. This distinction is important because it indicates that recovery is possible.

#1 Attempt                                                    {#Attempt}
====================================
I tried using a TTY before going into recovery mode or reinstalling anything.
`CTRL + Alt + F2`

That single detail tells you almost everything:
- System booted
- Kernel is fine
- The problem is userspace (desktop stack)

After logging in, recovery is straightforward

Fixing the system                                                    {#Fix}
====================================
The goal is simple:
- Finish what the interrupted upgrade started
- Restore the display manager
- Reboot cleanly

From a root shell:
```bash
dpgk --configure -a
apt --fix-broken install
apt update
apt full-upgrade -y
```

Unfinished package states are cleared up in this way.

Reinstall the desktop stack after that.

For GNOME (Kali default)
```bash
apt install --reinstall gdm3
kali-desktop-gnome
systemctl enable gdm
```

For XFCE (which in my case)
```bash
apt install --reinstall
lightdm kali-desktop-xfce
systemctl enable lightdm
```

Finally, rebuild boot artifacts:
```bash
update-initramfs -u -k -all
update-grub
reboot
```

The login screen reappeared as if nothing had happened after the reboot.
