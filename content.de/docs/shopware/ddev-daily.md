---
title: Taeglich damit arbeiten
weight: 40
---

# Taeglich damit arbeiten

Der Shop laeuft. Was jetzt zaehlt, sind die zehn Kommandos, die man wirklich taeglich benutzt —
und ein paar Gewohnheiten, die viel Zeit sparen.

## Die Grundregel

**Alles, was den Shop betrifft, laeuft im Container.** Also `ddev composer` statt `composer`,
`ddev console` statt `bin/console`, `ddev npm` statt `npm`. Der Grund ist immer derselbe: im
Container gelten die PHP- und Node-Versionen des Projekts, auf dem Rechner die des Rechners.

```bash
ddev composer require tydace/beispiel-plugin
ddev console cache:clear
ddev php -v
ddev npm install
```

## An- und ausschalten

```bash
ddev start        # Projekt hochfahren
ddev stop         # nur dieses Projekt
ddev restart      # nach Aenderungen an .ddev/config.yaml
ddev poweroff     # alle Projekte und den Router stoppen
ddev describe     # Status, URLs, Ports, Zugangsdaten
ddev list         # alle Projekte auf dem Rechner
```

`ddev poweroff` ist das Mittel gegen die meisten seltsamen Zustaende — und gegen
Port-Konflikte, wenn ein anderes Projekt noch laeuft.

Container laufen nur, solange man sie braucht. Sie kosten Arbeitsspeicher; zwei Dutzend
gestartete Shops bremsen den Rechner spuerbar.

## Shopware-Konsole

```bash
ddev console cache:clear
ddev console plugin:list
ddev console plugin:refresh
ddev console plugin:install --activate TydaceBeispielPlugin
ddev console database:migrate --all TydaceBeispielPlugin
ddev console dal:refresh:index
ddev console theme:compile
ddev console messenger:stats
```

