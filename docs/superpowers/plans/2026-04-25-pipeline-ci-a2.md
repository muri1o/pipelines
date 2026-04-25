# A2 — Stryker, Playwright e k6 na Pipeline CI — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Adicionar mutation testing (Stryker), API E2E tests (Playwright) e performance tests (k6) à pipeline CI reutilizável do `muri1o/pipelines`, ativados por inputs booleanos no caller (`muri1o/whatsapp-bot`).

**Architecture:** Os 3 tools são implementados primeiro no whatsapp-bot (config + testes) e verificados localmente antes de serem conectados ao CI. A pipeline `pipelines/ci.yml` ganha 3 novos inputs opcionais e 6 novos steps no final do job. O whatsapp-bot passa os inputs com `true` no seu `ci.yml`.

**Tech Stack:** Node.js 20, `@stryker-mutator/core@^8`, `@stryker-mutator/node-test-runner@^8`, `@playwright/test@^1.44`, k6 (instalado via apt no runner), GitHub Actions.

---

## File Structure

```
muri1o/whatsapp-bot/
  stryker.config.json              ← NEW: config do Stryker (mutate session.js)
  playwright.config.js             ← NEW: config do Playwright (webServer + baseURL)
  test/e2e/api.test.js             ← NEW: 4 testes de API com Playwright
  test/performance/load.js         ← NEW: script k6 (20 VUs, 30s, /health)
  package.json                     ← MODIFY: adicionar 3 devDeps
  .github/workflows/ci.yml         ← MODIFY: passar enable-stryker, enable-playwright, k6-script

muri1o/pipelines/
  .github/workflows/ci.yml         ← MODIFY: 3 inputs + 6 steps novos
```

---

## Task 1: Instalar devDependencies e criar configuração do Stryker

**Repo:** `muri1o/whatsapp-bot`

**Files:**
- Modify: `package.json`
- Create: `stryker.config.json`

- [ ] **Step 1: Instalar devDependencies do Stryker e Playwright**

```bash
cd /Users/murilocarvalhodesouza/Documents/claude/whatsapp-bot
npm install --save-dev \
  @stryker-mutator/core@^8 \
  @stryker-mutator/node-test-runner@^8 \
  @playwright/test@^1.44
```

Expected: `added N packages` sem erros. O `package.json` e `package-lock.json` serão atualizados automaticamente.

- [ ] **Step 2: Verificar que as dependências foram instaladas**

```bash
npm ls @stryker-mutator/core @stryker-mutator/node-test-runner @playwright/test --depth=0
```

Expected:
```
whatsapp-bot@1.0.0 /Users/murilocarvalhodesouza/Documents/claude/whatsapp-bot
├── @playwright/test@1.x.x
├── @stryker-mutator/core@8.x.x
└── @stryker-mutator/node-test-runner@8.x.x
```

- [ ] **Step 3: Criar `stryker.config.json`**

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

- [ ] **Step 4: Adicionar `reports/` ao `.gitignore`**

Se o arquivo `.gitignore` não existir, criar. Se existir, acrescentar no final:

```
reports/
playwright-report/
k6-summary.json
```

- [ ] **Step 5: Rodar Stryker localmente para verificar**

```bash
npx stryker run
```

Expected (últimas linhas):
```
INFO All tests green! Starting mutation testing...
...
INFO Mutation testing complete.
INFO Ran X.XX tests per mutant on average.
INFO ----------|---------|----------|-----------|------------|----------|---------|
INFO File      | % score | # killed | # timeout | # survived | # no cov | # error |
INFO ----------|---------|----------|-----------|------------|----------|---------|
INFO All files |   XX.XX |       XX |         0 |          X |        0 |       0 |
INFO ----------|---------|----------|-----------|------------|----------|---------|
```

O score deve ser ≥ 70 para não falhar. Se for < 70, os testes unitários existentes em `test/health.test.js` não cobrem suficientemente `session.js` — nesse caso, ver a seção de troubleshooting abaixo.

