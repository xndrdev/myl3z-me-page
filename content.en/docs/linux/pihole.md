---
title: Pi-hole as a DNS Server
weight: 70
---

# Pi-hole as a DNS Server

Pi-hole is a DNS server with a filter. When a device asks for a domain that sits on a
blocklist, Pi-hole does not pass the query on to the upstream but answers it itself — with an
address that leads nowhere. The advantage over a blocker in the browser is where the filtering
happens: name resolution is used by every device on the network, including the TV and a
visitor's phone, neither of which lets you install anything.

The price is in the same sentence: a service every device relies on is also a service whose
outage every device notices.

## Prerequisites

- A fixed address, because every client has it hardcoded
  ([Static IP with ifupdown]({{< relref "/docs/linux/static-ip" >}})).
- `curl` for the install script
  ([Base Packages After the Install]({{< relref "/docs/linux/base-packages" >}})).

## Installation

```sh
curl -sSL https://install.pi-hole.net | bash
```

What that one-liner takes on trust, and how to separate downloading from running, is written
up under
[Running scripts from the internet]({{< relref "/docs/linux/base-packages#running-scripts-from-the-internet" >}}).

The script opens a dialog and asks, in order, for the interface, the upstream DNS, the
blocklist it suggests, whether queries should be logged and in how much detail. At the end it
prints a generated password for the web interface — the only moment it is visible in plain
text. `pihole setpassword` sets your own.

## What version 6 changed

Guides written for Pi-hole 5 describe a substantially different system. What changed, and what
keeps coming up while looking things up:

| Before | Since version 6 |
|--------|-----------------|
| files copied straight into the system | installed as a Debian package `pihole-meta` that pulls its dependencies via `apt` |
| `lighttpd` and PHP for the web interface | a web server built into `pihole-FTL`, no extra service |
| `setupVars.conf` plus separate `dnsmasq` files | one configuration: `/etc/pihole/pihole.toml` |
| DNS and optionally DHCP | an NTP server on top, active by default |

What is listening afterwards:

```sh
ss -tulpn
```

| Port | For |
|------|-----|
| 53/tcp, 53/udp | DNS — the actual service |
| 80/tcp | web interface, unencrypted |
| 443/tcp | web interface over TLS, with a self-issued certificate |
| 123/udp | the NTP server FTL brings along |

All of it is answered by a single process, `pihole-FTL`. Anyone planning to run a web server
or their own time service on the same machine later should keep that list in mind.

## Reading the configuration

Instead of digging through 70 KB of `pihole.toml`, query individual values:

```sh
pihole-FTL --config dns.upstreams      # one value
pihole-FTL --config                    # every value, flat
```

In the file itself every value is documented, and anything deviating from the default carries
a `### CHANGED` marker — so what makes this installation specific is visible at a glance, as
opposed to what is simply the default.

The state after the install on the homelab box:

| Key | Value | Meaning |
|-----|-------|---------|
| `dns.interface` | `enp1s0` | the interface answers are served on |
| `dns.listeningMode` | `LOCAL` | only queries from the local subnet are answered |
| `dns.upstreams` | `8.8.8.8`, `8.8.4.4` | where queries that are not blocked go |
| `dns.dnssec` | `false` | signatures on answers are not validated |
| `dns.queryLogging` | `true` | every query is logged along with its client |
| `dns.hosts` | `10.10.10.3 dns01.xlab.internal` | a name of its own for the machine itself |
| `dns.blocking.mode` | `NULL` | blocked names are answered with `0.0.0.0` |
| `dhcp.active` | `false` | the router keeps handing out addresses |

`listeningMode` is the value that decides the attack surface. `LOCAL` answers queries from
networks the machine itself has an address in and ignores the rest. `ALL` answers everything —
on a machine that ever gets a public address, that turns it into an open resolver, ready to be
abused for amplification attacks.

`blocking.mode = NULL` means a blocked domain resolves to `0.0.0.0`. The client fails
immediately instead of opening a connection to Pi-hole and hitting a block page there. That is
faster and takes load off the service, but in a browser it produces a connection error rather
than an explanation.

Values are changed through the same option:

```sh
sudo pihole-FTL --config dns.dnssec true
```

> [!NOTE]
> Editing `pihole.toml` by hand works too, followed by `pihole reloaddns`. FTL rewrites the
> file itself whenever something changes, though — your own comments in it will not survive.

