---
title: MediCat & Ventoy
weight: 15
---

# MediCat & Ventoy

I have been using **MediCat USB** to install Windows and Proxmox for a while. It is built on
**Ventoy**: several installation images live on one USB drive as ISO files, ready to select
at boot. That also fits the next step in my
[homelab]({{< relref "/posts/homelab/umbau-proxmox" >}}), where both servers are due for a
fresh Proxmox installation.

## What MediCat and Ventoy each do

**Ventoy** makes the USB drive bootable and provides the menu for selecting images.
After the initial setup, multiple ISOs can be copied onto the drive. Adding another version
does not require writing a new image over the entire drive each time.
[Ventoy: feature overview](https://www.ventoy.net/en/)

**MediCat USB** builds on this with a collection of diagnostic, repair and maintenance tools.
These include backup, partitioning and data recovery utilities as well as live systems.
The drive can hold installation media alongside tools for dealing with a computer that no
longer boots properly.
[MediCat: overview](https://medicatusb.com/docs/medicat/general/overview/),
[MediCat: building on Ventoy](https://medicatusb.com/docs/medicat/installation/manualinstall/)

| Component | Role |
|-----------|------|
| Ventoy | boots the selected image from the USB drive |
| MediCat | adds the collection of maintenance and recovery tools |
| Windows or Proxmox ISO | provides the respective operating system installer |

Ventoy alone is enough for booting several installation ISOs. MediCat adds the extra tools
to that setup.

## Preparing the drive

MediCat requires a drive of **at least 32 GB** and recommends USB 3.0 or newer. Additional
Windows and Proxmox ISOs need more free space, so the drive should be large enough for the
whole collection.
[MediCat: requirements](https://medicatusb.com/docs/medicat/installation/requirements/)

For a new drive, start with the
[official MediCat download](https://medicatusb.com/docs/medicat/installation/download/)
and [installation guide](https://medicatusb.com/docs/medicat/installation/manualinstall/).
The manual setup installs Ventoy first, formats the large data partition as NTFS, then
extracts the contents of the MediCat archive into its root directory. The separate Ventoy
boot partition stays in place. In this setup, MediCat is a collection of files; the
downloaded archive is not a bootable ISO.

> [!WARNING]
> The initial setup formats the drive. Back up its data and check the target drive first.
> Once MediCat is set up, new installation ISOs are simply copied onto it as files.

## Adding Windows and Proxmox

I download the installation images separately from their respective vendors:

- **Windows:** On [Microsoft's Windows 11 download page](https://www.microsoft.com/en-us/software-download/windows11),
  choose the ISO for the appropriate architecture and language.
- **Proxmox VE:** Choose the appropriate installer from the
  [official ISO downloads](https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso).
  The SHA256 checksum listed there can be used to verify the download.

Copy the ISO files unchanged onto the large data partition of the prepared drive, for
example into the included `OSimages` folder. Do not extract them or write them to the
device with `dd`. Ventoy also finds images in subdirectories. A newer installer can be
added later by replacing the corresponding file.
[Ventoy: copying images](https://www.ventoy.net/en/doc_start.html)

The steps on the target computer are then:

1. Safely eject the drive after copying and connect it to the target computer.
2. Select the USB drive in the computer's boot menu.
3. Start the desired Windows or Proxmox ISO from the MediCat/Ventoy menu.
4. Follow the installer and select the intended internal target drive there.

From this point, the operating system's own installer handles the installation. The USB
drive is the boot medium; the selected target drive may be erased during installation.
For what Proxmox provides once it is on the server, see
[Understanding Proxmox]({{< relref "/docs/linux/proxmox" >}}).

## If the drive does not boot

Whether an image boots also depends on the firmware, boot mode and image itself. Ventoy
supports Legacy BIOS and UEFI; **Secure Boot** may require extra steps such as enrolling
a Ventoy key. Support varies between machines. Ventoy describes the current procedure in
its [Secure Boot notes](https://www.ventoy.net/en/doc_secure.html).

For a single Linux installer, writing an image directly to a drive remains an option, as
described in [Bootable USB from the Terminal]({{< relref "/docs/linux/bootable-usb" >}}).
Use a separate drive for that: writing a hybrid ISO with `dd` would overwrite the existing
MediCat/Ventoy setup.
