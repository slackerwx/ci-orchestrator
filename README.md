# ci-orchestrator

Reusable GitHub Actions workflow for the VirtuaLab DevSecOps pipeline. Runs build, SAST (Semgrep), secret scanning (Gitleaks) and vulnerability scanning (Trivy) in parallel, then packages the image, generates an SBOM and signs it with Cosign (keyless, via OIDC). SARIF results and image lifecycle events are sent to VirtuaLab via `VLAB_URL`.

Each job carries the name of its corresponding stage (`build`, `sast`, `secrets`, `vuln`, `package`) to match the stage ids expected by VirtuaLab. The `language` input (`go`, `node`, `java`, `dotnet` or `python`) controls the `build` job setup: go uses `setup-go` + `go build`, node uses `setup-node` + `npm ci`, java uses `setup-java` (temurin 17) + `mvn package`, dotnet uses `setup-dotnet` (8.x) + `dotnet build`, python uses `setup-python` (3.12) + `pip install -r requirements.txt`.

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
    uses: slackerwx/ci-orchestrator/.github/workflows/devsecops.yaml@main
    with:
      language: go
    secrets:
      VLAB_URL: ${{ secrets.VLAB_URL }}
      VLAB_INGEST_TOKEN: ${{ secrets.VLAB_INGEST_TOKEN }}
```

The `permissions` block on the caller job is mandatory: a reusable workflow can only restrict the permissions inherited from the caller, never broaden them. Without `packages: write` and `id-token: write` declared there, the `package` job fails (GHCR push with 403, Cosign signing without an OIDC token).

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
