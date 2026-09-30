---
title: Homelab, fuenfter Schritt — zwei Proxmox-Nodes, eine Oberflaeche
date: 2026-09-29
---

Die beiden Proxmox-Server sind eingerichtet und bilden jetzt den Cluster **`xlab`**.
`pve01` und `pve02` lassen sich gemeinsam verwalten. Die dritte Stimme liefert der
Thin Client, auf dem schon Pi-hole laeuft.

<!--more-->

Im [vierten Teil]({{< relref "/posts/homelab/umbau-proxmox" >}}) standen die beiden
Maschinen noch vor der Installation. Inzwischen laeuft auf beiden Proxmox VE 9.2.2.
Der naechste Wunsch war schlicht: eine Oberflaeche, in der ich beide Server sehe und
verwalten kann.

## Der DNS-Server bekommt eine zweite Aufgabe

`pve01` liegt auf `10.10.10.10`, `pve02` auf `10.10.10.11`. Der bestehende
[DNS-Server]({{< relref "/posts/homelab/thin-client" >}}) ist weiterhin unter
`10.10.10.3` erreichbar. Alle drei gehoeren zum Namensschema `xlab.internal`.

Fuer einen Cluster aus zwei Nodes ist eine dritte Stimme nuetzlich. Faellt einer der
beiden Server aus, haette der andere allein keine Mehrheit mehr. Ein externer
Abstimmungsdienst kann ihm die zweite von insgesamt drei Stimmen geben.

Dafuer musste keine weitere Maschine her: Der Thin Client laeuft bereits unabhaengig von
den beiden PVE-Hosts. Auf ihm ist jetzt `corosync-qnetd` installiert, auf den Proxmox-Hosts
jeweils `corosync-qdevice`. Pi-hole arbeitet daneben weiter. Der Thin Client uebernimmt
die Abstimmung und wird dadurch kein dritter Virtualisierungshost.

## Erst pruefen, dann zusammenfuehren

Beide PVE-Hosts waren noch leer. Vor dem Beitritt wurden Versionen, Uhrzeit,
Netzwerkverbindung und Storage-Definitionen geprueft und die bestehenden Konfigurationen
gesichert. Das war ein guter Zeitpunkt fuer den Cluster: Beim Beitritt ersetzt Proxmox
die bisherige Cluster-Konfiguration des hinzukommenden Hosts.

Der Cluster wurde auf `pve01` als `xlab` angelegt, anschliessend kam `pve02` dazu.
Die Cluster-Kommunikation verwendet die festen Adressen auf `vmbr0`.

Ein Detail brauchte einen eigenen Schritt: Fuer die QDevice-Zertifikate erwartet Proxmox
Root-SSH-Zugriff auf den DNS-Server. Mein normaler Login mit `sudo` reichte dafuer nicht.
Der oeffentliche Schluessel von `pve01` wurde deshalb auf `dns01` hinterlegt, beschraenkt
auf die Quelladresse `10.10.10.10`. Danach liess sich die externe Stimme einbinden.

## Drei Stimmen, zwei erforderlich

Am Ende meldeten beide Nodes denselben Zustand:

```text
Quorate:          Yes
Expected votes:   3
Total votes:      3
Quorum:           2
Flags:            Quorate Qdevice
```

Beide PVE-Nodes sind online, beide mit dem QDevice verbunden. Der Abstimmungsverkehr
ist per TLS verschluesselt, die Dienste starten automatisch. Auch Pi-hole war nach
der Einrichtung weiterhin aktiv. Ein absichtlicher Ausfalltest steht noch aus.

Im Alltag kann ich jetzt die Proxmox-Oberflaeche eines beliebigen Nodes oeffnen und beide
Server verwalten. Fuer spaetere Wartung ist besonders interessant, VMs zwischen den
Hosts verschieben zu koennen, wenn deren Konfiguration und die freien Ressourcen passen.

## Die VM-Festplatten bleiben lokal

Auf beiden Hosts gibt es `local` und `local-lvm`. Hinter dem gleichen Namen liegen aber
weiterhin zwei getrennte lokale Speicher. Der Cluster fuehrt ihre Verwaltung zusammen;
eine Kopie der VM-Daten auf dem anderen Host entsteht dadurch nicht.

HA habe ich vorerst unkonfiguriert gelassen. Fuer eine automatische Uebernahme nach einem
Ausfall muessten auch die Festplatten der betroffenen VMs auf dem anderen Host verfuegbar
sein. Gerade bei einer Datenbank waere ausserdem zu klaeren, wie aktuell dieser Stand ist.
Ein aelterer replizierter Stand kann trotz erfolgreichem VM-Neustart bereits bestaetigte
Aenderungen vermissen lassen.

Die Infoseite
[Proxmox-Cluster mit zwei Nodes und QDevice]({{< relref "/docs/linux/proxmox-cluster" >}})
haelt den Aufbau, die Befehle, das Verhalten bei Ausfaellen und die Grenzen des aktuellen
Storage-Setups fest.
