---
name: estrategia-qa
description: QA: estratégia de testes, pirâmide de testes, automação, BDD, testes de API, carga, performance, regressão
metadata:
  type: specialized
  category: quality
  agents: [deboaz-qa, deboaz-backend, deboaz-frontend]
---

# Skill: Estratégia de QA e Testes

## Responsabilidade e Escopo

Competência para definição de estratégia de qualidade, automação de testes em múltiplos níveis, validação de requisitos via BDD/Gherkin, testes de performance/carga, e prevenção de defeitos. Aplicável a backend (.NET), frontend (Vue.js) e mobile (.NET MAUI).

## Filosofia de Qualidade

> **Qualidade não é testar mais, é testar melhor — e prevenir defeitos antes que existam.**

- **Shift Left**: Qualidade desde requisitos (BDD) → design → código → pipeline
- **Pirâmide de Testes**: Muitos unitários → Alguns integração → Poucos E2E
- **Testes como Documentação Viva**: Gherkin = Requisitos = Testes de Aceitação
- **Qualidade Compartilhada**: Dev escreve unitários/integração; QA foca em estratégia, E2E, performance, exploratórios

## Pirâmide de Testes (Adaptada para Hexagonal)

```
                    ┌─────────────┐
                    │   E2E       │  ← Poucos (5-10%) - Fluxos críticos usuário
                    │  (Cypress/  │     Journeys completos, cross-browser
                    │  Playwright)│
               ┌────┴──────┬──────┴────┐
               │           │           │
         ┌─────┴────┐ ┌────┴────┐ ┌───┴────┐
         │ Contract │ │Component│ │  API   │  ← Integração (20-30%)
         │  Tests   │ │  Tests  │ │ Tests  │     Contratos API, DB real, Componentes Vue
         └──────────┘ └─────────┘ └────────┘
    ┌─────────────────────────────────────────────┐
    │           UNITÁRIOS (60-70%)                │  ← Muitos, rápidos, isolados
    │  Domain (entidades, VOs, services)          │     Domain puro, Use Cases (Ports mockados)
    │  Use Cases (application logic)              │     Stores, Composables, Utils (frontend)
    │  Value Objects, Validators                  │
    └─────────────────────────────────────────────┘
```

## 1. Testes de Aceitação (BDD/Gherkin → Automação)

### Origem: Requisitos em Markdown + Gherkin
```gherkin
# 1_Requisitos/autenticacao.md
Feature: Autenticação de usuário

  Rule: Credenciais válidas concedem acesso
  
  Background:
    Given que o usuário "joao@email.com" possui conta ativa
    And sua senha é "SenhaSegura123"

  Scenario: Autenticar com credenciais válidas
    When informar email e senha corretos
    Then deverá receber token de acesso
    And deverá ser redirecionado ao dashboard

  Scenario Outline: Tentativas de senha incorreta
    When informar email correto e senha "<senha>"
    Then deverá receber erro de credenciais inválidas
    And o contador de tentativas deve incrementar

    Examples:
      | senha           |
      | SenhaErrada1    |
      | senhafraca      |
      |                 |
```

### Implementação: SpecFlow / Reqnroll (.NET) ou Cucumber.js (Frontend)
```csharp
// Backend - SpecFlow/Reqnroll (tests/acceptance/AutenticacaoSteps.cs)
[Binding]
public class AutenticacaoSteps
{
    private readonly ApiTestContext _context;
    private LoginCommand _command;
    private Result<LoginResponse> _result;

    public AutenticacaoSteps(ApiTestContext context) => _context = context;

    [Given(@"que o usuário ""(.*)"" possui conta ativa")]
    public async Task GivenUsuarioPossuiContaAtiva(string email)
    {
        var usuario = UsuarioFactory.Criar(email, "SenhaSegura123", "João");
        await _context.UsuarioRepository.AdicionarAsync(usuario);
    }

    [When(@"informar email e senha corretos")]
    public async Task WhenInformarCredenciaisValidas()
    {
        _command = new LoginCommand("joao@email.com", "SenhaSegura123");
        _result = await _context.AutenticarUseCase.ExecuteAsync(_command);
    }

    [Then(@"deverá receber token de acesso")]
    public void ThenDeveReceberToken()
    {
        _result.IsSuccess.Should().BeTrue();
        _result.Value.AccessToken.Should().NotBeNullOrEmpty();
        _result.Value.RefreshToken.Should().NotBeNullOrEmpty();
    }
}
```

