# Security Policy

## Supported versions

CrownOS has not made a release. There are no versioned artifacts, no published
packages, and no ISO downloads. Everything is built from source at a commit.

Security reports are accepted against the **current default branch** of any
repository in the [Crown-OS](https://github.com/Crown-OS) organization.

## Reporting a vulnerability

**Do not open a public issue for a security problem.**

Report it through
[GitHub private security advisories](https://docs.github.com/en/code-security/security-advisories/guidance-on-reporting-and-writing/privately-reporting-a-security-vulnerability)
on the affected repository, or contact a maintainer listed in
[CONTRIBUTING.md](CONTRIBUTING.md) directly.

Please include:

- The repository and commit hash
- What the issue is and what an attacker can do with it
- Reproduction steps, ideally minimal
- Anything you know about scope — which components are affected

We do not currently operate a bug bounty, and there is no formal response SLA.
This is a small project; you will get a human reply rather than an automated one.

## Known security posture

These are not vulnerabilities to report — they are documented, known properties
of the current code. They are listed here so you can assess risk before running
CrownOS, and because fixing them is welcome work.

### crowncrate-linux — no authentication or encryption

`crowncrate-linux` opens a TCP listener on **`0.0.0.0:5252`** and processes CBOR
messages from any peer that connects. There is **no pairing, no authentication,
and no transport encryption**. Handled actions include:

- `SHUTDOWN` — shells out to `shutdown now`
- `VOLUME` — shells out to `pactl set-sink-volume`
- `CLIPBOARD`, `MEDIA`, `OPEN`, `OTPSYNC`, `MONITOR`

Anyone on the same network can shut the machine down.

**This changed in August 2026.** The crate previously did not compile, and that
was the only thing preventing exploitation. It compiles now. Nothing starts it
automatically and it is published only as a `0.0.0` placeholder, so the exposure
is limited to anyone who deliberately runs it — but the "it doesn't build"
mitigation is gone. Do not run it on an untrusted network. See
[crowncrate-linux](docs/30-components/crowncrate-linux.md).

### crownos-iso — live-medium defaults

`crownos-iso` is currently an unmodified copy of the upstream Arch Linux
`archiso` `releng` profile. It carries that profile's live-medium defaults, which
are deliberate for a rescue image and inappropriate for an installed system:

- Empty root password (`airootfs/etc/shadow`)
- Root autologin on tty1
- `sshd` with `PermitRootLogin yes` and `PasswordAuthentication yes`

These are upstream Arch's choices for a live ISO, not CrownOS decisions. They
would need to change before the profile becomes an installable desktop image.

### crownpositor — privileged protocol access is unrestricted

The compositor's `privileged_client_filter` currently returns `true` for every
client, so any Wayland client can bind privileged protocols such as layer-shell
and session-lock, not just shell components launched by the compositor. Tracked
in the source as a TODO.

### crowndictator — device and network access

`crowndictator` reads `/dev/input/event*` directly to detect its global hotkey,
which requires membership in the `input` group. That grants the process the
ability to observe all keyboard input, by design — it is a dictation daemon. It
also downloads model weights from Hugging Face on first run.

## Disclosure

We will credit reporters in the fix commit unless you ask us not to. Once a fix
is on the default branch, the advisory can be made public.
