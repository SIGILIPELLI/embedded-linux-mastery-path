# 01 · Production BSP Maintenance

Everything through Level 3 assumed you're building a BSP once. A shipped
product's BSP has a multi-year lifecycle: CVE patches land upstream every
week, the vendor SDK gets superseded, and a fleet in the field cannot
tolerate the same "just rebuild from scratch" workflow you used during
bring-up. This module is about the practices that keep a BSP maintainable
for years, not the initial build.

## Pinning and tracking: the layer/meta problem

A Yocto-based BSP is a graph of layers (`meta`, `meta-freescale`,
`meta-openembedded`, your own `meta-product`), each moving independently.
"It built fine six months ago" is not reproducible without exact
revision pins:

```yaml
# manifest.yml — repo tool style, or an equivalent kas config
sources:
  meta-freescale:
    remote: https://github.com/Freescale/meta-freescale.git
    revision: a3f9c21e8b4d5f0e1a2b3c4d5e6f7089abcdef01   # pin, not a branch
  meta-openembedded:
    remote: https://github.com/openembedded/meta-openembedded.git
    revision: 5c8e91a0d2f3b4c5d6e7f8091a2b3c4d5e6f7089
```

**Never pin to a branch name** (`dunfell`, `kirkstone`) in a manifest
meant to be reproducible — branches move; a build triggered today and one
triggered next month against the same manifest file can silently pull
different commits, and "the CI build passed but the release build failed"
becomes a recurring, hard-to-explain support burden. Pin to a commit SHA,
and update deliberately.

