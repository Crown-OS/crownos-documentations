# Component map

All sixteen repositories in the [Crown-OS](https://github.com/Crown-OS)
organization, plus `crowncrate-chrome`, which is not in the listing.

## Status vocabulary

Used consistently across all documentation.

| Marker | Meaning |
|---|---|
| **Stable** | Builds, has tests, API is settling |
| **Early** | Builds and runs; API will move |
| **Partial** | Builds, but a headline feature is a no-op |
| **Skeleton** | Scaffolding only — little or no implementation |
| **Broken** | Does not currently compile |
| **Empty** | Repository has no commits |

---

## Desktop shell

| Repo | Status | Purpose | Default branch |
|---|---|---|---|
| [crownpositor](../30-components/crownpositor.md) | **Early** | Wayland compositor. Smithay-based, tiling, DRM/KMS + winit backends. The largest codebase in the org. | `main` |
| [crownshell](../30-components/crownshell.md) | **Early** | Layer-shell framework. Wraps Wayland boilerplate, paints with Vello, lays out text with Parley. Everything below is built on it. | `main` |
| [crownbar](../30-components/crownbar.md) | **Partial** | Status bar — clock, battery, wifi, bluetooth, brightness, volume. Reads `/sys` directly. | `main` |
| [crowndock](../30-components/crowndock.md) | **Partial** | Auto-hiding dock with drag-and-drop pinning. **Cannot launch applications.** | `main` |
| [crownotify](../30-components/crownotify.md) | **Partial** | Notification daemon. Implements `org.freedesktop.Notifications` plus a CrownOS interface. The notification centre is still a no-op. | `main` |
| [crowndictator](../30-components/crowndictator.md) | **Early** | Push-to-talk voice dictation. Local ASR via ONNX Runtime. Needs OpenSSL headers as well as ALSA and evdev. | `main` |
| [crownlauncher](../30-components/crownlauncher.md) | **Skeleton** | App launcher. Currently `cargo new` output. | `main` |
| [crownuikit](../30-components/crownuikit.md) | **Early** | Widget kit for [xilem](https://github.com/linebender/xilem) — sidebar, sliders, toggles, selects. Intended for the settings panel; nothing depends on it yet. | `main` |

## Platform

| Repo | Status | Purpose | Default branch |
|---|---|---|---|
| [crownos-config](../30-components/crownos-config.md) | **Stable** | Shared settings. RON files in `~/.config/crownos/`, live-reloaded with inotify. **This is how components coordinate.** | `main` |
| [crownos-iso](../30-components/crownos-iso.md) | **Skeleton** | archiso profile. Currently an unmodified upstream Arch `releng`, 128 packages, no CrownOS content. | `main` |

## Ecosystem (phone bridge)

| Repo | Status | Purpose | Default branch |
|---|---|---|---|
| [crowncrate-linux](../30-components/crowncrate-linux.md) | **Skeleton** | Desktop side of the phone bridge. TCP + CBOR on port 5252. | `main` |
| [crowncrate-android](../30-components/crowncrate-android.md) | **Skeleton** | Android companion app. Currently an unmodified Android Studio template. | `main` |
| [crowncrate-chrome](../30-components/crowncrate-chrome.md) | **Empty** | Planned browser extension for OTP sync. Zero commits, zero objects; does not appear in the org's repository listing. | — |
| [lls-protocol](../30-components/lls-protocol.md) | **Skeleton** | Low-latency media streaming protocol (screen mirroring, second screen). RTP-shaped, UDP. | `main` |

## Project

| Repo | Status | Purpose | Default branch |
|---|---|---|---|
| [crownos-website](../30-components/crownos-website.md) | **Early** | Landing page. Next.js + Tailwind v4 + Biome, built with Bun. | `main` |
| [crownos-documentations](../30-components/crownos-documentations.md) | **Early** | This repository. | `main` |
| [crownos-setup](https://github.com/Crown-OS/crownos-setup) | **Stable** | Cross-distro native-dependency manifest, bootstrap script, Nix flake and CI container image. `deps.toml` is the single source for the package lists used by the script, CI and the docs. | `main` |

---

## Dependency graph

Five crates depend on other CrownOS crates. Everything else is independent.

```
crownshell "0.3" ──► crownbar
                 ──► crowndock
                 ──► crownotify
                 ──► crowndictator

crownos-config "0.2" ──► crowndictator
                     ──► crownpositor        (default-features = false)
                     ──► crownpositor-config (default-features = false)
                                │
                                └──(path "../config")──► crownpositor

crownuikit       ── no CrownOS dependencies (xilem/winit only); nothing depends on it
crowncrate-linux ── no CrownOS dependencies
lls-client       ── no CrownOS dependencies
lls-server       ── no CrownOS dependencies
crownlauncher    ── no dependencies at all
```

Every cross-repository edge is now a **version requirement** — no `path`, no git
URLs. That is deliberate, and recent. Until August 2026:

1. `crownotify`, `crowndictator` and `crownpositor` used relative `path`
   dependencies, so they built against whatever was in your working tree and
   carried no version at all.
2. `crownbar` and `crowndock` pinned `crownshell` by git URL with **no rev or
   tag**. Their lockfiles sat at `crownshell` 0.1.0, eight commits behind, and a
   `cargo update` would have moved them onto HEAD.

**But neither version exists on crates.io.** `crownshell` is published at 0.1.0
and 0.2.0 only; `crownos-config` has never been published. So those five repos
resolve nothing from a clean clone — they fail at `cargo metadata`. What supplies
the crates is a `[patch.crates-io]` overlay above the checkouts, written by
`crownos-setup`'s `./bootstrap.sh --dev`. It is mandatory. See
[Workspace setup](../10-getting-started/workspace-setup.md#the-overlay-mandatory).

The one remaining `path` edge, `crownpositor` → `crownpositor-config`, is inside
a single repository's own workspace.

Detail: [Dependency graph](../20-architecture/dependency-graph.md).

---

## Runtime relationships

At runtime the picture is simpler than the build graph. `crownpositor` is the
Wayland server; everything else in the shell is a client that attaches to it via
`wlr-layer-shell`, and they coordinate through `~/.config/crownos/*.ron`.

The only D-Bus in the desktop is `crownotify`, which owns
`org.freedesktop.Notifications` and `io.crownos.crownotify`, and calls out to
`io.crownos.crowncrate` for call pickup and decline.

See [Architecture overview](../20-architecture/overview.md) and
[IPC and protocols](../20-architecture/ipc-and-protocols.md).

---

## Languages and toolchains

| Language | Repos | Toolchain |
|---|---|---|
| Rust | 11 | Edition 2024, minimum Rust **1.88** (set by `vello 0.9`/`xilem 0.4`, not by the edition). Pinned per repo in `rust-toolchain.toml`. |
| Kotlin | 1 | AGP 8.13.1, Kotlin 2.0.21, compileSdk 36, minSdk 29, JVM 11 |
| TypeScript | 1 | Bun, Next.js 16, React 19, Tailwind v4, Biome 2.2 |
| Shell / archiso | 1 | `archiso`, run as root on Arch |
| Markdown | 1 | None |
