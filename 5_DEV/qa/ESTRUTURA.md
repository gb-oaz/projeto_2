# Estrutura de Pastas - QA/Automação

## Visão Geral

Esta estrutura organiza os testes automatizados para **backend** (API, integração, contrato) e **frontend** (E2E, component, visual regression) de forma separada e escalável.

```
5_DEV/qa/
├── backend/                    # Testes de backend (API, Integração, Contrato)
│   ├── specs/                  # Especificações de testes (Gherkin/Feature files)
│   │   ├── acceptance/         # Testes de aceitação BDD (Reqnroll/SpecFlow)
│   │   ├── integration/        # Testes de integração (Testcontainers)
│   │   ├── contract/           # Testes de contrato (Pact consumer/provider)
│   │   └── api/                # Testes de API REST (RestSharp/FluentAssertions)
│   ├── fixtures/               # Dados de teste estáticos e builders
│   │   ├── data/               # JSON/XML/CSV de massa de teste
│   │   └── builders/           # Test Data Builders (pattern Builder)
│   ├── helpers/                # Utilitários e helpers de teste
│   │   ├── api/                # Helpers para chamadas HTTP, auth, assertions
│   │   ├── db/                 # Helpers MongoDB, Redis (Testcontainers)
│   │   ├── messaging/          # Helpers para eventos, message bus
│   │   └── auth/               # Helpers JWT, tokens, roles
│   ├── config/                 # Configurações de ambiente de teste
│   │   ├── environments/       # Config por ambiente (local, ci, staging)
│   │   └── testsettings.json   # Config xUnit/Reqnroll
│   ├── reports/                # Relatórios de execução (Allure, HTML, TRX)
│   └── scripts/                # Scripts de automação (setup, teardown, CI)
│
├── frontend/                   # Testes de frontend (E2E, Component, Visual)
│   ├── specs/                  # Especificações de testes
│   │   ├── e2e/                # Testes E2E (Playwright) - jornadas críticas
│   │   ├── component/          # Testes de componente (Vitest + Vue Test Utils)
│   │   ├── visual/             # Visual regression (Playwright + pixelmatch)
│   │   └── accessibility/      # Testes a11y (axe-core + Playwright)
│   ├── page-objects/           # Page Object Model (POM)
│   │   ├── pages/              # Page Objects por tela/fluxo
│   │   ├── components/         # Component Objects (reutilizáveis)
│   │   └── fragments/          # Fragment Objects (partes de página)
│   ├── fixtures/               # Dados de teste para frontend
│   │   ├── data/               # Mock data, API responses, test users
│   │   └── builders/           # Builders para entidades de UI
│   ├── helpers/                # Utilitários de teste frontend
│   │   ├── auth/               # Login, session storage, tokens
│   │   ├── api/                # Mock servers (MSW), API helpers
│   │   ├── selectors/          # Seletores robustos, data-testid helpers
│   │   └── wait/               # Wait utilities, conditions
│   ├── config/                 # Configurações Playwright/Vitest
│   │   ├── playwright.config.ts
│   │   ├── vitest.config.ts
│   │   └── environments/       # Config por ambiente
│   ├── reports/                # Relatórios (Playwright HTML, Allure, screenshots)
│   └── scripts/                # Scripts de automação frontend
│
├── shared/                     # Compartilhado entre backend e frontend
│   ├── contracts/              # Contratos Pact (consumer-driven)
│   ├── test-data/              # Dados de teste compartilhados
│   └── utils/                  # Utilitários comuns
│
├── docs/                       # Documentação QA
│   ├── strategy/               # Estratégia de testes, pirâmide, risk-based
│   ├── runbooks/               # Runbooks de execução, troubleshooting
│   └── metrics/                # Dashboards, baselines, quality gates
│
├── scripts/                    # Scripts globais de QA
│   ├── ci/                     # Scripts para pipeline CI/CD
│   ├── local/                  # Scripts para execução local
│   └── performance/            # Scripts k6, artillery
│
└── ESTRUTURA.md                # Este arquivo
```

## Detalhamento por Camada

### Backend (`backend/`)

| Pasta | Responsabilidade | Tecnologias |
|-------|------------------|-------------|
| `specs/acceptance` | BDD - Cenários Gherkin → Automação | Reqnroll, SpecFlow, xUnit |
| `specs/integration` | Integração com DB/Message Broker reais | Testcontainers (MongoDB, Redis), xUnit |
| `specs/contract` | Contract Testing Consumer/Provider | PactNet, Pact |
| `specs/api` | Testes de API REST diretos | RestSharp, FluentAssertions, xUnit |
| `fixtures/builders` | Test Data Builder Pattern | C#, Bogus, AutoFixture |
| `helpers/api` | HttpClient wrappers, auth, assertions | HttpClient, Polly (retry) |
| `helpers/db` | MongoDB/Redis containers, seed/cleanup | Testcontainers, MongoDB.Driver |
| `config/environments` | appsettings.Test.json, variáveis | .NET Configuration |

