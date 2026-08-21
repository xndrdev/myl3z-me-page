---
title: Setting Up Shopware 6
weight: 30
---

# Setting Up Shopware 6

From an empty directory to a configured shop. The example is **tydace**, a freshly installed
Shopware 6.7 shop.

Time needed: just under ten minutes, most of it waiting for Composer.

## The Skeleton

### 1. Create the Project

```bash
mkdir tydace && cd tydace
ddev config --project-type=shopware6 --docroot=public
```

`ddev config` writes `.ddev/config.yaml`. The project type `shopware6` sets the matching nginx
configuration and the environment variables Shopware expects; `--docroot=public` tells ddev
where `index.php` lives.

The project name comes from the directory name and determines the later address: `tydace` ->
`https://tydace.ddev.site`.

### 2. Pin the Versions

The generated `.ddev/config.yaml` is mostly comments. The lines that matter — here the state
of tydace:

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
> **Align these versions with the production environment, not with personal taste.** A shop
> running on PHP 8.4 locally and 8.2 on the server surfaces its errors at deployment time. The
> Shopware version sets the ceiling: 6.7 requires at least PHP 8.2.

`disable_settings_management: true` keeps ddev from writing its own configuration files into
the project — the Shopware configuration stays in your own hands.

### 3. Start the Containers

```bash
ddev start
```

The first run downloads the Docker images, which takes a while. After that it is seconds.

## Installing Shopware

### 4. Fetch the Source

```bash
ddev composer create-project shopware/production
```

`shopware/production` is the official production template: Shopware core, administration,
storefront and the directory structure for your own plugins.

For a specific version:

```bash
ddev composer create-project "shopware/production:6.7.13.0"
```

> [!NOTE]
> **`ddev composer`, not `composer`.** The command runs inside the container — with its PHP
> version and extensions. A locally installed Composer uses the machine's PHP version, and
> that rarely matches.

### 5. Install

```bash
ddev console system:install --basic-setup --create-database
```

This creates the database, imports the schema and sets up the sales channel and the
administrator. `--basic-setup` creates the default sales channel and the admin account
`admin` / `shopware`.

If the command hits an existing installation, `--force` helps — **it empties the database**,
so only on a shop whose data is expendable.

### 6. Build the Assets

```bash
ddev exec bin/build-storefront.sh
ddev exec bin/build-administration.sh
```

Both take a few minutes. Without this step the storefront comes up without styles and the
administration stays blank.

### 7. Open It

```bash
ddev launch          # storefront
ddev launch /admin   # administration
```

| What | Address | Credentials |
|---|---|---|
| Storefront | `https://tydace.ddev.site` | — |
| Administration | `https://tydace.ddev.site/admin` | `admin` / `shopware` |
| Mailpit | `ddev mailpit` | — |

That is a complete shop.

## Adding Services

The shop now runs on MySQL and the filesystem cache. For a realistic setup, the services that
run alongside it in production are missing. ddev ships them as add-ons:

```bash
ddev add-on get ddev/ddev-opensearch    # product search
ddev add-on get ddev/ddev-redis         # cache, sessions, cart
ddev add-on get ddev/ddev-rabbitmq      # message queue
ddev add-on get ddev/ddev-adminer       # database UI
ddev restart
```

This is what tydace looks like — `ddev describe` lists all of it:

| Service | Version | What for |
|---|---|---|
| web | PHP 8.4, nginx-fpm | the shop |
| db | MySQL 8.4 | database |
| OpenSearch | 3.x | product search and indexing |
| Redis | 7 | cache, sessions, cart, number ranges |
| RabbitMQ | — | message queue |
| Adminer | — | database in the browser |
| Mailpit | — | catches all outgoing mail |

An add-on installs the container — **it does not connect it to Shopware.** Shopware has to
learn through environment variables and configuration files that it should use those services:

```yaml
# .ddev/config.shopware.yaml
web_environment:
    - OPENSEARCH_URL=http://opensearch:9200
    - SHOPWARE_ES_ENABLED=1
    - SHOPWARE_ES_INDEXING_ENABLED=1
    - MESSENGER_TRANSPORT_DSN=amqp://rabbitmq:rabbitmq@rabbitmq:5672/%2f/messages
```

> [!NOTE]
> **Why a separate `config.shopware.yaml` instead of the `.env`?**
> ddev merges all `.ddev/config.*.yaml` files. Environment variables from the container take
> precedence over the `.env`, which keeps the `.env` in the state the template shipped it in —
> and therefore conflict-free during Shopware updates. That is why tydace keeps its Shopware
> variables together in that file.

After enabling search, index once:

```bash
ddev console es:index
```

### Two Pitfalls From tydace

**Redis with the wrong eviction policy makes the cache silently useless.** The Redis add-on
sets `maxmemory-policy allkeys-lfu`. Symfony's `RedisTagAwareAdapter`, which Shopware uses,
refuses **every** write under that policy — without a visible error, the cache is simply
ineffective. The correct value is `noeviction` in `.ddev/redis/redis.conf`. When editing that
file, remove the `#ddev-generated` line from it, otherwise ddev overwrites it on the next
add-on update.

**The admin worker is not a message queue.** By default Shopware processes the queue through
the browser tab of the administration. That is convenient and behaves differently from
production. In tydace the admin worker is switched off and a real consumer runs as a daemon in
the web container instead:

```yaml
# .ddev/config.shopware.yaml
web_extra_daemons:
    - name: messenger-consume
      command: "php bin/console messenger:consume async low_priority --time-limit=120"
      directory: /var/www/html
```

`--time-limit=120` makes the worker restart regularly and pick up changed code — otherwise you
debug against an old version of it.

## Into the Repository

What belongs under version control:

```text
.ddev/config.yaml              # the environment - the important part
.ddev/config.*.yaml            # service configuration
custom/static-plugins/         # own plugins and themes
composer.json, composer.lock   # dependencies
```

What does not:

```text
vendor/                        # composer install
public/bundles/, public/theme/ # build artifacts
var/                           # cache and logs
custom/plugins/                # marketplace plugins
```

The point of the exercise: whoever clones the repository needs exactly three commands:

```bash
ddev start
ddev composer install
ddev console system:install --basic-setup
```

## When Something Jams

| Symptom | Cause and fix |
|---|---|
| `ddev start` fails with a port conflict | A local Apache/nginx occupies port 80. Stop it, or change `router_http_port` in `~/.ddev/global_config.yaml` |
| Certificate warning in the browser | `mkcert -install` was never run |
| Storefront without styles | `bin/build-storefront.sh` is missing or failed |
| Administration stays blank | `bin/build-administration.sh` is missing; the browser console shows the real error |
| `Connection refused` to the database | The host is `db`, not `localhost` — the shop runs inside the container |
| Search returns nothing | `SHOPWARE_ES_ENABLED=1` is set, but `ddev console es:index` never ran |
| Changes have no effect | `ddev console cache:clear` |

## Next

The shop runs — now for working with it:
[Working With It Daily]({{< relref "ddev-daily" >}}).
