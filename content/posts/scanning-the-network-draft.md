---
title: "Scanning the network - draft"
date: 2023-06-28T22:11:36+02:00
draft: true
---

- What other techniques can we find to scan the network
-- netdiscover -p
-- p0f -i eth0 -p -o /tmp/p0f.log
-- # Bettercap
-- net.recon on/off #Read local ARP cache periodically
-- net.show
-- set net.show.meta true #more info

```
#ARP discovery
nmap -sn <Network> #ARP Requests (Discover IPs)
netdiscover -r <Network> #ARP requests (Discover IPs)
​
#NBT discovery
nbtscan -r 192.168.0.1/24 #Search in Domain
​
# Bettercap
net.probe on/off #Discover hosts on current subnet by probing with ARP, mDNS, NBNS, UPNP, and/or WSD
set net.probe.mdns true/false #Enable mDNS discovery probes (default=true)
set net.probe.nbns true/false #Enable NetBIOS name service discovery probes (default=true)
set net.probe.upnp true/false #Enable UPNP discovery probes (default=true)
set net.probe.wsd true/false #Enable WSD discovery probes (default=true)
set net.probe.throttle 10 #10ms between probes sent (default=10)
​
#IPv6
alive6 <IFACE> # Send a pingv6 to multicast.
```