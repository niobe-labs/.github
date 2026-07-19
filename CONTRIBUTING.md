# Contributing

This guide applies to Go repositories across the Niobe Labs organization
(`go-sdk`, `template`, and services scaffolded from it). Repo-specific
details — layout, make targets, package-level conventions — live in each
repo's own `AGENTS.md`.

## Development workflow

1. Create a branch from `main` (or `master`), make your change, and keep
   commits focused — one logical change per commit.
2. Rebase-only: keep history linear. Rebase your branch onto the latest
   `main` instead of merging it in; merge commits are not used.
3. Sign every commit (`git config commit.gpgsign true`).
4. Open a pull request using the repository's PR template, with a clear
   description and a test plan a reviewer can reproduce.

## Commit messages

Use [Conventional Commits](https://www.conventionalcommits.org/):
`type(scope): description`

- Types: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`, `ci`
- Example: `feat(auth): add JWT validation`

Every commit should compile and pass tests on its own — avoid "fixup"
commits that only make sense combined with a later one.

## Formatting and linting

Formatting is tool-enforced, not a matter of taste: `gofumpt`, `goimports`,
`gci`, and `golines` run via `make format`, and `make lint` (golangci-lint,
containerized) checks the result. Don't hand-format around them — run
`make format` and commit what it produces.

Every `.go` file starts with an SPDX header:

```go
// SPDX-License-Identifier: Apache-2.0
// Copyright Authors of Niobe Labs
```

## Tests

- Write tests alongside the code you're changing, as `<name>_test.go`.
- Prefer table-driven tests when there's more than one case.
- Use the `_test` package suffix (black-box testing) unless the test needs
  white-box access to unexported state — then use the same package name and
  explain why in a comment.
- Test behavior, not implementation: assert on inputs and outputs, not on
  internal call sequences.

## Error handling

Wrap errors with context using `%w` so callers can unwrap and inspect the
cause: `fmt.Errorf("doing the thing: %w", err)`. Don't swallow errors or log
and return them separately — pick one.

## Dependencies

Dependencies are vendored. After any `go.mod` change (adding, removing, or
upgrading a dependency), run:

```bash
make vendor
```

which tidies, vendors, and verifies modules, and commit the resulting
`vendor/` diff in the same change as the `go.mod`/`go.sum` update. CI fails
if `vendor/` is out of sync.

## Before you push

Run `make validate` (lint + test + build, plus proto-breaking checks where
applicable) locally. It's the same gate CI runs, so a clean local run means
the PR should pass CI without back-and-forth.
