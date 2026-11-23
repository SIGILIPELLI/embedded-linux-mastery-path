---
title: "Embedded Linux & Architectures Interview Questions and Answers (179 Questions)"
description: "179 Embedded Linux & Architectures interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the Embedded Linux & Architectures Mastery Path."
---

# Embedded Linux & Architectures Interview Questions and Answers

**179 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free Embedded Linux & Architectures Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · What Is Embedded Linux?](level-1/01-what-is-embedded-linux.md)

**Q1.** Why is Linux used in so many embedded products?

**Answer:** Free, portable, scalable and with a huge driver and software ecosystem

Routers, TVs, cars, controllers.

**Q2.** What does 'embedded Linux' mean?

**Answer:** Linux customized for a dedicated device, typically headless and built from source

You own the whole stack.

**Q3.** Why is there no package manager like apt on most products?

**Answer:** Images are minimal and fixed for size, security and reproducibility

You rebuild the image.

**Q4.** Which is a typical embedded constraint?

**Answer:** Limited RAM/flash, no display and power-loss risks

Design for it.

### [02 · CPU Architectures 101](level-1/02-cpu-architectures-101.md)

**Q5.** What is an ISA?

**Answer:** The instruction set architecture defining the instructions a CPU executes

ARM, x86, RISC-V.

**Q6.** Why can't you run an ARM binary on x86 directly?

**Answer:** Different instruction sets require emulation or cross-compilation

QEMU emulates.

**Q7.** What are the Arm Cortex families A, R and M for?

**Answer:** A: application (Linux), R: real-time, M: microcontroller

Different roles.

**Q8.** What does the 'endianness' of a CPU describe?

**Answer:** The byte order of multi-byte values in memory

Mostly little-endian today.

### [03 · The Boot Flow](level-1/03-boot-flow.md)

**Q9.** What's the typical boot order?

**Answer:** Boot ROM, SPL, U-Boot, kernel, init

A relay race.

**Q10.** What does the boot ROM do?

**Answer:** Runs from on-chip memory to load the first-stage bootloader from boot media

Fixed in silicon.

**Q11.** What does U-Boot do?

**Answer:** Initializes hardware, loads the kernel, device tree and initramfs, then jumps to the kernel

Interactive for debugging.

**Q12.** What does the kernel hand control to at the end of boot?

**Answer:** The init process (PID 1)

Starts userspace.

### [04 · Running Embedded Linux in QEMU](level-1/04-qemu-lab.md)

**Q13.** What is QEMU used for in the course?

**Answer:** Emulating a full ARM board on a laptop for safe practice

No hardware needed.

**Q14.** How do you stop a QEMU session started with -nographic?

**Answer:** Ctrl-A then X

QEMU's escape sequence.

**Q15.** What does `-M virt` select?

**Answer:** A generic virtual ARM machine type

Board model.

**Q16.** Why is emulation slower than native?

**Answer:** Instructions are translated at run time

Fine for labs.

### [05 · BusyBox &amp; the Minimal Userland](level-1/05-busybox-minimal-userland.md)

**Q17.** What is BusyBox?

**Answer:** A single binary providing many Unix utilities via applet names (symlinks)

Small userland.

**Q18.** Why is `ls` in BusyBox a symlink?

**Answer:** BusyBox decides what to run from argv[0], the name it was invoked with

Multi-call binary.

**Q19.** What's missing in a minimal userland?

**Answer:** apt, bash and a compiler

By design.

**Q20.** Why minimize userland?

**Answer:** Smaller attack surface and flash footprint

Ship what you need.

### [06 · Cross-Compilation](level-1/06-cross-compilation.md)

**Q21.** Why can't the target compile its own software?

**Answer:** Production images ship no compiler and have limited resources

Cross-compile on the host.

**Q22.** What does a cross-compiler's prefix like arm-linux-gnueabihf- indicate?

**Answer:** Target architecture, OS and ABI

CROSS_COMPILE.

**Q23.** What is a sysroot?

**Answer:** The target's headers and libraries used while cross-compiling

--sysroot.

**Q24.** Why match the ABI and libc?

**Answer:** Binaries must be compatible with the target's libraries

glibc vs musl.

### [07 · Device Trees](level-1/07-device-trees.md)

**Q25.** Why do ARM boards need a device tree?

