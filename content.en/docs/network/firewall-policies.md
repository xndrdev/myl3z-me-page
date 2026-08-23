---
title: Firewalling Between Segments
weight: 40
---

# Firewalling Between Segments

Segments on their own separate nothing. Once the networks are routed — and they are, otherwise
nobody reaches the internet — the ruleset on the gateway alone decides who reaches whom. The
VLAN only ensures the traffic passes the gateway at all and can be judged there.

## The base rule

Deny by default, permit by exception. Every segment may reach the internet, none may reach
another — and departures from that need a concrete reason.

The reverse is tempting because everything works immediately, and it reliably ends in a ruleset
nobody dares dismantle. Start with *deny* and a broken printer tells you which exception is
missing. Start with *allow* and nothing tells you anything.

## The exceptions

| From | To | For |
|------|----|-----|
| all segments | `10.10.10.3` port 53 TCP/UDP | **Name resolution.** Without this rule the household has no DNS after the rebuild |
| all segments | `10.10.10.3` port 123 UDP | Time, if Pi-hole serves as the NTP source |
| Clients | IoT: printer, cast devices, NAS | Printing and casting — targeted at address and port, not wholesale |
| Clients | Servers | Administration: web interfaces, SSH |
| Clients | Infrastructure `10.10.1.1` port 443 | the UDM interface, if you want to reach it from the everyday network |
| Kids | internet, DNS | nothing else — in particular not clients or servers |
| IoT, Guests | internet and DNS only | no segment crossings |
| all | established/related | without this rule no reply to a permitted request works |

That last line is not optional polish. A firewall judges packets individually; without a state
rule the request would be allowed out but the reply not back in. On UniFi it exists by default
— you just have to leave it alone while tidying up.

## Direction is not symmetric

This is where rulesets usually grow too wide. *Clients may reach the printer on the IoT
network* does not mean *IoT and clients may talk to each other*. The reverse direction stays
closed, and the printer's reply still arrives — it falls under established/related.

That is exactly where the separation pays off: the TV can be compromised and still not open a
connection to the work laptop, while casting from a phone keeps working.

## Zones instead of individual rules

Newer UniFi versions group networks into **zones** (*Settings → Security → Policy Engine*,
depending on the build also *Firewall → Zone Matrix*). Instead of rules between individual
networks you maintain a matrix between zones — with six segments, by far the clearer option.

A sensible mapping for this layout:

| Zone | Networks |
|------|----------|
| *Internal* | Infrastructure, Servers |
| *Trusted* | Clients |
| *Restricted* | Kids |
| *IoT* | IoT |
| *Guest* | Guests |

The zone matrix then sits on *block* except where the table above calls for an exception. Older
builds instead offer *Settings → Firewall Rules* with *LAN In / LAN Out / LAN Local* — the same
logic, just maintained one rule at a time.

> [!NOTE]
> **LAN In** judges traffic entering the UDM from a network to be routed onwards — that is
> where inter-segment rules belong. **LAN Local** judges traffic aimed **at the UDM itself**,
> meaning its interface and its services. Build segment separation in *LAN Local* and you will
> wonder why it does not apply.

## Order

Rules are evaluated top to bottom, first match wins. From which follows the only sorting rule
you need: **specific before general.**

```text
1. allow  IoT -> 10.10.10.3:53          (exception)
2. block  IoT -> all local networks     (blanket)
```

Swap the two and the blanket rule matches first, so the exception below is never reached — the
rule sits visibly in the list and does nothing. This is the most common reason for "but the
rule is right there and it doesn't work".

## mDNS across segment boundaries

Cast devices, AirPlay and Sonos find each other via multicast, and multicast stops at the
segment boundary. A phone on the client network simply no longer sees the TV on the IoT
network.

The UDM ships a repeater for this, listed in the network configuration as *Multicast DNS* or
*mDNS*. It relays the announcements between selected networks.

> [!NOTE]
> The repeater only makes devices *visible*. The actual connection afterwards still needs the
> firewall exception — the two get confused whenever the TV shows up in the list but cannot be
> controlled.

## Forcing DNS

Some IoT hardware ships with `8.8.8.8` hard-coded and never asks what DHCP suggests. Only the
firewall helps against that:

| Approach | Effect |
|----------|--------|
| Block port 53 outbound except to `10.10.10.3` | The device gets no answer and usually falls back to the DHCP resolver |
| Redirect port 53 to `10.10.10.3` via NAT | The device notices nothing and gets filtered anyway |
| Block port 853 | Disables DNS over TLS |

DNS over HTTPS cannot be caught this way — it is indistinguishable from ordinary HTTPS and can
only be curbed through blocklists of known endpoints inside Pi-hole itself. The details are
under
[Handing Out Pi-hole via DHCP]({{< relref "/docs/network/udm-dhcp-dns#what-dhcp-does-not-solve" >}}).

## Verifying

Rules are only finished once both directions have been tested — the permitted **and** the
forbidden one. From a device in each segment:

```sh
dig +short @10.10.10.3 example.com   # must work everywhere
ping -c1 9.9.9.9                     # internet, must work everywhere
ping -c1 10.10.20.5                  # from IoT/guests/kids: must fail
ping -c1 10.10.1.1                   # from IoT/guests: must fail
```

The third line is the meaningful one. A separation you only recognise by nothing being broken
has not been tested.

The difference in failure mode is useful: an immediate *no route to host* comes from a rule
actively rejecting. A silent timeout means either *drop* or: there is no device there at all.
Confuse the two and you will hunt for a firewall rule covering a machine that is merely
switched off.
