# 06 · Containers & Virtualization on Embedded

Containers on a server solve "isolate and deploy independently
versioned services on one kernel." On an embedded board, the same
technology gets used for a narrower, harder problem: isolating an update
domain, a fault domain, or a licensing/certification boundary on a device
with a fraction of the RAM, no orchestration control plane, and hard
real-time constraints living right next to whatever you're containerizing.

## Why not just use Docker as-is

Docker's daemon-based architecture (`dockerd` running persistently,
managing a full image store, network bridges, and overlay filesystems)
carries real memory and storage overhead that matters on a board with
512MB-2GB RAM and eMMC, not a datacenter SSD. Embedded Linux almost always
uses a lighter-weight, daemonless runtime instead:

```console
$ podman run --rm -it \
    --device=/dev/ttyUSB0 \
    --memory=64m --cpus=0.5 \
    my-registry/sensor-agent:2.3.1
```

`podman` (daemonless, rootless-capable) or a bare `runc`/`crun` invocation
under a minimal init are both common; the OCI image format and runtime
spec are the same regardless of which higher-level tool wraps them, which
is the actual portability guarantee worth caring about here — not brand
loyalty to a specific tool.

## cgroups: the actual isolation mechanism, and its embedded-specific limits

```console
$ cat /sys/fs/cgroup/system.slice/sensor-agent.service/memory.max
67108864
$ cat /sys/fs/cgroup/system.slice/sensor-agent.service/memory.current
41943040
$ cat /sys/fs/cgroup/system.slice/sensor-agent.service/cpu.max
50000 100000
```

`cpu.max` of `50000 100000` caps this cgroup to 50% of one CPU, measured
over 100ms periods. On an embedded quad-core SoC, a container CPU-limited
this way that's also fighting Module 6 (Level 3)'s PREEMPT_RT motor
control loop for the same physical core is a scheduling interaction that
must be reasoned about explicitly, not assumed away — cgroup CPU quotas
and SCHED_FIFO real-time priorities interact (a throttled cgroup's tasks
still preempt-and-block normally scheduled work up to their quota, but a
real-time task at a higher static priority still wins regardless of
quota). Isolating a container to specific non-RT cores via `cpuset` is
usually the correct, unambiguous fix:

```console
$ cat /sys/fs/cgroup/system.slice/sensor-agent.service/cpuset.cpus
2-3
```

## Device passthrough: containers touching real hardware

A containerized service that needs a GPIO, an I2C bus, or a V4L2 camera
needs explicit device node exposure — container isolation by default
hides `/dev` entirely, which is correct security posture but breaks
naively:

```console
$ podman run --rm \
    --device=/dev/i2c-1 \
    --device=/dev/gpiochip0 \
    --security-opt=label=disable \
    my-registry/imu-reader:1.4.0
```

**Trap**: exposing `/dev/gpiochip0` doesn't grant the *specific line*
permission model some GPIO consumers expect (`gpiod`'s line-request
model checks kernel-level line ownership, not just device-node
permissions) — a container that opens the chip successfully can still
fail to request a specific line if another process (inside or outside any
container) already holds it, and this failure looks identical whether or
not containerization is even involved, which makes it a confusing first
debugging path for anyone new to the setup.

## Multi-container coordination without a datacenter orchestrator

Kubernetes is almost always the wrong tool for a single embedded board —
its control-plane overhead alone can exceed the device's entire RAM
budget. `podman-compose`, `docker-compose`-equivalent tooling, or simply
systemd unit dependencies between container services are the right scale:

```ini
# /etc/systemd/system/sensor-agent.service
[Unit]
Description=Sensor agent container
After=network-online.target
Requires=network-online.target

[Service]
ExecStartPre=-/usr/bin/podman rm -f sensor-agent
ExecStart=/usr/bin/podman run --name sensor-agent \
    --device=/dev/i2c-1 --memory=64m \
    my-registry/sensor-agent:2.3.1
ExecStop=/usr/bin/podman stop -t 10 sensor-agent
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Wiring containers into systemd this way gets you the same restart/
dependency/logging integration (`journalctl -u sensor-agent`) as any
native service, without adopting an orchestration control plane the
device has no business running.

## When you actually need a hypervisor instead

Containers share one kernel; if a workload needs a **different** kernel,
a hard real-time guarantee stronger than cgroups/RT-priority tuning can
provide, or certification isolation (a safety-certified subsystem that
must not share a kernel with general-purpose Linux at all), that's a
job for a type-1 hypervisor, not a container — Jailhouse is the common
lightweight embedded choice:

```console
$ jailhouse enable imx8mp.cell
$ jailhouse cell create rtos-cell.cell
$ jailhouse cell load rtos-cell rtos-image.bin
$ jailhouse cell start rtos-cell
```

Jailhouse statically partitions CPUs, memory, and specific peripherals
between cells at boot — Linux running in the root cell genuinely cannot
touch resources assigned to an RTOS cell, a much stronger isolation
guarantee than cgroups provide, at the cost of static (not dynamically
rebalanceable) resource partitioning decided ahead of time.

## Traps

- **Assuming container memory limits prevent OOM entirely** — a container
  hitting its `memory.max` gets its own processes OOM-killed by the
  kernel's cgroup-aware OOM killer, which is correct isolation, but a
  container that's *supposed* to be always-running (a safety monitor,
  say) needs its own supervision/restart logic on top, not just a memory
  cap.
- **CPU quota fights with real-time priority** — see above; reasoning
  about `cpu.max` alone without also considering `cpuset` and any
  SCHED_FIFO tasks on the same cores produces surprising, hard-to-explain
  latency for whichever workload loses the interaction.
- **Treating containers as a substitute for proper OTA/rollback design**
  — a container image update is still an update; it needs the same
  Module 3 health-check-before-commit discipline, or a bad container
  image is just as capable of degrading the product as a bad rootfs
  image, just scoped smaller.

## Cheat sheet

| Need | Right tool |
|---|---|
| Isolate an update/fault domain, share the kernel | Podman/runc container, daemonless preferred |
| Pin CPU/memory for a workload | cgroups (`cpu.max`, `memory.max`, `cpuset.cpus`) |
| Grant hardware access to a container | `--device=`, understand it's node-level not line-level |
| Coordinate a small fixed set of container services | systemd unit dependencies, not Kubernetes |
| A genuinely different kernel or certified isolation | Type-1 hypervisor (Jailhouse), static partitioning |

!!! note "On verification"
    cgroup v2 interfaces, OCI/podman device-passthrough behavior, and the
    Jailhouse static-partitioning model are checked against documented
    kernel cgroup, OCI runtime, and Jailhouse conventions; no container
    was actually run and no hypervisor cell was created on this machine.

## Exercise

(1) Write the systemd unit for a containerized camera-preprocessing
service that needs `/dev/video0`, is capped at 128MB RAM and one full
CPU, restarts on failure, and logs to the journal — then explain what
`--device=` does and does not grant compared to running the same process
natively. (2) Given a container CPU-limited to 50% via `cpu.max` sharing a
core with a `SCHED_FIFO` priority-90 real-time task, describe what happens
to the container's latency under sustained RT-task load, and what
`cpuset` change would isolate the two properly. (3) One paragraph: explain
to a colleague proposing "let's just run Kubernetes on the board for
consistency with our cloud stack" why that's the wrong tool here, citing
the specific resource constraint that makes it wrong.
