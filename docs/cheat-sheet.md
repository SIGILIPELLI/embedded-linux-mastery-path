---
title: "Embedded Linux & Architectures Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free Embedded Linux & Architectures cheat sheet: 39 quick-reference tables from every level of the Embedded Linux & Architectures Mastery Path, each linked to its full lesson."
---

# Embedded Linux & Architectures Cheat Sheet

Quick-reference tables from **39 lessons** of the free Embedded Linux & Architectures Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [01 · What Is Embedded Linux?](level-1/01-what-is-embedded-linux.md)

| Term | Meaning |
|------|---------|
| MCU | Microcontroller: no MMU, KB of SRAM, runs bare-metal/RTOS (ESP32, STM32) |
| MPU / applications processor | Full CPU with MMU + external DDR, runs Linux (i.MX95) |
| MMU | Memory Management Unit — virtual memory hardware; required by standard Linux |
| SoC | System-on-chip: CPU cores + peripherals + accelerators on one die |
| Bootloader / U-Boot | First mutable code; initializes DDR, loads the kernel |
| Rootfs | The root filesystem: `/bin`, `/lib`, `/etc`, init, your apps |
| BusyBox | One small binary providing hundreds of Unix commands — the standard embedded userland |
| BSP | Board Support Package: bootloader + kernel + configs that make Linux run on a specific board |
| Buildroot / Yocto | Build systems that produce complete images from source |
| Heterogeneous SoC | Chip mixing core types, e.g. i.MX95's Cortex-A55 (Linux) + Cortex-M7 (real-time) |

### [02 · CPU Architectures 101](level-1/02-cpu-architectures-101.md)

| Term | Meaning |
|------|---------|
| ISA | The instruction-set contract (x86-64, AArch64, RV64) |
| Microarchitecture | A specific implementation of an ISA (Cortex-A55 vs A76) |
| AArch64 / arm64 | 64-bit ARM ISA — what modern ARM Linux targets |
| Cortex-A / R / M | Application (MMU, Linux) / Real-time / Microcontroller profiles |
| CISC vs RISC | Variable-length rich instructions (x86) vs fixed-length load/store (ARM, RISC-V) |
| SoC licensing | ARM licenses cores → vendors build custom SoCs (i.MX95); Intel/AMD sell finished x86 chips |
| ACPI vs device tree | x86 firmware self-describes hardware; embedded ARM needs an explicit hardware description |
| Heterogeneous SoC | Mixed core types on one die (A55 + M7 + M33 + NPU) |
| RISC-V | Open, license-free ISA; same embedded concepts apply |
| `uname -m`, `/proc/cpuinfo` | Identify ISA and core from a running system |

### [03 · The Boot Flow](level-1/03-boot-flow.md)

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

### [04 · Running Embedded Linux in QEMU](level-1/04-qemu-lab.md)

| Command / flag | Purpose |
|----------------|---------|
| `qemu-system-aarch64` | 64-bit ARM system emulator |
| `-M virt` | Generic ARM virtual board (the standard lab machine) |
| `-cpu cortex-a53` | Emulated core (AArch64, i.MX95-adjacent) |
| `-nographic` | Serial console in your terminal — the embedded way |
| `-bios QEMU_EFI.fd` / `edk2-aarch64-code.fd` | AArch64 UEFI firmware (apt: `qemu-efi-aarch64`) |
| `-nic user,model=virtio-net-pci` | NAT networking, zero host setup |
| `sha256sum -c` / `shasum -a 256 -c` | Verify image integrity before booting |
| ++ctrl+a++ ++x++ / ++ctrl+a++ ++c++ | Kill QEMU / toggle monitor |
| `poweroff` (in guest) | Clean shutdown |
| `/dev/ttyAMA0` | The PL011 serial UART — your console device |

### [05 · BusyBox &amp; the Minimal Userland](level-1/05-busybox-minimal-userland.md)

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

### [06 · Cross-Compilation](level-1/06-cross-compilation.md)

| Command | Purpose |
|---------|---------|
| `aarch64-linux-gnu-gcc hello.c -o hello` | Cross-compile for AArch64 Linux (glibc) |
| `zig cc -target aarch64-linux-musl ...` | Cross-compile from macOS (or anywhere), musl, static |
| `file <binary>` | Verify target arch, linkage, and interpreter |
| `-static` | Self-contained binary — no libc match needed |
| `aarch64-linux-gnu-gcc -print-sysroot` | Show the toolchain's target headers/libs root |
| `python3 -m http.server 8000` (host) | One-line file server for the lab |
| `udhcpc -i eth0` (guest) | BusyBox DHCP — bring up guest networking |
| `wget http://10.0.2.2:8000/f` (guest) | Fetch from host (10.0.2.2 = host in QEMU user networking) |
| `uname -m` / `Exec format error` | ISA check / the "wrong architecture" symptom |
| triplet `arch-os-libc` | Reads as: what the output binary runs on |

