---
title: VLANs auf der UDM einrichten
weight: 30
---

# VLANs auf der UDM einrichten

Der Ablauf, in dem der Umbau von einem flachen `10.10.0.0/16` auf sechs Segmente tatsaechlich
funktioniert hat — inklusive der zwei Stellen, an denen der geplante Ablauf nicht funktioniert
hat. Das Layout dahinter steht unter
[Netz-Layout mit VLANs]({{< relref "/docs/network/vlan-layout" >}}), die Begriffe unter
[VLANs verstehen]({{< relref "/docs/network/vlan-grundlagen" >}}).

## Zuerst: das alte Netz verkleinern

Der Plan sah vor, die neuen Netze anzulegen, waehrend das bestehende unberuehrt bleibt, und das
alte Netz erst zum Schluss auf `/24` zu verkleinern. Das laesst sich nicht durchhalten.

**UniFi verweigert jedes neue Netz, dessen Bereich sich mit einem bestehenden ueberschneidet.**
Solange das Default-Netz `10.10.0.0/16` ist, liegt `10.10.20.0/24` darin — und das Anlegen
scheitert mit einer Overlap-Meldung. Betroffen ist jedes einzelne neue Segment, denn alle
liegen in `10.10.0.0/16`.

Der Schritt muss deshalb ganz nach vorne:

1. *Settings → Networks →* das Default-Netz oeffnen
2. Den Bereich von `10.10.0.0/16` auf `10.10.1.0/24` aendern, Gateway bleibt `10.10.1.1`
3. DHCP-Bereich auf das neue `/24` anpassen

Ab diesem Moment ist alles, was ausserhalb von `10.10.1.0/24` eine Adresse hatte, nicht mehr
erreichbar. Bei einem `/16` mit gewachsenem DHCP-Pool ist das der halbe Haushalt.

> [!WARNING]
> Geraete verlieren dabei nicht sofort ihre Adresse — sie behalten die alte bis zum naechsten
> Renew und sind bis dahin in einem Netz, das es nicht mehr gibt. Wer nicht warten will,
> startet die betroffenen Geraete durch. Die eigene Arbeitsmaschine sollte man dabei bewusst
> zuletzt anfassen.

Praktisch heisst das: Der Umbau beginnt mit dem Schritt, der am meisten kaputt macht. Das
ist unangenehm, aber die Alternative gibt es nicht.

## Die neuen Netze

*Settings → Networks → Create New Network*, je Segment einmal. Die Felder, auf die es ankommt:

| Feld | Wert | Warum |
|------|------|-------|
| *Router* / *VLAN ID* | 10, 20, 25, 30, 40 | die VLAN-Nummer aus dem Layout |
| *Gateway IP / Subnet* | `10.10.<vlan>.1/24` | dritte Stelle gleich VLAN-ID, als Lesehilfe |
| *DHCP Mode* | Server | jedes Segment bringt seinen eigenen Bereich mit |
| *DHCP Name Server* | **Manual**, `10.10.10.3` | sonst faellt das Segment still aus dem Filter |

Das letzte Feld ist das, das man vergisst. Neue Netze starten auf `Auto`, tragen damit die UDM
als Nameserver ein und laufen an Pi-hole vorbei — ohne Fehlermeldung, ohne Eintrag im Log,
nur ohne Filter. Die Begruendung steht unter
[Pi-hole per DHCP verteilen]({{< relref "/docs/network/udm-dhcp-dns" >}}).

## Der teuerste Fehler: die NAT-Regel

Die Netze standen, die Gateways antworteten, DNS loeste auf — und kein Geraet in den neuen
Segmenten kam ins Internet. Kein Timeout beim Gateway, kein DNS-Problem: Namen wurden
aufgeloest, Pakete gingen raus, es kam nichts zurueck.

Die Ursache liegt in einer Regel, die man beim Anlegen eines Netzes nie zu Gesicht bekommt.
Unter *Settings → Routing → NAT* steht eine Masquerade-Regel mit dem Namen **LAN to WAN
Masquerade** — und deren *Source* war auf das Infrastruktur-Netz `10.10.1.0/24` gesetzt.

Was daraus folgt, ist genau das beobachtete Bild:

- Pakete aus `10.10.20.0/24` werden korrekt **geroutet**, das Gateway nimmt sie an
- Beim Verlassen Richtung WAN werden sie **nicht maskiert**, behalten also ihre private
  Absenderadresse
- Der Upstream verwirft sie oder kann die Antwort nicht zurueckrouten

