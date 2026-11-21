---
title: "Final Quiz"
description: "Test what you have learned across all four levels of Embedded Linux \u0026 Architectures Mastery Path with a 19-question final quiz."
---

# Final Quiz

19 questions drawn from all four levels of this course. You need **70% (14 out of 19)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "List the boot stages in order.",
    "options": [
      "Init, kernel, ROM",
      "Boot ROM, SPL/bootloader, U-Boot, kernel, init",
      "Kernel, U-Boot, ROM, init",
      "U-Boot, init, kernel"
    ],
    "correctIndex": 1,
    "explanation": "Each stage hands off to the next."
  },
  {
    "question": "The kernel boots but can't find the UART on an ARM board. What is the likely cause?",
    "options": [
      "Wrong keyboard",
      "Bad compiler",
      "Missing apt",
      "The device tree doesn't describe it (or marks it disabled), since the hardware isn't discoverable"
    ],
    "correctIndex": 3,
    "explanation": "Check the dts, compatible and status."
  },
  {
    "question": "`ls` on a BusyBox system is a symlink to busybox. How does it know to act as ls?",
    "options": [
      "A config file",
      "The kernel",
      "It inspects argv[0], the name it was invoked with",
      "Environment"
    ],
    "correctIndex": 2,
    "explanation": "A multi-call binary."
  },
  {
    "question": "You must build a C app for a 64 MB ARM board with no compiler. What do you do?",
    "options": [
      "Compile on the target",
      "Use Python",
      "Use apt",
      "Cross-compile on the host with the matching toolchain/sysroot and copy the binary"
    ],
    "correctIndex": 3,
    "explanation": "Match ABI and libc."
  },
  {
    "question": "A service must restart if it crashes and be flagged if it hangs. What systemd settings help?",
    "options": [
      "Nothing",
      "Type=simple only",
      "RemainAfterExit",
      "Restart=on-failure and a watchdog (WatchdogSec with sd_notify)"
    ],
    "correctIndex": 3,
    "explanation": "Supervision."
  },
  {
    "question": "Field units lose power often and the root filesystem corrupts. What's the standard design?",
    "options": [
      "A bigger flash",
      "A read-only root with writable data on a separate partition (and tmpfs for volatile files)",
      "Run fsck",
      "Disable caching only"
    ],
    "correctIndex": 1,
    "explanation": "Prevents corruption by construction."
  },
  {
    "question": "A kernel driver needs to share data between an interrupt handler and process context. What lock?",
    "options": [
      "nothing",
      "spin_lock_irqsave (a mutex can sleep, which is forbidden in IRQ context)",
      "semaphore",
      "mutex"
    ],
    "correctIndex": 1,
    "explanation": "Atomic context."
  },
  {
    "question": "Why do threaded IRQs help?",
    "options": [
      "Speed",
      "Style",
      "Law",
      "Handlers run in process context where they can sleep, keeping hard IRQ time short"
    ],
    "correctIndex": 3,
    "explanation": "Use for slow buses."
  },
  {
    "question": "An optional daughter card appears on some units. How do you avoid maintaining two device trees?",
    "options": [
      "Use a device tree overlay applied by the bootloader or at runtime",
      "Ignore",
      "Two images",
      "Edit the kernel"
    ],
    "correctIndex": 0,
    "explanation": "Overlays."
  },
  {
    "question": "A Cortex-M core handles hard real-time I/O beside Linux. Which frameworks connect them?",
    "options": [
      "USB",
      "SSH",
      "remoteproc to load/control firmware and RPMsg for messaging",
      "NFS"
    ],
    "correctIndex": 2,
    "explanation": "Heterogeneous SoCs."
  },
  {
    "question": "Latency spikes appear on a Linux control loop. What do you try?",
    "options": [
      "Bigger cache",
      "PREEMPT_RT with SCHED_FIFO priorities, measured with cyclictest",
      "More RAM",
      "Disable RT"
    ],
    "correctIndex": 1,
    "explanation": "Measure worst case."
  },
  {
    "question": "What chain does secure boot rely on?",
    "options": [
      "Network",
      "Each stage verifies the next stage's signature, anchored in the immutable boot ROM and fused keys",
      "Checksums",
      "Passwords"
    ],
    "correctIndex": 1,
    "explanation": "Chain of trust."
  },
  {
    "question": "A firmware update fails halfway and the device must still boot. What OTA design handles it?",
    "options": [
      "A/B slots with a boot counter and automatic rollback",
      "Two chips",
      "Overwrite in place",
      "Manual recovery"
    ],
    "correctIndex": 0,
    "explanation": "Always keep a known-good image."
  },
  {
    "question": "Why sign OTA bundles?",
    "options": [
      "Law",
      "Speed",
      "So devices only install authentic, untampered images",
      "Style"
    ],
    "correctIndex": 2,
    "explanation": "Integrity and authenticity."
  },
  {
    "question": "Where should secrets such as keys live on an i.MX board?",
    "options": [
      "In the app",
      "In /etc",
      "In U-Boot env",
      "In the TrustZone secure world (OP-TEE) or secure hardware, not in the Linux rootfs"
    ],
    "correctIndex": 3,
    "explanation": "Isolate."
  },
  {
    "question": "Why must you track open-source licenses in a product?",
    "options": [
      "Style",
      "Cost",
      "Speed",
      "GPL and other licenses impose obligations on distribution (source offers, notices)"
    ],
    "correctIndex": 3,
    "explanation": "Generate an SBOM."
  },
  {
    "question": "A Yocto build is not reproducible and security fixes are missed. What practices help?",
    "options": [
      "Update randomly",
      "Pin layer revisions, track CVEs, and build in CI",
      "Fork everything",
      "Skip CI"
    ],
    "correctIndex": 1,
    "explanation": "Maintain the BSP."
  },
  {
    "question": "What reduces the attack surface of a shipped device?",
    "options": [
      "Leave UART open",
      "Add tools",
      "Remove unused services/packages, disable debug ports, least privilege and a firewall",
      "Root login"
    ],
    "correctIndex": 2,
    "explanation": "Hardening."
  },
  {
    "question": "How do you roll out an update to a fleet safely?",
    "options": [
      "Staged rollout with health telemetry and automatic rollback",
      "Email",
      "Manual",
      "All at once"
    ],
    "correctIndex": 0,
    "explanation": "Canary first."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### A/B update with rollback

Complete the toy A/B bootloader: after install(), boot() should commit the pending slot when healthy, otherwise count failed tries and roll back to the active slot after MAX_TRIES (discarding the pending image). Print the sequence for a bad v2 update followed by a good v3.

**Starter code**

```python
class ABBoot:
    """Toy bootloader state for an A/B OTA scheme with a boot counter."""
    MAX_TRIES = 3

    def __init__(self):
        self.slots = {"A": "v1", "B": None}
        self.active = "A"
        self.pending = None       # slot being trialled after an update
        self.tries = 0

    def install(self, version):
        inactive = "B" if self.active == "A" else "A"
        self.slots[inactive] = version
        self.pending = inactive
        self.tries = 0
        return inactive

    def boot(self, healthy):
        """Simulate one boot of the pending slot (or the active one if nothing is pending).
        Healthy: commit the pending slot as active. Unhealthy: count the try; after MAX_TRIES
        roll back to the active slot and discard the pending one. Return a status string."""
        return ""

b = ABBoot()
b.install("v2")
for h in [False, False, False]:
    print(b.boot(h))
print(b.boot(True))
b.install("v3")
print(b.boot(True))
print(b.boot(True))
```

**Expected output**

```text
boot failed 1x -> retry B
boot failed 2x -> retry B
boot failed 3x -> rollback to A (v1)
boot A (v1)
boot B (v3) -> committed
boot B (v3)
```

<details><summary>Show solution and explanation</summary>

<pre><code>class ABBoot:
    &quot;&quot;&quot;Toy bootloader state for an A/B OTA scheme with a boot counter.&quot;&quot;&quot;
    MAX_TRIES = 3

    def __init__(self):
        self.slots = {&quot;A&quot;: &quot;v1&quot;, &quot;B&quot;: None}
        self.active = &quot;A&quot;
        self.pending = None       # slot being trialled after an update
        self.tries = 0

    def install(self, version):
        inactive = &quot;B&quot; if self.active == &quot;A&quot; else &quot;A&quot;
        self.slots[inactive] = version
        self.pending = inactive
        self.tries = 0
        return inactive

    def boot(self, healthy):
        &quot;&quot;&quot;Simulate one boot. `healthy` says whether the new software comes up fine.&quot;&quot;&quot;
        if self.pending is None:
            return f&quot;boot {self.active} ({self.slots[self.active]})&quot;
        if healthy:
            self.active, self.pending = self.pending, None
            return f&quot;boot {self.active} ({self.slots[self.active]}) -&gt; committed&quot;
        self.tries += 1
        if self.tries &gt;= self.MAX_TRIES:
            bad = self.pending
            self.pending = None
            self.slots[bad] = None
            return f&quot;boot failed {self.tries}x -&gt; rollback to {self.active} ({self.slots[self.active]})&quot;
        return f&quot;boot failed {self.tries}x -&gt; retry {self.pending}&quot;

b = ABBoot()
b.install(&quot;v2&quot;)
for h in [False, False, False]:
    print(b.boot(h))
print(b.boot(True))
b.install(&quot;v3&quot;)
print(b.boot(True))
print(b.boot(True))
</code></pre>

<p>The active slot is never overwritten, so a failed update falls back to a known-good image. Real systems (RAUC, SWUpdate, Mender with U-Boot boot counts) implement the same idea. This is a teaching model. Verified by running the solution.</p>
</details>

### Match device tree nodes to drivers

Implement match(node): skip nodes whose status is not okay, then walk the node's compatible list in order (most specific first) and return the first driver whose table contains one of the strings, along with the matching string.

**Starter code**

```python
# A tiny model of the Linux driver core matching a device-tree node to a driver.
DRIVERS = {
    "imx-uart": ["fsl,imx6q-uart", "fsl,imx21-uart"],
    "ns16550": ["ns16550a", "ns16550"],
    "gpio-led": ["gpio-leds"],
}

NODES = [
    {"name": "serial@4000", "compatible": ["fsl,imx6q-uart", "fsl,imx21-uart"], "status": "okay"},
    {"name": "serial@5000", "compatible": ["vendor,new-uart", "ns16550a"], "status": "okay"},
    {"name": "leds", "compatible": ["gpio-leds"], "status": "disabled"},
    {"name": "mystery@9000", "compatible": ["acme,unknown"], "status": "okay"},
]

def match(node):
    # return (driver, matching compatible string) or None; skip nodes whose status is not "okay";
    # check node["compatible"] in order (most specific first) against each driver table
    return None

for n in NODES:
    r = match(n)
    print(n["name"], "->", f"{r[0]} (via {r[1]})" if r else "no driver bound")
```

**Expected output**

```text
serial@4000 -> imx-uart (via fsl,imx6q-uart)
serial@5000 -> ns16550 (via ns16550a)
leds -> no driver bound
mystery@9000 -> no driver bound
```

<details><summary>Show solution and explanation</summary>

<pre><code># A tiny model of the Linux driver core matching a device-tree node to a driver.
DRIVERS = {
    &quot;imx-uart&quot;: [&quot;fsl,imx6q-uart&quot;, &quot;fsl,imx21-uart&quot;],
    &quot;ns16550&quot;: [&quot;ns16550a&quot;, &quot;ns16550&quot;],
    &quot;gpio-led&quot;: [&quot;gpio-leds&quot;],
}

NODES = [
    {&quot;name&quot;: &quot;serial@4000&quot;, &quot;compatible&quot;: [&quot;fsl,imx6q-uart&quot;, &quot;fsl,imx21-uart&quot;], &quot;status&quot;: &quot;okay&quot;},
    {&quot;name&quot;: &quot;serial@5000&quot;, &quot;compatible&quot;: [&quot;vendor,new-uart&quot;, &quot;ns16550a&quot;], &quot;status&quot;: &quot;okay&quot;},
    {&quot;name&quot;: &quot;leds&quot;, &quot;compatible&quot;: [&quot;gpio-leds&quot;], &quot;status&quot;: &quot;disabled&quot;},
    {&quot;name&quot;: &quot;mystery@9000&quot;, &quot;compatible&quot;: [&quot;acme,unknown&quot;], &quot;status&quot;: &quot;okay&quot;},
]

def match(node):
    if node[&quot;status&quot;] != &quot;okay&quot;:
        return None
    # compatible is ordered most specific first; the first string any driver claims wins
    for compat in node[&quot;compatible&quot;]:
        for driver, table in DRIVERS.items():
            if compat in table:
                return driver, compat
    return None

for n in NODES:
    r = match(n)
    print(n[&quot;name&quot;], &quot;-&gt;&quot;, f&quot;{r[0]} (via {r[1]})&quot; if r else &quot;no driver bound&quot;)
</code></pre>

<p>The driver core binds a driver by compatible string, preferring the earliest (most specific) entry, so serial@5000 binds to ns16550 via its fallback string. Disabled or unknown nodes bind nothing. This models the idea, not the kernel&#x27;s actual code. Verified by running the solution.</p>
</details>
