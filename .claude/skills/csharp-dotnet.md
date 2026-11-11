---
name: csharp-dotnet
description: C# e .NET: ASP.NET Core, Minimal APIs, DI, Middleware, MongoDB, Redis, testes
metadata:
  type: specialized
  category: backend
  agents: [deboaz-arc, deboaz-backend]
---

# Skill: C# e .NET (Backend)

## Responsabilidade e Escopo

Competência técnica para desenvolvimento backend em C#/.NET 8+ com ASP.NET Core. Foco em APIs REST, Minimal APIs, Arquitetura Hexagonal, MongoDB, Redis, autenticação, observabilidade e testes.

## Stack Principal

| Tecnologia | Versão | Uso |
|---|---|---|
| .NET | 8.0 LTS | Runtime e SDK |
| ASP.NET Core | 8.0 | Web framework |
| C# | 12 | Linguagem |
| MongoDB.Driver | 2.25+ | Persistência principal |
| StackExchange.Redis | 2.7+ | Cache distribuído |
| JWT Bearer | 8.0 | Autenticação stateless |
| FluentValidation | 11+ | Validação de DTOs/Commands |
| xUnit | 2.6+ | Testes unitários/integração |
| Moq / NSubstitute | Latest | Mocking |
| Testcontainers | 3.7+ | Testes de integração com containers reais |
| Serilog | 3.1+ | Structured logging |
| OpenTelemetry | 1.7+ | Observabilidade (metrics, traces, logs) |

## Padrões de Implementação

### 1. Minimal APIs (Preferido para APIs simples)
```csharp
// Program.cs - Composition Root
var builder = WebApplication.CreateBuilder(args);

// Services
builder.Services.AddApplication();      // Use Cases, Ports
builder.Services.AddInfrastructure();   // Adapters, MongoDB, Redis
builder.Services.AddPresentation();     // Controllers/Minimal APIs, Filters

var app = builder.Build();

app.UseMiddleware<ExceptionHandlingMiddleware>();
app.UseSerilogRequestLogging();
app.MapEndpoints(); // Extension method com todos os endpoints
app.Run();
```

```csharp
// Endpoints/AutenticacaoEndpoints.cs
public static class AutenticacaoEndpoints
{
    public static void MapAutenticacaoEndpoints(this WebApplication app)
    {
        var group = app.MapGroup("/api/auth")
            .WithTags("Autenticação")
            .ProducesProblem(StatusCodes.Status400BadRequest)
            .ProducesProblem(StatusCodes.Status401Unauthorized)
            .ProducesProblem(StatusCodes.Status500InternalServerError);

        group.MapPost("/login", async (
            LoginCommand command,
            IAutenticarUsuarioUseCase useCase,
            CancellationToken ct) =>
        {
            var result = await useCase.ExecuteAsync(command, ct);
            return result.IsSuccess 
                ? Results.Ok(result.Value) 
                : result.ToProblemDetails();
        })
        .WithName("Login")
        .WithSummary("Autentica usuário")
        .Produces<LoginResponse>(StatusCodes.Status200OK)
        .ProducesProblem(StatusCodes.Status401Unauthorized);
    }
}
```

### 2. Controllers (Para APIs complexas com convenções MVC)
```csharp
[ApiController]
[Route("api/[controller]")]
[Produces("application/json")]
public sealed class PedidosController : ControllerBase
{
    private readonly ICriarPedidoUseCase _criarUseCase;
    private readonly IObterPedidoQuery _obterQuery;
    private readonly IListarPedidosQuery _listarQuery;

    public PedidosController(
        ICriarPedidoUseCase criarUseCase,
        IObterPedidoQuery obterQuery,
        IListarPedidosQuery listarQuery)
    {
        _criarUseCase = criarUseCase;
        _obterQuery = obterQuery;
        _listarQuery = listarQuery;
    }

    [HttpPost]
    [ProducesResponseType<CriarPedidoResponse>(StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> Criar(
        [FromBody] CriarPedidoCommand command,
        CancellationToken ct)
    {
        var result = await _criarUseCase.ExecuteAsync(command, ct);
        return result.IsSuccess
            ? CreatedAtAction(nameof(Obter), new { id = result.Value.Id }, result.Value)
            : result.ToProblemDetails();
    }
}
```

