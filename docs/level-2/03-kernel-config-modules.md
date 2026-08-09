# 03 · Kernel Configuration & Modules

In Level 1 you booted a kernel someone else built. In a product you own
that kernel: which drivers are compiled in, which are modules, which are
absent, and how a customer-specific driver gets loaded. This module covers
the configuration system (Kconfig), the difference between `y` and `m`,
config fragments — the only sane way to carry changes across kernel
upgrades — and building an out-of-tree module against your kernel's build
tree.

## Kconfig: what the kernel actually reads

The kernel does not read `menuconfig` output directly. Everything funnels
into one generated file, `.config`, in the build directory:

```console
$ cd linux
$ make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig
  HOSTCC  scripts/basic/fixdep
  HOSTCC  scripts/kconfig/conf.o
#
# configuration written to .config
#
$ grep -c . .config
2114
```

Each line is a symbol declared by a `Kconfig` file somewhere in the tree:

```text
config EXT4_FS
	tristate "The Extended 4 (ext4) filesystem"
	select JBD2
	select CRC16
	help
	  This is the next generation of the ext3 filesystem.
```

Three facts follow from that snippet, and they explain most kernel-config
confusion:

- **`tristate` means three values**: `y` (built into the kernel image),
  `m` (a loadable `.ko` module), or unset. `bool` symbols only offer `y`
  or unset.
- **`select` forces a dependency on**, without asking. That is why turning
  on one option silently turns on four others.
- **`depends on` hides an option** when its prerequisite is off. If a
  symbol you need "isn't in menuconfig", it is almost always hidden by an
  unmet `depends on`, not missing from your tree.

## Driving the configuration

`menuconfig` is the ncurses browser; the scriptable interfaces matter more
inside a build system:

```console
$ make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- menuconfig
$ make ARCH=arm64 olddefconfig        # accept defaults for new symbols
$ ./scripts/config --enable  CONFIG_OVERLAY_FS
$ ./scripts/config --module  CONFIG_USB_SERIAL_FTDI_SIO
$ ./scripts/config --disable CONFIG_DEBUG_INFO
```

Search inside `menuconfig` with `/` — it shows the symbol name, its
dependencies, and which menu it lives in. That search is the fastest way
to answer "why can't I select this?".

To find a board's starting point:

```console
$ ls arch/arm64/configs/
defconfig
$ make ARCH=arm64 defconfig      # arm64 has one unified defconfig
```

On 32-bit ARM you get per-family files instead (`imx_v6_v7_defconfig`,
`multi_v7_defconfig`). A vendor BSP always states which defconfig its board
expects — using the wrong one produces a kernel that boots to a blank
serial console.

## Config fragments: the only maintainable approach

Never ship a 5,000-line `.config` as your product configuration. You
cannot review a diff of it, and it does not survive a kernel version bump.
Ship a **fragment** — just the deltas:

```text
# my-appliance.cfg
CONFIG_OVERLAY_FS=y
CONFIG_SQUASHFS=y
CONFIG_SQUASHFS_XZ=y
CONFIG_WATCHDOG=y
CONFIG_WATCHDOG_SYSFS=y
# CONFIG_DEBUG_INFO is not set
```

Note the comment form: `# CONFIG_FOO is not set` is how Kconfig spells
"off". A bare `CONFIG_FOO=n` line is *ignored* by the merge tooling — a
classic silent failure.

Merge it:

```console
$ ARCH=arm64 ./scripts/kconfig/merge_config.sh -m .config my-appliance.cfg
Using .config as base
Merging my-appliance.cfg
$ make ARCH=arm64 olddefconfig
```

Always verify afterwards that the merge actually took, because `select`
and `depends on` can quietly override you:

```console
$ grep -E "OVERLAY_FS|SQUASHFS_XZ" .config
CONFIG_OVERLAY_FS=y
CONFIG_SQUASHFS_XZ=y
```

In Yocto the same fragment goes in a `.bbappend` beside the kernel recipe,
and the `kernel-yocto` class runs the merge for you:

```python
# meta-mylayer/recipes-kernel/linux/linux-imx_%.bbappend
FILESEXTRAPATHS:prepend := "${THISDIR}/${PN}:"
SRC_URI += "file://my-appliance.cfg"
```

## Building and installing

```console
$ make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc) Image dtbs modules
  ...
  LD      vmlinux
  OBJCOPY arch/arm64/boot/Image
  DTC     arch/arm64/boot/dts/freescale/imx95-19x19-evk.dtb
$ make ARCH=arm64 INSTALL_MOD_PATH=/srv/rootfs modules_install
  INSTALL /srv/rootfs/lib/modules/6.6.23/kernel/fs/overlayfs/overlay.ko
  DEPMOD  /srv/rootfs/lib/modules/6.6.23
```

