# 09 · systemd Deep Dive

Level 1 introduced systemd as "the init system that starts your service".
That is the tutorial version. In a product, systemd is the thing that
decides whether a failed daemon takes the device down, whether a hung board
recovers by itself at 3 a.m., and whether your logs destroy the flash. This
module is about those decisions: dependency and ordering semantics, restart
policy, activation, timers, watchdogs, and journald sizing.

## Dependencies and ordering are separate axes

The single most common systemd mistake is assuming `Requires=` implies
ordering. It does not.

| Directive | Meaning |
|---|---|
| `Requires=b.service` | If b fails to start, a is not started. **No ordering.** |
| `Wants=b.service` | Try to start b; carry on regardless. No ordering. |
| `BindsTo=b.service` | Like Requires, and a is *stopped* if b stops |
| `After=b.service` | Ordering only: start a after b. **No dependency.** |
| `Before=b.service` | Ordering only, the other direction |
| `Conflicts=b.service` | Starting a stops b |

You almost always want **both**: `Requires=` (or `Wants=`) *and* `After=`.
`Requires=` alone starts both units in parallel and your daemon races the
thing it depends on — a bug that reproduces on 1 boot in 50, only on the
fastest hardware.

`Wants=` is the right default for anything optional, because a `Requires=`
chain propagates failure further than people expect.

## A production service unit

```ini
# /etc/systemd/system/appd.service
[Unit]
Description=Appliance control daemon
Documentation=man:appd(8)
Wants=network-online.target
After=network-online.target data.mount
RequiresMountsFor=/data
StartLimitIntervalSec=300
StartLimitBurst=5

[Service]
Type=notify
ExecStart=/usr/sbin/appd --config /data/etc/appd.conf
ExecReload=/bin/kill -HUP $MAINPID
Restart=always
RestartSec=5s
TimeoutStartSec=30s
TimeoutStopSec=20s

WatchdogSec=30s

User=appd
Group=appd
RuntimeDirectory=appd
StateDirectory=appd

# Hardening — cheap, and each line removes a class of exploit
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
ReadWritePaths=/data/appd
RestrictSUIDSGID=yes
MemoryMax=64M

[Install]
WantedBy=multi-user.target
```

Points worth dwelling on:

- **`Type=`** decides when systemd considers the service "up".
  `simple` (default) means "immediately after fork" — almost always wrong,
  because dependent units start before you are ready. `forking` needs
  `PIDFile=`. `notify` is correct: the daemon calls `sd_notify(0, "READY=1")`
  when it is genuinely serving. `oneshot` + `RemainAfterExit=yes` is for
  setup tasks.
- **`StartLimitIntervalSec`/`StartLimitBurst`** cap the restart loop. Five
  failures in five minutes and systemd gives up and leaves the unit
  `failed`. Without this, `Restart=always` on a daemon that crashes at
  startup spins the CPU forever and writes a journal entry every time.
- **`ProtectSystem=strict`** makes the entire filesystem read-only for this
  service except what `ReadWritePaths=` allows. On a read-only rootfs
  (module 8) this costs nothing and is nearly free defence in depth.
- **`StateDirectory=appd`** creates and owns `/var/lib/appd` with the right
  permissions — no `mkdir` in an `ExecStartPre=`.

```console
root@target:~# systemd-analyze verify /etc/systemd/system/appd.service
root@target:~# systemctl daemon-reload && systemctl enable --now appd
root@target:~# systemctl status appd
● appd.service - Appliance control daemon
     Loaded: loaded (/etc/systemd/system/appd.service; enabled)
     Active: active (running) since Mon 2026-08-04 09:02:11 UTC; 3min ago
   Main PID: 310 (appd)
     Status: "Polling 4 sensors"
     Memory: 18.2M (max: 64.0M)
```

`systemd-analyze verify` catches typos, unknown directives and missing
dependencies without starting anything — run it in CI on every unit you
ship.

## Targets and boot time

```console
root@target:~# systemctl get-default
multi-user.target
root@target:~# systemd-analyze
Startup finished in 1.412s (kernel) + 2.980s (userspace) = 4.392s
multi-user.target reached after 2.901s in userspace
root@target:~# systemd-analyze blame | head -5
1.204s appd.service
 812ms systemd-udev-settle.service
 344ms systemd-networkd-wait-online.service
root@target:~# systemd-analyze critical-chain appd.service
multi-user.target @2.901s
└─appd.service @1.697s +1.204s
  └─network-online.target @1.688s
```

