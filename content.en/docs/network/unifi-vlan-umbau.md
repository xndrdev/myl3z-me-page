---
title: Setting Up VLANs on the UDM
weight: 30
---

# Setting Up VLANs on the UDM

The order in which the rebuild from a flat `10.10.0.0/16` to six segments actually worked —
including the two places where the planned order did not. The layout behind it is under
[Network Layout with VLANs]({{< relref "/docs/network/vlan-layout" >}}), the terminology under
[Understanding VLANs]({{< relref "/docs/network/vlan-grundlagen" >}}).

## First: shrink the old network

The plan was to create the new networks while leaving the existing one untouched, and to shrink
the old network to `/24` only at the very end. That cannot be sustained.

**UniFi refuses any new network whose range overlaps an existing one.** While the default
network is `10.10.0.0/16`, `10.10.20.0/24` sits inside it — and creation fails with an overlap
error. Every single new segment is affected, because all of them live inside `10.10.0.0/16`.

So the step has to move right to the front:

1. *Settings → Networks →* open the default network
2. Change the range from `10.10.0.0/16` to `10.10.1.0/24`, gateway stays `10.10.1.1`
3. Adjust the DHCP range to the new `/24`

From that moment on, anything that held an address outside `10.10.1.0/24` is unreachable. On a
`/16` with a grown DHCP pool that is half the household.

> [!WARNING]
> Devices do not lose their address immediately — they keep the old one until the next renew
> and until then sit on a network that no longer exists. If you do not want to wait, restart
> the affected devices. Deliberately leave your own workstation for last.

In practice this means the rebuild starts with the step that breaks the most. That is
uncomfortable, but there is no alternative.

## The new networks

*Settings → Networks → Create New Network*, once per segment. The fields that matter:

| Field | Value | Why |
|-------|-------|-----|
| *Router* / *VLAN ID* | 10, 20, 25, 30, 40 | the VLAN number from the layout |
| *Gateway IP / Subnet* | `10.10.<vlan>.1/24` | third octet equals VLAN ID, as a reading aid |
| *DHCP Mode* | Server | every segment brings its own range |
| *DHCP Name Server* | **Manual**, `10.10.10.3` | otherwise the segment silently drops out of the filter |

The last field is the one people forget. New networks start on `Auto`, which enters the UDM as
the nameserver and routes around Pi-hole — with no error, no log entry, just no filtering. The
reasoning is under
[Handing Out Pi-hole via DHCP]({{< relref "/docs/network/udm-dhcp-dns" >}}).

## The most expensive mistake: the NAT rule

The networks were up, the gateways answered, DNS resolved — and not a single device in the new
segments reached the internet. No timeout at the gateway, no DNS problem: names resolved,
packets went out, nothing came back.

The cause lies in a rule you never encounter while creating a network. Under
*Settings → Routing → NAT* there is a masquerade rule named **LAN to WAN Masquerade** — and its
*Source* was set to the infrastructure network `10.10.1.0/24`.

What follows is exactly the observed picture:

- Packets from `10.10.20.0/24` are **routed** correctly, the gateway accepts them
- On the way out towards the WAN they are **not masqueraded**, so they keep their private
  source address
- The upstream drops them, or cannot route the reply back

The fix is one field: set *Source* to **Any**. Alternatively a separate masquerade rule per
segment, which gives more control and costs more upkeep.

> [!NOTE]
> Whether the rule is visible at all depends on the UniFi version. On older builds the UDM
> masquerades implicitly and the rule does not exist — in which case this step simply does not
> apply. If it is there, check its *Source* before the first devices move.

The symptom is expensive precisely because it looks like a DNS problem, and that is where you
search first. The test that tells them apart:

```sh
ping -c1 10.10.20.1                  # own gateway     -> works
dig +short @10.10.10.3 example.com   # DNS             -> works
ping -c1 9.9.9.9                     # internet by IP  -> fails
```

If the gateway answers and DNS resolves while a bare internet IP is unreachable, it is not the
name and not the route — it is NAT.

## The rest of the evening

Once shrinking, networks and NAT are in place, the order is uncritical:

1. **Pi-hole to `listeningMode = ALL`** — before the first device lands in another segment.
   Details under [Network Layout with VLANs]({{< relref "/docs/network/vlan-layout#the-pitfall-listeningmode" >}})
2. **Move `dns01`** — netmask to `/24`, gateway to `10.10.10.1`, switch port to the server VLAN
   untagged
3. **Map SSIDs to networks** — see [Mapping WLANs to VLANs]({{< relref "/docs/network/wlan-ssid-vlan" >}})
4. **Assign port profiles to the remaining switch ports**
5. **Set firewall policies** — see [Firewalling Between Segments]({{< relref "/docs/network/firewall-policies" >}})

> [!WARNING]
> Switches and access points sit on the infrastructure network themselves. While its range
> changes they lose contact with the controller and show up as *disconnected*. That usually
> resolves itself once they pick up an address in the new range. Rebuild the firewall at the
> same time and you will be searching in two places at once.

## Port profiles

*Settings → Profiles → Port Profiles*. A profile describes which VLANs a switch port carries:

| Profile | Native VLAN | Tagged | For |
|---------|-------------|--------|-----|
| Clients | Clients (20) | — | workstations, docks, consoles |
| IoT | IoT (30) | — | printer, TV, smart home bridge |
| Servers | Servers (10) | — | `dns01` |
| AP uplink | Infrastructure (1) | *Allow All* | access points serving several SSIDs |
| Hypervisor | Servers (10) | Clients (20), IoT (30) | Proxmox host with VMs in several segments |

The AP uplink is where you are most likely to lock yourself out: if a VLAN is missing from the
tagged list, its SSID does not work — devices associate but never get an address. *Allow All*
is the pragmatic choice on a home network, because every future SSID then works without
touching the port.

## Verifying

Once from each segment, ideally in this order:

```sh
ip -br a                                     # address in the right network?
ping -c1 10.10.20.1                          # own gateway
ping -c1 9.9.9.9                             # internet by IP (tests NAT)
dig +short @10.10.10.3 example.com           # DNS across the segment boundary
dig +short @10.10.10.3 dns01.xlab.internal   # local name
ping -c1 10.10.10.3                          # from IoT and guests: must fail
```

On `dns01`, watch whether queries arrive with their real client address:

```sh
pihole -t
```

Traffic between segments is routed, not NATed — the log keeps showing individual devices rather
than the gateway address. If the gateway address shows up there, traffic is taking a NAT rule
it should not.
