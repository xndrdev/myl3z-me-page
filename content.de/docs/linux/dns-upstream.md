---
title: Upstream-DNS waehlen
weight: 80
---

# Upstream-DNS waehlen

[Pi-hole]({{< relref "/docs/linux/pihole" >}}) kennt die Adressen der Welt nicht. Es kennt
Blocklisten und einen Cache — alles andere reicht es weiter an einen **Upstream**, und dessen
Antwort gibt es an den Client zurueck. Damit ist der Upstream die Stelle, an der die
Namensaufloesung des ganzen Haushalts das eigene Netz verlaesst.

Der Installer stellt die Frage einmal, mit Google als erstem Vorschlag, und wer bestaetigt,
hat eine Entscheidung getroffen, ohne sie zu treffen.

## Was der Upstream sieht

Vor dem Filter fragte jedes Geraet einzeln beim Resolver des Anbieters an. Danach fragt genau
eine Adresse — der Thin Client — und zwar alles, was nicht auf einer Blockliste steht. Die
Menge der Anfragen sinkt dadurch deutlich, ihr Inhalt bleibt derselbe: welche Dienste im
Haushalt benutzt werden, wann jemand aufsteht, wann niemand da ist.

Der Filter aendert also, *wie viele* Anfragen nach draussen gehen, nicht *ob* jemand sie sieht.
Wer Pi-hole aus Datenschutzgruenden aufgesetzt hat und Google als Upstream stehen laesst, hat
den Adressaten nicht gewechselt, sondern nur die Zustellung gebuendelt.

## Die Anbieter

| Upstream | Adressen | Lage |
|----------|----------|------|
| **Quad9** | `9.9.9.9`, `149.112.112.112` | Schweizer Stiftung ohne Werbegeschaeft, speichert keine Client-Adressen, validiert DNSSEC selbst und blockt bekannte Malware- und Phishing-Domains. Sendet standardmaessig kein ECS |
| Cloudflare | `1.1.1.1`, `1.0.0.1` | In Messungen meist der schnellste, mit externem Audit der Datenschutzzusagen. Ein US-Konzern, dessen Kerngeschaeft CDN ist und nicht DNS |
| Google | `8.8.8.8`, `8.8.4.4` | Sehr schnell, sehr zuverlaessig, ueberall dokumentiert — und betrieben von dem Unternehmen, dessen Geschaeftsmodell auf Werbung beruht. Die Voreinstellung des Installers |
| Der eigene Anbieter | vom Router genannt | Sieht den Verkehr ohnehin. Dafuer leiten manche Anbieter nicht existierende Namen auf eigene Suchseiten um, statt `NXDOMAIN` zu antworten |
| Eigener rekursiver Resolver | `127.0.0.1#5335` | `unbound` fragt selbst bei den Root-Servern an. Kein Anbieter bekommt mehr das vollstaendige Bild — dafuer eine eigene Software mehr auf der Maschine |

Gewaehlt ist Quad9. Der Malware-Filter ergaenzt Pi-hole, statt es zu doppeln: Pi-hole filtert
Werbung und Tracking, Quad9 filtert Schadsoftware. Der Preis steht daneben — ein zweiter
Filter, den man nicht selbst pflegt, kann falsch liegen. Wer eine Domain verdaechtigt, prueft
gegen `9.9.9.10`, die ungefilterte Variante desselben Dienstes.

> [!NOTE]
> **ECS** (EDNS Client Subnet) reicht einen Teil der Client-Adresse an den Upstream weiter,
> damit dieser einen nahe gelegenen CDN-Knoten nennen kann. Quad9 verzichtet darauf. Im
> Alltag faellt das selten auf; wenn ein grosser Download einmal spuerbar langsamer laeuft
> als gewohnt, ist das die erste Stelle, an die man denkt.

## Warum nicht zwei Anbieter mischen

Das Feld nimmt mehrere Adressen an, und der Reflex ist, Quad9 und Cloudflare nebeneinander zu
stellen. Damit hat man aber nicht das Beste von beiden, sondern von jedem die Haelfte: Ein
Teil der Anfragen geht dorthin, ein Teil dahin, die Zusagen gelten je nur fuer ihren Teil, und
bei einer merkwuerdigen Antwort ist die erste Frage, wer sie ueberhaupt gegeben hat.

Zwei Adressen **desselben** Anbieters sind etwas anderes — das ist Redundanz innerhalb einer
Vertrauensentscheidung, kein Splitten. Deshalb stehen hier `9.9.9.9` und `149.112.112.112`.

## Umstellen

```sh
sudo pihole-FTL --config dns.upstreams '["9.9.9.9","149.112.112.112"]'
```

Im Webinterface liegt dieselbe Einstellung unter *Settings → DNS → Upstream Servers*. Ein
Neustart des Dienstes ist nicht noetig, FTL uebernimmt die Aenderung selbst.

## Pruefen

Der Wert selbst:

```sh
sudo pihole-FTL --config dns.upstreams
```

```text
[ 9.9.9.9, 149.112.112.112 ]
```

Aussagekraeftiger ist die Gegenprobe von einem anderen Rechner aus. `whoami.akamai.net`
antwortet mit der Adresse desjenigen Resolvers, der die Anfrage tatsaechlich gestellt hat —
also der des Upstreams, nicht der eigenen:

```sh
dig +short @10.10.10.3 whoami.akamai.net
```

Die Antwort allein sagt aber noch nichts — man braucht einen Vergleichswert. Den liefert
dieselbe Frage, direkt an den Upstream gestellt:

