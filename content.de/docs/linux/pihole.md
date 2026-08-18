---
title: Pi-hole als DNS-Server
weight: 70
---

# Pi-hole als DNS-Server

Pi-hole ist ein DNS-Server mit Filter. Fragt ein Geraet nach einer Domain, die auf einer
Blockliste steht, reicht Pi-hole die Anfrage nicht an den Upstream weiter, sondern beantwortet
sie selbst — mit einer Adresse, die ins Leere zeigt. Der Vorteil gegenueber einem Blocker im
Browser ist die Stelle, an der gefiltert wird: die Namensaufloesung nutzt jedes Geraet im
Netz, auch das Fernsehgeraet und das Telefon des Besuchs, auf denen sich nichts installieren
laesst.

Der Preis dafuer steht im selben Satz: ein Dienst, den jedes Geraet nutzt, ist auch ein
Dienst, dessen Ausfall jedes Geraet merkt.

## Voraussetzungen

- Eine feste Adresse, denn jeder Client traegt sie fest ein
  ([Statische IP mit ifupdown]({{< relref "/docs/linux/static-ip" >}})).
- `curl` fuer das Installations-Skript
  ([Grundausstattung nach der Installation]({{< relref "/docs/linux/base-packages" >}})).

## Installation

```sh
curl -sSL https://install.pi-hole.net | bash
```

Was dieser Einzeiler an Vertrauen voraussetzt und wie man Herunterladen und Ausfuehren trennt,
steht unter
[Skripte aus dem Netz ausfuehren]({{< relref "/docs/linux/base-packages#skripte-aus-dem-netz-ausfuehren" >}}).

Das Skript startet einen Dialog und fragt der Reihe nach das Interface ab, den Upstream-DNS,
die vorgeschlagene Blockliste, ob Anfragen protokolliert werden sollen und wie ausfuehrlich.
Am Ende gibt es ein generiertes Passwort fuer das Webinterface aus — das ist der einzige
Moment, in dem es im Klartext zu sehen ist. Ein eigenes setzt man mit `pihole setpassword`.

## Was Version 6 anders macht

Anleitungen zu Pi-hole 5 beschreiben in weiten Teilen ein anderes System. Was sich geaendert
hat und beim Nachschlagen immer wieder auffaellt:

| Frueher | Seit Version 6 |
|---------|----------------|
| Dateien direkt ins System kopiert | Installation ueber ein Debian-Paket `pihole-meta`, das die Abhaengigkeiten per `apt` zieht |
| `lighttpd` und PHP fuer das Webinterface | eigener Webserver in `pihole-FTL`, kein zusaetzlicher Dienst |
| `setupVars.conf` plus eigene `dnsmasq`-Dateien | eine Konfiguration: `/etc/pihole/pihole.toml` |
| nur DNS und optional DHCP | zusaetzlich ein NTP-Server, standardmaessig aktiv |

Was danach lauscht, zeigt:

```sh
ss -tulpn
```

| Port | Wofuer |
|------|--------|
| 53/tcp, 53/udp | DNS — der eigentliche Dienst |
| 80/tcp | Webinterface, unverschluesselt |
| 443/tcp | Webinterface ueber TLS, mit einem selbst ausgestellten Zertifikat |
| 123/udp | NTP-Server, den FTL mitbringt |

Alles davon beantwortet ein einziger Prozess, `pihole-FTL`. Wer auf derselben Maschine
spaeter einen Webserver oder einen eigenen Zeitdienst betreiben will, sollte diese Liste im
Kopf haben.

## Die Konfiguration lesen

Statt in der 70 KB grossen `pihole.toml` zu suchen, fragt man einzelne Werte ab:

```sh
pihole-FTL --config dns.upstreams      # ein Wert
pihole-FTL --config                    # alle Werte, flach untereinander
```

In der Datei selbst ist jeder Wert kommentiert, und alles, was vom Standard abweicht, traegt
die Markierung `### CHANGED` — damit ist auf einen Blick sichtbar, was diese Installation
ausmacht und was einfach nur Voreinstellung ist.

Der Stand nach der Installation auf dem Homelab-Rechner:

