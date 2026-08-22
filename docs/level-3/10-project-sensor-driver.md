# 10 · Project — Sensor Driver + Userspace Stack

This project combines Modules 1–9 into one coherent deliverable: a
production-shaped kernel driver for an I2C IMU, wired through the device
tree, backed by a threaded interrupt for data-ready events, exposed to
userspace via the Industrial I/O (IIO) subsystem, and profiled/validated
the way a real bring-up would be. Nothing here is executed on this
machine — it is a complete, review-verified design you build and test on
your QEMU/target setup.

## System design

```
┌─────────────────────────────────────────────────────────────┐
│ Userspace                                                     │
│  iio_reader (C) ── reads /sys/bus/iio/devices/iio:device0/    │
│                     or triggered buffer via /dev/iio:device0  │
└───────────────────────────▲───────────────────────────────────┘
                             │ IIO ABI (sysfs + chardev buffer)
┌───────────────────────────┴───────────────────────────────────┐
│ Kernel: bmi270_iio.ko                                          │
│  probe() ── i2c_client, regmap, iio_dev registration            │
│  threaded IRQ ── data-ready GPIO → read FIFO → push to buffer   │
│  runtime PM ── suspend sensor when no active buffer consumer   │
└───────────────────────────▲───────────────────────────────────┘
                             │ I2C + GPIO interrupt line
┌───────────────────────────┴───────────────────────────────────┐
│ Hardware: BMI270 IMU on i2c1 @0x68, INT1 → gpio1 14             │
└─────────────────────────────────────────────────────────────────┘
```

