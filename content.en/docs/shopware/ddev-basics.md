---
title: What ddev Is
weight: 10
---

# What ddev Is

ddev is an open-source tool that creates and manages local development environments in Docker
containers. Instead of installing and maintaining PHP, MySQL, Node and nginx on your own
machine, you describe in a single file what a project needs — and ddev turns that into a
running environment.

In practice: `ddev start` inside a project directory, and a few seconds later the shop is
served over HTTPS.

## The Problem It Solves

Without a tool like this, every developer ends up with a different environment. One shop wants
PHP 8.2, the next one 8.4. The server runs MySQL 8.4, your machine runs MariaDB 10.6. A
colleague has a different Node version, so their storefront build looks different. Everyone
knows how that ends: "works on my machine."

ddev turns this around. The environment belongs to the project, not to the machine:

- **Versions live in the project.** `.ddev/config.yaml` pins the PHP, database and Node
  version. The file is in the Git repository — whoever clones the project gets the same
  environment automatically.
- **Projects do not interfere.** Twenty shops across five PHP versions can coexist, each in
  its own containers.
- **The machine stays clean.** No Homebrew PHP, no local MySQL installation, no Node version
  manager. Just Docker and ddev.
- **Disposable.** `ddev delete` removes containers and database while the project directory
  stays. A broken shop is no longer a lost day.

## How It Works

At heart, ddev is a generator for Docker Compose files. From the configuration it creates a
set of containers and wires them together:

| Container | Role |
|---|---|
| **web** | nginx or Apache, PHP-FPM, Node, Composer — this is where the shop runs |
| **db** | MySQL, MariaDB or PostgreSQL in the configured version |
| **ddev-router** | A central reverse proxy for *all* projects, routing by hostname and terminating HTTPS |
| **ddev-ssh-agent** | Passes the host's SSH key into the containers, for private Composer repositories |

Router and SSH agent run once for all projects, `web` and `db` once per project.

Two things make daily work pleasant:

**The project directory is mounted into the container.** A file you save in your editor is
immediately there inside the container. No deployment, no sync — the container is not a black
box here, just a runtime wrapped around your files.

**HTTPS simply works.** ddev assigns the address `https://<projectname>.ddev.site` and issues
a locally trusted certificate via [mkcert](https://github.com/FiloSottile/mkcert). No
certificate warning, no `/etc/hosts` entries — the `ddev.site` domain resolves to `127.0.0.1`
by wildcard DNS anyway.

For Shopware, HTTPS is not a luxury: payment methods, service workers and several admin
features behave differently over `http://` than they do in production.

## What Comes Included

A lot of what you would otherwise set up separately is built in:

- **Mailpit** catches every mail the shop sends. Order confirmations end up in a web interface
  instead of in real inboxes.
- **Xdebug** is enabled with `ddev xdebug on` — deliberately not permanently, it costs
  noticeable performance.
- **Database import and export** through `ddev import-db` and `ddev export-db`.
- **Snapshots**: `ddev snapshot` freezes the database state, `ddev snapshot restore` brings it
  back. Handy before risky migrations.
- **Add-ons** for extra services — OpenSearch, Redis, RabbitMQ and more are added with
  `ddev add-on get`.

## Why ddev and Not Something Else

There are alternatives: hand-written Docker Compose, Lando, competitors such as Warden, or
Shopware's own Nix-based `devenv`.

What speaks for ddev:

- **Shopware is a supported project type.** `type: shopware6` sets docroot, nginx
  configuration and the environment variables to match.
- **Getting started is quick.** Maintaining your own Docker Compose setup means writing and
  maintaining the same 200 lines in every project.
- **It is common in the Shopware world.** Which means ready-made add-ons for the services a
  shop needs, and answers to most questions.

The honest counterpoint: ddev does **not** replicate the production environment exactly. It is
a development environment — convenient, fast, full of debugging tools. Testing whether a
deployment works on the target server needs a staging system, not a local simulation.

## Next

Up next: [Installing ddev]({{< relref "ddev-install" >}}).
