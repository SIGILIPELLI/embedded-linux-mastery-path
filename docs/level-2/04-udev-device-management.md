# 04 · udev & Device Management

`/dev` is not a directory of files someone created. It is a live view of
the hardware the kernel currently knows about, maintained by a userspace
daemon reacting to kernel **uevents**. On a desktop this is invisible
plumbing. On an embedded product it is where you solve the problem that
your USB-to-serial adapter is `/dev/ttyUSB0` on Monday and `/dev/ttyUSB1`
on Tuesday — which, if your daemon hardcodes the first, is a field failure.

## How a device node appears

```text
hardware event
     │
     ▼
kernel driver binds  ──►  sysfs entry appears (/sys/class/tty/ttyUSB0)
     │
     ▼
uevent broadcast over a netlink socket
     │
     ▼
udevd matches rules in /etc/udev/rules.d, /lib/udev/rules.d
     │
     ├──► creates /dev/ttyUSB0 (via devtmpfs, then applies perms)
     ├──► creates symlinks: /dev/serial/by-id/usb-FTDI_...-if00-port0
     ├──► sets owner/group/mode, adds tags
     └──► optionally RUN{program}= a helper
```

Two independent things are happening. **devtmpfs** (a kernel feature,
`CONFIG_DEVTMPFS=y`) creates the bare node so the system is usable before
any daemon starts. **udev** then applies *policy*: names, symlinks,
permissions, and actions.

## Three implementations, one concept

| | `systemd-udevd` | `eudev` | BusyBox `mdev` |
|---|---|---|---|
| Size | ~1 MB+ | ~300 KB | a few KB (already in BusyBox) |
| Rule syntax | full udev | full udev | its own, much simpler |
| Typical use | systemd images | musl/OpenRC images | tiny BusyBox appliances |

`mdev` is configured through `/etc/mdev.conf` and hooked up via
`/proc/sys/kernel/hotplug` or `mdev -s` at boot. It is the right answer for
a 8 MB image; it is the wrong answer as soon as you need `by-id` symlinks
or property-based matching. This module uses real udev syntax.

## Reading the device you want to match

Never guess attributes. Ask udev:

```console
root@target:~# udevadm info --query=all --name=/dev/ttyUSB0
P: /devices/platform/soc/38200000.usb/usb1/1-1/1-1:1.0/ttyUSB0/tty/ttyUSB0
N: ttyUSB0
L: 0
S: serial/by-id/usb-FTDI_FT232R_USB_UART_A50285BI-if00-port0
S: serial/by-path/platform-38200000.usb-usb-0:1:1.0-port0
E: DEVNAME=/dev/ttyUSB0
E: SUBSYSTEM=tty
E: ID_VENDOR_ID=0403
E: ID_MODEL_ID=6001
E: ID_SERIAL_SHORT=A50285BI
```

`udevadm info --attribute-walk` walks up the sysfs parent chain and is what
you use when the property you need lives on a parent device:

```console
root@target:~# udevadm info --attribute-walk --name=/dev/ttyUSB0
  looking at device '/devices/.../ttyUSB0/tty/ttyUSB0':
    KERNEL=="ttyUSB0"
    SUBSYSTEM=="tty"
    DRIVER==""

  looking at parent device '/devices/.../1-1':
    KERNELS=="1-1"
    SUBSYSTEMS=="usb"
    ATTRS{idVendor}=="0403"
    ATTRS{idProduct}=="6001"
    ATTRS{serial}=="A50285BI"
```

The single most important rule of udev rules: **all match keys in one rule
must be satisfiable at one level of the chain**, except the plural forms
(`KERNELS`, `SUBSYSTEMS`, `ATTRS`) which search up the parent chain. Mixing
`ATTR{}` (this device) with `ATTRS{}` (any ancestor) incorrectly is the
number-one reason a rule "does nothing".

## Writing rules

