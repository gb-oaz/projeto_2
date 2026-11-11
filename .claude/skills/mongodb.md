---
name: mongodb
description: MongoDB: modelagem, queries, índices, agregação, transações, change streams, melhorias de performance
metadata:
  type: reference
  category: data
  agents: [deboaz-arc, deboaz-backend]
---

# Skill: MongoDB

## Responsabilidade e Escopo

Competência de referência para modelagem, operações, otimização e administração de MongoDB no contexto da aplicação Arteris (Azure Cosmos DB com API MongoDB).

## Conceitos Fundamentais

### Modelagem de Dados
| Padrão | Quando Usar | Exemplo |
|---|---|---|
| **Embedded** | Relação 1:1 ou 1:poucos, acesso sempre junto | `Pedido` → `Itens` (array de subdocumentos) |
| **Referenced** | Relação 1:muitos ou m:n, acesso independente | `Usuario` ↔ `Pedidos` (array de ObjectIds) |
| **Hybrid** | Acesso frequente a subconjunto + detalhes sob demanda | `Pedido` com `itensResumo` embedded + `itensDetalhados` referenced |

### Regras para Arteris
1. **Aggregates do DDD = Documents** (um document por Aggregate Root)
2. **Value Objects = Subdocumentos** (embedded)
3. **Entities filhas = Subdocumentos** se cycle de vida ligado ao root
4. **Referências por ObjectId** (string) para outros Aggregates
5. **Sem joins** — duplicação controlada (denormalização) para leitura

### Exemplo: Pedido (Aggregate Root)
```json
{
  "_id": "ped-123",
  "clienteId": "usr-456",
  "status": "confirmado",
  "itens": [
    { "produtoId": "prod-1", "quantidade": 2, "precoUnitario": 1500.00, "subtotal": 3000.00 },
    { "produtoId": "prod-2", "quantidade": 1, "precoUnitario": 250.00, "subtotal": 250.00 }
  ],
  "totais": { "subtotal": 3250.00, "frete": 45.00, "desconto": 0, "total": 3295.00 },
  "enderecoEntrega": {
    "logradouro": "Rua das Flores",
    "numero": "123",
    "bairro": "Centro",
    "cidade": "São Paulo",
    "estado": "SP",
    "cep": "01000-000"
  },
  "historicoStatus": [
    { "status": "rascunho", "timestamp": "2024-01-15T10:00:00Z", "usuarioId": "usr-456" },
    { "status": "confirmado", "timestamp": "2024-01-15T10:05:00Z", "usuarioId": "usr-456" }
  ],
  "criadoEm": "2024-01-15T10:00:00Z",
  "atualizadoEm": "2024-01-15T10:05:00Z",
  "versao": 2
}
```

## Operações Principais

### Connection & Configuration (C#)
```csharp
// Program.cs
builder.Services.Configure<MongoSettings>(config.GetSection("MongoDB"));
builder.Services.AddSingleton<IMongoClient>(sp => 
    new MongoClient(sp.GetRequiredService<IOptions<MongoSettings>>().Value.ConnectionString));
builder.Services.AddScoped(sp => sp.GetRequiredService<IMongoClient>().GetDatabase("arteris"));

// Settings
public class MongoSettings
{
    public string ConnectionString { get; set; } = "";
    public string DatabaseName { get; set; } = "arteris";
    public int MaxConnectionPoolSize { get; set; } = 100;
    public int MinConnectionPoolSize { get; set; } = 10;
}
```

### Repository Pattern (Adapter)
```csharp
public sealed class PedidoMongoRepository : IPedidoRepository
{
    private readonly IMongoCollection<PedidoDocument> _collection;
    
    public PedidoMongoRepository(IMongoDatabase database)
    {
        _collection = database.GetCollection<PedidoDocument>("pedidos");
        CreateIndexesAsync().GetAwaiter().GetResult();
    }

    public async Task<Pedido?> ObterPorIdAsync(PedidoId id, CancellationToken ct = default)
    {
        var filter = Builders<PedidoDocument>.Filter.Eq(d => d.Id, id.Value);
        var doc = await _collection.Find(filter).FirstOrDefaultAsync(ct);
        return doc?.ToDomain();
    }

    public async Task AdicionarAsync(Pedido pedido, CancellationToken ct = default)
    {
        var doc = pedido.ToDocument();
        await _collection.InsertOneAsync(doc, cancellationToken: ct);
    }

    public async Task AtualizarAsync(Pedido pedido, CancellationToken ct = default)
    {
        var doc = pedido.ToDocument();
        var filter = Builders<PedidoDocument>.Filter.Eq(d => d.Id, pedido.Id.Value);
        var result = await _collection.ReplaceOneAsync(filter, doc, cancellationToken: ct);
        if (result.MatchedCount == 0) 
            throw new ConcurrencyException($"Pedido {pedido.Id} não encontrado ou modificado por outro processo");
    }

    private async Task CreateIndexesAsync()
    {
        var indexKeys = Builders<PedidoDocument>.IndexKeys;
        await _collection.Indexes.CreateManyAsync(new[]
        {
            new CreateIndexModel<PedidoDocument>(indexKeys.Ascending(d => d.ClienteId)),
            new CreateIndexModel<PedidoDocument>(indexKeys.Ascending(d => d.Status)),
            new CreateIndexModel<PedidoDocument>(indexKeys.Ascending(d => d.CriadoEm)),
            new CreateIndexModel<PedidoDocument>(indexKeys.Combine(
                indexKeys.Ascending(d => d.ClienteId), 
                indexKeys.Descending(d => d.CriadoEm)))
        });
    }
}
```

