# Glossary

Terms you will meet in the CrownOS codebase, and what they mean here.

---

## CrownOS names

**crowncrate** — The phone-bridge subsystem: clipboard sync, notification sync,
call bridging, OTP sync, second screen. Three repos —
[`crowncrate-linux`](../30-components/crowncrate-linux.md) (daemon),
[`crowncrate-android`](../30-components/crowncrate-android.md),
[`crowncrate-chrome`](../30-components/crowncrate-chrome.md). Not a package
manager, not a container runtime.

**crownpositor** — The Wayland compositor. Note the website calls it "Hyprcrown";
that name appears nowhere in the code.

**crownshell** — The layer-shell framework, not a command shell. It is a Rust
library for building desktop surfaces with Vello.

**lls** — Not expanded anywhere in the source. Almost certainly **Low-Latency
Streaming**, based on the module names (`streaming`, `signaling`, `nvenc`) and
the product features it lines up with. *(Inference.)*

**launchpad** — The application launcher UI, referenced in `keybinds.ron`'s doc
comments. Its implementation is
[`crownlauncher`](../30-components/crownlauncher.md), currently a stub.

**section** — One settings file. `appearance.ron` is the `appearance` section.
The settings-menu page name and the file name are deliberately the same string.

**predule** — `crownshell`'s prelude module. The typo is in the public API and
every downstream crate uses it. Not `prelude`.

---

## Wayland

**Wayland** — The display server protocol that replaces X11. A *compositor* is
both window manager and display server; clients render their own windows and hand
buffers over.

**compositor** — In Wayland terms, the server. `crownpositor` is one.

**wlr-layer-shell** — A protocol for surfaces that are part of the desktop rather
than application windows: bars, docks, wallpapers, notification toasts, lock
screens. Originated in wlroots and is supported by most non-GNOME compositors.
Every CrownOS shell component is a layer-shell client.

**layer** — Where a layer surface sits in the stack: `Background`, `Bottom`,
`Top`, `Overlay`, in that order.

**anchor** — Which screen edges a layer surface sticks to. Anchoring to opposite
edges stretches it.

**exclusive zone** — Pixels a layer surface reserves so tiled windows do not
overlap it. `crownbar` reserves 40; `crowndock` reserves 0 because it auto-hides.

**input region** — The part of a surface that receives pointer events. Setting it
empty makes a surface click-through — `crowndictator`'s waveform overlay does
this permanently, and `crowndock` shrinks its region to a 1-pixel strip while
hidden.

**xdg-shell** — The protocol ordinary application windows use.

**XWayland** — Compatibility layer running X11 clients under a Wayland
compositor.

**ext-background-effect-v1** — A staging protocol for compositor-side background
blur. `crownshell` binds it; `crownpositor` does not implement it, so blur
silently does not happen.

**wp_fractional_scale_v1** — The protocol for true fractional scaling.
`crownshell` does not bind it, so only integer buffer scales are supported.

**seat** — A collection of input devices (keyboard, pointer, touch) belonging to
one user.

**libseat / seatd** — Manages access to seats and DRM devices without root.
Needed by `crownpositor`'s KMS backend.

---

## Graphics

**Smithay** — The Rust Wayland compositor library `crownpositor` is built on.

**Vello** — A GPU compute-centric 2D renderer from Linebender. Paths, gradients,
blurs, images and glyphs, all encoded into a `Scene` and rasterised on the GPU.
CrownOS paints everything with it.

**Scene** — Vello's display list. You push draw commands into it in `paint`, and
Vello rasterises it.

**Parley** — Linebender's text layout library — shaping, line breaking,
measurement. Sits on top of the system font database.

**wgpu** — The portable GPU abstraction Vello renders through, targeting Vulkan
or GL here.

**xilem** — Linebender's reactive UI framework. `crownuikit` is built on it, and
it is a deliberately different stack from `crownshell`.

**Masonry** — The widget layer beneath xilem.

**DRM/KMS** — Direct Rendering Manager / Kernel Mode Setting. How a compositor
drives real displays from a TTY.

**GBM** — Generic Buffer Management. Allocates buffers that can be scanned out.

**EGL / GLES** — The default rendering path (`CROWN_RENDER_API=egl`).

**libinput** — The library that turns kernel input events into usable pointer,
keyboard and gesture events.

**evdev** — The kernel's raw input event interface, `/dev/input/event*`.
`crowndictator` reads it directly, bypassing the compositor.

**pixman** — Software pixel manipulation, used as a fallback renderer.

**libdisplay-info** — EDID parsing. Deliberately excluded from `crownpositor`
because the sys crate does not accept current system versions.

---

## Configuration

**RON** — [Rusty Object Notation](https://github.com/ron-rs/ron). Rust's own
serialisation format; maps cleanly onto enums and tuples. CrownOS's config
format. The website's claim of "TOML profiles" is wrong.

**implicit Some** — A RON extension CrownOS enables so optional fields are
written `network: "home"` rather than `network: Some("home")`.

**echo suppression** — `crownos-config`'s mechanism for stopping an app receiving
its own write back as an external change. `save()` records a hash; the watcher
drops events matching it.

**inotify** — The Linux filesystem-change notification API, used through the
`notify` crate, that makes live config reload work.

**XDG base directories** — The freedesktop spec for where config, cache and data
live. `~/.config` is `XDG_CONFIG_HOME`.

---

## Layout

**tiling** — Windows are arranged automatically to fill the screen without
overlapping.

**master-stack** — The dwm/xmonad model: one large master area plus a stack of
the rest. CrownOS's default.

**scrolling columns** — The niri/PaperWM model: an infinite horizontal ribbon of
columns you scroll through.

**floating** — Nothing is tiled; every window keeps its own rectangle.

**tile** — In `crownpositor`'s model, one window's slot. Monitors own workspaces,
workspaces own tiles.

**gaps** — Pixels between tiled windows (`gaps_inner`) and between the tiled area
and the screen edge (`gaps_outer`).

**window rule** — A pattern matched against `app_id` or `title` at a window's
first buffer commit, applying overrides such as floating or a target workspace.

---

## Other

**archiso** — Arch Linux's ISO build tooling. `crownos-iso` is an archiso
*profile*; `mkarchiso` is the tool that consumes it.

**releng** — The upstream Arch profile that `crownos-iso` is currently an
unmodified copy of. It builds a rescue/installer image.

**airootfs** — In an archiso profile, the directory tree overlaid onto the live
image's root filesystem.

**CBOR** — Concise Binary Object Representation. The wire format `crowncrate`
actually uses, despite its README saying JSON.

**RTP** — Real-time Transport Protocol, RFC 3550. `lls-protocol`'s packet header
copies its fields — sequence number, timestamp, marker bit, SSRC.

**SSRC** — In RTP, a 32-bit identifier for the origin of a media stream.

**ONNX Runtime** — The inference engine `crowndictator` runs its speech model on,
via the `ort` crate.

**Parakeet TDT** — The NVIDIA speech recognition model `crowndictator` uses,
downloaded from Hugging Face on first run.

**spring** — The animation primitive used throughout. Critically damped,
integrated at a fixed 1/240 s substep. There are five separate implementations of
it across the project.

**calloop** — The event loop `crownshell` and `crownpositor` both run on.

**zbus** — The Rust D-Bus library `crownotify` uses.

**Biome** — The linter and formatter for `crownos-website`. Replaces both ESLint
and Prettier.
