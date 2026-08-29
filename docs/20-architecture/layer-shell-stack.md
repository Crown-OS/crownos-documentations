# The layer-shell stack

How `crownbar`, `crowndock`, `crownotify` and `crowndictator` are built, and how
to write a new one.

---

## What layer shell is

`wlr-layer-shell` is a Wayland protocol for surfaces that are part of the desktop
rather than application windows — bars, docks, wallpapers, notification toasts,
lock screens. A layer surface declares:

- **Layer** — `Background`, `Bottom`, `Top`, `Overlay`, in stacking order
- **Anchor** — which screen edges it sticks to
- **Size** — with `0` meaning "as large as the anchoring allows"
- **Exclusive zone** — how many pixels to reserve so tiled windows do not overlap it
- **Keyboard interactivity** — whether it can take focus

`crownpositor` implements the compositor side in
`compositor/src/handlers/layer_shell.rs`. Every CrownOS shell component is a
client of it, and so can run under Hyprland, Sway, river or KWin equally well.

---

## crownshell

Rather than each component speaking Wayland directly, they all build on
`crownshell`, which owns the boilerplate: registry, outputs, seat, layer shell,
buffer scale, drag-and-drop, and a wgpu/Vello rendering surface per window.

What you write is a config and a paint callback.

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

> **The prelude module is spelled `predule`.** It is a typo in `crownshell`'s
> public API, present since the crate was extracted. Every downstream crate uses
> it. Fixing it is a breaking change to four consumers, so it stands for now —
> just do not assume `prelude` will work.

### The API surface

| Type | Role |
|---|---|
| `App` | Top-level context; `app.create_window(config, handler)` |
| `WindowConfig` | Where the surface sits and how it behaves |
| `SurfaceHandler` | The trait you implement. `paint` is the only required method. |
| `SurfaceCtx` | Passed to every callback: logical `size`, output `scale`, shared `text` context, region setters |
| `Text` / `TextStyle` | Retained, measurable text |
| `RawSurfaceHandler` | Lower-level escape hatch; used by the `raw_wgpu` wallpaper example |

`crownshell` re-exports `calloop`, `parley`, `vello` and `wayland_client` whole,
so downstream crates do not need matching versions in their own manifests.

### Repaint on demand

Event callbacks return `bool` — `true` means "I need a redraw". Returning `false`
when nothing visual changed is what keeps a bar from repainting on every pointer
motion event.

Two other redraw sources:

- **`on_frame`** fires when the compositor is ready for the next frame. Use it
  for animation, so it is paced by the frame clock rather than a timer.
- **`on_tick`** fires on a timer, default 1 s, set by
  `WindowConfig::tick_interval`. Use it for clocks and battery readings.

There is also `needs_redraw`, polled on every surface after each batch of events.
It is the one hook that lets a handler repaint a surface *other* than the one an
event arrived on — which is what you want for a D-Bus signal, a worker thread
message, or a bar click that must open a popup.

### Text

