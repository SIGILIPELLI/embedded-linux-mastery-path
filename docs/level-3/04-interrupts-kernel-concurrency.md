# 04 · Interrupts & Kernel Concurrency

Module 1's driver had exactly one writer and one reader, and neither ran
concurrently with itself. Real drivers have interrupt handlers firing
asynchronously against process context, and on SMP boards — every i.MX8
you'll ship — a second CPU can be inside your `read()` at the exact
instant an IRQ fires on the first. Getting this wrong doesn't crash
reliably; it corrupts data occasionally, under load, in the field, weeks
after your QEMU testing looked clean.

## Requesting an interrupt

```c
#include <linux/interrupt.h>
#include <linux/spinlock.h>

struct mydev_priv {
	void __iomem *regs;
	spinlock_t   lock;
	u32          last_status;
	struct work_struct bh_work;
};

static irqreturn_t mydev_irq(int irq, void *dev_id)
{
	struct mydev_priv *priv = dev_id;
	u32 status;
	unsigned long flags;

	status = readl(priv->regs + REG_STATUS);
	if (!(status & STATUS_IRQ_PENDING))
		return IRQ_NONE;          /* not ours (shared line) */

	writel(status, priv->regs + REG_STATUS_CLEAR);  /* ack in hardware */

	spin_lock_irqsave(&priv->lock, flags);
	priv->last_status = status;
	spin_unlock_irqrestore(&priv->lock, flags);

	schedule_work(&priv->bh_work);
	return IRQ_HANDLED;
}
```

```c
ret = devm_request_irq(&pdev->dev, irq, mydev_irq, IRQF_SHARED,
			dev_name(&pdev->dev), priv);
```

`IRQ_NONE` vs `IRQ_HANDLED` matters more than it looks: on a shared IRQ
line, returning `IRQ_HANDLED` for an interrupt your device didn't
actually raise fools the kernel's "which handler is this?" logic, and
after enough spurious claims the kernel disables the line entirely
(`irq nobody cared` in `dmesg`) — taking down every device sharing it.

## The hard rule: what you cannot do in a hard IRQ handler

A hard interrupt handler runs with (mostly) interrupts disabled on that
CPU, cannot sleep, and must be fast. Concretely, from inside
`mydev_irq` you must never:

- call anything that can block: `mutex_lock`, `kmalloc(GFP_KERNEL)`,
  `copy_to_user`, any I2C/SPI transaction (those sleep waiting on their
  own completion)
- take a spinlock also taken by *another* interrupt handler without
  `_irqsave` — see below
- do meaningful work that isn't "read status, ack hardware, hand off"

