---
title: Proxmox-Cluster mit zwei Nodes und QDevice
weight: 100
---

# Proxmox-Cluster mit zwei Nodes und QDevice

Zwei Proxmox-Server lassen sich ueber dieselbe Weboberflaeche verwalten, wenn sie einem
Cluster angehoeren. Auch ohne automatische VM-Uebernahme braucht dieser Cluster eine
Mehrheit fuer Aenderungen. Bei zwei Nodes hilft eine dritte Stimme auf einem unabhaengigen
Rechner: ein **QDevice**.

Diese Notiz beschreibt den am **29. September 2026** eingerichteten Cluster `xlab`. Die
Grundlagen stehen unter [Proxmox verstehen]({{< relref "/docs/linux/proxmox" >}}), der
Aufbau im [Blogbeitrag]({{< relref "/posts/homelab/proxmox-cluster" >}}).

## Der konkrete Aufbau

| Rechner | Adresse | System | Aufgabe |
|---------|---------|--------|---------|
| `pve01` | `10.10.10.10` | Proxmox VE 9.2.2 | Cluster-Node, eine Stimme |
| `pve02` | `10.10.10.11` | Proxmox VE 9.2.2 | Cluster-Node, eine Stimme |
| `dns01` | `10.10.10.3` | Debian 13 auf dem Thin Client | Pi-hole und externe Quorum-Stimme |

Die Namen liegen unter `xlab.internal`. Beide PVE-Hosts haben eine feste Adresse auf
`vmbr0` im Netz `10.10.10.0/24`. Darueber laeuft hier auch die Cluster-Kommunikation.
Der Thin Client ist ein eigener physischer Rechner und bleibt erreichbar, wenn einer
der PVE-Hosts ausfaellt.

