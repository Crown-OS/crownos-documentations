# CI and releases

How CrownOS CI is wired, what it checks, and how to change it.

---

## Where it lives

The real logic is in **[`Crown-OS/.github`](https://github.com/Crown-OS/.github)**
as reusable workflows. Each repository carries a thin caller:

```yaml
# <repo>/.github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:

jobs:
  ci:
    uses: Crown-OS/.github/.github/workflows/rust.yml@main
```

The doubled `.github/.github/` is not a typo — the first is the repository name,
the second the directory inside it.

The reason for centralising: the native-dependency list is long and shared by
eleven Rust repos. Duplicating it sixteen times guarantees it drifts.

| Reusable workflow | Used by | Checks |
|---|---|---|
| `rust.yml` | the 11 Rust repos | rustfmt · build · clippy · test |
| `web.yml` | crownos-website | `bun install` · `biome check` · `next build` |
| `android.yml` | crowncrate-android | `assembleDebug` · unit tests · APK artifact |
| `docs.yml` | crownos-documentations | link check · status-marker consistency |
| `shell.yml` | crownos-iso | shellcheck |
| `release.yml` | 5 binary crates, on `v*` tags | release build · tarball · draft Release |

---

## What blocks a merge

| Check | Blocks? |
|---|---|
| `cargo fmt --all --check` | **Yes** |
| `cargo build --all-targets` | **Yes** |
| `cargo test --all` | **Yes** |
| `cargo clippy` | No — advisory |
| `biome check`, `next build` | **Yes** |
| `assembleDebug`, Gradle tests | **Yes** |
| Docs link check, status markers | **Yes** |
| shellcheck | **Yes** |

### Why clippy does not block

Roughly 15,000 lines of Rust have never been linted. Turning on `-D warnings`
today would make every repository red for reasons unrelated to whatever change is
under review, which trains people to ignore the badge.

So clippy runs on every PR and writes its findings to the job summary, without
failing the build. **Read it anyway** — the point is to burn the backlog down.

When it is clear, the change is one edit in `rust.yml`: drop
`continue-on-error: true` and append `-- -D warnings`. Nothing in the individual
repos needs touching.

---

## The sibling-checkout problem

This is the part that makes CrownOS CI non-generic.

Several crates declare `path = "../crownshell"` or `path = "../crownos-config"`.
A plain `actions/checkout` puts the repo at the workspace root, where `../`
resolves outside it and the build fails.

So `rust.yml` checks the caller repo into a directory named after itself, then
clones the named siblings alongside:

```
$GITHUB_WORKSPACE/
├── crowndictator/     <- the repo under test
├── crownshell/        <- sibling
└── crownos-config/    <- sibling
```

Callers declare what they need:

```yaml
    uses: Crown-OS/.github/.github/workflows/rust.yml@main
    with:
      siblings: crownshell crownos-config
```

| Repo | `siblings` |
|---|---|
| crownpositor | `crownos-config` |
| crownotify | `crownshell` |
| crowndictator | `crownshell crownos-config` |
| everything else | none |

Siblings are cloned at their default branch, shallow. That means **CI tests
against sibling `main`, not against your PR** — a cross-repo change needs
coordinated PRs. See
[Working across repositories](workflow.md#working-across-repositories).

---

## Native dependencies

`rust.yml` installs a base set on every Rust job:

```
pkg-config
libwayland-dev wayland-protocols libxkbcommon-dev
libvulkan-dev mesa-common-dev libegl1-mesa-dev libgbm-dev
libfontconfig-1-dev libfreetype-dev
libdbus-1-dev libbluetooth-dev
libxcb1-dev libx11-dev
```

Repos needing more pass `packages`:

| Repo | Extra |
|---|---|
| crownpositor | `libdrm-dev libinput-dev libseat-dev libudev-dev libpixman-1-dev` |
| crowndictator | `libasound2-dev libevdev-dev` |
| crowncrate-linux | `libgtk-4-dev` |

`libbluetooth-dev` is in the base set only because `crownshell` declares `bluer`
and never uses it. Removing that dependency would shorten this list for every
downstream repo.

---

## Repo-specific behaviour

**crownotify** passes `dbus: true`. Its tests register real well-known names on
the session bus, so CI runs them as
`dbus-run-session -- cargo test --all -- --test-threads=1`.

**crownos-config** tests set `CROWN_CONFIG_DIR=/tmp/crownos-ci` at the workflow
level, so nothing touches a real config directory.

**crowncrate-linux** and **lls-protocol** are **red on purpose**. Neither
compiles. The badge should say so rather than hide it — the errors are documented
in [Project status](../00-overview/project-status.md#known-broken-in-detail) and
both are small.

**crownbar** needs `ld.bfd` because of its committed `.cargo/config.toml`.
`binutils` is preinstalled on GitHub runners, so nothing extra is required.

---

## Releases

CD is deliberately minimal. Push a `v*` tag to `crownbar`, `crowndock`,
`crownotify`, `crowndictator` or `crownpositor` and `release.yml`:

1. builds `--release` for `x86_64-unknown-linux-gnu`
2. packages the binary, README and LICENSE into a `.tar.gz`
3. writes a `.sha256` alongside it
4. attaches both to a **draft** GitHub Release with generated notes

Draft, so you review before publishing.

The binary name is an input because several packages are not named after their
repository:

| Repo | Binary |
|---|---|
| crownpositor | `compositor` |
| crownlauncher | `launcher` |
| everything else | same as the repo |

### Not automated

- **No crates.io publish.** `crownshell` is the only crate with publish-ready
  metadata, and publishing freezes the misspelled `predule` module as permanent
  public API. Resolve that first.
- **No website deploy.** `crownos-website` has no `output: "export"` in
  `next.config.ts`, so a static Pages deploy cannot work without a source change.
  CI builds it; deploying is a separate decision.
- **No ISO build.** The profile is still unmodified upstream Arch, so a nightly
  would publish a generic rescue image.
- **No cross-compilation, no packaging, no signing.**

---

## Adding CI to a new repo

1. Create `.github/workflows/ci.yml` in the repo calling the right reusable
   workflow.
2. Pass `siblings` if it has path dependencies, and `packages` if it links
   anything outside the base set.
3. If it produces a binary worth shipping, add `.github/workflows/release.yml`
   too.
4. Add it to the tables on this page.

---

## Changing CI

Anything shared — the dependency list, the toolchain, the lint policy — changes
in `Crown-OS/.github`, once. Anything repo-specific goes in that repo's caller.

Reusable workflows are referenced `@main`, so a change there takes effect
everywhere on the next run. That is the point, and also the risk: **test a change
to `rust.yml` on one repo before merging it.** Point one caller at your branch:

```yaml
    uses: Crown-OS/.github/.github/workflows/rust.yml@my-branch
```

---

## Still missing

Worth knowing, and all reasonable contributions:

- **No CODEOWNERS and no branch protection.** Review requirements are convention,
  not enforcement.
- **No dependabot.** Several crates pin git dependencies with no `rev`, which
  automated bumps would make worse. Fix the pinning first — see
  [Dependency graph](../20-architecture/dependency-graph.md#version-skew).
- **No MSRV job.** CI uses stable. Only `crownshell` declares
  `rust-version = "1.85"`, and nothing verifies it.
- **No coverage.** `cargo-llvm-cov` and `tarpaulin` are both absent.
- **Issue and PR templates are not installed.** They are staged in
  [`templates/.github/`](../../templates/.github).
