# 09 · Building a Rootfs with Buildroot

Every lab so far used images someone else built. Time to build your own:
kernel, cross-toolchain, and BusyBox rootfs, from source, configured
entirely by you, booting in the same QEMU you already know. **Buildroot**
is the tool — a `make menuconfig`-driven system that turns a single
configuration into a complete, bootable embedded Linux image. This is also
where the pieces from modules 1–8 stop being separate lessons and become
one pipeline you control end to end.

## Install prerequisites and get the source

Buildroot needs a normal Linux build toolchain plus `ncurses` for the
config UI (works natively on Linux; on macOS, build inside a Linux VM or
container — Buildroot itself targets Linux hosts):

```console
$ sudo apt install build-essential libncurses-dev git bc cpio unzip rsync \
    file wget bison flex
$ git clone https://github.com/buildroot/buildroot.git
$ cd buildroot
$ git checkout 2025.02.1        # pin a real release tag, not a moving branch
```

## Start from a defconfig, then tune it

Buildroot ships **defconfigs** — starting points for real boards and QEMU
machines. Ours matches the machine you've used since module 4:

```console
$ ls configs | grep qemu_aarch64
qemu_aarch64_virt_defconfig
$ make qemu_aarch64_virt_defconfig
```

Now open the interactive menu to see — and change — every decision this
config made:

```console
$ make menuconfig
```

Tour it like an embedded engineer, not a checkbox-clicker:

| Menu | What you're deciding |
|------|----------------------|
| **Target options** | Architecture — already `AArch64` (this defconfig's whole point) |
| **Toolchain** | Buildroot builds its *own* cross-toolchain (glibc/musl/uClibc, GCC version) — no manual triplet-hunting like module 6 |
| **System configuration** | Hostname, `/etc/inittab` — yes, the very inittab from module 8, generated for you |
| **Kernel** | Version, and which **defconfig** the kernel itself uses (`qemu_aarch64_virt` maps to a matching kernel config) |
| **Target packages** | Every userspace package — BusyBox is on by default; add `nano`, `dropbear` (SSH), whatever your product needs |
| **Filesystem images** | Output format — `ext4` for our QEMU disk |

Turn on **`dropbear`** (Target packages → Networking applications) — a
tiny SSH server — so your image can be logged into over the network, then
exit saving `.config`.

## Build

```console
$ make -j$(nproc)
```

This compiles a full cross-toolchain, the Linux kernel, BusyBox, and every
selected package — expect **30–90+ minutes** on a first run (Buildroot
caches downloads and build state, so later tweaks rebuild only what
changed). When it finishes:

```console
$ ls output/images/
Image  rootfs.ext4  rootfs.ext2  sdcard.img
```

Recognize every one of these from earlier modules: `Image` is the kernel
(module 3), `rootfs.ext4` is your BusyBox userland (module 5) — now
compiled by *your* toolchain, matched to *your* kernel, with *no*
libc-mismatch risk (the module-6 bug is structurally impossible here,
because Buildroot builds the toolchain and the rootfs from the same
config).

## Boot your own image in QEMU

Compare this to module 4's command — same machine type, but now you supply
your own kernel and disk instead of a distro ISO:

```console
$ qemu-system-aarch64 \
    -M virt -cpu cortex-a53 -smp 2 -m 512M \
    -kernel output/images/Image \
    -append "root=/dev/vda console=ttyAMA0" \
    -drive file=output/images/rootfs.ext4,if=none,format=raw,id=hd0 \
    -device virtio-blk-device,drive=hd0 \
    -netdev user,id=net0 -device virtio-net-device,netdev=net0 \
    -nographic
```

Expect a *much* faster, sparser boot than Alpine — this rootfs contains
only what you selected:

```text
Welcome to Buildroot
buildroot login: root
# uname -a
Linux buildroot 6.6.x #1 SMP ... aarch64 GNU/Linux
# cat /etc/os-release
NAME=Buildroot
# busybox | head -1
BusyBox v1.3x.x (....) multi-call binary.
```

No password on this default config. Confirm dropbear made it in:
`# netstat -lnt | grep :22` (or `ps | grep dropbear`).

## Where Yocto fits

Buildroot just gave you **one image from one config** — perfect for a
single product. The moment you need *several related products* sharing
most software but differing in a few packages, or you need to consume a
silicon vendor's official BSP, the industry reaches for **Yocto** instead
— it models software as reusable **layers** (a kernel layer, a vendor BSP
layer, your product layer) and **recipes**, not one flat config. This is
exactly why NXP ships i.MX support as `meta-imx`, a Yocto layer — Level 2
is built entirely around this.

| | Buildroot | Yocto |
|---|---|---|
| Mental model | One `.config` → one image | Layers + recipes → a customizable Linux *distro* |
| Learning curve | Days | Weeks |
| Build speed (first build) | Faster | Slower (but shared package cache helps at scale) |
| Multiple similar products | Awkward (config-per-product) | Natural (share layers, vary one) |
| Silicon-vendor BSPs (i.MX, etc.) | Rare/community | **Standard delivery format** |
| Package management on-device | Not really the point | opkg/rpm/deb variants available |
| Best for | Single products, appliances, learning, fast iteration | Product families, vendor BSP consumption, long-term maintenance |

Rule of thumb this course uses: **Buildroot to learn and to ship something
small and fixed; Yocto once you're consuming a vendor BSP (i.MX95
included) or maintaining a product line.**

## Cheat sheet

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

## Exercise

(1) Add `nano` and `htop` to Target packages, rebuild (`make -j$(nproc)` —
only the changed pieces recompile), and confirm both run in the booted
image. (2) In `make menuconfig` → System configuration, change the
hostname and add a line to the root filesystem overlay skeleton
(`BR2_ROOTFS_OVERLAY`) that drops a file into `/etc/motd`; rebuild and
confirm it's shown at login. (3) Run
`output/host/bin/aarch64-*-readelf -d output/target/usr/bin/nano | grep NEEDED`
and compare against what module 6 taught you about dynamic linking and
libc matching — explain in one sentence why this binary is guaranteed to
run on this rootfs. (4) One paragraph: your team ships three i.MX95
products sharing 90% of their software but differing in one application
and one kernel driver — would you keep three separate Buildroot configs
or move to Yocto, and why?
