---
name: github-actions
description: GitHub Actions: workflows, actions, matriz, environments, secrets, caching, security scanning
metadata:
  type: reference
  category: devops
  agents: [deboaz-arc, deboaz-backend, deboaz-frontend, deboaz-qa]
---

# Skill: GitHub Actions

## Responsabilidade e Escopo

Competência de referência para criação, manutenção e otimização de workflows GitHub Actions para CI/CD, qualidade, segurança e deploy.

## Conceitos Fundamentais

### Estrutura de Workflow
```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
    paths-ignore: ['**.md', 'docs/**']
  pull_request:
    branches: [main]
  workflow_dispatch: # Manual trigger
  schedule:
    - cron: '0 2 * * 1' # Weekly (seg 02h)

env:
  REGISTRY: ${{ secrets.AZURE_CONTAINER_REGISTRY }}
  DOTNET_VERSION: '8.0.x'
  NODE_VERSION: '20'

jobs:
  # Jobs definidos aqui
```

### Jobs, Steps, Actions
```yaml
jobs:
  job-name:
    name: Display Name
    runs-on: ubuntu-latest
    timeout-minutes: 30
    permissions:
      contents: read
      packages: write
      security-events: write
    outputs:
      version: ${{ steps.set-version.outputs.version }}
    env:
      VAR: value
    defaults:
      run:
        shell: bash
        working-directory: ./path
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0 # Para git history (versionamento, changelog)
      
      - name: Setup Tool
        uses: actions/setup-{dotnet|node|python|java}@vX
        with: { version: 'X.Y' }
      
      - name: Cache Dependencies
        uses: actions/cache@v4
        with:
          path: ~/.nuget/packages
          key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj') }}
          restore-keys: ${{ runner.os }}-nuget-
      
      - name: Run Command
        run: |
          echo "Multi-line command"
          dotnet build
      
      - name: Upload Artifact
        uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: ./artifacts/
          retention-days: 7
      
      - name: Download Artifact
        uses: actions/download-artifact@v4
        with:
          name: build-output
          path: ./artifacts/
```

## Padrões Avançados

### Matrix Strategy (Testes Multi-versão/OS)
```yaml
jobs:
  test:
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, windows-latest]
        dotnet: ['8.0', '9.0']
        include:
          - os: ubuntu-latest
            dotnet: '8.0'
            coverage: true
        exclude:
          - os: windows-latest
            dotnet: '9.0'
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: ${{ matrix.dotnet }} }
      - run: dotnet test --collect:"XPlat Code Coverage"
      - name: Upload Coverage
        if: matrix.coverage
        uses: codecov/codecov-action@v3
```

### Reusable Workflows (Call Workflow)
```yaml
# .github/workflows/reusable-dotnet-build.yml
name: Reusable .NET Build
on:
  workflow_call:
    inputs:
      solution-path:
        required: true
        type: string
      configuration:
        required: false
        type: string
        default: 'Release'
    outputs:
      version:
        description: "Version built"
        value: ${{ jobs.build.outputs.version }}
    secrets:
      NUGET_API_KEY:
        required: false

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.version }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: '8.0.x' }
      - name: Build
        run: dotnet build ${{ inputs.solution-path }} -c ${{ inputs.configuration }}
      - id: version
        run: echo "version=$(dotnet ${{ inputs.solution-path }} --version)" >> $GITHUB_OUTPUT

# Uso em outro workflow
jobs:
  build-backend:
    uses: ./.github/workflows/reusable-dotnet-build.yml
    with:
      solution-path: 5_DEV/backend/Arteris.Backend.sln
    secrets:
      NUGET_API_KEY: ${{ secrets.NUGET_API_KEY }}
```

### Environments & Approval Gates
```yaml
# Configuração no GitHub: Settings → Environments
# development: auto-deploy
# staging: auto-deploy after dev
# production: required reviewers (1-2), wait timer (5min)

jobs:
  deploy-staging:
    needs: build
    environment: staging
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Staging
        run: ./deploy.sh staging

  deploy-prod:
    needs: deploy-staging
    environment: production
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Production
        run: ./deploy.sh prod
```

### Secrets & Variables
```yaml
# Repository/Organization Settings → Secrets and variables → Actions
# Secrets: AZURE_CREDENTIALS, ACR_PASSWORD, JWT_SECRET, SONAR_TOKEN
# Variables: AZURE_RG_PROD, ACR_NAME, APP_NAME

steps:
  - name: Azure Login
    uses: azure/login@v1
    with:
      creds: ${{ secrets.AZURE_CREDENTIALS }}
  
  - name: Use Variable
    run: echo "Deploying to ${{ vars.AZURE_RG_PROD }}"
```

