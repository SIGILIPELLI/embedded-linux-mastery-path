# 03 · OTA Updates (RAUC, SWUpdate, OSTree)

A device that can be field-updated can also be field-bricked by that same
mechanism — a power loss mid-flash, a corrupted download, or a bad image
that boots into a crash loop. Every production-grade OTA design exists to
make one guarantee: **a failed update never leaves the device in a worse
state than before the update started.** This module covers the two
dominant strategies for that guarantee — A/B partition swapping and
OSTree's atomic filesystem checkout — and where each still fails if built
carelessly.

## A/B (dual-copy) updates: the core mechanism

```
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│  Bootloader   │──▶│  rootfs A     │   │  rootfs B     │
│ (boot counter,│   │ (active)      │   │ (inactive,    │
│  slot select) │   └──────────────┘   │  update target)│
└──────────────┘                        └──────────────┘
```

The bootloader tracks which slot is active and a **boot counter**; a new
update writes to the *inactive* slot entirely, so the currently-running
system is never touched during the write. Only after the write completes
and is verified does the bootloader's slot pointer flip — and even then,
only provisionally, pending a successful boot.

## RAUC: bundle format and slot management

```console
$ rauc bundle --cert=dev.cert.pem --key=dev.key.pem \
    update-content/ product-v2.3.0.raucb
$ rauc info product-v2.3.0.raucb
Compatible:  'product-imx8mp'
Version:     '2.3.0'
Bundle Format: verity
Hooks:       'none'
Images:
  [rootfs]
    filename: rootfs.ext4
    sha256:   3f2a...
```

Verity-format bundles pair each image with a dm-verity hash tree checked
continuously during install, not just via a single upfront checksum —
this closes the gap where a bundle passes an initial SHA-256 check but a
storage-media bit flip during the (potentially minutes-long) write
corrupts data that's never re-verified.

```console
$ rauc install product-v2.3.0.raucb
installing
0% Installing
20% Determining slot states
40% Checking bundle
60% Copying image to rootfs.1
100% Installing done.
$ rauc status
=== System Info ===
Compatible: product-imx8mp
Booted from: rootfs.0 (system0)
=== Slot States ===
o [rootfs.0] (booted, active)
  [rootfs.1] (inactive, marked good, contains 'v2.3.0')
```

Note: **install does not itself switch the boot slot to "active" until
the mark-good step below** — this is deliberate two-phase commit
behavior, not a bug.

## The mark-good/boot-counter dance: how rollback actually happens

```console
=> setenv bootcount 0
=> setenv upgrade_available 1
=> setenv altbootcmd "setenv rootpart 0; saveenv; run bootcmd"
=> saveenv
=> reset
```

U-Boot boots the newly-flashed slot with a nonzero boot counter armed. If
the new image never reaches a point in userspace where it explicitly
calls `rauc status mark-good` (or the equivalent U-Boot env write), the
**next** reset finds the boot counter already elapsed and automatically
falls back to the previous known-good slot:

```console
target$ systemctl status product-healthcheck.service
● product-healthcheck.service - Post-update health check
   Active: active (exited)
$ rauc status mark-good
marking slot rootfs.1 as good
```

**The single most important design decision in this entire module**: the
health check that triggers `mark-good` must verify the *product actually
works*, not merely that Linux booted. A device that boots to a login
prompt but whose application crash-loops, or whose critical peripheral
driver silently failed to bind, still marks itself "good" under a
naive "did init finish" check — and now there is no automatic path back
to the last working version, because the bootloader has no way to know
the difference between "booted" and "working."

```bash
#!/bin/sh
# product-healthcheck.sh — runs post-boot, before mark-good
systemctl is-active --quiet product-app.service || exit 1
curl -sf --max-time 5 http://localhost:8080/healthz || exit 1
i2cget -y 1 0x68 0x00 || exit 1     # confirm the IMU actually bound
rauc status mark-good
```

## SWUpdate: the alternative, streaming-first design