`blame` shows the slowest units; `critical-chain` shows the ones that
actually delayed boot — a unit can be slow and irrelevant. On embedded,
`systemd-udev-settle.service` and `*-wait-online` are the usual culprits,
and both are usually removable once you express your real dependency
(a specific device unit, or nothing at all).

## Socket and path activation

Activation lets a service not run until it is needed — which on a
memory-constrained board is the difference between 12 and 20 resident
daemons.

```ini
# /etc/systemd/system/diagnostics.socket
[Unit]
Description=Diagnostics API socket

[Socket]
ListenStream=127.0.0.1:9000
Accept=no

[Install]
WantedBy=sockets.target
```

```ini
# /etc/systemd/system/diagnostics.service
[Unit]
Description=Diagnostics API
Requires=diagnostics.socket
After=diagnostics.socket

[Service]
ExecStart=/usr/sbin/diagd
StandardInput=socket
```

systemd holds the listening socket from boot; the first connection starts
`diagd`. Connections made while it starts are queued, not refused — so
there is no race for the client either.

Path activation reacts to the filesystem:

```ini
# /etc/systemd/system/config-reload.path
[Path]
PathChanged=/data/etc/appd.conf
Unit=config-reload.service

[Install]
WantedBy=multi-user.target
```

## Timers instead of cron

```ini
# /etc/systemd/system/log-rotate.timer
[Unit]
Description=Rotate application logs

[Timer]
OnCalendar=daily
Persistent=true
RandomizedDelaySec=900
AccuracySec=1min

[Install]
WantedBy=timers.target
```

```console
root@target:~# systemctl list-timers --all
NEXT                        LEFT     UNIT             ACTIVATES
Tue 2026-08-05 00:11:43 UTC 14h left log-rotate.timer log-rotate.service
root@target:~# systemd-analyze calendar "daily"
  Normalized form: *-*-* 00:00:00
    Next elapse: Tue 2026-08-05 00:00:00 UTC
```

Three things cron cannot do and a fleet needs. `Persistent=true` runs a
missed job after a device that was powered off comes back. `RandomizedDelaySec`
stops ten thousand devices hitting your server in the same second.
`OnBootSec=`/`OnUnitActiveSec=` express "5 minutes after boot, then every
hour" without arithmetic. And the job's output goes to the journal
automatically instead of a mail spool that does not exist.

## Watchdogs

Two layers, and you want both.

**Software watchdog** — systemd kills and restarts a service that stops
pinging it. `WatchdogSec=30s` in the unit, plus in the daemon:

```c
/* Call at least twice per WatchdogSec interval. */
sd_notify(0, "WATCHDOG=1");
```

Only ping from the code path that proves the daemon is *working* — pinging
from a timer thread while the main loop is deadlocked is a watchdog that
reports health it does not have.

**Hardware watchdog** — the SoC resets the board if the kernel itself
hangs. systemd drives `/dev/watchdog`:

```text
# /etc/systemd/system.conf.d/watchdog.conf
[Manager]
RuntimeWatchdogSec=60s
RebootWatchdogSec=10min
ShutdownWatchdogSec=10min
```

```console
root@target:~# wdctl
Device:        /dev/watchdog0
Identity:      imx2+ watchdog [version 0]
Timeout:       60 seconds
Pre-timeout:    0 seconds
root@target:~# journalctl -b -1 -n 5      # what the *previous* boot said
```

`journalctl -b -1` is the first command to run after an unexplained reboot:
if the last boot's log just stops mid-sentence, the watchdog fired.

## journald on flash

Default journald settings will happily write hundreds of MB. On eMMC that
is module 6's failure mode.

```text
# /etc/systemd/journald.conf.d/embedded.conf
[Journal]
Storage=volatile
RuntimeMaxUse=16M
RuntimeMaxFileSize=2M
MaxLevelStore=info
ForwardToSyslog=no
RateLimitIntervalSec=30s
RateLimitBurst=200
```

