# Your first change

A walkthrough from picking something to work on to opening a pull request.

---

## 1. Pick something

CrownOS is early, which means there is a lot of obvious work. Some of it is
better as a first task than others.

### Good first tasks

**Add a LICENSE file.** Thirteen of fifteen repositories have none. This is a
real problem — legally they are all-rights-reserved despite being presented as
open source. Ask a maintainer which license and copyright holder to use (the two
existing files disagree), then add it.

**Make `crownbar` read its height from config.** It hardcodes 40 while
`appearance.bar_height` defaults to 32. `crowndictator` is the model to copy —
`settings.rs` there is 40 lines and shows the whole pattern.

**Write a missing README.** Twelve repos have none. `crownshell`'s README is the
house style; there is a shape to follow in
[Documentation style](../40-contributing/documentation-style.md).

**Fix `lls-protocol`'s workspace dependencies.** The members reference
`{ workspace = true }` deps that the root does not declare. Small, mechanical,
and it unblocks the crate.

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

---

## 2. Set up

Follow [Workspace setup](workspace-setup.md). The short version:

```bash
mkdir -p ~/src/crownos && cd ~/src/crownos
git clone git@github.com:<you>/<repo>.git <repo>     # your fork, upstream name
cd <repo>
git remote add upstream git@github.com:Crown-OS/<repo>.git
```

Then isolate your config so a broken build cannot damage your real settings:

```bash
export CROWN_CONFIG_DIR=/tmp/crownos-dev
```

---

## 3. Confirm it builds *before* you change anything

This matters more than usual here, because several crates do not build on their
default branch. Establish a baseline:

```bash
cargo build
cargo test
```

If that fails before you have touched anything, check
[Project status](../00-overview/project-status.md) — the failure may be known and
documented. Do not spend an afternoon debugging `lls-protocol`'s missing
`[workspace.dependencies]` thinking it is your environment.

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

**Do not add a fifth spring implementation.** There are already four
(`crownpositor`, `crownshell`, `crownbar`, `crowndock`, `crownuikit`). If you
need easing, use the one in the crate you are in.

---

## 6. Check it locally

There is no CI. Everything is on you:

```bash
cargo fmt --all
cargo clippy --all-targets --all-features -- -D warnings
cargo test --all
```

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

Say explicitly which checks you ran and what they reported. Since there is no CI,
that statement is the only evidence a reviewer has.

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
