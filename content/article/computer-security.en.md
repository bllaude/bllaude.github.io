---
title: "Computer Security"
date: 2026-10-10T12:08:43+08:00
draft: false

categories: []
tags: []
toc: false
author: ""
---
- hardening your system
  - full-disk encryption with a long passphrase (FileVault / LUKS), + a firmware password or secure boot with your own keys
  - [FIDO2](https://www.yubico.com/authentication-standards/fido2/) hardware keys for login, sudo, and SSH instead of TOTP or SMS
  - egress-aware firewalling per application
    - [Little Snitch](www.obdev.at/products/littlesnitch/index.html)/[LuLu](https://objective-see.org/products/lulu.html) on macOS,
    - [OpenSnitch](https://github.com/evilsocket/opensnitch) on Linux
  - kernel and userland hardening on Linux: `sysctl` (`kernel.kptr_restrict`, `kernel.dmesg_restrict`, unprivileged user namespaces off where unused), + audit with [lynis](https://github.com/cisofy/lynis)
  - randomize MAC addresses on untrusted Wi-Fi, and disable Bluetooth and AirDrop discovery when idle
  - compartmentalize high-risk work (Qubes OS, or dedicated VMs per trust level)
  - verify package and release signatures, and prefer reproducible builds where available
- hardening network access
  - SSH: Ed25519 keys only, PasswordAuthentication no, AllowUsers, and consider [fwknop](https://github.com/mrash/fwknop) *(port knocking)* in front of it
  - WireGuard -> remote access rather than exposing services directly
  - encrypted DNS (DoT/DoH) through a resolver you control, with DNSSEC validation
    - why? mitm, cache poisoning
  - monitor Certificate Transparency logs for your own domains (crt.sh, certspotter)
  - disable IPv6 where unused so it can’t bypass your v4 firewall rules
