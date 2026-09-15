# Workspace setup

How to lay out your checkouts, and why the layout is mandatory.

> **Read this before cloning anything.** A plain `git clone` of `crownbar`,
> `crowndock`, `crownotify`, `crowndictator` or `crownpositor` does **not**
> build. It fails before compilation starts, at `cargo metadata`, with
>
> ```
> error: failed to select a version for the requirement `crownshell = "^0.3"`
> ```
>
> The fix is one file, and `crownos-setup` writes it for you. Do not work around
> it by editing a `Cargo.toml`.

---

## Why a plain clone fails

CrownOS crates used to depend on each other by relative path
(`crownshell = { path = "../crownshell" }`) or by unpinned git URL
(`crownshell = { git = "…" }` with no `rev`). Both are reproducibility hazards:

- A **path** dependency builds against whatever is in your working tree. Two
  contributors with different local edits got different builds, and there was no
  version number involved at all.
- An **unpinned git** dependency floats on whatever `main` is. `crownbar` and
  `crowndock` were locked eight commits behind `crownshell` and would have moved
  the moment anyone ran `cargo update`.

Both are gone. Every CrownOS crate now declares a **version**:

```toml
crownshell = "0.3"
crownos-config = "0.2"
```

The problem is that **neither of those versions exists on crates.io**. The only
CrownOS crates ever published are `crownshell` 0.1.0 and 0.2.0. `crownos-config`
has never been published at all. So the manifests describe a world that is one
publish away from being true, and until that publish happens the version numbers
resolve to nothing.

Something has to supply those crates locally. That something is a
`[patch.crates-io]` overlay.

---

## The overlay (mandatory)

Put a `[patch.crates-io]` table in a `.cargo/config.toml` **above** your
checkouts. Cargo walks up from the working directory to find it, so one file
covers every repo:

```
~/src/crownos/
├── .cargo/config.toml      <- the override; in no repo's git history
├── crownshell/
├── crownos-config/
├── crownbar/
└── …
```

```toml
# ~/src/crownos/.cargo/config.toml
[patch.crates-io]
crownshell = { path = "crownshell" }
crownos-config = { path = "crownos-config" }
```

**Do not edit the dependency in `Cargo.toml` instead.** That is a committed file,
and a local edit to it is exactly how the two checkouts drift apart again.

`crownos-setup` writes the overlay for you, and this is the recommended path:

```bash
git clone https://github.com/Crown-OS/crownos-setup && cd crownos-setup
./bootstrap.sh --dev
```

That clones the eleven Rust repositories plus `crownos-documentations` and
`crownos-setup` into `~/src/crownos` (override with `--prefix=DIR` or
`CROWNOS_SRC`), installs the native dependencies for your distribution, and
generates the file. It does **not** clone `crownos-iso`, `crownos-website`,
`crowncrate-android` or `crowncrate-chrome` — clone those by hand if you need
them.

Three things to know:

- Paths in a config-file `[patch]` are resolved **relative to the directory
  containing `.cargo/`**, not to the repo you are building.
- Cargo prints `Patch … was not used in the crate graph` for repos that do not
  depend on every patched crate. Harmless.
- Every committed `Cargo.lock` in a dependent repo was generated *inside* a
  patched tree: the `crownshell` and `crownos-config` entries carry no
  `source =` line, and the files contain `[[patch.unused]]` stanzas. So
  `cargo build --locked` from a fresh clone fails even with the overlay in
  place. Build without `--locked` and let cargo regenerate.

Deleting the overlay does not get you back to "building against the release" —
there is no release to build against. It gets you back to the resolver error.

### Why not a committed `[patch]`?

`crownpositor` used to commit one:

```toml
[patch."https://github.com/crown-os/crownos-config"]
crownos-config = { path = "../crownos-config" }
```

Two problems. It made a sibling checkout **mandatory** for everyone who cloned
the repo — and when the path did not exist, `cargo build` failed outright rather
than falling back. And it keyed the patch on `github.com/crown-os/…` while
`crownshell`'s metadata says `github.com/Crown-OS/…`; cargo treats those as
different sources, so a patch written against the other spelling silently does
nothing.

The overlay above the checkouts has neither problem: it is keyed on
`crates-io`, and it lives outside every repo's history.

---

## Which repos actually need it

| Repo | Needs the overlay? |
|---|---|
| `crownbar`, `crowndock`, `crownotify` | Yes — `crownshell = "0.3"` |
| `crowndictator` | Yes — `crownshell = "0.3"` and `crownos-config = "0.2"` |
| `crownpositor` | Yes — `crownos-config = "0.2"` |
| `crownshell`, `crownos-config`, `crownuikit`, `crownlauncher`, `crowncrate-linux`, `lls-protocol` | No — no CrownOS dependencies. These five do clone and build standalone. |

If you are only ever going to touch `crownshell` or `crownos-config`
themselves, a bare clone is enough. Everything else needs the layout.

---

## Cloning everything

```bash
mkdir -p ~/src/crownos && cd ~/src/crownos
gh repo list Crown-OS --limit 200 --json name,sshUrl --jq '.[] | [.name, .sshUrl] | @tsv' \
  | while IFS=$'\t' read -r name url; do
      [ -d "$name" ] && continue
      git clone "$url" "$name" || echo "FAILED: $name"
    done
```

Then write the overlay by hand, or run `crownos-setup`'s `./bootstrap.sh --dev`
in the same prefix — it skips repositories that are already cloned.

`crowncrate-chrome` has zero commits and does not appear in the org listing at
all.

### Forks

Clone your fork under the **upstream repository name**. The `[patch.crates-io]`
overlay looks for directories by crate name, so a fork checked out as
`crownshell-myfork` will not be found:

```bash
git clone git@github.com:<you>/crownshell.git crownshell
cd crownshell
git remote add upstream git@github.com:Crown-OS/crownshell.git
```

---

## Default branches

**Every repository defaults to `main`.**

### If you cloned before the rename

If you cloned before the August 2026 rename, nine repos (`crownpositor`,
`crownshell`, `crownbar`, `crowndock`, `crownlauncher`, `crownotify`,
`crowndictator`, `crownuikit`, `crowncrate-linux`) still have a local `master`
tracking a branch that no longer exists:

```bash
git branch -m master main
git fetch origin
git branch -u origin/main main
git remote set-head origin -a
```

Nothing was rewritten, so this is a rename rather than a history change.

---

## Toolchain

Every Rust repo pins its toolchain:

```toml
[toolchain]
channel = "1.88.0"
components = ["rustfmt", "clippy"]
```

rustup reads it and downloads that exact compiler on your first build, whatever
your default toolchain is. **1.88, not 1.85** — edition 2024 only needs 1.85, but
`vello 0.9` and `xilem 0.4` declare `rust-version = "1.88"`, and the dependency
graph sets the floor.

One exception: the `issue1` worktree in `crowndictator` has no
`rust-toolchain.toml` and still uses path dependencies. It predates the
migration.

---

## Config isolation while developing

CrownOS components read `~/.config/crownos/`. Point them elsewhere so a
work-in-progress build cannot damage your real settings:

```bash
export CROWN_CONFIG_DIR=/tmp/crownos-dev
```

Every component honours it, because they all go through `crownos-config`. It is
what the test suite uses, and what the Nix devShell sets. See
[Environment variables](../50-reference/environment-variables.md).

---

## Next

[Build and run](build-and-run.md) — per-component build and run commands,
including how to get a nested compositor session.