SWUpdate favors a single `.swu` archive applied via a handler chain,
commonly used when updates arrive over a constrained/streaming channel
(a serial link, a low-bandwidth cellular connection) rather than a
downloaded-then-installed file:

```console
$ swupdate -i product-v2.3.0.swu -k public.pem
[INFO] SWUPDATE running :  [read_lines_notify] : Software Update
started !
[INFO] SWUPDATE running :  [install_single_image] : Installing rootfs.ext4
[INFO] SWUPDATE running :  [install_single_image] : Copy Successful
[INFO] SWUPDATE running :  [swupdate_verify_file] : Signature verify OK
```

Both RAUC and SWUpdate solve the same core problem; the choice is usually
driven by existing infrastructure (RAUC pairs naturally with a Yocto-based
BSP and `meta-rauc`; SWUpdate has a more mature story for streaming
updates and custom handler chains) rather than one being categorically
better.

## OSTree: atomic filesystem checkout instead of partition-swap

OSTree takes a different approach entirely — a Git-like content-addressed
object store for the *entire* rootfs, where an update is a new checked-out
deployment, and rollback means booting the previous deployment's
bootloader entry, not flashing anything:

```console
$ ostree admin status
* product 3f2a9c1.0 (booted)
    origin refspec: product:product/x86_64/stable
  product a8b7e21.1
    origin refspec: product:product/x86_64/stable
$ ostree admin deploy product:product/x86_64/stable
$ ostree admin status
  product 3f2a9c1.0
* product c91d4f0.2 (booted)
    origin refspec: product:product/x86_64/stable
```

Rollback is just booting the other GRUB/U-Boot entry that still points at
the previous deployment's unchanged, content-addressed files — nothing
was overwritten, so there's no "restore" step, only "boot the other
already-complete deployment":

```console
$ ostree admin rollback
$ reboot
```

OSTree's deduplication (shared file objects across deployments, hardlinked)
also means incremental updates transfer only genuinely changed content,
which matters a great deal on cellular-connected fleets where a full
rootfs image download every release is a real cost.

## Traps

- **A "successful" install that never gets a real health check before
  mark-good** — see above; this is the trap that turns a solved rollback
  problem back into an unsolved one.
- **Power loss during the write phase, before verification completes** —
  the inactive slot being mid-write is exactly why it must stay marked
  inactive until a post-write verification (verity hash, checksum) passes;
  never flip the active pointer before that check.
- **A/B updates that don't also version the bootloader/DT overlay
  combination** — if a new rootfs assumes a DT change that only ships in
  a new bootloader, but the bootloader update failed or is a separate,
  unsynchronized OTA channel, the new rootfs can boot against the *old*
  device tree and misbehave in ways that look like a rootfs bug but
  aren't.

## Cheat sheet

| Mechanism | Rollback trigger | Storage model |
|---|---|---|
| RAUC A/B | Bootloader boot-counter timeout, no `mark-good` | Two full rootfs partitions |
| SWUpdate | Handler-chain failure / signature failure | Configurable, often A/B similar |
| OSTree | `ostree admin rollback`, boot other deployment | Content-addressed, deduplicated, no partition swap |
| dm-verity (any) | Continuous hash-tree check during read | Read-only, tamper-evident |

!!! note "On verification"
    RAUC/SWUpdate/OSTree command syntax and the boot-counter rollback
    mechanism are checked against each project's documented workflow and
    Level 4's secure-boot module; no update bundle was built, signed, or
    installed against real or emulated hardware from this machine.

## Exercise

(1) Write the full boot-counter U-Boot environment sequence for a fresh
A/B install: arm the counter, boot the new slot, and the fallback command
that fires if `mark-good` never runs. (2) Design a health-check script
(pseudocode is fine) for a product with a network service, a GPIO-based
watchdog LED, and one I2C sensor — list the three checks in the order
you'd run them and justify the order. (3) One paragraph: explain why
OSTree's rollback needs no "restore" step the way A/B partition rollback
conceptually does, in terms of what OSTree does and does not overwrite
during a deployment.
