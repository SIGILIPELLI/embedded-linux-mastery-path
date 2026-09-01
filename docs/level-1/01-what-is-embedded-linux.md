# 01 · What Is Embedded Linux?

Your WiFi router runs Linux. So does your smart TV, your car's infotainment
system, the ticket machine at the station, the industrial controller on a
factory floor, and (probably) your doorbell camera. **Embedded Linux** is the
same Linux kernel that powers servers and Android phones, stripped down and
tailored to run *inside a product* — a device that the user never thinks of as
"a computer." This module maps the territory: where embedded Linux lives, why
some chips can run it and some can't, and what the software stack looks like
from bootloader to application.

## Where embedded Linux lives

| Product | What Linux does there |
|---------|----------------------|
| Home router / access point | Routing, firewall, web admin UI (often OpenWrt) |
| Smart TV / set-top box | App platform, media decoding, DRM (webOS, Tizen, Android TV) |
| Car | Infotainment (IVI), instrument cluster, telematics — alongside separate real-time microcontrollers |
| Industrial equipment | PLC-adjacent controllers, HMI touch panels, protocol gateways |
| Robots & drones | Vision, planning, connectivity — while a microcontroller handles motor control |
| Payment terminals, kiosks, medical devices | UI, networking, crypto, updates |

The pattern to notice: Linux handles the *complex, connected, UI-heavy,
multi-process* part of the product. Hard real-time jobs (spinning a motor at
exactly the right microsecond) usually go to a separate microcontroller — or,
on modern chips, to a microcontroller core *on the same silicon*. Keep that
thought; it becomes the i.MX95 story below.

## MCU vs MPU — why an ESP32 can't run Linux

The embedded world splits into two camps:

| | **MCU** (microcontroller) | **MPU** (microprocessor / applications processor) |
|---|---|---|
| Example | ESP32, STM32, ATmega328 | i.MX95, Raspberry Pi's BCM2712, TI AM62 |
| Typical CPU | ARM Cortex-M, Xtensa, AVR | ARM Cortex-A, x86 |
| Clock | 16–240 MHz | 1–2+ GHz, multiple cores |
| RAM | 2 KB – ~1 MB **on-chip SRAM** | 512 MB – 16 GB **external DDR** |
| Storage | On-chip flash (kB–MB) | eMMC / SD / SSD (GB) |
| **MMU** | No (at most an MPU — memory *protection* unit) | **Yes** |
| Runs | Bare-metal loops, RTOS (FreeRTOS, Zephyr) | Linux, Android, QNX |
| Boots in | Microseconds–milliseconds | Seconds |
| Power | µW–mW, runs on a coin cell | Hundreds of mW–several W |

Two hardware facts decide whether Linux is possible:

1. **The MMU (Memory Management Unit).** Standard Linux requires one. The MMU
   translates each process's *virtual* addresses to physical RAM, which is
   what makes separate address spaces, `fork()`, demand paging, and "a crashed
   app doesn't take down the system" possible. Cortex-M cores (the ESP32
   class) have no MMU. (A niche no-MMU Linux exists — µClinux heritage — but
   nothing mainstream ships it.)
2. **RAM.** A useful Linux system wants tens of megabytes at absolute
   minimum; real products use 512 MB–4 GB of external DDR. An ESP32 has
   ~520 KB of SRAM. It's not a close call.

So: an **ESP32 runs FreeRTOS or bare-metal firmware** (that's the
[Embedded Systems Mastery Path](https://sigilipelli.github.io/embedded-mastery-path/)),
while an **i.MX95 runs Linux** — because it has six Cortex-A55 cores with
MMUs, a DDR controller, and gigabytes of RAM to manage.

!!! note "The i.MX95 — this course's running example"
    NXP's **i.MX95** is a current-generation applications processor:
    **6× ARM Cortex-A55** (the Linux side), plus a **Cortex-M7** real-time
    core, a **Cortex-M33** system-manager core, and an **eIQ Neutron NPU**
    for ML — all on one chip. It's aimed at cars, industrial systems, and
    smart devices. We use it throughout as the concrete "real product
    platform," but you never need the hardware: every lab in this course runs
    in QEMU on your laptop.

## The software stack at a glance

Every embedded Linux device is built from the same four layers:

```text
┌─────────────────────────────────────────────┐
│  Applications        your product's code    │  ← what the customer sees
├─────────────────────────────────────────────┤
│  Root filesystem     libraries, BusyBox,    │  ← /bin, /lib, /etc ...
│  (rootfs)            init system, configs   │
├─────────────────────────────────────────────┤
│  Linux kernel        scheduling, memory,    │  ← one file: the kernel image
│  (+ device tree)     drivers, filesystems   │
├─────────────────────────────────────────────┤
│  Bootloader          U-Boot: init DDR,      │  ← first code you control
│  (SPL + U-Boot)      load & start kernel    │
└─────────────────────────────────────────────┘
        Hardware: SoC (e.g. i.MX95) + DDR + eMMC
```

- **Bootloader** — typically [U-Boot](https://www.denx.de/wiki/U-Boot).
  Initializes RAM, finds the kernel, starts it. Module 3 walks the whole boot
  flow.
- **Kernel** — the one true Linux kernel, configured for your board, told
  about the hardware by a **device tree** (module 7).
- **Rootfs** — *not* Ubuntu. Usually a minimal, purpose-built filesystem
  around **BusyBox** (module 5), a few shared libraries, and an init system
  (module 8).
- **Applications** — your daemon, your UI, your business logic —
  cross-compiled on a PC (module 6).

## Who builds all this? Buildroot and Yocto (preview)

Nobody assembles those four layers by hand in production. Two build systems
dominate:

- **Buildroot** — simple, fast, `make menuconfig`-driven; produces one image
  from one config. Great for learning and small products. You'll use it in
  modules 9–10.
- **The Yocto Project** — layer-based, package-aware, industrial-strength,
  and the basis of nearly every silicon vendor's official BSP (board support
  package) — including NXP's i.MX BSPs. Steeper curve; it's the heart of
  Level 2.

A useful mental model: *Buildroot builds you a firmware image; Yocto builds
you a custom Linux distribution.*

## Cheat sheet

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

## Exercise

Pick two devices you own that you suspect run Linux (router, TV, camera, car
head unit...). For each, find evidence online: search
"`<device model>` GPL source code" or "`<device model>` open source notice" —
vendors shipping Linux must publish GPL sources, and those pages usually name
the exact kernel version and SoC. Write down for each device: (1) the SoC and
its CPU core type (Cortex-A? which one?), (2) how much RAM it has, and
(3) one job in that product you think is *not* handled by Linux (hint: think
motor control, power sequencing, radio real-time work). Then look up the
i.MX95 product page on nxp.com and identify which of its cores would handle
each of your three answers.
