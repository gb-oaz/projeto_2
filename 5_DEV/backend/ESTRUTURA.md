# Estrutura do Backend - Arquitetura Hexagonal (Ports & Adapters)

## Visão Geral

Esta documentação descreve a estrutura de pastas do projeto backend seguindo o padrão **Arquitetura Hexagonal** (Ports & Adapters), garantindo separação clara entre o **Core** (Domain + Application) e os **Adapters** (Inbound/Outbound) + **Infrastructure**.

## Estrutura de Diretórios

```
5_DEV/backend/
├── src/
│   ├── Domain/                    # Núcleo puro - ZERO dependências externas
│   │   ├── Entities/              # Entidades de domínio
│   │   ├── ValueObjects/          # Value Objects
│   │   ├── Aggregates/            # Agregados (raízes de agregação)
│   │   ├── Events/                # Domain Events
│   │   ├── Services/              # Domain Services
│   │   ├── Repositories/          # Interfaces de Repository (Ports Out)
│   │   └── Exceptions/            # Exceções de domínio
│   │
│   ├── Application/               # Casos de uso / Application Services
│   │   ├── Commands/              # Commands (Write side - CQRS)
│   │   │   ├── <Action>Command.cs
│   │   │   └── Handlers/
│   │   ├── Queries/               # Queries (Read side - CQRS)
│   │   │   ├── <Action>Query.cs
│   │   │   └── Handlers/
│   │   ├── Ports/                 # Ports In (Interfaces de Use Cases)
│   │   │   ├── Inbound/
│   │   │   │   ├── I<Action>UseCase.cs
│   │   │   │   └── I<Action>Query.cs
│   │   │   └── Outbound/          # Interfaces para Adapters (Ports Out)
│   │   │       ├── I<Resource>Repository.cs
│   │   │       └── I<ExternalService>.cs
│   │   ├── DTOs/                  # Data Transfer Objects
│   │   ├── Validators/            # Validação (FluentValidation)
│   │   └── Common/                # Compartilhado da Application
│   │
│   ├── Adapters/                  # Adapters (Implementações dos Ports)
│   │   ├── Inbound/               # Driving Adapters (Entrada)
│   │   │   ├── Api/               # Controllers / Minimal APIs
│   │   │   ├── SignalR/           # Hubs SignalR
│   │   │   └── BackgroundJobs/    # Hosted Services / Workers
│   │   │
│   │   └── Outbound/              # Driven Adapters (Saída)
│   │       ├── Persistence/       # Repositories MongoDB
│   │       ├── Cache/             # Redis Cache
│   │       ├── Messaging/         # Event Bus / Message Broker
│   │       ├── ExternalApis/      # HTTP Clients para APIs externas
│   │       ├── Auth/              # JWT Token Service
│   │       ├── Email/             # Email Service
│   │       └── Storage/           # Blob Storage
│   │
│   ├── Infrastructure/            # Configuração técnica cross-cutting
│   │   ├── DependencyInjection/   # Composition Root / DI Setup
│   │   ├── Middleware/            # Middlewares ASP.NET Core
│   │   ├── Configuration/         # Options / Settings
│   │   ├── HealthChecks/          # Health Checks customizados
│   │   ├── Logging/               # Structured Logging setup
│   │   ├── Observability/         # OpenTelemetry, Metrics, Traces
│   │   └── Security/              # AuthZ, Policies, Rate Limiting
│   │
│   └── Shared/                    # Kernel compartilhado
│       └── Kernel/                # Building blocks comuns
│           ├── Primitives/        # Result, Entity, VO, AggregateRoot base
│           ├── Events/            # IDomainEvent, IEventHandler
│           ├── Specifications/    # Specification Pattern
│           ├── Guard/             # Guard Clauses
│           └── Extensions/        # Extension Methods
│
├── tests/                         # Testes (espelham estrutura do src)
│   ├── Unit/
│   │   ├── Domain/
│   │   ├── Application/
│   │   └── Shared/
│   ├── Integration/
│   │   ├── Adapters/
│   │   └── Infrastructure/
│   ├── Contract/
│   └── Architecture/              # Testes de arquitetura (NetArchTest)
│
├── ESTRUTURA.md                   # Este arquivo
├── Directory.Build.props          # Props compartilhados MSBuild
├── Directory.Build.targets        # Targets compartilhados MSBuild
├── global.json                    # SDK version pinning
├── nuget.config                   # NuGet sources
└── README.md                      # Documentação do projeto
```

