# 08 · Security Hardening & CVE Management

Modules 1-7 built a chain of trust from boot through updates through
fleet identity. Hardening is the discipline of reducing everything that
chain doesn't already cover: the attack surface exposed by every running
service, every enabled kernel feature, every default credential, and
every dependency's undisclosed vulnerability. It's less glamorous than
secure boot and catches more real-world compromises.

## Attack surface reduction: the rootfs itself

The single highest-leverage hardening step is removing things, not
configuring them more carefully:

```console
$ opkg list-installed | wc -l
    284
$ opkg list-installed | grep -E "gcc|make|perl|python3-dev"
gcc
make
perl
python3-dev
```

A production image shipping a compiler is a production image that makes
a successful attacker's job dramatically easier — no compiler means an
attacker who gets code execution can't build exploit tooling in-place,
has to bring cross-compiled binaries, and every static-analysis and
supply-chain tool you'd want to point at "what's actually on this device"
has a much smaller surface to cover.

```console
# In your Yocto image recipe:
IMAGE_INSTALL:remove = "gcc make perl python3-dev"
IMAGE_FEATURES:remove = "debug-tweaks"
EXTRA_IMAGE_FEATURES = "read-only-rootfs"
```

`debug-tweaks` (passwordless root, auto-login, insecure defaults meant for
bring-up) shipping in a production image is one of the most common real
hardening failures found in the field — convenient during bring-up,
catastrophic if it survives into the release image, and it survives
exactly because nobody explicitly removed it once bring-up ended.

## Kernel hardening: config options that cost little and block a lot

```console
$ grep -E "CONFIG_STACKPROTECTOR_STRONG|CONFIG_FORTIFY_SOURCE|CONFIG_HARDENED_USERCOPY|CONFIG_RANDOMIZE_BASE" .config
CONFIG_STACKPROTECTOR_STRONG=y
CONFIG_FORTIFY_SOURCE=y
CONFIG_HARDENED_USERCOPY=y
CONFIG_RANDOMIZE_BASE=y
```

- `STACKPROTECTOR_STRONG`: canaries on stack buffers, catches stack
  overflow before it overwrites a return address.
- `FORTIFY_SOURCE`: compile-time and runtime bounds checking on common
  libc calls (`memcpy`, `strcpy`) where the size is statically knowable.
