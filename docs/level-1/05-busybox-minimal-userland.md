# 05 · BusyBox & the Minimal Userland

Log into an embedded device and it *feels* like Linux — `ls`, `ps`, `mount`
all work — but almost nothing is what it seems: there's no `apt`, no
`bash`, and every command is secretly the *same program*. This module is a
guided spelunk through a real minimal userland. Boot your module-4 Alpine
guest (login `root`) and follow along — Alpine's userland **is BusyBox**,
the exact same software running in millions of routers and industrial
devices, and in every image you'll build with Buildroot later.

## Why an embedded rootfs is not Ubuntu

A desktop distro optimizes for *generality*: any user, any package, any
hardware — costing gigabytes of disk and hundreds of daemons. An embedded
rootfs optimizes for *the product*: it ships only what the device needs,
because flash costs money, every binary is attack surface, every daemon is
boot time and RAM, and — most fundamentally — **nobody installs packages on
a shipped device**; you replace the whole image (that's what OTA updates
are, Level 4). Compare the numbers from your guest:

```console
localhost:~# df -h /
Filesystem                Size      Used Available Use% Mounted on
/dev/loop0               47.4M     47.4M         0 100% /.modloop
localhost:~# ls /bin /sbin /usr/bin | wc -l
400
```

A whole operating system in tens of megabytes. Ubuntu's `/usr/bin` alone is
~100× larger.

## BusyBox: 300 commands, one binary

Look closely at the "commands" on the system:

```console
localhost:~# ls -l /bin/ls /bin/cat /sbin/init
lrwxrwxrwx  1 root  root  12 ... /bin/ls -> /bin/busybox
lrwxrwxrwx  1 root  root  12 ... /bin/cat -> /bin/busybox
lrwxrwxrwx  1 root  root  17 ... /sbin/init -> /bin/busybox
```

`ls`, `cat` — even `init`, PID 1 itself — are all **symlinks to one
~800 KB binary**. BusyBox looks at the name it was invoked as (`argv[0]`)
and behaves like that tool. Each personality is called an **applet**. See
them all:

```console
localhost:~# busybox | tail -n +5 | head -6
Currently defined functions:
        [, [[, acpid, add-shell, addgroup, adduser, adjtimex, ar, arch,
        arp, arping, ash, awk, base64, basename, bc, beep, blkdiscard,
        ...
localhost:~# busybox uname -m     # any applet also runs directly like this
aarch64
```

Two consequences you'll hit in real work:

- **The shell is `ash`, not `bash`.** BusyBox applets are deliberately
  lean POSIX implementations — bash-isms like arrays and `[[ ]]` may be
  missing or partial, and flags you know from GNU tools may not exist
  (`grep -P`, `ls --color=auto`...). Write POSIX-portable scripts
  (`#!/bin/sh`) for embedded targets.
- **Which applets exist is a build-time menu.** When you build BusyBox
  (via Buildroot in module 9), you tick applets on and off — a device that
  doesn't need `awk` simply doesn't ship it.

## Spelunking /proc — the kernel's live status board

`/proc` is not a real disk directory: it's a **virtual filesystem** the
kernel generates on the fly. Reading its files *is* asking the kernel
questions — and on a minimal device with no monitoring tools, it's often
the only instrumentation you have:

```console
localhost:~# cat /proc/uptime
124.51 243.80                      # seconds up, seconds idle (sum of CPUs)
localhost:~# grep -E "MemTotal|MemFree|MemAvailable" /proc/meminfo
MemTotal:         504524 kB
MemFree:          411924 kB
MemAvailable:     441808 kB
localhost:~# cat /proc/loadavg
0.00 0.01 0.00 1/62 1130
localhost:~# cat /proc/1/cmdline | tr '\0' ' '; echo
/sbin/init
localhost:~# cat /proc/cmdline    # the bootargs from module 3!
BOOT_IMAGE=/boot/vmlinuz-virt initrd=/boot/initramfs-virt console=ttyAMA0 ...
```

