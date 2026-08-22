# 10 · Capstone — Production-Grade i.MX95 Product

This capstone combines every Level 4 module into one coherent product
design: a fleet-deployed i.MX95 vision-and-control appliance with signed
boot, A/B OTA, on-device ML inference, a containerized application layer,
and the maintenance/compliance processes to keep it running for years.
Nothing here was built or flashed on this machine — this is a complete,
review-verified system design meant as your own build's starting
architecture.

## Product shape

An i.MX95-based edge appliance: a camera-fed object-detection pipeline
(Module 5's NPU/eIQ stack) driving a real-world actuator over GPIO/I2C,
deployed across a fleet of several thousand units, updated over the air,
with a signed boot chain protecting the update mechanism itself.

## System architecture

```
┌──────────────────────────────────────────────────────────────────┐
│ Boot chain (Module 2)                                             │
│  Boot ROM → AHAB-verified SPL/ATF → U-Boot → signed kernel+initramfs│
│  dm-verity rootfs, root hash embedded in signed kernel cmdline     │
└───────────────────────────┬────────────────────────────────────────┘
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ A/B slots (Module 3, RAUC)                                        │
│  Slot A (active) ←──────── OTA update ────────→ Slot B (staged)   │
│  Boot counter + health-check gate before mark-good                │
└───────────────────────────┬────────────────────────────────────────┘
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ Application layer (Module 6, containers)                          │
│  ┌────────────────────┐   ┌─────────────────────┐                │
│  │ vision-pipeline     │   │ actuator-control     │                │
│  │ container            │   │ container (RT-      │                │
│  │ (NPU delegate,       │──▶│  priority host       │                │
│  │  camera, Module 5)   │   │  process, not        │                │
│  │                       │   │  containerized —     │                │
│  │                       │   │  see design note)    │                │
│  └────────────────────┘   └─────────────────────┘                │
└───────────────────────────┬────────────────────────────────────────┘
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ Identity, telemetry, fleet ops (Module 7)                         │
│  On-device key (OP-TEE/secure element) → fleet enrollment          │
│  Staged rollout, health telemetry, remote diagnostics               │
└──────────────────────────────────────────────────────────────────┘
```

**Deliberate design decision**: the actuator-control loop runs as a
native, `SCHED_FIFO`-prioritized host process, *not* inside a container.
This directly applies Module 6's warning about cgroup CPU quotas
interacting unpredictably with real-time scheduling — for a control loop
whose latency directly affects a physical actuator, removing an extra
layer of scheduling interaction is worth the isolation the container
would have provided. The vision pipeline, which is throughput-oriented
and tolerant of the same containerization overhead, is containerized.

## Boot chain wiring (Modules 2 + 3 + 4)

The signed kernel command line embeds the dm-verity root hash, closing
the gap Module 2 flagged — an attacker who can modify the rootfs cannot
also modify the root hash without an unavailable signing key:

```console
=> setenv bootargs "root=/dev/mapper/vroot ro dm-mod.create=\"vroot,,,ro,0 2097152 verity 1 /dev/mmcblk0p3 /dev/mmcblk0p4 4096 4096 262144 1 sha256 ${VERITY_ROOT_HASH}\""
```

OP-TEE (Module 4) hosts the per-device identity key used for fleet
enrollment (Module 7) — generated on-device via the SoC's hardware unique
key, never injected:

```console
$ optee_example_secure_storage generate-key device_identity_key
$ optee_example_secure_storage sign-csr device_identity_key "$SERIAL" > device.csr
```

A/B slot selection (Module 3) is what the OTA agent flips after a
successful RAUC install, gated by the same boot-counter/mark-good
discipline — and critically, the health check that gates `mark-good` here
checks the **actual product function**, not just boot success:

```bash
#!/bin/sh
# capstone health check — must pass before mark-good
systemctl is-active --quiet vision-pipeline.service || exit 1
curl -sf --max-time 5 http://localhost:8081/vision/healthz || exit 1
systemctl is-active --quiet actuator-control.service || exit 1
python3 -c "import tflite_runtime.interpreter as t; \
    i = t.Interpreter('/opt/models/current.tflite'); i.allocate_tensors()" || exit 1
rauc status mark-good
```

## Vision pipeline container (Modules 5 + 6)

```console
$ podman run -d --name vision-pipeline \
    --device=/dev/video0 --device=/dev/ion \
    --memory=256m --cpuset-cpus=2-3 \
    --security-opt=label=disable \
    fleet-registry.example.com/vision-pipeline:2.3.1
```

Pinned to specific cores (`--cpuset-cpus=2-3`) deliberately away from
whichever cores host `actuator-control`'s `SCHED_FIFO` threads — this is
Module 6's "isolate with cpuset, don't rely on CPU quota alone" guidance
applied directly. The delegate placement log (Module 5) is checked as
part of the container's own startup health check, not assumed:

```console
$ podman logs vision-pipeline | grep -i "falling back"
VX_DELEGATE: unsupported op RESIZE_BILINEAR, falling back to CPU
```

One CPU-fallback op here was accepted as a known, measured tradeoff
during development — documented in the design, not discovered as a
surprise during a field performance complaint.

## Fleet identity, hardening, and licensing tie-together (Modules 7, 8, 9)

```console
$ ss -tulnp
tcp  LISTEN  0.0.0.0:8081   vision-pipeline    # local healthz only, firewalled externally
tcp  LISTEN  0.0.0.0:22     sshd               # key-only, no root login
```

Every listening socket has a specific justification, per Module 8's
audit discipline; `telnetd`/`tftpd` were never enabled in this image's
recipe in the first place, rather than disabled after the fact.

```console
$ jq '.package[] | select(.issue[].status=="Unpatched")' \
    tmp/deploy/images/*/core-image-imx95-product.rootfs.cve.json | wc -l
7
```

Each of those 7 findings gets triaged by reachability (Module 8) — an
unpatched CVE in a build-time-only tool that never ships would be a
build recipe bug, not a triage entry; the actual list is what's present
in the shipped rootfs.

```console
$ oe-spdx-creator -i core-image-imx95-product
```

The SBOM produced here doubles as the license-compliance manifest
(Module 9) and the CVE-tracking input (Module 1/8) — one pipeline, two
consumers, exactly as Module 9 recommended.

## Rollout plan for a fleet update (Modules 3 + 7)

```console
$ fleetctl rollout create --version=2.4.0 \
    --strategy=canary --stages="1%,10%,50%,100%" \
    --stage-duration=48h \
    --abort-on="mark_good_rate<95%,silent_over_24h_delta>0.5%"
```

48-hour stages (not 24) reflect the actuator-control loop's own
operational cadence — a subtle regression in the real-time loop under
sustained load might not manifest in the first few hours, which is
exactly the trap Module 7 flagged about rollout durations that don't
match the failure modes you're actually trying to catch.

## Design review: where each module's failure mode was specifically designed against

| Module's core risk | This design's specific mitigation |
|---|---|
| Bricked unit from bad SRK fuse burn (M2) | Full chain validated on open-mode hardware before any fuse burn; documented as a separate, gated manufacturing step |
| Update leaves device worse off (M3) | A/B + boot counter + function-level (not boot-level) health check |
| Key material crossing the TrustZone boundary (M4) | Identity key generated and used entirely inside OP-TEE; only CSR/signature crosses out |
| NPU fallback silently eating the latency budget (M5) | Delegate log checked at container startup, not just at model-conversion time |
| Real-time loop starved by container CPU quota (M6) | Control loop kept off containers entirely; vision pipeline pinned via `cpuset`, not quota alone |
| Device identity leaking via off-device key generation (M7) | On-device HUK-derived key, never injected |
| debug-tweaks / open ports surviving to production (M8) | Never enabled in the recipe; audited via `ss -tulnp` per release |
| GPL/SBOM drift release over release (M9) | SBOM regenerated as part of every release's CI pipeline, not a launch-only artifact |

!!! note "On verification"
    This capstone is a design review and architecture synthesis of
    Modules 1-9, cross-checked internally for consistency (e.g. that the
    cpuset assignments in the container section don't overlap the cores
    implied for the real-time actuator loop); no part of this system —
    boot chain, OTA bundle, container, or fleet backend — was built,
    signed, flashed, or run on real or emulated i.MX95 hardware from this
    machine. Treat it as the starting architecture to validate against
    your actual hardware and organizational constraints.

## Stretch goals

- Extend the health-check script to include a canary inference against a
  fixed, known-good input frame and assert the output matches expected
  bounds — catching a silently corrupted or mismatched model file that a
  simple "did it load" check would miss.
- Add a Jailhouse cell (Module 6) hosting a hard-real-time RTOS partition
  for the actuator loop instead of a `SCHED_FIFO` host process, and
  compare the two approaches' worst-case latency using `cyclictest`-style
  measurement (Level 3, Module 9) under sustained vision-pipeline load.
- Design the GPLv3 audit process (Module 9) specifically for this
  product's signed-boot chain — identify every GPLv3 component in the
  dependency tree and document, per component, whether its presence
  creates a genuine Installation Information obligation given your
  signing policy.
- Build the actual staged-rollout abort conditions into a real dashboard
  query against the telemetry schema from Module 7, and simulate (with
  synthetic data) what a bad update's `silent_over_24h_delta` signal would
  look like at each rollout stage.
