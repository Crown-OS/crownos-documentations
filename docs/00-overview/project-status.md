# Project status

An honest account of what works today. Written so you can pick something to work
on without discovering the hard way that it is an empty skeleton.

Last verified on **2026-09-05**.

**How this was verified.** All eleven Rust crates were built with
`cargo check --all-targets` inside `nix develop`, using the toolchain the repos
pin — **rustc 1.88.0**, not the host's — and with the `[patch.crates-io]`
overlay in place. **All eleven passed**, and all eleven also pass
`cargo fmt --all --check`. Everything else on this page comes from reading the
source and its manifests. Where a claim is inferred rather than observed, it
says so.

> **Two different truths, and this page keeps them apart.**
>
> The build result above is a result about a **local working tree**: the repos
> as checked out here, with uncommitted fixes applied and an overlay above them.
> The **default branch** of each repository on GitHub is a different and much
> worse thing — most of the fixes described below have not been pushed. A
> visitor cloning today gets the broken state.
>
> Anything under [Landed locally, not yet pushed](#landed-locally-not-yet-pushed)
> is in the first category. Assume it is invisible to everyone but the
> maintainer until it is pushed.

---

## At a glance

| Component | Status | Builds? | Tests | Notes |
|---|---|---|---|---|
| crownos-config | **Stable** | Yes | 26 unit + 1 e2e | Best-documented crate in the org |
| crownpositor | **Early** | Yes | 183 unit | Largest codebase; ~17 open TODOs |
| crownshell | **Early** | Yes | 42 unit | The org's only published crate — 0.1.0 and 0.2.0. Its manifest says 0.3.0; that has not been published |
| crowndictator | **Early** | Yes | 11 unit | Heavy runtime prerequisites; needs OpenSSL headers |
| crownuikit | **Early** | Yes | None | Wired to nothing yet |
| crownotify | **Partial** | Yes | 7 integration | Only repo with real integration tests |
| crownbar | **Partial** | Yes | None | Reads no CrownOS config |
| crowndock | **Partial** | Yes | None | Cannot launch applications |
| crownos-website | **Early** | Yes | None | Copy is placeholder |
| crownos-setup | **Stable** | n/a | container-verified | Bootstrap for 4 distro families + Nix |
| crownos-iso | **Skeleton** | n/a | n/a | Unmodified upstream archiso |
| crownlauncher | **Skeleton** | Yes | None | `cargo new` output; `version = "0.0.0"`, unpublished |
| crowncrate-android | **Skeleton** | Yes | 2 template stubs | Android Studio template |
| crowncrate-linux | **Skeleton** | Local only | None | Compiles here, but does nothing; `version = "0.0.0"`, unpublished |
| lls-protocol | **Skeleton** | Local only | None | Compiles here; six source files are empty |
| crowncrate-chrome | **Empty** | n/a | n/a | Zero commits, zero objects; not in the org listing |

**All eleven Rust crates compile** — in a local tree, under the pinned 1.88.0
toolchain, with the `[patch.crates-io]` overlay — and all eleven are
rustfmt-clean. "Local only" in the Builds column means the fix that makes a
crate compile has not been pushed.

Two caveats that apply to every "Yes" in that column:

- **Nothing builds from a plain clone.** `crownbar`, `crowndock`, `crownotify`,
  `crowndictator` and `crownpositor` declare `crownshell = "0.3"` and
  `crownos-config = "0.2"`. Neither version exists on crates.io. Without the
  overlay these five fail at `cargo metadata`, before compilation starts. See
  [Workspace setup](../10-getting-started/workspace-setup.md#the-overlay-mandatory).
- **Every committed `Cargo.lock` in a dependent repo is contaminated.** They were
  generated inside the patched tree: the `crownshell` and `crownos-config`
  entries have no `source =` line, and the files carry `[[patch.unused]]`
  stanzas. `cargo build --locked` from a fresh clone fails even with the overlay.
  Those locks need regenerating outside the overlay before anything is published.

"Builds" is not "works" either: several of these are still skeletons, and the
**Status** column is the one to read.

---

## Landed locally, not yet pushed

Everything in this section is fixed **in a working tree on the maintainer's
machine and nowhere else**. On the default branch of each repository the
original defect is still there, and still what a contributor cloning today will
hit. Pushing these is the single highest-value thing anyone could do.

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

What is still true: this crate is **346 lines and does essentially nothing**.
`src/lib.rs` is empty, `src/ui/mod.rs` is empty, and `src/predule.rs` is not
reachable from `main.rs`. Its manifest carries `version = "0.0.0"` as a
placeholder, but it is **not published** — the name is not held on crates.io by
anything.

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

1. **`crownos-config` had no `startup` field.** It does now. `state/actions.rs:171` reads
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
closure. `crownotify` now compiles against `crownshell` at its current 0.3.0
manifest version — supplied by the overlay, not by crates.io, since 0.3.0 has
never been published — as do `crownbar` and `crowndock`. The eight commits of
`crownshell` drift turned out to be additive, so neither needed a source change.

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

### CI: written, and not running

**No repository in the organization has ever run CI.** The reusable workflows
were written, and the per-repo callers exist, but they live only as untracked
files in local checkouts — and the repository they call into,
`Crown-OS/.github`, **does not exist on GitHub**. Fetching it returns 404.

So there are no badges to read, no green checks on any pull request, and no
gate on anything. The table below is the intended design, not a description of
what happens today.

| Repos | Checks (planned) |
|---|---|
| The 11 Rust repos | `cargo fmt --check` · build · clippy · `cargo test` |
| crownos-website | `bun install` · `biome check` · `next build` |
| crowncrate-android | `assembleDebug` · unit tests · APK artifact |
| crownos-documentations | relative-link check · status-marker consistency |
| crownos-iso | shellcheck |

Three things to know about the design:

- **rustfmt blocks, clippy does not.** Roughly 15,000 lines have never been
  linted, so `-D warnings` would make every repo red for reasons unrelated to
  the change under review. Clippy is set to run and report to the job summary.
  Flipping it to blocking is a one-line change in `rust.yml`.
- **The Rust jobs will need the overlay too.** A CI runner cloning a single
  repository hits the same `crownshell = "0.3"` resolver failure a contributor
  does. `crownbar` and `crowndock` now pass `siblings: crownshell` to the
  reusable workflow so it checks out the sibling and writes the patch; every
  other dependent repo needs the same before its job can pass.
- **The formatting backlog is gone.** All eleven crates pass
  `cargo fmt --all --check` on rustfmt 1.88, so a blocking fmt job would be
  green from its first run. `crowncrate-linux`'s legacy `rustfmt.toml` — a
  defaults dump that declared edition 2015 — was deleted; no repo carries one
  now.

CD is intended to be minimal: pushing a `v*` tag to `crownbar`, `crowndock`,
`crownotify`, `crowndictator` or `crownpositor` would build a release binary and
attach a tarball to a draft GitHub Release, and `publish.yml` would push the
crate to crates.io. None of that has ever run. The organization has exactly one
git tag — `crownshell v0.2.0` — and `crownshell` 0.1.0 and 0.2.0 were both
published by hand. Nothing goes to the AUR or ships as an ISO.

Still absent: the **`Crown-OS/.github` repository itself**, **CODEOWNERS**,
**dependabot**, **branch protection**, and installed issue/PR templates — those
are staged in [`templates/.github/`](../../templates/.github) but not yet copied
into the repos.

### Toolchain pin (resolved)

Every Rust repo now ships a `rust-toolchain.toml` pinning **1.88.0**, and every
crate declares `rust-version = "1.88"`.

1.88 rather than 1.85 is the correction that mattered: edition 2024 needs only
1.85, and 1.85 is what `crownshell` published in its 0.2.0 metadata — but
`vello 0.9` and `xilem 0.4` both declare `rust-version = "1.88"`, and `wgpu 29`
and `zbus 5.16` declare 1.87. The dependency graph sets the floor, not the
edition. The declared MSRV was wrong by three minor versions.

### Licensing

**Still unresolved on GitHub.** Exactly **three of the sixteen repositories**
carry a LICENSE file on their default branch:

| Repo | LICENSE | Copyright |
|---|---|---|
| `crownshell` | MIT | `marvelxcodes` |
| `crownos-documentations` | MIT | `Crown-OS` |
| `crownOs-setup` | MIT | `The CrownOS Authors` |

The other thirteen have none, which legally makes them all-rights-reserved
despite being presented as open source. MIT files for them, and the
`license = "MIT"` line in each manifest, exist as local uncommitted changes —
see [Landed locally, not yet pushed](#landed-locally-not-yet-pushed). crates.io
will not accept a crate without that field, so this blocks every publish.

**The copyright line is unsettled and needs a decision.** The new files say
`The CrownOS Authors` rather than extending one individual's copyright claim
across the whole organization, but that disagrees with `crownshell`'s existing
file. Three maintainers are listed in CONTRIBUTING. Settle it before publishing
widely — it is baked into every released crate permanently.

`crownos-iso` is a verbatim copy of Arch Linux's archiso `releng` profile, whose
scripts carry `SPDX-License-Identifier: GPL-3.0-or-later`. A GPL-3.0-or-later
LICENSE has been added there locally, which is the right answer, but
redistributing that profile under an MIT umbrella anywhere else is still a
problem.

### Versioning

**`crownshell` is the only CrownOS crate that has ever been published** — 0.1.0
on 2026-07-15 and 0.2.0 on 2026-08-04, with docs.rs builds — and it carries the
organization's only git tag, `v0.2.0`.

Nothing else is on crates.io: not `crownos-config`, `crownuikit`, `crownbar`,
`crowndock`, `crownotify`, `crowndictator`, `crownpositor`,
`crownpositor-config`, `crownlauncher`, `crowncrate-linux`, `lls-client` or
`lls-server`. Where this documentation used to say a crate was "published as a
0.0.0 placeholder to hold the name", that was wrong — the names are not held.

`crownshell`'s manifest now says 0.3.0 and its dependents ask for `"0.3"`, but
0.3.0 has not been published either. That gap is what makes the
`[patch.crates-io]` overlay mandatory. See
[Releasing](../40-contributing/releasing.md) for the intended order and the
state of each crate.

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

- Every dependent repo's committed `Cargo.lock` records `crownshell` and
  `crownos-config` with **no `source =` line**, and carries `[[patch.unused]]`
  stanzas. They were generated inside the local patched tree. `cargo build
  --locked` from a fresh clone fails on all of them.
- `crowndictator` pulls `crownos-config` with default features, which includes
  `xilem` — a headless daemon dragging in a whole GUI toolkit. It probably wants
  `default-features = false`.
- `crownshell` declares `bluer`, `battery` and `tracing` and uses none of them in
  `src/` or `examples/`. `bluer` alone pulls in a large D-Bus tree.
- Version drift across siblings: `tiny-skia` 0.11 vs 0.12, `dirs` 5 vs 6,
  `calloop` 0.13 vs 0.14.

### Five separate spring implementations

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

1. **Push what is already fixed.** Everything under
   [Landed locally, not yet pushed](#landed-locally-not-yet-pushed) is finished
   work that no contributor can see. This is worth more than the rest of the
   list combined.
2. **Publish `crownos-config` 0.2.0 and `crownshell` 0.3.0**, then regenerate
   every dependent lockfile outside the overlay. That is what would make a plain
   clone build, and it is what retires the overlay.
3. **Create `Crown-OS/.github`** so CI runs at all.
4. Implement `ext-background-effect-v1` in `crownpositor` so blur works.
5. Make `crowndock` launch applications.
6. Add LICENSE files to the 13 repos that lack one on their default branch.
7. Make `crownbar` read `appearance.ron` instead of hardcoding its height.
8. Give `crownos-iso` actual CrownOS branding and packages.
