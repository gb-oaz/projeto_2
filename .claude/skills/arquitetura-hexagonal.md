---
name: arquitetura-hexagonal
description: Arquitetura Hexagonal (Ports and Adapters): estrutura, ports in/out, use cases, domain, adapters
metadata:
  type: specialized
  category: architecture
  agents: [deboaz-arc, deboaz-backend]
---

# Skill: Arquitetura Hexagonal (Ports and Adapters)

## Responsabilidade e Escopo

Define a estrutura arquitetural do backend baseada em Arquitetura Hexagonal (Ports and Adapters). Garante isolamento do domínio, testabilidade, e substitutibilidade de tecnologias de infraestrutura.

## Conceitos Fundamentais

### Princípio Central
> **O domínio não conhece a infraestrutura. A infraestrutura conhece o domínio.**

Dependências apontam **para dentro** (em direção ao domínio).

### Camadas (de fora para dentro)

```
┌─────────────────────────────────────────────────────────────┐
│                      DRIVERS (Externos)                     │
│  Controllers  │  CLI  │  Event Handlers  │  Message Consumers │
└──────────────────────────┬────────────────────────────────────┘
                           │ Chamam Ports In
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                     PORTS IN (Interfaces)                   │
│  IAutenticarUsuarioUseCase  │  ICriarPedidoUseCase          │
│  IObterProdutoQuery         │  IListarPedidosQuery          │
└──────────────────────────┬────────────────────────────────────┘
                           │ Implementados por Use Cases
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                      USE CASES (Application)                │
│  AutenticarUsuarioUseCase  │  CriarPedidoUseCase            │
│  ObterProdutoQuery         │  ListarPedidosQuery            │
│  - Orquestração            │  - Validação de entrada        │
│  - Transações              │  - Coordenação de domínio      │
└──────────────────────────┬────────────────────────────────────┘
                           │ Usam Domain + Ports Out
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                        DOMAIN (Core)                        │
│  Entities: Usuario, Pedido, Produto                         │
│  Value Objects: Email, CPF, Money, Endereco                 │
│  Aggregates: Pedido (root), Usuario (root)                  │
│  Domain Services: CalculoFrete, ValidacaoCpf                │
│  Domain Events: PedidoCriado, UsuarioAutenticado            │
│  Repository Interfaces: IUsuarioRepository, IPedidoRepository│
└──────────────────────────┬────────────────────────────────────┘
                           │ Definem Ports Out
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                     PORTS OUT (Interfaces)                  │
│  IUsuarioRepository    │  IPedidoRepository                 │
│  ITokenService         │  IEmailService                     │
│  ICacheService         │  IExternalApiClient                │
└──────────────────────────┬────────────────────────────────────┘
                           │ Implementados por Adapters
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                      ADAPTERS (Infraestrutura)              │
│  UsuarioMongoAdapter  │  PedidoMongoAdapter                │
│  JwtTokenAdapter      │  SmtpEmailAdapter                  │
│  RedisCacheAdapter    │  HttpExternalApiAdapter            │
│  EF Core Migrations   │  Health Checks                     │
└─────────────────────────────────────────────────────────────┘
```

## Estrutura de Pastas (Referência)

```
src/
├── Domain/
│   ├── Entities/
│   ├── ValueObjects/
│   ├── Aggregates/
│   ├── Services/
│   ├── Events/
│   ├── Interfaces/          ← Ports Out (Repository interfaces)
│   └── Exceptions/
├── Application/
│   ├── UseCases/
│   │   ├── Commands/        ← Write operations
│   │   └── Queries/         ← Read operations (CQRS leve)
│   ├── Ports/               ← Ports In (Use Case interfaces)
│   ├── DTOs/
│   └── Validators/
├── Infrastructure/
│   ├── Persistence/
│   │   ├── MongoDB/
│   │   │   ├── Adapters/    ← Repository implementations
│   │   │   ├── Collections/
│   │   │   └── Mappings/
│   │   └── Redis/
│   │       └── Adapters/    ← Cache implementations
│   ├── ExternalServices/
│   │   ├── Adapters/        ← HTTP clients, etc.
│   │   └── DTOs/
│   ├── Security/
│   │   ├── JwtTokenAdapter/
│   │   └── PasswordHasher/
│   └── CrossCutting/
│       ├── Logging/
│       ├── Middleware/
│       └── HealthChecks/
└── Presentation/
    ├── Controllers/         ← Drivers (ASP.NET Core)
    ├── MinimalApis/
    ├── Middleware/
    └── Filters/
```

## Responsabilidades por Camada

### Domain (Núcleo Puro)
- **ZERO dependências externas** (nem .NET BCL além de tipos primitivos)
- Regras de negócio invariantes
- Entidades com identidade e ciclo de vida
- Value Objects imutáveis, validados na criação
- Aggregates garantem consistência transacional
- Domain Services para lógica multi-entidade
- Domain Events para side effects desacoplados
- Repository **interfaces** (Ports Out)

### Application (Use Cases)
- Orquestração de fluxos de aplicação
- Validação de entrada (DTOs → Domain)
- Gestão de transações (Unit of Work)
- Coordenação: Domain + Ports Out
- **Não** contêm regras de negócio complexas (estão no Domain)
- Retornam DTOs ou Result<T> para a camada de apresentação

