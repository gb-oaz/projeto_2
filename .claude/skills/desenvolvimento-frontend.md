---
name: desenvolvimento-frontend
description: Frontend: Vue.js 3, TypeScript, Vite, Material UI 3, PWA, testes, arquitetura de componentes
metadata:
  type: specialized
  category: frontend
  agents: [deboaz-frontend, deboaz-uxui]
---

# Skill: Desenvolvimento Frontend (Vue.js + TypeScript)

## Responsabilidade e Escopo

Competência técnica para desenvolvimento de aplicações frontend modernas com Vue.js 3, TypeScript, Vite, Material UI 3 (MUI), PWA, arquitetura de componentes, gerenciamento de estado, integração com APIs REST, acessibilidade e testes.

## Stack Principal

| Tecnologia | Versão | Uso |
|---|---|---|
| Vue.js | 3.4+ (Composition API) | Framework reativo |
| TypeScript | 5.3+ | Tipagem estática |
| Vite | 5+ | Build tool & dev server |
| Material UI (MUI) | 5+ (Material Design 3) | Component library |
| Pinia | 2.1+ | State management |
| Vue Router | 4.3+ | Roteamento SPA |
| Axios / Fetch | - | HTTP client |
| Vitest | 1+ | Testes unitários |
| Vue Test Utils | 2+ | Testes de componentes |
| Cypress / Playwright | Latest | Testes E2E |
| ESLint + Prettier | Latest | Linting & formatação |
| Stylelint | Latest | CSS/SCSS linting |
| Workbox | 7+ | PWA Service Worker |

## Arquitetura de Pastas (Referência)

```
src/
├── app/                      # Bootstrap da aplicação
│   ├── main.ts               # Entry point
│   ├── App.vue               # Root component
│   ├── router/               # Vue Router config
│   │   ├── index.ts
│   │   ├── routes.ts
│   │   └── guards.ts         # Auth guards, etc.
│   └── providers/            # Providers globais (Pinia, MUI, i18n)
│
├── core/                     # Código compartilhado, independente de features
│   ├── api/                  # Cliente HTTP, interceptors, tipos base
│   │   ├── client.ts         # Axios instance configurada
│   │   ├── interceptors/     # Auth, error handling, logging
│   │   └── types.ts          # ApiResponse, PaginatedResponse, ErrorResponse
│   ├── components/           # Componentes base reutilizáveis (Button, Input, Card, etc.)
│   │   ├── ui/               # Wrappers MUI customizados
│   │   └── layout/           # Header, Footer, Sidebar, Container
│   ├── composables/          # Composables compartilhados (useApi, useAuth, useToast, etc.)
│   ├── constants/            # Constantes da aplicação
│   ├── hooks/                # Vue hooks personalizados
│   ├── plugins/              # Plugins Vue (MUI, i18n, PWA, etc.)
│   ├── stores/               # Stores Pinia globais (auth, ui, notification)
│   ├── styles/               # Estilos globais, theme MUI, variáveis CSS
│   ├── types/                # Tipos TypeScript globais
│   ├── utils/                # Funções utilitárias puras
│   └── validation/           # Schemas Zod/Yup compartilhados
│
├── features/                 # Features organizadas por domínio de negócio
│   ├── autenticacao/
│   │   ├── components/       # LoginForm, RegisterForm, etc.
│   │   ├── composables/      # useLogin, useRegister
│   │   ├── stores/           # useAuthStore
│   │   ├── types/            # LoginRequest, User, TokenResponse
│   │   ├── api/              # authApi.ts (endpoints tipados)
│   │   └── pages/            # LoginPage, RegisterPage
│   ├── pedidos/
│   │   ├── components/
│   │   ├── composables/
│   │   ├── stores/
│   │   ├── types/
│   │   ├── api/
│   │   └── pages/
│   └── catalogo/
│       └── ...
│
├── layouts/                  # Layouts de página (DefaultLayout, AuthLayout, DashboardLayout)
│
├── pages/                    # Páginas roteadas (views)
│   ├── HomePage.vue
│   ├── NotFoundPage.vue
│   └── ...
│
└── assets/                   # Assets estáticos (imagens, fontes, ícones)
```

## Padrões de Implementação