```sh
dig +short @10.10.10.3 whoami.akamai.net   # ueber Pi-hole
dig +short @9.9.9.9    whoami.akamai.net   # direkt bei Quad9, als Referenz
```

Beide Antworten muessen im selben Netzbereich liegen. Auf dieser Installation:

| Messung | Antwort |
|---------|---------|
| ueber Pi-hole, vorher | `172.253.1.215` — ein Google-Netz |
| ueber Pi-hole, nachher | `193.46.232.196` |
| direkt bei Quad9 | `193.46.232.197` |

Dieselbe `/24`, also derselbe Ausgang. Die genaue Adresse schwankt, weil der Dienst mehrere
davon benutzt.

> [!WARNING]
> Der naheliegende Weg — `whois` auf die Antwort loslassen — fuehrt hier in die Irre. Quad9
> betreibt seine Anycast-Knoten in den Netzen lokaler Anbieter, weshalb dort der Name eines
> beliebigen Providers steht und nirgends "Quad9". Wer das nicht weiss, haelt die Adresse fuer
> den eigenen Anschluss und sucht einen Fehler, den es nicht gibt. Zur Sicherheit hilft die
> eigene oeffentliche Adresse als drittes Vergleichsstueck:
>
> ```sh
> dig +short myip.opendns.com @resolver1.opendns.com
> ```

> [!NOTE]
> Der Cache antwortet schneller als der Upstream und verraet dabei nichts ueber ihn. Kommt
> nach der Umstellung noch das alte Ergebnis, ist es die zwischengespeicherte Antwort —
> `pihole restartdns` leert den Cache.

## DNSSEC: bewusst noch nicht

Pi-hole kann Antworten kryptografisch pruefen (`dns.dnssec`). Jede signierte Zone haengt an
ihre Eintraege eine Signatur, deren Kette bis zur Root-Zone reicht; der Resolver rechnet sie
nach und verwirft, was nicht passt. Das schuetzt davor, dass jemand unterwegs eine Antwort
austauscht und ein Geraet auf einen fremden Server schickt.

Was es **nicht** tut, steht seltener in den Anleitungen:

- **Es verschluesselt nichts.** Signiert heisst nicht vertraulich — wer auf der Strecke
  mitliest, sieht weiterhin jeden angefragten Namen. Das waere DNS over TLS oder over HTTPS,
  eine andere Baustelle.
- **Es wirkt nur bei signierten Zonen.** Ein grosser Teil der taeglich aufgerufenen Domains
  ist nicht signiert, und *unsigniert* ist ein gueltiger Zustand, kein Fehler.
- **Es endet am Resolver.** Zwischen Client und Pi-hole wird nichts geprueft. Das Telefon
  glaubt dem Filter aufs Wort.

Hier kommt hinzu, dass **Quad9 bereits selbst validiert**. Die eigene Pruefung wuerde genau
einen Punkt gewinnen: nicht mehr darauf angewiesen zu sein, dass der Upstream korrekt geprueft
hat und niemand zwischen Gateway und Quad9 sitzt. Dagegen steht ein reales Ausfallrisiko —
abgelaufene Signaturen kommen vor, auch bei grossen Betreibern, und die betroffene Domain
antwortet dann mit `SERVFAIL` statt gar nicht. Im Haushalt sieht das aus wie ein Ausfall von
Pi-hole, und genau dort wird dann gesucht.

Die Entscheidung hier: **aus**, bis der Upstream ein eigener rekursiver Resolver ist. Dann
wird die Pruefung zum eigentlichen Punkt der Uebung, weil die Kette bis zur Root selbst
nachgerechnet wird, statt einem Anbieter zu glauben.

Wer es dennoch anschalten will:

```sh
sudo pihole-FTL --config dns.dnssec true
```

Die Gegenprobe braucht zwei Domains — eine mit gueltiger Signatur, eine mit absichtlich
kaputter:

```sh
dig +dnssec @10.10.10.3 dnssec.works        # NOERROR, Flag "ad" gesetzt
dig @10.10.10.3 fail01.dnssec.works         # SERVFAIL, wenn validiert wird
dig +cd @10.10.10.3 fail01.dnssec.works     # antwortet trotzdem: +cd umgeht die Pruefung
```

Das `ad`-Flag (*authenticated data*) in der ersten Antwort ist der Beweis, dass geprueft
wurde. `+cd` (*checking disabled*) ist im Stoerfall das Werkzeug der Wahl: Kommt eine Antwort
nur damit, ist die Ursache DNSSEC und nicht der Dienst.

> [!NOTE]
> Fuer geblockte Domains antwortet Pi-hole selbst — eine erfundene Antwort, die keine
> gueltige Signatur haben kann. Die Validierung wird dafuer bewusst uebergangen. Auffallen
> wuerde das nur einem Client, der selbst validiert; die uebliche Stub-Resolver-Bibliothek
> tut das nicht.

## Was danach offen bleibt

**Der eigene rekursive Resolver.** `unbound` neben Pi-hole, das selbst bei den Root-Servern
anfaengt. Danach sieht kein Anbieter mehr das vollstaendige Bild, und DNSSEC wird zur
sinnvollen Ergaenzung statt zur doppelten Buchfuehrung. Die ersten Antworten sind langsamer,
weil jede Kette einmal komplett gelaufen sein will.

**Die Strecke nach draussen ist Klartext.** Auch mit Quad9 geht jede Anfrage unverschluesselt
ueber den Anschluss. Dagegen hilft nur DoT oder DoH zwischen Pi-hole und Upstream — was den
Anbieter nicht aendert, sondern nur die Mitleser auf dem Weg dorthin ausschliesst.
