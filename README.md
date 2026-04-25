# pipelines

Repositório central de workflows reutilizáveis GitHub Actions para todos os produtos da organização.

## Visão Geral

Cada produto referencia os workflows deste repositório via `workflow_call` — nenhuma lógica de CI/CD é duplicada entre projetos.

## Documentação

- [Arquitetura — Geral](docs/architecture.md) — padrões comuns a todos os tipos de pipeline
- [Arquitetura — Aplicações](docs/architecture-application.md) — apps containerizadas em Kubernetes
- `docs/architecture-databases.md` — em breve
- `docs/architecture-queues.md` — em breve
- `docs/architecture-topics.md` — em breve

## Workflows Disponíveis

| Workflow | Descrição |
|---|---|
| `ci.yml` | Build Docker + SonarQube + Semgrep + Trivy + testes |
| `build-push.yml` | Push da imagem para GHCR |
| `deploy.yml` | Deploy Blue-Green + Canary + Testes Sintéticos |

## Como Usar em um Produto

```yaml
# meu-produto/.github/workflows/deploy-prod.yml
jobs:
  deploy:
    uses: muri1o/pipelines/.github/workflows/deploy.yml@main
    with:
      environment: prod
      image: ghcr.io/muri1o/meu-produto
    secrets: inherit
```

## Contratos Obrigatórios

Todo app que usa estas pipelines deve ter:

- `Dockerfile` na raiz
- `GET /health` retornando `200`
- Configuração 100% via variáveis de ambiente
- Porta definida via env var `PORT`
- Diretório `environments/` com `dev.yaml` e `prod.yaml`
