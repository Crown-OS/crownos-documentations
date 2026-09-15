# crowncrate-chrome

**Status: Empty** — no commits, and not listed in the organization ·
default branch not yet established ·
[repo](https://github.com/Crown-OS/crowncrate-chrome)

The planned browser companion for the CrownOS phone bridge.

---

## Current state

It is a **completely empty bare repository**. Zero commits, zero objects —
`objects/pack` and `objects/info` are both empty directories — and `refs/heads`
and `refs/tags` carry nothing. `rev-list --all --count` returns `0`. Cloning it
produces an empty working tree, and `git worktree add` fails because there is no
commit to check out.

It also **does not appear in the Crown-OS repository listing** on GitHub. The 16
repositories the organization lists do not include it; this page exists because
the repo is referenced elsewhere, not because there is anything in it.

There is no code, no manifest, no README, no branch.

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
   [`crowncrate-linux`](crowncrate-linux.md) does not compile on its default
   branch — the fixes for that exist only as unpushed local changes — and even
   with them it implements nothing, with no discovery and no pairing;
   [`crowncrate-android`](crowncrate-android.md) is an untouched template with no
   network permission. A browser extension has nothing to connect to yet.
2. **A browser extension cannot open a raw TCP socket.** The desktop daemon's
   protocol is TCP plus CBOR on port 5252, which the extension APIs do not
   expose. This client would need either a WebSocket endpoint on the daemon, a
   native messaging host, or a local HTTP interface — a protocol decision that
   does not exist yet.

If you want to work on the ecosystem layer, `crowncrate-linux`'s missing
`io.crownos.crowncrate` D-Bus interface is the useful starting point. Its compile
errors are the other obvious target, but a fix for those is already written and
unpushed — ask a maintainer before duplicating it.

---

## See also

- [IPC and protocols](../20-architecture/ipc-and-protocols.md#4-crowncrate--the-phone-bridge)
- [crowncrate-linux](crowncrate-linux.md)
- [crowncrate-android](crowncrate-android.md)
