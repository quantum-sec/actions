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
| `semantic-release.yaml` | semantic-release on the consumer's local release stack; interface mirrors `armor/actions` (same `GIT_TOKEN_BASIC` secret, same `semver` output — empty when no release was created). |
| `update-source-reference.yaml` | After a package release, rewrite the package's `?ref=` pins in the target repository (default `infrastructure-modules`). Minor/patch → direct GPG-signed push; major → feature branch + PR. |

## Consumption

Pin by tag — never a branch:

```yaml
jobs:
  validation:
    uses: quantum-sec/actions/.github/workflows/terraform-module-validation.yaml@v1.0.0
```

See any migrated `package-*` repo for a complete consumer example.

## Versioning

Semantic tags (`vX.Y.Z`). Breaking interface changes bump the major
version; consumers upgrade deliberately, one repo at a time — this is the
property the old mainline-only pipeline-library lacked.

## Security posture

Carried from the TRU-327/342 security review: strict-semver guards on all
tag/dispatch-derived versions before they reach privileged automation;
checksum-verified tool binaries; no shell tracing (consumers are public
repos with world-readable logs); secrets appear only in jobs gated to
push events; least-privilege `GITHUB_TOKEN` throughout (pushes
authenticate via `GIT_TOKEN_BASIC`/SSH).