**Troubleshooting Stryker:**
- Se `Error: Could not find test files` → adicionar `"spec": "test/**/*.test.js"` ao `stryker.config.json`
- Se score < 70 → adicionar mais testes em `test/health.test.js` que exercitem os branches de `session.js` (ex: testar `resetHistory`, testar o limite de MAX_TURNS)

- [ ] **Step 6: Commit**

```bash
git add package.json package-lock.json stryker.config.json .gitignore
git commit -m "feat(a2): add Stryker mutation testing config"
```

---

## Task 2: Criar configuração do Playwright e testes E2E de API

**Repo:** `muri1o/whatsapp-bot`

**Files:**
- Create: `playwright.config.js`
- Create: `test/e2e/api.test.js`

- [ ] **Step 1: Criar `playwright.config.js`**

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
    timeout: 15000,
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

- [ ] **Step 2: Criar diretório `test/e2e/`**

```bash
mkdir -p test/e2e
```

- [ ] **Step 3: Escrever os testes em `test/e2e/api.test.js`**

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

- [ ] **Step 4: Rodar os testes Playwright localmente**

```bash
npx playwright test
```

Expected:
```
Running 4 tests using 1 worker

  ✓  1 GET /health retorna 200 e status ok (XXXms)
  ✓  2 POST /webhook corpo vazio retorna 200 (XXms)
  ✓  3 POST /webhook de número não autorizado retorna 200 (XXms)
  ✓  4 GET /media com path traversal retorna 400 (XXms)

  4 passed (Xs)
```

**Troubleshooting Playwright:**
- Se `Error: browserType.launch` → os testes estão tentando subir um browser (não deveria ocorrer com `request` fixture). Verificar que nenhum teste usa `page` ou `browser`.
- Se `Error: connect ECONNREFUSED` → o `webServer` não subiu a tempo. Aumentar `timeout` em `playwright.config.js` para `30000`.
- Se o servidor falha ao subir por falta de `context.md` → criar um arquivo vazio: `touch context.md`.

- [ ] **Step 5: Commit**

```bash
git add playwright.config.js test/e2e/api.test.js
git commit -m "feat(a2): add Playwright API E2E tests"
```

---

## Task 3: Criar script k6 de performance

**Repo:** `muri1o/whatsapp-bot`

**Files:**
- Create: `test/performance/load.js`

- [ ] **Step 1: Criar diretório `test/performance/`**

```bash
mkdir -p test/performance
```

- [ ] **Step 2: Criar `test/performance/load.js`**

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

- [ ] **Step 3: Commit**

```bash
git add test/performance/load.js
git commit -m "feat(a2): add k6 performance script for /health"
```

---

## Task 4: Atualizar `whatsapp-bot/.github/workflows/ci.yml`

**Repo:** `muri1o/whatsapp-bot`

**Files:**
- Modify: `.github/workflows/ci.yml`

O arquivo atual é:

```yaml
name: CI

on:
  push:
    branches: [dev]
  pull_request:
    branches: [main]

jobs:
  ci:
    uses: muri1o/pipelines/.github/workflows/ci.yml@main
    with:
      image: muri1o/whatsapp-bot
      node-version: '20'
      test-command: npm test
    secrets: inherit
```

- [ ] **Step 1: Adicionar os 3 novos inputs ao bloco `with`**

Substituir o conteúdo do arquivo por:

```yaml
name: CI

on:
  push:
    branches: [dev]
  pull_request:
    branches: [main]

jobs:
  ci:
    uses: muri1o/pipelines/.github/workflows/ci.yml@main
    with:
      image: muri1o/whatsapp-bot
      node-version: '20'
      test-command: npm test
      enable-stryker: true
      enable-playwright: true
      k6-script: test/performance/load.js
    secrets: inherit
```

- [ ] **Step 2: Commit e push**

