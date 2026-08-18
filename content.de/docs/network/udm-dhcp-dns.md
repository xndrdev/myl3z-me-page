---
title: Pi-hole per DHCP verteilen (UDM)
weight: 10
---

# Pi-hole per DHCP verteilen

Ein DNS-Server, den niemand fragt, filtert nichts. Nach der Installation von
[Pi-hole]({{< relref "/docs/linux/pihole" >}}) laeuft der Dienst zwar, aber jedes Geraet im
Haushalt bekommt beim Verbinden weiterhin die Adresse des Gateways als Nameserver genannt und
haelt sich daran. Der Filter steht daneben und sieht zu.

Die Stelle, an der das entschieden wird, ist ein einziges Feld im DHCP-Server des Gateways —
bei einer UniFi Dream Machine also in der Netzwerk-Konfiguration, nicht auf dem Server, auf
dem Pi-hole liegt.

## Was DHCP dabei tut

Wenn ein Geraet ins Netz kommt, fragt es per Broadcast nach einer Konfiguration. Der
DHCP-Server antwortet mit einem Paket, das mehr enthaelt als nur eine Adresse: Maske,
Gateway, Lease-Dauer — und in **Option 6** eine Liste von Nameservern. Genau diese Liste
traegt der Client anschliessend bei sich ein.

Das ist der Hebel und zugleich seine Grenze: Option 6 ist ein Vorschlag. Ein Geraet, das
seinen DNS fest eingetragen hat, ignoriert das Feld vollstaendig.

## Der Eintrag in der UDM

*Settings → Networks →* das betreffende Netz *→ DHCP Name Server*. Der Wert steht
standardmaessig auf **Auto**; umgestellt auf **Manual** erscheinen die Felder fuer die Server.

| Einstellung | Wirkung |
|-------------|---------|
| `Auto` | Die UDM traegt sich selbst als Nameserver ein und loest stellvertretend auf. Pi-hole sieht nichts |
| `Manual` → `10.10.10.3` | Die Clients bekommen die Adresse von `dns01` genannt und fragen dort direkt an |

> [!NOTE]
> Die Benennung wandert zwischen den Versionen von UniFi Network. Je nach Stand heisst der
> Abschnitt *DHCP Name Server*, *DNS Server* oder liegt hinter *Advanced → Manual*. Gesucht ist
> immer das Feld, das im DHCP-Angebot landet — nicht der Resolver, den die UDM fuer sich selbst
> benutzt.

Eingetragen ist hier genau eine Adresse:

```text
10.10.10.3
```

## Warum kein zweiter Eintrag

Das Feld nimmt mehrere Server auf, und der Reflex ist, einen oeffentlichen Resolver als
Absicherung danebenzustellen. Das ist an dieser Stelle falsch, aus demselben Grund, der schon
unter [Zwei Nameserver als Absicherung]({{< relref "/docs/linux/static-ip#zwei-nameserver-als-absicherung" >}})
steht: Der Client arbeitet die Liste nicht als Rangfolge mit Ausfallschutz ab, sondern nimmt
den zweiten Server, sobald der erste einmal nicht rechtzeitig antwortet. Ab da laufen Anfragen
ungefiltert und ohne Protokoll am Filter vorbei — ohne dass irgendetwas kaputt aussieht.

Ein zweiter Eintrag ist erst dann richtig, wenn dahinter ein *zweiter Pi-hole* steht, der
dieselben Listen und dieselben lokalen Namen kennt.

Der Preis der sauberen Variante steht im selben Absatz: Der Thin Client ist damit ein Single
Point of Failure fuer die Namensaufloesung im ganzen Haushalt. Faellt er aus, faellt das Netz
gefuehlt komplett aus.

## Wann die Clients es uebernehmen

Nicht sofort. Ein Geraet holt sich die Konfiguration erst beim naechsten Renew — ueblicherweise
nach der Haelfte der Lease-Dauer, bei UniFi also nach rund zwoelf Stunden bei den
voreingestellten 86400 Sekunden. Wer nicht warten will, erzwingt es:

| System | Befehl |
|--------|--------|
| Linux (dhclient) | `sudo dhclient -r && sudo dhclient` |
| Linux (NetworkManager) | `nmcli con down <name> && nmcli con up <name>` |
| Windows | `ipconfig /release && ipconfig /renew` |
| macOS | *Systemeinstellungen → Netzwerk → Details → TCP/IP → DHCP-Lease erneuern* |
| alles andere | WLAN trennen und neu verbinden |