### Ports In (Interfaces de Entrada)
- Contratos para drivers (Controllers, CLI, Events)
- Um interface por Use Case (Command ou Query)
- Nomes: `I<Action><Entity>UseCase` ou `I<Action><Entity>Query`

### Ports Out (Interfaces de Saída)
- Contratos para infraestrutura
- Repository interfaces no Domain
- Serviços externos: `ITokenService`, `IEmailService`, `ICacheService`
- Nomes: `I<Recurso>Repository` ou `I<Servico>Service`

### Adapters (Implementações)
- **Persistence Adapters**: Implementam `IRepository` usando MongoDB, EF Core, etc.
- **External Service Adapters**: Implementam `IExternalService` usando HttpClient, SDKs
- **Security Adapters**: JWT, Hashing, etc.
- **Cross-cutting**: Logging, Metrics, HealthChecks

### Presentation (Drivers)
- Controllers / Minimal APIs
- Model binding → DTOs → Use Cases
- Tratamento de erros → ProblemDetails (RFC 7807)
- Serialização, versionamento, documentação (OpenAPI)

## Regras de Ouro

1. **Domain não referencia Application, Infrastructure, Presentation**
2. **Application referencia apenas Domain + Ports Out (interfaces)**
3. **Infrastructure referencia Domain + Application (interfaces)**
4. **Presentation referencia Application (interfaces) + DTOs**
5. **Dependency Injection** conecta Ports → Adapters na Composition Root

## Padrões de Implementação

### Use Case Pattern (Command)
```csharp
// Port In
public interface ICriarPedidoUseCase
{
    Task<Result<CriarPedidoResponse>> ExecuteAsync(CriarPedidoCommand command);
}

// Use Case
public class CriarPedidoUseCase : ICriarPedidoUseCase
{
    private readonly IPedidoRepository _pedidoRepo;
    private readonly IProdutoRepository _produtoRepo;
    private readonly IUnitOfWork _uow;
    private readonly IDomainEventPublisher _events;

    public async Task<Result<CriarPedidoResponse>> ExecuteAsync(CriarPedidoCommand cmd)
    {
        // 1. Validar comando
        // 2. Carregar agregados via Ports Out
        // 3. Invocar método no Aggregate Root (domain logic)
        // 4. Persistir via Port Out
        // 5. Publicar Domain Events
        // 6. Retornar DTO de resposta
    }
}
```

### Repository Pattern (Port Out + Adapter)
```csharp
// Domain - Port Out
public interface IPedidoRepository
{
    Task<Pedido?> ObterPorIdAsync(PedidoId id);
    Task AdicionarAsync(Pedido pedido);
    Task AtualizarAsync(Pedido pedido);
}

// Infrastructure - Adapter
public class PedidoMongoAdapter : IPedidoRepository
{
    private readonly IMongoCollection<PedidoDocument> _collection;
    
    public async Task<Pedido?> ObterPorIdAsync(PedidoId id)
    {
        var doc = await _collection.Find(x => x.Id == id.Value).FirstOrDefaultAsync();
        return doc?.ToDomain();
    }
    // ... mapeamento Document ↔ Domain
}
```

## Decisões Arquiteturais (ADRs) Esperadas

- ADR-001: Arquitetura Hexagonal como padrão base
- ADR-002: CQRS leve (Commands/Queries separados em Use Cases)
- ADR-003: MongoDB como persistence principal
- ADR-004: Redis para cache distribuído
- ADR-005: JWT para autenticação stateless
- ADR-006: ProblemDetails (RFC 7807) para erros de API
- ADR-007: Domain Events para integração assíncrona

## Boas Práticas

1. **Um Use Case = Uma responsabilidade** (Single Responsibility)
2. **DTOs nas fronteiras** (entrada/saída), Domain interno
3. **Result<T> pattern** para falhas esperadas (não exceções de controle de fluxo)
4. **Validação em duas camadas**: DTO (formato) + Domain (regras de negócio)
5. **Transações no Use Case** (Unit of Work), não no Repository
6. **Domain Events** para side effects (email, notificações, integração)
7. **Mapeamento explícito** Document ↔ Domain nos Adapters (não AutoMapper cego)
8. **Testes de Use Case** com Ports Out mockados (rápidos, isolados)

## Critérios de Qualidade

- [ ] Domain compila sem referenciar Infrastructure/Presentation
- [ ] Use Cases têm testes unitários com Ports Out mockados
- [ ] Adapters implementam interfaces do Domain/Application
- [ ] DI configurado na Composition Root (Program.cs)
- [ ] Nenhum `new` de implementação concreta no Domain/Application
- [ ] Domain Events publicados, não processados no mesmo Use Case
- [ ] ProblemDetails padronizado para erros de API
- [ ] Health checks nos Adapters críticos

## Referências

- `engenharia-software` — SOLID, Clean Architecture, DDD base
- `ddd-conceitos` — modelagem tática detalhada
- `csharp-dotnet` — implementação em .NET
- `testes-backend` — estratégia de testes para hexagonal