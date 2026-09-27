---
title: Homelab, step four — tidied up, Proxmox is next
date: 2026-09-26
---

# Homelab, step four — tidied up, Proxmox is next

[Part three]({{< relref "/posts/homelab/vlan-umbau" >}}) was about VLANs, firewall rules and a
network that did not always do what it was supposed to during the rebuild. By now, the network
rebuild is finished for the time being. Everyone at home has internet access again. After
the recent changes, that is the most important news for now.

Since then, I have also completely rearranged and tidied up the homelab itself. The hardware
collection has grown in the process: a second home-built server and two old rack servers
have joined it.

## A second server from old parts

I put together a second server from old consumer hardware parts. They get another job in the
homelab, and I will be installing Proxmox on this machine as well.

That leaves two machines ready for the next step: the existing server and the new build.
Both are due for a fresh Proxmox installation. The hardware is assembled; the installation
is still ahead.

## Two rack servers will have to wait

I also bought two old rack servers. They come with a problem that is difficult to ignore
once you put them somewhere: they are too loud.

So those two will get a different home later on. The plan is to build a wooden rack in the
garage and eventually move the rack servers there. Building the rack and moving them is a
separate project for later; the other two servers come first.

## Next: Proxmox on both machines

The network is in place, the homelab is tidy and the hardware for the next step is assembled.
Now it is time to set up both Proxmox servers from scratch.

I have been using MediCat USB, which is built on Ventoy, to install Windows and Proxmox
for a while. It keeps the installation ISOs together on one USB drive, ready to select at
boot. I will be using it for these two servers as well. How MediCat and Ventoy fit together
and how to add the installers to the drive is covered in
[MediCat & Ventoy]({{< relref "/docs/linux/medicat-ventoy" >}}).

What Proxmox actually is, how it builds on Debian and what a first step towards a small
distribution of your own might look like is covered in
[Understanding Proxmox]({{< relref "/docs/linux/proxmox" >}}).

The last post still talked about a single hypervisor. Now there are two machines waiting to
be set up and another two waiting for a place in the garage. For now, the task at hand is
enough: a clean Proxmox installation on both servers. How that goes will be in the next post.
