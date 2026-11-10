# ci-templates

Reusable GitHub Actions workflows and composite actions for my repositories — one place to change
how CI works instead of a copy in every repo.

## Contents

| Path | Kind | What it does |
|---|---|---|
| `actions/tf-setup` | composite action | Terraform from `.terraform-version` (no wrapper) → OIDC hub role → role chaining into the target role → `terraform init` with a shared `backend.hcl` |
| `.github/workflows/tf-plan.yml` | reusable workflow | matrix per stack: `fmt -check` → `validate` → `plan`; plan posted as one PR comment per stack, updated in place |

Composite actions are building blocks (steps on the caller's runner); reusable workflows are whole
pipelines (own jobs, matrix, permissions, environments).

## Usage

```yaml
jobs:
  plan:
    permissions:
      contents: read
      id-token: write       # OIDC token for AWS
      pull-requests: write  # plan comment
    uses: prohres/ci-templates/.github/workflows/tf-plan.yml@v1
    with:
      stacks: '["org", "identity"]'
      gha-role-arn: arn:aws:iam::<hub-account>:role/gha-plan
      target-role-arn: arn:aws:iam::<target-account>:role/org-tf-plan
      target-account-id: "<target-account>"
```

A reusable workflow never gets more permissions than its caller grants.

## Versioning

| Ref | Meaning |
|---|---|
| `@v1` | moving major tag: compatible fixes and features, never breaking changes — use this |
| `@v1.2.3` | exact release |
| `@main` / `@develop` | never in consumers: a change would reach every repo at once |

Breaking change (removed or renamed input, changed behavior) → new major `v2`; `v1` keeps working
until consumers migrate. Prefer expand → contract: add the new input in `v1.x`, deprecate the old one,
remove it in `v2`.

## Security

- Third-party actions are pinned by commit SHA (tags can be rewritten), with the version in a comment.
- Inputs reach shell steps through `env`, never inline `${{ }}` in `run` (script injection).
- `actions/checkout` runs with `persist-credentials: false`.