`INSTALL_MOD_PATH` is not optional when cross-building — omit it and you
install target modules into your *build host's* `/lib/modules`, which at
best does nothing and at worst confuses the host's own kernel.

## Out-of-tree modules

Vendor and customer drivers usually live outside the kernel tree. The
minimal case:

```c
/* hello_embed.c */
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/utsname.h>

static int __init hello_init(void)
{
	pr_info("hello_embed: loaded on %s\n", init_utsname()->machine);
	return 0;
}

static void __exit hello_exit(void)
{
	pr_info("hello_embed: unloaded\n");
}

module_init(hello_init);
module_exit(hello_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("You");
MODULE_DESCRIPTION("Minimal out-of-tree module example");
```

```makefile
# Makefile
obj-m := hello_embed.o

KDIR ?= /srv/linux
all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules
clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

```console
$ make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- KDIR=/srv/linux
make -C /srv/linux M=/home/dev/hello modules
  CC [M]  /home/dev/hello/hello_embed.o
  MODPOST /home/dev/hello/Module.symvers
  LD [M]  /home/dev/hello/hello_embed.ko
```

`MODULE_LICENSE("GPL")` is load-bearing, not paperwork: a non-GPL string
taints the kernel and denies the module access to GPL-only exported
symbols, which surfaces as an undefined-symbol error at `insmod` time.

On target:

```console
root@qemuarm64:~# insmod hello_embed.ko
root@qemuarm64:~# dmesg | tail -1
[  142.883021] hello_embed: loaded on aarch64
root@qemuarm64:~# lsmod
Module                  Size  Used by
hello_embed            16384  0
root@qemuarm64:~# modinfo hello_embed.ko
filename:       /root/hello_embed.ko
description:    Minimal out-of-tree module example
license:        GPL
vermagic:       6.6.23 SMP preempt mod_unload aarch64
root@qemuarm64:~# rmmod hello_embed
```

`insmod` takes a path and does nothing clever. `modprobe` takes a *name*,
resolves dependencies from `modules.dep`, and honours
`/etc/modprobe.d/*.conf` — use `modprobe` everywhere except when
hand-testing a freshly built `.ko`.

## Traps

!!! danger "Kernel config traps"
    - **vermagic mismatch.** A module built against 6.6.23 refuses to load on
      6.6.24 (`insmod: ERROR: could not insert module: Invalid module format`).
      Modules are bound to the exact kernel build — rebuild them whenever the
      kernel changes, and ship the two together in one image.
    - **Filesystem driver as `m` under the root filesystem.** If the driver
      that mounts `/` is a module and there is no initramfs to load it, the
      kernel panics with `VFS: Unable to mount root fs`. Root filesystem,
      storage controller and console drivers belong in the image as `y`.
    - **A bare `CONFIG_FOO=n` in a fragment is ignored.** Use
      `# CONFIG_FOO is not set`.
    - **`select` cannot be overridden.** If something keeps turning back on,
      find who selects it: `grep -rn "select FOO" --include=Kconfig .`
    - **Silent console.** `CONFIG_SERIAL_..._CONSOLE=y` and a matching
      `console=` in `bootargs` are separate requirements; missing either gives
      you a board that boots perfectly and says nothing — indistinguishable
      from a brick until you attach JTAG.
    - **Forgetting `ARCH=` on a later make.** The build silently falls back to
      the host architecture and reconfigures your tree.

## Cheat sheet

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

## Exercise

(1) Starting from `arch/arm64/configs/defconfig`, write a fragment that
enables `CONFIG_OVERLAY_FS` and `CONFIG_SQUASHFS` as built-ins and disables
`CONFIG_DEBUG_INFO`, merge it with `merge_config.sh`, and prove with `grep`
that all three landed — then explain why `CONFIG_DEBUG_INFO=n` in the
fragment would not have worked. (2) In `menuconfig`, search for a symbol
you cannot select (try `CONFIG_UBIFS_FS` with MTD off) and write down the
exact `depends on` chain hiding it. (3) Build the `hello_embed` module
out-of-tree, load it in QEMU, capture `dmesg`, `lsmod` and `modinfo`; then
rebuild your kernel with any config change and show the resulting vermagic
failure when you load the *old* `.ko`. (4) One paragraph: your storage
driver currently ships as `m`, and the device now boots from that storage.
Describe what breaks, the two valid fixes (built-in versus initramfs), and
which one you would choose for a product that must also support field
kernel upgrades.
