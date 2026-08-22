# 08 · Power Management & Thermal

A battery-powered or fanless product lives or dies by power management
done right. Linux's PM stack spans four largely independent layers —
CPU frequency, CPU/SoC idle states, runtime device power, and thermal
governance — and a bug in any one of them shows up as the same two
symptoms: the board is hotter than it should be, or the battery drains
faster than the spec sheet promises.

## cpufreq: how fast the CPU runs

```console
$ cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_governors
performance powersave ondemand conservative schedutil
$ cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
schedutil
$ cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
800000
$ echo performance > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
```

`schedutil` (the modern default) reads scheduler utilization directly and
scales frequency proportionally — generally the right choice unless you
have a specific reason to pin a governor. `performance` locks max
frequency (useful for repeatable benchmarking, wasteful for a battery
product); `powersave` locks minimum (useful for a thermally-constrained
enclosure with no performance requirement).

Per-core governors matter on asymmetric (big.LITTLE-style) SoCs — setting
`performance` on `cpu0` and leaving `schedutil` on `cpu4` is a legitimate
pattern for pinning one real-time-ish workload to a known-fast core while
letting everything else scale normally.

## CPU idle states (C-states)

```console
$ cat /sys/devices/system/cpu/cpu0/cpuidle/state*/name
WFI
CPU_OFF
$ cat /sys/devices/system/cpu/cpu0/cpuidle/state1/latency
    500
$ cat /sys/devices/system/cpu/cpu0/cpuidle/state1/usage
    128441
```

Deeper idle states (`CPU_OFF`, powering down the core entirely) save more
but cost more to exit — `latency` is in microseconds and is exactly the
number a real-time deadline calculation must account for. A cpuidle
governor picking too deep a state for a workload with frequent short
wakeups (a sensor polled every 2 ms, say) burns more power re-entering
and exiting idle repeatedly than it would have spent just staying awake —
this is a real, measurable failure mode called "C-state thrashing," and
it's invisible unless you're actually watching `usage` counters and power
draw together.

## Runtime PM: devices sleep independently of the CPU

A device (I2C controller, USB PHY, camera sensor) can suspend itself
while the CPU stays fully awake, if nothing is actively using it:

```c
static int mydev_probe(struct platform_device *pdev)
{
	pm_runtime_enable(&pdev->dev);
	...
}

static int mydev_open(struct inode *inode, struct file *filp)
{
	int ret = pm_runtime_get_sync(dev);  /* wakes device, blocks until ready */
	if (ret < 0) {
		pm_runtime_put_noidle(dev);
		return ret;
	}
	return 0;
}

static int mydev_release(struct inode *inode, struct file *filp)
{
	pm_runtime_mark_last_busy(dev);
	pm_runtime_put_autosuspend(dev);  /* suspends after idle timeout */
	return 0;
}
```

```console
$ cat /sys/bus/platform/devices/30a20000.i2c/power/runtime_status
active
$ cat /sys/bus/platform/devices/30a20000.i2c/power/runtime_suspended_time
842103
```

**Trap**: every `pm_runtime_get_sync` (or its modern non-deprecated
sibling `pm_runtime_resume_and_get`) must be matched by exactly one
`pm_runtime_put*` — an unbalanced get leaves the device's usage count
above zero forever, and it never autosuspends again for the rest of
uptime. This bug is invisible functionally (the device still works, it
just never sleeps) and shows up only as "why is idle power 40 mW higher
than the reference design," which is exactly the kind of regression that
survives code review and ships.

## System suspend: `suspend`/`resume`

```c
static int mydev_suspend(struct device *dev)
{
	struct mydev_priv *priv = dev_get_drvdata(dev);

	disable_irq(priv->irq);
	writel(0, priv->regs + REG_CTRL);   /* power the block down */
	return 0;
}

static int mydev_resume(struct device *dev)
{
	struct mydev_priv *priv = dev_get_drvdata(dev);

	writel(CTRL_ENABLE, priv->regs + REG_CTRL);  /* restore state */
	enable_irq(priv->irq);
	return 0;
}

static DEFINE_SIMPLE_DEV_PM_OPS(mydev_pm_ops, mydev_suspend, mydev_resume);
```

```console
$ echo mem > /sys/power/state
[   88.201113] PM: suspend entry (deep)
[   88.220441] Disabling non-boot CPUs ...
...
[   88.998821] PM: suspend exit
```

