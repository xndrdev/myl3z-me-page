---
title: Understanding VLANs
weight: 15
---

# Understanding VLANs

A switch forwards frames by MAC address and knows nothing about networks. Everything attached
to it sits in the same broadcast domain: an ARP request reaches every port, every device sees
every other one. That is precisely what segmentation ends — and a VLAN is the means by which a
single switch pretends to be several.

## The tag

The basis is IEEE 802.1Q: four additional bytes in the Ethernet frame, right behind the source
MAC. Among other things they carry twelve bits of VLAN ID, so 4094 usable numbers — `0` and
`4095` are reserved.

Nothing more happens. A VLAN is not a tunnel and not encryption, but a number in the frame that
all participating devices agree to honour. Anyone who bypasses the switch and taps the wire
directly sees the traffic unchanged. The separation is an organisational one that the switch
enforces — not a cryptographic one.

## Untagged and tagged

The distinction that everything hinges on in practice:

| Port type | What travels the wire | Used for |
|-----------|----------------------|----------|
| **untagged** (access) | Frames without a tag. The switch adds it on ingress and strips it on egress | End devices: laptop, printer, TV, `dns01` |
| **tagged** (trunk) | Frames with a tag, several VLANs in parallel over one wire | Switch-to-switch links, access points, hypervisors |

An end device on an untagged port notices nothing about VLANs. It gets an address over DHCP,
sees a gateway, and considers itself to be on the only network in the world. That is why
`dns01` needed no VLAN-specific configuration during the rebuild — only netmask and gateway.

Tagging is only required where **several** segments have to travel over **one** wire. For an
access point serving four SSIDs in four VLANs it is mandatory: there is only one cable.

## Native VLAN and PVID

Untagged traffic can still arrive on a trunk. What happens to it is governed by the **native
VLAN** — in UniFi the field is called *Native VLAN / Network*, other interfaces call it *PVID*
(port VLAN ID). Anything arriving without a tag is assigned that VLAN ID.

This is why a port can be untagged **and** tagged at the same time, and it is exactly the
arrangement a hypervisor needs:

```text
Native VLAN:  Servers (10)     -> host management, untagged
Tagged VLANs: Clients (20)     -> VMs in the client segment
              IoT (30)         -> VMs in the IoT segment
```

The host itself speaks untagged and lands in the server VLAN. Its VMs are given a tag by the
hypervisor and land wherever that tag points. One cable, three segments.

> [!NOTE]
> The native VLAN is the most common source of error on a first trunk. If it does not match
> what the device at the other end expects, tagged traffic works flawlessly while untagged
> traffic disappears — which feels like a half-broken cable.

## A VLAN is not a subnet

The two get treated as one thing in everyday use, and in this layout each VLAN does map to
exactly one `/24`. Technically they are two different layers:

- The **VLAN** is layer 2. It decides who can see whom directly at all.
- The **subnet** is layer 3. It decides when a device goes via the gateway instead of sending
  directly.

You can put two subnets in one VLAN, or stretch one subnet across several VLANs. Both mainly
produce confusion. One-to-one is the only mapping you still understand a year later.

The consequence matters more: **a VLAN on its own separates nothing the firewall does not
separate.** Once the segments are routed — and they are, otherwise nobody reaches the internet
— the actual boundary lives in the rules on the gateway. The VLAN only ensures the traffic
passes the gateway at all and can be judged there. Without that detour no rule applies, because
no packet ever reaches it.

## Why the netmask is part of it

The point at which this rebuild nearly came apart. As long as every device is configured with
`10.10.0.0/16`, they consider each other neighbours on the same network. A device holding
`10.10.20.5/16` sends to `10.10.30.7` **directly** via ARP instead of asking the gateway.

If a VLAN sits between them, the ARP request never arrives and the connection fails — but it
fails as a timeout, not as a clean refusal. If the VLAN is not cleanly separated, on the other
hand, the connection succeeds straight past every firewall rule.

Both outcomes are unpleasant. Which is why the VLAN cut and the netmask change belong together:
only with `/24` does a device ask the gateway for anything outside its segment — and only then
does a place exist where rules can apply.

## Security boundaries

Two attacks are worth knowing about, even though both are rare on a home network:

**Switch spoofing.** A device pretends to be a switch and negotiates a trunk in order to see
all VLANs. This affects vendors with automatic trunk negotiation; UniFi does not negotiate
trunks automatically, every port is assigned its profile explicitly.

**Double tagging.** A frame is sent with two tags; the first switch strips the outer one, the
second forwards by the inner one. This only works if a trunk's native VLAN matches the
attacker's VLAN. Setting a trunk's native VLAN to one with no end devices in it removes the
basis for the attack.

For a home network the more relevant statement is the simpler one: a VLAN protects against a
device that misbehaves, not against someone with physical access to the switch.

## Where it is written down here

The concrete six-segment layout is under
[Network Layout with VLANs]({{< relref "/docs/network/vlan-layout" >}}); the rebuild procedure,
including the order in which it actually works, is under
[Setting Up VLANs on the UDM]({{< relref "/docs/network/unifi-vlan-umbau" >}}).