Rules live in `/etc/udev/rules.d/` (yours) and `/lib/udev/rules.d/`
(the distribution's). Files are processed in **lexical order across both
directories**, so the numeric prefix matters, and an `/etc` file with the
same name completely masks the `/lib` one.

```text
# /etc/udev/rules.d/70-appliance-serial.rules
# Persistent name for the RS-485 modem adapter
SUBSYSTEM=="tty", ATTRS{idVendor}=="0403", ATTRS{idProduct}=="6001", \
  ATTRS{serial}=="A50285BI", SYMLINK+="modem", GROUP="dialout", MODE="0660"

# The GPS puck on the second FTDI port, matched by port not serial number
SUBSYSTEM=="tty", ATTRS{idVendor}=="0403", ATTRS{idProduct}=="6001", \
  ENV{ID_USB_INTERFACE_NUM}=="01", SYMLINK+="gps"

# Give the sensor bus device to the app user without running the app as root
SUBSYSTEM=="i2c-dev", KERNEL=="i2c-1", GROUP="sensors", MODE="0660"

# Start a systemd unit when the maintenance USB stick is inserted
ACTION=="add", SUBSYSTEM=="block", ENV{ID_FS_LABEL}=="SERVICE", \
  TAG+="systemd", ENV{SYSTEMD_WANTS}="usb-service.service"
```

Operators are the other half of the syntax and are routinely confused:

| Operator | Meaning |
|---|---|
| `==` | match (rule continues only if true) |
| `!=` | match if not equal |
| `=` | assign, replacing any previous value |
| `+=` | append to a list (**always** use for `SYMLINK`) |
| `:=` | assign and forbid later rules from changing it |

`SYMLINK="modem"` with a single `=` wipes every symlink udev already
created, including the `by-id` ones. Use `+=`.

## Testing without rebooting

```console
root@target:~# udevadm test /sys/class/tty/ttyUSB0
...
Reading rules file: /etc/udev/rules.d/70-appliance-serial.rules
70-appliance-serial.rules:3 Adding link 'modem'
GROUP 20 /etc/udev/rules.d/70-appliance-serial.rules:3
MODE 0660 /etc/udev/rules.d/70-appliance-serial.rules:3
```

```console
root@target:~# udevadm control --reload-rules
root@target:~# udevadm trigger --subsystem-match=tty --action=add
root@target:~# ls -l /dev/modem
lrwxrwxrwx 1 root root 7 Aug  4 11:20 /dev/modem -> ttyUSB0
root@target:~# udevadm monitor --udev --property
KERNEL[812.1] add  /devices/.../ttyUSB0 (tty)
UDEV  [812.2] add  /devices/.../ttyUSB0 (tty)
DEVLINKS=/dev/modem /dev/serial/by-id/usb-FTDI_...-if00-port0
```

`udevadm test` does *not* create nodes — it only shows what would happen,
which makes it safe to run on a live product. `udevadm monitor` in a second
terminal while you plug the device in is the fastest way to learn what a
device actually reports.

## Persistent naming beyond serial ports

The same mechanism gives you predictable network interfaces
(`enp1s0` instead of `eth0`, from `/lib/udev/rules.d/80-net-setup-link.rules`)
and stable block device paths. Prefer the paths the system already provides
over inventing your own:

```console
root@target:~# ls /dev/disk/by-partlabel/
rootfs_a  rootfs_b  data
root@target:~# ls /dev/serial/by-id/
usb-FTDI_FT232R_USB_UART_A50285BI-if00-port0
```

Mounting by `PARTLABEL=` or `UUID=` in `/etc/fstab` instead of
`/dev/mmcblk0p2` is the same idea applied to storage, and it is what makes
an A/B update scheme (Level 4) possible at all.

## Traps

!!! danger "udev traps"
    - **`SYMLINK=` instead of `SYMLINK+=`** destroys the `by-id` and `by-path`
      links other software depends on.
    - **Long-running `RUN{program}=`.** udev kills the process after a timeout
      (default 180 s) and blocks event processing meanwhile — a slow script
      here can stall your entire boot. Use `TAG+="systemd"` +
      `ENV{SYSTEMD_WANTS}` to hand the work to a service instead.
    - **Matching a serial number that not every unit has.** Cheap USB devices
      often ship with an identical (or empty) `ATTRS{serial}` across the whole
      batch; a rule that works on your desk fails on the production line.
      Match by USB *port path* when you control the hardware layout.
    - **Rules that run before the filesystem you need is mounted.** udev starts
      very early; a `RUN` script referencing `/var` or a data partition may
      find nothing there.
    - **Race with the application.** A daemon started at boot may open
      `/dev/modem` before the device has been enumerated. Depend on the device
      unit (`BindsTo=dev-modem.device`) rather than sleeping and hoping.
    - **`mdev` is not udev.** Copying udev rules into an mdev system silently
      does nothing — `mdev.conf` has a completely different, positional syntax.

## Cheat sheet

| Command / item | Purpose |
|---|---|
| `udevadm info --query=all --name=/dev/X` | Everything udev knows about a node |
| `udevadm info --attribute-walk --name=/dev/X` | Walk sysfs parents for `ATTRS{}` keys |
| `udevadm test /sys/class/.../X` | Dry-run the rules against one device |
| `udevadm control --reload-rules` | Reload rules after editing |
| `udevadm trigger --action=add` | Replay uevents for existing devices |
| `udevadm monitor --udev --property` | Watch events live as you plug things in |
| `udevadm settle` | Wait for the event queue to drain (scripts) |
| `/etc/udev/rules.d/` | Your rules; masks same-named files in `/lib` |
| `KERNEL` / `SUBSYSTEM` / `ATTR{}` | Match this device |
| `KERNELS` / `SUBSYSTEMS` / `ATTRS{}` | Match this device *or any ancestor* |
| `SYMLINK+=`, `MODE=`, `GROUP=`, `OWNER=` | Assign name, permissions |
| `TAG+="systemd"`, `ENV{SYSTEMD_WANTS}=` | Trigger a unit on device appearance |
| `/dev/serial/by-id/`, `/dev/disk/by-*` | Ready-made persistent paths |
| `devtmpfs` | Kernel-created bare nodes, before udev runs |
| BusyBox `mdev -s` + `/etc/mdev.conf` | The tiny-image alternative |

!!! note "On verification"
    The rule and `udevadm` syntax here follows the documented udev rules
    grammar (match keys, operators, `ATTRS` parent walking), but these rules
    were not exercised against live hardware while writing this page. Confirm
    every rule on your own target with `udevadm test` before shipping it —
    that command exists precisely so you never have to guess.

## Exercise

(1) On any Linux machine, plug in a USB serial adapter (or use an existing
device such as `/dev/sda`) and capture both `udevadm info --query=all` and
`udevadm info --attribute-walk` for it; identify one attribute that lives
on the device itself and one that only exists on a parent. (2) Write a rule
that gives it a stable `SYMLINK+=` name and group ownership, verify with
`udevadm test` *before* reloading, then reload and confirm with `ls -l`.
(3) Deliberately break it: change `+=` to `=` on the `SYMLINK` line, reload,
and show what disappeared from `/dev/serial/by-id/`. (4) One paragraph: your
appliance has two identical FTDI adapters — a modem and a GPS — with
identical vendor, product and serial strings. Explain how you would still
name them deterministically, and why a rule that runs a 30-second
`RUN{program}=` script to probe each one is the wrong fix.
