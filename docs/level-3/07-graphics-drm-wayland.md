# 07 · Graphics — DRM/KMS & Wayland

Fbdev is gone from any modern embedded product with a display. The kernel
side is **DRM/KMS** (Direct Rendering Manager / Kernel Mode Setting); the
userspace side that composites and hands buffers to it is almost always a
**Wayland** compositor now, not X11. This module is about the plumbing
between a GPU driver and a pixel actually appearing on a panel — where
things go wrong, they go wrong at that boundary.

## The DRM/KMS object model

Four objects, always in this relationship:

```
CRTC ──drives──▶ Encoder ──drives──▶ Connector ──▶ physical output
 │
 uses a
 │
Plane (framebuffer source: primary / overlay / cursor)
```

- **Connector**: the physical thing (HDMI port, MIPI-DSI panel, eDP).
  Carries EDID and hotplug status.
- **Encoder**: converts a CRTC's pixel stream to the connector's signal
  format (TMDS for HDMI, DSI lanes for a panel).
- **CRTC**: scans out a framebuffer at a mode (resolution/refresh),
  reading from one or more planes.
- **Plane**: an actual buffer source — primary (fullscreen), overlay
  (e.g. video), cursor.

Inspect the live topology with `modetest` (from `libdrm-tests`) before
touching any code:

```console
$ modetest -M imx-drm -c
Connectors:
id      encoder status          name            size (mm)       modes   encoders
34      33      connected       DSI-1           154x86          1       33
  modes:
        name refresh (Hz) hdisp hss hse htot vdisp vss vse vtot)
  1920x1080  60.00 1920 1968 2000 2200 1080 1083 1088 1125 flags: nhsync, nvsync; type: preferred, driver
```

`status disconnected` on a panel physically attached almost always means
either the DSI lane count/timing in the device tree doesn't match the
panel's actual EDID/datasheet timings, or the panel driver never bound —
check `dmesg` for the panel's `compatible` string before assuming a wiring
fault.

## The panel device tree node

```dts
&mipi_dsi {
	status = "okay";

	panel@0 {
		compatible = "acme,dsi-panel-1080p";
		reg = <0>;
		reset-gpios = <&gpio1 5 GPIO_ACTIVE_LOW>;
		port {
			panel_in: endpoint {
				remote-endpoint = <&dsi_out>;
			};
		};
	};
};
```

The `port`/`endpoint`/`remote-endpoint` pattern — the DT **graph binding**
— is how DRM components describe their pixel-data connections separately
from the bus they're probed on; a mismatched or missing
`remote-endpoint` phandle is a very common reason a panel driver probes
successfully (its `compatible` matched) but never gets linked into the
DRM pipeline, so nothing ever lights up despite no error anywhere in
`dmesg`.

## KMS atomic modesetting

Legacy KMS set one property at a time, with intermediate invalid states
possible. Atomic KMS validates an entire desired state — CRTC mode, plane
buffer, connector — as one transaction, either fully applied or rejected:

```c
drmModeAtomicReq *req = drmModeAtomicAlloc();

drmModeAtomicAddProperty(req, plane_id, prop_fb_id, fb_id);
drmModeAtomicAddProperty(req, plane_id, prop_crtc_id, crtc_id);
drmModeAtomicAddProperty(req, plane_id, prop_crtc_x, 0);
drmModeAtomicAddProperty(req, plane_id, prop_crtc_y, 0);

int ret = drmModeAtomicCommit(fd, req, DRM_MODE_ATOMIC_ALLOW_MODESET, NULL);
if (ret)
	fprintf(stderr, "atomic commit failed: %s\n", strerror(-ret));

drmModeAtomicFree(req);
```

A rejected atomic commit returns a clean errno rather than leaving the
display in a half-configured state — this is the entire point over legacy
KMS, where a failed intermediate `drmModeSetCrtc` call could leave a mode
set but no framebuffer bound, producing a black or garbage screen with no
single failing call to point at.

## Wayland: who owns the screen now

X11 let any client draw anywhere and read any other client's pixels — a
security and architecture mismatch for embedded kiosk/HMI products.
Wayland's model: the **compositor** is the only DRM/KMS client; every
application is a Wayland client that hands the compositor buffers to
composite, never touching KMS directly.

