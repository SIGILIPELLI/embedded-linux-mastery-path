# Level 1 · Entry <span class="level-badge">Foundations</span>

Goal: understand what embedded Linux is and how it differs from both desktop
Linux and microcontroller firmware; become literate in CPU architectures (ARM
Cortex-A vs Cortex-M vs x86); learn the boot flow from power-on to login
prompt; and get real hands-on practice — booting ARM Linux in QEMU, exploring
a BusyBox userland, cross-compiling C programs, reading device trees, writing
init services, and building your own root filesystem with Buildroot —
finishing with a complete tiny "Linux appliance" you assemble yourself.

**You do not need to own any hardware for this level.** Every lab runs in
**QEMU** (`qemu-system-aarch64`) on a normal Linux or macOS machine, using
documented, reproducible images. NXP's **i.MX95** (6× Cortex-A55 + Cortex-M7 +
NPU) appears throughout as the "this is what a real product platform looks
like" example — you'll be able to read its spec sheet and BSP documentation
with confidence by the end, but you never need the board.

## Modules

1. [What Is Embedded Linux?](01-what-is-embedded-linux.md)
2. [CPU Architectures 101](02-cpu-architectures-101.md)
3. [The Boot Flow](03-boot-flow.md)
4. [Running Embedded Linux in QEMU](04-qemu-lab.md)
5. [BusyBox & the Minimal Userland](05-busybox-minimal-userland.md)
6. [Cross-Compilation](06-cross-compilation.md)
7. [Device Trees](07-device-trees.md)
8. [Init Systems & Services](08-init-systems-services.md)
9. [Building a Rootfs with Buildroot](09-buildroot-rootfs.md)
10. [Capstone — Tiny Linux Appliance](10-capstone-appliance.md)

By the end of this level you'll be able to explain the MCU-vs-MPU divide, read
a heterogeneous SoC's spec sheet, trace a board from power-on to login prompt,
boot and drive an ARM Linux system in QEMU, cross-compile and deploy programs
to it, read a device tree, wire a daemon into both BusyBox init and systemd,
and build a bootable root filesystem from source with Buildroot.
