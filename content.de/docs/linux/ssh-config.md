---
title: SSH-Config und Key-Login
weight: 40
---

# SSH-Config und Key-Login

Ohne Konfiguration besteht jede Verbindung aus derselben Tipparbeit:

```sh
ssh -i ~/.ssh/id_rsa xander@10.10.10.3
```

Adresse, Benutzer und Schluessel gehoeren nicht in die Kommandozeile, sondern in die
Client-Konfiguration unter `~/.ssh/config`. Dort steht pro Host, wie die Verbindung aussehen
soll — danach genuegt der Alias:

```sh
ssh dns01
```

## Der Eintrag

```text
Host dns01
    HostName 10.10.10.3
    IdentityFile /home/xander/.ssh/id_rsa
    IdentitiesOnly yes
```

| Zeile | Bedeutung |
|-------|-----------|
| `Host dns01` | der Name, den man auf der Kommandozeile tippt. Frei waehlbar und unabhaengig vom echten Hostnamen; alles bis zum naechsten `Host` gehoert zu diesem Block |
| `HostName 10.10.10.3` | wohin tatsaechlich verbunden wird — IP oder DNS-Name |
| `IdentityFile` | der **private** Schluessel fuer diese Verbindung. Der oeffentliche daneben (`.pub`) liegt auf dem Server |
| `IdentitiesOnly yes` | nur diesen Schluessel anbieten, nichts anderes |

Der Datei-Abschnitt ist eine Sammlung von Defaults, keine Verbindung fuer sich. Er greift bei
allem, was auf den Alias passt — also auch bei `scp`, `rsync` und `git`, die denselben
SSH-Client verwenden:

```sh
scp bericht.txt dns01:/tmp/
rsync -a ./config/ dns01:/etc/pihole/
```

> [!NOTE]
> Der Eintrag setzt kein `User`. SSH nimmt dann den lokalen Benutzernamen — hier `xander`.
> Heisst der Account auf dem Server anders, gehoert eine Zeile `User <name>` dazu, sonst
> scheitert die Anmeldung mit einem Namen, den es dort nicht gibt. Fuer einen abweichenden
> Port entsprechend `Port 2222`.

## Warum IdentitiesOnly

Ohne diese Option bietet der Client dem Server der Reihe nach alles an, was er finden kann:
jeden Schluessel im Agenten, dazu die Standardnamen in `~/.ssh/`. Der explizit angegebene
`IdentityFile` ist dann nur ein Kandidat unter mehreren.

Das Problem ist serverseitig: `sshd` erlaubt pro Verbindung nur eine begrenzte Zahl an
Versuchen (`MaxAuthTries`, standardmaessig 6). Wer viele Schluessel im Agenten hat, wird
abgewiesen, bevor der richtige an der Reihe ist:

```text
Received disconnect from 10.10.10.3 port 22:2: Too many authentication failures
```

`IdentitiesOnly yes` beschraenkt den Client auf die im Block genannten Schluessel. Ein
Versuch, der richtige, fertig.

## Den Public Key auf dem Server hinterlegen

Der Server muss den oeffentlichen Schluessel kennen. Das erledigt ein Befehl — er legt
Verzeichnis und Datei mit den korrekten Rechten an, falls noetig, und haengt nichts doppelt an:

```sh
ssh-copy-id -i ~/.ssh/id_rsa.pub dns01
```

Beim ersten Mal laeuft das noch ueber das Passwort des Accounts. Danach:

```sh
ssh dns01        # ohne Passwort-Abfrage
```

Von Hand ist es dasselbe in laenger — der Inhalt der `.pub`-Datei kommt als eine Zeile in
`~/.ssh/authorized_keys` des Zielbenutzers:

```sh
cat ~/.ssh/id_rsa.pub | ssh dns01 'mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys'
```

## Rechte

SSH verweigert den Dienst bei zu grosszuegigen Rechten — auf beiden Seiten, und ohne
Kulanz. Ein Schluessel, den auch andere lesen koennen, gilt als kompromittiert:

| Pfad | Rechte | Seite |
|------|--------|-------|
| `~/.ssh` | `700` | beide |
| `~/.ssh/config` | `600` | Client |
| `~/.ssh/id_rsa` | `600` | Client |
| `~/.ssh/id_rsa.pub` | `644` | Client |
| `~/.ssh/authorized_keys` | `600` | Server |

```sh
chmod 700 ~/.ssh && chmod 600 ~/.ssh/config ~/.ssh/id_rsa
```

## Pruefen

Welche Optionen fuer einen Host tatsaechlich gelten — mit allen Defaults, aufgeloest:

```sh
ssh -G dns01
```

Wenn die Anmeldung nicht klappt, zeigt der ausfuehrliche Modus, welche Schluessel angeboten
wurden und was der Server damit gemacht hat:

```sh
ssh -v dns01
```

> [!WARNING]
> Innerhalb der Config gewinnt der **erste** gefundene Wert je Schluesselwort, nicht der
> letzte. Spezifische Hosts gehoeren deshalb nach oben, ein allgemeiner `Host *`-Block ans
> Ende der Datei.

## Weitere nuetzliche Optionen

| Option | Zweck |
|--------|-------|
| `User <name>` | abweichender Benutzername auf dem Server |
| `Port <n>` | abweichender Port |
| `ServerAliveInterval 60` | haelt die Verbindung offen, statt sie in Leerlaufphasen abreissen zu lassen |
| `ForwardAgent yes` | reicht den lokalen Agenten weiter — nur bei Hosts, denen man vertraut |
| `ProxyJump <host>` | Verbindung ueber einen Sprungserver, wenn das Ziel nicht direkt erreichbar ist |

> [!NOTE]
> `id_rsa` funktioniert weiterhin, sofern der Schluessel mindestens 3072 Bit hat — die alte
> SHA-1-Signatur `ssh-rsa` ist seit OpenSSH 8.8 abgeschaltet, moderne RSA-Signaturen sind es
> nicht. Fuer neue Schluessel ist `ssh-keygen -t ed25519` heute die Standardempfehlung:
> kuerzer, schneller und ohne die Frage nach der Schluessellaenge.
