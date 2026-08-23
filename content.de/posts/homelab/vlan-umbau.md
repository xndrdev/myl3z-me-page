---
title: Homelab, dritter Schritt — das Netz in Segmente schneiden
date: 2026-08-23
---

# Homelab, dritter Schritt — das Netz in Segmente schneiden

Der [zweite Teil]({{< relref "/posts/homelab/dns-upstream" >}}) endete mit einem sauberen
Aufbau und einem Plan fuer den naechsten Schritt: fuenf Segmente statt einem flachen `/16`.
Der Plan lag seit Tagen fertig da, mit einer nummerierten Reihenfolge fuer den Umbau-Abend.

Von dieser Reihenfolge hat Schritt eins funktioniert. Danach fing das Netz an,
zurueckzureden.

## Warum ueberhaupt schneiden

Bis zu diesem Abend war das Heimnetz ein einziger Raum. Alles darin: die Dream Machine, der
Thin Client mit dem DNS-Filter, Telefone, der Arbeitsrechner, der Fernseher, der Drucker, ein
paar Smart-Home-Geraete. Jedes Geraet konnte jedes andere direkt erreichen. Nicht, weil das
irgendwo so eingestellt war, sondern weil nichts es verhinderte.

Das ist die Voreinstellung praktisch jedes Heimnetzes, und sie faellt nie auf, solange nichts
passiert. Der eigentliche Einwand dagegen ist kein Angriffsszenario, sondern eine Frage der
Zustaendigkeit: Ein Fernseher hat keinen Grund, das Management-Interface des Routers zu sehen.
Ein Saugroboter hat keinen Grund, den Arbeitsrechner zu sehen. Diese Geraete laufen mit
Firmware, die ich nicht gelesen habe, die selten Updates bekommt und die auch nach dem Ende
des Herstellersupports weiterlaeuft. Ihnen Zugriff auf alles zu geben, ist keine Entscheidung
gewesen — es ist nur nie eine getroffen worden.

Dazu kommt der praktische Anlass. Als naechstes zieht ein Hypervisor ein, auf dem VMs entstehen
und wieder verschwinden. Eine Maschine zum Experimentieren gehoert nicht in dasselbe Netz wie
die Geraete, mit denen man das Netz verwaltet. Und ein Hypervisor will schon bei der
Installation wissen, in welchem Segment er liegt — wer das Netz danach umbaut, konfiguriert
ihn ein zweites Mal, an den Bridges einer Maschine ohne Bildschirm.

Bei einer Maschine kostet der Schnitt einen Abend. Bei fuenf kostet er ein Wochenende.

## Sechs Raeume statt einem

Aus einem Netz wurden sechs, jedes mit eigener Aufgabe: **Infrastruktur** fuer alles, womit man
das Netz verwaltet. **Server** fuer selbst gebaute Dienste. **Clients** fuer Arbeitsgeraete.
**IoT** fuer fremde Firmware. **Gaeste** fuer Besuch. Und **Kids**.

Das sechste war nicht geplant. Es entstand mitten im Umbau, aus einer simplen Rechnung: Die
Netze waren ohnehin offen, die SSIDs ohnehin neu anzulegen — ein weiteres Segment kostete an
diesem Abend fuenf Minuten. Nachtraeglich waere es ein zweiter Abend gewesen. Kindergeraete
brauchen andere Blocklisten und andere Zeiten als der Rest des Haushalts, und beides ist an
ein Netz gebunden billiger zu pflegen als geraeteweise.

Die Adressen folgen einem Schema, das sich selbst erklaert: Die dritte Stelle ist die
VLAN-Nummer. An `10.10.30.47` sieht man ohne Nachschlagen, dass ein IoT-Geraet gemeint ist.
Das ist keine Technik, sondern eine Lesehilfe fuer den Moment, in dem man in einem Logfile
sucht. Das vollstaendige Layout samt der Abwaegung, wie fein man sinnvoll trennt, steht unter
[Netz-Layout mit VLANs]({{< relref "/docs/network/vlan-layout" >}}).