## Local names

A DNS server on your own network can do more than filter: it can hand out names. Instead of
memorising addresses, you record once which name points at which address — in the web
interface under *Settings → Local DNS Records*, in the configuration it is `dns.hosts`:

```sh
pihole-FTL --config dns.hosts
```

```text
[ 10.10.10.3 dns01.xlab.internal ]
```

The entries are written to `/etc/pihole/hosts/custom.list` in `/etc/hosts` format. The first
name is the machine itself: `dns01` for the role, `xlab.internal` as the zone for the homelab.

Picking the suffix is not a matter of taste:

| Suffix | Situation |
|--------|-----------|
| `.internal` | reserved by ICANN for private networks since 2024, and therefore permanently free of collisions with public domains |
| `.local` | claimed by mDNS (RFC 6762). Using it in DNS means arguing with Avahi and Bonjour over who gets to answer |
| `.lan`, `.home`, `.box` | common, but reserved nowhere. Works until somebody runs the suffix as a public TLD |
| a real domain of your own | clean and more work: the zone has to be maintained, and it is supposed to answer differently inside and outside |

> [!NOTE]
> The entry only covers forward resolution. In reverse, `dig -x 10.10.10.3` still gets
> `pi.hole` — the PTR Pi-hole assigns to itself (`dns.piholePTR`). Making both match means
> changing that value.

Not to be confused with `dns.domain.name` (default: `lan`). That domain is appended to names
Pi-hole learns over DHCP — as long as the router runs DHCP, the value does not matter.

## Verify

The honest test does not come from the server itself but from another machine on the network:

```sh
dig +short @10.10.10.3 example.com           # normal answer, upstream works
dig +short @10.10.10.3 doubleclick.net       # 0.0.0.0 = blocked
dig +short @10.10.10.3 pi.hole               # 10.10.10.3, its own address
dig +short @10.10.10.3 dns01.xlab.internal   # 10.10.10.3, the entry of your own
```

On the server:

```sh
pihole status                           # is FTL listening on port 53?
systemctl status pihole-FTL
pihole -t                               # follow queries live
```

## What is still open afterwards

The install leaves a DNS server running. It does not leave it being used.

**Nobody asks it.** As long as the router hands out its own address as the DNS server, every
query keeps going past it. Only once `10.10.10.3` is set as the nameserver in the router's
DHCP — or Pi-hole takes over DHCP itself — does anything arrive. Clients with a hardcoded DNS
server, or with DNS-over-HTTPS in the browser, stay outside either way.

**The server does not ask itself.** Its `/etc/resolv.conf` still points at the router and a
public resolver:

```sh
cat /etc/resolv.conf
```

That makes the machine running the filter the only one on the network not using it.
`nameserver 127.0.0.1` belongs there — in the right place, or the change is gone on the next
`ifup`
([Pitfall: dns-nameservers without resolvconf]({{< relref "/docs/linux/static-ip#pitfall-dns-nameservers-without-resolvconf" >}})).

**The upstream is a decision, not a default.** The installer suggests Google, and confirming
it builds a filter that still reports every single query to Google — just bundled behind one
address. Quad9 (`9.9.9.9`) and Cloudflare (`1.1.1.1`) are different providers with different
promises, but the same construction. Not reporting to anyone what a household resolves takes a
recursive resolver of your own (`unbound`) behind Pi-hole.

**DNSSEC is off.** Upstream answers are taken as they come. Turning it on costs a few
milliseconds per query and breaks on domains with broken signatures — which then looks like a
Pi-hole outage.

**The web interface sits on port 80.** Reachable across the LAN, unencrypted, with a password
in front of it. The TLS certificate on 443 is self-issued, so every browser warns about it.

## Day-to-day commands

| Command | Effect |
|---------|--------|
| `pihole -up` | update Pi-hole (core, web, FTL) |
| `pihole -g` | re-read the blocklists |
| `pihole disable 5m` | suspend filtering for five minutes, back on automatically |
| `pihole -t` | follow queries live |
| `sudo pihole -q domain.tld` | check which list a domain is on — without `sudo` the CLI asks for the web interface password |
| `pihole setpassword` | set the web interface password |
| `pihole -r` | repair, when something stops starting after an update |

> [!WARNING]
> `pihole disable` without a duration turns filtering off indefinitely. Nobody notices — after
> all, everything works.
