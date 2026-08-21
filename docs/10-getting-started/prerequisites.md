# Prerequisites

What to install before building anything. None of this is documented in the
component repos, so read this page rather than guessing from compile errors.

---

## Toolchain

### Rust

Twelve of the sixteen repos are Rust. Every crate is **edition 2024**, which
requires **Rust 1.85 or newer**. Only `crownshell` declares this explicitly
(`rust-version = "1.85"`), and there is no `rust-toolchain.toml` anywhere, so
nothing will stop you using a toolchain that is too old — you will just get
confusing parse errors.

```bash
rustup toolchain install stable
rustup default stable
rustc --version    # must be >= 1.85
```

`crownpositor` uses let-chains and `resolver = "3"`, so a recent stable is
safest.

### Other toolchains

| Repo | Needs |
|---|---|
| `crownos-website` | [Bun](https://bun.sh) (the lockfile is `bun.lock`; do not use npm or yarn) |
| `crowncrate-android` | Android SDK 36, JDK 11+. The Gradle wrapper is committed — use `./gradlew`. |
| `crownos-iso` | `archiso`, and root on an Arch system |

---

## Native dependencies

The Rust crates link against a lot of system libraries through `-sys` crates.
`pkg-config` resolves all of them, so install it first.

### Everything (any crownshell-based app)

Required by `crownshell`, `crownbar`, `crowndock`, `crownotify`,
`crowndictator`, `crownuikit`, `crownos-config`.

**Arch:**

```bash
sudo pacman -S --needed base-devel pkgconf \
  wayland wayland-protocols libxkbcommon \
  vulkan-icd-loader vulkan-headers mesa libglvnd \
  fontconfig freetype2 \
  dbus bluez bluez-libs \
  libxcb libx11
```

**Debian / Ubuntu:**

```bash
sudo apt install build-essential pkg-config \
  libwayland-dev wayland-protocols libxkbcommon-dev \
  libvulkan-dev mesa-common-dev libegl1-mesa-dev libgbm-dev \
  libfontconfig-1-dev libfreetype-dev \
  libdbus-1-dev libbluetooth-dev \
  libxcb1-dev libx11-dev
```

You also need a **working Vulkan or GL driver** — wgpu needs a real GPU
adapter. In a VM, install the appropriate guest drivers or expect
`crownshell`-based apps to fail at surface creation.

> **Note on `bluer`.** `crownshell` depends on `bluer` with the `bluetoothd`,
> `id` and `rfcomm` features, which drags in the whole D-Bus and BlueZ tree.
> Nothing in `crownshell`'s source actually uses it — it is a leftover from when
> `crownbar`'s code lived there. You still need the libraries to link.

### crownpositor (the compositor)

Everything above, plus:

**Arch:**

```bash
sudo pacman -S --needed \
  libdrm libinput seatd systemd-libs pixman \
  xorg-xwayland
```

**Debian / Ubuntu:**

```bash
sudo apt install \
  libdrm-dev libinput-dev libseat-dev libudev-dev libpixman-1-dev \
  xwayland
```

Two things worth knowing:

- **`libdisplay-info` is deliberately excluded.** `crownpositor` declares
  `smithay-drm-extras` with `default-features = false` because the
  `display-info` sys crate does not accept the version of `libdisplay-info`
  shipped on current systems. EDID make/model naming is off until that catches
  up. Do not "fix" this by re-enabling the feature.
- **Running on real hardware** (the KMS backend) needs a seat. `seatd` must be
  running, or you must be on a systemd-logind session. For development, use the
  nested `winit` backend instead — see [Build and run](build-and-run.md).

### crownbar

`crownbar` forces the BFD linker via a committed `.cargo/config.toml`:

```toml
[target.x86_64-unknown-linux-gnu]
rustflags = ["-C", "link-arg=-fuse-ld=bfd"]
```

So you need `ld.bfd` on `PATH` — it ships in `binutils`, which you almost
certainly have. If you use `mold` or `lld` globally, note that this file
overrides it for this crate only.

### crowndictator

The heaviest prerequisites in the project.

**Arch:**

```bash
sudo pacman -S --needed alsa-lib libevdev
# Optional but recommended, for the injection fallback chain:
sudo pacman -S --needed wtype ydotool wl-clipboard libnotify
```

**Debian / Ubuntu:**

```bash
sudo apt install libasound2-dev libevdev-dev wtype wl-clipboard libnotify-bin
```

Beyond packages:

- **`/dev/input` read access.** The global hotkey is detected by reading
  `/dev/input/event*` directly with evdev, which means your user must be in the
  `input` group:
  ```bash
  sudo usermod -aG input "$USER"   # log out and back in
  ```
- **ONNX Runtime with CUDA.** The `ort` dependency is pinned exactly
  (`= "2.0.0-rc.12"`) and the `cuda` feature is **not optional**. On a machine
  without an NVIDIA stack this is still a hard build dependency, though the
  runtime falls back to CPU.
- **A large first-run download.** Model weights come from Hugging Face on first
  use: roughly **700 MB** for the int8 (CPU) model, **2.5 GB** for fp32 (GPU).
  Use `cargo run -- --demo` to work on the UI without downloading anything.

### crowncrate-linux

```bash
# Arch
sudo pacman -S --needed gtk4 glib2 pango gdk-pixbuf2 graphene

# Debian / Ubuntu
sudo apt install libgtk-4-dev libglib2.0-dev libpango1.0-dev \
  libgdk-pixbuf-2.0-dev libgraphene-1.0-dev
```

Needs GTK4 **4.10 or newer** (`features = ["v4_10"]`).

> This crate does not currently compile, and it has a glib 0.17/0.18 version
> conflict on top of the compile errors. See
> [Project status](../00-overview/project-status.md#crowncrate-linux-does-not-build).

### Testing extras

`crownotify`'s tests need a live D-Bus session bus:

```bash
# Arch
sudo pacman -S --needed dbus
# Debian / Ubuntu
sudo apt install dbus
```

Then run them under `dbus-run-session` — see [Testing](../40-contributing/testing.md).

---

## What you do *not* need

Worth stating, because they are reasonable guesses:

- **PipeWire / PulseAudio dev headers.** No `pipewire` or `libspa` crate appears
  in any lockfile. Audio capture is ALSA-only via `cpal`. (`crownbar` shells out
  to the `wpctl` and `pactl` *binaries* for volume, but does not link against
  them.)
- **GTK3 or Qt.** Not used anywhere.
- **`libdisplay-info`.** Explicitly avoided — see above.

---

## Runtime requirements

To actually *run* the shell components you need a Wayland compositor that
supports `wlr-layer-shell`. That includes `crownpositor` itself, and also
Hyprland, Sway, river, KWin and most wlroots-based compositors — so you can
develop `crownbar`, `crowndock` and `crownotify` inside your existing desktop
without building the compositor at all.

---

## Next

[Workspace setup](workspace-setup.md) — how to lay out your checkouts. Get this
wrong and several crates will not build.