### 3. Dependency Injection (Composition Root)
```csharp
// Extensions/ServiceCollectionExtensions.cs
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddApplication(this IServiceCollection services)
    {
        // Use Cases (Scoped - vida útil da request)
        services.Scan(scan => scan
            .FromAssemblyOf<ICriarPedidoUseCase>()
            .AddClasses(classes => classes.AssignableTo<IUseCase<,>>())
            .AsImplementedInterfaces()
            .WithScopedLifetime());

        // Queries
        services.Scan(scan => scan
            .FromAssemblyOf<IObterPedidoQuery>()
            .AddClasses(classes => classes.AssignableTo<IQuery<,>>())
            .AsImplementedInterfaces()
            .WithScopedLifetime());

        // Validators
        services.AddValidatorsFromAssemblyContaining<CriarPedidoCommandValidator>();
        
        // Domain Events
        services.AddScoped<IDomainEventPublisher, DomainEventPublisher>();
        
        return services;
    }

    public static IServiceCollection AddInfrastructure(this IServiceCollection services, IConfiguration config)
    {
        // MongoDB
        services.Configure<MongoSettings>(config.GetSection("MongoDB"));
        services.AddSingleton<IMongoClient>(sp => 
            new MongoClient(sp.GetRequiredService<IOptions<MongoSettings>>().Value.ConnectionString));
        services.AddScoped(sp => sp.GetRequiredService<IMongoClient>().GetDatabase("arteris"));
        
        // Repository Adapters
        services.Scan(scan => scan
            .FromAssemblyOf<PedidoMongoRepository>()
            .AddClasses(classes => classes.AssignableTo(typeof(IRepository<>)))
            .AsImplementedInterfaces()
            .WithScopedLifetime());

        // Redis
        services.Configure<RedisSettings>(config.GetSection("Redis"));
        services.AddSingleton<IConnectionMultiplexer>(sp => 
            ConnectionMultiplexer.Connect(sp.GetRequiredService<IOptions<RedisSettings>>().Value.ConnectionString));
        services.AddScoped<ICacheService, RedisCacheAdapter>();

        // External Services
        services.AddHttpClient<IExternalApiClient, HttpExternalApiAdapter>();
        
        // Security
        services.AddScoped<ITokenService, JwtTokenAdapter>();
        services.AddScoped<IPasswordHasher, Argon2PasswordHasher>();

        return services;
    }
}
```

### 4. Result Pattern (Erro como valor, não exceção)
```csharp
// BuildingBlocks/Result.cs
public readonly struct Result<T>
{
    public T Value { get; }
    public Error Error { get; }
    public bool IsSuccess => Error is null;
    public bool IsFailure => !IsSuccess;

    private Result(T value) { Value = value; Error = null; }
    private Result(Error error) { Value = default!; Error = error; }

    public static Result<T> Success(T value) => new(value);
    public static Result<T> Failure(Error error) => new(error);

    public IActionResult ToProblemDetails() => IsSuccess
        ? new OkObjectResult(Value)
        : Error.ToProblemDetails();
}

public readonly record struct Error(string Code, string Message, object? Details = null)
{
    public static Error Validation(string message, object? details = null) 
        => new("VALIDATION_ERROR", message, details);
    public static Error NotFound(string message) 
        => new("NOT_FOUND", message);
    public static Error Unauthorized(string message = "Não autorizado") 
        => new("UNAUTHORIZED", message);
    public static Error Conflict(string message) 
        => new("CONFLICT", message);
    public static Error Internal(string message, Exception? ex = null) 
        => new("INTERNAL_ERROR", message, ex?.ToString());

    public IActionResult ToProblemDetails() => new ObjectResult(new ProblemDetails
    {
        Type = $"https://arteris.com/errors/{Code.ToLower()}",
        Title = Code.Replace("_", " "),
        Detail = Message,
        Status = Code switch
        {
            "VALIDATION_ERROR" => StatusCodes.Status400BadRequest,
            "NOT_FOUND" => StatusCodes.Status404NotFound,
            "UNAUTHORIZED" => StatusCodes.Status401Unauthorized,
            "CONFLICT" => StatusCodes.Status409Conflict,
            _ => StatusCodes.Status500InternalServerError
        },
        Extensions = { ["details"] = Details, ["code"] = Code }
    });
}
```

