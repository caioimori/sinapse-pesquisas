# Pesquisa Definitiva: Naming Conventions, Organizacao de Projeto & Scaffolding

> **Data:** 2026-04-04
> **Nivel:** DEFINITIVE (Pyramid Level 4)
> **Objetivo:** Estabelecer as melhores praticas absolutas para organizacao de projeto, naming conventions e scaffolding para o SINAPSE-AI — superando o que Apple, Google, Netflix e Vercel fazem internamente.
> **Agente:** Prism (Research Operations Conductor)

---

## Sumario Executivo

Esta pesquisa consolida as melhores praticas de naming conventions, estrutura de diretorios, scaffolding, gerenciamento de .env, .gitignore, CI/CD, templates de PR/commit, design system, documentacao e productizacao. O resultado e um framework de organizacao que combina o rigor do Angular Style Guide, a elegancia da Apple HIG, a praticidade do Turborepo/Vercel, a profundidade do Feature-Sliced Design e as licoes do codigo-fonte vazado do Claude Code da Anthropic.

---

## 1. FILE & FOLDER NAMING CONVENTIONS

### 1.1 Quando Usar Cada Convencao

| Convencao | Onde Usar | Exemplos | Referencia |
|-----------|-----------|----------|------------|
| **kebab-case** | Nomes de arquivos (padrao), URLs, diretories de apps/libs, nomes de pacotes npm, arquivos de config, CSS classes | `user-profile.ts`, `create-story.md`, `@sinapse/core-config` | Angular Style Guide, Turborepo, npm ecosystem |
| **PascalCase** | Classes, React components (arquivo = nome do componente), Types/Interfaces, Enums | `UserProfile.tsx`, `StoryValidator.ts`, `AgentConfig` | React ecosystem, TypeScript conventions |
| **camelCase** | Variaveis, funcoes, metodos, propriedades, arquivos de utilidades que exportam funcoes | `getUserById.ts`, `validateStory.ts`, `formatDate` | JavaScript/TypeScript padrao, Airbnb Style Guide |
| **snake_case** | Variaveis de ambiente, database columns/tables, arquivos Python, config keys legados | `DATABASE_URL`, `user_profiles`, `created_at` | PostgreSQL, Python PEP 8, .env padrao |
| **SCREAMING_SNAKE_CASE** | Constantes, variaveis de ambiente, enum values | `MAX_RETRIES`, `API_BASE_URL`, `NODE_ENV` | Convencao universal |

### 1.2 Regra de Ouro: Arquivo Reflete o Export

A regra mais consistente entre todos os projetos de classe mundial:

> **"Nomeie o arquivo pelo seu export default. Se for React Component ou Classe: PascalCase. Se for funcao/utilidade: camelCase. Se for config/task/workflow: kebab-case."**

Essa regra e usada pelo Iceland Digital (gov.is), Angular, e pela maioria dos projetos enterprise.

### 1.3 Convencoes por Tipo de Arquivo

| Tipo de Arquivo | Padrao de Nome | Exemplo |
|-----------------|---------------|---------|
| React Component | `PascalCase.tsx` | `Button.tsx`, `StoryCard.tsx` |
| Hook customizado | `camelCase.ts` (prefixo `use`) | `useStoryStatus.ts` |
| Utilidade/Helper | `camelCase.ts` | `formatDate.ts`, `validateEnv.ts` |
| Servico | `kebab-case.service.ts` | `story.service.ts` |
| Tipo/Interface | `kebab-case.types.ts` ou `PascalCase.ts` | `story.types.ts` |
| Teste unitario | `{nome}.test.ts` ou `{nome}.spec.ts` | `Button.test.tsx`, `formatDate.spec.ts` |
| Teste E2E | `{feature}.e2e.ts` | `authentication.e2e.ts` |
| Configuracao | `kebab-case.config.ts` | `eslint.config.ts`, `tailwind.config.ts` |
| Constantes | `kebab-case.constants.ts` | `api-endpoints.constants.ts` |
| Task/Workflow (SINAPSE) | `kebab-case.md` ou `kebab-case.yaml` | `create-next-story.md` |
| Agent definition | `kebab-case.md` | `deep-researcher.md` |
| Knowledge base | `kebab-case.md` | `research-depth-pyramid.md` |
| Schema/Migration | `{timestamp}_{descricao}.sql` | `20260404_create_users.sql` |
| Story (docs) | `{epicNum}.{storyNum}.story.md` | `1.3.story.md` |

### 1.4 Convencoes de Diretorio

| Contexto | Convencao | Exemplo |
|----------|-----------|---------|
| Diretorios de apps/libs (Nx/Turborepo) | kebab-case | `apps/web-client`, `packages/core-config` |
| Diretorios de React components | PascalCase | `components/Button/`, `components/StoryCard/` |
| Diretorios de features/modulos | kebab-case | `features/story-management/`, `modules/auth/` |
| Diretorios gerais (utils, config, etc.) | kebab-case | `lib/validators/`, `config/eslint/` |
| Diretorios de agentes SINAPSE | kebab-case | `agents/deep-researcher/` |

### 1.5 Anti-Patterns (PROIBIDO)

- Misturar convencoes no mesmo nivel de diretorio
- Usar `PascalCase` para arquivos que nao sao componentes/classes
- Usar `camelCase` para nomes de pacotes npm (npm exige lowercase)
- Usar espacos ou caracteres especiais em nomes de arquivo
- Nomes genericos como `utils.ts`, `helpers.ts`, `misc.ts` (sempre ser especifico)
- Prefixos redundantes: `IUserInterface` (use `User` para interface, `UserImpl` para classe)
- Sufixos inconsistentes: misturar `.service.ts` com `Service.ts`

### 1.6 Licoes do Angular Style Guide (2025)

O Angular atualizou seu style guide em 2025 com duas modalidades:

- **2025 Style (novo padrao):** Formato conciso — `app.ts` para root component
- **2016 Style (legado):** Formato com tipo — `app.component.ts`

