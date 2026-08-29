# crowndock

**Status: Partial** · Rust · default branch `main` ·
[repo](https://github.com/Crown-OS/crowndock)

The CrownOS dock. A bottom-anchored, auto-hiding overlay surface with
drag-and-drop application pinning.

---

## Surface configuration

| | |
|---|---|
| Layer | `Overlay` |
| Anchor | BOTTOM |
| Size | `(1044, 100)` — fixed |
| Exclusive zone | 0 |
| Blur | requested (`true`) |
| Namespace | `Crowndock` |

---

## Build and run

```bash
cd crowndock
cargo run
```

Drag a `.desktop` file onto it to pin an application. Pinned items persist across
restarts.

> `crowndock` does **not** call `env_logger::init()`, unlike its siblings, so its
> `log::warn!` output is invisible by default. Add the init if you need to debug
> it.

`crownshell` is a crates.io dependency (`"0.3"`). It used to be a git dependency with no rev or tag, whose lockfile entry had **no
`source` line**, meaning the lock was generated against a local path checkout
rather than the git URL — manifest and lock have diverged. See
[Dependency graph](../20-architecture/dependency-graph.md#version-skew).

---

## Source layout

| File | Role |
|---|---|
| `lib.rs` | Window config, calls `crownshell::run` |
| `main.rs` | Six lines |
| `config.rs` | Geometry constants and spring tuning — stiffness 240, damping 28, `SPRING_SUBSTEP` 1/240, `SPRING_MAX_DT` 1/30 (damping ratio ≈ 0.904) |
| `dock_handler.rs` | 501 lines — the `SurfaceHandler` |
| `persistence.rs` | `$XDG_CONFIG_HOME/crowndock/items.toml`, atomic write |
| `ui/` | `mod.rs`, `icon.rs`, `state.rs` |

### The visibility state machine

`VisState`: `Hidden → PendingShow → Showing → Shown → PendingHide → Hiding`,
driven by a hand-rolled spring.

Two implementation details worth copying:

- **Dynamic input region.** While hidden, `apply_input_region` shrinks the
  surface's input region to a **1-pixel strip** at the bottom edge, so clicks
  fall through to whatever is underneath.
- **Striped blur region.** The blur region is divided into 32 bands
  (`BLUR_STRIPS`) so the compositor blurs behind the icon row rather than the
  whole rectangle.

### Icons

Reads the `Icon=` key from `.desktop` files with `freedesktop_entry_parser` and
resolves it through `freedesktop-icons`, rasterising SVG with `resvg` and
`tiny-skia`, and raster formats with `image`.

### Drag and drop

Accepts `text/uri-list`, filters to `.desktop` files, and hand-rolls `file://`
percent-decoding.

---

## Known limitations

**Clicking a dock icon does not launch anything.** There is no `Exec=` parsing
and no `std::process::Command` anywhere in the crate. `on_pointer_press` and
`on_pointer_release` only drive drag state. The dock is currently a pin manager
with an animation — which is a problem, since launching applications is the point
of a dock. There is no TODO marker admitting this.

This is one of the highest-value open tasks in the project.

**It deviates from the config convention.** Instead of a section in
`~/.config/crownos/`, it stores pinned items in `~/.config/crowndock/items.toml`
— TOML, in its own directory. Bringing it onto `crownos-config` would require
adding a `dock` section.

**The size is fixed at 1044×100.** Not derived from screen width, icon count, or
any setting.

**Blur is requested but does not happen** under `crownpositor`.

**No tests.** It has its own spring implementation, separate from `crownshell`'s.
`tracing` is declared and unused. Its `tiny-skia` (0.11) and `dirs` (5) are a
major version behind its siblings.

---

## License

**No LICENSE file.** See
[Project status](../00-overview/project-status.md#licensing).
