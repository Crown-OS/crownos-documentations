# Troubleshooting

Symptoms you are likely to hit, and what causes them. Most of these are not bugs
in your setup.

---

## Quick table

| Symptom | Cause | Fix |
|---|---|---|
| `failed to select a version for the requirement crownshell = "^0.3"` | The `[patch.crates-io]` overlay is missing. `crownshell` 0.3 and `crownos-config` 0.2 do not exist on crates.io. | [Write the overlay](../10-getting-started/workspace-setup.md#the-overlay-mandatory), or run `./bootstrap.sh --dev` |
| `Patch … was not used in the crate graph` | The overlay patches two crates; this repo depends on one of them, or neither. | Nothing. It is harmless noise. |
| `cargo build --locked` fails on a fresh clone | Every committed `Cargo.lock` was generated inside the patched tree — no `source =` line, plus `[[patch.unused]]` stanzas. | Build without `--locked`. |
| A `crownshell` app exits at surface creation | No working Vulkan or GL adapter. Common in VMs without GPU passthrough. | Install guest GPU drivers, or run on real hardware. |
| `crowndictator` fails to build in `openssl-sys` | `ort` reaches `openssl-sys` through `ureq` and `native-tls`. OpenSSL headers are in `deps.toml` now, but not in older package lists. | Install `openssl` / `libssl-dev` / `openssl-devel`, or re-run `./bootstrap.sh`. |
| `crownotify` exits: D-Bus name already taken | Another notification daemon owns `org.freedesktop.Notifications` — `mako`, `dunst`, GNOME Shell, Plasma. | Stop the other daemon first. Only one can hold the name. |
| `crowndictator`'s hotkey does nothing | Your user is not in the `input` group. It reads `/dev/input/event*` directly with evdev. | `sudo usermod -aG input "$USER"`, then log out and back in. |
| `crowndictator` appears to hang on first run | It is downloading model weights: 700 MB (int8/CPU) or 2.5 GB (fp32/GPU). There is no progress output. | Wait, or watch `~/.cache/huggingface/hub` grow. `cargo run -- --demo` skips it entirely. |
| A config change did nothing, and nothing was logged | A keybind chord failed to parse. The whole section silently reverts to defaults. | See [below](#a-config-change-silently-did-nothing). |
| `crowndock` produces no output at all | It never calls a logger init, so its `log::warn!` calls go nowhere. | Not fixable from outside. Add the init if you are debugging it. |
| No blur anywhere | `crownpositor`'s `ext-background-effect-v1` handler is entirely commented out. | Not fixable today. |

---

## `failed to select a version for crownshell ^0.3`

The single most common first failure. In full:

```
error: failed to select a version for the requirement `crownshell = "^0.3"`
candidate versions found which didn't match: 0.2.0, 0.1.0
```

`crownbar`, `crowndock`, `crownotify`, `crowndictator` and `crownpositor` all
declare version requirements that crates.io cannot satisfy — `crownshell` was
published at 0.1.0 and 0.2.0 only, and `crownos-config` has never been published
at all. Nothing compiles; this happens during `cargo metadata`, before the build
starts.

The fix is a `.cargo/config.toml` **above** your checkouts:

```toml
[patch.crates-io]
crownshell = { path = "crownshell" }
crownos-config = { path = "crownos-config" }
```

with `crownshell/` and `crownos-config/` cloned as siblings. `crownos-setup`'s
`./bootstrap.sh --dev` writes it. Do not fix this by editing a `Cargo.toml` —
that file is committed, and a local edit to it is how checkouts drift apart.

---

## A `crownshell` app exits at surface creation

`crownshell`, and therefore `crownbar`, `crowndock`, `crownotify` and
`crowndictator`, paint with Vello on top of wgpu. wgpu requires a **real GPU
adapter**: Vulkan, or GL through EGL.

This is the failure you get in a virtual machine with no GPU passthrough and no
guest drivers, and it is not something a package list can detect for you — the
packages install cleanly and the adapter still is not there.

Check whether your system has one at all:

```bash
vulkaninfo --summary     # from vulkan-tools
eglinfo                  # from mesa-utils / mesa-demos
```

For `crownpositor` specifically you can try a different renderer before giving
up:

```bash
CROWN_RENDER_API=vulkan cargo run    # or: egl / gles / gles3
```

An unrecognised value logs a warning and falls back rather than failing, so
check the log if a setting seems to be ignored.

---

## `crownotify` will not start: the D-Bus name is taken

`crownotify` owns `org.freedesktop.Notifications`. That name is exclusive — one
process on the session bus, no queueing. If you already run `mako`, `dunst`,
`swaync`, GNOME Shell or Plasma, that process holds it and `crownotify` cannot
start.

```bash
busctl --user get-property org.freedesktop.DBus \
  /org/freedesktop/DBus org.freedesktop.DBus GetNameOwner 2>/dev/null
busctl --user list | grep -i notification
```

Stop the incumbent, then start `crownotify`. Inside a nested session this bites
particularly often, because the host desktop's notification daemon is still
running.

---

## `crowndictator`: the hotkey does nothing

The global push-to-talk hotkey is read from `/dev/input/event*` with evdev, not
through the compositor. That requires read access to those devices, which on
almost every distribution means membership of the `input` group:

```bash
sudo usermod -aG input "$USER"
```

Group membership is established at login, so **log out and back in** — `newgrp`
in one shell is not enough for a daemon you start elsewhere. Confirm with
`id -nG`.

Nothing about this failure is loud. The daemon starts, runs, and never sees a
key.

---

## `crowndictator`: the silent multi-gigabyte download

On its first real run, `crowndictator` fetches model weights from Hugging Face
into `~/.cache/huggingface/hub`:

- **~700 MB** for the int8 model, used when CUDA does not initialise
- **~2.5 GB** for the fp32 model, used when it does

Which one you get is decided at runtime, and there is no progress indicator, so
the process looks hung. On a slow connection this is tens of minutes.

There is also a **build-time** download: `ort` is declared with the `cuda`
feature and without `default-features = false`, so `ort-sys` fetches a prebuilt
ONNX Runtime during `cargo build`. That is why `crowndictator`'s docs.rs build
fails permanently — docs.rs blocks network access.

To exercise the whole overlay UI with no model and no microphone:

```bash
cargo run -- --demo
```

---

## A config change silently did nothing

This is the failure mode most likely to waste your afternoon, because there is
**no error message anywhere**.

Keybind chords are parsed leniently and fail silently. A chord that does not
parse makes the whole file fail to deserialize, and `load()` returns the
section's `Default` — so one typo in one binding reverts *every* setting in that
section, not just the binding.

Two things cause it:

1. **A key name that does not exist.** The parser matches labels like `A`,
   `F5`, `Space`, `Left`, `LeftBracket`. W3C `KeyboardEvent.code` spellings —
   `KeyA`, `ArrowLeft`, `Digit1` — do **not** parse. Accepted spellings differ
   slightly between `crownos-config`'s `Keybind` type and `crownpositor`'s own
   chord parser; both lists are in
   [Keybindings](../50-reference/keybindings.md) and
   [Configuration schema](../50-reference/config-schema.md).
2. **A missing field.** `Binding`'s `keys` and `action` are both mandatory —
   there is no serde default — so omitting either one takes the whole section
   down with it.

If a section stops behaving, restore its file from defaults by deleting it and
restarting the component, then reapply your changes one at a time.

A file that fails to parse is **left alone** on disk, so your edit is never
overwritten. It is simply ignored.

---

## Where the logs are

Different components log differently, which is itself a source of confusion:

| Component | Logger | `RUST_LOG` syntax | Default |
|---|---|---|---|
| `crownpositor` | `tracing-subscriber` `fmt` to stderr | tracing env-filter — `crownpositor=debug,smithay=warn` | whatever `fmt()` defaults to |
| `crownbar` | `env_logger` | `env_logger` — `debug`, or `crownbar=debug` | `info` |
| `crownotify` | `env_logger` | as above | `info` |
| `crowndictator` | `env_logger` | as above | `info` |
| `crowndock` | **none** | — | produces no output at all |

`crownpositor` declares `tracing-journald` in its manifest but never installs
that layer, so nothing it prints reaches the journal — read it from the terminal
you launched it in.

More detail, and what to include in a bug report:
[Testing and reporting](../40-contributing/testing-and-reporting.md).

---

## See also

- [Try it without installing](try-it-without-installing.md) — the nested session
- [Known limitations](known-limitations.md) — things that are absent by design
- [Project status](../00-overview/project-status.md) — what is and is not built
