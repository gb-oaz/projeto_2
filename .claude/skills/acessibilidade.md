---
name: acessibilidade
description: Acessibilidade Digital: WCAG 2.1 AA, ARIA, testes com screen readers, navegação teclado, design inclusivo
metadata:
  type: reference
  category: design
  agents: [deboaz-uxui, deboaz-frontend, deboaz-qa]
---

# Skill: Acessibilidade Digital (WCAG 2.1 AA)

## Responsabilidade e Escopo

Competência de referência para garantir que todas as interfaces digitais sejam acessíveis conforme WCAG 2.1 nível AA. Aplicável a design (UX/UI), implementação (Frontend) e validação (QA).

## Princípios POUR (WCAG 2.1)

| Princípio | Diretrizes | Critérios de Sucesso (AA) |
|---|---|---|
| **Perceptível** | 1.1 Não-texto, 1.2 Mídia temporal, 1.3 Adaptável, 1.4 Distinguível | 1.1.1, 1.2.1-1.2.5, 1.3.1-1.3.5, 1.4.1-1.4.5, 1.4.10-1.4.12 |
| **Operável** | 2.1 Teclado, 2.2 Tempo suficiente, 2.3 Convulsões, 2.4 Navegável, 2.5 Modalidades | 2.1.1-2.1.4, 2.2.1-2.2.6, 2.3.1-2.3.2, 2.4.1-2.4.7, 2.5.1-2.5.6 |
| **Compreensível** | 3.1 Legível, 3.2 Previsível, 3.3 Ajuda à entrada | 3.1.1-3.1.2, 3.2.1-3.2.4, 3.3.1-3.3.4, 3.3.6-3.3.8 |
| **Robusto** | 4.1 Compatível | 4.1.1-4.1.3 |

## Checklist Prático (AA)

### 1. Contraste e Cores
- [ ] Texto normal: razão ≥ 4.5:1
- [ ] Texto grande (≥18pt ou ≥14pt bold): razão ≥ 3:1
- [ ] Elementos UI (bordas, ícones): razão ≥ 3:1
- [ ] Não usar cor como único meio de transmitir informação

### 2. Navegação Teclado
- [ ] Todos elementos interativos alcançáveis via Tab
- [ ] Ordem de foco lógica (visual = DOM order)
- [ ] Foco visível (`:focus-visible` com outline claro)
- [ ] Skip links para conteúdo principal
- [ ] Modais: focus trap, ESC para fechar, foco retorna ao trigger

### 3. Semântica HTML
- [ ] Landmarks: `<header>`, `<nav>`, `<main>`, `<aside>`, `<footer>`
- [ ] Headings: h1→h2→h3 sem saltos (um h1 por página)
- [ ] Lists: `<ul>`, `<ol>`, `<dl>` para conteúdo listado
- [ ] Buttons: `<button>` para ações, `<a>` para navegação
- [ ] Formulários: `<label for="id">` associado a `<input id="id">`

### 4. ARIA (Quando HTML Nativo Insuficiente)
- [ ] `aria-label` / `aria-labelledby` para elementos sem label visível
- [ ] `aria-describedby` para instruções/erros associados
- [ ] `aria-expanded`, `aria-controls` para disclosure (accordion, menu, modal)
- [ ] `aria-live="polite"` / `aria-live="assertive"` para updates dinâmicos
- [ ] `role="alert"` para mensagens críticas (erros, sucesso)
- [ ] `aria-invalid="true"` + `aria-describedby` para campos com erro

### 5. Formulários
- [ ] Labels visíveis e associados programaticamente
- [ ] Instruções claras antes do campo
- [ ] Validação inline com mensagem de erro acessível
- [ ] Agrupamento: `<fieldset>` + `<legend>` para relacionados
- [ ] Autocomplete attributes (`autocomplete="email"`, etc.)

### 6. Mídia e Conteúdo Não-Texto
- [ ] Imagens informativas: `alt` descritivo
- [ ] Imagens decorativas: `alt=""` (vazio) ou `role="presentation"`
- [ ] Ícones SVG: `aria-hidden="true"` se decorativo, ou `<title>` se informativo
- [ ] Vídeos: Legendas (closed captions), transcrição, audio description
- [ ] Áudio: Transcrição, controles de reprodução

### 7. Responsividade e Zoom
- [ ] Funcional em 200% zoom (sem scroll horizontal em 320px)
- [ ] Reflow: conteúdo se adapta sem perda de função
- [ ] Touch targets: ≥ 44×44px (WCAG 2.5.5)

### 8. Movimento e Animação
- [ ] Respeitar `prefers-reduced-motion: reduce`
- [ ] Pausar/parar/esconder auto-carrosséis, animações > 5s
- [ ] Sem conteúdo que pisque > 3x/seg

