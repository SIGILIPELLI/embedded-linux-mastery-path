# 02 · CPU Architectures 101

"ARM or x86?" is the first question about any computer, yet most developers
can't say precisely what the difference is. This module gives you working
CPU-architecture literacy: what an ISA actually is, why ARM ships in three
"profiles" (Cortex-A/R/M) that solve completely different problems, how
x86 and ARM differ as ecosystems, and — the payoff — how to read a real
chip's spec sheet without drowning. Our worked example is the chip this
course keeps returning to: NXP's i.MX95.

## ISA vs microarchitecture

Two ideas people constantly blur:

- **ISA (Instruction Set Architecture)** — the *contract*: which
  instructions exist, how many registers, how memory is addressed. x86-64,
  AArch64 (64-bit ARM), and RV64 (64-bit RISC-V) are ISAs. A binary is
  compiled *for an ISA*.
- **Microarchitecture** — a particular *implementation* of that contract:
  pipeline depth, cache sizes, out-of-order machinery. Cortex-A55 and
  Cortex-A72 are different microarchitectures of the same AArch64 ISA — a
  binary runs on both, but at very different speed and power.

Check what your own machine is:

```console
$ uname -m
arm64        # Apple Silicon Mac — AArch64 ISA
# or:
x86_64       # Intel/AMD PC
```

That one string decides which binaries your machine can natively run — and
is exactly why module 6 needs a *cross*-compiler to build ARM programs on an
x86 PC.

## ARM's three profiles: Cortex-A, Cortex-R, Cortex-M

ARM doesn't sell one kind of core; it sells three families tuned for three
jobs. The letters spell **A**pplication, **R**eal-time, **M**icrocontroller:

| | **Cortex-A** | **Cortex-R** | **Cortex-M** |
|---|---|---|---|
| Job | Run OSes & apps | Hard real-time, safety | Microcontroller firmware |
| MMU | **Yes** → runs Linux | MPU only | MPU only (or nothing) |
| Clock | 1–3+ GHz | 400 MHz–1.5 GHz | 16–600 MHz |
| Interrupt latency | Variable (caches, OS) | Low & *deterministic* | Very low, deterministic (NVIC) |
| Runs | Linux, Android | AUTOSAR, safety RTOS | FreeRTOS, Zephyr, bare metal |
| Found in | Phones, cars' head units, i.MX95's A55s | Disk controllers, engine control, 5G modems | Sensors, motor control, i.MX95's M7 |
| Examples | A55, A76, X4 | R5, R52 | M0+, M4, M7, M33 |

The rule of thumb: **A = has an MMU = can run Linux. M = tiny, instant-on,
deterministic. R = M's determinism at A-class speeds, for safety-critical
work.**

## x86/x86-64 vs ARM/AArch64

| | **x86-64** | **ARM / AArch64** |
|---|---|---|
| Who makes chips | Intel and AMD design *and* sell the chips | ARM licenses core designs; NXP, Qualcomm, Apple, TI build the chips |
| Business model | Buy a finished CPU | License a core, surround it with *your* peripherals → custom SoCs |
| ISA style | CISC heritage: variable-length instructions, decoded into micro-ops | RISC: fixed-length instructions, load/store architecture |
| Power character | Historically desktop/server-first; idles high | Designed mobile-first; excellent perf/watt at low power |
| Firmware world | UEFI/BIOS + ACPI: firmware *describes* the machine, one generic kernel boots anywhere | Bootloader + **device tree**: the kernel must be *told* the hardware layout (module 7) |
| Embedded presence | Kiosks, industrial PCs, some cars | Overwhelmingly dominant in embedded |

The licensing model is the deep reason ARM owns embedded: NXP could take six
Cortex-A55s, a Cortex-M7, its own NPU, CAN controllers, camera interfaces,
and safety logic, and fuse them into *one custom chip* — the i.MX95. You
can't do that with an x86 core. The trade-off is fragmentation: every ARM
board is a little different, which is why device trees and BSPs exist — and
why this course spends real time on them.

!!! note "RISC-V, in one paragraph"
    **RISC-V** is an *open* ISA — no license fee to implement it. It's
    already common in small controller roles (many SoCs, including recent
    NXP parts, embed RISC-V helper cores) and Linux-capable RISC-V boards
    exist. The concepts in this course — boot flow, device trees, rootfs,
    cross-compilation — transfer to RISC-V almost unchanged; only the
    toolchain prefix and QEMU machine name differ.

## Reading a spec sheet: the i.MX95 worked example

Open any SoC product brief and extract these five things. For the i.MX95:

| What to find | i.MX95 answer | Why it matters |
|---|---|---|
| **Application cores** | 6× Cortex-A55 (AArch64) | Linux runs here; A55 = efficiency-class → fanless designs |
| **Real-time / helper cores** | 1× Cortex-M7 + 1× Cortex-M33 | M7 runs an RTOS for real-time I/O; M33 is a *system manager* (power, safety) that boots first |
| **Accelerators** | eIQ Neutron NPU; Arm Mali GPU; ISP | ML inference, graphics, camera — jobs the CPU shouldn't burn watts on |
| **Memory interface** | LPDDR5/LPDDR4X | External DDR = MPU-class = Linux-capable (module 1's test) |
| **I/O & niche features** | CAN FD, 10G Ethernet, PCIe, MIPI camera/display; functional-safety (ISO 26262) support | Tells you the target market: automotive & industrial |

A chip like this is called **heterogeneous**: different core types sharing
one die, each doing what it's best at. Linux on the A55s runs the UI and
networking; the M7 spins motors or samples sensors with microsecond
determinism; the M33 supervises power and safety; the NPU runs the vision
model. Level 3's remoteproc/RPMsg module shows how Linux and the M7
actually talk to each other.

```console
# Inside any ARM Linux system (including your module-4 QEMU guest):
$ cat /proc/cpuinfo | head -8
processor       : 0
BogoMIPS        : 125.00
Features        : fp asimd evtstrm aes pmull sha1 sha2 crc32 ...
CPU implementer : 0x41        # 0x41 = ARM Ltd
CPU architecture: 8           # ARMv8 → AArch64
CPU variant     : 0x0
CPU part        : 0xd05       # 0xd05 = Cortex-A55
```

That `CPU part` field is how you identify a core from a running system —
`0xd05` is literally the Cortex-A55's part number, the same core the i.MX95
uses.

## Cheat sheet

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

## Exercise

Do a spec-sheet teardown of **two** chips using the five-row method above
(application cores / helper cores / accelerators / memory / I/O): (1) the
**i.MX95** (product brief on nxp.com), and (2) one chip from a device you
own — the Raspberry Pi 5's BCM2712 and the ESP32-S3 are both easy to find.
Then answer in one sentence each: Which of your two chips could run Linux,
and what hardware fact proves it? For the i.MX95, why does it carry *both*
an M7 and six A55s instead of just eight A55s? And which ISA are the
binaries in module 6 of this course compiled for?
