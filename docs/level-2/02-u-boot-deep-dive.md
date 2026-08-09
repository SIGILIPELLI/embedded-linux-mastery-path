# 02 · U-Boot Deep Dive

Level 1's boot-flow module named U-Boot as the relay runner between the
boot ROM and the kernel. That was the tourist view. On a real board U-Boot
is where you spend your worst days: the kernel won't load, the device tree
is stale, the board comes up in a boot loop at a customer site, and your
only interface is a serial console and a two-second window to hit a key.
This module makes that console familiar — the shell, the environment,
boot scripts, and loading a kernel over TFTP and MMC.

## Interrupting the boot

Power on with a serial terminal attached (115200 8N1 on most i.MX boards)
and U-Boot prints a banner and counts down:

```text
U-Boot 2024.04 (Apr 12 2024 - 09:15:33 +0000)

CPU:   i.MX95 rev1.0 2000 MHz (running at 1000 MHz)
Model: NXP i.MX95 15X15 board
DRAM:  8 GiB
Core:  156 devices, 31 uclasses, devicetree: separate
MMC:   FSL_SDHC: 0, FSL_SDHC: 1
Net:   eth0: ethernet@428a0000
Hit any key to stop autoboot:  2
u-boot=>
```

That prompt is a full command interpreter with variables, conditionals and
scripting. `help` lists everything the board was built with; `help <cmd>`
explains one command.

## The environment is the configuration

U-Boot's environment is a set of key/value strings stored in
non-volatile memory (a raw eMMC offset, a SPI-NOR region, a UBI volume, or
a file). It is *the* thing you edit to change how a board boots.

```console
u-boot=> printenv bootargs
bootargs=console=ttyLP0,115200 root=/dev/mmcblk1p2 rootwait rw
u-boot=> printenv bootcmd
bootcmd=run distro_bootcmd
u-boot=> setenv bootargs "console=ttyLP0,115200 root=/dev/mmcblk1p2 rootwait ro"
u-boot=> saveenv
Saving Environment to MMC... Writing to MMC(0)... OK
u-boot=> printenv | wc -l
118
```

`setenv` changes RAM only; **`saveenv` makes it persist**. `setenv foo`
with no value deletes the variable. `env default -a -f` restores the
compiled-in defaults — your escape hatch when you have edited yourself
into a boot loop.

Variables can hold *commands*, and `run` executes them. That is the whole
scripting model:

```console
u-boot=> setenv myboot 'echo Booting...; bootm ${loadaddr} - ${fdt_addr}'
u-boot=> run myboot
```

Note `'single quotes'` around anything containing `;` — otherwise the
shell splits the command at the semicolon and you save only the first half.

## Loading a kernel from MMC

The manual boot sequence every embedded engineer should be able to type
from memory: load kernel, load device tree, hand over.

```console
u-boot=> mmc list
FSL_SDHC: 0 (eMMC)
FSL_SDHC: 1 (SD)
u-boot=> mmc dev 1
switch to partitions #0, OK
mmc1 is current device
u-boot=> ls mmc 1:1
   38797312   Image
      53248   imx95-15x15-verdin.dtb
        512   boot.scr
u-boot=> fatload mmc 1:1 ${kernel_addr_r} Image
38797312 bytes read in 812 ms (45.6 MiB/s)
u-boot=> fatload mmc 1:1 ${fdt_addr_r} imx95-15x15-verdin.dtb
53248 bytes read in 15 ms (3.4 MiB/s)
u-boot=> setenv bootargs "console=ttyLP0,115200 root=/dev/mmcblk1p2 rootwait rw"
u-boot=> booti ${kernel_addr_r} - ${fdt_addr_r}
## Flattened Device Tree blob at 83000000
   Booting using the fdt blob at 0x83000000
Starting kernel ...
```

The `-` in the middle is the initramfs slot: "no initramfs here." Getting
that argument order wrong (`booti <kernel> <initrd> <fdt>`) is the source
of an enormous number of `Bad Linux ARM64 Image magic!` reports.

| Command | Image type |
|---------|-----------|
| `booti` | Raw ARM64 `Image` |
| `bootz` | Raw ARM32 `zImage` |
| `bootm` | Legacy or FIT `uImage` (has a U-Boot header) |

## Loading a kernel over the network (TFTP)

During bring-up nobody reflashes eMMC for every kernel rebuild. You serve
the kernel over TFTP from your workstation and keep the rootfs on NFS —
edit, rebuild, reset, in seconds.

On the host:

```console
$ sudo apt install tftpd-hpa nfs-kernel-server
$ sudo cp arch/arm64/boot/Image /srv/tftp/
$ grep TFTP_DIRECTORY /etc/default/tftpd-hpa
TFTP_DIRECTORY="/srv/tftp"
```

On the board:

```console
u-boot=> setenv ipaddr 192.168.1.50
u-boot=> setenv serverip 192.168.1.10
u-boot=> ping ${serverip}
Using ethernet@428a0000 device
host 192.168.1.10 is alive
u-boot=> tftp ${kernel_addr_r} Image
Using ethernet@428a0000 device
TFTP from server 192.168.1.10; our IP address is 192.168.1.50
Filename 'Image'.
Load address: 0x80400000
Loading: #################################################  45.6 MiB/s
done
Bytes transferred = 38797312 (2500000 hex)
u-boot=> setenv bootargs "console=ttyLP0,115200 root=/dev/nfs rw \
    nfsroot=192.168.1.10:/srv/nfs/rootfs,v3,tcp ip=dhcp"
u-boot=> booti ${kernel_addr_r} - ${fdt_addr_r}
```

