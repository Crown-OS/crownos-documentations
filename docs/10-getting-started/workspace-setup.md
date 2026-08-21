# Workspace setup

**Read this before cloning anything.** Several CrownOS crates depend on each
other by relative path. If your checkouts are not flat siblings, they will not
build, and the error message will not tell you why.

---

## The required layout

One directory, all repos as direct children, named exactly as the repository:

```
~/src/crownos/
├── crownpositor/
├── crownshell/
├── crownos-config/
├── crownbar/
├── crowndock/
├── crownotify/
├── crowndictator/
├── crownuikit/
├── crownlauncher/
├── crowncrate-linux/
├── crowncrate-android/
├── lls-protocol/
├── crownos-iso/
├── crownos-website/
└── crownos-documentations/
```

The parent directory name does not matter. What matters is that
`crownotify/../crownshell` resolves.

## Why

Three crates declare path dependencies that walk up one level:

| Crate | Declaration | Resolves to |
|---|---|---|
| `crownotify` | `crownshell = { path = "../crownshell" }` | `<parent>/crownshell` |
| `crowndictator` | `crownshell = { path = "../crownshell" }` | `<parent>/crownshell` |
| `crowndictator` | `crownos-config = { path = "../crownos-config" }` | `<parent>/crownos-config` |
| `crownpositor` | `[patch."https://github.com/crown-os/crownos-config"]`<br>`crownos-config = { path = "../crownos-config" }` | `<parent>/crownos-config` |

`crownpositor`'s manifest has an inline comment explaining the patch: it builds
against "the checkout next door rather than the last push", so a change to
`crownos-config` is visible to the compositor immediately.

Get the layout wrong and the error names the path it looked for, which is the
quickest way to see what went wrong:

```
error: failed to load source for dependency `crownos-config`
Caused by:
  unable to update /path/to/wrong/place/crownos-config
Caused by:
  failed to read /path/to/wrong/place/crownos-config/Cargo.toml
Caused by:
  No such file or directory (os error 2)
```

Note that **cargo canonicalises symlinks** before resolving `../`, so symlinking
repos into a flat directory does not work — the checkouts have to genuinely be
siblings on disk.

`crownbar` and `crowndock` use git dependencies instead and will build standalone
— but see the [version skew warning](#version-skew) below.

---

## Cloning

### Everything at once

```bash
mkdir -p ~/src/crownos && cd ~/src/crownos
gh repo list Crown-OS --limit 200 --json name,sshUrl --jq '.[] | [.name, .sshUrl] | @tsv' \
  | while IFS=$'\t' read -r name url; do
      [ -d "$name" ] && continue
      git clone "$url" "$name" || echo "FAILED: $name"
    done
```

`crowncrate-chrome` has no commits and will clone as an empty repository. That is
expected.

### Just what you need

You rarely need all of them. Minimum sets:

| Working on | Clone |
|---|---|
| `crownshell` itself | `crownshell` |
| `crownbar` or `crowndock` | that repo alone (git deps) |
| `crownotify` | `crownotify` + `crownshell` |
| `crowndictator` | `crowndictator` + `crownshell` + `crownos-config` |
| `crownpositor` | `crownpositor` + `crownos-config` |
| `crownos-config` | `crownos-config` |

### Forks

If you are contributing, fork on GitHub and clone your fork under the **upstream
repository name**, not the fork's name — the path dependencies use the repo name:

```bash
cd ~/src/crownos
git clone git@github.com:<you>/crownshell.git crownshell
cd crownshell
git remote add upstream git@github.com:Crown-OS/crownshell.git
```

---

## Default branches

**Every repository defaults to `main`.** There is nothing to remember.

If you want to confirm for a given repo:

```bash
git remote show origin | grep 'HEAD branch'
```

### If you cloned before the rename

Nine repositories used `master` until August 2026: `crownpositor`, `crownshell`,
`crownbar`, `crowndock`, `crownlauncher`, `crownotify`, `crowndictator`,
`crownuikit` and `crowncrate-linux`. A clone made before then still has a local
`master` tracking a branch that no longer exists.

To move an existing clone across:

```bash
git branch -m master main
git fetch origin
git branch -u origin/main main
git remote set-head origin -a
```

Nothing was rewritten — `main` points at the same commits `master` did — so this
is a rename, not a history change. Your own feature branches are unaffected;
rebase them onto `main` as usual.

---

## Version skew

`crownbar` and `crowndock` declare:

```toml
crownshell = { git = "https://github.com/crown-os/crownshell" }
```

with **no `rev`, `tag` or `branch`**. Their committed lockfiles pin commit
`de4ab90` at `crownshell` 0.1.0 — three commits behind current HEAD (0.2.0).

Consequences:

- As long as you do not run `cargo update`, they build against the old pinned
  revision and are unaffected by `crownshell` changes.
- The moment you run `cargo update`, they move to `crownshell` HEAD and may stop
  compiling.
- A change you make in your local `crownshell` checkout is **not** visible to
  `crownbar` or `crowndock`, because they fetch from GitHub, not from `../`.

If you need `crownbar` or `crowndock` to build against your local `crownshell`,
add a patch to the crate's `Cargo.toml`:

```toml
[patch."https://github.com/crown-os/crownshell"]
crownshell = { path = "../crownshell" }
```

Do not commit that patch unless the change is meant to land together.

There is also a live incompatibility between siblings: `crownotify` calls
`request_frame` with two arguments, `crowndictator` with three, and current
`crownshell` takes three. **They cannot both build against the same `crownshell`
revision.** See
[Project status](../00-overview/project-status.md#crownotify-does-not-build-against-current-crownshell).

---

## A worked example

Set up enough to build the compositor and run a bar inside it:

```bash
mkdir -p ~/src/crownos && cd ~/src/crownos

git clone git@github.com:Crown-OS/crownos-config.git
git clone git@github.com:Crown-OS/crownshell.git
git clone git@github.com:Crown-OS/crownpositor.git

# crownos-config is the foundation — build and test it first.
cd crownos-config && cargo test && cd ..

# crownshell has runnable examples that need no compositor of ours.
cd crownshell && cargo run --example text_bar && cd ..

# The compositor picks up ../crownos-config through its [patch] table.
cd crownpositor && cargo build && cd ..
```

If `crownpositor` fails to resolve `crownos-config`, your layout is wrong —
`crownos-config` must be a sibling of `crownpositor`, not inside it.

---

## Config isolation while developing

CrownOS components read `~/.config/crownos/`. To avoid a work-in-progress build
scribbling over your real settings, point them somewhere else:

```bash
export CROWN_CONFIG_DIR=/tmp/crownos-dev
```

Every component honours this, because they all go through `crownos-config`. It is
also what the test suite uses. See
[Environment variables](../50-reference/environment-variables.md).

---

## Next

[Build and run](build-and-run.md) — per-component build and run commands,
including how to get a nested compositor session.
