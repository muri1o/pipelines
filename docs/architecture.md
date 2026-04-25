# Arquitetura de Pipelines — Padrões Gerais

Este documento define os padrões e decisões que se aplicam a **todos os tipos de pipeline** da organização. Cada tipo de pipeline tem seu próprio documento com as especificidades:

- [Aplicações](architecture-application.md) — apps containerizadas em Kubernetes
- `architecture-databases.md` — provisionamento e migração de bancos de dados (em breve)
- `architecture-queues.md` — filas de mensageria (SQS, Pub/Sub, Service Bus) (em breve)
- `architecture-topics.md` — tópicos de eventos (Kafka, SNS, EventGrid) (em breve)

---

## Multi-Cloud

### Providers Suportados

AWS, GCP e Azure. Cada produto escolhe sua cloud de forma independente — sem lock-in.

### Repositórios de Infraestrutura

```
muri1o/infra-aws    → módulos Terraform para recursos AWS
muri1o/infra-gcp    → módulos Terraform para recursos GCP
muri1o/infra-azure  → módulos Terraform para recursos Azure
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

4. pipeline seleciona as credenciais corretas com base nesses valores
```

### Campos Customizados no GitHub Projects

| Campo | Tipo | Valores |
|---|---|---|
| `Cloud` | Seleção única | AWS / GCP / Azure |
| `Region` | Texto | ex: `us-east-1`, `us-central1`, `eastus` |

---

## Autenticação

### GitHub Actions → Cloud

Credenciais de longa duração armazenadas como GitHub Secrets, com permissão mínima por operação:

| Secret | Cloud |
|---|---|
| `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` | AWS |
| `GCP_SA_KEY` | GCP |
| `AZURE_CREDENTIALS` | Azure |

### Segredos das Aplicações — HashiCorp Vault

Vault auto-hospedado no cluster. Apps recebem segredos via Vault Agent sidecar — sem acoplamento direto ao Vault na aplicação.

```
app → Vault Agent (sidecar) → Vault → secret injetado como env var
```

---

## Segurança

### Controle de Acesso

| Camada | Tecnologia | Propósito |
|---|---|---|
| RBAC | Kubernetes nativo | Permissões por namespace e tipo de recurso |
| Network Policies | Kubernetes nativo | Controla comunicação entre pods |
| Service Mesh | Istio | mTLS — comunicação entre serviços criptografada |

Namespaces de ambientes distintos são completamente isolados entre si.

---

## Observabilidade

Padrão aplicado a todos os produtos e tipos de recurso:

- **Prometheus + Grafana** — métricas de uso, latência e erros
- **Alertas via WhatsApp** — notificação automática quando métricas ultrapassam limites definidos
- **Kubecost** — custo por produto/namespace em tempo real
- **Drift detection** — alerta se o estado real divergir do que está no Git

---

## IaC — Estimativa de Custo

Infracost roda automaticamente a cada pull request nos repositórios de infraestrutura (`infra-aws`, `infra-gcp`, `infra-azure`), gerando estimativa de custo antes de qualquer mudança ser aplicada.

---

## Atuação em Caso de Falha

Padrão para todos os tipos de pipeline:

```
1. pipeline falha
2. Claude analisa o log e identifica a causa raiz
3. WhatsApp: "❌ [pipeline] falhou — [causa] — Aplicar correção? (sim/não)"
4. sim → Claude aplica o fix e re-dispara a pipeline
   não → usuário trata manualmente
```

---

## Convenções Gerais

### Estrutura de `environments/`

Todo repositório de produto tem um diretório `environments/` na raiz com arquivos por ambiente. Esses arquivos são a **fonte de verdade única** para configurações de ambiente — consumidos pela pipeline, pelo Dockerfile e pelas ferramentas de IaC.

```
environments/
  dev.yaml
  prod.yaml
```

Campos obrigatórios em todo arquivo de ambiente:

```yaml
cloud: aws          # aws | gcp | azure
region: us-east-1
```

### Nomenclatura de Arquivos de Arquitetura

```
docs/architecture.md                  ← padrões gerais (este arquivo)
docs/architecture-application.md      ← apps Kubernetes
docs/architecture-databases.md        ← bancos de dados
docs/architecture-queues.md           ← filas
docs/architecture-topics.md           ← tópicos de eventos
```
