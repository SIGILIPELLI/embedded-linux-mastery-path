# 01 · Kernel Driver Development — Char Drivers

Level 2 built and configured kernels; this module writes code that runs
inside one. A character driver is the smallest complete unit of kernel
development: it owns a device number, exposes `open`/`read`/`write`/
`ioctl` to userspace, and lives or dies by rules that don't exist in
userspace — no libc, no page faults on kernel addresses, and a crash here
takes the whole board down. Everything runs against a QEMU `virt` guest so
a bad module costs a reboot, not a bricked board.

## Anatomy of a minimal char driver

Every char driver needs a device number, a `file_operations` table, and a
registration call. The modern API is `cdev` plus the dynamic-major
allocator — hardcoding a major number is a Level-0 mistake that breaks the
moment two drivers want the same number.

```c
// hello_chr.c
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/uaccess.h>
#include <linux/slab.h>

#define DEVICE_NAME "hello_chr"
#define BUF_SIZE    256

static dev_t   dev_num;
static struct cdev hc_cdev;
static struct class *hc_class;
static char    kbuf[BUF_SIZE];
static size_t  kbuf_len;

static int hc_open(struct inode *inode, struct file *filp)
{
	pr_info("hello_chr: open (pid %d)\n", current->pid);
	return 0;
}

static ssize_t hc_read(struct file *filp, char __user *ubuf,
			size_t count, loff_t *off)
{
	if (*off >= kbuf_len)
		return 0;               /* EOF */
	if (count > kbuf_len - *off)
		count = kbuf_len - *off;

	if (copy_to_user(ubuf, kbuf + *off, count))
		return -EFAULT;

	*off += count;
	return count;
}

static ssize_t hc_write(struct file *filp, const char __user *ubuf,
			 size_t count, loff_t *off)
{
	if (count > BUF_SIZE)
		count = BUF_SIZE;

	if (copy_from_user(kbuf, ubuf, count))
		return -EFAULT;

	kbuf_len = count;
	return count;
}

static const struct file_operations hc_fops = {
	.owner = THIS_MODULE,
	.open  = hc_open,
	.read  = hc_read,
	.write = hc_write,
};

static int __init hc_init(void)
{
	int ret;

	ret = alloc_chrdev_region(&dev_num, 0, 1, DEVICE_NAME);
	if (ret)
		return ret;

	cdev_init(&hc_cdev, &hc_fops);
	ret = cdev_add(&hc_cdev, dev_num, 1);
	if (ret)
		goto err_region;

	hc_class = class_create(DEVICE_NAME);
	if (IS_ERR(hc_class)) {
		ret = PTR_ERR(hc_class);
		goto err_cdev;
	}
	device_create(hc_class, NULL, dev_num, NULL, DEVICE_NAME);

	pr_info("hello_chr: registered as %d:%d\n",
		MAJOR(dev_num), MINOR(dev_num));
	return 0;

err_cdev:
	cdev_del(&hc_cdev);
err_region:
	unregister_chrdev_region(dev_num, 1);
	return ret;
}

static void __exit hc_exit(void)
{
	device_destroy(hc_class, dev_num);
	class_destroy(hc_class);
	cdev_del(&hc_cdev);
	unregister_chrdev_region(dev_num, 1);
	pr_info("hello_chr: unloaded\n");
}

module_init(hc_init);
module_exit(hc_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("you");
MODULE_DESCRIPTION("Minimal char driver");
```

Note the unwind order in `hc_exit`: it's the exact reverse of `hc_init`'s
success path. Getting this backwards — destroying the class before the
cdev, say — is one of the most common ways to panic on `rmmod`.

Build and load it in the guest:

```console
$ make -C $KDIR M=$PWD modules
$ scp hello_chr.ko qemu-target:
target$ insmod hello_chr.ko
target$ dmesg | tail -1
[   12.441233] hello_chr: registered as 238:0
target$ ls -l /dev/hello_chr
crw-------  1 root root 238, 0 Jan  1 00:00 /dev/hello_chr
target$ echo "hi" > /dev/hello_chr
target$ cat /dev/hello_chr
hi
```

## `copy_to_user`/`copy_from_user`: the only safe crossing

Userspace pointers are not directly dereferenceable from kernel code —
the address might be unmapped, swapped out, or simply garbage from a
hostile or buggy caller. `copy_to_user`/`copy_from_user` do the page
fault handling and return the number of bytes that **failed** to copy
(0 on full success):

```c
if (copy_from_user(kbuf, ubuf, count))
	return -EFAULT;   /* wrong: ignoring a nonzero return */
```

The pattern above is actually correct because `copy_from_user` returns
0 on success and nonzero (truthy) on any failure — but a very common bug
is writing `ret = copy_from_user(...); return ret;` and returning the
**number of uncopied bytes** as if it were a byte count on success, which
silently corrupts short writes. Always branch on zero/nonzero, then
return a proper negative errno.

Directly dereferencing a user pointer instead of going through these
functions builds fine, runs fine in casual testing, and then oopses in
production the first time it hits an address that isn't resident:

```text
Unable to handle kernel paging request at virtual address 0000007f8a3c1000
Internal error: Oops: 96000021 [#1] PREEMPT SMP
```

