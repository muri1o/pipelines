# A2 — Stryker, Playwright e k6 na Pipeline CI

> **For agentic workers:** Use superpowers:subagent-driven-development or superpowers:executing-plans to implement this spec.

**Goal:** Adicionar mutation testing (Stryker), API E2E tests (Playwright) e performance tests (k6) à pipeline CI reutilizável, com opt-in via inputs booleanos no caller.

**Scope:** `muri1o/pipelines` (ci.yml) + `muri1o/whatsapp-bot` (config, testes, workflow).

**Out of scope:** SonarQube (aguarda cluster — card no board), Pact (aguarda múltiplos serviços — card no board).

---

## Arquitetura

A pipeline reutilizável `pipelines/ci.yml` recebe três novos inputs opcionais. Os steps correspondentes rodam somente quando o input está habilitado. Os artefatos (relatórios) são sempre salvos mesmo quando há falha. O whatsapp-bot passa `true`/caminho para habilitar tudo.

---

## Alterações em `pipelines/.github/workflows/ci.yml`

### Novos inputs

```yaml
enable-stryker:
  description: 'Rodar mutation testing com Stryker'
  required: false
  type: boolean
  default: false

enable-playwright:
  description: 'Rodar testes E2E de API com Playwright'
  required: false
  type: boolean
  default: false

k6-script:
  description: 'Caminho para o script k6 (vazio = desabilitado)'
  required: false
  type: string
  default: ''
```

### Novos steps (após "Run tests")

**Mutation testing (Stryker)**
```yaml
- name: Mutation testing (Stryker)
  if: inputs.enable-stryker
  run: npx stryker run
- name: Upload mutation report
  if: inputs.enable-stryker && always()
  uses: actions/upload-artifact@v4
  with:
    name: stryker-report-${{ github.sha }}
    path: reports/mutation/
```
Bloqueia CI se o mutation score ficar abaixo do threshold definido em `stryker.config.json` (threshold: 70).

**E2E tests (Playwright)**
```yaml
- name: E2E tests (Playwright)
  if: inputs.enable-playwright
  run: npx playwright test
  env:
    TWILIO_ACCOUNT_SID: test
    TWILIO_AUTH_TOKEN: test
    TWILIO_WHATSAPP_NUMBER: '+15550000000'
    AUTHORIZED_NUMBER: '5511999999999'
    ANTHROPIC_API_KEY: test
    PORT: 3000
- name: Upload Playwright report
  if: inputs.enable-playwright && always()
  uses: actions/upload-artifact@v4
  with:
    name: playwright-report-${{ github.sha }}
    path: playwright-report/
```
O `playwright.config.js` da app sobe o servidor via `webServer` antes dos testes.

**Performance tests (k6)**
```yaml
- name: Install k6
  if: inputs.k6-script != ''
  run: |
    curl -fsSL https://dl.k6.io/key.gpg | sudo gpg --dearmor -o /usr/share/keyrings/k6-archive-keyring.gpg
    echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
    sudo apt-get update && sudo apt-get install k6 -y

- name: Performance tests (k6)
  if: inputs.k6-script != ''
  continue-on-error: true
  run: |
    node index.js &
    SERVER_PID=$!
    npx wait-on http://localhost:3000/health --timeout 15000
    k6 run --summary-export=k6-summary.json ${{ inputs.k6-script }}
    kill $SERVER_PID
  env:
    TWILIO_ACCOUNT_SID: test
    TWILIO_AUTH_TOKEN: test
    TWILIO_WHATSAPP_NUMBER: '+15550000000'
    AUTHORIZED_NUMBER: '5511999999999'
    ANTHROPIC_API_KEY: test
    PORT: '3000'
    BASE_URL: http://localhost:3000

- name: Upload k6 summary
  if: inputs.k6-script != ''
  uses: actions/upload-artifact@v4
  with:
    name: k6-summary-${{ github.sha }}
    path: k6-summary.json
```
Não bloqueia CI (`continue-on-error: true`). O step sobe o servidor, espera com `wait-on` (disponível via npx pois as deps já foram instaladas), roda o k6 e derruba o servidor — independente do Playwright.

---

