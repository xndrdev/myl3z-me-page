---
title: sudo nachruesten
weight: 50
---

# sudo nachruesten

Auf einer frischen Debian-Installation kann `sudo` fehlen — und zwar abhaengig von einer
Entscheidung, die man beim Installieren getroffen hat, ohne die Folge zu sehen:

| Root-Passwort im Installer | Ergebnis |
|----------------------------|----------|
| vergeben | root ist aktiv, `sudo` wird **nicht** installiert |
| leer gelassen | root ist gesperrt, `sudo` wird installiert und der erste Benutzer in die Gruppe `sudo` aufgenommen |

Wer also ein Root-Passwort gesetzt hat, steht anschliessend vor einem `sudo: command not
found` und muss nachruesten.

## Installieren und Benutzer freischalten

```sh
su -
apt install sudo
usermod -aG sudo xander
exit
```

`su -` mit Bindestrich ist wichtig: es startet eine Login-Shell mit der Umgebung von root.
Ohne den Bindestrich bleibt der `PATH` des eigenen Benutzers gesetzt, und `/usr/sbin` fehlt
darin — Befehle wie `usermod` sind dann scheinbar nicht vorhanden.

> [!WARNING]
> `usermod -aG` — das `-a` fuer *append* darf nicht fehlen. Ohne `-a` ersetzt der Befehl
> saemtliche Nebengruppen des Benutzers durch die angegebene. Wer so aus Gruppen wie `audio`,
> `video` oder `docker` fliegt, merkt es erst spaeter an kaputten Berechtigungen.

## Neu anmelden

Gruppenmitgliedschaften wertet der Kernel beim Login aus. In einer bereits offenen Sitzung
aendert sich nichts, auch nicht nach `usermod` — die Shell traegt die Gruppen von vorhin. Also
abmelden und neu verbinden:

```sh
exit
ssh dns01
```

Pruefen, was tatsaechlich gilt:

```sh
id                # sudo muss in der Gruppenliste auftauchen
sudo -v           # fragt einmal das eigene Passwort, dann still
sudo -l           # listet auf, was erlaubt ist
```

> [!NOTE]
> `newgrp sudo` zieht die Gruppe in die laufende Shell nach, ohne Neuanmeldung. Das ist ein
> Behelf fuer den Moment — Kindprozesse anderer Sitzungen sehen die Gruppe weiterhin nicht.

## Eigene Regeln: sudoers.d statt sudoers

`/etc/sudoers` wird nicht mit dem Editor geoeffnet. Ein Syntaxfehler in dieser Datei sperrt
den Zugang zu `sudo` komplett aus — und wenn root gleichzeitig gesperrt ist, bleibt nur noch
der Weg ueber ein Live-System. `visudo` verhindert genau das: es prueft die Syntax beim
Speichern und weigert sich, kaputte Regeln zu uebernehmen.

Eigene Regeln gehoeren nicht in die Hauptdatei, sondern als eigene Datei daneben — die
Hauptdatei bleibt so unveraendert und paketaktualisierbar:

```sh
sudo visudo -f /etc/sudoers.d/xander
```

Fuer den Dateinamen gelten zwei Regeln: kein Punkt und keine Tilde darin, sonst ignoriert
`sudo` die Datei stillschweigend. Die Rechte muessen `0440` sein — `visudo` setzt sie selbst.

## Passwortabfrage

Standardmaessig fragt `sudo` nach dem *eigenen* Passwort, nicht nach dem von root, und merkt
sich die Eingabe fuenfzehn Minuten fuer dasselbe Terminal.

```sh
sudo -k           # Merker sofort verwerfen
```

Die Dauer laesst sich pro Benutzer anpassen:

```text
Defaults:xander timestamp_timeout=30
```

Verbreitet ist auch, die Abfrage ganz abzuschalten:

```text
xander ALL=(ALL) NOPASSWD: ALL
```

> [!WARNING]
> Auf einem Server, der per SSH-Key erreichbar ist, ist die Passwortabfrage die letzte
> Huerde zwischen einem abhandengekommenen Schluessel und root. `NOPASSWD` nimmt sie weg —
> vertretbar fuer einzelne, klar begrenzte Befehle, die ein Automatismus braucht:
>
> ```text
> xander ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart pihole-FTL
> ```
>
> Als Pauschalfreigabe fuer `ALL` ist es das nicht.