### [07 · Device Trees](level-1/07-device-trees.md)

| Concept / command | Meaning |
|-------------------|---------|
| DTS → (dtc) → DTB | Source → compiler → blob the bootloader passes to the kernel |
| `compatible = "vendor,device"` | Binds a node to a driver — the key property |
| `reg = <addr size>` | Register window(s); grouping set by `#address-cells`/`#size-cells` |
| `interrupts`, `clocks` | Wiring facts a driver needs to operate the device |
| `status = "okay" / "disabled"` | Board switches peripherals on/off |
| `.dtsi` vs `.dts` | SoC include (all disabled) vs board file (enables + overrides via `&label`) |
| `chosen { stdout-path }` | Which device is the boot console |
| `qemu -M virt,dumpdtb=f.dtb` | Dump QEMU's generated tree |
| `dtc -I dtb -O dts f.dtb` | Decompile a blob back to source |
| `/proc/device-tree/` | The live tree inside a running system |

### [08 · Init Systems &amp; Services](level-1/08-init-systems-services.md)

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

### [09 · Building a Rootfs with Buildroot](level-1/09-buildroot-rootfs.md)

| Command | Purpose |
|---------|---------|
| `make qemu_aarch64_virt_defconfig` | Load the QEMU AArch64 starting config |
| `make menuconfig` | Interactively edit target, toolchain, kernel, packages, filesystem |
| `make -j$(nproc)` | Build everything: toolchain, kernel, rootfs, image |
| `output/images/Image` | The built kernel — same role as module 3/4's kernel |
| `output/images/rootfs.ext4` | The built root filesystem — your BusyBox userland |
| `output/host/bin/aarch64-linux-*-gcc` | The toolchain Buildroot built — usable standalone too |
| `-kernel` / `-drive ...virtio-blk...` (QEMU) | Boot your own kernel + disk instead of an ISO |
| Buildroot vs Yocto | Single fixed image vs layered, multi-product distro builder |

### [10 · Capstone — Tiny Linux Appliance](level-1/10-capstone-appliance.md)

| Artifact | Role |
|----------|------|
| `sensord.c` | Cross-compiled C daemon — module 6's skill, product-shaped |
| `healthcheck.sh` | POSIX `/bin/sh` health probe — module 5's shell literacy |
| `BR2_ROOTFS_OVERLAY` | Buildroot mechanism for injecting your own files into the rootfs |
| `::respawn:/usr/sbin/sensord` | BusyBox init variant — module 8 |
| `sensord.service` (`Restart=always`) | systemd variant — module 8 |
| `/etc/issue` | Boot-time banner shown before login |
| `kill $(cat /var/run/sensord.pid)` then re-check | Proves the respawn contract works |

## Level 2 · Intermediate

### [01 · Yocto Fundamentals](level-2/01-yocto-fundamentals.md)

| Item | Purpose |
|------|---------|
| `source oe-init-build-env <dir>` | Create/enter a build dir, set up the environment |
| `conf/local.conf` | `MACHINE`, `DISTRO`, parallelism, image features |
| `conf/bblayers.conf` | Which layers are active |
| `bitbake-layers add-layer <path>` | Add a layer safely |
| `bitbake core-image-minimal` | Build the smallest bootable image |
| `runqemu qemuarm64 nographic` | Boot the result in QEMU |
| `.bb` recipe | Fetch + build + package one component |
| `.bbappend` | Modify a recipe owned by another layer |
| `SRC_URI` / `SRCREV` | Where source comes from / exact pinned commit |
| `${S}` / `${D}` / `${WORKDIR}` | Source dir / install staging root / work dir |
| `:append`, `:prepend`, `:${PN}` | Override syntax (colon, not underscore) |
| `bitbake -e <recipe>` | Dump every resolved variable — the debugging hammer |
| `bitbake -c cleansstate <recipe>` | Force one recipe to rebuild from scratch |
| `DL_DIR` / `SSTATE_DIR` | Download cache / shared-state cache — keep outside `build/` |

### [02 · U-Boot Deep Dive](level-2/02-u-boot-deep-dive.md)