### 1. Composition API + `<script setup>` + TypeScript
```vue
<!-- features/pedidos/components/PedidoCard.vue -->
<script setup lang="ts">
import { computed } from 'vue'
import type { PedidoResumo } from '@/features/pedidos/types'
import { formatCurrency } from '@/core/utils/currency'
import { usePedidoStore } from '@/features/pedidos/stores/pedidoStore'
import BaseButton from '@/core/components/ui/BaseButton.vue'
import BaseBadge from '@/core/components/ui/BaseBadge.vue'

interface Props {
  pedido: PedidoResumo
  onDetalhes?: (id: string) => void
}

const props = withDefaults(defineProps<Props>(), {
  onDetalhes: () => {}
})

const pedidoStore = usePedidoStore()

const statusColor = computed(() => {
  const colors: Record<string, 'success' | 'warning' | 'error' | 'info'> = {
    rascunho: 'info',
    confirmado: 'success',
    em_preparacao: 'warning',
    enviado: 'info',
    entregue: 'success',
    cancelado: 'error'
  }
  return colors[props.pedido.status] || 'default'
})

const handleDetalhes = () => props.onDetalhes?.(props.pedido.id)
</script>

<template>
  <Card class="pedido-card" variant="outlined">
    <CardHeader>
      <CardTitle class="pedido-id">Pedido #{{ pedido.id.slice(0, 8) }}</CardTitle>
      <BaseBadge :color="statusColor" :label="pedido.status" />
    </CardHeader>
    <CardContent>
      <div class="pedido-info">
        <div><strong>Cliente:</strong> {{ pedido.clienteNome }}</div>
        <div><strong>Total:</strong> {{ formatCurrency(pedido.total) }}</div>
        <div><strong>Data:</strong> {{ new Date(pedido.criadoEm).toLocaleDateString('pt-BR') }}</div>
      </div>
    </CardContent>
    <CardActions>
      <BaseButton @click="handleDetalhes" variant="outlined" size="small">
        Ver Detalhes
      </BaseButton>
    </CardActions>
  </Card>
</template>

<style scoped>
.pedido-card { min-width: 280px; }
.pedido-id { font-family: monospace; font-size: 0.875rem; }
.pedido-info div { margin-bottom: 4px; }
</style>
```

### 2. API Client Tipado com Interceptors
```typescript
// core/api/client.ts
import axios, { AxiosError, AxiosInstance, InternalAxiosRequestConfig } from 'axios'
import { useAuthStore } from '@/core/stores/authStore'
import { useNotificationStore } from '@/core/stores/notificationStore'

const API_BASE_URL = import.meta.env.VITE_API_BASE_URL || '/api'

export const apiClient: AxiosInstance = axios.create({
  baseURL: API_BASE_URL,
  timeout: 30000,
  headers: { 'Content-Type': 'application/json' }
})

// Request Interceptor - Auth Token
apiClient.interceptors.request.use(
  (config: InternalAxiosRequestConfig) => {
    const authStore = useAuthStore()
    if (authStore.accessToken) {
      config.headers.Authorization = `Bearer ${authStore.accessToken}`
    }
    return config
  },
  (error) => Promise.reject(error)
)

// Response Interceptor - Error Handling + Token Refresh
let isRefreshing = false
let failedQueue: Array<{
  resolve: (value: unknown) => void
  reject: (reason: unknown) => void
}> = []

const processQueue = (error: Error | null) => {
  failedQueue.forEach(({ resolve, reject }) => {
    error ? reject(error) : resolve(null)
  })
  failedQueue = []
}

apiClient.interceptors.response.use(
  (response) => response,
  async (error: AxiosError) => {
    const originalRequest = error.config as InternalAxiosRequestConfig & { _retry?: boolean }
    
    if (error.response?.status === 401 && !originalRequest._retry) {
      if (isRefreshing) {
        return new Promise((resolve, reject) => {
          failedQueue.push({ resolve, reject })
        }).then(() => apiClient(originalRequest))
      }
      
      originalRequest._retry = true
      isRefreshing = true
      
      try {
        const authStore = useAuthStore()
        await authStore.refreshAccessToken()
        processQueue(null)
        return apiClient(originalRequest)
      } catch (refreshError) {
        processQueue(refreshError as Error)
        authStore.logout()
        return Promise.reject(refreshError)
      } finally {
        isRefreshing = false
      }
    }
    
    // Error handling padronizado
    const notificationStore = useNotificationStore()
    const message = error.response?.data?.detail || error.message || 'Erro inesperado'
    notificationStore.showError(message)
    
    return Promise.reject(error)
  }
)

// Typed API helpers
export const api = {
  get: <T>(url: string, params?: object) => 
    apiClient.get<T>(url, { params }).then(r => r.data),
  post: <T>(url: string, data?: object) => 
    apiClient.post<T>(url, data).then(r => r.data),
  put: <T>(url: string, data?: object) => 
    apiClient.put<T>(url, data).then(r => r.data),
  patch: <T>(url: string, data?: object) => 
    apiClient.patch<T>(url, data).then(r => r.data),
  delete: <T>(url: string) => 
    apiClient.delete<T>(url).then(r => r.data)
}
```

