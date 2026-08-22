# 09 · Compliance & Open-Source Licensing

A product built on Linux, U-Boot, BusyBox, and a hundred other open
source packages inherits a hundred different licenses' obligations —
and getting this wrong is a legal and business risk, not a technical
inconvenience. GPLv2 (the kernel), GPLv3, LGPL, and permissive licenses
(MIT, BSD, Apache) each impose different obligations, and a shipped
product is where those obligations become enforceable.

## What GPLv2 actually requires for a shipped kernel

The Linux kernel is GPLv2. Shipping a device running it obligates you,
on request, to provide the **complete corresponding source** for the
exact kernel you shipped — including your out-of-tree patches — to anyone
who receives the binary, not just customers who ask nicely:

```console
$ cat linux-source-offer.txt
This product contains software licensed under the GNU General
Public License v2. To obtain the complete corresponding source
code, including all patches applied to this build, write to:
  Open Source Compliance, Acme Devices Inc, ...
or download it directly at:
  https://opensource.acmedevices.com/product-x/v2.3.1/
```

A written offer is only compliant if the source you actually provide
**matches the exact binary shipped** — the same kernel version, the same
patch stack, the same `.config`. This is exactly why Module 1's manifest
archiving discipline (exact layer pins, exact patch stack per release)
isn't just good engineering practice, it's the artifact that makes GPL
compliance provable rather than aspirational.

```console
$ tar czf product-x-v2.3.1-gpl-source.tar.gz \
    linux-6.6.35-product-x/ u-boot-2024.01-product-x/ busybox-1.36.1/
$ sha256sum product-x-v2.3.1-gpl-source.tar.gz > product-x-v2.3.1-gpl-source.sha256
```

## GPLv2 vs LGPL vs permissive: what changes obligation-wise

| License | Distributing binary obligates you to... |
|---|---|
| **GPLv2** (kernel, many GNU tools) | Provide complete corresponding source for that exact binary, on request |
| **LGPL** (glibc, many libraries) | Provide source for the LGPL'd library itself; permits **dynamic** linking from proprietary code without extending GPL obligations to that code |
| **MIT / BSD / Apache** | Retain copyright notice and license text; no source-disclosure obligation |
| **GPLv3** | Adds anti-Tivoization language in some cases — for locked-down/signed-boot devices, this is the license clause with the most direct interaction with Module 2's secure boot design |

**GPLv3's anti-Tivoization concern is the one that directly collides with
secure boot.** GPLv3 (used by some but not all GNU tools — `bash`,
`coreutils`, modern `gcc`) requires that if you provide "Installation
Information" for a GPLv3-covered work, a user must actually be able to
install a *modified* version and have it run — which is in direct tension
with a secure-boot chain (Module 2) that refuses to boot anything not
signed by your key. Practically: audit your dependency tree for GPLv3
components specifically, separately from the more common GPLv2 audit,
because the compliance answer is genuinely different and harder.

## License scanning: automating the audit

```console
$ bitbake -c populate_lic core-image-product
$ cat tmp/deploy/licenses/core-image-product-*/license.manifest | \
    awk -F': ' '{print $2}' | sort | uniq -c | sort -rn | head
    142 MIT
     84 GPLv2
     41 BSD-3-Clause
     12 LGPLv2.1
      3 GPLv3
      2 Apache-2.0
```

Yocto's `populate_lic` task, run automatically as part of a normal build
when `LICENSE_CREATE_PACKAGE_ARCHIVE` and related license variables are
set, produces exactly this manifest per image — the raw input a
compliance review actually needs, generated as a build artifact rather
than reconstructed after the fact by manually reading every recipe.

```console
$ scancode --license --json-pp scan-results.json vendor-sdk-blob/
```

For binary vendor SDK blobs (an NPU delegate library, a proprietary
camera ISP tuning tool) with no visible license file, `scancode` (or an
equivalent license-fingerprinting tool) at least surfaces embedded
copyright strings and license text fragments worth escalating to the
vendor for clarification — silence from a vendor about licensing terms is
not the same as "no obligation exists," it's an unresolved compliance gap.

## The SBOM: what shipped, in a form auditors and customers can read

A Software Bill of Materials is the modern deliverable a growing number
of customers and regulations (in some jurisdictions, increasingly a
procurement requirement) expect alongside the product itself:

```json
{
  "bomFormat": "CycloneDX",
  "specVersion": "1.5",
  "components": [
    {"type": "library", "name": "linux", "version": "6.6.35",
     "licenses": [{"license": {"id": "GPL-2.0-only"}}]},
    {"type": "library", "name": "openssl", "version": "3.0.9",
     "licenses": [{"license": {"id": "Apache-2.0"}}]}
  ]
}
```

```console
$ oe-spdx-creator -i core-image-product
$ ls tmp/deploy/spdx/
core-image-product.spdx.json
```

An SBOM is also exactly the artifact that makes Module 1's CVE-tracking
obligation and this module's license-tracking obligation the *same
underlying data* viewed two ways — the component list that tells you
what to scan for known vulnerabilities is the same list that tells you
what license obligations apply. Building one pipeline that produces both
views is far less effort than maintaining them as separate manual
processes.

## Traps

- **A written GPL source offer whose actual downloadable archive doesn't
  match the shipped binary** — a mismatched patch stack or wrong kernel
  version in the offered source, discovered by anyone who actually
  exercises the offer, converts a compliance program into evidence of
  non-compliance.
- **Treating "it's open source, so it's automatically fine" as a blanket
  answer.** Every license has different obligations; a GPLv3 component
  buried in a dependency tree can create genuine tension with a secure
  boot design that nobody flagged until legal review, late in a program.
- **No process for vendor SDK blobs with unclear licensing** — silence is
  not clearance; unresolved license status on a shipped binary component
  is a real, open risk that a scan will surface but won't resolve on its
  own.
- **License compliance treated as a one-time pre-launch checklist**
  instead of a per-release artifact — every dependency bump (Module 1's
  ongoing maintenance work) can introduce a new license into the tree,
  and the SBOM/manifest needs regenerating every release, not once at
  launch.

## Cheat sheet

| Obligation | Practice |
|---|---|
| GPLv2 kernel/tools | Written offer + archive matching the *exact* shipped source, per release |
| LGPL libraries | Dynamic linking preserves proprietary-code exemption; confirm linking mode |
| GPLv3 components | Audit separately — direct tension with secure boot (Module 2) |
| Unclear-license vendor blobs | Scan (`scancode`), escalate to vendor, don't assume clearance |
| Ongoing tracking | Generate license manifest + SBOM per release, same pipeline as CVE data |

!!! note "On verification"
    License obligations summarized here reflect the commonly understood
    terms of GPLv2/LGPL/MIT/BSD/GPLv3 as documented by their respective
    license texts and standard OSS compliance practice, and the Yocto
    `populate_lic`/SPDX tooling described follows documented BitBake
    conventions; this module is not legal advice — a real product's
    compliance program should involve counsel, and no license scan was
    actually run on this machine.

## Exercise

(1) Write the written GPL source-offer text your product would ship
(printed manual, or a settings-menu string) and describe exactly what
archive you'd need to keep on hand to honor a request against it a year
after that specific release shipped. (2) Given a Yocto `license.manifest`
showing 3 GPLv3 packages in an image for a device with a closed secure
boot chain, describe the compliance question you'd raise and to whom. (3)
One paragraph: explain why an SBOM built as a side effect of the same
pipeline that runs `cve_check` is a better process than maintaining
license and vulnerability tracking as two separate manual spreadsheets.
