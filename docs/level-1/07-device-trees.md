# 07 · Device Trees

How does the kernel know your board has a UART at address `0x9000000`? On a
PC it can *ask* — but on embedded ARM, nobody answers. The **device tree**
is the fix: a text file describing the hardware, compiled to a binary blob,
and handed to the kernel at boot (you saw U-Boot do exactly that in
module 3). Device trees are where beginners drown when bringing up real
boards, so this module builds the skill gently: why they exist, how the
syntax works, and how to read a *real* one — first from your QEMU machine,
then a glimpse of the i.MX95's, which describes hundreds of devices the
same way.

## Why x86 doesn't need this — and embedded ARM does

Some buses are **discoverable**: PCI and USB devices carry ID registers, so
the kernel can enumerate the bus, read IDs, and load matching drivers.
On x86 the story is even more complete — firmware hands the kernel **ACPI
tables** describing everything else. That's why one Ubuntu ISO boots on
every PC ever made.

An SoC's internal peripherals sit on simple **memory-mapped buses** with no
enumeration protocol: a UART at `0x44380000` is just registers at an
address. Probe blindly and you hang the chip. And since every ARM SoC —
every i.MX, every Snapdragon — wires different peripherals at different
addresses, the kernel *must be told* the layout. Historically this was
hardcoded C ("board files"); the kernel community replaced that with device
trees: **one kernel binary + a per-board data file.**

```text
DTS (source, human-readable)
  └─ dtc (device tree compiler) → DTB (binary blob)
        └─ U-Boot loads kernel + DTB → kernel parses DTB → drivers bind
```

## DTS syntax in ten lines

A device tree is nodes (devices) with properties (facts), nested like the
hardware:

```dts
/dts-v1/;

/ {                                  // root node
    compatible = "vendor,board";     // most important property in the file
    #address-cells = <2>;            // how many 32-bit cells form an address
    #size-cells = <2>;               // ...and a size, in child "reg" props

    uart0: serial@9000000 {          // label: name@unit-address
        compatible = "arm,pl011";    // which DRIVER binds to this node
        reg = <0x0 0x9000000 0x0 0x1000>;  // address 0x9000000, size 0x1000
        interrupts = <0 1 4>;        // interrupt specifier (SPI 1, level-high)
        status = "okay";             // "okay" = enabled, "disabled" = off
    };

    chosen {
        stdout-path = "/serial@9000000";   // boot console lives here
    };
};
```

The load-bearing ideas:

- **`compatible`** is the match-maker: the kernel finds a driver whose
  table lists `"arm,pl011"` and gives it this node. Bring-up debugging is
  half "why didn't anything bind to my compatible string?"
- **`reg`** gives register windows; how its numbers group into
  address/size pairs is set by the parent's `#address-cells`/`#size-cells`.
- **Labels** (`uart0:`) let other nodes reference this one as `&uart0` —
  the mechanism board files use to *override* SoC files.
- **`status`** lets an SoC's `.dtsi` describe every peripheral as
  `disabled`, with each board's `.dts` flipping on only the ones actually
  wired up.

That split is the standard layering: **`imx95.dtsi`** (the chip — every
peripheral, disabled) is `#include`d by **`imx95-<board>.dts`** (the board
— enables UART1, sets the eMMC, describes *this* product), compiled
together into one `.dtb` per board.

## Read a real one: your QEMU machine's tree

QEMU *generates* a device tree for the `virt` machine — dump and decompile
it. Install the device tree compiler on the host
(`sudo apt install device-tree-compiler` / `brew install dtc`):

```console
$ qemu-system-aarch64 -M virt,dumpdtb=virt.dtb -cpu cortex-a53 -m 512M
$ dtc -I dtb -O dts virt.dtb -o virt.dts
$ grep -A6 'pl011' virt.dts
        pl011@9000000 {
                clock-names = "uartclk", "apb_pclk";
                clocks = <0x8000 0x8000>;
                interrupts = <0x00 0x01 0x04>;
                reg = <0x00 0x9000000 0x00 0x1000>;
                compatible = "arm,pl011\0arm,primecell";
        };
```

There it is: the serial console you've been typing into since module 4 —
`/dev/ttyAMA0` exists *because* this node says a PL011 lives at
`0x9000000`. Cross-check from **inside** the running guest, where the
kernel re-exports its parsed tree at `/sys/firmware/devicetree/base`
(aliased as `/proc/device-tree`):

```console
localhost:~# ls /proc/device-tree/
#address-cells  chosen  cpus  intc@8000000  memory@40000000  pl011@9000000 ...
localhost:~# cat /proc/device-tree/pl011@9000000/compatible | tr '\0' ' '; echo
arm,pl011 arm,primecell
```

## The i.MX95's tree: the same thing, at production scale

The mainline kernel carries the i.MX95's device tree at
`arch/arm64/boot/dts/freescale/imx95.dtsi` — thousands of lines describing
six A55 CPU nodes, LPUARTs, CAN controllers, Ethernet, PCIe, the works,
each with exactly the properties you just learned. A representative slice
(simplified):

```dts
uart1: serial@44380000 {
    compatible = "fsl,imx95-lpuart", "fsl,imx7ulp-lpuart";
    reg = <0x44380000 0x1000>;
    interrupts = <GIC_SPI 19 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&scmi_clk IMX95_CLK_LPUART1>;
    status = "disabled";
};
```

Same grammar as QEMU's PL011 — different address, different `compatible`
(matching NXP's LPUART driver), and note the fallback list: if no
`imx95-lpuart` driver exists, the older `imx7ulp-lpuart` driver still
binds. A board `.dts` then contains just `&uart1 { status = "okay"; };`
plus pin muxing. When an i.MX95 board's console is dead at bring-up, the
checklist is literally this node: right address? right clocks? status
okay? — which is why device-tree literacy is a hiring signal in embedded.

## Cheat sheet

| Concept / command | Meaning |
|-------------------|---------|
| DTS → (dtc) → DTB | Source → compiler → blob the bootloader passes to the kernel |
| `compatible = "vendor,device"` | Binds a node to a driver — the key property |
| `reg = <addr size>` | Register window(s); grouping set by `#address-cells`/`#size-cells` |
| `interrupts`, `clocks` | Wiring facts a driver needs to operate the device |
| `status = "okay" / "disabled"` | Board switches peripherals on/off |
| `.dtsi` vs `.dts` | SoC include (all disabled) vs board file (enables + overrides via `&label`) |
| `chosen { stdout-path }` | Which device is the boot console |
| `qemu -M virt,dumpdtb=f.dtb` | Dump QEMU's generated tree |
| `dtc -I dtb -O dts f.dtb` | Decompile a blob back to source |
| `/proc/device-tree/` | The live tree inside a running system |

## Exercise

(1) From the decompiled `virt.dts`, answer: at what address does the
interrupt controller (`intc`) live, what does the `memory@40000000` node's
`reg` say — decode it into "RAM starts at X, size Y bytes" using the
address/size-cells rule — and what does the `chosen` node set? (2) Inside
the guest, verify the memory answer against `/proc/device-tree/memory@40000000/reg`
(`hexdump -C` it) and against `free -m`. (3) Boot QEMU with `-m 1024M`,
re-dump the DTB, and diff the memory node — you've just watched a
"bootloader" patch a device tree, which is exactly what U-Boot does to
inject boot-time facts on real boards. (4) On paper: your i.MX95 board's
console prints nothing, but the kernel boots (you can see it on a network
log). Name the two device-tree properties you'd check first on the UART
node, and the file (`.dtsi` or board `.dts`) where each most likely needs
fixing.
