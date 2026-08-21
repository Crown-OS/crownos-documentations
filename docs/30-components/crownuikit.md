# crownuikit

**Status: Early** · Rust · default branch `main` ·
[repo](https://github.com/Crown-OS/crownuikit)

A widget kit built on [xilem](https://github.com/linebender/xilem) and Masonry —
sidebar, sliders, toggles, selects, context menus, icons.

Deliberately a **different stack** from `crownshell`. `crownshell` is raw layer
shell plus direct Vello painting, for surfaces that are part of the desktop.
`crownuikit` is a reactive widget toolkit for ordinary application windows.

---

## What it is for

Most plausibly the CrownOS settings panel. Two pieces of evidence point that way:

- `crownos-config` ships a `xilem` feature — on by default — providing `watch`,
  `watched` and `watched_key` views that plumb config changes through xilem's
  message path. That exists to feed exactly this kind of app.
- Every `crownos-config` schema section is documented as being written by "the
  settings panel", and several name their specific page ("the settings panel's
  Input page").

*(This is inference. There is no README or crate-level doc comment stating it.)*

---

## Build and run

```bash
cd crownuikit
cargo run
```

Opens a desktop window with the widget gallery — sidebar, sliders, toggles, and a
`select`.

Three dependencies only: `xilem 0.4.0`, `winit 0.30.13`, `blinc_icons 0.5.1`
(Lucide icon bodies). **No CrownOS dependencies at all** — it is currently an
island.

---

## Source layout

Public API (`src/lib.rs`, five lines):

```rust
pub mod animation;
pub mod config;
pub mod layouts;
pub mod util;
pub mod widgets;
```

| Module | Contents |
|---|---|
| `config.rs` | Global `Theme` behind an `RwLock`; `theme()` / `set_theme()`; `GradientStops`, `PopoverColors` (a shadcn-derived dark palette) |
| `widgets/` | `context_menu`, `icon`, `select`, `slider`, `toggle` |
| `layouts/sidebar/` | `sidebar`, `sidebar_brand`, `sidebar_group`, `sidebar_item`, `sidebar_subitem`, `sidebar_separator`, plus `collapse.rs` (animated-height container) and `item.rs` |
| `util/` | `lerp_color`, the bundled Inter font, `inner_ring` / `outer_glow` shadows, `inflated_pill` |
| `animation.rs` | Its own spring implementation |

`layouts/sidebar/mod.rs` opens with a long SOLID-principles design note — the
most deliberate architectural writing in the repo.

`resources/fonts/Inter.ttf` is the only asset, embedded with `include_bytes!`.

---

## Known limitations

**The demo content is placeholder.** The sidebar in `main.rs` shows fintech
material lifted from a design mock — "Untitled UI", "Bank accounts", "Local
currency", "Upgrade to PRO". It is not CrownOS settings navigation.

**It is wired to nothing.** No `crownos-config` dependency, so it reads no
settings and writes none. Connecting it is the obvious next step, and
`crownos-config`'s `xilem_view` module exists to make that straightforward.

**Dead and empty files:**

- `widgets/status.rs` — zero bytes, not declared in `widgets/mod.rs`
- `widgets/search.rs` — zero bytes, but **is** declared, so it is an empty live
  module
- `layouts/header/mod.rs` — zero bytes, not declared in `layouts/mod.rs`

**It has both a `lib.rs` and a `main.rs`** with no explicit `[[bin]]`, so `cargo
run` builds the gallery binary and `cargo build` builds both.

**No tests, no examples, no benches.**

**A fourth spring implementation.** `crownpositor`, `crownshell`, `crownbar`,
`crowndock` and this crate each have their own.

---

## License

**No LICENSE file.** See
[Project status](../00-overview/project-status.md#licensing).
