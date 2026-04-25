# Arquitetura de Pipelines — Aplicações

> Pipeline para aplicações containerizadas em Kubernetes.
> Padrões gerais (multi-cloud, segurança, observabilidade) em [architecture.md](architecture.md).

---

## Stack

| Componente | Tecnologia |
|---|---|
| Containers | Docker |
| Orquestração | Kubernetes |
| Registry | GHCR (`ghcr.io`) — neutro de cloud |
| Helm | Deploy de aplicações no cluster |
| CI/CD | GitHub Actions com workflows reutilizáveis em `muri1o/pipelines` |

## Ambientes

| Ambiente | Trigger |
|---|---|
| `dev` | Push na branch `dev` |
| `prod` | Tag semântica `v*.*.*` em `main` |

---

## Pipeline CI

Executada a cada push em `dev`. Qualquer falha bloqueia o deploy.

```
build Docker
  → secret scanning       (truffleHog — credenciais no código)
  → dependency scanning   (Dependabot — dependências vulneráveis)
  → SonarQube Community   (qualidade + cobertura de código)
  → code coverage gate    (bloqueia se cobertura < threshold)
  → Semgrep               (SAST — vulnerabilidades no código)
  → Trivy                 (CVEs na imagem Docker)
  → SBOM                  (Syft — inventário de dependências)
  → OPA                   (policy as code — limites CPU/memória obrigatórios)
  → testes unitários
  → mutation testing      (Stryker — qualidade dos testes)
  → API contract testing  (Pact — contratos entre serviços)
  → E2E tests             (Playwright)
  → performance tests     (k6)
```

| Ferramenta | Propósito | Custo |
|---|---|---|
| truffleHog | Secret scanning | Gratuito |
| Dependabot | Dependency scanning | Gratuito |
| SonarQube Community | Qualidade + cobertura | Gratuito (self-hosted) |
| Semgrep | SAST | Gratuito (até 10 devs) |
| Trivy | CVEs na imagem | Gratuito |
| Syft | SBOM | Gratuito |
| OPA | Policy as code | Gratuito |
| Stryker | Mutation testing | Gratuito |
| Pact | API contract testing | Gratuito |
| Playwright | E2E tests | Gratuito |
| k6 | Performance tests | Gratuito |

---

## Pipeline de Deploy — Produção

```
semantic version bump automático
  → changelog gerado a partir dos commits
  → green deploy (blue continua em 100%)
  → Nginx roteia 10% para green (canary)
  → testes sintéticos contra green
     ✅ passou → migra 100% para green → remove blue
     ❌ falhou → rollback automático: remove green, blue permanece
```

---

## Estrutura Kubernetes

```
cluster/
  namespace: dev
    └─ [app]-deployment
    └─ [app]-service
    └─ [app]-ingress

  namespace: prod
    └─ [app]-blue-deployment   ← versão atual
    └─ [app]-green-deployment  ← nova versão (durante deploy)
    └─ [app]-service
    └─ [app]-ingress           ← split de tráfego via Nginx weight
```

- **Ingress Controller**: Nginx (open source, qualquer cloud)
- **Secrets**: Kubernetes Secrets gerenciados pelo Vault — nunca commitados no Git

---

## Estrutura do Repositório `muri1o/pipelines`

```
.github/
  workflows/
    ci.yml            → pipeline CI completa
    build-push.yml    → push da imagem para GHCR
    deploy.yml        → blue-green + canary + testes sintéticos
    sync-cloud.yml    → sincroniza cloud/region do GitHub Projects → repo
```

---

## Estrutura por Repositório de Produto

```
meu-produto/
  Dockerfile
  environments/
    dev.yaml          ← fonte de verdade para dev
    prod.yaml         ← fonte de verdade para prod
  helm/
    Chart.yaml
    values.yaml       ← valores padrão
    templates/
      deployment.yaml
      service.yaml
      ingress.yaml
  .github/
    workflows/
      ci.yml          ← chama pipelines/ci.yml
      deploy-dev.yml  ← chama pipelines/deploy.yml (env: dev)
      deploy-prod.yml ← chama pipelines/deploy.yml (env: prod)
```

### Formato de `environments/prod.yaml`

```yaml
cloud: aws          # aws | gcp | azure (sincronizado via GitHub Projects)
region: us-east-1

app:
  port: 3000
  replicas: 2
  memory: 512Mi
  cpu: 250m

image:
  registry: ghcr.io/muri1o
  tag: ""             # preenchido pela pipeline

env:
  NODE_ENV: production
  LOG_LEVEL: info

synthetic_tests:
  - path: /health
    expected_status: 200
    max_response_ms: 500
  - path: /api/status
    expected_status: 200
    max_response_ms: 1000
```

---

## Contratos Obrigatórios por App

| Contrato | Detalhe |
|---|---|
| `Dockerfile` na raiz | Obrigatório |
| Health check | `GET /health` retorna `200` |
| Configuração via env vars | Nenhuma config em arquivo — tudo via variável de ambiente |
| Porta | Definida via env var `PORT` |
