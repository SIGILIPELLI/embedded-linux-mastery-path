# 06 · Real-Time Linux (PREEMPT_RT)

Vanilla Linux is *fast on average* and occasionally very slow — a spinlock
held too long, an interrupt storm, a scheduling decision deferred behind
higher-priority work — because nothing in the mainline scheduler promises
a worst-case latency. PREEMPT_RT patches the kernel so that worst case
becomes bounded and small (single-digit to low double-digit
microseconds on typical embedded ARM), at the cost of some average-case
throughput. That trade is exactly right for a motor controller or a
industrial fieldbus stack and exactly wrong for a video decode pipeline.

## What PREEMPT_RT actually changes

Three changes matter more than the rest:

- **Spinlocks become sleepable "rt-mutexes."** In vanilla Linux, code
  holding a spinlock cannot be preempted — the CPU is unavailable to
  anything else until the lock is released, even a lower-priority holder
  blocking a higher-priority waiter. On PREEMPT_RT, most spinlocks
  (`spinlock_t`, not `raw_spinlock_t`) become priority-inheriting mutexes:
  the holder can be preempted, and a contending higher-priority task
  boosts the holder's priority until it releases the lock.
- **Interrupt handlers become threads by default.** Most hard IRQ handlers
  run as `IRQ-<n>` kernel threads with real-time priority, schedulable and
  preemptible like any other RT task, instead of running fully atomically
  on the CPU that took the interrupt.
- **`raw_spinlock_t` stays a true spinlock.** For the small amount of code
  that genuinely must never be preempted (deep scheduler internals,
  certain hot paths), the kernel keeps a separate, non-preemptible
  primitive — using the wrong one in a driver you're porting to RT is the
  single most common porting bug.

## Priority inheritance: the problem it solves

Without it, **priority inversion** can stall a high-priority task
indefinitely:

```
Low-priority task L takes mutex M.
Medium-priority task Med preempts L (Med doesn't need M — pure priority).
High-priority task H blocks waiting for M, held by L.
Med runs indefinitely; L never gets scheduled to release M; H starves.
```

This is not a hypothetical — it's the Mars Pathfinder bug, and it
recurs constantly in embedded systems with three or more priority tiers
sharing a lock. PREEMPT_RT's rt-mutex temporarily boosts L's priority to
H's the moment H blocks on M, so Med can no longer preempt L, L finishes
and releases M quickly, and H proceeds. This is automatic for kernel
rt-mutexes; in **userspace** you must explicitly opt in with
`PTHREAD_PRIO_INHERIT`:

```c
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_setprotocol(&attr, PTHREAD_PRIO_INHERIT);
pthread_mutex_init(&mutex, &attr);
```

A real-time application using plain `pthread_mutex_init(&m, NULL)` gets
none of this protection — it is a correct, common, and very hard to
diagnose way to lose the latency guarantee PREEMPT_RT is supposed to buy
you, because the failure is intermittent and depends on exact scheduling
timing.

## SCHED_FIFO / SCHED_RR and priority assignment

```c
struct sched_param sp = { .sched_priority = 80 };

if (sched_setscheduler(0, SCHED_FIFO, &sp) != 0)
	perror("sched_setscheduler");   /* needs CAP_SYS_NICE or rtprio limit */
```

`SCHED_FIFO` runs a task until it blocks or a higher-priority task
becomes runnable — no time-slicing among equal priority unless you
explicitly yield. This is powerful and dangerous: a `SCHED_FIFO` task
with a bug that spins without blocking can starve *everything* at or
below its priority, including kernel worker threads, on that CPU.
Production RT systems isolate that CPU (`isolcpus=`) and set a watchdog.

Grant the capability without running as root everywhere:

```console
$ cat /etc/security/limits.d/rt.conf
@realtime   -   rtprio   90
@realtime   -   memlock  unlimited
```

## Measuring latency: cyclictest

`cyclictest` is the standard PREEMPT_RT acceptance test — it measures the
delta between when a timer *should* fire and when the thread actually
wakes:

```console
$ cyclictest -m -Sp90 -i200 -h400 -q -D 4h
# /dev/cpu_dma_latency set to 0us
policy: fifo: loadavg: 1.02 1.10 1.05 3/421 8821

T: 0 (  801) P:90 I:200 C: 720000 Min: 4 Act: 6 Avg: 7 Max: 41
T: 1 (  802) P:90 I:200 C: 720000 Min: 3 Act: 5 Avg: 6 Max: 38
```

`Max` is the number that matters for a real-time SLA — a system with a
6 μs average but a 41 μs max only meets a deadline requirement stricter
than 41 μs by accident, and a 4-hour soak is the minimum to catch rare
outliers from thermal throttling, a periodic housekeeping task, or an SMI
from firmware you don't control. `-m` locks memory (`mlockall`) so page
faults don't add latency to the measurement itself.

## Common latency killers on embedded ARM

- **SMIs / firmware-level interrupts** invisible to Linux entirely — the
  CPU vanishes for tens to hundreds of microseconds servicing something
  in EL3/TrustZone firmware, and no amount of kernel tuning fixes it; the
  fix is at the firmware/BSP level (disable unnecessary SMM/PSCI paths).
- **A non-RT-aware driver still using `raw_spinlock_t` correctly but
  holding it across a long loop** — technically legal, but it reintroduces
  a real preemption-disabled window on an RT kernel, showing up as an
  outlier in `cyclictest -h` histograms that vanilla-kernel testing never
  surfaced.
- **CPU frequency scaling and thermal throttling** — a `cyclictest` run
  that looks clean at idle temperature can show new max-latency outliers
  once the SoC heats up and cpufreq/thermal governors start intervening;
  always soak-test under representative thermal load, not on a cold board.

## Traps

- **Porting a driver written against vanilla assumptions** — code that
  disables interrupts (`local_irq_disable`) expecting a bounded, tiny
  critical section may now run inside a schedulable region on RT if it
  also (incorrectly) relies on spinlock-implies-atomic semantics that
  changed.
- **Testing PREEMPT_RT under `-smp 1` in QEMU** and concluding latency is
  fine — priority inversion scenarios often require multiple real cores
  and asymmetric load to reproduce; a single-core VM test tells you little
  about the multi-core worst case.
- **Forgetting `mlockall`/hugepage tuning for the actual RT application**
  and only tuning the kernel — a page fault taken by your real-time thread
  the first time it touches a heap page can dwarf every kernel-side
  latency improvement you made.

## Cheat sheet

| Concept | PREEMPT_RT behavior |
|---|---|
| `spinlock_t` | Becomes a preemptible, priority-inheriting rt-mutex |
| `raw_spinlock_t` | Stays a true non-preemptible spinlock |
| Hard IRQ handlers | Run as preemptible `IRQ-<n>` kthreads by default |
| `PTHREAD_PRIO_INHERIT` | Required opt-in for userspace priority inheritance |
| `SCHED_FIFO`/`SCHED_RR` | Real-time scheduling classes, run-to-block semantics |
| `cyclictest -m -Sp90 -D 4h` | Standard long-soak latency acceptance test |
| `isolcpus=` | Reserve a CPU from the general scheduler for RT work |

!!! note "On verification"
    PREEMPT_RT's spinlock/rt-mutex conversion, threaded-IRQ default, and
    priority-inheritance model were checked against the documented
    PREEMPT_RT design and mainline scheduler behavior; `cyclictest` output
    above is representative formatting, not a real measurement — no RT
    kernel was booted or measured on this machine.

## Exercise

(1) Write the `pthread_mutexattr_setprotocol(PTHREAD_PRIO_INHERIT)` setup
for a three-thread priority-inversion scenario (low/medium/high priority,
low and high share a mutex), then explain in your own words the exact
sequence of events PI prevents. (2) Given a `cyclictest` histogram with
Avg 7 μs and Max 340 μs at one specific hour of a 4-hour run, list three
concrete hypotheses you'd check before blaming "the kernel," in priority
order. (3) One paragraph: your team wants to run a non-RT video decode
pipeline and an RT motor-control loop on the same SoC. Describe the
CPU-isolation and scheduling-class strategy you'd use so the two don't
compete for the same core.