**Answer:** The hardware isn't discoverable, so the tree describes it to the kernel

PCs enumerate via ACPI/PCI.

**Q26.** What format is the compiled device tree?

**Answer:** A .dtb binary produced by dtc from .dts source

U-Boot passes it to the kernel.

**Q27.** What does a node's `compatible` property do?

**Answer:** Matches the node to a driver

Driver binding.

**Q28.** What does `status = "disabled"` do?

**Answer:** Tells the kernel not to use that device

Overlays can enable.

### [08 · Init Systems &amp; Services](level-1/08-init-systems-services.md)

**Q29.** What does an init system do?

**Answer:** Starts services at boot and supervises them

PID 1.

**Q30.** What's the difference between BusyBox init and systemd?

**Answer:** BusyBox init is tiny and script-based; systemd adds dependencies, supervision and features

Choose by need.

**Q31.** Why restart a crashed service automatically?

**Answer:** A headless device has no user to restart it

Restart=always.

**Q32.** What does a service need to start reliably?

**Answer:** Correct dependencies and ordering

After= and Requires=.

### [09 · Building a Rootfs with Buildroot](level-1/09-buildroot-rootfs.md)

**Q33.** What is Buildroot?

**Answer:** A build system generating a complete embedded Linux image from source

Simple and fixed-image oriented.

**Q34.** What does `make menuconfig` do in Buildroot?

**Answer:** Configures the target, toolchain, kernel and packages

Writes .config.

**Q35.** When is Buildroot a better fit than Yocto?

**Answer:** One fixed image with a simple workflow

Yocto scales for BSPs.

**Q36.** What does Buildroot output include?

**Answer:** Kernel image, rootfs and bootloader images ready to boot

output/images.

### [10 · Capstone — Tiny Linux Appliance](level-1/10-capstone-appliance.md)

**Q37.** What does the appliance capstone assemble?

**Answer:** A Buildroot QEMU image running a cross-compiled C application as a service

All Level 1 pieces.

**Q38.** How do you verify it boots?

**Answer:** Boot in QEMU and check the service output and logs

Test.

**Q39.** What makes it an 'appliance'?

**Answer:** It does one job automatically at boot

No user needed.

**Q40.** What should the service do on crash?

**Answer:** Restart automatically

Supervised.

## Level 2 · Intermediate

### [01 · Yocto Fundamentals](level-2/01-yocto-fundamentals.md)

**Q41.** What is Yocto?

**Answer:** A framework for building custom Linux distributions with layers and recipes

BitBake builds.

**Q42.** What is a layer?

**Answer:** A collection of recipes and config (e.g. a vendor BSP layer)

meta-*.

**Q43.** What does BitBake do?

**Answer:** Executes recipes' tasks to fetch, build and package software

The build engine.

**Q44.** Why do SoC vendors ship Yocto layers?

**Answer:** Product lines need reproducible, maintainable BSPs across boards

Standard practice.

### [02 · U-Boot Deep Dive](level-2/02-u-boot-deep-dive.md)

**Q45.** What problem does U-Boot's environment solve?

**Answer:** Stores boot commands and variables like bootargs

printenv/setenv.

**Q46.** What's `bootargs`?

**Answer:** The kernel command line passed by the bootloader

root=, console=.

**Q47.** What does `saveenv` do?

**Answer:** Persists environment changes to storage

Else lost on reboot.

**Q48.** If the kernel won't load, what do you check in U-Boot?

**Answer:** Load addresses, file locations and device tree/bootargs

Common issues.

### [03 · Kernel Configuration &amp; Modules](level-2/03-kernel-config-modules.md)

**Q49.** What's the difference between built-in and a module?

**Answer:** Built-in code is in the kernel image; modules load at run time

=y vs =m.

**Q50.** What does `modprobe` do?

**Answer:** Loads a module and its dependencies

insmod doesn't resolve deps.

**Q51.** What does `make menuconfig` produce for the kernel?

**Answer:** A .config of selected options

Then build.

**Q52.** Why trim kernel config for products?

**Answer:** Smaller size, faster boot and smaller attack surface

Only needed drivers.

### [04 · udev &amp; Device Management](level-2/04-udev-device-management.md)

**Q53.** What does udev do?

**Answer:** Creates device nodes in /dev in response to kernel uevents

Rules-driven.

