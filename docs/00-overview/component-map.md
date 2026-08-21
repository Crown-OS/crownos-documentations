# Component map

Every repository in the [Crown-OS](https://github.com/Crown-OS) organization.

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
| [crownotify](../30-components/crownotify.md) | **Partial** | Notification daemon. Implements `org.freedesktop.Notifications` plus a CrownOS interface. Does not build against current `crownshell`. | `main` |
| [crowndictator](../30-components/crowndictator.md) | **Early** | Push-to-talk voice dictation. Local ASR via ONNX Runtime. | `main` |
| [crownlauncher](../30-components/crownlauncher.md) | **Skeleton** | App launcher. Currently `cargo new` output. | `main` |
| [crownuikit](../30-components/crownuikit.md) | **Early** | Widget kit for [xilem](https://github.com/linebender/xilem) — sidebar, sliders, toggles, selects. Intended for the settings panel. | `main` |

## Platform

| Repo | Status | Purpose | Default branch |
|---|---|---|---|
| [crownos-config](../30-components/crownos-config.md) | **Stable** | Shared settings. RON files in `~/.config/crownos/`, live-reloaded with inotify. **This is how components coordinate.** | `main` |
| [crownos-iso](../30-components/crownos-iso.md) | **Skeleton** | archiso profile. Currently unmodified upstream Arch `releng`. | `main` |

## Ecosystem (phone bridge)

| Repo | Status | Purpose | Default branch |
|---|---|---|---|
| [crowncrate-linux](../30-components/crowncrate-linux.md) | **Broken** | Desktop side of the phone bridge. TCP + CBOR on port 5252. | `main` |
| [crowncrate-android](../30-components/crowncrate-android.md) | **Skeleton** | Android companion app. Currently an unmodified Android Studio template. | `main` |
| [crowncrate-chrome](../30-components/crowncrate-chrome.md) | **Empty** | Planned browser extension for OTP sync. No commits. | — |
| [lls-protocol](../30-components/lls-protocol.md) | **Broken** | Low-latency media streaming protocol (screen mirroring, second screen). RTP-shaped, UDP. | `main` |

## Project

| Repo | Status | Purpose | Default branch |
|---|---|---|---|
| [crownos-website](../30-components/crownos-website.md) | **Early** | Landing page. Next.js + Tailwind v4 + Biome, built with Bun. | `main` |
| [crownos-documentations](../30-components/crownos-documentations.md) | **Early** | This repository. | `main` |

---

## Dependency graph

Only three crates depend on other CrownOS crates. Everything else is
independent.

```
crownos-config ──(path)──────────────► crowndictator
               ──(git, patched to path)► crownpositor/config
                                            │
                                            └──(path)──► crownpositor/compositor

crownshell ──(git, unpinned)──► crownbar
           ──(git, unpinned)──► crowndock
           ──(path)───────────► crownotify
           ──(path)───────────► crowndictator

crownuikit       ── no CrownOS dependencies (xilem/winit only)
crowncrate-linux ── no CrownOS dependencies
lls-protocol     ── no CrownOS dependencies
crownlauncher    ── no dependencies at all
```

Two consequences you will hit immediately:

1. **`path = "../crownshell"` requires flat sibling checkouts.** See
   [Workspace setup](../10-getting-started/workspace-setup.md).
2. **`crownbar` and `crowndock` pin `crownshell` by git URL with no rev or tag.**
   Their lockfiles sit at `crownshell` 0.1.0 while `crownshell` is at 0.2.0. A
   `cargo update` will move them onto HEAD and may break them.

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
| Rust | 12 | Edition 2024, minimum Rust **1.85**. No `rust-toolchain.toml` anywhere — use a recent stable. |
| Kotlin | 1 | AGP 8.13.1, Kotlin 2.0.21, compileSdk 36, minSdk 29, JVM 11 |
| TypeScript | 1 | Bun, Next.js 16, React 19, Tailwind v4, Biome 2.2 |
| Shell / archiso | 1 | `archiso`, run as root on Arch |
| Markdown | 1 | None |
