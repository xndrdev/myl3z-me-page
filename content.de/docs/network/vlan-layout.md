---
title: Netz-Layout mit VLANs
weight: 20
---

# Netz-Layout mit VLANs

Das Netz bestand aus einem einzigen Segment: `10.10.0.0/16`, alles darin, vom Gateway ueber den
Thin Client bis zum Fernsehgeraet. Das funktioniert, solange man nur eine Maschine hat. Es
bedeutet aber, dass jedes Geraet jedes andere direkt erreicht — der Saugroboter den
Arbeitsrechner, der Fernseher das Management der UDM.

Seit dem 23. August 2026 sind es sechs Segmente. Diese Notiz beschreibt das Layout und die
Ueberlegungen dahinter; der Ablauf des Umbaus steht unter
[VLANs auf der UDM einrichten]({{< relref "/docs/network/unifi-vlan-umbau" >}}), die Begriffe
unter [VLANs verstehen]({{< relref "/docs/network/vlan-grundlagen" >}}).

## Warum vor dem Hypervisor

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
| fuenf Segmente | Infrastruktur, Server, Clients, IoT, Gaeste | Trennt die drei Dinge, die wirklich getrennt gehoeren: Netz-Management, selbst gebaute Dienste, fremde Firmware. Das Regelwerk bleibt ueberschaubar |
| **sechs Segmente** | zusaetzlich Kids | Kindergeraete brauchen andere Blocklisten und Zeitfenster als der Rest. An ein Netz gebunden ist das billiger zu pflegen als geraeteweise |
| plus Lab | zusaetzlich eine Spielwiese, die nur ins Internet darf | Sinnvoll, aber noch ohne Anlass. Laesst sich spaeter als weiteres VLAN ergaenzen, ohne dass sich am Rest etwas aendert |

Geplant waren fuenf, gebaut sind sechs. Das Kids-Segment kam waehrend des Umbaus dazu, weil der
Aufwand fuer ein weiteres Netz an diesem Abend gegen null ging — die Netze waren ohnehin offen,
die SSID war ohnehin neu anzulegen. Nachtraeglich haette es einen zweiten Abend gekostet.

## Das Schema

| Segment | VLAN | Netz | Gateway | Was hinein gehoert |
|---------|------|------|---------|--------------------|
| Infrastruktur | 1 (untagged) | `10.10.1.0/24` | `10.10.1.1` | UDM, Switches, Access Points — alles, womit man das Netz selbst verwaltet |
| Server | 10 | `10.10.10.0/24` | `10.10.10.1` | `dns01`, spaeter Proxmox und dessen VMs |
| Clients | 20 | `10.10.20.0/24` | `10.10.20.1` | Laptops, Telefone, Arbeitsrechner, Konsolen |
| Kids | 25 | `10.10.25.0/24` | `10.10.25.1` | Kindergeraete, eigene Filtergruppe und Zeitfenster |
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
liegt im Server-Segment. Es aenderten sich Maske und Gateway, keine einzige Adresse — womit
alle bestehenden Notizen weiter stimmen.

## Was sich an dns01 geaendert hat

In `/etc/network/interfaces` ([Statische IP mit ifupdown]({{< relref "/docs/linux/static-ip" >}})):

| Zeile | vorher | nachher |
|-------|--------|---------|
| `address` | `10.10.10.3/16` | `10.10.10.3/24` |
| `gateway` | `10.10.1.1` | `10.10.10.1` |

Das Geraet selbst muss nichts von VLANs wissen, solange sein Switch-Port das Server-VLAN
**untagged** fuehrt. Tagging braucht erst der Proxmox-Host, der mehrere Segmente gleichzeitig
bedienen soll.

## Der Fallstrick: listeningMode

Der Punkt, an dem der Umbau sonst kippt. Pi-hole stand auf `dns.listeningMode = LOCAL` und
beantwortet damit ausschliesslich Anfragen aus Netzen, in denen der Rechner selbst eine
Adresse hat. Vorher war das dank `/16` das gesamte Netz — nach dem Schnitt nur noch
`10.10.10.0/24`.

