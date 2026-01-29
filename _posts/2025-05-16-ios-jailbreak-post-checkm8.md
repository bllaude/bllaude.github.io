---
layout: post
title:  iOS jailbreak post-checkm8
date:   2025-05-16 22:00:00 +0800
categories: document
tag: random
topping: false
---

* content
{:toc}
This follows Lars Fröder's [@opa334dev](https://infosec.exchange/@opa334) latest update about the status of the Dopamine jailbreak and the future of iOS exploitation. Lars and a few other committed developers have been working on this for the past few years by creating *"rootless"* jailbreaks that respect iOS' contemporary security posture while allowing users some degree of autonomy.

The issue isn't just *"Apple is locking things down"*, which has been the case for more than 10 years. The problem lies in the fact that Apple is now doing it well and that the opportunity for new public exploitation has completely closed.

## Here's why:
- No Public Exploits 
: It has been years since we last saw a functional exploit for A11+ devices. The ones that are available are either under NDA (held by commercial exploit brokers), private, or costly.
- Mitigations Everywhere
: Apple has used a variety of hardening methods, such as:
    - Pointer Authentication (PAC) — makes ROP/JOP chains much harder to build.
    - Page protections, KTRR, APRR — ensure code can't be written or executed outside of strict boundaries.
    - SEP & Secure Boot — hardware-level isolation that prevents persistent modifications.
- Hardware Tied Security
: Starting from A11 onwards, Apple began locking critical functions into non-exploitable, read-only hardware paths. You can't just *“bypass”* these with a clever kernel bug anymore.
- No Kernel Debugging
: On-device introspection (LLDB, KDP, etc.) is practically impossible now without a working jailbreak or developer entitlement. So you can’t even reverse-engineer the system efficiently.

The only viable jailbreaks left with persistence, stability, and root access are for A7 through A10X devices. These are affected by **checkm8**, a BootROM exploit discovered by axi0mX back in 2019.

Because BootROM code lives in read-only memory burned into the chip itself, Apple can’t patch it with software updates. That makes checkm8 permanent.

But, checkm8 does not affect devices that are A11 or later. And they never will be unless another BootROM-level vulnerability is found, which is highly unlikely considering Apple's level of hardware security maturity. Even if something were to be found, Apple’s increasingly aggressive revocation of researcher access, tighter lockdowns on debugging tools, and constant security updates mean it likely wouldn’t stay unpatched for long.

Is jailbreaking dead?                                                   {#Question}
====================================
Not exactly. But it might as well be for the majority of users, particularly those using the most recent iPhone models.
We got WinterBoard, Activator, f.lux, and whole ecosystems like Cydia and Sileo during the golden age of jailbreaking. Many of iOS' greatest features, including Control Center, custom widgets, and dark mode, were initially jailbroken concepts that it popularized.

But this year, jailbreaking is no longer a playground. The target audience for this security research project is getting smaller, with most of them still using outdated hardware. So if you have an iPhone 8 or later and are still hoping for a jailbreak for iOS 17 or later...you're most likely out of luck.
