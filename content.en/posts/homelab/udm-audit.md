---
title: Homelab, step six — firewall rules and lower Wi-Fi utilisation
date: 2026-10-04
---

The six network segments now have the firewall rules that were missing after the VLAN
rebuild. Guests, IoT and kids have limited access, and router administration is blocked from
those networks. The DNS server is also back on a gigabit link, and the two access points
share the work on 2.4 GHz more effectively.

<!--more-->

## What was missing between the segments

Since the [VLAN rebuild]({{< relref "/posts/homelab/vlan-umbau" >}}), infrastructure, servers,
clients, kids, IoT and guests have had their own networks. Checking the Dream Machine revealed
that all six still belonged to the same firewall zone, *Internal*, with
*Internal → Internal: Allow All*. The VLANs existed, but routed traffic between them was
unrestricted.

All 63 existing firewall policies came from the system. The planned
[rules between segments]({{< relref "/docs/network/firewall-policies" >}}) had never been
implemented. Client isolation on the guest WLAN did not replace rules blocking access to
servers or router administration either.

The internet connection was in better shape: no port forwards, UPnP off, IPv6 off on the WAN,
and unsolicited inbound connections blocked by default rules. Firmware, automatic updates
and backups were also in order. The gap was in access within the house.

## Five zones for six networks

The networks are now grouped by their level of trust:

| Zone | Networks |
|------|----------|
| Internal | Infrastructure and servers |
| Trusted | Clients including work machines, phones and laptops |
| Restricted | Kids |
| IoT | Smart home devices and TVs |
| Guest | Guests |

The following permissions apply to new connections:

| From | To | Allowed |
|------|----|---------|
| Trusted | Internal, IoT and UDM | all services |
| Restricted, IoT and Guest | DNS server `10.10.10.3` | DNS on port 53, TCP and UDP |
| Restricted, IoT and Guest | UDM | DHCP for address assignment |
| Restricted and IoT | UDM | additionally mDNS for device discovery |
| all | internet | outbound connections |

Other new connections between these local zones are blocked. Replies to permitted
connections can return. A client can therefore contact a service on the IoT network without
allowing an IoT device to initiate a connection to the work machine.

Infrastructure and servers remain together in *Internal*, so they are still not isolated
from each other. Permissions from *Trusted* are also broad: they are not restricted to
individual management services or devices.

The new explainer [Understanding Firewalls]({{< relref "/docs/network/firewall-grundlagen" >}})
covers what a firewall checks and how it recognises replies.

## Access to the UDM has its own rules

The newly created zones initially had unrestricted access to *Gateway*. That zone represents
the UDM itself, including both its services and its management interface. Blocking access to
servers therefore does not also protect the router's interface.

Guests, IoT and kids can now reach only the gateway services listed above. The management
interface is blocked from those networks and remains accessible from *Trusted*. SSH on the
console is disabled. Intrusion detection was switched from notification only to blocking.

## The TVs now belong to the IoT network

Both TVs previously belonged to the client network. They now receive addresses from
`10.10.30.0/24` and follow the IoT rules. They stay on the existing `xlab` WLAN: a device-specific
VLAN assignment places them on the other network.

This avoided moving them to the 2.4 GHz only `xlab-IoT` SSID. The Wi-Fi name and password on
the TVs stayed the same; their network permissions changed.

## Gigabit for the DNS server and less airtime used

The DNS server's link was running at only 100 Mbit/s. After replacing the patch cable, it
negotiates 1000 Mbit/s again. There was no need to replace the network card.

On 2.4 GHz, both access points had automatically selected channel 11, which was also busy in
the neighbourhood. They now use fixed channels 1 and 6. The minimum data rate was raised from
1 to 6 Mbit/s so that slow transmissions consume less airtime. After the changes, measured
channel utilisation was 17 and 30 % respectively, down from around 60 %.

On 5 GHz, channel width stayed at 40 MHz. Wider channels were not needed for this rebuild.

## What has been checked and what remains open

After moving the networks into their zones in stages, UDM, DNS, Proxmox and internet access
were checked from the work network. The rule blocking IoT traffic towards the servers
recorded its first hits shortly after the change.

That shows traffic is being blocked by this rule. A direct test of the blocked access from
a device on the guest network is still outstanding.

Two other items remain open: the connection to the second switch still runs at 100 Mbit/s
over the cable in the wall. And all six segments depend on a single machine for DNS. A second
resolver is next.
