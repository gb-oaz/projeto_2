---
name: testcontainers
description: Testcontainers: testes de integração com containers reais (MongoDB, Redis, PostgreSQL, etc.)
metadata:
  type: reference
  category: testing
  agents: [deboaz-backend, deboaz-qa]
---

# Skill: Testcontainers

## Responsabilidade e Escopo

Competência de referência para escrever testes de integração confiáveis usando containers reais (Testcontainers) para MongoDB, Redis, PostgreSQL, Kafka, Azure services (Azurite), etc. Substitui mocks de infraestrutura por instâncias reais.

## Por que Testcontainers?

| Abordagem | Prós | Contras |
|---|---|---|
| **Mocks (Moq/NSubstitute)** | Rápidos, isolados | Não testam queries reais, serialização, índices, transações |
| **In-memory (EF Core InMemory)** | Rápido | Não replica comportamento de banco real |
| **Testcontainers** | **Real DB, queries reais, transações, índices, change streams** | Mais lento (segundos), precisa Docker |

> **Regra**: Use Testcontainers para **Adapters (Repositories, External Services)**. Use mocks para **Use Cases (testar lógica de aplicação isolada)**.

## Setup Básico (.NET)

```xml
<!-- tests/integration/Arteris.IntegrationTests.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <IsPackable>false</IsPackable>
    <Nullable>enable</Nullable>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="xunit" Version="2.6.*" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.5.*" />
    <PackageReference Include="Testcontainers.MongoDb" Version="3.7.*" />
    <PackageReference Include="Testcontainers.Redis" Version="3.7.*" />
    <PackageReference Include="Testcontainers.Azurite" Version="3.7.*" />
    <PackageReference Include="MongoDB.Driver" Version="2.25.*" />
    <PackageReference Include="StackExchange.Redis" Version="2.7.*" />
    <PackageReference Include="Microsoft.Extensions.Configuration.Json" Version="8.0.*" />
  </ItemGroup>
</Project>
```

## Base Class para Testes de Integração

```csharp
// tests/integration/Infrastructure/IntegrationTestBase.cs
public abstract class IntegrationTestBase : IAsyncLifetime
{
    protected readonly IServiceProvider Services;
    protected readonly IMongoDatabase Database;
    protected readonly IConnectionMultiplexer Redis;
    protected readonly HttpClient HttpClient;

    private readonly MongoDbContainer _mongo = new MongoDbContainer("mongo:7")
        .WithDatabase("arteris_test")
        .WithUsername("test")
        .WithPassword("test")
        .WithCleanUp(true);

    private readonly RedisContainer _redis = new RedisContainer("redis:7-alpine")
        .WithCleanUp(true);

    protected IntegrationTestBase()
    {
        // Inicialização lazy - containers start no InitializeAsync
        var services = new ServiceCollection();
        ConfigureServices(services);
        Services = services.BuildServiceProvider();
        
        Database = Services.GetRequiredService<IMongoDatabase>();
        Redis = Services.GetRequiredService<IConnectionMultiplexer>();
        HttpClient = Services.GetRequiredService<HttpClient>();
    }

    protected virtual void ConfigureServices(IServiceCollection services)
    {
        // Configuração padrão - override em derived classes
        services.AddSingleton(_ => _mongo.GetConnectionString());
        services.AddSingleton(_ => _redis.GetConnectionString());
    }

    public async Task InitializeAsync()
    {
        await _mongo.StartAsync();
        await _redis.StartAsync();
        
        // Reconfigurar services com connection strings reais
        var services = new ServiceCollection();
        ConfigureServicesWithRealConnections(services);
        Services = services.BuildServiceProvider();
        
        Database = Services.GetRequiredService<IMongoDatabase>();
        Redis = Services.GetRequiredService<IConnectionMultiplexer>();
        HttpClient = Services.GetRequiredService<HttpClient>();
        
        // Setup indexes, seed data
        await SetupDatabaseAsync();
    }

    protected virtual void ConfigureServicesWithRealConnections(IServiceCollection services)
    {
        var mongoConn = _mongo.GetConnectionString();
        var redisConn = _redis.GetConnectionString();
        
        services.Configure<MongoSettings>(o => o.ConnectionString = mongoConn);
        services.AddSingleton<IMongoClient>(new MongoClient(mongoConn));
        services.AddScoped(sp => sp.GetRequiredService<IMongoClient>().GetDatabase("arteris_test"));
        
        services.AddSingleton<IConnectionMultiplexer>(ConnectionMultiplexer.Connect(redisConn));
        
        // Register application services (Use Cases, Repositories, etc.)
        services.AddApplication();
        services.AddInfrastructure(); // Usa connection strings acima
        
        // HttpClient para API tests
        services.AddHttpClient("test", c => c.BaseAddress = new Uri("http://localhost:8080"));
    }

    protected virtual Task SetupDatabaseAsync()
    {
        // Criar índices, seed data comum
        return Task.CompletedTask;
    }

    public async Task DisposeAsync()
    {
        await _mongo.DisposeAsync();
        await _redis.DisposeAsync();
        if (Services is IDisposable d) d.Dispose();
    }
}
```

