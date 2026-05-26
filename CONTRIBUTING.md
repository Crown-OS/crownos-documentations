# Contributing to Crown-OS

Crown-OS is an AI-ready, Arch-based Linux distribution. This guide applies to all repositories in the [Crown-OS](https://github.com/Crown-OS) organization.

## Maintainers

| GitHub | Role |
|---|---|
| [@Viscous106](https://github.com/Viscous106) | Core maintainer |
| [@marvelxcodes](https://github.com/marvelxcodes) | Core maintainer |
| [@joe-Daniel29](https://github.com/joe-Daniel29) | Core maintainer |

All PRs require approval from one of the above before merge.

---

## Quickstart

```bash
# 1. Fork the target repo on GitHub, then clone your fork
git clone git@github.com:<you>/<repo>.git && cd <repo>

# 2. Create a branch
git checkout -b feat/your-feature

# 3. Make focused commits (see Commit Messages below)
git commit -s -m "feat(compositor): add corner radius animation"

# 4. Rebase onto current main before opening a PR
git fetch origin && git rebase origin/main

# 5. Open a PR — mark as Draft if work is still in progress
```

That's the core loop. The sections below explain each step in detail.

---

## Project Layout

Crown-OS is a **multi-repo** organization. Each repo has its own dev-setup instructions in its own `CONTRIBUTING.md` stub. The canonical guide is this document.

| Repository | Purpose | Primary language |
|---|---|---|
| `Crownpositor` | Wayland compositor + shell crates | Rust |
| `crowncrate-linux` | Ecosystem daemon (port 5252) | Rust |
| `CrownCrate` | Desktop app (Tauri + Solid.js) | TypeScript / Rust |
| `crowncrate-android` | Android companion | Kotlin |
| `crownos-iso` | Bootable archiso installer | Bash / config |
| `crownos-website` | Marketing site (Next.js) | TypeScript |
| `crownos-documentations` | This repo — all documentation | Markdown |

For architecture context, read [`CLAUDE.md`](../CLAUDE.md) at the workspace root.

---

## Branch Naming

Use the same type prefix as your commit message:

| Branch prefix | Use for |
|---|---|
| `feat/` | New features |
| `fix/` | Bug fixes |
| `docs/` | Documentation changes |
| `refactor/` | Refactoring without behavior change |
| `chore/` | Maintenance — deps, tooling, config |
| `ci/` | CI/CD pipeline changes |
| `test/` | Adding or fixing tests |

**Format:** `<type>/<short-description-in-kebab-case>`

```
feat/clipboard-sync-daemon
fix/compositor-crash-on-resize
docs/contribution-guide
chore/bump-smithay-0.8
```

---

## Commit Messages

Crown-OS enforces [Conventional Commits 1.0](https://www.conventionalcommits.org/en/v1.0.0/). CI rejects non-compliant commits.

### Format

```
<type>(<scope>): <description>

[optional body]

[optional footers]
```

### Rules

- **Subject line:** lowercase, imperative mood, no trailing period, max 72 characters
- **Body:** wrap at 72 characters, explain *why* not *what*
- **Breaking changes:** add `BREAKING CHANGE: <description>` in the footer
- **DCO sign-off required:** every commit must be signed with `git commit -s`

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
| `ci` | CI/CD pipeline changes |
| `style` | Formatting only — no logic change |
| `revert` | Reverts a prior commit |

### Scopes

Use the scope that matches the affected repository or crate:

| Scope | Covers |
|---|---|
| `compositor` | `Crownpositor/crates/compositor` |
| `bar` | `Crownpositor/crates/bar` |
| `launcher` | `Crownpositor/crates/launcher` |
| `notification` | `Crownpositor/crates/notification` |
| `greeter` | `Crownpositor/crates/greeter` |
| `dropshare` | `Crownpositor/crates/dropshare` |
| `crownpositor` | Workspace-level `Crownpositor` changes |
| `crowncrate` | `CrownCrate` desktop app |
| `crowncrate-linux` | `crowncrate-linux` daemon |
| `android` | `crowncrate-android` |
| `iso` | `crownos-iso` |
| `website` | `crownos-website` |
| `docs` | `crownos-documentations` |
| `ci` | Any repo's CI pipeline |

### Examples

```
feat(compositor): implement layer-shell surface support

fix(crowncrate-linux): resolve clipboard sync race on rapid paste

chore(crowncrate): bump smithay to 0.8

docs(contribution): add branch naming convention section

feat(android)!: migrate protocol to CBOR v2

BREAKING CHANGE: devices running the old JSON protocol must upgrade
crowncrate-linux before pairing.
```

---

## Keeping Your Branch Clean

Crown-OS uses **rebase merge**. Every commit from your PR lands individually on `main`. This means:

1. Each commit must pass CI on its own.
2. `main` stays perfectly linear — easy to `git bisect`.
3. WIP / fixup commits must be cleaned before review.

### Before requesting review

**Squash fixups into their parent commit:**

```bash
# Mark fixup commits at staging time
git commit --fixup HEAD

# Then collapse them
git rebase -i --autosquash origin/main
```

**Rebase onto current main:**

```bash
git fetch origin
git rebase origin/main
```

**Force-push to your branch** (your branch, never `main`):

```bash
git push --force-with-lease origin feat/your-feature
```

### Commit hygiene checklist

Before marking a PR ready for review, each commit should:

- [ ] Compile and pass tests independently (`cargo check`, `bun run check-types`, etc.)
- [ ] Have a valid Conventional Commits message
- [ ] Be signed off (`Signed-off-by:` line present — added by `git commit -s`)
- [ ] Not bundle unrelated changes

---

## Pull Request Process

### Opening a PR

1. Push your branch and open a PR against `main`.
2. Use **Draft PR** if the work is still in progress — it's encouraged for early feedback.
3. Fill in the PR template (auto-populated on GitHub) — the **Closes** field is required.

### PR template

```markdown
## Closes

<!-- Required. Link the issue this PR resolves. -->
<!-- Use one of: Closes #<n>  |  Fixes #<n>  |  Resolves #<n> -->
<!-- If no issue exists, open one before submitting this PR. -->

Closes #

## Summary
<!-- What does this PR do and why? -->

## Changes
<!-- Bullet list of notable changes -->

## Testing
<!-- How was this tested? Include commands or screenshots. -->

## Checklist
- [ ] Linked issue in the "Closes" section above
- [ ] Each commit follows Conventional Commits
- [ ] Each commit is signed off (`git commit -s`)
- [ ] Each commit compiles and passes CI independently
- [ ] Branch is rebased onto current `main`
- [ ] Relevant tests added or updated
```

### Review requirements

- **CI must pass** — all checks green before review is requested.
- **Linked issue required** — every PR must close an issue; PRs without a `Closes #<n>` line will be returned to the author.
- **One maintainer approval required** — [@Viscous106](https://github.com/Viscous106), [@marvelxcodes](https://github.com/marvelxcodes), or [@joe-Daniel29](https://github.com/joe-Daniel29). Enforced via CODEOWNERS.
- **No unresolved review threads** — address or explicitly dismiss all comments before merge.
- Maintainers may request a rebase if `main` has moved significantly.

### Merge

Maintainers merge using **rebase merge only**. Do not use squash or merge commits — the repo is configured to disable them.

---

## Code Standards

Automated formatters and linters run in CI. Run them locally before pushing.

### Rust (Crownpositor, crowncrate-linux, CrownCrate src-tauri)

```bash
cargo fmt --all          # format
cargo clippy --all-targets --all-features -- -D warnings   # lint
cargo test --all         # tests
```

- Follow `rustfmt` defaults — no manual overrides.
- All `clippy` warnings are errors in CI.
- Use `edition = "2024"` in all new crates.
- Minimize comments: only write one when the *why* is non-obvious.

### TypeScript / Solid.js (CrownCrate, crownos-website)

```bash
bun run lint     # biome lint
bun run format   # biome format
bun run check-types   # tsc --noEmit
```

- [Biome](https://biomejs.dev/) handles both linting and formatting. No ESLint, no Prettier.
- `crownos-website`: run `bun run lint` (Biome) in the website repo.

### Kotlin (crowncrate-android)

```bash
./gradlew ktlintCheck    # lint
./gradlew ktlintFormat   # format
./gradlew test           # unit tests
```

- Follows the official [Kotlin coding conventions](https://kotlinlang.org/docs/coding-conventions.html).
- Jetpack Compose: prefer stateless composables, hoist state.

### Shell / archiso (crownos-iso)

```bash
shellcheck airootfs/**/*.sh
```

- All shell scripts must pass `shellcheck` with no warnings.

---

## Issue Reporting

Use GitHub Issues on the relevant repository.

### Bug report

Include:
- Crown-OS version or commit hash
- Steps to reproduce (minimal)
- Expected vs. actual behavior
- System info: GPU, display server, relevant package versions

### Feature request

Include:
- The problem you're solving (not just the solution)
- How it fits the Crown-OS design language (monochromatic, minimal, professional)
- Whether it touches a single repo or multiple

### Security vulnerabilities

**Do not open a public issue.** Report via [GitHub private security advisories](https://docs.github.com/en/code-security/security-advisories/guidance-on-reporting-and-writing/privately-reporting-a-security-vulnerability) on the affected repository.

---

## DCO Sign-off

Crown-OS uses the [Developer Certificate of Origin (DCO)](https://developercertificate.org/). Every commit must contain:

```
Signed-off-by: Your Name <your@email.com>
```

Add it automatically with `git commit -s`. By signing off you certify that you have the right to submit the contribution under the project's license.

---

## Releases and Changelog

Changelogs are auto-generated from Conventional Commits. Breaking changes (`BREAKING CHANGE:` footer or `!` type suffix) trigger a major version bump. Release cadence is TBD — changelogs will be published in this documentation repo when releases begin.

---

## Code of Conduct

Crown-OS follows the [Contributor Covenant v2.1](https://www.contributor-covenant.org/version/2/1/code_of_conduct/). By contributing you agree to abide by its terms.

---

## Questions

Open a Discussion on the relevant GitHub repository or file an issue tagged `question`.
