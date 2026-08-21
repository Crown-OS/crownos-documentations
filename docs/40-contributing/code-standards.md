# Code standards

Per-language conventions, and the commands that check them.

None of this is enforced by automation — there is no CI. Run it yourself.

---

## Rust

Twelve of sixteen repositories.

```bash
cargo fmt --all
cargo clippy --all-targets --all-features -- -D warnings
cargo test --all
```

### Formatting

**Use `rustfmt` defaults.** Do not add a `rustfmt.toml`.

There is exactly one in the organization —
`crowncrate-linux/rustfmt.toml` — and it is a liability rather than a model. It
is an ~80-key dump of default values containing:

- `edition = "2015"`, while the crate is edition 2024. `cargo fmt` passes the
  crate edition so it works, but a bare `rustfmt` invocation will mis-parse 2024
  syntax.
- `required_version = "1.5.1"` — rustfmt refuses to run on any other version.
- `fn_args_layout`, which is deprecated in favour of `fn_params_layout`.
- Numerous nightly-only keys alongside `unstable_features = false`, so they are
  silently ignored with warnings.

Deleting it is a reasonable patch.

### Linting

`cargo clippy --all-targets --all-features -- -D warnings` should be clean before
you open a PR.

There is no `clippy.toml` anywhere, and **no crate-level lint attributes** — a
grep for `#![deny]`, `#![warn]`, `#![forbid]` and `#![allow]` across every `.rs`
file returns nothing. Lint policy is by convention only.

### Language

- **Edition 2024** for all new crates.
- **Minimum Rust 1.85.** Only `crownshell` declares `rust-version`; edition 2024
  requires it regardless. There is no `rust-toolchain.toml` in any repo.
- `crownpositor` uses let-chains and `resolver = "3"`.

### Comments and documentation

This is where the project's own guidance conflicts, so be deliberate.

`crownos-website/AGENTS.md` says *"write maintainable modular code without too
much comments."* That is a website-repo instruction. The Rust codebase does the
opposite where it matters most: `crownos-config` and `crowndictator` — the two
best-regarded crates — carry substantial module-level documentation explaining
*why* a design is the way it is.

The working rule:

- **Module docs earn their place.** If a module encodes a non-obvious decision,
  write it down. `crownos-config/src/lib.rs` explains echo suppression;
  `crownpositor/src/layout/mod.rs` states the surface-blindness rule;
  `crowndictator/src/controller.rs` explains the single-channel design. Those
  comments are why those modules are maintainable.
- **Inline comments explain the non-obvious `why`**, not the `what`. Do not
  narrate the code.
- **Match the file you are editing.** `crownbar` has terse per-widget headers;
  `crownos-config` has essays. Both are fine in context.

### Structure conventions worth following

- **Keep layers honest.** `crownpositor`'s `layout/` cannot see a `WlSurface`,
  and that is why its tiling is unit-testable. Do not reach across a stated
  boundary.
- **One vocabulary per concept.** Chords and gestures both produce the same
  `Action` enum, deliberately. Do not add a parallel enum for a new input source.
- **Do not add a fifth spring.** There are already five separate spring
  implementations across the project. Use the one in the crate you are in.
- **Call `env_logger::init()` in `main`.** `crowndock` does not, and its warnings
  are invisible as a result.
- **Read settings through `crownos-config`**, not an ad-hoc file.
  `crowndictator/src/settings.rs` is the 40-line model.

### Dependencies

- Do not add a dependency you do not use. `crownshell` currently declares
  `bluer`, `battery` and `tracing` and uses none of them — `bluer` alone forces a
  D-Bus and BlueZ link requirement on every downstream consumer.
- **Pin git dependencies.** `crownbar` and `crowndock` declare `crownshell` by
  URL with no `rev`, so `cargo update` can silently move them onto a breaking
  HEAD. New git deps should carry a `rev` or `tag`.
- Match versions with siblings where crates interoperate. `crowndictator`'s
  manifest documents its `smithay-client-toolkit` and `calloop` versions as
  "matched to crownshell"; that discipline is worth extending.

---

## TypeScript — `crownos-website`

```bash
bun run lint      # biome check
bun run format    # biome format --write
bun run build     # next build, which also type-checks
```

- **Use Bun.** The lockfile is `bun.lock`; npm or yarn will create a competing
  one.
- **[Biome](https://biomejs.dev/) handles lint and format.** No ESLint, no
  Prettier. Config is `biome.json`: spaces, indent width 2, recommended rules
  plus the `next` and `react` domains, import organisation on.
- `suspicious.noUnknownAtRules` is off, because Tailwind v4 uses `@theme` and
  `@apply`.
- **There is no `check-types` script** and no test framework. Type errors surface
  through `bun run build`.
- Follow `AGENTS.md` in that repository for design language — monochromatic,
  minimal, smooth transitions, parallax between sections.
- Content is data-driven from `src/data/landing-page.ts`. Prefer editing data
  over components for copy changes.

---

## Kotlin — `crowncrate-android`

```bash
./gradlew assembleDebug
./gradlew test
```

- Follow the official
  [Kotlin coding conventions](https://kotlinlang.org/docs/coding-conventions.html).
- Jetpack Compose: prefer stateless composables and hoist state.
- **There is no ktlint plugin configured.** `./gradlew ktlintCheck` and
  `ktlintFormat` do not exist and will fail with "task not found". Adding the
  plugin would be a welcome `chore` PR.
- Requires Android SDK 36 and JDK 11+.

---

## Shell and archiso — `crownos-iso`

```bash
shellcheck path/to/script.sh
```

CrownOS-authored scripts should pass `shellcheck` cleanly. The profile is
currently unmodified upstream archiso, so there is very little CrownOS shell to
check yet — but that changes as soon as the profile is branded.

Note the inherited archiso scripts carry
`SPDX-License-Identifier: GPL-3.0-or-later` headers. Preserve them.

---

## Markdown — `crownos-documentations`

See [Documentation style](documentation-style.md).

---

## Licensing

**Add a LICENSE file to any new repository.** Thirteen of fifteen existing repos
have none, which makes them all-rights-reserved by default despite being
presented as open source.

Two further inconsistencies to resolve before copying either: the existing files
disagree on copyright holder (`marvelxcodes` vs `Crown-OS`), and `crowndictator`
declares `license = "MIT"` in `Cargo.toml` with no LICENSE file to back it.

Ask a maintainer which license and holder to use rather than guessing.
