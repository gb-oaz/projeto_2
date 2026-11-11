---
name: ux-ui-design
description: UX/UI Design: usabilidade, acessibilidade, design system, prototipação, user journeys, Material Design 3
metadata:
  type: specialized
  category: design
  agents: [deboaz-uxui, deboaz-frontend]
---

# Skill: UX/UI Design

## Responsabilidade e Escopo

Competência para design de experiência do usuário (UX) e interface do usuário (UI). Abrange pesquisa, heurísticas de usabilidade, acessibilidade, design system (Material Design 3), prototipação, user journeys, validação e entrega para desenvolvimento.

## Conhecimentos Técnicos

### 1. Heurísticas de Usabilidade de Nielsen (10 Princípios)
| # | Heurística | Aplicação Prática |
|---|---|---|
| 1 | **Visibilidade do status do sistema** | Feedback imediato (loaders, toasts, skeletons), progress indicators |
| 2 | **Compatibilidade sistema-mundo real** | Linguagem do usuário (não jargão técnico), ícones reconhecíveis |
| 3 | **Controle e liberdade do usuário** | Undo/Redo, "Cancelar", navegação clara, sair de modais |
| 4 | **Consistência e padrões** | Design System, padrões de navegação, nomenclatura unificada |
| 5 | **Prevenção de erros** | Validação inline, confirmações destrutivas, constraints de input |
| 6 | **Reconhecimento em vez de memorização** | Menus visíveis, breadcrumbs, autocomplete, histórico recente |
| 7 | **Flexibilidade e eficiência de uso** | Atalhos de teclado, power user features, personalização |
| 8 | **Estética e design minimalista** | Remover ruído visual, hierarquia clara, whitespace intencional |
| 9 | **Ajuda a reconhecer, diagnosticar, recuperar erros** | Mensagens de erro claras (o quê, por quê, como corrigir), inline validation |
| 10 | **Ajuda e documentação** | Tooltips contextuais, onboarding, help center, empty states guiados |

### 2. Acessibilidade Digital (WCAG 2.1 AA)
#### Princípios POUR
- **Perceptível**: Text alternatives, captions, adaptável, distinguível (contraste 4.5:1)
- **Operável**: Teclado, tempo suficiente, sem convulsões, navegável, inputs modais
- **Compreensível**: Legível, previsível, ajuda a entrada (labels, instruções, validação)
- **Robusto**: Compatível com tecnologias assistivas (ARIA, HTML semântico)

#### Checklist Prático
- [ ] Contraste de cores ≥ 4.5:1 (texto normal), ≥ 3:1 (texto grande/UI)
- [ ] Navegação 100% por teclado (Tab, Enter, Esc, Setas)
- [ ] Foco visível (`:focus-visible`) em todos elementos interativos
- [ ] Labels associados a inputs (`<label for>` ou `aria-label`)
- [ ] ARIA landmarks (`main`, `nav`, `aside`, `header`, `footer`)
- [ ] Live regions para updates dinâmicos (`aria-live="polite"`)
- [ ] Textos alternativos em imagens informativas
- [ ] Heading hierarchy (h1→h2→h3) sem saltos
- [ ] Redução de movimento respeitada (`prefers-reduced-motion`)
- [ ] Zoom 200% sem perda de funcionalidade

