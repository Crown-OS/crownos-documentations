# Project status

An honest account of what works today. Written so you can pick something to work
on without discovering the hard way that it does not compile.

Last verified against the default branch of each repository on **2026-08-21**.

**How this was verified.** `crownos-config`, `crownshell`, `crowncrate-linux` and
`lls-protocol` were built and their tests run on rustc 1.95.0 — the results below
are what those commands actually printed. Every other row comes from reading the
source and its manifests, not from a build. Where a claim is inferred rather than
observed, it says so.

---

## At a glance

| Component | Status | Builds? | Tests | Notes |
|---|---|---|---|---|
| crownos-config | **Stable** | Yes | 26 unit + 1 e2e | Best-documented crate in the org |
| crownpositor | **Early** | Yes | 183 unit | Largest codebase; ~17 open TODOs |
| crownshell | **Early** | Yes | 42 unit | Self-described "API will move" |
| crowndictator | **Early** | Yes | 11 unit | Heavy runtime prerequisites |
| crownuikit | **Early** | Yes | None | Wired to nothing yet |
| crownotify | **Partial** | **No** ¹ | 7 integration | Only repo with real integration tests |
| crownbar | **Partial** | Yes | None | Reads no CrownOS config |
| crowndock | **Partial** | Yes | None | Cannot launch applications |
| crownos-website | **Early** | Yes | None | Copy is placeholder |
| crownos-iso | **Skeleton** | n/a | n/a | Unmodified upstream archiso |
| crownlauncher | **Skeleton** | Yes | None | `cargo new` output |
| crowncrate-android | **Skeleton** | Yes | 2 template stubs | Android Studio template |
| crowncrate-linux | **Broken** | **No** | None | Compile errors |
| lls-protocol | **Broken** | **No** | None | Compile errors |
| crowncrate-chrome | **Empty** | n/a | n/a | No commits |

¹ `crownotify` compiles against `crownshell` 0.1.0 but not against current
`crownshell` HEAD — see below.

---

## Known-broken, in detail

These are the concrete blockers. Each is a good first issue.

### lls-protocol does not build

It fails before compilation even starts. The root `Cargo.toml` declares both a
`[workspace]` and a `[package]`, but there is no `src/` at the root:

```
$ cargo metadata --no-deps
error: failed to parse manifest at `.../lls-protocol/Cargo.toml`
Caused by:
  no targets specified in the manifest
  either src/lib.rs, src/main.rs, a [lib] section, or [[bin]] section
  must be present
```

Fixing that exposes a second problem: `client/Cargo.toml` and
`server/Cargo.toml` both use `serde = { workspace = true }`,
`tokio = { workspace = true }` and `tokio-stream = { workspace = true }`, but the
root has **no `[workspace.dependencies]` table** — those dependencies are
declared under `[dependencies]` for the root package instead.

Then `server/src/protocol.rs` calls `UdpSocket::try_from(device.ip_address)`,
which does not exist.

Five module files (`config.rs`, `signaling.rs`, `streaming.rs`, `nvenc/mod.rs`,
`platform/mod.rs`) and `client/src/lib.rs` are zero bytes.

### crowncrate-linux does not build

`cargo check` reports two errors:

