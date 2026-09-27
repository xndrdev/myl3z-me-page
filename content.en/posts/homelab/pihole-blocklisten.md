---
title: Homelab — new blocklists for Pi-hole
date: 2026-09-27
---

I have fine-tuned my Pi-hole setup and added more HaGeZi blocklists. DNS queries in
the homelab already go through the filter; this time, the change is to the rules
it uses to make its decisions.

<!--more-->

## What I added

The selection consists of **Multi Pro**, **Threat Intelligence Feeds in the Full version**,
**Gambling Full**, **NSFW** and **No-SafeSearch**. This
[HaGeZi preset](https://hagezi-mirror.dnsbunker.org/dlg.html?src=github&tif=full&gambling=full&tier=pro&items=nsfw%2Cnosafesearch&tool=pi-hole)
records the combination.

Pro covers advertising, tracking and telemetry. TIF expands local filtering of known
malicious domains. The other three lists add content filters for gambling, adult
content and search engines without SafeSearch support.

This means some decisions about known malware domains now happen locally in Pi-hole.
The Quad9 upstream described in the
[earlier DNS post]({{< relref "/posts/homelab/dns-upstream" >}}) remains another layer
of filtering for queries that Pi-hole forwards.

## What this means in everyday use

Pi-hole imports the lists through Gravity and checks DNS queries against those local
entries. It does not contact the list providers for every website visit. Updates
bring in any entries the providers have added or removed.

A broader selection can also block a service you need. When that happens, the Query
Log is the first place to look for the responsible domain and allow it specifically.
And the name No-SafeSearch is easy to misunderstand: the list does not switch
SafeSearch on automatically.

The new information page,
[Understanding Pi-hole and Blocklists]({{< relref "/docs/linux/pihole-blocklisten" >}}),
explains how filtering works, provides the five URLs for this selection and covers
the limits of DNS blocklists.
