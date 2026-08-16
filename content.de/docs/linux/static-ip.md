---
title: Statische IP mit ifupdown
weight: 30
---

# Statische IP mit ifupdown

Ein Server, den andere Geraete ansprechen sollen, braucht eine Adresse, die sich nicht
aendert. Per DHCP bekommt er irgendeine — nach einem Neustart des Routers moeglicherweise eine
andere, und jede Konfiguration, die auf die alte zeigt, laeuft ins Leere. Besonders bei einem
DNS-Server ist das keine Option: dessen Adresse tragen alle Clients fest ein.

Eine Debian-Installation ohne Desktop verwaltet das Netzwerk mit **ifupdown** ueber
`/etc/network/interfaces`. NetworkManager und `systemd-networkd` loesen dieselbe Aufgabe, sind
auf einem Netinst-System aber nicht im Spiel.

## Interface-Namen ermitteln

```sh
ip -br link
```

```text
lo               UNKNOWN        00:00:00:00:00:00 <LOOPBACK,UP,LOWER_UP>
enp1s0           UP             a4:bb:6d:12:34:56 <BROADCAST,MULTICAST,UP,LOWER_UP>
```

`enp1s0` ist kein Zufallsname: `en` fuer Ethernet, `p1` fuer PCI-Bus 1, `s0` fuer Slot 0. Diese
*predictable interface names* bleiben ueber Neustarts und Kernel-Updates stabil — anders als
das fruehere `eth0`, dessen Nummerierung von der Reihenfolge der Geraeteerkennung abhing.

## Die Konfiguration

Der Block fuer eine feste Adresse in `/etc/network/interfaces`:

```text
auto enp1s0
iface enp1s0 inet static
    address 10.10.10.3/16
    gateway 10.10.1.1
    dns-nameservers 10.10.1.1 1.1.1.1
```

| Zeile | Bedeutung |
|-------|-----------|
| `auto enp1s0` | Interface beim Booten hochfahren. Fehlt die Zeile, existiert die Konfiguration, wird aber nur bei manuellem `ifup` angewandt |
| `iface enp1s0 inet static` | IPv4 (`inet`), Adresse fest vergeben statt per DHCP. Fuer IPv6 waere es `inet6` |
| `address 10.10.10.3/16` | Adresse in CIDR-Notation. `/16` bedeutet Netz `10.10.0.0` mit Hosts von `10.10.0.1` bis `10.10.255.254`. Aeltere Anleitungen schreiben stattdessen `address` und `netmask 255.255.0.0` — gleichwertig, aber unnoetig umstaendlich |
| `gateway 10.10.1.1` | Default-Route fuer alles, was nicht im eigenen Netz liegt. Muss innerhalb des durch die Maske aufgespannten Netzes liegen |
| `dns-nameservers 10.10.1.1 1.1.1.1` | Nameserver fuer die Aufloesung, durch Leerzeichen getrennt. Der zweite ist der Ausweichweg, falls der erste nicht antwortet |

> [!WARNING]
> Die feste Adresse muss ausserhalb des DHCP-Bereichs des Routers liegen. Sonst vergibt der
> Router dieselbe Adresse irgendwann an ein anderes Geraet, und beide sind gestoert.

## Zwei Nameserver als Absicherung

Ein Server ohne funktionierende Namensaufloesung kann keine Pakete aktualisieren, keine Zeit
synchronisieren und nichts nach aussen melden. Ein zweiter Eintrag faengt den Ausfall des
ersten ab — hier der Router als lokaler Resolver, dahinter ein oeffentlicher.

Wie die Liste abgearbeitet wird, entscheidet der Resolver der C-Bibliothek, nicht das Netz:

- **Strikt der Reihe nach.** Der erste Eintrag wird immer zuerst gefragt. Es gibt weder
  Lastverteilung noch eine Bevorzugung des schnelleren Servers.
- **Umschalten kostet Zeit.** Antwortet der erste nicht, laeuft erst der Timeout ab
  (standardmaessig 5 Sekunden), danach ist der zweite dran. Solange der erste tot ist,
  faellt diese Wartezeit bei *jeder* Anfrage an.
