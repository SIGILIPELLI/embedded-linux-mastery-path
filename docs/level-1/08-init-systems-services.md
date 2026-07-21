# 08 · Init Systems & Services

An embedded device has no user to double-click your program. Whatever your
product *does* — reading sensors, serving a UI, talking to the cloud — some
piece of software must start it at boot, restart it when it crashes, and
stop it cleanly at shutdown. That piece is the **init system**, the very
PID 1 from module 3. Embedded Linux really uses two: tiny **BusyBox init**
(a dozen lines of config, the Buildroot default) and full **systemd** (what
production i.MX BSP images actually ship). A professional writes the same
service for both without blinking — so that's exactly what you'll do here,
with the sensor daemon pattern the capstone will reuse.

## BusyBox init: the whole system in one file

BusyBox init reads a single config, **`/etc/inittab`**, of lines
`id::action:command`. Your Alpine guest uses it as PID 1 — look at the
real thing:

```console
localhost:~# ps -p 1 -o comm=
init
localhost:~# grep -v "^#" /etc/inittab
::sysinit:/sbin/openrc sysinit
::sysinit:/sbin/openrc boot
::wait:/sbin/openrc default
tty1::respawn:/sbin/getty 38400 tty1
ttyAMA0::respawn:/sbin/getty -L 115200 ttyAMA0 vt100
::shutdown:/sbin/openrc shutdown
::ctrlaltdel:/sbin/reboot
```

The three actions that matter:

| Action | Meaning |
|--------|---------|
| `sysinit` / `wait` | Run once at boot, in order (mount filesystems, one-shot setup) |
| `respawn` | Start the command **and restart it whenever it exits** — the embedded watchdog-lite |
| `shutdown` | Run at poweroff/reboot |

That `ttyAMA0::respawn:...getty` line *is* your serial login prompt: log
out, and init respawns getty — which is why the prompt comes back.
(Alpine layers the OpenRC service manager on top for its boot scripts —
a common middle ground — but PID 1 itself is pure BusyBox inittab.)

### A service the BusyBox way

