---
title: VLANs verstehen
weight: 15
---

# VLANs verstehen

Ein Switch leitet Frames anhand von MAC-Adressen weiter und kennt keine Netze. Alles, was an
ihm haengt, liegt in derselben Broadcast-Domain: Ein ARP-Request erreicht jeden Port, jedes
Geraet sieht jedes andere. Genau das soll die Segmentierung beenden — und ein VLAN ist das
Mittel, mit dem ein einzelner Switch so tut, als waere er mehrere.

## Das Tag

Die Grundlage ist IEEE 802.1Q: vier zusaetzliche Bytes im Ethernet-Frame, direkt hinter der
Absender-MAC. Darin stehen unter anderem zwoelf Bit VLAN-ID, also 4094 nutzbare Nummern —
`0` und `4095` sind reserviert.

Mehr passiert nicht. Ein VLAN ist kein Tunnel und keine Verschluesselung, sondern eine Zahl im
Frame, an die sich alle beteiligten Geraete halten. Wer den Switch umgeht und direkt an der
Leitung mitliest, sieht den Verkehr unveraendert. Die Trennung ist eine organisatorische, die
der Switch durchsetzt — keine kryptografische.

## Untagged und tagged

Der Unterschied, an dem sich in der Praxis alles entscheidet:

| Port-Typ | Was ueber die Leitung geht | Wofuer |
|----------|---------------------------|--------|
| **untagged** (Access) | Frames ohne Tag. Der Switch fuegt es beim Eintritt hinzu und entfernt es beim Austritt | Endgeraete: Laptop, Drucker, Fernseher, `dns01` |
| **tagged** (Trunk) | Frames mit Tag, mehrere VLANs parallel ueber eine Leitung | Verbindungen zwischen Switches, Access Points, Hypervisor |

Ein Endgeraet an einem untagged Port merkt von VLANs nichts. Es bekommt eine Adresse per DHCP,
sieht ein Gateway und haelt sich fuer das einzige Netz der Welt. Deshalb musste an `dns01` beim
Umbau auch nichts VLAN-spezifisches konfiguriert werden — nur Maske und Gateway.

Tagging braucht erst, wer **mehrere** Segmente ueber **eine** Leitung fuehren muss. Bei einem
Access Point, der vier SSIDs in vier VLANs bedient, ist das zwingend: Es gibt nur ein Kabel.

## Native VLAN und PVID

Auf einem Trunk kann trotzdem ungetaggter Verkehr ankommen. Was damit geschieht, regelt das
**Native VLAN** — bei UniFi heisst das Feld *Native VLAN / Network*, in anderen Oberflaechen
*PVID* (Port VLAN ID). Alles, was ohne Tag hereinkommt, wird dieser VLAN-ID zugeordnet.

Das ist der Grund, warum ein Port gleichzeitig untagged **und** tagged sein kann, und genau die
Konstruktion, die ein Hypervisor braucht:

```text
Native VLAN:  Server (10)      -> Management des Hosts, ungetaggt
Tagged VLANs: Clients (20)     -> VMs im Client-Segment
              IoT (30)         -> VMs im IoT-Segment
```

Der Host selbst redet ungetaggt und landet im Server-VLAN. Seine VMs bekommen im Hypervisor
ein Tag mitgegeben und landen dort, wo dieses Tag hinzeigt. Ein Kabel, drei Segmente.

> [!NOTE]
> Das Native VLAN ist die haeufigste Fehlerquelle beim ersten Trunk. Stimmt es nicht mit dem
> ueberein, was das Geraet am anderen Ende erwartet, funktioniert der getaggte Verkehr
> tadellos und der ungetaggte verschwindet — was sich anfuehlt wie ein halb kaputtes Kabel.

## VLAN ist nicht Subnetz

Beides wird im Alltag gleichgesetzt, und im Layout hier entspricht tatsaechlich jedes VLAN
genau einem `/24`. Technisch sind es aber zwei verschiedene Schichten:

- Das **VLAN** ist Layer 2. Es entscheidet, wer wen ueberhaupt direkt sehen kann.
- Das **Subnetz** ist Layer 3. Es entscheidet, wann ein Geraet ueber das Gateway geht statt
  direkt zu senden.

Man kann zwei Subnetze in ein VLAN legen oder ein Subnetz ueber mehrere VLANs spannen. Beides
erzeugt vor allem Verwirrung. Die Eins-zu-eins-Zuordnung ist die einzige, die man ein Jahr
spaeter noch versteht.

Wichtiger ist die Konsequenz daraus: **Ein VLAN allein trennt nichts, was die Firewall nicht
trennt.** Sobald die Segmente geroutet werden — und das tun sie, sonst kaeme niemand ins
Internet — liegt die eigentliche Grenze in den Regeln auf dem Gateway. Das VLAN sorgt nur
dafuer, dass der Verkehr ueberhaupt am Gateway vorbeikommt und dort bewertet werden kann.
Ohne den Umweg ueber das Gateway greift keine Regel, weil kein Paket sie je zu sehen bekommt.

## Warum die Maske dazugehoert

Der Punkt, an dem der Umbau hier fast gescheitert waere. Solange alle Geraete `10.10.0.0/16`
konfiguriert haben, halten sie einander fuer Nachbarn im selben Netz. Ein Geraet mit
`10.10.20.5/16` sendet an `10.10.30.7` **direkt** per ARP, statt das Gateway zu fragen.

Steckt zwischen beiden ein VLAN, kommt der ARP-Request nicht an, und die Verbindung
scheitert — aber sie scheitert als Timeout, nicht als saubere Ablehnung. Wird das VLAN
hingegen nicht sauber getrennt, klappt die Verbindung an jeder Firewall-Regel vorbei.

Beide Faelle sind unangenehm. Deshalb gehoeren VLAN-Schnitt und Maskenwechsel zusammen: Erst
mit `/24` fragt ein Geraet fuer alles ausserhalb seines Segments das Gateway — und erst dann
gibt es eine Stelle, an der Regeln greifen.

## Sicherheitsgrenzen

Zwei Angriffe sind es wert, sie zu kennen, auch wenn beide im Heimnetz selten sind:

**Switch Spoofing.** Ein Geraet gibt sich als Switch aus und handelt einen Trunk aus, um alle
VLANs zu sehen. Betrifft Hersteller mit automatischer Trunk-Aushandlung; UniFi handelt Trunks
nicht automatisch aus, jeder Port bekommt sein Profil explizit zugewiesen.

**Double Tagging.** Ein Frame bekommt zwei Tags mit; der erste Switch entfernt das aeussere,
der zweite leitet nach dem inneren weiter. Funktioniert nur, wenn das Native VLAN eines Trunks
mit dem VLAN des Angreifers uebereinstimmt. Wer das Native VLAN eines Trunks auf ein VLAN legt,
in dem keine Endgeraete haengen, nimmt dem Angriff die Grundlage.

Fuer ein Heimnetz ist die praktisch relevantere Aussage die einfachere: Ein VLAN schuetzt
gegen ein Geraet, das sich falsch verhaelt, nicht gegen jemanden mit physischem Zugang zum
Switch.

## Wo es hier steht

Das konkrete Layout mit sechs Segmenten steht unter
[Netz-Layout mit VLANs]({{< relref "/docs/network/vlan-layout" >}}), der Ablauf des Umbaus samt
der Reihenfolge, in der es funktioniert, unter
[VLANs auf der UDM einrichten]({{< relref "/docs/network/unifi-vlan-umbau" >}}).
