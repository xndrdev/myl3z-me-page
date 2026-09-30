---
title: Homelab, step five — two Proxmox nodes, one interface
date: 2026-09-29
---

Both Proxmox servers are set up and now form the **`xlab`** cluster. `pve01` and `pve02`
can be managed together. The third vote comes from the thin client already running
Pi-hole.

<!--more-->

In [part four]({{< relref "/posts/homelab/umbau-proxmox" >}}), the two machines were
still waiting for installation. Both now run Proxmox VE 9.2.2. The next thing I wanted
was simple: one interface where I could see and manage both servers.

## A second job for the DNS server

`pve01` is at `10.10.10.10`, `pve02` at `10.10.10.11`. The existing
[DNS server]({{< relref "/posts/homelab/thin-client" >}}) is still reachable at
`10.10.10.3`. All three follow the `xlab.internal` naming scheme.

A third vote is useful in a cluster of two nodes. If one server fails, the other
would no longer have a majority on its own. An external voting service can give it
the second of three votes.

This did not require another machine: the thin client already runs independently of
both PVE hosts. It now has `corosync-qnetd` installed, with `corosync-qdevice` on each
Proxmox host. Pi-hole continues alongside it. The thin client handles voting without
becoming a third virtualization host.

## Check first, then join

Both PVE hosts were still empty. Before joining them, versions, time synchronisation,
network connectivity and storage definitions were checked, and the existing
configurations backed up. This was a good time to form the cluster: joining replaces
the incoming host's previous cluster configuration.

The cluster was created on `pve01` as `xlab`, then `pve02` joined it. Cluster
communication uses the static addresses on `vmbr0`.

One detail needed a separate step: Proxmox expects root SSH access to the DNS server
to provision the QDevice certificates. My regular login with `sudo` was not enough.
The public key from `pve01` was therefore added on `dns01`, restricted to the source
address `10.10.10.10`. The external vote could then be added.

## Three votes, two required

Both nodes reported the same state at the end:

```text
Quorate:          Yes
Expected votes:   3
Total votes:      3
Quorum:           2
Flags:            Quorate Qdevice
```

Both PVE nodes are online and connected to the QDevice. Voting traffic is encrypted
with TLS, and the services start automatically. Pi-hole was still active after setup
as well. A deliberate failure test remains to be done.

For everyday use, I can now open the Proxmox interface on either node and manage both
servers. For future maintenance, being able to move VMs between hosts is especially
useful, provided their configuration and the available resources allow it.

## VM disks remain local

Both hosts have `local` and `local-lvm`. Those matching names still refer to two
separate sets of local storage. The cluster brings their management together; it
does not create a copy of the VM data on the other host.

I have left HA unconfigured for now. Automatic recovery after a failure would also
require the affected VMs' disks to be available on the other host. For a database
in particular, the age of that data would matter too. An older replicated copy can
be missing changes that were already acknowledged, even if the VM restarts successfully.

The information page,
[Two-node Proxmox cluster with a QDevice]({{< relref "/docs/linux/proxmox-cluster" >}}),
records the setup, commands, failure behaviour and limits of the current storage
configuration.
