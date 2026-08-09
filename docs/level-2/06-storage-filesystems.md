# 06 · Storage & Filesystems (UBI, overlayfs)

Storage is where embedded Linux products die. Not from bugs — from power
cuts. A desktop is shut down cleanly; a field device has its power yanked
mid-write, thousands of times over a ten-year life. This module is about
picking a storage stack that survives that, and understanding why the flash
under your filesystem behaves nothing like a disk.

## What is actually under the filesystem

```text
raw NAND / NOR                      eMMC / SD / UFS
      │                                    │
  MTD layer  (/dev/mtd0, mtdblock)   block layer (/dev/mmcblk0)
      │                                    │
  UBI  (wear levelling, bad blocks)   FTL inside the chip's controller
      │                                    │
  UBIFS / squashfs                     ext4 / f2fs / squashfs
```

The critical difference: **eMMC hides flash management in firmware you
cannot see or audit** (a Flash Translation Layer doing wear levelling,
garbage collection and bad-block handling). **Raw NAND exposes all of it to
Linux**, which is why raw NAND needs MTD + UBI and eMMC does not.

Three flash facts drive every design decision:

- You can only **erase in blocks** (128 KB is typical), never per byte.
- Each block tolerates a limited number of erases — roughly 100k for SLC,
  as few as **1k–3k for consumer TLC/QLC**. Writing a 1 KB log line every
  second will destroy a cheap part in months.
- A write interrupted by power loss can leave a page in an **indeterminate**
  state — not old data, not new data.

```console
root@target:~# cat /proc/mtd
dev:    size   erasesize  name
mtd0: 00100000 00020000 "u-boot"
mtd1: 00040000 00020000 "u-boot-env"
mtd2: 00800000 00020000 "kernel"
mtd3: 0f000000 00020000 "rootfs"
root@target:~# lsblk
NAME         MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
mmcblk0      179:0    0 14.6G  0 disk
├─mmcblk0p1  179:1    0   64M  0 part /boot
├─mmcblk0p2  179:2    0    1G  0 part /
└─mmcblk0p3  179:3    0 13.5G  0 part /data
mmcblk0boot0 179:32   0    4M  1 disk
```

Note `mmcblk0boot0` — eMMC has separate hardware boot partitions that the
SoC ROM reads before anything else. That is where your bootloader lives on
a real i.MX design, and writing to it is how boards get bricked.

## Choosing a filesystem

| Filesystem | Writable | Power-safe | Use it for |
|---|---|---|---|
| **squashfs** | no | inherently | read-only rootfs, compressed; the safe default |
| **ext4** | yes | with journal | data partitions on eMMC/SD |
| **f2fs** | yes | yes | large write-heavy eMMC data partitions |
| **UBIFS** | yes | yes (designed for it) | raw NAND, on top of UBI |
| **overlayfs** | yes | inherits from upper | writable layer over a read-only base |
| **tmpfs** | yes | RAM — lost on reboot | `/tmp`, `/run`, volatile state |

The single most valuable habit: **make the root filesystem read-only and
put every writable thing somewhere you chose deliberately.** Module 8 is
entirely about that; this module builds the pieces.

## ext4 tuned for flash

```console
root@target:~# mkfs.ext4 -L data -O ^has_journal -E stride=2,stripe-width=1024 \
    -m 0 /dev/mmcblk0p3
Creating filesystem with 3538944 4k blocks and 884736 inodes
Filesystem UUID: 4c3a9d1e-7b2f-4c88-9a01-5e7d3f1b2c60
Writing superblocks and filesystem accounting information: done
```

`-m 0` drops the 5 % root reserve (pointless on a data partition, and 5 %
of 13 GB is real money). `-O ^has_journal` removes the journal — **do not
do this** unless the partition is genuinely disposable; it roughly halves
write amplification and completely removes your crash consistency.

Mount options matter more than format options:

```text
# /etc/fstab
# <device>                  <mount>  <type>   <options>                        <dump> <pass>
PARTLABEL=rootfs            /        squashfs ro                                0      0
PARTLABEL=data              /data    ext4     defaults,noatime,errors=remount-ro 0     2
tmpfs                       /tmp     tmpfs    defaults,noatime,nosuid,size=32M   0     0
tmpfs                       /var/log tmpfs    defaults,noatime,nosuid,size=16M   0     0
```

- **`noatime`** stops a read from causing a write. On flash this is free
  lifetime.
- **`errors=remount-ro`** turns silent corruption into a loud, diagnosable
  failure instead of a filesystem that keeps writing over damage.
- **`PARTLABEL=`** instead of `/dev/mmcblk0p3` — device enumeration order
  is not guaranteed (module 4).

`commit=` is the tuning knob people reach for:

```text
PARTLABEL=data /data ext4 defaults,noatime,commit=30 0 2
```

That batches journal commits every 30 seconds instead of 5. Fewer writes,
better flash life — and up to 30 seconds of data lost on power cut. This
is a deliberate trade, not an optimisation.

## UBI and UBIFS on raw NAND

UBI is a volume manager that sits on MTD and handles wear levelling and
bad blocks across the *whole* partition, so UBIFS above it does not have
to. Build it on the host:

```console
$ mkfs.ubifs -m 2048 -e 126976 -c 2000 -r rootfs/ -o rootfs.ubifs
$ ubinize -o ubi.img -m 2048 -p 128KiB ubinize.cfg
```

```text
# ubinize.cfg
[rootfs-volume]
mode=ubi
image=rootfs.ubifs
vol_id=0
vol_type=dynamic
vol_name=rootfs
vol_flags=autoresize
```

