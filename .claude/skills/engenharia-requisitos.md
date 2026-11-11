---
name: engenharia-requisitos
description: Engenharia de requisitos: elicitação, análise, especificação, validação, gestão e rastreabilidade
metadata:
  type: specialized
  category: product
  agents: [deboaz-po, deboaz-arc, deboaz-uxui, deboaz-qa]
---

# Skill: Engenharia de Requisitos

## Responsabilidade e Escopo

Competência para elicitação, análise, especificação, validação e gestão de requisitos de software. Foco em transformar necessidades de negócio em requisitos claros, testáveis e rastreáveis.

## Conhecimentos Técnicos

### 1. Tipos de Requisitos
| Tipo | Descrição | Exemplos |
|---|---|---|
| **Funcionais** | O que o sistema deve fazer | Autenticação, cadastro, relatórios |
| **Não Funcionais** | Qualidades do sistema | Performance, segurança, usabilidade |
| **De Negócio** | Regras e políticas organizacionais | Limite de crédito, aprovação hierárquica |
| **De Interface** | Interações com sistemas externos | APIs REST, webhooks, EDI |
| **De Restrição** | Limitações impostas | Tecnologia, orçamento, prazo, conformidade |

### 2. Técnicas de Elicitação
- Entrevistas estruturadas e semiestruturadas
- Workshops colaborativos (Event Storming, Example Mapping)
- Análise de documentação existente
- Observação e shadowing
- Prototipação e validação precoce
- Análise de concorrentes/benchmarking

### 3. Especificação de Requisitos
#### User Stories (Histórias de Usuário)
```
Como [ator/role]
Eu quero [ação/função]
Para que [benefício/valor]
```

**Critérios INVEST:**
- **I**ndependent — independente de outras stories
- **N**egotiable — negociável, não contrato rígido
- **V**aluable — entrega valor ao usuário/negócio
- **E**stimable — estimável pela equipe
- **S**mall — pequena o suficiente para uma iteração
- **T**estable — testável com critérios claros

#### Critérios de Aceitação em Gherkin
```gherkin
Feature: [Funcionalidade]
  Rule: [Regra de negócio]
  
  Background:
    Given [pré-condição comum]
  
  Scenario: [Cenário principal]
    Given [contexto]
    When [ação]
    Then [resultado esperado]
```

### 4. Modelagem de Processos de Negócio
- BPMN 2.0 para fluxos formais
- Fluxogramas simples para visão geral
- Event Storming para descoberta de domínio
- User Journey Maps para perspectiva do usuário

### 5. Priorização de Requisitos
| Método | Quando Usar |
|---|---|
| MoSCoW (Must/Should/Could/Won't) | Priorização rápida com stakeholders |
| WSJF (Weighted Shortest Job First) | Otimização de fluxo de valor (SAFe) |
| Kano Model | Diferenciação: Básico, Performance, Encantador |
| Value vs Effort Matrix | Decisões táticas de backlog |
| RICE (Reach, Impact, Confidence, Effort) | Produto orientado a métricas |

### 6. Rastreabilidade de Requisitos
- **Forward**: Requisito → Arquitetura → Código → Testes
- **Backward**: Teste → Código → Arquitetura → Requisito
- Matriz de rastreabilidade bidirecional
- Ferramentas: Links em Markdown, tags em issues, matrizes em planilhas

### 7. Gestão de Backlog
- Product Backlog ordenado por valor/risco/dependência
- Definition of Ready (DoR) para entrada no Sprint
- Definition of Done (DoD) para conclusão
- Refinamento contínuo (Backlog Refinement)
- Story Mapping para visão holística

### 8. Validação e Verificação
- **Validação**: "Estamos construindo o produto certo?" (com stakeholders)
- **Verificação**: "Estamos construindo o produto corretamente?" (com specs)
- Inspeções, walkthroughs, revisões formais
- Testes de aceitação baseados em Gherkin

### 9. Gestão de Mudanças
- Change Control Board (CCB) para mudanças de escopo
- Análise de impacto (arquitetura, testes, prazo, custo)
- Versionamento de requisitos (semântico ou sequencial)
- Comunicação de mudanças aos stakeholders

## Boas Práticas

1. **Escrever requisitos em linguagem do negócio** (Ubiquitous Language)
2. **Um requisito = um comportamento testável**
3. **Evitar soluções técnicas nos requisitos** (o *qué*, não o *cómo*)
4. **Validar com exemplos concretos** (Specification by Example)
5. **Manter rastreabilidade viva** (atualizada a cada mudança)
6. **Priorizar valor de negócio**, não complexidade técnica
7. **Dizer "não" explicitamente** (Won't do no MoSCoW)
8. **Refinar continuamente** — requisitos evoluem

## Diretrizes de Utilização

### Para PO (deboaz-po) — Responsável Principal
- Elicitar, especificar, priorizar e validar requisitos
- Escrever User Stories e critérios Gherkin
- Gerenciar Product Backlog
- Decidir sobre mudanças de escopo
- Comunicar "porquê" para a equipe

### Para ARC (deboaz-arc) — Consumidor
- Analisar requisitos para decisões arquiteturais
- Identificar requisitos não funcionais (NFRs)
- Mapear requisitos a componentes/ports
- Avaliar viabilidade técnica e trade-offs

### Para UX/UI (deboaz-uxui) — Consumidor
- Transformar requisitos em jornadas/fluxos
- Identificar necessidades de usabilidade/acessibilidade
- Prototipar para validar entendimento

### Para QA (deboaz-qa) — Consumidor/Validador
- Derivar testes de aceitação dos critérios Gherkin
- Identificar cenários não cobertos
- Validar se implementação atende aos requisitos
- Reportar gaps e ambiguidades

## Critérios de Qualidade

- [ ] Requisitos seguem formato User Story + Gherkin
- [ ] Critérios de aceitação são testáveis (pass/fail claro)
- [ ] Rastreabilidade bidirecional mantida
- [ ] Priorização documentada com justificativa
- [ ] Mudanças passam por análise de impacto
- [ ] Stakeholders validam antes de desenvolvimento
- [ ] Backlog refinado regularmente (DoR atendido)

## Referências

- `gherkin-bdd` — linguagem para critérios de aceitação
- `product-ownership` — gestão de produto
- `modelagem-processos` — BPMN, Event Storming
- `estrategia-qa` — derivação de testes