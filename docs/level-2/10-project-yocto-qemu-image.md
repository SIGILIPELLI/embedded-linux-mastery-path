# 10 · Project — Custom Yocto Image for QEMU

Nine modules of parts; now one product. You build your own layer containing
an application recipe, a systemd unit, an image recipe and a persistent
`/data` partition, then boot the result in QEMU with a **read-only root
filesystem**. This is not a toy exercise — swap `MACHINE = "qemuarm64"` for
`imx8mm-lpddr4-evk`, add NXP's layers, and the same tree builds the
shipping image. That substitutability is the entire point of Yocto.

## What you are building

```text
meta-appliance/
├── conf/layer.conf                          layer metadata
├── recipes-apps/appd/{appd_1.0.bb, files/}  application + sources
├── recipes-core/base-files/*.bbappend       adds /data to fstab
└── recipes-core/images/appliance-image.bb   the product image
```

Result: `appliance-image-qemuarm64.rootfs.ext4` — read-only `/`, a
persistent `/data` on a second virtio disk, and `appd` started by systemd
at boot and restarted if it dies.

## Step 1 — create the layer

Never hand-roll `conf/layer.conf`; let the tool write it.

```console
$ source oe-init-build-env build-qemuarm64
$ bitbake-layers create-layer ../meta-appliance
Add your new layer with 'bitbake-layers add-layer ../meta-appliance'
$ bitbake-layers add-layer ../meta-appliance
$ bitbake-layers show-layers | tail -1
meta-appliance   /home/dev/poky/meta-appliance             6
```

The generated `conf/layer.conf` needs exactly one edit — the release
branches the layer claims to support:

```python
LAYERSERIES_COMPAT_meta-appliance = "scarthgap"
```

That is a promise, not a preference: if it does not name the poky branch
you are on, every build stops with `Layer meta-appliance is not compatible
with the core layer`.

## Step 2 — the application

A deliberately small daemon, so that the packaging is what you study:

```c
/* meta-appliance/recipes-apps/appd/files/appd.c */
#include <stdio.h>
#include <time.h>
#include <unistd.h>

int main(void) {
    setvbuf(stdout, NULL, _IOLBF, 0);      /* line-buffer for journald */
    FILE *log = fopen("/data/appd.log", "a");
    if (!log) { perror("/data/appd.log"); return 1; }
    setvbuf(log, NULL, _IOLBF, 0);
    for (unsigned long t = 0; ; t++) {
        fprintf(log, "%ld tick=%lu\n", (long)time(NULL), t);
        printf("tick=%lu\n", t);
        sleep(5);
    }
}
```

It writes to `/data` *and* to stdout on purpose: `/data` proves the
writable partition is mounted, stdout proves journald is capturing the
unit. Its unit file, in the style of module 9:

```ini
# meta-appliance/recipes-apps/appd/files/appd.service
[Unit]
Description=Appliance control daemon
RequiresMountsFor=/data
After=data.mount

[Service]
ExecStart=/usr/sbin/appd
Restart=always
RestartSec=5s
StartLimitIntervalSec=300
StartLimitBurst=5

[Install]
WantedBy=multi-user.target
```

## Step 3 — the recipe

```python
# meta-appliance/recipes-apps/appd/appd_1.0.bb
SUMMARY = "Appliance control daemon"
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/MIT;md5=0835ade698e0bcf8506ecda2f7b4f302"

SRC_URI = "file://appd.c file://appd.service"
S = "${WORKDIR}"

inherit systemd
SYSTEMD_SERVICE:${PN} = "appd.service"
SYSTEMD_AUTO_ENABLE = "enable"

do_compile() {
    ${CC} ${CFLAGS} ${LDFLAGS} -o appd ${S}/appd.c
}

do_install() {
    install -d ${D}${sbindir} ${D}${systemd_system_unitdir}
    install -m 0755 appd              ${D}${sbindir}/appd
    install -m 0644 ${S}/appd.service ${D}${systemd_system_unitdir}/
}

FILES:${PN} += "${systemd_system_unitdir}/appd.service"
```

Because the recipe lives in your own layer, bitbake finds `files/`
automatically through `FILESPATH`; `FILESEXTRAPATHS` is only needed in a
`.bbappend` reaching into someone else's recipe. Build the recipe alone
first — a five-second compile error is much cheaper to find now than inside
a two-hour image build:

```console
$ bitbake appd
NOTE: Tasks Summary: Attempted 812 tasks of which 806 didn't need to be rerun
$ oe-pkgdata-util list-pkg-files appd
appd:
        /lib/systemd/system/appd.service
        /usr/sbin/appd
```

If a file you installed is missing from that list, it went into no package
at all — `FILES:${PN}` is what claims it.

## Step 4 — mount `/data`

The rootfs is going to be read-only, so the writable partition must be in
`/etc/fstab` at build time. A `.bbappend` on `base-files` does it:

```python
# meta-appliance/recipes-core/base-files/base-files_%.bbappend
do_install:append() {
    cat >> ${D}${sysconfdir}/fstab <<'EOF'
/dev/vdb   /data      ext4   defaults,noatime,errors=remount-ro  0  2
tmpfs      /var/tmp   tmpfs  defaults,size=16M,mode=1777         0  0
EOF
}
```

