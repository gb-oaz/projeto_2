# Arteris - Monorepo

> **Documentação de execução** — Este arquivo contém apenas instruções de como rodar cada projeto em ambiente de desenvolvimento. Para detalhes de arquitetura, consulte os `ESTRUTURA.md` de cada subprojeto.

---

## 📁 Estrutura do Monorepo

```
projeto_2/
├── 0_Assets/          # Recursos reutilizáveis (marca, temas, diagramas)
├── 1_Requisitos/      # Requisitos do produto (.md versionado + pdf/)
├── 2_Diagramas/       # Diagramas de desenvolvimento (.svg versionado)
├── 3_Arquitetura/     # Diagramas de alto nível (.svg versionado)
├── 4_CI_CD/           # Pipelines e deploy
└── 5_DEV/             # Código fonte
    ├── backend/       # .NET 8 - Arquitetura Hexagonal
    ├── frontend/      # Vue 3 + TypeScript + Vite
    └── qa/            # Automação de testes (Backend + Frontend)
```

---

## 🔧 Backend — .NET 8 (Arquitetura Hexagonal)

**Localização:** `5_DEV/backend/`

### Pré-requisitos
- .NET 8 SDK
- Docker (para banco de dados, message broker, cache)
- IDE: Rider ou VS Code + C# Dev Kit

### Configuração inicial
```bash
cd 5_DEV/backend
dotnet restore
```

### Executar em desenvolvimento
```bash
# Via Docker Compose (recomendado - sobe DB, Redis, RabbitMQ, Kafka)
docker compose -f docker-compose.yml -f docker-compose.override.yml up -d

# Aplicar migrations (Entity Framework)
dotnet ef database update --project src/Arteris.Infrastructure/Arteris.Infrastructure.csproj --startup-project src/Arteris.Api/Arteris.Api.csproj

# Rodar a API
dotnet run --project src/Arteris.Api/Arteris.Api.csproj
```

A API estará disponível em:
- **HTTP:** `http://localhost:5000`
- **HTTPS:** `https://localhost:5001`
- **Swagger:** `https://localhost:5001/swagger`

### Testes
```bash
# Unitários + Integração
dotnet test

# Apenas unitários (Core)
dotnet test tests/Arteris.Core.Tests/

# Apenas integração (Adapters + Infrastructure)
dotnet test tests/Arteris.Adapters.Tests/ tests/Arteris.Infrastructure.Tests/

# Com cobertura
dotnet test --collect:"XPlat Code Coverage"
```

### Estrutura de pastas (resumo)
```
src/
├── Arteris.Core/           # Domain + Application (isolado)
│   ├── Domain/             # Entities, VOs, Aggregates, Events, Interfaces
│   └── Application/        # Commands, Queries, Handlers, DTOs, Validators
├── Arteris.Adapters/       # Ports & Adapters
│   ├── Inbound/            # REST, GraphQL, gRPC (driving)
│   └── Outbound/           # Persistence, Messaging, External, FileStorage (driven)
├── Arteris.Infrastructure/ # DI, Config, Middleware, Security, BackgroundJobs
└── Arteris.Api/            # Entry point (Program.cs, Middleware pipeline)
```

> Detalhes completos: [`5_DEV/backend/ESTRUTURA.md`](5_DEV/backend/ESTRUTURA.md)

---

## 🎨 Frontend — Vue 3 + TypeScript + Vite

**Localização:** `5_DEV/frontend/`

### Pré-requisitos
- Node.js 20+ (LTS)
- pnpm 9+ (package manager oficial do projeto)
- VS Code + Volar (Vue Official)

### Configuração inicial
```bash
cd 5_DEV/frontend
pnpm install
```

### Executar em desenvolvimento
```bash
# Dev server com HMR
pnpm dev

# Build de produção
pnpm build

# Preview do build
pnpm preview

# Lint + format
pnpm lint
pnpm format
```

A aplicação estará disponível em:
- **Local:** `http://localhost:3000`
- **Network:** `http://<seu-ip>:3000`

### Testes
```bash
# Unitários (Vitest)
pnpm test:unit

# Componentes (Vue Test Utils + Vitest)
pnpm test:component

# E2E (Playwright)
pnpm test:e2e

# E2E com UI
pnpm test:e2e:ui

# Visual Regression
pnpm test:visual

# Acessibilidade
pnpm test:a11y

# Todos os testes + cobertura
pnpm test:all
```

### Estrutura de pastas (resumo)
```
src/
├── app/              # Entry point (main.ts, App.vue, router, styles)
├── core/             # Shared logic (composables, services, utils, types, plugins)
├── shared/           # Componentes reutilizáveis (ui, layout, feedback, navigation, data-display)
├── features/         # Features por domínio (auth, dashboard, users, settings)
│   └── <feature>/    # components, pages, stores, services, types, composables
├── layouts/          # Layouts de página (Default, Auth, Admin, Empty)
├── pages/            # Páginas globais (Home, NotFound, Unauthorized, Maintenance)
└── assets/           # Imagens, ícones, fontes
```

