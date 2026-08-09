# 07 · Debugging & Tracing (gdbserver, strace)

Embedded debugging is different in one specific way: the machine running
the program is not the machine running the debugger. The target has 64 MB
of RAM, no compiler, no source code and stripped binaries; your laptop has
the source, the symbols and the cross-toolchain. Every technique here is a
variation on bridging that gap — and on the harder problem of diagnosing a
failure that happened at 3 a.m. on a device you cannot reach.

## Start with the cheap tools

Before attaching a debugger, exhaust the things that cost nothing:

```console
root@target:~# dmesg -T --level=err,warn
[Mon Aug  4 09:14:22 2026] mmc1: Timeout waiting for hardware interrupt.
[Mon Aug  4 09:14:23 2026] EXT4-fs (mmcblk0p3): warning: mounting fs with errors
root@target:~# dmesg -w                    # follow, like tail -f
root@target:~# cat /proc/sys/kernel/printk
7	4	1	7
```

Those four numbers are current, default, minimum and boot-time console log
levels. Raising the first to `8` sends debug messages to the console:

```console
root@target:~# echo 8 > /proc/sys/kernel/printk
root@target:~# dmesg -n 8
```

On a slow serial console this can dominate CPU time and change the timing
of the bug you are chasing — a real form of heisenbug.

Then the process view:

```console
root@target:~# ps -eo pid,ppid,stat,rss,comm --sort=-rss | head -5
  PID  PPID STAT   RSS COMMAND
  310     1 Ssl  18244 appd
  201     1 Ss     820 dropbear
root@target:~# cat /proc/310/status | grep -E "VmRSS|Threads|State"
State:	S (sleeping)
Threads:	4
VmRSS:	   18244 kB
root@target:~# ls -l /proc/310/fd | wc -l
41
root@target:~# cat /proc/310/stack        # kernel stack of a hung task
```

A process count of open file descriptors that grows monotonically is a
descriptor leak, and it is visible from `/proc` alone — no tooling needed.

## strace: what the program asked the kernel to do

`strace` is the highest value-per-effort tool in embedded debugging,
because most "the app is broken" reports are really "the app opened the
wrong path" or "the app got EACCES".

```console
root@target:~# strace -f -e trace=openat,connect -p 310
strace: Process 310 attached with 4 threads
[pid   310] openat(AT_FDCWD, "/etc/appd/config.toml", O_RDONLY) = -1 ENOENT (No such file or directory)
[pid   310] openat(AT_FDCWD, "/usr/share/appd/config.toml", O_RDONLY) = 4
[pid   312] connect(6, {sa_family=AF_INET, sin_port=htons(8883),
                        sin_addr=inet_addr("10.0.4.9")}, 16) = -1 ECONNREFUSED
```

That output answers two support tickets at once: the config is coming from
the fallback path, and the broker is refusing connections.

Useful invocations:

```console
$ strace -f -tt -T -o /tmp/trace.log ./appd     # follow forks, timestamps, durations
$ strace -c -p 310                              # summary: syscall counts and time
$ strace -e trace=%file  ./appd                 # all filesystem-touching calls
$ strace -e trace=%network ./appd               # sockets only
$ ltrace ./appd                                 # library calls (needs dynamic linking)
```

`strace -c` is how you find that a daemon is calling `stat()` 40,000 times
a second on a file that does not exist. `-T` shows how long each call took,
which turns "it feels slow" into a number.

The cost: strace stops the process at every syscall via `ptrace`. Slowdowns
of 10–100× are normal. It is a diagnostic tool, never a monitoring one.

## Cross-debugging with gdbserver

The split is simple: `gdbserver` (a few hundred KB, no symbols needed) runs
on the target; the full cross-gdb runs on your host with the unstripped
binary and the source.

On the target:

```console
root@target:~# gdbserver :2345 /usr/sbin/appd --verbose
Process /usr/sbin/appd created; pid = 415
Listening on port 2345
```

