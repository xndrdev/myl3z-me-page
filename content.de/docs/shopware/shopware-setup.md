---
title: Shopware 6 aufsetzen
weight: 30
---

# Shopware 6 aufsetzen

Vom leeren Verzeichnis zum eingerichteten Shop. Als Beispiel dient **tydace**, ein frisch
aufgesetzter Shopware-6.7-Shop.

Zeitbedarf: knapp zehn Minuten, davon das meiste Warten auf Composer.

## Grundgeruest

### 1. Projekt anlegen

```bash
mkdir tydace && cd tydace
ddev config --project-type=shopware6 --docroot=public
```

`ddev config` schreibt `.ddev/config.yaml`. Der Projekttyp `shopware6` setzt die passende
nginx-Konfiguration und die Umgebungsvariablen, die Shopware erwartet; `--docroot=public` sagt
ddev, wo `index.php` liegt.

Der Projektname ergibt sich aus dem Verzeichnisnamen und bestimmt die spaetere Adresse:
`tydace` -> `https://tydace.ddev.site`.

### 2. Versionen festlegen

Die erzeugte `.ddev/config.yaml` enthaelt vor allem Kommentare. Die Zeilen, die zaehlen — hier
der Stand von tydace:

```yaml
name: tydace
type: shopware6
docroot: public
php_version: "8.4"
webserver_type: nginx-fpm
database:
    type: mysql
    version: "8.4"
nodejs_version: "22"
timezone: Europe/Berlin
disable_settings_management: true
```

> [!WARNING]
> **Diese Versionen an der Produktionsumgebung ausrichten, nicht am eigenen Geschmack.** Ein
> Shop, der lokal auf PHP 8.4 laeuft und auf dem Server auf 8.2, bringt Fehler erst beim
> Deployment zum Vorschein. Die Shopware-Version bestimmt, was moeglich ist: 6.7 verlangt
> mindestens PHP 8.2.

`disable_settings_management: true` haelt ddev davon ab, eigene Konfigurationsdateien in das
Projekt zu schreiben — die Shopware-Konfiguration bleibt damit in eigener Hand.

### 3. Container starten

```bash
ddev start
```

Beim ersten Mal laedt Docker die Images herunter, das dauert. Danach sind es Sekunden.

## Shopware installieren

### 4. Quellcode holen

```bash
ddev composer create-project shopware/production
```

`shopware/production` ist das offizielle Produktions-Template: Shopware-Kern, Administration,
Storefront und die Verzeichnisstruktur fuer eigene Plugins.

Eine bestimmte Version:

```bash
ddev composer create-project "shopware/production:6.7.13.0"
```

> [!NOTE]
> **`ddev composer`, nicht `composer`.** Der Befehl laeuft im Container — mit dessen
> PHP-Version und Extensions. Lokal installiertes Composer benutzt die PHP-Version des
> Rechners, und die passt selten.

### 5. Einrichten

```bash
ddev console system:install --basic-setup --create-database
```

Das legt die Datenbank an, spielt das Schema ein, richtet Sales Channel und Administrator ein.
`--basic-setup` erzeugt den Standard-Sales-Channel und den Admin-Zugang `admin` / `shopware`.

Laeuft der Befehl in eine bestehende Installation, hilft `--force` — **es leert die
Datenbank**, also nur auf einem Shop, dessen Daten entbehrlich sind.

### 6. Assets bauen

```bash
ddev exec bin/build-storefront.sh
ddev exec bin/build-administration.sh
```

Beides dauert einige Minuten. Ohne diesen Schritt erscheint die Storefront ohne Styles und die
Administration bleibt weiss.

### 7. Aufrufen

```bash
ddev launch          # Storefront
ddev launch /admin   # Administration
```

| Was | Adresse | Zugang |
|---|---|---|
| Storefront | `https://tydace.ddev.site` | — |
| Administration | `https://tydace.ddev.site/admin` | `admin` / `shopware` |
| Mailpit | `ddev mailpit` | — |

Damit laeuft ein vollstaendiger Shop.

## Dienste ergaenzen

Der Shop laeuft jetzt mit MySQL und dem Dateisystem-Cache. Fuer ein realistisches Setup fehlen
die Dienste, die in Produktion mitlaufen. ddev bringt sie als Add-ons mit:

```bash
ddev add-on get ddev/ddev-opensearch    # Produktsuche
ddev add-on get ddev/ddev-redis         # Cache, Sessions, Warenkorb
ddev add-on get ddev/ddev-rabbitmq      # Message Queue
ddev add-on get ddev/ddev-adminer       # Datenbank-Oberflaeche
ddev restart
```

So sieht tydace aus — `ddev describe` listet alles auf:

