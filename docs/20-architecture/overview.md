# Architecture overview

How CrownOS fits together at runtime.

---

## The process model

```
    ┌──────────────────────────────────────────────────────────────┐
    │                        crownpositor                          │
    │                                                              │
    │  • Creates the Wayland socket, exports WAYLAND_DISPLAY        │
    │  • DRM/KMS on a TTY, or winit nested in another session       │
    │  • Owns monitors → workspaces → tiles                         │
    │  • Spawns the rest of the desktop from compositor.startup     │
    └───────────────────────────┬──────────────────────────────────┘
                                │ wlr-layer-shell
        ┌───────────┬───────────┼────────────┬──────────────┐
        │           │           │            │              │
   ┌────┴────┐ ┌────┴────┐ ┌────┴─────┐ ┌────┴────────┐ ┌───┴──────┐
   │crownbar │ │crowndock│ │crownotify│ │crowndictator│ │ ordinary │
   │ Top     │ │ Overlay │ │  Top     │ │  Overlay    │ │ xdg-shell│
   │ bar     │ │ dock    │ │  toasts  │ │  waveform   │ │ apps     │
   └────┬────┘ └────┬────┘ └────┬─────┘ └────┬────────┘ └──────────┘
        │           │           │            │
        └───────────┴─────┬─────┴────────────┘
                          │  all built on
                    ┌─────┴──────┐
                    │ crownshell │  Wayland boilerplate + Vello painting
                    └────────────┘

  ┌────────────────────────────────────────────────────────────────┐
  │  ~/.config/crownos/*.ron   ←→   crownos-config                 │
  │  Read, written and watched by every component that cares.      │
  │  This is the coordination mechanism. There is no IPC daemon.   │
  └────────────────────────────────────────────────────────────────┘

  ┌────────────────────────────────────────────────────────────────┐
  │  D-Bus (session)                                               │
  │  crownotify owns org.freedesktop.Notifications                 │
  │                and io.crownos.crownotify                       │
  │  crownotify calls  io.crownos.crowncrate  (pickup/decline)     │
  └────────────────────────────────────────────────────────────────┘
```

---

## Three ideas that explain most of the code

### 1. The compositor is the session

There is no session manager, no `systemd --user` target, no `.desktop` session
file. `crownpositor` binds an auto-named Wayland socket, sets `WAYLAND_DISPLAY`
in its own environment, and spawns the bar, wallpaper and notification daemon
itself from a `startup` list in its configuration.

When it spawns a child it sets `WAYLAND_DISPLAY`, removes `DISPLAY`, and calls
`setsid` in the pre-exec hook so the child does not share the compositor's
controlling terminal.

> **Caveat.** The compositor reads `config.compositor.startup`, but the
> `Compositor` schema in `crownos-config` has no `startup` field. The two
> checkouts do not currently agree, so this path does not work as written. See
> [Project status](../00-overview/project-status.md).

### 2. Configuration is the IPC

Components do not send each other messages. They read and write RON files in
`~/.config/crownos/` and watch that directory with inotify. Changing
`appearance.ron` propagates to every process that subscribed to it, live, with no
restart and no bus.

This is unusual enough that it has [its own page](config-as-ipc.md).

### 3. One action vocabulary

Keyboard chords and trackpad gestures both resolve to the same `Action` enum and
go down one dispatch path, `State::handle_action`. The source comment is explicit
about why:

> No second enum for gestures — that is how "swipe left" and "Super+Tab" drift
> apart.

Actions parse from strings, which is what makes the config file's
`(keys: "Super+Q", action: "close-window")` rows work. See
[Keybindings](../50-reference/keybindings.md).

---

## Layering inside the compositor

`crownpositor` keeps hard boundaries between layers, and they are worth
respecting when you add code.

| Layer | Modules | Rule |
|---|---|---|
| **Layout** | `layout/{master_stack,scrolling,floating}` | Pure geometry. Nothing here can see a `WlSurface`, a `Window`, an `Output` or the `Shell`. |
| **Shell** | `shell/{monitor,workspace,tile,grab,transaction}` | The model: monitors own workspaces, workspaces own tiles. Indices are the only route from a surface to anything else. |
| **Handlers** | `handlers/*` | One file per Wayland protocol delegate. |
| **Backend** | `backend/{kms,winit,render}` | Hardware. `backend/mod.rs` carries a four-step guide for adding a new one. |
| **State** | `state/*` | The aggregate everything else hangs off. |

The layout layer being surface-blind is the reason the tiling algorithms are
unit-testable — 183 tests live in `crownpositor`, mostly there and in `config/`.

---

## The rendering stack

Every CrownOS surface, in the compositor and in the shell components, is painted
the same way:

```
your paint(&mut Scene)  →  Vello  →  wgpu  →  Vulkan or GL
                              ↑
                     Parley (text shaping and layout)
                              ↑
                   fontconfig (system font database)
```

`crownshell` gives shell components a `SurfaceHandler` trait whose only required
method is `paint(&mut self, scene: &mut Scene, ctx: SurfaceCtx)`. Everything else
— pointer, keyboard, drag-and-drop, ticks, frame callbacks — has a default no-op.

Event callbacks return `bool` meaning "I need a repaint". This is why moving the
mouse across the bar does not cost a frame: the handler returns `false` unless
something visual actually changed.

HiDPI is handled below you. You draw in logical pixels; `crownshell` tracks the
buffer scale, sizes the wgpu surface in physical pixels, and lays text out at the
physical density so glyphs rasterise at their true ppem. Only integer buffer
scales are supported — `wp_fractional_scale_v1` is not bound.

---

## Where the boundaries actually are

A summary of what talks to what, and how:

| From | To | Mechanism |
|---|---|---|
| Shell components | compositor | `wlr-layer-shell` (Wayland) |
| Applications | compositor | `xdg-shell` (Wayland), XWayland for X11 clients |
| Any component | any component | `~/.config/crownos/*.ron` + inotify |
| Applications | `crownotify` | D-Bus `org.freedesktop.Notifications` |
| Settings UI | `crownotify` | D-Bus `io.crownos.crownotify` |
| `crownotify` | `crowncrate` | D-Bus `io.crownos.crowncrate` |
| Phone | desktop | TCP :5252, CBOR (`crowncrate`) |
| Phone screen | desktop | UDP, RTP-shaped (`lls-protocol`) |
| `crowndictator` | kernel | `/dev/input/event*` via evdev — bypasses the compositor |
| `crowndictator` | focused window | `wtype` / `ydotool` / `wl-copy` subprocess |

Note the two deliberate bypasses at the bottom. `crowndictator` reads the
keyboard directly rather than asking the compositor for a global shortcut, and
injects text by shelling out. Both are so it works on any compositor, not just
`crownpositor`.

Detail: [IPC and protocols](ipc-and-protocols.md).

---

## Read next

- [Config as IPC](config-as-ipc.md) — the coordination mechanism, in detail
- [The layer-shell stack](layer-shell-stack.md) — how shell components are built
- [IPC and protocols](ipc-and-protocols.md) — D-Bus, crowncrate, lls-protocol
- [Dependency graph](dependency-graph.md) — build-time relationships and skew
