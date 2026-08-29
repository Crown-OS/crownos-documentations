# crowncrate-chrome

**Status: Empty** — no commits · default branch not yet established ·
[repo](https://github.com/Crown-OS/crowncrate-chrome)

The planned browser companion for the CrownOS phone bridge.

---

## Current state

The repository exists and has **zero commits**. `refs/heads` and `refs/tags` are
both empty. Cloning it produces an empty working tree, and `git worktree add`
fails because there is no commit to check out.

There is no code, no manifest, no README.

---

## What it is meant to be

The GitHub repository description is the only statement of intent:

> A Chrome extension that syncs the OTPs from mobile and more

That lines up with the `OTPSYNC` action in the desktop daemon's protocol:

```rust
pub enum Actions {
    CLIPBOARD, MEDIA, OPEN, OTPSYNC, MONITOR, VOLUME, SHUTDOWN,
}
```

So the intended flow is presumably: phone receives a one-time passcode →
`crowncrate-android` forwards it → `crowncrate-linux` receives it → the browser
extension offers to fill it.

`CLIPBOARD` and `OPEN` would be the other natural fits for a browser client.

---

## Before starting work here

Two things are worth knowing:

1. **Neither of the other two halves works.**
   [`crowncrate-linux`](crowncrate-linux.md) compiles but implements nothing, and has no
   discovery or pairing; [`crowncrate-android`](crowncrate-android.md) is an
   untouched template with no network permission. A browser extension has nothing
   to connect to yet.
2. **A browser extension cannot open a raw TCP socket.** The desktop daemon's
   protocol is TCP plus CBOR on port 5252, which the extension APIs do not
   expose. This client would need either a WebSocket endpoint on the daemon, a
   native messaging host, or a local HTTP interface — a protocol decision that
   does not exist yet.

If you want to work on the ecosystem layer, `crowncrate-linux`'s compile errors
and its missing `io.crownos.crowncrate` D-Bus interface are the useful starting
points.

---

## See also

- [IPC and protocols](../20-architecture/ipc-and-protocols.md#4-crowncrate--the-phone-bridge)
- [crowncrate-linux](crowncrate-linux.md)
- [crowncrate-android](crowncrate-android.md)