**Trap**: any hardware register state written in `probe` but not
re-written in `resume` is lost across suspend-to-RAM on most SoCs — the
block is genuinely powered off, not just clock-gated. A driver that
"forgets" one register write in `resume` works perfectly through every
test that never suspends, then silently misbehaves on every device after
its first sleep/wake cycle in the field — this is one of the highest-yield
things to specifically test before shipping any PM support at all.

## Thermal governance

```console
$ cat /sys/class/thermal/thermal_zone0/temp
68500
$ cat /sys/class/thermal/thermal_zone0/trip_point_0_temp
85000
$ cat /sys/class/thermal/thermal_zone0/trip_point_0_type
passive
$ cat /sys/class/thermal/cooling_device0/type
thermal-cpufreq-0
```

Passive trips throttle (usually via cpufreq) as temperature approaches
the limit; a `critical` trip triggers an actual shutdown to protect
silicon. Device tree wires zones to trips to cooling devices:

```dts
thermal-zones {
	cpu-thermal {
		polling-delay-passive = <250>;
		polling-delay = <1000>;
		thermal-sensors = <&tmu 0>;

		trips {
			cpu_alert: cpu-alert {
				temperature = <85000>;
				hysteresis = <2000>;
				type = "passive";
			};
			cpu_crit: cpu-crit {
				temperature = <95000>;
				hysteresis = <2000>;
				type = "critical";
			};
		};

		cooling-maps {
			map0 {
				trip = <&cpu_alert>;
				cooling-device = <&cpu0 THERMAL_NO_LIMIT THERMAL_NO_LIMIT>;
			};
		};
	};
};
```

**Trap**: a fanless enclosure design validated only at room temperature
often discovers its actual thermal trip behavior for the first time in a
40°C ambient field test — throttling that never triggered on a lab bench
can cut sustained CPU frequency by more than half in a hot enclosure,
turning a "fast enough" product into a visibly sluggish one purely from
enclosure thermal design, with zero software change.

## Traps

- **Unbalanced `pm_runtime_get`/`put`** — see above; the single most
  common silent idle-power regression.
- **Missing register restore in `resume`** — the single most common
  "worked in the lab, broke after the first field suspend" bug.
- **Trip points set from a datasheet's absolute maximum instead of the
  enclosure's actual sustained thermal ceiling** — a trip at the silicon's
  survivable limit still lets sustained throttling degrade UX badly before
  ever reaching the trip; tune trips from real enclosure thermal testing,
  not just the SoC datasheet.

## Cheat sheet

| Interface | Purpose |
|---|---|
| `scaling_governor` | Pick CPU frequency policy (`schedutil` default) |
| `cpuidle/stateN/{latency,usage}` | Idle-state cost and how often it's used |
| `pm_runtime_get_sync`/`put_autosuspend` | Per-device sleep independent of CPU |
| `SIMPLE_DEV_PM_OPS` | Wire `suspend`/`resume` callbacks into a driver |
| `echo mem > /sys/power/state` | Trigger suspend-to-RAM |
| `/sys/class/thermal/thermal_zoneN/temp` | Current temperature reading |
| `trip_point_N_type` (`passive`/`critical`) | Throttle vs. shutdown behavior |

!!! note "On verification"
    cpufreq/cpuidle/runtime-PM/thermal sysfs interfaces and the DT thermal
    binding were checked against the documented kernel PM and thermal
    frameworks; the suspend/resume driver code follows the documented
    `dev_pm_ops` contract but was not compiled or exercised through an
    actual suspend/resume cycle on real hardware from this machine.

## Exercise

(1) Add `suspend`/`resume` callbacks to Module 2's `mydev` platform driver
that correctly save and restore the one register it writes in `probe`,
and explain what observable symptom you'd expect if `resume` were
accidentally deleted. (2) Given `cpuidle` `usage` counters showing state1
entered 50,000 times in one minute with a 500 μs exit latency each,
estimate the idle-transition overhead and explain whether this workload
is a candidate for C-state thrashing. (3) One paragraph: your fanless
product passes every functional test at room temperature but a customer
reports it "feels slow" after 20 minutes of continuous use in a warm
room. Describe the exact thermal-zone/trip data you'd pull first and what
each possible reading would tell you.
