---
title: Grundausstattung nach der Installation
weight: 60
---

# Grundausstattung nach der Installation

Eine Netinst-Installation ohne Desktop bringt bewusst wenig mit. Das ist die richtige
Ausgangslage fuer einen Server — es haelt Angriffsflaeche und Update-Aufwand klein —, hat aber
zur Folge, dass Werkzeuge erst dann auffallen, wenn man sie braucht. Diese Seite sammelt, was
auf dem Homelab-Rechner nachtraeglich dazugekommen ist.

Ob ein Programm ueberhaupt fehlt, beantwortet:

```sh
command -v curl        # Pfad, wenn vorhanden; sonst leer und Exit-Code 1
dpkg -l curl           # Paketstatus: ii = installiert und konfiguriert
```

## curl

```sh
sudo apt install curl
```

`curl` holt Daten ueber HTTP, HTTPS und ein Dutzend weiterer Protokolle und schreibt sie
standardmaessig nach stdout. Auf einem Server ohne Browser ist es das Standardwerkzeug fuer
alles, was ueber das Netz kommt: Installations-Skripte, API-Abfragen, der schnelle Test, ob
ein Dienst antwortet.

| Option | Wirkung |
|--------|---------|
| `-s` | still — keine Fortschrittsanzeige, aber auch keine Fehlermeldungen |
| `-S` | Fehler trotz `-s` zeigen. Deshalb die uebliche Kombination `-sS` |
| `-L` | Weiterleitungen folgen. Ohne das liefert eine umgezogene URL nur die Weiterleitungsseite |
| `-o datei` / `-O` | in eine Datei schreiben statt nach stdout; `-O` uebernimmt den Namen aus der URL |
| `-I` | nur die Antwort-Header holen |
| `-f` | bei HTTP-Fehlern mit Exit-Code 22 abbrechen statt die Fehlerseite auszugeben |
| `-m 10` | nach 10 Sekunden aufgeben |
| `-w '%{http_code}'` | einzelne Werte der Antwort gezielt ausgeben |

Ein Dienst-Check, der sich in ein Skript einbauen laesst:

```sh
curl -sS -m 5 -o /dev/null -w '%{http_code}\n' http://10.10.10.3/
```

> [!NOTE]
> `wget` und `curl` ueberschneiden sich, sind aber unterschiedlich gebaut: `wget` ist auf das
> Herunterladen von Dateien ausgelegt und kann rekursiv ganze Verzeichnisse spiegeln, `curl`
> auf den einzelnen Aufruf, dessen Antwort weiterverarbeitet wird. Auf einem Debian-System ist
> `wget` meist schon vorhanden — fuer Skripte, die Ausgaben durch eine Pipe schicken, ist
> `curl` trotzdem der naheliegendere Weg.

### Skripte aus dem Netz ausfuehren

Viele Projekte installieren sich mit einem Einzeiler dieser Form:

```sh
curl -sSL https://install.beispiel.net | bash
```

Das ist bequem und gleichzeitig die weiteste denkbare Vertrauenserklaerung: Der Inhalt wird
ungesehen mit den Rechten der aufrufenden Shell ausgefuehrt, und was der Server ausliefert,
kann sich zwischen zwei Aufrufen aendern. Wer nicht blind vertrauen will, trennt Herunterladen
und Ausfuehren:

```sh
curl -sSL https://install.beispiel.net -o install.sh
less install.sh
bash install.sh
```

Der Umweg kostet zwei Minuten und macht aus einem Vertrauensvorschuss eine Entscheidung.

> [!WARNING]
> Scheitert ein HTTPS-Aufruf an der Zertifikatspruefung, fehlt in der Regel das Paket
> `ca-certificates` oder die Systemzeit ist falsch. `-k` schaltet die Pruefung ab und laesst
> den Aufruf gelingen — es behebt nichts, sondern entfernt genau den Schutz, um den es bei
> HTTPS geht.

## Was erfahrungsgemaess noch fehlt

Keine Empfehlung zum Vorrat-Installieren, sondern eine Liste der Pakete, nach denen man auf
einem frischen Debian-Server am haeufigsten greift:

| Paket | Wofuer |
|-------|--------|
| `git` | Konfiguration versionieren, Repositories klonen |
| `htop` | Prozesse und Last im Blick, angenehmer als `top` |
| `rsync` | Dateien und Backups uebertragen, nur Aenderungen |
| `tmux` | Sitzungen, die einen Verbindungsabbruch ueberleben |
| `ncdu` | herausfinden, was den Speicherplatz belegt |
| `unattended-upgrades` | Sicherheitsupdates automatisch einspielen |

Jedes davon wird hier ergaenzt, sobald es tatsaechlich auf der Maschine landet.