### Mapeamento Domain ↔ Document
```csharp
// Domain → Document (para persistência)
public static PedidoDocument ToDocument(this Pedido pedido)
{
    return new PedidoDocument
    {
        Id = pedido.Id.Value,
        ClienteId = pedido.ClienteId.Value,
        Status = pedido.Status.ToString().ToLowerInvariant(),
        Itens = pedido.Itens.Select(i => new ItemPedidoDocument
        {
            ProdutoId = i.ProdutoId.Value,
            Quantidade = i.Quantidade.Valor,
            PrecoUnitario = i.PrecoUnitario.Valor,
            Subtotal = i.Subtotal.Valor
        }).ToList(),
        Totais = new TotaisDocument
        {
            Subtotal = pedido.Totais.Subtotal.Valor,
            Frete = pedido.Totais.Frete.Valor,
            Desconto = pedido.Totais.Desconto.Valor,
            Total = pedido.Totais.Total.Valor
        },
        EnderecoEntrega = pedido.EnderecoEntrega?.ToDocument(),
        HistoricoStatus = pedido.HistoricoStatus.Select(h => new HistoricoStatusDocument
        {
            Status = h.Status.ToString().ToLowerInvariant(),
            Timestamp = h.Timestamp,
            UsuarioId = h.UsuarioId.Value
        }).ToList(),
        CriadoEm = pedido.CriadoEm,
        AtualizadoEm = pedido.AtualizadoEm,
        Versao = pedido.Versao
    };
}

// Document → Domain (para reconstituição)
public static Pedido ToDomain(this PedidoDocument doc)
{
    var pedido = Pedido.Reconstituir(
        id: new PedidoId(doc.Id),
        clienteId: new UsuarioId(doc.ClienteId),
        status: Enum.Parse<StatusPedido>(doc.Status, true),
        itens: doc.Itens.Select(i => ItemPedido.Reconstituir(
            new ProdutoId(i.ProdutoId), 
            new Quantidade(i.Quantidade), 
            new Preco(i.PrecoUnitario)
        )).ToList(),
        totais: Totais.Reconstituir(
            new Money(doc.Totais.Subtotal),
            new Money(doc.Totais.Frete),
            new Money(doc.Totais.Desconto),
            new Money(doc.Totais.Total)
        ),
        enderecoEntrega: doc.EnderecoEntrega?.ToDomain(),
        historicoStatus: doc.HistoricoStatus.Select(h => 
            HistoricoStatus.Criar(Enum.Parse<StatusPedido>(h.Status, true), h.Timestamp, new UsuarioId(h.UsuarioId))
        ).ToList(),
        criadoEm: doc.CriadoEm,
        atualizadoEm: doc.AtualizadoEm,
        versao: doc.Versao
    );
    return pedido;
}
```

## Índices e Performance

### Estratégia de Índices
```csharp
// Índices compostos para queries frequentes
// Query: Pedidos por cliente ordenados por data
IndexKeys.Ascending(d => d.ClienteId).Descending(d => d.CriadoEm)

// Query: Pedidos por status + cliente
IndexKeys.Ascending(d => d.Status).Ascending(d => d.ClienteId)

// Query: Busca textual (se aplicável)
IndexKeys.Text(d => d.ClienteNome).Text(d => d.ProdutoNomes)
```

### Análise de Queries
```javascript
// MongoDB Shell - Explain plan
db.pedidos.find({ clienteId: "usr-456", status: "confirmado" })
  .sort({ criadoEm: -1 })
  .explain("executionStats")

// Verificar: 
// - "winningPlan.stage": "IXSCAN" (usa índice)
// - "executionStats.totalKeysExamined" vs "totalDocsExamined" (baixo ratio = bom)
// - "executionStats.executionTimeMillis"
```

### Aggregation Pipeline (Relatórios)
```csharp
// Total de vendas por mês
var pipeline = new[]
{
    PipelineDefinition<PedidoDocument, VendasPorMes>.Create()
        .Match(d => d.Status == "entregue")
        .Group(new BsonDocument
        {
            { "_id", new BsonDocument 
                { { "ano", "$year: $criadoEm" }, { "mes", "$month: $criadoEm" } } 
            },
            { "totalVendas", new BsonDocument("$sum", "$totais.total") },
            { "qtdPedidos", new BsonDocument("$sum", 1) },
            { "ticketMedio", new BsonDocument("$avg", "$totais.total") }
        })
        .Sort(Builders<VendasPorMes>.Sort.Descending("_id.ano").Descending("_id.mes"))
        .Project(Builders<VendasPorMes>.Projection
            .Include("_id")
            .Include("totalVendas")
            .Include("qtdPedidos")
            .Include("ticketMedio"))
};

var resultado = await _collection.Aggregate<VendasPorMes>(pipeline).ToListAsync();
```