**Q54.** What's a udev rule used for?

**Answer:** Naming devices, setting permissions and running actions on events

/etc/udev/rules.d.

**Q55.** What does devtmpfs provide?

**Answer:** A kernel-populated /dev with basic nodes before userspace runs

mdev/udev refine.

**Q56.** What triggers a uevent?

**Answer:** Hardware being added, removed or changed

Netlink.

### [05 · Embedded Networking](level-2/05-embedded-networking.md)

**Q57.** What must an embedded network setup handle?

**Answer:** Automatic link recovery without user interaction

Robust.

**Q58.** What do systemd-networkd or ifupdown replace?

**Answer:** NetworkManager's GUI-oriented management on headless devices

Lightweight.

**Q59.** What does a static IP vs DHCP choice depend on?

**Answer:** Deployment: fixed infrastructure vs flexible networks

Both supported.

**Q60.** Why configure timeouts and retries?

**Answer:** Links come and go; recovery shouldn't need a human

At 3 a.m.

### [06 · Storage &amp; Filesystems (UBI, overlayfs)](level-2/06-storage-filesystems.md)

**Q61.** What kills embedded products most often?

**Answer:** Power cuts corrupting writable flash filesystems

Plan for it.

**Q62.** Why use UBIFS/F2FS or read-only squashfs on raw flash/eMMC?

**Answer:** Flash wear and corruption require suitable filesystems

ext4 can work with care.

**Q63.** What is wear leveling?

**Answer:** Spreading writes across flash blocks to extend life

FTL or filesystem.

**Q64.** What does `fsync` guarantee?

**Answer:** That data has reached storage before returning

Critical for durability.

### [07 · Debugging &amp; Tracing (gdbserver, strace)](level-2/07-debugging-tracing.md)

**Q65.** What tools debug a remote target?

**Answer:** gdbserver on the target with gdb on the host, plus strace/ltrace and logs

Debug cross.

**Q66.** What does `strace` show?

**Answer:** System calls a process makes

Great for behavior.

**Q67.** Why keep an unstripped copy of binaries on the host?

**Answer:** Symbols are needed for readable backtraces and gdb

Strip only the deployed copy.

**Q68.** What does `dmesg` show?

**Answer:** Kernel log messages

Driver and boot issues.

### [08 · Read-Only Rootfs &amp; Factory Reset](level-2/08-read-only-rootfs.md)

**Q69.** What does a read-only rootfs prevent?

**Answer:** Corruption of the root filesystem from unexpected power loss

Writable data goes elsewhere.

**Q70.** Where does writable data go?

**Answer:** Separate /data partition (and tmpfs for volatile data)

Persistent vs temporary.

**Q71.** How do you handle /var and /tmp on a read-only root?

**Answer:** Mount tmpfs or bind to /data

Overlay or tmpfs.

**Q72.** What's an overlayfs used for?

**Answer:** Layering a writable layer over a read-only base

Volatile or persistent.

### [09 · systemd Deep Dive](level-2/09-systemd-deep-dive.md)

**Q73.** What does a systemd `Restart=on-failure` do?

**Answer:** Restarts the service if it exits with failure

RestartSec sets delay.

**Q74.** What does the watchdog (WatchdogSec) enable?

**Answer:** Systemd restarts a service that stops sending heartbeats

Detect hangs.

**Q75.** What does `Type=notify` mean?

**Answer:** The service tells systemd when it's ready

sd_notify.

**Q76.** How do you see why a unit failed?

**Answer:** journalctl -u name and systemctl status

Logs.

### [10 · Project — Custom Yocto Image for QEMU](level-2/10-project-yocto-qemu-image.md)

**Q77.** What does the Yocto QEMU project combine?

**Answer:** A custom layer with an app recipe, unit, image recipe and persistent /data partition, booted with a read-only root

Capstone for Level 2.

**Q78.** How do you verify read-only root?

**Answer:** Try writing to / (should fail) and confirm data persists in /data

Test.

**Q79.** What's in an image recipe?

**Answer:** Which packages and features go into the rootfs

IMAGE_INSTALL.

**Q80.** Why a custom layer?

**Answer:** Keep your changes separate from upstream layers

Maintainability.

## Level 3 · Advanced

### [01 · Kernel Driver Development — Char Drivers](level-3/01-kernel-driver-development.md)