Auf `dns01` laeuft **`corosync-qnetd`**, auf jedem PVE-Host **`corosync-qdevice`**. Der
Thin Client wird dadurch kein Proxmox-Node und fuehrt keine VMs aus. Die Verbindung zum
Abstimmungsdienst verwendet TCP-Port `5403` und ist im eingerichteten Cluster per TLS
verschluesselt. Pi-hole laeuft daneben als eigener Dienst weiter.
[Proxmox: externe Quorum-Stimme](https://github.com/proxmox/pve-docs/blob/master/pvecm.adoc#corosync-external-vote-support)

## Was gemeinsam verwaltet wird

Die Oberflaechen unter `https://10.10.10.10:8006` und `https://10.10.10.11:8006` zeigen
beide Nodes. Jede davon kann den Cluster verwalten. Dazu kommen clusterweite
Konfigurationen, gemeinsame Backup-Jobs und die Moeglichkeit, Gaeste zwischen den Hosts
zu migrieren. Fuer Live-Migration muessen unter anderem CPUs, Netzwerke und VM-Konfiguration
zusammenpassen; durchgereichte PCI- oder USB-Geraete koennen sie verhindern.
[Proxmox: VM-Migration](https://github.com/proxmox/pve-docs/blob/master/qm.adoc#migration),
[Backup-Jobs](https://github.com/proxmox/pve-docs/blob/master/vzdump.adoc#backup-jobs)

Die VM-Festplatten bleiben hier **lokal**. Beide Hosts haben `local` und `local-lvm`, aber
derselbe Storage-Name bezeichnet jeweils den Speicher des betreffenden Hosts. `local-lvm`
verwendet LVM-Thin. Der Cluster verteilt die Konfiguration unter `/etc/pve`; er kopiert
dadurch keine VM-Festplatten. Bei einer Migration koennen lokale Disks ueber das Netzwerk
auf den Zielhost uebertragen werden.
[Proxmox: Cluster-Dateisystem](https://github.com/proxmox/pve-docs/blob/master/pmxcfs.adoc),
[lokale Disks bei der Migration](https://github.com/proxmox/pve-docs/blob/master/qm.adoc#online-migration)

## Wie das Quorum funktioniert

Jeder PVE-Node hat eine Stimme, das QDevice steuert eine weitere bei. Von den insgesamt
**drei Stimmen sind zwei erforderlich**. Bei einer Netztrennung vergibt das QDevice seine
Stimme nur an eine der getrennten Gruppen.

| Verfuegbar und miteinander verbunden | Stimmen | Quorum |
|------------------------------------|---------|--------|
| Beide PVE-Nodes und DNS | 3 | ja |
| Beide PVE-Nodes, DNS ausgefallen | 2 | ja |
| Ein PVE-Node und DNS | 2 | ja |
| Nur ein PVE-Node | 1 | nein |

Ohne Quorum wird `/etc/pve` schreibgeschuetzt. Normale VM-Starts und
Konfigurationsaenderungen sind dann blockiert. Bereits laufende Gaeste koennen in diesem
Aufbau ohne aktive HA-Verwaltung weiterlaufen, solange ihr Storage und Netzwerk verfuegbar
bleiben. Das gilt fuer die Gaeste auf dem noch laufenden Host; die Gaeste eines
ausgefallenen Hosts werden hier nicht automatisch uebernommen.
[Proxmox: Quorum und QDevice](https://github.com/proxmox/pve-docs/blob/master/pvecm.adoc#corosync-external-vote-support)

## Einrichtung auf den leeren Hosts

Beim Aufbau hatten beide PVE-Hosts dieselbe Version, synchronisierte Uhren und noch keine
VMs oder Container. Die vorhandenen Konfigurationen wurden vorher gesichert, die
Storage-Definitionen verglichen und die Erreichbarkeit zwischen den Hosts geprueft.

> [!WARNING]
> Die folgenden Schritte gelten fuer diesen geprueften Ausgangszustand. Beim Cluster-Beitritt
> wird die bisherige Konfiguration unter `/etc/pve` des beitretenden Hosts ersetzt. Ein Host
> mit vorhandenen Gaesten braucht vorher eine geplante Sicherung und Wiederherstellung;
> den Beitritt nicht mit `--force` erzwingen.

### Pakete installieren

Auf `dns01` mit dem normalen Benutzer und `sudo`:

```sh
sudo apt-get update
sudo apt-get install --no-install-recommends corosync-qnetd
systemctl is-active corosync-qnetd pihole-FTL
```

Auf **beiden** PVE-Hosts als `root`:

```sh
apt-get install --no-install-recommends corosync-qdevice
```

`qnetd` stellt die externe Stimme bereit, `qdevice` verbindet die einzelnen Cluster-Nodes
damit. Die Pakete gehoeren also auf unterschiedliche Rechner.

### Cluster erstellen und den zweiten Node aufnehmen

Auf `pve01` als `root`:

```sh
pvecm create xlab --link0 10.10.10.10
```

Der Cluster-Name wird bei der Erstellung festgelegt. Fuer den hier verwendeten
SSH-Beitritt muss der Root-Schluessel von `pve02` auf `pve01` zugelassen sein. Der auf dem
eigenen Laptop eingerichtete Key-Login ersetzt diese Verbindung zwischen den Hosts nicht.
SSH-Host-Schluessel vor dem Bestaetigen ueber eine bereits vertrauenswuerdige Verbindung
pruefen; die Host-Pruefung bleibt eingeschaltet.

Auf `pve02` als `root`, falls dessen Schluessel noch nicht auf `pve01` hinterlegt ist:

```sh
ssh-copy-id -i /root/.ssh/id_rsa.pub root@10.10.10.10
```

Danach ebenfalls auf `pve02`:

```sh
pvecm add 10.10.10.10 --use_ssh 1 --link0 10.10.10.11
```

Nach dem Beitritt muessen beide Nodes online sein. Erst dann wird das QDevice hinzugefuegt.
[Proxmox: Cluster-Beitritt](https://github.com/proxmox/pve-docs/blob/master/pvecm.adoc#adding-nodes-to-the-cluster)

### Root-Zugriff fuer die QDevice-Einrichtung

`pvecm qdevice setup` richtet die Zertifikate ueber SSH ein und erwartet dafuer Root-Zugriff
auf den externen Rechner. Ein funktionierender Login als `xander` mit passwortgeschuetztem
`sudo` reicht diesem Werkzeug nicht.

Auf `dns01` wurde deshalb der **oeffentliche** Schluessel aus
`/root/.ssh/id_rsa.pub` von `pve01` in `/root/.ssh/authorized_keys` ergaenzt. Der Eintrag
ist auf die Quelladresse von `pve01` beschraenkt und erlaubt keine SSH-Weiterleitungen
oder interaktive TTY-Sitzung. Sein Format, mit einem Platzhalter fuer den Schluessel:

```text
restrict,from="10.10.10.10" ssh-rsa <OEFFENTLICHER-SCHLUESSEL-VON-PVE01> root@pve01
```

Vorhandene Eintraege bleiben erhalten. Das Verzeichnis `/root/.ssh` gehoert `root` und hat
Modus `700`, `authorized_keys` Modus `600`. Root-Login per Schluessel muss in SSH zugelassen
sein. Der private Schluessel bleibt auf `pve01`. Die Grundlagen dazu stehen unter
[SSH-Config und Key-Login]({{< relref "/docs/linux/ssh-config" >}}).

### QDevice einbinden

Auf `pve01` als `root`:

```sh
pvecm qdevice setup 10.10.10.3
```

Der Befehl richtet die Zertifikate ein, ergaenzt die Cluster-Konfiguration und startet
und aktiviert `corosync-qdevice` auf beiden Nodes. Fuer den laufenden Abstimmungsverkehr
wird danach die Verbindung zu `qnetd` auf Port `5403` verwendet.

## Den fertigen Zustand pruefen

Auf beiden PVE-Hosts:

```sh
pvecm status
corosync-qdevice-tool -s
systemctl is-active corosync-qdevice
systemctl is-enabled corosync-qdevice
```

Der relevante Auszug aus `pvecm status` war auf beiden Nodes:

```text
Nodes:            2
Quorate:          Yes
Expected votes:   3
Highest expected: 3
Total votes:      3
Quorum:           2
Flags:            Quorate Qdevice
```

In der Mitgliederliste steht zusaetzlich zu den zwei Hosts ein `Qdevice` mit einer Stimme.
`corosync-qdevice-tool -s` meldet `Connected` und `10.10.10.3:5403` als QNetd-Host.

Auf `dns01`:

```sh
sudo corosync-qnetd-tool -l
systemctl is-active corosync-qnetd pihole-FTL
systemctl is-enabled corosync-qnetd
```

Dort werden der Cluster `xlab` und beide verbundenen Nodes angezeigt. Beide Dienste
muessen `active` sein. Die Pruefung nach der Einrichtung bestaetigte diesen Zustand;
ein absichtlicher Host-Ausfall wurde dabei nicht getestet.

Wenn `pvecm status` nur zwei erwartete Stimmen zeigt, ist die externe Stimme noch nicht
konfiguriert. Bei einem konfigurierten, aber nicht erreichbaren QDevice zuerst dessen
Dienst und die TCP-Verbindung auf Port `5403` pruefen. `Permission denied` waehrend des
Setups betrifft dagegen den Root-SSH-Zugriff fuer die Einrichtung.

## HA und Datenbankdaten

Im Cluster `xlab` sind **keine HA-Ressourcen und keine Storage-Replikation eingerichtet**.
HA koennte spaeter ausgewaehlte VMs nach einem Host-Ausfall automatisch auf dem anderen
Host neu starten. Dafuer muessten dort die VM-Festplatten, passende Netzwerke und genug
freie Ressourcen verfuegbar sein. Die dritte Stimme allein stellt diese Voraussetzungen
nicht her.
[Proxmox: High Availability](https://github.com/proxmox/pve-docs/blob/master/ha-manager.adoc)

Bei einer Datenbank-VM ist der verfuegbare Datenstand entscheidend. Auf gemeinsamem,
weiterhin funktionierendem Storage kann die neu gestartete VM dieselben Festplatten nutzen.
Bei asynchroner ZFS-Replikation steht dagegen nur der letzte erfolgreich uebertragene
Snapshot bereit. Auch bereits bestaetigte Datenbankaenderungen danach koennen auf dem
Ersatzhost fehlen. Eine Datenbank kann aus ebenfalls fehlenden Transaktionslogs nichts
wiederherstellen.

Die eingebaute Proxmox-Storage-Replikation setzt derzeit lokalen ZFS-Storage voraus; sie
ist kein Schalter, der die hier vorhandenen LVM-Thin-Volumes spiegelt. Fuer wichtige Daten
bleiben eigenstaendige Backups und eine bewusst gewaehlte Storage-Strategie erforderlich.
[Proxmox: Storage-Replikation](https://github.com/proxmox/pve-docs/blob/master/pvesr.adoc)
