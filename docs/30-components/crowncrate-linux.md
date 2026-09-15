# crowncrate-linux

**Status: Skeleton** — does not compile on the default branch; compiles with
local, unpushed fixes, and then does almost nothing · Rust · default branch
`main` · [repo](https://github.com/Crown-OS/crowncrate-linux)

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

## What is wrong, and what has been fixed locally

`cargo check` on the default branch reports two errors, with two more defects
behind them.

> **Read this before starting work here.** All four fixes below, and the glib
> removal, exist only as **uncommitted changes in a working tree** — `Cargo.toml`,
> `Cargo.lock`, `src/actions/action.rs`, `src/actions/action_manager.rs`,
> `src/communication/message.rs` and `src/communication/server.rs` are all
> modified and unpushed. Clone `main` today and you get the original two compile
> errors and the glib conflict. With the fixes applied, the pinned 1.88.0
> toolchain and the workspace overlay, the crate passes
> `cargo check --all-targets`.

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

The **glib conflict** is addressed in the same unpushed set. `Cargo.toml` on the
default branch declares `glib = "0.17"` while `gtk4 = "0.7"` requires 0.18, and
the lockfile carries both. Nothing in `src/` references `glib`, so the direct
dependency was removed outright.

Compiling is not the same as working: `src/lib.rs` and `src/ui/mod.rs` are empty,
`src/predule.rs` is unreachable from `main.rs`, and the whole crate is 346 lines.
**It is not published.** Nothing in the Crown-OS organization is on crates.io
except `crownshell` 0.1.0 and 0.2.0 — there is no placeholder release holding
this name.

---

## Prerequisites

GTK4 4.10 or newer plus the glib/pango/gdk-pixbuf/graphene development packages.
See [Prerequisites](../10-getting-started/prerequisites.md#crowncrate-linux).

Formatting note: this was the only repository in the organization with a
`rustfmt.toml`, and it was a liability rather than an asset — an 80-key dump of
defaults containing `edition = "2015"` (the crate is edition 2024),
`required_version = "1.5.1"` (rustfmt refuses to run on a mismatch), the
deprecated `fn_args_layout`, and a number of nightly-only keys with
`unstable_features = false`. **It has been deleted**, so the crate now formats on
rustfmt defaults like every other repo, and all eleven Rust repos pass
`cargo fmt --all --check` on rustfmt 1.88. That deletion is part of the same
unpushed change set — the file is still tracked on the default branch. Do not
reinstate it, and do not copy it into a new repo.

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

On the default branch the "it doesn't build" mitigation still holds, and the
crate is not published anywhere, so nobody is running it by accident. Once the
unpushed compile fixes land, that mitigation is gone and the listener **is**
reachable by anyone who starts it. Fix the authentication before merging the
build fix, not after. See [SECURITY.md](../../SECURITY.md).

---

## Other known gaps

- `src/ui/mod.rs`, `src/lib.rs` and `src/predule.rs` are **zero bytes**. `gtk4`
  and `glib` are declared but there is no UI. (`predule.rs` copies the
  misspelling `crownshell` originally shipped. `crownshell` has since corrected
  its own to `prelude`; this one is still spelled wrong, and since the file is
  empty and unreachable, deleting it is simpler than renaming it.)
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
| `crowncrate-linux` | Server | Does not compile on `main`; compiles with unpushed fixes |
| [`crowncrate-android`](crowncrate-android.md) | Client | Android Studio template; no network permission |
| [`crowncrate-chrome`](crowncrate-chrome.md) | Browser client | Empty bare repo — zero commits, zero objects |

**The two halves have never communicated.**

`lls-protocol` is unrelated in code — neither references the other. The apparent
division of labour is that `crowncrate` is the low-bandwidth control plane and
`lls-protocol` the media plane for screen and camera streaming, but nothing
states this.

---

## License

**No LICENSE file.** See
[Project status](../00-overview/project-status.md#licensing).
