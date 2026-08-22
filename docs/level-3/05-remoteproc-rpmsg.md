# 05 · Heterogeneous Compute — remoteproc & RPMsg

Most i.MX8/i.MX9 SoCs ship a Cortex-M core alongside the Cortex-A cluster
Linux runs on — for real-time I/O, sensor pre-processing, or work that
must keep running through a Linux crash or reboot. Linux doesn't run on
that core; it **loads, boots, and talks to** it. `remoteproc` is the
subsystem that manages the M-core's lifecycle; `rpmsg` is the messaging
protocol that runs over a shared-memory mailbox once it's up.

## The mental model

```
 Cortex-A (Linux)                    Cortex-M (bare-metal / RTOS)
┌─────────────────────┐             ┌──────────────────────────┐
│ remoteproc driver    │──loads────▶│  firmware.elf in TCM/DDR  │
│  /sys/class/remoteproc│──starts──▶│  (your M-core firmware)   │
│                      │            │                            │
│ rpmsg_char / custom  │◀──mailbox─▶│  RPMsg endpoint            │
│  driver on /dev/     │  (shared   │  (virtio ring buffers in  │
│                      │   memory)  │   a fixed DDR region)      │
└─────────────────────┘             └──────────────────────────┘
```

Linux is the *master*: it loads an ELF or plain binary into a
memory region the M-core can execute from, releases the M-core out of
reset, and from then on the two cores communicate purely through shared
memory and a hardware mailbox interrupt — no bus arbitration, no shared
OS state.

## remoteproc: loading and controlling the M-core

```console
$ ls /sys/class/remoteproc/remoteproc0/
firmware  state  coredump  name  ...
$ cat /sys/class/remoteproc/remoteproc0/state
offline
$ cp my_m_core_fw.elf /lib/firmware/
$ echo my_m_core_fw.elf > /sys/class/remoteproc/remoteproc0/firmware
$ echo start > /sys/class/remoteproc/remoteproc0/state
$ cat /sys/class/remoteproc/remoteproc0/state
running
$ dmesg | tail -3
[   14.881220] remoteproc remoteproc0: powering up imx-rproc
[   14.902341] remoteproc remoteproc0: Booting fw image my_m_core_fw.elf
[   14.905671] remoteproc remoteproc0: remote processor imx-rproc is now up
```

Firmware must be placed where `/lib/firmware` (or the path configured via
`firmware_class.path`) can find it — a common first-run failure is a
missing file, which surfaces as a generic "request_firmware failed" with
`-ENOENT`, not a helpful "file not found at /lib/firmware/X".

Stopping is the mirror image, and matters for coredump-then-restart
recovery flows:

```console
$ echo stop > /sys/class/remoteproc/remoteproc0/state
$ echo default > /sys/class/remoteproc/remoteproc0/coredump   # capture on crash
```

## Resource table: the contract between the two firmwares

The M-core firmware embeds a **resource table** — a struct compiled into
its ELF that tells the Linux-side remoteproc driver what memory regions,
vdevs (virtio devices, i.e. rpmsg channels), and trace buffers to set up
before starting the core:

```c
/* M-core side, roughly (NXP MCUXpresso SDK style) */
struct rsc_table {
	struct resource_table hdr;
	uint32_t offset[2];
	struct fw_rsc_carveout   vdev_mem;
	struct fw_rsc_vdev       vdev;
	struct fw_rsc_vdev_vring vring0;
	struct fw_rsc_vdev_vring vring1;
} __attribute__((packed, aligned(0x10)));
```

If this struct's memory layout doesn't match what the Linux `imx_rproc`
driver expects for that SoC's remoteproc binding, the M-core firmware
loads and even appears to start, but `rpmsg` channel creation silently
never happens — the failure surfaces as "no `/dev/rpmsg*` node ever
appears," not as a load error, because remoteproc genuinely did its job
correctly with the (wrong) table it was given.

## RPMsg: message passing once both sides are up

Once the resource table advertises an rpmsg vdev, Linux creates a
`/dev/rpmsg_ctrl*` control device and, per named endpoint, a
`/dev/rpmsg<N>` char device:

```console
$ ls /dev/rpmsg*
/dev/rpmsg_ctrl0  /dev/rpmsg0
$ dmesg | grep rpmsg
[   15.112004] virtio_rpmsg_bus virtio0: rpmsg host is online
[   15.118829] rpmsg_char rpmsg0: new channel: 0x400 -> 0x0!
```

