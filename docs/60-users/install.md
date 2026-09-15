# Installing CrownOS

**You cannot install CrownOS today.** There is no image, no package, and nothing
published to crates.io. This page explains why, and what you can run instead.

If you just want to *see* CrownOS, skip to
[Try it without installing](try-it-without-installing.md) — a nested compositor
session is the only way anyone can run it right now.

---

## There is no installable image

[`crownos-iso`](../30-components/crownos-iso.md) is an archiso profile, and it is
currently an **unmodified copy of the upstream Arch Linux `releng` profile**:

```sh
iso_name="archlinux"
iso_publisher="Arch Linux <https://archlinux.org>"
iso_application="Arch Linux Live/Rescue DVD"
```

The hostname is `archiso`. The motd points at the Arch install guide. The package
list is upstream's 128-package rescue set — **no compositor, no Wayland stack, no
CrownOS component**. `pacman.conf` enables only `[core]` and `[extra]`; there is
no CrownOS repository.

Building it produces a generic Arch rescue image.

There are no PKGBUILDs, no AUR entries and no `.deb`. There is nothing on
crates.io either: the only CrownOS crates ever published are `crownshell` 0.1.0
and 0.2.0, a library with no binaries. Every runnable component has to be built
from a checkout — see [Building from source](#building-from-source) below.

### About the download page

The CrownOS website advertises three editions — Desktop 2.4 GB, Minimal 780 MB,
ARM aarch64 2.1 GB — across five mirrors at `dl.crownos.org` and
`{eu,na,ap,sa}.mirror.crownos.org`.

**None of those exist.** They are placeholder copy. See
[Project status](../00-overview/project-status.md#documentation-drift).

---

## Building from source

This is the only path. `cargo install crownbar` and friends **cannot work** —
none of those crates has ever been published, so there is nothing for cargo to
fetch. `crownos-setup`'s `./bootstrap.sh --user` mode issues exactly those
`cargo install` commands and will fail for the same reason; use `--dev`.

```bash
git clone https://github.com/Crown-OS/crownos-setup && cd crownos-setup
./bootstrap.sh --dev
```

That does three things: installs the system libraries CrownOS links against,
clones the eleven Rust repositories into `~/src/crownos`, and writes the
`[patch.crates-io]` overlay without which five of them do not resolve their
dependencies at all. See
[Workspace setup](../10-getting-started/workspace-setup.md#the-overlay-mandatory)
for what the overlay is and why it is not optional.

It reads `/etc/os-release` and dispatches to `pacman`, `apt`, `dnf`, `zypper`,
`apk`, `xbps` or `emerge`, so Arch, Debian, Ubuntu, Fedora, openSUSE, Alpine,
Void, Gentoo and their derivatives all work. There is also a Nix flake
(`nix develop github:Crown-OS/crownos-setup`) and a container image. See
[Prerequisites](../10-getting-started/prerequisites.md).

To inspect your machine without changing it:

```bash
./bootstrap.sh --check
./bootstrap.sh --dev --dry-run
```

Do **not** skip the native dependencies. Everything compiles from source and
will fail at link time without them — `crownpositor` in particular needs libdrm,
libinput, libseat, libudev and pixman.

> **`crowndictator` downloads a lot.** Its build fetches a prebuilt ONNX Runtime,
> and its first run fetches 700 MB (CPU) or 2.5 GB (GPU) of model weights into
> `~/.cache/huggingface/hub`. `crowndictator -- --demo` skips the model entirely.
> Its `ort` dependency also reaches `openssl-sys`, so it needs OpenSSL
> development headers. `deps.toml` covers that now; install by hand only if you
> are working from an older package list.

---

## What you *can* run

Individual components work, and several are worth running on an existing Linux
system. All of them need a Wayland compositor supporting `wlr-layer-shell` —
Hyprland, Sway, river, KWin and most wlroots-based compositors qualify, so you do
not need `crownpositor`.

| Component | What you get | Status |
|---|---|---|
| [`crowndictator`](../30-components/crowndictator.md) | Local push-to-talk voice dictation | Genuinely usable |
| [`crownbar`](../30-components/crownbar.md) | A status bar | Usable; no configuration |
| [`crownpositor`](../30-components/crownpositor.md) | A tiling compositor, nested or on hardware | Early but functional |
| [`crowndock`](../30-components/crowndock.md) | A dock | Cannot launch applications |
| [`crownotify`](../30-components/crownotify.md) | A notification daemon | Builds; the notification centre is a no-op |

Everything is built from source. See
[Prerequisites](../10-getting-started/prerequisites.md) and
[Build and run](../10-getting-started/build-and-run.md).

### The quickest thing worth trying

`crowndictator` is the component most likely to be immediately useful — local,
offline speech-to-text on a push-to-talk key.

```bash
mkdir -p ~/src/crownos && cd ~/src/crownos
git clone git@github.com:Crown-OS/crownshell.git
git clone git@github.com:Crown-OS/crownos-config.git
git clone git@github.com:Crown-OS/crowndictator.git

# Without this file, crowndictator fails at `cargo metadata`: it asks for
# crownshell 0.3 and crownos-config 0.2, and neither exists on crates.io.
mkdir -p .cargo
cat > .cargo/config.toml <<'EOF'
[patch.crates-io]
crownshell = { path = "crownshell" }
crownos-config = { path = "crownos-config" }
EOF

cd crowndictator
cargo run -- --demo     # try the UI with no model download
cargo run               # the real daemon — first run downloads 700 MB–2.5 GB
```

`./bootstrap.sh --dev` writes that same file for you and clones the rest.

Be aware of what it needs: `input` group membership, an ALSA-capable microphone,
`wtype` or `ydotool` or `wl-clipboard`, and a large first-run model download from
Hugging Face. Details in
[Prerequisites](../10-getting-started/prerequisites.md#crowndictator).

### Trying the compositor safely

```bash
cd crownpositor
CROWN_BACKEND=winit cargo run
```

This opens a window containing a full CrownOS session, nested inside your
existing desktop. `Super+Return` spawns `foot`; `Super+Shift+E` quits.

**[Try it without installing](try-it-without-installing.md)** walks through the
whole thing — finding the nested session's Wayland socket, attaching a bar and a
dock to it, and where the logs go.

Running it on real hardware (`CROWN_BACKEND=kms`, from a bare TTY) works, but
have a second TTY or an SSH session available before you try.

---

## Configuring what you run

Settings live in `~/.config/crownos/<section>.ron`. Files are created with
defaults the first time a component starts, so run it once and then edit.

```bash
$EDITOR ~/.config/crownos/compositor.ron
$EDITOR ~/.config/crownos/appearance.ron
```

Changes apply live for `crownpositor` and `crowndictator` — no restart.

Two things to know:

- A file that fails to parse is **left alone**, and defaults are used. Your
  half-finished edit will not be overwritten.
- Not every setting has an effect yet. `sound.ron`, `wifi.ron`,
  `bluetooth.ron`, `power.ron` and `keybinds.ron` have no reader at all, and
  `crownbar` and `crownotify` ignore their sections.

Full reference: [Configuration schema](../50-reference/config-schema.md).

---

## What would have to happen for an installable CrownOS

Roughly, in order:

1. **Package the components.** No PKGBUILDs exist. This is the gate on
   everything else.
2. **Brand the ISO profile** — name, label, publisher, hostname, motd.
3. **Add a CrownOS pacman repository** or build packages into the profile.
4. **Add the Wayland stack** to the package list.
5. **Provide a session** — there is no session file, no greeter, no `.desktop`
   entry for the compositor.
6. **Fix the live-medium security defaults** — the inherited profile has an empty
   root password, root autologin and permissive sshd, which are fine for a rescue
   ISO and not for an installed system.
7. **Resolve licensing** — the profile is GPL-3.0 upstream content, and only
   three of the sixteen repositories carry a LICENSE on their default branch.

If you want to help with any of that, see
[Your first change](../10-getting-started/your-first-change.md).

---

## See also

- [Try it without installing](try-it-without-installing.md) — the nested session
- [Troubleshooting](troubleshooting.md) — symptoms and their causes
- [Known limitations](known-limitations.md) — what to expect if you run it anyway
- [Project status](../00-overview/project-status.md) — component-by-component
- [SECURITY.md](../../SECURITY.md) — known security posture
