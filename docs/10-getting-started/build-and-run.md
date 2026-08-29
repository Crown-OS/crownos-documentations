# Build and run

Per-component commands. Assumes you have installed the
[prerequisites](prerequisites.md) — `crownos-setup`'s `bootstrap.sh --dev` does
that and the checkout in one step.

Every component now builds from a plain clone, anywhere on disk, because crates
depend on published crates.io versions rather than on relative paths. You only
need a particular layout if you are changing `crownshell` or `crownos-config` and
want a component to pick that change up — see
[Workspace setup](workspace-setup.md#developing-across-repositories).

Every Rust crate in the organization compiles as of this writing. Several are
still **Skeleton** — they build and do almost nothing — and
[Project status](../00-overview/project-status.md) says which.

---

## Start here: crownos-config

The foundation crate, and the fastest thing to verify your toolchain with. No
native dependencies beyond what `notify` needs.

```bash
cd crownos-config
cargo build
cargo test
```

`cargo test` runs 26 unit tests plus a single integration test named `e2e`. That
one test does eight things sequentially — this is deliberate, because
`CROWN_CONFIG_DIR` is process-global and cargo runs test functions on parallel
threads. Do not split it up.

---

## crownshell

A library, so there is no binary. The examples are the way to see it work, and
they run under **any** `wlr-layer-shell` compositor — Hyprland, Sway, river, KWin
— so you do not need `crownpositor`.

```bash
cd crownshell
cargo build

cargo run --example text_bar     # top bar with a label and a live clock
cargo run --example menu_popup   # bar plus an animated overlay popup
cargo run --example raw_wgpu     # one wallpaper surface per output
```

`menu_popup` is the best-documented code in the project — its module header
explains the popup pattern properly. Read it before writing a new surface.

If an example fails at surface creation, you do not have a working Vulkan or GL
adapter. See [Prerequisites](prerequisites.md#everything-any-crownshell-based-app).

---

## crownpositor

### Nested (what you want for development)

The compositor autodetects: if `WAYLAND_DISPLAY` or `DISPLAY` is set it runs
nested via winit, otherwise it takes over the TTY with DRM/KMS. Inside an
existing desktop session you therefore get nesting for free, but being explicit
is better:

```bash
cd crownpositor
CROWN_BACKEND=winit cargo run
```

That opens a window containing a complete CrownOS session. It logs
`crownpositor is running` and prints the Wayland socket name it created.

To attach a client to it, use that socket rather than your host compositor's:

```bash
# In the nested session's log, find the socket name (e.g. wayland-2)
WAYLAND_DISPLAY=wayland-2 foot
```

`Super+Return` inside the nested window spawns `foot`, if you have it installed.
The full default binding table is in
[Keybindings](../50-reference/keybindings.md).

### On real hardware

```bash
# From a bare TTY, with seatd running or on a logind session
CROWN_BACKEND=kms cargo run --release
```

This takes over the display. Have a way back — a second TTY, or SSH — before you
try it. `Super+Shift+E` quits.

### Renderer selection

```bash
CROWN_RENDER_API=vulkan cargo run    # or: egl / gles / gles3
```

Defaults to EGL/GLES3. An unrecognised value logs a warning and falls back rather
than failing, so check the log if a setting seems to be ignored.

### Tests

```bash
cargo test          # 183 unit tests across the workspace
```

---

## crownbar

```bash
cd crownbar
cargo run
RUST_LOG=debug cargo run    # it initialises env_logger; default level is info
```

Runs under any layer-shell compositor. Widgets that cannot find their hardware
return `None` and are silently skipped, so on a desktop with no battery you
simply get no battery indicator — that is not a bug.

Note this crate forces the BFD linker through a committed `.cargo/config.toml`.
If you use `mold` or `lld` globally, it is overridden here.

---

## crowndock

```bash
cd crowndock
cargo run
```

Drag a `.desktop` file onto it to pin an application. Pinned items persist to
`~/.config/crowndock/items.toml`.

**Clicking an icon does not launch anything** — that code does not exist yet. See
[Project status](../00-overview/project-status.md#crowndock-cannot-launch-applications).

`crowndock` does not call `env_logger::init()`, so its `log::warn!` output is
invisible by default. Add the init if you need to debug it.

---

## crowndictator

The most demanding component to run. Read
[its prerequisites](prerequisites.md#crowndictator) first — you need `input`
group membership and a multi-gigabyte model download.

```bash
cd crowndictator

cargo run -- --demo               # cycle the overlay states with fake audio.
                                  # No model, no microphone, no input group.
cargo run                         # the real daemon
cargo run -- --transcribe f.wav   # one-shot transcription of a 16 kHz wav
cargo run -- --cpu                # skip CUDA
```

**`--demo` is the contributor-friendly path.** It exercises the whole overlay UI
without downloading anything or touching `/dev/input`. Use it for any work on
`waveform.rs` or the visual states.

The real daemon holds `Super+Space` as push-to-talk by default, configurable at
`~/.config/crownos/input.ron`. It types the result using `wtype`, falling back to
`ydotool`, falling back to `wl-copy` plus a `notify-send`.

The ASR engine drops its model from memory after 300 s idle to free VRAM, so the
first transcription after a pause is slower.

---

## crownuikit

```bash
cd crownuikit
cargo run
```

Opens a desktop window showing the widget gallery — sidebar, sliders, toggles,
selects. Note the sidebar content is placeholder material from a design mock
("Bank accounts", "Upgrade to PRO"); it is not CrownOS settings navigation.

This crate has no CrownOS dependencies. It is intended to become the settings
panel, which is also why `crownos-config` ships a `xilem` feature — but nothing
connects them yet.

---

## crownos-website

```bash
cd crownos-website
bun install
bun run dev       # http://localhost:3000
bun run lint      # biome check
bun run format    # biome format --write
bun run build     # next build, also type-checks
```

Use Bun. The lockfile is `bun.lock`; npm or yarn will produce a competing
lockfile.

There is no `check-types` script and no test framework in this repo.

---

## crowncrate-android

```bash
cd crowncrate-android
./gradlew assembleDebug
./gradlew installDebug     # to a connected device or emulator
./gradlew test
```

Needs Android SDK 36 and JDK 11+. The app is currently an unmodified Android
Studio template — it has no network permission and cannot talk to the desktop.

There is **no ktlint plugin configured**, so `./gradlew ktlintCheck` will fail
with "task not found". Do not run it.

---

## crownos-iso

Requires `archiso` and root on an Arch system. There is no build script in the
repo — you invoke `mkarchiso` against the profile directory:

```bash
sudo mkarchiso -v -w /tmp/crownos-work -o /tmp/crownos-out ./crownos-iso
```

The profile is currently an unmodified copy of upstream Arch's `releng`, so what
comes out is a generic Arch rescue image named `archlinux`, with no CrownOS
packages and no CrownOS branding. See
[crownos-iso](../30-components/crownos-iso.md).

---

## Running the whole desktop

There is no single command that starts a full CrownOS session, and no session
file to select from a display manager. The compositor spawns startup programs
itself from `compositor.startup` in its config, but the `Compositor` schema in
`crownos-config` does not currently define a `startup` field — so the two
checkouts do not agree, and that path does not work today.

To approximate a session by hand:

```bash
# Terminal 1 — the compositor, nested
cd crownpositor && CROWN_BACKEND=winit cargo run
# note the socket name it logs, e.g. wayland-2

# Terminal 2 — a bar inside it
cd crownbar && WAYLAND_DISPLAY=wayland-2 cargo run

# Terminal 3 — the dock
cd crowndock && WAYLAND_DISPLAY=wayland-2 cargo run
```

Expect no blur: `crownpositor`'s `ext-background-effect-v1` handler is commented
out, so the surfaces that request it degrade silently.

---

## Next

- [Your first change](your-first-change.md) — pick something and send a patch
- [Architecture overview](../20-architecture/overview.md) — how the pieces relate
- [Testing](../40-contributing/testing.md) — what tests exist and how to run them
