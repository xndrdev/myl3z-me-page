---
title: Boot-Stick im Terminal
weight: 10
---

# Boot-Stick im Terminal

Ein bootfaehiger USB-Stick aus einer ISO — ohne Zusatzsoftware, nur mit `lsblk`, `umount`
und `dd`. Das Beispiel schreibt einen Debian-Netinst-Installer, funktioniert aber mit jeder
Hybrid-ISO gleich. Was die einzelnen Befehle genau tun, steht unter
[lsblk, umount, dd, sync]({{< relref "disk-commands" >}}).

> [!WARNING]
> `dd` fragt nicht nach und kennt kein Undo. Ein falscher Geraetename ueberschreibt die
> Systemplatte. Das Ziel vor dem Ausfuehren zweimal pruefen.

{{% steps %}}

1. ## Geraet ermitteln
   ```sh
   lsblk -o NAME,SIZE,MODEL,TRAN,MOUNTPOINTS
   ```

   Beispielausgabe:

   ```text
   NAME        SIZE MODEL                   TRAN MOUNTPOINTS
   nvme0n1     1.8T Samsung SSD
   ├─nvme0n1p1   1G                              /boot
   └─nvme0n1p2 1.8T                              /
   sdb        28.7G SanDisk Ultra          usb
   └─sdb1     28.7G                              /run/media/xander/USB
   ```

   `TRAN usb`, die passende Groesse und der Modellname identifizieren den Stick: hier
   `/dev/sdb`.

2. ## Partition aushaengen
   ```sh
   sudo umount /dev/sdb1
   ```

   Das Geraet selbst bleibt eingehaengt-frei — nur die gemountete Partition muss weg.

3. ## ISO schreiben
   ```sh
   sudo dd \
     if="$HOME/Downloads/debian-13.6.0-amd64-netinst.iso" \
     of=/dev/sdb \
     bs=4M \
     status=progress \
     conv=fsync
   ```

   Den ISO-Dateinamen gegebenenfalls anpassen. Den exakten Namen liefert:

   ```sh
   ls -lh ~/Downloads/debian*.iso
   ```

4. ## Puffer leeren
   ```sh
   sync
   ```

   Erst danach ist wirklich alles auf dem Stick und er darf abgezogen werden.

{{% /steps %}}

## Ganzes Geraet, nicht die Partition

```text
richtig: /dev/sdb
falsch:  /dev/sdb1
```

Debian weist ausdruecklich darauf hin, dass die ISO auf das vollstaendige Laufwerk und nicht
auf eine einzelne Partition geschrieben werden muss. Die Hybrid-ISO bringt ihre eigene
Partitionstabelle mit — landet sie in einer bestehenden Partition, fehlt dem Stick der
Bootsektor.

## Die dd-Optionen

| Option | Bedeutung |
|--------|-----------|
| `if=` | Quelle (input file), die ISO |
| `of=` | Ziel (output file), das Blockgeraet |
| `bs=4M` | Blockgroesse; ohne diese Angabe kopiert `dd` in 512-Byte-Haeppchen und braucht ewig |
| `status=progress` | Fortschrittsanzeige statt stiller Wartezeit |
| `conv=fsync` | schreibt Puffer vor dem Beenden auf das Geraet |

> [!NOTE]
> `conv=fsync` macht das abschliessende `sync` streng genommen ueberfluessig. Es kostet nichts
> und schadet auch dann nicht, wenn die ISO mal ohne diese Option geschrieben wurde.
