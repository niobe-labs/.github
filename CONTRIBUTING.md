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
- Mark breaking changes with `!` after the type/scope and a
  `BREAKING CHANGE:` footer explaining what consumers must change.

Every commit should compile and pass tests on its own — avoid "fixup"
commits that only make sense combined with a later one.

## Code style

The rules below are the org's Go style guide. They build on
[Effective Go](https://go.dev/doc/effective_go), the
[Google Go Style Guide](https://google.github.io/styleguide/go/), and the
[Uber Go Style Guide](https://github.com/uber-go/guide/blob/master/style.md);
where those disagree, this section wins.

Each rule is tagged:

- **[lint: _name_]** — enforced by `make lint`; CI fails if it's broken.
- **[review]** — not machine-checkable; reviewers enforce it.

Suppressing a linter requires a `//nolint:<linter> // <why>` comment naming
the specific linter and the reason **[lint: nolintlint]**. Prefer fixing the
code; a nolint is a claim a reviewer must agree with.

### Formatting

- Formatting is tool-owned: `gofumpt`, `goimports`, `gci`, and `golines`
  (120 columns) run via `make format` **[lint: gofumpt, goimports, gci,
  golines]**. Don't hand-format around them — run `make format` and commit
  what it produces.
- Imports are grouped standard library, third party, then this module
  **[lint: gci]**.
- Every `.go` file starts with an SPDX header **[lint: goheader]**:

  ```go
  // SPDX-License-Identifier: Apache-2.0
  // Copyright Authors of Niobe Labs
  ```

### Naming and documentation

- Use Go naming: `MixedCaps`, initialisms in one case (`URL`, `ID`,
  `userID`), short receiver names used consistently **[lint: revive
  var-naming, receiver-naming]**. Don't shadow builtins (`len`, `new`,
  `error`) **[lint: predeclared, revive redefines-builtin-id]**.
- Prefer descriptive names over abbreviations, and named constants over
  magic numbers and strings **[review]**.
- Every package has a package comment, and every exported identifier outside
  `internal/` has a doc comment starting with its name **[lint: revive
  package-comments, exported]**. Comments explain *why*, not *what*
  **[review]**.
- Don't name packages `util`, `common`, `helpers`, or `misc`; name them for
  what they provide **[review]**.
- Sentinel errors are `ErrXxx`; error types are `XxxError` **[lint:
  errname]**.

### Errors

- Wrap errors with context using `%w` so callers can unwrap and inspect the
  cause: `fmt.Errorf("loading config: %w", err)` **[lint: errorlint]**.
  Compare with `errors.Is` / `errors.As`, never `==` or a type assertion
  **[lint: errorlint]**.
- Handle every error, or assign it to `_` with an evident reason
  **[lint: errcheck]**. Don't swallow errors, and don't both log and return
  the same error — pick one **[review]**.
- Don't return `nil, nil` **[lint: nilnil]**, and don't return a nil error
  after checking a non-nil one **[lint: nilerr, nilnesserr]**.
- Return early to keep the happy path unindented **[lint: revive
  indent-error-flow, superfluous-else; nestif]**.
- Library code returns errors; only `main` decides the process exit code
  **[lint: forbidigo `os.Exit`]**.

### Context

- `context.Context` is the first parameter, named `ctx` **[lint: revive
  context-as-argument]**. Never store it in a struct **[lint: containedctx]**;
  pass it down rather than creating a fresh `context.Background()`
  **[lint: contextcheck]**.
- Use context-aware APIs: `exec.CommandContext` **[lint: forbidigo]**,
  `http.NewRequestWithContext`, `net.ListenConfig.Listen` **[lint: noctx]**.
- Long-running work selects on `ctx.Done()` and returns promptly when it is
  canceled **[review]**.

### Concurrency

- Every goroutine has an owner that waits for it to finish; there are no
  fire-and-forget goroutines. Use `errgroup.Group` (or `sync.WaitGroup.Go`)
  so errors and shutdown propagate **[review]**.
- Workers stop when their context is canceled. Shutdown timing — drain,
  grace period, force-exit — lives only in `go-sdk/pkg/shutdown`; don't add
  per-worker shutdown timers **[review]**.
- Avoid package-level mutable state; pass dependencies in (constructors,
  `Options` structs) so code is testable and concurrency is explicit
  **[review]**. Where a process-wide singleton is unavoidable (as with the
  gops agent), model it honestly with package-level functions **[review]**.
- Document whether a type is safe for concurrent use **[review]**.

### Use the go-sdk

Shared concerns are implemented once in `github.com/niobe-labs/go-sdk`.
Services use it instead of the raw primitive:

| Concern | Use | Not | Enforced |
|---------|-----|-----|----------|
| Logging | `pkg/logger` (`logger.FromContext(ctx)` in request paths) | `log`, `log/slog`, `zerolog/log` | **[lint: depguard]** |
| Signals and shutdown | `pkg/shutdown` | `os/signal` | **[lint: depguard]** |
| Metrics | `pkg/metrics` registry and helpers | `promauto`, `promhttp`, the global registry | **[lint: depguard, forbidigo]** |
| Diagnostics | `pkg/monitor` | `google/gops` | **[lint: depguard]** |
| Writing files, creating dirs | `pkg/fsguard` (`WriteFileAtomic`, `Root`, `MkdirPrivate`) | `os.WriteFile`, `os.Create`, `os.OpenFile`, `os.Mkdir*` | **[lint: forbidigo]** |
| Unix socket listeners | `fsguard.ListenUnix` | hand-rolled remove/listen/chmod | **[review]** |
| Memory limit | `pkg/memlimit` at startup | hand-set `GOMEMLIMIT` | **[review]** |
| Health probes | `pkg/health` | ad-hoc handlers | **[review]** |
| Build metadata | `pkg/version` | custom ldflags vars | **[review]** |

If go-sdk lacks something you need, add it there rather than working around
the rule locally.

### Logging

- Log structured fields, not formatted strings:
  `log.Info().Str("addr", addr).Msg("listening")` **[review]**.
- Never log secrets, tokens, credentials, or full request bodies
  **[review]**.
- Library code never writes to stdout; write to an injected `io.Writer`
  (`cmd.OutOrStdout()`, a `ChunkWriter`) or log **[lint: forbidigo
  `fmt.Print*`]**.

### Configuration

- Read environment variables in exactly one place — a service's
  `internal/config` (go-sdk: `internal/envutil`) — and pass values through a
  `Config` **[lint: forbidigo `os.Getenv`]**.
- A set-but-malformed value is a startup error, not a silent fallback to the
  default **[review]**.
- Never hardcode secrets; read them from the environment or a mounted file
  checked with `fsguard.CheckPrivate` **[review; gosec]**.

### Files, network, and untrusted input

- Any path built from untrusted input (requests, config values, file
  contents) goes through `fsguard.Root` or `fsguard.JoinLocal` **[review]**.
- Create files and directories owner-only (`fsguard.PermPrivateFile`,
  `fsguard.PermPrivateDir`) unless something else must read them
  **[review; gosec]**.
- Bound everything an outside party controls: read sizes
  (`fsguard.ReadFileLimit`), message sizes, concurrency, and timeouts
  **[review]**.
- HTTP servers set `ReadHeaderTimeout`; don't use the package-level
  `http.ListenAndServe` helpers **[lint: forbidigo, gosec]**.
- Metric labels come from small, fixed sets — never user IDs, request IDs,
  or caller-supplied strings **[review]**.

### Complexity

Functions stay within 100 lines / 50 statements, cyclomatic complexity 15,
and cognitive complexity 20 **[lint: funlen, gocyclo, cyclop, gocognit]**.
Split by responsibility before reaching for a nolint **[review]**.

### Tests

- Tests live alongside the code as `<name>_test.go` and use the `_test`
  package suffix (black-box testing) unless they need white-box access to
  unexported state — then use the same package name and explain why in a
  comment **[review]**.
- Test behavior, not implementation: assert on inputs and outputs, not on
  internal call sequences **[review]**.
- Name tests `Test<Function>_<Scenario>` or
  `Test<Function>_<Scenario>_<Expected>` **[review]**.
- When there's more than one case, write a **table-driven test** in exactly
  this shape **[review]**:

  ```go
  func TestJoinLocal(t *testing.T) {
      t.Parallel()

      tests := []struct {
          name    string
          in      string
          want    string
          wantErr error
      }{
          {"plain name", "a.txt", "/base/a.txt", nil},
          {"parent escape", "../a.txt", "", fsguard.ErrNotLocal},
      }

      for _, tt := range tests {
          t.Run(tt.name, func(t *testing.T) {
              t.Parallel()

              got, err := fsguard.JoinLocal("/base", tt.in)
              if !errors.Is(err, tt.wantErr) {
                  t.Fatalf("JoinLocal(%q) error = %v, want %v", tt.in, err, tt.wantErr)
              }
              if got != tt.want {
                  t.Errorf("JoinLocal(%q) = %q, want %q", tt.in, got, tt.want)
              }
          })
      }
  }
  ```

  - The cases are a slice of anonymous structs named `tests`, each with a
    `name` field.
  - The loop is `for _, tt := range tests`, and every case runs in its own
    `t.Run(tt.name, ...)` so failures name the case and `-run` can select it.
  - Failure messages read `Func(input) = got, want want` — got before want.
  - Don't copy the loop variable (`tt := tt`); Go 1.22+ scopes it per
    iteration **[lint: copyloopvar]**.
- Call `t.Parallel()` in every test and subtest **[lint: paralleltest,
  tparallel]**. A test that touches process-wide state (signals, environment
  variables, package-level vars, global singletons) can't run in parallel;
  mark its file `//nolint:paralleltest // <which global state>`.
- Use `t.Context()`, `t.TempDir()`, and `t.Setenv()` instead of hand-rolled
  equivalents **[lint: usetesting]**, and call `t.Helper()` in test helpers
  **[lint: thelper]**.
- Tests must be repeatable and isolated: no network beyond loopback, no
  dependence on run order, and `make test` runs them with `-race`
  **[review]**.

## Dependencies

Prefer the standard library when it does the job **[review]**. Dependencies
are vendored. After any `go.mod` change (adding, removing, or upgrading a
dependency), run:

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
