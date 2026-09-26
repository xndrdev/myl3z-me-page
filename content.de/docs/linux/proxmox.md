---
title: Proxmox verstehen
weight: 90
---

# Proxmox verstehen

Auf die [beiden Server im Homelab]({{< relref "/posts/homelab/umbau-proxmox" >}}) soll
Proxmox. Dabei interessiert mich auch, was hinter dem Installer steckt: Wenn das auf Debian
basiert, wo hoert Debian auf und wo faengt Proxmox an? Und koennte ich selbst etwas in dieser
Richtung bauen?

Ja: Eine kleine eigene Debian-Variante ist ein erreichbares Lernprojekt. Um zu verstehen,
welche Arbeit darueber hinaus in Proxmox steckt, hilft ein Blick auf die einzelnen Teile.

## Was Proxmox VE ist

Gemeint ist **Proxmox Virtual Environment**, kurz **PVE**: eine auf Debian basierende
Plattform fuer virtuelle Maschinen und Linux-Container. Sie wird normalerweise direkt auf
dem Server installiert. Der physische Rechner heisst **Host** oder **Node**, die darauf
laufenden virtuellen Systeme sind seine **Gaeste**. Die Verwaltung erfolgt unter anderem
ueber eine Weboberflaeche. Proxmox verbindet dafuer bestehende Virtualisierungstechnik mit
eigener Verwaltungssoftware. [Proxmox: Funktionsuebersicht](https://proxmox.com/en/products/proxmox-virtual-environment/features)

Die Versionsnummern gehoeren zu unterschiedlichen Projekten: **Proxmox VE 9 basiert auf
Debian 13 „Trixie“**. Proxmox liefert einen eigenen, angepassten Linux-Kernel mit. Das ist
weiterhin Linux; die Proxmox-Entwickler waehlen und pflegen eine fuer ihre Plattform passende
Variante. [Proxmox VE 9.0: technische Basis](https://www.proxmox.com/en/about/company-details/press-releases/proxmox-virtual-environment-9-0)

## Virtuelle Maschine und Container

Eine **virtuelle Maschine** bekommt virtuelle Hardware: Prozessoren, Arbeitsspeicher,
Festplatten und Netzwerkkarten. Darauf startet ein eigenes Betriebssystem mit eigenem Kernel.
**QEMU** stellt die virtuelle Maschine bereit; **KVM** im Linux-Kernel nutzt die
Virtualisierungsfunktionen der CPU zur Beschleunigung. Der Gast sieht beispielsweise ein
DVD-Laufwerk, obwohl dahinter nur eine ISO-Datei liegt.
[QEMU: Systememulation und Beschleuniger](https://www.qemu.org/docs/master/system/introduction.html),
[Proxmox: QEMU/KVM](https://github.com/proxmox/pve-docs/blob/master/qm.adoc)

Ein **LXC-Container** teilt sich dagegen den Kernel mit dem Host. Linux grenzt seine Prozesse
und deren Sicht auf das System ab; sogenannte *cgroups* regeln den Zugriff auf Ressourcen.
Der Container hat seine eigenen Programme und Systemdateien, bootet aber keinen eigenen
Kernel. [Proxmox: Container und Isolation](https://github.com/proxmox/pve-docs/blob/master/pct.adoc)

| Merkmal | Virtuelle Maschine | LXC-Container |
|---|---|---|
| Kernel | eigener Gast-Kernel | Kernel des Hosts |
| Betriebssystem | beispielsweise Linux oder Windows | Linux-Userspace, etwa Debian |
| Ressourcenbedarf | zusaetzliches vollstaendiges Gastsystem | meist geringer |
| Passendes Lernprojekt | eine selbst gebaute Linux-ISO booten | einen Linux-Dienst mit eigenen Paketen betreiben |

Fuer das Experiment mit einer eigenen Distribution ist deshalb eine VM passend: Dort laesst
sich auch der Bootvorgang des eigenen Images ausprobieren.

## Wie die Verwaltung aufgebaut ist

Die Weboberflaeche fuehrt die Gaeste nicht selbst aus. Dahinter arbeiten Dienste mit
unterschiedlichen Aufgaben:

| Bestandteil | Aufgabe |
|-------------|---------|
| Weboberflaeche und API | nehmen Verwaltungsauftraege entgegen |
| `pveproxy` | stellt den HTTPS-Zugang auf Port `8006` bereit |
| `pvedaemon` | bearbeitet API-Auftraege, fuer die hoehere Rechte noetig sind |
| QEMU/KVM und LXC | fuehren die virtuellen Maschinen beziehungsweise Container aus |
| `pmxcfs` unter `/etc/pve` | stellt Proxmox-Konfigurationsdateien bereit und verteilt sie im Cluster |

`pveproxy` reicht privilegierte Auftraege an den lokalen `pvedaemon` weiter. `pmxcfs` speichert
Konfigurationen, etwa die Einstellungen einer VM; deren virtuelle Festplatte ist davon
getrennt. [API-Zugang](https://github.com/proxmox/pve-docs/blob/master/pveproxy.adoc),
[API-Dienst](https://pve.proxmox.com/pve-docs/pvedaemon.8.html),
[Konfigurationsdateisystem](https://github.com/proxmox/pve-docs/blob/master/pmxcfs.adoc)

Ein Klick auf **„VM starten“** laesst sich vereinfacht so verfolgen:

1. Der Browser schickt einen authentifizierten API-Auftrag an den Host.
2. Die Verwaltung prueft die Berechtigung und liest die VM-Konfiguration.
3. Sie startet QEMU mit den vorgesehenen Ressourcen, Laufwerken und Netzwerkanschluessen.
4. Der Gast bootet sein eigenes Betriebssystem mithilfe von QEMU und KVM.

Die Oberflaeche beschreibt und steuert also den Betrieb. Die eigentliche Ausfuehrung liegt
bei den Virtualisierungskomponenten.
[Proxmox: API und Weboberflaeche](https://github.com/proxmox/pve-manager),
[VM-Konfiguration und Ausfuehrung](https://github.com/proxmox/pve-docs/blob/master/qm.adoc)

### Speicher, Netzwerk und mehrere Hosts

Virtuelle Festplatten liegen in einem konfigurierten **Storage**. Je nach Backend sind das
beispielsweise Image-Dateien in einem Verzeichnis oder logische Volumes. Proxmox bietet eine
gemeinsame Verwaltung fuer diese unterschiedlichen Speicherarten; welche davon verwendet
wird, bleibt eine Entscheidung beim Aufbau des Hosts.
[Proxmox: Storage Manager](https://github.com/proxmox/pve-docs/blob/master/pvesm.adoc)

Eine Linux-Bridge wie `vmbr0` funktioniert wie ein virtueller Switch. An ihr koennen die
virtuellen Netzwerkkarten der Gaeste und eine physische Netzwerkkarte des Hosts haengen.
So gelangen die Gaeste ins Heimnetz. VLANs lassen sich dabei passend zum
[vorhandenen Netz-Layout]({{< relref "/docs/network/vlan-layout" >}}) zuordnen.
[Proxmox: Netzwerkaufbau](https://github.com/proxmox/pve-docs/blob/master/pve-network.adoc)

Mehrere Hosts lassen sich zu einem **Cluster** fuer die gemeinsame Verwaltung verbinden.
Zwei frisch installierte Server bilden noch keinen Cluster. Fuer hochverfuegbare Gaeste
kommen weitere Entscheidungen hinzu, etwa ueber Speicher und Quorum: die erforderliche
Mehrheit der Stimmen im Cluster. Gemeinsame Verwaltung und automatische Wiederherstellung
nach einem Host-Ausfall sind unterschiedliche Funktionen.
[Proxmox: Cluster Manager](https://github.com/proxmox/pve-docs/blob/master/pvecm.adoc)

## Wie aus Debian Proxmox wird

Der **Kernel** verwaltet unter anderem Prozessor, Speicher und Hardware. Eine **Distribution**
stellt darum ein benutzbares System zusammen: Programme, Bibliotheken, Paketverwaltung,
Voreinstellungen und einen Weg, alles zu installieren und zu aktualisieren. Debian leistet
diese Arbeit bereits fuer eine grosse Auswahl an Software.
[Debian: Was ist Debian?](https://www.debian.org/intro/about)

Eine darauf aufbauende Distribution kann grosse Teile uebernehmen und sich auf ihren eigenen
Zweck konzentrieren. Debian bezeichnet solche eigenstaendigen Varianten als **Derivate** und
unterstuetzt ausdruecklich, dass sie entstehen.
[Debian: Derivate](https://www.debian.org/derivatives/)

Bei Proxmox laesst sich das an drei Stellen nachvollziehen:

- **Eigene Software:** Im Repository `pve-manager` liegen unter anderem die Verwaltung,
  die Weboberflaeche, Dienstdefinitionen und unter `debian/` die Dateien fuer die
  Debian-Paketierung. Aus Quellcode werden installierbare Pakete.
  [Quellcode von pve-manager](https://github.com/proxmox/pve-manager)
- **Eigene Paketquellen:** Das installierte System bezieht Debian-Pakete und
  Proxmox-Pakete aus den jeweils passenden Repositories. APT kann so beide Teile
  aktualisieren. [Proxmox: Paketquellen](https://github.com/proxmox/pve-docs/blob/master/pve-package-repos.adoc)
- **Ein gemeinsamer Installer:** Das Installationsmedium richtet daraus ein System fuer
  den vorgesehenen Einsatzzweck ein. Es bringt unter anderem Debian, den Proxmox-Kernel und
  die Verwaltungswerkzeuge zusammen.
  [Proxmox: Installation](https://proxmox.com/en/products/proxmox-virtual-environment/get-started)

Die Entwicklungsarbeit umfasst damit die Verbindung und Pflege dieser Teile: Eine neue
Version muss mit bestehenden Konfigurationen umgehen, Updates muessen zusammenpassen, und
Fehler muessen sich nachvollziehen lassen. Ein eigenes Logo auf einer ISO waere nur ein
kleiner Teil davon.

## Was eine eigene kleine Distribution bedeuten kann

Fuer ein persoenliches Projekt wuerde ich den Umfang stufenweise vergroessern:

| Stufe | Eigenes Ergebnis |
|-------|------------------|
| Debian automatisch einrichten | eine festgehaltene Paketauswahl und Konfiguration fuer frische Installationen |
| Eigenes Live-Image | eine bootfaehige ISO mit diesen Paketen und eigenen Dateien |
| Gepflegtes Debian-Derivat | zusaetzlich eigene Pakete, Releases, Paketquellen und einen getesteten Updateweg |

Schon die erste Stufe kann fuer das Homelab reichen. Fuer ein eigenes bootfaehiges System
ist die zweite ein ueberschaubarer Einstieg. Debian liefert mit **live-build** das Werkzeug,
um ein Image aus einer beschriebenen Konfiguration zusammenzustellen.
[Debian Live: Werkzeuge und Grundlagen](https://live-team.pages.debian.net/live-manual/html/live-manual/the-basics.en.html)

Wer stattdessen lernen moechte, wie Compiler, Bibliotheken und Grundsystem aus Quellcode
zusammengebaut werden, kann **Linux From Scratch** durcharbeiten. Das ist ein anderer
Lernweg; fuer eine persoenliche Debian-Variante muss man diesen Unterbau nicht selbst bauen.
[Linux From Scratch](https://www.linuxfromscratch.org/lfs/view/stable/)

## Erstes Lernprojekt: ein eigenes Debian-Live-Image

Als Beispiel soll eine kleine Konsole mit `curl`, `htop` und `tmux` entstehen. Dazu kommt eine
eigene Textdatei, an der sich das Image erkennen laesst. Die folgende Anleitung ist fuer eine
**separate Debian-13-VM auf amd64 mit Internetzugang und eingerichtetem `sudo`** gedacht.
Der Build braucht mehrere GB freien Speicherplatz. Auf dem Proxmox-Host selbst ist dafuer
keine Installation noetig.

### Das Werkzeug und die Konfiguration

`live-build` wird in der Build-VM aus Debian installiert.
[Debian Live: Installation](https://live-team.pages.debian.net/live-manual/html/live-manual/installation.en.html)

```sh
sudo apt update
sudo apt install live-build

mkdir xlab-live
cd xlab-live

lb config --distribution trixie --architectures amd64 --binary-images iso-hybrid --debian-installer none
```

Das legt `config/` an. Der Codename `trixie` haelt die gewaehlte Debian-Version fest;
`iso-hybrid` bestimmt das Ausgabeformat. Mit `--debian-installer none` wird explizit ein
Live-System ohne Installer gebaut.
[lb_config fuer Debian 13](https://manpages.debian.org/trixie/live-build/lb_config.1.en.html)

### Eigene Pakete und Dateien

```sh
mkdir -p config/package-lists config/includes.chroot/etc

printf '%s\n' curl htop tmux > config/package-lists/xlab.list.chroot
printf '%s\n' 'xlab-live: mein erstes Debian-Image' > config/includes.chroot/etc/xlab-release

sudo lb build
```

Die Datei mit der Endung `.list.chroot` bestimmt die zusaetzlichen Pakete.
`config/includes.chroot/` entspricht dem spaeteren Wurzelverzeichnis: Aus der eigenen Datei
wird im Live-System `/etc/xlab-release`.
[Paketauswahl](https://live-team.pages.debian.net/live-manual/html/live-manual/customizing-package-installation.en.html),
[Eigene Dateien](https://live-team.pages.debian.net/live-manual/html/live-manual/customizing-contents.en.html)

### Was beim Bauen passiert

Das Werkzeug baut ein Debian-Grundsystem in einem Arbeitsverzeichnis auf; dafuer dient
`debootstrap`. Dazu kommen die ausgewaehlten Pakete und Dateien. Fuer das Booten werden
Kernel, ein fruehes Startsystem namens **initramfs** und ein Bootloader eingebunden. Beim
Start findet `live-boot` das enthaltene Root-Dateisystem und macht es mit einer beschreibbaren
Zusatzschicht nutzbar. `live-config` richtet anschliessend unter anderem den Live-Benutzer ein.
[debootstrap](https://manpages.debian.org/trixie/debootstrap/debootstrap.8.en.html),
[Debian Live: Zusammenspiel der Werkzeuge](https://live-team.pages.debian.net/live-manual/html/live-manual/overview-of-tools.en.html)

### Das Ergebnis ausprobieren

Nach erfolgreichem Build liegt `live-image-amd64.hybrid.iso` im Arbeitsverzeichnis. Die ISO
laesst sich als virtuelles DVD-Laufwerk an eine neue Test-VM haengen und dort booten.
[Debian Live: Bauen und Testen](https://live-team.pages.debian.net/live-manual/html/live-manual/the-basics.en.html)

In der gebooteten Test-VM:

```sh
cat /etc/xlab-release
command -v curl htop tmux
```

Das ist ein eigenes zusammengestelltes Debian-Live-System. Aenderungen waehrend der Sitzung
gehen ohne gesondert eingerichtete Persistenz beim Neustart verloren. Fuer die Installation
auf Festplatte waere ein weiterer Schritt mit einem Installer noetig.
[Debian Live: Persistenz](https://live-team.pages.debian.net/live-manual/html/live-manual/customizing-run-time-behaviours.en.html),
[Debian Live: Installer](https://live-team.pages.debian.net/live-manual/html/live-manual/customizing-installer.en.html)

## Was danach zur eigenen Distribution fehlt

Als naechstes wuerde ich die Build-Konfiguration in Git ablegen und den Bau aus einer
frischen VM wiederholen. So ist festgehalten, wie das Image entsteht. Bytegleiche Builds
verlangen zusaetzlich Kontrolle ueber Paketversionen und Build-Umgebung; ein Codename allein
friert die enthaltenen Pakete nicht ein.

Eigene Programme und dauerhafte Systemanpassungen koennen spaeter in `.deb`-Pakete wandern.
Mit eigenen Paketquellen lassen sie sich auch auf bereits installierten Systemen verteilen.
Fuer ein veroeffentlichtes Derivat gehoeren ausserdem getestete Updates, Sicherheitsupdates
und klar benannte Zustaendigkeiten dazu.
[Debian: Leitlinien fuer Derivate](https://wiki.debian.org/Derivatives/Guidelines)

Fuer mein Homelab waere das erste konkrete Ziel eine ISO, die in einer Proxmox-VM startet
und genau die ausgewaehlten Werkzeuge enthaelt. Eine eigene Verwaltungsoberflaeche fuer
virtuelle Maschinen waere danach ein separates Softwareprojekt — auf derselben Grundlage,
aber mit deutlich mehr eigener Entwicklung.
