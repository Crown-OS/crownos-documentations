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

CI runs a relative-link check and a status-marker consistency check on every
push. Both are in `Crown-OS/.github`'s `docs.yml`. To run the link check
yourself before pushing:

```bash
grep -ro '](\.\{1,2\}/[^)]*' docs/ README.md CONTRIBUTING.md
```

CI also prints every mention of CI, CODEOWNERS, sign-off and DCO to the job
summary for review. Each one must describe what is actually configured — an
unqualified "CI will reject…" is a bug.

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
  [docs.rs](https://docs.rs/crownshell), and every crate published from now on
  gets the same automatically. These pages do not duplicate it — the
  [Reference](../50-reference) section covers configuration, environment
  variables and keybindings, not crate APIs.
- **No screenshots.** No page shows what CrownOS looks like.
- **Component pages are written from source reading**, not from running every
  component on hardware. Where a build was not verified, the page says so.
- **The user-facing section is thin** by design — CrownOS is not installable yet,
  so there is little to say. See [Install](../60-users/install.md).

---

## License

MIT. One of only two repositories in the organization with a LICENSE file.