O formato novo reconhece que IDEs modernas tornam o sufixo de tipo menos necessario. Porem, para frameworks como SINAPSE onde o tipo do arquivo e critico para navegacao (task vs workflow vs agent), **manter o sufixo descritivo e recomendado**.

**FINDING:** Angular migrou para formato conciso, mas projetos com muitos tipos de artefatos se beneficiam de sufixos.
**IMPLICATION:** SINAPSE tem 7+ tipos de artefatos (agent, task, workflow, knowledge-base, template, checklist, story). Sem sufixos, a navegacao fica ambigua.
**RECOMMENDATION:** SINAPSE deve usar o padrao `{nome}.{tipo}.{ext}` para artefatos do framework (ex: `create-story.task.md`) mas kebab-case simples para codigo de aplicacao.

---

## 2. DIRECTORY STRUCTURE PATTERNS

### 2.1 Os 5 Paradigmas Principais

#### Paradigma 1: Layer-Based (Tradicional)
```
src/
  controllers/
  services/
  models/
  repositories/
  views/
```
**Quando usar:** Projetos pequenos (<10 files por camada), APIs REST simples.
**Problema:** Nao escala. Arquivos relacionados ficam em diretorios diferentes.

#### Paradigma 2: Feature-Based (Moderno)
```
src/
  features/
    authentication/
      components/
      hooks/
      services/
      types/
    story-management/
      components/
      hooks/
      services/
      types/
```
**Quando usar:** Aplicacoes medias-grandes. O padrao mais popular em 2025-2026.

#### Paradigma 3: Feature-Sliced Design (FSD) — Avancado
```
src/
  app/          # Setup global, providers, router
  pages/        # Composicao de paginas
  widgets/      # Blocos de UI compostos
  features/     # Acoes do usuario (CRUD, auth)
  entities/     # Modelos de dominio (user, story)
  shared/       # UI base, utils, config, API
```
**Regra chave:** Modulos de uma camada SO PODEM importar de camadas ABAIXO.
**Quando usar:** Aplicacoes frontend de grande porte com equipes grandes.

#### Paradigma 4: Domain-Driven Design (DDD)
```
src/
  domains/
    billing/
      invoice/
      transactions/
    user/
      authentication/
      profile/
  shared/
    infrastructure/
    kernel/
```
**Quando usar:** Backend complexo, microservicos, bounded contexts claros.

#### Paradigma 5: Screaming Architecture (Uncle Bob)
```
src/
  health-care/
  patient-records/
  billing/
  scheduling/
```
> "Ao olhar o top-level da estrutura, voce deve GRITAR o que o sistema faz — nao que framework usa."

**FINDING:** Uncle Bob defende que a estrutura deve comunicar o dominio, nao a tecnologia.
**IMPLICATION:** Uma pasta `controllers/` diz "uso MVC" mas nao diz "sistema de saude". Uma pasta `patient-records/` diz exatamente o que o sistema faz.
**RECOMMENDATION:** SINAPSE deve adotar Screaming Architecture no nivel mais alto: `agents/`, `tasks/`, `workflows/`, `knowledge-base/` — gritando que e um framework de IA.

### 2.2 Estrutura de Monorepo (Padrao Turborepo/Vercel)

```
root/
  apps/                    # Aplicacoes deployaveis
    web/                   # Next.js app principal
    docs/                  # Site de documentacao
    cli/                   # CLI tool
  packages/                # Codigo compartilhado
    ui/                    # Design system / components
    config/                # Configuracoes compartilhadas
      eslint/
      typescript/
      tailwind/
    utils/                 # Utilidades compartilhadas
    types/                 # Tipos compartilhados
    features/              # Features compartilhadas
      auth/
      payments/
  turbo.json               # Task orchestration
  pnpm-workspace.yaml      # Workspace definition
  package.json             # Root (private: true, minimal)
```

**Regras do Turborepo:**
- Root `package.json`: APENAS delega para `turbo run` — zero logica de build
- Cada package: `package.json` com nome namespace (`@repo/ui`)
- Cada package: `src/` directory, `tsconfig.json`, campo `exports` explicito
- Profundidade maxima: 2-3 niveis. Mais que isso = navegacao lenta
- **Lockfile obrigatorio** para builds reprodutiveis

### 2.3 Estrutura Ideal para SINAPSE-AI

Combinando Screaming Architecture + Feature-Based + Turborepo:

```
sinapse-ai/                          # Framework core (L1-L2)
  agents/                            # Agent definitions
    {agent-name}/
      {agent-name}.agent.md          # Persona definition
      MEMORY.md                      # Agent memory (L3)
  tasks/                             # Executable task workflows
    {task-name}.task.md
  workflows/                         # Multi-step workflows
    {workflow-name}.workflow.yaml
  knowledge-base/                    # Knowledge bases
    {kb-name}.kb.md
  templates/                         # Document/code templates
    {template-name}.tmpl.md
  checklists/                        # Validation checklists
    {checklist-name}.checklist.md
  core/                              # Core modules
    orchestration/
    memory/
    code-intel/
    graph-dashboard/
  data/                              # Registry & config (L3)
    entity-registry.yaml
    tool-registry.yaml

docs/                                # Project docs (L4)
  stories/                           # Development stories
    {epicNum}.{storyNum}.story.md
  prd/                               # Product requirements
  architecture/                      # Architecture docs
  guides/                            # User/dev guides
  research/                          # Research outputs
    {date}-{topic}/
      README.md

squads/                              # Squad definitions
  {squad-name}/
    squad.yaml                       # Manifest
    agents/                          # Squad-specific agents
    tasks/                           # Squad-specific tasks
    workflows/                       # Squad-specific workflows
    knowledge-base/                  # Squad knowledge
    templates/                       # Squad templates
    checklists/                      # Squad checklists

packages/                            # Shared code (monorepo)
  core-config/                       # Core configuration
  ui/                                # Design system
  utils/                             # Shared utilities
  types/                             # Shared types

apps/                                # Deployable applications
  web/                               # Main web app
  docs/                              # Documentation site
  cli/                               # CLI tool

tests/                               # Test suites (L4)
  unit/
  integration/
  e2e/
```