The three geometry numbers are not optional and not guessable: `-m` is the
page (min I/O) size, `-e` the *logical* erase block size (physical erase
block minus UBI overhead — 131072 − 4096 = 126976 for a 128 KB block with
2 KB pages), and `-p` the physical erase block size. Get them from
`/sys/class/mtd/mtd3/` on the real part. Wrong values produce an image that
attaches and then corrupts.

On target:

```console
root@target:~# ubiattach -m 3
UBI device number 0, total 1920 LEBs
root@target:~# ubimkvol /dev/ubi0 -N data -s 64MiB
root@target:~# mount -t ubifs ubi0:data /data
root@target:~# ubinfo -a
Volume ID:   0 (on ubi0)
Name:        rootfs
Type:        dynamic
```

## overlayfs

overlayfs unions a read-only **lower** directory with a writable **upper**
one, so a read-only rootfs looks writable:

```console
root@target:~# mkdir -p /data/overlay/upper /data/overlay/work /merged
root@target:~# mount -t overlay overlay \
    -o lowerdir=/etc,upperdir=/data/overlay/upper,workdir=/data/overlay/work \
    /merged
root@target:~# echo "changed" >> /merged/hostname
root@target:~# ls /data/overlay/upper
hostname
```

Two rules with no exceptions: **`workdir` and `upperdir` must be on the
same filesystem**, and `workdir` must be an empty directory that nothing
else touches. Violating either gives `mount: wrong fs type, bad option`
with no useful detail.

## Traps

!!! danger "Storage traps — the ones that lose customer data"
    - **Power loss during a write.** The only real defence is a design where
      the thing being written is not the thing you need to boot: read-only
      rootfs, journalled or log-structured data partition, and atomic
      write-to-temp-then-`rename()` in your application. A `write()` that
      returned is *not* on the media until `fsync()` — and `fsync()` on the
      file is not enough if you also created it, you must `fsync()` the
      containing directory.
    - **Logging to flash.** Unbounded `/var/log` on eMMC is the most common
      cause of dead units at scale. Put it on tmpfs, or cap journald
      (module 9).
    - **Writing to `mmcblk0boot0` by mistake.** `dd` to the wrong device node
      overwrites the bootloader in the eMMC hardware boot partition, and the
      ROM then finds nothing. Recovery requires the SoC's serial-download
      mode (`uuu` on i.MX) — or a rework station. Double-check every `dd`
      target, and prefer writing images by PARTLABEL.
    - **Assuming an SD card is a disk.** Consumer cards lie about flush,
      remap silently, and fail without warning. They are fine for
      development and unacceptable in a product.
    - **Filling the filesystem to 100 %.** Flash filesystems need free blocks
      to garbage-collect; a full UBIFS or f2fs volume degrades badly and can
      fail writes that "should" fit. Reserve headroom and monitor it.
    - **`mkfs` geometry guessed rather than read** from sysfs — silent
      corruption weeks later, not an error at format time.

## Cheat sheet

| Command / item | Purpose |
|---|---|
| `cat /proc/mtd` / `lsblk` | Raw flash partitions / block devices |
| `mkfs.ext4 -m 0 -L data` | Format a data partition, no root reserve |
| `noatime` | Stop reads from writing — free flash lifetime |
| `errors=remount-ro` | Fail loudly instead of writing over corruption |
| `commit=30` | Batch journal commits: fewer writes, more data at risk |
| `PARTLABEL=` / `UUID=` in fstab | Stable identity, immune to enumeration order |
| `mksquashfs dir img.sqfs -comp xz` | Build a compressed read-only rootfs |
| `mkfs.ubifs -m -e -c` | Build a UBIFS image (page / LEB / max-LEB count) |
| `ubinize -o ubi.img -m -p cfg` | Wrap UBIFS volumes into a flashable UBI image |
| `ubiattach -m N` / `ubimkvol` / `ubinfo -a` | Attach MTD to UBI, make volume, inspect |
| `mount -t overlay -o lowerdir=,upperdir=,workdir=` | Writable layer over read-only |
| `tmpfs ... size=32M` | Volatile, RAM-backed, zero flash wear |
| `fsync()` file **and** parent dir | What actually makes data durable |
| `sync; echo 3 > /proc/sys/vm/drop_caches` | Flush and drop caches when testing |
| `df -h` / `df -i` | Free space / free **inodes** (a separate way to fill up) |

!!! note "On verification"
    Command syntax and mount options here follow the documented `mkfs.ubifs`,
    `ubinize`, overlayfs and ext4 interfaces. No flash hardware was available
    while writing this page, so the UBI geometry numbers are worked examples
    — always read `/sys/class/mtd/mtdN/{writesize,erasesize}` on your own
    part rather than copying them.

## Exercise

(1) In QEMU, add a second virtual disk, partition it, format one partition
ext4 with `-m 0` and mount it with `noatime,errors=remount-ro`; confirm the
options took with `findmnt` and `/proc/mounts`. (2) Build a squashfs image
of a directory tree, mount it, prove it is read-only, then union it with an
overlayfs upper on your ext4 partition and show that a modified file appears
in `upperdir` while the squashfs is untouched. (3) Simulate power loss:
write a loop that appends to a file, kill QEMU with `SIGKILL` mid-write, and
compare the result with and without `fsync()` in the writer — report how much
data each variant lost. (4) One paragraph: your product logs 2 KB per second
to `/var/log` on a 4 GB eMMC rated at 3,000 program/erase cycles. Estimate
the write amplification and rough lifetime, then describe the storage layout
you would ship instead, naming the filesystem for each mount point.
