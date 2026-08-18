---
title: Network Layout with VLANs
weight: 20
---

# Network Layout with VLANs

The network currently consists of a single segment: `10.10.0.0/16`, everything inside it, from
the gateway through the thin client to the television. That works as long as you own one
machine. It also means every device reaches every other one directly — the robot vacuum reaches
the work laptop, the TV reaches the UDM's management interface.

> [!NOTE]
> This note describes the **plan**, not the current state. So far only the DNS entry in DHCP is
> done ([Handing Out Pi-hole via DHCP]({{< relref "/docs/network/udm-dhcp-dns" >}})).

## Why now rather than later

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
| **five segments** | infrastructure, servers, clients, IoT, guests | Separates the three things that genuinely belong apart: network management, self-built services, foreign firmware. The ruleset stays manageable |
| five plus lab | additionally a playground that may only reach the internet | Reasonable, but there is no occasion for it yet. It can be added later as another VLAN without touching anything else |

Five it is.

## The scheme

| Segment | VLAN | Network | Gateway | What belongs in it |
|---------|------|---------|---------|--------------------|
| Infrastructure | 1 (untagged) | `10.10.1.0/24` | `10.10.1.1` | UDM, switches, access points — everything you manage the network itself with |
| Servers | 10 | `10.10.10.0/24` | `10.10.10.1` | `dns01`, later Proxmox and its VMs |
| Clients | 20 | `10.10.20.0/24` | `10.10.20.1` | Laptops, phones, work machines, consoles |
| IoT | 30 | `10.10.30.0/24` | `10.10.30.1` | Smart home, TV, cast devices, printer |
| Guests | 40 | `10.10.40.0/24` | `10.10.40.1` | Visitors, isolated from each other |
| *(Lab)* | *50* | *`10.10.50.0/24`* | — | reserved, not created yet |

The third octet matches the VLAN ID. That is not a technical requirement but a reading aid:
`10.10.30.47` is recognisably an IoT device without looking anything up.

**Why `/24` instead of staying on `/16`:** A `/24` holds 254 hosts — more than a household will
ever need — and confines the broadcast domain to one segment. More importantly, the netmask is
the very place where separation happens: as long as every device lives in `10.10.0.0/16`, they
consider each other neighbours and talk directly, bypassing the gateway. Without the trip
through the gateway, no firewall rule applies.

**Why the scheme looks like this:** Both existing addresses stay valid. The UDM keeps
`10.10.1.1` and lands in the infrastructure segment, `dns01` keeps `10.10.10.3` and lands in
the server segment. Netmask and gateway change, not a single address — which keeps every
existing note accurate.

## What changes on dns01

In `/etc/network/interfaces` ([Static IP with ifupdown]({{< relref "/docs/linux/static-ip" >}})):

| Line | before | after |
|------|--------|-------|
| `address` | `10.10.10.3/16` | `10.10.10.3/24` |
| `gateway` | `10.10.1.1` | `10.10.10.1` |

The machine itself needs to know nothing about VLANs as long as its switch port carries the
server VLAN **untagged**. Tagging is only required for the Proxmox host, which will serve
several segments at once.

## The pitfall: listeningMode

This is where the rebuild otherwise falls apart. Pi-hole is set to
`dns.listeningMode = LOCAL`, which answers only queries from networks the machine itself holds
an address in. Today, thanks to `/16`, that is the entire network — after the cut it is only
`10.10.10.0/24`.

The consequence: clients in VLAN 20, 30 and 40 send their queries and Pi-hole discards them
without comment. No error in the log, no hint, just a network without name resolution.

```sh
sudo pihole-FTL --config dns.listeningMode ALL
sudo systemctl restart pihole-FTL
```

That makes FTL answer routed queries from the other segments too. The warning from
[Pi-hole as a DNS Server]({{< relref "/docs/linux/pihole" >}}) still applies unchanged: `ALL`
turns the service into an open resolver the moment it is reachable from outside. Behind a
gateway with no port forward on 53 that is harmless — but the responsibility has moved from
Pi-hole into the firewall.

## What has to be allowed between segments

The base rule is denial: segments may reach the internet, but not each other. Exceptions are
needed, and the first one matters most.

