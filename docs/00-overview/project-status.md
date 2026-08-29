# Project status

An honest account of what works today. Written so you can pick something to work
on without discovering the hard way that it is an empty skeleton.

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
| crownshell | **Early** | Yes | 42 unit | Published on crates.io at 0.3.0 |
| crowndictator | **Early** | Yes | 11 unit | Heavy runtime prerequisites |
| crownuikit | **Early** | Yes | None | Wired to nothing yet |
| crownotify | **Partial** | Yes | 7 integration | Only repo with real integration tests |
| crownbar | **Partial** | Yes | None | Reads no CrownOS config |
| crowndock | **Partial** | Yes | None | Cannot launch applications |
| crownos-website | **Early** | Yes | None | Copy is placeholder |
| crownos-setup | **Stable** | n/a | container-verified | Bootstrap for 4 distro families + Nix |
| crownos-iso | **Skeleton** | n/a | n/a | Unmodified upstream archiso |
| crownlauncher | **Skeleton** | Yes | None | `cargo new` output; 0.0.0 placeholder |
| crowncrate-android | **Skeleton** | Yes | 2 template stubs | Android Studio template |
| crowncrate-linux | **Skeleton** | Yes | None | Compiles, but does nothing; 0.0.0 placeholder |
| lls-protocol | **Skeleton** | Yes | None | Compiles; six source files are empty |
| crowncrate-chrome | **Empty** | n/a | n/a | No commits |

**Every Rust crate in the organization now compiles.** Four did not as of August
2026 — `crownotify`, `crowncrate-linux`, `lls-protocol` and `crownpositor` (which
could not even resolve its dependencies). Each is covered below. "Builds" is not
"works": several of these are still skeletons, and the **Status** column is the
one to read.

---

## Previously broken, in detail

These are the concrete blockers. Each is a good first issue.

### lls-protocol: compiles now, still nearly empty

It used to fail before compilation started: the root `Cargo.toml` declared both a
`[workspace]` and a `[package]` with no `src/` at the root, so `cargo metadata`
died with `no targets specified in the manifest`. Behind that, `client/` and
`server/` referenced a `[workspace.dependencies]` table that had never been
written.

The root is now a virtual workspace with a real `[workspace.dependencies]`, and
the members are renamed `lls-client` and `lls-server` (`client` and `server` are
both taken on crates.io).

What remains:

- `server/src/protocol.rs` called `UdpSocket::try_from(device.ip_address)`, which
  did not compile — `TryFrom` is implemented for `std::net::UdpSocket`, not for
  an `IpAddr`. `Connection::connect` now binds a socket and connects it, and is
  async because `tokio`'s is. **The crate builds.**
- Five module files (`config.rs`, `signaling.rs`, `streaming.rs`, `nvenc/mod.rs`,
  `platform/mod.rs`) and `client/src/lib.rs` are zero bytes.
- `serde` is declared without the `derive` feature. Nothing derives yet, so it is
  not an error — it will be as soon as wire types land.
- The repository name does not correspond to a crate. Extracting the wire types
  from `server/` into a shared `protocol/` member is what would claim
  `lls-protocol` on crates.io.

### crowncrate-linux: compiles now, still a skeleton

It used to fail with two errors and carry two more defects behind them. All four
are fixed:

1. **`Box<dyn Action>` could not cross a thread boundary.** `server.rs::listen`
   handed the `ActionManager` to a `thread::spawn` closure, but the trait object
   had no `Send` bound. `Action` is now `: Send + Sync`.
2. **`&mut self` escaped into a `'static` closure.** `ActionManager` is now
   `Clone` — its `actions` field was already `Arc<Mutex<..>>`, so the clone shares
   one table — and `listen` moves a handle in rather than borrowing `self`.
3. **`notify` iterated an `Arc<Mutex<HashMap<..>>>` directly** and had an empty
   loop body. It now locks, iterates `.values()`, and actually calls
   `handle_message`.
4. **`unsubscribe` was inverted.** `retain(|&i, _| i == action)` kept only the
   entry it was asked to remove. Now `i != action`.

The **glib conflict is gone too.** `Cargo.toml` declared `glib = "0.17"` while
`gtk4 = "0.7"` requires 0.18, and the lockfile carried both. Nothing in `src/`
referenced `glib` at all, so the direct dependency was simply removed; the
lockfile now has one `glib`.

What is still true: this crate is **333 lines and does essentially nothing**.
`src/lib.rs` is empty, `src/ui/mod.rs` is empty, and `src/predule.rs` is not
reachable from `main.rs`. It is published as a **0.0.0 placeholder** to hold the
name, not as a usable crate.

