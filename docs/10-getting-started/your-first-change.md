# Your first change

A walkthrough from picking something to work on to opening a pull request.

---

## 1. Pick something

CrownOS is early, which means there is a lot of obvious work. Some of it is
better as a first task than others.

### Good first tasks

**Add a LICENSE file.** Thirteen of the sixteen repositories have none on their
default branch — only `crownos-setup`, `crownos-documentations` and `crownshell`
carry one. This is a real problem: legally the rest are all-rights-reserved
despite being presented as open source. Ask a maintainer which license and
copyright holder to use — the three existing files say "The CrownOS Authors",
"Crown-OS" and "marvelxcodes" respectively — then add it.

**Make `crownbar` read its height from config.** It hardcodes 40 while
`appearance.bar_height` defaults to 32. `crowndictator` is the model to copy —
`settings.rs` there is 40 lines and shows the whole pattern.

**Give a config section a reader.** `sound.ron`, `wifi.ron`, `bluetooth.ron` and
`power.ron` exist and nothing consumes them. `crownbar` already has widgets for
sound, wifi and bluetooth that read `/sys` directly — wiring them to the config
is a self-contained change.

### Bigger, high-impact tasks

- Implement `ext-background-effect-v1` in `crownpositor` so blur actually works.
- Make `crowndock` launch applications — parse `Exec=` and spawn.
- Update `crownotify` to current `crownshell` (the `request_frame` arity change).
- Build the workspace overview: `shell/windows_view/` and
  `shell/workspaces_view/` are zero-byte files with actions and gestures already
  bound to them.

Full list: [Project status](../00-overview/project-status.md#where-help-is-most-useful).

> Before you start, ask a maintainer whether the fix already exists. A number of
> the build fixes these pages describe live only as uncommitted local changes —
> `crownotify`'s `request_frame` arity fix among them. On the default branch of
> each repo the old state is still what you will clone.

---

## 2. Set up

Follow [Workspace setup](workspace-setup.md). The short version:

```bash
mkdir -p ~/src/crownos && cd ~/src/crownos
git clone git@github.com:<you>/<repo>.git <repo>     # your fork, upstream name
cd <repo>
git remote add upstream git@github.com:Crown-OS/<repo>.git
```

**That clone will not build on its own.** Most Rust repos here declare
`crownshell = "0.3"` and `crownos-config = "0.2"`, and neither version exists on
crates.io — `crownshell` 0.1.0 and 0.2.0 are the only crates the organization has
ever published. Resolution comes from a `[patch.crates-io]` overlay in
`~/src/crownos/.cargo/config.toml`, above the checkouts, pointing both names at
local siblings. `crownos-setup`'s `./bootstrap.sh --dev` clones everything and
writes it; otherwise see
[Workspace setup](workspace-setup.md#the-overlay-mandatory).

Then isolate your config so a broken build cannot damage your real settings:

```bash
export CROWN_CONFIG_DIR=/tmp/crownos-dev
```

---

## 3. Confirm it builds *before* you change anything

This matters more than usual here, because several crates do not build on their
default branch, and several of the fixes described in these pages exist only as
uncommitted local changes. Establish a baseline:

```bash
cargo build
cargo test
```

If that fails before you have touched anything, check
[Project status](../00-overview/project-status.md) — the failure may be known and
documented. Two failures in particular are not your environment:

- **`error: failed to select a version for the requirement crownshell = "^0.3"`**
  — the overlay is missing or not above your checkout. See step 2.
- **`crowndictator` failing in the `openssl-sys` build script** — `ort` pulls
  `ureq` → `native-tls` → `openssl-sys`. You are missing the OpenSSL development
  headers; `crownos-setup`'s `dictation` dependency group installs them.

With the overlay, the pinned 1.88.0 toolchain and the native dependencies
installed, **all eleven Rust repositories pass `cargo check --all-targets`** and
`cargo fmt --all --check`.

`cargo build --locked` still fails from a fresh clone in every dependent repo:
the committed `Cargo.lock` files were generated inside a patched tree, record
`crownshell` and `crownos-config` with no `source` line, and carry
`[[patch.unused]]` stanzas.

---

## 4. Branch

```bash
git switch -c fix/bar-reads-config-height
```

Prefix matches your commit type: `feat/`, `fix/`, `docs/`, `refactor/`,
`chore/`, `test/`.

---

## 5. Write the change

Two conventions worth knowing before you start.

**Match the surrounding documentation density.** This codebase varies a lot.
`crownos-config` and `crowndictator` carry substantial module-level docs that
explain *why* a design is the way it is; `crownbar` has terse per-widget headers.
Write what the file around you writes.

**Do not add a sixth spring implementation.** There are already five
(`crownpositor`, `crownshell`, `crownbar`, `crowndock`, `crownuikit`). If you
need easing, use the one in the crate you are in.

---

## 6. Check it locally

**Nothing runs these but you.** There is no CI in any Crown-OS repository, and
the `Crown-OS/.github` repo that would host the shared workflows does not exist
on GitHub yet. What you run locally is the only check the change gets.

```bash
cargo fmt --all
cargo clippy --all-targets --all-features
cargo test --all
```

`cargo fmt` and `cargo test` are the ones that matter. **Clippy is advisory** —
non-blocking in the intended CI design, and the tree does not pass
`-D warnings` today, so running it gives you a wall of warnings unrelated to your
change. Read the ones your own diff caused and ignore the rest; a drive-by lint
sweep belongs in its own PR.

For `crownotify`, tests need a session bus:

```bash
dbus-run-session -- cargo test -- --test-threads=1
```

Full detail: [Testing](../40-contributing/testing.md).

---

## 7. Commit

[Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/), lowercase
subject, imperative mood, no trailing period, 72 characters:

```bash
git commit -m "fix(bar): read bar height from appearance.ron"
```

No sign-off is needed — CrownOS does not use the DCO.

Existing history does **not** follow this convention. Do not copy it.

---

## 8. Rebase and push

Every repository bases on `main`:

```bash
git fetch upstream
git rebase upstream/main
git push --force-with-lease origin fix/bar-reads-config-height
```

---

## 9. Open the pull request

Copy the body from
[`templates/.github/PULL_REQUEST_TEMPLATE.md`](../../templates/.github/PULL_REQUEST_TEMPLATE.md).
It is not installed in the repos yet, so GitHub will not fill it in for you.

Since nothing runs automatically, the PR body is the only record of what was
checked. Say which commands you ran and what they reported, and say what you
verified by hand — which compositor you ran under, and what you saw.

Open it as a **Draft** if you want early feedback — that is encouraged.

---

## 10. Review

One maintainer approval is required. Maintainers merge with **rebase merge**, so
each of your commits lands individually. Clean up fixups before requesting
review:

```bash
git commit --fixup HEAD
git rebase -i --autosquash upstream/main
```

---

## If you get stuck

- The failure is probably documented in
  [Project status](../00-overview/project-status.md).
- Build problems are usually the checkout layout — see
  [Workspace setup](workspace-setup.md).
- Otherwise, open an issue tagged `question`, or a Discussion on the repo.
