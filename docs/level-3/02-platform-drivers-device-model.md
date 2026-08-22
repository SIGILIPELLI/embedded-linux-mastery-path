# 02 · Platform Drivers & the Device Model

Module 1's driver only existed because you typed `insmod`. Real hardware
on a real board is described declaratively — in a device tree — and the
kernel's driver core matches that description to a driver automatically
at boot, with no manual step. That match-then-bind mechanism is the
**device model**, and `platform_device`/`platform_driver` is how the vast
majority of SoC-integrated peripherals (UARTs, I2C controllers, GPIO
blocks, custom ASIC blocks on a memory-mapped bus) plug into it.

## Why "platform" devices exist

PCI and USB devices announce themselves — enumerate the bus, read a
vendor/device ID, done. Memory-mapped SoC peripherals don't enumerate;
nothing on an AHB/APB bus says "I am an i.MX8 UART at 0x30860000." The
**platform bus** is the kernel's answer: a virtual bus for devices that
are described out-of-band, today almost always by the device tree, and
matched to a driver by a compatible string instead of a PCI ID.

## The driver side: probe/remove

```c
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/of.h>
#include <linux/io.h>

struct mydev_priv {
	void __iomem *regs;
	struct device *dev;
};

#define REG_CTRL   0x00
#define REG_STATUS 0x04
#define CTRL_ENABLE BIT(0)

static int mydev_probe(struct platform_device *pdev)
{
	struct mydev_priv *priv;
	struct resource *res;

	priv = devm_kzalloc(&pdev->dev, sizeof(*priv), GFP_KERNEL);
	if (!priv)
		return -ENOMEM;

	priv->regs = devm_platform_get_and_ioremap_resource(pdev, 0, &res);
	if (IS_ERR(priv->regs))
		return PTR_ERR(priv->regs);

	priv->dev = &pdev->dev;
	platform_set_drvdata(pdev, priv);

	writel(CTRL_ENABLE, priv->regs + REG_CTRL);

	dev_info(&pdev->dev, "mydev bound at %pR\n", res);
	return 0;
}

static void mydev_remove(struct platform_device *pdev)
{
	struct mydev_priv *priv = platform_get_drvdata(pdev);

	writel(0, priv->regs + REG_CTRL);
	dev_info(&pdev->dev, "mydev removed\n");
}

static const struct of_device_id mydev_of_match[] = {
	{ .compatible = "acme,mydev-v1" },
	{ /* sentinel */ }
};
MODULE_DEVICE_TABLE(of, mydev_of_match);

static struct platform_driver mydev_driver = {
	.probe  = mydev_probe,
	.remove = mydev_remove,
	.driver = {
		.name           = "mydev",
		.of_match_table = mydev_of_match,
	},
};
module_platform_driver(mydev_driver);

MODULE_LICENSE("GPL");
```

Everything allocated with a `devm_*` function is tied to the device's
lifetime and freed automatically on `remove` or a failed `probe` — this
is why `mydev_remove` doesn't `kfree(priv)` or `iounmap`. Mixing manual
and `devm_*` allocation in the same driver is a common source of
double-frees; pick one discipline per resource and be consistent.

`MODULE_DEVICE_TABLE(of, mydev_of_match)` matters even for a built-in
driver: it's what lets `depmod`/module autoloading find this driver for
a given compatible string at boot, without it a modular build silently
never autoloads.

## The description side: a matching DT node

```dts
soc {
	mydev0: mydev@30860000 {
		compatible = "acme,mydev-v1";
		reg = <0x30860000 0x1000>;
		clocks = <&clk IMX8MP_CLK_MYDEV>;
		status = "okay";
	};
};
```

Bind and confirm from userspace:

```console
$ dmesg | grep mydev
[    1.204112] mydev 30860000.mydev: mydev bound at [mem 0x30860000-0x30860fff]
$ ls /sys/bus/platform/drivers/mydev/
30860000.mydev  bind  uevent  unbind
```

The `bind`/`unbind` files under `/sys/bus/platform/drivers/mydev/` are a
genuinely useful debugging tool — writing a device's name to `unbind`
then back to `bind` re-runs `remove`/`probe` without a reboot, which is
much faster than power-cycling a board while iterating on probe logic:

```console
$ echo 30860000.mydev > /sys/bus/platform/drivers/mydev/unbind
$ echo 30860000.mydev > /sys/bus/platform/drivers/mydev/bind
```

## Deferred probe: the ordering problem you will hit

`probe` frequently needs a resource — a clock, a regulator, a GPIO — owned
by a driver that hasn't loaded yet. The kernel doesn't guarantee driver
init order across subsystems, so the correct response to "resource not
ready yet" is not an error, it's `-EPROBE_DEFER`:

