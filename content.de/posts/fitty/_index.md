---
title: Fitty
---

# Fitty

Mit [Fitty](https://github.com/xndrdev/fitty) baue ich mir einen persoenlichen
Fitness-Begleiter fuer iOS und den Browser. Die Idee: ein Chat pro Tag, in dem Essen,
Training, Walking-Pad-Runden und Fragen zum Alltag zusammenkommen. Eine Mahlzeit beschreiben,
ein Foto dazu schicken oder Bewegung eintragen — daraus soll eine nachvollziehbare
Tagesuebersicht entstehen.

## Die Idee

Die KI hilft dabei, Lebensmittel und Aktivitaeten aus Nachrichten zu erfassen und Kalorien
und Naehrwerte abzuschaetzen. Auch Fotos von Mahlzeiten, Naehrwertetiketten oder
Trainingsanzeigen koennen als Eingabe dienen. Schaetzungen bleiben als solche erkennbar;
Eintraege lassen sich nachtraeglich im Chat oder direkt in der Uebersicht korrigieren.

Persoenliche Ziele und Ernaehrungsvorlieben geben den Kontext. Optionale Tagesziele fuer
Kalorien, Protein, Kohlenhydrate und Fett machen den Fortschritt sichtbar. Die Tageswerte
werden aus gespeicherten Eintraegen berechnet, Aktivitaetskalorien separat angezeigt.
Eine eigene Fotohistorie soll helfen, koerperliche Veraenderungen ueber die Zeit zu
vergleichen.

## Tech Stack

| Bereich | Technologie und Aufgabe |
| --- | --- |
| iOS und Web | React Native mit [Expo]({{< relref "/docs/development/expo" >}}), TypeScript und Expo Router — eine gemeinsame Codebasis fuer App und Browser. |
| Backend | Go mit pgx — API, Anwendungslogik und Zugriff auf PostgreSQL. |
| Datenbank | PostgreSQL innerhalb von Supabase — Profile, Chats, Ziele und Tracking-Eintraege. |
| Anmeldung und Bilder | Supabase Auth fuer Login und Sessions, Supabase Storage fuer private Fotos. |
| KI | OpenAI Responses API ueber das Go-Backend — Text- und Bildauswertung mit austauschbarem Modell. |
| Hosting, geplant | Coolify fuer den Betrieb von Weboberflaeche, Backend und Supabase auf eigener Infrastruktur. |

Die KI-Auswertung erfolgt ueber die externe OpenAI API. Dafuer werden die benoetigten
Chat-, Profil- und Bilddaten uebermittelt; die separaten Fortschrittsfotos bleiben davon
ausgenommen.

## Stand und naechste Schritte

Laut [Projekt-README](https://github.com/xndrdev/fitty/blob/main/README.md) sind Login,
Profile, Tageschats, Text- und Foto-Tracking, Tagesziele und der Vergleich von
Fortschrittsfotos bereits umgesetzt. Tests auf einem echten iPhone und das Deployment
ueber Coolify stehen noch aus. Ohne OpenAI-Key bleibt Fitty als Tagebuch nutzbar.

Hier halte ich fest, wie sich die App im Alltag bewaehrt und was beim Weiterbauen dazukommt.