### 3. Feature API Tipada (Endpoints)
```typescript
// features/pedidos/api/pedidosApi.ts
import { api } from '@/core/api/client'
import type { 
  PedidoResumo, 
  PedidoDetalhado, 
  CriarPedidoRequest, 
  AtualizarPedidoRequest,
  ListarPedidosParams,
  PaginatedResponse 
} from '@/features/pedidos/types'

export const pedidosApi = {
  listar: (params: ListarPedidosParams = {}) => 
    api.get<PaginatedResponse<PedidoResumo>>('/pedidos', params),
  
  obter: (id: string) => 
    api.get<PedidoDetalhado>(`/pedidos/${id}`),
  
  criar: (data: CriarPedidoRequest) => 
    api.post<PedidoDetalhado>('/pedidos', data),
  
  atualizar: (id: string, data: AtualizarPedidoRequest) => 
    api.put<PedidoDetalhado>(`/pedidos/${id}`, data),
  
  cancelar: (id: string) => 
    api.post<void>(`/pedidos/${id}/cancelar`),
  
  obterHistorico: (id: string) => 
    api.get<PedidoHistorico[]>(`/pedidos/${id}/historico`)
}
```

### 4. Pinia Store (State Management)
```typescript
// features/pedidos/stores/pedidoStore.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import { pedidosApi } from '@/features/pedidos/api/pedidosApi'
import type { PedidoResumo, PedidoDetalhado, ListarPedidosParams } from '@/features/pedidos/types'
import { useNotificationStore } from '@/core/stores/notificationStore'

export const usePedidoStore = defineStore('pedidos', () => {
  // State
  const pedidos = ref<PedidoResumo[]>([])
  const pedidoAtual = ref<PedidoDetalhado | null>(null)
  const loading = ref(false)
  const error = ref<string | null>(null)
  const pagination = ref({ page: 1, limit: 20, total: 0, totalPages: 0 })
  const filtros = ref<ListarPedidosParams>({})

  // Getters
  const pedidosPorStatus = computed(() => {
    const grouped: Record<string, PedidoResumo[]> = {}
    pedidos.value.forEach(p => {
      (grouped[p.status] ||= []).push(p)
    })
    return grouped
  })

  const totalPedidos = computed(() => pedidos.value.length)

  // Actions
  async function carregarPedidos(params?: ListarPedidosParams) {
    loading.value = true
    error.value = null
    try {
      const queryParams = { ...filtros.value, ...params }
      const response = await pedidosApi.listar(queryParams)
      pedidos.value = response.items
      pagination.value = {
        page: response.page,
        limit: response.limit,
        total: response.total,
        totalPages: response.totalPages
      }
      if (params) filtros.value = queryParams
    } catch (e) {
      error.value = 'Falha ao carregar pedidos'
      throw e
    } finally {
      loading.value = false
    }
  }

  async function obterPedido(id: string) {
    loading.value = true
    try {
      pedidoAtual.value = await pedidosApi.obter(id)
      return pedidoAtual.value
    } catch (e) {
      error.value = 'Falha ao carregar pedido'
      throw e
    } finally {
      loading.value = false
    }
  }

  async function criarPedido(data: CriarPedidoRequest) {
    loading.value = true
    try {
      const novo = await pedidosApi.criar(data)
      pedidos.value.unshift(novo)
      pagination.value.total++
      notificationStore.showSuccess('Pedido criado com sucesso')
      return novo
    } catch (e) {
      throw e
    } finally {
      loading.value = false
    }
  }

  async function cancelarPedido(id: string) {
    try {
      await pedidosApi.cancelar(id)
      const idx = pedidos.value.findIndex(p => p.id === id)
      if (idx !== -1) pedidos.value[idx].status = 'cancelado'
      if (pedidoAtual.value?.id === id) pedidoAtual.value.status = 'cancelado'
      notificationStore.showSuccess('Pedido cancelado')
    } catch (e) {
      throw e
    }
  }

  function limparPedidoAtual() { pedidoAtual.value = null }

  return {
    // State
    pedidos, pedidoAtual, loading, error, pagination, filtros,
    // Getters
    pedidosPorStatus, totalPedidos,
    // Actions
    carregarPedidos, obterPedido, criarPedido, cancelarPedido, limparPedidoAtual
  }
})
```