**Q81.** What's the smallest complete kernel driver unit?

**Answer:** A character driver with file operations

open/read/write/release.

**Q82.** What does `insmod` do?

**Answer:** Loads a kernel module into the running kernel

rmmod removes.

**Q83.** Why use copy_to_user instead of memcpy to userspace?

**Answer:** It safely handles user pointers and faults

Kernel/user boundary.

**Q84.** What does a driver's major/minor identify?

**Answer:** The device number mapping a node to the driver

cdev_add.

### [02 · Platform Drivers &amp; the Device Model](level-3/02-platform-drivers-device-model.md)

**Q85.** What does the platform bus do?

**Answer:** Matches platform devices (from DT) to platform drivers by compatible string

Non-discoverable devices.

**Q86.** What does `probe()` do?

**Answer:** Initializes the device when a driver is bound to it

Allocate resources.

**Q87.** What's devm_ for?

**Answer:** Managed resource allocation freed automatically on unbind

Prevents leaks.

**Q88.** How does the driver core match a driver?

**Answer:** Using of_match_table compatible strings

Declarative.

### [03 · Device Tree Overlays](level-3/03-device-tree-overlays.md)

**Q89.** What do device tree overlays do?

**Answer:** Modify the running hardware description without rebuilding the base dtb

Daughter cards.

**Q90.** Why are overlays useful?

**Answer:** Support optional hardware or per-unit configuration

Flexibility.

**Q91.** How are overlays applied?

**Answer:** By the bootloader (fdt apply) or at runtime via configfs

Platform-dependent.

**Q92.** What must match for an overlay to apply?

**Answer:** Target nodes/labels in the base tree

Else fails.

### [04 · Interrupts &amp; Kernel Concurrency](level-3/04-interrupts-kernel-concurrency.md)

**Q93.** What's the top-half/bottom-half interrupt split?

**Answer:** Minimal work in the handler, deferred work to softirq/tasklet/workqueue

Keep ISRs short.

**Q94.** Why can't you sleep in an interrupt handler?

**Answer:** It runs in atomic context with no process to schedule

Use threaded IRQs.

**Q95.** What protects data shared between an ISR and process context?

**Answer:** spin_lock_irqsave

Mutex can sleep.

**Q96.** What's SMP's effect on drivers?

**Answer:** Code can run concurrently on multiple CPUs, requiring locking

Concurrency.

### [05 · Heterogeneous Compute — remoteproc &amp; RPMsg](level-3/05-remoteproc-rpmsg.md)

**Q97.** What is remoteproc?

**Answer:** A framework to load and control firmware on a coprocessor like a Cortex-M

Heterogeneous cores.

**Q98.** What is RPMsg?

**Answer:** A messaging protocol between Linux and remote processors over virtio/shared memory

Mailbox + vrings.

**Q99.** Why use a Cortex-M alongside Linux?

**Answer:** Hard real-time and low-power tasks Linux can't guarantee

Offload.

**Q100.** How is firmware loaded?

**Answer:** Via the remoteproc sysfs interface from /lib/firmware

echo start &gt; state.

### [06 · Real-Time Linux (PREEMPT_RT)](level-3/06-real-time-preempt-rt.md)

**Q101.** What does PREEMPT_RT do?

**Answer:** Makes the kernel mostly preemptible to bound latency

Hard-ish real time.

**Q102.** What's a typical way to measure latency?

**Answer:** cyclictest

Worst-case matters.

**Q103.** What causes latency spikes?

**Answer:** Long non-preemptible sections, interrupts and priority inversion

RT tuning.

**Q104.** Which scheduling policy for RT tasks?

**Answer:** SCHED_FIFO or SCHED_RR with priorities

Set via chrt.

### [07 · Graphics — DRM/KMS &amp; Wayland](level-3/07-graphics-drm-wayland.md)

**Q105.** What does DRM/KMS handle?

**Answer:** Display modes and scanout via the kernel graphics subsystem

Direct display.

**Q106.** What's Wayland?

**Answer:** A display server protocol where compositors manage clients (e.g. Weston)

Replacing X11.

**Q107.** Why prefer DRM/Wayland in embedded?

**Answer:** Lean, modern and efficient versus X11

Fewer components.

**Q108.** What's a framebuffer?