- **Nur Ausfaelle werden umgangen.** Ein Timeout oder ein `SERVFAIL` fuehrt zum naechsten
  Server. Ein `NXDOMAIN` ist dagegen eine gueltige Antwort und wird durchgereicht — ein
  Nameserver, der falsch antwortet, wird nicht uebersprungen.
- **Maximal drei Eintraege.** Alles darueber hinaus ignoriert die glibc.

Das Verhalten laesst sich anpassen, wenn die Wartezeit stoert:

```text
options timeout:2 attempts:1
```

> [!WARNING]
> Sobald ein eigener DNS-Server im Netz laeuft — etwa Pi-hole —, wird der oeffentliche
> Zweiteintrag zum Loch im Filter: Anfragen gehen daran vorbei, ungefiltert und ohne
> Protokoll. Und zeigt der Eintrag auf einen Resolver, der seinerseits an den eigenen
> DNS-Server weiterleitet, entsteht eine Schleife. Auf dem DNS-Server selbst gehoert deshalb
> `127.0.0.1` in die Liste, und der oeffentliche Resolver wird *in* dessen Konfiguration als
> Upstream eingetragen, nicht daneben.

## Uebernehmen

```sh
systemctl reboot
```

Ein Neustart ist der ehrliche Test: er beweist, dass die Konfiguration den Bootvorgang
ueberlebt und nicht nur im laufenden System gesetzt ist. Ohne Neustart geht es auch:

```sh
sudo ifdown enp1s0 && sudo ifup enp1s0
```

> [!NOTE]
> Beides trennt eine bestehende SSH-Sitzung — die Adresse aendert sich ja gerade. Bei einem
> Geraet ohne Bildschirm die neue Adresse vorher kennen, sonst hilft nur noch ein Monitor.

## Pruefen

```sh
ip -br a               # zugewiesene Adresse
ip route               # default via 10.10.1.1 dev enp1s0
ping -c1 10.10.1.1     # Gateway erreichbar
cat /etc/resolv.conf   # welcher Nameserver tatsaechlich verwendet wird
```

## Fallstrick: dns-nameservers ohne resolvconf

`ifupdown` schreibt die Zeile `dns-nameservers` nicht selbst nach `/etc/resolv.conf`. Das
erledigt ein Hook-Script unter `/etc/network/if-up.d/`, das erst mit dem Paket `resolvconf`
(alternativ `openresolv`) dazukommt. Fehlt es, wird die Zeile **stillschweigend ignoriert** —
kein Fehler, keine Warnung. Uebrig bleibt der Nameserver, den der Installer eingetragen hat.

```sh
dpkg -l resolvconf openresolv 2>/dev/null | grep '^ii'
```

Kommt nichts zurueck, gibt es zwei Wege:

{{< tabs >}}

{{% tab "resolvconf nachinstallieren" %}}
```sh
sudo apt install resolvconf
sudo ifdown enp1s0 && sudo ifup enp1s0
```
Die Konfiguration bleibt an einer Stelle — `/etc/network/interfaces` ist die Quelle.
{{% /tab %}}

{{% tab "resolv.conf direkt pflegen" %}}
```sh
printf 'nameserver 10.10.1.1\nnameserver 1.1.1.1\n' | sudo tee /etc/resolv.conf
```
Ein Paket weniger, dafuer zwei Dateien, die zusammenpassen muessen. Die Zeile
`dns-nameservers` ist dann nur noch Dokumentation — wer sie spaeter aendert und die
`resolv.conf` vergisst, sucht den Fehler an der falschen Stelle.
{{% /tab %}}

{{< /tabs >}}

Umgekehrt gilt: ist `resolvconf` installiert, wird `/etc/resolv.conf` generiert und
Handaenderungen daran verschwinden beim naechsten `ifup`. Welcher Fall vorliegt, verraet der
Kopf der Datei — eine generierte traegt einen `DO NOT EDIT`-Hinweis:

```sh
head -3 /etc/resolv.conf
```
