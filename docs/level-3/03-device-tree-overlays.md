# 03 · Device Tree Overlays

Level 2 treated the device tree as a fixed description compiled once into
the boot image. In production that's rarely true: a carrier board vendor
adds a daughter card, a factory line needs to disable a sensor on units
missing that BOM option, or the same base image must support three board
revisions with different peripheral wiring. **Overlays** patch a base
device tree at boot or load time without recompiling or forking it.

## What an overlay actually is

A `.dtbo` is a compiled device tree fragment plus metadata describing
where it attaches — a *target* node in the base tree, referenced either
by full path or by phandle label (`&label`). The bootloader or kernel
applies the overlay by splicing the fragment's properties and child nodes
into the target.

```dts
// base.dts (excerpt)
/ {
	soc {
		i2c1: i2c@30a20000 {
			#address-cells = <1>;
			#size-cells = <0>;
			status = "okay";
		};
	};
};
```

```dts
// imu-overlay.dts
/dts-v1/;
/plugin/;

/ {
	compatible = "acme,mainboard";
};

&i2c1 {
	#address-cells = <1>;
	#size-cells = <0>;

	imu@68 {
		compatible = "bosch,bmi270";
		reg = <0x68>;
		interrupt-parent = <&gpio1>;
		interrupts = <14 IRQ_TYPE_EDGE_RISING>;
		status = "okay";
	};
};
```

`/plugin/;` is what marks this as an overlay rather than a full tree —
without it, `dtc` compiles it as a standalone (and broken) device tree
instead of a fragment. `&i2c1` is the target: the overlay's `imu@68` node
becomes a child of the base tree's real `i2c1` node after application.

## Compiling and inspecting

```console
$ dtc -@ -I dts -O dtb -o imu-overlay.dtbo imu-overlay.dts
$ fdtdump imu-overlay.dtbo | head -20
// magic:  0xd00dfeed
// totalsize: 0x2a4 (676)
...
/ {
    compatible = "acme,mainboard";
    fragment@0 {
        target = <0xffffffff>;
        __overlay__ {
            imu@68 { ... };
        };
    };
    __symbols__ {
        i2c1 = "/soc/i2c@30a20000";
    };
};
```

The `-@` flag is what generates the `__symbols__` table the target
resolver needs — omit it and label-based targeting (`&i2c1`) silently
fails to resolve at apply time, because the compiler had no symbol table
to record where `i2c1` actually points.

## Applying at U-Boot time (build-time-adjacent, most common in production)

```console
=> load mmc 0:1 ${loadaddr} base.dtb
=> load mmc 0:1 ${fdtoverlay_addr} imu-overlay.dtbo
=> fdt addr ${loadaddr}
=> fdt resize 8192
=> fdt apply ${fdtoverlay_addr}
=> bootz ${kerneladdr} - ${loadaddr}
```

`fdt resize` before `fdt apply` is not optional — the loaded base DTB has
exactly enough room for itself, and applying an overlay without growing
the buffer first corrupts memory past the end of the blob. This is one of
the most common "board hangs before console output" bugs reported against
overlay-based boot flows, and it looks nothing like a device tree problem
from the symptom alone.

## Applying via kernel configfs (runtime, for hot-pluggable expansion)

```console
$ mount -t configfs none /sys/kernel/config
$ mkdir /sys/kernel/config/device-tree/overlays/imu
$ cat imu-overlay.dtbo > /sys/kernel/config/device-tree/overlays/imu/dtbo
$ dmesg | tail -3
[   45.221009] OF: overlay: Overlay ID 0 applied
[   45.223441] bmi270 1-0068: chip id 0x24
$ rmdir /sys/kernel/config/device-tree/overlays/imu   # removes it
```

Removal only works cleanly if every driver bound to the overlay's nodes
properly implements `remove()` — a driver that leaks a `devm_*` resource
or holds a raw pointer past teardown will crash or leave the platform bus
in an inconsistent state on overlay removal. This is why Module 2's
emphasis on clean `probe`/`remove` symmetry matters even more once
overlays are in play: overlay removal is essentially forced unbind.

