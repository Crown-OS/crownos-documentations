<!--
  CrownOS pull request template.

  CI covers fmt, build and tests. The Testing section below is for what CI
  cannot check — which compositor you ran under, and what you actually saw.
  Please be specific there.
-->

## Closes

<!-- Link the issue this resolves: Closes #<n> | Fixes #<n> | Resolves #<n>
     For a small, self-explanatory change, a clear Summary below is enough. -->

Closes #

## Summary

<!-- What does this change do, and why? -->

## Changes

<!-- Bullet list of notable changes. -->

-

## Testing

<!-- What did you run, and what did it say? Be specific — "tests pass" is not
     useful without saying which. Include manual verification too: which
     compositor you ran under, what you observed. -->

```
cargo fmt --all              →
cargo clippy -- -D warnings  →
cargo test --all             →
```

Manual verification:

## Checklist

- [ ] Branch is rebased onto `main`
- [ ] Each commit follows [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/)
- [ ] Each commit is a single logical change, with fixups squashed
- [ ] Each commit builds on its own
- [ ] `cargo fmt` / `bun run format` / equivalent has been run
- [ ] Relevant tests added or updated
- [ ] Documentation updated if this changes behaviour, config, env vars or keybindings

<!--
  Cross-repo changes: if this breaks a downstream repo (e.g. a crownshell API
  change affecting crownotify or crowndictator), say so here and link the
  companion PRs so a maintainer can sequence the merges.
-->
