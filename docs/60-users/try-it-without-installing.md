# Try it without installing

CrownOS cannot be installed. There is no image, no package, and nothing on
crates.io you can `cargo install`. A **nested compositor session** is the only
way anyone can run CrownOS today, and it is safe: it opens as a window inside
your existing desktop, so there is no TTY to get stranded on and nothing to
recover from if it crashes.

This page is the whole procedure.

---

## What you need first

- A Linux machine with a working Wayland or X11 session, and a **working Vulkan
  or GL driver**. wgpu needs a real GPU adapter; in a VM without GPU passthrough
  the session fails at surface creation. No package list can check that for you.
- The native build dependencies, and the `[patch.crates-io]` overlay without
  which `crownpositor` does not resolve its dependencies at all.

Both come from one command:

```bash
git clone https://github.com/Crown-OS/crownos-setup && cd crownos-setup
./bootstrap.sh --dev
```

That clones the Rust repositories into `~/src/crownos` (override with
`--prefix=DIR` or `CROWNOS_SRC`), installs the native packages for your
distribution, and writes the overlay. See
[Prerequisites](../10-getting-started/prerequisites.md) and
[Workspace setup](../10-getting-started/workspace-setup.md#the-overlay-mandatory)
for what each part does.

Expect the first build to take a while. `crownpositor` locks around 400 crates.

---

## 1. Start the nested session

```bash
cd ~/src/crownos/crownpositor
CROWN_BACKEND=winit cargo run
```

A window opens containing a complete CrownOS session — the compositor, its
tiling layout, its keybindings, its cursor.

The backend is autodetected: if `WAYLAND_DISPLAY` or `DISPLAY` is set,
`crownpositor` runs nested via winit; otherwise it takes over the TTY with
DRM/KMS. Inside a desktop session you get nesting for free, but setting
`CROWN_BACKEND=winit` explicitly means you cannot be surprised.

Do **not** use `CROWN_BACKEND=kms` for this. That takes over your display.

---

## 2. Find the socket name

The nested compositor creates its **own** Wayland socket, separate from your
host compositor's. Everything else on this page depends on knowing its name.

`crownpositor` logs it on startup, to stderr, in the terminal you launched it
from:

```
INFO crownpositor: crownpositor is running socket="wayland-2"
```

If you do not see it, raise the log level. `crownpositor` uses `tracing`, so
`RUST_LOG` takes tracing's env-filter syntax — target-and-level pairs, not a
bare level:

```bash
RUST_LOG=crownpositor=info CROWN_BACKEND=winit cargo run
```

Failing that, list the sockets directly and take the one that is not your host's:

```bash
ls "${XDG_RUNTIME_DIR:-/run/user/$UID}"/wayland-*
echo "host session is: $WAYLAND_DISPLAY"
```

The name is almost always `wayland-1` or `wayland-2`. It is **not** stable
across runs — a second nested session gets a different number.

---

## 3. Attach a client

Open a second terminal, and point `WAYLAND_DISPLAY` at the nested socket rather
than your host's:

```bash
WAYLAND_DISPLAY=wayland-2 foot
```

Any Wayland client works — a terminal is the easiest way to confirm you are
really inside the nested session. `Super+Return` inside the window does the same
thing, if `foot` is installed.

### Attaching the CrownOS shell

`crownbar`, `crowndock` and `crownotify` are layer-shell clients, so they attach
the same way:

```bash
# Terminal 2 — the bar
cd ~/src/crownos/crownbar  && WAYLAND_DISPLAY=wayland-2 cargo run

# Terminal 3 — the dock
cd ~/src/crownos/crowndock && WAYLAND_DISPLAY=wayland-2 cargo run

# Terminal 4 — notifications
cd ~/src/crownos/crownotify && WAYLAND_DISPLAY=wayland-2 cargo run
```

That is as close to a full CrownOS desktop as currently exists. There is no
session file, no greeter, and no single command that starts all of it.

Those three also run perfectly well under **any** `wlr-layer-shell` compositor —
Hyprland, Sway, river, KWin — so if you only want to try the bar or the dock,
you do not need `crownpositor` at all. Just leave `WAYLAND_DISPLAY` alone.

---

## 4. Keep your real settings safe

Every component reads `~/.config/crownos/`. Point them somewhere else before you
start experimenting:

```bash
export CROWN_CONFIG_DIR=/tmp/crownos-try
```

Set it in each terminal, or export it once in the shell you launch everything
from. Files are created with defaults on first run, so you can start the
compositor once and then edit what appears there.

---

## What to expect

| | |
|---|---|
| Blur | **None.** `crownpositor`'s `ext-background-effect-v1` handler is entirely commented out, so surfaces that request frosted glass degrade silently. |
| Workspace overview | Not implemented. The bound actions log `"action is not implemented yet"`. |
| Clicking a dock icon | Nothing. `crowndock` has no `Exec=` parsing at all. |
| Notification centre | Nothing. `crownotify` logs and returns. |
| `crowndock` logs | None — it never initialises a logger, so its warnings are invisible. |
| Quitting | `Super+Shift+E`, or close the window. |

The full default binding table is in
[Keybindings](../50-reference/keybindings.md). Everything that does not work is
catalogued in [Known limitations](known-limitations.md) and
[Project status](../00-overview/project-status.md).

---

## If it does not start

See [Troubleshooting](troubleshooting.md). The two most common failures are a
missing GPU adapter (surface creation fails) and a missing `[patch.crates-io]`
overlay (`failed to select a version for crownshell ^0.3`, before anything
compiles).

---

## See also

- [Installing CrownOS](install.md) — why you cannot, and what would have to change
- [Build and run](../10-getting-started/build-and-run.md) — per-component commands
- [Testing and reporting](../40-contributing/testing-and-reporting.md) — if you
  want to send back what you find
