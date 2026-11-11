---
name: ddd-conceitos
description: Domain-Driven Design: conceitos estratégicos e táticos, modelagem de domínio, bounded contexts
metadata:
  type: specialized
  category: architecture
  agents: [deboaz-arc, deboaz-backend, deboaz-po]
---

# Skill: Domain-Driven Design (DDD)

## Responsabilidade e Escopo

Competência para modelagem de domínio complexo usando DDD. Abrange conceitos estratégicos (bounded contexts, ubiquitous language, context mapping) e táticos (entities, value objects, aggregates, domain services, events, repositories).

## Conceitos Estratégicos

### 1. Ubiquitous Language (Linguagem Ubíqua)
- Vocabulário compartilhado entre especialistas de domínio e desenvolvedores
- Refletido **diretamente no código** (nomes de classes, métodos, propriedades)
- Evolui com o entendimento do domínio
- Documentado em glossário vivo (Markdown no repositório)

### 2. Bounded Context (Contexto Delimitado)
- Fronteira explícita onde um modelo de domínio se aplica
- Cada contexto tem seu próprio Ubiquitous Language
- Modelo válido apenas dentro do contexto
- Tamanho: "Grande o bastante para ser útil, pequeno o bastante para ser gerenciável"

### 3. Context Mapping (Mapeamento de Contextos)
| Padrão | Descrição | Quando Usar |
|---|---|---|
| **Partnership** | Dois contextos evoluem juntos, coordenação próxima | Times alinhados, domínio interdependente |
| **Shared Kernel** | Subconjunto de modelo compartilhado | Poucas entidades compartilhadas, governança clara |
| **Customer/Supplier** | Upstream (fornecedor) atende necessidades do Downstream (cliente) | Dependência clara, priorização negociada |
| **Conformist** | Downstream adota modelo do Upstream sem negociação | Upstream não negocia, downstream aceita |
| **Anticorruption Layer** | Camada de tradução/isolação | Integração com legado/sistema externo, modelos divergentes |
| **Open Host Service** | API pública bem definida para consumidores | Muitos consumidores, estabilidade necessária |
| **Published Language** | Linguagem documentada para integração (ex: JSON Schema, Protobuf) | Integração entre organizações/times distintos |

## Conceitos Táticos

### 1. Entity (Entidade)
- **Identidade** contínua através do tempo e estados
- Igualdade por **identidade** (Id), não por atributos
- Ciclo de vida gerenciado (criação, modificação, exclusão)
- Encapsula comportamento e invariantes
- Exemplo: `Usuario`, `Pedido`, `Produto`

```csharp
public class Pedido : Entity<PedidoId>
{
    public UsuarioId ClienteId { get; private set; }
    public StatusPedido Status { get; private set; }
    private readonly List<ItemPedido> _itens = new();
    public IReadOnlyCollection<ItemPedido> Itens => _itens.AsReadOnly();

    public void AdicionarItem(Produto produto, Quantidade qtd, Preco preco)
    {
        // Regra de negócio: só pode adicionar se status = Rascunho
        if (Status != StatusPedido.Rascunho)
            throw new InvalidOperationException("Pedido não está em rascunho");
        
        _itens.Add(new ItemPedido(produto.Id, qtd, preco));
    }
}
```

### 2. Value Object (Objeto de Valor)
- **Sem identidade** — igualdade por **valor** (todos os atributos)
- **Imutável** — criado válido, nunca modificado
- Substituível por outro com mesmo valor
- Encapsula validação e formatação
- Exemplo: `Email`, `CPF`, `Money`, `Endereco`, `Quantidade`

```csharp
public record Email(string Valor) : ValueObject
{
    public static Email Criar(string valor)
    {
        if (string.IsNullOrWhiteSpace(valor))
            throw new ArgumentException("Email é obrigatório");
        if (!Regex.IsMatch(valor, @"^[^@\s]+@[^@\s]+\.[^@\s]+$"))
            throw new ArgumentException("Formato de email inválido");
        return new Email(valor.ToLowerInvariant());
    }
}
```

### 3. Aggregate (Agregado)
- **Cluster** de Entities e Value Objects tratados como unidade
- **Aggregate Root (Raiz)**: Única entidade acessível externamente
- **Consistência transacional** dentro do agregado (invariantes garantidas)
- **Consistência eventual** entre agregados (Domain Events)
- Regras:
  - Referências externas apenas para Aggregate Roots
  - Aggregate Root garante invariantes do cluster todo
  - Exclusão em cascata via root

```csharp
// Pedido é Aggregate Root
public class Pedido : AggregateRoot<PedidoId>
{
    private readonly List<ItemPedido> _itens = new();
    public IReadOnlyCollection<ItemPedido> Itens => _itens.AsReadOnly();
    
    public void Confirmar()
    {
        if (!_itens.Any())
            throw new DomainException("Pedido sem itens não pode ser confirmado");
        
        Status = StatusPedido.Confirmado;
        AddDomainEvent(new PedidoConfirmadoEvent(this.Id, this.ClienteId, this.Total));
    }
}

// ItemPedido é Entity interno ao agregado Pedido
public class ItemPedido : Entity<ItemPedidoId>
{
    public ProdutoId ProdutoId { get; }
    public Quantidade Quantidade { get; }
    public Preco PrecoUnitario { get; }
    public Preco Subtotal => PrecoUnitario * Quantidade;
}
```

