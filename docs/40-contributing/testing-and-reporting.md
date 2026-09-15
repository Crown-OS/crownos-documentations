# Testing and reporting

For people who want to help by **running** CrownOS rather than writing it.

This is genuinely useful work. There is no CI, no test hardware, and no user
base — every component has been exercised on a handful of machines at most, and
the failure modes that matter most (GPU drivers, seat management, D-Bus
conflicts) are exactly the ones that differ from machine to machine.

Written for testers. If you are changing code, see
[Testing](testing.md) for the test suites and
[Your first change](../10-getting-started/your-first-change.md) for the workflow.

---

## What is worth testing

In rough order of how much a report would help:

| Target | Why |
|---|---|
| `crownpositor` nested, on your GPU | Surface creation is the single most environment-dependent thing in the project. wgpu needs a real Vulkan or GL adapter, and "which adapter, on which driver" has barely been sampled. |
| `crownpositor` on real hardware (`CROWN_BACKEND=kms`) | Needs `seatd` or logind, DRM/KMS, libinput. Untested on most hardware. Have a second TTY or an SSH session open. |
| `crownbar` under a **non-CrownOS** compositor | It is a plain layer-shell client. Hyprland, Sway, river and KWin all qualify, and each implements the protocol slightly differently. |
| `crowndictator` end to end | ALSA capture, evdev hotkey, and one of three text-injection helpers. Many ways to fail per machine. |
| `crownotify` against real applications | It claims `org.freedesktop.Notifications`. Whether real senders are happy with its replies is not something the tests cover. |

Skip `crownlauncher`, `crowncrate-linux`, `lls-protocol` and `crowncrate-chrome`
— they are scaffolding, and there is nothing to observe.

---

## Running it

Everything you need is in
[Try it without installing](../60-users/try-it-without-installing.md): the
nested compositor session, how to find its Wayland socket, and how to attach
clients to it. Start there.

Two things to do before you begin, so your report is reproducible:

```bash
export CROWN_CONFIG_DIR=/tmp/crownos-test   # do not test against your real settings
git -C ~/src/crownos/crownpositor rev-parse --short HEAD
```

The commit hash is the first field the bug template asks for, and it is the one
people most often leave out. "Latest main" is not an answer — there is no CI, so
there is no build anyone can correlate against a date.

---

## Getting logs

Logging is inconsistent across components, and knowing which is which saves you
reporting "no output" as if it were a symptom.

| Component | Logger | Goes to | `RUST_LOG` syntax | Default level |
|---|---|---|---|---|
| `crownpositor` | `tracing-subscriber` `fmt` | stderr | tracing env-filter | `fmt()` default |
| `crownbar` | `env_logger` | stderr | `env_logger` | `info` |
| `crownotify` | `env_logger` | stderr | `env_logger` | `info` |
| `crowndictator` | `env_logger` | stderr | `env_logger` | `info` |
| `crowndock` | **none** | nowhere | — | — |

### crownpositor uses tracing, not env_logger

The syntax is different, and this trips people up. `tracing`'s `EnvFilter` wants
**target-and-level pairs**, so you can silence `smithay`'s firehose while turning
the compositor itself up:

```bash
RUST_LOG=crownpositor=debug,smithay=warn CROWN_BACKEND=winit cargo run
```

A bare `RUST_LOG=debug` works too, but under a Smithay backend it produces
thousands of lines a second and is close to useless. Filter first.

`crownpositor` declares `tracing-journald` in its manifest but **never installs
that layer**, so nothing reaches the journal — `journalctl` will show you
nothing. Read the terminal you launched it from, and redirect if you want to
keep it:

```bash
RUST_LOG=crownpositor=debug CROWN_BACKEND=winit cargo run 2>&1 | tee /tmp/crownpositor.log
```

### The others use env_logger

Plain level, or `crate=level`:

```bash
RUST_LOG=debug cargo run
RUST_LOG=crownbar=debug cargo run
```

### crowndock produces nothing at all

It never calls `env_logger::init()`, so its `log::warn!` calls go nowhere no
matter what you set. If you are testing `crowndock`, say so in the report and do
not spend time hunting for a log file — there isn't one. Adding the init is a
one-line patch and a good first change.

---

## Filing a good report

Use the bug template — it is staged in
[`templates/.github/ISSUE_TEMPLATE/`](../../templates/.github/ISSUE_TEMPLATE)
and asks for exactly the right things. Note that it has **not been copied into
the individual repositories yet**, so on most repos you will get a blank issue
box; fill in the same fields by hand.

File against the repository of the component that misbehaved, not against
`crownos-documentations`.

What to include, and why each one earns its place:

- **The commit hash** — `git rev-parse --short HEAD` in the affected repo. There
  are no releases and no CI builds to refer to instead.
- **Which compositor you were under.** The same symptom under `crownpositor` and
  under Sway is usually a different bug. Say which, and say whether
  `crownpositor` was nested (`CROWN_BACKEND=winit`) or on hardware
  (`CROWN_BACKEND=kms`).
- **GPU and driver.** For anything that fails at or near surface creation this
  is the report. `vulkaninfo --summary` and your mesa version.
- **Distribution and kernel**, and `rustc --version` — which should say 1.88.0,
  because every repo pins it exactly. If it says something else, that is itself
  worth mentioning.
- **The exact commands you ran**, including the environment variables. `RUST_LOG`
  and `CROWN_BACKEND` change behaviour enough to matter.
- **Logs**, filtered, from the table above.
- **Any `~/.config/crownos/*.ron` involved.** Redact anything personal.
- **Whether the `[patch.crates-io]` overlay was in place.** Almost every build
  failure reported without this turns out to be a missing overlay.

### Check first that it is not already known

A large amount of CrownOS does not work *by design, and knowingly*:

- [Known limitations](../60-users/known-limitations.md) — what is absent
- [Troubleshooting](../60-users/troubleshooting.md) — symptoms with known causes
- [Project status](../00-overview/project-status.md) — component by component

Blur not rendering, the dock not launching applications, the workspace overview
doing nothing, and five config sections having no reader are all documented
gaps, not bugs. A report that says "I read the limitations page and this is not
one of them" is much easier to act on.

### The failure that is not a bug and looks like one

A mistyped keybind chord reverts an **entire config section** to defaults, with
no error and nothing logged. If a component suddenly ignores every setting in
one file, suspect a typo in a `keys` field before you suspect the component. See
[Troubleshooting](../60-users/troubleshooting.md#a-config-change-silently-did-nothing).

---

## Reporting something that works

Also useful, and nobody does it. "`crownpositor` runs nested on Intel Arc /
mesa 25.0 / kernel 6.14" is information the project does not otherwise have —
there is no test matrix and no CI to build one. A short note on an issue or a
discussion is enough.

---

## See also

- [Try it without installing](../60-users/try-it-without-installing.md) — how to run it
- [Troubleshooting](../60-users/troubleshooting.md) — before you file
- [Testing](testing.md) — the automated test suites
- [CI](ci.md) — why none of this is checked automatically yet