Userspace talks over the char device like any stream:

```c
int fd = open("/dev/rpmsg0", O_RDWR);
write(fd, "PING", 4);

char buf[32];
int n = read(fd, buf, sizeof(buf));   /* blocks until M-core replies */
```

On the M-core side (bare-metal RPMsg-Lite or Zephyr), the mirror call
receives the buffer, does its work — sensor fusion, PWM timing, whatever
justified a second core — and replies over the same endpoint. The
round-trip latency is dominated by the mailbox IRQ and scheduling, not by
any bus transaction, typically tens of microseconds.

## Crash and restart: designing for the M-core dying

The M-core runs firmware you wrote, possibly without an MMU or memory
protection; a stray write can hang it. remoteproc surfaces this as a
detected crash if the firmware supports a watchdog/keepalive convention,
and the recovery is a **reload**, not a Linux reboot:

```console
$ dmesg | grep -i crash
[  204.301112] remoteproc remoteproc0: crash detected in imx-rproc: type watchdog
[  204.301998] remoteproc remoteproc0: handling crash #1 in imx-rproc
$ echo stop  > /sys/class/remoteproc/remoteproc0/state
$ echo start > /sys/class/remoteproc/remoteproc0/state
```

Any driver holding an open `/dev/rpmsg*` fd across a crash gets that fd
invalidated — the channel is torn down and recreated on restart with a
**new** device node in general, so a production recovery path must watch
for the device disappearing (via udev/netlink) and re-open it, not assume
the same fd stays valid.

## Traps

- **Racing the M-core's boot against Linux's own driver probe order.**
  If a Linux platform driver depends on data the M-core is expected to
  produce over rpmsg, and remoteproc autostarts at boot via a `status`
  property, there's no built-in guarantee the M-core has finished its own
  init before Linux's rpmsg channel driver probes — design an explicit
  handshake message, don't assume ordering.
- **Shared-memory carveout overlap.** The DDR region used for the
  resource table's vdev/vring memory must be reserved (via a DT
  `reserved-memory` node) so the Linux kernel's own allocator never hands
  that physical range to something else — an unreserved carveout is a
  silent memory-corruption bug that looks like random kernel or M-core
  instability with no clear trigger.
- **Assuming firmware reload resets M-core peripherals.** `remoteproc`
  controls the M-core's execution and memory, not necessarily every
  peripheral it was driving (a shared I2C bus, a GPIO left high) — a
  reload after a crash can leave hardware in the state the crashed
  firmware left it, which the fresh firmware image must handle on its own
  init path, not assume a clean slate.

## Cheat sheet

| Path/Command | Purpose |
|---|---|
| `/sys/class/remoteproc/remoteprocN/firmware` | Which file to load next |
| `/sys/class/remoteproc/remoteprocN/state` | `start` / `stop`, read current state |
| `/sys/class/remoteproc/remoteprocN/coredump` | Capture M-core memory on crash |
| `/dev/rpmsg_ctrl*` | Control device for creating rpmsg endpoints |
| `/dev/rpmsgN` | Per-channel data stream |
| Resource table (`fw_rsc_*`) | M-core-side contract: carveouts, vdevs, vrings |
| DT `reserved-memory` | Must cover every carveout the resource table declares |

!!! note "On verification"
    The remoteproc sysfs ABI and rpmsg char-device model are checked
    against the documented kernel remoteproc/rpmsg framework; the resource
    table struct layout is representative of NXP's SDK convention, not
    copied from a specific SDK version, and none of this was loaded onto
    real M-core/A-core silicon from this machine — treat it as a
    conceptual and structural reference, not tested firmware.

## Exercise

(1) Starting from the `remoteproc0` sysfs sequence above, write a short
shell script that loads a firmware image, waits for `state` to read
`running`, and fails loudly with the `dmesg` tail if it doesn't within 2
seconds. (2) Design (on paper) the `reserved-memory` DT node your
resource table's vdev carveout would need, sized for two 4 KB vrings plus
a 64 KB shared buffer, and explain what happens if you undersize it. (3)
One paragraph: describe the handshake message you'd add so Linux never
opens `/dev/rpmsg0` before the M-core firmware has finished its own
sensor-init sequence, given that remoteproc autostart and rpmsg channel
creation happen before you get any hook into "is the app on the other end
actually ready."
