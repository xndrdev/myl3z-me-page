---
title: Homelab, Step One — A Thin Client Running Debian 13
date: 2026-08-16
---

# Homelab, Step One — A Thin Client Running Debian 13

I am building a homelab. Not a rack in the basement, but something that grows one piece at a
time: one machine, one service, write it down, keep going.

The first node is a thin client. For a start that is the right class of hardware — silent,
barely any power draw when running around the clock, and cheap second-hand. For the services
I want on it first, it is plenty.

It now runs **Debian 13**, installed from the netinst image, no desktop. Debian because the
stable branch does exactly what you want from a machine that is supposed to just keep
running: little movement, long support windows, no surprises on upgrade. Netinst because a
box without a screen should not have anything installed that I did not ask for.

I wrote the installer stick from the terminal — the steps are under
[Bootable USB from the Terminal]({{< relref "/docs/linux/bootable-usb" >}}), and the commands
involved are explained in
[lsblk, umount, dd, sync]({{< relref "/docs/linux/disk-commands" >}}).

After the install the box got a fixed address, written to `/etc/network/interfaces` and
applied with `systemctl reboot`. Under DHCP that address might be a different one after the
next router reboot — the wrong footing for a machine that is about to handle name resolution
for the network. How the configuration is put together, and one pitfall in the nameserver
line, is written up under
[Static IP with ifupdown]({{< relref "/docs/linux/static-ip" >}}).

## Next up: Pi-hole

The first service will be Pi-hole. DNS is the obvious place to start: it is the one service
every other device on the network benefits from immediately, without configuring anything on
those devices. It is also an honest test of the rest of the setup — when the DNS server goes
down, the household notices within minutes. A service that has to hold up 24/7 forces clean
foundations from day one.

Notes will follow once it runs.