## Der Plan war in der falschen Reihenfolge

Der geschriebene Ablauf begann damit, die neuen Netze anzulegen, waehrend das bestehende
unberuehrt bleibt. Das alte `/16` sollte zuletzt verkleinert werden — zuletzt, weil ab da
alles, was noch im alten Bereich haengt, nicht mehr erreichbar ist.

Das laesst sich nicht durchhalten. UniFi verweigert jedes neue Netz, dessen Bereich sich mit
einem bestehenden ueberschneidet, und `10.10.20.0/24` liegt nun einmal in `10.10.0.0/16`. Es
betraf nicht ein Segment, sondern alle: Jedes neue Netz liegt im alten.

Der vorsichtigste Schritt des Plans musste damit an den Anfang. Der Umbau beginnt mit genau der
Aktion, die am meisten kaputt macht — das Default-Netz von `/16` auf `/24` verkleinern und
damit jedem Geraet ausserhalb von `10.10.1.0/24` den Boden wegziehen. Es gibt keine Variante,
in der man sich vorsichtig herantastet.

Das ist die Art Fehler im eigenen Plan, die man nicht durch Nachdenken findet. Sie steht auch
nicht in der Anleitung, an der man sich orientiert hat, weil sie nur auftritt, wenn man ein
bestehendes Netz umbaut statt eines auf der gruenen Wiese aufzusetzen.

## Alles routet, nichts kommt an

Der teuerste Fehler des Abends kam spaeter und sah zuerst wie ein Erfolg aus. Die sechs Netze
standen, die Gateways antworteten, die Namensaufloesung funktionierte ueber Segmentgrenzen
hinweg. Nur kam kein Geraet in den neuen Segmenten ins Internet.

Kein Timeout beim Gateway. Kein DNS-Fehler — Namen wurden sauber aufgeloest. Pakete gingen
hinaus, es kam nichts zurueck.

Ich habe an dieser Stelle lange in die falsche Richtung gesucht, weil das Symptom nach einem
DNS-Problem aussieht und DNS in diesem Homelab das Thema der letzten zwei Wochen war. Man
sucht dort, wo man zuletzt gearbeitet hat.

Die Ursache lag in einer Regel, die man beim Anlegen eines Netzes nie zu Gesicht bekommt: eine
NAT-Regel namens *LAN to WAN Masquerade*, deren *Source* auf das Infrastruktur-Netz gesetzt
war — auf `10.10.1.0/24`, also auf genau das eine Netz, das es vor dem Umbau schon gab. Pakete
aus den neuen Segmenten wurden korrekt geroutet und beim Verlassen Richtung Internet **nicht
maskiert**. Sie gingen mit privater Absenderadresse hinaus, und dort endet jede Reise.

Der Fix war ein einziges Feld: *Source* auf *Any*. Die halbe Stunde davor ging fuer die
Erkenntnis drauf, dass ein Netz drei Dinge unabhaengig voneinander koennen muss — routen,
aufloesen, maskieren — und dass der Ausfall des dritten aussieht wie ein Problem mit dem
zweiten. Der Test, der die beiden trennt, ist ein Ping auf eine nackte IP: Antwortet das
Gateway und loest DNS auf, waehrend `9.9.9.9` nicht erreichbar ist, liegt es nicht am Namen.

Beide Fallstricke samt Ablauf stehen jetzt unter
[VLANs auf der UDM einrichten]({{< relref "/docs/network/unifi-vlan-umbau" >}}).

## Vier Funknetze und ein Kompromiss

Am Kabel entscheidet der Switch-Port, in welchem Segment ein Geraet landet. Im WLAN gibt es
keinen Port — dort entscheidet die SSID. Aus einem Funknetz wurden vier: `xlab` fuer Clients,
`xlab-IoT`, `xlab-Kids`, `xlab-Guests`.