## Testes de Repository (MongoDB)

```csharp
// tests/integration/Infrastructure/Persistence/PedidoMongoRepositoryTests.cs
public class PedidoMongoRepositoryTests : IntegrationTestBase
{
    private PedidoMongoRepository _repository = null!;
    private IMapper _mapper = null!;

    protected override void ConfigureServicesWithRealConnections(IServiceCollection services)
    {
        base.ConfigureServicesWithRealConnections(services);
        
        var config = new MapperConfiguration(cfg => cfg.AddProfile<PedidoMappingProfile>());
        services.AddSingleton(config.CreateMapper());
    }

    protected override async Task SetupDatabaseAsync()
    {
        _mapper = Services.GetRequiredService<IMapper>();
        _repository = new PedidoMongoRepository(Database, _mapper);
        
        // Limpar coleção antes de cada teste
        await Database.GetCollection<PedidoDocument>("pedidos").DeleteManyAsync(_ => true);
    }

    [Fact]
    public async Task AdicionarAsync_DevePersistirERecuperarPedidoComItens()
    {
        // Arrange
        var pedido = PedidoFactory.CriarNovo(
            clienteId: "usr-1",
            itens: [
                new ItemPedidoData("prod-1", 2, 1500.00m),
                new ItemPedidoData("prod-2", 1, 250.00m)
            ]
        );

        // Act
        await _repository.AdicionarAsync(pedido);
        var recuperado = await _repository.ObterPorIdAsync(pedido.Id);

        // Assert
        recuperado.Should().NotBeNull();
        recuperado!.Id.Should().Be(pedido.Id);
        recuperado.ClienteId.Should().Be(pedido.ClienteId);
        recuperado.Itens.Should().HaveCount(2);
        recuperado.Itens.Should().ContainEquivalentOf(pedido.Itens.First());
        recuperado.Totais.Total.Should().Be(3250.00m);
    }

    [Fact]
    public async Task AtualizarAsync_ComConcorrenciaOtimista_DeveFalharSeVersaoDiferente()
    {
        // Arrange
        var pedido = PedidoFactory.CriarNovo("usr-1", [new ItemPedidoData("prod-1", 1, 100m)]);
        await _repository.AdicionarAsync(pedido);
        
        // Simular leitura concorrente
        var pedido1 = await _repository.ObterPorIdAsync(pedido.Id);
        var pedido2 = await _repository.ObterPorIdAsync(pedido.Id);
        
        pedido1!.AdicionarItem(new ProdutoId("prod-2"), new Quantidade(1), new Preco(50m));
        await _repository.AtualizarAsync(pedido1); // Sucesso - versão 1 → 2
        
        pedido2!.AdicionarItem(new ProdutoId("prod-3"), new Quantidade(1), new Preco(75m));
        
        // Act & Assert
        var act = async () => await _repository.AtualizarAsync(pedido2); // Falha - versão 1 ≠ 2
        await act.Should().ThrowAsync<ConcurrencyException>();
    }

    [Fact]
    public async Task ObterPorClienteId_DeveUsarIndiceERetornarPaginado()
    {
        // Arrange - Criar 50 pedidos para cliente, 10 para outro
        var tasks = Enumerable.Range(1, 50).Select(i => 
            _repository.AdicionarAsync(PedidoFactory.CriarNovo("usr-1", [new ItemPedidoData($"prod-{i}", 1, 100m)])));
        await Task.WhenAll(tasks);
        await _repository.AdicionarAsync(PedidoFactory.CriarNovo("usr-2", [new ItemPedidoData("prod-x", 1, 100m)]));

        // Act
        var (pedidos, total) = await _repository.ObterPorClienteIdPaginadoAsync("usr-1", page: 2, limit: 10);

        // Assert
        pedidos.Should().HaveCount(10);
        total.Should().Be(50);
        pedidos.All(p => p.ClienteId.Value == "usr-1").Should().BeTrue();
    }
}
```

