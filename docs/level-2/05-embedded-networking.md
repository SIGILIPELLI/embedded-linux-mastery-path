# 05 · Embedded Networking

A desktop distribution hides networking behind NetworkManager and a GUI.
An embedded product usually has no GUI, no user to click "connect", and a
requirement that the link comes back by itself at 3 a.m. after the site's
switch reboots. This module covers the layer you actually configure —
`ip`, `systemd-networkd`, DHCP, DNS, wpa_supplicant and nftables — and the
failure modes that only show up in the field.

## The one command that replaced five

`ifconfig`, `route`, `arp` and `netstat` are deprecated. Everything is now
`ip` (from iproute2) and `ss`:

```console
root@target:~# ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP
    link/ether 00:04:9f:06:1b:2c brd ff:ff:ff:ff:ff:ff

root@target:~# ip addr add 192.168.7.20/24 dev eth0
root@target:~# ip link set eth0 up
root@target:~# ip route add default via 192.168.7.1
root@target:~# ip -brief addr
lo    UNKNOWN  127.0.0.1/8
eth0  UP       192.168.7.20/24
```

Read the flags carefully — they answer different questions:

- **`UP`** means *you* administratively enabled the interface.
- **`LOWER_UP`** means the driver sees **carrier**: a cable is plugged in
  and the PHY has link. No `LOWER_UP` is a hardware or cable problem, and
  no amount of IP configuration will fix it.
- **`NO-CARRIER`** appearing in the flags is the same message, stated
  explicitly.

```console
root@target:~# ip -s link show eth0
    RX: bytes  packets  errors  dropped  missed   mcast
    18829301   41208    0       0        0        1129
    TX: bytes  packets  errors  dropped  carrier  collsns
    2910488    18822    0       0        0        0
$ cat /sys/class/net/eth0/carrier
1
```

Non-zero `errors` or `carrier` counters on a board that "works fine" are a
signal of a marginal PHY, bad magnetics or a RGMII clock-delay problem —
classic hardware bring-up findings.

## Static configuration with systemd-networkd

For a systemd image, `systemd-networkd` is the right choice: small, no
D-Bus dependency for basic use, declarative files, and it handles
carrier-loss recovery for you. Files live in `/etc/systemd/network/` and
are applied in lexical order; the first `[Match]` that matches wins.

```ini
# /etc/systemd/network/10-eth0-static.network
[Match]
Name=eth0

[Network]
Address=192.168.7.20/24
Gateway=192.168.7.1
DNS=192.168.7.1
DNS=9.9.9.9

[Link]
RequiredForOnline=yes
```

DHCP instead:

```ini
# /etc/systemd/network/20-eth0-dhcp.network
[Match]
Name=eth0

[Network]
DHCP=ipv4

[DHCPv4]
UseDNS=yes
UseNTP=yes
RouteMetric=100
SendHostname=yes
```

```console
root@target:~# systemctl enable --now systemd-networkd
root@target:~# networkctl status eth0
● 2: eth0
             Link File: /lib/systemd/network/99-default.link
          Network File: /etc/systemd/network/20-eth0-dhcp.network
                  Type: ether
                 State: routable (configured)
                Online: online
            HW Address: 00:04:9f:06:1b:2c
               Address: 192.168.7.20 (DHCP4 via 192.168.7.1)
               Gateway: 192.168.7.1
                   DNS: 192.168.7.1
```

`networkctl status` is the first thing to run on any networking complaint.
`State: configuring` means DHCP has not completed; `State: off` means no
`.network` file matched at all.

## DNS

Name resolution is a separate subsystem from addressing, and confusing the
two wastes a lot of debugging time. `systemd-resolved` manages
`/etc/resolv.conf` — usually as a symlink into `/run`:

```console
root@target:~# ls -l /etc/resolv.conf
lrwxrwxrwx 1 root root 39 Aug  4 09:02 /etc/resolv.conf -> ../run/systemd/resolve/stub-resolv.conf
root@target:~# resolvectl status
Link 2 (eth0)
  Current Scopes: DNS
       Protocols: +DefaultRoute
Current DNS Server: 192.168.7.1
       DNS Servers: 192.168.7.1 9.9.9.9
```

On a read-only rootfs (module 8) that symlink is mandatory — a real
`/etc/resolv.conf` file cannot be rewritten when `/` is mounted read-only,
so DNS silently keeps stale servers forever.

Diagnose in the right order: `ip route get 8.8.8.8` (routing), then
`ping 8.8.8.8` (connectivity), then `resolvectl query example.com` (DNS).
If ping-by-IP works and by-name does not, stop looking at the network.

## WiFi with wpa_supplicant

```console
root@target:~# wpa_passphrase FactoryNet 'correct-horse-battery' >> \
    /etc/wpa_supplicant/wpa_supplicant-wlan0.conf
```

```text
# /etc/wpa_supplicant/wpa_supplicant-wlan0.conf
ctrl_interface=/run/wpa_supplicant
update_config=1
country=DE

network={
	ssid="FactoryNet"
	# psk="correct-horse-battery"
	psk=6ae0f1e02b6d0e5c11f42d3f9b0a1d7c6f5e4a39821b0c7d6e5f4a3928170615
}
```

`wpa_passphrase` writes both the plaintext (commented) and the derived PSK.
**Delete the commented plaintext line before shipping** — it is a
cleartext credential in your image.

```console
root@target:~# systemctl enable --now wpa_supplicant@wlan0
root@target:~# wpa_cli -i wlan0 status
wpa_state=COMPLETED
ssid=FactoryNet
ip_address=192.168.7.53
root@target:~# iw dev wlan0 link
	signal: -62 dBm
	rx bitrate: 65.0 MBit/s
```

