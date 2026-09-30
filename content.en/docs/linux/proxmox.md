---
title: Understanding Proxmox
weight: 90
---

# Understanding Proxmox

The [two homelab servers]({{< relref "/posts/homelab/umbau-proxmox" >}}) are going to run
Proxmox. That also makes me curious about what sits behind the installer: if it is based on
Debian, where does Debian end and Proxmox begin? Could I build something along those lines
myself?

Yes: a small Debian variant of your own is an achievable learning project. Looking at the
individual parts helps explain how much more work goes into something like Proxmox.

## What Proxmox VE is

Here, Proxmox means **Proxmox Virtual Environment**, or **PVE**: a Debian-based platform for
virtual machines and Linux containers. It is normally installed directly on the server.
The physical machine is the **host** or **node**; the virtual systems running on it are its
**guests**. A web interface is one way to manage them. Proxmox combines existing
virtualization technology with its own management software.
[Proxmox: feature overview](https://proxmox.com/en/products/proxmox-virtual-environment/features)

The version numbers belong to different projects: **Proxmox VE 9 is based on Debian 13
“Trixie”**. Proxmox ships its own adapted Linux kernel. It is still Linux; the Proxmox
developers select and maintain a variant that suits their platform.
[Proxmox VE 9.0: technical foundations](https://www.proxmox.com/en/about/company-details/press-releases/proxmox-virtual-environment-9-0)

## Virtual machines and containers

A **virtual machine** gets virtual hardware: processors, memory, disks and network cards.
It boots its own operating system with its own kernel. **QEMU** provides the virtual machine;
**KVM** in the Linux kernel uses the CPU's virtualization features to accelerate execution.
A guest might see a DVD drive when what sits behind it is simply an ISO file.
[QEMU: system emulation and accelerators](https://www.qemu.org/docs/master/system/introduction.html),
[Proxmox: QEMU/KVM](https://github.com/proxmox/pve-docs/blob/master/qm.adoc)

An **LXC container** shares the host's kernel instead. Linux isolates its processes and their
view of the system; mechanisms called *cgroups* control resource access. A container has
its own programs and system files, but does not boot a kernel of its own.
[Proxmox: containers and isolation](https://github.com/proxmox/pve-docs/blob/master/pct.adoc)

| Aspect | Virtual machine | LXC container |
|---|---|---|
| Kernel | its own guest kernel | the host's kernel |
| Operating system | Linux or Windows, for example | Linux userspace, such as Debian |
| Resource needs | an additional complete guest system | usually lower |
| Suitable learning project | booting a Linux ISO you built | running a Linux service with its own packages |

A VM is therefore a good fit for experimenting with a distribution of your own: it also
lets you test how your image boots.

## How the management layer is organised

The web interface does not execute the guests itself. Services behind it have different
responsibilities:

| Component | Responsibility |
|-----------|----------------|
| Web interface and API | accept management requests |
| `pveproxy` | provides HTTPS access on port `8006` |
| `pvedaemon` | handles API requests that need elevated permissions |
| QEMU/KVM and LXC | run virtual machines and containers respectively |
| `pmxcfs` at `/etc/pve` | exposes Proxmox configuration files and distributes them within a cluster |

`pveproxy` forwards privileged requests to the local `pvedaemon`. `pmxcfs` stores
configuration, such as a VM's settings; its virtual disk is stored separately.
[API access](https://github.com/proxmox/pve-docs/blob/master/pveproxy.adoc),
[API daemon](https://pve.proxmox.com/pve-docs/pvedaemon.8.html),
[Configuration filesystem](https://github.com/proxmox/pve-docs/blob/master/pmxcfs.adoc)

Clicking **“Start”** on a VM follows roughly this path:

1. The browser sends an authenticated API request to the host.
2. The management layer checks permissions and reads the VM configuration.
3. It starts QEMU with the intended resources, drives and network connections.
4. The guest boots its own operating system with the help of QEMU and KVM.

The interface describes and controls the operation. The virtualization components handle
the actual execution.
[Proxmox: API and web interface](https://github.com/proxmox/pve-manager),
[VM configuration and execution](https://github.com/proxmox/pve-docs/blob/master/qm.adoc)

### Storage, networking and multiple hosts

Virtual disks live on a configured **storage backend**. Depending on the backend, they
might be image files in a directory or logical volumes. Proxmox provides a common management
layer for these different storage types; choosing which one to use remains part of setting
up the host. [Proxmox: Storage Manager](https://github.com/proxmox/pve-docs/blob/master/pvesm.adoc)

A Linux bridge such as `vmbr0` acts like a virtual switch. Guests' virtual network cards
and a physical network card on the host can connect to it, giving guests access to the home
network. VLANs can be assigned to match the
[existing network layout]({{< relref "/docs/network/vlan-layout" >}}).
[Proxmox: networking](https://github.com/proxmox/pve-docs/blob/master/pve-network.adoc)

Multiple hosts can be joined into a **cluster** for shared management. Two freshly installed
servers do not yet form a cluster. Highly available guests require further decisions about
storage and quorum: the required majority of votes in the cluster. Shared management and
automatic recovery after a host failure are different functions.
[Proxmox: Cluster Manager](https://github.com/proxmox/pve-docs/blob/master/pvecm.adoc)

The `xlab` cluster is now configured, with `pve01`, `pve02` and an external quorum vote
on the DNS server. The setup is documented in
[Two-node Proxmox cluster with a QDevice]({{< relref "/docs/linux/proxmox-cluster" >}}).

## How Debian becomes Proxmox

The **kernel** manages the processor, memory and hardware, among other things. A
**distribution** assembles a usable system around it: programs, libraries, package
management, defaults and a way to install and update everything. Debian already does this
work for a large selection of software.
[Debian: about Debian](https://www.debian.org/intro/about)

A distribution built on that foundation can reuse large parts of it and focus on its own
purpose. Debian calls these independent variants **derivatives** and explicitly encourages
their development. [Debian: derivatives](https://www.debian.org/derivatives/)

With Proxmox, three parts make that visible:

- **Its own software:** The `pve-manager` repository contains management code, the web
  interface, service definitions and, under `debian/`, the files used for Debian packaging.
  Source code becomes installable packages.
  [pve-manager source code](https://github.com/proxmox/pve-manager)
- **Its own package repositories:** The installed system obtains Debian packages and
  Proxmox packages from the appropriate repositories. APT can then update both parts.
  [Proxmox: package repositories](https://github.com/proxmox/pve-docs/blob/master/pve-package-repos.adoc)
- **A combined installer:** The installation medium turns those parts into a system for
  the intended purpose, bringing together Debian, the Proxmox kernel and the management
  tools. [Proxmox: installation](https://proxmox.com/en/products/proxmox-virtual-environment/get-started)

Development therefore includes integrating and maintaining those parts. New releases have
to handle existing configurations, updates need to work together, and failures need to be
traceable. Putting a new logo on an ISO would only be a small part of it.

## What a small distribution of your own can mean

For a personal project, I would increase the scope gradually:

| Stage | Your own result |
|-------|-----------------|
| Set up Debian automatically | a recorded package selection and configuration for fresh installations |
| Build a live image | a bootable ISO containing those packages and your own files |
| Maintain a Debian derivative | your own packages, releases, repositories and a tested update path as well |

The first stage alone may be enough for a homelab. For a bootable system of your own, the
second is a manageable starting point. Debian provides **live-build** to assemble an image
from a configuration you describe.
[Debian Live: tools and basics](https://live-team.pages.debian.net/live-manual/html/live-manual/the-basics.en.html)

If you want to learn how compilers, libraries and a base system are assembled from source,
you can work through **Linux From Scratch**. That is a different learning path; you do not
need to build those foundations yourself for a personal Debian variant.
[Linux From Scratch](https://www.linuxfromscratch.org/lfs/view/stable/)

## First learning project: a Debian live image of your own

The example is a small console system with `curl`, `htop` and `tmux`, plus a text file that
identifies the image. These instructions are intended for a **separate Debian 13 VM on
amd64 with internet access and `sudo` configured**. The build needs several GB of free disk
space. There is no need to install the build tools on the Proxmox host itself.

### The tool and its configuration

Install `live-build` from Debian in the build VM.
[Debian Live: installation](https://live-team.pages.debian.net/live-manual/html/live-manual/installation.en.html)

```sh
sudo apt update
sudo apt install live-build

mkdir xlab-live
cd xlab-live

lb config --distribution trixie --architectures amd64 --binary-images iso-hybrid --debian-installer none
```

This creates `config/`. The codename `trixie` fixes the chosen Debian release;
`iso-hybrid` sets the output format. `--debian-installer none` explicitly builds a live
system without an installer.
[lb_config for Debian 13](https://manpages.debian.org/trixie/live-build/lb_config.1.en.html)

### Your own packages and files

```sh
mkdir -p config/package-lists config/includes.chroot/etc

printf '%s\n' curl htop tmux > config/package-lists/xlab.list.chroot
printf '%s\n' 'xlab-live: my first Debian image' > config/includes.chroot/etc/xlab-release

sudo lb build
```

The file ending in `.list.chroot` selects the additional packages.
`config/includes.chroot/` corresponds to the eventual root directory: the custom file
becomes `/etc/xlab-release` in the live system.
[Package selection](https://live-team.pages.debian.net/live-manual/html/live-manual/customizing-package-installation.en.html),
[Custom files](https://live-team.pages.debian.net/live-manual/html/live-manual/customizing-contents.en.html)

### What happens during the build

The tool assembles a Debian base system in a working directory, using `debootstrap` for that
step. The selected packages and files are added. A kernel, an early boot environment called
the **initramfs**, and a bootloader are included to make the image bootable. At startup,
`live-boot` finds the included root filesystem and adds a writable layer on top.
`live-config` then handles tasks such as creating the live user.
[debootstrap](https://manpages.debian.org/trixie/debootstrap/debootstrap.8.en.html),
[Debian Live: how the tools fit together](https://live-team.pages.debian.net/live-manual/html/live-manual/overview-of-tools.en.html)

### Trying the result

After a successful build, `live-image-amd64.hybrid.iso` is in the working directory.
Attach it as a virtual DVD drive to a new test VM and boot from it.
[Debian Live: building and testing](https://live-team.pages.debian.net/live-manual/html/live-manual/the-basics.en.html)

Inside the booted test VM:

```sh
cat /etc/xlab-release
command -v curl htop tmux
```

This is a Debian live system assembled to your specifications. Changes made during a
session are lost on reboot unless persistence is configured separately. Installing it to
a disk would require another step involving an installer.
[Debian Live: persistence](https://live-team.pages.debian.net/live-manual/html/live-manual/customizing-run-time-behaviours.en.html),
[Debian Live: installer](https://live-team.pages.debian.net/live-manual/html/live-manual/customizing-installer.en.html)

## What remains before it becomes a maintained distribution

Next, I would put the build configuration in Git and repeat the build from a fresh VM.
That records how the image is made. Byte-for-byte reproducibility additionally requires
control over package versions and the build environment; a codename alone does not freeze
the included packages.

Your own programs and lasting system customizations can later become `.deb` packages.
Your own repositories can deliver them to systems that are already installed. A published
derivative also needs tested updates, security updates and clearly assigned maintenance
responsibilities. [Debian: derivative guidelines](https://wiki.debian.org/Derivatives/Guidelines)

For my homelab, the first concrete goal would be an ISO that boots in a Proxmox VM and
contains exactly the selected tools. A management interface for virtual machines would
then be a separate software project — using the same foundation, but requiring considerably
more development of my own.
