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

**Live site:** https://learnmasterypath.com/embedded-linux-mastery-path/

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

- [Embedded Systems Mastery Path](https://learnmasterypath.com/embedded-mastery-path/) — the MCU side (Arduino/ESP32, bare-metal Cortex-M)
- [Shell Mastery Path](https://learnmasterypath.com/shell-mastery-path/) — recommended before this course
- [C Mastery Path](https://learnmasterypath.com/c-mastery-path/)
- [C++ Mastery Path](https://learnmasterypath.com/cpp-mastery-path/)
- [Python Mastery Path](https://learnmasterypath.com/python-mastery-path/)
- [Java Mastery Path](https://learnmasterypath.com/java-mastery-path/)
- [JavaScript Mastery Path](https://learnmasterypath.com/javascript-mastery-path/)
- [Go Mastery Path](https://learnmasterypath.com/go-mastery-path/)
- [SQL Mastery Path](https://learnmasterypath.com/sql-mastery-path/)
- [Rust Mastery Path](https://learnmasterypath.com/rust-mastery-path/)
- [TypeScript Mastery Path](https://learnmasterypath.com/typescript-mastery-path/)
- [Ruby Mastery Path](https://learnmasterypath.com/ruby-mastery-path/)
- [PHP Mastery Path](https://learnmasterypath.com/php-mastery-path/)
- [Kotlin Mastery Path](https://learnmasterypath.com/kotlin-mastery-path/)
- [Swift Mastery Path](https://learnmasterypath.com/swift-mastery-path/)
- [Scala Mastery Path](https://learnmasterypath.com/scala-mastery-path/)
- [R Mastery Path](https://learnmasterypath.com/r-mastery-path/)
- [Dart Mastery Path](https://learnmasterypath.com/dart-mastery-path/)
- [PowerShell Mastery Path](https://learnmasterypath.com/powershell-mastery-path/)
- [AWS Mastery Path](https://learnmasterypath.com/aws-mastery-path/)
- [Azure Mastery Path](https://learnmasterypath.com/azure-mastery-path/)
- [Google Cloud Mastery Path](https://learnmasterypath.com/gcp-mastery-path/)
- [IBM Cloud Mastery Path](https://learnmasterypath.com/ibm-cloud-mastery-path/)
- [Adobe Creative Cloud Mastery Path](https://learnmasterypath.com/adobe-mastery-path/)
- [AI & Machine Learning Mastery Path](https://learnmasterypath.com/ai-ml-mastery-path/)
- [LLM Development Mastery Path](https://learnmasterypath.com/llm-dev-mastery-path/)
- [RAG Pipelines Mastery Path](https://learnmasterypath.com/rag-mastery-path/)