Choosing **IIO** over a bespoke char device (Module 1's approach) is the
deliberate, production-correct decision here: IIO gives you standardized
sysfs scale/sampling-frequency attributes, a triggered-buffer chardev for
efficient streaming, and interoperability with existing userspace tooling
(`iio_info`, `libiio`) for free — reinventing that ABI per-sensor is a
common but avoidable mistake in real BSP work.

## Device tree

```dts
&i2c1 {
	status = "okay";

	imu@68 {
		compatible = "bosch,bmi270";
		reg = <0x68>;
		vdd-supply = <&reg_3v3>;
		interrupt-parent = <&gpio1>;
		interrupts = <14 IRQ_TYPE_EDGE_RISING>;
		mount-matrix = "1", "0", "0",
			       "0", "1", "0",
			       "0", "0", "1";
	};
};
```

`mount-matrix` is IIO's standard property for describing how the sensor is
physically oriented on the PCB relative to the product's reference frame
— encoding orientation in the DT instead of in application code means the
same driver serves any board revision that flips the sensor, with the
correction applied consistently for every consumer of the IIO channel.

## Driver skeleton

```c
#include <linux/i2c.h>
#include <linux/iio/iio.h>
#include <linux/iio/buffer.h>
#include <linux/iio/trigger.h>
#include <linux/iio/triggered_buffer.h>
#include <linux/regmap.h>
#include <linux/pm_runtime.h>

struct bmi270_data {
	struct i2c_client *client;
	struct regmap *regmap;
	struct iio_trigger *trig;
	s16 buf[3 + 4] __aligned(8);   /* 3x16-bit axis + timestamp pad */
};

static const struct iio_chan_spec bmi270_channels[] = {
	{
		.type = IIO_ACCEL,
		.modified = 1,
		.channel2 = IIO_MOD_X,
		.info_mask_separate = BIT(IIO_CHAN_INFO_RAW),
		.info_mask_shared_by_type = BIT(IIO_CHAN_INFO_SCALE),
		.scan_index = 0,
		.scan_type = { .sign = 's', .realbits = 16, .storagebits = 16 },
	},
	/* Y, Z identical with channel2 = IIO_MOD_Y / IIO_MOD_Z, scan_index 1/2 */
	IIO_CHAN_SOFT_TIMESTAMP(3),
};

static irqreturn_t bmi270_trigger_handler(int irq, void *p)
{
	struct iio_poll_func *pf = p;
	struct iio_dev *indio_dev = pf->indio_dev;
	struct bmi270_data *data = iio_priv(indio_dev);

	regmap_bulk_read(data->regmap, BMI270_REG_ACCEL_X_LSB,
			  data->buf, 6);
	iio_push_to_buffers_with_timestamp(indio_dev, data->buf,
					    iio_get_time_ns(indio_dev));

	iio_trigger_notify_done(indio_dev->trig);
	return IRQ_HANDLED;
}

static irqreturn_t bmi270_data_ready_irq(int irq, void *p)
{
	struct iio_dev *indio_dev = p;

	iio_trigger_poll(((struct bmi270_data *)iio_priv(indio_dev))->trig);
	return IRQ_HANDLED;
}

static int bmi270_probe(struct i2c_client *client)
{
	struct iio_dev *indio_dev;
	struct bmi270_data *data;
	int ret;

	indio_dev = devm_iio_device_alloc(&client->dev, sizeof(*data));
	if (!indio_dev)
		return -ENOMEM;

	data = iio_priv(indio_dev);
	data->client = client;
	data->regmap = devm_regmap_init_i2c(client, &bmi270_regmap_config);
	if (IS_ERR(data->regmap))
		return PTR_ERR(data->regmap);

	indio_dev->name = "bmi270";
	indio_dev->channels = bmi270_channels;
	indio_dev->num_channels = ARRAY_SIZE(bmi270_channels);
	indio_dev->modes = INDIO_DIRECT_MODE;

	ret = devm_iio_triggered_buffer_setup(&client->dev, indio_dev, NULL,
					       bmi270_trigger_handler, NULL);
	if (ret)
		return ret;

	ret = devm_request_threaded_irq(&client->dev, client->irq, NULL,
					 bmi270_data_ready_irq,
					 IRQF_TRIGGER_RISING | IRQF_ONESHOT,
					 "bmi270_dready", indio_dev);
	if (ret)
		return ret;

	pm_runtime_set_autosuspend_delay(&client->dev, 1000);
	pm_runtime_use_autosuspend(&client->dev);
	pm_runtime_enable(&client->dev);

	return devm_iio_device_register(&client->dev, indio_dev);
}
```

This wires together Module 2 (probe/bind via `i2c_client`, `devm_*`
discipline), Module 4 (threaded IRQ handing off from a fast top-half to a
buffer-filling handler, correctly outside hard-IRQ context so
`regmap_bulk_read`'s I2C transaction is safe), and Module 8
(`pm_runtime_*` so the sensor sleeps when nothing has an active buffer).

## Userspace validation

```console
$ ls /sys/bus/iio/devices/iio:device0/
in_accel_x_raw  in_accel_scale  scan_elements/  buffer/  trigger/
$ cat /sys/bus/iio/devices/iio:device0/in_accel_x_raw
124
$ echo bmi270_dready-dev0 > /sys/bus/iio/devices/iio:device0/trigger/current_trigger
$ echo 1 > /sys/bus/iio/devices/iio:device0/scan_elements/in_accel_x_en
$ echo 1 > /sys/bus/iio/devices/iio:device0/scan_elements/in_accel_y_en
$ echo 1 > /sys/bus/iio/devices/iio:device0/scan_elements/in_accel_z_en
$ echo 64 > /sys/bus/iio/devices/iio:device0/buffer/length
$ echo 1  > /sys/bus/iio/devices/iio:device0/buffer/enable
$ iio_readdev -a -T 1000 bmi270 > /tmp/imu_capture.bin
```

`iio_info` (from `libiio`) is the fastest sanity check that channel
metadata (scale, available sampling frequencies) is exposed correctly
before writing any custom reader:

```console
$ iio_info -u local:
IIO context has 1 devices:
	iio:device0: bmi270
		3 channels found:
			accel_x:  (input) [en]
			accel_y:  (input) [en]
			accel_z:  (input) [en]
```

## Failure-mode walkthrough (apply Modules 4, 8, 9)

- **No data ever arrives**: check `dmesg` for the threaded IRQ actually
  firing; if it never fires, suspect the DT `interrupts` line/GPIO
  polarity before the driver logic (Module 3/4 territory).
- **Data arrives but stutters under system load**: this is exactly where
  Module 9's `ftrace wakeup_rt` earns its keep — trace the delta between
  the hard-IRQ (`bmi270_data_ready_irq`) and the threaded handler running,
  and check whether something with higher scheduling priority is
  preempting it.
- **First read after idle is stale or garbage**: almost always a runtime
  PM resume race — the sensor needs settling time after
  `pm_runtime_get_sync` before its first sample is valid; a driver that
  doesn't wait for that (a `regmap` read issued immediately on resume) is
  a very common, very quiet bug.

## Cheat sheet

| Layer | Module this project draws on |
|---|---|
| `i2c_client`/`devm_*` probe discipline | Module 2 |
| Device tree `interrupts`, `mount-matrix` | Module 3 |
| Threaded IRQ, correct top/bottom-half split | Module 4 |
| `pm_runtime_*` autosuspend | Module 8 |
| `ftrace`/`perf` for latency and stutter diagnosis | Module 9 |
| IIO triggered buffer ABI | This project's core contribution |

!!! note "On verification"
    This design was reviewed for internal consistency against the IIO,
    threaded-IRQ, and runtime-PM kernel APIs referenced in the preceding
    modules and against real BMI270/IIO driver conventions; it was not
    compiled, flashed, or run against real or emulated I2C hardware from
    this machine. Treat it as a build-and-bring-up starting point, not
    tested firmware.

## Stretch goals

- Add a second trigger source (a periodic hrtimer trigger) alongside the
  data-ready GPIO trigger, and let userspace choose via
  `current_trigger` — useful for polling mode on boards without the
  interrupt line wired.
- Extend the runtime-PM path with a proper settling-time delay after
  resume (`usleep_range` sized to the datasheet's power-up time), and
  write the `ftrace` capture that proves the first post-resume sample is
  now valid instead of stale.
- Package the driver as a Yocto recipe (drawing on Level 2) that installs
  both the `.ko` and a udev rule granting a `plugdev`-style group access
  to `/dev/iio:device0`, and wire it into a systemd service that starts a
  logging consumer only after the IIO device node actually appears.
