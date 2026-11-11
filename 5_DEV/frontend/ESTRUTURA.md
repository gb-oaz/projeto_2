# Estrutura do Frontend - Vue 3 + TypeScript + Vite

## Visão Geral

Arquitetura baseada em **componentes** com organização **feature-based** (por domínio), seguindo as melhores práticas de Vue 3, TypeScript estrito, Vite, Pinia, Vue Router e Material UI 3 (MUI).

---

## Estrutura de Diretórios

```
5_DEV/frontend/
├── public/                    # Assets estáticos servidos diretamente
├── src/
│   ├── app/                   # Bootstrap da aplicação
│   │   ├── main.ts            # Ponto de entrada
│   │   ├── App.vue            # Componente raiz
│   │   ├── router/            # Vue Router + Guards de navegação
│   │   │   ├── index.ts       # Configuração de rotas
│   │   │   └── guards.ts      # Navigation guards (auth, roles, etc.)
│   │   └── providers/         # Providers globais (Pinia, MUI, i18n, QueryClient)
│   │       ├── pinia.ts
│   │       ├── mui.ts
│   │       ├── i18n.ts
│   │       └── queryClient.ts
│   │
│   ├── core/                  # Código compartilhado (cross-feature)
│   │   ├── api/               # Cliente HTTP tipado + interceptors
│   │   │   ├── client.ts      # Instância Axios configurada
│   │   │   ├── interceptors/  # Request/response interceptors
│   │   │   ├── types.ts       # Tipos base da API (ProblemDetails, PaginatedResponse)
│   │   │   └── endpoints.ts   # Funções tipadas por endpoint
│   │   ├── components/        # Componentes base reutilizáveis (wrappers MUI)
│   │   │   ├── ui/            # BaseButton, BaseInput, BaseSelect, BaseCard, BaseTable, BaseModal, BaseBadge, BaseAvatar, BaseTooltip, BaseDivider, BaseChip, BaseIcon, BaseLoader, BaseAlert, BaseDialog, BaseDrawer, BaseMenu, BaseTabs, BasePagination, BaseBreadcrumb
│   │   │   └── layout/        # Header, Sidebar, Footer, Container, PageLayout, Section, Grid, Flex
│   │   ├── composables/       # Composables globais reutilizáveis
│   │   │   ├── useApi.ts      # Hook para chamadas API com loading/error
│   │   │   ├── useAuth.ts     # Hook de autenticação (login, logout, token)
│   │   │   ├── useToast.ts    # Notificações toast
│   │   │   ├── useDebounce.ts # Debounce para inputs
│   │   │   ├── useLocalStorage.ts # Persistência no localStorage
│   │   │   ├── useMediaQuery.ts   # Breakpoints responsivos
│   │   │   └── usePermission.ts   # Verificação de permissões/roles
│   │   ├── stores/            # Stores Pinia globais
│   │   │   ├── authStore.ts   # Estado de autenticação (user, tokens, roles)
│   │   │   ├── uiStore.ts     # Estado global de UI (sidebar, theme, loading global)
│   │   │   └── notificationStore.ts # Toasts/snackbars globais
│   │   ├── styles/            # Estilos globais e tema MUI
│   │   │   ├── theme.ts       # Configuração do tema MUI (tokens, paleta, tipografia)
│   │   │   ├── variables.css  # CSS custom properties (cores, espaçamentos, breakpoints)
│   │   │   ├── globals.css    # Reset, tipografia base, utilitários globais
│   │   │   └── overrides.css  # Sobrescritas de componentes MUI
│   │   ├── types/             # Tipos TypeScript globais
│   │   │   ├── global.d.ts    # Declarações globais (Window, ImportMeta, etc.)
│   │   │   ├── api.ts         # Tipos base da API
│   │   │   ├── common.ts      # Tipos comuns (Id, Timestamp, Status, etc.)
│   │   │   └── env.d.ts       # Variáveis de ambiente (Vite)
│   │   ├── utils/             # Funções puras utilitárias
│   │   │   ├── currency.ts    # Formatação de moeda (BRL)
│   │   │   ├── date.ts        # Formatação e manipulação de datas
│   │   │   ├── validation.ts  # Validadores comuns (CPF, CNPJ, email, phone)
│   │   │   ├── string.ts      # Utilitários de string (slugify, truncate, capitalize)
│   │   │   ├── array.ts       # Utilitários de array (groupBy, sortBy, unique)
│   │   │   ├── object.ts      # Utilitários de objeto (pick, omit, deepClone)
│   │   │   └── format.ts      # Formatação genérica (números, percentuais, bytes)
│   │   └── validation/        # Schemas de validação Zod compartilhados
│   │       ├── commonSchemas.ts # Schemas base (id, email, password, pagination)
│   │       └── formSchemas.ts   # Schemas de formulários comuns
│   │
│   ├── features/              # Features por domínio de negócio (feature-first)
│   │   ├── autenticacao/      # Domínio: Autenticação e Autorização
│   │   │   ├── components/    # Componentes específicos (LoginForm, RegisterForm, ResetPasswordForm, MfaForm)
│   │   │   ├── composables/   # Composables específicos (useLogin, useRegister, usePasswordReset, useMfa)
│   │   │   ├── stores/        # Stores Pinia do domínio (authFeatureStore)
│   │   │   ├── types/         # Tipos do domínio (Credentials, User, TokenPair, MfaChallenge)
│   │   │   ├── api/           # Chamadas API do domínio (authApi.ts)
│   │   │   └── pages/         # Páginas do domínio (LoginPage, RegisterPage, ResetPasswordPage, MfaPage)
│   │   │
│   │   ├── pedidos/           # Domínio: Pedidos
│   │   │   ├── components/    # PedidoCard, PedidoTable, PedidoForm, PedidoFilters, PedidoStatusBadge, PedidoTimeline, ItemPedidoRow
│   │   │   ├── composables/   # usePedidos, usePedidoDetalhes, usePedidoFiltros, usePedidoAcoes
│   │   │   ├── stores/        # pedidoStore (lista, filtros, paginação, seleção)
│   │   │   ├── types/         # PedidoResumo, PedidoDetalhe, ItemPedido, StatusPedido, FiltrosPedido
│   │   │   ├── api/           # pedidosApi.ts (listar, buscar, criar, atualizar, cancelar, exportar)
│   │   │   └── pages/         # PedidosListPage, PedidoDetalhePage, PedidoFormPage
│   │   │
│   │   └── catalogo/          # Domínio: Catálogo de Produtos
│   │       ├── components/    # ProdutoCard, ProdutoTable, ProdutoForm, ProdutoFilters, CategoriaTree, ProdutoImageUpload
│   │       ├── composables/   # useProdutos, useCategorias, useProdutoDetalhes, useEstoque
│   │       ├── stores/        # catalogoStore (produtos, categorias, filtros, paginação)
│   │       ├── types/         # Produto, Categoria, Variante, Estoque, FiltrosCatalogo
│   │       ├── api/           # catalogoApi.ts (produtos, categorias, variantes, estoque)
│   │       └── pages/         # CatalogoListPage, ProdutoDetalhePage, ProdutoFormPage, CategoriasPage
│   │
│   ├── layouts/               # Layouts de página (wrappers de roteamento)
│   │   ├── DefaultLayout.vue  # Layout padrão com header/sidebar/footer
│   │   ├── AuthLayout.vue     # Layout para páginas de autenticação (centralizado)
│   │   ├── DashboardLayout.vue # Layout para dashboard (sidebar colapsável, métricas)
│   │   └── EmptyLayout.vue    # Layout vazio (para modais full-screen, impressão)
│   │
│   ├── pages/                 # Views roteadas (entry points das rotas)
│   │   ├── HomePage.vue
│   │   ├── NotFoundPage.vue
│   │   ├── UnauthorizedPage.vue
│   │   └── MaintenancePage.vue
│   │
│   ├── assets/                # Assets processados pelo Vite
│   │   ├── images/            # Imagens (logos, backgrounds, placeholders)
│   │   ├── fonts/             # Fontes locais (woff2)
│   │   └── icons/             # Ícones SVG customizados
│   │
│   └── tests/                 # Testes automatizados (colocados ao lado do código quando possível)
│       ├── unit/              # Testes unitários (stores, composables, utils) - Vitest
│       │   ├── core/
│       │   │   ├── stores/
│       │   │   ├── composables/
│       │   │   └── utils/
│       │   └── features/
│       │       ├── autenticacao/
│       │       ├── pedidos/
│       │       └── catalogo/
│       ├── component/         # Testes de componente (UI crítica) - Vitest + Vue Test Utils
│       │   ├── core/
│       │   │   └── components/
│       │   └── features/
│       │       ├── autenticacao/
│       │       ├── pedidos/
│       │       └── catalogo/
│       └── e2e/               # Testes End-to-End (fluxos críticos) - Playwright
│           ├── auth/
│           ├── pedidos/
│           └── catalogo/
│
├── index.html                 # HTML template
├── package.json
├── tsconfig.json              # TypeScript strict mode
├── tsconfig.node.json         # Config para Vite/Node
├── vite.config.ts             # Config Vite + PWA + aliases
├── vitest.config.ts           # Config Vitest + coverage
├── playwright.config.ts       # Config Playwright E2E
├── .eslintrc.cjs              # ESLint (Vue, TypeScript, Prettier)
├── .prettierrc                # Prettier
├── .stylelintrc               # Stylelint (CSS/SCSS)
├── .env.example               # Exemplo de variáveis de ambiente
├── .gitignore
└── README.md
```