Der Fix ist ein Feld: *Source* auf **Any** setzen. Alternativ eine eigene Masquerade-Regel je
Segment, was mehr Kontrolle gibt und mehr Pflege kostet.

> [!NOTE]
> Ob die Regel ueberhaupt sichtbar ist, haengt von der UniFi-Version ab. Auf aelteren Staenden
> maskiert die UDM implizit alles und die Regel existiert nicht — dann faellt dieser Schritt
> ersatzlos weg. Taucht sie auf, ist ihre *Source* zu pruefen, bevor die ersten Geraete
> umziehen.

Das Symptom ist deshalb so teuer, weil es nach einem DNS-Problem aussieht und man dort zuerst
sucht. Der Test, der die beiden auseinanderhaelt:

```sh
ping -c1 10.10.20.1          # eigenes Gateway  -> geht
dig +short @10.10.10.3 example.com   # DNS      -> geht
ping -c1 9.9.9.9             # Internet per IP  -> geht nicht
```

Antwortet das Gateway und loest DNS auf, waehrend eine nackte IP im Internet nicht erreichbar
ist, liegt es nicht am Namen und nicht an der Route, sondern am NAT.

## Der Rest des Abends

Nachdem Verkleinerung, Netze und NAT stehen, ist die Reihenfolge unkritisch:

1. **Pi-hole auf `listeningMode = ALL`** — bevor das erste Geraet in einem anderen Segment
   landet. Details unter [Netz-Layout mit VLANs]({{< relref "/docs/network/vlan-layout#der-fallstrick-listeningmode" >}})
2. **`dns01` umziehen** — Maske auf `/24`, Gateway auf `10.10.10.1`, Switch-Port auf das
   Server-VLAN untagged
3. **SSIDs den Netzen zuordnen** — siehe [WLAN auf VLANs verteilen]({{< relref "/docs/network/wlan-ssid-vlan" >}})
4. **Switch-Ports auf Profile verteilen**
5. **Firewall-Policies setzen** — siehe [Firewall zwischen Segmenten]({{< relref "/docs/network/firewall-policies" >}})

> [!WARNING]
> Switches und Access Points haengen selbst im Infrastruktur-Netz. Waehrend dessen Bereich
> wechselt, verlieren sie die Verbindung zum Controller und erscheinen als *disconnected*. Das
> loest sich meist von selbst, sobald sie eine Adresse im neuen Bereich bekommen. Wer
> gleichzeitig die Firewall umbaut, sucht danach an zwei Stellen auf einmal.

## Port-Profile

*Settings → Profiles → Port Profiles*. Ein Profil beschreibt, welche VLANs ein Switch-Port
fuehrt:

| Profil | Native VLAN | Tagged | Fuer |
|--------|-------------|--------|------|
| Clients | Clients (20) | — | Arbeitsplaetze, Dock, Konsole |
| IoT | IoT (30) | — | Drucker, Fernseher, Smart-Home-Bridge |
| Server | Server (10) | — | `dns01` |
| AP-Uplink | Infrastruktur (1) | *Allow All* | Access Points, die mehrere SSIDs bedienen |
| Hypervisor | Server (10) | Clients (20), IoT (30) | Proxmox-Host mit VMs in mehreren Segmenten |

Der AP-Uplink ist der Port, an dem man sich am ehesten selbst aussperrt: Fehlt dort ein VLAN in
der Tagged-Liste, funktioniert die zugehoerige SSID nicht — die Geraete verbinden sich, bekommen
aber keine Adresse. *Allow All* ist im Heimnetz der pragmatische Weg, weil damit jede kuenftige
SSID ohne Portaenderung funktioniert.

## Pruefen

Aus jedem Segment einmal, am besten in dieser Reihenfolge:

```sh
ip -br a                                     # Adresse im richtigen Netz?
ping -c1 10.10.20.1                          # eigenes Gateway
ping -c1 9.9.9.9                             # Internet per IP  (prueft NAT)
dig +short @10.10.10.3 example.com           # DNS ueber die Segmentgrenze
dig +short @10.10.10.3 dns01.xlab.internal   # lokaler Name
ping -c1 10.10.10.3                          # aus IoT und Gaesten: soll scheitern
```

Auf `dns01` mitlesen, ob die Anfragen mit ihrer echten Client-Adresse ankommen:

```sh
pihole -t
```

Zwischen den Segmenten wird geroutet, nicht genattet — im Log stehen weiterhin die einzelnen
Geraete und nicht die Gateway-Adresse. Steht dort die Gateway-Adresse, laeuft der Verkehr
ueber eine NAT-Regel, die er nicht nehmen sollte.
