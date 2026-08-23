---
title: WLAN auf VLANs verteilen
weight: 50
---

# WLAN auf VLANs verteilen

Am Kabel entscheidet der Switch-Port, in welchem Segment ein Geraet landet. Im WLAN gibt es
keinen Port — dort entscheidet die SSID. Jedes Funknetz wird genau einem VLAN zugeordnet, und
womit sich ein Geraet verbindet, bestimmt, wo es liegt.

## Eine SSID je Segment

| SSID | Netz | Besonderheit |
|------|------|--------------|
| `xlab` | Clients (20) | WPA3/WPA2 gemischt, beide Baender |
| `xlab-IoT` | IoT (30) | nur 2,4 GHz, WPA2, PMF aus |
| `xlab-Kids` | Kids (25) | wie Clients, zusaetzlich Zeitfenster |
| `xlab-Guests` | Gaeste (40) | Client Device Isolation |

Die Zuordnung steht in UniFi unter *Settings → WiFi →* SSID *→ Network*. Das Feld heisst je
nach Version *Network* oder *VLAN*; gemeint ist immer dasselbe.

Der Preis dieser Loesung ist Funkzeit: Jede SSID sendet ihre eigenen Beacons, und vier
Funknetze auf demselben Kanal kosten messbar Kapazitaet. Die Alternative waere eine einzige
SSID mit dynamischer VLAN-Zuweisung per RADIUS — sauberer im Funk, aber mit einem
Authentifizierungsserver als zusaetzlichem Dienst, der ausfallen kann. Bei vier Segmenten in
einem Haushalt ist der Aufwand nicht gerechtfertigt.

## Warum IoT anders konfiguriert ist

Die drei Einschraenkungen bei `xlab-IoT` sehen nach Nachlaessigkeit aus und sind das Gegenteil:

**Nur 2,4 GHz.** Viele Smart-Home-Geraete beherrschen ausschliesslich dieses Band. Steht die
SSID auf beiden Baendern, kuendigt der Access Point sie auch auf 5 GHz an — und ein Teil der
Geraete scheitert dann schon beim Verbindungsaufbau, weil Band Steering sie dorthin zu
schieben versucht. Die SSID auf 2,4 GHz zu beschraenken, nimmt die Frage aus dem Spiel.

**WPA2 statt WPA3.** Der ESP32 und aehnliche Funkmodule, die in einem Grossteil der guenstigen
Geraete stecken, koennen WPA3 nicht. Im gemischten Modus scheitern sie oft schon an der
Aushandlung.

**PMF ausgeschaltet.** Protected Management Frames sind bei WPA3 Pflicht und bei WPA2
optional. Aeltere Firmware bricht die Verbindung ab, sobald der Access Point sie verlangt —
oder verbindet sich und verliert die Verbindung nach Minuten wieder. Das ist der Fehler, der
sich am unangenehmsten sucht, weil er nicht sofort auftritt.

> [!NOTE]
> Diese drei Punkte senken das Sicherheitsniveau dieser einen SSID bewusst. Genau deshalb liegt
> sie in einem eigenen Segment: Was dort einbricht, findet ein Netz mit Fernseher und Drucker
> vor und keinen Arbeitsrechner. Die schwaechere Verschluesselung ist tragbar, weil die
> Segmentgrenze dahinter steht — nicht, weil sie egal waere.

## Gaeste

*Client Device Isolation* sorgt dafuer, dass Geraete im Gaestenetz einander nicht sehen, auch
nicht innerhalb desselben Segments. Ohne diese Einstellung sind zwei Besucher im selben WLAN
weiterhin direkte Nachbarn — die Firewall sieht diesen Verkehr nie, weil er den Access Point
gar nicht verlaesst.

Das ist der Unterschied zwischen Isolation auf Layer 2 und einer Firewall-Regel, und der Grund,
warum die Regel *Gaeste duerfen nur ins Internet* allein nicht reicht.

## Kids

Das Segment existiert, damit Kindergeraete nicht im selben Netz liegen wie Arbeitsrechner —
und weil sich Regeln, die an ein Netz gebunden sind, leichter pflegen lassen als Regeln pro
Geraet. Zwei Dinge haengen daran:

- **eine eigene Pi-hole-Gruppe**, damit fuer diese Clients strengere Blocklisten gelten
  koennen, ohne den Rest des Haushalts zu betreffen
- **ein Zeitfenster**, entweder als *Blackout Schedule* an der SSID oder als *Traffic Rule* mit
  Zeitplan

Der Blackout Schedule schaltet die SSID zu festen Zeiten ab. Das ist wirksam und grob: Ein
Geraet mit gespeichertem Zugang zu `xlab` wechselt einfach dorthin. Die Trennung ueber ein
eigenes Netz funktioniert nur so gut wie die Passwortdisziplin bei den anderen SSIDs.

## Der Access Point braucht die VLANs

Ein Access Point haengt an einem Kabel und bedient vier Segmente. Das geht nur getaggt: Sein
Switch-Port fuehrt das Infrastruktur-VLAN untagged fuer das Geraet selbst und alle
SSID-VLANs getagged.

Fehlt eines davon in der Tagged-Liste, verbinden sich Geraete mit der SSID und bekommen keine
Adresse — ein Fehlerbild, das nach DHCP aussieht und keines ist. Deshalb steht der AP-Uplink
im Port-Profil auf *Allow All*: Jede kuenftige SSID funktioniert dann ohne Portaenderung. Die
Profile stehen unter
[VLANs auf der UDM einrichten]({{< relref "/docs/network/unifi-vlan-umbau#port-profile" >}}).

## Umbenennen kostet einen Durchlauf

Die alte SSID wurde beim Umbau durch die vier neuen ersetzt. Eine SSID ist fuer Endgeraete die
Identitaet des Netzes — mit dem neuen Namen ist jedes Geraet im Haushalt einmal von Hand neu zu
verbinden.

Bei Telefonen und Laptops ist das eine Minute. Unangenehm wird es bei Geraeten ohne
Bedienoberflaeche, die nur ueber eine Hersteller-App eingerichtet werden koennen, und bei
solchen, die dafuer in einen Kopplungsmodus muessen. Wer den Umbau plant, rechnet diesen Teil
grosszuegig: Er dauert laenger als die gesamte Netzkonfiguration davor.

## Pruefen

Vom Telefon aus, je SSID einmal:

```sh
ip -br a                             # liegt die Adresse im erwarteten Segment?
dig +short @10.10.10.3 example.com   # DNS ueber die Segmentgrenze
```

Am schnellsten sieht man es an der dritten Stelle der Adresse: `10.10.30.x` heisst IoT,
`10.10.20.x` heisst Clients. Genau dafuer ist das Schema so gewaehlt.

In der UniFi-Oberflaeche zeigt *Client Devices* zu jedem verbundenen Geraet Netz und SSID. Wer
dort nach dem Umbau noch Geraete im Infrastruktur-Netz findet, hat eine SSID uebersehen — oder
ein Geraet, das sich noch mit dem alten Funknetz verbunden hat.
