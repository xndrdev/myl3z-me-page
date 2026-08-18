---
title: Netz-Layout mit VLANs
weight: 20
---

# Netz-Layout mit VLANs

Das Netz besteht bisher aus einem einzigen Segment: `10.10.0.0/16`, alles darin, vom Gateway
ueber den Thin Client bis zum Fernsehgeraet. Das funktioniert, solange man nur eine Maschine
hat. Es bedeutet aber, dass jedes Geraet jedes andere direkt erreicht — der Saugroboter den
Arbeitsrechner, der Fernseher das Management der UDM.

> [!NOTE]
> Diese Notiz beschreibt den **Plan**, nicht den Zustand. Umgesetzt ist bisher nur der
> DNS-Eintrag im DHCP ([Pi-hole per DHCP verteilen]({{< relref "/docs/network/udm-dhcp-dns" >}})).

## Warum jetzt und nicht spaeter

Ein Hypervisor will schon bei der Installation wissen, in welchem Segment er liegt, welche
Adresse er bekommt und ob seine Bridge getaggt arbeitet. Wer das Netz danach umbaut,
konfiguriert ihn ein zweites Mal — an den Bridges einer Maschine ohne Bildschirm, wobei man
sich zuverlaessig selbst aussperrt.

Solange das Homelab aus einer Maschine besteht, kostet der Schnitt einen Abend. Bei fuenf
Maschinen kostet er ein Wochenende und eine Liste von Dingen, die danach nicht mehr gehen.

## Wie fein trennen

Jede Grenze, die man zieht, muss man anschliessend mit Ausnahmen wieder durchloechern — der
Drucker soll erreichbar bleiben, der Fernseher soll sich weiterhin vom Telefon aus bespielen
lassen. Zu viele Segmente erzeugen ein Regelwerk, das niemand mehr im Kopf hat, und ein
Regelwerk, das niemand im Kopf hat, wird beim ersten Problem pauschal aufgemacht.

| Zuschnitt | Aufbau | Lage |
|-----------|--------|------|
| drei Segmente | Infrastruktur und Server zusammen, Clients, IoT und Gaeste zusammen | Wenig Regeln, schnell gebaut. Kuenftige VMs sitzen aber neben dem Management der Netz-Hardware — genau die Nachbarschaft, die man auf einem Hypervisor zum Experimentieren nicht will |
| **fuenf Segmente** | Infrastruktur, Server, Clients, IoT, Gaeste | Trennt die drei Dinge, die wirklich getrennt gehoeren: Netz-Management, selbst gebaute Dienste, fremde Firmware. Das Regelwerk bleibt ueberschaubar |
| fuenf plus Lab | zusaetzlich eine Spielwiese, die nur ins Internet darf | Sinnvoll, aber noch ohne Anlass. Laesst sich spaeter als weiteres VLAN ergaenzen, ohne dass sich am Rest etwas aendert |

Gewaehlt sind fuenf.

## Das Schema

| Segment | VLAN | Netz | Gateway | Was hinein gehoert |
|---------|------|------|---------|--------------------|
| Infrastruktur | 1 (untagged) | `10.10.1.0/24` | `10.10.1.1` | UDM, Switches, Access Points — alles, womit man das Netz selbst verwaltet |
| Server | 10 | `10.10.10.0/24` | `10.10.10.1` | `dns01`, spaeter Proxmox und dessen VMs |
| Clients | 20 | `10.10.20.0/24` | `10.10.20.1` | Laptops, Telefone, Arbeitsrechner, Konsolen |
| IoT | 30 | `10.10.30.0/24` | `10.10.30.1` | Smart Home, Fernseher, Cast-Geraete, Drucker |
| Gaeste | 40 | `10.10.40.0/24` | `10.10.40.1` | Besuch, untereinander isoliert |
| *(Lab)* | *50* | *`10.10.50.0/24`* | — | reserviert, noch nicht angelegt |

Die dritte Stelle der Adresse entspricht der VLAN-ID. Das ist keine technische Notwendigkeit,
sondern eine Lesehilfe: An `10.10.30.47` sieht man ohne Nachschlagen, dass es sich um ein
IoT-Geraet handelt.

**Warum `/24` und nicht weiter `/16`:** Ein `/24` fasst 254 Hosts — mehr als ein Haushalt je
braucht — und begrenzt die Broadcast-Domain auf ein Segment. Vor allem aber ist die Maske
ueberhaupt die Stelle, an der die Trennung stattfindet: Solange alle Geraete in `10.10.0.0/16`
liegen, halten sie einander fuer Nachbarn und reden aneinander vorbei am Gateway. Ohne den
Weg ueber das Gateway greift keine Firewall-Regel.