### 5. Domain Events (Publicação assíncrona)
```csharp
// Domain/Events/IDomainEventPublisher.cs
public interface IDomainEventPublisher
{
    Task PublishAsync<TEvent>(TEvent @event, CancellationToken ct = default) where TEvent : IDomainEvent;
}

// Application/Events/DomainEventPublisher.cs
public class DomainEventPublisher : IDomainEventPublisher
{
    private readonly IServiceProvider _sp;
    private readonly ILogger<DomainEventPublisher> _logger;

    public DomainEventPublisher(IServiceProvider sp, ILogger<DomainEventPublisher> logger)
    {
        _sp = sp;
        _logger = logger;
    }

    public async Task PublishAsync<TEvent>(TEvent @event, CancellationToken ct = default) 
        where TEvent : IDomainEvent
    {
        using var scope = _sp.CreateScope();
        var handlers = scope.ServiceProvider.GetServices<IDomainEventHandler<TEvent>>();
        
        foreach (var handler in handlers)
        {
            try { await handler.HandleAsync(@event, ct); }
            catch (Exception ex) { _logger.LogError(ex, "Erro ao processar {EventType}", typeof(TEvent).Name); }
        }
    }
}
```

### 6. MongoDB Adapter (Repository Pattern)
```csharp
// Infrastructure/Persistence/MongoDB/Adapters/PedidoMongoRepository.cs
public sealed class PedidoMongoRepository : IPedidoRepository
{
    private readonly IMongoCollection<PedidoDocument> _collection;
    private readonly IMapper _mapper;

    public PedidoMongoRepository(IMongoDatabase database, IMapper mapper)
    {
        _collection = database.GetCollection<PedidoDocument>("pedidos");
        _mapper = mapper;
        CreateIndexesAsync().Wait();
    }

    public async Task<Pedido?> ObterPorIdAsync(PedidoId id, CancellationToken ct = default)
    {
        var filter = Builders<PedidoDocument>.Filter.Eq(d => d.Id, id.Value);
        var doc = await _collection.Find(filter).FirstOrDefaultAsync(ct);
        return doc is null ? null : _mapper.Map<Pedido>(doc);
    }

    public async Task AdicionarAsync(Pedido pedido, CancellationToken ct = default)
    {
        var doc = _mapper.Map<PedidoDocument>(pedido);
        await _collection.InsertOneAsync(doc, cancellationToken: ct);
    }

    public async Task AtualizarAsync(Pedido pedido, CancellationToken ct = default)
    {
        var doc = _mapper.Map<PedidoDocument>(pedido);
        var filter = Builders<PedidoDocument>.Filter.Eq(d => d.Id, pedido.Id.Value);
        await _collection.ReplaceOneAsync(filter, doc, cancellationToken: ct);
    }

    private async Task CreateIndexesAsync()
    {
        await _collection.Indexes.CreateOneAsync(new CreateIndexModel<PedidoDocument>(
            Builders<PedidoDocument>.IndexKeys.Ascending(d => d.ClienteId)));
        await _collection.Indexes.CreateOneAsync(new CreateIndexModel<PedidoDocument>(
            Builders<PedidoDocument>.IndexKeys.Ascending(d => d.Status)));
    }
}
```

