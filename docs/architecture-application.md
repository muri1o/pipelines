# Arquitetura de Pipelines — Kubernetes

> **Escopo:** Este documento define a pipeline para **aplicações containerizadas em Kubernetes**. Outros tipos de pipeline serão documentados separadamente:
> - `docs/pipeline-databases.md` — provisionamento e migração de bancos de dados
> - `docs/pipeline-queues.md` — filas e mensageria (SQS, Pub/Sub, Service Bus)
> - `docs/pipeline-topics.md` — tópicos de eventos (Kafka, SNS, EventGrid)

## Stack

| Componente | Tecnologia |
|---|---|
| Containers | Docker |
| Orquestração | Kubernetes |
| Registry | GHCR (`ghcr.io`) — neutro de cloud |
| IaC | Terraform (clusters) + Helm (apps) |
| CI/CD | GitHub Actions com workflows reutilizáveis em `muri1o/pipelines` |

## Ambientes

| Ambiente | Trigger |
|---|---|
| `dev` | Push na branch `dev` |
| `prod` | Tag semântica `v*.*.*` em `main` |

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
  → SBOM                  (inventário de dependências da imagem)
  → OPA                   (policy as code — limites de CPU/memória obrigatórios)
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
| SBOM (Syft) | Inventário de dependências | Gratuito |
| OPA | Policy as code | Gratuito |
| Stryker | Mutation testing | Gratuito |
| Pact | API contract testing | Gratuito |
| Playwright | E2E tests | Gratuito |
| k6 | Performance tests | Gratuito |

## Estratégia de Deploy em Produção

Pipeline combinada: Blue-Green + Canary + Testes Sintéticos.

```
semantic version bump automático
  → changelog gerado a partir dos commits
  → green deploy (blue continua em 100%)
  → Nginx roteia 10% para green (canary)
  → testes sintéticos contra green
     ✅ passou → migra 100% para green → remove blue
     ❌ falhou → rollback automático: remove green, blue permanece
```

## Atuação em Caso de Falha

Claude analisa o log automaticamente, identifica a causa raiz e envia diagnóstico + proposta de correção via WhatsApp. O fix só é aplicado após confirmação.

```
1. pipeline falha
2. Claude analisa o log e identifica a causa raiz
3. WhatsApp: "❌ [repo] falhou — [causa] — Aplicar correção? (sim/não)"
4. sim → Claude aplica o fix e re-dispara a pipeline
   não → usuário trata manualmente
```

## Estrutura do Repositório `muri1o/pipelines`

```
.github/
  workflows/
    ci.yml            → build + SonarQube + Semgrep + Trivy + testes
    build-push.yml    → push da imagem para GHCR
    deploy.yml        → blue-green + canary + testes sintéticos
```

Cada produto chama esses workflows via `workflow_call`, passando apenas os parâmetros específicos do app.

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
      ci.yml
      deploy-dev.yml
      deploy-prod.yml
```

Os arquivos em `environments/` são consumidos por:
- **GitHub Actions** — variáveis do workflow (registry, porta, tag)
- **Dockerfile** — build args via `--build-arg`
- **Helm** — valores via `-f environments/prod.yaml`

### Formato dos Arquivos de Ambiente

```yaml
app:
  port: 3000
  replicas: 2
  memory: 512Mi
  cpu: 250m

image:
  registry: ghcr.io/muri1o
  tag: ""             # preenchido pela pipeline com SHA ou tag semântica

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
- **Secrets**: Kubernetes Secrets — criados via pipeline, nunca commitados no Git

## Multi-Cloud

### Providers Suportados

AWS, GCP e Azure. Cada produto escolhe sua cloud de forma independente.

### Repositórios de Infraestrutura

```
muri1o/infra-aws    → módulos Terraform para EKS (AWS)
muri1o/infra-gcp    → módulos Terraform para GKE (GCP)
muri1o/infra-azure  → módulos Terraform para AKS (Azure)
```

Cada repositório é independente — sem código compartilhado entre clouds.

### Atribuição de Cloud por Produto

A cloud de cada produto é definida no GitHub Projects e sincronizada automaticamente para o repositório do produto.

```
1. usuário define no GitHub Projects:
   Cloud: AWS  |  Region: us-east-1

2. workflow em muri1o/pipelines detecta a mudança via webhook

3. workflow atualiza environments/prod.yaml no repo do produto:
   cloud: aws
   region: us-east-1

4. pipeline de deploy lê esses valores e seleciona
   as credenciais corretas do GitHub Secrets
```

### Campos Customizados no GitHub Projects

| Campo | Tipo | Valores |
|---|---|---|
| `Cloud` | Seleção única | AWS / GCP / Azure |
| `Region` | Texto | ex: `us-east-1`, `us-central1`, `eastus` |

### GitHub Secrets por Produto

| Secret | Descrição |
|---|---|
| `KUBECONFIG_AWS` | Acesso ao cluster EKS |
| `KUBECONFIG_GCP` | Acesso ao cluster GKE |
| `KUBECONFIG_AZR` | Acesso ao cluster AKS |

A pipeline seleciona automaticamente o secret correto com base no campo `cloud` do `environments/prod.yaml`.

### Formato Atualizado de `environments/prod.yaml`

```yaml
cloud: aws          # aws | gcp | azure
region: us-east-1

app:
  port: 3000
  replicas: 2
  memory: 512Mi
  cpu: 250m

image:
  registry: ghcr.io/muri1o
  tag: ""

env:
  NODE_ENV: production
  LOG_LEVEL: info

synthetic_tests:
  - path: /health
    expected_status: 200
    max_response_ms: 500
```

## Autenticação e Segurança Multi-Cloud

### Autenticação GitHub Actions → Cloud

Credenciais de longa duração armazenadas como GitHub Secrets, com permissão mínima (apenas deploy no cluster):

| Secret | Cloud |
|---|---|
| `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` | AWS → EKS |
| `GCP_SA_KEY` | GCP → GKE |
| `AZURE_CREDENTIALS` | Azure → AKS |

### Segredos das Aplicações — HashiCorp Vault

Vault auto-hospedado no cluster. Apps recebem segredos via Vault Agent sidecar — sem acoplamento direto ao Vault.

```
app → Vault Agent (sidecar) → Vault → secret injetado como env var
```

### Controle de Acesso no Cluster

| Camada | Tecnologia | Propósito |
|---|---|---|
| RBAC | Kubernetes nativo | Permissões por namespace (dev/prod isolados) |
| Network Policies | Kubernetes nativo | Controla comunicação entre pods |
| Service Mesh | Istio | mTLS — toda comunicação entre serviços criptografada |

Namespaces `dev` e `prod` completamente isolados.

## Observabilidade

- **Prometheus + Grafana** — métricas de CPU, memória, latência e erros por produto
- **Alertas via WhatsApp** — notificação automática quando métricas ultrapassam limites
- **Kubecost** — custo por namespace/produto em tempo real
- **Drift detection** — alerta se estado do cluster divergir do Git

## IaC — Infracost

Estimativa de custo gerada automaticamente a cada mudança nos repositórios Terraform (`infra-aws`, `infra-gcp`, `infra-azure`) antes de aplicar.

## Contratos Obrigatórios por App

Todo repositório de produto deve seguir estes contratos para funcionar nas pipelines sem customização:

| Contrato | Detalhe |
|---|---|
| `Dockerfile` na raiz | Obrigatório |
| Health check | `GET /health` retorna `200` |
| Configuração via env vars | Nenhuma config em arquivo — tudo via variável de ambiente |
| Porta | Definida via env var `PORT` |
