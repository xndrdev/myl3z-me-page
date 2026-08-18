---
title: Handing Out Pi-hole via DHCP (UDM)
weight: 10
---

# Handing Out Pi-hole via DHCP

A DNS server nobody asks filters nothing. After installing
[Pi-hole]({{< relref "/docs/linux/pihole" >}}) the service is running, but every device in the
house still gets told the gateway's address as its nameserver when it joins, and it obediently
uses it. The filter sits next to the traffic and watches.

The place where this is decided is a single field in the gateway's DHCP server — on a UniFi
Dream Machine that means the network configuration, not the box Pi-hole runs on.

## What DHCP does here

When a device joins the network it broadcasts a request for configuration. The DHCP server
replies with a packet that carries more than an address: netmask, gateway, lease time — and in
**option 6** a list of nameservers. That list is exactly what the client writes down.

That is the lever and also its limit: option 6 is an offer. A device with a hardcoded resolver
ignores the field entirely.

## The setting on the UDM

*Settings → Networks →* the network in question *→ DHCP Name Server*. It defaults to **Auto**;
switching it to **Manual** reveals the server fields.

| Setting | Effect |
|---------|--------|
| `Auto` | The UDM advertises itself as the nameserver and resolves on the client's behalf. Pi-hole sees nothing |
| `Manual` → `10.10.10.3` | Clients are handed the address of `dns01` and query it directly |

> [!NOTE]
> The wording moves around between UniFi Network releases. Depending on your version the
> section is called *DHCP Name Server*, *DNS Server*, or hides behind *Advanced → Manual*. What
> you are looking for is always the field that ends up in the DHCP offer — not the resolver the
> UDM uses for itself.

Exactly one address is configured here:

```text
10.10.10.3
```

## Why there is no second entry

The field accepts several servers, and the reflex is to add a public resolver as a safety net.
That is wrong in this spot, for the reason already covered under
[Two nameservers as a safety net]({{< relref "/docs/linux/static-ip#two-nameservers-as-a-safety-net" >}}):
the client does not treat the list as a ranked failover, it moves to the second server the
moment the first is slow to answer once. From then on queries bypass the filter — unlogged,
unfiltered, and without anything looking broken.

A second entry only becomes correct once there is a *second Pi-hole* behind it that knows the
same lists and the same local names.

The cost of the clean variant is in the same paragraph: the thin client is now a single point
of failure for name resolution in the entire household. If it goes down, the network feels
completely dead.

## When clients pick it up

Not immediately. A device only fetches the configuration on its next renewal — usually after
half the lease time, which with UniFi's default of 86400 seconds means roughly twelve hours.
To avoid waiting:

| System | Command |
|--------|---------|
| Linux (dhclient) | `sudo dhclient -r && sudo dhclient` |
| Linux (NetworkManager) | `nmcli con down <name> && nmcli con up <name>` |
| Windows | `ipconfig /release && ipconfig /renew` |
| macOS | *System Settings → Network → Details → TCP/IP → Renew DHCP Lease* |
| everything else | disconnect from Wi-Fi and reconnect |

> [!NOTE]
> To push the change through the household quickly, lower the lease time beforehand and raise
> it again afterwards. A permanently short lease only creates needless chatter.

## Verify

The honest test comes from any device on the network, not from the UDM:

```sh
resolvectl status | grep 'DNS Server'   # systemd-resolved
cat /etc/resolv.conf                    # ifupdown, without systemd-resolved
scutil --dns | grep nameserver          # macOS
ipconfig /all                           # Windows
```

`10.10.10.3` and nothing else is what you want to see.

The second test carries more weight, because it proves the queries actually arrive. On `dns01`:

```sh
pihole -t
```

Before the change this showed little more than the server itself. Afterwards the addresses of
the household's devices scroll past. The web interface shows the same thing under
*Settings → Clients*: the number of known clients is the proof that the service is being used,
not merely running.

## What DHCP does not solve

**Devices with a hardcoded resolver.** TVs, consoles and some IoT hardware ship with
`8.8.8.8` baked in. They never ask, so no DHCP field reaches them. The only thing that works is
a firewall rule blocking outbound port 53 for everything except `10.10.10.3` — or redirecting
those queries to Pi-hole via NAT.

**DNS over TLS and DNS over HTTPS.** Android offers a fixed provider under *Private DNS*, and
browsers ship DoH of their own. DoT can be blocked on port 853; DoH cannot — it is
indistinguishable from ordinary HTTPS and can only be curbed through a blocklist of known
endpoints inside Pi-hole.

> [!WARNING]
> Blocking port 853 only makes Android fall back to plaintext DNS when *Private DNS* is set to
> **Automatic**. With a provider entered explicitly there is no fallback, just a connection
> failure with no obvious cause.

**IPv6.** If the UDM hands out IPv6 through router advertisements, it names itself as the
resolver in them (RDNSS) — and many clients prefer that one. The filter keeps running but only
catches part of the traffic. Whether this applies here has not been checked yet:

```sh
ip -6 addr show          # does the client have a global IPv6 address at all?
resolvectl status        # is there an fe80:: nameserver next to 10.10.10.3?
```

If there is, two clean paths exist: give Pi-hole a static IPv6 address and advertise that as
well, or turn IPv6 off on the LAN while it is not needed. The half-configured state is the
worst of the three.

## What is still open

**Every additional network needs the setting again.** The field belongs to the network, not to
the appliance. Once the network is cut into segments
([Network Layout with VLANs]({{< relref "/docs/network/vlan-layout" >}})), each new VLAN starts
out on `Auto` — and silently drops out of the filter.

**Pi-hole only listens locally.** `dns.listeningMode` is set to `LOCAL`, which answers only
queries from networks the machine itself has an address in. As long as everything lives in one
flat network this goes unnoticed. With separate subnets it does not.

**The UDM's own ad blocking.** If the firmware ships a filter of its own and it is enabled, two
systems filter on top of each other. That costs no functionality, but it doubles the price of
every future debugging session — switching it off is the cheaper decision.
