# crownbar

**Status: Partial** · Rust · default branch `main` ·
[repo](https://github.com/Crown-OS/crownbar)

The CrownOS status bar. A top-anchored layer surface showing clock, battery,
wifi, bluetooth, brightness and volume.

---

## Surface configuration

| | |
|---|---|
| Layer | `Top` |
| Anchor | TOP, LEFT, RIGHT |
| Size | `(0, 40)` — full width, 40 px tall |
| Exclusive zone | 40 |
| Blur | requested (`true`) |
| Namespace | `crownbar` |

---

## Build and run

```bash
cd crownbar
cargo run
RUST_LOG=debug cargo run     # env_logger is initialised; default level is info
```

Runs under any `wlr-layer-shell` compositor, so you can develop it inside your
existing desktop.

> **This crate forces the BFD linker** via a committed
> `.cargo/config.toml`:
> `rustflags = ["-C", "link-arg=-fuse-ld=bfd"]`. If you use `mold` or `lld`
> globally, it is overridden here. You need `ld.bfd` from `binutils`.

`crownshell` is now `"0.3"` from crates.io. It used to be a **git dependency with no rev or tag** whose lockfile pinned commit
`de4ab90` at version 0.1.0, three commits behind HEAD. Your local `crownshell`
checkout is not used. See
[Dependency graph](../20-architecture/dependency-graph.md#version-skew).

---

## Source layout

| File | Role |
|---|---|
| `lib.rs` | `app()` — builds the `WidgetRegistry`, calls `crownshell::run` |
| `main.rs` | env_logger init plus `app()` |
| `config.rs` | Two constants: `BAR_NAMESPACE`, `BAR_HEIGHT = 40` |
| `bar_handler.rs` | The `SurfaceHandler`: hit-test → hover spring → paint |
| `theme.rs` | `THEME` const — colours, paddings, `font_size: 14.0` |
| `animation.rs` | A local `Spring`/`Clock`, not `crownshell`'s |
| `ui/` | `BarPainter`, `pill.rs`, `text.rs`, `icons/` (7 procedurally drawn Vello icons) |
| `widgets/` | `BarWidget` trait and registry, plus battery, bluetooth, brightness, clock, layout, volume, wifi |
| `util/poll.rs` | `PollGate` — an "every N ticks" gate over the 1 Hz tick |

---

## How it reads the system

Deliberately, `crownbar` talks to the kernel and to command-line tools rather
than to daemons — the source comments say this is to keep the bar
self-sufficient and able to work pre-login.

| Widget | Source |
|---|---|
| Wi-Fi | `/sys/class/net/*/wireless` and `/proc/net/wireless` — no NetworkManager |
| Bluetooth | `/sys/class/rfkill/*` — no BlueZ, no D-Bus |
| Brightness | `/sys/class/backlight/` |
| Volume | subprocess `wpctl`, falling back to `pactl` |
| Battery | the `battery` crate |
| Clock | `chrono` |

A widget whose hardware is absent returns `None` from `try_new()` and is silently
skipped. On a desktop with no battery you simply get no battery indicator — that
is not a bug.

---

## Known limitations

**It reads no CrownOS configuration.** `crownbar` has no `crownos-config`
dependency. It hardcodes `BAR_HEIGHT = 40` while `appearance.bar_height` defaults
to 32, and it ignores accent colour, transparency and animation profile. Wiring
it to `appearance.ron` is a well-scoped first task — copy
`crowndictator/src/settings.rs`, which is 40 lines.

**The layout widget is a no-op.** From its own header: *"The actual compositor
switch is out of scope for the bar — for now the widget just owns the local UI
state and surfaces it as a smoothly-animated icon so the visual treatment is in
place when the compositor protocol lands."* Clicking it changes an icon and
nothing else.

**Blur is requested but does not happen** under `crownpositor`, which never
advertises `ext-background-effect-v1`.

**Widgets are hardcoded** in `lib.rs`. Nothing is configurable at runtime.

**No tests.** `theme.rs` carries a crate-wide `#[allow(dead_code)]`.
`assets/battery.svg` is committed but referenced nowhere — the icons are drawn in
code. `bluer` and `tracing` are declared dependencies and unused.

**It has its own spring implementation**, separate from `crownshell`'s. It
predates the framework's and never migrated.

---

## License

**No LICENSE file.** See
[Project status](../00-overview/project-status.md#licensing).
