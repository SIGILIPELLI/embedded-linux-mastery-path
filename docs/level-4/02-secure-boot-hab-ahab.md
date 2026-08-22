# 02 · Secure Boot Chain (HAB/AHAB)

Nothing in this module should be run against real fuses until you have
tested the entire chain exhaustively on a spare, expendable unit. Secure
boot fuses (HAB on i.MX6/7/8, AHAB on i.MX8/9) are, on most silicon,
**one-time programmable**. A wrong SRK hash burned to fuses does not
brick that one boot — it permanently and unrecoverably bricks that
specific chip, forever, because it now demands a valid signature nothing
you possess can produce.

## The chain of trust, concretely

```
Boot ROM (immutable, in silicon)
   │ verifies signature of
   ▼
SPL / first-stage bootloader (signed)
   │ verifies signature of
   ▼
U-Boot proper (signed)
   │ verifies signature of
   ▼
Kernel + initramfs / OS image (signed, or verified via dm-verity)
```

Each stage's signature is checked by the stage *before* it, rooted in a
public key hash burned into fuses — the **Super Root Key (SRK) hash**.
The Boot ROM's own code is immutable and cannot itself be updated; it is
the only link in the chain you cannot patch after the fact, which is
exactly why it only ever checks a hash of a key, never the key material
directly (key material stays revisable; the hash-of-keys mechanism does
not).

## HAB: generating and using keys

```console
$ cst --version
CST version 3.4.1
$ ./hab4_pki_tree.sh
Do you want to use an existing CA key (y/n)?: n
... generates SRK1-4 key pairs, CA, SRK table ...
$ srktool --hab_ver 4 --table SRK_1_2_3_4_table.bin \
    --efuses SRK_1_2_3_4_fuse.bin --digest sha256
```

**The SRK fuse burn is the point of no return.** Everything up to burning
`SRK_1_2_3_4_fuse.bin` into OTP fuses is fully reversible — regenerate
keys, rebuild images, try again. The instant those fuses are blown, the
Boot ROM on that specific chip will refuse to boot anything not signed by
a key matching that exact hash, forever. There is no vendor recovery
procedure that un-blows a fuse.

```console
# On a UNIT YOU CAN AFFORD TO DESTROY, and only after
# exhaustively validating signing on unfused hardware first:
=> fuse prog 6 0 0x12345678
=> fuse prog 6 1 0x9abcdef0
=> fuse prog 6 2 0x...
=> fuse prog 6 3 0x...
=> fuse sense 6 0 4      # read back and verify before trusting it
```

Practice on **HAB open mode with closed-mode-equivalent signature
checking disabled** (or an emulated/QEMU target where fusing isn't
physical) until every image in your chain signs and verifies correctly,
repeatedly, before touching real OTP fuses on hardware that matters.

## Signing images

```console
$ cst -i csf_uboot.txt -o csf_uboot_signed.bin
$ objcopy -I binary -O binary --pad-to 0x2000 \
    u-boot.bin u-boot-pad.bin
$ cat u-boot-pad.bin csf_uboot_signed.bin > u-boot-signed.bin
```

The CSF (Command Sequence File) describes exactly which memory ranges get
signed and where the signature block is appended — an image whose IVT
(Image Vector Table) offset or CSF pointer doesn't exactly match what the
Boot ROM expects will fail HAB verification with an event log the Boot
ROM can report over serial (on an open/engineering unit) but a closed
unit gives you nothing but silence:

```console
=> hab_status
--------- HAB Configuration ---------
config: 0xf0 (closed)
--------- HAB Event Log ---------
event 0: STATUS: 0x33 (FAIL) ENG 0x00 (ENGINE ANY)
  0xdb 0x00 0x08 0x42 0x33 0x0c 0x02
```

`hab_status` is only available on **open** or **engineering-closed**
units with debug enabled — a fully closed production unit that fails HAB
verification simply refuses to boot, with no diagnostic output at all.
This is exactly why the "practice extensively on open-mode hardware
first" discipline above is not optional: it's the only place you get to
see *why* something failed before you lose that visibility permanently.

## AHAB: the i.MX8/9 evolution

