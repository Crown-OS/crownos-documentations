# What is CrownOS

CrownOS is an Arch Linux–based distribution with a Wayland desktop written from
scratch in Rust, and an ecosystem layer that bridges the desktop to a phone.

It is three projects that happen to share a name:

1. **A compositor and shell.** `crownpositor` is a tiling Wayland compositor
   built on [Smithay](https://github.com/Smithay/smithay). The bar, dock,
   notification daemon and dictation overlay are separate processes that attach
   to it as `wlr-layer-shell` clients, all built on a shared rendering framework
   called `crownshell`.
2. **An ecosystem bridge.** `crowncrate` links the desktop to a phone —
   clipboard sync, notification sync, call bridging, file sharing, second
   screen. The Linux side is a daemon; there are Android and browser clients.
3. **A distribution.** `crownos-iso` is the archiso profile that packages the
   above into a bootable image.

---

## Design principles

These are drawn from the project's own design brief and from how the code is
actually written.

**Performance first.** The compositor renders through Vulkan or GL via wgpu.
Springs are integrated at a fixed 1/240 s substep. The shell repaints on demand
rather than on a clock — every event callback returns a `bool` meaning "I need a
repaint", so moving the mouse across the bar does not cost a frame.

**Monochromatic and minimal.** The visual language is a restrained greyscale with
a single accent, blur and rounded corners for depth rather than colour.

**Customisable without being fragile.** Settings are plain text files you can
hand-edit. A malformed config falls back to defaults and is *not* overwritten —
the code assumes you might be halfway through editing it.

**Local-first.** Voice dictation runs entirely on-device with ONNX Runtime.
Nothing is sent to a server for transcription.

---

## What makes it architecturally unusual

**There is no IPC daemon.** Most desktop environments coordinate over D-Bus or a
custom socket. CrownOS components coordinate by reading and writing RON files in
`~/.config/crownos/` and watching that directory with inotify. Change
`appearance.ron` and every process that cares picks it up live, with no restart
and no message bus.

This is the single most important thing to understand about the codebase. It is
described in detail in [Config as IPC](../20-architecture/config-as-ipc.md).

**The compositor is the session.** `crownpositor` creates the Wayland socket,
exports `WAYLAND_DISPLAY`, and spawns the bar, wallpaper and notification daemon
itself from a `startup` list in its config. There is no separate session manager.

**One action vocabulary.** Keyboard chords and trackpad gestures both resolve to
the same `Action` enum and go down the same dispatch path — deliberately, so that
"swipe left" and `Super+Tab` cannot drift apart.

---

## What CrownOS is not, yet

Being direct about this saves you time:

- **It is not installable.** `crownos-iso` is currently an unmodified copy of the
  upstream Arch `releng` profile. It builds a generic Arch rescue image with no
  CrownOS packages in it.
- **It is not a complete desktop.** The dock cannot launch applications. The
  launcher is a `cargo new` stub. There is no settings panel, no greeter, no
  notification centre.
- **The phone bridge does not work.** Neither the Linux daemon nor the Android
  app is functional; they have never talked to each other.
- **Nothing is packaged.** No crates.io, no AUR, no releases.

What *does* work, and works well, is the compositor, the shell framework, the
config system, and voice dictation. See
[Project status](project-status.md) for the component-by-component picture.

---

## Where to go next

- [Component map](component-map.md) — every repository, what it does, its status
- [Project status](project-status.md) — what builds, what is broken, what is planned
- [Architecture overview](../20-architecture/overview.md) — how it fits together
- [Prerequisites](../10-getting-started/prerequisites.md) — start building
