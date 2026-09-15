# Dependency graph

Build-time relationships between CrownOS crates, and the version skew you need to
know about.

Read from the manifests as they stand today. Where a manifest disagrees with
what crates.io actually contains, that is called out — it is the central fact
about this graph.

---

## The graph

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

crownuikit        ── no CrownOS dependencies (xilem, winit, blinc_icons only)
crowncrate-linux  ── no CrownOS dependencies
lls-client        ── no CrownOS dependencies
lls-server        ── no CrownOS dependencies
crownlauncher     ── no dependencies at all; [dependencies] is empty
```

Only two crates are depended upon: `crownshell` and `crownos-config`. Five crates
depend on one or both of them — `crownbar`, `crowndock`, `crownotify`,
`crowndictator` and `crownpositor` (through its own `crownpositor-config`
member). Everything else is a leaf.

`crownuikit` is an **orphan**: nothing in the organization depends on it. It is
meant to become the settings panel, which is why `crownos-config` ships a
`xilem` feature, but nothing connects the two.

## Exact declarations

| Consumer | Declaration |
|---|---|
| `crownbar` | `crownshell = "0.3"` |
| `crowndock` | `crownshell = "0.3"` |
| `crownotify` | `crownshell = "0.3"` |
| `crowndictator` | `crownshell = "0.3"` <br> `crownos-config = "0.2"` |
| `crownpositor` (workspace) | `crownos-config = { version = "0.2", default-features = false }` |
| `crownpositor` (compositor) | `config = { package = "crownpositor-config", path = "../config", version = "0.1.0" }` <br> `crownos-config = { workspace = true }` |
| `crownpositor-config` | `crownos-config = { workspace = true }` |

`crownpositor` disables default features on `crownos-config` to drop the `xilem`
feature — the compositor has no business pulling in a GUI toolkit.

The one remaining path dependency in the organization is `crownpositor`'s
`../config`, and that is within a single repository's own workspace, which is
fine. Every cross-repository edge is now a version requirement.

---

## The versions do not exist

| Declared | On crates.io |
|---|---|
| `crownshell = "0.3"` | 0.1.0 and 0.2.0 only |
| `crownos-config = "0.2"` | never published |

Those are the only two cross-repo requirements in the graph, and neither
resolves. `crownbar`, `crowndock`, `crownotify`, `crowndictator` and
`crownpositor` therefore **fail at `cargo metadata`** from a clean clone:

```
error: failed to select a version for the requirement `crownshell = "^0.3"`
```

What makes them build is a `[patch.crates-io]` overlay in a `.cargo/config.toml`
placed above the checkouts, written by `crownos-setup`'s `./bootstrap.sh --dev`.
That file is mandatory, and it is in no repository's history — see
[Workspace setup](../10-getting-started/workspace-setup.md#the-overlay-mandatory).

Publishing `crownos-config` 0.2.0 and `crownshell` 0.3.0 is what would retire
the overlay.

### The lockfiles are contaminated

Because every dependent lockfile was generated *inside* the patched tree, the
`crownshell` and `crownos-config` entries carry **no `source =` line**, and the
files contain `[[patch.unused]]` stanzas. `cargo build --locked` from a fresh
clone fails on all of them even with the overlay present. They must be
regenerated outside the overlay before anything is published.

### Why not a committed `[patch]`?

`crownpositor` used to commit one, keyed on `github.com/crown-os/crownos-config`
while `crownshell`'s own metadata says `github.com/Crown-OS/…`. Cargo treats
those as different sources, so the patch silently did nothing — and the
`../crownos-config` path it named made a sibling checkout mandatory for every
clone. It is gone. Keep it gone.

---

## Workspaces

Two repos are Cargo workspaces. There is **no org-wide workspace**.

| Repo | Members | Package names | Notes |
|---|---|---|---|
| `crownpositor` | `compositor/`, `config/` | `crownpositor`, `crownpositor-config` | `resolver = "3"`, has `[workspace.dependencies]` |
| `lls-protocol` | `client/`, `server/` | `lls-client`, `lls-server` | Virtual workspace, `resolver = "3"`, `[workspace.package]` + `[workspace.dependencies]` |

`lls-protocol`'s root manifest used to be broken twice over — a `[package]`
declared alongside the `[workspace]` with no `src/` at the root, so cargo failed
at manifest parse with `no targets specified in the manifest`; and members
referencing `serde`, `tokio` and `tokio-stream` as `{ workspace = true }` against
a `[workspace.dependencies]` table that had never been written. Both are fixed:
the root is a virtual workspace and the table exists. **That fix has not been
pushed** — the default branch on GitHub still carries the broken root.

Everything else is a standalone single-crate repository.

---

## Version skew (resolved)

Three problems used to live here. All three are fixed in the manifests; the
history is kept because it explains why the current rules exist.

### 1. crownbar and crowndock were pinned to a stale crownshell — fixed

Their manifests declared `crownshell` by git URL with **no `rev`, `tag` or
`branch`**. What actually pinned them was the committed lockfile:

| Crate | Lock pinned | crownshell HEAD |
|---|---|---|
| `crownbar` | `de4ab90…` @ 0.1.0 | `6f3189a` @ 0.3.0 |
| `crowndock` | 0.1.0, **no `source` line** | `6f3189a` @ 0.3.0 |

`crownbar` was eight commits behind. `crowndock`'s lock entry had no `source` at
all, meaning it had been generated against a path checkout while its manifest
said git — the two files disagreed, and `cargo build --locked` would have failed.

Both now declare `crownshell = "0.3"`. Worth recording: when they were finally
built against 0.3.0 through the overlay, **both compiled unchanged**. The eight
commits of drift were additive.

### 2. crownotify could not share a crownshell revision — fixed

`crownshell`'s `request_frame` gained a parameter when text rendering landed:

```rust
pub fn request_frame(&mut self, compositor_state: &CompositorState,
                     qh: &QueueHandle<App>, text_cx: &mut TextContext)
