---
title: Wie diese Seite gebaut ist
weight: 10
---

# Wie diese Seite gebaut ist

Die Seite laeuft auf [Hugo](https://gohugo.io) mit dem Theme
[hugo-book](https://github.com/alex-shpak/hugo-book) und wird auf GitHub Pages deployed.
Diese Seite ist gleichzeitig Referenz fuer das Setup und Demo der Theme-Shortcodes.

## Struktur

```text
myl3z.me/
├── hugo.toml              # eine Config, beide Sprachen
├── content.en/            # Standardsprache → myl3z.me/
│   ├── docs/              # Sidebar-Baum
│   └── posts/             # datierte Eintraege
├── content.de/            # → myl3z.me/de/
├── assets/_custom.scss    # eigene Styles
├── themes/hugo-book/      # Git-Submodule, gepinnt auf v13
└── .github/workflows/hugo.yml
```

Uebersetzungen werden ueber den Pfad zugeordnet: `content.en/docs/tooling/site-setup.md` und
`content.de/docs/tooling/site-setup.md` sind dieselbe Seite in zwei Sprachen. Der
Sprachumschalter erscheint nur, wenn die Uebersetzung existiert — es muss also nicht alles
doppelt geschrieben werden.

## Schreiben

{{% steps %}}

1. ## Seite anlegen
   `hugo new docs/linux/systemd-timers.md` nutzt den Archetype des Themes und schreibt das
   Front Matter inklusive `weight`, das die Reihenfolge in der Sidebar steuert.

2. ## Vorschau
   `hugo server -D` liefert die Seite inklusive Entwuerfe unter `localhost:1313` mit Live-Reload.

3. ## Veroeffentlichen
   Push auf `main`. Die GitHub Action baut und deployed — kein manueller Schritt.

{{% /steps %}}

{{% hint info %}}
Seiten mit `draft: true` fehlen im Produktions-Build. Der Schalter `-D` macht sie lokal sichtbar.
{{% /hint %}}

## Lokale Befehle

{{< tabs >}}

{{% tab "Entwicklung" %}}
```sh
hugo server -D              # inkl. Entwuerfe, Live-Reload
hugo server --navigateToChanged
```
{{% /tab %}}

{{% tab "Build" %}}
```sh
hugo --minify --gc          # das laeuft in der Action
hugo --printPathWarnings    # findet kaputte Links und verirrte Dateien
```
{{% /tab %}}

{{% tab "Theme-Update" %}}
```sh
cd themes/hugo-book
git fetch --tags
git checkout v14           # Breaking Changes zwischen Majors — Changelog lesen
cd ../..
git commit -am "chore: bump hugo-book to v14"
```
{{% /tab %}}

{{< /tabs >}}

{{% hint warning %}}
Das Theme bringt Breaking Changes zwischen Major-Releases. Das Submodule ist bewusst auf einen
Tag gepinnt — Updates passieren, wenn sie gewollt sind, nicht beim naechsten Klon.
{{% /hint %}}