### Caching Estratégico
```yaml
# .NET NuGet
- uses: actions/cache@v4
  with:
    path: ~/.nuget/packages
    key: ${{ runner.os }}-nuget-${{ hashFiles('**/packages.lock.json') }}
    restore-keys: ${{ runner.os }}-nuget-

# Node npm
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
    restore-keys: ${{ runner.os }}-npm-

# Docker layers
- uses: docker/build-push-action@v5
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

### Conditional Execution
```yaml
jobs:
  deploy:
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    # ou
    if: startsWith(github.ref, 'refs/tags/') # Tags apenas
    # ou
    if: github.event.pull_request.draft == false
    # ou
    if: contains(github.event.head_commit.message, '[deploy]')
```

### Security Scanning Integrado
```yaml
jobs:
  codeql:
    name: CodeQL Analysis
    runs-on: ubuntu-latest
    permissions:
      security-events: write
    steps:
      - uses: actions/checkout@v4
      - name: Initialize CodeQL
        uses: github/codeql-action/init@v2
        with:
          languages: csharp, javascript, typescript
      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v2

  dependabot:
    name: Dependabot Alerts
    runs-on: ubuntu-latest
    steps:
      - name: Check for vulnerable dependencies
        uses: actions/github-script@v7
        with:
          script: |
            const alerts = await github.rest.dependabot.getAlertsForRepo({
              owner: context.repo.owner,
              repo: context.repo.repo,
              state: 'open'
            });
            const critical = alerts.data.filter(a => a.security_vulnerability.severity === 'critical');
            if (critical.length > 0) {
              core.setFailed(`${critical.length} critical vulnerabilities found`);
            }

  trivy:
    name: Container Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build image
        run: docker build -t ${{ env.REGISTRY }}/app:${{ github.sha }} .
      - name: Scan with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/app:${{ github.sha }}
          format: 'sarif'
          output: 'trivy.sarif'
      - name: Upload to GitHub Security
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy.sarif'
```

### Quality Gates (Status Checks)
```yaml
# Branch Protection Rules → Require status checks to pass before merging
# Checks obrigatórios:
# - "build-and-test" (backend)
# - "build-and-test" (frontend)
# - "codeql"
# - "coverage" (threshold)

jobs:
  coverage:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - name: Check Coverage Threshold
        run: |
          COVERAGE=$(cat coverage/coverage.json | jq '.coverage')
          THRESHOLD=80
          if (( $(echo "$COVERAGE < $THRESHOLD" | bc -l) )); then
            echo "Coverage $COVERAGE% below threshold $THRESHOLD%"
            exit 1
          fi
```

### Concurrency Control
```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true # Cancela runs anteriores na mesma branch/PR
```

### Outputs entre Jobs
```yaml
jobs:
  version:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.get_version.outputs.version }}
    steps:
      - id: get_version
        run: echo "version=$(cat VERSION)" >> $GITHUB_OUTPUT

  build:
    needs: version
    runs-on: ubuntu-latest
    steps:
      - name: Use Version
        run: echo "Building version ${{ needs.version.outputs.version }}"
```

## Boas Práticas

1. **Workflows pequenos e focados** — um propósito por workflow
2. **Reusable workflows** — evitar duplicação (build, test, deploy)
3. **Pin actions to SHA** — `uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683` (v4.2.2)
4. **Least privilege permissions** — `permissions:` no job level
5. **Cache agressivo** — dependencies, docker layers, build outputs
6. **Fail fast** — `fail-fast: true` default em matrix (exceto testes)
7. **Timeouts** — `timeout-minutes` em todos jobs (default 360min)
8. **Artifacts com retention** — `retention-days: 7` (não forever)
9. **Secrets no menor escopo** — Environment > Repository > Organization
10. **Logs estruturados** — `echo "::notice::message"`, `echo "::error::message"`

## Critérios de Qualidade

- [ ] Pipeline CI < 10 min (parallelização adequada)
- [ ] Reusable workflows para build/test/deploy compartilhados
- [ ] Actions pinadas por SHA (não tags mutáveis)
- [ ] Permissions mínimas por job
- [ ] Cache funcionando (cache hit rate > 80%)
- [ ] Security scanning (CodeQL, Trivy, Dependabot) em todo PR
- [ ] Quality gates como required status checks
- [ ] Environments com approval gates para prod
- [ ] Concurrency control evitando runs duplicados
- [ ] Documentação de workflows (README em `.github/workflows/`)

## Referências

- `devops-azure` — Deploy para Azure Container Apps
- `csharp-dotnet` — Build, test, publish .NET
- `desenvolvimento-frontend` — Build, test, preview deploy Vue
- `estrategia-qa` — Quality gates, coverage thresholds