### 2.4 Regra de Profundidade

| Nivel | Conteudo | Exemplo |
|-------|----------|---------|
| 1 | Dominio/categoria | `agents/`, `tasks/`, `packages/` |
| 2 | Feature/modulo | `agents/deep-researcher/`, `packages/ui/` |
| 3 | Segmento (maximo) | `packages/ui/src/components/` |

**NUNCA mais que 3 niveis de profundidade para chegar a um arquivo.** Se precisar de mais, repense a estrutura.

---

## 3. PROJECT SCAFFOLDING & TEMPLATES

### 3.1 Comparacao de Ferramentas

| Ferramenta | Quando Usar | Complexidade | Tipo |
|------------|-------------|--------------|------|
| **Yeoman** | Project generators publicos, starter kits completos | Alta | Full scaffold |
| **Plop** | Micro-geradores dentro do projeto (new component, new page) | Baixa | Micro-generator |
| **Hygen** | Geradores file-based, leves, convention-over-config | Baixa | File templates |
| **create-* scripts** | CLI installer publico (como create-next-app) | Media | Project bootstrap |
| **Vite create** | Templates pre-configurados com prompt interativo | Media | Project bootstrap |
| **Turborepo generators** | Geradores nativos do monorepo | Baixa | Workspace scaffold |

### 3.2 Best Practices para CLI Installer

Baseado em create-next-app, create-t3-app, Angular CLI:

**Perguntas essenciais do installer:**
1. Nome do projeto (validar: kebab-case, sem caracteres especiais)
2. Template/preset (LP, SaaS, Fintech, Custom)
3. Package manager (npm, yarn, pnpm, bun)
4. Features opcionais (auth, database, analytics, i18n)
5. Estilo (Tailwind, CSS Modules, Styled Components)
6. Inicializar git? (default: sim)
7. Instalar dependencias? (default: sim)

**Arquivos que DEVEM ser gerados:**
```
{project}/
  .env.example          # Todas as vars documentadas com placeholders
  .gitignore            # Comprehensive (ver secao 5)
  .eslintrc.js          # Linting configurado
  .prettierrc           # Formatting configurado
  tsconfig.json         # TypeScript configurado
  package.json          # Dependencies + scripts
  README.md             # Getting started
  next.config.js        # (se Next.js)
  tailwind.config.ts    # (se Tailwind)
  src/
    env.ts              # Validacao de env vars com Zod
    app/
      layout.tsx
      page.tsx
  .github/
    CODEOWNERS           # Code ownership
    PULL_REQUEST_TEMPLATE.md
    workflows/
      ci.yml             # CI pipeline
      deploy.yml         # Deploy pipeline
  .claude/               # SINAPSE config (se ativado)
    CLAUDE.md
    settings.json
```

### 3.3 Micro-Generators (Plop/Hygen)

Para uso dentro do projeto, criar generators para:

| Generator | Comando | Output |
|-----------|---------|--------|
| Component | `pnpm generate component Button` | `src/components/Button/Button.tsx`, `Button.test.tsx`, `Button.stories.tsx`, `index.ts` |
| Feature | `pnpm generate feature auth` | `src/features/auth/components/`, `hooks/`, `services/`, `types/`, `index.ts` |
| API Route | `pnpm generate api users` | `src/app/api/users/route.ts` |
| Story | `pnpm generate story 1.4` | `docs/stories/1.4.story.md` |
| Agent | `pnpm generate agent hawk` | `agents/hawk/hawk.agent.md`, `MEMORY.md` |

---

## 4. .ENV MANAGEMENT

### 4.1 Hierarquia de Arquivos .env

| Arquivo | Proposito | Git | Prioridade |
|---------|-----------|-----|------------|
| `.env` | Valores default compartilhados (nao sensivel) | SIM (controverso) | Mais baixa |
| `.env.local` | Overrides locais (secrets) | NAO | Alta |
| `.env.development` | Vars especificas de dev | SIM | Media |
| `.env.production` | Vars especificas de prod | NAO (usar Vercel/AWS) | Media |
| `.env.test` | Vars para testes | SIM | Media |
| `.env.example` | Documentacao de TODAS as vars | SIM (OBRIGATORIO) | N/A |

**Regra de carregamento (Next.js):**
`.env.local` > `.env.{NODE_ENV}` > `.env`

### 4.2 Validacao com T3 Env + Zod

A abordagem do T3 (create-t3-app) e o padrao-ouro atual:

```typescript
// src/env.ts
import { createEnv } from "@t3-oss/env-nextjs";
import { z } from "zod";

export const env = createEnv({
  server: {
    DATABASE_URL: z.string().url(),
    SUPABASE_SERVICE_ROLE_KEY: z.string().min(1),
    OPENAI_API_KEY: z.string().startsWith("sk-"),
    NODE_ENV: z.enum(["development", "production", "test"]).default("development"),
  },
  client: {
    NEXT_PUBLIC_SUPABASE_URL: z.string().url(),
    NEXT_PUBLIC_SUPABASE_ANON_KEY: z.string().min(1),
    NEXT_PUBLIC_APP_URL: z.string().url(),
  },
  runtimeEnv: {
    DATABASE_URL: process.env.DATABASE_URL,
    SUPABASE_SERVICE_ROLE_KEY: process.env.SUPABASE_SERVICE_ROLE_KEY,
    OPENAI_API_KEY: process.env.OPENAI_API_KEY,
    NODE_ENV: process.env.NODE_ENV,
    NEXT_PUBLIC_SUPABASE_URL: process.env.NEXT_PUBLIC_SUPABASE_URL,
    NEXT_PUBLIC_SUPABASE_ANON_KEY: process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY,
    NEXT_PUBLIC_APP_URL: process.env.NEXT_PUBLIC_APP_URL,
  },
});
```

**Beneficios:**
- Crash no build se variavel faltando (fail-fast)
- Type safety e autocomplete no editor
- Separacao server/client previne vazamento de secrets
- Variaveis de server ficam `undefined` no client (seguro)