| Command | Purpose |
|---------|---------|
| `printenv` / `setenv` / `saveenv` | Show / set (RAM) / persist environment |
| `env default -a -f` | Restore compiled-in defaults — un-brick your env |
| `run <var>` | Execute a variable's contents as commands |
| `mmc list` / `mmc dev N` / `ls mmc 1:1` | Enumerate, select, list an MMC device |
| `fatload mmc 1:1 ${addr} <file>` | Load a file from a FAT partition into RAM |
| `ext4load` / `load` | Same for ext4 / filesystem-agnostic |
| `tftp ${addr} <file>` | Load a file over the network |
| `ping ${serverip}` | Prove the board's Ethernet works before blaming TFTP |
| `booti / bootz / bootm` | Boot ARM64 Image / ARM32 zImage / uImage-FIT |
| `booti ${kernel} - ${fdt}` | The `-` means "no initramfs" |
| `md` / `mw` / `cmp` | Memory display / write / compare |
| `mkimage -T script -d boot.cmd boot.scr` | Compile a boot script |
| `boot_targets` | Distro-boot device search order |
| `bdinfo` / `version` / `help <cmd>` | Board info / build info / per-command help |

### [03 · Kernel Configuration &amp; Modules](level-2/03-kernel-config-modules.md)

| Command / item | Purpose |
|---|---|
| `make ARCH=arm64 defconfig` | Start from the architecture's default config |
| `make menuconfig` | Interactive config browser (`/` to search) |
| `make olddefconfig` | Accept defaults for newly introduced symbols |
| `./scripts/config --enable/--module/--disable` | Scriptable single-symbol edits |
| `scripts/kconfig/merge_config.sh -m .config frag.cfg` | Merge a config fragment |
| `# CONFIG_FOO is not set` | The correct way to spell "off" |
| `y` / `m` / unset | Built-in / loadable module / absent |
| `make Image dtbs modules` | Build kernel image, device trees, modules |
| `INSTALL_MOD_PATH=<rootfs> make modules_install` | Install modules into a target rootfs |
| `obj-m := foo.o` + `make -C $KDIR M=$PWD` | Out-of-tree module build |
| `insmod` / `rmmod` | Load/unload one exact `.ko` file |
| `modprobe` / `modprobe -r` | Load/unload by name, resolving dependencies |
| `lsmod` / `modinfo` | List loaded modules / inspect a module's metadata |
| `depmod -a` | Regenerate `modules.dep` after installing modules |
| `/etc/modules-load.d/*.conf` | Modules to load at boot (systemd) |
| `/etc/modprobe.d/*.conf` | Module options, aliases, blacklists |

!!! note "On verification"
    The Kconfig, fragment and kbuild syntax on this page follows the kernel's
    documented rules, but a full cross-kernel build was not run while writing
    it. Budget one build cycle to confirm symbol names against *your* kernel
    version — symbols are added, renamed and retired between releases.

### [04 · udev &amp; Device Management](level-2/04-udev-device-management.md)

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

### [05 · Embedded Networking](level-2/05-embedded-networking.md)

| Command / item | Purpose |
|---|---|
| `ip -brief addr` / `ip link` | Addresses / interfaces and carrier state |
| `LOWER_UP` vs `UP` | Carrier present vs administratively enabled |
| `ip route get 8.8.8.8` | Which route and source IP would be used |
| `ip -s link show eth0` | Error/drop/carrier counters — PHY health |
| `ss -tlnp` / `ss -tunap` | Listening sockets / all sockets with processes |
| `/etc/systemd/network/*.network` | Declarative address, DHCP, DNS, routes |
| `networkctl status <iface>` | Which file matched and what state it reached |
| `resolvectl status` / `resolvectl query <name>` | DNS servers / test resolution |
| `wpa_passphrase <ssid> <pass>` | Generate a PSK block (strip the plaintext!) |
| `wpa_supplicant@wlan0.service` | Instanced unit → `wpa_supplicant-wlan0.conf` |
| `wpa_cli -i wlan0 status` / `iw dev wlan0 link` | Association state / signal |
| `nft -c -f /etc/nftables.conf` | Syntax-check a ruleset without applying it |
| `nft list ruleset` | Dump the live ruleset |
| `ct state established,related accept` | The rule that must come first |
| `network-online.target` | Ordering target for "needs a working network" |

!!! note "On verification"
    The `.network`, `wpa_supplicant.conf` and `nftables.conf` syntax here
    follows the documented grammars for each, and the nftables ruleset is
    written so `nft -c -f` can check it before it is ever applied. Live
    hardware testing was not possible while writing this page — run the
    check-only commands on your own target first.

### [06 · Storage &amp; Filesystems (UBI, overlayfs)](level-2/06-storage-filesystems.md)

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

### [07 · Debugging &amp; Tracing (gdbserver, strace)](level-2/07-debugging-tracing.md)