## Testes de Acessibilidade

### Automatizados (CI/CD)
```bash
# axe-core no Playwright/Cypress
npm test -- --grep "@accessibility"

# Lighthouse CI
lhci autorun
```

### Manuais (Obrigatórios)
| Teste | Ferramenta | Frequência |
|---|---|---|
| Navegação 100% teclado | Teclado apenas | Cada PR |
| Screen reader (NVDA/JAWS/VoiceOver) | NVDA (Win), VoiceOver (Mac) | Cada release |
| Zoom 200% + 400% | Browser zoom | Cada release |
| Contraste | Colour Contrast Analyser | Cada PR |
| ARIA validation | axe DevTools, WAVE | Cada PR |

## Implementação no Frontend (Vue + MUI)

### MUI Theme (Contraste Garantido)
```typescript
// core/plugins/mui.ts
const theme = createTheme({
  palette: {
    mode: 'light',
    primary: { main: '#1976d2', contrastText: '#fff' }, // 4.5:1+
    error: { main: '#d32f2f', contrastText: '#fff' },
    // ...
  },
  typography: {
    // Tamanhos base ≥ 16px (1rem)
    body1: { fontSize: '1rem', lineHeight: 1.6 },
    body2: { fontSize: '0.875rem', lineHeight: 1.5 },
  },
  components: {
    MuiButton: { 
      styleOverrides: { 
        root: { 
          textTransform: 'none', 
          fontWeight: 500,
          minHeight: 44, // Touch target
          minWidth: 44 
        } 
      }
    },
    MuiTextField: { 
      defaultProps: { 
        variant: 'outlined', // Labels sempre visíveis
        size: 'small' 
      } 
    }
  }
})
```

### Componentes Acessíveis (Base)
```vue
<!-- core/components/ui/BaseInput.vue -->
<script setup lang="ts">
import { computed } from 'vue'
import TextField from '@mui/material/TextField'
import FormHelperText from '@mui/material/FormHelperText'

interface Props {
  label: string
  error?: string
  helperText?: string
  'aria-describedby'?: string
  required?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  required: false,
  error: '',
  helperText: ''
})

const describedBy = computed(() => {
  const ids = []
  if (props.helperText) ids.push(`${props.id}-helper`)
  if (props.error) ids.push(`${props.id}-error`)
  return ids.join(' ') || undefined
})
</script>

<template>
  <TextField
    :id="props.id"
    :label="props.label"
    :required="props.required"
    :error="!!props.error"
    :aria-describedby="describedBy"
    :aria-invalid="!!props.error"
  />
  <FormHelperText v-if="props.helperText" :id="`${props.id}-helper`">
    {{ props.helperText }}
  </FormHelperText>
  <FormHelperText v-if="props.error" :id="`${props.id}-error`">
    {{ props.error }}
  </FormHelperText>
</template>
```

### Skip Link Global
```vue
<!-- App.vue -->
<template>
  <a href="#main-content" class="skip-link">Pular para conteúdo principal</a>
  <main id="main-content" tabindex="-1">
    <router-view />
  </main>
</template>

<style>
.skip-link {
  position: absolute;
  top: -100%;
  left: 50%;
  transform: translateX(-50%);
  padding: 12px 24px;
  background: var(--primary-main);
  color: white;
  z-index: 10000;
  border-radius: 0 0 8px 8px;
}
.skip-link:focus {
  top: 0;
}
</style>
```

## Boas Práticas

1. **Acessibilidade nativa** — HTML semântico primeiro, ARIA só quando necessário
2. **Design System acessível** — Tokens de cor com contraste válido, componentes base acessíveis
3. **Testes contínuos** — axe-core em CI, testes manuais por release
4. **Documentação** — Anotações de acessibilidade no Figma (ARIA, focus order, contraste)
5. **Inclusão na Definition of Done** — "Acessível" = passa axe-core + teste teclado

## Critérios de Qualidade

- [ ] 0 violações critical/serious do axe-core em CI
- [ ] 100% navegação teclado funcional (Tab, Enter, Esc, Setas)
- [ ] Contraste ≥ 4.5:1 (texto), ≥ 3:1 (UI large text)
- [ ] Screen reader (NVDA/VoiceOver) anuncia corretamente
- [ ] Zoom 200% sem perda de funcionalidade
- [ ] `prefers-reduced-motion` respeitado
- [ ] Formulários: labels, instruções, validação acessível
- [ ] ARIA usado corretamente (não abusivo)

## Referências

- `ux-ui-design` — Design System, handoff, prototipação
- `desenvolvimento-frontend` — Implementação Vue + MUI
- `estrategia-qa` — Testes automatizados (axe-core), manuais
- WCAG 2.1 Guidelines: https://www.w3.org/WAI/WCAG21/quickref/