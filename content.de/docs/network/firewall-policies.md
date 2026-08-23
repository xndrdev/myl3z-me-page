---
title: Firewall zwischen Segmenten
weight: 40
---

# Firewall zwischen Segmenten

Segmente allein trennen nichts. Sobald die Netze geroutet werden — und das tun sie, sonst kaeme
niemand ins Internet — entscheidet ausschliesslich das Regelwerk auf dem Gateway, wer wen
erreicht. Das VLAN sorgt nur dafuer, dass der Verkehr ueberhaupt am Gateway vorbeikommt und
dort bewertet werden kann.

## Die Grundregel

Verbot als Standard, Erlaubnis als Ausnahme. Jedes Segment darf ins Internet, keines
untereinander — und davon abweichend nur, was einen konkreten Anlass hat.

Der umgekehrte Weg ist verlockend, weil sofort alles funktioniert, und endet zuverlaessig in
einem Regelwerk, das niemand mehr abbauen traut. Wer mit *deny* beginnt, merkt am kaputten
Drucker, welche Ausnahme fehlt. Wer mit *allow* beginnt, merkt gar nichts.

## Die Ausnahmen

| Von | Nach | Wofuer |
|-----|------|--------|
| alle Segmente | `10.10.10.3` Port 53 TCP/UDP | **Namensaufloesung.** Ohne diese Regel steht der Haushalt nach dem Umbau ohne DNS da |
| alle Segmente | `10.10.10.3` Port 123 UDP | Zeit, falls Pi-hole als NTP-Server dient |
| Clients | IoT: Drucker, Cast-Geraete, NAS | Drucken und Streamen — gezielt auf Adresse und Port, nicht pauschal |
| Clients | Server | Verwaltung: Webinterfaces, SSH |
| Clients | Infrastruktur `10.10.1.1` Port 443 | die UDM-Oberflaeche, falls man sie aus dem Alltagsnetz erreichen will |
| Kids | Internet, DNS | sonst nichts — insbesondere nicht Clients oder Server |
| IoT, Gaeste | nur Internet und DNS | keine Segmentuebergaenge |
| alle | established/related | ohne diese Regel funktioniert keine Antwort auf eine erlaubte Anfrage |

Die letzte Zeile ist keine Kuer. Eine Firewall bewertet Pakete einzeln; ohne Zustandsregel
duerfte die Anfrage hinaus, die Antwort aber nicht zurueck. Bei UniFi ist sie voreingestellt
vorhanden — man muss sie nur beim Aufraeumen stehen lassen.

## Richtung ist nicht symmetrisch

Der Punkt, an dem Regelwerke meist zu weit werden. *Clients duerfen zum Drucker im IoT-Netz*
heisst nicht *IoT und Clients duerfen miteinander reden*. Die Gegenrichtung bleibt gesperrt,
und die Antwort des Druckers kommt trotzdem an — sie faellt unter established/related.

Genau darin liegt der Gewinn der Trennung: Der Fernseher kann kompromittiert sein und trotzdem
keine Verbindung zum Arbeitsrechner aufbauen, waehrend das Streamen vom Telefon aus
funktioniert.

## Zonen statt Einzelregeln

Neuere UniFi-Versionen fassen Netze zu **Zonen** zusammen (*Settings → Security → Policy
Engine*, je nach Stand auch *Firewall → Zone Matrix*). Statt Regeln zwischen einzelnen Netzen
pflegt man eine Matrix zwischen Zonen — bei sechs Segmenten der deutlich uebersichtlichere
Weg.

Sinnvolle Zuordnung fuer dieses Layout:

| Zone | Netze |
|------|-------|
| *Internal* | Infrastruktur, Server |
| *Trusted* | Clients |
| *Restricted* | Kids |
| *IoT* | IoT |
| *Guest* | Gaeste |

Die Zone-Matrix steht dann auf *Block* ausser dort, wo die Tabelle oben eine Ausnahme
vorsieht. Auf aelteren Staenden gibt es stattdessen *Settings → Firewall Rules* mit
*LAN In / LAN Out / LAN Local* — dieselbe Logik, nur einzeln zu pflegen.

