# Testing

What tests exist, how to run them, and the traps that will waste your afternoon
otherwise.

---

## Coverage, honestly

270 test functions across 5 of 15 repositories. Six Rust repos have zero tests.

| Repo | `tests/` | `#[test]` fns | Style |
|---|---|---|---|
| **crownpositor** | no | **183** | Colocated `#[cfg(test)] mod tests` — animations, backend, handlers, input, layout, rendering, shell, and the `config` crate |
| **crownshell** | no | **42** | Colocated in `animations`, `blur`, `handler`, `text`, `wayland/pointer` |
| **crownos-config** | **yes** | **26 + 1** | Colocated units plus a 232-line integration test |
| **crowndictator** | no | 11 | Colocated in `controller`, `hotkey` |
| **crownotify** | **yes** | 7 | **Integration only** — real D-Bus, no unit tests |
| crownbar, crowndock, crownuikit, crownlauncher, crowncrate-linux, lls-protocol | no | **0** | — |
| crownos-website | — | 0 | No test framework at all |
| crowncrate-android | — | 2 | Untouched Android Studio template stubs |

CI runs `cargo test --all` on every push and pull request and it blocks, so a
failing test stops a merge. There is still **no fixtures directory, no test
runner script, and no coverage tooling** (`cargo-llvm-cov` and `tarpaulin` are
both absent).

---

## Running them

```bash
cargo test --all
```

That works for `crownpositor`, `crownshell`, `crowndictator` and
`crownos-config`. Two repos need more care.

### crownotify — needs a live session bus

Its tests register real well-known D-Bus names and drive them through `#[proxy]`
clients. Without a session bus they fail immediately.

```bash
dbus-run-session -- cargo test -- --test-threads=1
```

Both parts matter:

- **`dbus-run-session`** provides `DBUS_SESSION_BUS_ADDRESS`. In a headless
  container or a bare shell there is no session bus.
- **`--test-threads=1`** — the tests share well-known names, so they must not run
  concurrently. `call_notification.rs` holds a `OnceLock<Mutex<()>>` for this,
  but that lock is per-binary and there are two test binaries.

They will also fail if a **real `crownotify` is already running**, because the
name is taken. Stop it first.

An ignored test exercises the real phone-bridge daemon rather than a mock:

```bash
cargo test -- --ignored real_crowncrate    # start crowncrate first
```

(`crowncrate-linux` does not currently compile or expose D-Bus, so this cannot
pass today.)

### crownos-config — one test function, on purpose

`tests/e2e.rs` is a **single `#[test] fn e2e()`** that calls eight sub-checks in
sequence. The reason is stated in the file: `CROWN_CONFIG_DIR` is process-global,
and cargo runs test functions on parallel threads, so two tests pointing the
config directory at different places would race.

**Do not split it into separate `#[test]` functions.** Add a sub-check and call
it from `e2e()`.

What it covers:

| Sub-check | Asserts |
|---|---|
| `check_paths` | `config_dir()` and `path_for()` resolve correctly |
| `check_load_materialises_default` | Missing file writes defaults to disk |
| `check_save_load_round_trip` | Save then load returns the same value |
| `check_save_records_its_own_hash` | `save()` records what it wrote |
| `check_external_edit_breaks_the_hash` | An outside write is detected |
| `check_unparseable_file_is_not_clobbered` | A bad file survives a `load()` |
| `check_watcher` | Own-save suppressed · half-written file dropped · external write delivered · dropped `Subscription` stops delivery |
| `check_key_watcher` | Fires for its own key · silent for a neighbouring field · stops after drop |

Timing constants: `DELIVERED = 5s` (how long to wait for a change that should
arrive), `SILENT = 400ms` (how long to wait to prove one does not).

---

## Writing tests

### Isolate the config directory

Any test that touches settings must set `CROWN_CONFIG_DIR` to a temp path.
`crownos-config`'s e2e test keys its temp directory on the process ID.

```bash
CROWN_CONFIG_DIR=/tmp/crownos-test cargo test
```

Never let a test write to a real `~/.config/crownos/`.

### Prove the documented shape parses

`crownos-config/src/schema/compositor.rs` has a test that parses a **literal
hand-written RON sample** — the same shape the documentation shows — through the
same code path `load()` uses:

```rust
let parsed: Compositor = crate::parser::options()
    .from_str(sample)
    .expect("the documented shape must parse");
```

Copy this pattern when you add a config section. It is the only thing that keeps
documented examples from silently rotting.

### Keep the layout layer testable

`crownpositor`'s 183 tests exist largely because `layout/` cannot see a
`WlSurface`, a `Window`, an `Output` or the `Shell` — so tiling can be tested as
pure geometry. Preserve that boundary and new layout code stays testable; break
it and it stops being.

### Prefer colocated unit tests

The house style is `#[cfg(test)] mod tests` at the bottom of the file under test.
Use `tests/` only for genuine integration — `crownos-config` and `crownotify` do,
and both have a specific reason (process-global state; a real D-Bus surface).

---

## What is not tested at all

Worth knowing, both as risk and as opportunity:

- **`crownshell`'s Wayland layer.** The 42 tests cover animation, blur, text and
  pointer handling. Registry, output, seat, layer and data-device dispatch are
  untested.
- **Rendering.** Nothing does image comparison or golden-frame testing anywhere.
- **Everything in `crownbar`, `crowndock`, `crownuikit`.** Zero tests.
- **The phone bridge.** `crowncrate-linux` has no tests and does not compile;
  `crowncrate-android` has only template stubs.
- **The ISO.** No validation that the profile builds.

Adding a first test to a repo that has none is a genuinely useful contribution —
`crownbar`'s widgets, which parse `/sys` and `/proc` files, are a good target
because their inputs are just text.

---

## Reporting results

CI covers `cargo test`, but not whether the thing actually works on screen. Use
the PR description for what CI cannot check, and be specific:

```
cargo test --all             → 42 passed, 0 failed
cargo clippy -- -D warnings  → clean
Manually ran `cargo run --example text_bar` under Sway: the clock
updates every second and the exclusive zone is reserved correctly.
```

"Tests pass" without saying which is not useful.