---

## Convenções de Nomenclatura

| Tipo | Padrão | Exemplo |
|------|--------|---------|
| Diretórios | kebab-case | `pedidos`, `components/ui` |
| Arquivos Vue | PascalCase | `PedidoCard.vue`, `BaseButton.vue` |
| Arquivos TypeScript | camelCase | `pedidoStore.ts`, `usePedidos.ts` |
| Stores Pinia | camelCase + Store | `pedidoStore.ts`, `authStore.ts` |
| Composables | camelCase + use | `usePedidos.ts`, `useAuth.ts` |
| Tipos/Interfaces | PascalCase | `PedidoResumo`, `StatusPedido` |
| Constantes | UPPER_SNAKE_CASE | `API_BASE_URL`, `STATUS_PEDIDO` |
| CSS Classes | kebab-case (BEM) | `.pedido-card__status--confirmado` |

---

## Princípios Arquiteturais

1. **Feature-First**: Código organizado por domínio de negócio, não por tipo técnico
2. **Core Compartilhado**: `core/` contém apenas código verdadeiramente transversal
3. **Composition API + `<script setup>`**: Padrão obrigatório para componentes Vue
4. **TypeScript Strict**: `strict: true` no tsconfig, sem `any` explícito
5. **Barrel Exports**: `index.ts` em cada pasta para exports públicos controlados
6. **Co-location**: Testes unitários/componentes ao lado do código testado (quando viável)
7. **Path Aliases**: `@/` para `src/`, `@core/` para `src/core/`, `@features/` para `src/features/`