```

`crownotify` called it with two arguments and `crowndictator` with three. Because
both used `path = "../crownshell"`, they resolved to the *same* checkout, so one
of them always failed. `crownotify` was two arguments behind the **published**
0.2.0, not just behind HEAD.

Fixed by binding `text_cx` out of the destructured `App` in the ping-source
closure and passing it through.

### 3. Dependency drift between siblings — partly fixed

`anyhow` was declared four different ways (`"1"`, `"1.0.100"`, `"1.0.102"`,
`"1.0.104"`), `serde` three (`"1"`, `"1.0"`, `"1.0.228"`), and `crowndictator`
carried a hand-written comment saying its Wayland pins were "versions matched to
crownshell" — alignment nothing enforced.

The manifests have since been aligned by hand: `anyhow = "1.0.104"` and
`serde = "1.0.228"` everywhere. What was *supposed* to keep them that way is a
shared `crown-versions.toml` in `Crown-OS/.github` checked by a `check-versions.py`
CI step. **That repository does not exist on GitHub**, so nothing enforces the
alignment and it will drift again. Some skew survives regardless: `tiny-skia`
0.11 in `crowndock` against 0.12 in `crownpositor`, `dirs` 5 against 6, `calloop`
0.13 against 0.14.

---

## The two graphics lanes (open)

The one piece of skew that is **not** resolved, because the decision is deferred.

| Lane | Repos | Stack |
|---|---|---|
| A | `crownshell`, `crownbar`, `crowndock`, `crownotify` | `vello 0.9` → `wgpu 29`, `peniko 0.6`, `kurbo 0.13` |
| B | `crownuikit`, `crownos-config` | `xilem 0.4` → `vello 0.6` → `wgpu 26`, `peniko 0.5`, `kurbo 0.12` |

`crowndictator` depends on both and locks **661 packages** — two `wgpu`, two
`vello`, two `calloop`, in one binary. Every other repo is 300–430.

The mechanism is one line in `crownos-config`:

```toml
[features]
default = ["xilem"]        # a settings crate pulling a GPU stack by default
xilem = ["dep:xilem"]
```

`crownpositor` already opts out with `default-features = false`. **`crowndictator`
could do the same unilaterally** — no change to `crownos-config`, no effect on
`crownuikit` — and would immediately drop to one `wgpu`, one `vello` and one
`calloop`. Flipping the default to opt-in is the fuller fix.

## Unused declared dependencies

Worth knowing because they affect build time and native prerequisites:

| Crate | Declared but unused |
|---|---|
| `crownshell` | `bluer` (pulls the whole D-Bus + BlueZ tree), `battery`, `tracing` |
| `crownbar` | `bluer`, `tracing` |
| `crowndock` | `tracing` |
| `crowncrate-linux` | `serde_json` (code uses CBOR), `gtk4` (no UI exists) |
| `lls-client`, `lls-server` | `serde` (nothing derives it) |

`crownshell`'s `bluer` is the expensive one — a leftover from when `crownbar`'s
code lived in that crate.

Also: `crowndictator` pulls `crownos-config` with **default features**, which
includes `xilem`. A headless daemon should not need a GUI toolkit;
`default-features = false` is likely correct.

### The dependency nobody would have predicted

`crowndictator` declares `ort` with the `cuda` feature and without
`default-features = false`. That reaches **`openssl-sys`** by way of `ureq` and
`native-tls`, so the dictation daemon needs OpenSSL development headers to
build. Nothing in the manifest suggests it, and the native dependency lists
originally omitted it; `deps.toml`'s `dictation` group now carries `openssl`.

---

## Toolchain

| | |
|---|---|
| Edition | 2024 in every Rust crate |
| MSRV | **1.88** — set by `vello 0.9` and `xilem 0.4`, declared as `rust-version = "1.88"` by all 11 crates |
| `rust-toolchain.toml` | Present in **all 11 Rust repos**, pinning `channel = "1.88.0"` exactly, with `rustfmt` and `clippy` |
| Resolver | `"3"` in `crownpositor` and `lls-protocol` |

The pin is exact, not a channel name, so rustup downloads 1.88.0 and uses it
regardless of your default toolchain. `crownpositor` uses let-chains, which is
part of why the floor is where it is.

One exception: the `issue1` worktree in `crowndictator` predates the migration —
it still uses path dependencies and has no `rust-toolchain.toml`.

### Cargo profiles

| Setting | Crates |
|---|---|
| `[profile.release] lto = "thin"`, `codegen-units = 1`, `strip = "symbols"` | `crownshell`, `crownbar`, `crowndock`, `crownotify` |
| same, without `strip` | `crowndictator` |
| `[profile.dev] split-debuginfo = "unpacked"` | `crownos-config`, `crownuikit` |
| no profile tuning | `crownpositor`, `crowncrate-linux`, `lls-protocol`, `crownlauncher` |

### Special cargo config

One file committed in a repository, in `crownbar`:

```toml
# crownbar/.cargo/config.toml
[target.x86_64-unknown-linux-gnu]
rustflags = ["-C", "link-arg=-fuse-ld=bfd"]
```

Forces the BFD linker for that crate. If you use `mold` or `lld` globally, it is
overridden here. You need `ld.bfd` from `binutils` on `PATH`.

The other `.cargo/config.toml` that matters — the `[patch.crates-io]` overlay —
sits *above* the checkouts and is deliberately in no repository's history.

---

## Crate naming

Package names no longer match their directories in two places, and the
repository name does not match a crate in one:

| Repo | Directory | Package name |
|---|---|---|
| `crownpositor` | `compositor/` | `crownpositor` |
| `crownpositor` | `config/` | `crownpositor-config` |
| `lls-protocol` | `client/`, `server/` | `lls-client`, `lls-server` |

The old names — `compositor`, `config`, `launcher` — were renamed because they
are unusable on crates.io and because `config` was a particularly unfortunate
name for a crate coexisting with `crownos-config` in the same dependency tree.
Inside `crownpositor` the dependency is still *aliased* to `config`
(`config = { package = "crownpositor-config", … }`), so `use config::…` in the
compositor source still means the compositor's own configuration types, not the
shared on-disk schema.

There is no crate named `lls-protocol`. The wire types live in
`server/src/protocol.rs` and `server/src/packet.rs`; extracting them into a
shared `protocol/` member is what would claim the name.

---

## See also

- [Workspace setup](../10-getting-started/workspace-setup.md) — the overlay this
  graph depends on
- [Project status](../00-overview/project-status.md) — what is broken right now
