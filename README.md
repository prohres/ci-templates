# ci-templates

Reusable GitHub Actions workflows and composite actions for my repositories — one place to change
how CI works instead of a copy in every repo.

## Contents

| Path | Kind | What it does |
|---|---|---|
| `actions/tf-setup` | composite action | Terraform from `.terraform-version` (no wrapper) → OIDC hub role → role chaining into the target role → `terraform init` with a shared `backend.hcl` |
| `.github/workflows/tf-plan.yml` | reusable workflow | `lint` job without cloud access (tflint + checkov, configured by the caller's `.tflint.hcl` / `.checkov.yaml`) → matrix per stack: `fmt -check` → `validate` → `plan`; plan posted as one PR comment per stack, updated in place |
| `.github/workflows/tf-apply.yml` | reusable workflow | after merge: plan per stack (saved to a private S3 bucket) → approval in a GitHub environment, only for stacks with changes → apply exactly the saved plan → follow-up plan must be clean |
| `.github/workflows/tf-drift.yml` | reusable workflow | scheduled plan per stack: drift opens or updates one issue per stack (resource addresses only), a clean plan closes it; no auto-apply |

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

Apply after merge:

```yaml
on:
  push:
    branches: [develop]
concurrency:
  group: terraform-apply
  cancel-in-progress: false  # never interrupt an apply
jobs:
  apply:
    permissions:
      contents: read
      id-token: write
    uses: prohres/ci-templates/.github/workflows/tf-apply.yml@v1
    with:
      stacks: '["org", "identity"]'
      plan-gha-role-arn: arn:aws:iam::<hub-account>:role/gha-plan
      plan-target-role-arn: arn:aws:iam::<target-account>:role/org-tf-plan
      apply-gha-role-arn: arn:aws:iam::<hub-account>:role/gha-apply
      apply-target-role-arn: arn:aws:iam::<target-account>:role/org-tf-apply
      target-account-id: "<target-account>"
      plan-bucket: <private-bucket>
      plan-prefix: <repo-name>
```

Saved plans hold values in clear text, so they go to a private S3 bucket, never to GitHub artifacts
(artifacts of a public repo are downloadable by anyone). Only a `true` / `false` "has changes" flag per
stack travels as an artifact.

Drift detection (schedule runs on the default branch only; GitHub disables schedules in public
repos after 60 days without activity):

```yaml
on:
  schedule:
    - cron: "0 6 * * *"
  workflow_dispatch:
jobs:
  drift:
    permissions:
      contents: read
      id-token: write
      issues: write
    uses: prohres/ci-templates/.github/workflows/tf-drift.yml@v1
    with:
      stacks: '["org", "identity"]'
      gha-role-arn: arn:aws:iam::<hub-account>:role/gha-plan
      target-role-arn: arn:aws:iam::<target-account>:role/org-tf-plan
      target-account-id: "<target-account>"
```

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
- Static checks run in a job without `id-token`; tool versions are pinned and at least two weeks old.
- `tf-setup` fails if any `TF_LOG*` variable is set (except `TF_LOG_PATH`): debug logs print provider
  API responses, and job logs of a public repo are public.