### 3. Design System (Material Design 3 + MUI)
#### Tokens de Design
```typescript
// core/styles/tokens.ts
export const tokens = {
  color: {
    primary: { main: '#1976d2', light: '#42a5f5', dark: '#1565c0', contrastText: '#fff' },
    secondary: { main: '#9c27b0', light: '#ba68c8', dark: '#7b1fa2', contrastText: '#fff' },
    error: { main: '#d32f2f', light: '#ef5350', dark: '#c62828' },
    warning: { main: '#ed6c02', light: '#ff9800', dark: '#e65100' },
    success: { main: '#2e7d32', light: '#4caf50', dark: '#1b5e20' },
    info: { main: '#0288d1', light: '#03a9f4', dark: '#01579b' },
    neutral: { 
      50: '#fafafa', 100: '#f5f5f5', 200: '#eeeeee', 300: '#e0e0e0',
      400: '#bdbdbd', 500: '#9e9e9e', 600: '#757575', 700: '#616161',
      800: '#424242', 900: '#212121', 950: '#121212'
    },
    background: { default: '#fafafa', paper: '#ffffff', dark: '#121212' }
  },
  spacing: (factor: number) => `${factor * 8}px`, // 8px base unit
  typography: {
    fontFamily: '"Inter", "Roboto", "Helvetica", "Arial", sans-serif',
    fontWeightLight: 300, fontWeightRegular: 400, fontWeightMedium: 500, fontWeightBold: 600,
    h1: { size: '2.5rem', weight: 600, lineHeight: 1.2 },
    h2: { size: '2rem', weight: 600, lineHeight: 1.3 },
    h3: { size: '1.75rem', weight: 600, lineHeight: 1.3 },
    h4: { size: '1.5rem', weight: 500, lineHeight: 1.4 },
    h5: { size: '1.25rem', weight: 500, lineHeight: 1.4 },
    h6: { size: '1rem', weight: 500, lineHeight: 1.5 },
    body1: { size: '1rem', weight: 400, lineHeight: 1.6 },
    body2: { size: '0.875rem', weight: 400, lineHeight: 1.5 },
    caption: { size: '0.75rem', weight: 400, lineHeight: 1.5 },
    button: { size: '0.875rem', weight: 500, lineHeight: 1.75, textTransform: 'none' }
  },
  shape: { borderRadius: { sm: 4, md: 8, lg: 12, xl: 16, full: 9999 } },
  shadows: {
    0: 'none', 1: '0 1px 2px rgba(0,0,0,0.05)', 2: '0 1px 3px rgba(0,0,0,0.1)',
    3: '0 4px 6px rgba(0,0,0,0.07)', 4: '0 10px 15px rgba(0,0,0,0.1)',
    5: '0 20px 25px rgba(0,0,0,0.15)'
  },
  breakpoints: { xs: 0, sm: 600, md: 900, lg: 1200, xl: 1536 },
  transitions: { fast: '150ms ease', standard: '250ms ease', slow: '350ms ease' },
  zIndex: { drawer: 1200, modal: 1300, snackbar: 1400, tooltip: 1500 }
}
```

#### Componentes Base (Core/UI)
- `BaseButton` — variants: contained, outlined, text, tonal; sizes: sm, md, lg
- `BaseInput` / `BaseTextField` — label, helper, error, prefix/suffix, masked
- `BaseSelect` / `BaseAutocomplete` — searchable, multiple, chips
- `BaseCard` — elevated, outlined, interactive, media
- `BaseTable` — sorting, pagination, selection, expandable rows
- `BaseDialog` / `BaseBottomSheet` — responsive, focus trap
- `BaseSnackbar` / `BaseToast` — severity, action, persist
- `BaseAvatar` — image, fallback, status indicator
- `BaseChip` — filter, input, clickable, deletable
- `BaseTabs` / `BaseStepper` — horizontal, vertical, mobile responsive

### 4. Arquitetura da Informação
#### Sitemap / Navegação
- **Global Navigation**: Header (logo, busca, usuário, notificações), Sidebar (módulos principais)
- **Local Navigation**: Tabs, Breadcrumbs, Stepper (wizards)
- **Contextual**: Action menus, Dropdowns, Tooltips

