# Installing CrownOS

**You cannot install CrownOS today.** This page explains why, and what you can
run instead.

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
list is upstream's 127-package rescue set — **no compositor, no Wayland stack, no
CrownOS component**. `pacman.conf` enables only `[core]` and `[extra]`; there is
no CrownOS repository.

Building it produces a generic Arch rescue image.

There are no PKGBUILDs, no AUR entries and no `.deb`. What there *is* is
crates.io — see [Install with cargo](#install-with-cargo) below.

### About the download page

The CrownOS website advertises three editions — Desktop 2.4 GB, Minimal 780 MB,
ARM aarch64 2.1 GB — across five mirrors at `dl.crownos.org` and
`{eu,na,ap,sa}.mirror.crownos.org`.

**None of those exist.** They are placeholder copy. See
[Project status](../00-overview/project-status.md#documentation-drift).

---

## Install with cargo

The quickest path on any distribution. `crownos-setup` installs the system
libraries CrownOS links against, then `cargo install`s the components:

```bash
git clone https://github.com/Crown-OS/crownos-setup && cd crownos-setup
./bootstrap.sh --user
```

It reads `/etc/os-release` and dispatches to `pacman`, `apt`, `dnf` or `zypper`,
so Arch, Debian, Ubuntu, Fedora, openSUSE and their derivatives all work. There
is also a Nix flake (`nix develop github:Crown-OS/crownos-setup`) and a container
image. See [Prerequisites](../10-getting-started/prerequisites.md).

To inspect your machine without changing it:

```bash
./bootstrap.sh --check
./bootstrap.sh --user --dry-run
```

If you would rather do it by hand, install the native dependencies for your
distro and then:

```bash
cargo install crownbar crowndock crownotify crowndictator
```

Do **not** skip the native dependencies. `cargo install` compiles from source and
will fail at link time without them — `crownpositor` in particular needs libdrm,
libinput, libseat, libudev and pixman.

> **`crowndictator` downloads a lot.** Its build fetches a prebuilt ONNX Runtime,
> and its first run fetches 700 MB (CPU) or 2.5 GB (GPU) of model weights into
> `~/.cache/huggingface/hub`. `crowndictator --demo` skips the model entirely.

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
| [`crownotify`](../30-components/crownotify.md) | A notification daemon | Does not currently build |

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

cd crowndictator
cargo run -- --demo     # try the UI with no model download
cargo run               # the real daemon — first run downloads 700 MB–2.5 GB
```

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
[Keybindings](../50-reference/keybindings.md) has the full table.

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
7. **Resolve licensing** — the profile is GPL-3.0 upstream content in a repo with
   no LICENSE file.

If you want to help with any of that, see
[Your first change](../10-getting-started/your-first-change.md).

---

## See also

- [Known limitations](known-limitations.md) — what to expect if you run it anyway
- [Project status](../00-overview/project-status.md) — component-by-component
- [SECURITY.md](../../SECURITY.md) — known security posture