## Princípios Arquiteturais

### 1. Regra de Dependência (Dependency Rule)
- **Domain** → **NENHUMA** dependência externa
- **Application** → Depende apenas de **Domain** + **Ports Out** (interfaces)
- **Adapters** → Implementam **Ports** (Application/Domain)
- **Infrastructure** → Configura DI, Middleware, cross-cutting concerns

### 2. Ports & Adapters

| Camada | Responsabilidade | Exemplos |
|--------|------------------|----------|
| **Ports In** (Application) | Interfaces que expõem casos de uso | `ICriarClienteUseCase`, `IObterClienteQuery` |
| **Ports Out** (Domain/Application) | Interfaces para infraestrutura | `IClienteRepository`, `IEmailService` |
| **Adapters Inbound** | Recebem requisições externas | Controllers, SignalR Hubs, Workers |
| **Adapters Outbound** | Implementam Ports Out | MongoRepository, RedisCache, HttpEmailService |

### 3. CQRS Leve
- **Commands** (Escrita): Validação, transação, domain events
- **Queries** (Leitura): Projeções otimizadas, paginação, DTOs
- Separação clara via pastas `Commands/` e `Queries/`

### 4. Domain Events
- Publicados dentro do Domain (Aggregate Roots)
- Processados por Handlers na Application
- Permitem integração assíncrona entre Bounded Contexts

## Convenções de Nomenclatura

| Elemento | Padrão | Exemplo |
|----------|--------|---------|
| Entities | PascalCase, singular | `Cliente`, `Pedido` |
| Value Objects | PascalCase, singular | `Email`, `Cpf`, `Endereco` |
| Aggregates | PascalCase, singular | `ClienteAggregate` |
| Domain Events | PascalCase + `Event` | `ClienteCriadoEvent` |
| Commands | PascalCase + `Command` | `CriarClienteCommand` |
| Queries | PascalCase + `Query` | `ObterClienteQuery` |
| Use Case Interfaces | `I` + Ação + `UseCase` | `ICriarClienteUseCase` |
| Query Interfaces | `I` + Ação + `Query` | `IObterClienteQuery` |
| Repository Interfaces | `I` + Aggregate + `Repository` | `IClienteRepository` |
| Adapters (Impl) | Nome + `Adapter` ou sufixo da tech | `MongoClienteRepository`, `RedisCacheService` |

## Configuração .NET 8

- **Target Framework**: `net8.0`
- **Nullable**: `enable` (treat warnings as errors)
- **ImplicitUsings**: `enable`
- **TreatWarningsAsErrors**: `true`
- **AnalysisLevel**: `latest-all`

## Testes

- **Unit**: Domain, Application, Shared (mock Ports Out)
- **Integration**: Adapters reais (Testcontainers: MongoDB, Redis)
- **Contract**: API contracts (Pact ou similar)
- **Architecture**: NetArchTest para validar Dependency Rule

## Referências

- ADR-001: Arquitetura Hexagonal como padrão base
- ADR-002: CQRS leve (Commands/Queries separados)
- ADR-003: MongoDB (Cosmos DB API) como persistence principal
- ADR-004: Redis (Azure Cache) para cache distribuído
- Clean Architecture (Robert C. Martin)
- Domain-Driven Design (Eric Evans / Vaughn Vernon)
- .NET 8 / ASP.NET Core Documentation