**Answer:** A memory region mapped to pixels on the display

fbdev legacy.

### [08 · Power Management &amp; Thermal](level-3/08-power-management-thermal.md)

**Q109.** What do runtime PM and cpuidle do?

**Answer:** Lower power by suspending idle devices and CPUs

Battery life.

**Q110.** What does the thermal framework do?

**Answer:** Monitors temperatures and throttles via cooling devices

Trip points.

**Q111.** What is DVFS?

**Answer:** Dynamic voltage and frequency scaling to trade power for speed

cpufreq.

**Q112.** How do you check wake sources?

**Answer:** /sys/power and device wakeup attributes

Debug suspend.

### [09 · Performance Profiling (perf, ftrace)](level-3/09-performance-profiling.md)

**Q113.** What does `perf` do?

**Answer:** Profiles CPU events and finds hotspots

perf top/record.

**Q114.** What's ftrace?

**Answer:** A kernel tracing framework for function and event tracing

/sys/kernel/tracing.

**Q115.** What first for a performance problem?

**Answer:** Measure to find the bottleneck

Evidence.

**Q116.** What does `top` show?

**Answer:** Processes' CPU and memory use

Starting point.

### [10 · Project — Sensor Driver + Userspace Stack](level-3/10-project-sensor-driver.md)

**Q117.** What does the sensor driver project combine?

**Answer:** A platform driver bound via DT, an IIO/char interface and tests

Level 3 capstone.

**Q118.** What's IIO?

**Answer:** The Industrial I/O subsystem for sensors and converters

Standard interface.

**Q119.** How do you test a driver without hardware?

**Answer:** Emulate with QEMU or a mock device

Simulate.

**Q120.** What should the driver handle?

**Answer:** Probe errors, locking and cleanup on removal

Robust.

## Level 4 · Master

### [01 · Production BSP Maintenance](level-4/01-production-bsp-maintenance.md)

**Q121.** What does BSP maintenance involve?

**Answer:** Tracking upstream kernel/bootloader updates, security fixes and vendor layers

Long product life.

**Q122.** Why pin versions and track CVEs?

**Answer:** Reproducible builds and timely security patches

Supply chain.

**Q123.** What's an LTS kernel for?

**Answer:** Long-term stable updates for products

Plan for it.

**Q124.** What helps product maintenance?

**Answer:** CI building images and tests on every change

Automation.

### [02 · Secure Boot Chain (HAB/AHAB)](level-4/02-secure-boot-hab-ahab.md)

**Q125.** What does secure boot do?

**Answer:** Verifies each boot stage's signature before running it

Chain of trust.

**Q126.** What's the root of trust?

**Answer:** Immutable boot ROM and fused keys in the SoC

HAB/AHAB.

**Q127.** What protects the private signing key?

**Answer:** Offline HSM or secured infrastructure, never on developer laptops

Key management.

**Q128.** What happens when a signature check fails?

**Answer:** Boot is halted (or a fallback used)

Closed devices.

### [03 · OTA Updates (RAUC, SWUpdate, OSTree)](level-4/03-ota-updates.md)

**Q129.** What does an A/B OTA scheme do?

**Answer:** Updates the inactive slot and switches, rolling back if boot fails

Safe updates.

**Q130.** What tools implement OTA?

**Answer:** RAUC, SWUpdate and Mender

Signed bundles.

**Q131.** Why sign update bundles?

**Answer:** Prevent installing tampered or malicious images

Integrity.

**Q132.** What triggers rollback?

**Answer:** A failed health check or boot count exceeded

Watchdog + bootcount.

### [04 · Trusted Execution — OP-TEE &amp; TrustZone](level-4/04-op-tee-trustzone.md)

**Q133.** What does ARM TrustZone provide?

**Answer:** Hardware-isolated secure world for sensitive code and data

Secure vs normal world.

**Q134.** What is OP-TEE?

**Answer:** An open-source Trusted Execution Environment OS

Trusted applications.

**Q135.** What would you store in the secure world?

**Answer:** Keys, certificates and security-sensitive operations

Minimize exposure.

**Q136.** How does Linux talk to OP-TEE?

**Answer:** Through the TEE driver and client API (libteec)

SMC calls.

### [05 · NPU &amp; ML Acceleration (eIQ)](level-4/05-npu-ml-acceleration.md)