| Command / item | Purpose |
|---|---|
| `dmesg -T --level=err,warn` / `dmesg -w` | Kernel errors with timestamps / follow |
| `echo 8 > /proc/sys/kernel/printk` | Raise console log verbosity |
| `/proc/<pid>/{status,fd,stack,maps}` | Memory, descriptors, kernel stack, mappings |
| `strace -f -tt -T -o log ./prog` | Trace syscalls: forks, timestamps, durations |
| `strace -c -p PID` | Syscall count/time summary — find hot or failing calls |
| `strace -e trace=%file,%network` | Filter to filesystem / socket calls |
| `gdbserver :2345 <prog>` | Debug stub on target |
| `gdbserver :2345 --attach PID` | Attach to a running process |
| `target remote <ip>:2345` | Connect from cross-gdb on the host |
| `set sysroot <sdk sysroot>` | **Required** for correct library symbols |
| `set substitute-path <build> <src>` | Map recorded build paths to your sources |
| `bt full` / `info threads` / `p expr` | Backtrace with locals / threads / evaluate |
| `core_pattern` + `coredumpctl list/info` | Post-mortem capture and inspection |
| `/sys/kernel/debug/tracing` (ftrace) | Kernel function and event tracing |
| `perf record -g` / `perf top` | Sampling profiler for "it's slow" |

!!! note "On verification"
    The command forms and gdb options here follow the documented interfaces
    for gdb, gdbserver, strace and ftrace; the transcripts are representative
    rather than captured, since no cross-toolchain or target build was run
    while writing this page. The outputs are useful as a guide to *what to
    look for*, not as literal expected text for your toolchain version.

### [08 · Read-Only Rootfs &amp; Factory Reset](level-2/08-read-only-rootfs.md)

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

### [09 · systemd Deep Dive](level-2/09-systemd-deep-dive.md)

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

### [10 · Project — Custom Yocto Image for QEMU](level-2/10-project-yocto-qemu-image.md)

| Item | Purpose |
|------|---------|
| `bitbake-layers create-layer ../meta-x` | Scaffold a new layer correctly |
| `LAYERSERIES_COMPAT_meta-x` | Release branches the layer supports — required |
| `recipes-*/<pkg>/files/` | Where `file://` sources live in your own layer |
| `LIC_FILES_CHKSUM` + `${COMMON_LICENSE_DIR}` | License check with no upstream tarball |
| `inherit systemd` + `SYSTEMD_SERVICE:${PN}` | Package and register a unit file |
| `SYSTEMD_AUTO_ENABLE = "enable"` | Enable at build time (mandatory if `/` is ro) |
| `inherit core-image` | Base class for a product image recipe |
| `IMAGE_FEATURES += "read-only-rootfs"` | Mount `/` read-only and adjust packaging |
| `IMAGE_INSTALL +=` (recipe) vs `:append` (local.conf) | `+=` is silently lost in `local.conf` |
| `ROOTFS_POSTPROCESS_COMMAND` | Last-moment edits to the assembled rootfs |
| `IMAGE_FSTYPES` / `IMAGE_ROOTFS_EXTRA_SPACE` | Output formats / spare space in KB |
| `base-files_%.bbappend` + `do_install:append` | Ship extra `/etc/fstab` entries |
| `oe-pkgdata-util list-pkg-files <pkg>` | Exactly what landed in a package |
| `runqemu <machine> <image> qemuparams="…"` | Boot it, with extra QEMU arguments |
| `findmnt -no OPTIONS /` | One-line proof that `/` really is read-only |

!!! note "On verification"
    Every recipe, class, variable and command here follows the documented
    interfaces in the Yocto Project Reference Manual for the `scarthgap` LTS
    release, and the unit file is written to pass `systemd-analyze verify`.
    A full Yocto build was **not** run while writing this page — it needs a
    Linux host, 50+ GB of disk and hours of CPU — so treat the task counts
    and image sizes as representative rather than exact, and expect to
    adjust variable names on a different release branch.

## Level 3 · Advanced

### [01 · Kernel Driver Development — Char Drivers](level-3/01-kernel-driver-development.md)

| API | Purpose |
|---|---|
| `alloc_chrdev_region` | Get a dynamic major/minor pair |
| `cdev_init` / `cdev_add` | Wire a `file_operations` table to a `dev_t` |
| `class_create` / `device_create` | Make `/dev/<name>` appear via udev |
| `copy_to_user` / `copy_from_user` | Only safe way to cross the user/kernel boundary |
| `_IO`/`_IOR`/`_IOW`/`_IOWR` | Encode ioctl direction + size in the command number |
| `pr_info`/`pr_err`/`__ratelimit` | Kernel logging, with overrun protection |
| `dmesg`, `modinfo`, `lsmod` | Inspect what's loaded and what it printed |