### 5. Composables Reutilizáveis
```typescript
// core/composables/useApi.ts
import { ref, type Ref } from 'vue'

export function useApi<T>() {
  const data = ref<T | null>(null)
  const loading = ref(false)
  const error = ref<Error | null>(null)

  async function execute(promise: Promise<T>) {
    loading.value = true
    error.value = null
    try {
      data.value = await promise
      return data.value
    } catch (e) {
      error.value = e as Error
      throw e
    } finally {
      loading.value = false
    }
  }

  function reset() {
    data.value = null
    error.value = null
    loading.value = false
  }

  return { data, loading, error, execute, reset }
}

// Uso no componente
const { data: pedidos, loading, execute } = useApi<PedidoResumo[]>()
await execute(pedidosApi.listar())
```

### 6. MUI Theme (Material Design 3)
```typescript
// core/plugins/mui.ts
import { createTheme, ThemeProvider, CssBaseline } from '@mui/material'
import { ptBR } from '@mui/material/locale'

export const theme = createTheme({
  palette: {
    mode: 'light',
    primary: { main: '#1976d2', light: '#42a5f5', dark: '#1565c0' },
    secondary: { main: '#9c27b0', light: '#ba68c8', dark: '#7b1fa2' },
    error: { main: '#d32f2f' },
    warning: { main: '#ed6c02' },
    info: { main: '#0288d1' },
    success: { main: '#2e7d32' },
    background: { default: '#fafafa', paper: '#ffffff' }
  },
  typography: {
    fontFamily: '"Inter", "Roboto", "Helvetica", "Arial", sans-serif',
    h1: { fontWeight: 600, fontSize: '2.5rem' },
    h2: { fontWeight: 600, fontSize: '2rem' },
    h3: { fontWeight: 600, fontSize: '1.75rem' },
    h4: { fontWeight: 500, fontSize: '1.5rem' },
    h5: { fontWeight: 500, fontSize: '1.25rem' },
    h6: { fontWeight: 500, fontSize: '1rem' },
    body1: { fontSize: '1rem', lineHeight: 1.6 },
    body2: { fontSize: '0.875rem', lineHeight: 1.5 }
  },
  shape: { borderRadius: 8 },
  components: {
    MuiButton: { 
      styleOverrides: { 
        root: { textTransform: 'none', fontWeight: 500 } 
      },
      defaultProps: { variant: 'contained', size: 'medium' }
    },
    MuiCard: { styleOverrides: { root: { boxShadow: '0 1px 3px rgba(0,0,0,0.1)' } } },
    MuiTextField: { defaultProps: { variant: 'outlined', size: 'small' } }
  }
}, ptBR)
```

### 7. PWA Configuration (Vite + Workbox)
```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import { VitePWA } from 'vite-plugin-pwa'

export default defineConfig({
  plugins: [
    vue(),
    VitePWA({
      registerType: 'autoUpdate',
      includeAssets: ['favicon.ico', 'robots.txt', 'apple-touch-icon.png'],
      manifest: {
        name: 'Arteris',
        short_name: 'Arteris',
        description: 'Sistema de gestão Arteris',
        theme_color: '#1976d2',
        background_color: '#fafafa',
        display: 'standalone',
        scope: '/',
        start_url: '/',
        icons: [
          { src: 'pwa-192x192.png', sizes: '192x192', type: 'image/png' },
          { src: 'pwa-512x512.png', sizes: '512x512', type: 'image/png' },
          { src: 'pwa-512x512.png', sizes: '512x512', type: 'image/png', purpose: 'any maskable' }
        ]
      },
      workbox: {
        globPatterns: ['**/*.{js,css,html,ico,png,svg,woff2}'],
        runtimeCaching: [
          {
            urlPattern: /^https:\/\/api\.arteris\.com\/.*/i,
            handler: 'NetworkFirst',
            options: {
              cacheName: 'api-cache',
              expiration: { maxEntries: 100, maxAgeSeconds: 60 * 60 * 24 },
              networkTimeoutSeconds: 10
            }
          }
        ]
      }
    })
  ]
})
```

### 8. Testes (Vitest + Vue Test Utils)