**Q137.** How do embedded NPUs integrate?

**Answer:** Via vendor drivers and runtimes (e.g. eIQ, Ethos-U) from userspace

Delegate.

**Q138.** What must a model be for an NPU?

**Answer:** Quantized to supported ops, compiled with the vendor tool

Fallbacks hurt.

**Q139.** What should you benchmark?

**Answer:** End-to-end latency and power on the target

Real data.

**Q140.** What does the kernel provide to NPU stacks?

**Answer:** Drivers, memory management (CMA/DMA-BUF) and scheduling

Plumbing.

### [06 · Containers &amp; Virtualization on Embedded](level-4/06-containers-virtualization.md)

**Q141.** What do containers use from Linux?

**Answer:** Namespaces and cgroups for isolation and limits

Lightweight.

**Q142.** Why containers on embedded?

**Answer:** Deploy apps independently with isolation and easy updates

Podman/Docker, LXC.

**Q143.** What's the overhead trade-off?

**Answer:** More storage/RAM and complexity

Choose deliberately.

**Q144.** What does a hypervisor add?

**Answer:** Stronger isolation between guests on one SoC

Jailhouse, Xen.

### [07 · Fleet Management &amp; Provisioning](level-4/07-fleet-management.md)

**Q145.** What does fleet management provide?

**Answer:** Remote monitoring, updates and configuration of many devices

Scale.

**Q146.** What should each device report?

**Answer:** Health, version and metrics

Telemetry.

**Q147.** What's a device identity?

**Answer:** A unique credential (cert/key) per device for authentication

Provision securely.

**Q148.** How do you roll out safely?

**Answer:** Staged rollouts with monitoring and rollback

Canary.

### [08 · Security Hardening &amp; CVE Management](level-4/08-security-hardening.md)

**Q149.** Name a hardening measure.

**Answer:** Disable unused services, least privilege, read-only root and secure boot

Reduce attack surface.

**Q150.** Why remove debug interfaces in production?

**Answer:** They can give attackers access (UART, JTAG)

Lock them down.

**Q151.** What does a firewall do?

**Answer:** Restricts inbound/outbound traffic by policy

nftables.

**Q152.** Why drop privileges after start?

**Answer:** A compromised service has limited power

Least privilege.

### [09 · Compliance &amp; Open-Source Licensing](level-4/09-compliance-licensing.md)

**Q153.** Why track open-source licenses?

**Answer:** Compliance obligations (GPL source offers, attribution)

Legal risk.

**Q154.** What does the GPL require when you distribute binaries?

**Answer:** Offering corresponding source under the same license

Include notices.

**Q155.** What's an SBOM?

**Answer:** A software bill of materials listing components and versions

Generated by Yocto/Buildroot.

**Q156.** What's a license-compliance tool?

**Answer:** FOSSology/ORT/Yocto's license manifest

Automate.

### [10 · Capstone — Production-Grade i.MX95 Product](level-4/10-capstone-imx95-product.md)

**Q157.** What does the i.MX95 capstone combine?

**Answer:** BSP, secure boot, OTA, containers/NPU and compliance for a product

Full product.

**Q158.** How do you prove production readiness?

**Answer:** Reproducible builds, tests on hardware, rollback drills and compliance review

Operate it.

**Q159.** What documentation matters?

**Answer:** Architecture, BSP maintenance, update and security procedures

Lifecycle.

**Q160.** What's the lifecycle concern?

**Answer:** Years of security updates and field support

Plan for it.

## Mixed questions across all levels

**Q161.** List the boot stages in order.

**Answer:** Boot ROM, SPL/bootloader, U-Boot, kernel, init

Each stage hands off to the next.

**Q162.** The kernel boots but can't find the UART on an ARM board. What is the likely cause?

**Answer:** The device tree doesn't describe it (or marks it disabled), since the hardware isn't discoverable

Check the dts, compatible and status.

**Q163.** `ls` on a BusyBox system is a symlink to busybox. How does it know to act as ls?

**Answer:** It inspects argv[0], the name it was invoked with

A multi-call binary.

**Q164.** You must build a C app for a 64 MB ARM board with no compiler. What do you do?

**Answer:** Cross-compile on the host with the matching toolchain/sysroot and copy the binary

Match ABI and libc.

