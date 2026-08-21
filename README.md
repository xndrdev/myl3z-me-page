# myl3z.me

Quellcode von [myl3z.me](https://myl3z.me) — Hugo mit dem Theme
[hugo-book](https://github.com/alex-shpak/hugo-book), zweisprachig (EN/DE),
Deployment ueber GitHub Pages.

## Setup

```sh
git clone --recurse-submodules git@github.com:xndrdev/myl3z-me-page.git
cd myl3z-me-page
hugo server -D
```

Ohne `--recurse-submodules` geklont? Dann `git submodule update --init`.

Vorausgesetzt wird Hugo **extended** ab 0.158 (`hugo version` muss `extended` enthalten,
sonst schlaegt das SCSS-Kompilieren fehl). Unter Arch: `sudo pacman -S hugo`.

## Inhalte anlegen

```sh
hugo new docs/linux/systemd-timers.md      # Notiz im Sidebar-Baum (englisch)
hugo new posts/homelab/mein-erster-post.md # datierter Blogeintrag im Projekt homelab
```

Die Archetypes des Themes setzen das Front Matter. Wichtig in `docs/`:

- `weight` steuert die Reihenfolge in der Sidebar
- `bookCollapseSection: true` klappt eine Sektion standardmaessig zu
- `bookToc: false` blendet das Inhaltsverzeichnis rechts aus
- `bookHidden: true` haelt eine Seite aus der Navigation heraus

### Zweisprachigkeit

| Sprache | Verzeichnis | URL |
|---------|-------------|-----|
| Englisch (Standard) | `content.en/` | `myl3z.me/…` |
| Deutsch | `content.de/` | `myl3z.me/de/…` |

Zwei Dateien mit **identischem Pfad** unterhalb ihres Sprachverzeichnisses gelten als
Uebersetzung voneinander. Dank `BookTranslatedOnly = true` erscheint der Sprachumschalter nur
dort, wo die Uebersetzung tatsaechlich existiert — es muss also nicht alles doppelt geschrieben
werden.

## Deployment

Push auf `main` startet `.github/workflows/hugo.yml`: Build mit Hugo extended, Deploy auf
GitHub Pages. In den Repo-Settings unter *Pages* muss **Source: GitHub Actions** stehen.

### DNS fuer die Custom-Domain

Steht bereits, hier nur zur Referenz. Beim Registrar von `myl3z.me` gesetzt:

| Typ | Name | Wert |
|-----|------|------|
| A | `@` | `185.199.108.153` |
| A | `@` | `185.199.109.153` |
| A | `@` | `185.199.110.153` |
| A | `@` | `185.199.111.153` |
| AAAA | `@` | `2606:50c0:8000::153` |
| AAAA | `@` | `2606:50c0:8001::153` |
| AAAA | `@` | `2606:50c0:8002::153` |
| AAAA | `@` | `2606:50c0:8003::153` |
| CNAME | `www` | `xndrdev.github.io.` |

Die Domain und *Enforce HTTPS* sind in den Pages-Settings hinterlegt. Bei Deployment ueber
Actions zieht GitHub die Domain aus den Repo-Settings, nicht aus einer `CNAME`-Datei im Repo —
eine solche Datei im Repo waere wirkungslos.

## Theme aktualisieren

Das Theme haengt als Submodule auf Tag `v13`. Zwischen Major-Releases gibt es Breaking Changes,
deshalb ist es bewusst gepinnt:

```sh
cd themes/hugo-book
git fetch --tags
git checkout v14
cd ../..
git add themes/hugo-book && git commit -m "chore: bump hugo-book to v14"
```

## Anpassungen

- `assets/_custom.scss` — eigene Styles (ueberschreibt die gleichnamige Theme-Datei)
- `assets/_variables.scss` — Theme-Variablen wie Farben und Breiten ueberschreiben
- `layouts/…` — einzelne Partials des Themes gezielt ersetzen, Pfad wie in
  `themes/hugo-book/layouts/`
- `layouts/posts/list.html` und `layouts/posts/list.rss.xml` — Blog-Uebersicht und -Feed.
  Beide lesen rekursiv, damit `/posts/` die Eintraege aus den Projektordnern
  (`posts/homelab/`, `posts/shop/`) gemeinsam zeigt