Take a trivial daemon (the capstone's sensor simulator in miniature):

```sh
#!/bin/sh
# /usr/sbin/sensord-lite — fake sensor: log a reading every 2 s
while true; do
    echo "sensord: temp=$((20 + $(date +%S) % 5)).${$(date +%N):-0}C" \
        >> /tmp/sensord.log 2>/dev/null || true
    echo "sensord: reading logged $(date)" >> /tmp/sensord.log
    sleep 2
done
```

Simpler, portable version to actually use:

```sh
#!/bin/sh
# /usr/sbin/sensord-lite
while true; do
    echo "$(date '+%H:%M:%S') temp=$((20 + $(cat /proc/uptime | cut -d. -f1) % 5))C" \
        >> /tmp/sensord.log
    sleep 2
done
```

Wiring it in is **one line** in `/etc/inittab`:

```text
::respawn:/usr/sbin/sensord-lite
```

Try it live in the guest:

```console
localhost:~# cat > /usr/sbin/sensord-lite   # paste the script, then Ctrl-D
localhost:~# chmod +x /usr/sbin/sensord-lite
localhost:~# echo "::respawn:/usr/sbin/sensord-lite" >> /etc/inittab
localhost:~# kill -HUP 1                    # tell init to re-read inittab
localhost:~# tail -2 /tmp/sensord.log
12:04:11 temp=22C
12:04:13 temp=22C
localhost:~# kill $(pgrep -f sensord-lite) && sleep 1 && pgrep -f sensord-lite
1201                                        # new PID — init respawned it!
```

That respawn is the entire crash-recovery story of countless shipped
devices. Its limits are real though: no dependencies ("start after
network"), no shutdown ordering, no per-service logging — you get to about
five services and start reinventing systemd badly.

## systemd: what production BSPs ship

**systemd** models services as declarative **unit files** with
dependencies, restart policy, logging, watchdogs, and resource limits.
It's heavier (RAM, image size, boot complexity) — and it's what NXP's
Yocto-based i.MX BSP images use by default, because products need exactly
those features. The same service, the systemd way:

```ini
# /etc/systemd/system/sensord.service
[Unit]
Description=Sensor simulator daemon
After=network.target

[Service]
ExecStart=/usr/sbin/sensord-lite
Restart=always
RestartSec=1

[Install]
WantedBy=multi-user.target
```

And its lifecycle commands (run these on any systemd machine — your Linux
PC, a Raspberry Pi, or the capstone's systemd image):

```console
$ sudo systemctl daemon-reload          # re-read unit files
$ sudo systemctl enable --now sensord   # start at boot + start now
$ systemctl status sensord
● sensord.service - Sensor simulator daemon
     Active: active (running) since ...
$ journalctl -u sensord -n 3            # this unit's logs only
```

Line-by-line against inittab: `ExecStart` is the command,
`Restart=always` is `respawn`, `WantedBy=multi-user.target` is "part of
normal boot," and `After=network.target` is the dependency line inittab
simply cannot express.

## Logging: syslog vs journald

Where do a device's logs *go*? Two answers, matching the two inits:

| | BusyBox syslogd | systemd-journald |
|---|---|---|
| Format | Plain text (`/var/log/messages`) | Indexed binary journal |
| Read with | `logread`, `tail` | `journalctl -u <unit>`, `-b`, `--since` |
| Size control | `-s` rotate size | `SystemMaxUse=`, `RuntimeMaxUse=` |
| Embedded concern | Log to RAM (`/var/log` on tmpfs) to spare flash | `Storage=volatile` for the same reason |

The shared embedded rule: **flash wears out**. Production devices either
log to RAM (losing logs at reboot — often fine), cap and rotate
aggressively, or forward logs off-device (Level 4's fleet management).

## Choosing, in one table

| | BusyBox init | systemd |
|---|---|---|
| Config | `/etc/inittab` lines | Unit files |
| Crash restart | `respawn` | `Restart=always` (+ backoff, rate limits) |
| Dependencies | None (script it yourself) | First-class (`After=`, `Wants=`) |
| Footprint | ~0 (inside busybox) | Tens of MB + several daemons |
| Boot time | Fastest possible | Fast (parallel) but heavier |
| Best for | Minimal appliances, fast-boot devices | Products with many interacting services (i.MX-class) |

## Cheat sheet

| Item | Purpose |
|------|---------|
| `/etc/inittab` — `id::action:command` | Entire BusyBox init config |
| `::respawn:/usr/sbin/mydaemon` | Start + auto-restart a service |
| `kill -HUP 1` | Make BusyBox init re-read inittab |
| `.service` file: `[Unit] [Service] [Install]` | systemd unit anatomy |
| `Restart=always`, `After=`, `WantedBy=` | Restart policy, ordering, boot hookup |
| `systemctl enable --now foo` | Enable at boot and start immediately |
| `systemctl status foo` / `journalctl -u foo` | State + logs per service |
| `logread` | Read BusyBox syslog ring buffer |
| `Storage=volatile` / tmpfs `/var/log` | Spare the flash |
| `ps -p 1 -o comm=` | Identify the init system you're on |

## Exercise

(1) In the guest, extend `sensord-lite` to write a *pid file*
(`echo $$ > /tmp/sensord.pid` at the top) and prove across three kills
that init respawns it with a new PID each time. (2) Add a second inittab
entry that runs **once** at boot (action `wait`) creating
`/tmp/boot-marker` with a timestamp — which module-3 stage is this
imitating? (3) Rewrite `sensord.service` so it (a) only starts after
`/tmp/boot-marker` exists (hint: `ConditionPathExists=`), and (b) gives
up if it crashes 5 times in 10 seconds (hint: `StartLimitBurst=`,
`StartLimitIntervalSec=` — check the systemd.unit man page online).
(4) One paragraph: your i.MX95 product has a UI, a CAN daemon, an OTA
agent, and a cloud agent with strict start ordering — which init do you
ship and which three specific features justify it?
