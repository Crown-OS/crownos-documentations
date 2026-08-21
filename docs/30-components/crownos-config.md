# crownos-config

**Status: Stable** · Rust · default branch `main` ·
[repo](https://github.com/Crown-OS/crownos-config)

On-disk configuration for CrownOS desktop apps, and — because there is no IPC
daemon — the mechanism by which components coordinate with each other.

**This is the best-documented and best-tested crate in the organization.** If you
want a model for how CrownOS code should be written, read this one.

---

## What it does

Every settings *section* is one RON file in the CrownOS config directory:
`~/.config/crownos/appearance.ron`, `~/.config/crownos/wifi.ron`, and so on. The
flat `<section>.ron` layout is a convention — the settings menu page name and the
file name are the same string, so a user who wants to hand-edit "Display" knows
to open `display.ron`.

Components `load()`, `save()` and `subscribe()`. Changes propagate live over
inotify, with echo suppression so a writer does not receive its own write back as
an external change.

The full mechanism is described in
[Config as IPC](../20-architecture/config-as-ipc.md). The complete field
reference is in [Config schema](../50-reference/config-schema.md).

---

## Build and test

No native dependencies beyond what `notify` needs. This is the fastest crate to
verify your toolchain with.

```bash
cd crownos-config
cargo build
cargo test
```

26 unit tests plus one integration test. Verified passing on rustc 1.95.0.

Dependencies: `dirs 6`, `notify 8`, `ron 0.12`, `serde 1`, and optional
`xilem 0.4.0` behind a **default-on** `xilem` feature. Consumers that do not need
the GUI views should pass `default-features = false` — `crownpositor` does;
`crowndictator` currently does not, and probably should.

---

## API

```rust
use crownos_config::{load, save, subscribe_typed, subscribe_key, schema::Appearance};

let mut a: Appearance = load();
a.dark_mode = false;
save(&a);

// Keep the Subscription alive — dropping it unregisters.
let _sub = subscribe_typed::<Appearance, _>(|a| { /* … */ });
```

| Function | Delivers |
|---|---|
| `load::<T>()` | Parsed section; materialises defaults if the file is missing |
| `save(&T)` | Atomic write (tmp + rename), records a hash for echo suppression |
| `subscribe(section, cb)` | Raw contents on change |
| `subscribe_typed::<T, _>(cb)` | Parsed `T` on change |
| `subscribe_key(key, cb)` | Only when one specific field changes |
| `config_dir()` | `$CROWN_CONFIG_DIR`, else `dirs::config_dir()/crownos` |

### Behaviour worth knowing

- **Missing file → defaults written to disk.** A fresh install ends up with a
  complete config rather than nothing.
- **Atomic saves.** Serialise, write `<file>.ron.tmp`, rename. A reader never
  sees a partial file.
- **Parse failure does not clobber.** `load()` returns the default and leaves the
  file alone — clobbering a config someone is halfway through editing would be
  worse.
- **Omitted fields fall back.** Every section derives `#[serde(default)]`.
- **Implicit `Some`.** The parser enables `IMPLICIT_SOME`, so optional fields are
  written `floating: true` rather than `floating: Some(true)`.

### The `section!` macro

Sections are declared through one macro that generates the struct with
`Serialize`/`Deserialize`/`#[serde(default)]`, a `SECTION` constant, a `Default`
from the per-field `= value`, a **zero-sized unit key type per field**, and a
`<Name>Key` enum.

The unit key types are what make `subscribe_key` type-safe: you pass `DarkMode`,
not the string `"dark_mode"`, so a typo is a compile error.

### Keybind type

`Keybind` = `Mods { meta, ctrl, alt, shift }` plus `Option<KeyCode>`. It
**serialises as a Display string**, so `input.ron` contains
`dictation_hotkey: "Super+Space"` rather than a nested struct.

Accepted modifier aliases: Super/Meta/Cmd/Win, Ctrl/Control, Alt/Option, Shift.
Key names are W3C `KeyboardEvent.code` values — `KeyA`, `Space`, `ArrowLeft`.
Unbinding is the literal value `"None"`.

### xilem integration

Behind the default-on `xilem` feature, `src/xilem_view.rs` provides `watch`,
`watched` and `watched_key` views that plumb config changes through xilem's
message path rather than a background callback. Implemented with `fork` +
`task_raw`, because the watcher view produces `NoElement` and so is not a
`WidgetView`.

---

## Testing style

Worth reading before you add tests here.

`tests/e2e.rs` is **a single `#[test] fn e2e()`** that calls eight sub-checks in
sequence. The reason is in the file: `CROWN_CONFIG_DIR` is process-global and
cargo runs test functions on parallel threads. Do not split it up.

It covers: paths · default materialisation · save/load round-trip · save
recording its own hash · external edit breaking the hash · unparseable file not
being clobbered · the watcher (own-save suppressed, half-written file dropped,
external write delivered, dropped subscription stops delivering) · the key
watcher (fires for its own key, silent for a neighbouring field, stops after
drop).

Timing constants: `DELIVERED = 5s`, `SILENT = 400ms`.

Unit tests live in `util.rs`, `key.rs`, `keybind.rs`, `xilem_view.rs` and
`schema/{compositor,input,keybinds}.rs`. The `compositor.rs` test parses a
literal hand-written RON sample to prove the documented shape actually works —
a pattern worth copying when you add a section.

---

## Consumers

Named explicitly in the schema doc comments:

| Section | Consumer |
|---|---|
| `compositor` | `crownpositor`, live |
| `input` | `crowndictator`, live |
| `keybinds` | the compositor — the only process that can honour a global chord |
| `appearance`, `display` | `crownpositor` |

Five sections have **no reader at all**: `sound`, `wifi`, `bluetooth`, `power`,
`keybinds`. Giving one of them a consumer is good, self-contained work.

---

## Known limitations

- **Schema skew with `crownpositor`.** The compositor reads
  `config.compositor.startup`; the `Compositor` struct here has no `startup`
  field. The two checkouts do not compile together as-is.
- **The `xilem` feature is on by default**, which means a headless consumer
  pulls in a whole GUI toolkit unless it opts out.
- Five sections are defined and unconsumed (above).

---

## License

**No LICENSE file.** See
[Project status](../00-overview/project-status.md#licensing).
