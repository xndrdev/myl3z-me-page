---
title: Homelab, erster Schritt — Thin Client mit Debian 13
date: 2026-08-16
aliases:
  - /posts/homelab-thin-client/
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

Zum Schluss hat der Rechner noch einen Namen bekommen, eingetragen in Pi-hole selbst:
`dns01.xlab.internal`. `dns01` fuer die Rolle, `xlab.internal` als Zone fuer das Homelab. Die
Endung ist bewusst gewaehlt — `.internal` ist seit 2024 ausdruecklich fuer private Netze
reserviert, waehrend `.local` schon mDNS gehoert und `.lan` nirgends festgeschrieben ist.
Damit steht das Schema, unter dem alles Weitere ansprechbar sein soll, bevor es mehr als eine
Maschine gibt.

## Als naechstes: das Netz

Der naechste Schritt ist nicht der naechste Rechner, sondern die UDM. Der Grund ist der erste
der drei offenen Punkte: Solange das Gateway seine eigene Adresse als Nameserver verteilt,
fragt kein Geraet im Haushalt den Filter, der jetzt laeuft. Ein Feld im DHCP entscheidet
darueber, ob der Aufwand der letzten Tage ueberhaupt etwas bewirkt.

Der zweite Grund ist die Reihenfolge. Ein Hypervisor will schon bei der Installation wissen,
in welchem Segment er liegt, welche Adresse er bekommt und ob seine Bridge getaggt arbeitet.
Wer das Netz danach umbaut, konfiguriert ihn ein zweites Mal — und wer an den Bridges einer
Maschine ohne Bildschirm schraubt, sperrt sich dabei gern selbst aus. Adressbereiche,
Segmente und Namen legt man besser fest, solange es eine Maschine ist und nicht fuenf.

Danach kommt Proxmox. Bis hierher ist das Homelab eine Maschine mit einem Dienst darauf, und
jeder weitere Dienst wuerde sich dieselbe Debian-Installation teilen: gemeinsame Pakete,
gemeinsame Ausfaelle, kein sauberes Zurueck. Virtualisierung dreht das um — eine Basis,
darauf getrennte Maschinen, die einzeln gesichert, kopiert und weggeworfen werden koennen.

Notizen dazu folgen, wenn es laeuft.