**Q165.** A service must restart if it crashes and be flagged if it hangs. What systemd settings help?

**Answer:** Restart=on-failure and a watchdog (WatchdogSec with sd_notify)

Supervision.

**Q166.** Field units lose power often and the root filesystem corrupts. What's the standard design?

**Answer:** A read-only root with writable data on a separate partition (and tmpfs for volatile files)

Prevents corruption by construction.

**Q167.** A kernel driver needs to share data between an interrupt handler and process context. What lock?

**Answer:** spin_lock_irqsave (a mutex can sleep, which is forbidden in IRQ context)

Atomic context.

**Q168.** Why do threaded IRQs help?

**Answer:** Handlers run in process context where they can sleep, keeping hard IRQ time short

Use for slow buses.

**Q169.** An optional daughter card appears on some units. How do you avoid maintaining two device trees?

**Answer:** Use a device tree overlay applied by the bootloader or at runtime

Overlays.

**Q170.** A Cortex-M core handles hard real-time I/O beside Linux. Which frameworks connect them?

**Answer:** remoteproc to load/control firmware and RPMsg for messaging

Heterogeneous SoCs.

**Q171.** Latency spikes appear on a Linux control loop. What do you try?

**Answer:** PREEMPT_RT with SCHED_FIFO priorities, measured with cyclictest

Measure worst case.

**Q172.** What chain does secure boot rely on?

**Answer:** Each stage verifies the next stage's signature, anchored in the immutable boot ROM and fused keys

Chain of trust.

**Q173.** A firmware update fails halfway and the device must still boot. What OTA design handles it?

**Answer:** A/B slots with a boot counter and automatic rollback

Always keep a known-good image.

**Q174.** Why sign OTA bundles?

**Answer:** So devices only install authentic, untampered images

Integrity and authenticity.

**Q175.** Where should secrets such as keys live on an i.MX board?

**Answer:** In the TrustZone secure world (OP-TEE) or secure hardware, not in the Linux rootfs

Isolate.

**Q176.** Why must you track open-source licenses in a product?

**Answer:** GPL and other licenses impose obligations on distribution (source offers, notices)

Generate an SBOM.

**Q177.** A Yocto build is not reproducible and security fixes are missed. What practices help?

**Answer:** Pin layer revisions, track CVEs, and build in CI

Maintain the BSP.

**Q178.** What reduces the attack surface of a shipped device?

**Answer:** Remove unused services/packages, disable debug ports, least privilege and a firewall

Hardening.

**Q179.** How do you roll out an update to a fleet safely?

**Answer:** Staged rollout with health telemetry and automatic rollback

Canary first.

## Practical coding questions

### A/B update with rollback

Complete the toy A/B bootloader: after install(), boot() should commit the pending slot when healthy, otherwise count failed tries and roll back to the active slot after MAX_TRIES (discarding the pending image). Print the sequence for a bad v2 update followed by a good v3.

Starter code:

```
class ABBoot:
    """Toy bootloader state for an A/B OTA scheme with a boot counter."""
    MAX_TRIES = 3

    def __init__(self):
        self.slots = {"A": "v1", "B": None}
        self.active = "A"
        self.pending = None       # slot being trialled after an update
        self.tries = 0

    def install(self, version):
        inactive = "B" if self.active == "A" else "A"
        self.slots[inactive] = version
        self.pending = inactive
        self.tries = 0
        return inactive

    def boot(self, healthy):
        """Simulate one boot of the pending slot (or the active one if nothing is pending).
        Healthy: commit the pending slot as active. Unhealthy: count the try; after MAX_TRIES
        roll back to the active slot and discard the pending one. Return a status string."""
        return ""

b = ABBoot()
b.install("v2")
for h in [False, False, False]:
    print(b.boot(h))
print(b.boot(True))
b.install("v3")
print(b.boot(True))
print(b.boot(True))
```

Expected output:

```text
boot failed 1x -> retry B
boot failed 2x -> retry B
boot failed 3x -> rollback to A (v1)
boot A (v1)
boot B (v3) -> committed
boot B (v3)
```