Text is shaped and laid out by [Parley](https://github.com/linebender/parley) and
drawn as glyph outlines by Vello. A `Text` is retained: build it once, keep it on
your handler, update the content each frame. The layout is cached and rebuilt only
when content, style or scale actually change — which matters because painting is
on demand.

`TextStyle` takes a CSS-syntax family stack (`"Inter, Noto Sans, sans-serif"`).
Missing families are skipped, so always end the stack with a generic family.
`ctx.text.register_font(bytes)` adds a font from memory if you want to ship your
own.

Text is currently single-line: no wrapping, no alignment.

### HiDPI

You draw in logical pixels. `crownshell` tracks the `wl_surface` buffer scale,
sizes the wgpu surface in physical pixels, and lays text out at the physical
density so glyphs rasterise at their true ppem rather than being upscaled.
Nothing in your `paint` changes when display density does.

Only integer buffer scales are supported, which is what
`wl_surface.set_buffer_scale` accepts. On a fractionally-scaled output the
compositor advertises the next integer up — a 1.25× display reports 2 — so text
renders at 2× and is scaled down. Sharp, but not pixel-exact. True fractional
scaling needs `wp_fractional_scale_v1`, which `crownshell` does not bind.

### Blur

Set `blur: true` and `crownshell` registers a blur region covering the surface
via `ext-background-effect-v1`. If the compositor does not advertise the
protocol, it is silently skipped.

For a surface that is only partly opaque — a menu panel on a full-screen surface
— set `auto_blur_region: false` and drive the region yourself with
`ctx.set_blur_region(&[rect])`, so the compositor is not blurring behind pixels
you never drew.

> **This does not work under `crownpositor` today.** Its
> `handlers/background_effect.rs` is entirely commented out and the global is
> never created, so every CrownOS surface that requests blur degrades silently.
> Under Hyprland or KWin it depends on their support for the staging protocol.
> See [Project status](../00-overview/project-status.md).

---

## The popup pattern

A popup is a second layer surface, not a region of the bar. `crownshell`'s
`examples/menu_popup.rs` is the reference implementation, and its module header
is the best-written explanation in the codebase. The pattern:

- **Create the popup surface up front**, on `Layer::Overlay`, next to the bar in
  `run`, and keep it for the life of the process. Opening it then costs one
  repaint instead of a wgpu surface, a set of Vello pipelines and a round of text
  shaping.
- **Anchor it on all four sides** with `size: (0, 0)` and `exclusive_zone: 0`.
  The compositor sizes it to the usable area, so its top edge sits just below the
  bar, and a click anywhere outside the panel arrives as an ordinary pointer
  event on that surface — which is how it dismisses.
- **While closed, draw nothing and drop both regions.**
  `ctx.set_input_region(&[])` makes it click-through;
  `ctx.set_blur_region(&[])` stops the compositor blurring behind an invisible
  panel.
- **Draw the panel into a scratch `Scene`** and `append` it under one `Affine`,
  so a frame of animation re-encodes draw commands without re-laying-out text.
- **Drive frames from `on_frame`**, returning `true` while animating.
- **Share state with an `Rc<RefCell<_>>`** — it is all one thread. A click lands
  on the *bar*, so the popup gets its first frame via `needs_redraw`.

---

## What the four components actually configure

| Component | Layer | Anchor | Size | Exclusive | Blur |
|---|---|---|---|---|---|
| `crownbar` | `Top` | TOP, LEFT, RIGHT | `(0, 40)` | 40 | yes |
| `crowndock` | `Overlay` | BOTTOM | `(1044, 100)` | 0 | yes |
| `crownotify` | `Top` | TOP, RIGHT, BOTTOM | width 400 | — | no |
| `crowndictator` | `Overlay` | bottom overlay | — | 0 | no |

Two details worth copying:

- **`crowndock`** rewrites its input region as it hides — down to a 1-pixel strip
  at the bottom edge — so clicks fall through to whatever is underneath while the
  dock is away. Its blur region is striped into 32 bands to follow the icon row.
- **`crowndictator`** sets an **empty input region** permanently, so the waveform
  overlay is purely decorative and never intercepts a click.

---

## Writing a new shell component

1. Depend on `crownshell` from crates.io (`crownshell = "0.3"`) — that is what
   `crownotify` and `crowndictator` do, and it means your changes to the
   framework are visible immediately. The git-URL form used by `crownbar` and
   `crowndock` pins an old revision.
2. Implement `SurfaceHandler`. Override only the callbacks you need.
3. Return `false` from event callbacks unless something visual changed.
4. Read your settings through `crownos-config` rather than inventing a config
   file. `crowndictator/src/settings.rs` is the 40-line model.
5. Call `env_logger::init()` in `main` — `crowndock` does not, and its warnings
   are invisible as a result.
6. Test it under Hyprland or Sway first. You do not need `crownpositor` running
   to develop a layer surface.

---

## See also

- [crownshell](../30-components/crownshell.md)
- [Architecture overview](overview.md)
- [Config as IPC](config-as-ipc.md)
