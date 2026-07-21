# 04 · Running Embedded Linux in QEMU

Time to stop reading and start booting. **QEMU** is a free, open-source
machine emulator that can pretend to be a complete ARM board — CPU, RAM,
UART, disks — on your ordinary x86 or Apple Silicon laptop. In this module
you'll install QEMU, boot a real AArch64 Linux system on an emulated
Cortex-A machine, drive it entirely over the serial console (exactly how you
talk to a real board's UART), and shut it down cleanly. This QEMU guest is
**the lab for the rest of Level 1** — modules 5, 6, and the capstone all
happen inside it.

## Install QEMU

=== "Ubuntu / Debian"

    ```console
    $ sudo apt update
    $ sudo apt install qemu-system-arm qemu-efi-aarch64
    ```

    `qemu-system-arm` ships both `qemu-system-arm` (32-bit) and
    `qemu-system-aarch64` (64-bit) emulators; `qemu-efi-aarch64` provides
    the UEFI firmware file `/usr/share/qemu-efi-aarch64/QEMU_EFI.fd`.

=== "Fedora"

    ```console
    $ sudo dnf install qemu-system-aarch64 edk2-aarch64
    ```

=== "macOS (Homebrew)"

    ```console
    $ brew install qemu
    ```

    Homebrew's QEMU bundles the firmware as `edk2-aarch64-code.fd` in its
    data directory — QEMU finds it by name automatically.

Verify:

```console
$ qemu-system-aarch64 --version
QEMU emulator version 9.x.x
```

## Get a Linux image to boot

We'll boot **Alpine Linux** — a tiny (~60 MB), security-oriented distro
whose userland is **BusyBox + musl**, which makes it a perfect stand-in for
a real embedded rootfs (and a perfect setup for module 5). Alpine publishes
versioned, checksummed images at stable URLs — the `virt` flavor is built
exactly for VMs:

```console
$ mkdir -p ~/embedded-linux-lab && cd ~/embedded-linux-lab
$ curl -LO https://dl-cdn.alpinelinux.org/alpine/v3.22/releases/aarch64/alpine-virt-3.22.0-aarch64.iso
$ curl -LO https://dl-cdn.alpinelinux.org/alpine/v3.22/releases/aarch64/alpine-virt-3.22.0-aarch64.iso.sha256
```

**Always verify a downloaded image** — embedded habit number one:

```console
$ sha256sum -c alpine-virt-3.22.0-aarch64.iso.sha256     # Linux
$ shasum -a 256 -c alpine-virt-3.22.0-aarch64.iso.sha256 # macOS
alpine-virt-3.22.0-aarch64.iso: OK
```

The expected digest (from the official `.sha256` file) is
`00b25acf7acc9e5503091c0de657f25a66a93df846a6175664de4ca76d64b6b3`.

## Boot it

=== "Linux host"

    ```console
    $ qemu-system-aarch64 \
        -M virt -cpu cortex-a53 -smp 2 -m 512M \
        -bios /usr/share/qemu-efi-aarch64/QEMU_EFI.fd \
        -cdrom alpine-virt-3.22.0-aarch64.iso \
        -nic user,model=virtio-net-pci \
        -nographic
    ```

=== "macOS host"

    ```console
    $ qemu-system-aarch64 \
        -M virt -cpu cortex-a53 -smp 2 -m 512M \
        -bios edk2-aarch64-code.fd \
        -cdrom alpine-virt-3.22.0-aarch64.iso \
        -nic user,model=virtio-net-pci \
        -nographic
    ```

What each flag means — this is the vocabulary of every QEMU lab to come:

| Flag | Meaning |
|------|---------|
| `-M virt` | The *machine* (board) to emulate: `virt` is QEMU's generic ARM board — UART, virtio devices, no vendor quirks |
| `-cpu cortex-a53` | Which core to emulate — A53 is the A55's predecessor, same AArch64 ISA as the i.MX95 |
| `-smp 2 -m 512M` | 2 CPU cores, 512 MB of "DDR" |
| `-bios ...` | UEFI firmware — the boot-ROM/bootloader stage of module 3, playing the loader role here |
| `-cdrom ...` | Attach the ISO as a virtual CD |
| `-nic user,model=virtio-net-pci` | A NAT'd network card — no host config needed |
| `-nographic` | **No window: the guest's serial UART is wired to your terminal** — exactly like a real board's serial console |

You'll see the firmware, then GRUB, then a scrolling kernel log — module 3
live in front of you — ending at:

```text
Welcome to Alpine Linux 3.22
Kernel 6.12.x on an aarch64 (/dev/ttyAMA0)

localhost login:
```

Log in as **`root`** — no password (live image). Prove you're on ARM:

```console
localhost:~# uname -m
aarch64
localhost:~# grep -m1 "CPU part" /proc/cpuinfo
CPU part        : 0xd03          # 0xd03 = Cortex-A53
localhost:~# free -m | head -2
              total   used   free ...
Mem:            492     50    400 ...
```

Note the login line says `/dev/ttyAMA0` — that's the ARM PL011 UART device.
You are using the serial console, not a screen. On a real i.MX95 board this
same text would arrive over a USB-serial adapter at 115200 baud.

!!! tip "If you get GRUB but then silence"
    The `virt`-flavor ISO directs its console to the serial port, so
    `-nographic` just works. If you ever boot an image that goes quiet
    after GRUB, press ++e++ at the GRUB menu and append `console=ttyAMA0`
    to the `linux` line — you're doing bootloader surgery, module-3 style.

## Serial-console survival kit

With `-nographic`, your terminal is shared between the guest and QEMU
itself. Three key chords:

| Keys | Effect |
|------|--------|
| ++ctrl+a++ then ++x++ | **Kill QEMU** immediately (emergency exit) |
| ++ctrl+a++ then ++c++ | Toggle the QEMU *monitor* (type `info network`, `system_powerdown`...) — press again to return |
| ++ctrl+a++ then ++h++ | List all Ctrl-A commands |

## Shut down cleanly

Embedded habit number two: never just kill the power on a writable system.
From inside the guest:

```console
localhost:~# poweroff
 * Unmounting filesystems ...
reboot: Power down
```

QEMU exits and you're back at your host shell. (The live ISO runs from RAM
so it can't be corrupted — but the capstone image will have a writable
disk, so build the habit now.) `Ctrl-A x` is the fallback if a guest ever
hangs.

!!! note "Where do 'real' embedded images come from?"
    Today you booted a distro ISO because it's one download and perfectly
    reproducible. Real products don't run distro ISOs — they run purpose-
    built images. In module 9 you will *build* your own bootable kernel +
    rootfs from source with Buildroot and boot it with almost the same
    QEMU command line (swapping `-cdrom` for `-kernel` and a disk image).
    Everything you learned here transfers directly.

## Cheat sheet

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

## Exercise

Boot the guest three more times, changing one thing each time, and write
down what you observe: (1) `-smp 4 -m 1024M` — confirm the changes from
inside with `nproc` and `free -m`; (2) `-cpu cortex-a72` — check
`CPU part` in `/proc/cpuinfo` again (A72 is `0xd08`: you just swapped
*microarchitecture* without touching the ISA — module 2 made real);
(3) use ++ctrl+a++ ++c++ to enter the QEMU monitor and run
`system_powerdown`, then `quit`. Finally, from inside the guest, run
`dmesg | grep -i "ttyAMA0"` and paste the line that proves the kernel
found the serial console the firmware told it about.