The `@wlan0` instance name is what makes systemd look for
`wpa_supplicant-wlan0.conf`; a mismatch between the two is the usual reason
the service starts and immediately exits.

## Firewalling with nftables

`iptables` is legacy; `nft` is the current interface. A minimal
default-deny ruleset for an appliance:

```text
# /etc/nftables.conf
#!/usr/sbin/nft -f
flush ruleset

table inet filter {
	chain input {
		type filter hook input priority filter; policy drop;

		ct state established,related accept
		ct state invalid drop
		iif lo accept

		ip protocol icmp accept
		ip6 nexthdr ipv6-icmp accept

		tcp dport 22 accept comment "ssh"
		tcp dport 8080 accept comment "device web UI"
	}

	chain forward { type filter hook forward priority filter; policy drop; }
	chain output  { type filter hook output  priority filter; policy accept; }
}
```

```console
root@target:~# nft -c -f /etc/nftables.conf     # check syntax, apply nothing
root@target:~# systemctl enable --now nftables
root@target:~# nft list ruleset | head -8
root@target:~# ss -tlnp
State   Local Address:Port   Process
LISTEN  0.0.0.0:22           users:(("dropbear",pid=201,fd=3))
LISTEN  0.0.0.0:8080         users:(("appd",pid=310,fd=6))
```

Note `ct state established,related accept` sitting *first*. Without it, a
default-drop input policy blocks the replies to your own outbound
connections and the board appears to have no network at all.

## Traps

!!! danger "Networking traps that only bite in the field"
    - **Locking yourself out.** Applying a default-drop ruleset over SSH on a
      remote board with no serial console is unrecoverable without a site
      visit. Always stage it behind a timer that reverts after 5 minutes until
      you have confirmed access, and always `nft -c -f` first.
    - **MAC address from a random pool.** Many SoCs, i.MX included, fall back
      to a random MAC when the fuse or EEPROM is unprogrammed. The board gets a
      new DHCP lease on every boot, ARP caches go stale, and any
      MAC-based licensing breaks. Check `ip link` for the same MAC across two
      boots during bring-up.
    - **`eth0` is not guaranteed.** Predictable interface names (`enp1s0`,
      `end0`) depend on udev rules and kernel version; a BSP upgrade can rename
      your interface and orphan every config file. Match on MAC or path in
      `[Match]` when it must be stable.
    - **DHCP at boot as a hard dependency.** `RequiredForOnline=yes` plus a
      service ordered after `network-online.target` means a board with an
      unplugged cable hangs for the DHCP timeout on every boot. Decide
      deliberately whether the product should boot without a network.
    - **Time before TLS.** With no RTC, the clock starts at the epoch and every
      certificate looks "not yet valid". NTP needs the network, the network's
      TLS needs the time. Ship a build-time fallback timestamp
      (`systemd-timesyncd` maintains one) or an RTC.
    - **`/etc/resolv.conf` as a regular file on a read-only rootfs** — DNS
      updates are silently discarded.

## Cheat sheet

| Command / item | Purpose |
|---|---|
| `ip -brief addr` / `ip link` | Addresses / interfaces and carrier state |
| `LOWER_UP` vs `UP` | Carrier present vs administratively enabled |
| `ip route get 8.8.8.8` | Which route and source IP would be used |
| `ip -s link show eth0` | Error/drop/carrier counters — PHY health |
| `ss -tlnp` / `ss -tunap` | Listening sockets / all sockets with processes |
| `/etc/systemd/network/*.network` | Declarative address, DHCP, DNS, routes |
| `networkctl status <iface>` | Which file matched and what state it reached |
| `resolvectl status` / `resolvectl query <name>` | DNS servers / test resolution |
| `wpa_passphrase <ssid> <pass>` | Generate a PSK block (strip the plaintext!) |
| `wpa_supplicant@wlan0.service` | Instanced unit → `wpa_supplicant-wlan0.conf` |
| `wpa_cli -i wlan0 status` / `iw dev wlan0 link` | Association state / signal |
| `nft -c -f /etc/nftables.conf` | Syntax-check a ruleset without applying it |
| `nft list ruleset` | Dump the live ruleset |
| `ct state established,related accept` | The rule that must come first |
| `network-online.target` | Ordering target for "needs a working network" |

!!! note "On verification"
    The `.network`, `wpa_supplicant.conf` and `nftables.conf` syntax here
    follows the documented grammars for each, and the nftables ruleset is
    written so `nft -c -f` can check it before it is ever applied. Live
    hardware testing was not possible while writing this page — run the
    check-only commands on your own target first.

## Exercise

(1) In your QEMU image, bring `eth0` up entirely by hand with `ip` (address,
route, and a nameserver in `/etc/resolv.conf`), confirm with
`ip route get 8.8.8.8` and a ping, then tear it all down and reproduce the
same result declaratively with a single `.network` file. (2) Break DNS only
— keep addressing intact — and write down the exact sequence of commands
that lets you prove it is DNS and not routing. (3) Apply the nftables
ruleset above, verify with `ss -tlnp` that your service is still reachable,
then remove the `ct state established,related accept` line and describe
precisely what stops working and why. (4) One paragraph: a customer's
gateway ships with an unprogrammed MAC fuse and gets a new DHCP lease every
boot. Explain the three separate symptoms that will be reported by support
(intermittent connectivity, "the device changed IP", licence failures), and
where in the boot chain — fuse, U-Boot environment, device tree, or
`.link` file — you would fix it.
