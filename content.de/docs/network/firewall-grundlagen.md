---
title: Firewalls verstehen
weight: 35
---

# Firewalls verstehen

Eine Firewall kontrolliert Netzwerkverkehr anhand von Regeln. Sie entscheidet, welche
Verbindungen erlaubt sind und welche sie sperrt. Im Heimnetz kann das bedeuten: Der Laptop
darf ins Internet, der Fernseher darf den DNS-Server fragen, ein Gast darf aber nicht auf die
Verwaltung des Routers zugreifen.

Die Firewall kann auf einem Router wie der Dream Machine laufen oder direkt auf einem
Rechner. Auf dem Router kontrolliert sie den Verkehr, der durch ihn hindurch oder an ihn
selbst geht. Auf dem Rechner schuetzt sie dessen eigene Netzwerkzugriffe. Diese beiden
Einsatzorte beschreibt auch [NIST in den Grundlagen zu Firewalls](https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-41r1.pdf).

## Was eine Regel prueft

Geraete verschicken Daten in Paketen. Zu deren Angaben gehoeren Absender und Empfaenger.
Bei TCP und UDP kommen Portnummern dazu: Sie helfen dabei, den Verkehr einem Dienst auf dem
Zielgeraet zuzuordnen. DNS verwendet zum Beispiel Port 53, HTTPS normalerweise Port 443.

Eine einfache Firewall-Regel beschreibt, welcher Verkehr gemeint ist und was damit passiert:

| Merkmal | Beispiel aus dem Homelab |
|---------|-------------------------|
| Quelle | Gaestenetz `10.10.40.0/24` |
| Ziel | DNS-Server `10.10.10.3` |
| Protokoll | TCP oder UDP |
| Zielport | 53 fuer DNS |
| Aktion | erlauben |

Damit duerfen Gaeste Namen beim DNS-Server aufloesen. Die Freigabe gilt nur fuer diesen
Dienst. Seine Weboberflaeche auf einem anderen Port bleibt von dieser Regel unberuehrt und
kann durch eine nachfolgende Sperre geschuetzt werden.

## Wie eine Entscheidung entsteht

Bei einem geordneten Regelwerk wie dem hier verwendeten prueft die Firewall die Regeln von
oben nach unten. Die erste passende Regel entscheidet. Eine Ausnahme muss deshalb vor der
allgemeinen Sperre stehen.

Fuer neue Anfragen aus dem Gaestenetz koennte der relevante Ausschnitt so aussehen:

| Reihenfolge | Verkehr | Entscheidung |
|-------------|---------|--------------|
| 1 | zum DNS-Server auf Port 53, TCP oder UDP | erlauben |
| 2 | zu den uebrigen internen Zielen und Diensten | sperren |

Eine DNS-Anfrage passt zur ersten Regel und darf passieren. Ein Zugriff auf die
Server-Verwaltung passt erst zur zweiten Regel und wird blockiert. Stuende die pauschale
Sperre zuerst, wuerde sie auch DNS treffen. Fuer Internetzugang und Dienste des Routers gibt
es eigene Regeln; die Tabelle zeigt nur diesen Ausschnitt.

*Allow* laesst passenden Verkehr durch. *Drop* verwirft ihn ohne Antwort, *Reject* lehnt ihn
mit einer Rueckmeldung ab. Ein Timeout allein beweist deshalb keine funktionierende Sperre:
Das Ziel koennte auch ausgeschaltet sein.

## Warum Antworten zurueckkommen duerfen