Or attach to something already running:

```console
root@target:~# gdbserver :2345 --attach 310
```

On the host:

```console
$ aarch64-poky-linux-gdb build/appd
GNU gdb (GDB) 14.1
Reading symbols from build/appd...
(gdb) set sysroot /srv/sdk/sysroots/cortexa53-poky-linux
(gdb) set substitute-path /usr/src/debug /home/dev/appd
(gdb) target remote 192.168.7.20:2345
Remote debugging using 192.168.7.20:2345
Reading symbols from /srv/sdk/.../lib/ld-linux-aarch64.so.1...
(gdb) break sensor_read
Breakpoint 1 at 0x4008a4: file sensor.c, line 42.
(gdb) continue
Breakpoint 1, sensor_read (fd=4) at sensor.c:42
42	    ssize_t n = read(fd, buf, sizeof(buf));
(gdb) bt
#0  sensor_read (fd=4) at sensor.c:42
#1  0x0000000000400a10 in poll_loop () at main.c:88
#2  0x0000000000400b3c in main () at main.c:120
(gdb) info threads
(gdb) p *cfg
$1 = {interval = 5, path = 0x412030 "/dev/modem", retries = 3}
```

`set sysroot` is the step everyone forgets. Without it gdb loads *your
host's* libc symbols against the target's libc addresses and prints
confident nonsense for every backtrace that crosses a library boundary.

Compile with `-g` and, on optimised builds, keep `-O2 -g` rather than
dropping to `-O0`: you want to debug what you ship. Yocto's
`dbg-pkgs`/`-dbg` packages carry the separated debug symbols.

## Post-mortem: core dumps

Live debugging is a luxury. Field failures need core dumps.

```console
root@target:~# ulimit -c unlimited
root@target:~# cat /proc/sys/kernel/core_pattern
|/usr/lib/systemd/systemd-coredump %P %u %g %s %t %c %h
root@target:~# coredumpctl list
TIME                        PID  UID  SIG COREFILE  EXE
Mon 2026-08-04 09:31:02 UTC  415 1000  11 present   /usr/sbin/appd
root@target:~# coredumpctl info 415
           Signal: 11 (SEGV)
    Command Line: /usr/sbin/appd --verbose
       Stack trace of thread 415:
       #0  0x0000ffff8a1c2b40 sensor_read (appd)
```

On a small system, skip systemd-coredump and write plain files:

```console
root@target:~# echo '/data/cores/core.%e.%p.%t' > /proc/sys/kernel/core_pattern
```

Then analyse on the host — never on the target:

```console
$ aarch64-poky-linux-gdb build/appd core.appd.415.1754301062
Core was generated by `/usr/sbin/appd --verbose'.
Program terminated with signal SIGSEGV, Segmentation fault.
#0  0x0000ffff8a1c2b40 in sensor_read (fd=-1) at sensor.c:42
(gdb) bt full
```

Note `fd=-1` — the crash is a missing error check on an `open()` that
failed, which the strace above already hinted at.

## Tracing the kernel side

```console
root@target:~# cd /sys/kernel/debug/tracing
root@target:/sys/kernel/debug/tracing# cat available_tracers
timerlat osnoise hwlat blk function_graph wakeup function nop
root@target:/sys/kernel/debug/tracing# echo function > current_tracer
root@target:/sys/kernel/debug/tracing# echo mmc_* > set_ftrace_filter
root@target:/sys/kernel/debug/tracing# echo 1 > tracing_on; sleep 2; echo 0 > tracing_on
root@target:/sys/kernel/debug/tracing# head -6 trace
# TASK-PID   CPU#  ||||  TIMESTAMP  FUNCTION
    appd-310  [001] d..1  812.44219: mmc_request_start <-mmc_start_request
