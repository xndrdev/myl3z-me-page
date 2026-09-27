---
title: Understanding Pi-hole and Blocklists
weight: 75
---

# Understanding Pi-hole and Blocklists

Pi-hole filters DNS queries on the local network. Before a browser can load a website,
it needs the IP address for its name. This name lookup is where Pi-hole decides whether
to return a usable answer. A blocklist supplies the rules: domains known for advertising,
tracking or malicious software, for example.

The installation is covered in [Pi-hole as a DNS Server]({{< relref "/docs/linux/pihole" >}}).
This page explains how subscribed lists become a filter and which HaGeZi lists I have
now added.

## What happens during a DNS query

A website often includes content from several domains: the article itself, images,
an analytics service and an advertising network. The device needs an address for each
of those names, unless it already has one in its cache.

1. **The device asks Pi-hole.** It must be using Pi-hole as its DNS server. In the homelab,
   the UDM distributes this setting through DHCP
   ([Handing Out Pi-hole via DHCP]({{< relref "/docs/network/udm-dhcp-dns" >}})).
2. **Pi-hole checks the applicable rules.** These include blocklists and custom allow
   and deny rules for the client making the query.
3. **A blocked domain receives a blocking response.** In the default `NULL` mode,
   that is `0.0.0.0` for IPv4 or `::` for IPv6. Neither provides a usable destination address.
4. **An allowed query is resolved.** Pi-hole can answer from local DNS records or its
   cache. Otherwise, it asks the configured upstream resolver.

The actual page content then travels directly between the device and the web server.
Pi-hole is not acting as a proxy that reads HTML or inspects images. The
[Pi-hole documentation on blocking modes](https://docs.pi-hole.net/ftldns/blockingmode/)
also describes the alternatives to `NULL` mode.

## Gravity: turning lists into a local database

A subscribed blocklist starts as a URL pointing to a text file. Its provider maintains
the entries; Pi-hole fetches and processes them locally during an update. This process
is called **Gravity**. The resulting blocking entries are stored in
`/etc/pihole/gravity.db`.

A DNS query does not trigger a lookup on GitHub or at the list provider. Pi-hole makes
its decision using the most recently imported entries. Changes at the provider only
arrive with the next Gravity run. By default, that happens weekly; to run it manually
on the Pi-hole machine:

```sh
pihole -g
```

This updates the lists, not the Pi-hole software. The
[official Gravity documentation](https://docs.pi-hole.net/main/pihole-command/#gravity)
describes the process.

Overlap between lists is normal. Finding the same domain on three lists does not mean
three times the protection. Pi-hole keeps the association with each source, partly so
different groups can use different lists. That is also why the number of list entries
and the number of unique domains are not the same thing.
([Domain database structure](https://docs.pi-hole.net/database/domain-database/))

## My HaGeZi selection

I have added these five lists. The selection can be reopened using
[my HaGeZi preset](https://hagezi-mirror.dnsbunker.org/dlg.html?src=github&tif=full&gambling=full&tier=pro&items=nsfw%2Cnosafesearch&tool=pi-hole):

| List | Purpose |
|------|---------|
| **Multi Pro** | Broad filtering of advertising, tracking, telemetry and known malicious domains |
| **Threat Intelligence Feeds (TIF), Full** | Additional coverage of known malware, phishing and command-and-control domains |
| **Gambling, Full** | Domains related to gambling and betting |
| **NSFW** | Domains with adult content |
| **No-SafeSearch** | Search engines that do not support SafeSearch |

Pro and TIF address privacy and known threats. Gambling, NSFW and No-SafeSearch also
define which content should be accessible. The
[HaGeZi list descriptions](https://github.com/hagezi/dns-blocklists) explain their scope.

**No-SafeSearch does not enable a search filter on Google or Bing.** The list blocks
certain search engines entirely. Enforcing SafeSearch on an allowed provider requires
separate configuration. NSFW is also a domain list, not a way to detect individual
images or posts on an otherwise allowed platform.

## Adding the right URLs

The linked generator has **Pi-hole** selected as the tool, which selects the
**Adblock format** here. The `src=github` parameter sets the source to **GitHub (raw)**,
even though the generator itself is hosted on a mirror. This produces these five list URLs:

```text
https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/pro.txt
https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/tif.txt
https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/gambling.txt
https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/nsfw.txt
https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/nosafesearch.txt
```

In Pi-hole 6, add these URLs as **blocklists** under **Lists** in the web interface.
The preset link itself is a web page and does not belong in that field. Then enable
the lists, assign the intended groups and update Gravity. Check the output to confirm
that all five sources were processed.

The format name does not mean Pi-hole gains every feature of a browser blocker.
It processes rules suitable for DNS filtering, such as `||example.com^`, which covers
a domain and its subdomains. Rules that hide individual page elements still belong
in the browser. The format mapping is also listed in the
[HaGeZi overview](https://github.com/hagezi/dns-blocklists#overview).

## When a website stops working

Even well-maintained lists can block a domain you need. Start in the **Query Log**
with the affected client and the time of the failure. Often the website itself is
allowed, but another domain used for signing in or loading content is blocked.

On the Pi-hole machine, you can look up which lists or custom rules match a name.
Replace `example.com` with the affected domain:

```sh
sudo pihole -q example.com
```

The [list query](https://docs.pi-hole.net/main/pihole-command/#query) explains where a
rule comes from; the Query Log shows how Pi-hole handled the actual request. If the
block is a false positive, allow the specific domain you need and record the reason
in a comment. A matching custom allowlist rule takes precedence over a subscribed
blocklist. It must apply to the same client group.
([Rule priorities](https://docs.pi-hole.net/database/domain-database/#priorities))

Groups let you assign content filters to particular devices, for example. What matters
is the association between client, group and list; a VLAN alone does not configure this.
The [Pi-hole group management examples](https://docs.pi-hole.net/group_management/example/)
show how these assignments work.

## Where the filter reaches its limits

Pi-hole sees domain names, not complete URLs. If advertising and the content you want
share the same name, a DNS filter cannot reliably separate them. That is why it cannot
reliably remove YouTube ads, for example. A browser blocker can work at other levels
and remains a useful complement.

The filter also only applies to queries that reach it. An external DNS server,
DNS-over-HTTPS in the browser or a VPN with its own DNS can bypass it. Previously
cached answers and existing connections do not disappear immediately after a list
change either.

A high blocking percentage alone therefore says little about quality. What matters
is stopping unwanted connections while keeping necessary services working. Selecting
and maintaining the lists is as much a part of running Pi-hole as installing it.
