---
title: lsblk, umount, dd, sync
weight: 20
---

# lsblk, umount, dd, sync

Die vier Befehle, mit denen man unter Linux an Blockgeraete herangeht: eines finden, es
freigeben, es beschreiben und sicherstellen, dass die Daten wirklich drauf sind. Angewandt
werden sie unter [Boot-Stick im Terminal]({{< relref "bootable-usb" >}}).

## lsblk — Blockgeraete auflisten

`lsblk` liest den Geraetebaum aus dem Kernel (`/sys/block`) und stellt ihn als Baum dar:
physische Datentraeger als Wurzel, ihre Partitionen als Kinder. Es braucht keine
Root-Rechte und fasst nichts an.

```sh
lsblk
```

```text
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
nvme0n1     259:0    0   1.8T  0 disk
├─nvme0n1p1 259:1    0     1G  0 part /boot
└─nvme0n1p2 259:2    0   1.8T  0 part /
sdb           8:16   1  28.7G  0 disk
└─sdb1        8:17   1  28.7G  0 part /run/media/xander/USB
```

Die Standardspalten sagen wenig ueber die Herkunft eines Geraets. Mit `-o` waehlt man selbst
aus:

```sh
lsblk -o NAME,SIZE,MODEL,TRAN,MOUNTPOINTS
```

| Spalte | Bedeutung |
|--------|-----------|
| `NAME` | Kernel-Name, z. B. `sdb`, `sdb1`, `nvme0n1p2` |
| `SIZE` | Kapazitaet — das schnellste Unterscheidungsmerkmal bei USB-Sticks |
| `TYPE` | `disk`, `part`, `rom`, `lvm`, `crypt` |
| `TRAN` | Anbindung: `usb`, `nvme`, `sata` — verraet, was extern ist |
| `MODEL` | Herstellerbezeichnung, z. B. `SanDisk Ultra` |
| `RM` | `1` = Wechselmedium |
| `MOUNTPOINTS` | wo die Partition gerade eingehaengt ist (leer = nicht gemountet) |
| `FSTYPE` | Dateisystem, z. B. `ext4`, `vfat`, `iso9660` |

Nuetzliche Schalter:

```sh
lsblk -f              # Dateisystem, Label und UUID statt der Geraete-Details
lsblk -p              # vollstaendige Pfade (/dev/sdb statt sdb) — direkt kopierbar
lsblk -d              # nur die Datentraeger, ohne Partitionen
lsblk -J              # JSON-Ausgabe fuer Skripte
lsblk /dev/sdb        # nur ein Geraet
```

> [!TIP]
> Beim Identifizieren eines USB-Sticks lohnt sich der Vorher-Nachher-Vergleich: `lsblk` ohne
> Stick, Stick einstecken, `lsblk` erneut. Was neu dazugekommen ist, ist der Stick. Alternativ
> zeigt `dmesg -w` beim Einstecken direkt den vergebenen Geraetenamen.

## umount — Dateisystem aushaengen

Ein gemountetes Dateisystem gehoert dem Kernel: er haelt Metadaten im Cache und schreibt
verzoegert zurueck. Solange das so ist, darf niemand sonst auf das Geraet schreiben — sonst
kollidieren zwei Instanzen auf denselben Bloecken. `umount` beendet diesen Zustand: aus dem
Cache wird geschrieben, das Dateisystem wird aus dem Verzeichnisbaum geloest.

Beide Schreibweisen meinen dasselbe Dateisystem:

```sh
sudo umount /dev/sdb1                        # ueber das Geraet
sudo umount /run/media/xander/USB            # ueber den Mountpoint
```

Ausgehaengt wird die **Partition**, nicht das Geraet — `/dev/sdb` selbst ist nie gemountet.
Hat ein Stick mehrere Partitionen, muessen alle weg:

```sh
sudo umount /dev/sdb?*                       # alle Partitionen von sdb
```

Haeufige Stolpersteine:

| Meldung | Ursache | Loesung |
|---------|---------|---------|
| `target is busy` | ein Prozess hat eine Datei offen oder steht mit der Shell im Verzeichnis | `lsof +f -- /dev/sdb1` oder `fuser -vm /dev/sdb1` zeigt wen; Verzeichnis verlassen |
| `not mounted` | war nie eingehaengt | nichts zu tun |
| `must be superuser` | Mount stammt aus `/etc/fstab` ohne `user`-Option | mit `sudo` |

