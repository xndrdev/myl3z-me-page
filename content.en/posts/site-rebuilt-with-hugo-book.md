---
title: Site Rebuilt With Hugo Book
date: 2026-08-15
---

# Site Rebuilt With Hugo Book

The old version of this site used the `terminal` theme and a flat list of posts. It worked
for a handful of articles and stopped working the moment the notes started referring to each
other.

The rebuild swaps that for [hugo-book](https://github.com/alex-shpak/hugo-book), which is a
documentation theme: a sidebar tree, full-text search and a table of contents per page. Dated
posts still exist, but they are no longer the primary structure — most of what I write is
reference material that gets updated, not published once.

Two things changed alongside the theme:

- **Two languages.** English by default, German under `/de/`. Pages are translated where it
  makes sense; the switcher only shows up when a translation exists.
- **Deploy on push.** A GitHub Action builds and deploys to Pages. No local build step, no
  uploading.

The setup itself is written down under
[How This Site Is Built]({{< relref "/docs/tooling/site-setup" >}}).