## Arquivos novos em `muri1o/whatsapp-bot`

### `stryker.config.json`

```json
{
  "$schema": "./node_modules/@stryker-mutator/core/schema/stryker-schema.json",
  "testRunner": "node-test",
  "mutate": ["session.js"],
  "reporters": ["html", "json", "clear-text"],
  "htmlReporter": { "baseDir": "reports/mutation" },
  "jsonReporter": { "fileName": "reports/mutation/report.json" },
  "thresholds": { "high": 80, "low": 70, "break": 70 },
  "coverageAnalysis": "perTest",
  "disableTypeChecks": true
}
```

Muta apenas `session.js`. Threshold de corte (break): 70%.

### `playwright.config.js`

```javascript
const { defineConfig } = require('@playwright/test');

module.exports = defineConfig({
  testDir: './test/e2e',
  reporter: [['html', { open: 'never' }], ['list']],
  use: { baseURL: 'http://localhost:3000' },
  webServer: {
    command: 'node index.js',
    port: 3000,
    reuseExistingServer: false,
    env: {
      TWILIO_ACCOUNT_SID: 'test',
      TWILIO_AUTH_TOKEN: 'test',
      TWILIO_WHATSAPP_NUMBER: '+15550000000',
      AUTHORIZED_NUMBER: '5511999999999',
      ANTHROPIC_API_KEY: 'test',
      PORT: '3000',
    },
  },
});
```

### `test/e2e/api.test.js`

```javascript
const { test, expect } = require('@playwright/test');

test('GET /health retorna 200 e status ok', async ({ request }) => {
  const res = await request.get('/health');
  expect(res.status()).toBe(200);
  expect(await res.json()).toEqual({ status: 'ok' });
});

test('POST /webhook corpo vazio retorna 200', async ({ request }) => {
  const res = await request.post('/webhook', { form: {} });
  expect(res.status()).toBe(200);
});

test('POST /webhook de número não autorizado retorna 200', async ({ request }) => {
  const res = await request.post('/webhook', {
    form: { From: 'whatsapp:+5511000000000', Body: 'hello' },
  });
  expect(res.status()).toBe(200);
});

test('GET /media com path traversal retorna 400', async ({ request }) => {
  const res = await request.get('/media/../../etc/passwd');
  expect(res.status()).toBe(400);
});
```

### `test/performance/load.js`

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  vus: 20,
  duration: '30s',
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:3000';

export default function () {
  const res = http.get(`${BASE_URL}/health`);
  check(res, {
    'status 200': (r) => r.status === 200,
    'body ok': (r) => r.json('status') === 'ok',
  });
  sleep(1);
}
```

20 VUs por 30s no `/health`. Não tem threshold — só gera métricas.

### `package.json` — novos devDependencies

```json
"@stryker-mutator/core": "^8.0.0",
"@stryker-mutator/node-test-runner": "^8.0.0",
"@playwright/test": "^1.44.0"
```

### `whatsapp-bot/.github/workflows/ci.yml` — novos inputs

```yaml
jobs:
  ci:
    uses: muri1o/pipelines/.github/workflows/ci.yml@main
    with:
      image: muri1o/whatsapp-bot
      node-version: '20'
      enable-stryker: true
      enable-playwright: true
      k6-script: test/performance/load.js
    secrets: inherit
```

---

## Ordem de execução final no CI

```
checkout → setup node → npm ci
→ truffleHog → semgrep → docker build → trivy → sbom → opa → testes unitários
→ mutation testing (Stryker)     ← bloqueia se score < 70%
→ E2E tests (Playwright)         ← bloqueia se teste falhar
→ performance tests (k6)         ← nunca bloqueia, só artefato
```

---

## Considerações

- **k6 é autossuficiente:** sobe o servidor, roda o teste e o derruba no mesmo step. Não depende de `enable-playwright`.
- **Stryker é lento:** muta `session.js` (30 linhas) — tempo esperado < 2 min. Aceitável.
- **Playwright usa env vars dummy:** o servidor sobe com credenciais fictícias. Chamadas externas (Twilio, Anthropic) só ocorrem ao processar mensagens do `AUTHORIZED_NUMBER`, que nos testes nunca coincide.
