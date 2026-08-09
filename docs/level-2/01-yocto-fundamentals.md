# 01 · Yocto Fundamentals

Level 1 ended with a rule: Buildroot for one fixed image, **Yocto** once
you consume a silicon vendor's BSP or maintain a product line. Every major
SoC vendor — NXP included — ships board support as Yocto layers, so
"reading a BSP" and "reading Yocto" turn out to be the same skill. This
module takes you from zero to a `core-image-minimal` booting in QEMU, and
— more importantly — teaches you to *read* the handful of files that
decide what a Yocto build actually produces.

## The mental model: layers, recipes, bitbake

Yocto is not a build system you configure. It is a **set of layers** you
compose, each holding **recipes**, driven by **bitbake**, the task engine.

```text
   your build/            ← conf/local.conf + conf/bblayers.conf (YOUR choices)
        │
        ├── meta          ← OpenEmbedded-Core: gcc, busybox, systemd, base images
        ├── meta-poky     ← the "poky" reference distro definition
        ├── meta-openembedded/meta-oe ← thousands of extra packages
        └── meta-imx / meta-freescale ← NXP BSP: i.MX kernel, U-Boot, firmware
                                        (this is what "vendor BSP" means)
```

A **recipe** (`.bb`) says how to fetch, configure, compile and package one
piece of software. A **layer** is a directory of recipes plus metadata. A
**machine** (`MACHINE=`) selects the hardware. A **distro** (`DISTRO=`)
selects policy — libc, init system, feature set. Change `MACHINE`, keep
everything else, and you get the same product for different hardware. That
one property is why product families live in Yocto and not in Buildroot.

## Setting up a build

Yocto builds on a Linux host, with unusually specific prerequisites and
**50+ GB of free disk**:

```console
$ sudo apt install gawk wget git diffstat unzip texinfo gcc build-essential \
    chrpath socat cpio python3 python3-pip python3-pexpect xz-utils \
    debianutils iputils-ping python3-git python3-jinja2 python3-subunit \
    zstd liblz4-tool file locales libacl1
$ git clone -b scarthgap git://git.yoctoproject.org/poky
$ cd poky
```

`scarthgap` is an LTS release branch. **Pin a release branch, never
`master`** — Yocto's tip moves daily and vendor layers are branch-matched.

Sourcing the init script creates and enters a build directory:

```console
$ source oe-init-build-env build-qemuarm64
You had no conf/local.conf file. This configuration file has therefore been
created for you from .../local.conf.sample

### Shell environment set up for builds. ###

You can now run 'bitbake <target>'
Common targets are:
    core-image-minimal
    core-image-full-cmdline
    meta-toolchain
$ pwd
/home/dev/poky/build-qemuarm64
```

## The two files that decide everything

**`conf/bblayers.conf`** — which layers are in play:

```python
BBLAYERS ?= " \
  /home/dev/poky/meta \
  /home/dev/poky/meta-poky \
  /home/dev/poky/meta-yocto-bsp \
  /home/dev/poky/meta-openembedded/meta-oe \
  "
```

Avoid hand-editing it — `bitbake-layers` keeps the syntax and priorities
right:

```console
$ bitbake-layers add-layer ../meta-openembedded/meta-oe
$ bitbake-layers show-layers
layer                 path                                      priority
==========================================================================
meta                  /home/dev/poky/meta                       5
meta-poky             /home/dev/poky/meta-poky                  5
meta-oe               /home/dev/poky/meta-openembedded/meta-oe  6
```

**`conf/local.conf`** — your machine, distro and knobs:

```python
MACHINE ??= "qemuarm64"
DISTRO ?= "poky"
PACKAGE_CLASSES ?= "package_rpm"
EXTRA_IMAGE_FEATURES ?= "debug-tweaks"
INIT_MANAGER = "systemd"

# Build-host tuning — these save hours across a project
BB_NUMBER_THREADS ?= "8"
PARALLEL_MAKE ?= "-j 8"
DL_DIR ?= "/data/yocto/downloads"
SSTATE_DIR ?= "/data/yocto/sstate-cache"
```

`debug-tweaks` gives you an empty root password — convenient in the lab,
and a shipping-blocker in production. Remove it before any real image.

## Build and boot

```console
$ bitbake core-image-minimal
Loading cache: 100% |###############################| Time: 0:00:02
Loaded 1683 entries from dependency cache.
Parsing recipes: 100% |#############################| Time: 0:00:31
Build Configuration:
BB_VERSION           = "2.8.0"
MACHINE              = "qemuarm64"
DISTRO               = "poky"
TARGET_SYS           = "aarch64-poky-linux"
Initialising tasks: 100% |##########################| Time: 0:00:03
NOTE: Executing Tasks
NOTE: Tasks Summary: Attempted 3241 tasks of which 0 didn't need to be rerun
```

A first build is **2–6 hours** and tens of GB. Later builds hit the
shared-state cache and finish in minutes. Then:

