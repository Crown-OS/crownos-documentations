# lls-protocol

**Status: Broken** — does not compile · Rust · default branch `main` ·
[repo](https://github.com/Crown-OS/lls-protocol)

A low-latency audio/video streaming protocol, intended to carry screen mirroring
and camera sharing between a phone and the desktop.

Roughly 119 lines of real code across two commits. It is a skeleton.

---

## What "lls" stands for

**Not stated anywhere.** The string appears only as the crate name — no README,
no doc comment, no commit message expands it.

The most likely reading is **Low-Latency Streaming**, supported by two lines of
evidence:

- The module names are an A/V stack, not a general IPC bus: `streaming`,
  `signaling`, `nvenc` (NVIDIA hardware video encoder), `packet`, `discovery`,
  `util/reorder_buffer`.
- The website describes matching product features — "low-latency screen mirroring"
  and "turn your tablet into a responsive second monitor".

*(This is inference. Treat it as such.)*

The repository description says "Low Latency Protocol is a protocol that powers
crowncrate", but **no code in either repository references the other**. See
[relationship to crowncrate](#relationship-to-crowncrate) below.

---

## Design

### Transport

UDP. `server/src/protocol.rs`:

```rust
pub struct Connection {
    receiver_ip: IpAddr,
    socket: UdpSocket,
}
```

### Packet format

Custom binary, modelled directly on RTP (RFC 3550). Not serde, not JSON, not
bincode.

```rust
pub enum PacketType { Audio = 0, Video = 1 }

pub enum FrameRateMode { Low = 30, Standard = 60, Smooth = 90, SuperSmooth = 120 }

pub struct ConfigurationPacket {
    packet_type: PacketType,
    mode: FrameRateMode,
}

pub struct MediaPacket {
    pub packet_type: PacketType,
    /// Spot missing packets.
    pub sequence_number: u16,
    /// Sampling instant of the first octet.
    /// Audio sample rate 8 kHz, video 90 kHz.
    /// At 60 fps this advances 3000 per frame; at 120 fps, 1500.
    pub timestamp: u32,
    /// For video: marks the last packet of a frame, telling the decoder to render.
    pub marker: bool,
    /// 32-bit identifier for the origin of a media stream.
    pub ssrc: u32,
}
```

Those are the RTP header fields verbatim, including the standard 90 kHz video and
8 kHz audio clock rates.

### Jitter buffer

`server/src/util/reorder_buffer.rs` is the only part with real logic — a
`BTreeMap<u16, MediaPacket>` keyed on sequence number, with
`next_sequence_number: Option<u16>`. Only `insert` is implemented; there is no
drain or pop.

---

## Why it does not compile

Three problems, in the order you hit them.

**1. The root package has no targets.** The root `Cargo.toml` declares both a
`[workspace]` and a `[package]` named `lls-protocol`, but there is no `src/` at
the root. Cargo fails at manifest parse, before it does anything else:

```
$ cargo metadata --no-deps
error: failed to parse manifest at `.../lls-protocol/Cargo.toml`
Caused by:
  no targets specified in the manifest
  either src/lib.rs, src/main.rs, a [lib] section, or [[bin]] section
  must be present
```

The root `[package]` almost certainly should not be there at all — the two real
crates are `client` and `server`.

**2. The workspace dependency table does not exist.** `client/Cargo.toml` and
`server/Cargo.toml` both declare `serde = { workspace = true }`,
`tokio = { workspace = true }` and `tokio-stream = { workspace = true }`, but
those dependencies sit under `[dependencies]` for the root package rather than in
`[workspace.dependencies]`. Removing the root `[package]` without moving them
turns the first error into
`` error: `serde` was not found in `workspace.dependencies` ``.

**3. `UdpSocket::try_from(IpAddr)` does not exist.**

```rust
pub fn connect(&mut self, device: DiscoveredDevice) {
    let socket = UdpSocket::try_from(device.ip_address);   // no such impl
    self.receiver_ip = device.ip_address
}
```

The bound `socket` is also immediately dropped.

---

## Everything that is a stub

- `client/src/lib.rs` — **0 bytes**
- `server/src/config.rs` — **0 bytes**
- `server/src/signaling.rs` — **0 bytes**
- `server/src/streaming.rs` — **0 bytes**, and not declared in `lib.rs`
- `server/src/nvenc/mod.rs` — **0 bytes**
- `server/src/platform/mod.rs` — **0 bytes**
- `client/src/main.rs` and `server/src/main.rs` — both `println!("Hello, world!")`
- `MediaPacket::parse()` — empty, private, takes no arguments
- `DiscoveredDevice::discover()` — always returns an empty `Vec`
- `Connection::get_byte_stream()` — empty
- `serde` is declared and **no type derives it**
- `#[derive()]` on `PacketType` is empty

**The `client` crate does not depend on `server`**, and all shared types live
inside `server`. There is currently no shared protocol crate — which, for a
repository named "protocol", is the structural thing to fix after the build.

---

## Relationship to crowncrate

**None, in code.** Neither repository references the other, and they use entirely
different transports and encodings.

The apparent division of labour:

| | crowncrate | lls-protocol |
|---|---|---|
| Plane | Control | Media |
| Transport | TCP :5252 | UDP |
| Encoding | CBOR | RTP-shaped binary |
| Carries | Clipboard, notifications, calls, OTP | Screen and camera streams |

*(Inference — nothing states this.)*

---

## If you want to work on it

Suggested order:

1. Remove the root `[package]` — the workspace has no root crate — and move
   `serde`, `tokio` and `tokio-stream` into `[workspace.dependencies]`, which is
   what the members already reference.
2. Fix `Connection::connect` to bind a socket properly.
3. Move `packet`, `protocol` and `discovery` into a shared crate both `client`
   and `server` depend on.
4. Implement `MediaPacket::parse` and a matching serialiser.
5. Complete the reorder buffer — it needs a drain with a playout deadline.

---

## License

**No LICENSE file.** See
[Project status](../00-overview/project-status.md#licensing).