```typescript
// Frontend - Cucumber.js (tests/e2e/features/autenticacao.feature)
// Mesmo arquivo Gherkin! Step definitions em TypeScript
import { Given, When, Then } from '@cucumber/cucumber'
import { expect } from '@playwright/test'
import { test } from '@playwright/test'

let page: Page

Given('que o usuário {string} possui conta ativa', async (email: string) => {
  await api.setupUser({ email, password: 'SenhaSegura123', name: 'João' })
})

When('informar email e senha corretos', async () => {
  await page.goto('/login')
  await page.fill('[data-testid=email]', 'joao@email.com')
  await page.fill('[data-testid=password]', 'SenhaSegura123')
  await page.click('[data-testid=login-submit]')
})

Then('deverá receber token de acesso', async () => {
  await expect(page).toHaveURL('/dashboard')
  const token = await page.evaluate(() => localStorage.getItem('accessToken'))
  expect(token).toBeTruthy()
})
```

## 2. Testes Unitários

### Backend (.NET + xUnit + Moq/NSubstitute)
```csharp
// tests/unit/Domain/Entities/UsuarioTests.cs
public class UsuarioTests
{
    [Fact]
    public void Criar_ComEmailInvalido_DeveLancarExcecao()
    {
        // Act & Assert
        var act = () => Usuario.Criar("email-invalido", "Senha123", "João");
        act.Should().Throw<ArgumentException>().WithMessage("*email*");
    }

    [Fact]
    public void Autenticar_ComSenhaCorreta_DeveRetornarVerdadeiro()
    {
        // Arrange
        var usuario = Usuario.Criar("joao@email.com", "SenhaSegura123", "João");
        
        // Act
        var resultado = usuario.VerificarSenha("SenhaSegura123");
        
        // Assert
        resultado.Should().BeTrue();
    }
}

// tests/unit/Application/UseCases/CriarPedidoUseCaseTests.cs
public class CriarPedidoUseCaseTests
{
    private readonly Mock<IPedidoRepository> _pedidoRepo = new();
    private readonly Mock<IProdutoRepository> _produtoRepo = new();
    private readonly Mock<IUnitOfWork> _uow = new();
    private readonly Mock<IDomainEventPublisher> _events = new();
    private readonly CriarPedidoUseCase _sut;

    public CriarPedidoUseCaseTests()
    {
        _sut = new CriarPedidoUseCase(_pedidoRepo.Object, _produtoRepo.Object, _uow.Object, _events.Object);
    }

    [Fact]
    public async Task ExecuteAsync_ComDadosValidos_DeveCriarPedidoEPublicarEvento()
    {
        // Arrange
        var command = new CriarPedidoCommand(ClienteId: "usr-1", Itens: [new("prod-1", 2, 50.00m)]);
        var produto = ProdutoFactory.Criar("prod-1", "Produto Teste", 50.00m);
        _produtoRepo.Setup(x => x.ObterPorIdAsync("prod-1", It.IsAny<CancellationToken>())).ReturnsAsync(produto);

        // Act
        var result = await _sut.ExecuteAsync(command, CancellationToken.None);

        // Assert
        result.IsSuccess.Should().BeTrue();
        _pedidoRepo.Verify(x => x.AdicionarAsync(It.IsAny<Pedido>(), It.IsAny<CancellationToken>()), Times.Once);
        _events.Verify(x => x.PublishAsync(It.IsAny<PedidoCriadoEvent>(), It.IsAny<CancellationToken>()), Times.Once);
    }
}
```

