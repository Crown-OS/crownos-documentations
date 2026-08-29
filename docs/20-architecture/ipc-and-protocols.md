# IPC and protocols

CrownOS has four separate communication mechanisms. They are not alternatives to
each other — each covers a different boundary.

| Mechanism | Boundary | Status |
|---|---|---|
| Config files + inotify | Between desktop components | **Working** |
| Wayland (`wlr-layer-shell`, `xdg-shell`) | Shell and apps ↔ compositor | **Working** |
| D-Bus (session) | Notifications, and desktop ↔ phone-bridge daemon | **Partly working** |
| TCP + CBOR / UDP + RTP | Desktop ↔ phone | **Not working** |

---

## 1. Configuration files

The primary coordination mechanism between desktop components. Covered in full
in [Config as IPC](config-as-ipc.md).

Summary: RON files in `~/.config/crownos/`, one per section, watched with a
single process-wide inotify watcher, with hash-based echo suppression so a writer
does not receive its own write back as an external change.

---

## 2. Wayland

`crownpositor` is the Wayland server. Everything else in the session is a client.

**Protocols the compositor implements** — one delegate file each under
`compositor/src/handlers/`: `xdg_shell`, `layer_shell`, `seat`, `selection`,
`session_lock`, `dmabuf`, `shm`, `fractional_scale`, `xdg_decoration`, the idle
protocols, and XWayland for X11 clients.

**Known gaps in the Wayland surface:**

- `ext-background-effect-v1` — handler entirely commented out; blur does not work
- `shm_formats` is an empty `Vec`; the dmabuf global is never created
- `privileged_client_filter` returns `true` for every client, so any client can
  bind privileged protocols
- Fractional scale sends the wrong scale
- Client-side decorations are not honoured
- Popups are not unconstrained
- Session lock confirms before all outputs are covered

---

## 3. D-Bus

The only D-Bus in the desktop is `crownotify`. Everything is on the **session**
bus.

### Names crownotify owns

| Name | Path | Interface |
|---|---|---|
| `org.freedesktop.Notifications` | `/org/freedesktop/Notifications` | `org.freedesktop.Notifications` |
| `io.crownos.crownotify` | `/io/crownos/crownotify` | `io.crownos.crownotify` |

**`org.freedesktop.Notifications`** is the standard freedesktop interface, so any
application that can raise a desktop notification works with CrownOS. Implemented:
`GetServerInformation`, `GetCapabilities`, `Notify`, `CloseNotification`, plus
three declared signals.

Caveats: `CloseNotification` returns `Ok(())` and does nothing. The signals are
never emitted. `GetCapabilities` advertises `action-icons`, `actions`,
`body-images`, `sound` and `persistence`, none of which are implemented. `Notify`
discards `app_icon` and `hints`.

**`io.crownos.crownotify`** is the CrownOS-specific interface, intended for a
settings panel and for rich notification types:
`OpenNotificationCenter`, `CloseNotificationCenter`, `SendGeneralNotification`,
`SendCallNotification`, `SendMusicNotification`, `SendChatNotification`.

The two notification-centre methods only log — there is no notification centre.

### The name crownotify calls

| Name | Path | Interface | Methods used |
|---|---|---|---|
| `io.crownos.crowncrate` | `/io/crownos/crowncrate` | `io.crownos.crowncrate` | `PickupCall`, `DeclineCall` |

When a call notification arrives from your phone, pressing pickup or decline in
the toast calls through to `crowncrate`.

> **This contract is one-sided.** `crownotify` implements the calling side and
> has an integration test with a mock. `crowncrate-linux` has **no zbus
> dependency and no D-Bus code at all** — the service it is supposed to expose
> does not exist. Implementing it is well-scoped work.

### Threading

`crownotify` runs zbus on a `smol` thread. Notifications land in an
`Arc<Mutex<VecDeque<Notification>>>` inbox, and a `calloop::ping::Ping` wakes the
`crownshell` event loop and requests a frame. This is the pattern `crownshell`'s
docs describe for getting a D-Bus signal onto the render thread.

---

## 4. crowncrate — the phone bridge

The control plane between desktop and phone. Low-bandwidth, persistent
connection.

### Transport

