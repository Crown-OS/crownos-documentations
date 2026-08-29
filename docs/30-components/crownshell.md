# crownshell

**Status: Early** · Rust · default branch `main` · version **0.2.0** ·
[repo](https://github.com/Crown-OS/crownshell)

A framework for building Wayland **layer shell** surfaces — bars, docks,
notification toasts, wallpapers. It abstracts the Wayland boilerplate; you
configure a surface and implement a paint callback.

Painting is done with [Vello](https://github.com/linebender/vello), so you get
GPU-accelerated 2D graphics with paths, gradients, blurs, images and text.

This is the foundation for `crownbar`, `crowndock`, `crownotify` and
`crowndictator`. It is a pure library — there is no binary.

---

## What it provides

- **Layer shell windows** via `wlr-layer-shell`: layer, anchor, size, exclusive
  zone, keyboard interactivity
- **Vello rendering** wired to the surface — you push into a `Scene`
- **Text** shaped and laid out by [Parley](https://github.com/linebender/parley)
  using system fonts, with measurement so you can align around it
- **Input** — pointer enter/leave/motion/press/release, and keyboard
- **Drag and drop** with mime-type negotiation and payload delivery
- **Background blur** through `ext-background-effect-v1` when the compositor
  supports it
- **HiDPI** — you draw in logical pixels, `crownshell` renders into a correctly
  sized physical buffer
- **Ticks and frame callbacks** for animation and periodic redraw
- **Multiple windows** in one app on a single `calloop` event loop

---

## Prerequisites

The base set in
[Prerequisites](../10-getting-started/prerequisites.md#everything-any-crownshell-based-app).
You need a working Vulkan or GL adapter — wgpu requires a real device.

To run it you need any compositor supporting `wlr-layer-shell`: `crownpositor`,
Hyprland, Sway, river, KWin, or most wlroots-based compositors.

Minimum Rust **1.88**, set by `vello 0.9` rather than by edition 2024.

---

## Build and run

```bash
cd crownshell
cargo build
cargo test                       # 42 unit tests

cargo run --example text_bar     # top bar with a label and a live clock
cargo run --example menu_popup   # bar plus an animated overlay popup
cargo run --example raw_wgpu     # one wallpaper surface per output
```

`menu_popup` is the best-documented code in the project. Read its module header
before writing a new surface.

---

## The API

```rust
use crownshell::predule::*;
use vello::kurbo::RoundedRect;
use vello::peniko::{Color, Fill};

struct Bar;

impl SurfaceHandler for Bar {
    fn paint(&mut self, scene: &mut Scene, ctx: SurfaceCtx<'_>) {
        let (w, h) = ctx.size;
        scene.fill(
            Fill::NonZero,
            Default::default(),
            Color::from_rgba8(20, 20, 30, 220),
            None,
            &RoundedRect::new(0.0, 0.0, w as f64, h as f64, 12.0),
        );
    }
}

fn main() -> Result<()> {
    run(|app| {
        app.create_window(
            WindowConfig {
                namespace: "example-bar".into(),
                layer: Layer::Top,
                anchor: Anchor::TOP | Anchor::LEFT | Anchor::RIGHT,
                size: (0, 40),
                exclusive_zone: 40,
                blur: true,
                ..Default::default()
            },
            Bar,
        );
        Ok(())
    })
}
```

> **The prelude module is spelled `predule`.** A typo in the public API, present
> since extraction, used by all four downstream crates. Fixing it is a breaking
> change; it stands for now.

| Type | Role |
|---|---|
| `App` | Top-level context, given to the `run` closure |
| `WindowConfig` | `namespace`, `layer`, `anchor`, `size`, `exclusive_zone`, `keyboard_interactivity`, `blur`, `auto_blur_region`, `tick_interval` |
| `SurfaceHandler` | The trait you implement. `paint` is the only required method; everything else defaults to a no-op. |
| `SurfaceCtx` | Logical `size`, output `scale`, shared `text` context, commit and region setters |
| `Text` / `TextStyle` | Retained, measurable text |
| `RawSurfaceHandler` | Lower-level escape hatch (see the `raw_wgpu` example) |

`crownshell` re-exports `calloop`, `parley`, `vello` and `wayland_client` whole,
plus `Alignment`, `OutputInfo`, `Anchor`, `KeyboardInteractivity` and `Layer` —
so downstream crates do not need matching versions in their own manifests.

**Event callbacks return `bool`** meaning "I need a redraw". Return `false` when
nothing visual changed; that is what keeps a bar from repainting on every pointer
motion. `on_frame` fires on the compositor frame clock (use it for animation);
`on_tick` fires on a timer, default 1 s (use it for clocks and battery readings).

Detail, including the popup pattern:
[The layer-shell stack](../20-architecture/layer-shell-stack.md).

---

## Source layout

| File | Lines | Role |
|---|---|---|
| `lib.rs` | 84 | Re-exports and the `run(setup)` entry point |
| `app.rs` | 393 | Registry, outputs, seat, layer shell, DnD state, output hooks |
| `window.rs` | 631 | `Window`, `WindowConfig`, per-surface dispatch, buffer scale |
| `handler.rs` | 410 | `SurfaceHandler`, `SurfaceCtx`, `KeyPress`, `PointerButton`, `ScrollDelta`, `DragOffer`, `DropPayload` |
| `renderer.rs` | 373 | Vello + wgpu surface from a raw `wl_surface` handle |
| `blur.rs` | 479 | Separable Gaussian post-process, two full-screen passes |
| `text.rs` | 1151 | `Text`, `TextStyle`, `TextContext` |
| `animations.rs` | 243 | `Spring`, `SpringProfile`, `Clock` |
| `predule.rs` | 14 | The (mis-spelled) prelude |
| `wayland/` | — | Per-protocol dispatch, including `background_effect` |

---

## How it fits into CrownOS

`crownshell` is purely a Wayland client library. It does **not** depend on
`crownos-config` — components read their own settings.

Consumed by `crownbar`, `crowndock`, `crownotify` and `crowndictator` as
`crownshell = "0.3"` from crates.io. No path or git dependencies remain.

---

## Known limitations

Stated in the project's own README: *"This is early crate… The API will move. If
you're going to use it, expect breaking changes."*

- **Text is single-line.** No wrapping, no alignment.
- **Integer buffer scales only.** On a fractionally-scaled output the compositor
  advertises the next integer up — a 1.25× display reports 2 — so text renders at
  2× and is scaled down. Sharp, but not pixel-exact. `wp_fractional_scale_v1` is
  not bound.
- **Blur does not work under `crownpositor`**, which never advertises
  `ext-background-effect-v1`. The degradation is silent and correct.
- **No tests of the Wayland layer** — the 42 unit tests cover `animations`,
  `blur`, `handler`, `text` and `wayland/pointer`.
- **`bluer`, `battery` and `tracing` are declared and never used.** Leftovers
  from when `crownbar`'s code lived here. `bluer` alone pulls in a large D-Bus
  and BlueZ tree that you must still have installed to link.
- **`predule` is a typo that is now permanent public API.** The prelude module
  is spelled `predule`, and 0.1.0 and 0.2.0 shipped it. 0.3.0 adds a correctly
  spelled `prelude` and keeps `predule` as a deprecated re-export.

---

## License

MIT — the only shell component with a LICENSE file. Copyright is attributed to
`marvelxcodes` rather than to Crown-OS, which is inconsistent with the other
licensed repo. See
[Project status](../00-overview/project-status.md#licensing).