> Detalhes completos: [`5_DEV/frontend/ESTRUTURA.md`](5_DEV/frontend/ESTRUTURA.md)

---

## 🧪 QA — Automação de Testes

**Localização:** `5_DEV/qa/`

### Pré-requisitos
- Node.js 20+ (para testes frontend)
- .NET 8 SDK (para testes backend)
- Docker (para ambientes de teste isolados)
- k6 / NBomber (para performance)
- Playwright browsers: `pnpm playwright install` (no frontend)

### Backend Tests (API, Integração, Contrato, Performance)
```bash
cd 5_DEV/qa/backend

# Configurar ambiente
cp config/environments/local.json.example config/environments/local.json
# Editar local.json com conexões reais

# Testes de API (REST/GraphQL)
dotnet test api/specs/integration/
dotnet test api/specs/contract/

# Testes de performance (k6)
k6 run performance/k6/scripts/load-test.js

# Testes de performance (NBomber)
dotnet run --project performance/nbomber/
```

### Frontend Tests (E2E, Component, Visual, A11y)
```bash
cd 5_DEV/qa/frontend

# Instalar dependências
pnpm install

# E2E (Playwright)
pnpm test:e2e

# Componentes (Vitest + Vue Test Utils)
pnpm test:component

# Visual Regression
pnpm test:visual

# Acessibilidade (axe-core)
pnpm test:a11y
```

### Scripts globais (CI/CD local)
```bash
cd 5_DEV/qa/scripts

# Setup completo do ambiente de teste
./local/setup-test-env.sh

# Rodar todos os testes (backend + frontend)
./ci/run-all-tests.sh

# Apenas backend
./ci/run-backend-tests.sh

# Apenas frontend
./ci/run-frontend-tests.sh

# Contratos (Pact)
./ci/run-contract-tests.sh

# Performance
./ci/run-performance-tests.sh

# Teardown
./local/teardown-test-env.sh
```

### Docker para testes isolados
```bash
cd 5_DEV/qa/docker

# Subir stack de testes
docker compose -f docker-compose.test.yml up -d

# Rodar testes dentro do container
docker compose -f docker-compose.test.yml run backend-tests
docker compose -f docker-compose.test.yml run frontend-tests

# Derrubar
docker compose -f docker-compose.test.yml down -v
```

### Estrutura de pastas (resumo)
```
qa/
├── backend/
│   ├── api/specs/{acceptance,integration,contract,api}
│   ├── contract/{provider,consumer}
│   ├── performance/{k6,nbomber}
│   └── database/
├── frontend/
│   ├── e2e/{specs,page-objects,components,fixtures,helpers,config,reports}
│   ├── component/{specs,fixtures,helpers}
│   ├── visual/{specs,baselines,diffs,config}
│   └── accessibility/
├── shared/{test-data,helpers,config,reporting}
├── scripts/{ci,local,performance}
└── docker/
```

> Detalhes completos: [`5_DEV/qa/ESTRUTURA.md`](5_DEV/qa/ESTRUTURA.md)

---

## 🐳 Docker Compose (Stack completo de desenvolvimento)

Na raiz do monorepo (`projeto_2/`):

```yaml
# docker-compose.yml (exemplo)
services:
  # Backend dependencies
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: arteris
      POSTGRES_USER: arteris
      POSTGRES_PASSWORD: arteris
    ports: ["5432:5432"]
    volumes: ["postgres_data:/var/lib/postgresql/data"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  rabbitmq:
    image: rabbitmq:3-management
    ports: ["5672:5672", "15672:15672"]

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    ports: ["9092:9092"]
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092

  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    ports: ["2181:2181"]

volumes:
  postgres_data:
```

```bash
# Subir tudo
docker compose up -d

# Ver logs
docker compose logs -f

# Derrubar
docker compose down -v
```

---

## 📋 Comandos rápidos (resumo)

| Projeto | Comando | Descrição |
|---------|---------|-----------|
| **Backend** | `dotnet run --project src/Arteris.Api` | Inicia API em dev |
| **Backend** | `dotnet test` | Roda todos os testes |
| **Frontend** | `pnpm dev` | Inicia dev server (Vite) |
| **Frontend** | `pnpm test:all` | Roda todos os testes |
| **QA Backend** | `./scripts/ci/run-backend-tests.sh` | Suite completa backend |
| **QA Frontend** | `./scripts/ci/run-frontend-tests.sh` | Suite completa frontend |
| **QA Todos** | `./scripts/ci/run-all-tests.sh` | Tudo (backend + frontend) |

---

## 🔗 Links úteis

- [Arquitetura Hexagonal - Documentação](5_DEV/backend/ESTRUTURA.md)
- [Frontend Component Architecture](5_DEV/frontend/ESTRUTURA.md)
- [QA Automation Strategy](5_DEV/qa/ESTRUTURA.md)
- [Requisitos do Produto](1_Requisitos/)
- [Diagramas de Desenvolvimento](2_Diagramas/)
- [Diagramas de Arquitetura](3_Arquitetura/)

---

**Última atualização:** 2026-11-10  
**Versão:** 1.0.0