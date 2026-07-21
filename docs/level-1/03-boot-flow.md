# 03 · The Boot Flow

Between applying power and seeing `login:` on the console, an embedded Linux
board walks through a fixed relay race: **boot ROM → SPL → U-Boot → kernel →
init**. Each runner owns one job, does it, and hands the baton forward. When
a board "doesn't boot," debugging is simply asking *which runner dropped the
baton* — so this mental model is the single most useful diagnostic tool in
embedded Linux. This module walks the whole race, then compares it with how
a PC (UEFI/BIOS) does the same thing.

## Stage 0 — Boot ROM: the code you can't change

The first instructions ever executed live in **mask ROM inside the SoC**,
burned at the factory. The boot ROM is tiny and does exactly one thing:
find the next stage and load it.

- **Where does it look?** Determined by **boot mode pins/fuses** — physical
  straps or one-time-programmable fuses. On i.MX chips these select eMMC,
  SD card, FlexSPI NOR flash, or **serial download mode** (USB recovery —
  the reason a "bricked" i.MX board is almost never truly bricked: NXP's
  `uuu` tool can always push a new image over USB).
- **What can it load into?** Only **on-chip SRAM** (a few hundred KB) —
  because external DDR *is not initialized yet*. This size limit is the
  entire reason the next stage exists.
- On secure-boot products, the ROM also verifies the next stage's signature
  (i.MX calls this HAB/AHAB — Level 4 territory).

## Stage 1 — SPL: waking up the RAM

The **SPL** (Secondary Program Loader — U-Boot's first stage, small enough
to fit in SRAM) has one critical job: **initialize the DDR controller**.
DDR training is chip- and board-specific (timings, drive strengths, the
exact RAM part you soldered), which is why SPL comes from your board's BSP,
not from generic code. With DDR alive, SPL loads full U-Boot into it and
jumps.

On modern i.MX parts the picture is richer: the boot image also carries ARM
Trusted Firmware and system-manager firmware for the M33 core, bundled by
NXP's `imx-mkimage` into one signed container the ROM understands. Same
relay race, more batons.

## Stage 2 — U-Boot: the loader you can talk to

**U-Boot** is the workhorse bootloader of embedded Linux. Now running from
DDR with drivers for storage and network, it:

1. Loads three things into RAM: the **kernel image** (`Image` on ARM64), the
   **device tree blob** (`.dtb` — module 7), and optionally an
   **initramfs**.
2. Sets kernel arguments (`bootargs`) — e.g.
   `console=ttyAMA0 root=/dev/mmcblk0p2 rootwait`.
3. Jumps to the kernel, passing the DTB address.

Interrupt it over serial and you get a shell — the embedded engineer's
best friend:

```console
Hit any key to stop autoboot:  0
u-boot=> printenv bootargs
bootargs=console=ttyAMA0 root=/dev/vda rootwait
u-boot=> ls mmc 0:1          # list files on first eMMC partition
u-boot=> booti ${loadaddr} - ${fdtaddr}   # boot: kernel, (no initrd), dtb
```

## Stage 3 — Kernel: from "Booting Linux" to mounting root

The kernel decompresses, reads the device tree to learn what hardware
exists, initializes drivers, then must **mount the root filesystem** named
by `root=`. Two paths:

- **Direct mount** — `root=/dev/mmcblk0p2` and the driver is built in.
  Typical embedded fast path.
- **initramfs first** — a small cpio filesystem in RAM runs early userspace
  (load storage modules, unlock encrypted disks, pick an A/B update slot)
  and then pivots to the real root. Desktop distros always do this;
  embedded uses it when updates or encryption demand it.

You can replay the whole kernel stage after boot:

```console
$ dmesg | head -4
[    0.000000] Booting Linux on physical CPU 0x0000000000 [0x410fd050]
[    0.000000] Linux version 6.6.x ...
[    0.000000] Machine model: linux,dummy-virt
[    0.000000] earlycon: pl11 at MMIO 0x0000000009000000
```

## Stage 4 — init: PID 1 and the login prompt

The kernel starts exactly one userspace program — **init, PID 1** (by
convention `/sbin/init`; override with `init=` in bootargs). Init mounts
`/proc` and `/sys`, starts services, and spawns `getty` on the console —
which prints the `login:` prompt. Everything running on the system descends
from PID 1. Module 8 covers the two embedded choices: BusyBox init and
systemd.

## The full relay, and how x86 does it

```text
Power on
  └─ Boot ROM (in SoC, immutable)  — reads fuses, finds boot media
      └─ SPL (in SRAM)             — initializes DDR
          └─ U-Boot (in DDR)       — loads kernel + DTB (+ initramfs)
              └─ Linux kernel      — drivers, mounts rootfs
                  └─ init (PID 1)  — services, getty → login:
```

| Stage | Embedded ARM (i.MX95) | PC (x86-64) |
|-------|----------------------|-------------|
| First code | SoC boot ROM, reads **boot fuses/pins** | UEFI/BIOS flash on the motherboard |
| RAM init | SPL (from your BSP) | UEFI firmware (board vendor's) |
| Loader | U-Boot | GRUB / systemd-boot (via UEFI boot manager) |
| Hardware description | **Device tree blob** passed to kernel | **ACPI tables** provided by firmware |
| Kernel image | `Image` + `.dtb` | `bzImage` |
| Early userspace | Optional initramfs | Practically always initramfs |
| Recovery story | Serial download mode (`uuu` over USB) | Boot from USB stick |

The deep difference is the middle rows: on x86 the *firmware* owns RAM init
and self-describes the hardware via ACPI, so one generic Ubuntu ISO boots
on any PC. On embedded ARM, *you* (via the BSP) own RAM init and must hand
the kernel an explicit device tree — the price of custom silicon, and the
reason BSPs exist.

## Cheat sheet

| Term | Meaning |
|------|---------|
| Boot ROM | Immutable first-stage loader inside the SoC |
| Boot fuses / mode pins | Select boot source (eMMC / SD / NOR / serial download) on i.MX |
| Serial download mode | i.MX USB recovery path; `uuu` tool pushes images — unbrickable |
| SPL | Tiny U-Boot stage in SRAM; initializes DDR |
| U-Boot | Full bootloader: shell, env, loads kernel + DTB, `booti` |
| `bootargs` | Kernel command line set by U-Boot (`console=`, `root=`, `init=`) |
| DTB | Compiled device tree — hardware description handed to the kernel |
| initramfs | Optional RAM-based early userspace before the real rootfs |
| init / PID 1 | First process; parents everything; prints `login:` via getty |
| UEFI + ACPI | The x86 equivalents of (SPL+U-Boot) + device tree |

## Exercise

From a running Linux system (your PC, or the QEMU guest you'll boot in
module 4), reconstruct its boot story: run `cat /proc/cmdline` and identify
what set each argument (bootloader) and what consumes it (kernel or init);
run `dmesg | grep -i "Machine model\|command line\|Freeing unused"` to find
the device-tree model name and the moment the kernel finishes; and run
`ps -p 1 -o comm=` to name PID 1. Then, on paper: your i.MX95 board boots
from eMMC but the kernel panics with
`VFS: Unable to mount root fs on unknown-block(0,0)`. Which relay stage
dropped the baton, which stages provably *succeeded*, and which single
`bootargs` variable would you inspect first in the U-Boot shell?
