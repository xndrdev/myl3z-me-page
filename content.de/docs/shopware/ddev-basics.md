---
title: Was ist ddev
weight: 10
---

# Was ist ddev

ddev ist ein Open-Source-Werkzeug, das lokale Entwicklungsumgebungen in Docker-Containern
erzeugt und verwaltet. Statt PHP, MySQL, Node und nginx auf dem eigenen Rechner zu
installieren und zu pflegen, beschreibt man in einer Datei, was ein Projekt braucht — und ddev
baut daraus eine laufende Umgebung.

Konkret: `ddev start` in einem Projektverzeichnis, und wenige Sekunden spaeter laeuft der Shop
unter einer HTTPS-Adresse.

## Das Problem, das es loest

Ohne so ein Werkzeug hat jeder Entwickler eine andere Umgebung. Ein Shop verlangt PHP 8.2, der
naechste 8.4. Auf dem Server laeuft MySQL 8.4, lokal MariaDB 10.6. Der Kollege hat eine andere
Node-Version, deshalb sieht sein Storefront-Build anders aus. Das Ergebnis kennt jeder: "Bei
mir laeuft es."

ddev dreht das um. Die Umgebung gehoert zum Projekt, nicht zum Rechner:

- **Versionen stehen im Projekt.** `.ddev/config.yaml` legt PHP-, Datenbank- und Node-Version
  fest. Die Datei liegt im Git-Repository — wer das Projekt klont, bekommt automatisch dieselbe
  Umgebung.
- **Projekte stoeren sich nicht.** Zwanzig Shops mit fuenf verschiedenen PHP-Versionen koennen
  nebeneinander existieren. Jeder in eigenen Containern.
- **Der Rechner bleibt sauber.** Kein Homebrew-PHP, keine lokale MySQL-Installation, keine
  Node-Versionsverwaltung. Nur Docker und ddev.
- **Wegwerfbar.** `ddev delete` entfernt Container und Datenbank, das Projektverzeichnis
  bleibt. Ein kaputter Shop ist kein verlorener Tag mehr.

## Wie es funktioniert

ddev ist im Kern ein Generator fuer Docker-Compose-Dateien. Aus der Konfiguration erzeugt es
einen Satz Container und verbindet sie:

| Container | Rolle |
|---|---|
| **web** | nginx oder Apache, PHP-FPM, Node, Composer — hier laeuft der Shop |
| **db** | MySQL, MariaDB oder PostgreSQL in der konfigurierten Version |
| **ddev-router** | Ein zentraler Reverse Proxy fuer *alle* Projekte, verteilt Anfragen nach Hostname und terminiert HTTPS |
| **ddev-ssh-agent** | Reicht den SSH-Key des Hosts in die Container, fuer private Composer-Repositories |

Router und SSH-Agent laufen einmal fuer alle Projekte, `web` und `db` je Projekt.

Zwei Dinge, die den Alltag angenehm machen:

**Das Projektverzeichnis ist in den Container gemountet.** Eine Datei, die du im Editor
speicherst, ist im Container sofort da. Kein Deployment, kein Sync — Container sind hier keine
Blackbox, sondern nur eine Laufzeitumgebung um deine Dateien herum.

**HTTPS funktioniert einfach.** ddev vergibt automatisch die Adresse
`https://<projektname>.ddev.site` und stellt ueber
[mkcert](https://github.com/FiloSottile/mkcert) ein lokal vertrauenswuerdiges Zertifikat aus.
Keine Zertifikatswarnung, keine `/etc/hosts`-Eintraege — die Domain `ddev.site` loest per
Wildcard-DNS ohnehin auf `127.0.0.1` auf.

Fuer Shopware ist HTTPS kein Luxus: Zahlungsarten, Service Worker und einige Admin-Funktionen
verhalten sich unter `http://` anders als in Produktion.

## Was mitgeliefert wird

Vieles, was man sonst separat aufsetzt, ist eingebaut:

- **Mailpit** faengt jede Mail ab, die der Shop verschickt. Bestellbestaetigungen landen in
  einer Weboberflaeche statt bei echten Empfaengern.
- **Xdebug** wird mit `ddev xdebug on` eingeschaltet — aus gutem Grund nicht dauerhaft, es
  kostet spuerbar Geschwindigkeit.
- **Datenbank-Im- und -Export** ueber `ddev import-db` und `ddev export-db`.
- **Snapshots**: `ddev snapshot` friert den Datenbankstand ein, `ddev snapshot restore` holt
  ihn zurueck. Praktisch vor riskanten Migrations.
- **Add-ons** fuer zusaetzliche Dienste — OpenSearch, Redis, RabbitMQ und mehr werden mit
  `ddev add-on get` ergaenzt.

## Warum ddev und nicht etwas anderes

Es gibt Alternativen: Docker Compose von Hand, Lando, Konkurrenten wie Warden, oder Shopwares
eigenes `devenv` auf Nix-Basis.

Fuer ddev spricht:

- **Shopware ist ein unterstuetzter Projekttyp.** `type: shopware6` setzt docroot,
  nginx-Konfiguration und Umgebungsvariablen passend.
- **Der Einstieg ist kurz.** Wer Docker Compose selbst pflegt, schreibt und wartet dieselben
  200 Zeilen in jedem Projekt neu.
- **Es ist im Shopware-Umfeld verbreitet.** Entsprechend gibt es fertige Add-ons fuer die
  Dienste, die ein Shop braucht, und Antworten auf die meisten Fragen.

Der ehrliche Gegenpunkt: ddev bildet die Produktionsumgebung **nicht exakt** ab. Es ist eine
Entwicklungsumgebung — bequem, schnell, mit Debugging-Werkzeugen. Wer testen will, ob ein
Deployment auf dem Zielserver funktioniert, braucht ein Staging-System, keine lokale
Simulation.

## Weiter

Als Naechstes: [ddev installieren]({{< relref "ddev-install" >}}).
