# 06 · Cross-Compilation

Your laptop is (probably) x86-64 or Apple Silicon; your target is an ARM
Linux board. The target can't build its own software: production images
ship **no compiler** (attack surface, flash space — module 5's logic), and
even if one were there, a 6-core i.MX95 is still no match for your laptop —
and a 400 MHz single-core industrial module has no chance at all. So
embedded development runs on a fundamental split: **build on the host, run
on the target.** The tool that makes it possible is a
**cross-compiler** — a compiler that runs on one architecture and emits
binaries for another. In this module you'll install one, cross-compile a C
program, ship it into your QEMU guest over the network, and hit (then fix)
the most classic cross-compilation failure there is.

## Toolchains and triplets

A cross-toolchain's commands are prefixed with a **target triplet** that
encodes *architecture-vendor-OS-libc*:

```text
aarch64-linux-gnu-gcc
   │       │    └── userland ABI: GNU = glibc  (musl = musl libc)
   │       └────── kernel/OS: Linux
   └────────────── architecture: 64-bit ARM
```

The prefix answers "what will the output run on?" — `aarch64-linux-gnu-gcc`
produces AArch64 Linux binaries linked against glibc, no matter what
machine the compiler itself runs on. The libc part *matters*, as you're
about to see.

## Install a cross-toolchain

=== "Ubuntu / Debian"

    ```console
    $ sudo apt install gcc-aarch64-linux-gnu
    $ aarch64-linux-gnu-gcc --version
    aarch64-linux-gnu-gcc (Ubuntu ...) 13.x.x
    ```

=== "macOS (zig cc)"

    macOS has no native apt-style Linux cross-gcc, but **zig** embeds
    clang plus libc headers for dozens of targets — one install,
    every target:

    ```console
    $ brew install zig
    $ zig cc -target aarch64-linux-musl --version
    ```

    (Alternative: `brew tap messense/macos-cross-toolchains && brew
    install aarch64-unknown-linux-musl` for a classic GCC toolchain.)

## Cross-compile hello.c

```c
/* hello.c */
#include <stdio.h>
#include <sys/utsname.h>

int main(void) {
    struct utsname u;
    uname(&u);
    printf("Hello from %s on %s!\n", u.sysname, u.machine);
    return 0;
}
```

Build it natively first, then cross:

```console
$ gcc -o hello-native hello.c && ./hello-native
Hello from Linux on x86_64!          # (or Darwin on arm64 on a Mac)

$ aarch64-linux-gnu-gcc -o hello hello.c        # Linux host
$ # macOS:  zig cc -target aarch64-linux-musl -o hello hello.c

$ file hello
hello: ELF 64-bit LSB pie executable, ARM aarch64, dynamically linked,
interpreter /lib/ld-linux-aarch64.so.1, ...
```

`file` is your truth-teller: **ARM aarch64**. Your x86 machine cannot run
this (`./hello` → `Exec format error`) — that error is the proof cross-
compilation worked.

## Ship it to the target

Real boards receive binaries over serial, SD cards, or the network. Our
QEMU guest has user-mode networking, where the host is always reachable at
**10.0.2.2**. So: serve the file on the host, fetch it in the guest.

On the **host**, in the directory containing `hello`:

```console
$ python3 -m http.server 8000
```

In the **guest** (module-4 Alpine, logged in as root) — bring up the
network with pure BusyBox tools, then download:

```console
localhost:~# ifconfig eth0 up
localhost:~# udhcpc -i eth0          # BusyBox DHCP client
udhcpc: lease of 10.0.2.15 obtained ...
localhost:~# wget http://10.0.2.2:8000/hello
localhost:~# chmod +x hello
localhost:~# ./hello
```

## The classic failure — and what it teaches

If you built with `aarch64-linux-gnu-gcc` (a **glibc** toolchain), that
last command fails on Alpine:

```console
localhost:~# ./hello
-ash: ./hello: not found
```

"Not found"?! The file is *right there*. This misleading error is a rite of
passage: the shell isn't missing your program — it's missing the program's
**interpreter**, `/lib/ld-linux-aarch64.so.1`, the glibc dynamic loader
named inside the ELF header. Alpine uses **musl**, not glibc, so that
loader doesn't exist. Architecture matched; **libc didn't**. The general
law: *a dynamically linked binary must match the target's libc and library
versions* — which is why toolchains are built against a **sysroot**, a
copy of the target's headers and libraries that the cross-compiler compiles
and links against instead of your host's (`aarch64-linux-gnu-gcc
-print-sysroot` shows yours). Buildroot will generate a sysroot that
exactly matches your rootfs in module 9 — making this whole class of bug
impossible.

The quick fix today — **static linking**:

```console
$ aarch64-linux-gnu-gcc -static -o hello hello.c    # (zig musl builds are already static)
$ file hello
hello: ELF 64-bit LSB executable, ARM aarch64, statically linked, ...
```

Re-serve, re-`wget`, and:

```console
localhost:~# ./hello
Hello from Linux on aarch64!
```

**Static vs dynamic on embedded** is a genuine tradeoff, not a default:

| | Static | Dynamic |
|---|---|---|
| Binary size | Big (~600 KB+ for hello w/ glibc) | Tiny (~10 KB) |
| Deps on target | None — runs anywhere the ISA matches | Needs exact libc/libs on target |
| Many programs | Each duplicates libc → flash bloat | All share one libc → the norm in full images |
| Updates | Rebuild everything on a libc CVE | Update one shared library |
| Sweet spot | One-off tools, containers, rescue binaries | Complete images built by Buildroot/Yocto |

## Cheat sheet

| Command | Purpose |
|---------|---------|
| `aarch64-linux-gnu-gcc hello.c -o hello` | Cross-compile for AArch64 Linux (glibc) |
| `zig cc -target aarch64-linux-musl ...` | Cross-compile from macOS (or anywhere), musl, static |
| `file <binary>` | Verify target arch, linkage, and interpreter |
| `-static` | Self-contained binary — no libc match needed |
| `aarch64-linux-gnu-gcc -print-sysroot` | Show the toolchain's target headers/libs root |
| `python3 -m http.server 8000` (host) | One-line file server for the lab |
| `udhcpc -i eth0` (guest) | BusyBox DHCP — bring up guest networking |
| `wget http://10.0.2.2:8000/f` (guest) | Fetch from host (10.0.2.2 = host in QEMU user networking) |
| `uname -m` / `Exec format error` | ISA check / the "wrong architecture" symptom |
| triplet `arch-os-libc` | Reads as: what the output binary runs on |

## Exercise

(1) Cross-compile a `sysinfo.c` that prints total and free RAM using
`sysinfo(2)` (`struct sysinfo`, fields `totalram`, `freeram`, scaled by
`mem_unit`), ship it to the guest, and check its numbers against
`/proc/meminfo` from module 5. (2) Build it both `-static` and dynamic;
record both sizes with `ls -lh` and both `file` one-liners, and state
which one runs on Alpine and *exactly* why the other fails — quoting the
interpreter path from `file`'s output. (3) One-sentence answers: why do
production images ship no compiler? What is a sysroot? And if you needed
this same program on an **i.MX95**, what (if anything) changes about the
build command? (Hint: the i.MX95's A55s and QEMU's A53 speak the same
ISA; the answer is about the BSP's libc.)
