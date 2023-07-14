---
title: "03 - Hacking Lab - Scanning the network"
date: 2023-06-30T20:11:05+02:00
draft: false
---

# Scanning the network

Note: Before going into the fun stuff. Good news! I've fixed the internet connection problem, by using IP addresses that can communicate. Somehow 192.168.2.1 and 192.168.25.1 can't communicate, even if I'm using 255.255.0.0 as a subnet mask. That means that the network is really big now. If you would like to learn more about networking and subnetting, leave me a comment :)

## Update the VMs
To have fun with the fun stuff, I want to add some configurations to my domain controller. I create some SMB Shares, a new low privilege user called **holly.bird**, which will only be a Domain User.
And add some Groups, like **CEO**, **Accounting**, **Development** and **System Administration**.

Nothing special. Easy setup with a few click with the Server Manager on the Domain Controller.

## Preparation
To do some actual hacking I will use kali. It's a Debian-based Linux distribution packed with all the good stuff we need for hacking. Of course I could use any Linux distribution and install all the tools I need, but I really like kali and I want to focus on the hacking for now.

At first I need a second level Hypervisor to run kali on my Workstation. I choose VirtualBox. It's free and it works :) There are other second level Hypervisors you could use. Or even install it on you computer bare metal.

So I download the kali image, VirtualBox and install everything. After some time I have a running kali VM on my ubuntu Workstation. Let's get the party started.

![Kali Linux](/images/posts/03_hacking_lab_scanning_the_network/kali_linux.png)

## Scanning and Enumeration
Let's pretend I made it into the network somehow. There are multiple scenarios and at some point I will write about them. But for now, we're already in the network. But we don't know anything about this network yet. So let's scan it. In the first part **Scanning & Enumeration** my job is to find as much information as possible.

### nmap
One of the famous scanning tools, which is already installed in kali, is **nmap**. To start a scan I use the command `nmap -sn 192.168.0.0/16`. This will scan the whole network. Since it's a really big network, this will take a while. Time for a coffee break.

My goal is to find hosts that are up. After some time nmap will be finished. It also have found some machines. The 192.168.2.x is basically my infrastructure, so I will not test them :D

But after a long while, there are some ip addresses but not from 192.168.25.x, which are the machines I want to attack later! For now I will use other tools to get more hosts.

Are there alternatives to nmap?

### masscan
Another scanner, which is also already installed, is **masscan**.

It seems like it's a really fast scanner, so let's try it. With the command `sudo masscan -p20,21-23,25,53,80,110,111,135,139,143,443,445,993,995,1723,3306,3389,5900,8080 192.168.0.0/16` I can scan for all the open ports in my network.

To even speed this up I will scan only for port 53, which is DNS. Of course there could be other DNS Server with this port open, but like I've said, my network is very simple and I want to focus on the methodology and the tools.

After just a few minutes, masscan has found the ip address of the domain controller. Yey!

![Masscan results](/images/posts/03_hacking_lab_scanning_the_network/masscan_results.png)

The big goal here is not to find the domain controller only, but to find all hosts, so that we have some hosts for enumeration.

### bettercap
Last tool I want to look into is **bettercap**. It was not installed on my machine, but it's an easy step.

After installation, just run `sudo bettercap` and you get an interactive console. To start scanning ports I use `syn.scan 192.168.0.0/16 53`. This will look for hosts with the port 53.

![Bettercap progress](/images/posts/03_hacking_lab_scanning_the_network/bettercap_progress.png)

I got the Domain Controller ip address. In a real pentest I would not only check port 53, but also the other ports that can be interesting for hacking.

## Conclusion
Depending on how noisy you want to be scanning the network can be done with these tools. It's of course a very basic beginning, since my network is built simple. Sometimes one tool is not working as intended and doesn't provide useful information, so it's better to have some alternatives ready to go! Don't forget to save all ip addresses of the found hosts.

In the next episode I will try to get as much information as possible about the hosts to look for possible vulnerabilities.