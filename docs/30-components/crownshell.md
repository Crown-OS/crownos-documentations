# crownshell

**Status: Early** · Rust · default branch `main` · published version **0.2.0** ·
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
[Prerequisites](../10-getting-started/prerequisites.md#every-crownshell-based-app).
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
use crownshell::prelude::*;
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

> **The prelude used to be spelled `predule`.** That typo shipped in 0.1.0 and
> 0.2.0 and is what all four downstream crates import. 0.3.0 introduces the
> correctly spelled `prelude` module and keeps `predule` as a `#[deprecated]`
> re-export of it, so existing code still compiles with a warning; the alias is
> scheduled for removal in 0.4.0. **That correction is a local, uncommitted
> change** — on the default branch `src/lib.rs` still says `pub mod predule;`
> and no `prelude` exists.

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
| `prelude.rs` | 14 | The prelude. Was `predule.rs`; `lib.rs` keeps a deprecated `predule` alias for 0.1/0.2 consumers. |
| `wayland/` | — | Per-protocol dispatch, including `background_effect` |

---

## How it fits into CrownOS

`crownshell` is purely a Wayland client library. It does **not** depend on
`crownos-config` — components read their own settings.

Consumed by `crownbar`, `crowndock`, `crownotify` and `crowndictator`, all four
declaring `crownshell = "0.3"`. No path or git dependencies remain.

> **`crownshell` 0.3 has not been published.** crates.io carries 0.1.0 and 0.2.0
> and nothing else, and the organization's only git tag is `crownshell v0.2.0`.
> A working tree bumps the manifest to `version = "0.3.0"` and adds the `prelude`
> rename that release is meant to carry, but that is uncommitted and unreleased;
> the default branch still declares 0.2.0.
> Every dependent therefore fails to resolve from a plain clone; the requirement
> is satisfied by the `[patch.crates-io]` overlay described in
> [Workspace setup](../10-getting-started/workspace-setup.md#the-overlay-mandatory).
> `crownshell` itself has no CrownOS dependencies and builds from a plain clone.

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
- **`predule` was a typo in the public API.** Both published releases — 0.1.0
  and 0.2.0 — shipped the prelude under that spelling, and every downstream
  crate imports it. It is no longer permanent: 0.3.0 adds `prelude` and demotes
  `predule` to a `#[deprecated]` re-export, with removal scheduled for 0.4.0.
  Downstream crates still import `predule` and will warn until they are updated.
  All of that is uncommitted work — the default branch has only `predule`.

---

## License

MIT — the only shell component with a LICENSE file, and one of only three
repositories in the organization that carry one on their default branch
(`crownos-setup` and `crownos-documentations` are the others). Copyright is
attributed to `marvelxcodes`; the other two say "The CrownOS Authors" and
"Crown-OS". All three disagree. See
[Project status](../00-overview/project-status.md#licensing).
