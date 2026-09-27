---
title: MediCat & Ventoy
weight: 15
---

# MediCat & Ventoy

Fuer die Installation von Windows und Proxmox nutze ich seit einiger Zeit **MediCat USB**.
Die Grundlage dafuer ist **Ventoy**: Mehrere Installationsmedien liegen als ISO-Dateien auf
einem Stick und lassen sich beim Start auswaehlen. Das passt auch zum naechsten Schritt im
[Homelab]({{< relref "/posts/homelab/umbau-proxmox" >}}), in dem beide Server frisch mit
Proxmox aufgesetzt werden sollen.

## Was MediCat und Ventoy jeweils machen

**Ventoy** macht den USB-Stick bootfaehig und stellt das Auswahlmenue fuer die Images bereit.
Nach der einmaligen Einrichtung lassen sich mehrere ISOs auf den Stick kopieren. Fuer eine
weitere Version muss der Stick nicht jedes Mal neu beschrieben werden.
[Ventoy: Funktionsuebersicht](https://www.ventoy.net/en/)

**MediCat USB** baut darauf eine Sammlung fuer Diagnose, Reparatur und Wartung auf. Dazu
gehoeren unter anderem Werkzeuge fuer Backups, Partitionen und Datenrettung sowie
Live-Systeme. Der Stick kann damit sowohl Installationsmedien als auch Werkzeuge fuer den
Fall bereithalten, dass ein Rechner nicht mehr richtig startet.
[MediCat: Ueberblick](https://medicatusb.com/docs/medicat/general/overview/),
[MediCat: Aufbau auf Ventoy](https://medicatusb.com/docs/medicat/installation/manualinstall/)

| Bestandteil | Aufgabe |
|-------------|---------|
| Ventoy | startet das ausgewaehlte Image vom USB-Stick |
| MediCat | ergaenzt die Sammlung aus Wartungs- und Rettungswerkzeugen |
| Windows- oder Proxmox-ISO | liefert den jeweiligen Betriebssystem-Installer |

Wer nur mehrere Installations-ISOs starten moechte, kann dafuer bereits Ventoy allein nutzen.
MediCat erweitert den Stick um die zusaetzlichen Werkzeuge.

## Den Stick vorbereiten

MediCat nennt **mindestens 32 GB** als Voraussetzung und empfiehlt USB 3.0 oder neuer.
Zusaetzliche Windows- und Proxmox-ISOs brauchen weiteren freien Speicher; die Groesse des
Sticks sollte deshalb zur gesamten Sammlung passen.
[MediCat: Voraussetzungen](https://medicatusb.com/docs/medicat/installation/requirements/)

Fuer einen neuen Stick fuehrt der Weg ueber den
[offiziellen MediCat-Download](https://medicatusb.com/docs/medicat/installation/download/)
und die [Installationsanleitung](https://medicatusb.com/docs/medicat/installation/manualinstall/).
Bei der manuellen Einrichtung wird zuerst Ventoy installiert, dann die grosse Datenpartition
mit NTFS formatiert und der Inhalt des MediCat-Archivs in deren Hauptverzeichnis entpackt.
Die separate Ventoy-Bootpartition bleibt dabei erhalten. MediCat besteht hier aus einer
Dateisammlung; das heruntergeladene Archiv ist keine bootfaehige ISO.

> [!WARNING]
> Die erstmalige Einrichtung formatiert den Stick. Vorher dessen Daten sichern und das
> Ziellaufwerk pruefen. Auf einen bereits eingerichteten MediCat-Stick werden neue
> Installations-ISOs nur als Dateien kopiert.

## Windows und Proxmox hinzufuegen

Die Installationsmedien lade ich separat vom jeweiligen Hersteller herunter:

- **Windows:** Bei [Microsofts Windows-11-Download](https://www.microsoft.com/en-us/software-download/windows11)
  die ISO fuer die passende Architektur und Sprache waehlen.
- **Proxmox VE:** Den passenden Installer aus den
  [offiziellen ISO-Downloads](https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso)
  verwenden. Die dort angegebene SHA256-Pruefsumme dient zum Abgleich des Downloads.

Die ISO-Dateien kommen unveraendert auf die grosse Datenpartition des eingerichteten Sticks,
beispielsweise in den mitgelieferten Ordner `OSimages`. Sie werden weder entpackt noch mit
`dd` auf das Geraet geschrieben. Ventoy findet Images auch in Unterordnern. Eine neue
Installer-Version laesst sich spaeter durch Austauschen der entsprechenden Datei aufnehmen.
[Ventoy: Images kopieren](https://www.ventoy.net/en/doc_start.html)

Der Ablauf am Zielrechner ist dann:

1. Den Stick nach dem Kopieren sicher auswerfen und am Zielrechner anschliessen.
2. Im Bootmenue des Rechners den USB-Stick auswaehlen.
3. Im MediCat-/Ventoy-Menue die gewuenschte Windows- oder Proxmox-ISO starten.
4. Dem jeweiligen Installer folgen und dort das vorgesehene interne Ziellaufwerk auswaehlen.

Ab diesem Punkt fuehrt der jeweilige Betriebssystem-Installer durch die Installation.
Der USB-Stick dient als Startmedium; das ausgewaehlte Ziellaufwerk kann bei der Installation
geloescht werden. Was Proxmox anschliessend auf dem Server bereitstellt, steht unter
[Proxmox verstehen]({{< relref "/docs/linux/proxmox" >}}).

## Wenn der Stick nicht startet

Ob ein Image startet, haengt auch von der Firmware, dem Bootmodus und dem Image selbst ab.
Ventoy unterstuetzt Legacy BIOS und UEFI; bei **Secure Boot** koennen zusaetzliche Schritte
wie das Eintragen eines Ventoy-Schluessels erforderlich sein. Die Unterstuetzung funktioniert
nicht auf jeder Hardware gleich. Fuer diesen Fall beschreibt Ventoy den aktuellen Ablauf
in den [Hinweisen zu Secure Boot](https://www.ventoy.net/en/doc_secure.html).

Fuer einen einzelnen Linux-Installer ist weiterhin ein direkt geschriebener Stick moeglich,
wie unter [Boot-Stick im Terminal]({{< relref "/docs/linux/bootable-usb" >}}) beschrieben.
Dafuer einen separaten Stick nehmen: Das Schreiben einer Hybrid-ISO mit `dd` wuerde die
bestehende MediCat-/Ventoy-Einrichtung ueberschreiben.
