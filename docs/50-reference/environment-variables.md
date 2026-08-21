# Environment variables

Every environment variable CrownOS components read.

---

## CROWN_CONFIG_DIR

**Read by:** `crownos-config`, and therefore every component that uses it —
`crownpositor`, `crowndictator`.

Overrides the location of the CrownOS configuration directory.

```bash
export CROWN_CONFIG_DIR=/tmp/crownos-dev
```

Resolution order:

1. `$CROWN_CONFIG_DIR`, if set and **non-empty**
2. `dirs::config_dir()/crownos` — normally `~/.config/crownos`
3. `./crownos`, if the platform config dir cannot be determined

Section files are `<dir>/<section>.ron`.

**Use it while developing** so a work-in-progress build cannot damage your real
settings. It is also what the test suite uses.

> Because it is process-global, `crownos-config`'s integration test is
> deliberately a single `#[test]` function. See
> [Testing](../40-contributing/testing.md#crownos-config--one-test-function-on-purpose).

---

## CROWN_BACKEND

**Read by:** `crownpositor`.

Chooses the compositor backend, overriding autodetection.

| Value | Backend |
|---|---|
| `winit` | Nested inside an existing Wayland or X11 session |
| `kms`, `drm`, `udev` | Bare TTY: DRM/KMS outputs, libinput input, libseat session |

Matching is case-insensitive. An unrecognised value logs a warning and falls back
to autodetection rather than failing.

**Autodetection**, when unset: if `WAYLAND_DISPLAY` or `DISPLAY` is present the
compositor assumes it is nested and picks `winit`; otherwise it picks KMS.

The override exists because "run the KMS backend nested under a session that
leaks `DISPLAY`" is a real debugging situation.

```bash
CROWN_BACKEND=winit cargo run     # development
CROWN_BACKEND=kms cargo run --release   # real hardware, from a TTY
```

---

## CROWN_RENDER_API

**Read by:** `crownpositor`.

Chooses the graphics API for the renderer.

| Value | API |
|---|---|
| `egl`, `gles`, `gles3` | EGL / GLES 3 — **the default** |
| `vulkan`, `vk` | Vulkan |

Case-insensitive. Anything else — including unset — picks EGL/GLES3 and logs
that it did, because *"silently ignoring a typo in a tuning knob wastes an
afternoon."*

```bash
CROWN_RENDER_API=vulkan cargo run
```

---

## XCURSOR_THEME

**Read by:** `crownpositor`.

The XCursor theme name to load. Falls back to a built-in cursor if the theme
cannot be found.

## XCURSOR_SIZE

**Read by:** `crownpositor`.

Cursor size in pixels.

Both follow the standard freedesktop conventions, so setting them in your shell
profile affects CrownOS the same way it affects other desktops.

---

## WAYLAND_DISPLAY

**Set by:** `crownpositor`. **Read by:** every Wayland client.

The compositor binds an auto-named socket and exports its name. When it spawns a
startup program it sets `WAYLAND_DISPLAY` for the child, removes `DISPLAY`, and
calls `setsid`.

You need this when attaching a client to a nested session by hand:

```bash
# crownpositor logs the socket it created, e.g. wayland-2
WAYLAND_DISPLAY=wayland-2 cargo run    # in crownbar
```

## DISPLAY

**Read by:** `crownpositor`, only as an autodetection hint — its presence implies
a nested session. Deliberately **removed** from the environment of programs the
compositor spawns.

---

## RUST_LOG

**Read by:** `crownbar`, `crownotify`, `crowndictator` — every component that
calls `env_logger::init()`.

Standard [env_logger](https://docs.rs/env_logger/) filter syntax. Default level
is `info`.

```bash
RUST_LOG=debug cargo run
RUST_LOG=crownbar::widgets=trace cargo run
```

> **`crowndock` does not call `env_logger::init()`.** Its `log::warn!` output is
> invisible regardless of `RUST_LOG`. Add the init locally if you need to debug
> it.

`crownpositor` uses `tracing` with `tracing-subscriber` and `tracing-journald`
rather than `env_logger`, so it follows `RUST_LOG` through the tracing env
filter.

---

## DBUS_SESSION_BUS_ADDRESS

**Read by:** `crownotify` and its tests, through zbus.

`crownotify` owns `org.freedesktop.Notifications` and `io.crownos.crownotify` on
the **session** bus, and calls `io.crownos.crowncrate` there too.

Its tests will not run without a session bus:

```bash
dbus-run-session -- cargo test -- --test-threads=1
```

---

## XDG_CONFIG_HOME

**Read by:** `crowndock`, via `dirs`.

`crowndock` stores pinned items at `$XDG_CONFIG_HOME/crowndock/items.toml` —
outside the CrownOS config convention, in its own directory and in TOML. See
[crowndock](../30-components/crowndock.md#known-limitations).

`crownos-config` also resolves through `dirs::config_dir()`, which honours
`XDG_CONFIG_HOME`, but `CROWN_CONFIG_DIR` takes precedence over it.

---

## Quick reference

| Variable | Component | Purpose |
|---|---|---|
| `CROWN_CONFIG_DIR` | crownos-config (all consumers) | Override the config directory |
| `CROWN_BACKEND` | crownpositor | `winit` / `kms` |
| `CROWN_RENDER_API` | crownpositor | `egl` / `vulkan` |
| `XCURSOR_THEME` | crownpositor | Cursor theme |
| `XCURSOR_SIZE` | crownpositor | Cursor size |
| `WAYLAND_DISPLAY` | set by crownpositor | The Wayland socket |
| `DISPLAY` | crownpositor (hint only) | Nested-session detection |
| `RUST_LOG` | crownbar, crownotify, crowndictator | Log filter |
| `DBUS_SESSION_BUS_ADDRESS` | crownotify | Session bus |
| `XDG_CONFIG_HOME` | crowndock, crownos-config | Config root |
