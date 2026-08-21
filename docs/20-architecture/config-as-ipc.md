# Config as IPC

The single most important thing to understand about CrownOS.

Components do not send each other messages. They read and write RON files in a
shared directory and watch it with inotify. There is no message bus, no socket
protocol, and no daemon in the middle.

---

## The layout on disk

One file per *section*, flat, named exactly as the section:

```
~/.config/crownos/
├── appearance.ron
├── bluetooth.ron
├── compositor.ron
├── display.ron
├── input.ron
├── keybinds.ron
├── notifications.ron
├── power.ron
├── sound.ron
└── wifi.ron
```

The naming is a deliberate convention, stated in `crownos-config`'s module docs:
the settings-menu page name and the file name are the same string, so a user who
wants to hand-edit "Display" knows to open `display.ron`.

Full field reference: [Config schema](../50-reference/config-schema.md).

## Where the directory is

```rust
pub const CONFIG_DIR_ENV: &str = "CROWN_CONFIG_DIR";

pub fn config_dir() -> PathBuf {
    if let Some(dir) = std::env::var_os(CONFIG_DIR_ENV) && !dir.is_empty() {
        return PathBuf::from(dir);
    }
    dirs::config_dir().unwrap_or_else(|| PathBuf::from(".")).join("crownos")
}
```

`$CROWN_CONFIG_DIR` wins if set and non-empty; otherwise XDG's config dir plus
`crownos`. Use the environment variable when developing so a work-in-progress
build cannot damage your real settings.

---

## Why RON, not TOML or JSON

[RON](https://github.com/ron-rs/ron) is Rust's own object notation. It maps
cleanly onto enums and tuples, which matters here — `LayoutMode::ScrollingColumns`
and `position: (0, 0)` both round-trip without a lossy encoding.

The parser is configured with implicit `Some`:

```rust
pub(crate) fn options() -> ron::Options {
    ron::Options::default()
        .with_default_extension(ron::extensions::Extensions::IMPLICIT_SOME)
}
```

So an optional field is written the same way a required one is —
`floating: true`, not `floating: Some(true)`. Hand-editing stays natural.

> The project website says "TOML profiles". That is wrong; the implementation is
> RON. See [Documentation drift](../00-overview/project-status.md#documentation-drift).

---

## Reading and writing

```rust
use crownos_config::{load, save, schema::Appearance};

let mut appearance: Appearance = load();
appearance.dark_mode = false;
save(&appearance);
```

Three behaviours to know:

**A missing file materialises defaults.** `load()` on a file that does not exist
writes `T::default()` to disk and returns it. A fresh install therefore ends up
with a complete, readable, commented-shaped config rather than nothing.

**Saves are atomic.** Serialise to pretty RON, write `<file>.ron.tmp`, then
rename. A reader never sees a half-written file.

**A parse failure does not clobber.** If the file exists but does not parse,
`load()` returns the default and leaves the file alone. The reasoning, from the
source: clobbering a config the user is halfway through hand-editing would be
worse than running with defaults for a moment.

**Omitted fields fall back.** Every section derives `#[serde(default)]`, so a
hand-written file containing three of ten fields parses fine and the rest take
their defaults.

---

## Watching for changes

```rust
use crownos_config::{subscribe_typed, schema::Input};

// Keep the returned Subscription alive — dropping it unregisters.
let _sub = subscribe_typed::<Input, _>(|input| {
    // Called on every external change to input.ron.
});
```

Three subscription APIs:

| Function | Delivers |
|---|---|
| `subscribe(section, cb)` | Raw file contents for one section |
| `subscribe_typed::<T, _>(cb)` | A parsed `T` for `T`'s section |
| `subscribe_key(key, cb)` | Only when one specific field changes |

There is **one** process-wide `notify` watcher, created lazily on the first
subscribe, non-recursive on the config directory. Create, modify and rename
events count; removals are ignored. If the watcher fails to start, subscriptions
become inert rather than fatal.

### Echo suppression

This is the subtle part. Consider an app with a brightness slider that saves on
every tick. Without care:

1. Slider moves → `save()` writes `display.ron`
2. inotify fires
3. The app's own subscription delivers "external change"
4. The app applies it, possibly moving the slider

The app fights itself. So `save()` records a hash of exactly what it wrote, and
the watcher drops any event whose content hashes to either the last-delivered
hash *or* the last hash `save()` recorded.

From the crate docs:

> Without this, an app that saves on every slider tick would immediately get its
> own write back as an external change and fight itself.

The same hash check drops half-written files: if a reader catches a file mid-write
its hash matches neither, but a re-read on the following event delivers the
complete content.

### Whole section or single key?

`crowndictator` deliberately subscribes to the whole `Input` section rather than
four separate keys. Its comment:

> …so the controller sees one consistent snapshot instead of four independent
> edits.

Use `subscribe_key` when you genuinely care about one field in isolation — a
widget bound to one toggle. Use `subscribe_typed` when several fields together
describe a state you must apply atomically.

---

## Typed keys

Sections are declared through a `section!` macro that generates, from one
declaration:

- the struct, with `Serialize`, `Deserialize` and `#[serde(default)]`
- a `SECTION: &'static str` constant
- a `Default` impl from the per-field `= value`
- a zero-sized unit **key type per field**
- a `<Name>Key` enum listing every field

The unit key types are what make `subscribe_key` type-safe — you pass
`DarkMode`, not the string `"dark_mode"`, so a typo is a compile error.

---

## Threading

Watcher callbacks are `Send + Sync` and run on the notify thread. They cannot
touch your UI state directly. Both existing consumers bridge:

- **`crownpositor`** posts to a calloop channel, because the callback cannot
  borrow `&mut State`.
- **`crowndictator`** turns config changes and hotkey edges into the same
  `Event` type on one channel — which is what makes "the user switched dictation
  off while holding the chord" an ordinary message sequence rather than a race.

If you are writing a xilem app, `crownos-config` has a `xilem` feature (on by
default) providing `watch`, `watched` and `watched_key` views that plumb changes
through xilem's message path instead of a background callback.

---

## Who actually follows this convention

Honestly: not everyone yet.

| Component | Behaviour |
|---|---|
| `crownpositor` | **Follows.** Subscribes to `compositor`, `appearance`, `display`. Rebinding takes effect without a restart. |
| `crowndictator` | **Follows.** Live-follows `input.ron`; the cleanest example in the codebase. |
| `crownbar` | **Ignores.** Reads no CrownOS config. Hardcodes bar height 40 against a schema default of 32. |
| `crownotify` | **Ignores.** `notifications.ron`, including Do-Not-Disturb, has no effect. |
| `crowndock` | **Deviates.** Uses `~/.config/crowndock/items.toml` — TOML, own directory. |
| `crownuikit` | **Ignores.** Reads nothing. |

And five sections have no reader at all: `sound`, `wifi`, `bluetooth`, `power`,
`keybinds`.

Bringing a component onto the convention is well-scoped, self-contained work.
Copy `crowndictator/src/settings.rs` — it is 40 lines and shows the whole
pattern.

---

## Adding a new section

1. Add `src/schema/<name>.rs` using the `section!` macro.
2. Declare it in `src/schema/mod.rs`.
3. Write a module doc that says **who the consumer is** — every existing section
   does, and it is the only place that contract is recorded.
4. Add a test proving the shape you documented actually parses. `compositor.rs`
   does this with a literal hand-written RON sample.

---

## See also

- [Config schema](../50-reference/config-schema.md) — every section and field
- [Environment variables](../50-reference/environment-variables.md)
- [crownos-config](../30-components/crownos-config.md)