!!! note "On verification"
    The driver above follows the documented cdev/file_operations API and
    was reviewed line-by-line against kernel API conventions established in
    Level 1/2, but it was not compiled or booted on this machine (macOS
    cannot build or run a Linux kernel module). Treat it as review-verified
    source to build and test on your QEMU/target setup, not as executed code.

### [02 · Platform Drivers &amp; the Device Model](level-3/02-platform-drivers-device-model.md)

| Concept | Where it lives |
|---|---|
| `platform_driver` + `probe`/`remove` | Driver-side match/bind logic |
| `of_device_id` table + `compatible` | What links a DT node to this driver |
| `devm_platform_get_and_ioremap_resource` | Map `reg` safely, freed automatically |
| `-EPROBE_DEFER` | "Not ready yet, try me again later" |
| `/sys/kernel/debug/devices_deferred` | See what's stuck waiting on a dependency |
| `/sys/bus/platform/drivers/<name>/{bind,unbind}` | Force a re-probe without rebooting |
| `DEVICE_ATTR_RW` + `ATTRIBUTE_GROUPS` | Expose control/status via sysfs |

!!! note "On verification"
    Probe/remove flow, the `of_device_id` matching mechanism, and
    `-EPROBE_DEFER` semantics were checked against the documented driver
    core API and Level 2's device-tree material; no kernel was built or
    booted to execute this code on this machine.

### [03 · Device Tree Overlays](level-3/03-device-tree-overlays.md)

| Command | Purpose |
|---|---|
| `dtc -@ -I dts -O dtb -o x.dtbo x.dts` | Compile overlay with symbol table |
| `fdtdump x.dtbo` | Inspect a compiled overlay's fragments |
| `fdt resize <n>; fdt apply <addr>` | Apply overlay in U-Boot (resize first!) |
| `mount -t configfs none /sys/kernel/config` | Enable runtime overlay application |
| `.../overlays/<name>/dtbo` | Write compiled `.dtbo` here to apply at runtime |
| `cat /proc/device-tree/.../compatible` | Inspect the *merged*, live tree |
| `status = "disabled"` | Preferred way to turn off a node per board variant |

!!! note "On verification"
    Overlay fragment syntax, `/plugin/;`, and the configfs application
    interface were checked against the documented DT overlay ABI and
    dtc behavior; the U-Boot and configfs command sequences were reviewed
    for correctness but not executed on real hardware or QEMU here.

### [04 · Interrupts &amp; Kernel Concurrency](level-3/04-interrupts-kernel-concurrency.md)

| Primitive | Sleep-safe | Use from ISR? |
|---|---|---|
| `spin_lock`/`spin_unlock` | No | Only if never shared with an ISR context needing `_irqsave` |
| `spin_lock_irqsave`/`_irqrestore` | No | Yes — required if lock is shared with ISR |
| `mutex_lock`/`mutex_unlock` | Yes | Never |
| `READ_ONCE`/`WRITE_ONCE` | N/A | Yes, for single-word lock-free flags only |
| `devm_request_threaded_irq` | handler thread can sleep | primary handler cannot |
| `IRQF_ONESHOT` | — | Required pattern for level-triggered shared lines with a thread |

!!! note "On verification"
    Locking rules (`_irqsave` requirement, spinlock-vs-mutex context
    rules, threaded-IRQ semantics) were cross-checked against documented
    kernel locking conventions and Level 2/3 material; the race conditions
    described are real, well-documented failure classes, but no code here
    was compiled, booted, or stress-tested on real SMP hardware from this
    machine.

### [05 · Heterogeneous Compute — remoteproc &amp; RPMsg](level-3/05-remoteproc-rpmsg.md)

| Path/Command | Purpose |
|---|---|
| `/sys/class/remoteproc/remoteprocN/firmware` | Which file to load next |
| `/sys/class/remoteproc/remoteprocN/state` | `start` / `stop`, read current state |
| `/sys/class/remoteproc/remoteprocN/coredump` | Capture M-core memory on crash |
| `/dev/rpmsg_ctrl*` | Control device for creating rpmsg endpoints |
| `/dev/rpmsgN` | Per-channel data stream |
| Resource table (`fw_rsc_*`) | M-core-side contract: carveouts, vdevs, vrings |
| DT `reserved-memory` | Must cover every carveout the resource table declares |

!!! note "On verification"
    The remoteproc sysfs ABI and rpmsg char-device model are checked
    against the documented kernel remoteproc/rpmsg framework; the resource
    table struct layout is representative of NXP's SDK convention, not
    copied from a specific SDK version, and none of this was loaded onto
    real M-core/A-core silicon from this machine — treat it as a
    conceptual and structural reference, not tested firmware.

