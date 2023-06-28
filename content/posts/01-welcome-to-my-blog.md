---
title: "01 - Hacking Lab - The idea"
date: 2023-01-16T00:51:23+01:00
draft: false
tags: hacking lab
---

# Welcome

Hi! Hello! Welcome!

My name is xander and I like cyber security, software development and servers.

This blog will be my personal blog about cyber security, software development and servers. 

Let's see!

Leave me a comment if you have questions or just want to say hi.

# First project: Hacking Lab

What is a hacking lab?

It's basically a playground for my cyber security stuff. To get a better understand I want to build my own infrastructure and then attack it.

One of the first steps will be a small Windows environment with Active Directory.

## Disclaimer
This is just a personal blog. I have no idea what I'm doing here, because I've never worked as an Administrator. So if it makes no sense or you have any suggestions, please let me know in the comments.

## Promox

I will use the hypervisor Proxmox for my hacking lab. The computer I'm using is a very old gaming computer I bought about 6 years (I guess) ago. But it's still running and is good enough for my hacking lab. 

Installing Promox is easy. Download the ISO image https://www.proxmox.com/en/, put it on an USB flash drive with balenaEtcher https://etcher.balena.io/ , boot from it and run the installation routine. For now I will not go into detail, but if you're interested let me know, then I will write more about it :)

Proxmox is installed! We can start to create our virtual machines.

## Environment

To build a Windows infrastructure and figure out what we need, I have to think about the environment first. Let's pretend there is a WordPress Agency called **AnimalPress**.

### Domain
Our domain will be called **ad.animalpress.local**

### Users

Let's come up with some Users:

- Tony Penguin - CEO - 192.168.25.10
- Frank Bear - Accounting - 192.168.25.11
- Lisa Cat - Developer - 192.168.25.12
- Holly Bird - Developer - 192.168.25.13
- Steven Shark - Developer - 192.168.25.14
- Amber Owl - Project Management - 192.168.25.15
- Xander Wolf - System Administrator - 192.168.25.16

### Groups

- Management
- Accounting
- IT

### Computers

We will start very simple and with a basic infrastructure:

- Domain Controller - will also be the DNS Server - 192.168.25.1
- Tony Penguin - 192.168.25.10
- Frank Bear - 192.168.25.11
- Lisa Cat - 192.168.25.12
- Holly Bird - 192.168.25.13
- Steven Shark - 192.168.25.14
- Amber Owl - 192.168.25.15
- Xander Wolf - 192.168.25.16

For now there are no other requirements. No Home Office, no printers, etc.

## Next Steps
At first I will create eight new virtual machine on my proxmox node. One of them will be Windows Server 2022 as Domain Controller and seven Windows 11 computers for the employees. In the next part I will start with the Domain Controller. At first I need to find out what a Domain Controller exactly is, what it's purpose and how to configure it properly.