> [!NOTE]
> **LAN In** bewertet Verkehr, der aus einem Netz **in** die UDM kommt und weitergeroutet wird
> — dort gehoeren die Regeln zwischen Segmenten hin. **LAN Local** bewertet Verkehr **an die
> UDM selbst**, also ihre Oberflaeche und ihre Dienste. Wer die Segmenttrennung in *LAN Local*
> baut, wundert sich, warum sie nicht greift.

## Reihenfolge

Regeln werden von oben nach unten abgearbeitet, die erste passende gewinnt. Daraus folgt die
einzige Sortierregel, die man braucht: **spezifisch vor pauschal.**

```text
1. allow  IoT -> 10.10.10.3:53          (Ausnahme)
2. block  IoT -> alle lokalen Netze     (Pauschale)
```

Vertauscht man beides, greift die Pauschale zuerst, und die Ausnahme darunter wird nie
erreicht — die Regel steht sichtbar in der Liste und tut nichts. Das ist der haeufigste Grund
fuer „die Regel ist doch da, funktioniert aber nicht".

## mDNS ueber Segmentgrenzen

Cast-Geraete, AirPlay und Sonos finden sich per Multicast, und Multicast endet an der
Segmentgrenze. Das Telefon im Client-Netz sieht den Fernseher im IoT-Netz schlicht nicht mehr.

Die UDM bringt dafuer einen Repeater mit, in der Netzwerk-Konfiguration als *Multicast DNS*
oder *mDNS* gefuehrt. Er reicht die Ankuendigungen zwischen ausgewaehlten Netzen weiter.

> [!NOTE]
> Der Repeater macht Geraete nur *sichtbar*. Die eigentliche Verbindung danach braucht
> zusaetzlich die Firewall-Ausnahme — beides wird gern verwechselt, wenn der Fernseher zwar in
> der Liste auftaucht, sich aber nicht ansteuern laesst.

## DNS erzwingen

Ein Teil der IoT-Hardware bringt `8.8.8.8` fest eingetragen mit und fragt nie nach, was DHCP
vorschlaegt. Dagegen hilft nur die Firewall:

| Ansatz | Wirkung |
|--------|---------|
| Port 53 nach draussen sperren, ausser fuer `10.10.10.3` | Das Geraet bekommt keine Antwort und faellt in der Regel auf den DHCP-Resolver zurueck |
| Port 53 per NAT auf `10.10.10.3` umbiegen | Das Geraet merkt nichts und wird trotzdem gefiltert |
| Port 853 sperren | schaltet DNS over TLS ab |

DNS over HTTPS laesst sich so nicht fassen — es ist von normalem HTTPS nicht zu unterscheiden
und nur ueber Blocklisten bekannter Endpunkte in Pi-hole selbst einzudaemmen. Die Einzelheiten
stehen unter
[Pi-hole per DHCP verteilen]({{< relref "/docs/network/udm-dhcp-dns#was-dhcp-nicht-erledigt" >}}).

## Pruefen

Regeln sind erst fertig, wenn beide Richtungen geprueft sind — die erlaubte **und** die
verbotene. Von einem Geraet aus jedem Segment:

```sh
dig +short @10.10.10.3 example.com   # muss ueberall gehen
ping -c1 9.9.9.9                     # Internet, muss ueberall gehen
ping -c1 10.10.20.5                  # aus IoT/Gaesten/Kids: muss scheitern
ping -c1 10.10.1.1                   # aus IoT/Gaesten: muss scheitern
```

Die aussagekraeftige Zeile ist die dritte. Eine Trennung, die man nur daran erkennt, dass
nichts kaputt ist, hat man nicht geprueft.

Nuetzlich ist der Unterschied im Fehlerbild: Ein sofortiges *No route to host* kommt von einer
Regel, die aktiv ablehnt. Ein stiller Timeout heisst entweder *drop* oder: dort ist gar kein
Geraet. Wer das verwechselt, sucht eine Firewall-Regel fuer eine Maschine, die nur ausgeschaltet
ist.
