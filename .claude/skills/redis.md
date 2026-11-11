---
name: redis
description: Redis: cache distribuído, session store, pub/sub, rate limiting, distributed locks, padrões de uso
metadata:
  type: reference
  category: data
  agents: [deboaz-arc, deboaz-backend]
---

# Skill: Redis

## Responsabilidade e Escopo

Competência de referência para uso de Redis como cache distribuído, session store, pub/sub, rate limiting e distributed locks no contexto da aplicação Arteris (Azure Cache for Redis).

## Casos de Uso Principais

| Caso de Uso | Padrão | TTL Típico |
|---|---|---|
| **Cache de Queries** | Cache-Aside (Lazy Loading) | 5-30 min |
| **Cache de Objetos Completos** | Write-Through / Read-Through | 10-60 min |
| **Session Store** | Chave-valor com TTL sliding | 30 min (sliding) |
| **Rate Limiting** | Sliding Window / Token Bucket | Janela configurável |
| **Distributed Locks** | Redlock / SET NX EX | 10-30s (auto-release) |
| **Pub/Sub** | Canais para eventos leves | N/A (transient) |
| **Idempotency Keys** | SET NX EX | 24h |
| **Leader Election** | SET NX EX + heartbeat | TTL curto + renew |

## Configuration (C#)

```csharp
// Program.cs
builder.Services.Configure<RedisSettings>(config.GetSection("Redis"));
builder.Services.AddSingleton<IConnectionMultiplexer>(sp => 
{
    var settings = sp.GetRequiredService<IOptions<RedisSettings>>().Value;
    var config = ConfigurationOptions.Parse(settings.ConnectionString);
    config.AbortOnConnectFail = false;
    config.ConnectRetry = 3;
    config.ConnectTimeout = 5000;
    config.SyncTimeout = 5000;
    config.DefaultDatabase = settings.DatabaseId;
    return ConnectionMultiplexer.Connect(config);
});
builder.Services.AddScoped<ICacheService, RedisCacheAdapter>();
builder.Services.AddScoped<IDistributedLockProvider, RedisDistributedLockProvider>();
builder.Services.AddScoped<IRateLimiter, RedisRateLimiter>();

public class RedisSettings
{
    public string ConnectionString { get; set; } = "";
    public int DatabaseId { get; set; } = 0;
    public int DefaultTtlMinutes { get; set; } = 30;
    public int LockTimeoutSeconds { get; set; } = 15;
}
```

## Cache Service (Port Out + Adapter)