#### Unitário (Store/Composable)
```typescript
// features/pedidos/stores/pedidoStore.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { setActivePinia, createPinia } from 'pinia'
import { usePedidoStore } from './pedidoStore'
import { pedidosApi } from '@/features/pedidos/api/pedidosApi'

vi.mock('@/features/pedidos/api/pedidosApi')

describe('usePedidoStore', () => {
  beforeEach(() => setActivePinia(createPinia()))

  it('deve carregar pedidos e atualizar state', async () => {
    const store = usePedidoStore()
    const mockPedidos = [{ id: '1', clienteNome: 'João', total: 100, status: 'confirmado', criadoEm: new Date().toISOString() }]
    
    vi.mocked(pedidosApi.listar).mockResolvedValue({
      items: mockPedidos, page: 1, limit: 20, total: 1, totalPages: 1
    })

    await store.carregarPedidos()

    expect(store.pedidos).toHaveLength(1)
    expect(store.pedidos[0].clienteNome).toBe('João')
    expect(store.loading).toBe(false)
  })
})
```

#### Componente
```typescript
// features/pedidos/components/PedidoCard.test.ts
import { describe, it, expect } from 'vitest'
import { mount } from '@vue/test-utils'
import PedidoCard from './PedidoCard.vue'
import BaseButton from '@/core/components/ui/BaseButton.vue'
import BaseBadge from '@/core/components/ui/BaseBadge.vue'

describe('PedidoCard', () => {
  const props = {
    pedido: { id: 'ped-123', clienteNome: 'Maria', total: 250.00, status: 'confirmado', criadoEm: new Date().toISOString() },
    onDetalhes: vi.fn()
  }

  it('renderiza informações do pedido', () => {
    const wrapper = mount(PedidoCard, { 
      props, 
      global: { components: { BaseButton, BaseBadge } } 
    })
    
    expect(wrapper.text()).toContain('ped-123')
    expect(wrapper.text()).toContain('Maria')
    expect(wrapper.text()).toContain('R$ 250,00')
    expect(wrapper.text()).toContain('confirmado')
  })

  it('chama onDetalhes ao clicar no botão', async () => {
    const wrapper = mount(PedidoCard, { 
      props, 
      global: { components: { BaseButton, BaseBadge } } 
    })
    
    await wrapper.findComponent(BaseButton).trigger('click')
    expect(props.onDetalhes).toHaveBeenCalledWith('ped-123')
  })
})
```

## Acessibilidade (WCAG 2.1 AA)

1. **Semântica HTML**: `<main>`, `<nav>`, `<header>`, `<footer>`, `<section>`, `<article>`
2. **ARIA**: labels, roles, states (aria-expanded, aria-selected, aria-live)
3. **Contraste**: MUI theme garante 4.5:1 (AA) / 7:1 (AAA) para texto
4. **Teclado**: Todos os componentes interativos focáveis e operáveis por teclado
5. **Foco visível**: Outline visível em `:focus-visible`
6. **Screen readers**: Textos alternativos, labels associados, live regions para notificações
7. **Redução de movimento**: `@media (prefers-reduced-motion: reduce)`

## Boas Práticas

1. **Feature-first organization** — agrupar por domínio, não por tipo técnico
2. **Composition API + `<script setup>`** — padrão oficial Vue 3
3. **TypeScript strict mode** — `strict: true` no tsconfig
4. **Pinia stores por feature** — evita store global monolítico
5. **API client centralizado** — interceptors para auth, errors, refresh
6. **Componentes base (core/components/ui)** — wrappers MUI consistentes
7. **Composables para lógica reutilizável** — useApi, useAuth, useDebounce, etc.
8. **Testes em pirâmide**: Unit (stores, composables) > Component > E2E (fluxos críticos)
9. **PWA para instalação offline** — Service Worker com estratégias de cache adequadas

## Critérios de Qualidade

- [ ] TypeScript strict mode sem erros
- [ ] ESLint + Prettier + Stylelint passam
- [ ] Cobertura de testes unitários > 80% (stores, composables, utils)
- [ ] Testes de componentes para UI crítica
- [ ] E2E para fluxos principais (login, criar pedido, listar)
- [ ] Acessibilidade: axe-core sem violações críticas
- [ ] PWA instalável, funciona offline (cache first para assets)
- [ ] Bundle size otimizado (code splitting por rota, tree shaking)
- [ ] Lighthouse Performance > 90, Accessibility > 95

## Referências

- `ux-ui-design` — design system, usabilidade, prototipação
- `gherkin-bdd` — requisitos comportamentais
- `testes-frontend` — estratégia detalhada de testes
- `devops-azure` — build, deploy, preview environments