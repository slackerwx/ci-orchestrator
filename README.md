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