### 4.3 .env.example Padrao

```bash
# ============================================================
# App Configuration
# ============================================================
NODE_ENV=development
NEXT_PUBLIC_APP_URL=http://localhost:3000

# ============================================================
# Database (Supabase)
# ============================================================
DATABASE_URL=postgresql://postgres:password@localhost:54322/postgres
NEXT_PUBLIC_SUPABASE_URL=http://localhost:54321
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key-here
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key-here

# ============================================================
# Authentication
# ============================================================
NEXTAUTH_SECRET=your-secret-here
NEXTAUTH_URL=http://localhost:3000

# ============================================================
# AI / LLM
# ============================================================
OPENAI_API_KEY=sk-your-key-here
ANTHROPIC_API_KEY=sk-ant-your-key-here

# ============================================================
# External Services
# ============================================================
STRIPE_SECRET_KEY=sk_test_your-key-here
STRIPE_WEBHOOK_SECRET=whsec_your-key-here
RESEND_API_KEY=re_your-key-here
```

**FINDING:** T3 Env valida vars no build-time e runtime, previne vazamentos client/server.
**IMPLICATION:** Sem validacao, apps quebram em producao com erros cripticos por var faltando.
**RECOMMENDATION:** SINAPSE deve adotar T3 Env + Zod como padrao obrigatorio para todos os projetos.

---

## 5. .GITIGNORE TEMPLATES

### 5.1 Template Compreensivo para SINAPSE Projects

```gitignore
# ============================================================
# Dependencies
# ============================================================
node_modules/
.pnp
.pnp.js
.yarn/install-state.gz

# ============================================================
# Build & Output
# ============================================================
.next/
out/
dist/
build/
.turbo/
*.tsbuildinfo

# ============================================================
# Testing
# ============================================================
coverage/
.nyc_output/

# ============================================================
# Environment Variables (CRITICO: NUNCA commitar secrets)
# ============================================================
.env
.env.local
.env.development.local
.env.test.local
.env.production.local
.env.production
# MANTER no git: .env.example, .env.development, .env.test

# ============================================================
# IDE & Editor
# ============================================================
.vscode/settings.json
.vscode/launch.json
.idea/
*.swp
*.swo
*~

# ============================================================
# OS Files
# ============================================================
.DS_Store
Thumbs.db
Desktop.ini

# ============================================================
# Debug & Logs
# ============================================================
npm-debug.log*
yarn-debug.log*
yarn-error.log*
pnpm-debug.log*
*.log

# ============================================================
# Package Manager
# ============================================================
# Manter lockfiles no git (pnpm-lock.yaml, package-lock.json, yarn.lock)

# ============================================================
# Vercel
# ============================================================
.vercel

# ============================================================
# Supabase
# ============================================================
supabase/.branches
supabase/.temp

# ============================================================
# Next.js
# ============================================================
next-env.d.ts

# ============================================================
# Misc
# ============================================================
*.pem
*.p12
*.key
.cache/
.temp/
tmp/

# ============================================================
# SINAPSE Runtime (NAO commitar)
# ============================================================
.sinapse/handoffs/
.sinapse/scratchpad/
.sinapse/logs/
.sinapse/cache/
```

### 5.2 O Que DEVE Estar no Git

| Arquivo | Razao |
|---------|-------|
| `.env.example` | Documentacao de todas as vars |
| `.env.development` | Vars de dev nao-sensiveis |
| `.env.test` | Vars de teste |
| `pnpm-lock.yaml` | Reproducibilidade de builds |
| `.eslintrc.*` | Consistencia de linting |
| `.prettierrc` | Consistencia de formatting |
| `tsconfig.json` | Configuracao TypeScript |
| `.github/` | CI/CD, templates, CODEOWNERS |
| `.claude/CLAUDE.md` | SINAPSE agent config |

---

## 6. CI/CD TEMPLATES

### 6.1 Pipeline Padrao (GitHub Actions)

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  lint:
    name: Lint & Type Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm run lint
      - run: pnpm run typecheck

  test:
    name: Tests
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm run test -- --coverage
      - uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: coverage/

  security:
    name: Security Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm audit --audit-level=high
      - uses: github/codeql-action/init@v3
        with:
          languages: javascript-typescript
      - uses: github/codeql-action/analyze@v3
```

### 6.2 Deploy Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    name: Deploy to Vercel
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      - uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          vercel-args: --prod
```

### 6.3 Best Practices de CI/CD

| Pratica | Detalhes |
|---------|----------|
| **Pin actions por SHA** | Nunca usar `@v4` mutavel. Usar `@{sha}` para prevenir supply chain attacks |
| **Cache dependencies** | Usar `actions/cache` ou cache nativo do setup-node |
| **Concurrency groups** | Cancelar runs anteriores no mesmo PR |
| **Matrix builds** | Testar em multiplas versoes de Node quando necessario |
| **Secrets em Environments** | Usar GitHub Environments para separar staging/production |
| **Artifact upload** | Salvar coverage reports e build outputs |
| **Branch protection** | Exigir CI green + code review antes de merge |

---

## 7. PR & COMMIT TEMPLATES

### 7.1 Conventional Commits

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

**Tipos padrao:**

| Tipo | Quando Usar |
|------|-------------|
| `feat` | Nova funcionalidade |
| `fix` | Correcao de bug |
| `docs` | Documentacao apenas |
| `style` | Formatacao, sem mudanca de logica |
| `refactor` | Refatoracao sem mudanca de comportamento |
| `perf` | Melhoria de performance |
| `test` | Adicao/correcao de testes |
| `build` | Build system, dependencias |
| `ci` | CI/CD configuration |
| `chore` | Tarefas de manutencao |
| `revert` | Reverter commit anterior |

**Exemplos:**
```
feat(agents): add deep-researcher persona [Story 1.3]
fix(orchestration): prevent double-delegation in QA loop
docs(readme): add installation guide for Windows
refactor(core): extract memory management into separate module
```

### 7.2 PR Template

