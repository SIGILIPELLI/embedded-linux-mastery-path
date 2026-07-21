# Embedded Linux & Architectures Mastery Path

A free, structured, module-wise embedded Linux and CPU-architecture training
program — entry level to master level. It covers the full embedded Linux stack
(bootloader → kernel → rootfs → apps), CPU-architecture literacy (ARM
Cortex-A/M/R, x86/x86-64, a glimpse of RISC-V), and build systems (Buildroot
and Yocto), with NXP's **i.MX95** applications processor as the running
real-world platform example.

**No hardware required.** Every hands-on lab runs in **QEMU**
(`qemu-system-aarch64`) on an ordinary Linux or macOS machine, with real,
reproducible commands and documented image sources.

**Live site:** https://sigilipelli.github.io/embedded-linux-mastery-path/

## Contents

- **Level 1 · Entry** — what embedded Linux is (MCU vs MPU), CPU architectures 101 (Cortex-A/M/R, x86 vs ARM, reading an i.MX95 spec sheet), the boot flow, booting embedded Linux in QEMU, BusyBox & the minimal userland, cross-compilation, device trees, init systems (BusyBox init vs systemd), building a rootfs with Buildroot, capstone (tiny Linux appliance in QEMU)
- **Level 2 · Intermediate** — Yocto fundamentals, U-Boot deep dive, kernel config & modules, udev, embedded networking, storage & filesystems (UBI, overlayfs), debugging & tracing, read-only rootfs, systemd deep dive, custom Yocto image project
- **Level 3 · Advanced** — kernel driver development, platform drivers & the device model, device-tree overlays, interrupts & kernel concurrency, heterogeneous compute (remoteproc/RPMsg with the i.MX Cortex-M coprocessor), PREEMPT_RT, graphics (DRM/KMS & Wayland), power management, profiling
- **Level 4 · Master** — production BSP maintenance, secure boot (HAB/AHAB), OTA updates (RAUC/SWUpdate/OSTree), OP-TEE/TrustZone, NPU & ML acceleration (eIQ), containers on embedded, fleet management, security hardening, licensing & compliance, capstone (production-grade i.MX95 product)

## Local development

```bash
python3 -m venv .venv
.venv/bin/pip install mkdocs-material
.venv/bin/python -m mkdocs serve
```

## Related

- [Embedded Systems Mastery Path](https://sigilipelli.github.io/embedded-mastery-path/) — the MCU side (Arduino/ESP32, bare-metal Cortex-M)
- [Shell Mastery Path](https://sigilipelli.github.io/shell-mastery-path/) — recommended before this course
- [C Mastery Path](https://sigilipelli.github.io/c-mastery-path/)
- [C++ Mastery Path](https://sigilipelli.github.io/cpp-mastery-path/)
- [Python Mastery Path](https://sigilipelli.github.io/python-mastery-path/)
- [Java Mastery Path](https://sigilipelli.github.io/java-mastery-path/)
- [JavaScript Mastery Path](https://sigilipelli.github.io/javascript-mastery-path/)
- [Go Mastery Path](https://sigilipelli.github.io/go-mastery-path/)
- [SQL Mastery Path](https://sigilipelli.github.io/sql-mastery-path/)
- [Rust Mastery Path](https://sigilipelli.github.io/rust-mastery-path/)
- [TypeScript Mastery Path](https://sigilipelli.github.io/typescript-mastery-path/)
- [Ruby Mastery Path](https://sigilipelli.github.io/ruby-mastery-path/)
- [PHP Mastery Path](https://sigilipelli.github.io/php-mastery-path/)
- [Kotlin Mastery Path](https://sigilipelli.github.io/kotlin-mastery-path/)
- [Swift Mastery Path](https://sigilipelli.github.io/swift-mastery-path/)
- [Scala Mastery Path](https://sigilipelli.github.io/scala-mastery-path/)
- [R Mastery Path](https://sigilipelli.github.io/r-mastery-path/)
- [Dart Mastery Path](https://sigilipelli.github.io/dart-mastery-path/)
- [PowerShell Mastery Path](https://sigilipelli.github.io/powershell-mastery-path/)
- [AWS Mastery Path](https://sigilipelli.github.io/aws-mastery-path/)
- [Azure Mastery Path](https://sigilipelli.github.io/azure-mastery-path/)
- [Google Cloud Mastery Path](https://sigilipelli.github.io/gcp-mastery-path/)
- [IBM Cloud Mastery Path](https://sigilipelli.github.io/ibm-cloud-mastery-path/)
- [Adobe Creative Cloud Mastery Path](https://sigilipelli.github.io/adobe-mastery-path/)
- [AI & Machine Learning Mastery Path](https://sigilipelli.github.io/ai-ml-mastery-path/)
- [LLM Development Mastery Path](https://sigilipelli.github.io/llm-dev-mastery-path/)
- [RAG Pipelines Mastery Path](https://sigilipelli.github.io/rag-mastery-path/)