### 7. Redis Cache Adapter
```csharp
// Infrastructure/Persistence/Redis/Adapters/RedisCacheAdapter.cs
public sealed class RedisCacheAdapter : ICacheService
{
    private readonly IDatabase _db;
    private readonly ILogger<RedisCacheAdapter> _logger;

    public RedisCacheAdapter(IConnectionMultiplexer mux, ILogger<RedisCacheAdapter> logger)
    {
        _db = mux.GetDatabase();
        _logger = logger;
    }

    public async Task<T?> GetAsync<T>(string key, CancellationToken ct = default)
    {
        var value = await _db.StringGetAsync(key);
        return value.HasValue ? JsonSerializer.Deserialize<T>(value!) : default;
    }

    public async Task SetAsync<T>(string key, T value, TimeSpan? expiry = null, CancellationToken ct = default)
    {
        var json = JsonSerializer.Serialize(value);
        await _db.StringSetAsync(key, json, expiry ?? TimeSpan.FromMinutes(30));
    }

    public async Task RemoveAsync(string key, CancellationToken ct = default)
        => await _db.KeyDeleteAsync(key);

    public async Task RemoveByPatternAsync(string pattern, CancellationToken ct = default)
    {
        var server = _db.Multiplexer.GetServer(_db.Multiplexer.GetEndPoints().First());
        var keys = server.Keys(pattern: pattern).ToArray();
        if (keys.Length > 0) await _db.KeyDeleteAsync(keys);
    }
}
```

### 8. JWT Token Adapter
```csharp
// Infrastructure/Security/JwtTokenAdapter.cs
public sealed class JwtTokenAdapter : ITokenService
{
    private readonly JwtSettings _settings;
    private readonly ILogger<JwtTokenAdapter> _logger;

    public JwtTokenAdapter(IOptions<JwtSettings> settings, ILogger<JwtTokenAdapter> logger)
    {
        _settings = settings.Value;
        _logger = logger;
    }

    public string GenerateAccessToken(Usuario usuario, IEnumerable<string> roles)
    {
        var claims = new List<Claim>
        {
            new(JwtRegisteredClaimNames.Sub, usuario.Id.Value.ToString()),
            new(JwtRegisteredClaimNames.Email, usuario.Email.Valor),
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            new("nome", usuario.Nome)
        };
        claims.AddRange(roles.Select(r => new Claim(ClaimTypes.Role, r)));

        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_settings.SecretKey));
        var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var token = new JwtSecurityToken(
            issuer: _settings.Issuer,
            audience: _settings.Audience,
            claims: claims,
            expires: DateTime.UtcNow.AddMinutes(_settings.AccessTokenExpirationMinutes),
            signingCredentials: creds);

        return new JwtSecurityTokenHandler().WriteToken(token);
    }

    public string GenerateRefreshToken() => Convert.ToBase64String(RandomNumberGenerator.GetBytes(64));

    public ClaimsPrincipal? ValidateToken(string token)
    {
        var handler = new JwtSecurityTokenHandler();
        var validationParams = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidIssuer = _settings.Issuer,
            ValidateAudience = true,
            ValidAudience = _settings.Audience,
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_settings.SecretKey)),
            ValidateLifetime = true,
            ClockSkew = TimeSpan.Zero
        };

        try { return handler.ValidateToken(token, validationParams, out _); }
        catch { return null; }
    }
}
```

### 9. Exception Handling Middleware
```csharp
// Presentation/Middleware/ExceptionHandlingMiddleware.cs
public sealed class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;

    public ExceptionHandlingMiddleware(RequestDelegate next, ILogger<ExceptionHandlingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try { await _next(context); }
        catch (ValidationException ex)
        {
            _logger.LogWarning(ex, "Erro de validação");
            await WriteProblemDetails(context, Error.Validation(ex.Message, ex.Errors), StatusCodes.Status400BadRequest);
        }
        catch (DomainException ex)
        {
            _logger.LogWarning(ex, "Erro de domínio: {Message}", ex.Message);
            await WriteProblemDetails(context, Error.Conflict(ex.Message), StatusCodes.Status409Conflict);
        }
        catch (UnauthorizedAccessException ex)
        {
            _logger.LogWarning(ex, "Acesso não autorizado");
            await WriteProblemDetails(context, Error.Unauthorized(), StatusCodes.Status401Unauthorized);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Erro interno não tratado");
            await WriteProblemDetails(context, Error.Internal("Erro interno do servidor"), StatusCodes.Status500InternalServerError);
        }
    }

    private static async Task WriteProblemDetails(HttpContext context, Error error, int statusCode)
    {
        context.Response.StatusCode = statusCode;
        context.Response.ContentType = "application/problem+json";
        await context.Response.WriteAsJsonAsync(error.ToProblemDetails().Value);
    }
}
```