| Dienst | Version | Wofuer |
|---|---|---|
| web | PHP 8.4, nginx-fpm | Shop |
| db | MySQL 8.4 | Datenbank |
| OpenSearch | 3.x | Produktsuche und Indexierung |
| Redis | 7 | Cache, Sessions, Warenkorb, Nummernkreise |
| RabbitMQ | — | Message Queue |
| Adminer | — | Datenbank im Browser |
| Mailpit | — | faengt alle ausgehenden Mails ab |

Ein Add-on installiert den Container — **die Verbindung zu Shopware ist damit noch nicht
hergestellt.** Shopware muss ueber Umgebungsvariablen und Konfigurationsdateien erfahren, dass
es die Dienste nutzen soll:

```yaml
# .ddev/config.shopware.yaml
web_environment:
    - OPENSEARCH_URL=http://opensearch:9200
    - SHOPWARE_ES_ENABLED=1
    - SHOPWARE_ES_INDEXING_ENABLED=1
    - MESSENGER_TRANSPORT_DSN=amqp://rabbitmq:rabbitmq@rabbitmq:5672/%2f/messages
```

> [!NOTE]
> **Warum eine eigene `config.shopware.yaml` und nicht die `.env`?**
> ddev fuehrt alle `.ddev/config.*.yaml` zusammen. Env-Variablen aus dem Container haben
> Vorrang vor der `.env`, damit bleibt die `.env` im Auslieferungszustand des Templates — und
> ist bei Shopware-Updates konfliktfrei. In tydace stehen die Shopware-Variablen deshalb
> gesammelt dort.

Nach dem Aktivieren der Suche einmal indexieren:

```bash
ddev console es:index
```

### Zwei Fallstricke aus tydace

**Redis mit falscher Speicherstrategie macht den Cache still wirkungslos.** Das Redis-Add-on
setzt `maxmemory-policy allkeys-lfu`. Symfonys `RedisTagAwareAdapter`, den Shopware verwendet,
verweigert damit **jeden** Schreibvorgang — ohne sichtbaren Fehler, der Cache ist einfach
wirkungslos. Richtig ist `noeviction` in `.ddev/redis/redis.conf`. Wer die Datei anpasst, muss
die Zeile `#ddev-generated` daraus entfernen, sonst ueberschreibt ddev sie beim naechsten
Add-on-Update.

**Der Admin-Worker ist keine Message Queue.** Im Standard verarbeitet Shopware die Queue ueber
den Browser-Tab der Administration. Das ist bequem und verhaelt sich anders als Produktion. In
tydace ist der Admin-Worker abgeschaltet und stattdessen laeuft ein echter Consumer als Daemon
im Web-Container:

```yaml
# .ddev/config.shopware.yaml
web_extra_daemons:
    - name: messenger-consume
      command: "php bin/console messenger:consume async low_priority --time-limit=120"
      directory: /var/www/html
```

`--time-limit=120` sorgt dafuer, dass der Worker regelmaessig neu startet und geaenderten Code
mitbekommt — sonst debuggt man gegen eine alte Codeversion.

## Ins Repository

Was versioniert gehoert:

```text
.ddev/config.yaml              # Umgebung - der wichtigste Teil
.ddev/config.*.yaml            # Dienst-Konfiguration
custom/static-plugins/         # eigene Plugins und Themes
composer.json, composer.lock   # Abhaengigkeiten
```

Was nicht:

```text
vendor/                        # composer install
public/bundles/, public/theme/ # Build-Artefakte
var/                           # Cache und Logs
custom/plugins/                # Marketplace-Plugins
```

Der Sinn der Uebung: Wer das Repository klont, braucht genau drei Kommandos:

```bash
ddev start
ddev composer install
ddev console system:install --basic-setup
```

## Wenn etwas klemmt

| Symptom | Ursache und Loesung |
|---|---|
| `ddev start` scheitert mit Port-Konflikt | Ein lokaler Apache/nginx belegt Port 80. Beenden, oder `router_http_port` in `~/.ddev/global_config.yaml` aendern |
| Zertifikatswarnung im Browser | `mkcert -install` wurde nicht ausgefuehrt |
| Storefront ohne Styles | `bin/build-storefront.sh` fehlt oder ist fehlgeschlagen |
| Administration bleibt weiss | `bin/build-administration.sh` fehlt; Browser-Konsole zeigt den echten Fehler |
| `Connection refused` zur Datenbank | Host ist `db`, nicht `localhost` — der Shop laeuft im Container |
| Suche findet nichts | `SHOPWARE_ES_ENABLED=1` gesetzt, aber `ddev console es:index` nie gelaufen |
| Aenderungen wirken nicht | `ddev console cache:clear` |

## Weiter

Der Shop laeuft — jetzt geht es an die Arbeit damit:
[Taeglich damit arbeiten]({{< relref "ddev-daily" >}}).