### [06 · Real-Time Linux (PREEMPT_RT)](level-3/06-real-time-preempt-rt.md)

| Concept | PREEMPT_RT behavior |
|---|---|
| `spinlock_t` | Becomes a preemptible, priority-inheriting rt-mutex |
| `raw_spinlock_t` | Stays a true non-preemptible spinlock |
| Hard IRQ handlers | Run as preemptible `IRQ-<n>` kthreads by default |
| `PTHREAD_PRIO_INHERIT` | Required opt-in for userspace priority inheritance |
| `SCHED_FIFO`/`SCHED_RR` | Real-time scheduling classes, run-to-block semantics |
| `cyclictest -m -Sp90 -D 4h` | Standard long-soak latency acceptance test |
| `isolcpus=` | Reserve a CPU from the general scheduler for RT work |

!!! note "On verification"
    PREEMPT_RT's spinlock/rt-mutex conversion, threaded-IRQ default, and
    priority-inheritance model were checked against the documented
    PREEMPT_RT design and mainline scheduler behavior; `cyclictest` output
    above is representative formatting, not a real measurement — no RT
    kernel was booted or measured on this machine.

### [07 · Graphics — DRM/KMS &amp; Wayland](level-3/07-graphics-drm-wayland.md)

| Concept | Purpose |
|---|---|
| Connector / Encoder / CRTC / Plane | DRM/KMS object hierarchy |
| `modetest -M <driver> -c` | Inspect live connector/mode topology |
| DT `port`/`endpoint`/`remote-endpoint` | Graph binding linking DRM components |
| `drmModeAtomicCommit` | Validate-then-apply an entire display state at once |
| `wl_surface_frame` callback | Correct pacing signal for a Wayland client |
| `drmSetMaster`/`EBUSY` | Symptom of two processes fighting over DRM master |

!!! note "On verification"
    DRM/KMS object model, atomic modesetting semantics, and the Wayland
    frame-callback pattern were checked against documented DRM/libdrm and
    Wayland protocol behavior; no GPU driver or display pipeline was
    exercised on this machine — `modetest`/`drmModeAtomicCommit` output
    shown is representative, not captured from real hardware.

### [08 · Power Management &amp; Thermal](level-3/08-power-management-thermal.md)

| Interface | Purpose |
|---|---|
| `scaling_governor` | Pick CPU frequency policy (`schedutil` default) |
| `cpuidle/stateN/{latency,usage}` | Idle-state cost and how often it's used |
| `pm_runtime_get_sync`/`put_autosuspend` | Per-device sleep independent of CPU |
| `SIMPLE_DEV_PM_OPS` | Wire `suspend`/`resume` callbacks into a driver |
| `echo mem > /sys/power/state` | Trigger suspend-to-RAM |
| `/sys/class/thermal/thermal_zoneN/temp` | Current temperature reading |
| `trip_point_N_type` (`passive`/`critical`) | Throttle vs. shutdown behavior |

!!! note "On verification"
    cpufreq/cpuidle/runtime-PM/thermal sysfs interfaces and the DT thermal
    binding were checked against the documented kernel PM and thermal
    frameworks; the suspend/resume driver code follows the documented
    `dev_pm_ops` contract but was not compiled or exercised through an
    actual suspend/resume cycle on real hardware from this machine.

### [09 · Performance Profiling (perf, ftrace)](level-3/09-performance-profiling.md)

| Tool/file | Answers |
|---|---|
| `perf record -F 999 -a -g` | Where CPU time goes, with call graphs |
| `perf stat -e cycles,instructions,cache-misses` | IPC and cache behavior |
| `ftrace function_graph` | Exact call sequence and per-call duration |
| `ftrace wakeup_rt` | Ground-truth scheduling latency for a real event |
| Custom `TRACE_EVENT()` | Near-zero-cost driver-specific instrumentation |
| `/proc/pressure/{cpu,io,memory}` | Is anything actually stalled, not just busy |

!!! note "On verification"
    `perf`/`ftrace` command syntax and output formats were cross-checked
    against documented kernel tracing and perf-tools conventions and
    Level 2's debugging module; the specific numeric output shown (sample
    percentages, latency values) is illustrative, not captured from a
    real profiling run on this machine.

### [10 · Project — Sensor Driver + Userspace Stack](level-3/10-project-sensor-driver.md)

