---
title: Open WebUI
---

# Open WebUI

Ich moechte [Open WebUI](https://openwebui.com/) zu Hause aufsetzen und mit lokalen
KI-Modellen experimentieren. Erst einmal ausprobieren, wie gut das auf meiner Hardware
funktioniert und wofuer ich es im Alltag einsetzen kann. Ein konkretes Ziel steht schon
fest: meine Dokumentation so zugaenglich machen, dass ich Fragen dazu stellen kann und
eine Antwort mit Quelle bekomme.

## Die Idee

Mit dem [Homelab]({{< relref "/posts/homelab" >}}) waechst auch die Menge an Informationen:
Welche Dienste laufen auf welcher Maschine? Welches Geraet gehoert in welches Netz?
Wo liegt die Konfiguration, und warum habe ich sie damals so gesetzt?

Meine Notizen zu [Linux &amp; Server]({{< relref "/docs/linux" >}}) und
[Netzwerk]({{< relref "/docs/network" >}}) sind ein Ausgangspunkt. Dazu sollen weitere
Unterlagen zu Hause kommen. Fragen wie „Wo laeuft mein DNS-Server?“ oder „Wo habe ich die
VLAN-Aufteilung dokumentiert?“ sollen zur passenden Information fuehren — samt Verweis auf
das Dokument und die Textstelle, damit ich die Antwort nachpruefen kann.

## Dokumentation als Wissensbasis

Open WebUI kann Dokumente in einer
[Knowledge Base](https://docs.openwebui.com/features/workspace/knowledge/) sammeln und
fuer Chats verfuegbar machen. Fuer die Suche moechte ich RAG ausprobieren:
*Retrieval-Augmented Generation*. Dabei werden passende Textstellen aus den Unterlagen
gesucht und dem Modell als Kontext fuer die Antwort gegeben. Open WebUI unterstuetzt dazu
[Quellenangaben in den Antworten](https://docs.openwebui.com/features/chat-conversations/rag/).

Fuer mein Ziel „Was ist gerade wo?“ muss die Wissensbasis aktuell bleiben. Aenderungen an
Diensten, Geraeten und Konfigurationen gehoeren deshalb zuerst in die Dokumentation und
muessen anschliessend in Open WebUI uebernommen werden. Wie ich diese Aktualisierung
organisiere, ist Teil des Projekts. Der Chat allein erkennt keinen Umbau im Heimnetz.

## Geplanter Aufbau

| Baustein | Aufgabe |
| --- | --- |
| Open WebUI | Chatoberflaeche und Verwaltung der Wissensbasis. |
| Ollama mit einem lokalen Sprachmodell | Antworten auf eigener Hardware erzeugen; das passende Modell will ich ausprobieren. |
| Lokales Embedding-Modell und RAG | Dokumente fuer die inhaltliche Suche aufbereiten und passende Textstellen finden. |
| Docker Compose | Das lokale Setup mit dauerhaft gespeicherten Daten betreiben. |

Der [offizielle Quick Start](https://docs.openwebui.com/getting-started/quick-start/)
beschreibt den Betrieb mit Docker Compose und die Anbindung an Ollama. Fuer mein Setup
sollen sowohl die Antworten als auch die Dokumentverarbeitung lokal laufen.

## Der erste Versuch

Ich starte mit wenigen gut gepflegten Dokumenten und Fragen, deren Antwort ich kenne.
Damit pruefe ich, ob die richtige Stelle gefunden wird, ob die Quelle die Antwort traegt
und wie das Modell mit fehlenden oder widerspruechlichen Angaben umgeht. Mein Ziel ist,
dass es fehlendes Wissen offen benennt.

Das Projekt ist noch in Planung. Hier werden Aufbau, Versuche und die Einsatzmoeglichkeiten
festgehalten, die sich dabei ergeben.
