# Contribution workflow

The mechanics of getting a change from your editor into a CrownOS repository.

[CONTRIBUTING.md](../../CONTRIBUTING.md) is the summary; this page has the
detail.

---

## Before anything else

**Nothing checks your pull request automatically.**
[`Crown-OS/.github`](https://github.com/Crown-OS/.github) does not exist on
GitHub, so the reusable workflows have never run and no repository has a
workflow history. Once that repository is pushed, Rust repos will run
`cargo fmt --check`, a build, clippy and `cargo test` on every push and pull
request — see [CI and releases](ci.md).

Two caveats worth internalising:

- **Clippy will not block, even then.** It is to run and report to the job
  summary, but a warning will not fail your PR. Read it anyway.
- **There is no CODEOWNERS and no branch protection.** Approval is enforced by
  convention, not by GitHub.

So run the checks locally before pushing. Until CI exists it is the only thing
that runs them at all, and the "what I ran and what it said" section of your PR
description is the only evidence a reviewer gets — including for anything CI
could never check, like whether the bar actually renders.

---

## 1. Fork and clone

Clone into the [flat sibling layout](../10-getting-started/workspace-setup.md),
using the **upstream repository name** even for your fork — path dependencies
resolve by directory name.

```bash
mkdir -p ~/src/crownos && cd ~/src/crownos
git clone git@github.com:<you>/crownshell.git crownshell
cd crownshell
git remote add upstream git@github.com:Crown-OS/crownshell.git
```

## 2. Check your base branch

**Every repository defaults to `main`.** Nine of them used `master` until August
2026; if your clone predates the rename, fix it before branching — see
[Workspace setup](../10-getting-started/workspace-setup.md#if-you-cloned-before-the-rename).

```bash
git remote show upstream | grep 'HEAD branch'    # should say: main
```

## 3. Branch

`<type>/<short-description-in-kebab-case>`, with the same type prefix as your
commits:

| Prefix | Use for |
|---|---|
| `feat/` | New features |
| `fix/` | Bug fixes |
| `docs/` | Documentation changes |
| `refactor/` | Refactoring without behaviour change |
| `chore/` | Maintenance — deps, tooling, config |
| `test/` | Adding or fixing tests |

```bash
git switch -c fix/dock-launches-applications
```

---

## 4. Commit

[Conventional Commits 1.0](https://www.conventionalcommits.org/en/v1.0.0/):

```
<type>(<scope>): <description>

[optional body]

[optional footers]
```

- Lowercase subject, imperative mood, no trailing period, ≤ 72 characters
- Body wrapped at 72, explaining *why* rather than *what*
- Breaking changes: `!` after the type/scope **and** a `BREAKING CHANGE:` footer
- One logical change per commit

**No sign-off.** CrownOS does not use the DCO. `git commit -s` is unnecessary.

Types: `feat`, `fix`, `docs`, `refactor`, `test`, `perf`, `chore`, `style`,
`revert`.

Scopes are in [CONTRIBUTING.md](../../CONTRIBUTING.md#scopes).

> **Do not copy existing commit messages.** None of the 49 commits in the
> organization's history use this convention — they are sentence-case
> descriptions like "Added xcursor support". Conventional Commits start now.

---

## 5. Check locally

Locally is the only place these run today — see [CI and releases](ci.md).

```bash
cargo fmt --all
cargo clippy --all-targets --all-features -- -D warnings
cargo test --all
```

Per-language commands and their caveats:
[Code standards](code-standards.md). Test-specific traps (D-Bus, the global
config dir): [Testing](testing.md).

Establish a baseline **before** you start. A clean clone of `crownbar`,
`crowndock`, `crownotify`, `crowndictator` or `crownpositor` does not even reach
the compiler — it fails at `cargo metadata` with
`failed to select a version for crownshell ^0.3`, because that version has never
been published. You need the `[patch.crates-io]` overlay described below before
any of these commands mean anything. If something still fails, check
[Project status](../00-overview/project-status.md) before debugging your
environment.

---

## 6. Clean the history

CrownOS merges by **rebase**, so every commit in your PR lands individually on
the base branch. That means each one should be a coherent change, and fixup
commits must be squashed away.

```bash
# While working: mark fixups at staging time
git commit --fixup HEAD

# Before review: collapse them
git rebase -i --autosquash upstream/main
```

## 7. Rebase and push

```bash
git fetch upstream
git rebase upstream/main
git push --force-with-lease origin fix/dock-launches-applications
```

Use `--force-with-lease`, never plain `--force`. Force-push only to your own
branch.

---

## 8. Open the pull request

Target the repository's **default branch**.

Copy the template body from
[`templates/.github/PULL_REQUEST_TEMPLATE.md`](../../templates/.github/PULL_REQUEST_TEMPLATE.md)
— GitHub will not auto-populate it, because the template is not installed in the
repos yet.

Open as a **Draft** if the work is in progress. Early feedback is encouraged.

In the Testing section, be specific:

```
cargo test --all      → 42 passed
cargo clippy -- -D warnings → clean
Ran `cargo run --example text_bar` under Sway; the clock updates and
the bar reserves its exclusive zone correctly.
```

Link an issue with `Closes #<n>` where one exists. For small self-explanatory
changes a clear summary is enough.

---

## 9. Review

- One maintainer approval is required.
- Address or explicitly dismiss every review thread before merge.
- Maintainers may ask for a rebase if the base branch has moved significantly.
- Maintainers merge with **rebase merge**. Do not squash-merge or create merge
  commits.

---

## Working across repositories

Some changes span repos — updating `crownshell`'s `request_frame` signature
affects `crownotify` and `crowndictator`, and that exact change is what broke
`crownotify` for months.

### Seeing your change locally

Every crate depends on a **version**, not a path — and the versions the manifests
name (`crownshell = "0.3"`, `crownos-config = "0.2"`) do not exist on crates.io.
Only `crownshell` 0.1.0 and 0.2.0 have ever been published. So the override is
not how you make a local edit visible; it is how the build resolves at all. Do
not change the dependency in `Cargo.toml` — put the override in a
`.cargo/config.toml` **above** your checkouts:

```toml
# ~/src/crownos/.cargo/config.toml   (untracked, in no repo)
[patch.crates-io]
crownshell = { path = "crownshell" }
```

`crownos-setup`'s `./bootstrap.sh --dev` writes it, and you need it before
anything downstream of `crownshell` will build. Full detail:
[Workspace setup](../10-getting-started/workspace-setup.md#the-overlay-mandatory).

### Getting it merged

1. Open the `crownshell` PR first, and say in the description which downstream
   repos it breaks.
2. Open the downstream PRs referencing it.
3. Ask a maintainer to sequence the merges.

CI will help here once it exists: `crownbar`, `crowndock`, `crownotify`,
`crowndictator` and `crownpositor` clone their dependencies and patch to them, so
their pull requests build against the current state of
`crownshell`/`crownos-config`. Today nobody sees a breaking change until a
downstream maintainer builds by hand.

### Then release in order

A downstream crate cannot be *published* until its dependency is on crates.io —
`release.yml` deliberately builds against published versions only. So the merge
order and the release order are the same: `crownshell` first, then everything
that depends on it. Nothing but `crownshell` 0.1.0 and 0.2.0 has been published
so far, so that order is still entirely ahead of the project. See
[Releasing](releasing.md#publish-order).

---

## Issues

Templates to copy:
[`templates/.github/ISSUE_TEMPLATE/`](../../templates/.github/ISSUE_TEMPLATE).

**Bug reports** need the commit hash, minimal reproduction steps, expected versus
actual behaviour, and system info — GPU, compositor, kernel, relevant package
versions. For anything involving rendering, the compositor matters: a bug under
`crownpositor` and a bug under Sway are usually different bugs.

**Feature requests** should describe the problem rather than only the solution,
say how it fits the CrownOS design language, and say whether it touches one repo
or several.

**Security issues** go through [SECURITY.md](../../SECURITY.md), not public
issues.