| Layer | Module this project draws on |
|---|---|
| `i2c_client`/`devm_*` probe discipline | Module 2 |
| Device tree `interrupts`, `mount-matrix` | Module 3 |
| Threaded IRQ, correct top/bottom-half split | Module 4 |
| `pm_runtime_*` autosuspend | Module 8 |
| `ftrace`/`perf` for latency and stutter diagnosis | Module 9 |
| IIO triggered buffer ABI | This project's core contribution |

!!! note "On verification"
    This design was reviewed for internal consistency against the IIO,
    threaded-IRQ, and runtime-PM kernel APIs referenced in the preceding
    modules and against real BMI270/IIO driver conventions; it was not
    compiled, flashed, or run against real or emulated I2C hardware from
    this machine. Treat it as a build-and-bring-up starting point, not
    tested firmware.

## Level 4 · Master

### [01 · Production BSP Maintenance](level-4/01-production-bsp-maintenance.md)

| Practice | Tool/command |
|---|---|
| Pin layer set to exact commits | `kas` / `repo manifest`, never a branch name |
| Find unpatched CVEs in a build | `bitbake -c cve_check <image>` |
| Backport a targeted fix | `.bbappend` + `SRC_URI += "file://CVE-....patch"` |
| Track kernel LTS series | Rebase patch stack onto `<major>.<minor>.y` on a cadence |
| Prove what shipped | Archive `.rootfs.manifest` + layer pins + `local.conf` per release |

!!! note "On verification"
    The Yocto `cve_check` workflow, `.bbappend`/`SRC_URI` patch mechanism,
    and manifest-tracking practices are checked against documented
    Yocto/BitBake conventions and Level 2's build material; no image was
    actually built or CVE-scanned on this machine.

### [02 · Secure Boot Chain (HAB/AHAB)](level-4/02-secure-boot-hab-ahab.md)

| Concept | HAB (i.MX6/7/8) | AHAB (i.MX8/9) |
|---|---|---|
| Root of trust | SRK hash in Boot ROM-read fuses | SRK hash, verified by SECO/EdgeLock enclave |
| Signing tool | `cst` + CSF descriptor | `cst` + container-based CSF |
| Runtime status | `hab_status` (open/eng units only) | `ahab_log` |
| Multi-image signing | Per-stage, sequential | Single signed container, multiple images |
| rootfs integrity | Not covered — needs `dm-verity` | Not covered — needs `dm-verity` |

!!! note "On verification"
    The HAB/AHAB chain-of-trust model, CST signing workflow, and
    dm-verity integration pattern are checked against NXP's documented
    HAB/AHAB architecture and the dm-verity kernel documentation; no keys
    were generated, no image was signed, and — critically — no fuses were
    burned as part of writing this module. Treat every command here as
    reviewed syntax to validate on disposable hardware, never as a
    tested, ready-to-fuse procedure.

### [03 · OTA Updates (RAUC, SWUpdate, OSTree)](level-4/03-ota-updates.md)

| Mechanism | Rollback trigger | Storage model |
|---|---|---|
| RAUC A/B | Bootloader boot-counter timeout, no `mark-good` | Two full rootfs partitions |
| SWUpdate | Handler-chain failure / signature failure | Configurable, often A/B similar |
| OSTree | `ostree admin rollback`, boot other deployment | Content-addressed, deduplicated, no partition swap |
| dm-verity (any) | Continuous hash-tree check during read | Read-only, tamper-evident |

!!! note "On verification"
    RAUC/SWUpdate/OSTree command syntax and the boot-counter rollback
    mechanism are checked against each project's documented workflow and
    Level 4's secure-boot module; no update bundle was built, signed, or
    installed against real or emulated hardware from this machine.

### [04 · Trusted Execution — OP-TEE &amp; TrustZone](level-4/04-op-tee-trustzone.md)

| Component | Runs where | Role |
|---|---|---|
| `libteec` (client API) | Normal World, userspace | Opens sessions, invokes TA commands |
| Trusted Application (`.ta`, named by UUID) | Secure World | Does the actual protected work |
| `tee-supplicant` | Normal World, userspace daemon | Services storage/RPC requests from TAs |
| OP-TEE driver | Normal World kernel | SMC transport between Linux and Secure World |
| ARM Trusted Firmware (BL31) | EL3, both worlds | Arbitrates the SMC world switch |

!!! note "On verification"
    The client/TA API usage and two-world architecture are checked
    against OP-TEE's documented `libteec`/`tee_internal_api` and the
    ARM TrustZone SMC model; no TA was built, signed, or invoked against
    real or emulated Secure World hardware from this machine.

### [05 · NPU &amp; ML Acceleration (eIQ)](level-4/05-npu-ml-acceleration.md)

