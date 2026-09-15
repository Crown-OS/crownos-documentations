# crownos-iso

**Status: Skeleton** · Shell / archiso · default branch `main` ·
[repo](https://github.com/Crown-OS/crownos-iso)

The archiso profile that is intended to build the CrownOS installation image.

> **It is currently an unmodified copy of the upstream Arch Linux `releng`
> profile.** One commit, `Created the Boilerplate`. There is no CrownOS branding,
> no CrownOS package repository, and no CrownOS package in the package list. What
> it builds is a generic Arch rescue image.

---

## What is actually in it

`profiledef.sh`, verbatim:

```sh
iso_name="archlinux"
iso_label="ARCH_$(date --date="@${SOURCE_DATE_EPOCH:-$(date +%s)}" +%Y%m)"
iso_publisher="Arch Linux <https://archlinux.org>"
iso_application="Arch Linux Live/Rescue DVD"
install_dir="arch"
```

`airootfs/etc/hostname` is `archiso`. `airootfs/etc/motd` points at the Arch
install guide. `pacman.conf` enables only `[core]` and `[extra]` — there is no
CrownOS repository configured.

`packages.x86_64` is upstream's 128-package rescue set: filesystem tools, network
tools, firmware, VM guest agents, `archinstall`. **No compositor, no Wayland
stack, no CrownOS component, no AI runtime.**

`bootstrap_packages.x86_64` contains `arch-install-scripts` and `base`.

### Profile settings

| Key | Value |
|---|---|
| `buildmodes` | `('iso')` |
| `bootmodes` | `('bios.syslinux' 'uefi.systemd-boot')` |
| `airootfs_image_type` | `squashfs`, xz with the x86 BCJ filter, 1 M block and dict |
| `bootstrap_tarball_compression` | zstd `-19`, long, multithreaded |
| `file_permissions` | `/etc/shadow` 400, `/root` 750, the `/usr/local/bin` helpers 755, `/root/.gnupg` 700 |

### airootfs highlights

| Path | Effect |
|---|---|
| `etc/systemd/system/getty@tty1.service.d/autologin.conf` | Root autologin on tty1 |
| `etc/shadow` | **Empty root password** |
| `etc/ssh/sshd_config.d/10-archiso.conf` | `PermitRootLogin yes`, `PasswordAuthentication yes` |
| `etc/mkinitcpio.conf.d/archiso.conf` | The `archiso`, `archiso_loop_mnt` and `archiso_pxe_*` hooks |
| `etc/systemd/system/pacman-init.service` | `pacman-key --init` and `--populate` on boot |
| `etc/systemd/network/20-{ethernet,wlan,wwan}.network` | DHCP on all three classes |
| `etc/pacman.d/hooks/zzzz99-remove-custom-hooks-from-airootfs.hook` | Deletes ISO-only hooks so they do not leak into `pacstrap`ed systems |
| `root/.automated_script.sh` | Reads `script=` from the kernel command line and runs it — the kickstart mechanism |
| `usr/local/bin/choose-mirror` | Rewrites `mirrorlist` from `mirror=` on the command line |
| `usr/local/bin/livecd-sound` | ALSA unmute helper |
| `etc/systemd/system/livecd-talk.service` | speakup screen reader for accessibility boots |

Those live-medium defaults — empty root password, root autologin, permissive
sshd — are deliberate upstream choices for a rescue ISO and **inappropriate for
an installed desktop**. They must change before this becomes a real CrownOS
image. See [SECURITY.md](../../SECURITY.md).

---

## Boot flow

1. **BIOS** → syslinux. `whichsys.c32` dispatches to `archiso_pxe.cfg` or
   `archiso_sys.cfg`. Entries: `arch`, and `archspeech` (adds
   `accessibility=on`).
2. **UEFI** → systemd-boot. `timeout 15`, entries for normal boot, speech boot
   and Memtest86+.
3. **GRUB** (`grub/grub.cfg`, also used as `loopback.cfg`) — serial console on
   unit 0 at 115200, entries for normal, speech, Memtest86+ (EFI and PC), UEFI
   Shell for five architectures, firmware settings, shutdown and restart.
4. Kernel → mkinitcpio `archiso` hooks mount the squashfs → systemd → root
   autologin on tty1 → `.zlogin` → `.automated_script.sh`.

---

## Building it

**There is no CI here, or anywhere in the organization** — no workflow has ever
run, and the `Crown-OS/.github` repo that would host them does not exist on
GitHub. Nothing builds the image automatically.

A `build.sh` now exists at the repository root. It is cross-distro: native
`mkarchiso` when the host is Arch, and a privileged Arch container otherwise, so
"you need an Arch box to build the image" is no longer true.

```bash
./build.sh                 # native on Arch, container elsewhere
./build.sh --check         # report what this machine can do, build nothing
./build.sh --container     # force the container even on Arch
./build.sh --out DIR       # default ./out
```

> **`build.sh` is not on the default branch.** It is newly added and currently
> **untracked**, along with a `.gitignore` and a GPL-3.0-or-later `LICENSE`. A
> visitor cloning `main` today sees none of the three, and has to invoke
> `mkarchiso` by hand:
>
> ```bash
> sudo pacman -S archiso
> sudo mkarchiso -v -w /tmp/crownos-work -o /tmp/crownos-out ./crownos-iso
> ```
>
> That invocation is inferred from the profile layout and `buildmodes` — nothing
> committed to the repo documents it. It needs root and an Arch host.

---

## What making this a CrownOS ISO would involve

Roughly, in order:

1. **Brand it.** `iso_name`, `iso_label`, `iso_publisher`, `iso_application`,
   `install_dir`, hostname, motd.
2. **Ship the CrownOS packages.** That first needs PKGBUILDs — nothing in the
   organization is packaged today. Then either a CrownOS pacman repository in
   `pacman.conf`, or packages built into the profile.
3. **Add the Wayland stack** to `packages.x86_64` — it is not there.
4. **Add a session.** There is no session file, no greeter, and no
   `.desktop` entry for `crownpositor`.
5. **Fix the live-medium security defaults** for the installed case.
6. **Decide the installer.** Upstream `archinstall` is what is currently included.
7. **Resolve licensing** — see below.

---

## Licensing

This repository has **no LICENSE file on its default branch**, and it is a
verbatim copy of Arch Linux's archiso `releng` profile, whose scripts carry
`SPDX-License-Identifier: GPL-3.0-or-later`.

A GPL-3.0-or-later `LICENSE` — the correct answer for a derived archiso profile —
has been added to the working tree, but it is **untracked and unpushed**, so the
problem still stands on `main`. Only `crownos-setup`, `crownos-documentations`
and `crownshell` carry a committed LICENSE. Redistributing GPL-3.0 content under
an unlicensed or MIT umbrella is a problem that needs resolving before any
release. See [Project status](../00-overview/project-status.md#licensing).

---

## Note on the website's claims

The CrownOS website advertises three ISO editions (Desktop 2.4 GB, Minimal
780 MB, ARM aarch64 2.1 GB) across five mirrors. **None of that exists.** This
profile builds one unbranded x86_64 Arch rescue image, and there are no mirrors.
