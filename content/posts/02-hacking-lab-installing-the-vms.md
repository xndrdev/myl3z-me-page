---
title: "02 - Hacking Lab - Installing the Vms"
date: 2023-06-28T22:11:36+02:00
draft: false
---

## Installation
Installing Promox was quite easy. Download, copy with balenaEtcher and boot from the flash drive. Click, Click, Click, Done.

Following the mini plan I've made, I'm starting with the installation of the VMs. Downloading Windows for Evaluation. I need Windows Server 2022, which will run the Domain Controller. And for every Workstation I will use Windows 11 Enterprise, since it's also available for Evaluation in the Evaluation Center.

The first problem occured: I can't start the freshly created machines, because something with Virtualization and my mainboard needs to be configured. Easy fix: Flip the switch for Virtualization in the BIOS. Done. Restart.

Upload the ISO Image to Proxmox, mount it and start the installation process of Windows Server 2022.

### Specification
For the Server I will use 1TB of my space, 4GB of memory and 4 CPUs. The Workstations will have the same memory and CPU size, but only 200GB of space.

For now I only created 1 Server and 2 Workstations, because I'm not able to run all of them at the same time, since my Server only have 16GB of RAM :(

## Windows Server 2022
The installation went well. The server is running! 

From now on I've basically no idea what I should do next, so I will watch a basic YouTube video to learn it. Looking at my mini plan I realize that I need a Domain Controller and a Domain. So that's basically my next step: Creating a new Domain with a Domain Controller.

Before we start, I change the Name of the Computer to **AnimalPress-DC**, so I can identify it in the network and also I'm setting up a static IP address for this server, which is **192.168.25.1**.

Default Gateway will be my local router **192.168.2.1** and Subnet Mask **255.255.0.0** so that the devices are still in the same network. This was buggy until I've deactivated IPv6 on both machines. Maybe I need to investigate a bit more, but not now. The DNS will be the same IP Address **192.168.25.1**. Somehow the access to the internet is lost. I think that the IP Addresses are not in the same network. Maybe I'm not understanding how the subnet masks work, but I will figure it out.

... Watching the video: https://www.youtube.com/watch?v=pRf_uU0vrMM ...

Huh! It's easier than I've expected. The Server Manager will open and on the Dashboard there are some open todos: "Configure this local server".

Let's do it! I will add some roles and features to my Windows Server. The server need roles to add the correct features. For the start we will need the **Active Directory Domain Services** and the **DNS Server**. Just checking the box and adding the features. On the last page I click on **Install** and reboot the server after a while.

## Promotion
On the top right corner a notification arrives. It says that the installation is done and that the server can be promoted to domain controller, which basically means: **configuration**. Since we don't have a domain, I select **Add a new forest** and type in my domain name **ad.animalpress.local**. Adding other basic stuff, like in the video and at the end click on **install**.

Installation and Promotion is done. We have a basic Domain Controller!

## Installation of the Workstations
Follow the same process as for the Server to install Windows 11 on the Workstations. After the installation is done, the configuration begins and it will differ from the server setup. At some point we will need to join the Workstation to the domain. 

To identify the computer in the network, I change the name to **TONY-PENGUIN**.

Before I can join them, the IP Address needs to be set manually. For the first Workstation of **Tony Penguin** I've decided to use **192.168.25.10**. Important: The DNS also need the IP Address of the Domain Controller, which is **192.168.25.1**

After setting up everything I'm finally able to join the domain. Whoohoo!

Next Problem is, that the Machines have no access to the internet anymore. But I will figure it out in the next part! After that we are ready for some first basic attacks to get a foothold into the system.