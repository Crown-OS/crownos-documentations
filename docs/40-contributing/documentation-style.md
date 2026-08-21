# Documentation style

Conventions for the pages in this repository.

---

## The governing rule

**The code is authoritative.** Every page here is written against what the source
actually does, including where that is unflattering.

That means:

- If a component does not compile, the page says so, and says why.
- If a feature is specified and unimplemented, the page says which.
- If the website's copy disagrees with the implementation, the page records the
  discrepancy rather than repeating the copy.
- If something is an inference rather than a stated fact, the page marks it as
  one.

A contributor who follows these docs and hits a wall we knew about has been
failed. Write for that person.

---

## Status markers

Every component page opens with one, and it must match
[Component map](../00-overview/component-map.md).

| Marker | Meaning |
|---|---|
| **Stable** | Builds, has tests, API is settling |
| **Early** | Builds and runs; API will move |
| **Partial** | Builds, but a headline feature is a no-op |
| **Skeleton** | Scaffolding only |
| **Broken** | Does not currently compile |
| **Empty** | No commits |

If you change a marker on a component page, change it in the component map in the
same commit.

---

## Page structure

### Component pages (`docs/30-components/`)

```markdown
# <name>

**Status: <marker>** · <language> · default branch `<branch>` · [repo](…)

One or two sentences on what it is.

## <what it provides / design>
## Prerequisites          (only if non-obvious)
## Build and run
## How it fits into CrownOS
## Known limitations       (required — never omit this)
## License
```

**Known limitations is not optional.** It is the section contributors most need
and the one most likely to be quietly dropped.

### Everything else

Lead with what the page is for. Put the thing a reader most needs near the top —
`workspace-setup.md` leads with the required directory layout because that is the
single fact that makes builds work.

---

## Writing

**Plain, direct sentences.** No marketing register. "It does not compile" rather
than "compilation is currently unavailable".

**Say the specific thing.** "`crowndock` cannot launch applications — there is no
`Exec=` parsing and no `std::process::Command` anywhere in the crate" is useful.
"The dock is incomplete" is not.

**Quote the source when it explains itself well.** Several modules carry good
reasoning in their doc comments; quoting is better than paraphrasing:

> No second enum for gestures — that is how "swipe left" and "Super+Tab" drift
> apart.

**Mark inference explicitly.** Where a conclusion is drawn rather than stated,
say so in parentheses: *(This is inference. Treat it as such.)*

**Give the file path.** When you reference behaviour, name the file so a reader
can check. Paths are repo-relative — `compositor/src/handlers/layer_shell.rs`,
not an absolute path from your machine.

**Do not enumerate line numbers.** They rot on the next commit. Name the file and
the function.

---

## Formatting

- **Wrap at 80 columns.** Every page here does.
- **Tables for anything with more than two parallel facts.** Prose comparisons of
  four components are unreadable.
- **Fenced code blocks with a language tag** — `bash`, `rust`, `toml`, `sh`.
- **`---` between major sections** on long pages.
- **Bold for the load-bearing claim in a paragraph**, sparingly. If everything is
  bold, nothing is.
- **Blockquotes for warnings and caveats**, prefixed with a bold lead:
  ```markdown
  > **The prelude module is spelled `predule`.** A typo in the public API…
  ```

### Links

Relative, always. From `docs/30-components/crownbar.md`:

```markdown
[Project status](../00-overview/project-status.md)
[Prerequisites](../10-getting-started/prerequisites.md#crowndictator)
[CONTRIBUTING.md](../../CONTRIBUTING.md)
```

Deep-link to a heading anchor when you are pointing at one specific fact in a
long page.

---

## Commands

**Run every command before you document it.** If it fails today, document that it
fails and why — do not quietly write the version that would work.

Include the caveats that actually bite:

```bash
# Tests need a live session bus, and must not run concurrently.
dbus-run-session -- cargo test -- --test-threads=1
```

Show the working directory when it matters:

```bash
cd crownshell
cargo run --example text_bar
```

---

## Diagrams

ASCII, in a fenced block. They render everywhere, diff cleanly, and need no
tooling. Box-drawing characters are fine.

Keep them to one idea. The process-model diagram in
[Architecture overview](../20-architecture/overview.md) shows what runs and what
talks to what — it does not also try to show the build graph, which has its own
page.

---

## Checking your changes

There is no CI. Before opening a PR:

```bash
# No dead internal links
grep -ro '](\.\{1,2\}/[^)]*' docs/ README.md CONTRIBUTING.md

# No claims about automation that does not exist
grep -riE 'CI |CODEOWNERS|sign-off|DCO' docs/ CONTRIBUTING.md
```

The second must only match text describing those things as absent or planned. If
you have written "CI will reject…" anywhere, that is the bug this check exists to
find.

Then re-read your page as someone who has never seen the codebase. The most
common failure in these docs is assuming context that only exists in the author's
head.

---

## When the code changes

Documentation that describes a fixed bug as current is worse than no
documentation. If your PR:

- **makes a Broken component build** → update its status marker here and in the
  component map, and remove it from
  [Project status](../00-overview/project-status.md)
- **implements a listed gap** → remove it from that component's Known limitations
- **adds a config section** → add it to
  [Config schema](../50-reference/config-schema.md)
- **adds an environment variable** → add it to
  [Environment variables](../50-reference/environment-variables.md)
- **changes a default keybinding** → update
  [Keybindings](../50-reference/keybindings.md)

Do it in the same PR. A follow-up "update the docs" issue never gets closed.
