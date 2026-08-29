# Releasing

How CrownOS versions, publishes and ships. Read
[CI and releases](ci.md) first for how the workflows are wired.

---

## Why crates.io matters here

It is not primarily about distribution. It is how CrownOS makes everyone's build
identical.

Before this, `crownbar` and `crowndock` declared
`crownshell = { git = "…" }` with **no `rev`, `tag` or `branch`**, and
`crownotify`, `crowndictator` and `crownpositor` used relative `path`
dependencies. That meant three different answers to "which crownshell am I
building against?" depending on your checkout layout, when you last ran
`cargo update`, and what was in your working tree. `crownbar`'s lockfile was
pinned eight commits behind; `crowndock`'s lockfile recorded a *path* package
while its manifest said *git*.

Depending on a published version removes the question:

```toml
crownshell = "0.3"      # the same bytes for everyone, forever
```

Local cross-repo work uses an override that lives **outside** every repository,
so a checkout can never disagree with CI — see
[Workspace setup](../10-getting-started/workspace-setup.md#developing-across-repositories).

---

## Version policy

- [Semantic versioning](https://semver.org/). Pre-1.0, a minor bump may break.
- Versions live in `Cargo.toml`. `publish.yml` refuses to publish if the tag
  disagrees with the manifest.
- Shared dependency versions are declared once, in
  [`crown-versions.toml`](https://github.com/Crown-OS/.github/blob/main/crown-versions.toml),
  and `check-versions.py` fails CI on drift.
- **MSRV is 1.88**, pinned per repo in `rust-toolchain.toml`. Edition 2024 needs
  only 1.85; `vello 0.9` and `xilem 0.4` need 1.88, and the dependency graph
  sets the floor.

---

## Publish order

The graph is two tiers deep. Within a tier there is no ordering constraint.

| Tier | Crates | Depends on |
|---|---|---|
| 0 | `crownshell`, `crownos-config`, `crownuikit`, `crownlauncher`, `lls-client`, `lls-server` | nothing in CrownOS |
| 1 | `crownbar`, `crowndock`, `crownotify`, `crowndictator`, `crownpositor-config` | tier 0 |
| 2 | `crownpositor` | `crownpositor-config` |

**`cargo publish --workspace` does not help here.** It is still nightly-only on
the pinned 1.88 toolchain, so the two workspace repos publish their members one
at a time, in order, via `publish.yml`'s `packages:` input:

```yaml
    with:
      packages: crownpositor-config crownpositor
```

Across repos the order is manual too — a crate cannot be published until
everything it depends on is already on the registry.

### Name changes

Five packages were named generically and **all five names were already taken on
crates.io**, so they were renamed:

| Repo | Was | Now |
|---|---|---|
| `crownlauncher` | `launcher` | `crownlauncher` |
| `crownpositor` | `compositor` | `crownpositor` |
| `crownpositor/config` | `config` | `crownpositor-config` |
| `lls-protocol/client` | `client` | `lls-client` |
| `lls-protocol/server` | `server` | `lls-server` |

The compositor's `use config::…` statements were **not** rewritten. Cargo's
dependency-rename field keeps the local name while the published name changes:

```toml
config = { package = "crownpositor-config", path = "../config", version = "0.1.0" }
```

### Placeholders

`crownlauncher` is a three-line hello-world and `crowncrate-linux` has an empty
`src/lib.rs` and implements nothing. Both are published at **0.0.0** with a
"Placeholder — not yet released" description, purely to hold the name.

crates.io is append-only: a version can be yanked but never reused, and a name
is held forever. Shipping a broken crate as 0.1.0 is a permanent public record;
a 0.0.0 placeholder is not.

---

## Cutting a release

```bash
# 1. Bump the version in Cargo.toml.
# 2. Confirm it will publish, without publishing:
cargo publish --dry-run --locked

# 3. Tag and push. The tag must match the manifest version.
git tag v0.3.0
git push origin v0.3.0
```

The tag triggers two workflows:

- **`publish.yml`** — verifies the tag matches the manifest, dry-runs, then
  `cargo publish --locked`.
- **`release.yml`** — builds `--release`, tarballs the binary with README and
  LICENSE plus a `.sha256`, and attaches both to a **draft** GitHub Release.

Draft, so you review before publishing.

`cargo publish --dry-run` also runs on every pull request, so metadata problems
surface long before a tag is cut.

### Setup, once

`publish.yml` needs a `CARGO_REGISTRY_TOKEN` organization secret. Create it at
[crates.io/settings/tokens](https://crates.io/settings/tokens), scoped to
publish-update only, and add it under the organization's Actions secrets.

---

## Known publishing hazards

**`crowndictator` will fail on docs.rs.** `ort` is declared with the `cuda`
feature and without `default-features = false`, so `ort-sys` downloads a
prebuilt ONNX Runtime during the build. docs.rs
[blocks network access](https://docs.rs/about/builds) and will never enable it,
so the docs build goes red permanently. A `[package.metadata.docs.rs]` block
cannot fix this — it cannot switch off a default feature of a transitive
dependency. The fix is `default-features = false` plus moving `cuda` behind an
opt-in feature, in `crowndictator`'s own manifest. Publish it last.

**`crownshell`'s `predule` typo is already frozen.** 0.1.0 and 0.2.0 both shipped
the misspelling. 0.3.0 adds `prelude` and keeps `predule` as a deprecated
re-export; removing it waits for 1.0.

**Binary crates need system libraries that `cargo install` will not provide.**
`cargo install crownpositor` fails on a machine without libdrm, libinput,
libseat, libudev and pixman. Point users at
[`crownos-setup`](https://github.com/Crown-OS/crownos-setup), which installs them
first.

---

## Not automated

- **No `CHANGELOG.md`.** Conventional Commits are in place so generation is
  possible; nothing generates one yet.
- **No AUR packages.** The prerequisite for the ISO shipping CrownOS packages
  instead of upstream archiso's rescue set.
- **No website deploy.** `crownos-website` has no `output: "export"` in
  `next.config.ts`, so a static Pages deploy needs a source change first.
- **No ISO build.** The profile is still unmodified upstream Arch.
- **No cross-compilation and no signing.** `x86_64-unknown-linux-gnu` only.
