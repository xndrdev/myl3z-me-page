---
title: lsblk, umount, dd, sync
weight: 20
---

# lsblk, umount, dd, sync

The four commands you reach for when working with block devices on Linux: find one, release
it, write to it, and make sure the data actually landed. They are put to use in
[Bootable USB from the Terminal]({{< relref "bootable-usb" >}}).

## lsblk — list block devices

`lsblk` reads the device tree from the kernel (`/sys/block`) and prints it as a tree: physical
drives as roots, their partitions as children. It needs no root privileges and touches
nothing.

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

The default columns say little about where a device came from. Use `-o` to pick your own:

```sh
lsblk -o NAME,SIZE,MODEL,TRAN,MOUNTPOINTS
```

| Column | Meaning |
|--------|---------|
| `NAME` | kernel name, e.g. `sdb`, `sdb1`, `nvme0n1p2` |
| `SIZE` | capacity — the fastest way to spot a USB stick |
| `TYPE` | `disk`, `part`, `rom`, `lvm`, `crypt` |
| `TRAN` | transport: `usb`, `nvme`, `sata` — tells you what is external |
| `MODEL` | vendor string, e.g. `SanDisk Ultra` |
| `RM` | `1` = removable |
| `MOUNTPOINTS` | where the partition is currently mounted (empty = not mounted) |
| `FSTYPE` | filesystem, e.g. `ext4`, `vfat`, `iso9660` |

Useful flags:

```sh
lsblk -f              # filesystem, label and UUID instead of device details
lsblk -p              # full paths (/dev/sdb instead of sdb) — copy-paste ready
lsblk -d              # drives only, no partitions
lsblk -J              # JSON output for scripts
lsblk /dev/sdb        # a single device
```

> [!TIP]
> When identifying a USB stick, compare before and after: run `lsblk`, plug the stick in, run
> it again. Whatever is new is the stick. Alternatively `dmesg -w` prints the assigned device
> name the moment you plug it in.

## umount — unmount a filesystem

A mounted filesystem belongs to the kernel: it caches metadata and writes back lazily. While
that is the case, nothing else may write to the device — two writers on the same blocks
corrupt each other. `umount` ends that state: caches are flushed, the filesystem is detached
from the directory tree.

Both forms refer to the same filesystem:

```sh
sudo umount /dev/sdb1                        # by device
sudo umount /run/media/xander/USB            # by mount point
```

You unmount the **partition**, not the device — `/dev/sdb` itself is never mounted. If a stick
has several partitions, all of them have to go:

```sh
sudo umount /dev/sdb?*                       # every partition of sdb
```

Common obstacles:

| Message | Cause | Fix |
|---------|-------|-----|
| `target is busy` | a process holds a file open, or your shell sits in the directory | `lsof +f -- /dev/sdb1` or `fuser -vm /dev/sdb1` names it; leave the directory |
| `not mounted` | it never was | nothing to do |
| `must be superuser` | mount comes from `/etc/fstab` without the `user` option | use `sudo` |

For sticks the desktop auto-mounted, `udisksctl` is the cleaner route — no `sudo`, and the
desktop is told about it:

```sh
udisksctl unmount -b /dev/sdb1
```

> [!WARNING]
> `umount -l` (lazy) detaches the filesystem from the tree immediately but leaves open file
> handles alive. As preparation for `dd` that is the wrong choice: writes can still land while
> `dd` is already running.

## dd — copy blocks

`dd` copies data from a source to a target without interpreting it. That is exactly what an
ISO image needs: a hybrid ISO contains partition table, boot sector and filesystem as one
finished byte image. A file-based copy would transfer the files and lose the boot sector.

```sh
sudo dd if=source of=target bs=4M status=progress conv=fsync
```

| Operand | Meaning |
|---------|---------|
| `if=` | input file — the source; without it `dd` reads stdin |
| `of=` | output file — the target; without it `dd` writes to stdout |
| `bs=` | block size per read/write. The default is 512 bytes, which for an 800 MB image means over 1.5 million individual calls. `4M` is the usual compromise |
| `count=` | copy only this many blocks |
| `skip=` / `seek=` | skip blocks at the start of the source resp. the target |
| `status=progress` | live progress; without it `dd` stays silent until it is done |
| `conv=fsync` | flushes all buffers to the device and only then exits |
| `conv=noerror,sync` | skip read errors and pad with zeroes — for recovering failing media |

`dd` prints a summary when it finishes:

```text
209+1 records in
209+1 records out
878706688 bytes (879 MB, 838 MiB) copied, 62.4 s, 14.1 MB/s
```

`209+1` means 209 full blocks and one partial block. That is normal whenever the file size is
not a multiple of the block size.

> [!WARNING]
> `dd` never asks and never confirms. `of=/dev/sda` instead of `of=/dev/sdb` overwrites your
> system disk from byte 0, partition table included. Verify the device with `lsblk` before
> every run.

If `dd` is already running without `status=progress`, a signal from outside produces an
interim report:

```sh
sudo kill -USR1 $(pgrep -x dd)
```

## sync — flush buffers to the device

Linux does not write to storage immediately. Writes first become *dirty pages* in the page
cache and are flushed later — that batches access and keeps the system fast. The price: when
`dd` reports success, megabytes may still be sitting in RAM. A stick unplugged at that moment
is incomplete.

```sh
sync                  # all filesystems
sync /dev/sdb         # this device only
sync -f /path/file    # only the filesystem holding that file
```

`sync` returns only once the kernel has written everything out. On a slow USB stick that can
take minutes after an apparently finished `dd` — that is not a hang, that is the actual write.

To see what is still pending:

```sh
grep -E 'Dirty|Writeback' /proc/meminfo
```

> [!NOTE]
> `conv=fsync` does the same job for `dd` and makes a trailing `sync` redundant in that case.
> Calling it separately costs nothing and saves the runs where the option was forgotten.