`Storage=volatile` keeps the journal in `/run` (RAM) — nothing touches
flash, and logs are lost on reboot. That is the right default for most
appliances, paired with shipping important events off-device. If you must
persist, use `Storage=persistent` with `SystemMaxUse=32M` and put
`/var/log/journal` on the data partition, never the rootfs.

## Traps

!!! danger "systemd traps"
    - **`Requires=` without `After=`.** Parallel start, intermittent race.
      This is the classic.
    - **`Restart=always` with no start limit.** A daemon that crashes on
      startup restarts forever, pins a core, and floods the journal — which,
      if the journal is persistent, then fills the flash.
    - **`Type=simple` for something that needs to be ready.** Dependents start
      against a service that has not opened its socket yet.
    - **A watchdog pinged by the wrong thread** hides exactly the hangs it
      exists to catch.
    - **`RuntimeWatchdogSec` shorter than your slowest legitimate operation**
      reboots healthy devices during, say, a firmware update. Coordinate the
      value with every long-running task, and disable the watchdog explicitly
      around updates.
    - **Editing units under `/lib/systemd/system`** on a read-only rootfs is
      impossible, and on a writable one gets overwritten by the next update.
      Use `/etc/systemd/system/` or `systemctl edit`.
    - **Forgetting `systemctl daemon-reload`** after changing a unit — systemd
      keeps running the old definition and you debug a file that is not in
      effect.
    - **`ProtectSystem=strict` without `ReadWritePaths=`** makes a
      previously-working service fail with EROFS in a way whose error message
      points at the application, not the unit.

## Cheat sheet

| Item | Purpose |
|---|---|
| `Requires=` + `After=` | Dependency **and** ordering — you need both |
| `Wants=` | Optional dependency; failure does not propagate |
| `BindsTo=` | Stop this unit when the other stops (device units) |
| `Type=notify` + `sd_notify(READY=1)` | Accurate "service is up" signalling |
| `Restart=always`, `RestartSec=` | Restart policy and backoff |
| `StartLimitIntervalSec`/`StartLimitBurst` | Cap the restart loop |
| `WatchdogSec=` + `WATCHDOG=1` | Per-service software watchdog |
| `RuntimeWatchdogSec=` in system.conf | Hardware watchdog via `/dev/watchdog` |
| `ProtectSystem=strict`, `ReadWritePaths=` | Filesystem hardening |
| `StateDirectory=`, `RuntimeDirectory=` | Managed `/var/lib/X`, `/run/X` |
| `.socket` + `StandardInput=socket` | Socket activation — start on first use |
| `.path` + `PathChanged=` | Run a unit when a file changes |
| `.timer`, `Persistent=`, `RandomizedDelaySec=` | cron replacement for fleets |
| `systemd-analyze verify \| blame \| critical-chain` | Lint / slowest / boot-critical |
| `systemctl edit <unit>` | Override a vendor unit correctly |
| `journalctl -b -1 -n 50` | Previous boot's log — did the watchdog fire? |
| `journald.conf`: `Storage=volatile`, `RuntimeMaxUse=` | Keep logs off the flash |

!!! note "On verification"
    Every directive above is used per its documented semantics in
    `systemd.unit(5)`, `systemd.service(5)`, `systemd.timer(5)`,
    `systemd.socket(5)` and `journald.conf(5)`, and the units are written to
    pass `systemd-analyze verify`. They were not booted on a target while
    writing this page — run `systemd-analyze verify` yourself after copying
    them, since directive availability varies with systemd version.

## Exercise

(1) Write `appd.service` as above for any small daemon, run
`systemd-analyze verify` on it, then deliberately remove the `After=` line
and describe a boot scenario in which the service now fails intermittently.
(2) Convert the service to `Type=notify` (a five-line `sd_notify` call is
enough) and show, with `systemctl status` and `systemd-analyze
critical-chain`, that a dependent unit now starts at the right moment.
(3) Give it `WatchdogSec=10s`, stop pinging deliberately, and capture the
journal lines showing systemd killing and restarting it; then add
`StartLimitBurst=3` and show the unit going to `failed` instead of looping.
(4) One paragraph: your device must apply a firmware update that takes up
to four minutes with the storage busy, while a 60-second hardware watchdog
is armed. Describe exactly how you would keep the watchdog protection for
normal operation without the update rebooting the device halfway through —
and what the failure mode of your scheme is if the update itself hangs.
