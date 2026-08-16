---
title: Bootable USB from the Terminal
weight: 10
---

# Bootable USB from the Terminal

A bootable USB stick from an ISO — no extra tooling, just `lsblk`, `umount` and `dd`. The
example writes a Debian netinst installer, but the steps are the same for any hybrid ISO. For
what each command actually does, see [lsblk, umount, dd, sync]({{< relref "disk-commands" >}}).

> [!WARNING]
> `dd` does not ask and has no undo. The wrong device name overwrites your system disk. Check
> the target twice before running it.

{{% steps %}}

1. ## Identify the device
   ```sh
   lsblk -o NAME,SIZE,MODEL,TRAN,MOUNTPOINTS
   ```

   Example output:

   ```text
   NAME        SIZE MODEL                   TRAN MOUNTPOINTS
   nvme0n1     1.8T Samsung SSD
   ├─nvme0n1p1   1G                              /boot
   └─nvme0n1p2 1.8T                              /
   sdb        28.7G SanDisk Ultra          usb
   └─sdb1     28.7G                              /run/media/xander/USB
   ```

   `TRAN usb`, the matching size and the model name identify the stick — here `/dev/sdb`.

2. ## Unmount the partition
   ```sh
   sudo umount /dev/sdb1
   ```

   Only the mounted partition has to go; the device itself is never mounted.

3. ## Write the ISO
   ```sh
   sudo dd \
     if="$HOME/Downloads/debian-13.6.0-amd64-netinst.iso" \
     of=/dev/sdb \
     bs=4M \
     status=progress \
     conv=fsync
   ```

   Adjust the ISO filename if needed. This gives you the exact name:

   ```sh
   ls -lh ~/Downloads/debian*.iso
   ```

4. ## Flush the buffers
   ```sh
   sync
   ```

   Only now is everything actually on the stick and safe to unplug.

{{% /steps %}}

## Whole device, not the partition

```text
correct: /dev/sdb
wrong:   /dev/sdb1
```

Debian explicitly states that the ISO must be written to the complete drive, not to a single
partition. A hybrid ISO carries its own partition table — write it inside an existing
partition and the stick has no boot sector.

## The dd options

| Option | Meaning |
|--------|---------|
| `if=` | input file, the ISO |
| `of=` | output file, the block device |
| `bs=4M` | block size; without it `dd` copies in 512-byte chunks and takes forever |
| `status=progress` | progress output instead of silence |
| `conv=fsync` | flushes buffers to the device before exiting |

> [!NOTE]
> `conv=fsync` technically makes the trailing `sync` redundant. It costs nothing and still
> helps if the ISO was written without that option.
