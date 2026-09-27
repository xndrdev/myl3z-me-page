---
title: Homelab — Pi-hole bekommt neue Blocklisten
date: 2026-09-27
---

Ich habe mein Pi-hole optimiert und zusaetzliche HaGeZi-Blocklisten eingebunden. Nachdem
die DNS-Anfragen im Homelab bereits ueber den Filter laufen, geht es diesmal um die Regeln,
nach denen er entscheidet.

<!--more-->

## Was dazugekommen ist

Die Auswahl besteht aus **Multi Pro**, **Threat Intelligence Feeds in der Full-Version**,
**Gambling Full**, **NSFW** und **No-SafeSearch**. Das
[HaGeZi-Preset](https://hagezi-mirror.dnsbunker.org/dlg.html?src=github&tif=full&gambling=full&tier=pro&items=nsfw%2Cnosafesearch&tool=pi-hole)
haelt die Zusammenstellung fest.

Pro deckt Werbung, Tracking und Telemetrie ab. TIF erweitert die lokale Filterung bekannter
schadhafter Domains. Die anderen drei Listen ergaenzen Inhaltsfilter fuer Gluecksspiel,
Erwachseneninhalte und Suchmaschinen ohne SafeSearch-Unterstuetzung.

Damit faellt ein Teil der Entscheidung ueber bekannte Schadsoftware-Domains jetzt schon
lokal in Pi-hole. Der im
[frueheren DNS-Beitrag]({{< relref "/posts/homelab/dns-upstream" >}}) beschriebene
Quad9-Upstream bleibt eine weitere Filterstufe fuer Anfragen, die Pi-hole weiterleitet.

## Was das im Alltag bedeutet

Pi-hole liest die Listen ueber Gravity ein und prueft DNS-Anfragen gegen diesen lokalen
Bestand. Die Listenanbieter werden dabei nicht fuer jeden Website-Aufruf kontaktiert.
Welche Eintraege dazukommen oder verschwinden, uebernimmt Pi-hole beim Aktualisieren.

Eine groessere Auswahl kann allerdings auch einen benoetigten Dienst treffen. Dann ist das
Query Log die erste Anlaufstelle, um die verantwortliche Domain zu finden und gezielt
freizugeben. Und der Name No-SafeSearch ist leicht misszuverstehen: Die Liste schaltet
SafeSearch nicht automatisch ein.

Wie die Filterung funktioniert, welche fuenf URLs zu dieser Auswahl gehoeren und wo
DNS-Blocklisten an Grenzen stossen, steht auf der neuen Infoseite
[Pi-hole und Blocklisten verstehen]({{< relref "/docs/linux/pihole-blocklisten" >}}).
