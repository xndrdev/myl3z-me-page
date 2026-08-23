---
title: Network Layout with VLANs
weight: 20
---

# Network Layout with VLANs

The network used to consist of a single segment: `10.10.0.0/16`, everything inside it, from the
gateway through the thin client to the television. That works as long as you own one machine.
It also means every device reaches every other one directly — the robot vacuum reaches the work
laptop, the TV reaches the UDM's management interface.

Since 23 August 2026 there are six segments. This note describes the layout and the reasoning
behind it; the rebuild procedure is under
[Setting Up VLANs on the UDM]({{< relref "/docs/network/unifi-vlan-umbau" >}}), the terminology
under [Understanding VLANs]({{< relref "/docs/network/vlan-grundlagen" >}}).

## Why before the hypervisor

A hypervisor wants to know at install time which segment it sits in, which address it gets, and
whether its bridge runs tagged. Rebuilding the network afterwards means configuring it a second
time — on the bridges of a machine with no monitor attached, which is a reliable way to lock
yourself out.

While the homelab is one machine, the cut costs an evening. At five machines it costs a weekend
plus a list of things that no longer work afterwards.

## How finely to divide

Every boundary you draw has to be punched through again with exceptions — the printer should
stay reachable, the TV should still accept casts from a phone. Too many segments produce a
ruleset nobody holds in their head, and a ruleset nobody holds in their head gets opened up
wholesale at the first sign of trouble.

| Layout | Structure | Trade-off |
|--------|-----------|-----------|
| three segments | infrastructure and servers together, clients, IoT and guests together | Few rules, quick to build. Future VMs end up next to the management of the network hardware — exactly the neighbourhood you do not want on a hypervisor you experiment with |
| five segments | infrastructure, servers, clients, IoT, guests | Separates the three things that genuinely belong apart: network management, self-built services, foreign firmware. The ruleset stays manageable |
| **six segments** | additionally kids | Children's devices need different blocklists and time windows than the rest. Bound to a network that is cheaper to maintain than device by device |
| plus lab | additionally a playground that may only reach the internet | Reasonable, but there is no occasion for it yet. It can be added later as another VLAN without touching anything else |

Five were planned, six were built. The kids segment was added during the rebuild because the
cost of another network that evening was close to zero — the networks were open anyway, the
SSID had to be created anyway. Afterwards it would have cost a second evening.

## The scheme

| Segment | VLAN | Network | Gateway | What belongs in it |
|---------|------|---------|---------|--------------------|
| Infrastructure | 1 (untagged) | `10.10.1.0/24` | `10.10.1.1` | UDM, switches, access points — everything you manage the network itself with |
| Servers | 10 | `10.10.10.0/24` | `10.10.10.1` | `dns01`, later Proxmox and its VMs |
| Clients | 20 | `10.10.20.0/24` | `10.10.20.1` | Laptops, phones, work machines, consoles |
| Kids | 25 | `10.10.25.0/24` | `10.10.25.1` | Children's devices, own filter group and time windows |
| IoT | 30 | `10.10.30.0/24` | `10.10.30.1` | Smart home, TV, cast devices, printer |
| Guests | 40 | `10.10.40.0/24` | `10.10.40.1` | Visitors, isolated from each other |
| *(Lab)* | *50* | *`10.10.50.0/24`* | — | reserved, not yet created |

The third octet of the address matches the VLAN ID. That is not a technical necessity but a
reading aid: `10.10.30.47` is recognisably an IoT device without looking anything up.

**Why `/24` and not `/16` any longer:** a `/24` holds 254 hosts — more than a household will
ever need — and confines the broadcast domain to one segment. More importantly, the netmask is
the place where the separation actually happens: while every device sits in `10.10.0.0/16` they
consider each other neighbours and talk past the gateway. Without that detour no firewall rule
applies.

**Why the scheme is laid out this way:** both existing addresses stay valid. The UDM keeps
`10.10.1.1` and thus sits in the infrastructure segment, `dns01` keeps `10.10.10.3` and sits in
the server segment. Netmask and gateway changed, not a single address — which keeps every
existing note correct.

## What changed on dns01

In `/etc/network/interfaces` ([Static IP with ifupdown]({{< relref "/docs/linux/static-ip" >}})):

| Line | before | after |
|------|--------|-------|
| `address` | `10.10.10.3/16` | `10.10.10.3/24` |
| `gateway` | `10.10.1.1` | `10.10.10.1` |

The device itself needs to know nothing about VLANs as long as its switch port carries the
server VLAN **untagged**. Tagging is only needed by the Proxmox host, which has to serve
several segments at once.

## The pitfall: listeningMode

The point where the rebuild otherwise falls over. Pi-hole was set to
`dns.listeningMode = LOCAL`, which means it answers only queries from networks the machine
itself holds an address in. Thanks to `/16` that used to be the entire network — after the cut
it is only `10.10.10.0/24`.

The consequence would be: clients on VLAN 20, 25, 30 and 40 send their queries and Pi-hole
discards them without comment. No error in the log, no hint, just a network without name
resolution.

```sh
sudo pihole-FTL --config dns.listeningMode ALL
sudo systemctl restart pihole-FTL
```

With that, FTL also answers routed queries from the other segments. The warning from
[Pi-hole as a DNS Server]({{< relref "/docs/linux/pihole" >}}) applies unchanged: `ALL` turns
the service into an open resolver the moment it becomes reachable from outside. Behind a
gateway with no port forward on 53 that is uncritical — but the responsibility moves from
Pi-hole into the firewall.

## What is allowed between segments

The base rule is denial: segments may reach the internet, but not each other. The most
important exception is port 53 on `10.10.10.3` from every segment — without it the household
has no name resolution after the rebuild.

The full ruleset, including zones, ordering and the mDNS repeater, is under
[Firewalling Between Segments]({{< relref "/docs/network/firewall-policies" >}}).

## Verifying

From one device in each segment:

```sh
ip -br a                                     # is the address in the right network?
ping -c1 10.10.20.1                          # own gateway reachable
ping -c1 9.9.9.9                             # internet by IP
dig +short @10.10.10.3 example.com           # DNS across the segment boundary
dig +short @10.10.10.3 dns01.xlab.internal   # local name resolves
ping -c1 10.10.10.3                          # should fail from IoT and guests
```

On `dns01`, watch whether the queries arrive with their real client address:

```sh
pihole -t
```

Traffic between segments is routed, not NATed — the log keeps showing individual devices rather
than the gateway address. If the filter stays silent, the cause is either a firewall rule or
`listeningMode`.

## What is still open

**The devices themselves.** The scaffolding is up, it is not occupied: much of the IoT hardware
still sits where it was before the rebuild. Every device has to be moved into the new SSID by
hand — the part that takes longer than the entire network configuration before it.

**The Proxmox uplink.** The port for the hypervisor gets its own profile: server VLAN untagged
for management, client and IoT VLAN tagged for VMs. The host itself currently still sits in the
infrastructure segment and needs to move to the server VLAN.

**The second Pi-hole.** A segment layout does nothing about a single thin client carrying name
resolution for the whole house. The second resolver is the first sensible guest on the
hypervisor.

**The lab VLAN.** Reserved as VLAN 50, to be created once there is something to isolate.

**IPv6.** Unresolved, and segments make the question bigger rather than smaller: each network
would get its own prefix, and the nameserver announcement via router advertisement has to point
at Pi-hole everywhere, or the filter covers only half of each segment.
