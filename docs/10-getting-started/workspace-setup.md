# Workspace setup

How to lay out your checkouts.

**This page used to open with a warning that the layout was mandatory and that
getting it wrong broke the build. That is no longer true**, and the reason is
worth understanding, because it is the same reason everyone used to end up with
a different build.

---

## What changed

CrownOS crates used to depend on each other by relative path
(`crownshell = { path = "../crownshell" }`) or by unpinned git URL
(`crownshell = { git = "…" }` with no `rev`). Both are reproducibility hazards:

- A **path** dependency builds against whatever is in your working tree. Two
  contributors with different local edits got different builds, and there was no
  version number involved at all.
- An **unpinned git** dependency floats on whatever `main` is. `crownbar` and
  `crowndock` were locked eight commits behind `crownshell` and would have moved
  the moment anyone ran `cargo update`.

Both are gone. Every CrownOS crate now depends on a **published version**:

```toml
crownshell = "0.3"
crownos-config = "0.2"
```

So a plain `git clone` of any single repository builds, anywhere on disk, with no
siblings required.

---

## Just working on one repo

```bash
git clone git@github.com:<you>/crownbar.git
cd crownbar && cargo build
```

That is the whole thing. Dependencies come from crates.io.

---

## Developing across repositories

If you are changing `crownshell` or `crownos-config` and want a component to see
that change, you need an override. **Do not edit the dependency in `Cargo.toml`**
— that is a committed file, and a local edit to it is exactly how the two
checkouts drift apart again.

Instead, put a `[patch.crates-io]` table in a `.cargo/config.toml` **above** your
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

`crownos-setup` writes it for you:

```bash
git clone https://github.com/Crown-OS/crownos-setup && cd crownos-setup
./bootstrap.sh --dev
```

That clones every repository into `~/src/crownos` (override with `--prefix=DIR`),
installs the native dependencies for your distro, and generates the file.

Two things to know:

- Paths in a config-file `[patch]` are resolved **relative to the directory
  containing `.cargo/`**, not to the repo you are building.
- Cargo prints `Patch … was not used in the crate graph` for repos that do not
  depend on every patched crate. Harmless.

To go back to building against the release, delete the file.

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

`crowncrate-chrome` has no commits and clones as an empty repository. Expected.

### Forks

Clone your fork under the **upstream repository name**. It no longer affects
dependency resolution, but the `[patch.crates-io]` overlay looks for directories
by crate name:

```bash
git clone git@github.com:<you>/crownshell.git crownshell
cd crownshell
git remote add upstream git@github.com:Crown-OS/crownshell.git
```

---

## Default branches

**Every repository defaults to `main`.**

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

rustup reads it and installs the right compiler on your first build. **1.88, not
1.85** — edition 2024 only needs 1.85, but `vello 0.9` and `xilem 0.4` declare
`rust-version = "1.88"`, and the dependency graph sets the floor.

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