Wrap the whole thing into one variable so it is a single command later:

```console
u-boot=> setenv netboot 'tftp ${kernel_addr_r} Image; \
  tftp ${fdt_addr_r} imx95-15x15-verdin.dtb; \
  setenv bootargs console=ttyLP0,115200 root=/dev/nfs rw \
  nfsroot=${serverip}:/srv/nfs/rootfs,v3,tcp ip=dhcp; \
  booti ${kernel_addr_r} - ${fdt_addr_r}'
u-boot=> saveenv
u-boot=> run netboot
```

## Boot scripts and the distro boot flow

A `boot.scr` is a compiled U-Boot script placed on the boot partition, so
the *image* — not the board's saved environment — decides how it boots.
That is what makes an SD card portable between identical boards.

```console
$ cat boot.cmd
setenv bootargs console=ttyLP0,115200 root=/dev/mmcblk1p2 rootwait rw
fatload mmc 1:1 ${kernel_addr_r} Image
fatload mmc 1:1 ${fdt_addr_r} imx95-15x15-verdin.dtb
booti ${kernel_addr_r} - ${fdt_addr_r}
$ mkimage -A arm64 -O linux -T script -C none -n "boot script" \
    -d boot.cmd boot.scr
Image Name:   boot script
Image Type:   AArch64 Linux Script (uncompressed)
Data Size:    241 Bytes = 0.24 kB
```

Modern U-Boot ships **distro boot**: `bootcmd=run distro_bootcmd` walks
`boot_targets` in order, and on each device looks for `boot.scr`, then an
`extlinux/extlinux.conf`, then falls back to EFI.

```console
u-boot=> printenv boot_targets
boot_targets=mmc1 mmc0 usb0 pxe dhcp
u-boot=> setenv boot_targets "mmc0 mmc1"   # eMMC first, then SD
```

## Building U-Boot for a board

Same defconfig pattern as the kernel:

```console
$ export CROSS_COMPILE=aarch64-linux-gnu-
$ export ARCH=arm
$ ls configs | grep imx95
imx95_19x19_evk_defconfig
$ make imx95_19x19_evk_defconfig
$ make menuconfig          # e.g. Command line interface -> enable 'md5sum'
$ make -j$(nproc)
  ...
  LD      u-boot
  OBJCOPY u-boot.bin
  CAT     u-boot-nodtb.bin
```

For QEMU's `virt` machine — which needs no vendor blobs — the loop is
short enough to actually practise:

```console
$ make qemu_arm64_defconfig && make -j$(nproc)
$ qemu-system-aarch64 -M virt -cpu cortex-a57 -m 512M -nographic \
    -bios u-boot.bin
U-Boot 2024.04 ...
=> version
=> printenv
```

Do that once and every command above stops being theory.

## Traps

!!! danger "How boards get bricked"
    - **`saveenv` writes to flash.** Lose power mid-write and the environment
      region is corrupt; many boards then refuse to boot to a prompt. Boards
      with a redundant environment (`CONFIG_SYS_REDUNDAND_ENVIRONMENT`)
      survive this; boards without it need a recovery flash.
    - **Overwriting SPL or U-Boot itself** with a stray
      `mmc write ${loadaddr} 0x42 ...` at the wrong offset removes the only
      thing that can boot the board. Recovery then means serial download
      mode (`uuu` on i.MX) or a JTAG probe.
    - **Load addresses that overlap.** `${kernel_addr_r}` and `${fdt_addr_r}`
      too close together means the kernel image silently overwrites the DTB;
      the symptom is a hang right after "Starting kernel …". Check board
      defaults before inventing your own addresses.
    - **Editing `bootargs` without a fallback.** A typo in `root=` gets you
      `VFS: Unable to mount root fs` and a reboot loop. Keep a known-good
      copy in a second variable (`setenv bootargs_good "$bootargs"`).
    - **Unquoted `;` in `setenv`.** You save the first command only, then
      wonder why `run mybootcmd` half-works.

## Cheat sheet

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

## Exercise

(1) Build U-Boot with `qemu_arm64_defconfig` and run it under
`qemu-system-aarch64 -M virt -bios u-boot.bin`; at the prompt, capture the
output of `version`, `bdinfo` and `printenv boot_targets`. (2) Create a
variable `hello` that echoes two messages in sequence, save it, reset the
machine, and confirm it survived — then repeat *without* quoting the
semicolon and explain what was actually stored. (3) Write a `boot.cmd`
that loads a kernel and DTB from `mmc 0:1`, sets `bootargs` for an ext4
root on `/dev/mmcblk0p2`, and boots; compile it with `mkimage` and explain
in one sentence why shipping `boot.scr` on the image is safer than relying
on `saveenv`. (4) One paragraph: a field unit reboots in a loop because
`bootargs` names a root partition that no longer exists after an OTA
update — describe the U-Boot mechanism you would have designed in *before*
shipping so the board falls back to the previous known-good rootfs
(hint: `bootcount`, `altbootcmd`, and Level 4's OTA module).
