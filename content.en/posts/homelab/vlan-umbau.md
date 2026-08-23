---
title: Homelab, step three — cutting the network into segments
date: 2026-08-23
---

# Homelab, step three — cutting the network into segments

[Part two]({{< relref "/posts/homelab/dns-upstream" >}}) ended with a clean setup and a plan for
the next step: five segments instead of one flat `/16`. The plan had been sitting there
finished for days, with a numbered order for the rebuild evening.

Step one of that order worked. After that the network started talking back.

## Why cut it up at all

Until that evening the home network was a single room. Everything in it: the Dream Machine, the
thin client running the DNS filter, phones, the work laptop, the TV, the printer, a handful of
smart home devices. Every device could reach every other one directly. Not because it was
configured that way somewhere, but because nothing prevented it.

That is the default of virtually every home network, and it never stands out as long as nothing
happens. The real objection is not an attack scenario but a question of jurisdiction: a TV has
no reason to see the router's management interface. A robot vacuum has no reason to see the
work laptop. These devices run firmware I have not read, that rarely gets updates, and that
keeps running long after the vendor stops supporting it. Giving them access to everything was
never a decision — it was just never made.

Then there is the practical occasion. A hypervisor is moving in next, on which VMs will appear
and disappear again. A machine you experiment with does not belong on the same network as the
devices you manage the network with. And a hypervisor wants to know at install time which
segment it sits in — rebuild the network afterwards and you configure it a second time, on the
bridges of a machine with no monitor attached.

With one machine the cut costs an evening. With five it costs a weekend.

## Six rooms instead of one

One network became six, each with its own job: **infrastructure** for everything you manage the
network with. **Servers** for self-built services. **Clients** for work devices. **IoT** for
foreign firmware. **Guests** for visitors. And **kids**.

The sixth was not planned. It came up mid-rebuild, out of a simple calculation: the networks
were open anyway, the SSIDs had to be created anyway — another segment cost five minutes that
evening. Afterwards it would have been a second evening. Children's devices need different
blocklists and different hours than the rest of the household, and both are cheaper to maintain
bound to a network than device by device.

The addresses follow a scheme that explains itself: the third octet is the VLAN number.
`10.10.30.47` is recognisably an IoT device without looking anything up. That is not
engineering but a reading aid for the moment you are digging through a log file. The full
layout, including the trade-off on how finely to divide, is under
[Network Layout with VLANs]({{< relref "/docs/network/vlan-layout" >}}).

## The plan was in the wrong order

The written procedure began by creating the new networks while leaving the existing one
untouched. The old `/16` was to be shrunk last — last, because from that point everything still
holding an address in the old range becomes unreachable.

That cannot be sustained. UniFi refuses any new network whose range overlaps an existing one,
and `10.10.20.0/24` does sit inside `10.10.0.0/16`. It did not affect one segment but all of
them: every new network lives inside the old one.

So the most cautious step of the plan had to move to the front. The rebuild begins with exactly
the action that breaks the most — shrinking the default network from `/16` to `/24` and pulling
the floor out from under every device outside `10.10.1.0/24`. There is no variant where you
edge in carefully.

That is the kind of flaw in your own plan you do not find by thinking harder. It is also absent
from the guides you orient yourself by, because it only occurs when rebuilding an existing
network rather than setting one up on a greenfield.

## Everything routes, nothing arrives

The most expensive mistake of the evening came later and looked like success at first. The six
networks were up, the gateways answered, name resolution worked across segment boundaries. Only
no device in the new segments reached the internet.

No timeout at the gateway. No DNS error — names resolved cleanly. Packets went out, nothing
came back.

I searched in the wrong direction for a long time here, because the symptom looks like a DNS
problem and DNS had been the topic of the previous two weeks in this homelab. You search where
you last worked.

The cause was a rule you never encounter while creating a network: a NAT rule named *LAN to WAN
Masquerade*, whose *source* was set to the infrastructure network — to `10.10.1.0/24`, which is
to say the one network that existed before the rebuild. Packets from the new segments were
routed correctly and, on the way out towards the internet, **not masqueraded**. They left with
a private source address, and that is where every journey ends.

The fix was a single field: *source* set to *any*. The half hour before it went into realising
that a network has to do three things independently — route, resolve, masquerade — and that the
failure of the third looks like a problem with the second. The test that separates them is a
ping to a bare IP: if the gateway answers and DNS resolves while `9.9.9.9` is unreachable, it
is not the name.

Both pitfalls, along with the procedure, are now under
[Setting Up VLANs on the UDM]({{< relref "/docs/network/unifi-vlan-umbau" >}}).

## Four wireless networks and one compromise

On a wire the switch port decides which segment a device lands in. On WLAN there is no port —
there the SSID decides. One wireless network became four: `xlab` for clients, `xlab-IoT`,
`xlab-Kids`, `xlab-Guests`.

The IoT network is deliberately the weakest of them: 2.4 GHz only, WPA2 instead of WPA3,
Protected Management Frames disabled. Each of these settings lowers the security level, and
each is necessary because a good share of smart home hardware otherwise fails to connect
reliably — radio modules that cannot do WPA3, and firmware that drops the link a few minutes in
when PMF is enforced.

This is where segmentation shows its actual worth. I do not have to make these devices more
secure than they can be. I only have to make sure there is a boundary behind their network. The
weaker encryption is tolerable because whatever breaks in there finds a TV and a printer — and
no work laptop. The details are under
[Mapping WLANs to VLANs]({{< relref "/docs/network/wlan-ssid-vlan" >}}).

## What a VLAN does not do

The thing I take away most is a matter of vocabulary. A VLAN separates nothing.

It only ensures devices no longer see each other directly and that their traffic has to travel
via the gateway. There — and only there — the firewall decides who reaches whom. Without a
ruleset, a cleanly cut network routes every packet just as obligingly as before, only with more
intermediate steps.

There is a second point I had not cleanly separated before: the netmask is part of the
separation, not cosmetics. As long as a device is configured with `/16` it considers everything
in `10.10.x.x` its direct neighbour and asks via ARP instead of the gateway. Only with `/24`
does a place come into being where a rule can apply at all. VLAN and subnet are two different
layers that merely look like one because you sensibly map them one to one. What sits behind
that is under [Understanding VLANs]({{< relref "/docs/network/vlan-grundlagen" >}}), the
ruleset under
[Firewalling Between Segments]({{< relref "/docs/network/firewall-policies" >}}).

## What is not finished

The scaffolding is up and verified: six segments, each with its own gateway, Pi-hole answering
queries from every network, name resolution working across the segment boundaries, the client
network no longer able to reach the management of the infrastructure.

It is not occupied. Much of the IoT hardware still sits where it was before the rebuild, and
each of those devices has to be moved into the new SSID by hand. For phones that is a minute;
for devices that can only be coaxed into a pairing mode through a vendor app, it is the part
that takes longer than the entire network configuration before it.

And the hypervisor, for whose sake the whole rebuild was brought forward, still sits in the
wrong segment — on an address in the infrastructure network, with a hostname from a time when
the machine had no job yet. It needs to move to the server VLAN, and its switch port needs the
profile with tagged VLANs for the VMs to come.

That will be step four. This time with a network that already knows where it belongs.
