---
title: Understanding Firewalls
weight: 35
---

# Understanding Firewalls

A firewall controls network traffic according to rules. It decides which connections to allow
and which to block. At home, that might mean the laptop can access the internet, the TV can
query the DNS server, but a guest cannot access the router's management interface.

A firewall can run on a router such as the Dream Machine or directly on a computer. On the
router, it controls traffic passing through the router or addressed to the router itself.
On a computer, it protects that machine's own network access. [NIST's firewall guide](https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-41r1.pdf)
also describes these two placements.

## What a rule checks

Devices send data in packets. These include information identifying the sender and recipient.
TCP and UDP also use port numbers, which help direct traffic to a service on the destination
device. DNS uses port 53, for example, while HTTPS normally uses port 443.

A basic firewall rule describes the traffic it applies to and what should happen to it:

| Property | Example from the homelab |
|----------|-------------------------|
| Source | Guest network `10.10.40.0/24` |
| Destination | DNS server `10.10.10.3` |
| Protocol | TCP or UDP |
| Destination port | 53 for DNS |
| Action | allow |

This lets guests resolve names using the DNS server. The permission applies only to that
service. It does not cover the server's web interface on another port, which can be protected
by a subsequent block rule.

## How the firewall makes a decision

With an ordered ruleset like the one used here, the firewall checks rules from top to bottom.
The first matching rule decides the outcome. An exception therefore needs to appear before
the general block rule.

For new requests from the guest network, the relevant part might look like this:

| Order | Traffic | Decision |
|-------|---------|----------|
| 1 | to the DNS server on port 53, TCP or UDP | allow |
| 2 | to other internal destinations and services | block |

A DNS query matches the first rule and can pass. Access to server administration only matches
the second rule and is blocked. Putting the general block first would also block DNS.
Internet access and router services have their own rules; the table shows just this part.

*Allow* lets matching traffic through. *Drop* discards it without a reply; *Reject* refuses it
with a response. A timeout alone therefore does not prove a block is working: the destination
might simply be switched off.

## Why replies can come back

A **stateful firewall** keeps track of ongoing communication. It can associate a reply with a
previously permitted request. A matching state rule allows that reply back without generally
allowing new connections in the reverse direction. Even though UDP has no connection setup,
its traffic can be tracked using addresses, ports and a timeout. The [Netfilter documentation on connection tracking](https://wiki.nftables.org/wiki-nftables/index.php/Matching_connection_tracking_stateful_metainformation)
explains the states used for this.

In the example, a guest asks for a website's address. The DNS server can reply. That does not
give the server permission to initiate arbitrary new connections to the guest.

Rules often call this reply traffic *established*. *Related* covers traffic associated with
an existing connection, such as certain error messages. These permissions form part of the
ruleset.

## Where VLANs and zones fit in

A [VLAN]({{< relref "/docs/network/vlan-grundlagen" >}}) separates a local network at the
Ethernet layer. Devices in different VLANs need a router to communicate. In this homelab,
that is the UDM, whose firewall decides which traffic may pass between the networks.

Devices in the same VLAN and subnet normally communicate directly through a switch or
access point. This traffic does not pass through the router's firewall. Isolation within
that network needs additional measures, such as client isolation or a firewall on the
endpoint.

A **zone** groups networks that share firewall rules. *IoT* might contain the TV network,
while *Internal* can contain several networks for servers and infrastructure. A zone's name
does not block anything by itself; the rules decide. UniFi also distinguishes traffic
**through the router** from traffic **to the router itself**, which belongs to the *Gateway*
zone. This includes DHCP and the management interface. [Ubiquiti's guide to firewall zones](https://help.ui.com/hc/en-us/articles/115003173168-Zone-Based-Firewalls-in-UniFi)
explains this distinction.

## What a firewall does not automatically recognise

An allowed connection is not automatically harmless. If a device can reach the internet
using HTTPS, that connection can also carry unwanted traffic. A basic packet filter checks
addresses, ports and states; it cannot read encrypted HTTPS content without additional
decryption.

Intrusion detection and prevention, often called **IDS** and **IPS**, add inspection for
suspicious patterns. Updates and secure access are still necessary. A **DNS filter** such as
Pi-hole has another job: filtering name queries. A known IP address or a different resolver
can bypass that filter if the firewall permits the traffic.

**NAT** is a separate function too: it translates network addresses. The firewall ruleset
determines whether a connection is permitted. Both IPv4 and IPv6 need to be considered when
both are in use on the network.

## How to tell whether rules work

Each permission needs a test showing that the required service is reachable. Each block
needs a test from the affected network showing that the prohibited access fails. A known,
running service is more useful for this than an arbitrary ping target: ping tests ICMP,
not access to a web interface or an SSH port.

Hit counters and logs help associate the test with a rule. On the guest network, for example,
DNS should work while server administration remains unreachable.

The specific rules and exceptions are covered in
[Firewalling Between Segments]({{< relref "/docs/network/firewall-policies" >}}).
The [post on the firewall changes]({{< relref "/posts/homelab/udm-audit" >}}) describes which
of these were implemented in the homelab.
