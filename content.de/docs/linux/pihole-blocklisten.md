---
title: Pi-hole und Blocklisten verstehen
weight: 75
---

# Pi-hole und Blocklisten verstehen

Pi-hole filtert DNS-Anfragen im eigenen Netz. Bevor ein Browser eine Website laden kann,
braucht er die IP-Adresse zu ihrem Namen. Genau bei dieser Namensaufloesung entscheidet
Pi-hole, ob es eine nutzbare Antwort zurueckgibt. Eine Blockliste liefert die Regeln dafuer:
Domains, die etwa fuer Werbung, Tracking oder Schadsoftware bekannt sind.

Die Installation steht unter [Pi-hole als DNS-Server]({{< relref "/docs/linux/pihole" >}}).
Hier geht es darum, wie aus abonnierten Listen ein Filter wird und welche HaGeZi-Listen ich
inzwischen eingebunden habe.

## Was bei einer DNS-Anfrage passiert

Eine Website besteht oft aus Inhalten von mehreren Domains: dem eigentlichen Artikel,
Bildern, einem Analysedienst und einem Werbenetzwerk. Fuer diese Namen braucht das Geraet
jeweils eine Adresse, sofern sie nicht schon in seinem Cache liegt.

1. **Das Geraet fragt Pi-hole.** Dafuer muss es Pi-hole als DNS-Server benutzen. Im Homelab
   verteilt die UDM diese Einstellung per DHCP
   ([Pi-hole per DHCP verteilen]({{< relref "/docs/network/udm-dhcp-dns" >}})).
2. **Pi-hole prueft die geltenden Regeln.** Dazu gehoeren die Blocklisten sowie eigene
   Freigaben und Sperren fuer den jeweiligen Client.
3. **Eine gesperrte Domain bekommt eine Blockantwort.** Im Standardmodus `NULL` ist das
   bei IPv4 `0.0.0.0`, bei IPv6 `::`. Damit fehlt die nutzbare Zieladresse.
4. **Eine erlaubte Anfrage wird aufgeloest.** Pi-hole kann aus lokalen DNS-Eintraegen oder
   seinem Cache antworten. Andernfalls fragt es den konfigurierten Upstream.

