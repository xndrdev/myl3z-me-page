---
title: "02 18 Min Grub 2"
date: 2023-11-25T20:26:54+01:00
draft: false
tags: 18min server grub2
---

# GRUB 2
Todays topic was GRUB and bootloader. Of course I won't get to much into the deep, but just 
to read some of the explanations at least help me to understand it more.

What is GRUB? It stands for Grand Unified Bootloader and the current major version is 2.

## Bootloader
What is a bootloader? It's basically a program that loads the operating system. After the BIOS
make it's system checks the hard drives are red and the first part of the hard drive stores the
bootloader. 

The section is too small to have a lot of code, so it's used to reload other code which lies on
the hard drive to actually start the operating system.

## Installation
You can install it with `grub2-install /dev/sda` where **/dev/sda** the location is where the bootloader
will be installed.

## Configuration
GRUB 2 is highly configurable. The configuration file can be found in `/boot/grub2/grub.cfg`. But you
shouln't change the file by hand. The file is auto-generated. But you can change the scripts in `/etc/grub.d`.

This directories are used to change the configuration. After changing you need to update with `update-grub`, so that
the configuration file is written.