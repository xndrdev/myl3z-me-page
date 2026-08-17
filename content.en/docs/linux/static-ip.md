---
title: Static IP with ifupdown
weight: 30
---

# Static IP with ifupdown

A server other devices are supposed to reach needs an address that does not change. DHCP
hands it whatever is free — possibly a different one after the router reboots, and every
configuration pointing at the old one breaks. For a DNS server in particular that is not an
option: its address is what every client has hardcoded.

A Debian install without a desktop manages networking through **ifupdown** and
`/etc/network/interfaces`. NetworkManager and `systemd-networkd` solve the same problem but
are not in play on a netinst system.

## Find the interface name

```sh
ip -br link
```

```text
lo               UNKNOWN        00:00:00:00:00:00 <LOOPBACK,UP,LOWER_UP>
enp1s0           UP             a4:bb:6d:12:34:56 <BROADCAST,MULTICAST,UP,LOWER_UP>
```

`enp1s0` is not a random name: `en` for Ethernet, `p1` for PCI bus 1, `s0` for slot 0. These
*predictable interface names* stay stable across reboots and kernel updates — unlike the old
`eth0`, whose numbering depended on the order devices happened to be detected in.

## The configuration

The block for a fixed address in `/etc/network/interfaces`:

```text
auto enp1s0
iface enp1s0 inet static
    address 10.10.10.3/16
    gateway 10.10.1.1
    dns-nameservers 10.10.1.1 1.1.1.1
```

| Line | Meaning |
|------|---------|
| `auto enp1s0` | bring the interface up at boot. Without it the configuration exists but only applies on a manual `ifup` |
| `iface enp1s0 inet static` | IPv4 (`inet`), address assigned statically instead of via DHCP. For IPv6 it would be `inet6` |
| `address 10.10.10.3/16` | address in CIDR notation. `/16` means network `10.10.0.0` with hosts from `10.10.0.1` to `10.10.255.254`. Older guides write `address` plus `netmask 255.255.0.0` — equivalent, just more work |
| `gateway 10.10.1.1` | default route for everything outside the local network. Must fall inside the network the mask spans |
| `dns-nameservers 10.10.1.1 1.1.1.1` | nameservers for resolution, separated by spaces. The second one is the fallback if the first does not answer |

> [!WARNING]
> The fixed address has to sit outside the router's DHCP range. Otherwise the router will
> eventually hand the same address to another device and break both.

## Two nameservers as a safety net

A server without working name resolution cannot update packages, cannot sync time and cannot
report anything outward. A second entry catches the failure of the first — here the router as
the local resolver, a public one behind it.

How the list is worked through is decided by the C library's resolver, not by the network:

- **Strictly in order.** The first entry is always asked first. There is no load balancing and
  no preference for whichever server is faster.
- **Switching costs time.** If the first does not answer, the timeout has to expire first (5
  seconds by default) before the second is tried. While the first is down, that wait applies
  to *every* query.
- **Only outages are routed around.** A timeout or a `SERVFAIL` moves on to the next server.
  An `NXDOMAIN`, on the other hand, is a valid answer and is passed through — a nameserver
  that answers wrongly does not get skipped.
- **Three entries maximum.** glibc ignores anything beyond that.

The behaviour can be tuned if the wait is a problem:

```text
options timeout:2 attempts:1
```

> [!WARNING]
> As soon as your own DNS server runs on the network — on this machine that is now
> [Pi-hole]({{< relref "/docs/linux/pihole" >}}) — the public second entry becomes a hole in
> the filter: queries slip past it, unfiltered and unlogged. And if the entry points at a
> resolver that forwards back to your own DNS server, you get a loop. On the DNS server
> itself, `127.0.0.1` belongs in the list, and the public resolver is configured as an
> upstream *inside* it rather than alongside it.

## Apply

```sh
systemctl reboot
```

A reboot is the honest test: it proves the configuration survives boot instead of only being
set in the running system. It also works without one:

```sh
sudo ifdown enp1s0 && sudo ifup enp1s0
```

> [!NOTE]
> Either way an existing SSH session drops — the address is changing, after all. On a headless
> box, know the new address beforehand, or the only way back in is a monitor.

## Verify

```sh
ip -br a               # assigned address
ip route               # default via 10.10.1.1 dev enp1s0
ping -c1 10.10.1.1     # gateway reachable
cat /etc/resolv.conf   # which nameserver is actually in use
```

## Pitfall: dns-nameservers without resolvconf

`ifupdown` does not write the `dns-nameservers` line to `/etc/resolv.conf` itself. That is
done by a hook script under `/etc/network/if-up.d/` which only arrives with the `resolvconf`
package (or `openresolv`). Without it the line is **silently ignored** — no error, no warning.
What remains is whatever nameserver the installer wrote.

```sh
dpkg -l resolvconf openresolv 2>/dev/null | grep '^ii'
```

If that returns nothing, there are two ways forward:

{{< tabs >}}

{{% tab "install resolvconf" %}}
```sh
sudo apt install resolvconf
sudo ifdown enp1s0 && sudo ifup enp1s0
```
Configuration stays in one place — `/etc/network/interfaces` is the source of truth.
{{% /tab %}}

{{% tab "maintain resolv.conf directly" %}}
```sh
printf 'nameserver 10.10.1.1\nnameserver 1.1.1.1\n' | sudo tee /etc/resolv.conf
```
One package less, but two files that have to agree. The `dns-nameservers` line then serves as
documentation only — change it later and forget the `resolv.conf`, and you will be debugging
in the wrong file.
{{% /tab %}}

{{< /tabs >}}

The reverse also holds: with `resolvconf` installed, `/etc/resolv.conf` is generated and hand
edits to it vanish on the next `ifup`. The head of the file tells you which case you are in —
a generated one carries a `DO NOT EDIT` notice:

```sh
head -3 /etc/resolv.conf
```