#### Taxonomia de Conteúdo
- Agrupamento por **tarefa do usuário** (não por tecnologia)
- Card sorting com usuários reais
- Máximo 7±2 itens por nível (Miller's Law)

### 5. User Journey & User Flow
#### User Journey Map
```
ETAPA          | AÇÕES DO USUÁRIO        | PENSAMENTOS/SENTIMENTOS    | PONTOS DE CONTATO      | OPORTUNIDADES
---------------|-------------------------|----------------------------|------------------------|----------------------
Descoberta     | Busca no Google         | "Será que resolve meu..."  | Landing page, Ads      | SEO, Value prop claro
Consideração   | Compara alternativas    | "Preciso confiar"          | Demo, Cases, Reviews   | Trial grátis, Depoimentos
Onboarding     | Cadastro, primeiro uso  | "Espero que seja fácil"    | Signup, Tutorial       | Progressive disclosure
Ativação       | Realiza tarefa chave    | "Consegui!"                | Feature principal      | Success state, Next steps
Retenção       | Uso recorrente          | "Faz parte da rotina"      | Dashboard, Alertas     | Personalização, Insights
Advocacia      | Indica para outros      | "Vale a pena"              | Share, NPS             | Referral program
```

#### User Flow (Ex: Criar Pedido)
```
[Login] → [Dashboard] → [Novo Pedido] → [Selecionar Cliente] 
    → [Adicionar Itens] ←(busca) [Catálogo] 
    → [Revisar Resumo] → [Confirmar] → [Sucesso] → [Lista Pedidos]
         ↑                                                      ↓
         └────────────── (Editar/Cancelar) ← [Detalhes Pedido] ┘
```

### 6. Prototipação
#### Níveis de Fidelidade
| Nível | Ferramenta | Quando Usar |
|---|---|---|
| **Baixa (Wireframe)** | Papel, Figma (wireframe kit), Excalidraw | Descoberta, alinhamento rápido, validação de fluxo |
| **Média (Mockup)** | Figma, Sketch | Validação visual, design system, handoff inicial |
| **Alta (Protótipo Interativo)** | Figma (prototyping), Storybook | Testes de usabilidade, validação com stakeholders, dev handoff |

#### Entrega para Desenvolvimento (Design Handoff)
- **Figma**: Components publicados, Variants, Auto-layout, Design tokens (variables)
- **Especificações**: Spacing, Typography, Colors, States (hover, focus, disabled, loading)
- **Assets**: SVG otimizados, PNG @2x/@3x, Fontes (WOFF2)
- **Documentação**: Storybook stories para cada componente base
- **Acessibilidade**: Anotações de ARIA, ordem de foco, contrastes

### 7. Avaliação de Usabilidade
#### Métodos
- **Teste de Usabilidade Moderado** (5 usuários = 85% dos problemas — Nielsen)
- **Teste Não Moderado** (Maze, UserTesting, escala)
- **Heuristic Evaluation** (3-5 avaliadores independentes)
- **Cognitive Walkthrough** (foco em learnability)
- **A/B Testing** (validação quantitativa em produção)
- **Analytics + Heatmaps** (Hotjar, FullStory — comportamento real)

#### Métricas de Usabilidade
- **Task Success Rate** (% usuários que completam)
- **Time on Task** (eficiência)
- **Error Rate** (frequência/tipo de erros)
- **SUS (System Usability Scale)** — score 0-100 (≥ 68 = aceitável)
- **NPS** (Net Promoter Score)
- **CSAT** (Customer Satisfaction)

### 8. PWA — Experiência do Usuário
- **Installability**: Prompt nativo, ícones adaptativos, splash screen
- **Offline**: Cache-first para assets, Network-first para API, fallback gracioso
- **Performance**: LCP < 2.5s, FID < 100ms, CLS < 0.1
- **Engajamento**: Push notifications (com permissão), Background sync
- **Atualização**: Auto-update com notificação "Nova versão disponível"

## Processo de Design Integrado

### Dual-Track Agile (Discovery + Delivery)
```
DISCOVERY (Contínuo)                          DELIVERY (Sprint)
├── Research: Entrevistas, Analytics          ├── Design Handoff (Figma → Storybook)
├── Ideation: Crazy 8s, Story Mapping         ├── Dev Implementation
├── Prototyping: Wireframes → Hi-fi           ├── Design QA (revisão visual/UX)
├── Validation: Usability Tests               ├── Acceptance (PO + UX)
└── Specification: Specs, Tokens, Flows       └── Release
```

### Critérios de Pronto para Dev (Design DoR)
- [ ] Fluxo validado com PO e stakeholder
- [ ] Protótipo interativo em Figma (link compartilhado)
- [ ] Componentes usam Design System (tokens, variants)
- [ ] Estados documentados: default, hover, focus, disabled, loading, error, empty
- [ ] Acessibilidade anotada (ARIA, foco, contraste)
- [ ] Responsividade: breakpoints mobile (≤600), tablet (600-900), desktop (>900)
- [ ] Assets exportados (SVG otimizado, fontes WOFF2)
- [ ] Copy final revisado (português, tom de voz consistente)

## Boas Práticas

1. **Design para o caso real**, não para o caso ideal (empty states, erros, loading, dados longos)
2. **Mobile-first** — projetar para menor viewport primeiro, expandir progressivamente
3. **Consistência via Design System** — não reinventar componentes base
4. **Validar cedo, validar frequentemente** — wireframe → teste → mockup → teste → hi-fi
5. **Acessibilidade nativa** — não "adicionar depois", construir acessível desde o início
6. **Handoff colaborativo** — dev review durante design, design review durante dev
7. **Documentar decisões** — ADR leve para escolhas de UX significativas
8. **Métricas pós-lançamento** — acompanhar adoção, erros, satisfação

## Critérios de Qualidade

- [ ] Heurísticas de Nielsen atendidas nos fluxos principais
- [ ] WCAG 2.1 AA validado (axe-core, testes manuais teclado/screen reader)
- [ ] Design System usado consistentemente (0 componentes customizados desnecessários)
- [ ] Protótipo interativo validado com ≥ 5 usuários representativos
- [ ] Handoff completo: specs, tokens, assets, estados, copy, acessibilidade
- [ ] Design QA aprovado antes de merge para main
- [ ] Métricas de usabilidade baseline estabelecidas para comparação pós-release

## Referências

- `desenvolvimento-frontend` — implementação Vue.js + MUI
- `gherkin-bdd` — requisitos comportamentais como base para jornadas
- `acessibilidade` — diretrizes técnicas WCAG
- `design-system` — tokens, componentes, documentação