TCP, listening on **`0.0.0.0:5252`**, one thread per connected device, held open
for the life of the connection — the module README describes it as "similar to
websockets".

UDP **:5253** is bound for LAN discovery, but the discovery function is a stub
that does nothing and is never called.

### Wire format

**CBOR**, streamed:

```rust
serde_cbor::Deserializer::from_reader(reader).into_iter::<Message>()
```

> The module's own README says "one line json". That is out of date — the code
> uses CBOR, and `serde_json` is a declared but unused dependency.

### Message shape

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
added later.

### What is actually handled

| Action | Implementation |
|---|---|
| `SHUTDOWN` | `sh -c "shutdown now"` |
| `VOLUME` | `pactl set-sink-volume 0 {value}%` |
| `CLIPBOARD` | prints the `"type"` field and nothing else |
| everything else | not subscribed |

### Security

**There is no pairing, no authentication, and no encryption.** Any peer that can
reach port 5252 can shut the machine down. The crate does not currently compile,
so this is not exploitable as shipped — but it must be addressed before it is.
See [SECURITY.md](../../SECURITY.md).

### The Android side

`crowncrate-android` is an unmodified Android Studio Compose template. Its
manifest has **zero `<uses-permission>` entries** — no `INTERNET` — so it cannot
open a socket at all. There is no networking code.

The two halves have never communicated.

---

## 5. lls-protocol — the media plane

Separate from `crowncrate`, and unrelated to it in code: neither repository
references the other.

"lls" is not expanded anywhere in the source. Based on the module names
(`streaming`, `signaling`, `nvenc`, `packet`, `discovery`, `reorder_buffer`) and
the product features it lines up with, it is almost certainly **Low-Latency
Streaming** — the transport behind Second Screen, Remote Phone Access and Camera
Share. *(This is inference, not something the code states.)*

### Design

UDP, with an RTP-shaped packet header:

```rust
pub struct MediaPacket {
    pub packet_type: PacketType,   // Audio = 0, Video = 1
    pub sequence_number: u16,      // spot missing packets
    pub timestamp: u32,            // audio 8 kHz, video 90 kHz sample clock
    pub marker: bool,              // last packet of a video frame
    pub ssrc: u32,                 // stream origin identifier
}
```

Those are the RFC 3550 RTP header fields verbatim, including the 90 kHz video and
8 kHz audio clock rates. At 60 fps the timestamp advances 3000 per frame; at
120 fps, 1500.

A `ConfigurationPacket` carries a `FrameRateMode` — `Low` 30, `Standard` 60,
`Smooth` 90, `SuperSmooth` 120.

`util/reorder_buffer.rs` is a `BTreeMap<u16, MediaPacket>` jitter buffer. It is
the only part of the crate with real logic, and only `insert` is implemented.

### Status

**Skeleton.** It compiles now, but does nothing: `MediaPacket::parse` is an empty private
function. `Connection::connect` calls `UdpSocket::try_from(IpAddr)`, which does
not exist. `DiscoveredDevice::discover` always returns an empty `Vec`.
`client/src/lib.rs` and five server modules are zero bytes. `serde` is declared
and never used.

`streaming.rs` exists on disk but is not declared in `lib.rs`.

---

## 6. Deliberate bypasses

Two places where a component skips the compositor on purpose. Both are in
`crowndictator`, and both are so it works under any compositor rather than only
`crownpositor`.

**Global hotkey.** Reads `/dev/input/event*` directly with evdev rather than
asking the compositor for a global shortcut. Requires `input` group membership.
Modifier match is exact; `/dev/input` is rescanned every 4 seconds for hotplug.

**Text injection.** Shells out through a fallback chain — `wtype`, then
`ydotool`, then `wl-copy` plus `notify-send` — rather than using a Wayland input
method.

The trade-off is explicit: portability and simplicity, at the cost of needing
elevated device access and external binaries.

---

## See also

- [Config as IPC](config-as-ipc.md)
- [The layer-shell stack](layer-shell-stack.md)
- [crownotify](../30-components/crownotify.md) ·
  [crowncrate-linux](../30-components/crowncrate-linux.md) ·
  [lls-protocol](../30-components/lls-protocol.md)