`kas` (or Google's `repo`) is the standard tool for managing this
multi-layer pin set as one versioned file, reviewable in a pull request
like any other change:

```console
$ kas build ci/product-release.yml
$ kas shell ci/product-release.yml -c "bitbake-layers show-layers"
```

## CVE tracking: the recurring obligation

Every week, new CVEs land against packages your image ships — `openssl`,
`busybox`, the kernel itself. Yocto's `cve-check` class cross-references
your build's package versions against the NVD database:

```console
$ bitbake -c cve_check core-image-minimal
$ cat tmp/deploy/images/*/core-image-minimal.rootfs.cve.json | \
    jq '.package[] | select(.issue[].status=="Unpatched")'
{
  "package": "openssl",
  "version": "3.0.9",
  "issue": [
    { "id": "CVE-2024-XXXXX", "status": "Unpatched", "cvss_v3_score": 7.5 }
  ]
}
```

A CVE report with hundreds of entries is normal and not itself an
emergency — most are in code paths your product never exercises (a kernel
config option compiled out, a library function never called). The
disciplined process is: triage by actual exploitability/exposure for
*your* product, document the decision (patch, mitigate, or accept-with-
justification) per CVE, and never let "the list is long" become an excuse
to ignore all of it. A `cve_check` report nobody reads is worse than not
running it — it creates a paper trail proving you knew and did nothing.

## Backporting a fix without a full version bump

Bumping `openssl` to the latest upstream release mid-product-lifecycle
risks an ABI break across everything linking against it. The safer,
narrower fix is usually a backported patch on the version you already
ship:

```console
$ cat recipes-connectivity/openssl/openssl_%.bbappend
FILESEXTRAPATHS:prepend := "${THISDIR}/files:"
SRC_URI += "file://CVE-2024-XXXXX.patch"
```

```console
$ bitbake openssl -c cve_check
$ grep CVE-2024-XXXXX tmp/deploy/.../openssl.cve.json
"id": "CVE-2024-XXXXX", "status": "Patched"
```

**Trap**: a backported security patch that doesn't apply cleanly against
your pinned version (because you're several point releases behind
upstream's assumed base) fails silently in some Yocto configurations if
`PATCHTOOL`/fuzz settings are too permissive — always confirm the patch
actually applied by checking the built binary's behavior or the
`do_patch` log, not just that `bitbake` returned success.

## The long-term-support kernel reality

Embedded products routinely ship an LTS kernel (6.6, 6.1, 5.15 stable
series) for years past its "would I pick this today" window, because a
kernel major-version bump is itself a significant validation and
regression-risk event. Two disciplines make this survivable:

- **Track the stable series, not a fixed tag.** `6.6.y` receives ongoing
  fixes; a fixed `v6.6.21` tag does not. Rebase your BSP's kernel patch
  stack onto the current stable point release on a defined cadence
  (monthly is common), not ad hoc.
- **Keep your out-of-tree patch stack small and well-documented.** Every
  vendor/product patch against the kernel is something you must re-apply
  and re-verify on every stable rebase; a patch stack that has grown to
  dozens of undocumented changes turns every LTS bump into a multi-week
  archaeology project instead of a routine update.

```console
$ git log --oneline v6.6.21..v6.6.35 -- drivers/net/ethernet/ | wc -l
41
$ ./scripts/rebase-patch-stack.sh v6.6.35
Applying 0001-product-panel-timing-fix.patch... OK
Applying 0002-product-i2c-quirk.patch... FAILED (conflict)
```

A patch stack rebase that reports a conflict is not automatically a
regression to panic over — it usually means upstream touched the exact
lines your patch touched, often because upstream fixed the same class of
issue differently. Resolve by understanding *why* your patch existed, not
by mechanically forcing the old diff back in.

## Reproducible builds and the "who built this artifact" question

A production BSP needs to answer, months later, "exactly what source
produced the image running on unit #4,821 in the field." That requires:

```console
$ bitbake core-image-minimal
$ cat tmp/deploy/images/*/core-image-minimal.rootfs.manifest | head -5
busybox aarch64 1.36.1
openssl aarch64 3.0.9
linux-imx aarch64 6.6.35+gitAUTOINC+a3f9c21e8b
```

Archive the manifest, the exact layer pin set, and the build config
(`local.conf`, `bblayers.conf`) alongside every released image — not just
the image binary. Without this, a field failure investigation months
later has no way to distinguish "this bug was in the shipped build" from
"this bug was fixed after this build but before the report."

## Traps

- **Pinning to a moving branch instead of a commit SHA** — the single
  most common cause of "unreproducible build" incidents in Yocto-based
  products.
- **Treating a long CVE list as a single yes/no gate** instead of
  triaging by actual exposure — either blocks releases on irrelevant
  findings forever, or trains the team to ignore the report entirely.
- **A patch stack rebase that "succeeds" with silent fuzz** — a patch
  applying with large fuzz offsets can land in the wrong context entirely;
  always diff the *result*, not just check the exit code.

## Cheat sheet

| Practice | Tool/command |
|---|---|
| Pin layer set to exact commits | `kas` / `repo manifest`, never a branch name |
| Find unpatched CVEs in a build | `bitbake -c cve_check <image>` |
| Backport a targeted fix | `.bbappend` + `SRC_URI += "file://CVE-....patch"` |
| Track kernel LTS series | Rebase patch stack onto `<major>.<minor>.y` on a cadence |
| Prove what shipped | Archive `.rootfs.manifest` + layer pins + `local.conf` per release |

!!! note "On verification"
    The Yocto `cve_check` workflow, `.bbappend`/`SRC_URI` patch mechanism,
    and manifest-tracking practices are checked against documented
    Yocto/BitBake conventions and Level 2's build material; no image was
    actually built or CVE-scanned on this machine.

## Exercise

(1) Write the `.bbappend` and directory layout needed to backport a
hypothetical `CVE-2024-99999` patch against `busybox`, and describe the
exact command to confirm `cve_check` now reports it patched. (2) Given a
kernel patch-stack rebase that reports one conflicting patch out of
twelve, write the process you'd follow to resolve it safely, in order.
(3) One paragraph: your team wants to stop maintaining exact layer pins
"to save time" and just build against layer branch tips. Explain the
concrete failure mode this creates for a field-support engineer six
months from now.
