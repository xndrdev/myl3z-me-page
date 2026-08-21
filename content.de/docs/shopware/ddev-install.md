---
title: ddev installieren
weight: 20
---

# ddev installieren

Einmal pro Rechner. Danach ist jedes weitere Projekt eine Sache von Minuten.

Es braucht genau zwei Dinge: **Docker** und **ddev**. Alles andere — PHP, Composer, Node,
MySQL — liefert ddev in Containern mit.

## Schritt 1: Docker

ddev braucht einen Docker-Provider. Welcher, ist weitgehend egal, solange er laeuft.

{{< tabs "docker" >}}
{{% tab "Linux" %}}
Docker CE direkt aus der Distribution, das ist die schnellste Variante:

```bash
# Arch
sudo pacman -S docker docker-buildx
sudo systemctl enable --now docker
sudo usermod -aG docker "$USER"

# Debian / Ubuntu
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker "$USER"
```

Nach `usermod` einmal ab- und wieder anmelden, sonst fehlen die Rechte auf den Docker-Socket.
{{% /tab %}}
{{% tab "macOS" %}}
Docker Desktop, OrbStack oder Rancher Desktop. OrbStack ist auf Apple Silicon spuerbar
sparsamer mit Ressourcen:

```bash
brew install --cask orbstack
# oder
brew install --cask docker
```
{{% /tab %}}
{{% tab "Windows" %}}
Ueber WSL2 — Docker laeuft *innerhalb* der Linux-Distribution, nicht auf Windows:

```powershell
wsl --install --no-distribution
wsl --update
wsl --install Ubuntu --name DDEV
```

Danach in der WSL2-Distribution weitermachen wie unter Linux. Projekte muessen im
Linux-Dateisystem liegen (`~/projekte/`), nicht unter `/mnt/c/` — sonst ist die
Dateizugriffsgeschwindigkeit unbrauchbar.
{{% /tab %}}
{{< /tabs >}}

Pruefen, ob Docker laeuft:

```bash
docker ps
```

Kommt eine leere Tabelle statt einer Fehlermeldung, passt es.

## Schritt 2: ddev

{{< tabs "ddev" >}}
{{% tab "Arch" %}}
```bash
yay -S ddev-bin
```

Die Variante auf dieser Maschine.
{{% /tab %}}
{{% tab "Debian / Ubuntu" %}}
```bash
sudo apt-get update && sudo apt-get install -y curl
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://pkg.ddev.com/apt/gpg.key | sudo tee /etc/apt/keyrings/ddev.asc > /dev/null
sudo chmod a+r /etc/apt/keyrings/ddev.asc
printf "Types: deb\nURIs: https://pkg.ddev.com/apt/\nSuites: *\nComponents: *\nSigned-By: /etc/apt/keyrings/ddev.asc\n" | sudo tee /etc/apt/sources.list.d/ddev.sources >/dev/null
sudo apt-get update && sudo apt-get install -y ddev
```
{{% /tab %}}
{{% tab "macOS" %}}
```bash
brew install ddev/ddev/ddev
```
{{% /tab %}}
{{% tab "Sonst" %}}
Das Installationsskript funktioniert auf allen Linux-Varianten und macOS:

```bash
curl -fsSL https://ddev.com/install.sh | bash
```

Eine bestimmte Version:

```bash
curl -fsSL https://ddev.com/install.sh | bash -s v1.25.2
```
{{% /tab %}}
{{< /tabs >}}

## Schritt 3: Zertifikate

Damit `https://projekt.ddev.site` ohne Browserwarnung funktioniert, muss die lokale
Zertifizierungsstelle einmal registriert werden:

```bash
mkcert -install
```

> [!NOTE]
> Wird das uebersprungen, funktioniert der Shop trotzdem — aber jeder Aufruf beginnt mit einer
> Zertifikatswarnung, und die Administration verhaelt sich in Randfaellen anders. Einmal
> ausfuehren, nie wieder daran denken.

## Pruefen

```bash
ddev version
```

Zum Vergleich der Stand, auf dem diese Notizen beruhen:

```text
DDEV version      v1.25.2
docker            29.5.1
docker-compose    v5.1.3
docker-platform   linux-docker
```

## Aktualisieren

Ueber denselben Weg wie installiert:

```bash
yay -S ddev-bin                    # Arch
sudo apt-get update && sudo apt-get install --only-upgrade ddev   # Debian/Ubuntu
brew upgrade ddev/ddev/ddev        # macOS
```

Nach einem ddev-Update muessen die Projekt-Container neu gebaut werden — ddev weist beim
naechsten `ddev start` darauf hin. Im Zweifel:

```bash
ddev restart
```

> [!WARNING]
> Bei groesseren Versionsspruengen koennen sich Container-Images aendern. Vor einem Update in
> einem Projekt mit wichtigem Datenstand: `ddev snapshot` anlegen.

## Weiter

Damit steht die Grundlage: [Shopware 6 aufsetzen]({{< relref "shopware-setup" >}}).