## Testes de API (WebApplicationFactory + Testcontainers)

```csharp
// tests/integration/Api/CustomWebApplicationFactory.cs
public class CustomWebApplicationFactory<TProgram> : WebApplicationFactory<TProgram> where TProgram : class
{
    private readonly MongoDbContainer _mongo = new MongoDbContainer("mongo:7").WithCleanUp(true);
    private readonly RedisContainer _redis = new RedisContainer("redis:7-alpine").WithCleanUp(true);

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        _mongo.StartAsync().GetAwaiter().GetResult();
        _redis.StartAsync().GetAwaiter().GetResult();

        builder.ConfigureServices(services =>
        {
            // Remover configuração original
            var mongoDescriptor = services.SingleOrDefault(d => d.ServiceType == typeof(IMongoClient));
            if (mongoDescriptor != null) services.Remove(mongoDescriptor);
            
            var redisDescriptor = services.SingleOrDefault(d => d.ServiceType == typeof(IConnectionMultiplexer));
            if (redisDescriptor != null) services.Remove(redisDescriptor);

            // Adicionar containers de teste
            services.AddSingleton<IMongoClient>(new MongoClient(_mongo.GetConnectionString()));
            services.AddScoped(sp => sp.GetRequiredService<IMongoClient>().GetDatabase("arteris_test"));
            
            services.AddSingleton<IConnectionMultiplexer>(ConnectionMultiplexer.Connect(_redis.GetConnectionString()));
            
            // Configurar JWT para testes
            services.Configure<JwtSettings>(o => 
            {
                o.SecretKey = "test-secret-key-minimum-32-characters-long";
                o.Issuer = "test";
                o.Audience = "test";
            });
        });

        builder.UseEnvironment("Testing");
    }

    public string GetTestToken(string role = "User")
    {
        var claims = new[]
        {
            new Claim(JwtRegisteredClaimNames.Sub, "test-user"),
            new Claim(ClaimTypes.Role, role)
        };
        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes("test-secret-key-minimum-32-characters-long"));
        var token = new JwtSecurityToken(claims: claims, signingCredentials: new SigningCredentials(key, SecurityAlgorithms.HmacSha256));
        return new JwtSecurityTokenHandler().WriteToken(token);
    }

    protected override void Dispose(bool disposing)
    {
        if (disposing)
        {
            _mongo.DisposeAsync().GetAwaiter().GetResult();
            _redis.DisposeAsync().GetAwaiter().GetResult();
        }
        base.Dispose(disposing);
    }
}

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
    public async Task POST_api_pedidos_ComDadosValidos_DeveRetornar201EPedidoCriado()
    {
        // Arrange
        var command = new CriarPedidoCommand(
            ClienteId: "usr-1",
            Itens: [new CriarPedidoItemCommand("prod-1", 2, 1500.00m)]
        );

        // Act
        var response = await _client.PostAsJsonAsync("/api/pedidos", command);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        response.Headers.Location.Should().NotBeNull();
        
        var result = await response.Content.ReadFromJsonAsync<CriarPedidoResponse>();
        result!.Id.Should().NotBeEmpty();
        result.ClienteId.Should().Be("usr-1");
        result.Total.Should().Be(3000.00m);
    }

    [Fact]
    public async Task GET_api_pedidos_{id}_ComIdExistente_DeveRetornar200EPedido()
    {
        // Arrange - Criar via API
        var createResponse = await _client.PostAsJsonAsync("/api/pedidos", new CriarPedidoCommand("usr-1", []));
        var created = await createResponse.Content.ReadFromJsonAsync<CriarPedidoResponse>();

        // Act
        var response = await _client.GetAsync($"/api/pedidos/{created!.Id}");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        var pedido = await response.Content.ReadFromJsonAsync<PedidoDetalhadoResponse>();
        pedido!.Id.Should().Be(created.Id);
    }

    [Fact]
    public async Task GET_api_pedidos_ComFiltroCliente_DeveRetornarApenasDoCliente()
    {
        // Arrange
        await _client.PostAsJsonAsync("/api/pedidos", new CriarPedidoCommand("usr-1", []));
        await _client.PostAsJsonAsync("/api/pedidos", new CriarPedidoCommand("usr-2", []));

        // Act
        var response = await _client.GetAsync("/api/pedidos?clienteId=usr-1");
        var result = await response.Content.ReadFromJsonAsync<PaginatedResponse<PedidoResumoResponse>>();

        // Assert
        result!.Items.Should().AllSatisfy(p => p.ClienteId.Should().Be("usr-1"));
    }
}
```