```markdown
<!-- .github/PULL_REQUEST_TEMPLATE.md -->

## Summary

<!-- 1-3 bullet points descrevendo O QUE e POR QUE -->

-

## Story Reference

<!-- Link para a story no docs/stories/ -->

Story: `docs/stories/{epicNum}.{storyNum}.story.md`

## Changes

<!-- Lista de mudancas significativas -->

-

## Test Plan

<!-- Como testar estas mudancas -->

- [ ] Unit tests passando
- [ ] Lint e typecheck passando
- [ ] Testado manualmente em [cenario]

## Screenshots

<!-- Se aplicavel, screenshots antes/depois -->

## Checklist

- [ ] Codigo segue as convencoes do projeto
- [ ] Self-review realizado
- [ ] Documentacao atualizada (se necessario)
- [ ] Testes adicionados para nova funcionalidade
- [ ] Nenhum secret/credential no codigo
- [ ] Story file atualizado com progresso
```

### 7.3 CODEOWNERS

```
# .github/CODEOWNERS

# Default — equipe core revisa tudo
* @org/core-team

# Framework core — protecao maxima
.sinapse-ai/core/ @org/framework-leads
.sinapse-ai/constitution.md @org/framework-leads

# Agent definitions
agents/ @org/agent-leads
squads/ @org/agent-leads

# Infrastructure
.github/ @org/devops
Dockerfile @org/devops
docker-compose.yml @org/devops

# Documentation
docs/ @org/docs-team

# Design system
packages/ui/ @org/design-team

# Security-sensitive files
.env* @org/security-team
**/auth/ @org/security-team
**/rls/ @org/security-team
```

**Best practices:**
- Usar **teams** (`@org/team-name`) em vez de individuos
- Definir fallback rule (`* @org/core-team`) para cobrir tudo
- Manter granularidade adequada (nao muito especifico, nao muito generico)
- Atualizar quando estrutura de equipe mudar
- Combinar com branch protection: "Require review from CODEOWNERS"

---

## 8. DESIGN SYSTEM / COMPONENT ORGANIZATION

### 8.1 Atomic Design + Tailwind CSS 4

**Estrutura de componentes:**
```
packages/ui/
  src/
    atoms/                    # Elementos basicos indivisiveis
      Button/
        Button.tsx
        Button.test.tsx
        Button.stories.tsx
        index.ts
      Input/
      Badge/
      Avatar/
    molecules/                # Composicao de atoms
      SearchBox/
      FormField/
      UserCard/
    organisms/                # Blocos de UI complexos
      NavigationBar/
      StoryEditor/
      AgentPanel/
    templates/                # Layouts de pagina
      DashboardLayout/
      AuthLayout/
    tokens/                   # Design tokens
      colors.ts
      spacing.ts
      typography.ts
      shadows.ts
    index.ts                  # Barrel export
```

### 8.2 Design Tokens com Tailwind CSS 4

Tailwind CSS 4 introduziu `@theme` como source-of-truth CSS-first:

```css
/* tokens.css */
@theme {
  --color-brand-50: oklch(0.97 0.01 250);
  --color-brand-500: oklch(0.55 0.2 250);
  --color-brand-900: oklch(0.25 0.1 250);

  --font-sans: "Inter", system-ui, sans-serif;
  --font-mono: "JetBrains Mono", monospace;

  --spacing-xs: 0.25rem;
  --spacing-sm: 0.5rem;
  --spacing-md: 1rem;
  --spacing-lg: 1.5rem;
  --spacing-xl: 2rem;

  --radius-sm: 0.25rem;
  --radius-md: 0.5rem;
  --radius-lg: 1rem;
  --radius-full: 9999px;
}
```

**Pipeline de tokens:**
1. Design team define tokens no Figma
2. Export para JSON (plugin de Figma)
3. Build process transforma em Tailwind config
4. Publicar como pacote interno (`@repo/design-tokens`)

### 8.3 Storybook como Hub de Componentes

Cada componente deve ter:
- `.tsx` — Implementacao
- `.test.tsx` — Testes unitarios
- `.stories.tsx` — Storybook stories
- `index.ts` — Re-export

**FINDING:** Tailwind 4 + `@theme` consolida tokens em CSS puro, eliminando config JS.
**IMPLICATION:** Design tokens ficam em um unico lugar, sincronizados entre Figma e codigo.
**RECOMMENDATION:** SINAPSE deve adotar Tailwind 4 com `@theme` como padrao para design tokens.

---

## 9. DOCUMENTATION STRUCTURE

### 9.1 README de Classe Mundial

Secoes obrigatorias (em ordem):

| Secao | Conteudo |
|-------|----------|
| **Logo + Nome** | Visual identity, tagline |
| **Badges** | CI status, version, license, downloads |
| **One-liner** | O que e o projeto em 1 frase |
| **Demo/Screenshot** | GIF ou link para demo ao vivo |
| **Features** | Lista de funcionalidades principais |
| **Quick Start** | 3-5 passos para rodar |
| **Installation** | Instrucoes completas |
| **Usage** | Exemplos de uso comuns |
| **Documentation** | Link para docs completos |
| **Architecture** | Diagrama de alto nivel |
| **Contributing** | Como contribuir |
| **License** | Tipo de licenca |
| **Credits** | Agradecimentos |

### 9.2 Docs-as-Code

**Ferramentas por contexto:**

| Projeto | Ferramenta | Razao |
|---------|-----------|-------|
| React ecosystem | Docusaurus | MDX, versioning, Algolia search |
| Vue ecosystem | VitePress | Velocidade, simplicidade |
| Python | MkDocs + Material | Markdown puro, temas |
| Multi-linguagem | Nextra | Next.js nativo |
| API docs | Swagger/OpenAPI | Padrao da industria |
| Interno/rapido | Mintlify | Design premium, zero config |

