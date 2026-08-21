# crownotify

**Status: Partial** (and **does not build** against current `crownshell`) · Rust ·
default branch `main` · [repo](https://github.com/Crown-OS/crownotify)

The CrownOS notification daemon. Implements the freedesktop notification
specification plus a CrownOS-specific interface for rich notification types.

It is the **only real D-Bus surface in the desktop**, and the **only repo with
integration tests**.

---

## Surface configuration

| | |
|---|---|
| Layer | `Top` |
| Anchor | TOP, RIGHT, BOTTOM |
| Width | 400 |
| Blur | `false` |
| Namespace | `crownotify` |

---

## Build and run

```bash
cd crownotify
cargo run

# Tests need a live session bus.
dbus-run-session -- cargo test -- --test-threads=1
```

> **It does not currently compile against `crownshell` HEAD.** `src/main.rs`
> calls `window.request_frame(compositor_state, qh)` with two arguments; current
> `crownshell` takes three. `crownotify` was last touched before `crownshell`'s
> text work landed. Since it uses `path = "../crownshell"`, and `crowndictator`
> needs the three-argument form, **the two cannot both build against the same
> checkout.** Fixing this is a good first task.

`crownshell` is a **path** dependency, so your local checkout is used — see
[Workspace setup](../10-getting-started/workspace-setup.md).

---

## D-Bus interfaces

Both on the **session** bus. Note: `main.rs` uses
`zbus::connection::Builder::session()` for both, despite the local variable being
named `system_conn`.

### Owned

| Name | Path | Methods |
|---|---|---|
| `org.freedesktop.Notifications` | `/org/freedesktop/Notifications` | `GetServerInformation`, `GetCapabilities`, `Notify`, `CloseNotification`, plus 3 declared signals |
| `io.crownos.crownotify` | `/io/crownos/crownotify` | `OpenNotificationCenter`, `CloseNotificationCenter`, `SendGeneralNotification`, `SendCallNotification`, `SendMusicNotification`, `SendChatNotification` |

Owning `org.freedesktop.Notifications` means any application that raises a
desktop notification works with CrownOS out of the box.

### Called

| Name | Path | Methods used |
|---|---|---|
| `io.crownos.crowncrate` | `/io/crownos/crowncrate` | `PickupCall`, `DeclineCall` |

Pressing pickup or decline on a call toast calls through to `crowncrate`.

> **`crowncrate-linux` has no D-Bus code at all** — no `zbus` dependency. The
> service `crownotify` calls does not exist. `crownotify` has a mock-based
> integration test for it; the real implementation is missing.

---

## Threading model

A `smol` thread hosts zbus. Incoming notifications land in an
`Arc<Mutex<VecDeque<Notification>>>` inbox, and a `calloop::ping::Ping` wakes the
`crownshell` event loop and requests a frame on every window.

This is precisely the pattern `crownshell`'s documentation recommends for
delivering a D-Bus signal or worker-thread message onto the render thread.

Both zbus connections are `std::mem::forget`ed to keep them alive for the process
lifetime.

---

## Source layout

| File | Role |
|---|---|
| `dbus/freedesktop.rs` | `SystemNotificationInterface` |
| `dbus/custom.rs` | `CustomNotificationInterface` |
| `models/` | `Notification` enum: `General`, `Call`, `Music`, `Chat`, `Audio`, `Display` |
| `models/call.rs` | The crowncrate service constants and pickup/decline calls |
| `notify_handler.rs` | The `SurfaceHandler` — drains the inbox, expires by `expire_timeout`, hit-tests |
| `ui/painter.rs` | 377 lines |
| `ui/text.rs` | Text layout |

---

## Tests

The only integration tests in the organization.

| File | Lines | Covers |
|---|---|---|
| `tests/dbus_custom.rs` | 56 | The `io.crownos.crownotify` interface |
| `tests/call_notification.rs` | 231 | Call notification flow against a mock crowncrate |

Both register real well-known names on the session bus and drive them through
`#[proxy]` clients. `call_notification.rs` holds a `OnceLock<Mutex<()>>` because
*"Tests share well-known D-Bus names on the session bus, so they must not run
concurrently."*

**Running them:**

```bash
dbus-run-session -- cargo test -- --test-threads=1
```

The internal lock is per-binary and there are two binaries, so `--test-threads=1`
is still needed. They will also conflict with a real `crownotify` already holding
the name.

`call_notification.rs` has an ignored test that runs against a real crowncrate:

```bash
cargo test -- --ignored real_crowncrate    # start crowncrate first
```

---

## Known limitations

**It ignores `notifications.ron` entirely.** The section exists in
`crownos-config` with `enabled`, `do_not_disturb` and `show_previews`, and
nothing reads it. `NotifyHandler::do_not_disturb` is a private field with a
`toggle_dnd()` that nothing calls.

**There is no notification centre.** `OpenNotificationCenter` and
`CloseNotificationCenter` only `log::info!`.

**`CloseNotification` does nothing** — returns `Ok(())` with no effect. The three
declared signals are never emitted.

**`GetCapabilities` over-advertises.** It claims `action-icons`, `actions`,
`body-images`, `sound` and `persistence`. None are implemented.

**Half the notification model is empty.** `models/audio.rs` and
`models/display.rs` are empty structs, though the `Notification` enum has
variants for them.

**Only `General` notifications expire.** Everything else lives forever —
`expire_timeout` is honoured for one variant.

**`Notify` discards `app_icon` and `hints`.**

---

## License

**No LICENSE file.** `Cargo.toml` declares
`authors = ["Rama Krishnan <ramakrishnan@gmail.com>"]`. See
[Project status](../00-overview/project-status.md#licensing).