**Warum das Schema so gewaehlt ist:** Beide bestehenden Adressen bleiben gueltig. Die UDM
behaelt `10.10.1.1` und liegt damit im Infrastruktur-Segment, `dns01` behaelt `10.10.10.3` und
liegt im Server-Segment. Es aendern sich Maske und Gateway, keine einzige Adresse — womit alle
bestehenden Notizen weiter stimmen.

## Was sich an dns01 aendert

In `/etc/network/interfaces` ([Statische IP mit ifupdown]({{< relref "/docs/linux/static-ip" >}})):

| Zeile | vorher | nachher |
|-------|--------|---------|
| `address` | `10.10.10.3/16` | `10.10.10.3/24` |
| `gateway` | `10.10.1.1` | `10.10.10.1` |

Das Geraet selbst muss nichts von VLANs wissen, solange sein Switch-Port das Server-VLAN
**untagged** fuehrt. Tagging braucht erst der Proxmox-Host, der mehrere Segmente gleichzeitig
bedienen soll.

## Der Fallstrick: listeningMode

Der Punkt, an dem der Umbau sonst kippt. Pi-hole steht auf `dns.listeningMode = LOCAL` und
beantwortet damit ausschliesslich Anfragen aus Netzen, in denen der Rechner selbst eine
Adresse hat. Heute ist das dank `/16` das gesamte Netz — nach dem Schnitt ist es nur noch
`10.10.10.0/24`.

Die Folge: Clients aus VLAN 20, 30 und 40 stellen ihre Anfragen, und Pi-hole verwirft sie
kommentarlos. Kein Fehler im Log, kein Hinweis, nur ein Netz ohne Namensaufloesung.

```sh
sudo pihole-FTL --config dns.listeningMode ALL
sudo systemctl restart pihole-FTL
```

Damit beantwortet FTL auch geroutete Anfragen aus den anderen Segmenten. Die Warnung aus
[Pi-hole als DNS-Server]({{< relref "/docs/linux/pihole" >}}) gilt dabei unveraendert: `ALL`
macht aus dem Dienst einen offenen Resolver, sobald er von aussen erreichbar ist. Hinter einem
Gateway ohne Portfreigabe auf 53 ist das unkritisch — die Verantwortung wandert damit
allerdings von Pi-hole in die Firewall.

## Was zwischen den Segmenten erlaubt sein muss

Die Grundregel ist Verbot: Segmente duerfen ins Internet, aber nicht untereinander. Davon
braucht es Ausnahmen, und die erste ist die wichtigste.

| Von | Nach | Wofuer |
|-----|------|--------|
| alle Segmente | `10.10.10.3` Port 53 (TCP/UDP) | **Namensaufloesung.** Ohne diese Regel steht der Haushalt nach dem Umbau ohne DNS da |
| alle Segmente | `10.10.10.3` Port 123 (UDP) | Zeit, falls der NTP-Server von Pi-hole genutzt werden soll |
| Clients | IoT: Drucker, NAS, Cast-Geraete | Drucken und Streamen. Gezielt auf Adressen und Ports, nicht als pauschale Oeffnung |
| Clients | Infrastruktur, Server | Verwaltung — Webinterfaces, SSH. Wer streng sein will, beschraenkt das auf einzelne Adressen |
| IoT, Gaeste | irgendwohin ausser Internet | nichts |
| alle | zurueck auf bestehende Verbindungen | *established/related*, sonst funktioniert keine Antwort |

**mDNS ueber Segmentgrenzen.** Cast-Geraete, AirPlay und Sonos finden sich per Multicast, und
Multicast endet an der Segmentgrenze. Das Telefon im Client-Netz sieht den Fernseher im
IoT-Netz schlicht nicht mehr. Die UDM bringt dafuer einen mDNS-Repeater mit (in der
Netzwerk-Konfiguration als *Multicast DNS* oder *mDNS* gefuehrt), der die Ankuendigungen
zwischen den ausgewaehlten Netzen weiterreicht. Ohne ihn ist die Trennung von Clients und IoT
im Alltag nicht durchzuhalten.

> [!NOTE]
> Der Repeater macht Geraete nur *sichtbar*. Die eigentliche Verbindung danach braucht
> zusaetzlich die Firewall-Ausnahme — beides wird gern verwechselt, wenn der Fernseher zwar in
> der Liste auftaucht, sich aber nicht ansteuern laesst.