Das IoT-Netz ist dabei bewusst das schwaechste: nur 2,4 GHz, WPA2 statt WPA3, Protected
Management Frames abgeschaltet. Jede dieser Einstellungen senkt das Sicherheitsniveau, und
jede ist noetig, weil ein guter Teil der Smart-Home-Hardware sonst nicht stabil verbindet —
Funkmodule, die kein WPA3 koennen, und Firmware, die bei erzwungenem PMF die Verbindung nach
ein paar Minuten wieder verliert.

Das ist der Punkt, an dem Segmentierung ihren eigentlichen Nutzen zeigt. Ich muss diese
Geraete nicht sicherer machen, als sie sein koennen. Ich muss nur dafuer sorgen, dass hinter
ihrem Netz eine Grenze steht. Die schwaechere Verschluesselung ist tragbar, weil das, was
dort einbricht, einen Fernseher und einen Drucker vorfindet — und keinen Arbeitsrechner. Die
Einzelheiten stehen unter
[WLAN auf VLANs verteilen]({{< relref "/docs/network/wlan-ssid-vlan" >}}).

## Was ein VLAN nicht tut

Die Erkenntnis, die ich am ehesten mitnehme, ist eine begriffliche. Ein VLAN trennt nichts.

Es sorgt nur dafuer, dass Geraete einander nicht mehr direkt sehen und ihr Verkehr ueber das
Gateway laufen muss. Dort — und ausschliesslich dort — entscheidet die Firewall, wer wen
erreicht. Ohne Regelwerk routet ein sauber geschnittenes Netz jeden Verkehr genauso brav wie
vorher, nur mit mehr Zwischenschritten.

Dazu gehoert ein zweiter Punkt, den ich vorher nicht sauber getrennt hatte: Die Maske ist Teil
der Trennung, nicht Kosmetik. Solange ein Geraet `/16` konfiguriert hat, haelt es alles in
`10.10.x.x` fuer seinen direkten Nachbarn und fragt per ARP statt ueber das Gateway. Erst mit
`/24` entsteht ueberhaupt die Stelle, an der eine Regel greifen kann. VLAN und Subnetz sind
zwei verschiedene Schichten, die nur deshalb wie eine aussehen, weil man sie sinnvollerweise
eins zu eins zuschneidet. Was dahintersteckt, steht unter
[VLANs verstehen]({{< relref "/docs/network/vlan-grundlagen" >}}), das Regelwerk unter
[Firewall zwischen Segmenten]({{< relref "/docs/network/firewall-policies" >}}).

## Was noch nicht fertig ist

Das Geruest steht und ist geprueft: sechs Segmente, jedes mit eigenem Gateway, Pi-hole
beantwortet Anfragen aus allen Netzen, die Namensaufloesung laeuft ueber die Segmentgrenzen,
das Client-Netz kommt nicht mehr an das Management der Infrastruktur heran.

Bezogen ist es nicht. Ein Grossteil der IoT-Hardware haengt noch dort, wo sie vor dem Umbau
war, und jedes dieser Geraete ist einmal von Hand in die neue SSID zu bringen. Bei Telefonen
ist das eine Minute; bei Geraeten, die nur ueber eine Hersteller-App in einen Kopplungsmodus zu
bringen sind, ist es der Teil, der laenger dauert als die gesamte Netzkonfiguration davor.

Und der Hypervisor, dessentwegen der ganze Umbau vorgezogen wurde, liegt noch im falschen
Segment — auf einer Adresse im Infrastruktur-Netz, mit einem Hostnamen aus einer Zeit, in der
die Maschine noch keine Aufgabe hatte. Er muss ins Server-VLAN, sein Switch-Port braucht das
Profil mit getaggten VLANs fuer die kuenftigen VMs.

Das ist dann der vierte Schritt. Diesmal mit einem Netz, das schon weiss, wohin er gehoert.