**Estrutura de documentacao:**
```
docs/
  getting-started/
    installation.md
    quick-start.md
    configuration.md
  guides/
    naming-conventions.md
    project-structure.md
    agent-creation.md
  api-reference/
    core/
    agents/
    tasks/
  architecture/
    overview.md
    decisions/           # ADRs (Architecture Decision Records)
      adr-001-{titulo}.md
    diagrams/
  stories/               # Development stories
  prd/                   # Product requirements
  research/              # Research outputs
  changelog/
    CHANGELOG.md
```

### 9.3 Architecture Decision Records (ADRs)

Formato padrao:
```markdown
# ADR-{number}: {titulo}

## Status
Accepted | Proposed | Deprecated | Superseded by ADR-{n}

## Context
O problema que estamos tentando resolver.

## Decision
A decisao tomada.

## Consequences
Impactos positivos e negativos.
```

---

## 10. PRODUCTIZATION STRUCTURE

### 10.1 De Framework para Plataforma

| Componente | Descricao | Stack Recomendado |
|------------|-----------|-------------------|
| **Multi-tenancy** | Isolamento de dados por tenant | Supabase RLS + org_id column |
| **Auth & Billing** | Autenticacao + planos + pagamentos | Clerk/Auth.js + Stripe |
| **Usage Tracking** | Metricas de uso por tenant | Custom events + analytics |
| **Dashboard** | Painel do usuario | Next.js App Router |
| **Plugin System** | Extensibilidade | JSON Schema + runtime validation |
| **Marketplace** | Descoberta de plugins/templates | Catalog + ratings |
| **API Gateway** | Rate limiting + auth | Vercel Edge Functions |
| **Webhook System** | Integracoes externas | Queue-based (Inngest/QStash) |

### 10.2 Estrutura de Plataforma

```
platform/
  apps/
    web/                    # App principal (dashboard)
    marketing/              # Landing page + pricing
    docs/                   # Documentacao publica
    api/                    # API Gateway
  packages/
    auth/                   # Autenticacao compartilhada
    billing/                # Billing/subscription logic
    database/               # Schema + migrations + RLS
    ui/                     # Design system
    analytics/              # Usage tracking
    sdk/                    # SDK publico para plugins
  plugins/
    official/               # Plugins oficiais
    community/              # Plugins da comunidade (via marketplace)
  infrastructure/
    terraform/              # IaC
    docker/                 # Container configs
    scripts/                # Deploy/maintenance scripts
```

### 10.3 Plugin/Extension System

```typescript
// Plugin interface
interface SinapsePlugin {
  name: string;
  version: string;
  description: string;
  author: string;
  
  // Lifecycle hooks
  onInstall: () => Promise<void>;
  onActivate: () => Promise<void>;
  onDeactivate: () => Promise<void>;
  
  // Extension points
  agents?: AgentDefinition[];
  tasks?: TaskDefinition[];
  workflows?: WorkflowDefinition[];
  commands?: CommandDefinition[];
  
  // Configuration schema (JSON Schema)
  configSchema?: Record<string, unknown>;
}
```

---

## 11. LICOES DO LEAK DO CLAUDE CODE

O vazamento do codigo-fonte do Claude Code (Anthropic, marco 2026) revelou padroes valiosos:

### 11.1 Arquitetura de Memoria em 3 Camadas

- **MEMORY.md** — Index leve de ponteiros (~150 chars/linha), sempre no contexto
- **Scratchpad** — Notas temporarias por sessao/task
- **Long-term storage** — Persistencia em disco para sessoes futuras

**FINDING:** Claude Code usa um index leve (MEMORY.md) como portal para memoria profunda.
**IMPLICATION:** Carregar tudo no contexto e ineficiente. Um index com ponteiros e mais escalavel.
**RECOMMENDATION:** SINAPSE ja usa MEMORY.md por agente — expandir para incluir ponteiros para knowledge-bases relevantes.

### 11.2 Feature Flags Extensivos

O codigo referencia "KAIROS" 150+ vezes — um sistema de feature flags que controla modos de operacao (daemon mode, autonomous mode, etc.).

### 11.3 Licao de Seguranca

O leak aconteceu porque:
1. Bun gerou source maps por default
2. `*.map` nao estava no `.npmignore`
3. O map referenciava um ZIP completo dos fontes

**RECOMMENDATION:** SEMPRE incluir `*.map` no `.npmignore`. SEMPRE auditar o que `npm pack` inclui antes de publicar.

---

## 12. CHECKLIST FINAL — O PADRAO SINAPSE

### 12.1 Naming Convention Checklist

- [ ] Arquivos de componentes: PascalCase (`Button.tsx`)
- [ ] Arquivos de utilidades: camelCase (`formatDate.ts`)
- [ ] Arquivos de config/task/workflow: kebab-case (`create-story.task.md`)
- [ ] Diretorios: kebab-case (exceto componentes React)
- [ ] Pacotes npm: kebab-case com namespace (`@sinapse/core-config`)
- [ ] Variaveis de ambiente: SCREAMING_SNAKE_CASE (`DATABASE_URL`)
- [ ] Database tables/columns: snake_case (`user_profiles`, `created_at`)
- [ ] CSS classes: kebab-case (ou Tailwind utilities)
- [ ] Branches git: `{user}/{type}/{desc}` (`caio/feat/installer-ux`)
- [ ] Commits: Conventional Commits (`feat(agents): add researcher`)

### 12.2 Structure Checklist

- [ ] Top-level grita o dominio (Screaming Architecture)
- [ ] Maximo 3 niveis de profundidade ate o arquivo
- [ ] Monorepo com `apps/` e `packages/`
- [ ] Cada package com `package.json`, `src/`, `tsconfig.json`
- [ ] Testes co-localizados com o codigo (`.test.ts` junto ao `.ts`)
- [ ] `.env.example` documentando TODAS as variaveis
- [ ] `.gitignore` compreensivo (ver secao 5)
- [ ] `.github/` com CI, PR template, CODEOWNERS
- [ ] `docs/` com stories, PRD, architecture, guides

### 12.3 Quality Checklist

