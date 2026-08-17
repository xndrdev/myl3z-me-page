---
title: Homelab, erster Schritt — Thin Client mit Debian 13
date: 2026-08-16
---

# Homelab, erster Schritt — Thin Client mit Debian 13

Ich baue mir ein Homelab. Nicht als Rack im Keller, sondern als etwas, das schrittweise
waechst: ein Geraet, ein Dienst, aufschreiben, weitermachen.

Der erste Knoten ist ein Thin Client. Fuer den Anfang ist das die passende Klasse Hardware —
er laeuft lautlos, zieht im Dauerbetrieb kaum Strom und ist gebraucht guenstig zu bekommen.
Fuer die Dienste, die zuerst drauf sollen, reicht das mit Abstand.

Darauf laeuft jetzt **Debian 13**, installiert vom Netinst-Image, ohne Desktop. Debian, weil
die Stable-Reihe genau das tut, was man von einer Maschine erwartet, die einfach durchlaufen
soll: wenig Bewegung, lange Supportzeitraeume, keine Ueberraschungen beim Update. Netinst,
weil auf einer Kiste ohne Bildschirm nichts installiert sein muss, was ich nicht selbst
angefordert habe. Zu diesem Minimalismus gehoert, dass `sudo` nicht dabei war — das kam
nachtraeglich dazu, samt Freischaltung meines Benutzers
([sudo nachruesten]({{< relref "/docs/linux/sudo-setup" >}})).

Den Installer-Stick habe ich im Terminal geschrieben — die Schritte stehen unter
[Boot-Stick im Terminal]({{< relref "/docs/linux/bootable-usb" >}}), die Erklaerung der
beteiligten Befehle unter
[lsblk, umount, dd, sync]({{< relref "/docs/linux/disk-commands" >}}).

Nach der Installation hat die Kiste eine feste Adresse bekommen, eingetragen in
`/etc/network/interfaces` und mit `systemctl reboot` uebernommen. Per DHCP waere die Adresse
nach dem naechsten Router-Neustart moeglicherweise eine andere — fuer einen Rechner, der
demnaechst selbst die Namensaufloesung im Netz macht, waere das der falsche Anfang. Der
Aufbau der Konfiguration und ein Fallstrick bei der Nameserver-Zeile stehen unter
[Statische IP mit ifupdown]({{< relref "/docs/linux/static-ip" >}}).

Damit war der naechste Schritt naheliegend: Public Key auf die Maschine, Alias `dns01` in die
lokale `~/.ssh/config`. Der Thin Client steht ohne Bildschirm und Tastatur da, jeder weitere
Handgriff passiert ueber `ssh dns01`. Was in dem Config-Block steht und warum, habe ich unter
[SSH-Config und Key-Login]({{< relref "/docs/linux/ssh-config" >}}) notiert.

## Der erste Dienst: Pi-hole

Pi-hole laeuft. DNS war der naheliegende Anfang: es ist der eine Dienst, von dem jedes andere
Geraet im Netz sofort etwas hat, ohne dass dort irgendetwas konfiguriert werden muesste.
Ausserdem ist er ein ehrlicher Test fuer den Rest des Aufbaus — faellt der DNS-Server aus,
merkt man das im Haushalt binnen Minuten. Ein Dienst, der 24/7 halten muss, zwingt von Anfang
an zu sauberen Grundlagen.

Die Installation selbst ist ein Einzeiler und nach ein paar Minuten durch. Version 6 ist
dabei ein anderes System als das, was die meisten Anleitungen im Netz beschreiben: sie
installiert sich als Debian-Paket, bringt ihren Webserver selbst mit und legt alles in eine
einzige `pihole.toml`. Was sie ausserdem noch mitbringt — einen NTP-Server zum Beispiel —,
sieht man am ehesten daran, welche Ports danach belegt sind. Der Aufbau, die Werte dieser
Installation und die Befehle fuer den Alltag stehen unter
[Pi-hole als DNS-Server]({{< relref "/docs/linux/pihole" >}}).

Interessanter als die Installation ist, was sie nicht erledigt: Der Router verteilt weiterhin
seine eigene Adresse als Nameserver, also gehen alle Anfragen nach wie vor an Pi-hole vorbei.
Der Thin Client selbst fragt ebenfalls noch den Router statt sich selbst. Und der Upstream,
den der Installer vorschlaegt, ist Google — ein Filter, der jede Anfrage weiterhin dorthin
meldet, ist nur ein halber Gewinn. Ein laufender Dienst und ein benutzter Dienst sind zwei
verschiedene Dinge.

## Als naechstes

Diese drei Punkte zuerst: DHCP des Routers auf `10.10.10.3` umstellen, die `resolv.conf` des
Servers auf `127.0.0.1` ziehen, den Upstream bewusst waehlen. Erst danach ist der Filter
wirklich im Netz — und erst dann lohnt es sich, ueber den naechsten Dienst nachzudenken.
