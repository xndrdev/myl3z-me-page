---
title: Omarchy auf dem Laptop
date: 2026-10-12
---

Mein kleines Linux-Side-Project hat eine andere Richtung bekommen: Auf dem Laptop
ist inzwischen **Omarchy** installiert. Urspruenglich wollte ich Arch Linux
aufsetzen, habe mich dann aber umentschieden. Die Arch-Basis bleibt dabei erhalten;
mit Omarchy kommt eine bereits eingerichtete Desktop-Umgebung dazu.

## Was Omarchy mitbringt

[Omarchy](https://omarchy.org/manual/) ist eine Linux-Distribution auf Basis von
Arch Linux, mit Hyprland als Fenstermanager und einer Auswahl vorkonfigurierter
Programme. Das Projekt stammt von David Heinemeier Hansson, kurz DHH. Die Idee
dahinter ist ein System, bei dem viele Entscheidungen ueber Programme, Bedienung
und Gestaltung bereits getroffen sind.

Omarchy nennt diesen Ansatz *Omakase*: Man startet mit einer zusammengestellten
Auswahl und kann sie anschliessend anpassen. Dazu gehoeren Anwendungen fuer den
Alltag und die Entwicklung, etwa Chromium, Neovim und LibreOffice. Die
[Projektseite](https://omarchy.org/) beschreibt auch die gemeinsamen Themes,
die unter anderem Terminal, Leiste, Benachrichtigungen und Hintergrund aufeinander
abstimmen.

Fuer mein Side Project ist das ein konkreter Ausgangspunkt. Ich kann mit der
vorbereiteten Umgebung anfangen und beim Benutzen herausfinden, welche
Einstellungen und Programme fuer mich passen.

## Hyprland und die Bedienung mit der Tastatur

Ein zentraler Bestandteil ist **Hyprland**. Der Fenstermanager ordnet Fenster im
Tiling-Modus automatisch auf dem Bildschirm an. Browser und Terminal koennen
dadurch direkt nebeneinander stehen; weitere Fenster werden in das Layout
eingefuegt. Ueber Arbeitsflaechen lassen sich verschiedene Aufgaben trennen.

Omarchy setzt dabei stark auf Tastaturkuerzel: Programme starten, zwischen
Fenstern wechseln, Fenster verschieben und Arbeitsflaechen aufrufen gehoeren zum
normalen Ablauf. Freischwebende Fenster sind ebenfalls moeglich. Die
[Einfuehrung zur Navigation](https://omarchy.org/manual/navigation/) zeigt diese
Bedienung mit Beispielen.

Das wird ein Teil des Ausprobierens auf dem Laptop: die wichtigsten Kuerzel
lernen und herausfinden, wie gut diese Arbeitsweise fuer mich funktioniert.

## Arch Linux bleibt die Grundlage

Unter der vorbereiteten Oberflaeche steckt weiterhin Arch. Damit bleiben auch
die Paketverwaltung mit **pacman** und das grundlegende **Rolling-Release-Modell**
relevant: Software wird laufend aktualisiert, ohne dass fuer jede neue
Hauptversion der Basisdistribution eine Neuinstallation ansteht.
[Arch beschreibt diese Grundlagen auf seiner Projektseite](https://archlinux.org/about/).

Omarchy bringt zusaetzlich eigene Releases und einen abgestimmten Update-Ablauf
mit. Dieser aktualisiert Pakete und fuehrt auch die zugehoerigen Anpassungen an
der Omarchy-Konfiguration aus. Fuer Systemupdates ist deshalb die
Update-Funktion von Omarchy vorgesehen. Wie Pakete, Migrationen und Snapshots
dabei zusammenspielen, steht im
[Handbuch zu Updates](https://omarchy.org/manual/updates/).

## Vom Bootstick zur Installation

Fuer die Vorbereitung hatte ich bereits nach meiner Anleitung
[Boot-Stick im Terminal]({{< relref "/docs/linux/bootable-usb" >}}) einen
Installationsstick erstellt. Dort ist beschrieben, wie eine ISO mit `lsblk`,
`umount` und `dd` auf den passenden USB-Stick geschrieben wird.

Omarchy bietet fuer die Installation eine eigene ISO an. Der im
[offiziellen Getting Started](https://omarchy.org/manual/getting-started/)
beschriebene Weg fuehrt ueber einen USB-Stick und einen Installer, der nach der
Konfiguration das System samt Desktop einrichtet. Dafuer wird das
Omarchy-Installationsmedium verwendet.

Bei mir ist die Installation inzwischen abgeschlossen. Damit geht es jetzt um
die Einrichtung und die ersten Erfahrungen im Alltag.

## Die naechsten Schritte auf dem Laptop

Als Naechstes moechte ich mich mit der Bedienung vertraut machen und meine
Arbeitsumgebung einrichten. Auf einem Laptop interessieren mich dabei auch die
praktischen Dinge: Wie verhaelt sich das System beim Zuklappen, wie gut passt die
Darstellung zum Display und wie schlaegt es sich im Akkubetrieb?

Diese Erfahrungen und die Anpassungen, die sich daraus ergeben, halte ich hier
nach und nach fest. Der Anfang steht: Omarchy ist installiert, jetzt wird es
benutzt.