Der eigentliche Seiteninhalt laeuft anschliessend direkt zwischen Geraet und Webserver.
Pi-hole ist dabei kein Proxy, der den HTML-Code liest oder Bilder untersucht. Die
[Pi-hole-Dokumentation zu den Blockantworten](https://docs.pi-hole.net/ftldns/blockingmode/)
beschreibt auch die Alternativen zum `NULL`-Modus.

## Gravity: aus Listen wird eine lokale Datenbank

Eine abonnierte Blockliste ist zunaechst eine URL zu einer Textdatei. Die Eintraege pflegt
deren Anbieter; Pi-hole holt sie bei einer Aktualisierung ab und verarbeitet sie lokal.
Dieser Vorgang heisst **Gravity**. Die daraus aufgebauten Sperreintraege liegen in
`/etc/pihole/gravity.db`.

Bei einer DNS-Anfrage wird also nicht erst GitHub oder der Listenanbieter gefragt.
Pi-hole entscheidet anhand des zuletzt eingelesenen Bestands. Aenderungen beim Anbieter
kommen erst mit dem naechsten Gravity-Lauf an. Standardmaessig passiert das woechentlich;
manuell geht es auf dem Pi-hole-Rechner mit:

```sh
pihole -g
```

Das aktualisiert die Listen, nicht die Pi-hole-Software. Den Ablauf beschreibt die
[offizielle Gravity-Dokumentation](https://docs.pi-hole.net/main/pihole-command/#gravity).

Ueberschneidungen zwischen Listen sind normal. Dieselbe Domain auf drei Listen bedeutet
nicht dreifachen Schutz. Pi-hole behaelt die Zuordnung zur jeweiligen Quelle, unter anderem
damit unterschiedliche Gruppen unterschiedliche Listen verwenden koennen. Deshalb sind
Listeneintraege und einzigartige Domains auch nicht dieselbe Zahl.
([Aufbau der Domain-Datenbank](https://docs.pi-hole.net/database/domain-database/))

## Meine HaGeZi-Auswahl

Ich habe diese fuenf Listen eingebunden. Die Auswahl laesst sich mit
[meinem HaGeZi-Preset](https://hagezi-mirror.dnsbunker.org/dlg.html?src=github&tif=full&gambling=full&tier=pro&items=nsfw%2Cnosafesearch&tool=pi-hole)
wieder oeffnen:

| Liste | Aufgabe |
|-------|---------|
| **Multi Pro** | Breiter Filter gegen Werbung, Tracking, Telemetrie und bekannte schadhafte Domains |
| **Threat Intelligence Feeds (TIF), Full** | Zusaetzliche Abdeckung bekannter Malware-, Phishing- und Command-and-Control-Domains |
| **Gambling, Full** | Domains rund um Gluecksspiel und Wetten |
| **NSFW** | Domains mit Inhalten fuer Erwachsene |
| **No-SafeSearch** | Suchmaschinen, die SafeSearch nicht unterstuetzen |

Pro und TIF zielen auf Datenschutz und bekannte Bedrohungen. Gambling, NSFW und
No-SafeSearch legen zusaetzlich fest, welche Inhalte erreichbar sein sollen. Die
[HaGeZi-Listenbeschreibung](https://github.com/hagezi/dns-blocklists) erklaert den Umfang.

**No-SafeSearch aktiviert keinen Suchfilter bei Google oder Bing.** Die Liste sperrt
bestimmte Suchmaschinen als Ganzes. SafeSearch bei einem erlaubten Anbieter zu erzwingen
ist eine eigene Konfiguration. Auch NSFW ist eine Domainliste, keine Erkennung einzelner
Bilder oder Beitraege auf einer ansonsten erlaubten Plattform.

## Die passenden URLs eintragen

Im verlinkten Generator ist **Pi-hole** als Werkzeug ausgewaehlt. Das ergibt hier das
**Adblock-Format**. Die Quelle ist durch `src=github` auf **GitHub (raw)** gesetzt, obwohl
der Generator selbst auf einem Mirror liegt. Daraus entstehen diese fuenf Listen-URLs:

```text
https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/pro.txt
https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/tif.txt
https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/gambling.txt
https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/nsfw.txt
https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/nosafesearch.txt
```

In Pi-hole 6 kommen die URLs im Webinterface unter **Lists** als **Blocklisten** hinein.
Der Preset-Link selbst ist eine Webseite und gehoert nicht in dieses Feld. Anschliessend
die Listen aktivieren, die gewuenschten Gruppen zuordnen und Gravity aktualisieren.
In der Ausgabe pruefen, ob alle fuenf Quellen verarbeitet wurden.

Der Formatname bedeutet nicht, dass Pi-hole alle Funktionen eines Browserblockers bekommt.
Es verarbeitet DNS-taugliche Regeln wie `||example.com^`, die eine Domain samt Subdomains
abdecken. Regeln zum Ausblenden einzelner Seitenelemente gehoeren weiterhin in den Browser.
Die Formatzuordnung steht auch in der
[HaGeZi-Uebersicht](https://github.com/hagezi/dns-blocklists#overview).

## Wenn eine Seite nicht mehr funktioniert

Auch gepflegte Listen koennen eine benoetigte Domain treffen. Im **Query Log** zuerst den
betroffenen Client und den Zeitpunkt des Fehlers ansehen. Oft ist die Website selbst
erlaubt, aber eine zusaetzliche Domain fuer Anmeldung oder Inhalte gesperrt.

Auf dem Pi-hole-Rechner laesst sich nachsehen, welche Listen oder eigenen Regeln zu einem
Namen passen. `example.com` ist hier durch die betroffene Domain zu ersetzen:

```sh
sudo pihole -q example.com
```

Die [Listensuche](https://docs.pi-hole.net/main/pihole-command/#query) erklaert die Herkunft
einer Regel; das Query Log zeigt, wie Pi-hole die konkrete Anfrage behandelt hat. Bei einer
Fehlblockierung die benoetigte Domain gezielt erlauben und den Grund als Kommentar
festhalten. Eine passende eigene Allowlist-Regel hat Vorrang vor einer abonnierten
Blockliste. Sie muss fuer dieselbe Client-Gruppe gelten.
([Regelprioritaeten](https://docs.pi-hole.net/database/domain-database/#priorities))

Gruppen erlauben etwa, Inhaltsfilter nur bestimmten Geraeten zuzuordnen. Entscheidend ist
die Verbindung zwischen Client, Gruppe und Liste; ein VLAN allein stellt das nicht ein.
Die [Pi-hole-Beispiele zur Gruppenverwaltung](https://docs.pi-hole.net/group_management/example/)
zeigen diese Zuordnung.

## Wo der Filter seine Grenzen hat

Pi-hole sieht Domainnamen, keine vollstaendigen URLs. Liegen Werbung und gewuenschter Inhalt
unter demselben Namen, kann ein DNS-Filter sie nicht sauber trennen. Deshalb verschwinden
beispielsweise YouTube-Anzeigen damit nicht verlaesslich. Ein Browserblocker kann an
anderen Stellen ansetzen und bleibt eine sinnvolle Ergaenzung.

Der Filter greift ausserdem nur bei Anfragen, die ihn erreichen. Ein externer DNS-Server,
DNS-over-HTTPS im Browser oder ein VPN mit eigenem DNS kann daran vorbeifuehren. Bereits
zwischengespeicherte Antworten und bestehende Verbindungen verschwinden nach einer
Listen-Aenderung ebenfalls nicht sofort.

Eine hohe Blockierquote allein sagt deshalb wenig ueber die Qualitaet aus. Entscheidend
ist, dass unerwuenschte Verbindungen unterbunden werden und benoetigte Dienste funktionieren.
Die Auswahl und Pflege der Listen gehoert ebenso zum Betrieb wie die Installation.
