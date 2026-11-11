---
name: gherkin-bdd
description: Competência transversal obrigatória para interpretar requisitos escritos em Gherkin dentro de documentos Markdown
metadata:
  type: shared
  category: engineering
---

# Skill: Gherkin e BDD

## Responsabilidade e Escopo

Esta skill capacita os agentes a interpretar requisitos funcionais e comportamentais documentados em Gherkin dentro de arquivos Markdown (`.md`). É uma competência **obrigatória e transversal** para todos os seis agentes do ecossistema DeBoaz.

## Conhecimentos Técnicos

### 1. Leitura e Interpretação de Documentos Markdown
- Identificação de títulos, subtítulos, seções e subseções (heading levels # a ######)
- Navegação pela estrutura documental
- Extração de conteúdo relevante para a atividade em curso

### 2. Identificação de Requisitos em Gherkin
- Localização de blocos de código cercados com linguagem `gherkin`
- Reconhecimento de seções que contêm especificações comportamentais
- Não presumir existência de arquivos `.feature` separados — os requisitos estão **dentro** dos `.md`

### 3. Estruturas Gherkin Suportadas
| Elemento | Descrição |
|---|---|
| `Feature` | Agrupa funcionalidades relacionadas; contém descrição narrativa |
| `Rule` | Agrupa cenários sob uma regra de negócio específica |
| `Background` | Passos pré-condições comuns a todos os cenários da Feature/Rule |
| `Scenario` | Cenário único e concreto |
| `Scenario Outline` | Modelo de cenário com placeholders (`<param>`) |
| `Examples` | Tabela de dados para `Scenario Outline` |
| `Given/When/Then/And/But` | Passos executáveis que descrevem pré-condição, ação e resultado esperado |

### 4. Behavior-Driven Development (BDD)
- Ciclo: Descoberta → Formulação → Automação
- Linguagem ubíqua: vocabulário compartilhado entre domínio e implementação
- Critérios de aceitação como especificações executáveis
- Colaboração triádica: Negócio (PO), Desenvolvimento, QA

### 5. Tipos de Cenários
- **Positivos** (caminho feliz)
- **Negativos** (validações, erros esperados)
- **Alternativos** (fluxos secundários, exceções de negócio)
- **De borda** (limites, casos extremos)

### 6. Rastreabilidade
- Requisitos ↔ Implementação (código, componentes, APIs)
- Requisitos ↔ Testes (unitários, integração, aceitação, E2E)
- Matriz de cobertura por Feature/Rule/Scenario

### 7. Qualidade de Cenários Gherkin
- Independência de implementação (o *qué*, não o *cómo*)
- Atomicidade: um cenário, um comportamento verificável
- Determinismo: mesmos inputs → mesmos outputs
- Legibilidade para não técnicos
- Ausência de ambiguidade e lacunas

## Boas Práticas

1. **Consultar antes de agir**: Sempre ler os documentos `.md` relevantes antes de executar atividades relacionadas a requisitos existentes
2. **Não inventar requisitos**: Trabalhar apenas sobre o que está documentado
3. **Sinalizar ambiguidades**: Comunicar ao PO quando cenários forem ambíguos, incompletos ou contraditórios
4. **Preservar estrutura**: Não alterar requisitos silenciosamente; novas propostas em Gherkin dentro da estrutura Markdown existente
5. **Identificar gaps**: Detectar comportamentos não cobertos pelos cenários existentes
6. **Rastrear decisões**: Vincular implementações e testes aos cenários que atendem

## Diretrizes de Utilização

### Para Todos os Agentes (Comportamento Transversal)
Independentemente da especialidade, todo agente deve conseguir:

1. **Localizar** os documentos Markdown relevantes para a tarefa
2. **Identificar** os requisitos aplicáveis à atividade
3. **Interpretar** os cenários Gherkin (Given/When/Then, Rules, Backgrounds, Examples)
4. **Identificar** as regras de negócio envolvidas
5. **Identificar** os critérios de aceitação explícitos e implícitos
6. **Verificar** se existem ambiguidades, lacunas ou conflitos
7. **Utilizar** os requisitos como referência para decisões técnicas
8. **Comunicar** quais cenários foram atendidos e quais permanecem pendentes

### Fluxo de Trabalho com Requisitos
```
1. Localizar documento(s) .md relevante(s)
2. Extrair Features/Rules/Scenarios aplicáveis
3. Mapear Given → pré-condições necessárias
4. Mapear When → ações/trigger da funcionalidade
5. Mapear Then → resultados esperados/assertivas
6. Identificar Examples → variações de dados de teste
7. Verificar Background → setup comum
8. Validar rastreabilidade com implementação/testes existentes
```

## Critérios de Qualidade

- [ ] Agente localiza documentos `.md` sem presumir estrutura fixa
- [ ] Agente identifica blocos Gherkin em qualquer seção do documento
- [ ] Agente interpreta corretamente Feature, Rule, Background, Scenario, Scenario Outline, Examples
- [ ] Agente distingue cenários positivos, negativos e alternativos
- [ ] Agente identifica ambiguidades e sinaliza ao PO
- [ ] Agente mantém rastreabilidade requisitos ↔ implementação ↔ testes
- [ ] Agente não altera requisitos sem alinhamento com PO
- [ ] Agente usa linguagem ubíqua consistente com o domínio

## Referências a Outras Skills

- `engenharia-requisitos` — para elaboração e gestão de requisitos (PO)
- `estrategia-qa` — para derivação de testes de aceitação (QA)
- `arquitetura-hexagonal` — para mapear requisitos a Ports/Use Cases (ARC, Backend)
- `ux-ui-design` — para transformar cenários em jornadas/fluxos (UX/UI)
- `desenvolvimento-frontend` — para implementar comportamentos na UI (Frontend)
- `desenvolvimento-backend` — para implementar casos de uso e domínio (Backend)

## Exemplos Práticos

### Exemplo 1: Identificação de Requisitos em Markdown
```markdown
# Requisito: Autenticação

## História do Usuário
Como um usuário registrado, quero autenticar-me para acessar o sistema.

## Critérios de Aceitação (Gherkin)

```gherkin
Feature: Autenticação de usuário

  Rule: Credenciais válidas concedem acesso
  
  Background:
    Given que o usuário "joao@email.com" possui conta ativa
    And sua senha é "SenhaSegura123"

  Scenario: Autenticar com credenciais válidas
    When informar email e senha corretos
    Then deverá receber token de acesso
    And deverá ser redirecionado ao dashboard

  Scenario Outline: Tentativas de senha incorreta
    When informar email correto e senha "<senha>"
    Then deverá receber erro de credenciais inválidas
    And o contador de tentativas deve incrementar

    Examples:
      | senha           |
      | SenhaErrada1    |
      | senhafraca      |
      |                 |
```
```

### Exemplo 2: Mapeamento para Implementação (Backend)
```gherkin
# Cenário → Caso de Uso
Scenario: Autenticar com credenciais válidas
  → Use Case: AutenticarUsuarioUseCase
  → Port In: IAutenticarUsuarioPort
  → Domain: Usuario (aggregate), Credenciais (value object)
  → Port Out: IUsuarioRepository, ITokenService
  → Adapter: UsuarioMongoAdapter, JwtTokenAdapter
```

### Exemplo 3: Derivação de Testes (QA)
```gherkin
# Cenário → Teste de Aceitação
Scenario: Autenticar com credenciais válidas
  → Teste: POST /api/auth/login com credenciais válidas
  → Assert: 200 OK, response contém accessToken, refreshToken
  → Teste: GET /api/me com accessToken
  → Assert: 200 OK, retorna dados do usuário
```