```
error[E0277]: `(dyn Action + 'static)` cannot be sent between threads safely
  --> src/communication/server.rs:48:27
error[E0277]: `Arc<Mutex<HashMap<Actions, Box<(dyn Action + 'static)>>>>`
              is not an iterator
  --> src/actions/action_manager.rs:33:23
```

1. **`server.rs::listen`** hands the `ActionManager` to a `thread::spawn`
   closure, but `Box<dyn Action>` carries no `Send` bound, so it cannot cross a
   thread boundary. The `Action` trait needs `: Send + Sync`.
2. **`action_manager.rs::notify`** writes `for action in self.actions`, iterating
   an `Arc<Mutex<HashMap<..>>>` directly. It needs a `.lock()` first — and the
   loop body is empty regardless.

A third defect is a logic bug rather than a compile error:
**`action_manager.rs::unsubscribe`** uses `actions.retain(|&i, _| i == action)`,
which keeps only the entry it was asked to remove.

There is also a latent dependency conflict: `Cargo.toml` declares
`glib = "0.17"` while `gtk4 = "0.7"` requires glib 0.18, and the lockfile
contains both. It does not currently surface as an error because no code uses
gtk4 — `src/ui/mod.rs` is empty — but it will as soon as a UI is written.

### crownotify does not build against current crownshell

`crownotify/src/main.rs` calls `window.request_frame(compositor_state, qh)` with
two arguments. Current `crownshell` HEAD takes three —
`request_frame(&mut self, compositor_state, qh, text_cx)`. `crownotify` was last
touched before `crownshell`'s text work landed.

`crowndictator` uses the three-argument form, so **`crownotify` and
`crowndictator` cannot both build against the same `crownshell` revision** today.

---

## Features specified but not implemented

### Blur is requested everywhere and implemented nowhere

`crownbar`, `crowndock` and `crownotify`'s popup path all set `blur: true`.
`crownshell` correctly binds `ext-background-effect-v1` and degrades silently
when the compositor does not advertise it.

`crownpositor`'s handler — `compositor/src/handlers/background_effect.rs` — is
**entirely commented out**, all 18 lines, and the global is never created. Under
CrownOS the frosted-glass look silently does not happen. Under Hyprland or KWin
it depends on their support for the staging protocol.

### crowndock cannot launch applications

There is no `Exec=` parsing and no `std::process::Command` anywhere in the crate.
`on_pointer_press` and `on_pointer_release` only drive drag state. The dock is
currently a pin manager with an animation.

### The workspace overview does not exist

`shell/windows_view/mod.rs` and `shell/workspaces_view/mod.rs` are zero-byte
files. `MoveWorkspaceToOutput`, `OpenWorkspaceView` and `CloseWorkspaceView` log
`"action is not implemented yet"`. The four-finger trackpad gestures are bound to
those dead actions.

### Config sections with no reader

Five sections exist in `crownos-config` and nothing consumes them:

`sound.ron` · `wifi.ron` · `bluetooth.ron` · `power.ron` · `keybinds.ron`

`keybinds.ron` in particular defines `launcher: Super+Ctrl` for a launcher that
has not been written, and `crownpositor` does not read that section
(`Config::sections()` returns only `compositor`, `appearance` and `display`).

### Components that ignore the config convention

| Component | Behaviour |
|---|---|
| `crownbar` | Reads no CrownOS config. Hardcodes bar height 40 while `appearance.bar_height` defaults to 32. |
| `crownotify` | Ignores `notifications.ron` entirely, including Do-Not-Disturb. `toggle_dnd()` exists and nothing calls it. |
| `crowndock` | Uses its own `~/.config/crowndock/items.toml` — TOML, own directory. |
| `crownuikit` | Reads nothing. |

Follows the convention correctly: `crownpositor`, `crowndictator`.

### crownotify's declared-but-absent capabilities

`GetCapabilities` advertises `action-icons`, `actions`, `body-images`, `sound`
and `persistence`. None are implemented. `CloseNotification` returns `Ok(())`
with no effect, the three declared signals are never emitted, and
`OpenNotificationCenter` only logs — there is no notification centre.

`models/audio.rs` and `models/display.rs` are empty structs, though the
`Notification` enum has variants for them.

---

## Infrastructure gaps

### CI

Every repository runs CI on push and pull request, via reusable workflows in
[`Crown-OS/.github`](https://github.com/Crown-OS/.github).

| Repos | Checks |
|---|---|
| The 11 Rust repos | `cargo fmt --check` · build · clippy · `cargo test` |
| crownos-website | `bun install` · `biome check` · `next build` |
| crowncrate-android | `assembleDebug` · unit tests · APK artifact |
| crownos-documentations | relative-link check · status-marker consistency |
| crownos-iso | shellcheck |

Two things to know:

- **rustfmt blocks, clippy does not.** Roughly 15,000 lines have never been
  linted, so `-D warnings` would make every repo red for reasons unrelated to
  the change under review. Clippy runs and reports to the job summary. Flipping
  it to blocking is a one-line change in `rust.yml`.
- **`crowncrate-linux` and `lls-protocol` are red on purpose.** They do not
  compile; the badge reflects that rather than hiding it.

CD is deliberately minimal: pushing a `v*` tag to `crownbar`, `crowndock`,
`crownotify`, `crowndictator` or `crownpositor` builds a release binary and
attaches a tarball to a draft GitHub Release. Nothing is published to crates.io,
the AUR, or as an ISO.

Still absent: **CODEOWNERS**, **dependabot**, **branch protection**, and
installed issue/PR templates — those are staged in
[`templates/.github/`](../../templates/.github) but not yet copied into the
repos.

### No toolchain pin

There is no `rust-toolchain.toml` in any repo. Only `crownshell` declares an MSRV
(`rust-version = "1.85"`). Every crate is edition 2024, which requires 1.85
regardless.

### Licensing

**13 of 15 non-empty repositories have no LICENSE file.** Legally that makes them
all-rights-reserved despite being presented as open source.

| Repo | LICENSE | Copyright |
|---|---|---|
| `crownshell` | MIT | `marvelxcodes` |
| `crownos-documentations` | MIT | `Crown-OS` |
| everything else | **none** | — |

Two further inconsistencies: `crowndictator` declares `license = "MIT"` in its
`Cargo.toml` but ships no LICENSE file, and the two existing files disagree on
whether the copyright holder is an individual or the organization.

`crownos-iso` is a verbatim copy of Arch Linux's archiso `releng` profile, whose
scripts carry `SPDX-License-Identifier: GPL-3.0-or-later`. Redistributing it
under an MIT umbrella is a licensing problem that needs resolving.

### Versioning

One tag exists in the entire organization: `crownshell v0.2.0`. No `CHANGELOG.md`
anywhere. No release automation. Nothing published to crates.io, docs.rs, the AUR,
or as an ISO artifact.

### Branch names (resolved)

Every repository now defaults to `main`. The nine that used `master` —
`crownpositor`, `crownshell`, `crownbar`, `crowndock`, `crownlauncher`,
`crownotify`, `crowndictator`, `crownuikit`, `crowncrate-linux` — were renamed in
August 2026. No history was rewritten.

If your clone predates that, see
[Workspace setup](../10-getting-started/workspace-setup.md#if-you-cloned-before-the-rename).

### Dependency hygiene

- `crownbar` and `crowndock` declare `crownshell` by git URL with **no `rev`,
  `tag` or `branch`**. Their locks pin `de4ab90` at version 0.1.0 — three commits
  behind HEAD.
- `crowndock`'s lockfile entry for `crownshell` has no `source` line, meaning it
  was generated against a local path checkout rather than the git URL.
- `crowndictator` pulls `crownos-config` with default features, which includes
  `xilem` — a headless daemon dragging in a whole GUI toolkit. It probably wants
  `default-features = false`.
- `crownshell` declares `bluer`, `battery` and `tracing` and uses none of them in
  `src/` or `examples/`. `bluer` alone pulls in a large D-Bus tree.
- Version drift across siblings: `tiny-skia` 0.11 vs 0.12, `dirs` 5 vs 6,
  `calloop` 0.13 vs 0.14.

### Four separate spring implementations

`crownpositor/compositor/src/animations/spring.rs`,
`crownshell/src/animations.rs`, `crownbar/src/animation.rs`,
`crowndock/src/dock_handler.rs` and `crownuikit/src/animation.rs` each contain
their own. `crownbar` and `crowndock` predate `crownshell`'s and never migrated.

---

## Documentation drift

The website and the code disagree in several places. The code is authoritative.

| Website claims | Reality |
|---|---|
| "TOML profiles" | Config is **RON** |
| "Hyprcrown" compositor | The compositor is `crownpositor` |
| A `crownos` CLI (`sync`, `snapshot`, `rollback`, `pair`, `ask`) | No such binary exists in any repo |
| btrfs/snapper atomic rollback, ggml/ONNX AI runtime, BORE-EEVDF scheduler | None ship in the ISO profile |
| Three ISO editions (Desktop 2.4 GB, Minimal 780 MB, ARM 2.1 GB) and five mirrors | The profile builds one unbranded x86_64 Arch rescue image; no mirrors exist |
| "12.4k GitHub stars", "320+ contributors", "8.2k Discord members" | Placeholder values |

The site's `/docs` route is a shell of 32 cards where **every link points back at
`/docs`**. This repository is what those cards should point to.

---

## Where help is most useful

Roughly in order of impact:

1. Fix the `lls-protocol` workspace dependencies so it compiles.
2. Fix `crowncrate-linux`'s three compile errors and the glib version conflict.
3. Update `crownotify` to current `crownshell`, then pin `crownbar` and
   `crowndock` to a `crownshell` rev.
4. Implement `ext-background-effect-v1` in `crownpositor` so blur works.
5. Make `crowndock` launch applications.
6. Add LICENSE files to the 13 repos that lack one.
7. Make `crownbar` read `appearance.ron` instead of hardcoding its height.
8. Give `crownos-iso` actual CrownOS branding and packages.
