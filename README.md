# ci-templates

Reusable GitHub Actions workflows and composite actions for my repositories — one place to change
how CI works instead of a copy in every repo.

## Contents

| Path | Kind | What it does |
|---|---|---|
| `actions/tf-setup` | composite action | Terraform from `.terraform-version` (no wrapper) → email mask on `PATH` → OIDC hub role → role chaining into the target role → `terraform init` with a shared `backend.hcl` |
| `.github/workflows/tf-plan.yml` | reusable workflow | `lint` job without cloud access (tflint + checkov, configured by the caller's `.tflint.hcl` / `.checkov.yaml`) → optional selection of changed stacks → matrix per stack: `fmt -check` → `validate` → `plan`; plan posted as one PR comment per stack, updated in place; `terraform-ok` aggregates the result |
| `.github/workflows/tf-apply.yml` | reusable workflow | after merge: plan per stack (saved to a private S3 bucket) → approval in a GitHub environment, only for stacks with changes → apply exactly the saved plan → follow-up plan must be clean |
| `.github/workflows/tf-drift.yml` | reusable workflow | scheduled plan per stack: drift opens or updates one issue per stack (resource addresses only), a clean plan closes it; no auto-apply |
| `.github/workflows/container-build.yml` | reusable workflow | `lint` job without cloud access (hadolint + gitleaks) → build once → trivy scan → optional push to ECR as `<name>:<git sha>` (immutable); outputs the digest; a re-run of the same commit reuses the image |
| `.github/workflows/deploy-ecs.yml` | reusable workflow | in a GitHub environment: OIDC hub role → role chaining into the deploy role → new task definition revision with only the image changed → wait for the rollout, fail on a circuit-breaker rollback → smoke test |
| `.github/workflows/promote-ecr.yml` | reusable workflow | after an environment approval: add a release tag (e.g. `prod-<sha>`) to a digest that passed the earlier stages; retag only, no rebuild |

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

Plan only the stacks a PR changed (`<stack>/` plus extra prefixes; a change under `global-paths`
plans every stack). Mark `plan / terraform-ok` as the only required check: it always reports,
while a skipped workflow (`on.pull_request.paths`) or a skipped job would leave a required check waiting forever.

```yaml
    with:
      stacks: '["org", "identity"]'
      changed-only: true
      stack-paths: '{"org": ["policies/"]}'
```

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

App pipeline: build once, then deploy and promote the same digest.

```yaml
jobs:
  build:
    permissions:
      contents: read
      id-token: write
    uses: prohres/ci-templates/.github/workflows/container-build.yml@v1
    with:
      image-name: <ecr-repo>
      push: ${{ github.event_name == 'push' }}     # pull requests: lint, build and scan only
      gha-role-arn: arn:aws:iam::<hub-account>:role/gha-app-build

  deploy-dev:
    needs: build
    if: github.event_name == 'push'
    permissions:
      id-token: write
    uses: prohres/ci-templates/.github/workflows/deploy-ecs.yml@v1
    with:
      environment: dev
      image-ref: ${{ needs.build.outputs.image-ref }}
      gha-role-arn: arn:aws:iam::<hub-account>:role/gha-app-deploy-dev
      target-role-arn: arn:aws:iam::<dev-account>:role/app-deploy
      target-account-id: "<dev-account>"
      cluster: <cluster>
      service: <service>
      task-family: <family>
      container-name: <container>
      smoke-url: http://<alb-dns>/version.json
      smoke-expect: ${{ github.sha }}             # the new version answers, not the old task

  promote-prod:
    needs: [build, deploy-dev]
    permissions:
      id-token: write
    uses: prohres/ci-templates/.github/workflows/promote-ecr.yml@v1
    with:
      environment: prod                           # required reviewers
      image-ref: ${{ needs.build.outputs.image-ref }}
      tag: prod-${{ github.sha }}
      gha-role-arn: arn:aws:iam::<hub-account>:role/gha-app-promote
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
- Release binaries (hadolint, gitleaks, trivy) are checked against a sha256 pinned in the workflow;
  a version bump needs a new checksum.
- `container-build` scans the image before the push, and gitleaks runs with `--redact`: a finding never
  reaches the registry or the public log.
- `tf-setup` puts an email mask in front of `terraform` on `PATH`: every later Terraform call in the job
  prints `[redacted-email]` instead of an address (a provider error can print a sensitive value in clear
  text). Exit codes are kept; stderr is merged into stdout.
- `tf-setup` fails if any `TF_LOG*` variable is set (except `TF_LOG_PATH`): debug logs print provider
  API responses, and job logs of a public repo are public.