## Transações (Multi-document ACID)

```csharp
public async Task<Result> ProcessarPagamentoEPedidoAsync(PedidoId pedidoId, PagamentoId pagamentoId)
{
    using var session = await _mongoClient.StartSessionAsync();
    session.StartTransaction(new TransactionOptions(
        readConcern: ReadConcern.Snapshot,
        writeConcern: WriteConcern.WMajority
    ));

    try
    {
        // 1. Atualizar pedido
        var pedido = await _pedidoRepo.ObterPorIdAsync(pedidoId, session);
        pedido.MarcarComoPago(pagamentoId);
        await _pedidoRepo.AtualizarAsync(pedido, session);

        // 2. Registrar pagamento
        var pagamento = Pagamento.Criar(pagamentoId, pedidoId, pedido.Totais.Total);
        await _pagamentoRepo.AdicionarAsync(pagamento, session);

        // 3. Reservar estoque (outro aggregate)
        await _estoqueService.ReservarAsync(pedido.Itens, session);

        await session.CommitTransactionAsync();
        return Result.Success();
    }
    catch (Exception ex)
    {
        await session.AbortTransactionAsync();
        return Result.Failure(Error.Internal("Falha ao processar pagamento", ex));
    }
}
```

## Change Streams (Eventos em Tempo Real)

```csharp
public class PedidoChangeStreamService : BackgroundService
{
    private readonly IMongoCollection<PedidoDocument> _collection;
    private readonly IDomainEventPublisher _publisher;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        var pipeline = new EmptyPipelineDefinition<ChangeStreamDocument<PedidoDocument>>()
            .Match(ChangeStreamDocument<PedidoDocument>.OperationTypeIn(OperationType.Insert, OperationType.Update, OperationType.Replace));

        var cursor = await _collection.WatchAsync(pipeline, cancellationToken: stoppingToken);

        await cursor.ForEachAsync(async change => 
        {
            var pedidoId = change.DocumentKey["_id"].AsString;
            switch (change.OperationType)
            {
                case OperationType.Insert:
                    await _publisher.PublishAsync(new PedidoCriadoEvent(pedidoId));
                    break;
                case OperationType.Update or OperationType.Replace:
                    var statusNovo = change.FullDocument?.Status;
                    if (statusNovo != null)
                        await _publisher.PublishAsync(new PedidoStatusAlteradoEvent(pedidoId, statusNovo));
                    break;
            }
        }, stoppingToken);
    }
}
```

## Azure Cosmos DB (MongoDB API) — Especificidades

| Aspecto | MongoDB Nativo | Cosmos DB MongoDB API |
|---|---|---|
| **Sharding** | Manual (shard key) | Automático (partition key = `_id` por padrão) |
| **Transações** | Replica set required | Suportado (single partition) |
| **Change Streams** | Nativo | Suportado (com limitações) |
| **Índices** | Completos | Subconjunto (sem text search nativo, usar Azure Cognitive Search) |
| **TTL** | Suportado | Suportado |
| **Preço** | Infra gerenciada | Serverless ou Provisioned RU/s |

### Partition Key Strategy
```csharp
// Para Cosmos DB, partition key = _id (padrão) funciona bem para:
// - Acesso por ID (ponto de leitura único)
// - Queries com filtro no _id

// Para queries por clienteId frequentes, considerar:
// - Container separado "pedidos-por-cliente" com partition key = clienteId
// - Ou materialized view via Change Feed
```

## Boas Práticas

1. **Schema validation** — JSON Schema no MongoDB/Cosmos para validar documentos na escrita
2. **Connection pooling** — Singleton `IMongoClient`, scoped `IMongoDatabase`
3. **CancellationToken** — Sempre propagar em operações async
4. **Optimistic concurrency** — Campo `versao` (int) + `FindOneAndReplace` com filtro de versão
5. **Projection** — Selecionar apenas campos necessários (`Find(projection: ...)`)
6. **Batch operations** — `InsertManyAsync`, `BulkWriteAsync` para operações em lote
7. **Monitoramento** — Query latency, connection pool, RU/s (Cosmos), index usage

## Critérios de Qualidade

- [ ] Índices cobrem queries frequentes (explain plan = IXSCAN)
- [ ] Repositories testados com Testcontainers (MongoDB real)
- [ ] Transações usadas apenas quando invariante cross-aggregate
- [ ] Change streams para eventos de domínio (não polling)
- [ ] Connection pool configurado (min/max adequado ao load)
- [ ] Schema validation ativo em produção
- [ ] Backup/restore testado (Cosmos: continuous backup + point-in-time restore)

## Referências

- `csharp-dotnet` — Repository Adapter implementation
- `ddd-conceitos` — Aggregate persistence
- `arquitetura-hexagonal` — Port Out interface
- `devops-azure` — Cosmos DB Terraform, monitoring