Real hardware register reads/writes are fine (they don't sleep); a
sleeping I2C sensor read triggered directly from an interrupt handler is
a top-five reason a Linux driver deadlocks in the field.

## Deferring work: bottom halves

Three mechanisms hand off from hard-IRQ context to something that can
sleep or take longer, in increasing order of "can it block":

| Mechanism | Runs in | Can sleep? | Typical use |
|---|---|---|---|
| **Tasklet** (`tasklet_schedule`) | softirq | No | Fast, non-sleeping follow-up work |
| **Workqueue** (`schedule_work`) | kernel thread | Yes | I2C/SPI follow-up, anything that blocks |
| **Threaded IRQ** (`devm_request_threaded_irq`) | dedicated kthread | Yes | Preferred modern pattern; PREEMPT_RT friendly |

Tasklets are legacy in new drivers — prefer threaded IRQs, which the
kernel schedules as a normal (optionally real-time-priority) thread:

```c
static irqreturn_t mydev_irq_thread(int irq, void *dev_id)
{
	struct mydev_priv *priv = dev_id;
	unsigned long flags;
	u32 status;

	spin_lock_irqsave(&priv->lock, flags);
	status = priv->last_status;
	spin_unlock_irqrestore(&priv->lock, flags);

	/* safe to sleep here: I2C reads, mutex_lock, kmalloc(GFP_KERNEL) */
	mydev_read_extra_regs_over_i2c(priv);

	return IRQ_HANDLED;
}

devm_request_threaded_irq(&pdev->dev, irq, mydev_irq, mydev_irq_thread,
			   IRQF_ONESHOT, dev_name(&pdev->dev), priv);
```

`IRQF_ONESHOT` keeps the hardware IRQ line masked until the threaded
handler finishes — required whenever the primary handler (`mydev_irq`)
returns `IRQ_WAKE_THREAD` implicitly via a NULL primary, or explicitly
signals more work; omitting it on a level-triggered shared line can cause
an interrupt storm because the line re-fires before the thread has acked
the condition that caused it.

## spinlock vs mutex: pick by context, not habit

- **spinlock**: safe in interrupt context; never sleeps; the CPU literally
  spins. Use `spin_lock_irqsave`/`spin_unlock_irqrestore` whenever the
  *same* lock is ever taken from both process context and an interrupt
  handler — the `_irqsave` variant disables local interrupts, which is the
  only thing that prevents the handler from firing on the same CPU while
  you hold the lock and deadlocking against yourself.
- **mutex**: can sleep while waiting; only usable in process context or a
  threaded IRQ handler; never in a hard IRQ handler or with interrupts
  disabled.

```c
/* WRONG: plain spin_lock from process context, interrupt fires on
 * the same CPU while held, handler spins forever waiting for a lock
 * this exact CPU already holds. */
spin_lock(&priv->lock);
...
spin_unlock(&priv->lock);

/* RIGHT: disables interrupts on this CPU for the critical section */
spin_lock_irqsave(&priv->lock, flags);
...
spin_unlock_irqrestore(&priv->lock, flags);
```

The self-deadlock above is one of the single most common embedded kernel
bugs, and it is maddening to debug because it only manifests when the
interrupt happens to fire during that exact critical section — which can
be a one-in-ten-thousand race that passes days of soak testing and then
locks up a fielded unit.

## Data races beyond locking: memory ordering

Even with correct locking, a flag checked without synchronization between
an ISR and a reader thread ("did the interrupt fire yet?") can be reordered
or cached incorrectly by the compiler and CPU. Use `READ_ONCE`/`WRITE_ONCE`
for lock-free single-flag handoffs, and know that on SMP ARM this is not
just a compiler-barrier issue — it's a real memory-ordering issue:

```c
/* ISR */
WRITE_ONCE(priv->irq_fired, true);

/* poller, no lock, just a flag check */
if (READ_ONCE(priv->irq_fired)) {
	WRITE_ONCE(priv->irq_fired, false);
	...
}
```

Anything beyond a single flag (multiple related fields that must be seen
consistently together) needs a real lock — `READ_ONCE`/`WRITE_ONCE` are
not a substitute for locking multi-field state, only for single-word
lock-free handoffs.

## Traps

- **Calling `disable_irq()` from inside that same IRQ's handler** — it
  waits for the handler to finish, and the handler is the caller, so it
  deadlocks. Use `disable_irq_nosync()` from within the handler if you
  truly need to mask on the way out.
- **A shared spinlock between an ISR and a threaded IRQ handler using
  plain `spin_lock`** instead of `_irqsave` — threaded handlers run with
  interrupts enabled on that CPU, so the same self-deadlock risk from
  above applies there too, not just in process context.
- **Assuming single-core behavior on hardware you tested in QEMU with
  `-smp 1`.** A race that's actually impossible on one core (interrupt
  can only preempt, never truly overlap, the section it interrupted) can
  become a real concurrent-access bug the moment the same kernel boots on
  a quad-core SoC.

## Cheat sheet

| Primitive | Sleep-safe | Use from ISR? |
|---|---|---|
| `spin_lock`/`spin_unlock` | No | Only if never shared with an ISR context needing `_irqsave` |
| `spin_lock_irqsave`/`_irqrestore` | No | Yes — required if lock is shared with ISR |
| `mutex_lock`/`mutex_unlock` | Yes | Never |
| `READ_ONCE`/`WRITE_ONCE` | N/A | Yes, for single-word lock-free flags only |
| `devm_request_threaded_irq` | handler thread can sleep | primary handler cannot |
| `IRQF_ONESHOT` | — | Required pattern for level-triggered shared lines with a thread |

!!! note "On verification"
    Locking rules (`_irqsave` requirement, spinlock-vs-mutex context
    rules, threaded-IRQ semantics) were cross-checked against documented
    kernel locking conventions and Level 2/3 material; the race conditions
    described are real, well-documented failure classes, but no code here
    was compiled, booted, or stress-tested on real SMP hardware from this
    machine.

## Exercise

(1) Take Module 2's `mydev` platform driver, add an interrupt line with a
threaded handler using `IRQF_ONESHOT`, and move the (imagined) I2C
follow-up read into the threaded half. (2) Introduce the self-deadlock
bug deliberately — use plain `spin_lock` for a lock shared with the ISR —
and explain, without running it, exactly which two code paths deadlock
and under what timing. (3) One paragraph: your driver currently shares a
`u32 status` field between an ISR and a `read()` syscall using only
`READ_ONCE`/`WRITE_ONCE`. A teammate wants to add a second field that must
be read consistently with the first. Explain why the `READ_ONCE` pattern
no longer suffices and what you'd change.
