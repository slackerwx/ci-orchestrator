# ci-orchestrator

`devsecops.yaml` is a thin adapter over [`slackerwx/devsecops-pipeline`](https://github.com/slackerwx/devsecops-pipeline):
it maps `language` → `stack`, runs the pipeline in `audit` mode (the sample apps carry planted
CVEs), always publishes, signs with the legacy Cosign format Kyverno 3.8 reads, enables DAST on the
sample's port and forwards SARIF/custody events to VirtuaLab through the pipeline's evidence webhook
(`VLAB_URL/api/ingest`). Stage names seen by VirtuaLab are the pipeline's job names:
`plan, build, test, secrets, sast, sca, iac, image, dast, sign, evidence`.

## Usage

```yaml
name: pipeline
on: [push]

jobs:
  devsecops:
    permissions:
      contents: read
      packages: write
      id-token: write
      pull-requests: write
      security-events: write
    uses: slackerwx/ci-orchestrator/.github/workflows/devsecops.yaml@main
    with:
      language: go
    secrets:
      VLAB_URL: ${{ secrets.VLAB_URL }}
      VLAB_INGEST_TOKEN: ${{ secrets.VLAB_INGEST_TOKEN }}
```

The `permissions` block on the caller job is mandatory, and all five entries are required: a reusable workflow can only restrict the permissions inherited from the caller, never broaden them, and that rule applies down the whole chain (caller → `devsecops.yaml` → `devsecops-pipeline`). `packages: write` covers the GHCR push and the Cosign signature upload, `id-token: write` the keyless OIDC signing, and `pull-requests: write` + `security-events: write` are declared by the pipeline's `evidence` job (it declares both unconditionally, even though the sticky PR comment and the Code Scanning upload are themselves optional). Omitting any of them does not fail a job — it fails the whole run at startup with zero jobs created.

## terraform.yaml

Reusable workflow for the VirtuaLab IaC pipeline. Runs `trivy-action` (scan-type `config`) against the caller repo, then spins up a LocalStack in `services` and executes `plan`/`apply` via Terragrunt on the `network`, `storage` and `messaging` modules under `envs/<env>/`, in that order (network before storage/messaging due to implicit network dependencies). Jobs are named `iac`, `plan`, `apply` to match the VirtuaLab stage ids. The `iac` SARIF and the `plan.summary` / `resource.applied` / `verify.passed` events are sent via `VLAB_URL`.

Pinned versions: Terraform `1.15.8` (`terraform_wrapper: false`, since Terragrunt is what calls the binary), Terragrunt `v1.1.2` (binary downloaded from the release and validated against the official `SHA256SUMS`), `localstack/localstack:2026.07.2`. Terragrunt v1.x uses `--working-dir` (instead of the old `-chdir`) and `--non-interactive`/`TG_NON_INTERACTIVE` to run without prompts in CI.

The `apply` job counts the created resources (`s3api list-buckets` + `dynamodb list-tables` + `sqs list-queues` + `ec2 describe-vpcs` filtered by `tag:Name=vlab-*`, which excludes the LocalStack default VPC) and subtracts 1 from the total for the backend's `vlab-tfstate` bucket, which is not a demo resource.

```yaml
name: pipeline
on:
  workflow_dispatch:
jobs:
  terraform:
    permissions:
      contents: read
    uses: slackerwx/ci-orchestrator/.github/workflows/terraform.yaml@main
    with:
      env: dev
    secrets:
      VLAB_URL: ${{ secrets.VLAB_URL }}
      VLAB_INGEST_TOKEN: ${{ secrets.VLAB_INGEST_TOKEN }}
```