| Schluessel | Wert | Bedeutung |
|------------|------|-----------|
| `dns.interface` | `enp1s0` | das Interface, auf dem geantwortet wird |
| `dns.listeningMode` | `LOCAL` | nur Anfragen aus dem eigenen Subnetz werden beantwortet |
| `dns.upstreams` | `8.8.8.8`, `8.8.4.4` | wohin nicht geblockte Anfragen weitergehen |
| `dns.dnssec` | `false` | Signaturen der Antworten werden nicht geprueft |
| `dns.queryLogging` | `true` | jede Anfrage landet mit Client im Log |
| `dns.hosts` | `10.10.10.3 dns01.xlab.internal` | eigener Name fuer den Rechner selbst |
| `dns.blocking.mode` | `NULL` | geblockte Namen werden mit `0.0.0.0` beantwortet |
| `dhcp.active` | `false` | Adressen vergibt weiterhin der Router |

`listeningMode` ist der Wert, der ueber die Angriffsflaeche entscheidet. `LOCAL` beantwortet
Anfragen aus Netzen, in denen der Rechner selbst eine Adresse hat, und ignoriert den Rest.
`ALL` beantwortet alles — auf einer Maschine, die je eine oeffentliche Adresse bekommt, wird
daraus ein offener Resolver, der sich fuer Verstaerkungsangriffe missbrauchen laesst.

`blocking.mode = NULL` bedeutet: eine geblockte Domain loest auf `0.0.0.0` auf. Der Client
laeuft sofort ins Leere, statt eine Verbindung zu Pi-hole aufzubauen und dort auf eine
Sperrseite zu treffen. Das ist schneller und erzeugt weniger Rueckfragen an den Dienst, macht
aber im Browser einen Verbindungsfehler statt eines Hinweises.

Aendern lassen sich Werte ueber dieselbe Option:

```sh
sudo pihole-FTL --config dns.dnssec true
```

> [!NOTE]
> Die `pihole.toml` von Hand zu editieren geht auch, danach braucht es `pihole reloaddns`.
> FTL schreibt die Datei allerdings selbst neu, sobald sich etwas aendert — eigene Kommentare
> darin ueberleben das nicht.

## Lokale Namen

Ein DNS-Server im eigenen Netz kann mehr als filtern: er kann Namen vergeben. Statt sich
Adressen zu merken, traegt man einmal ein, welcher Name auf welche Adresse zeigt — im
Webinterface unter *Settings → Local DNS Records*, in der Konfiguration ist es `dns.hosts`:

```sh
pihole-FTL --config dns.hosts
```

```text
[ 10.10.10.3 dns01.xlab.internal ]
```

Geschrieben werden die Eintraege nach `/etc/pihole/hosts/custom.list`, im Format der
`/etc/hosts`. Der erste Name ist der Rechner selbst: `dns01` fuer die Rolle, `xlab.internal`
als Zone fuer das Homelab.

Die Wahl der Endung ist keine Geschmacksfrage:

| Endung | Lage |
|--------|------|
| `.internal` | seit 2024 von der ICANN ausdruecklich fuer private Netze reserviert und damit dauerhaft frei von Kollisionen mit oeffentlichen Domains |
| `.local` | von mDNS belegt (RFC 6762). Wer sie im DNS verwendet, streitet sich mit Avahi und Bonjour darum, wer antwortet |
| `.lan`, `.home`, `.box` | ueblich, aber nirgends reserviert. Funktioniert, bis jemand die Endung als oeffentliche TLD betreibt |
| eine echte eigene Domain | sauber und aufwendig: die Zone muss gepflegt werden, und intern wie extern soll sie unterschiedlich antworten |

> [!NOTE]
> Der Eintrag gilt nur fuer die Vorwaertsaufloesung. Rueckwaerts antwortet Pi-hole auf
> `dig -x 10.10.10.3` weiterhin mit `pi.hole` — das ist der PTR, den es sich selbst gibt
> (`dns.piholePTR`). Wer beides gleich haben will, aendert diesen Wert.

Davon zu unterscheiden ist `dns.domain.name` (Standard: `lan`). Diese Domain haengt Pi-hole an
Namen an, die es ueber DHCP lernt — solange der Router den DHCP macht, spielt der Wert keine
Rolle.