Per-process directories (`/proc/<pid>/`) expose each process's command
line, open files (`fd/`), and memory maps (`maps`) — `ps` and `top` are
just pretty-printers over them:

```console
localhost:~# ps | head -4
PID   USER     TIME  COMMAND
    1 root      0:00 /sbin/init
    2 root      0:00 [kthreadd]        # [brackets] = kernel thread
localhost:~# top -bn1 | head -5        # BusyBox top, batch mode
```

## Spelunking /sys — the hardware tree

`/sys` (sysfs) is the kernel's *hardware and driver model* exported as
files — where `/proc` says "what is running," `/sys` says "what devices
exist and how they're configured." This is the userspace face of the device
tree you'll meet in module 7:

```console
localhost:~# ls /sys/class/net/
eth0  lo
localhost:~# cat /sys/class/net/eth0/address
52:54:00:12:34:56                  # QEMU's default MAC
localhost:~# ls /sys/devices/system/cpu/ | grep "^cpu[0-9]"
cpu0
cpu1
localhost:~# cat /sys/firmware/devicetree/base/model; echo
linux,dummy-virt                   # the device tree, live — module 7 preview
```

On a real board, sysfs is also where you *control* hardware from the shell
— brightness, LED triggers, GPIO. There's simply less hardware inside
QEMU's `virt` machine.

## mount — what filesystems are actually in play

```console
localhost:~# mount | grep -Ev "cgroup" 
/dev/loop0 on /.modloop type squashfs (ro,relatime)
tmpfs on / type tmpfs (rw,relatime,mode=755)
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
sysfs on /sys type sysfs (rw,nosuid,nodev,noexec,relatime)
devtmpfs on /dev type devtmpfs (rw,nosuid,relatime,...)
tmpfs on /tmp type tmpfs (rw,nosuid,nodev,relatime)
```

Read that like an embedded engineer: the root is a **tmpfs** (RAM — this is
a live image), kernel modules come from a **squashfs** (compressed,
read-only — a classic embedded choice), and `/proc`, `/sys`, `/dev` are all
virtual. Real products use the same building blocks: a read-only squashfs
root plus a small writable data partition is one of the most common
production layouts (Level 2 goes deep on this).

## Cheat sheet

| Command / path | Purpose |
|----------------|---------|
| `busybox` | List every applet compiled into this build |
| `busybox <applet> ...` | Run an applet directly, no symlink needed |
| `ls -l /bin/ls` | Reveal the symlink-to-busybox trick |
| `/bin/sh` → `ash` | The BusyBox shell — write POSIX, not bash-isms |
| `/proc/meminfo`, `/proc/uptime`, `/proc/loadavg` | RAM, uptime, load — no tools needed |
| `/proc/cmdline` | The kernel bootargs (from U-Boot / the loader) |
| `/proc/<pid>/cmdline`, `fd/`, `maps` | Per-process introspection |
| `/sys/class/net/`, `/sys/class/leds/` | Hardware by category |
| `/sys/firmware/devicetree/base/` | The live device tree (ARM) |
| `mount`, `df -h` | What's mounted, from where, ro or rw |

## Exercise

Inside the guest, answer each question **using only `/proc` and `/sys`
reads** (no `ps`, `top`, `free`, or `ip`): (1) exactly how many kB of RAM
does the kernel consider available? (2) what is PID 1's executable path?
(3) how many CPUs does the system have, and what CPU part number (hex) are
they? (4) what is `eth0`'s MAC address and link state
(`/sys/class/net/eth0/operstate`)? (5) what device tree model string was
the kernel booted with? Then run `busybox | grep -o "wget\|vi\|awk\|tar"`
to check which of those four tools your build includes, and — thinking like
a product engineer — name one applet you'd *remove* from a production
image, and why.