Die Folge waere: Clients aus VLAN 20, 25, 30 und 40 stellen ihre Anfragen, und Pi-hole verwirft
sie kommentarlos. Kein Fehler im Log, kein Hinweis, nur ein Netz ohne Namensaufloesung.

```sh
sudo pihole-FTL --config dns.listeningMode ALL
sudo systemctl restart pihole-FTL
```

Damit beantwortet FTL auch geroutete Anfragen aus den anderen Segmenten. Die Warnung aus
[Pi-hole als DNS-Server]({{< relref "/docs/linux/pihole" >}}) gilt dabei unveraendert: `ALL`
macht aus dem Dienst einen offenen Resolver, sobald er von aussen erreichbar ist. Hinter einem
Gateway ohne Portfreigabe auf 53 ist das unkritisch — die Verantwortung wandert damit
allerdings von Pi-hole in die Firewall.

## Was zwischen den Segmenten erlaubt ist

Die Grundregel ist Verbot: Segmente duerfen ins Internet, aber nicht untereinander. Die
wichtigste Ausnahme ist Port 53 auf `10.10.10.3` aus allen Segmenten — ohne sie steht der
Haushalt nach dem Umbau ohne Namensaufloesung da.

Das vollstaendige Regelwerk samt Zonen, Reihenfolge und dem mDNS-Repeater steht unter
[Firewall zwischen Segmenten]({{< relref "/docs/network/firewall-policies" >}}).

## Pruefen

Von je einem Geraet aus jedem Segment:

```sh
ip -br a                                     # liegt die Adresse im richtigen Netz?
ping -c1 10.10.20.1                          # eigenes Gateway erreichbar
ping -c1 9.9.9.9                             # Internet per IP
dig +short @10.10.10.3 example.com           # DNS ueber die Segmentgrenze
dig +short @10.10.10.3 dns01.xlab.internal   # lokaler Name loest auf
ping -c1 10.10.10.3                          # sollte aus IoT und Gaesten fehlschlagen
```

Auf `dns01` mitlesen, ob die Anfragen mit ihrer echten Client-Adresse ankommen:

```sh
pihole -t
```

Zwischen den Segmenten wird geroutet, nicht genattet — im Log stehen weiterhin die einzelnen
Geraete und nicht die Gateway-Adresse. Bleibt der Filter stumm, sind entweder eine
Firewall-Regel oder der `listeningMode` die Ursache.

## Was offen bleibt

**Die Geraete selbst.** Das Geruest steht, bezogen ist es noch nicht: Ein Grossteil der
IoT-Hardware haengt weiterhin dort, wo sie vor dem Umbau war. Jedes Geraet ist einmal von Hand
in die neue SSID zu bringen — der Teil, der laenger dauert als die gesamte Netzkonfiguration
davor.

**Der Proxmox-Uplink.** Der Port fuer den Hypervisor bekommt ein eigenes Profil: Server-VLAN
untagged fuer das Management, Client- und IoT-VLAN getagged fuer VMs. Der Host selbst liegt
derzeit noch im Infrastruktur-Segment und muss ins Server-VLAN umziehen.

**Der zweite Pi-hole.** Ein Segment-Layout aendert nichts daran, dass ein einzelner Thin Client
die Namensaufloesung des ganzen Hauses traegt. Der zweite Resolver ist der erste sinnvolle Gast
auf dem Hypervisor.

**Das Lab-VLAN.** Reserviert als VLAN 50, angelegt wird es, wenn es etwas zu isolieren gibt.

**IPv6.** Ungeklaert, und mit Segmenten wird die Frage groesser statt kleiner: Jedes Netz
bekaeme ein eigenes Praefix, und die Nameserver-Ankuendigung per Router Advertisement muss
ueberall auf Pi-hole zeigen, sonst gilt der Filter je Segment nur zur Haelfte.
