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
angefordert habe.

Den Installer-Stick habe ich im Terminal geschrieben — die Schritte stehen unter
[Boot-Stick im Terminal]({{< relref "/docs/linux/bootable-usb" >}}), die Erklaerung der
beteiligten Befehle unter
[lsblk, umount, dd, sync]({{< relref "/docs/linux/disk-commands" >}}).

## Als naechstes: Pi-hole

Der erste Dienst wird Pi-hole. DNS ist der naheliegende Anfang: es ist der eine Dienst, von
dem jedes andere Geraet im Netz sofort etwas hat, ohne dass dort irgendetwas konfiguriert
werden muesste. Ausserdem ist er ein ehrlicher Test fuer den Rest des Aufbaus — faellt der
DNS-Server aus, merkt man das im Haushalt binnen Minuten. Ein Dienst, der 24/7 halten muss,
zwingt von Anfang an zu sauberen Grundlagen.

Notizen dazu folgen, wenn er laeuft.
