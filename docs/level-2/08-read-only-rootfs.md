# 08 · Read-Only Rootfs & Factory Reset

Module 6 established the problem: flash plus unexpected power loss equals
corruption. This module is the standard industry answer. Make the root
filesystem **physically read-only**, so nothing can corrupt it; move every
writable thing to a place you chose, sized and can wipe; and provide a
factory-reset path that restores a known-good state without a service
visit. Once `/` is read-only, a corrupted device becomes a device that
reboots into a working system.

## The layout

```text
mmcblk0p1  /boot   FAT, ro     kernel + DTB + boot script
mmcblk0p2  /       squashfs ro the entire OS — immutable, verifiable
mmcblk0p3  /data   ext4  rw    config, app state, logs that must persist
           /tmp    tmpfs rw    scratch, 32 MB, gone on reboot
           /run    tmpfs rw    PIDs, sockets, runtime state (always tmpfs)
           /var/log tmpfs rw   logs that may be lost
           /etc    overlay rw   read-only base + writable upper on /data
```

The rule that makes this work: **anything writable is either disposable
(tmpfs) or on `/data`.** There is no third category. When you find a
program insisting on writing to a fourth place, you either bind-mount it
onto `/data` or you accept that it will fail — and it is much better to
discover which at build time than in the field.

## Finding what writes where

Before flipping the switch, measure. Boot the system with a writable rootfs
and look at what changed:

```console
root@target:~# mount -o remount,ro /
mount: /: cannot remount read-only, is busy
root@target:~# lsof / 2>/dev/null | grep -v REG | head
root@target:~# find / -xdev -newer /etc/os-release -type f 2>/dev/null | head -20
/etc/machine-id
/etc/resolv.conf
/var/lib/dbus/machine-id
/var/lib/systemd/random-seed
/var/log/journal/...
```

That list is your work item list. Each entry gets a decision: symlink into
`/data`, tmpfs, overlay, or "make it stop".

`systemd-analyze` will also tell you which units want to write:

```console
root@target:~# systemd-analyze verify appd.service
root@target:~# systemctl status systemd-machine-id-commit
```

## Mounting read-only

`/etc/fstab` does most of the job:

```text
# <device>          <mount>      <type>    <options>                              <dump> <pass>
PARTLABEL=rootfs    /            squashfs  ro                                      0      0
PARTLABEL=boot      /boot        vfat      ro,noatime,umask=0077                   0      0
PARTLABEL=data      /data        ext4      rw,noatime,errors=remount-ro,nofail     0      2
tmpfs               /tmp         tmpfs     rw,nosuid,nodev,noatime,size=32M,mode=1777 0   0
tmpfs               /var/log     tmpfs     rw,nosuid,nodev,noatime,size=16M         0     0
tmpfs               /var/tmp     tmpfs     rw,nosuid,nodev,noatime,size=8M          0     0
```

`nofail` on `/data` is deliberate: if the data partition is unmountable —
which is exactly the corruption case this design exists to survive — the
device must still boot, not drop to an emergency shell in a locked cabinet.

The kernel command line must agree, or the initramfs will remount rw:

```text
root=PARTLABEL=rootfs ro rootwait console=ttymxc0,115200
```

## The overlay for `/etc`

Some state genuinely belongs in `/etc` and genuinely must persist —
`machine-id`, network config edited by an installer, SSH host keys. An
overlay gives you a writable `/etc` whose *changes* live on `/data`:

```ini
# /etc/systemd/system/etc-overlay.service
[Unit]
Description=Writable overlay for /etc
DefaultDependencies=no
After=data.mount
Before=local-fs.target sysinit.target
RequiresMountsFor=/data

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStartPre=/bin/mkdir -p /data/overlay/etc/upper /data/overlay/etc/work
ExecStart=/bin/mount -t overlay overlay \
    -o lowerdir=/etc,upperdir=/data/overlay/etc/upper,workdir=/data/overlay/etc/work \
    /etc

[Install]
WantedBy=local-fs.target
```

