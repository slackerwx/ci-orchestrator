# ci-orchestrator

Reusable GitHub Actions workflow para o pipeline DevSecOps do VirtuaLab. Executa build, SAST (Semgrep), secret scanning (Gitleaks) e scan de vulnerabilidades (Trivy) em paralelo, depois empacota a imagem, gera SBOM e assina com Cosign (keyless, via OIDC). Resultados SARIF e eventos do ciclo de vida da imagem são enviados ao VirtuaLab via `VLAB_URL`.

Cada job carrega o nome do stage correspondente (`build`, `sast`, `secrets`, `vuln`, `package`) para casar com os stage ids esperados pelo VirtuaLab. O input `language` (`go`, `node`, `java`, `dotnet` ou `python`) controla o setup do job `build`: go usa `setup-go` + `go build`, node usa `setup-node` + `npm ci`, java usa `setup-java` (temurin 17) + `mvn package`, dotnet usa `setup-dotnet` (8.x) + `dotnet build`, python usa `setup-python` (3.12) + `pip install -r requirements.txt`.

## Uso

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

O bloco `permissions` no job caller é obrigatório: um reusable workflow só restringe as permissões herdadas do chamador, nunca amplia. Sem `packages: write` e `id-token: write` declarados aí, o job `package` falha (push no GHCR com 403, assinatura Cosign sem token OIDC).

## terraform.yaml

Reusable workflow para o pipeline de IaC do VirtuaLab. Roda `trivy-action` (scan-type `config`) contra o repo chamador, depois sobe um LocalStack em `services` e executa `plan`/`apply` via Terragrunt nos módulos `network`, `storage` e `messaging` de `envs/<env>/`, nessa ordem (network antes de storage/messaging por causa de dependências implícitas de rede). Jobs nomeados `iac`, `plan`, `apply` para casar com os stage ids do VirtuaLab. SARIF do `iac` e eventos `plan.summary` / `resource.applied` / `verify.passed` são enviados via `VLAB_URL`.

Versões pinadas: Terraform `1.15.8` (`terraform_wrapper: false`, já que quem chama o binário é o Terragrunt), Terragrunt `v1.1.2` (binário baixado da release e validado contra o `SHA256SUMS` oficial), `localstack/localstack:2026.07.2`. Terragrunt v1.x usa `--working-dir` (em vez do antigo `-chdir`) e `--non-interactive`/`TG_NON_INTERACTIVE` para rodar sem prompts em CI.

O job `apply` conta os recursos criados (`s3api list-buckets` + `dynamodb list-tables` + `sqs list-queues` + `ec2 describe-vpcs` filtrado por `tag:Name=vlab-*`, que exclui a VPC default do LocalStack) e subtrai 1 do total pelo bucket `vlab-tfstate` do backend, que não é recurso da demo.

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
