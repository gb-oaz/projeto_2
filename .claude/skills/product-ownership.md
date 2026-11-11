---
name: product-ownership
description: Product Ownership: visão de produto, gestão de backlog, stakeholder management, métricas de valor
metadata:
  type: specialized
  category: product
  agents: [deboaz-po]
---

# Skill: Product Ownership

## Responsabilidade e Escopo

Competência completa de Product Ownership para o agente `deboaz-po`. Abrange visão estratégica, gestão tática de backlog, colaboração com stakeholders e maximização de valor do produto.

## Conhecimentos Técnicos

### 1. Visão e Estratégia de Produto
- **Product Vision**: Declaração clara de propósito, público-alvo, diferencial
- **Product Strategy**: Objetivos mensuráveis (OKRs), roadmap temático, horizonte temporal
- **Product Discovery**: Dual-track (Discovery + Delivery), validação contínua
- **Jobs-to-be-Done (JTBD)**: Entender progresso que usuário busca, não features
- **Product-Market Fit**: Indicadores de adoção, retenção, NPS, receita

### 2. Gestão de Backlog
#### Product Backlog
- Única fonte de trabalho para a equipe
- Ordenado por: Valor de negócio > Risco > Dependências > Esforço
- Itens prontos para Sprint (DoR) no topo
- Refinamento contínuo (10-15% da capacidade da equipe)

#### Sprint Backlog
- Compromisso da equipe para a Sprint
- Negociável durante a Sprint (com PO)
- Atualizado diariamente

#### Backlog Refinement
- Quebrar épicos em stories entregáveis
- Adicionar critérios de aceitação (Gherkin)
- Estimar (Story Points, T-shirt sizes, ou #noestimates)
- Identificar dependências e riscos

### 3. Stakeholder Management
| Stakeholder | Necessidade | Frequência |
|---|---|---|
| Negócio/Vendas | Roadmap, datas, features | Semanal/Quinzenal |
| Desenvolvimento | Clareza, viabilidade, prioridades | Diária (standup) + Refinamento |
| UX/UI | Contexto de uso, validação | Por feature |
| QA | Critérios testáveis, dados de teste | Por story |
| Operações/Segurança | NFRs, compliance, deploy | Por release |

- **Comunicação proativa**: Status, riscos, decisões, trade-offs
- **Gerenciar expectativas**: Dizer "não" ou "ainda não" com justificativa
- **Feedback loops**: Demo/Review, métricas de uso, suporte

### 4. Métricas de Produto
#### Métricas de Saída (Output)
- Velocity, Throughput, Cycle Time, Lead Time
- Sprint Goal達成率, Release Predictability

#### Métricas de Resultado (Outcome)
- **Aquisição**: CAC, conversão trial→pago, signups
- **Ativação**: Time-to-value, onboarding completion
- **Retenção**: DAU/MAU, churn rate, feature adoption
- **Receita**: MRR/ARR, LTV, expansion revenue
- **Satisfação**: NPS, CSAT, CES

#### North Star Metric
Uma métrica que melhor captura o valor central entregue ao cliente

### 5. Release Planning
- **Release Train** / cadência fixa vs. Continuous Delivery
- **Minimum Viable Product (MVP)** vs. **Minimum Marketable Features (MMF)**
- Feature flags para deploy desacoplado de release
- Rollback plan e comunicação de incidentes

### 6. Colaboração com a Equipe Técnica
- **Sprint Planning**: Apresentar objetivo, negociar escopo
- **Daily Standup**: Disponível para clarificações (não status report)
- **Refinement**: Preparar stories, responder dúvidas
- **Sprint Review**: Demonstrar increment, coletar feedback
- **Retrospective**: Participar como membro da equipe

### 7. Tomada de Decisão
- **Data-informed**: Métricas + insights qualitativos + intuição experiente
- **Reversível vs. Irreversível** (Type 1 vs Type 2 decisions — Bezos)
- **Bias for action**: Decidir com ~70% da informação
- **Escalation path**: Quando PO não decide, quem decide?

## Boas Práticas

1. **Maximizar valor, não output** — outcomes > features entregues
2. **Backlog saudável**: 2-3 sprints de trabalho refinado à frente
3. **Um Product Goal por Sprint** — foco coerente
4. **Critérios de aceitação em Gherkin** — compartilhados com Dev/QA
5. **Validar antes de construir** — protótipos, fake doors, concierge MVP
6. **Technical debt como item de backlog** — visível, priorizado, comunicado
7. **Say "no" by default** — proteger foco e capacidade da equipe
8. **Celebrar aprendizado** — experimentos falhos são dados, não desperdício

## Diretrizes de Utilização

### Atividades Principais do PO
1. **Descoberta contínua**: Entrevistas, analytics, suporte, concorrência
2. **Escrita de requisitos**: User Stories + Gherkin em Markdown
3. **Priorização**: WSJF/MoSCoW + alinhamento estratégico
4. **Refinamento**: Preparar stories com equipe (Dev, UX, QA)
5. **Validação**: Aceitar/rejeitar increment na Review
6. **Comunicação**: Roadmap, status, decisões para stakeholders

### Entregáveis
- Product Vision & Strategy doc
- Product Backlog (ordenado, refinado)
- Sprint Goals claros
- Release Notes / Changelog
- Métricas dashboard (Outcome > Output)
- Stakeholder updates regulares

## Critérios de Qualidade

- [ ] Product Vision clara e compartilhada
- [ ] Backlog ordenado por valor, não por pedido
- [ ] Stories atendem INVEST + DoR
- [ ] Critérios Gherkin validados com Dev/QA
- [ ] Métricas de outcome rastreadas e revisadas
- [ ] Stakeholders informados, não surpreendidos
- [ ] Technical debt visível e gerenciado
- [ ] Decisões documentadas (ADR leve para produto)

## Referências

- `gherkin-bdd` — critérios de aceitação
- `engenharia-requisitos` — especificação e rastreabilidade
- `modelagem-processos` — fluxos de negócio
- `estrategia-qa` — validação de aceitação