# Contribution workflow

The mechanics of getting a change from your editor into a CrownOS repository.

[CONTRIBUTING.md](../../CONTRIBUTING.md) is the summary; this page has the
detail.

---

## Before anything else

**There is no CI in any CrownOS repository.** No GitHub Actions, no CODEOWNERS,
no branch protection, no PR template installed. Every check is one you run
locally, and every claim you make about having run it is taken on trust.

That makes the "what I ran and what it said" section of your PR description the
most important part of it.

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

```bash
cargo fmt --all
cargo clippy --all-targets --all-features -- -D warnings
cargo test --all
```

Per-language commands and their caveats:
[Code standards](code-standards.md). Test-specific traps (D-Bus, the global
config dir): [Testing](testing.md).

Establish a baseline **before** you start, because several crates do not build on
their default branch. If `cargo build` fails on a clean checkout, check
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

Some changes span repos — for example, updating `crownshell`'s `request_frame`
signature affects `crownotify` and `crowndictator`.

When that happens:

1. Open the `crownshell` PR first and say in the description which downstream
   repos it breaks.
2. Open the downstream PRs referencing it.
3. Ask a maintainer to sequence the merges.

Note that `crownbar` and `crowndock` consume `crownshell` by **git URL with no
rev**, so they will not see your change until their lockfiles are updated — and
when they are, they may break. See
[Dependency graph](../20-architecture/dependency-graph.md#version-skew).

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