- `HARDENED_USERCOPY`: bounds-checks `copy_to_user`/`copy_from_user`
  (Module 1's primitives) against the actual size of the underlying
  kernel object, catching a whole class of driver bugs at runtime instead
  of letting them corrupt memory silently.
- `RANDOMIZE_BASE` (KASLR): randomizes kernel load address per boot,
  raising the cost of a return-oriented-programming exploit that needs a
  known kernel address to work.

None of these are exotic or expensive on modern embedded cores — the
performance cost is low single-digit percent at worst, and the "why
would I turn this off" bar should be high and specifically justified.

## Network exposure: audit what's actually listening

```console
$ ss -tulnp
Netid  State   Local Address:Port   Process
tcp    LISTEN  0.0.0.0:22           sshd
tcp    LISTEN  0.0.0.0:8080         product-app
tcp    LISTEN  0.0.0.0:23           telnetd
udp    LISTEN  0.0.0.0:69           in.tftpd
```

`telnetd` and unauthenticated `tftpd` listening on a product's default
network interface is an extremely common finding in real embedded
security audits — both were commonly enabled during bring-up (Level 1/2)
and never explicitly disabled for production. Every listening socket in
that output needs a specific justification for existing in the shipped
image; "it was there and worked" is not one.

```console
$ systemctl disable --now telnetd tftpd
$ ss -tulnp | grep -E "telnet|tftp"
(no output)
```

SSH itself needs hardening, not removal, for most products that need
remote access at all:

```console
$ grep -E "^PermitRootLogin|^PasswordAuthentication" /etc/ssh/sshd_config
PermitRootLogin no
PasswordAuthentication no
```

Key-based auth only, no root login — the same identity infrastructure
from Module 7's fleet provisioning (per-device keys) is the natural
mechanism for authorized SSH access, rather than a shared password or a
shared key across the whole fleet.

## Filesystem permissions and capability dropping

```console
$ ls -l /usr/bin/product-app
-rwsr-xr-x 1 root root  841232 product-app
```

A setuid binary running as root when it only needs one specific
capability (opening a privileged port, say) is a much larger attack
surface than the same binary run as a non-root user with exactly the
`CAP_NET_BIND_SERVICE` capability it needs:

```console
$ setcap 'cap_net_bind_service=+ep' /usr/bin/product-app
$ chmod -s /usr/bin/product-app
```

```ini
# systemd unit hardening
[Service]
User=product
DynamicUser=no
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
ReadWritePaths=/data/product-app
```

`ProtectSystem=strict` mounts the entire rootfs read-only for this
service specifically (composing with, but distinct from, a
system-wide read-only rootfs) — a service compromised through an
application bug still can't modify system binaries even if the attacker
achieves arbitrary file write within the service's own process, because
the mount namespace itself denies it.

## Fuzzing and static analysis: finding bugs before an attacker does

```console
$ afl-fuzz -i corpus/ -o findings/ -- ./parse_sensor_packet @@
american fuzzy lop ++4.09c
[+] All set and ready to roll!
...
  cycles done : 12         total execs : 84.2M
  saved crashes : 3         saved hangs : 1
```

Any code parsing untrusted input — a network protocol handler, a sensor
data frame parser, a file format your product ingests — is a
disproportionately high-value fuzzing target, because parsers written
against a "well-formed input" assumption are exactly where a hostile or
malformed input turns a functional bug into a security bug (a buffer
overread becoming an information leak, a heap overflow becoming code
execution).

```console
$ cppcheck --enable=warning,portability src/
$ clang-tidy src/parse_sensor_packet.c -checks='clang-analyzer-*,cert-*'
```

## The recurring CVE-triage obligation, applied here

Module 1 introduced `cve_check`; hardening is where you actually *act* on
what it reports, informed by attack surface:

```console
$ bitbake -c cve_check core-image-product
$ jq '.package[] | select(.issue[].status=="Unpatched") | .package' \
    tmp/deploy/images/*/core-image-product.rootfs.cve.json
"dropbear"
"busybox"
```

A CVE against a package still installed but never exposed to untrusted
input (a build-time-only tool accidentally left in the image, say) is a
lower-priority finding than the same severity CVE in something like
`dropbear`/`sshd` that's directly network-reachable — triage by
**reachability**, informed by the same `ss -tulnp` audit above, not by
CVSS score alone.

## Traps

- **`debug-tweaks` or equivalent bring-up conveniences surviving into
  production** — the single most common real hardening failure, and the
  easiest to prevent with an explicit release-image checklist.
- **Setuid-root binaries that only need one capability** — every setuid
  binary is a larger attack surface than the equivalent capability grant,
  and the difference in effort to fix is small.
- **CVE triage by severity score alone, ignoring reachability** — wastes
  effort patching unreachable findings while a lower-scored but
  network-exposed issue waits.
- **Fuzzing only the "interesting" parser and skipping the boring one** —
  the boring, rarely-touched parser (a legacy config file format, a
  vendor SDK's packet header) is disproportionately likely to be the one
  nobody has stress-tested at all.

## Cheat sheet

| Layer | Hardening action |
|---|---|
| rootfs contents | Remove compilers/debug tools; `IMAGE_FEATURES:remove = "debug-tweaks"` |
| Kernel config | `STACKPROTECTOR_STRONG`, `FORTIFY_SOURCE`, `HARDENED_USERCOPY`, KASLR |
| Network | Audit `ss -tulnp`; disable telnet/tftp; SSH key-only, no root login |
| Process privilege | `setcap` over setuid-root; systemd `ProtectSystem=strict`, `NoNewPrivileges` |
| Untrusted-input code | Fuzz parsers; static analysis (`cppcheck`, `clang-tidy`) |
| Ongoing | Triage CVEs by reachability, not score alone |

!!! note "On verification"
    Kernel hardening config options, systemd sandboxing directives, and
    the fuzzing/static-analysis workflow are checked against documented
    kernel, systemd, and AFL/clang-tidy conventions; no image was built,
    no fuzzing campaign was run, and no CVE scan was executed on this
    machine.

## Exercise

(1) Starting from the `ss -tulnp` output above, write the specific
systemd/opkg commands to eliminate the telnet and tftp exposure, and the
sshd_config changes to lock SSH to key-only, no-root-login. (2) Take
Module 1's `hello_chr` driver's `hc_ioctl` and identify the exact
attacker-controlled input surface (which parameter, from which syscall)
you would prioritize fuzzing first, and why. (3) One paragraph: your
`cve_check` report shows a critical CVE in a copy of `libpng` bundled
inside a vendor SDK blob nobody on the team can rebuild from source.
Describe your options and which you'd pick, given that "just patch it"
isn't available.