```csharp
// Application/Ports/ICacheService.cs
public interface ICacheService
{
    Task<T?> GetAsync<T>(string key, CancellationToken ct = default);
    Task SetAsync<T>(string key, T value, TimeSpan? expiry = null, CancellationToken ct = default);
    Task RemoveAsync(string key, CancellationToken ct = default);
    Task RemoveByPatternAsync(string pattern, CancellationToken ct = default);
    Task<bool> ExistsAsync(string key, CancellationToken ct = default);
    Task<long> IncrementAsync(string key, long value = 1, TimeSpan? expiry = null, CancellationToken ct = default);
}

// Infrastructure/Persistence/Redis/Adapters/RedisCacheAdapter.cs
public sealed class RedisCacheAdapter : ICacheService
{
    private readonly IDatabase _db;
    private readonly ILogger<RedisCacheAdapter> _logger;
    private readonly JsonSerializerOptions _jsonOptions;

    public RedisCacheAdapter(IConnectionMultiplexer mux, ILogger<RedisCacheAdapter> logger)
    {
        _db = mux.GetDatabase();
        _logger = logger;
        _jsonOptions = new JsonSerializerOptions { PropertyNamingPolicy = JsonNamingPolicy.CamelCase };
    }

    public async Task<T?> GetAsync<T>(string key, CancellationToken ct = default)
    {
        var value = await _db.StringGetAsync(key);
        if (!value.HasValue) return default;
        try { return JsonSerializer.Deserialize<T>(value!, _jsonOptions); }
        catch (JsonException ex) 
        { 
            _logger.LogWarning(ex, "Falha ao deserializar cache key {Key}", key);
            await _db.KeyDeleteAsync(key);
            return default;
        }
    }

    public async Task SetAsync<T>(string key, T value, TimeSpan? expiry = null, CancellationToken ct = default)
    {
        var json = JsonSerializer.Serialize(value, _jsonOptions);
        await _db.StringSetAsync(key, json, expiry ?? TimeSpan.FromMinutes(30), When.Always, CommandFlags.FireAndForget);
    }

    public async Task RemoveAsync(string key, CancellationToken ct = default)
        => await _db.KeyDeleteAsync(key, CommandFlags.FireAndForget);

    public async Task RemoveByPatternAsync(string pattern, CancellationToken ct = default)
    {
        var server = _db.Multiplexer.GetServer(_db.Multiplexer.GetEndPoints().First());
        var keys = server.Keys(pattern: pattern).ToArray();
        if (keys.Length > 0)
            await _db.KeyDeleteAsync(keys, CommandFlags.FireAndForget);
    }

    public async Task<bool> ExistsAsync(string key, CancellationToken ct = default)
        => await _db.KeyExistsAsync(key);

    public async Task<long> IncrementAsync(string key, long value = 1, TimeSpan? expiry = null, CancellationToken ct = default)
    {
        var result = await _db.StringIncrementAsync(key, value, CommandFlags.FireAndForget);
        if (expiry.HasValue && result == value) // Primeira vez
            await _db.KeyExpireAsync(key, expiry, CommandFlags.FireAndForget);
        return result;
    }
}
```

## Padrões de Cache

### 1. Cache-Aside (Lazy Loading) — Mais Comum
```csharp
public class ProdutoQueryService : IObterProdutoQuery
{
    private readonly ICacheService _cache;
    private readonly IProdutoRepository _repository;
    private readonly TimeSpan _cacheTtl = TimeSpan.FromMinutes(15);

    public async Task<Result<ProdutoDetalhado>> ExecuteAsync(string produtoId, CancellationToken ct)
    {
        var cacheKey = $"produto:{produtoId}";
        
        // 1. Tentar cache
        var cached = await _cache.GetAsync<ProdutoDetalhado>(cacheKey, ct);
        if (cached != null) return Result.Success(cached);

        // 2. Cache miss → buscar no banco
        var produto = await _repository.ObterPorIdAsync(new ProdutoId(produtoId), ct);
        if (produto == null) return Result.Failure(Error.NotFound("Produto não encontrado"));

        var dto = produto.ToDetalhadoDto();
        
        // 3. Popular cache (fire-and-forget para não bloquear)
        _ = _cache.SetAsync(cacheKey, dto, _cacheTtl, ct);
        
        return Result.Success(dto);
    }
}
```

### 2. Write-Through (Consistência Forte)
```csharp
public class ProdutoCommandService : IAtualizarProdutoUseCase
{
    private readonly ICacheService _cache;
    private readonly IProdutoRepository _repository;

    public async Task<Result> ExecuteAsync(AtualizarProdutoCommand command, CancellationToken ct)
    {
        var produto = await _repository.ObterPorIdAsync(command.Id, ct);
        if (produto == null) return Result.Failure(Error.NotFound("Produto não encontrado"));

        produto.Atualizar(command.Nome, command.Descricao, command.Preco);
        await _repository.AtualizarAsync(produto, ct);

        // Atualizar cache imediatamente (write-through)
        var cacheKey = $"produto:{command.Id}";
        await _cache.SetAsync(cacheKey, produto.ToDetalhadoDto(), TimeSpan.FromMinutes(15), ct);
        
        // Invalidar listas relacionadas
        await _cache.RemoveByPatternAsync("produtos:lista:*", ct);
        
        return Result.Success();
    }
}
```

