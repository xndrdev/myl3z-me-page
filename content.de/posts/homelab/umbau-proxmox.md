---
title: Homelab, vierter Schritt — aufgeraeumt, jetzt kommt Proxmox
date: 2026-09-26
---

# Homelab, vierter Schritt — aufgeraeumt, jetzt kommt Proxmox

Im [dritten Teil]({{< relref "/posts/homelab/vlan-umbau" >}}) ging es um VLANs, Firewall-Regeln
und ein Netz, das waehrend des Umbaus nicht immer das tat, was es sollte. Inzwischen ist der
Netzwerkumbau soweit abgeschlossen. Alle zu Hause haben wieder Internet. Das ist nach den
letzten Umbauten erst einmal die wichtigste Nachricht.

Seitdem habe ich auch das Homelab selbst komplett umgebaut und aufgeraeumt. Dabei ist es
allerdings nicht bei der bisherigen Hardware geblieben: Ein zweiter selbst gebauter Server
und zwei alte Rack-Server sind dazugekommen.

## Ein zweiter Server aus alten Teilen

Aus alten Consumer-Hardware-Teilen habe ich einen zweiten Server zusammengebastelt. Die Teile
bekommen damit noch einmal eine Aufgabe im Homelab. Auch auf diesem Rechner werde ich
Proxmox installieren.

Damit stehen fuer den naechsten Schritt zwei Maschinen bereit: der bisherige Server und der
neue Eigenbau. Beide sollen frisch mit Proxmox aufgesetzt werden. Der Zusammenbau ist
erledigt, die Installation steht noch bevor.

## Zwei Rack-Server muessen warten

Ausserdem habe ich zwei alte Rack-Server gekauft. Die bringen allerdings ein Problem mit,
das sich beim Aufstellen schlecht ignorieren laesst: Sie sind zu laut.

Deshalb bekommen die beiden spaeter einen anderen Platz. Geplant ist ein selbst gebautes
Holzrack in der Garage. Dort sollen die Rack-Server irgendwann unterkommen. Das Rack und
der Umzug sind ein eigenes Projekt fuer spaeter; jetzt sind erst einmal die beiden anderen
Server dran.

## Als naechstes: Proxmox auf beiden Maschinen

Das Netz steht, das Homelab ist aufgeraeumt und die Hardware fuer den naechsten Schritt ist
zusammengebaut. Jetzt geht es darum, die beiden Proxmox-Server neu aufzusetzen.

Fuer die Installation von Windows und Proxmox nutze ich seit einiger Zeit MediCat USB,
das auf Ventoy basiert. Damit liegen die Installations-ISOs zusammen auf einem Stick und
lassen sich beim Booten auswaehlen. Den werde ich auch fuer die beiden Server verwenden.
Wie MediCat und Ventoy zusammenspielen und wie die Installer auf den Stick kommen, steht
unter [MediCat & Ventoy]({{< relref "/docs/linux/medicat-ventoy" >}}).

Was Proxmox eigentlich ist, wie es auf Debian aufbaut und wie der Einstieg in eine eigene
kleine Distribution aussehen koennte, steht unter
[Proxmox verstehen]({{< relref "/docs/linux/proxmox" >}}).

Im letzten Beitrag war noch von einem Hypervisor die Rede. Inzwischen sind es zwei Maschinen,
die eingerichtet werden wollen, und zwei weitere, die auf einen Platz in der Garage warten.
Fuer den Moment reicht die Aufgabe auf dem Tisch: Proxmox auf beiden Servern sauber
installieren. Wie das laeuft, kommt in den naechsten Eintrag.

Inzwischen stehen beide Installationen und der gemeinsame Cluster. Weiter geht es mit
[Homelab, fuenfter Schritt — zwei Proxmox-Nodes, eine Oberflaeche]({{< relref "/posts/homelab/proxmox-cluster" >}}).