Two things deliberately left alone:

- **`serde_cbor 0.11.2` is unmaintained** (RUSTSEC-2021-0127). Swapping to
  `ciborium` changes the streaming-deserializer API that `server.rs` relies on,
  and the wire format is shared with the Android client, so it needs testing
  against both ends rather than a blind substitution.
- **The unauthenticated TCP listener on :5253/:5252** — see
  [SECURITY.md](../../SECURITY.md).

### crownpositor: fixed, and it revealed a missing config field

`crownpositor` had never built in a normal checkout. Its committed `[patch]`
pointed at `../crownos-config`, a path that usually does not exist, so
`cargo metadata` failed before compilation and hid everything behind it.

With the patch gone and `crownos-config = "0.2"` resolving properly, two real
problems surfaced:

1. **`crownos-config` had no `startup` field.** `state/actions.rs:171` reads
   `self.config.current.compositor.startup`, and `config/src/startup.rs` exists
   purely to parse those command lines — but the field was never added to the
   `Compositor` section. `crownpositor`'s own HEAD commit is *"…and added startup
   apps"*; the `crownos-config` half was never landed. Added now as
   `startup: Vec<String>`, which is what `startup::commands(&[String])` expects.
2. **`main.rs` called `compositor::run()`**, and the library is now named
   `crownpositor` after the crates.io rename. One line.

Both fixed; `cargo check --all-targets` is clean.

### crownotify: fixed

`crownotify/src/main.rs` called `window.request_frame(compositor_state, qh)` with
two arguments while `crownshell` had taken three since **0.2.0** —
`request_frame(&mut self, compositor_state, qh, text_cx)`. It was stale against
the published release, not merely against HEAD.

Fixed by binding `text_cx` out of the destructured `App` in the ping-source
closure. `crownotify` now compiles against `crownshell` 0.3.0, as do `crownbar`
and `crowndock` — the eight commits of `crownshell` drift turned out to be
additive, so neither needed a source change.

The notification centre is still a no-op and `notifications.ron` is still
ignored, so the component stays **Partial**.

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
attaches a tarball to a draft GitHub Release. A `v*` tag also publishes the
crate to crates.io via `publish.yml`. Nothing is published to the AUR or as an
ISO.

Still absent: **CODEOWNERS**, **dependabot**, **branch protection**, and
installed issue/PR templates — those are staged in
[`templates/.github/`](../../templates/.github) but not yet copied into the
repos.

### Toolchain pin (resolved)

Every Rust repo now ships a `rust-toolchain.toml` pinning **1.88.0**, and every
crate declares `rust-version = "1.88"`.

1.88 rather than 1.85 is the correction that mattered: edition 2024 needs only
1.85, and 1.85 is what `crownshell` published in its 0.2.0 metadata — but
`vello 0.9` and `xilem 0.4` both declare `rust-version = "1.88"`, and `wgpu 29`
and `zbus 5.16` declare 1.87. The dependency graph sets the floor, not the
edition. The declared MSRV was wrong by three minor versions.

### Licensing

**Mostly resolved.** 13 of 15 non-empty repositories had no LICENSE file, which
legally made them all-rights-reserved despite being presented as open source. All
now carry MIT, and every crate declares `license = "MIT"` in its manifest —
crates.io will not accept a crate without it.

| Repo | LICENSE | Copyright |
|---|---|---|
| `crownshell` | MIT | `marvelxcodes` |
| `crownos-documentations` | MIT | `Crown-OS` |
| the 12 added now | MIT | `The CrownOS Authors` |
| `crownos-iso` | **none** — see below | — |

**The copyright line is unsettled and needs a decision.** The new files say
`The CrownOS Authors` rather than extending one individual's copyright claim
across the whole organization, but that disagrees with `crownshell`'s existing
file. Three maintainers are listed in CONTRIBUTING. Settle it before publishing
widely — it is baked into every released crate permanently.

`crownos-iso` is a verbatim copy of Arch Linux's archiso `releng` profile, whose
scripts carry `SPDX-License-Identifier: GPL-3.0-or-later`. Redistributing it
under an MIT umbrella is a licensing problem that needs resolving.

### Versioning

`crownshell` is published on crates.io — 0.1.0 on 2026-07-15 and 0.2.0 on
2026-08-04, with docs.rs builds — and carries the organization's only git tag,
`v0.2.0`. Everything else is being published now; see
[Releasing](../40-contributing/releasing.md) for the order and the state of each
crate.

There is still no `CHANGELOG.md` anywhere, and nothing is published to the AUR
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
