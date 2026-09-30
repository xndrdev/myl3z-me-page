---
title: Two-node Proxmox cluster with a QDevice
weight: 100
---

# Two-node Proxmox cluster with a QDevice

Two Proxmox servers can share a management interface by joining a cluster. Even without
automatic VM failover, that cluster needs a majority of votes to make changes. With two
nodes, a third vote on an independent machine helps: a **QDevice**.

This note describes the `xlab` cluster configured on **29 September 2026**. The foundations
are covered in [Understanding Proxmox]({{< relref "/docs/linux/proxmox" >}}), and the
build is recorded in the [blog post]({{< relref "/posts/homelab/proxmox-cluster" >}}).

## The actual setup

| Machine | Address | System | Role |
|---------|---------|--------|------|
| `pve01` | `10.10.10.10` | Proxmox VE 9.2.2 | Cluster node, one vote |
| `pve02` | `10.10.10.11` | Proxmox VE 9.2.2 | Cluster node, one vote |
| `dns01` | `10.10.10.3` | Debian 13 on the thin client | Pi-hole and the external quorum vote |

The names live under `xlab.internal`. Both PVE hosts have a static address on `vmbr0`
in `10.10.10.0/24`. This network also carries cluster communication in this setup.
The thin client is a separate physical machine and remains reachable when one PVE host
fails.

