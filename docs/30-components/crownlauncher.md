# crownlauncher

**Status: Skeleton** · Rust · default branch `main` ·
[repo](https://github.com/Crown-OS/crownlauncher)

The CrownOS application launcher — or rather, where it will go. The repository
currently contains `cargo new` output and nothing else.

---

## Current contents

One commit, `First commit`, dated 2025-10-22 — a year older than everything else
in the organization.

```
Cargo.toml     6 lines
src/main.rs    fn main() { println!("Hello, world!"); }
```

`Cargo.toml` declares:

```toml
[package]
name = "launcher"      # not "crownlauncher"
version = "0.1.0"
edition = "2024"

[dependencies]
```

Note the **package is named `launcher`**, not `crownlauncher`. There is no
`Cargo.lock`, no README, no dependencies.

---

## The contract that already exists

Interestingly, the launcher is specified before it is implemented.

`crownos-config`'s `keybinds` section defines:

```rust
/// Shows and hides the launchpad.
pub launcher as Launcher: Keybind = Keybind::SUPER_CTRL,
```

with a doc comment explaining the design: one shortcut for both directions rather
than two, because *"the launchpad is a place you go and come back from, and a
separate 'close' chord would be one more thing to bind and one more thing to
forget."*

The default is **modifier-only** (`Super+Ctrl`) so it cannot collide with an
application's own bindings.

The section's module doc is equally specific about who must implement it:

> The consumer is the compositor. It is the only process that can honour a global
> chord — every other CrownOS app is a Wayland client and only ever sees keys
> while it is focused, which is exactly the state a launcher shortcut has to work
> from *outside*.

**`crownpositor` does not read the `keybinds` section.** `Config::sections()`
returns only `compositor`, `appearance` and `display`. So the shortcut is defined,
documented, and connected to nothing at either end.

---

## Building it

If you want to take this on, the shape is fairly well determined by what already
exists:

1. **A `crownshell` layer surface** on `Layer::Overlay`, following the popup
   pattern from `crownshell`'s `examples/menu_popup.rs` — created up front,
   anchored on all four sides with `size: (0, 0)`, input and blur regions dropped
   while closed so it is click-through.
2. **`.desktop` file enumeration and `Exec=` launching.** `crowndock` already
   parses `.desktop` files with `freedesktop_entry_parser` and resolves icons
   with `freedesktop-icons` — reuse that approach. (`crowndock` cannot launch
   applications either; solving it once for both would be sensible.)
3. **Compositor support for the `keybinds` section**, so `Super+Ctrl` actually
   reaches the launcher. That means extending `Config::sections()` in
   `crownpositor` and adding an action that toggles the launcher.

See [The layer-shell stack](../20-architecture/layer-shell-stack.md) for how to
write the surface.

---

## License

**No LICENSE file.** See
[Project status](../00-overview/project-status.md#licensing).