### Frontend (Vitest + Vue Test Utils)
```typescript
// tests/unit/stores/pedidoStore.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { setActivePinia, createPinia } from 'pinia'
import { usePedidoStore } from '@/features/pedidos/stores/pedidoStore'
import { pedidosApi } from '@/features/pedidos/api/pedidosApi'

vi.mock('@/features/pedidos/api/pedidosApi')

describe('usePedidoStore', () => {
  beforeEach(() => setActivePinia(createPinia()))

  it('deve carregar pedidos e atualizar state', async () => {
    const store = usePedidoStore()
    const mockPedidos = [{ id: '1', clienteNome: 'João', total: 100, status: 'confirmado', criadoEm: new Date().toISOString() }]
    
    vi.mocked(pedidosApi.listar).mockResolvedValue({
      items: mockPedidos, page: 1, limit: 20, total: 1, totalPages: 1
    })

    await store.carregarPedidos()

    expect(store.pedidos).toHaveLength(1)
    expect(store.loading).toBe(false)
  })
})
```

## 3. Testes de Integração

### Backend (Testcontainers - MongoDB/Redis reais)
```csharp
// tests/integration/Infrastructure/PedidoMongoRepositoryTests.cs
public class PedidoMongoRepositoryIntegrationTests : IAsyncLifetime
{
    private readonly MongoDbContainer _mongo = new MongoDbContainer("mongo:7");
    private IMongoDatabase _database = null!;
    private PedidoMongoRepository _repository = null!;

    public async Task InitializeAsync()
    {
        await _mongo.StartAsync();
        var client = new MongoClient(_mongo.GetConnectionString());
        _database = client.GetDatabase("test");
        var config = new MapperConfiguration(cfg => cfg.AddProfile<PedidoMappingProfile>());
        _repository = new PedidoMongoRepository(_database, config.CreateMapper());
    }

    [Fact]
    public async Task AdicionarAsync_DevePersistirERecuperarPedido()
    {
        var pedido = PedidoFactory.CriarNovo("usr-1", [new ItemPedidoData("prod-1", 2, 50.00m)]);
        
        await _repository.AdicionarAsync(pedido);
        var recuperado = await _repository.ObterPorIdAsync(pedido.Id);

        recuperado.Should().NotBeNull();
        recuperado!.Id.Should().Be(pedido.Id);
        recuperado.Itens.Should().HaveCount(1);
    }

    public async Task DisposeAsync() => await _mongo.DisposeAsync();
}
```

### API Integration (WebApplicationFactory)
```csharp
// tests/integration/Api/PedidosApiTests.cs
public class PedidosApiTests : IClassFixture<CustomWebApplicationFactory<Program>>
{
    private readonly HttpClient _client;
    private readonly CustomWebApplicationFactory<Program> _factory;

    public PedidosApiTests(CustomWebApplicationFactory<Program> factory)
    {
        _factory = factory;
        _client = factory.CreateClient();
        _client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", _factory.GetTestToken());
    }

    [Fact]
    public async Task POST_api_pedidos_ComDadosValidos_DeveRetornar201()
    {
        var command = new CriarPedidoCommand("usr-1", [new("prod-1", 2, 50.00m)]);
        var response = await _client.PostAsJsonAsync("/api/pedidos", command);

        response.StatusCode.Should().Be(HttpStatusCode.Created);
        var result = await response.Content.ReadFromJsonAsync<CriarPedidoResponse>();
        result!.Id.Should().NotBeEmpty();
    }
}
```

## 4. Testes de Contrato (Consumer-Driven Contracts)

```yaml
# tests/contracts/pedidos-consumer.yml (Pact)
consumer: "Arteris.Frontend"
provider: "Arteris.Backend"
interactions:
  - description: "Obter pedido por ID"
    request:
      method: GET
      path: "/api/pedidos/ped-123"
      headers:
        Authorization: "Bearer ${token}"
    response:
      status: 200
      headers:
        Content-Type: "application/json"
      body:
        id: "ped-123"
        clienteNome: "João"
        total: 250.00
        status: "confirmado"
```

## 5. Testes E2E (Playwright/Cypress)