```bash
git add .github/workflows/ci.yml
git commit -m "feat(a2): enable Stryker, Playwright and k6 in CI"
git push origin dev
```

---

## Task 5: Atualizar `pipelines/.github/workflows/ci.yml`

**Repo:** `muri1o/pipelines`

**Files:**
- Modify: `.github/workflows/ci.yml`

- [ ] **Step 1: Adicionar os 3 novos inputs à seção `on.workflow_call.inputs`**

O arquivo atual termina a seção `inputs` com:

```yaml
      test-command:
        description: 'Command to run tests'
        required: false
        type: string
        default: 'npm test'
```

Adicionar logo após (ainda dentro de `inputs`):

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

- [ ] **Step 2: Adicionar os 6 novos steps após o step "Run tests"**

Localizar no arquivo:

```yaml
      - name: Run tests
        run: ${{ inputs.test-command }}
```

Adicionar logo após (mantendo a indentação de 6 espaços):

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

      - name: E2E tests (Playwright)
        if: inputs.enable-playwright
        run: npx playwright test
        env:
          TWILIO_ACCOUNT_SID: test
          TWILIO_AUTH_TOKEN: test
          TWILIO_WHATSAPP_NUMBER: '+15550000000'
          AUTHORIZED_NUMBER: '5511999999999'
          ANTHROPIC_API_KEY: test
          PORT: '3000'

      - name: Upload Playwright report
        if: inputs.enable-playwright && always()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report-${{ github.sha }}
          path: playwright-report/

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

- [ ] **Step 3: Verificar que o arquivo inteiro tem indentação correta**

```bash
cd /Users/murilocarvalhodesouza/Documents/claude/pipelines
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/ci.yml'))" && echo "YAML válido"
```

Expected: `YAML válido`

- [ ] **Step 4: Commit e push**

```bash
git add .github/workflows/ci.yml
git commit -m "feat(a2): add Stryker, Playwright and k6 steps to reusable ci.yml"
git push origin main
```

---

## Task 6: Verificar que o CI passa no GitHub Actions

**Repos:** ambos

- [ ] **Step 1: Aguardar o run do whatsapp-bot disparado pelo push da Task 4**

```bash
gh run list --repo muri1o/whatsapp-bot --limit 3
```

Se o run da Task 4 ainda não apareceu (push foi antes do pipelines estar atualizado), disparar um novo push vazio:

```bash
cd /Users/murilocarvalhodesouza/Documents/claude/whatsapp-bot
git commit --allow-empty -m "ci: re-trigger after pipelines a2 update"
git push origin dev
```

- [ ] **Step 2: Aguardar conclusão do run e verificar status de cada job**

```bash
gh run watch --repo muri1o/whatsapp-bot $(gh run list --repo muri1o/whatsapp-bot --limit 1 --json databaseId --jq '.[0].databaseId')
```

Expected: todos os steps verdes, incluindo:
```
✓ Mutation testing (Stryker)
✓ Upload mutation report
✓ E2E tests (Playwright)
✓ Upload Playwright report
✓ Install k6
✓ Performance tests (k6)       ← pode mostrar warning se score baixo, mas não falha
✓ Upload k6 summary
```

- [ ] **Step 3: Verificar que os artefatos foram gerados**

```bash
gh run view --repo muri1o/whatsapp-bot $(gh run list --repo muri1o/whatsapp-bot --limit 1 --json databaseId --jq '.[0].databaseId')
```

Expected: seção `ARTIFACTS` com:
- `stryker-report-<sha>`
- `playwright-report-<sha>`
- `k6-summary-<sha>`
- `sbom-<sha>.spdx.json` (já existia)

- [ ] **Step 4: Merge dev → main no whatsapp-bot**

```bash
cd /Users/murilocarvalhodesouza/Documents/claude/whatsapp-bot
git checkout main
git merge dev --no-edit
git push origin main
git checkout dev
```