```
┌────────────┐   wl_surface + buffer    ┌───────────────┐   DRM/KMS
│ App (client)│ ───────────────────────▶│  Compositor    │──atomic──▶ display
│ (e.g. Qt,   │◀── input events ─────────│ (weston, etc.) │  commit
│  GTK app)   │                          └───────────────┘
└────────────┘
```

For embedded single-app kiosk products, a minimal compositor (`weston`
with a single fullscreen client, or a custom libweston-based shell) is
common — running a full desktop compositor stack on a resource-constrained
SoC is usually unnecessary weight.

```console
$ weston --backend=drm-backend.so --tty=1 &
$ WAYLAND_DISPLAY=wayland-1 my-kiosk-app
```

**Trap**: two processes racing to become the DRM master (`drmSetMaster`)
— usually a stray X server or a debug tool like `modetest` left running
— causes the compositor's own `drmModeSetCrtc`/atomic calls to fail with
`EACCES` or `EBUSY`, and the resulting symptom ("compositor won't start,
no clear error") gets misdiagnosed as a driver bug far more often than
the actual cause (something else holding DRM master).

## Tearing, vsync, and the buffer handoff

A client that renders and swaps buffers faster than the display's refresh
without waiting for the compositor's frame callback either tears (if
directly scanning out) or simply wastes power rendering frames that never
display (composited case). The correct client-side pattern under Wayland
is to render only after the compositor's `frame` callback fires:

```c
static void frame_done(void *data, struct wl_callback *cb, uint32_t time)
{
	wl_callback_destroy(cb);
	render_next_frame();
	struct wl_callback *next = wl_surface_frame(surface);
	wl_callback_add_listener(next, &frame_listener, NULL);
	wl_surface_commit(surface);
}
```

Rendering on a free-running timer instead of this callback is a common
cause of "smooth in isolation, stutters once a second client is running"
— you're now competing for compositor time with no back-pressure signal.

## Traps

- **DSI lane/timing mismatch** between the device tree and the panel
  datasheet — the panel may appear to init (backlight comes on) while
  showing garbage or a shifted image, because timing is close enough to
  clock but not to align correctly.
- **Overlay planes with format/scaling limits the SoC's blitter can't
  meet** — requesting a plane format or scale factor the hardware doesn't
  support silently falls back to a slow software composite path on some
  drivers, showing up only as "why is the CPU pegged during video
  playback," never as an explicit error.
- **Holding DRM master in a debug tool** left running after use —
  `modetest`, `kmscube`, or a stray X server holds it until killed, and
  the compositor's failure to acquire master looks exactly like a broken
  driver.

## Cheat sheet

| Concept | Purpose |
|---|---|
| Connector / Encoder / CRTC / Plane | DRM/KMS object hierarchy |
| `modetest -M <driver> -c` | Inspect live connector/mode topology |
| DT `port`/`endpoint`/`remote-endpoint` | Graph binding linking DRM components |
| `drmModeAtomicCommit` | Validate-then-apply an entire display state at once |
| `wl_surface_frame` callback | Correct pacing signal for a Wayland client |
| `drmSetMaster`/`EBUSY` | Symptom of two processes fighting over DRM master |

!!! note "On verification"
    DRM/KMS object model, atomic modesetting semantics, and the Wayland
    frame-callback pattern were checked against documented DRM/libdrm and
    Wayland protocol behavior; no GPU driver or display pipeline was
    exercised on this machine — `modetest`/`drmModeAtomicCommit` output
    shown is representative, not captured from real hardware.

## Exercise

(1) Given a panel that shows a correctly-timed but half-width, doubled
image, diagnose which DT property (lane count vs pixel clock vs timing
values) is the likely cause and explain your reasoning. (2) Write the
atomic-commit property list needed to move a fullscreen primary plane's
framebuffer without touching the CRTC mode, and explain why atomic commit
makes this safer than the legacy `drmModeSetCrtc` equivalent. (3) One
paragraph: your kiosk app renders on a free-running 60 Hz timer instead of
the compositor's frame callback, and testers report occasional stutter
only when a second background client starts. Explain the mechanism and
the fix.