## Testcontainers no GitHub Actions

```yaml
# .github/workflows/integration-tests.yml
jobs:
  integration-tests:
    runs-on: ubuntu-latest
    # Testcontainers precisa Docker daemon
    services:
      docker:
        image: docker:dind
        options: --privileged
        env:
          DOCKER_HOST: tcp://localhost:2375
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with: { dotnet-version: '8.0.x' }
      
      - name: Run Integration Tests
        run: dotnet test tests/integration/Arteris.IntegrationTests --configuration Release
        env:
          DOCKER_HOST: tcp://localhost:2375
          TESTCONTAINERS_RYUK_DISABLED: true # Desabilita Ryuk em CI (usa docker:dind)
          TESTCONTAINERS_DOCKER_SOCKET_OVERRIDE: /var/run/docker.sock
```

## Padrões Avançados

### Módulos Customizados (Para serviços específicos)
```csharp
// Infrastructure/Testcontainers/KeycloakContainer.cs
public class KeycloakContainer : ContainerBuilder
{
    public KeycloakContainer() : base("quay.io/keycloak/keycloak:24.0")
    {
        WithPortBinding(8080, true)
        .WithEnvironment("KEYCLOAK_ADMIN", "admin")
        .WithEnvironment("KEYCLOAK_ADMIN_PASSWORD", "admin")
        .WithEnvironment("KC_DB", "dev-file")
        .WithCommand("start-dev")
        .WithWaitStrategy(Wait.ForUnixContainer().UntilHttpRequestIsSucceeded(r => r.ForPort(8080).ForPath("/health/ready")));
    }

    public string GetAuthServerUrl() => $"http://{Hostname}:{GetMappedPublicPort(8080)}";
}
```

### Parallelização Segura
```csharp
// xUnit collection fixture para compartilhar containers entre classes
[CollectionDefinition("IntegrationTests")]
public class IntegrationTestCollection : ICollectionFixture<SharedContainerFixture> { }

public class SharedContainerFixture : IAsyncLifetime
{
    public MongoDbContainer Mongo { get; } = new MongoDbContainer("mongo:7").WithCleanUp(true);
    public RedisContainer Redis { get; } = new RedisContainer("redis:7-alpine").WithCleanUp(true);

    public async Task InitializeAsync()
    {
        await Task.WhenAll(Mongo.StartAsync(), Redis.StartAsync());
    }

    public async Task DisposeAsync()
    {
        await Task.WhenAll(Mongo.DisposeAsync(), Redis.DisposeAsync());
    }
}

// Uso nas classes de teste
[Collection("IntegrationTests")]
public class PedidoRepositoryTests 
{
    private readonly SharedContainerFixture _fixture;
    public PedidoRepositoryTests(SharedContainerFixture fixture) => _fixture = fixture;
    
    // Usar _fixture.Mongo.GetConnectionString(), etc.
}
```

## Boas Práticas

1. **Collection Fixtures** — Compartilhar containers entre testes (inicialização única)
2. **CleanUp(true)** — Auto-remoção ao final (mas não confiar 100% em CI)
3. **TESTCONTAINERS_RYUK_DISABLED=true** — Em CI com docker:dind
4. **Banco por teste** — `WithDatabase("test_{guid}")` ou limpar coleções no `SetupAsync`
5. **Imagens fixas** — `mongo:7`, `redis:7-alpine` (não `latest`)
6. **Wait Strategies** — Aguardar health checks antes de rodar testes
6. **Resource limits** — `WithResourceLimits(cpu: 1, memoryMB: 1024)` em CI
7. **Parallelização** — xUnit roda collections em paralelo, classes sequencial

## Critérios de Qualidade

- [ ] Testes de integração usam containers reais (não mocks de DB)
- [ ] Collection fixtures compartilham containers (performance)
- [ ] Cleanup automático funcionando (zero containers órfãos)
- [ ] CI roda integration tests (docker:dind ou host Docker)
- [ ] Imagens versionadas (não `latest`)
- [ ] Wait strategies para health checks
- [ ] Testes isolados (limpeza de dados entre testes)
- [ ] Cobertura de Adapters (Repositories, External Services) > 80%

## Referências

- `csharp-dotnet` — Repository Adapter, WebApplicationFactory
- `estrategia-qa` — Pirâmide de testes, quality gates
- `mongodb` — MongoDB específico
- `redis` — Redis específico
- `devops-azure` — CI/CD pipeline integration