### 10. Structured Logging (Serilog)
```csharp
// Program.cs - Configuração
Log.Logger = new LoggerConfiguration()
    .ReadFrom.Configuration(builder.Configuration)
    .Enrich.FromLogContext()
    .Enrich.WithProperty("Application", "Arteris.Backend")
    .Enrich.WithProperty("Environment", builder.Environment.EnvironmentName)
    .WriteTo.Console(outputTemplate: "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}")
    .WriteTo.Seq(builder.Configuration.GetConnectionString("Seq"))
    .CreateLogger();

builder.Host.UseSerilog();
```

```csharp
// Uso no Use Case
public class CriarPedidoUseCase : ICriarPedidoUseCase
{
    private readonly ILogger<CriarPedidoUseCase> _logger;

    public async Task<Result<CriarPedidoResponse>> ExecuteAsync(CriarPedidoCommand cmd, CancellationToken ct)
    {
        using (_logger.BeginScope(new Dictionary<string, object> 
        { 
            ["ClienteId"] = cmd.ClienteId, 
            ["CorrelationId"] = Activity.Current?.Id ?? Guid.NewGuid().ToString() 
        }))
        {
            _logger.LogInformation("Iniciando criação de pedido para cliente {ClienteId}", cmd.ClienteId);
            
            // ... lógica ...
            
            _logger.LogInformation("Pedido {PedidoId} criado com sucesso. Total: {Total}", pedido.Id, pedido.Total);
            return Result.Success(new CriarPedidoResponse(pedido.Id, pedido.Total));
        }
    }
}
```

## Testes

### Unitários (Use Cases com Ports Out mockados)
```csharp
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
        result.Value.Id.Should().NotBeEmpty();
        _pedidoRepo.Verify(x => x.AdicionarAsync(It.IsAny<Pedido>(), It.IsAny<CancellationToken>()), Times.Once);
        _events.Verify(x => x.PublishAsync(It.IsAny<PedidoCriadoEvent>(), It.IsAny<CancellationToken>()), Times.Once);
    }
}
```

### Integração (Testcontainers - MongoDB/Redis reais)
```csharp
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

## Boas Práticas

1. **Minimal APIs** para endpoints simples; **Controllers** para complexos
2. **Result<T>** para erros esperados; **exceptions** apenas para falhas inesperadas
3. **Scoped** para Use Cases/Repositories; **Singleton** para MongoClient/RedisConnection
4. **Validation** em DTOs (FluentValidation) + Domain (construtores/factories)
5. **Structured logging** com correlation IDs (OpenTelemetry/W3C TraceContext)
6. **Health checks** em Adapters críticos (MongoDB, Redis, External APIs)
7. **CancellationToken** propagado em toda cadeia async
8. **ProblemDetails (RFC 7807)** para erros de API padronizados

## Critérios de Qualidade

- [ ] APIs seguem REST/HTTP semantics corretos
- [ ] Domain compila sem referenciar Infrastructure
- [ ] Use Cases testáveis unitariamente (Ports Out mockados)
- [ ] Integração com MongoDB/Redis via Testcontainers
- [ ] Logs estruturados com correlation ID
- [ ] Erros padronizados com ProblemDetails
- [ ] JWT tokens com expiração curta + refresh token rotation
- [ ] DI configurado corretamente (Scoped/Transient/Singleton)
- [ ] Health checks expostos em `/health`

## Referências

- `arquitetura-hexagonal` — estrutura de camadas
- `ddd-conceitos` — modelagem de domínio
- `testes-backend` — estratégia completa de testes
- `devops-azure` — deploy, observabilidade, custos