## Conflicts: two overlays targeting the same node

```console
$ echo overlay-a.dtbo > .../overlays/a/dtbo
$ echo overlay-b.dtbo > .../overlays/b/dtbo   # also touches i2c1@68
mkdir: cannot create directory '.../overlays/b': File exists
```

or, more insidiously, both apply successfully but the *second* overlay's
properties silently win for any property both define — there's no
"merge conflict" error for property collisions the way there is for two
overlays claiming the exact same unit-address. Debug by dumping the live
tree, not by re-reading the source overlays:

```console
$ cat /proc/device-tree/soc/i2c@30a20000/imu@68/compatible
bosch,bmi270
```

**Trap**: a factory-configurable overlay set (e.g. one overlay per BOM
option) that isn't tested pairwise will occasionally produce a board that
boots fine with any overlay alone but hangs or misconfigures a peripheral
when two specific overlays are combined — because both quietly touch a
shared pinmux node. Treat the applied result (`/proc/device-tree` or
`fdtdump` on the merged blob), not the individual `.dts` sources, as the
thing you test.

## `status = "disabled"` vs deleting a node

To turn a peripheral off for a board variant, prefer overriding `status`:

```dts
&imu68 {
	status = "disabled";
};
```

over `/delete-node/`, unless you specifically need the node gone from
`/proc/device-tree` entirely (e.g. it would otherwise claim a GPIO another
overlay needs). `status = "disabled"` keeps the node's phandle valid for
anything still referencing it elsewhere, which `/delete-node/` does not —
a stray reference to a deleted node's phandle produces an obscure
`fdt_node_offset_by_phandle` failure at apply time rather than a clear
error pointing at the real cause.

## Traps

- **Missing `#address-cells`/`#size-cells` re-declaration** in the overlay
  fragment when adding children under a bus node — DT does not inherit
  these across fragment boundaries the way it does within one static tree,
  and the omission produces child `reg` values interpreted with the wrong
  cell count silently.
- **Interrupt parent mismatches**: an overlay's `interrupt-parent` must
  resolve to a phandle that exists in the *base* tree at apply time; if
  the base tree's GPIO controller node label differs from what the
  overlay assumes (common across board revisions), the overlay applies
  without error but the interrupt line never fires.
- **Forgetting `fdt resize`** in U-Boot before `fdt apply` — see above;
  the failure mode (silent hang, corrupted memory) gives almost no signal
  about the actual cause.

## Cheat sheet

| Command | Purpose |
|---|---|
| `dtc -@ -I dts -O dtb -o x.dtbo x.dts` | Compile overlay with symbol table |
| `fdtdump x.dtbo` | Inspect a compiled overlay's fragments |
| `fdt resize <n>; fdt apply <addr>` | Apply overlay in U-Boot (resize first!) |
| `mount -t configfs none /sys/kernel/config` | Enable runtime overlay application |
| `.../overlays/<name>/dtbo` | Write compiled `.dtbo` here to apply at runtime |
| `cat /proc/device-tree/.../compatible` | Inspect the *merged*, live tree |
| `status = "disabled"` | Preferred way to turn off a node per board variant |

!!! note "On verification"
    Overlay fragment syntax, `/plugin/;`, and the configfs application
    interface were checked against the documented DT overlay ABI and
    dtc behavior; the U-Boot and configfs command sequences were reviewed
    for correctness but not executed on real hardware or QEMU here.

## Exercise

(1) Write a base tree with an unpopulated `spi1` node and an overlay that
adds a `spi-nor` flash child at chip-select 0, compile it with `dtc -@`,
and use `fdtdump` to confirm the `__symbols__` table resolves `spi1`. (2)
Reproduce the "forgot `fdt resize`" failure by applying an overlay in
U-Boot without resizing first (or by reasoning through the U-Boot source
if you don't have hardware to hang), and document the exact symptom. (3)
Design two overlays that both touch a shared pinmux node with conflicting
settings, apply them in each order via configfs, and explain from
`/proc/device-tree` which one won and why order — not overlay "priority"
— decided it.
