# 09 · Performance Profiling (perf, ftrace)

"It feels slow" is not a diagnosis. Before Level 3 you had `top` and
`dmesg`; that's enough to tell you *something* is wrong but almost never
*where*. `perf` and `ftrace` are the tools that turn a vague performance
complaint into a specific function, a specific lock, or a specific
scheduling decision you can actually fix.

## perf: sampling where the CPU actually spends time

```console
$ perf record -F 999 -a -g -- sleep 10
[ perf record: Woken up 3 times to write data ]
[ perf record: Captured and wrote 2.1 MB perf.data (5321 samples) ]
$ perf report --stdio | head -20
# Overhead  Command    Shared Object       Symbol
# ........  .........  ..................  ..........................
#
    18.42%  camera-svc  libcamera.so        image_process_frame
    11.03%  camera-svc  [kernel.kallsyms]   copy_page
     7.61%  camera-svc  libc.so.6           memcpy
     6.90%  swapper    [kernel.kallsyms]   cpuidle_enter_state
```

`-F 999` (not a round 1000) deliberately avoids aliasing with periodic
kernel timers running at exactly 1000 Hz — sampling at a frequency that
coincides with another periodic event under-samples whatever's
synchronized with it. `-g` captures call graphs so `perf report` can show
*who called* the hot function, which is usually the actually useful
question ("why is memcpy 7% of runtime" is only answerable by seeing its
callers).

Cross-compiling `perf` for the target and getting symbols right is its
own small project — without matching debug symbols and an unstripped
kernel/`vmlinux`, `perf report` degrades to raw addresses:

```console
$ perf report --stdio
    18.42%  camera-svc  libcamera.so  [unknown] (0x00007f8a3c112340)
```

Fix by pointing `perf` at symbol files explicitly:

```console
$ perf report --symfs=/path/to/rootfs -k /path/to/vmlinux
```

## perf stat: hardware counters for "why is this slow"

```console
$ perf stat -e cycles,instructions,cache-misses,cache-references ./decode_test
 Performance counter stats for './decode_test':

       842,331,204      cycles
       501,220,881      instructions              #    0.60  insn per cycle
        18,204,112      cache-misses              #   41.2 % of all cache refs
        44,190,332      cache-references

       0.412839143 seconds time elapsed
```

An IPC (instructions per cycle) of 0.60 on an in-order or modest
out-of-order embedded core, paired with a 41% cache-miss rate, points
directly at a memory-access-pattern problem (poor locality, false
sharing, or an unnecessarily large working set) rather than "the CPU is
just slow" — the fix there is data layout, not a faster algorithm in the
abstract sense.

## ftrace: seeing kernel-level event sequences, not just hot spots

`perf` answers "where is time spent"; `ftrace` answers "what happened, in
what order" — essential for latency spikes that a statistical sampler
might miss entirely because the spike is rare but large.

```console
$ cd /sys/kernel/tracing
$ echo function_graph > current_tracer
$ echo mydev_irq > set_ftrace_filter
$ echo 1 > tracing_on
$ cat trace | head -15
 3)               |  mydev_irq() {
 3)   1.230 us    |    readl();
 3)   0.410 us    |    writel();
 3)   0.890 us    |    schedule_work();
 3)   3.102 us    |  }
```

For latency specifically, the **wakeup latency tracer** shows exactly how
long a high-priority task waited to actually run after becoming runnable
— the direct, ground-truth answer to a PREEMPT_RT latency question that
`cyclictest` only measures statistically:

```console
$ echo wakeup_rt > current_tracer
$ echo 1 > tracing_on
... trigger the scenario ...
$ echo 0 > tracing_on
$ cat trace | head -20
# tracer: wakeup_rt
#
# latency: 187 us, #4/4, CPU#1 | (X#0)
    <idle>-0       0d.h4    2us : <stack trace>
  motor_ctl-812    0d.h3  187us : sched_switch: prev=swapper next=motor_ctl
```

`d.h4` decodes IRQ/preempt state flags at that trace event —
`d`=interrupts disabled, `h`=hardirq context — reading these flags
correctly is how you distinguish "waiting on an IRQ handler to finish" from
"waiting on the scheduler" as the actual cause of a latency spike, which
matters because the fix is completely different for each.