| Step | What to check |
|---|---|
| Quantize (INT8, representative dataset) | Calibration data matches real deployment distribution |
| Convert to `.tflite`/ONNX | Conversion succeeding is not an accuracy guarantee |
| Load with NPU delegate | Read the per-op fallback log, not just "delegate loaded" |
| Benchmark | Discard warmup runs; report steady-state avg and p99 |
| Full pipeline | Profile capture→inference→postprocess, not the model alone |
| Field model updates | Version, hash, and roll back exactly like firmware |

!!! note "On verification"
    The eIQ/TFLite quantization and delegate workflow, and the
    representative-dataset calibration pattern, are checked against
    documented TensorFlow Lite and NXP eIQ conventions; no model was
    actually trained, quantized, or run against real or emulated NPU
    hardware from this machine — output shown is illustrative.

### [06 · Containers &amp; Virtualization on Embedded](level-4/06-containers-virtualization.md)

| Need | Right tool |
|---|---|
| Isolate an update/fault domain, share the kernel | Podman/runc container, daemonless preferred |
| Pin CPU/memory for a workload | cgroups (`cpu.max`, `memory.max`, `cpuset.cpus`) |
| Grant hardware access to a container | `--device=`, understand it's node-level not line-level |
| Coordinate a small fixed set of container services | systemd unit dependencies, not Kubernetes |
| A genuinely different kernel or certified isolation | Type-1 hypervisor (Jailhouse), static partitioning |

!!! note "On verification"
    cgroup v2 interfaces, OCI/podman device-passthrough behavior, and the
    Jailhouse static-partitioning model are checked against documented
    kernel cgroup, OCI runtime, and Jailhouse conventions; no container
    was actually run and no hypervisor cell was created on this machine.

### [07 · Fleet Management &amp; Provisioning](level-4/07-fleet-management.md)

| Concern | Practice |
|---|---|
| Device identity | Generate key on-device/in secure element, never inject from outside |
| Zero-touch provisioning | Idempotent script, tested against real rework/reflash flow |
| Fleet health signal | Small, actionable telemetry set; watch `silent_over_24h` |
| Update rollout | Staged (canary → wider), gated on real health data, not just a timer |
| Remote diagnostics | Predefined, signed, audited commands — not blanket SSH |

!!! note "On verification"
    Device-identity, provisioning, staged-rollout, and telemetry practices
    here are checked against widely documented IoT fleet-management and
    PKI conventions and consistent with Modules 2-4's secure-boot/OTA/
    TrustZone material; no fleet backend, CA, or provisioning flow was
    actually run from this machine.

### [08 · Security Hardening &amp; CVE Management](level-4/08-security-hardening.md)

| Layer | Hardening action |
|---|---|
| rootfs contents | Remove compilers/debug tools; `IMAGE_FEATURES:remove = "debug-tweaks"` |
| Kernel config | `STACKPROTECTOR_STRONG`, `FORTIFY_SOURCE`, `HARDENED_USERCOPY`, KASLR |
| Network | Audit `ss -tulnp`; disable telnet/tftp; SSH key-only, no root login |
| Process privilege | `setcap` over setuid-root; systemd `ProtectSystem=strict`, `NoNewPrivileges` |
| Untrusted-input code | Fuzz parsers; static analysis (`cppcheck`, `clang-tidy`) |
| Ongoing | Triage CVEs by reachability, not score alone |

!!! note "On verification"
    Kernel hardening config options, systemd sandboxing directives, and
    the fuzzing/static-analysis workflow are checked against documented
    kernel, systemd, and AFL/clang-tidy conventions; no image was built,
    no fuzzing campaign was run, and no CVE scan was executed on this
    machine.

### [09 · Compliance &amp; Open-Source Licensing](level-4/09-compliance-licensing.md)

| Obligation | Practice |
|---|---|
| GPLv2 kernel/tools | Written offer + archive matching the *exact* shipped source, per release |
| LGPL libraries | Dynamic linking preserves proprietary-code exemption; confirm linking mode |
| GPLv3 components | Audit separately — direct tension with secure boot (Module 2) |
| Unclear-license vendor blobs | Scan (`scancode`), escalate to vendor, don't assume clearance |
| Ongoing tracking | Generate license manifest + SBOM per release, same pipeline as CVE data |

!!! note "On verification"
    License obligations summarized here reflect the commonly understood
    terms of GPLv2/LGPL/MIT/BSD/GPLv3 as documented by their respective
    license texts and standard OSS compliance practice, and the Yocto
    `populate_lic`/SPDX tooling described follows documented BitBake
    conventions; this module is not legal advice — a real product's
    compliance program should involve counsel, and no license scan was
    actually run on this machine.

---

Go deeper with the full [Embedded Linux & Architectures Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
