---
title: How This Site Is Built
weight: 10
aliases:
  - /posts/site-rebuilt-with-hugo-book/
---

# How This Site Is Built

This site runs on [Hugo](https://gohugo.io) with the
[hugo-book](https://github.com/alex-shpak/hugo-book) theme, deployed to GitHub Pages.
This page doubles as a reference for the setup and as a demo of the theme's shortcodes.

## Structure

```text
myl3z.me/
├── hugo.toml              # single config, both languages
├── content.en/            # default language → myl3z.me/
│   ├── docs/              # sidebar tree
│   └── posts/             # dated entries, one folder per project
├── content.de/            # → myl3z.me/de/
├── assets/_custom.scss    # style overrides
├── layouts/posts/         # blog list and feed, see below
├── themes/hugo-book/      # git submodule, pinned to v13
└── .github/workflows/hugo.yml
```

Translations are matched by path: `content.en/docs/tooling/site-setup.md` and
`content.de/docs/tooling/site-setup.md` are the same page in two languages. The language
switcher only appears when a translation actually exists, so nothing has to be written twice.

## Writing

{{% steps %}}

1. ## Create the page
   `hugo new docs/linux/systemd-timers.md` picks up the theme archetype and writes the
   front matter, including the `weight` that controls sidebar order.

2. ## Preview it
   `hugo server -D` serves drafts on `localhost:1313` with live reload.

3. ## Publish it
   Push to `main`. The GitHub Action builds the site and deploys it — no manual step.

{{% /steps %}}

> [!NOTE]
> Pages with `draft: true` are excluded from the production build. The `-D` flag makes them
> visible locally.

## Blog by project

Dated entries live under `content.*/posts/<project>/`, currently `homelab/` and `shop/`. Each
project folder has an `_index.md` with a short description, which gives it its own overview
page and its own RSS feed. `/posts/` stays the umbrella and lists all entries together, newest
first.

For this to work, two templates live in the project instead of the theme:

| File | Why |
|---|---|
| `layouts/posts/list.html` | Renders the `_index.md` text above the list and paginates recursively, so `/posts/` shows entries from every project folder |
| `layouts/posts/list.rss.xml` | Same problem in the feed: Hugo's default only takes direct child pages, which would leave the umbrella feed empty |

A new project is a folder with an `_index.md` — plus a menu entry with `parent = 'blog'` in
`hugo.toml`.

## Local commands

{{< tabs >}}

{{% tab "Development" %}}
```sh
hugo server -D              # drafts included, live reload
hugo server --navigateToChanged
```
{{% /tab %}}

{{% tab "Build" %}}
```sh
hugo --minify --gc          # what the Action runs
hugo --printPathWarnings    # find broken links and stray files
```
{{% /tab %}}

{{% tab "Theme update" %}}
```sh
cd themes/hugo-book
git fetch --tags
git checkout v14           # breaking changes between majors — read the changelog
cd ../..
git commit -am "chore: bump hugo-book to v14"
```
{{% /tab %}}

{{< /tabs >}}

> [!WARNING]
> The theme ships breaking changes between major releases. The submodule is pinned to a tag on
> purpose — updates happen when they are wanted, not on the next clone.