### 3. Cache Invalidation Strategies
```csharp
// Invalidação por evento de domínio
public class ProdutoCacheInvalidationHandler : IDomainEventHandler<ProdutoAtualizadoEvent>
{
    private readonly ICacheService _cache;

    public async Task HandleAsync(ProdutoAtualizadoEvent @event, CancellationToken ct)
    {
        // Invalidação direta
        await _cache.RemoveAsync($"produto:@{event.ProdutoId}", ct);
        
        // Invalidação de listas (padrão: produtos:lista:{filtroHash})
        await _cache.RemoveByPatternAsync("produtos:lista:*", ct);
    }
}
```

## Rate Limiting (Sliding Window)

```csharp
// Application/Ports/IRateLimiter.cs
public interface IRateLimiter
{
    Task<RateLimitResult> CheckLimitAsync(string key, int maxRequests, TimeSpan window, CancellationToken ct = default);
}

public record RateLimitResult(bool Allowed, int Remaining, TimeSpan ResetIn, long TotalRequests);

// Infrastructure/Redis/RedisRateLimiter.cs
public sealed class RedisRateLimiter : IRateLimiter
{
    private readonly IDatabase _db;
    private readonly ILogger<RedisRateLimiter> _logger;

    public RedisRateLimiter(IConnectionMultiplexer mux, ILogger<RedisRateLimiter> logger)
    {
        _db = mux.GetDatabase();
        _logger = logger;
    }

    public async Task<RateLimitResult> CheckLimitAsync(string key, int maxRequests, TimeSpan window, CancellationToken ct = default)
    {
        var redisKey = $"ratelimit:{key}";
        var now = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds();
        var windowStart = now - (long)window.TotalMilliseconds;

        // Lua script para atomicidade (sliding window log)
        var script = @"
            local key = KEYS[1]
            local now = tonumber(ARGV[1])
            local window_start = tonumber(ARGV[2])
            local max_requests = tonumber(ARGV[3])
            local window_ms = tonumber(ARGV[4])
            
            -- Remover entradas antigas
            redis.call('ZREMRANGEBYSCORE', key, '-inf', window_start)
            
            -- Contar requests atuais
            local current = redis.call('ZCARD', key)
            
            if current >= max_requests then
                local ttl = redis.call('PTTL', key)
                if ttl < 0 then ttl = window_ms end
                return {0, max_requests - current, ttl, current}
            end
            
            -- Adicionar request atual
            redis.call('ZADD', key, now, now .. '-' .. math.random())
            redis.call('PEXPIRE', key, window_ms)
            
            return {1, max_requests - current - 1, window_ms, current + 1}
        ";

        var result = await _db.ScriptEvaluateAsync(script, 
            new RedisKey[] { redisKey },
            new RedisValue[] { now, windowStart, maxRequests, (long)window.TotalMilliseconds });

        var allowed = (int)result[0] == 1;
        var remaining = (int)result[1];
        var resetIn = TimeSpan.FromMilliseconds((int)result[2]);
        var total = (long)result[3];

        return new RateLimitResult(allowed, remaining, resetIn, total);
    }
}

// Uso no Middleware
public class RateLimitingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IRateLimiter _rateLimiter;

    public async Task InvokeAsync(HttpContext context)
    {
        var ip = context.Connection.RemoteIpAddress?.ToString() ?? "unknown";
        var path = context.Request.Path.Value ?? "/";
        var key = $"{ip}:{path}";

        var result = await _rateLimiter.CheckLimitAsync(key, 100, TimeSpan.FromMinutes(1));
        
        context.Response.Headers["X-RateLimit-Limit"] = "100";
        context.Response.Headers["X-RateLimit-Remaining"] = result.Remaining.ToString();
        context.Response.Headers["X-RateLimit-Reset"] = ((int)result.ResetIn.TotalSeconds).ToString();

        if (!result.Allowed)
        {
            context.Response.StatusCode = StatusCodes.Status429TooManyRequests;
            await context.Response.WriteAsJsonAsync(new ProblemDetails
            {
                Title = "Too Many Requests",
                Detail = "Limite de requisições excedido. Tente novamente em breve.",
                Status = StatusCodes.Status429TooManyRequests
            });
            return;
        }

        await _next(context);
    }
}
```

