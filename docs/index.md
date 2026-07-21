# Embedded Linux & Architectures Mastery Path

A structured, module-wise training program that takes you from "what even is
embedded Linux?" to building, booting, and shipping production-grade Linux
systems — covering the full software stack (bootloader → kernel → root
filesystem → applications), CPU-architecture literacy (ARM Cortex-A/M/R,
x86/x86-64, RISC-V), and the build systems the industry actually uses
(Buildroot and Yocto). NXP's **i.MX95** applications processor — six Cortex-A55
cores, a Cortex-M7 real-time core, and an eIQ Neutron NPU on one chip — is the
running real-world example woven through the lessons, so you always see how
each concept maps onto a real product platform.

**No hardware required.** Every hands-on lab runs in **QEMU** — the free,
open-source machine emulator — on an ordinary Linux or macOS machine. You will
boot real ARM Linux images, explore a BusyBox userland, cross-compile C
programs, and build your own root filesystem, all on your laptop.

## How the program is organized

| Level | Focus | Modules |
|-------|-------|---------|
| [Level 1 · Entry](level-1/index.md) | Embedded Linux fundamentals: architectures, boot flow, QEMU labs, BusyBox, cross-compiling, device trees, init, Buildroot | 9 topics + 1 capstone |
| [Level 2 · Intermediate](level-2/index.md) | Yocto, U-Boot, kernel config & modules, udev, networking, storage, debugging, systemd | 9 topics + 1 project |
| [Level 3 · Advanced](level-3/index.md) | Kernel drivers, device model, DT overlays, remoteproc/RPMsg, PREEMPT_RT, graphics, profiling | 9 topics + 1 project |
| [Level 4 · Master](level-4/index.md) | Production BSPs, secure boot (HAB/AHAB), OTA, OP-TEE, NPU/eIQ, fleet management, compliance | 9 topics + 1 capstone |

## Recommended background

- [Shell Mastery Path](https://sigilipelli.github.io/shell-mastery-path/) —
  you will live on the command line here; do that course first if the shell is
  new to you.
- [C Mastery Path](https://sigilipelli.github.io/c-mastery-path/) — the
  cross-compilation and capstone modules write small C programs.
- [Embedded Systems Mastery Path](https://sigilipelli.github.io/embedded-mastery-path/)
  — the microcontroller (Cortex-M/Arduino/ESP32) side of embedded; a great
  companion but not a prerequisite.

## How to use this site

- Work through each level in order — later modules assume earlier ones.
- Every topic page has real commands with expected output — run them in your
  own terminal and QEMU guest as you read.
- Each level ends with a project that combines everything learned in that
  level.
- Use the search bar (top of the page) to jump straight to a topic.

Start here → [Level 1 · Entry](level-1/index.md)

## More from the Mastery Path series

Free, module-wise, entry-to-master training for other languages and platforms:

- [Embedded Systems Mastery Path](https://sigilipelli.github.io/embedded-mastery-path/)
- [Shell Mastery Path](https://sigilipelli.github.io/shell-mastery-path/)
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