??? success "Show a solution"

    ```
    class ABBoot:
        """Toy bootloader state for an A/B OTA scheme with a boot counter."""
        MAX_TRIES = 3

        def __init__(self):
            self.slots = {"A": "v1", "B": None}
            self.active = "A"
            self.pending = None       # slot being trialled after an update
            self.tries = 0

        def install(self, version):
            inactive = "B" if self.active == "A" else "A"
            self.slots[inactive] = version
            self.pending = inactive
            self.tries = 0
            return inactive

        def boot(self, healthy):
            """Simulate one boot. `healthy` says whether the new software comes up fine."""
            if self.pending is None:
                return f"boot {self.active} ({self.slots[self.active]})"
            if healthy:
                self.active, self.pending = self.pending, None
                return f"boot {self.active} ({self.slots[self.active]}) -> committed"
            self.tries += 1
            if self.tries >= self.MAX_TRIES:
                bad = self.pending
                self.pending = None
                self.slots[bad] = None
                return f"boot failed {self.tries}x -> rollback to {self.active} ({self.slots[self.active]})"
            return f"boot failed {self.tries}x -> retry {self.pending}"

    b = ABBoot()
    b.install("v2")
    for h in [False, False, False]:
        print(b.boot(h))
    print(b.boot(True))
    b.install("v3")
    print(b.boot(True))
    print(b.boot(True))
    ```

    The active slot is never overwritten, so a failed update falls back to a known-good image. Real systems (RAUC, SWUpdate, Mender with U-Boot boot counts) implement the same idea. This is a teaching model. Verified by running the solution.

### Match device tree nodes to drivers

Implement match(node): skip nodes whose status is not okay, then walk the node's compatible list in order (most specific first) and return the first driver whose table contains one of the strings, along with the matching string.

Starter code:

```
# A tiny model of the Linux driver core matching a device-tree node to a driver.
DRIVERS = {
    "imx-uart": ["fsl,imx6q-uart", "fsl,imx21-uart"],
    "ns16550": ["ns16550a", "ns16550"],
    "gpio-led": ["gpio-leds"],
}

NODES = [
    {"name": "serial@4000", "compatible": ["fsl,imx6q-uart", "fsl,imx21-uart"], "status": "okay"},
    {"name": "serial@5000", "compatible": ["vendor,new-uart", "ns16550a"], "status": "okay"},
    {"name": "leds", "compatible": ["gpio-leds"], "status": "disabled"},
    {"name": "mystery@9000", "compatible": ["acme,unknown"], "status": "okay"},
]

def match(node):
    # return (driver, matching compatible string) or None; skip nodes whose status is not "okay";
    # check node["compatible"] in order (most specific first) against each driver table
    return None

for n in NODES:
    r = match(n)
    print(n["name"], "->", f"{r[0]} (via {r[1]})" if r else "no driver bound")
```

Expected output:

```text
serial@4000 -> imx-uart (via fsl,imx6q-uart)
serial@5000 -> ns16550 (via ns16550a)
leds -> no driver bound
mystery@9000 -> no driver bound
```

??? success "Show a solution"

    ```
    # A tiny model of the Linux driver core matching a device-tree node to a driver.
    DRIVERS = {
        "imx-uart": ["fsl,imx6q-uart", "fsl,imx21-uart"],
        "ns16550": ["ns16550a", "ns16550"],
        "gpio-led": ["gpio-leds"],
    }

    NODES = [
        {"name": "serial@4000", "compatible": ["fsl,imx6q-uart", "fsl,imx21-uart"], "status": "okay"},
        {"name": "serial@5000", "compatible": ["vendor,new-uart", "ns16550a"], "status": "okay"},
        {"name": "leds", "compatible": ["gpio-leds"], "status": "disabled"},
        {"name": "mystery@9000", "compatible": ["acme,unknown"], "status": "okay"},
    ]

    def match(node):
        if node["status"] != "okay":
            return None
        # compatible is ordered most specific first; the first string any driver claims wins
        for compat in node["compatible"]:
            for driver, table in DRIVERS.items():
                if compat in table:
                    return driver, compat
        return None

    for n in NODES:
        r = match(n)
        print(n["name"], "->", f"{r[0]} (via {r[1]})" if r else "no driver bound")
    ```

    The driver core binds a driver by compatible string, preferring the earliest (most specific) entry, so serial@5000 binds to ns16550 via its fallback string. Disabled or unknown nodes bind nothing. This models the idea, not the kernel's actual code. Verified by running the solution.

---

More free practice: the [Embedded Linux & Architectures Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
