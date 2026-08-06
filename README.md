# Actions

Reusable GitHub workflows shared by Quantum repositories.

## Workflows

| Workflow | Purpose |
|---|---|
| `terraform-module-validation.yaml` | Runs the repo's own pre-commit hooks (terraform pinned from `.terraform-version`, tflint, tfsec, terraform-docs) plus a Checkov scan. |
| `terragrunt-validation.yaml` | Pre-commit for terragrunt-based repos; optional `changed_files_only` mode. |
| `strict-yaml-validation.yaml` | Strict YAML scan of a directory (duplicate keys rejected). |
| `semantic-release.yaml` | Runs semantic-release on the consumer's release stack; outputs `semver` (empty when no release was created). Optional `release_artifact` input downloads a same-run workflow artifact before releasing. |
| `update-source-reference.yaml` | After a package release, rewrite the package's `?ref=` pins in a target repository. Minor/patch → direct push; major → feature branch + PR. |
| `import-release-content.yaml` | Import a release asset: download, validate everything it adds or changes, update the version pointer and changelog, commit. |
| `copy-source-to-repo.yaml` | Sync the paths in a path-list file into a mirror repository and tag the version. |
| `repository-dispatch.yaml` | Send a version-carrying `repository_dispatch` event to a downstream repository. |

Single-consumer automation deliberately stays in its own repository —
centralizing a one-off adds indirection without reuse.

## Consumption

Pin by tag, never a branch. Two conventions depending on the workflow:

- **Validation workflows** (no secrets): use the floating major tag, e.g.
  `terraform-module-validation.yaml@v1` — patch and minor improvements
  arrive without consumer changes.
- **Workflows that receive secrets**: pin an exact version, e.g.
  `semantic-release.yaml@v1.4.0` — new versions reach a consumer only
  through a reviewed pull request.

```yaml
jobs:
  validation:
    uses: quantum-sec/actions/.github/workflows/terraform-module-validation.yaml@v1

  release:
    needs: validation
    if: github.event_name == 'push' && (github.ref == 'refs/heads/master' || github.ref == 'refs/heads/main')
    permissions:
      contents: write
    uses: quantum-sec/actions/.github/workflows/semantic-release.yaml@v1.4.0
    secrets:
      GIT_TOKEN_BASIC: ${{ secrets.GIT_TOKEN_BASIC }}
```

## Versioning

Semantic tags (`vX.Y.Z`) plus a floating major tag (`v1`). Version tags
are protected against modification. Breaking interface changes bump the
major version.