## Pruefen

Der ehrliche Test kommt nicht vom Server selbst, sondern von einem anderen Rechner im Netz:

```sh
dig +short @10.10.10.3 example.com           # normale Antwort, Upstream funktioniert
dig +short @10.10.10.3 doubleclick.net       # 0.0.0.0 = geblockt
dig +short @10.10.10.3 pi.hole               # 10.10.10.3, die eigene Adresse
dig +short @10.10.10.3 dns01.xlab.internal   # 10.10.10.3, der eigene Eintrag
```

Auf dem Server selbst:

```sh
pihole status                           # lauscht FTL auf Port 53?
systemctl status pihole-FTL
pihole -t                               # Anfragen live mitlesen
```

## Was danach noch offen ist

Nach der Installation laeuft ein DNS-Server. Benutzt wird er deshalb noch nicht.

**Niemand fragt ihn.** Direkt nach der Installation verteilt das Gateway weiterhin seine
eigene Adresse als Nameserver, alle Anfragen gehen also am Filter vorbei. Erst wenn dort
`10.10.10.3` eingetragen ist — oder Pi-hole selbst den DHCP uebernimmt —, kommt etwas an.
Inzwischen erledigt: [Pi-hole per DHCP verteilen]({{< relref "/docs/network/udm-dhcp-dns" >}}).
Bei Clients mit fest eingetragenem DNS oder mit DNS-over-HTTPS im Browser bleibt es trotzdem
dabei.

**Der Server fragt sich nicht selbst.** Seine `/etc/resolv.conf` zeigt weiter auf den Router
und einen oeffentlichen Resolver:

```sh
cat /etc/resolv.conf
```

Damit ist der Rechner, der den Filter betreibt, der einzige im Netz, der ihn nicht nutzt.
Dort gehoert `nameserver 127.0.0.1` hin — und zwar an die richtige Stelle, sonst ist die
Aenderung beim naechsten `ifup` wieder weg
([Fallstrick: dns-nameservers ohne resolvconf]({{< relref "/docs/linux/static-ip#fallstrick-dns-nameservers-ohne-resolvconf" >}})).

**Der Upstream ist eine Entscheidung, keine Voreinstellung.** Der Installer schlaegt Google
vor, und wer bestaetigt, hat einen Filter aufgebaut, der jede einzelne Anfrage weiterhin an
Google meldet — nur gebuendelt ueber eine Adresse. Quad9 (`9.9.9.9`) oder Cloudflare
(`1.1.1.1`) sind andere Anbieter mit anderen Zusagen, aber dieselbe Bauart. Wer niemandem
melden will, was im Haushalt aufgeloest wird, braucht einen eigenen rekursiven Resolver
(`unbound`) hinter Pi-hole.

**DNSSEC ist aus.** Die Antworten des Upstreams werden ungeprueft uebernommen. Anschalten
kostet ein paar Millisekunden pro Anfrage und bricht bei Domains mit kaputten Signaturen — was
dann wie ein Ausfall von Pi-hole aussieht.

**Das Webinterface haengt auf Port 80.** Im LAN erreichbar, unverschluesselt, mit einem
Passwort davor. Das TLS-Zertifikat auf 443 ist selbst ausgestellt, jeder Browser warnt davor.

## Bedienung

| Befehl | Wirkung |
|--------|---------|
| `pihole -up` | Pi-hole aktualisieren (Core, Web, FTL) |
| `pihole -g` | Blocklisten neu einlesen |
| `pihole disable 5m` | Filter fuer fuenf Minuten aussetzen, danach automatisch wieder an |
| `pihole -t` | Anfragen live mitlesen |
| `sudo pihole -q domain.tld` | nachsehen, auf welcher Liste eine Domain steht — ohne `sudo` fragt die CLI nach dem Webinterface-Passwort |
| `pihole setpassword` | Passwort fuer das Webinterface setzen |
| `pihole -r` | Reparatur, wenn nach einem Update etwas nicht mehr startet |

> [!WARNING]
> `pihole disable` ohne Zeitangabe schaltet den Filter dauerhaft ab. Das faellt niemandem
> auf — es funktioniert ja alles.