```typescript
// tests/e2e/pedidos.spec.ts
import { test, expect } from '@playwright/test'

test.describe('Fluxo de Pedidos', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/login')
    await page.fill('[data-testid=email]', 'vendedor@arteris.com')
    await page.fill('[data-testid=password]', 'Senha123')
    await page.click('[data-testid=login-submit]')
    await expect(page).toHaveURL('/dashboard')
  })

  test('Criar pedido completo - caminho feliz', async ({ page }) => {
    // Navegar
    await page.click('[data-testid=nav-pedidos]')
    await page.click('[data-testid=btn-novo-pedido]')
    
    // Selecionar cliente
    await page.fill('[data-testid=cliente-search]', 'João')
    await page.click('[data-testid=cliente-option-usr-1]')
    
    // Adicionar itens
    await page.fill('[data-testid=produto-search]', 'Notebook')
    await page.click('[data-testid=produto-option-prod-1]')
    await page.fill('[data-testid=quantidade]', '2')
    await page.click('[data-testid=btn-adicionar-item]')
    
    // Revisar e confirmar
    await expect(page.locator('[data-testid=pedido-total]')).toContainText('R$ 5.000,00')
    await page.click('[data-testid=btn-confirmar]')
    
    // Sucesso
    await expect(page.locator('[data-testid=toast-success]')).toContainText('Pedido criado')
    await expect(page).toHaveURL(/\/pedidos\/ped-\w+/)
  })
})
```

## 6. Testes de Carga e Performance (k6)

```javascript
// tests/load/pedidos-load.js
import http from 'k6/http'
import { check, sleep } from 'k6'
import { Rate, Trend } from 'k6/metrics'

const errorRate = new Rate('errors')
const requestDuration = Trend('request_duration')

export const options = {
  stages: [
    { duration: '2m', target: 50 },   // Ramp up
    { duration: '5m', target: 50 },   // Stay at 50 VUs
    { duration: '2m', target: 100 },  // Spike to 100
    { duration: '5m', target: 100 },  // Stay
    { duration: '2m', target: 0 },    // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'], // 95% < 500ms
    http_req_failed: ['rate<0.01'],   // Error rate < 1%
    errors: ['rate<0.01'],
  }
}

const BASE_URL = __ENV.BASE_URL || 'https://api.staging.arteris.com'
const TOKEN = __ENV.AUTH_TOKEN

export default function () {
  const headers = {
    'Authorization': `Bearer ${TOKEN}`,
    'Content-Type': 'application/json'
  }

  // Cenário 1: Listar pedidos (read-heavy)
  let res = http.get(`${BASE_URL}/api/pedidos?page=1&limit=20`, { headers })
  check(res, { 'listar status 200': (r) => r.status === 200 }) || errorRate.add(1)
  requestDuration.add(res.timings.duration)

  sleep(Math.random() * 2 + 1)

  // Cenário 2: Criar pedido (write)
  if (Math.random() < 0.3) {
    const payload = JSON.stringify({
      clienteId: `usr-${Math.floor(Math.random() * 1000)}`,
      itens: [{ produtoId: 'prod-1', quantidade: 1, precoUnitario: 100 }]
    })
    res = http.post(`${BASE_URL}/api/pedidos`, payload, { headers })
    check(res, { 'criar status 201': (r) => r.status === 201 }) || errorRate.add(1)
    requestDuration.add(res.timings.duration)
  }
}
```

## 7. Testes de Regressão e Exploratórios

### Regressão Automatizada
- **Suite crítica**: Executa em todo PR (unit + integration + contract + E2E smoke)
- **Suite completa**: Executa nightly / pre-release (todos E2E, carga, performance)
- **Flaky test management**: Quarentena, análise de causa raiz, correção ou remoção

### Testes Exploratórios (Session-Based)
```
Charter: "Explorar fluxo de cancelamento de pedido com itens em diferentes status"
Timebox: 60 min
Tester: QA
Notes:
  - Cancelar pedido "confirmado" → OK, estorno automático
  - Cancelar pedido "enviado" → Bloqueado, requer autorização gerente
  - Cancelar pedido "entregue" → Não permitido, deve abrir devolução
  - Edge: Pedido com item "kit" (composto) → Cancelamento parcial?
Bugs found: ARTERIS-342 (cancelamento kit não tratado)
```

## 8. Análise de Cobertura e Métricas

