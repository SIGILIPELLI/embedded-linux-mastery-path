# 10 · Capstone — Tiny Linux Appliance

Every module so far taught one piece of the stack in isolation. This
capstone assembles all of them into one thing: a small, working "embedded
Linux appliance" — a Buildroot-built QEMU image running a cross-compiled C
sensor-simulator daemon, wired into init two different ways, with a
boot-time status banner and a shell health-check script. It is, in
miniature, exactly the deliverable a bring-up engineer hands off on a real
board. The stretch section maps every single piece onto an actual i.MX95
product bring-up, so you leave this level able to say precisely what
changes when the hardware becomes real.

## What you're building

```text
┌────────────────────────────────────────────────────┐
│ Buildroot rootfs.ext4  (module 9)                   │
│  ├─ /usr/sbin/sensord      ← cross-compiled C daemon│
│  ├─ /usr/bin/healthcheck.sh                         │
│  ├─ init wiring: BusyBox inittab respawn line        │
│  │   (variant B: systemd unit, same image family)   │
│  └─ boot banner: "Tiny Appliance vX — sensord: OK"  │
└────────────────────────────────────────────────────┘
       booted with: qemu-system-aarch64 -M virt ...
```

## Step 1 — Start from your module-9 Buildroot tree

```console
$ cd buildroot
$ make qemu_aarch64_virt_defconfig
$ make menuconfig    # keep dropbear from module 9; leave everything else default
```

## Step 2 — Write the sensor-simulator daemon (C, module-6 skills)

```c
/* sensord.c — cross-compiled sensor simulator daemon */
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <time.h>
#include <signal.h>
#include <sys/stat.h>

static volatile sig_atomic_t running = 1;
static void on_term(int sig) { (void)sig; running = 0; }

int main(void) {
    signal(SIGTERM, on_term);
    signal(SIGINT, on_term);

    const char *pidfile = "/var/run/sensord.pid";
    FILE *pf = fopen(pidfile, "w");
    if (pf) { fprintf(pf, "%d\n", getpid()); fclose(pf); }

    const char *logfile = "/var/log/sensord.log";
    int reading = 20;
    while (running) {
        FILE *f = fopen(logfile, "a");
        time_t now = time(NULL);
        reading = 20 + (int)(now % 5);          /* fake temp, 20-24 C */
        if (f) {
            fprintf(f, "[%ld] temp=%dC status=OK\n", (long)now, reading);
            fclose(f);
        }
        sleep(2);
    }

    remove(pidfile);
    return 0;
}
```

Cross-compile it with Buildroot's own toolchain — the module-6 lesson
about matching sysroots, now automatic:

```console
$ output/host/bin/aarch64-buildroot-linux-gnu-gcc -O2 -o sensord sensord.c
$ output/host/bin/aarch64-buildroot-linux-gnu-readelf -d sensord | grep NEEDED
 0x0000000000000001 (NEEDED)  Shared library: [libc.so.6]
```

## Step 3 — Write the health-check shell script (module-5 skills, POSIX sh)

```sh
#!/bin/sh
# /usr/bin/healthcheck.sh — is sensord alive and logging?
PIDFILE=/var/run/sensord.pid
LOGFILE=/var/log/sensord.log

if [ ! -f "$PIDFILE" ]; then
    echo "FAIL: no pidfile"; exit 1
fi
PID=$(cat "$PIDFILE")
if ! kill -0 "$PID" 2>/dev/null; then
    echo "FAIL: sensord pid $PID not running"; exit 1
fi
LAST=$(tail -1 "$LOGFILE" 2>/dev/null)
if [ -z "$LAST" ]; then
    echo "FAIL: no log entries yet"; exit 1
fi
echo "OK: sensord pid=$PID last='$LAST'"
exit 0
```

## Step 4 — Get both files into the image with a rootfs overlay

Buildroot's `BR2_ROOTFS_OVERLAY` copies a directory tree straight onto the
built rootfs — the cleanest way to inject your own files without writing a
full package:

```console
$ mkdir -p board/myappliance/overlay/usr/sbin
$ mkdir -p board/myappliance/overlay/usr/bin
$ cp sensord board/myappliance/overlay/usr/sbin/sensord
$ cp healthcheck.sh board/myappliance/overlay/usr/bin/healthcheck.sh
$ chmod +x board/myappliance/overlay/usr/sbin/sensord \
           board/myappliance/overlay/usr/bin/healthcheck.sh
```

In `make menuconfig` → System configuration → **Root filesystem
overlay directories**, set:

```text
board/myappliance/overlay
```

## Step 5 — Init integration, variant A: BusyBox inittab (respawn)

Still in System configuration, edit the **inittab** the build will
generate, or append after the build via the overlay
(`board/myappliance/overlay/etc/inittab.append` merged manually, or
simplest: add the line directly to
`board/myappliance/overlay/etc/inittab` if you supply a full custom
inittab). The one line that matters:

```text
::respawn:/usr/sbin/sensord
```

This is the module-8 pattern exactly: crash `sensord`, init restarts it,
your product stays alive.

## Step 6 — Init integration, variant B: systemd unit

If you rebuild the same image with systemd enabled (Target packages →
select `systemd` as the init system, in place of BusyBox init), ship this
unit instead via the overlay at `etc/systemd/system/sensord.service`:

