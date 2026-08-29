# Native packages, per distribution

<!-- GENERATED -->
**This page is generated.** Do not edit it. It comes from
[`crownos-setup/deps.toml`](https://github.com/Crown-OS/crownos-setup/blob/main/deps.toml),
the same file that drives `bootstrap.sh` and the apt list in CI. Regenerate with
`python3 scripts/gen.py` in that repo and copy `generated/prerequisites.md` here.

You usually do not need this page — `./bootstrap.sh --dev` installs all of it.
It exists for people who prefer to install by hand, and so the lists are
reviewable.

For *why* each dependency is needed, and the traps that are not package
names, see [Prerequisites](prerequisites.md).

---

### Compiler toolchain and pkg-config

**Arch:**

```bash
sudo pacman -S --needed \
  base-devel \
  pkgconf
```

**Debian / Ubuntu:**

```bash
sudo apt install -y \
  build-essential \
  pkg-config
```

**Fedora:**

```bash
sudo dnf install -y \
  gcc \
  gcc-c++ \
  make \
  pkgconf-pkg-config
```

**openSUSE:**

```bash
sudo zypper install -y \
  gcc \
  gcc-c++ \
  make \
  pkgconf-pkg-config
```

**Alpine:**

```bash
sudo apk add \
  build-base \
  pkgconf
```

**Void:**

```bash
sudo xbps-install -Sy \
  base-devel \
  pkgconf
```

**Gentoo:**

```bash
sudo emerge --noreplace \
  dev-util/pkgconf
```

### Wayland client, GPU and font stack

**Arch:**

```bash
sudo pacman -S --needed \
  wayland \
  wayland-protocols \
  libxkbcommon \
  vulkan-icd-loader \
  vulkan-headers \
  mesa \
  libglvnd \
  fontconfig \
  freetype2 \
  dbus \
  bluez \
  bluez-libs \
  libxcb \
  libx11
```

**Debian / Ubuntu:**

```bash
sudo apt install -y \
  libwayland-dev \
  wayland-protocols \
  libxkbcommon-dev \
  libvulkan-dev \
  mesa-common-dev \
  libegl1-mesa-dev \
  libgbm-dev \
  libfontconfig-dev \
  libfreetype-dev \
  libdbus-1-dev \
  libbluetooth-dev \
  libxcb1-dev \
  libx11-dev
```

**Fedora:**

```bash
sudo dnf install -y \
  wayland-devel \
  wayland-protocols-devel \
  libxkbcommon-devel \
  vulkan-loader-devel \
  mesa-libEGL-devel \
  mesa-libgbm-devel \
  fontconfig-devel \
  freetype-devel \
  dbus-devel \
  bluez-libs-devel \
  libxcb-devel \
  libX11-devel
```

**openSUSE:**

```bash
sudo zypper install -y \
  wayland-devel \
  wayland-protocols-devel \
  libxkbcommon-devel \
  vulkan-devel \
  Mesa-libEGL-devel \
  libgbm-devel \
  fontconfig-devel \
  freetype2-devel \
  dbus-1-devel \
  bluez-devel \
  libxcb-devel \
  libX11-devel
```

**Alpine:**

```bash
sudo apk add \
  wayland-dev \
  wayland-protocols \
  libxkbcommon-dev \
  vulkan-loader-dev \
  mesa-dev \
  mesa-egl \
  mesa-gbm \
  fontconfig-dev \
  freetype-dev \
  dbus-dev \
  bluez-dev \
  libxcb-dev \
  libx11-dev
```

**Void:**

```bash
sudo xbps-install -Sy \
  wayland-devel \
  wayland-protocols \
  libxkbcommon-devel \
  Vulkan-Loader-devel \
  MesaLib-devel \
  fontconfig-devel \
  freetype-devel \
  dbus-devel \
  libbluetooth-devel \
  libxcb-devel \
  libX11-devel
```

**Gentoo:**

```bash
sudo emerge --noreplace \
  dev-libs/wayland \
  dev-libs/wayland-protocols \
  x11-libs/libxkbcommon \
  media-libs/vulkan-loader \
  media-libs/mesa \
  media-libs/fontconfig \
  media-libs/freetype \
  sys-apps/dbus \
  net-wireless/bluez \
  x11-libs/libxcb \
  x11-libs/libX11
```

A working Vulkan or GL driver is also required -- wgpu needs a real adapter.
In a VM, install guest drivers or crownshell apps fail at surface creation.

bluez is here only because crownshell declares `bluer` with the bluetoothd, id
and rfcomm features and never uses it. Dropping that dependency would shorten
this group for every downstream repo.

### Extra packages for crownpositor (DRM/KMS, seat, input)

**Arch:**

```bash
sudo pacman -S --needed \
  libdrm \
  libinput \
  seatd \
  systemd-libs \
  pixman \
  xorg-xwayland
```

**Debian / Ubuntu:**

```bash
sudo apt install -y \
  libdrm-dev \
  libinput-dev \
  libseat-dev \
  libudev-dev \
  libpixman-1-dev \
  xwayland
```

**Fedora:**

```bash
sudo dnf install -y \
  libdrm-devel \
  libinput-devel \
  libseat-devel \
  systemd-devel \
  pixman-devel \
  xorg-x11-server-Xwayland
```

**openSUSE:**

```bash
sudo zypper install -y \
  libdrm-devel \
  libinput-devel \
  seatd-devel \
  systemd-devel \
  libpixman-1-0-devel \
  xwayland
```

**Alpine:**

```bash
sudo apk add \
  libdrm-dev \
  libinput-dev \
  seatd-dev \
  eudev-dev \
  pixman-dev \
  xwayland
```

**Void:**

```bash
sudo xbps-install -Sy \
  libdrm-devel \
  libinput-devel \
  libseat-devel \
  eudev-libudev-devel \
  pixman-devel \
  xorg-server-xwayland
```

**Gentoo:**

```bash
sudo emerge --noreplace \
  x11-libs/libdrm \
  dev-libs/libinput \
  sys-auth/seatd \
  virtual/libudev \
  x11-libs/pixman \
  x11-base/xwayland
```

libdisplay-info is deliberately excluded. crownpositor declares
smithay-drm-extras with default-features = false because the display-info sys
crate rejects the libdisplay-info shipped on current systems. Do not re-enable.

The KMS backend needs a seat: seatd running, or a systemd-logind session. For
development use CROWN_BACKEND=winit instead.

xwayland was missing from CI's package list despite smithay's xwayland feature
being enabled -- one of the drifts this file exists to prevent.

### Extra packages for crowndictator (audio capture, hotkey, injection)

**Arch:**

```bash
sudo pacman -S --needed \
  alsa-lib \
  libevdev

# optional but recommended:
sudo pacman -S --needed wtype ydotool wl-clipboard libnotify
```

**Debian / Ubuntu:**

```bash
sudo apt install -y \
  libasound2-dev \
  libevdev-dev

# optional but recommended:
sudo apt install -y wtype wl-clipboard libnotify-bin
```

**Fedora:**

```bash
sudo dnf install -y \
  alsa-lib-devel \
  libevdev-devel

# optional but recommended:
sudo dnf install -y wtype wl-clipboard libnotify
```

**openSUSE:**

```bash
sudo zypper install -y \
  alsa-devel \
  libevdev-devel

# optional but recommended:
sudo zypper install -y wtype wl-clipboard libnotify-tools
```

**Alpine:**

```bash
sudo apk add \
  alsa-lib-dev \
  libevdev-dev
```

**Void:**

```bash
sudo xbps-install -Sy \
  alsa-lib-devel \
  libevdev-devel
```

**Gentoo:**

```bash
sudo emerge --noreplace \
  media-libs/alsa-lib \
  dev-libs/libevdev
```

Needs /dev/input read access for the global hotkey (evdev):
    sudo usermod -aG input "$USER"   # log out and back in

TWO NETWORK DOWNLOADS, neither obvious from the manifest:

  1. BUILD TIME. `ort` is declared with the cuda feature and without
     default-features = false, so ort-sys downloads a prebuilt ONNX Runtime
     during `cargo build`. This also makes the docs.rs build fail permanently,
     because docs.rs blocks network access.
  2. RUN TIME. 700 MB (int8) or 2.5 GB (fp32) of model weights from
     huggingface.co/istupakov/parakeet-tdt-0.6b-v2-onnx, into hf-hub's default
     cache at ~/.cache/huggingface/hub. Which variant depends on whether CUDA
     initialises. There is no HF_HOME override in CrownOS.

`cargo run -- --demo` exercises the UI without downloading the model.

### Extra packages for crowncrate-linux (GTK4)

**Arch:**

```bash
sudo pacman -S --needed \
  gtk4 \
  glib2 \
  pango \
  gdk-pixbuf2 \
  graphene
```

**Debian / Ubuntu:**

```bash
sudo apt install -y \
  libgtk-4-dev \
  libglib2.0-dev \
  libpango1.0-dev \
  libgdk-pixbuf-2.0-dev \
  libgraphene-1.0-dev
```

**Fedora:**

```bash
sudo dnf install -y \
  gtk4-devel \
  glib2-devel \
  pango-devel \
  gdk-pixbuf2-devel \
  graphene-devel
```

**openSUSE:**

```bash
sudo zypper install -y \
  gtk4-devel \
  glib2-devel \
  pango-devel \
  gdk-pixbuf-devel \
  libgraphene-devel
```

**Alpine:**

```bash
sudo apk add \
  gtk4.0-dev \
  glib-dev \
  pango-dev \
  gdk-pixbuf-dev \
  graphene-dev
```

**Void:**

```bash
sudo xbps-install -Sy \
  gtk4-devel \
  glib-devel \
  pango-devel \
  gdk-pixbuf-devel \
  graphene-devel
```

**Gentoo:**

```bash
sudo emerge --noreplace \
  gui-libs/gtk \
  dev-libs/glib \
  x11-libs/pango \
  x11-libs/gdk-pixbuf \
  media-libs/graphene
```

Needs GTK4 >= 4.10. This crate does not currently compile.

### Needed to run the test suites

**Arch:**

```bash
sudo pacman -S --needed \
  dbus
```

**Debian / Ubuntu:**

```bash
sudo apt install -y \
  dbus
```

**Fedora:**

```bash
sudo dnf install -y \
  dbus-daemon
```

**openSUSE:**

```bash
sudo zypper install -y \
  dbus-1
```

**Alpine:**

```bash
sudo apk add \
  dbus
```

**Void:**

```bash
sudo xbps-install -Sy \
  dbus
```

**Gentoo:**

```bash
sudo emerge --noreplace \
  sys-apps/dbus
```

crownotify's tests register real names on a session bus; run them under dbus-run-session.

### Distro-neutral verification

Package names differ per distro; pkg-config module names do not. `bootstrap.sh` verifies against these, which is why `--check` is correct even on a system with no package-name column.

| Group | pkg-config modules | binaries |
|---|---|---|
| `toolchain` | — | `cc`, `pkg-config` |
| `base` | `wayland-client`, `wayland-cursor`, `wayland-protocols`, `xkbcommon`, `vulkan`, `egl`, `gbm`, `fontconfig`, `freetype2`, `dbus-1`, `bluez`, `xcb`, `x11` | — |
| `compositor` | `libdrm`, `libinput`, `libseat`, `libudev`, `pixman-1`, `wayland-server` | — |
| `dictation` | `alsa`, `libevdev` | — |
| `crowncrate` | `gtk4`, `glib-2.0`, `pango`, `gdk-pixbuf-2.0`, `graphene-gobject-1.0` | — |
| `testing` | — | `dbus-run-session` |

### Which component needs which group

| Component | Groups |
|---|---|
| `crownshell` | `toolchain`, `base` |
| `crownos-config` | `toolchain`, `base` |
| `crownuikit` | `toolchain`, `base` |
| `crownbar` | `toolchain`, `base` |
| `crowndock` | `toolchain`, `base` |
| `crownotify` | `toolchain`, `base`, `testing` |
| `crowndictator` | `toolchain`, `base`, `dictation` |
| `crownpositor` | `toolchain`, `base`, `compositor` |
| `crowncrate-linux` | `toolchain`, `base`, `crowncrate` |
| `crownlauncher` | `toolchain` |
| `lls-protocol` | `toolchain` |
