# Dependency graph

Build-time relationships between CrownOS crates, and the version skew you need to
know about.

---

## The graph

```
crownos-config ──(path)──────────────────────► crowndictator
               ──(git + [patch] to path)─────► crownpositor/config
                                                      │
                                                      └──(path)──► crownpositor/compositor

crownshell ──(git, unpinned)──► crownbar
           ──(git, unpinned)──► crowndock
           ──(path)───────────► crownotify
           ──(path)───────────► crowndictator

crownuikit        ── no CrownOS dependencies (xilem, winit, blinc_icons only)
crowncrate-linux  ── no CrownOS dependencies
lls-protocol      ── no CrownOS dependencies
crownlauncher     ── no dependencies at all
```

Only two crates are depended upon: `crownshell` and `crownos-config`. Everything
else is a leaf.

## Exact declarations

| Consumer | Declaration |
|---|---|
| `crownbar` | `crownshell = { git = "https://github.com/crown-os/crownshell" }` |
| `crowndock` | `crownshell = { git = "https://github.com/crown-os/crownshell" }` |
| `crownotify` | `crownshell = { path = "../crownshell" }` |
| `crowndictator` | `crownshell = { path = "../crownshell" }` |
| `crowndictator` | `crownos-config = { path = "../crownos-config" }` |
| `crownpositor` | `crownos-config = { git = "…/crownos-config", default-features = false }` <br> `[patch."https://github.com/crown-os/crownos-config"] crownos-config = { path = "../crownos-config" }` |

`crownpositor` disables default features on `crownos-config` to drop the `xilem`
feature — the compositor has no business pulling in a GUI toolkit. Its manifest
comment explains the patch: it builds against "the checkout next door rather than
the last push".

Note the **URL casing inconsistency**: dependencies use
`github.com/crown-os/…` (lowercase) while `crownshell`'s own `repository` field
says `github.com/Crown-OS/crownshell`. GitHub is case-insensitive here, so both
resolve, but `[patch]` keys must match the dependency URL exactly.

---

## Workspaces

Two repos are Cargo workspaces. There is **no org-wide workspace**.

| Repo | Members | Notes |
|---|---|---|
| `crownpositor` | `compositor`, `config` | `resolver = "3"`, has `[workspace.dependencies]` |
| `lls-protocol` | `client`, `server` | `resolver = "3"`, **also declares a root `[package]`** with no `src/` |

`lls-protocol`'s root manifest is broken twice over. It declares a `[package]`
alongside the `[workspace]` but has no `src/` at the root, so cargo fails at
manifest parse with `no targets specified in the manifest`. And its members
reference `serde`, `tokio` and `tokio-stream` as `{ workspace = true }` while
those dependencies sit under `[dependencies]` for that root package rather than
in `[workspace.dependencies]` — so fixing the first error just surfaces
`` `serde` was not found in workspace.dependencies ``.

Everything else is a standalone single-crate repository.

---

## Version skew (resolved)

Three problems used to live here. All three are fixed; the history is kept
because it explains why the current rules exist.

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

Both now declare `crownshell = "0.3"` from crates.io. Worth recording: when they
were finally built against 0.3.0, **both compiled unchanged**. The eight commits
of drift were additive.

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

### 3. Dependency drift between siblings — fixed

`anyhow` was declared four different ways (`"1"`, `"1.0.100"`, `"1.0.102"`,
`"1.0.104"`), `serde` three (`"1"`, `"1.0"`, `"1.0.228"`), and `crowndictator`
carried a hand-written comment saying its Wayland pins were "versions matched to
crownshell" — alignment nothing enforced.

Shared versions are now declared once in
[`crown-versions.toml`](https://github.com/Crown-OS/.github/blob/main/crown-versions.toml)
and `check-versions.py` fails CI on drift. Deliberate differences are recorded as
exceptions rather than being indistinguishable from accidents.

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

Both lanes are pinned in `crown-versions.toml` as recorded exceptions so neither
drifts further while the decision is open.

## Unused declared dependencies

Worth knowing because they affect build time and native prerequisites:

| Crate | Declared but unused |
|---|---|
| `crownshell` | `bluer` (pulls the whole D-Bus + BlueZ tree), `battery`, `tracing` |
| `crownbar` | `bluer`, `tracing` |
| `crowndock` | `tracing` |
| `crowncrate-linux` | `serde_json` (code uses CBOR), `gtk4`/`glib` (no UI exists) |
| `lls-protocol` | `serde` (nothing derives it) |

`crownshell`'s `bluer` is the expensive one — a leftover from when `crownbar`'s
code lived in that crate.

Also: `crowndictator` pulls `crownos-config` with **default features**, which
includes `xilem`. A headless daemon should not need a GUI toolkit;
`default-features = false` is likely correct.

---

## Toolchain

| | |
|---|---|
| Edition | 2024 in every Rust crate |
| MSRV | **1.88** — set by `vello 0.9` and `xilem 0.4`, declared by every crate, pinned in `rust-toolchain.toml` |
| `rust-toolchain.toml` | **Does not exist in any repo** |
| Resolver | `"3"` in `crownpositor` and `lls-protocol` |

`crownpositor` uses let-chains, so a recent stable toolchain is safest.

### Cargo profiles

| Setting | Crates |
|---|---|
| `[profile.release] lto = "thin"`, `codegen-units = 1`, `strip = "symbols"` | `crownshell`, `crownbar`, `crowndock`, `crownotify` |
| same, without `strip` | `crowndictator` |
| `[profile.dev] split-debuginfo = "unpacked"` | `crownos-config`, `crownuikit` |
| no profile tuning | `crownpositor`, `crowncrate-linux`, `lls-protocol`, `crownlauncher` |

### Special cargo config

One file, in `crownbar`:

```toml
# crownbar/.cargo/config.toml
[target.x86_64-unknown-linux-gnu]
rustflags = ["-C", "link-arg=-fuse-ld=bfd"]
```

Forces the BFD linker for that crate. If you use `mold` or `lld` globally, it is
overridden here. You need `ld.bfd` from `binutils` on `PATH`.

---

## Crate naming

Two package names do not match their repository:

| Repo | Package name |
|---|---|
| `crownlauncher` | `launcher` |
| `crownpositor` | workspace members are `compositor` and `config` |

`config` is a particularly unfortunate name for a crate that coexists with
`crownos-config` in the same dependency tree. When reading `crownpositor`,
`config` means the compositor's own compiled configuration; `crownos-config`
means the shared on-disk schema.

---

## See also

- [Workspace setup](../10-getting-started/workspace-setup.md) — the checkout
  layout these path dependencies require
- [Project status](../00-overview/project-status.md) — what is broken right now