## Die Reihenfolge am Umbau-Abend

Die Reihenfolge ist nicht Geschmack, sondern der Unterschied zwischen einem Abend und einem
Abend mit Monitor und Tastatur im Serverschrank.

1. **Neue Netze anlegen**, waehrend das bestehende Netz unveraendert bleibt. VLANs, Subnetze,
   DHCP-Bereiche — noch haengt kein Geraet daran.
2. **In jedem neuen Netz den DNS-Eintrag setzen.** Neue Netze starten mit `Auto` und wuerden
   sonst am Filter vorbeilaufen.
3. **Pi-hole auf `listeningMode = ALL` umstellen** — bevor das erste Geraet in einem anderen
   Segment landet, nicht danach.
4. **`dns01` umziehen.** Der heikle Schritt: Port-Profil auf dem Switch und IP-Konfiguration
   auf der Kiste muessen gleichzeitig passen, und dazwischen bricht die SSH-Sitzung ab. Der
   Weg ohne Bildschirm ist, die Aenderung vorzubereiten und den Neustart zeitversetzt zu
   starten:

   ```sh
   sudo shutdown -r +2
   ```

   In diesen zwei Minuten wird der Switch-Port auf das Server-VLAN gelegt. Die Maschine kommt
   im neuen Segment wieder hoch. Wer die Kiste ohnehin erreichen kann, haengt stattdessen
   einen Monitor an — das ist der ehrlichere Weg.
5. **WLAN-SSIDs den Netzen zuordnen**, Access Points durchstarten lassen.
6. **Restliche Switch-Ports** auf Clients und IoT verteilen.
7. **Das urspruengliche Netz auf `/24` verkleinern.** Zuletzt, weil ab hier alles, was noch im
   alten Bereich haengt, nicht mehr erreichbar ist.
8. **Firewall-Regeln setzen** und von jedem Segment aus gegenpruefen.

> [!WARNING]
> Die UniFi-Geraete selbst — Switches und Access Points — haengen im Infrastruktur-Netz.
> Werden Adressbereich oder VLAN dieses Netzes geaendert, verlieren sie waehrenddessen die
> Verbindung zum Controller und erscheinen als *disconnected*. Das loest sich in der Regel von
> selbst; wer aber gleichzeitig noch die Firewall umbaut, sucht den Fehler danach an zwei
> Stellen auf einmal.

## Pruefen

Von je einem Geraet aus jedem Segment:

```sh
ip -br a                                     # liegt die Adresse im richtigen Netz?
ping -c1 10.10.20.1                          # eigenes Gateway erreichbar
dig +short @10.10.10.3 example.com           # DNS ueber die Segmentgrenze
dig +short @10.10.10.3 dns01.xlab.internal   # lokaler Name loest auf
ping -c1 10.10.10.3                          # sollte aus IoT und Gaesten fehlschlagen
```

Auf `dns01` mitlesen, ob die Anfragen mit ihrer echten Client-Adresse ankommen:

```sh
pihole -t
```

Zwischen den Segmenten wird geroutet, nicht genattet — im Log stehen weiterhin die einzelnen
Geraete und nicht die Gateway-Adresse. Bleibt der Filter stumm, sind entweder die
Firewall-Regel aus Schritt 8 oder der `listeningMode` aus Schritt 3 die Ursache.

## Was danach offen bleibt

**Der Proxmox-Uplink.** Der Port fuer den kuenftigen Hypervisor bekommt ein eigenes Profil:
Server-VLAN untagged fuer das Management, Client- und IoT-VLAN getagged fuer VMs, die dort
hinein gehoeren. Das Profil laesst sich am selben Abend anlegen, solange die Netze ohnehin
offen sind — dann steht der Port fertig da, wenn die Maschine kommt.

**Der zweite Pi-hole.** Ein Segment-Layout aendert nichts daran, dass ein einzelner Thin Client
die Namensaufloesung des ganzen Hauses traegt. Der zweite Resolver ist der erste sinnvolle Gast
auf dem Hypervisor.

**Das Lab-VLAN.** Reserviert als VLAN 50, angelegt wird es, wenn es etwas zu isolieren gibt.

**IPv6.** Ungeklaert, und mit Segmenten wird die Frage groesser statt kleiner: Jedes Netz
bekaeme ein eigenes Praefix, und die Nameserver-Ankuendigung per Router Advertisement muss
ueberall auf Pi-hole zeigen, sonst gilt der Filter je Segment nur zur Haelfte.