## ioctl: the escape hatch, and its trap

`read`/`write` cover streams; `ioctl` covers everything else — get/set a
mode, trigger a calibration, fetch a struct of mixed fields. Define
commands with `_IO`/`_IOR`/`_IOW`/`_IOWR` so direction and size are
encoded in the number itself, which lets the kernel validate transfer
size before your handler even runs:

```c
#define HC_IOC_MAGIC   'h'
#define HC_IOC_RESET   _IO(HC_IOC_MAGIC, 0)
#define HC_IOC_GETLEN  _IOR(HC_IOC_MAGIC, 1, int)
#define HC_IOC_SETLEN  _IOW(HC_IOC_MAGIC, 2, int)

static long hc_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
	int val;

	switch (cmd) {
	case HC_IOC_RESET:
		kbuf_len = 0;
		return 0;
	case HC_IOC_GETLEN:
		val = kbuf_len;
		return copy_to_user((int __user *)arg, &val, sizeof(val)) ?
			-EFAULT : 0;
	case HC_IOC_SETLEN:
		if (copy_from_user(&val, (int __user *)arg, sizeof(val)))
			return -EFAULT;
		if (val < 0 || val > BUF_SIZE)
			return -EINVAL;
		kbuf_len = val;
		return 0;
	default:
		return -ENOTTY;
	}
}
```

**Trap**: never reuse a magic number/command combination across two
drivers that might both be loaded — `ioctl` numbers are not namespaced
beyond the magic byte you pick, and a collision means one driver's tool
silently drives the other driver instead of failing loudly. Check
`Documentation/userspace-api/ioctl/ioctl-number.rst` before inventing one
for anything that might ship.

## printk levels and rate limiting

`pr_info`/`pr_err`/`pr_debug` map to `KERN_INFO`/`KERN_ERR`/dynamic debug
under the hood. In an interrupt-adjacent or high-frequency path, an
unthrottled `pr_*` call can itself become the performance bug — printk
holds a lock and can block on a slow console (serial at 115200 baud is
*slow* compared to kernel time):

```c
static DEFINE_RATELIMIT_STATE(hc_ratelimit, 5 * HZ, 3);

if (__ratelimit(&hc_ratelimit))
	pr_warn("hello_chr: buffer overrun, dropping data\n");
```

A driver that floods the console on every packet is a classic cause of
"the board became unresponsive under load" — the fix is rate limiting,
not removing the diagnostic.

## Traps specific to embedded targets

- **Blocking in `open`/`release` on a slow bus**: an I2C or SPI transaction
  inside `open()` can stall for milliseconds; if that happens under a
  lock also needed by an interrupt handler, you've built a self-deadlock
  that only shows up under real hardware timing, not in QEMU.
- **Forgetting `MODULE_LICENSE("GPL")`**: without it, the module taints
  the kernel and loses access to GPL-only exports (`EXPORT_SYMBOL_GPL`),
  which fails at `insmod` time with an unhelpful "Unknown symbol" rather
  than a licensing message.
- **Leaking the `class`/`cdev` on a failed `probe`-style init**: every
  `err_*:` label must unwind everything allocated before it, or a second
  `insmod` after a failed first attempt leaks a device number permanently
  for that boot.
- **Reading `kbuf_len` without a lock** in a driver that might be opened
  from two processes concurrently — this module doesn't fix that
  (motivated deliberately, so you feel the gap); Module 4 covers the
  right primitive.

## Cheat sheet

| API | Purpose |
|---|---|
| `alloc_chrdev_region` | Get a dynamic major/minor pair |
| `cdev_init` / `cdev_add` | Wire a `file_operations` table to a `dev_t` |
| `class_create` / `device_create` | Make `/dev/<name>` appear via udev |
| `copy_to_user` / `copy_from_user` | Only safe way to cross the user/kernel boundary |
| `_IO`/`_IOR`/`_IOW`/`_IOWR` | Encode ioctl direction + size in the command number |
| `pr_info`/`pr_err`/`__ratelimit` | Kernel logging, with overrun protection |
| `dmesg`, `modinfo`, `lsmod` | Inspect what's loaded and what it printed |

!!! note "On verification"
    The driver above follows the documented cdev/file_operations API and
    was reviewed line-by-line against kernel API conventions established in
    Level 1/2, but it was not compiled or booted on this machine (macOS
    cannot build or run a Linux kernel module). Treat it as review-verified
    source to build and test on your QEMU/target setup, not as executed code.

## Exercise

(1) Extend `hello_chr` with the three ioctls shown above, build it against
your kernel's `KDIR`, load it in QEMU, and drive it from a small userspace
C program using `open()`/`ioctl()`. (2) Deliberately break the unwind order
in `hc_exit` (destroy `hc_class` before `cdev_del`), reload, and capture
the resulting oops or warning in `dmesg` — then fix it and explain in one
paragraph why unwind order matters even when each individual call "looks"
independent. (3) Add rate-limited logging to `hc_write` that fires only
when more than `BUF_SIZE` bytes are requested, and prove with a stress
script that most such requests are dropped from the log instead of
flooding it.
