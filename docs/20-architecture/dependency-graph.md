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

## Version skew

Three separate problems, all live.

### 1. crownbar and crowndock are pinned to a stale crownshell

Their manifests declare `crownshell` by git URL with **no `rev`, `tag` or
`branch`**. What actually pins them is the committed lockfile:

| Crate | Lock pins | crownshell HEAD |
|---|---|---|
| `crownbar` | `de4ab90…` @ 0.1.0 | `6f3189a` @ 0.2.0 |
| `crowndock` | 0.1.0, **no `source` line** | `6f3189a` @ 0.2.0 |

`crowndock`'s lock entry having no `source` means it was generated against a
local path checkout rather than the git URL — a sign the manifest and the lock
have diverged.

Consequences: `cargo update` silently moves both onto `crownshell` HEAD, and they
may stop compiling. Changes you make to your local `crownshell` are invisible to
them.

**Fix:** pin explicitly.

```toml
crownshell = { git = "https://github.com/Crown-OS/crownshell", rev = "6f3189a" }
```

### 2. crownotify and crowndictator cannot share a crownshell revision

`crownshell`'s `request_frame` gained a parameter when text rendering landed:

| Crate | Calls | Compatible with |
|---|---|---|
| `crownshell` HEAD | `request_frame(&mut self, compositor_state, qh, text_cx)` | — |
| `crowndictator` | 3 arguments | HEAD ✓ |
| `crownotify` | 2 arguments | 0.1.0 only ✗ |

Since both use `path = "../crownshell"`, they resolve to the *same* checkout. One
of them will fail. Updating `crownotify` is the fix.

### 3. Dependency drift between siblings

Crates that should agree do not:

| Dependency | Versions in use |
|---|---|
| `tiny-skia` | 0.11 (`crowndock`), 0.12 (`crownpositor`) |
| `dirs` | 5 (`crowndock`), 6 (`crownos-config`, `crowndictator`) |
| `calloop` | 0.13 (`crownshell` and its consumers), 0.14.4 (`crownpositor`) |
| `glib` | 0.17 declared vs 0.18 required by `gtk4` 0.7 (`crowncrate-linux` — both in the lock) |

`crowndictator`'s manifest comments note that its `smithay-client-toolkit` and
`calloop` versions are deliberately "matched to crownshell". That discipline is
not applied elsewhere.

---

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
| MSRV | **1.85** — declared only by `crownshell`, implied by edition 2024 everywhere |
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
