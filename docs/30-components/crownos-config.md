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

Every read, write and subscription names its **section** explicitly — the type
does not carry it. Use the generated `SECTION` constant rather than a string
literal, so a rename is a compile error instead of a file that silently stops
being read.

```rust
use crownos_config::{load, save, subscribe_typed, schema::Appearance};

let mut a: Appearance = load(Appearance::SECTION);
a.dark_mode = false;
save(Appearance::SECTION, &a)?;   // -> io::Result<()>

// Keep the Subscription alive — dropping it unregisters.
let _sub = subscribe_typed::<Appearance, _>(Appearance::SECTION, |a| { /* … */ });
```

| Signature | Delivers |
|---|---|
| `load<T: DeserializeOwned + Serialize + Default>(section: &str) -> T` | Parsed section; materialises defaults if the file is missing. **Never fails** — a parse error yields `T::default()`. |
| `save<T: Serialize>(section: &str, value: &T) -> io::Result<()>` | Atomic write (tmp + rename), records a hash for echo suppression |
| `subscribe(section: &str, cb) -> Subscription` | Raw `Vec<u8>` contents on change |
| `subscribe_typed::<T, _>(section: &str, cb) -> Subscription` | Parsed `T` on change |
| `subscribe_key(key, cb) -> Subscription` | Only when one specific field changes. **No section argument** — the key type carries `SECTION`. |
| `config_dir()` | `$CROWN_CONFIG_DIR`, else `dirs::config_dir()/crownos` |

`load` takes `Serialize` as well as `Deserialize` only because of the
"materialise the default" step. `save` is the only one that returns a `Result`.

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

The macro lives in `src/key.rs` — beside the `Key` trait whose impls it emits,
not in `src/schema/`.

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

**Key names are written labels, not W3C `KeyboardEvent.code` values.** Parsing
goes through `KeyCode::from_label`, so `KeyA`, `ArrowLeft` and `Digit1` **do
not parse**. The accepted spellings are exactly `A`–`Z`, `0`–`9`, `F1`–`F12`,
`Space`, `Enter`, `Tab`, `Escape`, `Backspace`, `Delete`, `Insert`, `Home`,
`End`, `PageUp`, `PageDown`, `CapsLock`, `Up`, `Down`, `Left`, `Right`,
`Minus`, `Equal`, `LeftBracket`, `RightBracket`, `Backslash`, `Semicolon`,
`Quote`, `Backquote`, `Comma`, `Period`, `Slash`. Matching is case-insensitive.

`KeyCode::from_code` does accept the W3C names, but nothing on the config path
calls it — it exists so a UI toolkit recording a chord has nothing to translate.

**A chord that does not parse fails the whole file's parse**, which makes
`load()` return the section default. One typo silently reverts every field in
that section, with nothing logged.

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

- **The `xilem` feature is on by default**, which means a headless consumer
  pulls in a whole GUI toolkit unless it opts out.
- Five sections are defined and unconsumed (above).

---

## License

**No LICENSE file.** See
[Project status](../00-overview/project-status.md#licensing).
