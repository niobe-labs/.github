# .github

This repository holds two unrelated things GitHub treats specially:

- **`profile/README.md`** — the org profile page shown on
  <https://github.com/niobe-labs>.
- **Community-health files at the repo root** — `CONTRIBUTING.md`,
  `CODE_OF_CONDUCT.md`, `SECURITY.md`, `SUPPORT.md`,
  `PULL_REQUEST_TEMPLATE.md`, and `ISSUE_TEMPLATE/`.

## How org defaulting works

For the second category, GitHub falls back to the file in the org's
`.github` repo *only* for repositories that don't have their own copy at the
expected path. If `go-sdk/SECURITY.md` exists, GitHub uses it; if it doesn't,
GitHub serves this repo's `SECURITY.md` instead. This lets every repo in the
org inherit one canonical policy/template set without copy-pasting it
everywhere, while still letting an individual repo override any single file
by adding its own.

See GitHub's docs on [creating a default community health file](https://docs.github.com/en/communities/setting-up-your-project-for-healthy-contributions/creating-a-default-community-health-file)
for the full list of file types this applies to.

## What's intentionally not here

- **No Renovate preset** — each repo keeps its own `renovate.json`.
- **No `workflow-templates/`** — CI workflows are not centralized here.
- **No `CODEOWNERS`** — GitHub does not default this file from `.github`;
  each repo defines its own.