## tracepoints and dynamic events in your own driver

```c
#include <trace/events/mydev.h>   /* generated from a TRACE_EVENT() decl */

TRACE_EVENT(mydev_irq_latency,
	TP_PROTO(u64 delta_ns),
	TP_ARGS(delta_ns),
	TP_STRUCT__entry(__field(u64, delta_ns)),
	TP_fast_assign(__entry->delta_ns = delta_ns;),
	TP_printk("irq_to_thread_delta=%llu ns", __entry->delta_ns)
);
```

```console
$ echo 1 > /sys/kernel/tracing/events/mydev/mydev_irq_latency/enable
$ cat /sys/kernel/tracing/trace
 mydev_worker-892   [001] ...1  4021.881233: mydev_irq_latency: irq_to_thread_delta=42000
```

A custom tracepoint costs essentially nothing when disabled (a static key
patches the call site to a no-op) — there's rarely a reason not to
instrument a driver's critical latency-sensitive transitions this way
instead of relying only on `pr_debug`, which can't be correlated across
subsystems the way ftrace events in a shared timeline can.

## Reading `/proc` and `/sys` under load, not just at idle

```console
$ mpstat -P ALL 1
$ vmstat 1
$ cat /proc/pressure/cpu
some avg10=12.40 avg60=8.11 avg300=3.02 total=48211023
$ cat /proc/pressure/io
some avg10=0.00 avg60=0.00 avg300=0.00 total=812004
```

PSI (`/proc/pressure/*`) answers a question `top` cannot: not just "is the
CPU busy" but "is something *stalled waiting* for CPU/IO/memory" —
`avg10=12.40` on `cpu` means tasks spent 12.4% of the last 10 seconds
stalled waiting for a CPU that was busy with something else, which is a
direct, quantified answer to "is this box actually overloaded" that raw
utilization percentages can't give you (a box at 95% CPU utilization
running one long job with headroom for everything else looks identical to
an actually-saturated box on `top` alone).

## Traps

- **Profiling a debug build.** Missing `-O2`, present `-fno-omit-frame-pointer`
  debug scaffolding, and disabled compiler optimizations all change where
  time is spent — profile the actual release build, or the hot spots you
  find won't exist in production.
- **Sampling at a frequency that aliases with a periodic source** — see the
  `-F 999` note above; this silently produces misleading `perf report`
  output with no warning.
- **Treating one `perf record` run as ground truth** for an intermittent
  issue — a workload with rare stalls needs either a long capture window
  or a targeted `ftrace` trigger (function filters, tracepoints) rather
  than a short statistical sample that may simply miss the event.

## Cheat sheet

| Tool/file | Answers |
|---|---|
| `perf record -F 999 -a -g` | Where CPU time goes, with call graphs |
| `perf stat -e cycles,instructions,cache-misses` | IPC and cache behavior |
| `ftrace function_graph` | Exact call sequence and per-call duration |
| `ftrace wakeup_rt` | Ground-truth scheduling latency for a real event |
| Custom `TRACE_EVENT()` | Near-zero-cost driver-specific instrumentation |
| `/proc/pressure/{cpu,io,memory}` | Is anything actually stalled, not just busy |

!!! note "On verification"
    `perf`/`ftrace` command syntax and output formats were cross-checked
    against documented kernel tracing and perf-tools conventions and
    Level 2's debugging module; the specific numeric output shown (sample
    percentages, latency values) is illustrative, not captured from a
    real profiling run on this machine.

## Exercise

(1) Using the `mydev_irq`/`mydev_irq_thread` split from Module 4, add a
`TRACE_EVENT()` that records the delta between the hard-IRQ ack and the
threaded handler starting, and describe the exact `ftrace` command
sequence to enable and read it. (2) Given a `perf stat` result showing IPC
0.35 and cache-miss rate 60% for a "just add more compute" candidate
function, write two sentences on why adding CPU cores wouldn't fix this
and what would. (3) One paragraph: `/proc/pressure/cpu` shows
`avg10=45.0` while `top` shows 70% idle. Explain how both can be true
simultaneously and what it implies about your workload's scheduling
pattern.