**`corosync-qnetd`** runs on `dns01`; **`corosync-qdevice`** runs on each PVE host. This
does not turn the thin client into a Proxmox node or make it run VMs. The connection to
the voting service uses TCP port `5403` and is encrypted with TLS in this cluster.
Pi-hole continues running alongside it as a separate service.
[Proxmox: external quorum vote](https://github.com/proxmox/pve-docs/blob/master/pvecm.adoc#corosync-external-vote-support)

## What is managed together

The interfaces at `https://10.10.10.10:8006` and `https://10.10.10.11:8006` both display
both nodes. Either interface can manage the cluster. Other benefits include cluster-wide
configuration, shared backup jobs and guest migration between hosts. Live migration
requires compatible CPUs, networks and VM configuration, among other things; PCI or USB
passthrough can prevent it.
[Proxmox: VM migration](https://github.com/proxmox/pve-docs/blob/master/qm.adoc#migration),
[Backup jobs](https://github.com/proxmox/pve-docs/blob/master/vzdump.adoc#backup-jobs)

VM disks remain **local** here. Both hosts have `local` and `local-lvm`, but the same
storage name refers to each host's own storage. `local-lvm` uses LVM-thin. The cluster
distributes configuration under `/etc/pve`; this does not copy VM disks. During a migration,
local disks can be transferred to the destination host over the network.
[Proxmox: cluster filesystem](https://github.com/proxmox/pve-docs/blob/master/pmxcfs.adoc),
[Local disks during migration](https://github.com/proxmox/pve-docs/blob/master/qm.adoc#online-migration)

## How quorum works

Each PVE node has one vote, and the QDevice contributes another. **Two of the three
votes are required**. During a network partition, the QDevice grants its vote to only
one of the separated groups.

| Available and connected to each other | Votes | Quorum |
|--------------------------------------|-------|--------|
| Both PVE nodes and DNS | 3 | yes |
| Both PVE nodes, DNS unavailable | 2 | yes |
| One PVE node and DNS | 2 | yes |
| Only one PVE node | 1 | no |

Without quorum, `/etc/pve` becomes read-only. Normal VM starts and configuration changes
are then blocked. In this setup without active HA management, guests that are already
running can continue as long as their storage and network remain available. This applies
to guests on the surviving host; guests on a failed host are not automatically recovered
in this setup.
[Proxmox: quorum and QDevice](https://github.com/proxmox/pve-docs/blob/master/pvecm.adoc#corosync-external-vote-support)

## Setting up the empty hosts

At setup time, both PVE hosts ran the same version, had synchronised clocks and contained
no VMs or containers. Existing configurations were backed up, storage definitions compared
and connectivity between the hosts checked first.

> [!WARNING]
> The following steps apply to that verified starting point. Joining a cluster replaces
> the joining host's existing configuration under `/etc/pve`. A host with existing guests
> needs a planned backup and restore procedure first; do not force the join with `--force`.

### Install the packages

On `dns01`, using the regular account and `sudo`:

```sh
sudo apt-get update
sudo apt-get install --no-install-recommends corosync-qnetd
systemctl is-active corosync-qnetd pihole-FTL
```

On **both** PVE hosts, as `root`:

```sh
apt-get install --no-install-recommends corosync-qdevice
```

`qnetd` provides the external vote; `qdevice` connects each cluster node to it. The two
packages belong on different machines.

### Create the cluster and join the second node

On `pve01`, as `root`:

```sh
pvecm create xlab --link0 10.10.10.10
```

The cluster name is set during creation. For the SSH join used here, the root SSH key
from `pve02` must be authorised on `pve01`. Key-based access from your laptop does not
replace this connection between the hosts. Verify SSH host keys through an already
trusted connection before accepting them; host verification remains enabled.

On `pve02`, as `root`, if its key has not yet been added to `pve01`:

```sh
ssh-copy-id -i /root/.ssh/id_rsa.pub root@10.10.10.10
```

Then, also on `pve02`:

```sh
pvecm add 10.10.10.10 --use_ssh 1 --link0 10.10.10.11
```

Both nodes must be online after the join. The QDevice is added after that.
[Proxmox: joining a cluster](https://github.com/proxmox/pve-docs/blob/master/pvecm.adoc#adding-nodes-to-the-cluster)

### Root access for QDevice setup

`pvecm qdevice setup` provisions certificates over SSH and expects root access to the
external machine. A working login as `xander` with password-protected `sudo` is not
sufficient for this tool.

The **public** key from `/root/.ssh/id_rsa.pub` on `pve01` was therefore added to
`/root/.ssh/authorized_keys` on `dns01`. The entry is restricted to the source address
of `pve01` and does not allow SSH forwarding or an interactive TTY session. Its format,
with a placeholder for the key:

```text
restrict,from="10.10.10.10" ssh-rsa <PUBLIC-KEY-FROM-PVE01> root@pve01
```

Existing entries are preserved. `/root/.ssh` belongs to `root` and has mode `700`;
`authorized_keys` has mode `600`. SSH must permit root login using a key. The private
key stays on `pve01`. The basics are covered in
[SSH config and key login]({{< relref "/docs/linux/ssh-config" >}}).

### Add the QDevice

On `pve01`, as `root`:

```sh
pvecm qdevice setup 10.10.10.3
```

The command provisions certificates, updates the cluster configuration, and starts
and enables `corosync-qdevice` on both nodes. Subsequent voting traffic uses the
connection to `qnetd` on port `5403`.

## Checking the result

On both PVE hosts:

```sh
pvecm status
corosync-qdevice-tool -s
systemctl is-active corosync-qdevice
systemctl is-enabled corosync-qdevice
```

The relevant excerpt from `pvecm status` was the same on both nodes:

```text
Nodes:            2
Quorate:          Yes
Expected votes:   3
Highest expected: 3
Total votes:      3
Quorum:           2
Flags:            Quorate Qdevice
```

The membership list contains a `Qdevice` with one vote alongside the two hosts.
`corosync-qdevice-tool -s` reports `Connected`, with `10.10.10.3:5403` as the QNetd host.

On `dns01`:

```sh
sudo corosync-qnetd-tool -l
systemctl is-active corosync-qnetd pihole-FTL
systemctl is-enabled corosync-qnetd
```

This lists the `xlab` cluster and both connected nodes. Both services should report
`active`. The checks after setup confirmed this state; a deliberate host failure was
not tested.

If `pvecm status` shows only two expected votes, the external vote has not yet been
configured. If the QDevice is configured but unreachable, check its service and TCP
connectivity on port `5403` first. `Permission denied` during setup instead concerns
the root SSH access needed for provisioning.

## HA and database data

The `xlab` cluster has **no HA resources or storage replication configured**. HA could
later restart selected VMs automatically on the other host after a failure. Their disks,
compatible networks and sufficient free resources would need to be available there.
The third vote alone does not provide these prerequisites.
[Proxmox: High Availability](https://github.com/proxmox/pve-docs/blob/master/ha-manager.adoc)

For a database VM, the available data state matters. With shared storage that remains
operational, the restarted VM can use the same disks. Asynchronous ZFS replication only
provides the last successfully transferred snapshot. Even database changes already
acknowledged after that point can be missing on the replacement host. A database cannot
recover them from transaction logs that were not transferred either.

Proxmox's built-in storage replication currently requires local ZFS storage; it cannot
simply mirror the LVM-thin volumes used here. Important data still needs independent
backups and a deliberate storage strategy.
[Proxmox: storage replication](https://github.com/proxmox/pve-docs/blob/master/pvesr.adoc)