## Distributed Locks (Redlock Simplificado)

```csharp
// Application/Ports/IDistributedLockProvider.cs
public interface IDistributedLockProvider
{
    Task<IDistributedLock?> AcquireAsync(string resource, TimeSpan expiry, TimeSpan? waitTimeout = null, CancellationToken ct = default);
}

public interface IDistributedLock : IDisposable
{
    string Resource { get; }
    bool IsAcquired { get; }
    Task RenewAsync(TimeSpan expiry, CancellationToken ct = default);
}

// Infrastructure/Redis/RedisDistributedLockProvider.cs
public sealed class RedisDistributedLockProvider : IDistributedLockProvider
{
    private readonly IDatabase _db;

    public RedisDistributedLockProvider(IConnectionMultiplexer mux)
        => _db = mux.GetDatabase();

    public async Task<IDistributedLock?> AcquireAsync(string resource, TimeSpan expiry, TimeSpan? waitTimeout = null, CancellationToken ct = default)
    {
        var key = $"lock:{resource}";
        var token = Guid.NewGuid().ToString();
        var timeout = waitTimeout ?? TimeSpan.FromSeconds(10);
        var endTime = DateTime.UtcNow.Add(timeout);

        while (DateTime.UtcNow < endTime)
        {
            var acquired = await _db.StringSetAsync(key, token, expiry, When.NotExists);
            if (acquired) return new RedisDistributedLock(_db, key, token, expiry);
            
            await Task.Delay(50, ct);
        }
        return null;
    }
}

public sealed class RedisDistributedLock : IDistributedLock
{
    private readonly IDatabase _db;
    private readonly string _key;
    private readonly string _token;
    private readonly TimeSpan _expiry;
    private bool _released = false;

    public RedisDistributedLock(IDatabase db, string key, string token, TimeSpan expiry)
    {
        _db = db; _key = key; _token = token; _expiry = expiry;
    }

    public string Resource => _key.Replace("lock:", "");
    public bool IsAcquired => !_released;

    public async Task RenewAsync(TimeSpan expiry, CancellationToken ct = default)
    {
        if (_released) return;
        var script = @"
            if redis.call('GET', KEYS[1]) == ARGV[1] then
                return redis.call('PEXPIRE', KEYS[1], ARGV[2])
            else
                return 0
            end";
        await _db.ScriptEvaluateAsync(script, new RedisKey[] { _key }, new RedisValue[] { _token, (long)expiry.TotalMilliseconds });
    }

    public void Dispose()
    {
        if (!_released)
        {
            _ = ReleaseAsync();
            _released = true;
        }
    }

    private async Task ReleaseAsync()
    {
        var script = @"
            if redis.call('GET', KEYS[1]) == ARGV[1] then
                return redis.call('DEL', KEYS[1])
            else
                return 0
            end";
        await _db.ScriptEvaluateAsync(script, new RedisKey[] { _key }, new RedisValue[] { _token });
    }
}

// Uso: Garantir processamento único de pedido
public async Task<Result> ConfirmarPedidoAsync(PedidoId id)
{
    using var lock = await _lockProvider.AcquireAsync($"pedido:{id}", TimeSpan.FromSeconds(30));
    if (lock == null) return Result.Failure(Error.Conflict("Pedido sendo processado por outra requisição"));

    try 
    {
        var pedido = await _repo.ObterPorIdAsync(id);
        pedido.Confirmar();
        await _repo.AtualizarAsync(pedido);
        return Result.Success();
    }
    finally { lock.Dispose(); }
}
```

