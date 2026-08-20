---
title: Choosing an Upstream DNS
weight: 80
---

# Choosing an Upstream DNS

[Pi-hole]({{< relref "/docs/linux/pihole" >}}) does not know the addresses of the world. It
knows blocklists and a cache — everything else it forwards to an **upstream**, and hands that
answer back to the client. This makes the upstream the point at which the whole household's
name resolution leaves the local network.

The installer asks the question once, with Google as its first suggestion, and confirming it
means making a decision without making one.

## What the upstream sees

Before the filter, every device queried the provider's resolver on its own. Afterwards
exactly one address asks — the thin client — and it asks for everything that is not on a
blocklist. The number of queries drops noticeably; their content does not change: which
services the household uses, when somebody gets up, when nobody is home.

So the filter changes *how many* queries leave the house, not *whether* somebody sees them.
Setting up Pi-hole for privacy reasons and leaving Google as the upstream does not change the
recipient — it only consolidates the delivery.

## The providers

| Upstream | Addresses | Where it stands |
|----------|-----------|-----------------|
| **Quad9** | `9.9.9.9`, `149.112.112.112` | A Swiss foundation with no advertising business, stores no client addresses, validates DNSSEC itself and blocks known malware and phishing domains. Sends no ECS by default |
| Cloudflare | `1.1.1.1`, `1.0.0.1` | Usually the fastest in measurements, with an external audit of its privacy claims. A US company whose core business is CDN, not DNS |
| Google | `8.8.8.8`, `8.8.4.4` | Very fast, very reliable, documented everywhere — and run by the company whose business model is advertising. The installer's default |
| Your own ISP | announced by the router | Sees the traffic anyway. On the other hand, some providers redirect non-existent names to their own search pages instead of answering `NXDOMAIN` |
| Your own recursive resolver | `127.0.0.1#5335` | `unbound` queries the root servers itself. No provider gets the full picture any more — at the price of one more piece of software on the machine |

Quad9 is the choice here. Its malware filter complements Pi-hole instead of duplicating it:
Pi-hole filters ads and tracking, Quad9 filters malicious software. The price sits right next
to it — a second filter you do not maintain yourself can be wrong. If a domain looks wrongly
blocked, check it against `9.9.9.10`, the unfiltered variant of the same service.

> [!NOTE]
> **ECS** (EDNS Client Subnet) passes part of the client address on to the upstream so it can
> name a nearby CDN node. Quad9 declines to do this. It rarely shows in daily use; if a large
> download is noticeably slower than usual one day, that is the first place to look.

## Why not mix two providers

The field accepts several addresses, and the reflex is to put Quad9 and Cloudflare side by
side. That does not give you the best of both, but half of each: some queries go here, some
go there, each promise applies only to its share, and when an answer looks odd the first
question is who gave it.

Two addresses from the **same** provider are a different matter — that is redundancy inside a
single trust decision, not a split. Which is why `9.9.9.9` and `149.112.112.112` are both
listed.

## Making the change

```sh
sudo pihole-FTL --config dns.upstreams '["9.9.9.9","149.112.112.112"]'
```

In the web interface the same setting lives under *Settings → DNS → Upstream Servers*.
Restarting the service is not necessary; FTL picks the change up itself.

## Verifying

The value itself:

```sh
sudo pihole-FTL --config dns.upstreams
```

```text
[ 9.9.9.9, 149.112.112.112 ]
```

The more telling check comes from another machine. `whoami.akamai.net` answers with the
address of the resolver that actually sent the query — the upstream's, not your own:

```sh
dig +short @10.10.10.3 whoami.akamai.net
```

The answer on its own says nothing, though — it needs something to compare against. The same
question, put directly to the upstream, provides it:

```sh
dig +short @10.10.10.3 whoami.akamai.net   # via Pi-hole
dig +short @9.9.9.9    whoami.akamai.net   # straight to Quad9, as a reference
```

Both answers have to sit in the same range. On this installation:

| Measurement | Answer |
|-------------|--------|
| via Pi-hole, before | `172.253.1.215` — a Google network |
| via Pi-hole, after | `193.46.232.196` |
| straight to Quad9 | `193.46.232.197` |

The same `/24`, so the same exit. The exact address varies because the service uses several
of them.

> [!WARNING]
> The obvious route — running `whois` on the answer — misleads here. Quad9 operates its
> anycast nodes inside local providers' networks, so what appears there is some provider's
> name and nowhere "Quad9". Not knowing that, you take the address for your own line and go
> hunting for a fault that does not exist. Your own public address makes a useful third point
> of comparison:
>
> ```sh
> dig +short myip.opendns.com @resolver1.opendns.com
> ```

> [!NOTE]
> The cache answers faster than the upstream and reveals nothing about it. If the old result
> still comes back after the change, that is the cached answer — `pihole restartdns` clears
> the cache.

## DNSSEC: deliberately not yet

Pi-hole can verify answers cryptographically (`dns.dnssec`). Every signed zone attaches a
signature to its records whose chain of trust reaches up to the root zone; the resolver
recomputes it and discards whatever does not match. That protects against somebody swapping
an answer in transit and sending a device to a stranger's server.

What it does **not** do appears less often in the guides:

- **It encrypts nothing.** Signed does not mean confidential — anyone reading along the way
  still sees every name you look up. That would be DNS over TLS or over HTTPS, a different
  building site.
- **It only works on signed zones.** A large share of the domains called up daily is not
  signed, and *unsigned* is a valid state, not an error.
- **It ends at the resolver.** Nothing is verified between client and Pi-hole. The phone
  takes the filter at its word.

On top of that, **Quad9 already validates**. Doing it locally as well would win exactly one
point: no longer depending on the upstream having checked correctly, and on nobody sitting
between the gateway and Quad9. Against that stands a real risk of outage — expired signatures
happen, to large operators too, and the affected domain then answers `SERVFAIL` rather than
not at all. In the household that looks like a Pi-hole outage, and that is exactly where
people will start looking.

The decision here: **off**, until the upstream is a recursive resolver of our own. At that
point validation becomes the actual purpose of the exercise, because the chain is recomputed
to the root instead of taking a provider's word for it.

For anyone who wants it on regardless:

```sh
sudo pihole-FTL --config dns.dnssec true
```

The counter-check needs two domains — one with a valid signature, one deliberately broken:

```sh
dig +dnssec @10.10.10.3 dnssec.works        # NOERROR, "ad" flag set
dig @10.10.10.3 fail01.dnssec.works         # SERVFAIL when validation is on
dig +cd @10.10.10.3 fail01.dnssec.works     # answers anyway: +cd bypasses the check
```

The `ad` flag (*authenticated data*) in the first answer is the proof that verification
happened. `+cd` (*checking disabled*) is the tool of choice when something breaks: if an
answer only arrives with it, the cause is DNSSEC and not the service.

> [!NOTE]
> For blocked domains Pi-hole answers itself — a made-up answer that cannot carry a valid
> signature. Validation is deliberately bypassed for those. It would only be noticed by a
> client that validates on its own; the usual stub resolver library does not.

## What remains open

**The recursive resolver of our own.** `unbound` next to Pi-hole, starting at the root
servers itself. After that no provider sees the full picture any more, and DNSSEC becomes a
sensible addition rather than double bookkeeping. The first answers are slower, because every
chain wants to be walked once in full.

**The path outwards is cleartext.** Even with Quad9, every query crosses the line
unencrypted. The remedy is DoT or DoH between Pi-hole and the upstream — which does not
change the provider, only rules out the readers along the way.