Bei Sticks, die der Desktop automatisch eingehaengt hat, ist `udisksctl` der sauberere Weg —
er laeuft ohne `sudo` und meldet den Vorgang an den Desktop zurueck:

```sh
udisksctl unmount -b /dev/sdb1
```

> [!WARNING]
> `umount -l` (lazy) loest das Dateisystem sofort aus dem Baum, laesst offene Dateihandles aber
> weiterlaufen. Als Vorbereitung fuer `dd` ist das die falsche Wahl: es kann noch geschrieben
> werden, waehrend `dd` bereits laeuft.

## dd — Bloecke kopieren

`dd` kopiert Daten von einer Quelle zu einem Ziel, ohne sie zu interpretieren. Genau das macht
es fuer ISO-Images richtig: eine Hybrid-ISO enthaelt Partitionstabelle, Bootsektor und
Dateisystem als fertiges Byte-Abbild. Ein dateiweises Kopierprogramm wuerde nur die Dateien
uebertragen und den Bootsektor verlieren.

```sh
sudo dd if=quelle of=ziel bs=4M status=progress conv=fsync
```

| Operand | Bedeutung |
|---------|-----------|
| `if=` | input file — Quelle; ohne Angabe liest `dd` von stdin |
| `of=` | output file — Ziel; ohne Angabe schreibt `dd` nach stdout |
| `bs=` | block size, Groesse pro Lese-/Schreibvorgang. Default sind 512 Byte, was bei einem 800-MB-Image ueber 1,5 Millionen Einzelaufrufe bedeutet. `4M` ist der uebliche Kompromiss |
| `count=` | nur so viele Bloecke kopieren |
| `skip=` / `seek=` | Bloecke am Anfang der Quelle bzw. des Ziels ueberspringen |
| `status=progress` | laufende Fortschrittsanzeige; ohne das bleibt `dd` bis zum Ende stumm |
| `conv=fsync` | schreibt am Ende alle Puffer auf das Geraet und beendet sich erst danach |
| `conv=noerror,sync` | Lesefehler ueberspringen und mit Nullen auffuellen — fuer Rettungsversuche von defekten Medien |

`dd` gibt am Ende eine Bilanz aus:

```text
209+1 records in
209+1 records out
878706688 bytes (879 MB, 838 MiB) copied, 62.4 s, 14.1 MB/s
```

`209+1` heisst: 209 vollstaendige Bloecke und ein unvollstaendiger. Das ist normal, wenn die
Dateigroesse kein Vielfaches der Blockgroesse ist.

> [!WARNING]
> `dd` kennt keine Rueckfrage und keine Bestaetigung. `of=/dev/sda` statt `of=/dev/sdb`
> ueberschreibt die Systemplatte ab Byte 0, inklusive Partitionstabelle. Das Geraet vor jedem
> Aufruf mit `lsblk` verifizieren.

Laeuft `dd` bereits ohne `status=progress`, liefert ein Signal von aussen eine
Zwischenausgabe:

```sh
sudo kill -USR1 $(pgrep -x dd)
```

## sync — Puffer auf das Geraet schreiben

Linux schreibt nicht sofort auf den Datentraeger. Schreibvorgaenge landen zunaechst als
*dirty pages* im Page Cache und werden verzoegert weggeschrieben — das buendelt Zugriffe und
macht das System schnell. Der Preis: wenn `dd` fertig meldet, koennen noch Megabytes im
Arbeitsspeicher stehen. Ein in diesem Moment abgezogener Stick ist unvollstaendig.

```sh
sync                  # alle Dateisysteme
sync /dev/sdb         # nur dieses Geraet
sync -f /pfad/datei   # nur das Dateisystem, auf dem die Datei liegt
```

`sync` kehrt erst zurueck, wenn der Kernel alles weggeschrieben hat. Bei einem langsamen
USB-Stick kann das nach einem scheinbar fertigen `dd` noch Minuten dauern — das ist kein
Haenger, sondern der eigentliche Schreibvorgang.

Was noch aussteht, zeigt:

```sh
grep -E 'Dirty|Writeback' /proc/meminfo
```

> [!NOTE]
> `conv=fsync` erledigt fuer `dd` dasselbe und macht das nachgelagerte `sync` in dem Fall
> ueberfluessig. Es getrennt aufzurufen kostet nichts und rettet die Faelle, in denen die
> Option vergessen wurde.
