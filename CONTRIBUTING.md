# Contributing to CrownOS

CrownOS is an Arch-based, Wayland-native Linux distribution. This guide applies
to every repository in the [Crown-OS](https://github.com/Crown-OS) organization.

New here? Read [Project status](docs/00-overview/project-status.md) first. It
tells you what actually builds today, which saves you from picking up a component
that is a bare skeleton as your first task.

---

## Maintainers

| GitHub | Role |
|---|---|
| [@Viscous106](https://github.com/Viscous106) | Core maintainer |
| [@marvelxcodes](https://github.com/marvelxcodes) | Core maintainer |
| [@joe-Daniel29](https://github.com/joe-Daniel29) | Core maintainer |

One maintainer approval is required before merge.

> CI runs on every push and pull request, from reusable workflows in
> [`Crown-OS/.github`](https://github.com/Crown-OS/.github). **`rustfmt` and the
> test suite block; clippy is advisory for now.** There is still **no CODEOWNERS
> file and no branch protection**, so approval is enforced by convention rather
> than by GitHub. Run the checks locally anyway — it is faster than waiting for
> a runner.

---

## Quickstart

```bash
# 1. Fork the target repo on GitHub, then clone your fork. Anywhere is fine.
git clone git@github.com:<you>/<repo>.git

# 2. Create a branch
cd <repo>
git switch -c feat/your-feature

# 3. Make focused commits (see Commit Messages below)
git commit -m "feat(compositor): add corner radius animation"

# 4. Rebase onto main before opening a PR
git fetch origin
git rebase origin/main

# 5. Open a PR — mark as Draft if work is still in progress
```

**Any layout works.** Crates depend on published crates.io versions, so a single
`git clone` builds anywhere. You only need siblings if you are changing
`crownshell` or `crownos-config` and want a component to see it — and then the
override goes in a `.cargo/config.toml` *above* your checkouts, never in a
committed `Cargo.toml`. See
[Workspace setup](docs/10-getting-started/workspace-setup.md#developing-across-repositories).

---

## Project layout

CrownOS is a **multi-repo** organization — 17 repositories, no monorepo, no
umbrella Cargo workspace. Crates depend on each other by published crates.io
version, never by path or git, which is what makes every checkout resolve
identically. Full detail with status markers is in the
[Component map](docs/00-overview/component-map.md); the short version:

| Repository | Purpose | Language | Default branch |
|---|---|---|---|
| [`crownpositor`](docs/30-components/crownpositor.md) | Wayland compositor (Smithay) | Rust | `main` |
| [`crownshell`](docs/30-components/crownshell.md) | Layer-shell + Vello framework | Rust | `main` |
| [`crownbar`](docs/30-components/crownbar.md) | Status bar | Rust | `main` |
| [`crowndock`](docs/30-components/crowndock.md) | Dock | Rust | `main` |
| [`crownlauncher`](docs/30-components/crownlauncher.md) | App launcher | Rust | `main` |
| [`crownotify`](docs/30-components/crownotify.md) | Notification daemon | Rust | `main` |
| [`crowndictator`](docs/30-components/crowndictator.md) | Voice dictation | Rust | `main` |
| [`crownuikit`](docs/30-components/crownuikit.md) | Widget kit (xilem) | Rust | `main` |
| [`crowncrate-linux`](docs/30-components/crowncrate-linux.md) | Phone-bridge daemon | Rust | `main` |
| [`crownos-config`](docs/30-components/crownos-config.md) | Shared settings crate | Rust | `main` |
| [`lls-protocol`](docs/30-components/lls-protocol.md) | Media streaming protocol | Rust | `main` |
| [`crowncrate-android`](docs/30-components/crowncrate-android.md) | Android companion | Kotlin | `main` |
| [`crowncrate-chrome`](docs/30-components/crowncrate-chrome.md) | Browser companion | — | *(empty)* |
| [`crownos-iso`](docs/30-components/crownos-iso.md) | archiso profile | Shell | `main` |
| [`crownos-website`](docs/30-components/crownos-website.md) | Landing page | TypeScript | `main` |
| [`crownos-documentations`](docs/30-components/crownos-documentations.md) | This repo | Markdown | `main` |
| [`crownos-setup`](https://github.com/Crown-OS/crownos-setup) | Cross-distro setup, native deps | Shell / Nix | `main` |

**Every repository defaults to `main`.** The nine that used `master` were renamed
in August 2026; if you have an older clone, see
[Workspace setup](docs/10-getting-started/workspace-setup.md#if-you-cloned-before-the-rename).

For how the pieces talk to each other, read
[Architecture overview](docs/20-architecture/overview.md).

---

## Branch naming

Use the same type prefix as your commit message:

| Branch prefix | Use for |
|---|---|
| `feat/` | New features |
| `fix/` | Bug fixes |
| `docs/` | Documentation changes |
| `refactor/` | Refactoring without behavior change |
| `chore/` | Maintenance — deps, tooling, config |
| `test/` | Adding or fixing tests |

**Format:** `<type>/<short-description-in-kebab-case>`

```
feat/dock-launch-applications
fix/compositor-crash-on-resize
docs/contribution-guide
chore/bump-smithay-0.8
```

---

## Commit messages

CrownOS uses [Conventional Commits 1.0](https://www.conventionalcommits.org/en/v1.0.0/)
**going forward**. History before 2026-08 does not follow it — do not use existing
commits as a style reference.

### Format

```
<type>(<scope>): <description>

[optional body]

[optional footers]
```

### Rules

- **Subject:** lowercase, imperative mood, no trailing period, max 72 characters
- **Body:** wrap at 72 characters, explain *why* rather than *what*
- **Breaking changes:** `!` after the type/scope, and a `BREAKING CHANGE:` footer
- **One logical change per commit.** Do not bundle a refactor with a fix.

No sign-off is required. CrownOS does not use the DCO.

### Types

| Type | Use for |
|---|---|
| `feat` | New user-facing feature |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `refactor` | Code change — no feature, no fix |
| `test` | Adding or correcting tests |
| `perf` | Performance improvement |
| `chore` | Tooling, deps, build, config |
| `style` | Formatting only — no logic change |
| `revert` | Reverts a prior commit |

### Scopes

The scope is the repository name, minus a `crown`/`crownos-` prefix where that
reads better. Within `crownpositor`, use the crate.

| Scope | Covers |
|---|---|
| `compositor` | `crownpositor/compositor` |
| `positor-config` | `crownpositor/config` |
| `shell` | `crownshell` |
| `bar` | `crownbar` |
| `dock` | `crowndock` |
| `launcher` | `crownlauncher` |
| `notify` | `crownotify` |
| `dictator` | `crowndictator` |
| `uikit` | `crownuikit` |
| `config` | `crownos-config` |
| `crowncrate-linux` | `crowncrate-linux` |
| `android` | `crowncrate-android` |
| `lls` | `lls-protocol` |
| `iso` | `crownos-iso` |
| `website` | `crownos-website` |
| `docs` | `crownos-documentations` |

### Examples

```
feat(compositor): implement ext-background-effect-v1 handler

fix(dock): launch the application on icon click

chore(bar): pin the crownshell git dependency to a rev

docs(architecture): describe config-as-IPC

feat(android)!: send CBOR frames instead of line-delimited JSON

BREAKING CHANGE: devices running the old JSON protocol must upgrade
crowncrate-linux before pairing.
```

---

## Keeping your branch clean

CrownOS uses **rebase merge**. Every commit from your PR lands individually, so
history stays linear and `git bisect` stays useful. That means WIP and fixup
commits must be cleaned up before review.

**Squash fixups into their parent commit:**

```bash
git commit --fixup HEAD          # mark it at staging time
git rebase -i --autosquash origin/main
```

**Rebase onto the current default branch:**

```bash
git fetch origin
git rebase origin/main
```

**Force-push to your branch** — your branch, never the default branch:

```bash
git push --force-with-lease origin feat/your-feature
```

### Commit hygiene checklist

Before marking a PR ready for review, each commit should:

- [ ] Build on its own (`cargo check`, `bun run build`, `./gradlew assembleDebug`)
- [ ] Have a valid Conventional Commits message
- [ ] Not bundle unrelated changes

---

## Pull request process

### Opening a PR

1. Push your branch and open a PR against the repo's **default branch**
   `main`.
2. Use a **Draft PR** if the work is in progress. Early feedback is encouraged.
3. Fill in the PR template. Copy it from
   [`templates/.github/PULL_REQUEST_TEMPLATE.md`](templates/.github/PULL_REQUEST_TEMPLATE.md)
   — it is not yet installed in the repos, so GitHub will not auto-populate it.

### Review requirements

- **Checks are run locally by you.** Say in the PR which ones you ran and what
  they reported. See [Code standards](docs/40-contributing/code-standards.md).
- **Link an issue** with `Closes #<n>` where one exists. If the change is small
  and self-explanatory, a clear summary is enough.
- **One maintainer approval.**
- **No unresolved review threads** before merge.
- Maintainers may ask for a rebase if the base branch has moved a lot.

### Merge

Maintainers merge with **rebase merge**. Do not squash-merge or create merge
commits.

---

## Code standards

Full detail per language: [Code standards](docs/40-contributing/code-standards.md).
The short version:

### Rust

```bash
cargo fmt --all
cargo clippy --all-targets --all-features -- -D warnings
cargo test --all
```

- Follow `rustfmt` defaults. The one `rustfmt.toml` in the org
  (`crowncrate-linux`) is a legacy artifact and should not be copied.
- Edition 2024 for all new crates. Minimum toolchain is **Rust 1.88**, pinned
  per repo in `rust-toolchain.toml`.
- Document the *why*. The best-documented crate in the org is
  [`crownos-config`](docs/30-components/crownos-config.md) — read it before
  deciding how much to write.

### TypeScript (`crownos-website`)

```bash
bun run lint      # biome check
bun run format    # biome format --write
bun run build     # next build — also type-checks
```

[Biome](https://biomejs.dev/) handles lint and format. No ESLint, no Prettier.
There is no `check-types` script and no test framework in this repo today.

### Kotlin (`crowncrate-android`)

```bash
./gradlew assembleDebug
./gradlew test
```

Follow the official [Kotlin coding conventions](https://kotlinlang.org/docs/coding-conventions.html).
Prefer stateless composables and hoist state. There is **no ktlint plugin
configured**, so `ktlintCheck` does not exist — do not run it.

### Shell / archiso (`crownos-iso`)

```bash
shellcheck path/to/script.sh
```

Applies to CrownOS-authored scripts. The profile is currently unmodified upstream
archiso, so there is little to check yet.

---

## CI

How it is wired, what blocks a merge, and how releases work:
[CI and releases](docs/40-contributing/ci.md).

---

## Testing

What exists, and how to run it, is in [Testing](docs/40-contributing/testing.md).
Two things that will bite you otherwise:

- `crownotify`'s tests need a live session bus:
  `dbus-run-session -- cargo test -- --test-threads=1`
- `crownos-config`'s e2e test is deliberately one `#[test]` function because
  `CROWN_CONFIG_DIR` is process-global. Do not split it up.

---

## Issue reporting

Use GitHub Issues on the relevant repository. Templates to copy are in
[`templates/.github/ISSUE_TEMPLATE/`](templates/.github/ISSUE_TEMPLATE).

**Bug report** — include the commit hash, minimal reproduction steps, expected vs.
actual behaviour, and system info (GPU, compositor, kernel, relevant package
versions).

**Feature request** — describe the problem, not just your solution; say how it
fits the CrownOS design language (monochromatic, minimal, professional); and say
whether it touches one repo or several.

**Security vulnerabilities** — do not open a public issue. See [SECURITY.md](SECURITY.md).

---

## Releases and changelog

`crownshell` is published on crates.io (0.2.0, with docs.rs builds) and carries
the organization's only git tag. The remaining crates are being published now —
the order, and what blocks each one, is in
[Releasing](docs/40-contributing/releasing.md).

A `v*` tag runs `publish.yml`, which pushes to crates.io, and `release.yml`,
which attaches a binary tarball to a draft GitHub Release. There is no
`CHANGELOG.md` yet and nothing is published to the AUR.

Conventional Commits are adopted now so that changelog generation becomes possible
later. When releases begin, changelogs will be published in this repository.

---

## Code of Conduct

CrownOS follows the [Contributor Covenant v2.1](CODE_OF_CONDUCT.md). By
contributing you agree to abide by its terms.

---

## Questions

Open a Discussion on the relevant repository, or file an issue tagged `question`.