## Pub/Sub (Eventos Leves Internos)

```csharp
// Infrastructure/Redis/RedisMessageBus.cs
public sealed class RedisMessageBus : IMessageBus
{
    private readonly ISubscriber _subscriber;
    private readonly ILogger<RedisMessageBus> _logger;

    public RedisMessageBus(IConnectionMultiplexer mux, ILogger<RedisMessageBus> logger)
    {
        _subscriber = mux.GetSubscriber();
        _logger = logger;
    }

    public async Task PublishAsync<T>(string channel, T message, CancellationToken ct = default)
    {
        var json = JsonSerializer.Serialize(message);
        await _subscriber.PublishAsync(channel, json, CommandFlags.FireAndForget);
    }

    public async Task SubscribeAsync<T>(string channel, Func<T, CancellationToken, Task> handler, CancellationToken ct = default)
    {
        await _subscriber.SubscribeAsync(channel, async (ch, msg) =>
        {
            try 
            {
                var message = JsonSerializer.Deserialize<T>(msg!);
                if (message != null) await handler(message, ct);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Erro ao processar mensagem no canal {Channel}", channel);
            }
        });
    }
}
```

## Session Store (Frontend/Backend)

```csharp
// Armazenar sessão do usuário (refresh token, permissões, preferências)
public class SessionService
{
    private readonly ICacheService _cache;
    private readonly TimeSpan _sessionTtl = TimeSpan.FromMinutes(30); // Sliding

    public async Task SetSessionAsync(string userId, UserSession session, CancellationToken ct)
    {
        var key = $"session:{userId}";
        await _cache.SetAsync(key, session, _sessionTtl, ct);
    }

    public async Task<UserSession?> GetSessionAsync(string userId, CancellationToken ct)
    {
        var key = $"session:{userId}";
        var session = await _cache.GetAsync<UserSession>(key, ct);
        if (session != null)
        {
            // Renovar TTL (sliding expiration)
            await _cache.SetAsync(key, session, _sessionTtl, ct);
        }
        return session;
    }

    public async Task InvalidateSessionAsync(string userId, CancellationToken ct)
        => await _cache.RemoveAsync($"session:{userId}", ct);
}
```

## Boas Práticas

1. **Sempre usar ConnectionMultiplexer singleton** — não criar conexões por request
2. **Keys com prefixo estruturado** — `app:cache:produto:{id}`, `app:ratelimit:ip:{ip}`
3. **Serialização JSON** — não binário (debugging, interoperabilidade)
4. **TTL adequado** — nem muito curto (thundering herd), nem muito longo (stale data)
5. **Circuit breaker** — falhar rápido se Redis indisponível (fallback para DB direto)
6. **Monitoramento** — Hit rate, memory usage, evictions, latency, connected clients
7. **Lua scripts** — para atomicidade (rate limiting, locks, compare-and-set)
8. **FireAndForget** — para writes de cache não críticos (performance)
9. **Key expiration policies** — `volatile-lru` / `allkeys-lru` conforme uso

## Critérios de Qualidade

- [ ] Cache hit rate > 80% para queries frequentes
- [ ] Latência P99 < 5ms (local) / < 20ms (Azure)
- [ ] Zero chaves sem TTL (exceto locks com renovação)
- [ ] Rate limiting protegendo endpoints sensíveis (auth, pagamento)
- [ ] Distributed locks com TTL + renovação para operações longas
- [ ] Circuit breaker testado (Redis down → fallback DB)
- [ ] Memory policy configurada (`allkeys-lru` para cache, `volatile-lru` para sessões)
- [ ] Alertas: memory > 80%, evictions > 0, hit rate < 50%

## Referências

- `csharp-dotnet` — ICacheService, Adapter implementation
- `devops-azure` — Azure Cache for Redis Terraform, monitoring
- `arquitetura-hexagonal` — Port Out interface
- `estrategia-qa` — Testes de integração com Testcontainers Redis