---

## Configuração de Aliases (vite.config.ts)

```typescript
export default defineConfig({
  resolve: {
    alias: {
      '@': '/src',
      '@core': '/src/core',
      '@features': '/src/features',
      '@app': '/src/app',
      '@layouts': '/src/layouts',
      '@pages': '/src/pages',
      '@assets': '/src/assets',
    },
  },
})
```

---

## Rastreabilidade Requisitos ↔ Código

- Cada feature em `features/` mapeia para um domínio de requisitos (Gherkin)
- Stores, composables e components da feature implementam comportamentos dos cenários
- Testes em `tests/` referenciam IDs de cenários Gherkin (ex: `@pedidos.listar`)
- Documentação em `ESTRUTURA.md` mantida sincronizada com a estrutura real

---

## Próximos Passos

1. Inicializar projeto com `npm create vite@latest . -- --template vue-ts`
2. Instalar dependências: `pinia`, `@pinia/plugin-persistedstate`, `vue-router`, `@mui/material`, `@emotion/react`, `@emotion/styled`, `axios`, `zod`, `@vee-validate/zod`, `vite-plugin-pwa`, `i18n`
3. Configurar TypeScript strict, ESLint, Prettier, Stylelint
4. Implementar tema MUI em `core/styles/theme.ts`
5. Criar componentes base em `core/components/ui/`
6. Configurar ApiClient com interceptors em `core/api/`
7. Implementar stores globais em `core/stores/`
8. Criar features iniciais: `autenticacao`, `pedidos`, `catalogo`
9. Configurar testes: Vitest (unit/component), Playwright (E2E)
10. Configurar PWA com `vite-plugin-pwa`
11. Configurar CI/CD no GitHub Actions