```console
$ runqemu qemuarm64 nographic
runqemu - INFO - Running MACHINE=qemuarm64 ...
Poky (Yocto Project Reference Distro) 5.0.x qemuarm64 ttyAMA0
qemuarm64 login: root
root@qemuarm64:~# cat /etc/os-release
ID=poky
VERSION_ID=5.0.x
```

`runqemu` is a wrapper that assembles the exact `qemu-system-aarch64`
invocation you wrote by hand in Level 1 — kernel, rootfs, `-append`,
networking.

## Reading a recipe

Recipes are the unit of work. Here is a complete, realistic one:

```python
# meta-mylayer/recipes-apps/sensord/sensord_1.0.bb
SUMMARY = "Sensor simulator daemon"
DESCRIPTION = "Reads a sensor and logs readings for the appliance stack."
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://LICENSE;md5=0835ade698e0bcf8506ecda2f7b4f302"

SRC_URI = "git://github.com/example/sensord.git;protocol=https;branch=main \
           file://sensord.service"
SRCREV = "9a1f2c3d4e5f60718293a4b5c6d7e8f901234567"

S = "${WORKDIR}/git"

inherit systemd

SYSTEMD_SERVICE:${PN} = "sensord.service"
SYSTEMD_AUTO_ENABLE = "enable"

do_compile() {
    ${CC} ${CFLAGS} ${LDFLAGS} -o sensord ${S}/sensord.c
}

do_install() {
    install -d ${D}${sbindir}
    install -m 0755 sensord ${D}${sbindir}/sensord
    install -d ${D}${systemd_system_unitdir}
    install -m 0644 ${WORKDIR}/sensord.service ${D}${systemd_system_unitdir}
}
```

Four things to internalise. `LIC_FILES_CHKSUM` makes the build **fail** if
upstream's license text changes — that is a feature, and Level 4's
compliance module depends on it. `SRCREV` pins an exact commit so builds
are reproducible. `${D}` is the install staging root, never the live
system. And `:${PN}` is the modern override syntax — older documents use
`_${PN}`, a form removed in the honister release.

## Modifying someone else's recipe: `.bbappend`

You never edit a vendor layer. You **append** to it from your own layer:

```python
# meta-mylayer/recipes-core/busybox/busybox_%.bbappend
FILESEXTRAPATHS:prepend := "${THISDIR}/${PN}:"
SRC_URI += "file://my-extra.cfg"
```

The `%` matches any version. `FILESEXTRAPATHS:prepend` is what lets
bitbake find your files — omit it and the append parses cleanly and does
nothing, which is the single most common Yocto beginner bug.

Useful inspection commands while you work:

```console
$ bitbake -e busybox | grep "^S="            # what a variable resolved to
$ bitbake-layers show-recipes busybox        # which layers provide it
$ bitbake -c listtasks busybox               # available tasks
$ bitbake -c devshell busybox                # shell inside its build environment
$ bitbake -c cleansstate busybox             # force a full rebuild of one recipe
```

## Traps

!!! danger "Yocto traps that cost real days"
    - **Never `sudo bitbake`.** It leaves root-owned files in `tmp/` and the
      only reliable fix is deleting the whole build tree.
    - **Disk exhaustion mid-build** can leave a damaged `sstate-cache`, after
      which later builds fail with bizarre unrelated errors. Budget
      50–100 GB, and keep `DL_DIR`/`SSTATE_DIR` outside `build/` so that a
      `rm -rf tmp` stays cheap.
    - **Mismatched branches.** `meta-imx` on `scarthgap` against poky on
      `kirkstone` produces parse errors that look like recipe bugs. Match
      release branches across every layer.
    - **`_append` vs `:append`.** Post-honister the underscore form is not an
      error — it is ignored, so your override silently does nothing.
    - **Case-insensitive filesystems** (macOS default, some network mounts)
      break the kernel build. Yocto needs a case-sensitive Linux filesystem.

## Cheat sheet

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

## Exercise

(1) Set up poky on the `scarthgap` branch, build `core-image-minimal` for
`MACHINE = "qemuarm64"`, and boot it with `runqemu`; record the wall-clock
time of the first build and of an immediate second `bitbake` of the same
target, then explain the difference in one sentence using the term
*shared state*. (2) Run
`bitbake -e core-image-minimal | grep "^IMAGE_INSTALL"` and list three
packages the image pulls in that you never asked for — where does each come
from? (3) Create your own layer with
`bitbake-layers create-layer ../meta-mylayer`, add it, and write a
`busybox_%.bbappend` that ships an extra config fragment; prove it took
effect with `bitbake -e busybox | grep my-extra.cfg`. (4) One paragraph:
your team must ship the same application on i.MX8M Mini and i.MX95 boards
with different peripherals — describe which Yocto concept (`MACHINE`, a
layer, a `.bbappend`, or the distro) you would use for each difference, and
why editing NXP's `meta-imx` in place would be the wrong answer.
