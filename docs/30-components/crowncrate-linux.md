# crowncrate-linux

**Status: Skeleton** — compiles, but does almost nothing · Rust · default branch `main` ·
[repo](https://github.com/Crown-OS/crowncrate-linux)

The desktop side of the CrownOS phone bridge. A daemon that holds a persistent
connection to a paired phone and carries clipboard, media, notification, call and
OTP traffic.

Comparable to KDE Connect or Windows Phone Link.

---

## Design

From the module README (`src/communication/README.md`):

> This module handles the communication between devices using a custom
> lightweight TCP powered protocol that keeps the connection alive till the
> device gets disconnected similar to websockets. Each device is handled by a
> thread.

### Transport

TCP, `0.0.0.0:5252`, one `thread::spawn` per connection. UDP `:5253` is bound for
LAN discovery, but `discover()` is a stub that does nothing and is never called.

### Wire format

**CBOR**, streamed:

```rust
serde_cbor::Deserializer::from_reader(reader).into_iter::<Message>()
```

> The module README says "one line json". That is out of date — the code uses
> CBOR, and `serde_json` is a declared but unused dependency.

### Messages

```rust
pub struct Message {
    pub client: Ipv4Addr,
    pub method: Actions,
    pub body: HashMap<String, String>,
}

#[repr(u8)]
pub enum Actions {
    CLIPBOARD, MEDIA, OPEN, OTPSYNC, MONITOR, VOLUME, SHUTDOWN,
}
```

The README lists five methods; the enum has seven. `VOLUME` and `SHUTDOWN` were
added in a later commit.

### Action dispatch

An `Action` trait with `fn handle_message(&self, message: Message)`, and an
`ActionManager` holding `Arc<Mutex<HashMap<Actions, Box<dyn Action>>>>`.
`main.rs` subscribes three:

| Action | Implementation |
|---|---|
| `SHUTDOWN` | `sh -c "shutdown now"` |
| `VOLUME` | `pactl set-sink-volume 0 {value}%` |
| `CLIPBOARD` | prints the `"type"` field and nothing more |

---

## What was wrong, and what was fixed

`cargo check` used to report two errors, with two more defects behind them. All
four are fixed and the crate compiles.

1. **`Box<dyn Action>` could not cross a thread boundary.**
   `communication/server.rs::listen` handed the `ActionManager` to a
   `thread::spawn` closure and the trait object had no `Send` bound. `Action` is
   now `: Send + Sync`.
2. **`&mut self` escaped into a `'static` closure.** `ActionManager` derives
   `Clone` — its `actions` field was already `Arc<Mutex<..>>`, so a clone shares
   one table — and `listen` moves a handle in rather than borrowing `self`.
3. **`notify` iterated an `Arc<Mutex<HashMap<..>>>` directly** and had an empty
   loop body. It locks, iterates `.values()` and calls `handle_message`.
4. **`unsubscribe` was inverted.** `retain(|&i, _| i == action)` kept only the
   entry it was asked to remove; it is now `i != action`.

The **glib conflict is gone too.** `Cargo.toml` declared `glib = "0.17"` while
`gtk4 = "0.7"` requires 0.18, and the lockfile carried both. Nothing in `src/`
referenced `glib`, so the direct dependency was removed outright.

Compiling is not the same as working: `src/lib.rs` and `src/ui/mod.rs` are empty,
`src/predule.rs` is unreachable from `main.rs`, and the whole crate is 333 lines.
It is published as a **0.0.0 placeholder** to hold the name.

---

## Prerequisites

GTK4 4.10 or newer plus the glib/pango/gdk-pixbuf/graphene development packages.
See [Prerequisites](../10-getting-started/prerequisites.md#crowncrate-linux).

Formatting note: this is the only repository with a `rustfmt.toml`, and it is a
liability rather than an asset — an 80-key dump of defaults containing
`edition = "2015"` (the crate is edition 2024), `required_version = "1.5.1"`
(rustfmt refuses to run on a mismatch), the deprecated `fn_args_layout`, and a
number of nightly-only keys with `unstable_features = false`. **Do not copy it.**
Deleting it and using rustfmt defaults, as every other repo does, is a reasonable
patch.

---

## The D-Bus contract it does not implement

`crownotify` calls into this daemon for call handling:

```rust
pub const CROWNCRATE_SERVICE:   &str = "io.crownos.crowncrate";
pub const CROWNCRATE_PATH:      &str = "/io/crownos/crowncrate";
pub const CROWNCRATE_INTERFACE: &str = "io.crownos.crowncrate";
// methods: PickupCall, DeclineCall
```

`crowncrate-linux` has **no `zbus` dependency and no D-Bus code at all**. The
service does not exist. `crownotify` has a mock-based integration test for it,
plus an ignored test that runs against a real daemon:

```bash
cargo test -- --ignored real_crowncrate
```

Implementing this interface is well-scoped work.

---

## Security

**There is no pairing, no authentication, and no encryption.** Any peer that can
reach port 5252 can send a `SHUTDOWN` message and power the machine off.

The crate now compiles, so this **is** reachable if anyone runs it. Nothing
starts it automatically and it is published only as a 0.0.0 placeholder, but the
"it doesn't build" mitigation is gone. See [SECURITY.md](../../SECURITY.md).

---

## Other known gaps

- `src/ui/mod.rs`, `src/lib.rs` and `src/predule.rs` are **zero bytes**. `gtk4`
  and `glib` are declared but there is no UI. (`predule.rs` mirrors the same
  typo as `crownshell`'s prelude module.)
- `src/discovery/mod.rs` is 13 lines: bind a socket, return.
- `src/logging.rs` mixes `fn log(&mut self, ..)` with receiver-less
  `warn`/`debug`/`error` in one trait; `FileLogger` is imported in `main.rs` and
  unused; `writeln!` results are ignored.
- `src/config/mod.rs` is one line:
  `pub const DEFAULT_LOGGING_FILE_PATH: &str = "/var/log/crowncrate.log";`
- No tests.

---

## Relationship to the other crowncrate repos

| Repo | Role | State |
|---|---|---|
| `crowncrate-linux` | Server | Does not compile |
| [`crowncrate-android`](crowncrate-android.md) | Client | Android Studio template; no network permission |
| [`crowncrate-chrome`](crowncrate-chrome.md) | Browser client | No commits |

**The two halves have never communicated.**

`lls-protocol` is unrelated in code — neither references the other. The apparent
division of labour is that `crowncrate` is the low-bandwidth control plane and
`lls-protocol` the media plane for screen and camera streaming, but nothing
states this.

---

## License

**No LICENSE file.** See
[Project status](../00-overview/project-status.md#licensing).