```ini
[Unit]
Description=Sensor simulator daemon
After=local-fs.target

[Service]
ExecStart=/usr/sbin/sensord
Restart=always
RestartSec=1

[Install]
WantedBy=multi-user.target
```

And enable it at build time by symlinking it into
`etc/systemd/system/multi-user.target.wants/` inside the overlay (the
same effect `systemctl enable` has at runtime).

## Step 7 — Boot-time status banner

A one-line motd-style banner, set via the overlay file
`etc/issue` (BusyBox getty prints this before every login prompt):

```text
Tiny Appliance v1.0 — sensord: check with `healthcheck.sh`
```

## Step 8 — Build and boot

```console
$ make -j$(nproc)
$ qemu-system-aarch64 \
    -M virt -cpu cortex-a53 -smp 2 -m 512M \
    -kernel output/images/Image \
    -append "root=/dev/vda console=ttyAMA0" \
    -drive file=output/images/rootfs.ext4,if=none,format=raw,id=hd0 \
    -device virtio-blk-device,drive=hd0 \
    -netdev user,id=net0 -device virtio-net-device,netdev=net0 \
    -nographic
```

**Expected output**, in order:

```text
Booting Linux on physical CPU 0x0000000000 ...
[    2.xx] Run /sbin/init as init process
Tiny Appliance v1.0 — sensord: check with `healthcheck.sh`
buildroot login: root
# healthcheck.sh
OK: sensord pid=187 last='[1737500001] temp=22C status=OK'
# kill $(cat /var/run/sensord.pid)
# sleep 3
# healthcheck.sh
OK: sensord pid=203 last='[1737500009] temp=24C status=OK'   # new PID = respawned
```

That last block — kill it, watch a new PID come back, health-check passes
again — **is the whole capstone working correctly.**

## Stretch — mapping this onto a real i.MX95 bring-up

Everything above targeted QEMU's generic `virt` machine. Here is exactly
what changes to run the same appliance on a real i.MX95 board — nothing
about `sensord.c` or `healthcheck.sh` changes at all:

| Piece | This capstone (QEMU) | Real i.MX95 board |
|-------|----------------------|--------------------|
| **Bootloader** | QEMU's built-in `virt` boot path (module 4) | i.MX95 boot ROM (fuses select eMMC/SD) → SPL → U-Boot, signed if HAB/AHAB secure boot is enabled (module 3, Level 4) |
| **Device tree** | QEMU generates a generic `virt` DTB (module 7) | `imx95-<board>.dtb`, built from NXP's `imx95.dtsi` + your board file, enabling exactly the peripherals your board wires up |
| **Kernel** | Buildroot's generic `qemu_aarch64_virt` kernel defconfig | NXP's BSP kernel tree/config (or mainline + i.MX95 defconfig fragments), likely via **Yocto's meta-imx** layer rather than raw Buildroot (module 9's decision table) |
| **Rootfs build** | Buildroot, one defconfig | Very likely Yocto, for multi-layer BSP integration and long-term maintenance (Level 2) |
| **This daemon (`sensord`)** | Reads `time()` for a fake reading | Reads a **real** sensor over I2C/SPI, or receives readings from the **Cortex-M7** core over RPMsg (Level 3) instead of faking them in C |
| **Init system** | Your choice (inittab shown, systemd variant given) | Production i.MX BSP images default to **systemd** (module 8) — ship the unit-file variant |
| **Storage** | Single `ext4` disk image | Real eMMC partitioning: read-only squashfs rootfs + a writable data partition is typical (Level 2's storage module) |
| **Networking to check on it** | `dropbear` SSH over QEMU user-mode NAT | Same `dropbear`/OpenSSH over the board's real Ethernet or the i.MX95's wireless options |
| **What's identical** | The C code, the shell script, the systemd unit, the inittab respawn line, the health-check logic | **Unchanged** — this is the entire point of cross-compilation and portable init config |

## Cheat sheet

| Artifact | Role |
|----------|------|
| `sensord.c` | Cross-compiled C daemon — module 6's skill, product-shaped |
| `healthcheck.sh` | POSIX `/bin/sh` health probe — module 5's shell literacy |
| `BR2_ROOTFS_OVERLAY` | Buildroot mechanism for injecting your own files into the rootfs |
| `::respawn:/usr/sbin/sensord` | BusyBox init variant — module 8 |
| `sensord.service` (`Restart=always`) | systemd variant — module 8 |
| `/etc/issue` | Boot-time banner shown before login |
| `kill $(cat /var/run/sensord.pid)` then re-check | Proves the respawn contract works |

## Exercise

Take this capstone one step further, in the QEMU image you already have:
(1) make `sensord` write its reading to `/sys`-style location isn't
possible from userspace, so instead have it also append one line to
`/tmp/sensord_history.csv` (`timestamp,temp` — CSV, for later analysis);
(2) modify `healthcheck.sh` to **exit 2** (a distinct code) if the last
log line is older than 10 seconds — simulating a hung daemon detector,
and test it by pausing sensord with `kill -STOP $(cat /var/run/sensord.pid)`
then `kill -CONT` to resume; (3) write a two-paragraph bring-up note, as
if handing this off to a hardware team getting the first i.MX95 prototype
boards: what stays exactly the same from this capstone, and what is the
*first* thing you'd verify once real hardware powers on (hint: revisit
module 3's boot relay — where would you attach a serial cable, and what's
the very first line of output you'd expect to see before your own daemon
ever runs)?
