---
title: Homelab, sechster Schritt — Firewall-Regeln und weniger WLAN-Auslastung
date: 2026-10-04
---

Die sechs Netzsegmente haben jetzt die Firewall-Regeln, die beim VLAN-Umbau noch gefehlt
haben. Gaeste, IoT und Kids bekommen begrenzte Zugriffe, die Router-Verwaltung ist aus diesen
Netzen gesperrt. Nebenbei laeuft der DNS-Server wieder mit Gigabit, und die beiden Access
Points teilen sich die Arbeit auf 2,4 GHz besser auf.

<!--more-->

## Was zwischen den Segmenten fehlte

Seit dem [VLAN-Umbau]({{< relref "/posts/homelab/vlan-umbau" >}}) gibt es eigene Netze fuer
Infrastruktur, Server, Clients, Kids, IoT und Gaeste. Bei der Pruefung der Dream Machine
zeigte sich allerdings: Alle sechs lagen in derselben Firewall-Zone *Internal*. Dort galt
*Internal → Internal: Allow All*. Die VLANs waren angelegt, der geroutete Verkehr zwischen
ihnen aber uneingeschraenkt erlaubt.

Die 63 vorhandenen Firewall-Regeln stammten alle vom System. Das geplante
[Regelwerk zwischen den Segmenten]({{< relref "/docs/network/firewall-policies" >}}) war noch
nicht umgesetzt. Auch die Client-Isolation im Gaeste-WLAN ersetzte keine Sperre in Richtung
Server oder Router-Verwaltung.

Am Internetanschluss war der Ausgangszustand besser: keine Port-Weiterleitungen, UPnP aus,
IPv6 am WAN aus und unerwuenschte eingehende Verbindungen durch Standardregeln gesperrt.
Firmware, automatische Updates und Backups waren ebenfalls in Ordnung. Die Luecke lag bei
den Zugriffen innerhalb des Hauses.

## Fuenf Zonen fuer sechs Netze

Die Netze sind jetzt nach ihrem Vertrauensniveau aufgeteilt:

| Zone | Netze |
|------|-------|
| Internal | Infrastruktur und Server |
| Trusted | Clients mit Arbeitsrechnern, Telefonen und Laptops |
| Restricted | Kids |
| IoT | Smart Home und Fernseher |
| Guest | Gaeste |

Fuer neue Verbindungen gelten diese Freigaben:

| Von | Nach | Erlaubt |
|-----|------|---------|
| Trusted | Internal, IoT und UDM | alle Dienste |
| Restricted, IoT und Guest | DNS-Server `10.10.10.3` | DNS auf Port 53, TCP und UDP |
| Restricted, IoT und Guest | UDM | DHCP fuer die Adressvergabe |
| Restricted und IoT | UDM | zusaetzlich mDNS fuer die Geraeteerkennung |
| alle | Internet | ausgehende Verbindungen |

Andere neue Verbindungen zwischen diesen lokalen Zonen sind gesperrt. Antworten auf erlaubte
Verbindungen duerfen zurueck. Damit kann ein Client einen Dienst im IoT-Netz ansprechen, ohne
dass ein IoT-Geraet selbst eine neue Verbindung zum Arbeitsrechner aufbauen darf.

Infrastruktur und Server bleiben gemeinsam in *Internal* und sind damit weiterhin nicht
gegeneinander abgeschottet. Auch die Freigaben aus *Trusted* sind noch breit; sie sind nicht
auf einzelne Verwaltungsdienste oder Geraete begrenzt.

Was eine Firewall dabei eigentlich prueft und wie sie Antworten erkennt, steht auf der neuen
Infoseite [Firewalls verstehen]({{< relref "/docs/network/firewall-grundlagen" >}}).

## Der Zugriff auf die UDM ist separat geregelt

Bei den neu angelegten Zonen war der Zugriff auf *Gateway* zunaechst vollstaendig erlaubt.
Diese Zone steht fuer die UDM selbst: sowohl ihre Dienste als auch ihre Verwaltungsoberflaeche.
Eine Sperre in Richtung Server schuetzt diese Oberflaeche deshalb noch nicht.

Aus Gaesten, IoT und Kids bleiben jetzt nur die oben genannten Gateway-Dienste erreichbar.
Die Verwaltungsoberflaeche ist aus diesen Netzen gesperrt, aus *Trusted* bleibt sie zugaenglich.
SSH auf der Konsole ist abgeschaltet. Die Angriffserkennung wurde vom reinen Meldemodus auf
Blockieren umgestellt.

## Die Fernseher gehoeren jetzt zum IoT-Netz

Beide Fernseher lagen bisher im Client-Netz. Jetzt bekommen sie Adressen aus
`10.10.30.0/24` und unterliegen den IoT-Regeln. Sie bleiben dabei im bisherigen WLAN `xlab`:
Eine geraetespezifische VLAN-Zuweisung ordnet sie dem anderen Netz zu.

Damit war kein Wechsel auf die nur mit 2,4 GHz betriebene SSID `xlab-IoT` noetig. WLAN-Name und
Passwort an den Fernsehern blieben gleich, ihre Zugriffsrechte im Netz haben sich geaendert.

## Gigabit am DNS-Server und weniger belegte Sendezeit

Der DNS-Server war nur mit 100 Mbit/s angebunden. Nach dem Austausch des Patchkabels handelt
der Anschluss wieder 1000 Mbit/s aus. Die Netzwerkkarte musste nicht ersetzt werden.

Auf 2,4 GHz hatten beide Access Points automatisch Kanal 11 gewaehlt, der auch in der
Nachbarschaft stark belegt war. Sie nutzen jetzt fest Kanal 1 und Kanal 6. Die Mindestdatenrate
wurde von 1 auf 6 Mbit/s angehoben, damit langsame Uebertragungen weniger Sendezeit belegen.
Nach den Aenderungen lag die gemessene Kanalauslastung bei 17 beziehungsweise 30 % statt zuvor
rund 60 %.

Auf 5 GHz blieb die Kanalbreite bei 40 MHz. Breitere Kanaele waren fuer diesen Umbau nicht
noetig.

## Was geprueft ist und was offen bleibt

Nach der schrittweisen Zuordnung der Netze wurden UDM, DNS, Proxmox und Internetzugang vom
Arbeitsnetz aus geprueft. Die Sperrregel vom IoT-Netz in Richtung Server verzeichnete kurz
nach der Umstellung erste Treffer.

Das zeigt, dass Verkehr an dieser Regel geblockt wird. Ein direkter Test der gesperrten
Zugriffe von einem Geraet im Gaestenetz steht noch aus.

Zwei weitere Punkte bleiben offen: Die Verbindung zum zweiten Switch laeuft ueber das Kabel
in der Wand weiterhin mit 100 Mbit/s. Und alle sechs Segmente haengen fuer DNS an einer
einzigen Maschine. Ein zweiter Resolver ist der naechste Schritt.
