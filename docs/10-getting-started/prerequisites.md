# Prerequisites

What to install before building anything.

## The short way

```bash
git clone https://github.com/Crown-OS/crownos-setup && cd crownos-setup
./bootstrap.sh --check      # what is missing on this machine
./bootstrap.sh --dev        # install it, clone the repos, set up the overlay
```

[`crownos-setup`](https://github.com/Crown-OS/crownos-setup) works **irrespective
of your distribution**, in three layers it falls through automatically:

1. **The system package manager.** Detected from `/etc/os-release` (`ID`, then
   `ID_LIKE`, so derivatives resolve without being listed), and failing that by
   probing for the package-manager binary. Arch, Debian/Ubuntu, Fedora/RHEL,
   openSUSE, Alpine, Void and Gentoo have package names recorded.
2. **Nix**, which needs no distro support and no root for the shell itself. With
   `--yes` the script installs it rather than giving up.
3. **A container** reproducing CI exactly.

Whether it worked is decided by **`pkg-config`**, not by the package manager's
exit code. Module names — `wayland-client`, `libseat`, `pixman-1` and 24 others —
are identical on every distro, so `--check` is correct even on a system with no
package-name mapping, and a wrong package name is caught rather than producing a
broken build. Two were wrong before this check existed: `libfontconfig-1-dev`,
which exists on neither Debian nor Ubuntu and was in CI too, and openSUSE's
`libseat-devel`, which is really `seatd-devel`.

**The package lists live on their own page:**
[Native packages, per distribution](native-packages.md). That page is generated
from `deps.toml`, so it cannot drift from the script or from CI. Arch,
Debian/Ubuntu, Fedora, openSUSE, Alpine, Void and Gentoo are covered there.

This page is the part a generator cannot produce: *why* each dependency is
needed, and the traps that are not package names.

---

## Toolchain

### Rust

Twelve of the seventeen repos are Rust. Every crate is **edition 2024**, and the
minimum toolchain is **Rust 1.88**.

Edition 2024 itself only needs 1.85, and 1.85 is what the repos used to declare.
That was wrong: `vello 0.9` and `xilem 0.4` both declare `rust-version = "1.88"`,
and `wgpu 29` and `zbus 5.16` declare 1.87. The dependency graph sets the floor,
not the edition.

Every Rust repo now pins the toolchain in `rust-toolchain.toml`, so rustup
installs the right one automatically the first time you build.

```bash
rustup toolchain install stable
rustup default stable
rustc --version    # must be >= 1.88
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

The Rust crates link against system libraries through `-sys` crates, and
`pkg-config` resolves all of them.

**Package names:** [Native packages, per distribution](native-packages.md), or
just run `./bootstrap.sh --dev`.

Below is what those lists do not tell you.

### Every crownshell-based app

`crownshell`, `crownbar`, `crowndock`, `crownotify`, `crowndictator`,
`crownuikit`, `crownos-config`.

You also need a **working Vulkan or GL driver** — wgpu needs a real GPU adapter.
In a VM, install the guest drivers or expect `crownshell`-based apps to fail at
surface creation. No package list can check this for you.

> **`bluer` is dead weight.** `crownshell` depends on it with the `bluetoothd`,
> `id` and `rfcomm` features, dragging in the whole D-Bus and BlueZ tree.
> Nothing in `crownshell`'s source uses it — a leftover from when `crownbar`'s
> code lived there. You still need the libraries to link. Dropping the
> dependency would shorten the list for every downstream repo.

### crownpositor (the compositor)

- **`libdisplay-info` is deliberately excluded.** `crownpositor` declares
  `smithay-drm-extras` with `default-features = false` because the
  `display-info` sys crate does not accept the `libdisplay-info` shipped on
  current systems. EDID make/model naming is off until that catches up. Do not
  "fix" this by re-enabling the feature.
- **Real hardware needs a seat.** The KMS backend requires `seatd` running or a
  systemd-logind session. For development use the nested backend:
  `CROWN_BACKEND=winit cargo run`. See [Build and run](build-and-run.md).

### crownbar

`crownbar` forces the BFD linker via a committed `.cargo/config.toml`:

```toml
[target.x86_64-unknown-linux-gnu]
rustflags = ["-C", "link-arg=-fuse-ld=bfd"]
```

So you need `ld.bfd` on `PATH` — it ships in `binutils`, which you almost
certainly have. If you use `mold` or `lld` globally, this file overrides it for
this crate only.

### crowndictator

The heaviest prerequisites in the project, and the ones least visible from a
package list.

- **`/dev/input` read access.** The global hotkey reads `/dev/input/event*`
  directly with evdev, so your user must be in the `input` group:
  ```bash
  sudo usermod -aG input "$USER"   # log out and back in
  ```
- **A build-time download.** `ort` is declared with the `cuda` feature and
  *without* `default-features = false`, so `ort-sys` fetches a prebuilt ONNX
  Runtime during `cargo build`. The `cuda` feature is not optional, so this is a
  hard build dependency even with no NVIDIA stack — though the runtime falls
  back to CPU. It is also why `crowndictator`'s docs.rs build fails: docs.rs
  blocks network access.
- **A large first-run download.** Model weights from Hugging Face on first use:
  roughly **700 MB** int8 (CPU), **2.5 GB** fp32 (GPU), into
  `~/.cache/huggingface/hub`. Which one depends on whether CUDA initialises.
  `cargo run -- --demo` exercises the UI without downloading anything.
- **At least one text-injection helper** — `wtype`, `ydotool` or `wl-clipboard`.
  `./bootstrap.sh --check` reports if none is present.

### crowncrate-linux

Needs GTK4 **4.10 or newer** (`features = ["v4_10"]`).

> The crate compiles, but implements almost nothing — `src/lib.rs` and
> `src/ui/mod.rs` are empty. It also opens an **unauthenticated** TCP listener;
> see [SECURITY.md](../../SECURITY.md) before running it.

### Testing extras

`crownotify`'s tests need a live D-Bus session bus, then run under
`dbus-run-session` — see [Testing](../40-contributing/testing.md).

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
