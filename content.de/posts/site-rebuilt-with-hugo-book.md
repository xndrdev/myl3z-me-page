---
title: Neue Seite mit Hugo Book
date: 2026-08-15
---

# Neue Seite mit Hugo Book

Die alte Version dieser Seite nutzte das Theme `terminal` und eine flache Liste von Beitraegen.
Das hat fuer eine Handvoll Artikel funktioniert und genau in dem Moment aufgehoert zu
funktionieren, als die Notizen anfingen, sich gegenseitig zu referenzieren.

Der Neuaufbau setzt stattdessen auf [hugo-book](https://github.com/alex-shpak/hugo-book), ein
Doku-Theme: Sidebar-Baum, Volltextsuche und Inhaltsverzeichnis pro Seite. Datierte Beitraege
gibt es weiterhin, sie sind aber nicht mehr die tragende Struktur — das meiste, was ich
schreibe, ist Nachschlagematerial, das aktualisiert und nicht einmalig veroeffentlicht wird.

Zwei Dinge haben sich mit dem Theme geaendert:

- **Zwei Sprachen.** Englisch als Standard, Deutsch unter `/de/`. Uebersetzt wird, wo es sich
  lohnt; der Umschalter erscheint nur bei vorhandener Uebersetzung.
- **Deploy per Push.** Eine GitHub Action baut und deployed auf Pages. Kein lokaler
  Build-Schritt, kein Hochladen.

Das Setup selbst steht unter
[Wie diese Seite gebaut ist]({{< relref "/docs/tooling/site-setup" >}}).