Eine **zustandsbehaftete Firewall**, auch *stateful firewall* genannt, merkt sich laufende
Kommunikation. So kann sie eine Antwort einer vorher erlaubten Anfrage zuordnen. Die passende
Zustandsregel laesst diese Antwort zurueck, ohne neue Verbindungen in Gegenrichtung generell
freizugeben. Bei UDP erfolgt die Zuordnung trotz fehlendem Verbindungsaufbau anhand der
Adressen, Ports und eines Zeitfensters. Die [Netfilter-Dokumentation zu Connection Tracking](https://wiki.nftables.org/wiki-nftables/index.php/Matching_connection_tracking_stateful_metainformation)
erklaert die dabei verwendeten Zustaende.

Im Beispiel fragt ein Gast nach der Adresse einer Website. Der DNS-Server darf darauf
antworten. Daraus folgt keine Erlaubnis fuer den Server, seinerseits beliebige neue
Verbindungen zum Gast aufzubauen.

In Regeln heisst solcher Antwortverkehr oft *established*. *Related* bezeichnet Verkehr,
der einer bestehenden Verbindung zugeordnet werden kann, etwa bestimmte Fehlermeldungen.
Diese Freigaben sind Teil des Regelwerks.

## Wo VLANs und Zonen ins Spiel kommen

Ein [VLAN]({{< relref "/docs/network/vlan-grundlagen" >}}) trennt ein lokales Netz auf der
Ethernet-Ebene. Damit Geraete in unterschiedlichen VLANs miteinander reden koennen, brauchen
sie einen Router. Im Homelab ist das die UDM. Dort entscheidet die Firewall, welcher Verkehr
zwischen den Netzen erlaubt ist.

Geraete im selben VLAN und Subnetz kommunizieren normalerweise direkt ueber den Switch oder
Access Point. Dieser Verkehr durchlaeuft die Router-Firewall nicht. Fuer eine Trennung auch
innerhalb dieses Netzes braucht es weitere Massnahmen, etwa Client-Isolation oder die
Firewall auf dem Endgeraet.

Eine **Zone** fasst Netze fuer gemeinsame Firewall-Regeln zusammen. *IoT* kann zum Beispiel
das Netz der Fernseher enthalten, *Internal* mehrere Netze mit Servern und Infrastruktur.
Der Zonenname allein bewirkt keine Sperre; entscheidend sind die Regeln. UniFi unterscheidet
ausserdem zwischen Verkehr **durch den Router** und Verkehr **zum Router selbst** in der Zone
*Gateway*. Dort laufen beispielsweise DHCP und die Verwaltungsoberflaeche. Das beschreibt
[Ubiquiti in der Erklaerung der Firewall-Zonen](https://help.ui.com/hc/en-us/articles/115003173168-Zone-Based-Firewalls-in-UniFi).

## Was eine Firewall nicht automatisch erkennt

Eine erlaubte Verbindung ist nicht automatisch harmlos. Darf ein Geraet per HTTPS ins
Internet, kann darueber auch unerwuenschter Verkehr laufen. Ein einfacher Paketfilter
beurteilt Adressen, Ports und Zustaende; den verschluesselten Inhalt einer HTTPS-Verbindung
kann er ohne zusaetzliche Entschluesselung nicht lesen.

Angriffserkennung und -abwehr, haeufig **IDS** und **IPS** genannt, ergaenzen das mit einer
Untersuchung auf verdaechtige Muster. Auch damit bleiben Updates und sichere Zugaenge noetig.
Ein **DNS-Filter** wie Pi-hole hat wieder eine andere Aufgabe: Er filtert Namensanfragen.
Eine bereits bekannte IP-Adresse oder ein anderer Resolver koennen diesen Filter umgehen,
wenn die Firewall den Verkehr erlaubt.

Auch **NAT** ist eine andere Funktion: Es uebersetzt Netzwerkadressen. Ob eine Verbindung
zulaessig ist, legt das Firewall-Regelwerk fest. IPv4 und IPv6 muessen dabei beide bedacht
werden, sobald beide im Netz genutzt werden.

## Woran man wirksame Regeln erkennt

Zu jeder Freigabe gehoert ein Test, ob der benoetigte Dienst erreichbar ist. Zu jeder Sperre
gehoert ein Test aus dem betroffenen Netz, ob der verbotene Zugriff scheitert. Dafuer eignet
sich ein bekannter, laufender Dienst besser als ein beliebiges Ping-Ziel: Ping prueft ICMP,
nicht den Zugriff auf eine Weboberflaeche oder einen SSH-Port.

Trefferzaehler und Logs helfen dabei, den Test einer Regel zuzuordnen. Im Gaestenetz muss
beispielsweise DNS funktionieren, waehrend die Server-Verwaltung unerreichbar bleibt.

Die konkreten Regeln und Ausnahmen stehen unter
[Firewall zwischen Segmenten]({{< relref "/docs/network/firewall-policies" >}}).
Welche davon im Homelab umgesetzt wurden, beschreibt der
[Beitrag zur Firewall-Umstellung]({{< relref "/posts/homelab/udm-audit" >}}).
