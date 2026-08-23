---
title: Mapping WLANs to VLANs
weight: 50
---

# Mapping WLANs to VLANs

On a wire the switch port decides which segment a device lands in. On WLAN there is no port —
there the SSID decides. Every wireless network maps to exactly one VLAN, and whatever a device
associates with determines where it sits.

## One SSID per segment

| SSID | Network | Particularity |
|------|---------|---------------|
| `xlab` | Clients (20) | WPA3/WPA2 mixed, both bands |
| `xlab-IoT` | IoT (30) | 2.4 GHz only, WPA2, PMF off |
| `xlab-Kids` | Kids (25) | like clients, plus time windows |
| `xlab-Guests` | Guests (40) | client device isolation |

The mapping lives in UniFi under *Settings → WiFi →* SSID *→ Network*. Depending on the version
the field is called *Network* or *VLAN*; it always means the same thing.

The price of this approach is airtime: every SSID sends its own beacons, and four wireless
networks on the same channel cost measurable capacity. The alternative would be a single SSID
with dynamic VLAN assignment via RADIUS — cleaner on the air, but with an authentication server
as an additional service that can fail. For four segments in one household the effort is not
justified.

## Why IoT is configured differently

The three restrictions on `xlab-IoT` look like carelessness and are the opposite:

**2.4 GHz only.** Many smart home devices support nothing else. If the SSID is enabled on both
bands, the access point advertises it on 5 GHz too — and some devices then fail during
association because band steering tries to push them there. Restricting the SSID to 2.4 GHz
takes the question off the table.

**WPA2 instead of WPA3.** The ESP32 and similar radio modules found in a large share of
inexpensive devices cannot do WPA3. In mixed mode they often fail during negotiation already.

**PMF disabled.** Protected Management Frames are mandatory under WPA3 and optional under WPA2.
Older firmware drops the connection as soon as the access point requires them — or associates
and loses the link again minutes later. That is the failure mode that is nastiest to chase,
because it does not show up immediately.

> [!NOTE]
> These three points deliberately lower the security level of this one SSID. Which is exactly
> why it lives in its own segment: whatever breaks in there finds a network with a TV and a
> printer, not a work laptop. The weaker encryption is tolerable because the segment boundary
> stands behind it — not because it does not matter.

## Guests

*Client device isolation* ensures devices on the guest network cannot see each other, not even
within the same segment. Without it, two visitors on the same WLAN remain direct neighbours —
the firewall never sees that traffic because it never leaves the access point.

That is the difference between layer 2 isolation and a firewall rule, and the reason why the
rule *guests may only reach the internet* is not sufficient on its own.

## Kids

The segment exists so that children's devices do not sit on the same network as work machines
— and because rules bound to a network are easier to maintain than rules per device. Two things
hang off it:

- **a dedicated Pi-hole group**, so stricter blocklists can apply to these clients without
  affecting the rest of the household
- **a time window**, either as a *blackout schedule* on the SSID or as a *traffic rule* with a
  schedule

The blackout schedule disables the SSID at fixed times. That is effective and crude: a device
with saved credentials for `xlab` simply moves there. Separation via its own network only works
as well as password discipline on the other SSIDs.

## The access point needs the VLANs

An access point hangs off one cable and serves four segments. That only works tagged: its
switch port carries the infrastructure VLAN untagged for the device itself, and all SSID VLANs
tagged.

If one of them is missing from the tagged list, devices associate with the SSID and never get
an address — a failure that looks like DHCP and is not. Which is why the AP uplink port profile
is set to *Allow All*: every future SSID then works without touching the port. The profiles are
under
[Setting Up VLANs on the UDM]({{< relref "/docs/network/unifi-vlan-umbau#port-profiles" >}}).

## Renaming costs one pass

The old SSID was replaced by the four new ones during the rebuild. To an end device an SSID is
the identity of the network — with a new name, every device in the household has to be
reconnected by hand.

For phones and laptops that is a minute. It gets tedious with devices that have no interface
and can only be set up through a vendor app, and with those that need to be put into a pairing
mode first. Anyone planning the rebuild should budget generously for this part: it takes longer
than the entire network configuration before it.

## Verifying

From a phone, once per SSID:

```sh
ip -br a                             # is the address in the expected segment?
dig +short @10.10.10.3 example.com   # DNS across the segment boundary
```

The fastest tell is the third octet: `10.10.30.x` means IoT, `10.10.20.x` means clients. That
is precisely what the scheme was chosen for.

In the UniFi interface, *Client Devices* shows network and SSID for every connected device.
Finding devices still on the infrastructure network after the rebuild means an SSID was missed
— or a device is still associated with the old wireless network.