> [!NOTE]
> `ddev console` ist kein eingebautes Kommando, sondern ein eigenes — siehe
> [Eigene Kommandos](#eigene-kommandos) weiter unten. Ohne dieses Kuerzel hiesse jeder Aufruf
> `ddev exec bin/console cache:clear`.

## Frontend bauen

Ein vollstaendiger Build dauert Minuten und ist beim Entwickeln der falsche Weg:

```bash
ddev sw-build              # Storefront und Administration
ddev sw-build storefront   # nur Storefront
ddev sw-build admin        # nur Administration
```

Beim Arbeiten stattdessen die Watcher — sie kompilieren bei jeder gespeicherten Datei neu und
laden die Seite automatisch:

```bash
ddev sw-watch storefront   # https://tydace.ddev.site:9998
ddev sw-watch admin        # https://tydace.ddev.site:5173
```

> [!WARNING]
> Der Storefront-Watcher laeuft auf einem eigenen Port und umgeht den HTTP-Cache. **Was dort
> funktioniert, muss nach einem echten `sw-build` noch einmal auf der normalen Adresse
> gegengeprueft werden.** Besonders bei Twig-Bloecken und Theme-Variablen gibt es
> Unterschiede.

Nach Plugin-Aenderungen raeumt ein Kommando alles auf einmal auf:

```bash
ddev sw-refresh
```

Es leert den Cache, aktualisiert die Plugin-Liste, kompiliert das Theme und stoesst die
DAL-Indizierung ueber die Queue an.

## Datenbank

```bash
ddev mysql                              # MySQL-Client
ddev import-db --file=dump.sql.gz       # Dump einspielen
ddev export-db --file=dump.sql.gz       # Dump ziehen
ddev adminer                            # Datenbank im Browser
```

Der praktischste Teil sind Snapshots:

```bash
ddev snapshot                           # Zustand sichern
ddev snapshot --list
ddev snapshot restore --latest          # zurueckholen
```

**Vor jeder Migration, jedem Import und jedem riskanten Konsolen-Kommando einen Snapshot
anlegen.** Das kostet Sekunden und ersetzt das Neuaufsetzen des Shops, wenn etwas schiefgeht.

## Debugging

```bash
ddev xdebug on      # einschalten
ddev xdebug off     # ausschalten
ddev xdebug status
```

Xdebug kostet spuerbar Geschwindigkeit, deshalb ist es standardmaessig aus. Im Editor auf Port
9003 lauschen, den Pfad-Mapping-Eintrag auf `/var/www/html` -> Projektverzeichnis setzen,
fertig.

Weitere nuetzliche Einblicke:

```bash
ddev logs -f                # Webserver-Logs live
ddev logs -s db             # Datenbank-Logs
ddev ssh                    # Shell im Web-Container
ddev mailpit                # abgefangene Mails
ddev redis-cli              # Redis-Konsole (Add-on)
ddev rabbitmqctl list_queues # Queue-Stand (Add-on)
```

Fuer Shopware-Fehler ist meist das Symfony-Log ergiebiger als die Webserver-Logs:

```bash
ddev exec tail -f var/log/dev.log
```

## Eigene Kommandos

Der Punkt, an dem ddev vom Werkzeug zum Projektwerkzeug wird: Haeufige Ablaeufe lassen sich als
Kommando ablegen. Eine ausfuehrbare Datei unter `.ddev/commands/web/` genuegt — sie laeuft im
Web-Container.

`.ddev/commands/web/console` aus tydace:

```bash
#!/bin/bash

## Description: Shopware-Konsole aufrufen (bin/console)
## Usage: console [args]
## Example: "ddev console cache:clear", "ddev console plugin:list"

php /var/www/html/bin/console "$@"
```

Ausfuehrbar machen nicht vergessen:

```bash
chmod +x .ddev/commands/web/console
```

Danach steht `ddev console` allen zur Verfuegung, die das Projekt klonen — die Kommandos
liegen im Repository. Die `## Description`-Zeile taucht in `ddev help` auf.

Kommandos, die auf dem **Host** laufen sollen (etwa etwas, das einen Browser oeffnet),
gehoeren nach `.ddev/commands/host/`.

In tydace gibt es davon vier: `console`, `sw-build`, `sw-watch` und `sw-refresh`. Sie ersetzen
Kommandoketten, die man sonst taeglich neu tippt.

## Ein neues Plugin anlegen

Das Produktions-Template hat `custom/static-plugins/*` bereits als Composer-`path`-Repository
eingetragen:

```bash
mkdir -p custom/static-plugins/TydaceBeispiel
# composer.json mit "name": "tydace/beispiel" und
# "type": "shopware-platform-plugin" anlegen
ddev composer require tydace/beispiel
ddev console plugin:refresh
ddev console plugin:install --activate TydaceBeispiel
```

Composer legt unter `vendor/` einen Symlink an. Aenderungen am Plugin wirken damit sofort,
ohne erneutes `composer install`.

Wo was hingehoert:

- `custom/static-plugins/` — eigene Plugins und Themes, **wird versioniert**
- `custom/plugins/` — Drittanbieter- und Marketplace-Plugins, **gitignored**

## Aufraeumen

```bash
ddev delete           # Container und Datenbank weg, Dateien bleiben
ddev delete --omit-snapshot   # ohne Sicherheitskopie der Datenbank
ddev clean            # nicht mehr benutzte Docker-Images entfernen
```

`ddev delete` klingt drastischer als es ist: Das Projektverzeichnis bleibt unangetastet, ein
`ddev start` baut die Umgebung neu auf. Nur der Datenbestand ist weg — deshalb legt ddev
vorher automatisch einen Snapshot an.

Docker-Images summieren sich ueber die Zeit auf viele Gigabyte. `ddev clean` gelegentlich
laufen zu lassen, schafft Platz.

## Was sich einzupraegen lohnt

| Situation | Kommando |
|---|---|
| Irgendetwas ist komisch | `ddev restart` |
| Immer noch komisch | `ddev poweroff && ddev start` |
| Aenderung wirkt nicht | `ddev console cache:clear` |
| Plugin taucht nicht auf | `ddev console plugin:refresh` |
| Vor etwas Riskantem | `ddev snapshot` |
| Nach Plugin-Aenderungen | `ddev sw-refresh` |
| Keine Ahnung, wo etwas laeuft | `ddev describe` |