```

`perf` gives you the profiling view when the problem is "slow" rather than
"wrong":

```console
root@target:~# perf top -p 310
root@target:~# perf record -g -p 310 -- sleep 10 && perf report --stdio
```

Both need kernel config (`CONFIG_FTRACE`, `CONFIG_PERF_EVENTS`,
`CONFIG_DEBUG_FS`) that a size-optimised production kernel often omits —
which is exactly why you keep a **debug image variant** built from the same
sources.

## Traps

!!! danger "Debugging traps"
    - **Symbol mismatch.** Debugging with a binary that is not bit-identical
      to the one on target gives plausible, wrong line numbers. Keep the
      unstripped build artefact for every release you ship, archived with the
      image.
    - **Missing `set sysroot`.** Backtraces through libc become fiction.
    - **strace in production.** The 10–100× slowdown will trip watchdogs
      (module 9) and reboot the board you were debugging.
    - **Cores written to the rootfs.** A 200 MB core dump on a read-only or
      nearly-full rootfs either fails or fills the filesystem. Point
      `core_pattern` at the data partition, and cap the size.
    - **Debug builds that behave differently.** `-O0` can hide a race that
      only exists at `-O2`. Debug the optimised build; accept "value optimized
      out".
    - **Raising `printk` to 8 changes timing** enough to hide timing bugs, and
      on a 115200-baud console can make the system unusably slow.
    - **debugfs mounted in a shipping image** exposes kernel internals to any
      root-adjacent process. Keep it in the debug variant only.

## Cheat sheet

| Command / item | Purpose |
|---|---|
| `dmesg -T --level=err,warn` / `dmesg -w` | Kernel errors with timestamps / follow |
| `echo 8 > /proc/sys/kernel/printk` | Raise console log verbosity |
| `/proc/<pid>/{status,fd,stack,maps}` | Memory, descriptors, kernel stack, mappings |
| `strace -f -tt -T -o log ./prog` | Trace syscalls: forks, timestamps, durations |
| `strace -c -p PID` | Syscall count/time summary — find hot or failing calls |
| `strace -e trace=%file,%network` | Filter to filesystem / socket calls |
| `gdbserver :2345 <prog>` | Debug stub on target |
| `gdbserver :2345 --attach PID` | Attach to a running process |
| `target remote <ip>:2345` | Connect from cross-gdb on the host |
| `set sysroot <sdk sysroot>` | **Required** for correct library symbols |
| `set substitute-path <build> <src>` | Map recorded build paths to your sources |
| `bt full` / `info threads` / `p expr` | Backtrace with locals / threads / evaluate |
| `core_pattern` + `coredumpctl list/info` | Post-mortem capture and inspection |
| `/sys/kernel/debug/tracing` (ftrace) | Kernel function and event tracing |
| `perf record -g` / `perf top` | Sampling profiler for "it's slow" |

!!! note "On verification"
    The command forms and gdb options here follow the documented interfaces
    for gdb, gdbserver, strace and ftrace; the transcripts are representative
    rather than captured, since no cross-toolchain or target build was run
    while writing this page. The outputs are useful as a guide to *what to
    look for*, not as literal expected text for your toolchain version.

## Exercise

(1) Write a small C program with a deliberate null-pointer dereference,
cross-compile it with `-O2 -g`, run it under `gdbserver` in QEMU, and get a
full backtrace from cross-gdb on your host — with and without
`set sysroot`, and describe the difference in what you see. (2) Take the
same program and diagnose it *only* from a core dump: set `core_pattern`,
crash it, copy the core to the host, and produce `bt full`. (3) Use
`strace -c` on a busy process (`busybox httpd` serving requests will do)
and identify the single most-called syscall; then use `strace -T` to find
the slowest individual call. (4) One paragraph: a field unit reboots every
few hours and you cannot reach it. List — in the order you would implement
them — the four artefacts you would add to the *next* firmware release so
the failure diagnoses itself, and note for each one what it costs in flash,
RAM or CPU.
