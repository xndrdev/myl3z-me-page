---
title: How This Site Is Built
weight: 10
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
│   └── posts/             # dated entries
├── content.de/            # → myl3z.me/de/
├── assets/_custom.scss    # style overrides
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
