# CrownOS Documentation

CrownOS is an Arch-based, Wayland-native Linux distribution with a desktop shell
written from scratch in Rust, plus an ecosystem layer that bridges the desktop to
your phone.

This repository is the canonical documentation for every repo in the
[Crown-OS](https://github.com/Crown-OS) organization. It is written for
**contributors first** — if you want to understand how CrownOS fits together, or
build a piece of it and send a patch, start here.

> **CrownOS is early.** Most components are pre-1.0, a few do not compile today,
> and the ISO is not yet a CrownOS ISO. These docs say so explicitly, component
> by component. Nothing here describes software that does not exist — see
> [Project status](docs/00-overview/project-status.md) for the honest picture.

---

## Start here

| If you want to… | Read |
|---|---|
| Understand what CrownOS is | [What is CrownOS](docs/00-overview/what-is-crownos.md) |
| See every repo and what it does | [Component map](docs/00-overview/component-map.md) |
| Know what actually works today | [Project status](docs/00-overview/project-status.md) |
| Set up a dev environment | [Prerequisites](docs/10-getting-started/prerequisites.md) → [Workspace setup](docs/10-getting-started/workspace-setup.md) |
| Install native packages by hand | [Native packages, per distribution](docs/10-getting-started/native-packages.md) |
| Build and run something | [Build and run](docs/10-getting-started/build-and-run.md) |
| Make your first patch | [Your first change](docs/10-getting-started/your-first-change.md) |
| Understand the design | [Architecture overview](docs/20-architecture/overview.md) |
| Contribute code | [CONTRIBUTING.md](CONTRIBUTING.md) |
| Set up any Linux distro | [`crownos-setup`](https://github.com/Crown-OS/crownos-setup) |
| Publish a release | [Releasing](docs/40-contributing/releasing.md) |

---

## The component map, in brief

CrownOS is a multi-repo project. There is no monorepo and no umbrella workspace.

```
                        ┌────────────────────┐
                        │   crownpositor     │  Wayland compositor.
                        │   (the Wayland     │  Owns the session, spawns
                        │      server)       │  the rest of the desktop.
                        └─────────┬──────────┘
                                  │ wlr-layer-shell
              ┌───────────┬───────┴────┬─────────────┐
              │           │            │             │
         ┌────┴───┐  ┌────┴────┐  ┌────┴─────┐  ┌────┴────────┐
         │crownbar│  │crowndock│  │crownotify│  │crowndictator│
         └────┬───┘  └────┬────┘  └────┬─────┘  └────┬────────┘
              └───────────┴────────────┴─────────────┘
                                  │ built on
                          ┌───────┴────────┐
                          │   crownshell   │  Layer-shell + Vello
                          │  (the library) │  framework.
                          └────────────────┘

        ┌───────────────────────────────────────────────────────┐
        │  crownos-config — ~/.config/crownos/<section>.ron      │
        │  Shared settings, watched with inotify. This is how    │
        │  components coordinate. There is no IPC daemon.        │
        └───────────────────────────────────────────────────────┘
```

Full table with status markers: [Component map](docs/00-overview/component-map.md).

---

## Documentation layout

```
docs/
  00-overview/          What CrownOS is, what exists, what works
  10-getting-started/   Toolchain, native packages, checkout layout, building
  20-architecture/      How the pieces fit and talk to each other
  30-components/        One page per repository
  40-contributing/      Workflow, code standards, testing, CI, releasing, doc style
  50-reference/         Config schema, env vars, keybindings, glossary
  60-users/             Installing and using CrownOS
templates/.github/      Issue and PR templates to copy into other repos
```

---

## Contributing to the documentation

Docs live in this repo as plain Markdown — no build step, no site generator.
Edit a file, open a PR. See [Documentation style](docs/40-contributing/documentation-style.md)
for the conventions these pages follow, and [CONTRIBUTING.md](CONTRIBUTING.md)
for the general workflow.

If you find a page that disagrees with the code, **the code is right and the page
is a bug**. Please open an issue or fix it.

---

## License

Documentation in this repository is [MIT licensed](LICENSE).

Individual CrownOS repositories carry their own licensing, and it is currently
inconsistent — see [Project status](docs/00-overview/project-status.md#licensing).