`/dev/vdb` is the second virtio disk QEMU will hand us. On real hardware
use `PARTLABEL=data` instead — module 6's warning about enumeration order
applies to eMMC far more than to QEMU.

## Step 5 — the image recipe

```python
# meta-appliance/recipes-core/images/appliance-image.bb
SUMMARY = "Read-only appliance image with appd"
LICENSE = "MIT"
inherit core-image

IMAGE_FEATURES += "read-only-rootfs ssh-server-dropbear"
IMAGE_INSTALL += "appd packagegroup-core-boot util-linux e2fsprogs-mke2fs strace"
IMAGE_ROOTFS_EXTRA_SPACE = "8192"
IMAGE_FSTYPES = "ext4 tar.bz2"

ROOTFS_POSTPROCESS_COMMAND += "make_data_mountpoint; "
make_data_mountpoint() {
    install -d ${IMAGE_ROOTFS}/data
}
```

The mountpoint has to exist *in the image*: with a read-only `/`, systemd
cannot create `/data` at boot, and the mount fails. Finally, in
`conf/local.conf`, drop the lab conveniences:

```python
MACHINE ??= "qemuarm64"
INIT_MANAGER = "systemd"
EXTRA_IMAGE_FEATURES = ""          # no debug-tweaks: no empty root password
IMAGE_INSTALL:append = " tzdata"   # note the leading space
```

`IMAGE_INSTALL += "tzdata"` in `local.conf` is a classic silent failure —
image recipes assign `IMAGE_INSTALL` after `local.conf` is parsed, so the
`+=` is thrown away. Then build:

```console
$ bitbake appliance-image
NOTE: Tasks Summary: Attempted 4102 tasks of which 3388 didn't need to be rerun
$ ls -lh tmp/deploy/images/qemuarm64/appliance-image-qemuarm64.rootfs.ext4
-rw-r--r-- 2 dev dev 62M ... appliance-image-qemuarm64.rootfs.ext4
```

## Running it

Make the data disk once on the host, then boot with it attached as the
second virtio disk:

```console
$ qemu-img create -f raw /tmp/data.img 256M
$ mkfs.ext4 -F -L data /tmp/data.img
$ runqemu qemuarm64 appliance-image nographic \
    qemuparams="-drive file=/tmp/data.img,if=virtio,format=raw"
```

Four checks, in order — each falsifies a different assumption:

```console
root@qemuarm64:~# findmnt -no OPTIONS /
ro,relatime
root@qemuarm64:~# touch /etc/proof
touch: /etc/proof: Read-only file system
root@qemuarm64:~# findmnt /data
TARGET SOURCE    FSTYPE OPTIONS
/data  /dev/vdb  ext4   rw,noatime,errors=remount-ro
root@qemuarm64:~# systemctl is-enabled appd; systemctl is-active appd
enabled
active
root@qemuarm64:~# tail -2 /data/appd.log
1754380815 tick=3
1754380820 tick=4
```

Then prove the restart policy — kill it and watch systemd bring it back:

```console
root@qemuarm64:~# systemctl kill -s KILL appd; sleep 8
root@qemuarm64:~# systemctl show -p NRestarts appd
NRestarts=1
```

Reboot with the same `/tmp/data.img` and `appd.log` still holds the earlier
lines; boot *without* the disk and `appd` fails on `RequiresMountsFor=`
rather than scribbling into a read-only `/`. That failure mode is the
design working.

## Traps

!!! danger "What actually goes wrong in this project"
    - **`SYSTEMD_AUTO_ENABLE` is build-time only.** With a read-only rootfs
      you cannot `systemctl enable` on the target — the symlink has nowhere
      to go. If the unit is not enabled in the image, it is not enabled.
    - **Missing `/data` directory.** No mountpoint, no mount, and the app
      writes into the rootfs — which, being read-only, fails at `fopen`.
    - **`S = "${WORKDIR}"`** is correct for scarthgap. Releases from
      walnascar onward unpack into `${UNPACKDIR}`; a recipe carried forward
      without that change fails with `appd.c: No such file or directory`.
    - **Editing `tmp/` by hand.** Everything under `build/tmp` is generated:
      fix the recipe, not the artifact, or the change vanishes at the next
      `cleansstate`. Likewise, create layers *beside* the build directory.

## Cheat sheet

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

## Stretch goals

1. **Make it one image.** Replace `ext4` in `IMAGE_FSTYPES` with `wic` and
   write a `.wks` kickstart laying out boot / rootfs / data partitions in a
   single file, so nothing has to be attached by hand.
2. **Squash it.** Switch the rootfs to squashfs and give `/etc` an
   overlayfs upper on `/data`, as in module 8. Which files under `/etc`
   actually changed after a first boot, and does anything break when you
   wipe the upper directory?
3. **Add the watchdog.** Convert `appd` to `Type=notify` with
   `WatchdogSec=30s` (module 9), stop pinging deliberately, and capture the
   journal lines showing systemd killing and restarting it.
4. **Factory reset.** Add a `factory-reset.service` ordered before `appd`
   that reformats `/data` when it finds a flag file — plus the U-Boot
   environment variable (module 2) that sets the flag from the bootloader.
5. **Retarget it.** Add `meta-freescale` on the matching branch, set
   `MACHINE = "imx8mm-lpddr4-evk"`, and rebuild. List every file in
   `meta-appliance` you had to change; the shorter that list, the better
   your layer.
