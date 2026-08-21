# crownpositor

**Status: Early** · Rust · default branch `main` ·
[repo](https://github.com/Crown-OS/crownpositor)

A tiling Wayland compositor built on [Smithay](https://github.com/Smithay/smithay).
The largest and most actively developed codebase in the organization, at roughly
11,000 lines.

`crownpositor` is not just a window manager — it *is* the CrownOS session. It
creates the Wayland socket, exports `WAYLAND_DISPLAY`, and spawns the rest of the
desktop itself.

---

## Layout

Cargo workspace with two members:

| Crate | Kind | Role |
|---|---|---|
| `compositor` | bin + lib | Everything. `main.rs` is three lines calling `compositor::run()`. |
| `config` | lib | The compositor's *compiled* configuration — regexes, chords, geometry in signed pixels |

Note the second crate is named `config`, not `crownpositor-config`. It coexists
in the dependency tree with `crownos-config`, which is the shared on-disk schema.
They are different things.

### Modules

| Module | Role |
|---|---|
| `animations/` | Critically-damped spring integrator, fixed 1/240 s substep. Profiles: `SNAPPY`, `SMOOTH`, `GESTURE`, `TRACK`. |
| `backend/mod.rs` | `Preference::detect()` — winit vs KMS. Module doc is a four-step guide to adding a backend. |
| `backend/kms/` | DRM/KMS: device, surface, Vulkan |
| `backend/winit.rs` | Nested-session backend |
| `backend/render/` | `CrownRenderer` / `CrownAllocator` seams; EGL vs Vulkan allocator switch |
| `backend/frame_clock.rs` | Presentation timing |
| `handlers/` | One file per Wayland protocol delegate |
| `input/` | `keyboard`, `mouse`, `trackpad`, `shortcuts` — chord parsing and the single `Action` vocabulary |
| `layout/` | **Pure geometry.** `master_stack`, `scrolling`, `floating` |
| `rendering/` | `output_elements`, cursor (xcursor + builtin), `decorate`, `rounded` |
| `shaders/rounded_corner/` | GLSL fragment shader |
| `shell/` | The model: `monitor`, `workspace`, `tile`, `grab`, `transaction`, `workspace_switch` |
| `state/` | The `State` aggregate plus `common`, `backend`, `wayland`, `input`, `config`, `client`, `actions` |
| `utils/runtime.rs` | Tokio ↔ calloop bridge, with an ASCII diagram in the header |

### The layering rule

`layout/` is surface-blind by design. Its module doc:

> Nothing here can see a `WlSurface`, a `Window`, an `Output` or the `Shell`.

That is why the tiling algorithms are unit-testable, and it is worth preserving.
`shell/` owns the model — monitors own workspaces, workspaces own tiles, and
indices are the only route from a surface to anything else.

---

## Prerequisites

Everything in [Prerequisites](../10-getting-started/prerequisites.md#everything-any-crownshell-based-app),
plus `libdrm`, `libinput`, `libseat`, `libudev`, `pixman` and `xorg-xwayland`.

Smithay 0.7 is pulled with an explicit feature list: `backend_drm`,
`backend_gbm`, `backend_egl`, `backend_libinput`, `backend_session_libseat`,
`backend_udev`, `backend_winit`, `backend_vulkan`, `backend_x11`, `desktop`,
`renderer_glow`, `renderer_multi`, `renderer_pixman`, `wayland_frontend`,
`xwayland`.

> **`libdisplay-info` is deliberately excluded.** `smithay-drm-extras` is declared
> `default-features = false` because the `display-info` sys crate does not accept
> the version of `libdisplay-info` on current systems. Only `drm_scanner` is
> needed; EDID make/model naming returns when the sys crate catches up. Do not
> "fix" this by re-enabling the feature.

`crownos-config` must be a **flat sibling** — the manifest patches the git
dependency to `path = "../crownos-config"`. See
[Workspace setup](../10-getting-started/workspace-setup.md).

---

## Build and run

```bash
cd crownpositor
cargo build
cargo test        # 183 unit tests

# Nested, for development
CROWN_BACKEND=winit cargo run

# On real hardware, from a TTY, with seatd running
CROWN_BACKEND=kms cargo run --release
```

Backend selection: if `WAYLAND_DISPLAY` or `DISPLAY` is set it runs nested,
otherwise it takes the TTY. `CROWN_BACKEND` overrides — `winit`, or
`kms`/`drm`/`udev`. `CROWN_RENDER_API` picks `egl`/`gles`/`gles3` or
`vulkan`/`vk`, defaulting to EGL/GLES3. Unknown values log a warning and fall
back rather than failing.

`Super+Shift+E` quits. Have a second TTY available before running on hardware.

---

## How it fits into CrownOS

**It is the Wayland server.** Binds an auto-named socket via
`ListeningSocketSource::new_auto()` and exports it as `WAYLAND_DISPLAY`. Logs
`crownpositor is running`.

**It launches the desktop.** `run_startup()` reads `compositor.startup` and
spawns each entry — the doc comment names bar, wallpaper and notification daemon.
Spawning sets `WAYLAND_DISPLAY`, removes `DISPLAY`, and calls `setsid` in
`pre_exec`.

**It implements `wlr-layer-shell`**, which is how `crownbar`, `crowndock`,
`crownotify` and `crowndictator` attach. Two subtle fixes in that handler are
worth knowing about: `layer_destroyed` releases the exclusive zone (without it a
bar that exits reserves its strip forever), and `handle_layer_commit` unblocks
initial mapping (without it a bar maps and waits forever).

**It follows configuration live.** `state/config.rs` subscribes to every section
in `Config::sections()` — `compositor`, `appearance`, `display` — and posts to a
calloop channel, because watcher callbacks are `Send + Sync` and cannot touch
`&mut State`.

### One action vocabulary

Keyboard chords and trackpad gestures resolve to the same `Action` enum and go
down one dispatch path. From the source:

> No second enum for gestures — that is how "swipe left" and "Super+Tab" drift
> apart.

Full table: [Keybindings](../50-reference/keybindings.md). 32 default bindings,
4 default gestures.

### Environment variables

`CROWN_BACKEND` · `CROWN_RENDER_API` · `XCURSOR_THEME` · `XCURSOR_SIZE`, plus
`CROWN_CONFIG_DIR` indirectly through `crownos-config`. See
[Environment variables](../50-reference/environment-variables.md).

---

## Known limitations

**Blur is not implemented.** `handlers/background_effect.rs` is entirely
commented out — all 18 lines — and `background_effect_state` is commented out in
`state/wayland.rs`. `ext-background-effect-v1` is never advertised, so every
CrownOS surface that requests blur degrades silently.

**The workspace overview does not exist.** `shell/windows_view/mod.rs` and
`shell/workspaces_view/mod.rs` are zero-byte files. `MoveWorkspaceToOutput`,
`OpenWorkspaceView` and `CloseWorkspaceView` log `"action is not implemented
yet"` — and the four-finger trackpad gestures are bound to them.

**Schema skew with crownos-config.** `state/actions.rs` reads
`config.current.compositor.startup`, but the `Compositor` struct in
`crownos-config` has no `startup` field. The two checkouts do not compile
together as-is.

**Other open TODOs:**

- `shm_formats` is an empty `Vec`; the dmabuf global is never created
- `privileged_client_filter` returns `true` for every client — any client can
  bind privileged protocols
- Fractional scale sends the wrong scale
- Client-side decorations are not honoured
- Popups are not unconstrained
- Session lock confirms before all outputs are covered
- No pinch, hold, touch, tablet or output hotplug handling

---

## License

**No LICENSE file.** See
[Project status](../00-overview/project-status.md#licensing).
