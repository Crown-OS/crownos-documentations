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

## How cross-repo dependencies are tested

CrownOS crates depend on each other by **published crates.io version** — no
`path`, no git URLs. So a plain checkout of any repo builds on its own, and CI
needs no special handling for most of them.

Three repos still clone their dependencies, though, and it is worth
understanding why:

| Repo | `siblings` |
|---|---|
| `crownpositor` | `crownos-config` |
| `crownotify` | `crownshell` |
| `crowndictator` | `crownshell crownos-config` |
| everything else | none |

`rust.yml` clones each named sibling next to the repo under test and writes a
`[patch.crates-io]` table at the workspace root:

```toml
# $GITHUB_WORKSPACE/.cargo/config.toml, generated
[patch.crates-io]
crownshell = { path = "crownshell" }
```

Two reasons this is worth the machinery:

1. **It tests against the current state of the dependency, not the last
   release.** A breaking change in `crownshell` shows up on `crownbar`'s pull
   request rather than after publishing.
2. **It lets CI pass before a version exists on crates.io.** `crownbar` declares
   `crownshell = "0.3"`; without the override, CI could not go green until 0.3
   was already published — and you would not want to publish something CI had
   never built.

This is the same mechanism `crownos-setup bootstrap.sh --dev` writes on a
contributor's machine, so CI and local development resolve identically. That is
the point: the old arrangement had committed `path` dependencies and committed
`[patch]` tables, which meant CI, your checkout and a fresh clone could each
resolve differently.

**`release.yml` deliberately does none of this.** A released binary is built from
published dependencies only, or it cannot be reproduced from its tag. That is why
a repo cannot be released until everything it depends on is already on crates.io
— see [Releasing](releasing.md#publish-order).

---

## Native dependencies

`rust.yml` installs a base apt set on every Rust job, and repos needing more pass
a `packages` input:

| Repo | Extra `packages` |
|---|---|
| crownpositor | `libdrm-dev libinput-dev libseat-dev libudev-dev libpixman-1-dev xwayland` |
| crowndictator | `libasound2-dev libevdev-dev` |
| crowncrate-linux | `libgtk-4-dev libglib2.0-dev libpango1.0-dev libgdk-pixbuf-2.0-dev libgraphene-1.0-dev` |
| everything else | none |

**Neither list is written here, and neither is written in `rust.yml` by hand.**
Both come from
[`crownos-setup/deps.toml`](https://github.com/Crown-OS/crownos-setup/blob/main/deps.toml)
via `scripts/gen.py`, which also produces the
[native packages page](../10-getting-started/native-packages.md) and the
bootstrap script. To change what CI installs, edit `deps.toml`, regenerate, and
copy `generated/ci-packages.txt` across.

That indirection exists because the same list used to be maintained in three
places and had already diverged:

- CI installed **`libfontconfig-1-dev`**, which exists on neither Debian nor
  Ubuntu. The first CI run would have failed on it.
- CI omitted **`xwayland`** for `crownpositor`, despite the compositor enabling
  smithay's `xwayland` feature.
- `crowncrate-linux` passed only `libgtk-4-dev`, relying on the rest arriving
  transitively.

`libbluetooth-dev` is in the base set only because `crownshell` declares `bluer`
and never uses it. Removing that dependency would shorten the list for every
downstream repo.

---|---|
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
in [Project status](../00-overview/project-status.md#previously-broken-in-detail) and
both are small.

**crownbar** needs `ld.bfd` because of its committed `.cargo/config.toml`.
`binutils` is preinstalled on GitHub runners, so nothing extra is required.

---

## Releases

Full detail, including publish order and known hazards:
[Releasing](releasing.md).

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

- **No AUR packages.** These are Arch-specific and remain the prerequisite for
  the ISO shipping CrownOS packages rather than upstream archiso's rescue set.
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
  `rust-version = "1.88"`, and `check-versions.py` verifies every repo agrees.
- **No coverage.** `cargo-llvm-cov` and `tarpaulin` are both absent.
- **Issue and PR templates are not installed.** They are staged in
  [`templates/.github/`](../../templates/.github).
