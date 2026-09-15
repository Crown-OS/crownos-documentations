# crownos-documentations

**Status: Early** · Markdown · default branch `main` ·
[repo](https://github.com/Crown-OS/crownos-documentations)

This repository. The canonical documentation for every repo in the Crown-OS
organization.

---

## Layout

```
README.md               Entry point, component map, doc index
CONTRIBUTING.md         Contribution workflow for the whole organization
CODE_OF_CONDUCT.md      Contributor Covenant 2.1
SECURITY.md             Reporting policy and known security posture
LICENSE                 MIT
docs/
  00-overview/          What CrownOS is, what exists, what works
  10-getting-started/   Toolchain, checkout layout, building, first patch
  20-architecture/      How the pieces fit and talk to each other
  30-components/        One page per repository
  40-contributing/      Workflow, code standards, testing, CI, doc style
  50-reference/         Config schema, env vars, keybindings, glossary
  60-users/             Installing and using CrownOS
templates/.github/      Issue and PR templates to copy into other repos
```

No build step, no site generator. Plain Markdown that renders on GitHub. If a
static site is wanted later, this tree can feed one without being rewritten.

---

## Working on the docs

```bash
cd crownos-documentations
# edit files
git switch -c docs/your-change
```

The conventions these pages follow are in
[Documentation style](../40-contributing/documentation-style.md). The general
contribution workflow is in [CONTRIBUTING.md](../../CONTRIBUTING.md).

### The one rule that matters

**The code is authoritative.** These pages are written against what the source
actually does, including where that is unflattering — components that do not
compile, features that are specified and unimplemented, marketing copy that
disagrees with the implementation.

If a page and the code disagree, the page is a bug. Fix it or open an issue.
Do not "correct" the code to match the documentation without a separate,
deliberate decision.

### Checking your changes

**There is no CI to catch anything.** No workflow has ever run in any Crown-OS
repository, and the `Crown-OS/.github` repo that a shared `docs.yml` would live
in does not exist on GitHub. The relative-link check and the status-marker
consistency check are things you run by hand:

```bash
grep -ro '](\.\{1,2\}/[^)]*' docs/ README.md CONTRIBUTING.md
```

Also grep your own diff for mentions of CI, CODEOWNERS, sign-off and DCO. Each
one must describe what is actually configured — and today that is nothing, so an
unqualified "CI will reject…" is a bug.

The same applies to "fixed". Most of the build fixes these pages describe exist
only as uncommitted changes in someone's working tree. If a page says a defect is
fixed, it must say whether that is true on the default branch or only locally.

If you document a command, run it first.

---

## Status and history

One commit before this documentation landed — a two-line README. An unmerged
branch, `docs/contribution-guide`, carried an earlier draft of the contribution
guide; its structure was kept and its factual content rewritten, because it
described a seven-repository layout that never existed and CI infrastructure that
has never been created.

That branch is superseded by `CONTRIBUTING.md` on `main`.

---

## Known gaps in the documentation itself

Being consistent about honesty, these pages have gaps too:

- **No API reference here.** `crownshell` has rustdoc on
  [docs.rs](https://docs.rs/crownshell), and it is the only crate in the
  organization that does, because it is the only one published. These pages do
  not duplicate it — the
  [Reference](../50-reference) section covers configuration, environment
  variables and keybindings, not crate APIs.
- **No screenshots.** No page shows what CrownOS looks like.
- **Component pages are written from source reading**, not from running every
  component on hardware. Where a build was not verified, the page says so.
- **The user-facing section is thin** by design — CrownOS is not installable yet,
  so there is little to say. See [Install](../60-users/install.md).

---

## License

MIT, copyright "Crown-OS". One of only three repositories in the organization
carrying a LICENSE file on its default branch — `crownos-setup` and `crownshell`
are the others, and all three name a different copyright holder.