- [ ] ESLint + Prettier configurados
- [ ] TypeScript strict mode ativo
- [ ] T3 Env + Zod validando env vars
- [ ] Conventional Commits enforced (commitlint + husky)
- [ ] CI pipeline com lint + typecheck + test + security
- [ ] Branch protection no main (CI green + review)
- [ ] CODEOWNERS com teams (nao individuos)
- [ ] Storybook para design system

---

## FONTES

### Naming Conventions
- [Unified Naming Strategy for Files and Directories — Iceland Digital](https://docs.devland.is/technical-overview/adr/0009-naming-files-and-directories)
- [Kebab-Case Filenames and PascalCase Classes — DEV Community](https://dev.to/adarshasnah/kebab-case-filenames-and-pascalcase-classes-naming-conventions-that-scale-7dp)
- [Programming Naming Conventions — freeCodeCamp](https://www.freecodecamp.org/news/programming-naming-conventions-explained/)
- [Naming Conventions Guide — Khalil Stemmler](https://khalilstemmler.com/blogs/camel-case-snake-case-pascal-case/)
- [useFilenamingConvention — Biome](https://biomejs.dev/linter/rules/use-filenaming-convention/)

### Angular & Google
- [Angular Coding Style Guide (official)](https://angular.dev/style-guide)
- [Angular 2025: New Style Guide Standards — Medium](https://medium.com/@sehban.alam/angular-2025-new-style-guide-standards-and-how-to-apply-them-3277eef541a3)
- [Material Design 3 Style Guide](https://m3.material.io/foundations/content-design/style-guide)

### Next.js & Vercel
- [Next.js Project Structure (official)](https://nextjs.org/docs/app/getting-started/project-structure)
- [Next.js Folder Structure Best Practices 2026](https://www.codebydeep.com/blog/next-js-folder-structure-best-practices-for-scalable-applications-2026-guide)
- [Battle-Tested NextJS Project Structure 2025 — Medium](https://medium.com/@burpdeepak96/the-battle-tested-nextjs-project-structure-i-use-in-2025-f84c4eb5f426)
- [Turborepo Structure Best Practices — GitHub](https://github.com/vercel/turborepo/blob/main/skills/turborepo/references/best-practices/structure.md)
- [Production Monorepos with Turborepo — Vercel Academy](https://vercel.com/academy/production-monorepos)

### Architecture Patterns
- [Feature-Sliced Design (official)](https://feature-sliced.design/docs/get-started/overview)
- [Screaming Architecture — Uncle Bob](https://blog.cleancoder.com/uncle-bob/2011/09/30/Screaming-Architecture.html)
- [Monorepo Architecture Guide 2025 — Feature-Sliced Design](https://feature-sliced.design/blog/frontend-monorepo-explained)
- [Structuring Monorepos — Mindful Chase](https://www.mindfulchase.com/deep-dives/monorepo-fundamentals-deep-dives-into-unified-codebases/structuring-your-monorepo-best-practices-for-directory-and-code-organization.html)
- [Virtuous Cycle of Workspace Structure — Nx Blog](https://nx.dev/blog/virtuous-cycle-of-workspace-structure)

### Scaffolding
- [Code Scaffolding Tools Comparison](https://blog.overctrl.com/code-scaffolding-tools-which-one-should-you-choose/)
- [Yeoman vs Plop Generators — Simply-How](https://simply-how.com/project-and-files-generators)
- [plop vs yeoman vs hygen — npm-compare](https://npm-compare.com/hygen,plop,yeoman-generator)

### Environment Variables
- [T3 Env Documentation (official)](https://env.t3.gg/docs/introduction)
- [Create T3 App — Env Variables](https://create.t3.gg/en/usage/env-variables)
- [Validating Env Variables with Zod — Francisco Sousa](https://jfranciscosousa.com/blog/validating-environment-variables-with-zod)
- [dotenv Configuration Guide 2026](https://oneuptime.com/blog/post/2026-01-25-dotenv-configuration-nodejs/view)

### CI/CD
- [GitHub Actions CI/CD Best Practices — GitHub/awesome-copilot](https://github.com/github/awesome-copilot/blob/main/instructions/github-actions-ci-cd-best-practices.instructions.md)
- [CI/CD Best Practices — Graphite](https://graphite.dev/guides/in-depth-guide-ci-cd-best-practices)
- [GitHub Actions Guide 2025 — DevOps Tooling](https://thedevopstooling.com/github-actions-ci-cd-guide/)

### Commits & PR Templates
- [Conventional Commits Specification](https://www.conventionalcommits.org/en/v1.0.0/)
- [CODEOWNERS Ultimate Guide — Graph AI](https://www.graphapp.ai/blog/the-ultimate-guide-to-github-codeowners-file)
- [CODEOWNERS Best Practices — Aviator](https://www.aviator.co/blog/a-modern-guide-to-codeowners/)
- [GitHub CODEOWNERS Docs](https://docs.github.com/articles/about-code-owners)

### Design System
- [Tailwind CSS Best Practices 2025-2026 — Frontend Tools](https://www.frontendtools.tech/blog/tailwind-css-best-practices-design-system-patterns)
- [Tailwind CSS 4 @theme Guide — Medium](https://medium.com/@sureshdotariya/tailwind-css-4-theme-the-future-of-design-tokens-at-2025-guide-48305a26af06)
- [Scalable Tailwind Component Library — Medium](https://hexshift.medium.com/how-to-design-and-maintain-a-scalable-tailwind-component-library-be41e87cbf1a)

### Documentation
- [Docs-as-Code with Docusaurus — freeCodeCamp](https://www.freecodecamp.org/news/set-up-docs-as-code-with-docusaurus-and-github-actions/)
- [Documentation Generator Comparison 2025 — OkiDoki](https://okidoki.dev/documentation-generator-comparison)
- [README Best Practices — GitHub](https://github.com/jehna/readme-best-practices)
- [How to Write a Good README — freeCodeCamp](https://www.freecodecamp.org/news/how-to-write-a-good-readme-file/)

### Apple Design System
- [Apple Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/)
- [Apple HIG Design System — DesignSystems.surf](https://designsystems.surf/design-systems/apple)

### Claude Code Leak Analysis
- [Claude Code Source Leak — VentureBeat](https://venturebeat.com/technology/claude-codes-source-code-appears-to-have-leaked-heres-what-we-know)
- [Claude Code Leak — Cybernews](https://cybernews.com/security/anthropic-claude-code-source-leak/)
- [Claude Code Leak — Axios](https://www.axios.com/2026/03/31/anthropic-leaked-source-code-ai)

### .gitignore
- [Next.js .gitignore — GitHub Official](https://github.com/github/gitignore/blob/main/Nextjs.gitignore)
- [Node.js .gitignore Template — GitIgnore.pro](https://gitignore.pro/templates/node)

---

---

## Referencias Historicas & Mundiais

### Pessoas Referencia

**Steve McConnell** -- Autor de "Code Complete" (1993/2004), a referencia mais abrangente em construcao de software. Capitulos sobre naming variables, organizing code, e layout/style sao os mais citados da industria em convencoes de nomenclatura. Relevancia para SINAPSE: suas regras de naming (nomes devem revelar intencao, evitar abreviacoes, ser consistentes) fundamentam as convencoes do framework.

**Robert C. Martin ("Uncle Bob")** -- Autor de "Clean Code" (2008). O capitulo 2 ("Meaningful Names") e a referencia mais influente sobre naming conventions em codigo moderno. Definiu regras como "use intention-revealing names," "avoid disinformation," e "make meaningful distinctions." Relevancia para SINAPSE: as regras de naming do SINAPSE (kebab-case para arquivos, PascalCase para componentes) derivam diretamente de Uncle Bob.

**Douglas Crockford** -- Criador do JSON e autor de "JavaScript: The Good Parts" (2008). Definiu as convencoes de formatacao JavaScript que se tornaram padrao: camelCase para variaveis/funcoes, PascalCase para construtores. Relevancia para SINAPSE: as convencoes JavaScript/TypeScript que o SINAPSE segue foram estabelecidas por Crockford.

**Misko Hevery** -- Criador do Angular (Google). O Angular Style Guide e uma das referencias mais completas de convencoes de projeto: file naming (kebab-case), module structure, barrel exports, e conventional suffixes (.component, .service, .module). Relevancia para SINAPSE: o update do Angular Style Guide 2025 (conciso vs legado) informou as convencoes de arquivo do SINAPSE.

**Guido van Rossum** -- Criador do Python e autor do PEP 8 (Python Style Guide). PEP 8 definiu snake_case como padrao para Python e influenciou convencoes de database (PostgreSQL usa snake_case). Relevancia para SINAPSE: snake_case para database columns e variaveis de ambiente no SINAPSE vem desta tradicao.

**Kent C. Dodds** -- Criador do Testing Library, Remix educator, e evangelista de boas praticas React. Popularizou patterns de organizacao de projeto como "colocation" (testes ao lado do codigo) e "feature folders." Relevancia para SINAPSE: o pattern de feature folders e colocation que o SINAPSE recomenda foi sistematizado por Dodds.

**Vercel Team (Guillermo Rauch, Lee Robinson)** -- Criadores do Next.js e Turborepo. Definiram convencoes de projeto que se tornaram padrao na industria: App Router structure, `app/` directory, route groups `(groupName)`, e `public/` assets. Relevancia para SINAPSE: a estrutura de diretorio recomendada pelo SINAPSE para projetos Next.js segue as convencoes da Vercel.

### Livros "Biblias"

**"Code Complete: A Practical Handbook of Software Construction"** -- Steve McConnell (2nd Edition, 2004). A "biblia" absoluta de construcao de software. 900+ paginas cobrindo naming, formatting, comments, complexity management, testing, debugging, e refactoring. Baseado em evidencias empiricas de centenas de estudos. O que extrair para o SINAPSE: capitulos 11 (Naming Variables), 31 (Layout and Style), e 32 (Self-Documenting Code) fundamentam todas as convencoes de naming e organizacao.

**"Clean Code: A Handbook of Agile Software Craftsmanship"** -- Robert C. Martin (2008). Capitulo 2 (Meaningful Names) e capitulo 3 (Functions) sao as referencias mais citadas sobre naming em codigo moderno. O que extrair para o SINAPSE: regras de naming conventions, tamanho de funcoes, e principio de responsabilidade unica que informam os linting rules do framework.

**"JavaScript: The Good Parts"** -- Douglas Crockford (2008). Definiu as convencoes fundamentais do ecossistema JavaScript: camelCase, strict mode, modulos. Embora datado em APIs, os principios de estilo permanecem validos. O que extrair para o SINAPSE: convencoes de naming JavaScript (camelCase variaveis, PascalCase classes) que o framework segue.

**"Docs for Developers: An Engineer's Field Guide to Technical Writing"** -- Jared Bhatt et al. (2021, Apress). Guia pratico para documentacao tecnica: README, API docs, tutorials, e reference guides. O que extrair para o SINAPSE: templates de README, CONTRIBUTING.md, CHANGELOG, e docs-as-code patterns que o framework usa.

**"The Art of Readable Code"** -- Dustin Boswell & Trevor Foucher (2011). Focado exclusivamente em legibilidade: naming things, simplifying expressions, reorganizing code, e writing comments. Mais pratico e conciso que Clean Code. O que extrair para o SINAPSE: heuristicas de naming (nomes curtos para escopos curtos, nomes longos para escopos longos) e patterns de organizacao de codigo.

**"A Philosophy of Software Design"** -- John Ousterhout (2018). Propoe que o principal problema em software e complexidade, e que "deep modules" (interface simples, implementacao rica) sao a solucao. Contraria algumas ideias de Clean Code (funcoes nao precisam ser minusculas). O que extrair para o SINAPSE: o conceito de "deep modules" justifica agents com interface simples (*help, *task) mas implementacao complexa internamente.

---

*Pesquisa conduzida por Prism (Research Operations Conductor) -- squad-research*
*Nivel: DEFINITIVE | Fontes: 40+ | Tiers: 1-4*
