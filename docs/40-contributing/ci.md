# CI and releases

How CrownOS CI is wired, what it checks, and how to change it.

> **No CI runs today, and none ever has.**
> [`Crown-OS/.github`](https://github.com/Crown-OS/.github) does not exist on
> GitHub — the URL is a 404. The reusable workflows and the per-repo callers are
> written, but they exist only as untracked files on a maintainer's machine. No
> repository in the organization has ever had a workflow run, so there are no
> results, no badges and no history. Everything on this page describes what CI
> **will** do once the `.github` repository is pushed and the callers are
> committed.

---

## Where it lives

The shared logic belongs in
**[`Crown-OS/.github`](https://github.com/Crown-OS/.github)** as reusable
workflows — once that repository exists. Each repository is to carry a thin
caller:

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
eleven Rust repos. Duplicating it across sixteen repositories guarantees it
drifts.

The intended split:

| Reusable workflow | Will be used by | Checks |
|---|---|---|
| `rust.yml` | the 11 Rust repos | rustfmt · build · clippy · test |
| `web.yml` | crownos-website | `bun install` · `biome check` · `next build` |
| `android.yml` | crowncrate-android | `assembleDebug` · unit tests · APK artifact |
| `docs.yml` | crownos-documentations | link check · status-marker consistency |
| `shell.yml` | crownos-iso | shellcheck |
| `release.yml` | 5 binary crates, on `v*` tags | release build · tarball · draft Release |

---

## What will block a merge

Nothing blocks a merge today — there is no CI to block one, and no branch
protection either. This is the intended policy:

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

### Why clippy will not block

Roughly 15,000 lines of Rust have never been linted. Turning on `-D warnings`
from the first run would make every repository red for reasons unrelated to
whatever change is under review, which trains people to ignore the badge.

So clippy is to run on every PR and write its findings to the job summary,
without failing the build. **Read it anyway** — the point is to burn the backlog
down.

rustfmt is a different matter: all eleven Rust repos already pass
`cargo fmt --all --check` on rustfmt 1.88, so a blocking fmt job is green from
its first run.

When it is clear, the change is one edit in `rust.yml`: drop
`continue-on-error: true` and append `-- -D warnings`. Nothing in the individual
repos needs touching.

---

## How cross-repo dependencies are tested

CrownOS crates depend on each other by **version** — no `path`, no git URLs:

```toml
crownshell = "0.3"
crownos-config = "0.2"
```

**Neither version exists on crates.io.** The only CrownOS crates ever published
are `crownshell` 0.1.0 and 0.2.0, both pushed by hand. Nothing else in the
organization is on any registry. So a plain checkout does *not* build on its own:
a fresh clone of `crownbar`, `crowndock`, `crownotify`, `crowndictator` or
`crownpositor` fails at `cargo metadata` with
`failed to select a version for crownshell ^0.3`.

The `[patch.crates-io]` overlay is therefore **mandatory**, not a convenience for
cross-repo work. Every repo that depends on `crownshell` or `crownos-config`
needs a sibling checkout and an override, in CI and on your machine alike:

| Repo | `siblings` |
|---|---|
| `crownbar` | `crownshell` |
| `crowndock` | `crownshell` |
| `crownotify` | `crownshell` |
| `crownpositor` | `crownos-config` |
| `crowndictator` | `crownshell crownos-config` |
| everything else | none |

`crownbar` and `crowndock` were missing from that list; their callers now pass
`siblings: crownshell` too.

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
2. **It is the only way the build resolves at all.** `crownbar` declares
   `crownshell = "0.3"`, which has never been published. Without the override
   there is nothing to resolve against and cargo stops before it compiles a line.

This is the same mechanism `crownos-setup`'s `./bootstrap.sh --dev` writes on a
contributor's machine, so CI and local development will resolve identically. That
is the point: the old arrangement had committed `path` dependencies and committed
`[patch]` tables, which meant CI, your checkout and a fresh clone could each
resolve differently.

**`release.yml` deliberately does none of this.** A released binary is built from
published dependencies only, or it cannot be reproduced from its tag. That is why
a repo cannot be released until everything it depends on is already on crates.io
— and today nothing is, so nothing downstream of `crownshell` can be released at
all. See [Releasing](releasing.md#publish-order).

---

## Native dependencies

`rust.yml` is written to install a base apt set on every Rust job, with repos
needing more passing a `packages` input:

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

- The workflow installed **`libfontconfig-1-dev`**, which exists on neither
  Debian nor Ubuntu. The first CI run would have failed on it.
- It omitted **`xwayland`** for `crownpositor`, despite the compositor enabling
  smithay's `xwayland` feature.
- `crowncrate-linux` passed only `libgtk-4-dev`, relying on the rest arriving
  transitively.

`libbluetooth-dev` is in the base set only because `crownshell` declares `bluer`
and never uses it. Removing that dependency would shorten the list for every
downstream repo.

---

## Repo-specific behaviour

**crownotify** passes `dbus: true`. Its tests register real well-known names on
the session bus, so CI will run them as
`dbus-run-session -- cargo test --all -- --test-threads=1`.

**crownos-config** tests set `CROWN_CONFIG_DIR=/tmp/crownos-ci` at the workflow
level, so nothing touches a real config directory.

**crowndictator needs OpenSSL on the runner.** `ort` pulls `ureq`, which pulls
`native-tls`, which pulls `openssl-sys` — nothing in the manifest suggests it,
and it was missing from the native dependency lists until recently. It is in
`deps.toml`'s `dictation` group now, so any job generated from that file gets
it. With the patch overlay in place and the pinned 1.88.0 toolchain, **all
eleven Rust crates** pass `cargo check --all-targets`, and all eleven pass
`cargo fmt --all --check`. See
[Project status](../00-overview/project-status.md#landed-locally-not-yet-pushed).

**crownbar** needs `ld.bfd` because of its committed `.cargo/config.toml`.
`binutils` is preinstalled on GitHub runners, so nothing extra is required.

---

## Releases

Full detail, including publish order and known hazards:
[Releasing](releasing.md).

CD is deliberately minimal. Once the workflows are live, pushing a `v*` tag to
`crownbar`, `crowndock`, `crownotify`, `crowndictator` or `crownpositor` will
make `release.yml`:

1. builds `--release` for `x86_64-unknown-linux-gnu`
2. packages the binary, README and LICENSE into a `.tar.gz`
3. writes a `.sha256` alongside it
4. attaches both to a **draft** GitHub Release with generated notes

Draft, so you review before publishing.

None of this has happened yet. The only git tag anywhere in the organization is
`crownshell v0.2.0`, and it predates these workflows; the two published
`crownshell` versions were uploaded by hand.

The binary name is a workflow input because packages have not always been named
after their repository. They are now — the renames are done, so the input is
currently the identity:

| Repo | Package and binary |
|---|---|
| crownpositor | `crownpositor` |
| crownlauncher | `crownlauncher` — has no `release.yml` at all |
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
2. Pass `siblings` if it depends on another CrownOS crate — every such
   dependency needs the overlay — and `packages` if it links anything outside
   the base set.
3. If it produces a binary worth shipping, add `.github/workflows/release.yml`
   too.
4. Add it to the tables on this page.

---

## Changing CI

Anything shared — the dependency list, the toolchain, the lint policy — changes
in `Crown-OS/.github`, once. Anything repo-specific goes in that repo's caller.

Reusable workflows are referenced `@main`, so a change there will take effect
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
- **No dependabot.** There are no git dependencies left anywhere in the tree, so
  the reason previously given here no longer holds. The current reason is that
  `crown-versions.toml` owns every dependency more than one repo declares, and
  per-repo bumps would fight it. Raise the version there and propagate with
  `scripts/sync-versions.py`.
- **No MSRV job.** All eleven Rust crates already declare
  `rust-version = "1.88"` and ship a `rust-toolchain.toml` pinning
  `channel = "1.88.0"`, so the floor is consistent — but nothing checks that it
  stays that way. `check-versions.py` is written to verify it and has never run.
- **No coverage.** `cargo-llvm-cov` and `tarpaulin` are both absent.
- **Issue and PR templates are not installed.** They are staged in
  [`templates/.github/`](../../templates/.github).
