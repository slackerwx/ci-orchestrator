# ci-orchestrator

Reusable GitHub Actions workflow para o pipeline DevSecOps do VirtuaLab. Executa build, SAST (Semgrep), secret scanning (Gitleaks) e scan de vulnerabilidades (Trivy) em paralelo, depois empacota a imagem, gera SBOM e assina com Cosign (keyless, via OIDC). Resultados SARIF e eventos do ciclo de vida da imagem são enviados ao VirtuaLab via `VLAB_URL`.

Cada job carrega o nome do stage correspondente (`build`, `sast`, `secrets`, `vuln`, `package`) para casar com os stage ids esperados pelo VirtuaLab. O input `language` (`go` ou `node`) controla o setup do job `build`.

## Uso

```yaml
name: pipeline
on: [push]

jobs:
  devsecops:
    uses: slackerwx/ci-orchestrator/.github/workflows/devsecops.yaml@main
    with:
      language: go
    secrets:
      VLAB_URL: ${{ secrets.VLAB_URL }}
      VLAB_INGEST_TOKEN: ${{ secrets.VLAB_INGEST_TOKEN }}
```