```console
root@target:~# findmnt /etc
TARGET SOURCE  FSTYPE  OPTIONS
/etc   overlay overlay rw,relatime,lowerdir=/etc,upperdir=/data/overlay/etc/upper,...
root@target:~# ls /data/overlay/etc/upper
machine-id  resolv.conf  ssh
```

`DefaultDependencies=no` is what lets this run before `sysinit.target`;
without it systemd inserts an ordering dependency that deadlocks against
the very target you are trying to run before. `upperdir` and `workdir` must
be on the same filesystem (module 6) — both are on `/data` here, which
satisfies that.

Simpler alternative, and often the better one: keep `/etc` read-only and
symlink the four files that need to change.

```console
root@target:~# ls -l /etc/machine-id /etc/resolv.conf
lrwxrwxrwx 1 root root 21 /etc/machine-id -> /data/etc/machine-id
lrwxrwxrwx 1 root root 39 /etc/resolv.conf -> ../run/systemd/resolve/stub-resolv.conf
```

Fewer moving parts, no overlay in your boot critical path, and trivially
auditable. Reach for the overlay only when the list of writable paths is
long or unknown.

## Factory reset

Factory reset means: destroy `/data`, keep everything else. Because the OS
is read-only and separate, this is a genuinely safe operation.

The trigger must survive a device that will not boot far enough to run an
application, so it belongs early and it belongs in the bootloader or a
first-boot unit. A flag file is the simplest reliable mechanism:

```bash
#!/bin/sh
# /usr/sbin/factory-reset — invoked by factory-reset.service at early boot
set -eu

MARKER=/data/.factory-reset
LOG=/dev/kmsg

[ -e "$MARKER" ] || exit 0

echo "factory-reset: wiping /data" > "$LOG"

# Reformat rather than rm -rf: faster, and it clears any corruption too.
umount /data 2>/dev/null || true
mkfs.ext4 -F -q -L data -m 0 "$(blkid -L data)"
mount /data

# Restore defaults shipped read-only with the OS image
mkdir -p /data/etc
cp -a /usr/share/factory/etc/. /data/etc/

echo "factory-reset: complete" > "$LOG"
sync
reboot -f
```

```ini
# /etc/systemd/system/factory-reset.service
[Unit]
Description=Factory reset check
DefaultDependencies=no
After=data.mount
Before=sysinit.target
ConditionPathExists=/data/.factory-reset

[Service]
Type=oneshot
ExecStart=/usr/sbin/factory-reset
StandardOutput=journal

[Install]
WantedBy=sysinit.target
```

The physical trigger — a recessed button held for 10 seconds — is read by
U-Boot (module 2), which creates the marker or sets an environment variable
and boots normally. Doing the *wipe* in Linux rather than U-Boot means you
get a real filesystem driver and real error handling.

`/usr/share/factory/` is a systemd convention (`systemd-tmpfiles --copy`
understands it) and it costs you nothing: the defaults ride along in the
read-only image, so they cannot themselves be corrupted.

## Verifying it holds

```console
root@target:~# touch /test
touch: cannot touch '/test': Read-only file system
root@target:~# findmnt -t squashfs,ext4,tmpfs,overlay -o TARGET,FSTYPE,OPTIONS
TARGET     FSTYPE   OPTIONS
/          squashfs ro,relatime
/data      ext4     rw,noatime,errors=remount-ro
/tmp       tmpfs    rw,nosuid,nodev,noatime,size=32768k
/etc       overlay  rw,relatime,lowerdir=/etc,...
root@target:~# grep " ro," /proc/mounts
/dev/mmcblk0p2 / squashfs ro,relatime 0 0
```

Then the test that actually matters: pull the power a few hundred times
under load and confirm the unit always boots. A soak rig that power-cycles
on a timer, with a script asserting the boot completed, finds problems no
code review will.

## Traps

