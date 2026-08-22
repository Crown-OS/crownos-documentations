# Templates

Canonical GitHub templates for the Crown-OS organization, kept here so there is
one copy to edit.

Every repository now has a `.github/workflows/` directory (CI callers), but
**none of them has the issue or pull request templates installed yet.** These
are staged for you to copy where you want them.

```
.github/
├── PULL_REQUEST_TEMPLATE.md
└── ISSUE_TEMPLATE/
    ├── bug_report.yml
    ├── feature_request.yml
    └── config.yml
```

## Installing them into a repository

From the root of the target repo:

```bash
mkdir -p .github/ISSUE_TEMPLATE
cp -r ../crownos-documentations/templates/.github/. .github/
```

Then commit:

```
chore: add issue and pull request templates
```

Once `.github/PULL_REQUEST_TEMPLATE.md` is present, GitHub auto-populates new
pull requests with it — at which point
[CONTRIBUTING.md](../CONTRIBUTING.md#opening-a-pr) can drop its note about
copying the body by hand.

## What is deliberately not here

**No GitHub Actions workflows.** Those live in
[`Crown-OS/.github`](https://github.com/Crown-OS/.github) as reusable workflows,
called by a thin `.github/workflows/ci.yml` in each repo. This directory is for
the human-facing templates only.

**No CODEOWNERS.** Review is by a human reading the diff; there is no branch
protection to enforce ownership.

**No `dependabot.yml`.** Several crates pin git dependencies with no `rev`, which
Dependabot cannot help with and automated bumps would make worse. Fix the pinning
first — see
[Dependency graph](../docs/20-architecture/dependency-graph.md#version-skew).

## Also worth copying

Two files at the root of this repository are organization-wide and can be copied
into other repos as-is:

- [`CODE_OF_CONDUCT.md`](../CODE_OF_CONDUCT.md)
- [`SECURITY.md`](../SECURITY.md)

GitHub will also apply a `CODE_OF_CONDUCT.md` and `SECURITY.md` from an
organization-level `.github` repository to every repo that lacks its own, which
is less to maintain than 16 copies.