```c
priv->clk = devm_clk_get(&pdev->dev, "mydev_clk");
if (IS_ERR(priv->clk))
	return PTR_ERR(priv->clk);   /* devm_clk_get already returns
					-EPROBE_DEFER when appropriate */
```

You can watch deferrals happen:

```console
$ cat /sys/kernel/debug/devices_deferred
30860000.mydev
$ dmesg | grep -i defer
[    0.812223] platform 30860000.mydev: deferred probe pending
```

**Trap**: swallowing `-EPROBE_DEFER` and converting it to a hard failure
(logging an error and returning `-ENODEV` instead of propagating the
defer) is one of the most common platform-driver bugs. It usually works
fine on your bench because your dependency happens to probe first, then
fails intermittently on a different boot order, a different board
revision, or after an unrelated driver is added upstream of yours in the
Makefile link order.

## sysfs attributes: the driver's userspace API

Beyond `/dev` nodes, drivers commonly expose control and status through
sysfs attributes:

```c
static ssize_t enable_show(struct device *dev, struct device_attribute *attr,
			    char *buf)
{
	struct mydev_priv *priv = dev_get_drvdata(dev);
	u32 ctrl = readl(priv->regs + REG_CTRL);

	return sysfs_emit(buf, "%d\n", !!(ctrl & CTRL_ENABLE));
}

static ssize_t enable_store(struct device *dev, struct device_attribute *attr,
			     const char *buf, size_t count)
{
	struct mydev_priv *priv = dev_get_drvdata(dev);
	bool en;
	int ret = kstrtobool(buf, &en);

	if (ret)
		return ret;

	writel(en ? CTRL_ENABLE : 0, priv->regs + REG_CTRL);
	return count;
}
static DEVICE_ATTR_RW(enable);

static struct attribute *mydev_attrs[] = {
	&dev_attr_enable.attr,
	NULL,
};
ATTRIBUTE_GROUPS(mydev);
```

Attach `.dev_groups = mydev_groups` to the `platform_driver` struct and
`/sys/.../mydev0/enable` appears automatically, readable and writable —
the same pattern GPIO, LED, and hwmon subsystems all build on internally.

## Traps

- **Reading `pdev->dev.of_node` fields without checking they exist** —
  `of_property_read_u32` returning nonzero means the property is absent,
  not zero; treating the error return as "value is 0" silently
  misconfigures hardware.
- **`ioremap` without a matching `iounmap`**, or vice versa in the wrong
  order relative to `devm_*` cleanup — mixing manual `ioremap` with
  `devm_platform_get_and_ioremap_resource` for the same region double-maps
  it, and the second driver instance (after an unbind/bind cycle) gets a
  different virtual address than the first, breaking anything that cached
  the old pointer.
- **Assuming probe order matches DT order.** The device tree's node order
  is not a promise about probe order; anything with a real dependency must
  express it via `-EPROBE_DEFER`, phandle-based lookups (`clocks`,
  `resets`, `power-domains`), or an explicit `depends-on`-style property —
  never by relying on link order or DT text order.

## Cheat sheet

| Concept | Where it lives |
|---|---|
| `platform_driver` + `probe`/`remove` | Driver-side match/bind logic |
| `of_device_id` table + `compatible` | What links a DT node to this driver |
| `devm_platform_get_and_ioremap_resource` | Map `reg` safely, freed automatically |
| `-EPROBE_DEFER` | "Not ready yet, try me again later" |
| `/sys/kernel/debug/devices_deferred` | See what's stuck waiting on a dependency |
| `/sys/bus/platform/drivers/<name>/{bind,unbind}` | Force a re-probe without rebooting |
| `DEVICE_ATTR_RW` + `ATTRIBUTE_GROUPS` | Expose control/status via sysfs |

!!! note "On verification"
    Probe/remove flow, the `of_device_id` matching mechanism, and
    `-EPROBE_DEFER` semantics were checked against the documented driver
    core API and Level 2's device-tree material; no kernel was built or
    booted to execute this code on this machine.

## Exercise

(1) Write the DT node and platform driver for a fictional "watchdog pet"
register block: one `reg` window, one `compatible` string, a sysfs
`period_ms` attribute (read/write) backed by a register write. (2) Break
it on purpose — reference a clock via `devm_clk_get` that doesn't exist in
your DT — and show the `-EPROBE_DEFER` (or hard failure, if you mistyped
the clock name) in `dmesg` and `devices_deferred`. (3) One paragraph:
explain why unbind/bind via sysfs is safe for iterating on probe logic but
dangerous to script into an automated recovery path on a shipped product.