> [!NOTE]
> Wer die Umstellung schnell durch den Haushalt bringen will, setzt die Lease-Dauer vorher
> kurzzeitig herunter. Danach wieder hochsetzen — eine dauerhaft kurze Lease erzeugt nur
> unnoetigen Verkehr.

## Pruefen

Der ehrliche Test kommt von einem beliebigen Geraet im Netz, nicht von der UDM:

```sh
resolvectl status | grep 'DNS Server'   # systemd-resolved
cat /etc/resolv.conf                    # ifupdown, ohne systemd-resolved
scutil --dns | grep nameserver          # macOS
ipconfig /all                           # Windows
```

Erwartet wird `10.10.10.3` und sonst nichts.

Der zweite Test ist der aussagekraeftigere, weil er zeigt, dass die Anfragen auch wirklich
ankommen. Auf `dns01`:

```sh
pihole -t
```

Vorher tauchte dort im Wesentlichen der Server selbst auf. Danach laufen die Adressen der
Geraete aus dem Haushalt durch. Dasselbe zeigt das Webinterface unter *Settings → Clients*: die
Zahl der bekannten Clients ist der Beweis, dass der Dienst benutzt wird und nicht nur laeuft.

## Was DHCP nicht erledigt

**Geraete mit fest eingetragenem DNS.** Fernsehgeraete, Konsolen und manche IoT-Hardware
bringen `8.8.8.8` ab Werk mit. Sie fragen nie nach, also hilft kein DHCP-Feld. Dagegen wirkt
nur eine Firewall-Regel, die Port 53 nach draussen fuer alles ausser `10.10.10.3` sperrt — oder
die Anfragen per NAT auf Pi-hole umbiegt.

**DNS over TLS und DNS over HTTPS.** Android bietet unter *Privates DNS* die Eingabe eines
festen Anbieters an, Browser bringen DoH teils selbst mit. DoT laesst sich am Port 853 sperren,
DoH nicht — es ist von normalem HTTPS nicht zu unterscheiden und laesst sich nur ueber eine
Blockliste bekannter Endpunkte in Pi-hole selbst eindaemmen.

> [!WARNING]
> Port 853 zu sperren bringt Android nur dann zum Rueckfall auf Klartext-DNS, wenn *Privates
> DNS* auf **Automatisch** steht. Ist dort ein Anbieter fest eingetragen, gibt es keinen
> Rueckfall, sondern einen Verbindungsfehler ohne erkennbare Ursache.

**IPv6.** Verteilt die UDM IPv6 per Router Advertisement, nennt sie sich darin selbst als
Nameserver (RDNSS) — und viele Clients bevorzugen den. Der Filter laeuft dann, greift aber nur
noch bei einem Teil der Anfragen. Ob das hier zutrifft, ist noch nicht geprueft:

```sh
ip -6 addr show          # hat der Client ueberhaupt eine globale IPv6-Adresse?
resolvectl status        # steht dort ein fe80::-Nameserver neben 10.10.10.3?
```

Steht dort einer, gibt es zwei saubere Wege: Pi-hole eine feste IPv6-Adresse geben und diese
ebenfalls verteilen, oder IPv6 im LAN abschalten, solange es nicht gebraucht wird. Der halbe
Zustand ist der schlechteste.

## Was danach noch offen ist

**Jedes weitere Netz braucht den Eintrag erneut.** Das Feld haengt am Netzwerk, nicht am
Geraet. Sobald das Netz in Segmente geschnitten wird
([Netz-Layout mit VLANs]({{< relref "/docs/network/vlan-layout" >}})), startet jedes neue VLAN
wieder mit `Auto` — und faellt damit still aus dem Filter.

**Pi-hole hoert nur lokal.** `dns.listeningMode` steht auf `LOCAL` und beantwortet damit nur
Anfragen aus Netzen, in denen der Rechner selbst eine Adresse hat. Solange alles in einem
flachen Netz liegt, faellt das nicht auf. Mit getrennten Subnetzen schon.

**Das eigene Adblocking der UDM.** Bringt die Firmware einen eigenen Filter mit und ist er
aktiv, filtern zwei Systeme uebereinander. Das kostet keine Funktion, macht aber jede spaetere
Fehlersuche doppelt so teuer — ein Ausschalten ist die guenstigere Entscheidung.