### 4. Domain Service (Serviço de Domínio)
- Lógica de domínio que **não pertence a uma única entidade**
- Stateless (sem estado interno)
- Operações que envolvem múltiplas entidades/agregados
- Nomeado com verbo do domínio: `CalcularFrete`, `ValidarCpf`, `AplicarDesconto`
- **Não** substitui comportamento de Entities/Aggregates

```csharp
public class CalculoFreteService
{
    private readonly ITabelaFreteRepository _tabelaFrete;
    
    public Frete Calcular(Endereco origem, Endereco destino, Peso peso, Valor valor)
    {
        var tabela = _tabelaFrete.ObterVigente();
        return tabela.Calcular(origem, destino, peso, valor);
    }
}
```

### 5. Domain Event (Evento de Domínio)
- Fato **ocorrido** no domínio (passado, imutável)
- Publicado pelo Aggregate Root após mudança de estado
- Processado assincronamente (outros agregados, contextos, integrações)
- Nomes no passado: `PedidoConfirmado`, `UsuarioCadastrado`, `EstoqueReservado`

```csharp
public record PedidoConfirmadoEvent(
    PedidoId PedidoId,
    UsuarioId ClienteId,
    Money Total,
    DateTime OcorridoEm
) : IDomainEvent;
```

### 6. Repository (Repositório)
- **Interface no Domain** (Port Out), implementação na Infrastructure (Adapter)
- Abstrai persistência — cliente (Use Case) não conhece SQL/Mongo
- Métodos orientados a **Aggregate Root**: `ObterPorId`, `Adicionar`, `Atualizar`
- **Não** expõe IQueryable / LINQ / filtros genéricos
- Um repositório por Aggregate Root

```csharp
// Domain - Interface
public interface IPedidoRepository
{
    Task<Pedido?> ObterPorIdAsync(PedidoId id);
    Task AdicionarAsync(Pedido pedido);
    Task AtualizarAsync(Pedido pedido);
}

// Infrastructure - Implementação (Adapter)
public class PedidoMongoRepository : IPedidoRepository { ... }
```

### 7. Factory (Fábrica)
- Criação complexa de Aggregates/Entities
- Encapsula lógica de construção e validação inicial
- Útil quando criação envolve múltiplos passos ou dependências

```csharp
public class PedidoFactory
{
    public Pedido CriarNovo(UsuarioId clienteId, IEnumerable<ItemPedidoData> itens)
    {
        var pedido = new Pedido(PedidoId.Novo(), clienteId);
        foreach (var item in itens)
            pedido.AdicionarItem(item.ProdutoId, item.Quantidade, item.Preco);
        return pedido;
    }
}
```

## Modelagem Prática: Fluxo de Descoberta

```
1. Event Storming
   ├── Domain Events (laranja) — o que acontece
   ├── Commands (azul) — quem dispara
   ├── Aggregates (amarelo) — quem decide
   ├── Policies/Rules (rosa) — restrições
   ├── Read Models (verde) — o que é consultado
   └── External Systems (rosa escuro) — integrações

2. Identificar Bounded Contexts
   ├── Agrupar eventos/comandos por linguagem comum
   ├── Definir fronteiras (Context Mapping)
   └── Nomear contextos (ex: Catalogo, Pedidos, Pagamentos, Usuarios)

3. Modelar Aggregates por Contexto
   ├── Identificar Aggregate Roots
   ├── Definir invariantes transacionais
   ├── Identificar Value Objects
   └── Desenhar Domain Events

4. Definir Use Cases (Application)
   ├── Commands (escrita) → Alteram estado
   ├── Queries (leitura) → Não alteram estado
   └── Mapear para Ports In/Out

5. Implementar com Arquitetura Hexagonal
   ├── Domain puro (zero deps externas)
   ├── Application (Use Cases + Ports)
   ├── Infrastructure (Adapters)
   └── Presentation (Controllers/API)
```

## Boas Práticas

1. **Comece pelo domínio, não pelo banco** — modelagem direcionada a comportamento
2. **Aggregates pequenos** — grandes agregados = contenda, transações longas
3. **Value Objects preferidos** — imutáveis, testáveis, expressivos
4. **Domain Events para desacoplamento** — não chame serviços externos no Domain
5. **Repositories retornam Aggregates** — não DTOs, não entidades parciais
6. **Validação no construtor/factory** — Value Objects sempre válidos
7. **Evite anêmicos** — comportamento no Domain, não em Services
8. **Ubiquitous Language no código** — nomes em português do domínio (se time BR)

## Critérios de Qualidade

- [ ] Bounded Contexts identificados e mapeados
- [ ] Ubiquitous Language documentada e usada no código
- [ ] Aggregates protegem invariantes transacionais
- [ ] Value Objects imutáveis e validados na criação
- [ ] Domain Events representam fatos de negócio relevantes
- [ ] Repositories são interfaces no Domain, implementados na Infra
- [ ] Domain Services são stateless e nomeados com verbos de negócio
- [ ] Nenhuma regra de negócio vazando para Application/Infrastructure

## Referências

- `arquitetura-hexagonal` — estrutura de camadas
- `engenharia-software` — SOLID, Clean Architecture base
- `csharp-dotnet` — implementação idiomática
- `testes-backend` — testes de domínio