| From | To | For |
|------|----|-----|
| all segments | `10.10.10.3` port 53 (TCP/UDP) | **Name resolution.** Without this rule the household has no DNS after the rebuild |
| all segments | `10.10.10.3` port 123 (UDP) | Time, if Pi-hole's NTP server is to be used |
| Clients | IoT: printer, NAS, cast devices | Printing and streaming. Targeted at addresses and ports, not as a blanket opening |
| Clients | Infrastructure, Servers | Administration — web interfaces, SSH. If you want to be strict, limit this to individual addresses |
| IoT, Guests | anywhere but the internet | nothing |
| all | back onto existing connections | *established/related*, otherwise no reply works |

**mDNS across segment boundaries.** Cast devices, AirPlay and Sonos find each other via
multicast, and multicast stops at the segment boundary. A phone on the client network simply no
longer sees the TV on the IoT network. The UDM ships an mDNS repeater for this (listed as
*Multicast DNS* or *mDNS* in the network configuration) which forwards the announcements
between the selected networks. Without it, separating clients from IoT is not sustainable in
day-to-day use.

> [!NOTE]
> The repeater only makes devices *visible*. The connection that follows still needs the
> firewall exception — the two get confused easily when the TV shows up in the list but refuses
> to be controlled.

## The order of operations

The order is not a matter of taste; it is the difference between an evening and an evening with
a monitor and keyboard in the server cupboard.

1. **Create the new networks** while the existing one stays untouched. VLANs, subnets, DHCP
   ranges — nothing is attached to them yet.
2. **Set the DNS entry in every new network.** New networks start on `Auto` and would otherwise
   bypass the filter.
3. **Switch Pi-hole to `listeningMode = ALL`** — before the first device lands in another
   segment, not after.
4. **Move `dns01`.** The delicate step: the switch port profile and the machine's IP
   configuration have to match at the same moment, and the SSH session drops in between. The
   way to do it without a monitor is to stage the change and schedule the reboot:

   ```sh
   sudo shutdown -r +2
   ```

   During those two minutes the switch port is moved to the server VLAN. The machine comes back
   up in its new segment. If you can reach the box physically, attaching a monitor is the more
   honest route.
5. **Map the Wi-Fi SSIDs to their networks** and let the access points restart.
6. **Distribute the remaining switch ports** across clients and IoT.
7. **Shrink the original network to `/24`.** Last, because from here on anything still sitting
   in the old range is unreachable.
8. **Apply the firewall rules** and test from within every segment.

> [!WARNING]
> The UniFi devices themselves — switches and access points — live on the infrastructure
> network. Changing that network's address range or VLAN makes them lose their connection to
> the controller and show up as *disconnected*. That usually resolves itself; but anyone
> rebuilding the firewall at the same time will be hunting the fault in two places at once.

## Verify

From one device in each segment:

```sh
ip -br a                                     # is the address in the right network?
ping -c1 10.10.20.1                          # own gateway reachable
dig +short @10.10.10.3 example.com           # DNS across the segment boundary
dig +short @10.10.10.3 dns01.xlab.internal   # local name resolves
ping -c1 10.10.10.3                          # should fail from IoT and guests
```

On `dns01`, watch whether queries arrive with their real client address:

```sh
pihole -t
```

Traffic between segments is routed, not NATed — the log keeps showing individual devices rather
than the gateway address. If the filter stays quiet, the cause is either the firewall rule from
step 8 or the `listeningMode` from step 3.

## What is still open afterwards

**The Proxmox uplink.** The port for the future hypervisor gets a profile of its own: server
VLAN untagged for management, client and IoT VLAN tagged for VMs that belong there. The profile
can be created the same evening while the networks are open anyway — then the port is ready
when the machine arrives.

**The second Pi-hole.** A segment layout does not change the fact that a single thin client
carries name resolution for the whole house. A second resolver is the first worthwhile guest on
the hypervisor.

**The lab VLAN.** Reserved as VLAN 50; it gets created once there is something to isolate.

**IPv6.** Unresolved, and segments make the question bigger rather than smaller: each network
would get its own prefix, and the nameserver announcement in the router advertisements has to
point at Pi-hole everywhere, or the filter covers each segment only halfway.
