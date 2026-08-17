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
box without a screen should not have anything installed that I did not ask for. Part of that
minimalism is that `sudo` was not included — it was added afterwards, along with enabling my
user for it ([Adding sudo]({{< relref "/docs/linux/sudo-setup" >}})).

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

That made the next step obvious: public key onto the machine, an alias `dns01` in the local
`~/.ssh/config`. The thin client sits there without screen or keyboard, so everything from
here happens over `ssh dns01`. What goes into that config block and why is written up under
[SSH Config and Key Login]({{< relref "/docs/linux/ssh-config" >}}).

## The first service: Pi-hole

Pi-hole is running. DNS was the obvious place to start: it is the one service every other
device on the network benefits from immediately, without configuring anything on those
devices. It is also an honest test of the rest of the setup — when the DNS server goes down,
the household notices within minutes. A service that has to hold up 24/7 forces clean
foundations from day one.

The install itself is a one-liner and done in a few minutes. Version 6 is a different system
from the one most guides out there describe, though: it installs as a Debian package, brings
its own web server, and puts everything into a single `pihole.toml`. What else it brings along
— an NTP server, for one — is easiest to spot in the list of ports occupied afterwards. How it
is put together, the values of this particular install and the commands for day-to-day use are
written up under [Pi-hole as a DNS Server]({{< relref "/docs/linux/pihole" >}}).

More interesting than the install is what it does not take care of: the router still hands out
its own address as the nameserver, so every query keeps going past Pi-hole. The thin client
itself still asks the router rather than itself. And the upstream the installer suggests is
Google — a filter that keeps reporting every query there is only half a win. A service that
runs and a service that is used are two different things.

One last thing the machine got is a name of its own, recorded in Pi-hole itself:
`dns01.xlab.internal`. `dns01` for the role, `xlab.internal` as the zone for the homelab. The
suffix is a deliberate pick — `.internal` has been reserved for private networks since 2024,
whereas `.local` already belongs to mDNS and `.lan` is written down nowhere. That settles the
scheme everything else will be addressed under, before there is more than one machine.

## Next up: the network

The next step is not the next machine but the UDM. The reason is the first of the three open
points: as long as the gateway hands out its own address as the nameserver, not a single
device in the house asks the filter that is now running. One field in the DHCP settings
decides whether the past few days achieved anything at all.

The second reason is order. A hypervisor wants to know at install time which segment it sits
in, which address it gets and whether its bridge runs tagged. Rebuild the network afterwards
and you configure it a second time — and anyone touching the bridges of a machine without a
screen tends to lock themselves out doing it. Address ranges, segments and names are better
settled while there is one machine rather than five.

Proxmox comes after that. So far the homelab is one machine with one service on it, and every
further service would share the same Debian install: shared packages, shared outages, no clean
way back. Virtualisation turns that around — one base, separate machines on top, each one
backed up, cloned and thrown away on its own.

Notes will follow once it runs.
