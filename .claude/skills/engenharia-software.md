---
name: engenharia-software
description: Fundamentos de engenharia de software, princípios SOLID, Clean Architecture, DDD, TDD, BDD
metadata:
  type: shared
  category: engineering
---

# Skill: Engenharia de Software

## Responsabilidade e Escopo

Fundamentos transversais de engenharia de software aplicáveis a todos os agentes técnicos. Base conceitual para decisões de design, arquitetura, implementação e qualidade.

## Conhecimentos Técnicos

### 1. Programação Orientada a Objetos (POO)
- Encapsulamento, herança, polimorfismo, abstração
- Princípios: GRASP (General Responsibility Assignment Software Patterns)
- Coesão alta, acoplamento baixo
- Tell, Don't Ask

### 2. SOLID
| Princípio | Descrição |
|---|---|
| **S** - Single Responsibility | Uma classe/módulo tem uma única razão para mudar |
| **O** - Open/Closed | Aberto para extensão, fechado para modificação |
| **L** - Liskov Substitution | Subtipos devem ser substituíveis por seus tipos base |
| **I** - Interface Segregation | Interfaces específicas são melhores que uma interface geral |
| **D** - Dependency Inversion | Depender de abstrações, não de concretude |

### 3. Clean Architecture
- Camadas: Entidades → Casos de Uso → Interface Adapters → Frameworks/Drivers
- Regra da dependência: camadas internas não conhecem externas
- Independência de framework, UI, banco de dados, agência externa

### 4. Domain-Driven Design (DDD)
#### Conceitos Estratégicos
- **Bounded Context**: Limite explícito de um modelo de domínio
- **Ubiquitous Language**: Linguagem compartilhada entre domínio e código
- **Context Mapping**: Relacionamentos entre contextos (Partnership, Shared Kernel, Customer/Supplier, Conformist, Anticorruption Layer, Open Host Service, Published Language)

#### Conceitos Táticos
- **Entity**: Identidade contínua, ciclo de vida
- **Value Object**: Imutável, igualdade por valor, sem identidade
- **Aggregate**: Consistência transacional, raiz de agregado (Aggregate Root)
- **Domain Event**: Fato ocorrido no domínio, publicado para outros agregados/contextos
- **Domain Service**: Lógica de domínio que não pertence a uma única entidade
- **Repository**: Abstração de persistência (interface no domínio, implementação na infraestrutura)
- **Factory**: Criação complexa de agregados/entidades

### 5. Arquitetura Hexagonal (Ports and Adapters)
```
          ┌─────────────────────┐
          │    Ports In         │  ← Drivers (Controllers, CLI, Events)
          │  (Use Case Interfaces)│
          └──────────┬──────────┘
                     │
          ┌──────────▼──────────┐
          │    Use Cases        │  ← Orquestração, regras de aplicação
          └──────────┬──────────┘
                     │
    ┌────────────────┼────────────────┐
    │                │                │
┌───▼───┐      ┌─────▼─────┐    ┌────▼────┐
│Domain │      │ Ports Out │    │  DTOs   │
│       │      │(Interfaces)│    │         │
└───┬───┘      └─────┬─────┘    └────┬────┘
    │                │               │
    │         ┌──────▼──────┐        │
    │         │  Adapters   │        │
    │         │(Implementações)       │
    │         └─────────────┘        │
    │    MongoDB  Redis  External    │
    └────────────────────────────────┘
```

- **Ports In**: Contratos de entrada (Use Case interfaces)
- **Use Cases**: Orquestração, validação, coordenação
- **Domain**: Entidades, Value Objects, Aggregates, Domain Services, Domain Events, Repositories (interfaces)
- **Ports Out**: Contratos de saída (Repository interfaces, External Service interfaces)
- **Adapters**: Implementações concretas (MongoDB, Redis, HTTP Clients, etc.)
- **DTOs**: Data Transfer Objects nas fronteiras

### 6. Test-Driven Development (TDD)
- Ciclo: Red → Green → Refactor
- Testes como especificação executável
- Design emergente guiado por testes
- Cobertura como consequência, não meta

### 7. Behavior-Driven Development (BDD)
- Especificação por exemplo
- Gherkin como linguagem ubíqua
- Cenários como critérios de aceitação
- Colaboração triádica (Business, Dev, QA)

### 8. Modelo Espiral de Desenvolvimento
- Quatro quadrantes: Objetivos → Riscos → Desenvolvimento/Validação → Planejamento
- Iterações baseadas em risco
- Protótipos para mitigação de risco técnico

### 9. Documentação Técnica
- Architecture Decision Records (ADR)
- Diagramas C4 (Context, Container, Component, Code)
- READMEs técnicos por módulo
- Comentários de *porquê*, não de *o quê*

## Boas Práticas

1. **Preferir composição sobre herança**
2. **Programar para interfaces, não implementações**
3. **Manter domínio puro** (sem dependências de infraestrutura)
4. **Testes rápidos e isolados** (unitários) + testes de contrato/integração
5. **Nomes reveladores de intenção** (Ubiquitous Language no código)
6. **Pequenas interfaces** (Interface Segregation)
7. **Inversão de controle** via DI container
8. **Imutabilidade** preferencial para Value Objects e DTOs

## Diretrizes de Utilização

- Aplicar SOLID em todas as decisões de design
- Usar DDD para modelar complexidade de domínio
- Seguir Arquitetura Hexagonal para isolar domínio
- Escrever testes primeiro (TDD) quando aplicável
- Documentar decisões arquiteturais (ADR)
- Revisar código com foco em acoplamento/coesão

## Critérios de Qualidade

- [ ] Código segue SOLID
- [ ] Domínio isolado de infraestrutura
- [ ] Testes cobrem regras de negócio críticas
- [ ] Nomes usam linguagem ubíqua
- [ ] Dependências apontam para dentro (Clean Architecture)
- [ ] ADRs documentam decisões significativas

## Referências

- `gherkin-bdd` — especificação comportamental
- `arquitetura-hexagonal` — implementação detalhada
- `ddd-conceitos` — modelagem de domínio
- `tdd-bdd` — ciclos de desenvolvimento guiado por testes