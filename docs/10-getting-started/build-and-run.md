# Build and run

Per-component commands. Assumes you have installed the
[prerequisites](prerequisites.md) — `crownos-setup`'s `bootstrap.sh --dev` does
that and the checkout in one step.

> **A plain clone does not build.** `crownbar`, `crowndock`, `crownotify`,
> `crowndictator` and `crownpositor` declare `crownshell = "0.3"` and
> `crownos-config = "0.2"`, and neither version exists on crates.io — only
> `crownshell` 0.1.0 and 0.2.0 were ever published. Without the
> `[patch.crates-io]` overlay described in
> [Workspace setup](workspace-setup.md#the-overlay-mandatory), those five repos
> fail at `cargo metadata` before a single line is compiled. Run
> `crownos-setup`'s `./bootstrap.sh --dev` first; everything below assumes you
> have.

With the overlay in place and the pinned 1.88.0 toolchain, **all eleven Rust
crates** pass `cargo check --all-targets`, and all eleven pass
`cargo fmt --all --check`.

That is true of a **local working tree**, not of the default branch you would
get from GitHub today. Several of the fixes involved are still uncommitted.

Several components are still **Skeleton** — they build and do almost nothing —
and [Project status](../00-overview/project-status.md) says which.

`bootstrap.sh --dev` clones the eleven Rust repositories plus
`crownos-documentations` and `crownos-setup`. It does **not** clone
`crownos-iso`, `crownos-website`, `crowncrate-android` or `crowncrate-chrome`;
clone those by hand.

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
adapter. See [Prerequisites](prerequisites.md#every-crownshell-based-app).

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

That opens a window containing a complete CrownOS session, and creates its own
Wayland socket. **[Try it without installing](../60-users/try-it-without-installing.md)
is the full walkthrough** — finding the socket name, attaching clients to it,
and what to expect.

`crownpositor` logs through `tracing-subscriber`'s `fmt` layer, so its output
goes to **stderr** and `RUST_LOG` takes tracing's env-filter syntax, not
`env_logger`'s:

```bash
RUST_LOG=crownpositor=debug,smithay=warn CROWN_BACKEND=winit cargo run
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

The repo now carries a `build.sh` that works off Arch too — it runs `mkarchiso`
natively on Arch, and otherwise drives a privileged Arch container:

```bash
cd crownos-iso
./build.sh
```

**That script is a local, untracked addition.** On the default branch there is
no build script, and you invoke `mkarchiso` against the profile directory
yourself, which needs `archiso` and root on an Arch system:

```bash
sudo mkarchiso -v -w /tmp/crownos-work -o /tmp/crownos-out ./crownos-iso
```

Either way the profile is an unmodified copy of upstream Arch's `releng`, so
what comes out is a generic Arch rescue image named `archlinux`, with no CrownOS
packages and no CrownOS branding. See
[crownos-iso](../30-components/crownos-iso.md).

---

## Running the whole desktop

There is no single command that starts a full CrownOS session, and no session
file to select from a display manager. The compositor does spawn startup
programs itself from `compositor.startup`, and the `Compositor` schema in
`crownos-config` defines the matching `startup: Vec<String>` field — that half
is landed. What is missing is anything that starts the compositor in the first
place.

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
