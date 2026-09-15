# Templates

Canonical GitHub templates for the Crown-OS organization, kept here so there is
one copy to edit.

These are **installed in `Crown-OS/.github`**, which GitHub applies to every
repository in the organisation that does not carry its own — so there is one copy
to edit rather than sixteen. They stay here as the source you edit; copy the
result across if a repo ever needs to override the org default.

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

**No `dependabot.yml`.** Not because of git pinning — there are no git
dependencies left in the tree. Because `crown-versions.toml` in `Crown-OS/.github`
is the single declaration of every dependency more than one repo uses, and
per-repo bumps would fight it. Raise versions there, then propagate with
`scripts/sync-versions.py`.

## Also worth copying

Two files at the root of this repository are organization-wide and can be copied
into other repos as-is:

- [`CODE_OF_CONDUCT.md`](../CODE_OF_CONDUCT.md)
- [`SECURITY.md`](../SECURITY.md)

GitHub will also apply a `CODE_OF_CONDUCT.md` and `SECURITY.md` from an
organization-level `.github` repository to every repo that lacks its own, which
is less to maintain than 16 copies.