### Metas de Cobertura
| Camada | Meta | Ferramenta |
|---|---|---|
| Domain (entidades, VOs, services) | 95%+ | coverlet / vitest coverage |
| Use Cases / Application | 90%+ | coverlet / vitest coverage |
| Adapters (repositories, external) | 80%+ | coverlet (integration tests) |
| Frontend (stores, composables, utils) | 85%+ | vitest coverage |
| Frontend (components) | 70%+ | vitest + Vue Test Utils |
| **Global** | **80%+** | Merged report |

### Quality Gates (Pipeline)
```yaml
# .github/workflows/quality.yml
quality-gates:
  - name: "Unit Tests Pass"
    condition: all unit tests pass
  - name: "Integration Tests Pass"
    condition: all integration tests pass
  - name: "Coverage Threshold"
    condition: coverage >= 80% (global), 90% (domain)
  - name: "No Flaky Tests"
    condition: 0 flaky in last 10 runs
  - name: "E2E Smoke Pass"
    condition: critical paths pass
  - name: "Performance Baseline"
    condition: p95 < 500ms, error rate < 1%
  - name: "Accessibility"
    condition: axe-core 0 critical violations
```

## 9. Estratégia por Camada

| Camada | Responsável | Tipos de Teste | Ferramentas |
|---|---|---|---|
| **Domain** | Backend Dev | Unitário (entidades, VOs, domain services, events) | xUnit, Moq |
| **Use Cases** | Backend Dev | Unitário (Ports Out mockados) | xUnit, Moq |
| **Adapters** | Backend Dev | Integração (Testcontainers MongoDB, Redis, HTTP) | xUnit, Testcontainers |
| **API Contratos** | Backend + QA | Contract Tests (Pact) | PactNet |
| **API E2E** | QA + Backend | Integration (WebApplicationFactory) | xUnit, WebApplicationFactory |
| **Frontend Stores/Composables** | Frontend Dev | Unitário | Vitest |
| **Frontend Components** | Frontend Dev | Component | Vitest, Vue Test Utils |
| **Frontend E2E** | QA + Frontend | E2E (journeys críticos) | Playwright |
| **Mobile** | Mobile Dev + QA | Unit, Integration, E2E | xUnit, Appium |
| **Performance/Carga** | QA | Load, Stress, Soak, Spike | k6 |
| **Segurança** | QA + Sec | SAST, DAST, SCA, Pen Test | SonarQube, OWASP ZAP, Trivy |

## Boas Práticas

1. **Testes como documentação** — nomes descrevem comportamento (`Should_ReturnError_When_EmailInvalid`)
2. **AAA Pattern** — Arrange, Act, Assert claramente separados
3. **Um assert por teste** (ou grupo coeso) — falha clara
4. **Dados de teste isolados** — factories, builders, não fixtures compartilhados
5. **Mock o que você possui** (Ports Out) — não mock tipos de terceiros
6. **Testcontainers para integração real** — não mock DB/cache
7. **E2E apenas para fluxos críticos** — não testar tudo via UI
8. **Contratos versionados** — Pact broker para consumer-driven contracts
9. **Pipeline rápido** — unit < 2min, integration < 5min, E2E smoke < 10min

## Critérios de Qualidade

- [ ] Pirâmide de testes respeitada (muitos unit, poucos E2E)
- [ ] Cobertura ≥ 80% global, ≥ 90% domain
- [ ] Testes de aceitação derivados de Gherkin (rastreabilidade)
- [ ] Testcontainers para integração com DB/cache reais
- [ ] Contract tests entre frontend/backend
- [ ] E2E cobre top 10 user journeys
- [ ] Load test baseline estabelecido e monitorado
- [ ] Quality gates no pipeline (blocking)
- [ ] Flaky tests < 1% ou em quarentena
- [ ] Relatórios de cobertura + tendência visíveis

## Referências

- `gherkin-bdd` — origem dos testes de aceitação
- `csharp-dotnet` — testes backend (.NET)
- `desenvolvimento-frontend` — testes frontend (Vue)
- `testes-mobile` — testes .NET MAUI
- `devops-azure` — pipeline, quality gates