# Known limitations

What to expect if you run CrownOS components on your own machine. Written for
users rather than contributors — the developer-facing version is
[Project status](../00-overview/project-status.md).

---

## The big ones

**There is no installable CrownOS.** No ISO, no packages, no releases. Everything
is built from source. See [Install](install.md).

**There is no complete desktop.** You can run a compositor, a bar, a dock and a
dictation daemon, but there is no settings panel, no launcher, no greeter, no
notification centre, and no session to log into from a display manager.

**Nothing is stable.** Every component is pre-1.0 and `crownshell` says so
directly: *"The API will move. If you're going to use it, expect breaking
changes."*

---

## Things that look like they should work and do not

### Blur

`crownbar`, `crowndock` and `crownotify` all request background blur.
`crownpositor` never advertises the protocol that provides it, so **under CrownOS
there is no frosted-glass effect** — surfaces just render normally.

Under Hyprland or KWin it depends on their support for
`ext-background-effect-v1`.

### Clicking dock icons

**The dock cannot launch applications.** You can drag `.desktop` files onto it
and they pin and persist, but clicking one does nothing. It is currently a pin
manager with a nice animation.

### Most settings

Five configuration sections have no reader at all — `sound.ron`, `wifi.ron`,
`bluetooth.ron`, `power.ron`, `keybinds.ron`. Editing them has no effect.

Two more are ignored by the component that should read them:

- **`appearance.bar_height`** — `crownbar` hardcodes 40 px regardless.
- **`notifications.ron`**, including **Do Not Disturb** — `crownotify` ignores
  the whole section.

The settings that *do* work are `compositor.ron`, `display.ron`,
`appearance.ron` (partly) and `input.ron`.

### The workspace overview

Four-finger trackpad swipes are bound to opening and closing a workspace
overview. That feature is not implemented — the gestures are recognised and then
do nothing.

Three-finger swipes for switching workspace do work.

### Notification actions

`crownotify` advertises support for notification actions, action icons, body
images, sound and persistence. **None are implemented.** Notifications also
cannot be closed programmatically, and only general notifications expire — calls,
music and chat notifications stay until dismissed by hand.

### The phone bridge

Clipboard sync, notification sync, call bridging, file sharing, second screen and
OTP sync are all advertised. **None of them work.** The desktop daemon does not
compile, the Android app is an empty template with no network permission, and the
browser extension repository has no commits.

---

## Conflicts to know about

**`Super+Space` is bound twice.** The compositor uses it for `cycle-layout`;
`crowndictator` uses it for push-to-talk by default. Because `crowndictator`
reads the keyboard directly rather than going through the compositor, **both
fire**. Rebind one — change `dictation_hotkey` in `input.ron`, or the
`cycle-layout` binding in `compositor.ron`.

**`crownotify` will not start if another notification daemon is running.** Only
one process can own `org.freedesktop.Notifications`.

---

## Hardware and environment

**Fractional scaling is approximate.** Only integer buffer scales are supported.
On a 1.25× display the compositor advertises 2, so text renders at 2× and is
scaled down — sharp, but not pixel-exact.

**Text does not wrap.** `crownshell` lays out single lines only, with no wrapping
and no alignment. Long notification bodies will overflow rather than reflow.

**You need a working Vulkan or GL driver.** Everything renders through wgpu. In a
VM without guest GPU drivers, the shell components will fail at surface creation.

**Voice dictation needs an NVIDIA stack for the fast path**, and downloads
700 MB (CPU model) to 2.5 GB (GPU model) from Hugging Face on first use. It also
needs your user in the `input` group, which means the daemon can observe all
keyboard input — inherent to what it does.

**No touch, tablet, pinch or hold gestures.** The compositor handles keyboard,
pointer and trackpad swipes only. Display hotplug is not handled either.

**Logging is uneven.** `crowndock` never initialises its logger, so its warnings
are invisible no matter what you set `RUST_LOG` to.

---

## Security

Please read [SECURITY.md](../../SECURITY.md) before running anything on an
untrusted network. In brief:

- `crowncrate-linux` accepts **unauthenticated, unencrypted** commands on TCP
  port 5252, including remote shutdown. It does not compile today, so this is not
  currently exploitable — but do not run a fixed build on a shared network until
  pairing exists.
- The ISO profile carries upstream Arch's live-medium defaults: empty root
  password, root autologin, permissive sshd. Appropriate for a rescue image, not
  for an installed system.
- `crownpositor` lets **any** Wayland client bind privileged protocols, not just
  shell components.

---

## Licensing

Thirteen of fifteen repositories have **no LICENSE file**, which legally makes
them all-rights-reserved despite being presented as open source. Only
`crownshell` and `crownos-documentations` carry MIT licences, and they name
different copyright holders.

If you plan to redistribute or build on this, that needs resolving first.

---

## Reporting something not on this list

Open an issue on the relevant repository with the commit hash, minimal
reproduction steps, and your system info — GPU, compositor, kernel. For anything
visual, say which compositor you were running under: a bug under `crownpositor`
and the same symptom under Sway are usually different bugs.

Security issues go through [SECURITY.md](../../SECURITY.md), not public issues.
