# Releasing

How CrownOS versions, publishes and ships. Read
[CI and releases](ci.md) first for how the workflows are wired.

> **Nothing on this page has happened yet.** Two crates exist on crates.io —
> `crownshell` 0.1.0 and 0.2.0 — and both were published by hand. No other
> CrownOS crate is on any registry. The only git tag in the organization is
> `crownshell v0.2.0`. `publish.yml` and `release.yml` have never run, because
> `Crown-OS/.github` does not exist on GitHub. Read this as the plan, not as a
> record.

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

That is the destination, not the current state. **`crownshell` 0.3 has never
been published**, and neither has `crownos-config` 0.2, which the same manifests
also name. Until they are, every repo that depends on them resolves only through
a `[patch.crates-io]` override living **outside** every repository — mandatory,
not optional. See
[Workspace setup](../10-getting-started/workspace-setup.md#the-overlay-mandatory).

---

## Version policy

- [Semantic versioning](https://semver.org/). Pre-1.0, a minor bump may break.
- Versions live in `Cargo.toml`. `publish.yml` is written to refuse a publish
  where the tag disagrees with the manifest — once it can run at all.
- Shared dependency versions are declared once, in
  [`crown-versions.toml`](https://github.com/Crown-OS/.github/blob/main/crown-versions.toml),
  with `check-versions.py` to fail on drift. Both live in the unpushed `.github`
  repository, so neither is enforced today.
- **MSRV is 1.88.** All eleven Rust crates declare `rust-version = "1.88"` and
  ship a `rust-toolchain.toml` pinning `channel = "1.88.0"`. Edition 2024 needs
  only 1.85; `vello 0.9` and `xilem 0.4` need 1.88, and the dependency graph
  sets the floor.

---

## Publish order

This is the order to publish in, not a description of what has been published.
Every crate below is unpublished except `crownshell`, which is at 0.2.0.

The graph is two tiers deep. Within a tier there is no ordering constraint.

| Tier | Crates | Depends on |
|---|---|---|
| 0 | `crownshell`, `crownos-config`, `crownuikit`, `crownlauncher`, `crowncrate-linux`, `lls-client`, `lls-server` | nothing in CrownOS |
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
`src/lib.rs` and implements nothing. Neither is published, and **there are no
0.0.0 placeholder crates on crates.io** — that claim has appeared in these docs
before and is false. The names are unclaimed.

The plan, when the names are worth holding, is to publish each at **0.0.0** with
a "Placeholder — not yet released" description. crates.io is append-only: a
version can be yanked but never reused, and a name is held forever. Shipping a
broken crate as 0.1.0 is a permanent public record; a 0.0.0 placeholder is not.

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

The tag will trigger two workflows, once `Crown-OS/.github` is pushed:

- **`publish.yml`** — verifies the tag matches the manifest, dry-runs, then
  `cargo publish --locked`.
- **`release.yml`** — builds `--release`, tarballs the binary with README and
  LICENSE plus a `.sha256`, and attaches both to a **draft** GitHub Release.

Draft, so you review before publishing.

Neither has ever fired. `crownshell` 0.1.0 and 0.2.0 were published from a
maintainer's machine, and `crownshell v0.2.0` is the only tag any repo carries.
Until the workflows exist, step 2 is not a formality — it is the whole check.

`cargo publish --dry-run` is also meant to run on every pull request so metadata
problems surface long before a tag is cut. That too is waiting on the `.github`
repository.

### Setup, once

`publish.yml` needs a `CARGO_REGISTRY_TOKEN` organization secret. Create it at
[crates.io/settings/tokens](https://crates.io/settings/tokens), scoped to
publish-update only, and add it under the organization's Actions secrets.

---

## Known publishing hazards

**`crowndictator` will still fail on docs.rs.** Half of this is now fixed: `ort`
is declared `default-features = false` with `tls-rustls`, which removed
native-tls and therefore system OpenSSL from the graph entirely — that was the
reason this was the one crate in the tree that would not compile. But
`download-binaries` is still enabled, so `ort-sys` fetches a prebuilt ONNX
Runtime during the build, and docs.rs
[blocks network access](https://docs.rs/about/builds) and will never enable it.
The remaining fix is to put the ASR backend behind an opt-in feature so
`[package.metadata.docs.rs]` has something to switch off. Until then this crate
has no rendered documentation. Publish it last.

**Ownership is one person.** `crownshell` on crates.io is owned solely by the
user account that published it. For an organisation that is a bus factor of one:
nobody else can publish, yank, or add an owner. Create a `publishers` team in the
GitHub organisation and add it as a crates.io owner before the other twelve
crates go out, so ownership is org-level from the start rather than a migration
later:

```bash
cargo owner --add github:Crown-OS:publishers crownshell
```

**The names are unclaimed, not reserved.** Every CrownOS crate name except
`crownshell` is currently free on crates.io — `crownos-config`, `crownuikit`,
`crownbar`, `crowndock`, `crownotify`, `crowndictator`, `crownlauncher`,
`crownpositor`, `crownpositor-config`, `crowncrate-linux`, `lls-client`,
`lls-server`, and `lls-protocol` and `crownos` besides. Free means anyone can
take them. Publishing `0.0.0` placeholders is the only way to hold a name.

**`crownshell`'s `predule` typo is fixed in the unpublished 0.3.0.** 0.1.0 and
0.2.0 both shipped the misspelling, so removing it outright would break every
existing consumer. 0.3.0 is a breaking release and therefore the last cheap
moment to correct it: `src/prelude.rs` is now the real module, and `predule`
remains as a `#[deprecated]` re-export of it, scheduled for removal in 0.4.0.

That fix is local and unpushed, like the rest of 0.3.0. It has to go out *with*
0.3.0 — if 0.3.0 publishes without it, the typo is frozen for another release.

**Every committed `Cargo.lock` in a dependent repo is contaminated.** The locks
in `crownbar`, `crowndock`, `crownotify`, `crowndictator` and `crownpositor`
were generated inside the patched tree: they record `crownshell` and
`crownos-config` with **no `source =` line**, and carry `[[patch.unused]]`
stanzas. `cargo build --locked` from a fresh clone fails on them, and
`cargo publish --locked` would too. Regenerate each lock outside the patch
overlay — against the real registry, after the dependency is actually published
— before publishing anything.

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
