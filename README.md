# Actions

Reusable GitHub Workflows for the Quantum content-flow repositories
([TRU-342](https://armor-defense.atlassian.net/browse/TRU-342)).

This repository deliberately mirrors the shape and interfaces of
[`armor/actions`](https://github.com/armor/actions): when the consumer
repositories move to the armor org ([TRU-329](https://armor-defense.atlassian.net/browse/TRU-329)),
each `uses:` reference converges with an org-name swap, and the
Terraform-module workflows here are the candidate contribution to
`armor/actions` as the canonical versions.

## Workflows

| Workflow | Purpose |
|---|---|
| `terraform-module-validation.yaml` | The repo's own pre-commit hooks (terraform pinned from `.terraform-version`, tflint, checksum-verified tfsec) + Checkov. No secrets — safe on fork-triggered runs. |
| `terragrunt-validation.yaml` | Pre-commit for terragrunt-flavored repos (live/staging/mirror repos): terraform + terragrunt tooling; optional `changed_files_only` mode for mirrors. No secrets. |
| `semantic-release.yaml` | semantic-release on the consumer's local release stack; interface mirrors `armor/actions` (same `GIT_TOKEN_BASIC` secret, same `semver` output — empty when no release was created). |
| `update-source-reference.yaml` | After a package release, rewrite the package's `?ref=` pins in the target repository (default `infrastructure-modules`). Minor/patch → direct GPG-signed push; major → feature branch + PR. |
| `import-release-content.yaml` | Import a library release asset: download, strict-validate everything it adds/changes (bad content fails *before* the commit), version pointer, changelog, commit + push. Caller owns the dispatch triggers and concurrency group. |
| `copy-source-to-repo.yaml` | Sync the paths in a path-list file into a mirror repository: rsync, GPG-signed bot commit, version tag, push. |
| `strict-yaml-validation.yaml` | Full-state strict YAML scan of a directory (duplicate keys rejected) — the release exit-door check. No secrets. |
| `repository-dispatch.yaml` | Send a version-carrying `repository_dispatch` to a downstream repo — the content chain's library-release → import trigger, defined once. |

Deliberately **not** centralized: single-consumer automation (e.g.
infrastructure-live-customer's template-update job) stays in its repo —
centralizing a one-off adds indirection without reuse.

## Consumption — the pin policy

Consumers pin by **blast radius**, never by branch:

```yaml
jobs:
  validation:                     # receives no secrets → floats
    uses: quantum-sec/actions/.github/workflows/terraform-module-validation.yaml@v1

  release:                        # receives credentials → exact pin
    uses: quantum-sec/actions/.github/workflows/semantic-release.yaml@v1.4.0
    secrets:
      GIT_TOKEN_BASIC: ${{ secrets.GIT_TOKEN_BASIC }}
```

| Workflow class | Pin | Workflows |
|---|---|---|
| **Secret-free validation** | floating major `@v1` | `terraform-module-validation`, `terragrunt-validation`, `strict-yaml-validation` |
| **Secret-handling** | exact `@vX.Y.Z` | `semantic-release`, `update-source-reference`, `import-release-content`, `copy-source-to-repo`, `repository-dispatch` |

**Why the split.** Floating a pin means accepting future versions without
review, and whether that is acceptable depends on what the job holds when
it runs:

- The validation workflows receive **zero secrets** and a read-only
  `GITHUB_TOKEN`. A bad version can misreport a check — visible and
  recoverable. They are also the workflows that change most often
  (tool additions, linter updates), so floating them eliminates the
  recurring pin-bump PRs across every consumer for changes that carry
  no credential risk.
- The secret-handling workflows receive org-write credentials, the CI
  bot's GPG signing key, and its SSH key. A bad version of one of these
  is a supply-chain compromise (the `tj-actions/changed-files` and
  Codecov incidents both targeted exactly this class). Exact pins mean a
  new version reaches a consumer only through a reviewed PR diff — that
  review is the one human checkpoint between this repo and those
  credentials, and these workflows change rarely enough that the cost is
  negligible. Interface-level changes need caller edits anyway, so
  floating would not even save the PR.

**Why floating `@v1` is safe to trust at all:** a `v*` tag ruleset on
this repository blocks tag creation, retargeting, and deletion for
everyone except repository admins. Combined with the required-PR rule on
`master` and the actionlint self-CI, moving `v1` requires admin access —
the same bar as changing repository settings. (The ruleset also makes
the exact pins immutable to non-admins, closing the mutable-tag attack
on that side.)

See any migrated `package-*` repo for a complete consumer example.

## Versioning

Semantic tags (`vX.Y.Z`), plus a floating major tag (`v1`) moved to each
release by an admin. Breaking interface changes bump the major version;
consumers of secret-handling workflows upgrade deliberately, one repo at
a time — this is the property the old mainline-only pipeline-library
lacked.

## Security posture

Carried from the TRU-327/342 security review: strict-semver guards on all
tag/dispatch-derived versions before they reach privileged automation;
checksum-verified tool binaries; no shell tracing (consumers are public
repos with world-readable logs); secrets appear only in jobs gated to
push events; least-privilege `GITHUB_TOKEN` throughout (pushes
authenticate via `GIT_TOKEN_BASIC`/SSH).