AHAB (used on i.MX8QM/QXP, i.MX9) delegates verification to the SECO/EdgeLock
secure enclave rather than the applications-core Boot ROM directly, and
adds container-based image signing (a single signed container can wrap
multiple images — SPL, DDR training firmware, ATF, OP-TEE, U-Boot — with
one verification pass):

```console
$ cst --i csf_ahab.txt --o flash.bin_signed
$ ahab_log  # from U-Boot console, i.MX9 style
SECO Event 0x00000000: 0x1234abcd, ind: 0x00, sts: 0xd6 (success)
```

The container model reduces the number of independent signing steps
versus HAB's stage-by-stage approach, but it also means a single
misconfigured container (wrong image offset, wrong core ID tag) can
prevent every image inside it from booting, not just one stage — test the
whole assembled container, not each image independently.

## dm-verity: extending the chain into the rootfs

Secure boot through the kernel doesn't protect the root filesystem after
that point unless you extend it. `dm-verity` provides read-only,
cryptographically verified block-level integrity for the rootfs, checked
continuously as blocks are read, not just once at boot:

```console
$ veritysetup format /dev/mmcblk0p3 /dev/mmcblk0p4
VERITY header information for /dev/mmcblk0p3
Root hash:      3f2a...e01c
$ cat rootfs.env
dm-mod.create="vroot,,,ro,0 2097152 verity 1 /dev/mmcblk0p3 /dev/mmcblk0p4 4096 4096 262144 1 sha256 3f2a...e01c"
```

The root hash is the value that must itself be embedded in something the
earlier signed stages verify (a signed kernel command line, or compiled
into the signed kernel/initramfs) — if the root hash is stored somewhere
mutable and unverified, an attacker can simply substitute both a modified
rootfs and a matching modified root hash, and the chain of trust has a
gap exactly at that handoff.

## Traps

- **Burning SRK fuses before the full chain (bootloader through
  dm-verity root hash) has been validated end-to-end on unfused
  hardware.** This is the single highest-consequence mistake in this
  entire curriculum — irreversible, per-unit, and unrecoverable.
- **Field-updating a signed bootloader without a fallback slot.** A
  signature-checking failure on a *sole* boot partition after a botched
  OTA leaves a unit that will never boot again; Module 3's A/B scheme
  exists specifically to make this recoverable.
- **Assuming HAB "closed" and "open" behave identically for debugging.**
  Every diagnostic you rely on during bring-up (`hab_status`, verbose
  Boot ROM serial output) disappears the moment a unit ships closed —
  validate your recovery/diagnostic strategy *before* closing production
  units, not after the first field failure with no serial output at all.

## Cheat sheet

| Concept | HAB (i.MX6/7/8) | AHAB (i.MX8/9) |
|---|---|---|
| Root of trust | SRK hash in Boot ROM-read fuses | SRK hash, verified by SECO/EdgeLock enclave |
| Signing tool | `cst` + CSF descriptor | `cst` + container-based CSF |
| Runtime status | `hab_status` (open/eng units only) | `ahab_log` |
| Multi-image signing | Per-stage, sequential | Single signed container, multiple images |
| rootfs integrity | Not covered — needs `dm-verity` | Not covered — needs `dm-verity` |

!!! note "On verification"
    The HAB/AHAB chain-of-trust model, CST signing workflow, and
    dm-verity integration pattern are checked against NXP's documented
    HAB/AHAB architecture and the dm-verity kernel documentation; no keys
    were generated, no image was signed, and — critically — no fuses were
    burned as part of writing this module. Treat every command here as
    reviewed syntax to validate on disposable hardware, never as a
    tested, ready-to-fuse procedure.

## Exercise

(1) Write out, in order, every validation step you would perform on
open-mode hardware before considering SRK fuse burning on a production
unit — be exhaustive; this list is the actual deliverable, not a
one-liner. (2) Given a `hab_status` event log showing `STATUS: 0x33
(FAIL)`, describe the categories of root cause you'd check first (IVT
offset, CSF pointer, wrong key) and how you'd narrow it down using only
open-mode diagnostics. (3) One paragraph: explain why storing a
dm-verity root hash in an unsigned, mutable location defeats the purpose
of the entire signed boot chain above it, even though every earlier stage
verified correctly.
