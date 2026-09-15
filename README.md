> [!IMPORTANT]
> **This repository is archived. The documentation moved to
> [Crown-OS/crownOs](https://github.com/Crown-OS/crownOs/tree/main/docs), beside the code it describes.**
>
> It was kept separate and it drifted. When the configuration schema changed on
> 7 September 2026, five pages here went on describing types that no longer
> existed, and nothing caught it -- because nothing here is compiled, tested or
> built against the code. The replacement is smaller on purpose: every default it
> quotes is printed from the real parser, the shipped example config is parsed by
> a test, and a CI job fails the build if a link stops resolving.
>
> | What you wanted | Where it is now |
> |---|---|
> | Contribution guide | [CONTRIBUTING.md](https://github.com/Crown-OS/crownOs/blob/main/CONTRIBUTING.md) |
> | What is CrownOS, project status | [README](https://github.com/Crown-OS/crownOs#verified-not-assumed) |
> | Build and run | [README](https://github.com/Crown-OS/crownOs#build-it) |
> | Prerequisites, per distribution | [crownOs-setup](https://github.com/Crown-OS/crownOs-setup/blob/main/generated/prerequisites.md) |
> | Architecture, layer-shell stack, IPC | [docs/architecture.md](https://github.com/Crown-OS/crownOs/blob/main/docs/architecture.md) |
> | Config schema reference | [docs/configuration.md](https://github.com/Crown-OS/crownOs/blob/main/docs/configuration.md) |
> | Keybindings reference | [docs/keybindings.md](https://github.com/Crown-OS/crownOs/blob/main/docs/keybindings.md) |
> | Known limitations, troubleshooting | [docs/troubleshooting.md](https://github.com/Crown-OS/crownOs/blob/main/docs/troubleshooting.md) |
> | Security policy | [.github/SECURITY.md](https://github.com/Crown-OS/.github/blob/main/SECURITY.md) |
>
> Nothing below is maintained. It is kept because the history is worth having and
> because old links should still resolve -- not because it is accurate.

# CrownOS Documentation

CrownOS is an Arch-based, Wayland-native Linux distribution with a desktop shell
written from scratch in Rust, plus an ecosystem layer that bridges the desktop to
your phone.

This repository is the canonical documentation for every repo in the
[Crown-OS](https://github.com/Crown-OS) organization. It is written for
**contributors first** — if you want to understand how CrownOS fits together, or
build a piece of it and send a patch, start here.

> **CrownOS is early, and cannot be installed.** There is no image and no
> package; the only CrownOS crate ever published to crates.io is `crownshell`,
> a library. A nested compositor session is the only way to run it — see
> [Try it without installing](docs/60-users/try-it-without-installing.md).
>
> A plain `git clone` of most repositories does not build either, because their
> manifests ask for versions that were never published. A `[patch.crates-io]`
> overlay above your checkouts is mandatory; see
> [Workspace setup](docs/10-getting-started/workspace-setup.md#the-overlay-mandatory).
>
> These docs say so explicitly, component by component. Nothing here describes
> software that does not exist — see
> [Project status](docs/00-overview/project-status.md) for the honest picture.

---

## Start here

| If you want to… | Read |
|---|---|
| Understand what CrownOS is | [What is CrownOS](docs/00-overview/what-is-crownos.md) |
| See every repo and what it does | [Component map](docs/00-overview/component-map.md) |
| Know what actually works today | [Project status](docs/00-overview/project-status.md) |
| **Actually try CrownOS** | [Try it without installing](docs/60-users/try-it-without-installing.md) |
| Work out why something is broken | [Troubleshooting](docs/60-users/troubleshooting.md) |
| Set up a dev environment | [Prerequisites](docs/10-getting-started/prerequisites.md) → [Workspace setup](docs/10-getting-started/workspace-setup.md) |
| Install native packages by hand | [Native packages, per distribution](docs/10-getting-started/native-packages.md) |
| Build and run something | [Build and run](docs/10-getting-started/build-and-run.md) |
| Make your first patch | [Your first change](docs/10-getting-started/your-first-change.md) |
| Help by testing, not coding | [Testing and reporting](docs/40-contributing/testing-and-reporting.md) |
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
  60-users/             Running CrownOS, troubleshooting, limitations
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
inconsistent — three of the sixteen repositories have a LICENSE file on their
default branch. See
[Project status](docs/00-overview/project-status.md#licensing).