!!! danger "Read-only rootfs traps"
    - **A service that fails silently because it cannot write.** Some daemons
      log the EROFS and carry on degraded rather than exiting, so the unit
      shows `active (running)` while doing nothing useful. Check the journal
      for `Read-only file system` after the first boot of every new image.
    - **`/etc/machine-id` regenerated every boot.** It ends up in tmpfs, so
      journald creates a new log directory each boot, D-Bus identity changes,
      and any server-side device identity keyed on it breaks. Persist it on
      `/data` explicitly.
    - **Package manager on a read-only rootfs.** `opkg`/`rpm` cannot work.
      That is intended — updates become image-based (Level 4) — but it
      surprises people who expect to hotfix a unit in the field.
    - **Factory reset that also wipes calibration data.** Per-unit calibration,
      certificates and the serial number are *not* user data. Store them
      outside `/data` (a separate small partition or a protected subdirectory
      the reset script skips) or your reset bricks the product functionally.
    - **Power loss during the reset itself.** The marker file must only be
      removed after the reformat succeeds, and the reset must be idempotent —
      interrupt it and the next boot simply starts over.
    - **`/var/log` on tmpfs with no size cap** can consume all RAM and trigger
      the OOM killer. Always set `size=`.
    - **Testing on a rootfs that is read-only only in fstab.** If the kernel
      cmdline says `rw`, an initramfs may remount it writable before fstab is
      ever read. Check `/proc/mounts`, not your intentions.

## Cheat sheet

| Item | Purpose |
|---|---|
| `root=PARTLABEL=rootfs ro rootwait` | Kernel cmdline — mount `/` read-only |
| squashfs for `/` | Immutable, compressed, cannot be corrupted by writes |
| `nofail` on the data partition | Boot even when `/data` is unmountable |
| `errors=remount-ro` | Fail loudly on data-partition corruption |
| tmpfs for `/tmp`, `/run`, `/var/log` | Volatile state, zero flash wear (cap `size=`) |
| overlayfs on `/etc` | Writable `/etc` whose deltas live on `/data` |
| symlink `/etc/X → /data/etc/X` | Simpler alternative to an overlay |
| `DefaultDependencies=no` + `Before=sysinit.target` | Run a unit early enough to mount |
| `RequiresMountsFor=/data` | Ordering against a mount, correctly |
| `/usr/share/factory/` | Read-only defaults to restore on reset |
| `ConditionPathExists=` | Run a unit only when a marker file is present |
| `findmnt` / `grep " ro," /proc/mounts` | Prove what is actually mounted read-only |
| `find / -xdev -newer <ref>` | Discover what the system writes to `/` |
| `mkfs.ext4 -F` on `/data` | Factory reset: reformat, don't `rm -rf` |
| Power-cycle soak test | The only real proof the design works |

!!! note "On verification"
    The unit files, mount options and overlay invocation follow the documented
    systemd and overlayfs rules — `DefaultDependencies=no` with explicit
    `Before=`/`After=` ordering, `upperdir`/`workdir` on one filesystem,
    `ConditionPathExists=` gating. They were not booted on a target while
    writing this page; run `systemd-analyze verify` on each unit and check
    `/proc/mounts` on your own image before trusting the layout.

## Exercise

(1) Convert your QEMU image to a read-only root: change the kernel cmdline
to `ro`, add the fstab above, boot, and use
`find / -xdev -newer /etc/os-release` plus the journal to build the complete
list of things that tried to write to `/`. (2) Fix that list two ways — once
with an `/etc` overlay unit, once with targeted symlinks into `/data` — and
write two sentences on which you would ship and why. (3) Implement
`factory-reset.service`, trigger it by touching the marker file, and prove
it is idempotent by killing QEMU partway through the reformat and rebooting.
(4) One paragraph: your product stores per-unit calibration constants, a TLS
client certificate, user settings and 30 days of logs. Assign each to a
mount point in the layout above, state whether factory reset destroys it,
and justify the two decisions most likely to be argued about in review.