### Frontend (`frontend/`)

| Pasta | Responsabilidade | Tecnologias |
|-------|------------------|-------------|
| `specs/e2e` | Jornadas críticas usuário | Playwright, TypeScript |
| `specs/component` | Componentes Vue isolados | Vitest, Vue Test Utils, @vue/test-utils |
| `specs/visual` | Visual Regression Testing | Playwright + pixelmatch/resemblejs |
| `specs/accessibility` | WCAG 2.1 AA compliance | axe-core, Playwright |
| `page-objects/pages` | Page Objects por rota/tela | TypeScript, Playwright |
| `page-objects/components` | Component Objects reutilizáveis | TypeScript |
| `fixtures/data` | Mock API responses, test users | JSON, TypeScript, MSW handlers |
| `helpers/selectors` | data-testid, seletores robustos | Playwright locators |
| `config` | playwright.config, vitest.config | TypeScript, Playwright, Vitest |

### Shared (`shared/`)

| Pasta | Responsabilidade |
|-------|------------------|
| `contracts` | Arquivos `.pact` gerados (consumer → provider) |
| `test-data` | Dados mestres compartilhados (ex: usuários, produtos) |
| `utils` | Funções utilitárias usadas por ambos os lados |

## Convenções de Nomenclatura

### Arquivos de Teste
- **Backend**: `*Tests.cs` (xUnit), `*.feature` (Gherkin), `*Steps.cs` (Reqnroll)
- **Frontend E2E**: `*.spec.ts` (Playwright)
- **Frontend Component**: `*.spec.ts` (Vitest)
- **Visual**: `*.visual.spec.ts`
- **Accessibility**: `*.a11y.spec.ts`

### Page Objects
- `pages/<feature>/<PageName>Page.ts` - Ex: `pages/pedidos/ListaPedidosPage.ts`
- `components/<ComponentName>Component.ts` - Ex: `components/ModalConfirmacaoComponent.ts`
- `fragments/<FragmentName>Fragment.ts` - Ex: `fragments/HeaderFragment.ts`

### Fixtures/Builders
- `builders/<Entity>Builder.cs/ts` - Ex: `builders/PedidoBuilder.cs`
- `data/<contexto>.json` - Ex: `data/pedidos-validos.json`

## Quality Gates (Pipeline)

```yaml
# Quality Gates obrigatórios no CI/CD
backend:
  - Unit Tests (xUnit)                    → coverage ≥ 90% domain
  - Integration Tests (Testcontainers)    → 100% pass
  - Contract Tests (Pact)                 → 100% pass
  - API Tests                             → 100% pass
  - Acceptance Tests (Reqnroll)           → 100% Gherkin covered

frontend:
  - Component Tests (Vitest)              → coverage ≥ 80%
  - E2E Smoke (Playwright)                → Top 5 journeys < 10 min
  - E2E Full (Nightly)                    → All journeys
  - Visual Regression                     → 0 diffs não aprovados
  - Accessibility (axe-core)              → 0 critical violations

shared:
  - Contract Tests (Pact verification)    → Provider passes consumer contracts
  - Performance (k6)                      → p95 < baseline, error rate < 1%
  - Security (CodeQL, Trivy)              → 0 high/critical
```

## Execução Local

```bash
# Backend
cd 5_DEV/qa/backend
dotnet test                              # Todos os testes
dotnet test --filter "Category=Integration"
dotnet test --filter "Category=Acceptance"

# Frontend
cd 5_DEV/qa/frontend
npm run test:unit                        # Vitest component
npm run test:e2e                         # Playwright E2E
npm run test:visual                      # Visual regression
npm run test:a11y                        # Accessibility

# Performance
cd 5_DEV/qa/scripts/performance
k6 run pedidos-load.js
```

## Rastreabilidade (Requirements Coverage)

| Requisito (Gherkin) | Testes de Aceitação | Testes Integração | Testes E2E | Status |
|---------------------|---------------------|-------------------|------------|--------|
| `1_Requisitos/pedidos.md` | `backend/specs/acceptance/pedidos.feature` | `backend/specs/integration/PedidosIntegrationTests.cs` | `frontend/specs/e2e/pedidos.spec.ts` | ✅ |

---

*Documento versionado junto com o código. Atualize ao alterar a estrutura.*