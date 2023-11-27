---
title: "03 - 18 min - Initial Ramdisk"
date: 2023-11-26T23:25:27+01:00
draft: false
---

# initrd

Today's topic is about **initrd** - the Initial Ramdisk.

We're still in the boot process. As I understand, the initial ramdisk contains all needed drivers and module to start the computer, before the 
computer has access to all drives.

The initrd is loaded by the bootloader, in our case it's GRUB2. Usually the initial ramdisk is created in the installation process of the
operating system, so it consists everything it needs to start the computer properly.

## Modification

We can modify it. Of course we can modify it. It's linux, so why shouldn't we?

There are different tools to do so. On Ubuntu / Debian you can use `mkinitramfs` or `update-initramfs`. I won't go into detail here, because
it's enough for me to know that this is existing.

## After the boot process
After the boot process is finished in which the kernel is loaded, the root filesystem mounted and all necessary loaded the `systemd` deamon takes
over.

# Change the structure

I think I will change the structure of the posts. For now I write for every 18 min day. But I would like to collect multiple days into one topic,
if the topic is big or interesting enough for me.

Since the next topic will have more pages in my book, it will take some days before a new post appears.