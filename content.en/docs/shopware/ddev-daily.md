---
title: Working With It Daily
weight: 40
---

# Working With It Daily

The shop runs. What matters now are the ten commands you genuinely use every day — and a few
habits that save a lot of time.

## The Ground Rule

**Anything concerning the shop runs inside the container.** So `ddev composer` instead of
`composer`, `ddev console` instead of `bin/console`, `ddev npm` instead of `npm`. The reason is
always the same: inside the container the project's PHP and Node versions apply, on the
machine those of the machine.

```bash
ddev composer require tydace/example-plugin
ddev console cache:clear
ddev php -v
ddev npm install
```

## Starting and Stopping

```bash
ddev start        # bring the project up
ddev stop         # this project only
ddev restart      # after changes to .ddev/config.yaml
ddev poweroff     # stop all projects and the router
ddev describe     # status, URLs, ports, credentials
ddev list         # all projects on the machine
```

`ddev poweroff` is the remedy for most strange states — and for port conflicts when another
project is still running.

Containers only need to run while you use them. They cost memory; two dozen started shops slow
the machine down noticeably.

## The Shopware Console

```bash
ddev console cache:clear
ddev console plugin:list
ddev console plugin:refresh
ddev console plugin:install --activate TydaceExamplePlugin
ddev console database:migrate --all TydaceExamplePlugin
ddev console dal:refresh:index
ddev console theme:compile
ddev console messenger:stats
```

> [!NOTE]
> `ddev console` is not a built-in command but a custom one — see
> [Custom Commands](#custom-commands) further down. Without that shortcut every call would
> read `ddev exec bin/console cache:clear`.

## Building the Frontend

A full build takes minutes and is the wrong approach while developing:

```bash
ddev sw-build              # storefront and administration
ddev sw-build storefront   # storefront only
ddev sw-build admin        # administration only
```

While working, use the watchers instead — they recompile on every saved file and reload the
page automatically:

```bash
ddev sw-watch storefront   # https://tydace.ddev.site:9998
ddev sw-watch admin        # https://tydace.ddev.site:5173
```

> [!WARNING]
> The storefront watcher runs on its own port and bypasses the HTTP cache. **Whatever works
> there has to be verified again on the normal address after a real `sw-build`.** Twig blocks
> and theme variables in particular behave differently.

After plugin changes, one command cleans up everything at once:

```bash
ddev sw-refresh
```

It clears the cache, refreshes the plugin list, compiles the theme and triggers DAL indexing
through the queue.

## Database

```bash
ddev mysql                              # MySQL client
ddev import-db --file=dump.sql.gz       # import a dump
ddev export-db --file=dump.sql.gz       # export a dump
ddev adminer                            # database in the browser
```

The most practical part is snapshots:

```bash
ddev snapshot                           # save the state
ddev snapshot --list
ddev snapshot restore --latest          # bring it back
```

**Take a snapshot before every migration, every import and every risky console command.** It
costs seconds and replaces rebuilding the shop when something goes wrong.

## Debugging

```bash
ddev xdebug on      # enable
ddev xdebug off     # disable
ddev xdebug status
```

Xdebug costs noticeable performance, which is why it is off by default. Listen on port 9003 in
the editor, set the path mapping `/var/www/html` -> project directory, done.

More useful views:

```bash
ddev logs -f                # live web server logs
ddev logs -s db             # database logs
ddev ssh                    # shell in the web container
ddev mailpit                # intercepted mail
ddev redis-cli              # Redis console (add-on)
ddev rabbitmqctl list_queues # queue state (add-on)
```

For Shopware errors the Symfony log is usually more informative than the web server logs:

```bash
ddev exec tail -f var/log/dev.log
```

## Custom Commands

This is where ddev turns from a tool into a project tool: frequent routines can be stored as
commands. An executable file under `.ddev/commands/web/` is enough — it runs in the web
container.

`.ddev/commands/web/console` from tydace:

```bash
#!/bin/bash

## Description: Call the Shopware console (bin/console)
## Usage: console [args]
## Example: "ddev console cache:clear", "ddev console plugin:list"

php /var/www/html/bin/console "$@"
```

Do not forget to make it executable:

```bash
chmod +x .ddev/commands/web/console
```

From then on `ddev console` is available to everyone who clones the project — the commands
live in the repository. The `## Description` line shows up in `ddev help`.

Commands meant to run on the **host** (something that opens a browser, say) belong in
`.ddev/commands/host/`.

tydace has four of them: `console`, `sw-build`, `sw-watch` and `sw-refresh`. They replace
command chains you would otherwise retype every day.

## Creating a New Plugin

The production template already registers `custom/static-plugins/*` as a Composer `path`
repository:

```bash
mkdir -p custom/static-plugins/TydaceExample
# create a composer.json with "name": "tydace/example" and
# "type": "shopware-platform-plugin"
ddev composer require tydace/example
ddev console plugin:refresh
ddev console plugin:install --activate TydaceExample
```

Composer creates a symlink under `vendor/`. Changes to the plugin take effect immediately,
without another `composer install`.

Where things go:

- `custom/static-plugins/` — own plugins and themes, **version controlled**
- `custom/plugins/` — third-party and marketplace plugins, **gitignored**

## Cleaning Up

```bash
ddev delete           # containers and database gone, files stay
ddev delete --omit-snapshot   # without a backup copy of the database
ddev clean            # remove unused Docker images
```

`ddev delete` sounds more drastic than it is: the project directory stays untouched and a
`ddev start` rebuilds the environment. Only the data is gone — which is why ddev takes a
snapshot beforehand automatically.

Docker images add up to many gigabytes over time. Running `ddev clean` occasionally frees
space.

## Worth Memorising

| Situation | Command |
|---|---|
| Something is off | `ddev restart` |
| Still off | `ddev poweroff && ddev start` |
| A change has no effect | `ddev console cache:clear` |
| A plugin does not show up | `ddev console plugin:refresh` |
| Before something risky | `ddev snapshot` |
| After plugin changes | `ddev sw-refresh` |
| No idea where something runs | `ddev describe` |
