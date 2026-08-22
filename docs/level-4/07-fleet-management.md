# 07 · Fleet Management & Provisioning

One device is a bring-up problem. Ten thousand devices in the field is a
different discipline entirely: every device needs a unique, unforgeable
identity, provisioning must be reproducible at manufacturing scale
without a human touching each unit, and the fleet's health has to be
observable in aggregate — because you will not get a support ticket for
every unit that's quietly degrading.

## Device identity: the foundation everything else depends on

Every unit needs a cryptographic identity established at manufacturing
time, not invented later — a serial number alone is not an identity, it's
a label anyone can claim.

```console
# At manufacturing time, per-unit:
$ openssl ecparam -genkey -name prime256v1 -noout -out device.key
$ openssl req -new -key device.key -subj "/CN=DEVICE-SN-00482931" \
    -out device.csr
$ openssl x509 -req -in device.csr -CA fleet-ca.pem -CAkey fleet-ca.key \
    -CAcreateserial -days 3650 -out device.crt
```

The private key should be generated **on the device itself** (or in a
secure element attached to it — an SE050, or the SoC's own OP-TEE-backed
keystore from Module 4) and never leave it — a key generated off-device
and injected exists somewhere else too, which means it can leak from
somewhere you don't control. Where the SoC has a hardware unique key
(HUK) or secure enclave, deriving the device key from that ties identity
to silicon that can't be cloned by copying files off the eMMC.

```console
$ tee-supplicant &
$ optee_example_secure_storage store-key device_identity_key
```

## Provisioning: zero-touch at manufacturing scale

A factory line cannot have an engineer manually configuring each board.
The standard pattern is a **provisioning image** that runs once, generates
or retrieves identity, registers with the fleet backend, and marks itself
provisioned so it never re-runs:

```bash
#!/bin/sh
# provision.sh — runs from a factory-flash initramfs, once per unit
if [ -f /factory/provisioned ]; then
	exit 0
fi

SERIAL=$(fw_printenv -n serial# 2>/dev/null || cat /sys/firmware/devicetree/base/serial-number)
optee_example_secure_storage generate-key device_identity_key
CSR=$(openssl req -new -key <(optee-key-export device_identity_key) \
        -subj "/CN=${SERIAL}")

curl -sf --cacert fleet-ca.pem \
     -F "csr=${CSR}" -F "serial=${SERIAL}" \
     https://provisioning.fleet.example.com/enroll \
     -o /factory/device.crt

touch /factory/provisioned
```

**Trap**: a provisioning script idempotency check that's wrong in the
"too permissive" direction (e.g. checking a file that a factory reflash
also wipes) causes re-provisioning on every reflash during rework —
usually harmless for identity but potentially exhausting a fleet backend's
rate limits or issuing certificates faster than the CA's issuance policy
expects during a bad manufacturing run. Test the idempotency check against
your actual rework/reflash procedure, not just the happy first-flash path.

## Fleet telemetry: aggregate observability, not per-device tickets

```console
$ cat /etc/fleet-agent/config.yml
metrics:
  interval: 300
  endpoints:
    - cpu_temp
    - mem_available
    - disk_free_pct
    - uptime_since_boot
    - update_status
report_url: https://telemetry.fleet.example.com/v1/ingest
```

The telemetry set should be small and specifically chosen to answer "is
this device degrading" and "is the fleet as a whole degrading" — not
everything the device could possibly report. A fleet dashboard drowning
in unused metrics is as useless as one with too few; the discipline is
choosing metrics you will actually act on:

```console
$ curl -s https://telemetry.fleet.example.com/v1/fleet-summary | jq
{
  "total_devices": 8412,
  "reporting_last_hour": 8390,
  "silent_over_24h": 22,
  "update_status": {"current": 8104, "pending": 264, "failed": 22, "rolled_back": 3}
}
```

`silent_over_24h` is often the single most important number on this
dashboard — a device that stops reporting isn't necessarily broken (it
could be offline for a benign reason), but a *growing* silent count
correlated with a recent OTA rollout is exactly the early warning signal
Module 3's health-check/rollback design exists to make rare, not eliminate
entirely.

## Staged rollout: never push to 100% at once

```console
$ fleetctl rollout create --version=2.3.1 \
    --strategy=canary --stages="1%,10%,50%,100%" \
    --stage-duration=24h \
    --abort-on="failed_rate>2%"
Rollout r-8821 created, stage 1 (1%, 84 devices) starting.
```

Fleet update tooling should gate each stage's progression on the previous
stage's *actual* observed health (via the telemetry above and Module 3's
mark-good rate), not on a fixed timer alone — a bad update that only
manifests after 48 hours of runtime (a slow memory leak, say) will sail
through a 24-hour canary stage regardless of stage duration unless the
rollout also tracks longer-horizon metrics for previously-updated cohorts,
not just the newest stage.

## Remote diagnostics without full remote shell access everywhere

Full SSH access to every fielded unit is both an operational convenience
and a substantial attack surface — a compromised backend credential
becomes root on the entire fleet. A narrower, audited command channel is
usually the better tradeoff:

```console
$ fleetctl exec --device=DEVICE-SN-00482931 --command=collect-diagnostics
Requesting diagnostic bundle from DEVICE-SN-00482931...
Bundle received: diag-00482931-20260822.tar.gz (dmesg, journalctl -u product-app, thermal_zone readings)
```

A predefined, signed set of diagnostic commands the device agent will
execute (log collection, a specific health check, a controlled restart)
gives support teams what they actually need without opening an arbitrary
remote-execution channel that then has to be secured with the same rigor
as SSH access to every unit.

## Traps

- **Device identity generated off-device and pushed during
  provisioning** instead of generated on-device — the moment a private
  key exists anywhere other than the device, it can leak from that other
  place, defeating the entire point of per-device identity.
- **A 100%-at-once rollout** — the single most common fleet-management
  incident pattern; any update mechanism without staged rollout gating on
  real health data will eventually ship a bad update to the entire fleet
  simultaneously.
- **Telemetry intervals aggressive enough to matter for battery-powered
  fleet devices** — a 5-minute telemetry check-in that wakes a
  battery-powered unit from deep sleep can dominate its power budget;
  telemetry interval is itself a power-management decision, not just an
  observability one.

## Cheat sheet

| Concern | Practice |
|---|---|
| Device identity | Generate key on-device/in secure element, never inject from outside |
| Zero-touch provisioning | Idempotent script, tested against real rework/reflash flow |
| Fleet health signal | Small, actionable telemetry set; watch `silent_over_24h` |
| Update rollout | Staged (canary → wider), gated on real health data, not just a timer |
| Remote diagnostics | Predefined, signed, audited commands — not blanket SSH |

!!! note "On verification"
    Device-identity, provisioning, staged-rollout, and telemetry practices
    here are checked against widely documented IoT fleet-management and
    PKI conventions and consistent with Modules 2-4's secure-boot/OTA/
    TrustZone material; no fleet backend, CA, or provisioning flow was
    actually run from this machine.

## Exercise

(1) Write the provisioning script's idempotency check so it survives a
factory rework reflash without re-enrolling a device that's already
provisioned, given that a reflash wipes `/factory` but not the secure
element's stored key — where would you check instead? (2) Design a
staged-rollout policy (stage percentages, duration, abort threshold) for
a fleet of 10,000 battery-powered devices reporting telemetry every 6
hours, and explain why your stage durations are sized the way they are
relative to the telemetry interval. (3) One paragraph: explain to a
product manager why "just give support SSH to all devices, it's faster"
is the wrong tradeoff, in terms of the actual attack surface it creates.
