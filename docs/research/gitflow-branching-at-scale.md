# GitFlow, Branching Strategies e Version Control at Scale

> **Research Depth:** DEEP DIVE (Level 3)
> **Source Credibility Target:** Tier 1-3 (documentacao oficial, engineering blogs, especificacoes)
> **Researcher:** Prism (research-orqx) via WebSearch
> **Date:** 2026-04-11
> **Verified:** Todas as afirmacoes verificadas via WebSearch em tempo real

---

## Table of Contents

1. [Branching Strategies -- Comparacao Completa](#1-branching-strategies--comparacao-completa)
2. [Branch Naming Conventions](#2-branch-naming-conventions)
3. [Commit Conventions](#3-commit-conventions)
4. [PR/MR Workflow](#4-prmr-workflow)
5. [Monorepo Git Strategies](#5-monorepo-git-strategies)
6. [Branch Protection & Security](#6-branch-protection--security)
7. [Release Management](#7-release-management)
8. [Decision Frameworks](#8-decision-frameworks)
9. [Fontes & Referencias](#9-fontes--referencias)

---

## 1. Branching Strategies -- Comparacao Completa

### 1.1 GitFlow (Vincent Driessen, 2010)

**Origem:** Publicado por Vincent Driessen em 5 de janeiro de 2010 no post "A successful Git branching model" no site nvie.com. Tornou-se o modelo de branching mais popular da decada, ao ponto de muitos times tratarem como padrao da industria.

**Nota de reflexao (2020):** Em marco de 2020, Driessen adicionou uma nota ao post original reconhecendo que o GitFlow foi pensado para software com releases versionados, nao para web apps com continuous delivery. Ele recomenda que times fazendo CD adotem workflows mais simples como GitHub Flow.

> "If your team is doing continuous delivery of software, I would suggest to adopt a much simpler workflow (like GitHub flow) instead of trying to shoehorn git-flow into your team."
> -- Vincent Driessen, 2020 ([nvie.com](https://nvie.com/posts/a-successful-git-branching-model/))

#### Branches Permanentes

| Branch | Proposito | Lifetime |
|--------|-----------|----------|
| `main` (ou `master`) | Codigo em producao. Cada commit e uma release. | Permanente |
| `develop` | Branch de integracao. Acumula features para a proxima release. | Permanente |

#### Branches Temporarias

| Branch | Origem | Destino | Proposito |
|--------|--------|---------|-----------|
| `feature/*` | `develop` | `develop` | Desenvolvimento de nova funcionalidade |
| `release/*` | `develop` | `main` + `develop` | Preparacao de release (bug fixes, metadata) |
| `hotfix/*` | `main` | `main` + `develop` | Correcao critica em producao |

#### Fluxo Detalhado

```
main ─────●─────────────────────●───────●─── (tags: v1.0, v1.1, v1.1.1)
           \                   / \     /
            \        release/1.1  \   / hotfix/1.1.1
             \         /    \      \ /
develop ──●───●───●───●──●───●──●───●───●───
          |       |           |
     feat/login  feat/api   feat/dashboard
```

#### Comandos com git-flow CLI

```bash
# Inicializar GitFlow no repositorio
git flow init

# Feature
git flow feature start login-system
git flow feature finish login-system

# Release
git flow release start 1.1.0
git flow release finish 1.1.0

# Hotfix
git flow hotfix start fix-critical-bug
git flow hotfix finish fix-critical-bug
```

#### Quando Usar GitFlow

- Software com releases versionados explicitos (mobile apps, SDKs, libraries)
- Necessidade de suportar multiplas versoes em producao simultaneamente
- Times que precisam de um staging rigido antes de releases
- Compliance que exige rastreabilidade completa de cada release

#### Quando NAO Usar GitFlow

- Web apps com continuous delivery/deployment
- Times pequenos (< 5 devs) onde a complexidade nao se justifica
- Projetos com deploy diario ou mais frequente

**Fonte:** [nvie.com -- A successful Git branching model](https://nvie.com/posts/a-successful-git-branching-model/) | [Atlassian -- Gitflow Workflow](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow)

---

### 1.2 GitHub Flow

**Origem:** Criado por Scott Chacon e o time do GitHub. Publicado em agosto de 2011 como alternativa simplificada ao GitFlow.

**Filosofia:** Scott Chacon acreditava que a maioria dos times nao precisa de uma estrategia complexa como GitFlow, e que um processo simplificado traz vantagens significativas. Na epoca, o GitHub tinha ~35 funcionarios, com 15-20 trabalhando no mesmo projeto simultaneamente.

#### Regras do GitHub Flow

1. Qualquer coisa na branch `main` e deployavel
2. Para trabalhar em algo novo, crie uma branch a partir de `main` com nome descritivo
3. Faca commits nessa branch localmente e envie regularmente para o remote
4. Abra um Pull Request quando quiser feedback ou achar que esta pronto para merge
5. Apos review e aprovacao, faca merge para `main`
6. Apos merge em `main`, faca deploy imediatamente

#### Fluxo Visual

```
main ────●────●────●────●────●────●──── (always deployable)
          \      / \      / \      /
          feat-A   feat-B   feat-C
          (PR #1)  (PR #2)  (PR #3)
```

#### Caracteristicas

| Aspecto | Detalhe |
|---------|---------|
| Branches permanentes | Apenas `main` |
| Feature branches | Curtas (horas a poucos dias) |
| Deploy | Apos cada merge em `main` |
| Review | Via Pull Request |
| Complexidade | Minima |

#### Quando Usar GitHub Flow

- Web apps com continuous deployment
- Times de qualquer tamanho que fazem CD
- SaaS com versao unica em producao
- Projetos open-source com contribuidores externos

**Fonte:** [Scott Chacon -- GitHub Flow](https://scottchacon.com/2011/08/31/github-flow/) | [GitHub Docs -- GitHub Flow](https://docs.github.com/en/get-started/using-github/github-flow)

---

### 1.3 GitLab Flow

**Origem:** Criado pelo GitLab como compromisso entre GitFlow (complexo demais) e GitHub Flow (simples demais para certos cenarios). Adiciona branches de ambiente ao modelo simplificado.

#### Variante 1: Environment Branches

```
main ────●────●────●────●────●────
          \      /      |
          feat-A        | (merge downstream)
                        v
staging ──────────●────●────
                        |
                        v
production ─────────────●────
```

Commits fluem "downstream" (main -> staging -> production), garantindo que todo codigo e testado em todos os ambientes antes de chegar a producao.

#### Variante 2: Release Branches

Para software versionado, o GitLab Flow usa branches de release criadas a partir de `main`:

```
main ─────●────●────●────●────●────
           \              \
            release/1.0    release/2.0
            (backports)    (backports)
```

#### Regras do GitLab Flow

1. Use feature branches a partir de `main`
2. Teste tudo em `main` primeiro
3. Promova codigo downstream via merge (nao cherry-pick)
4. Nunca faca commit direto em branches de ambiente
5. Se precisar de hotfix, faca em `main` e depois promova

#### Quando Usar GitLab Flow

- Times que gerenciam multiplos ambientes (dev, staging, production)
- Deployments regulados que precisam de aprovacao por ambiente
- Organizacoes com compliance requirements
- Quando GitHub Flow e simples demais mas GitFlow e complexo demais

**Fonte:** [GitLab Docs -- Branching Strategies](https://docs.gitlab.com/user/project/repository/branches/strategies/) | [GitLab -- What is GitLab Flow?](https://about.gitlab.com/topics/version-control/what-is-gitlab-flow/)

---

### 1.4 Trunk-Based Development (TBD)

**Origem:** Praticado em larga escala por Google, Meta (Facebook), Amazon, Netflix e Shopify. O Google opera com 35.000+ desenvolvedores em um unico trunk monorepo, processando ~25.000 changes por dia.

#### Modelo Core

```
trunk (main) ────●──●──●──●──●──●──●──●──●──── (always releasable)
                  |  |     |        |
                  |  |     |     short-lived
                  |  |     |     feature branch
                  |  |     |     (< 1 day)
                  |  |     |
               direct commits (paired/mobbed)
```

#### Dois Estilos

| Estilo | Descricao | Usado Por |
|--------|-----------|-----------|
| **Sem branches** | Commits diretos no trunk. Usado com pair programming e feature flags. | Google (internamente) |
| **Short-lived branches** | Branches de feature com vida < 1-2 dias. Merge via PR. | Times menores, open source |

#### Feature Flags: O Enabler Critico

Feature flags sao essenciais para TBD funcionar em escala. Permitem:

- Codigo incompleto ser merged no trunk sem afetar usuarios
- Deploy continuo de codigo com features desabilitadas
- Testes A/B e rollout gradual
- Rollback instantaneo sem reverter codigo

```javascript
// Exemplo de feature flag
if (featureFlags.isEnabled('new-checkout-flow', userId)) {
  return <NewCheckoutFlow />;
} else {
  return <LegacyCheckoutFlow />;
}
```

> "Google and Meta succeed in trunk-based development because they invested deeply in tooling, not because TBD is easy at scale."
> -- [trunkbaseddevelopment.com](https://trunkbaseddevelopment.com/)

#### Pre-requisitos para TBD

1. **CI robusto** -- Build e testes devem rodar em minutos, nao horas
2. **Feature flags** -- Infraestrutura para toggle de features
3. **Cultura de code review** -- Reviews rapidos (< 1 dia, idealmente horas)
4. **Testes automatizados** -- Coverage alta e confiavel
5. **Monitoramento** -- Detectar problemas em producao rapidamente

#### Quando Usar TBD

- Times com alta maturidade de engenharia
- CI/CD pipeline maduro e rapido
- Cultura de feature flags estabelecida
- Deployment frequency alta (diaria ou mais)
- Web apps e servicos cloud

**Fonte:** [trunkbaseddevelopment.com](https://trunkbaseddevelopment.com/) | [Paul Hammant -- Google's Scaled TBD](https://paulhammant.com/2013/05/06/googles-scaled-trunk-based-development/) | [Martin Fowler -- Feature Toggles](https://martinfowler.com/articles/feature-toggles.html)

---

### 1.5 Release Flow (Microsoft)

**Origem:** Usado pela Microsoft para o Azure DevOps (antigo VSTS) e outros produtos. Combina trunk-based development com release branches alinhadas a sprints.

#### Fluxo

```
main ──●──●──●──●──●──●──●──●──●──●──●──●──●──
       |     |        \              \
    feat-A  feat-B    releases/M129  releases/M130
                      (sprint 129)   (sprint 130)
```

#### Etapas

1. **Branch** -- Desenvolvedor cria branch a partir de `main` para feature ou bug fix
2. **Push** -- Commits frequentes na branch de feature
3. **Pull Request** -- Abrir PR para revisao e CI
4. **Merge** -- Merge em `main` apos aprovacao
5. **Release** -- No final do sprint (~3 semanas), cria-se branch `releases/M{N}` a partir de `main`
6. **Deploy** -- A release branch e deployada. Hotfixes sao cherry-picked para ela

#### Caracteristicas Distintas

| Aspecto | Detalhe |
|---------|---------|
| Trunk | `main` -- sempre pronto para release |
| Feature branches | Curtas, com feature flags |
| Release branches | Criadas por sprint, nunca merged back |
| Hotfixes | Cherry-pick de `main` para release branch |
| Cadencia | Alinhada a sprints (~3 semanas) |

#### Quando Usar Release Flow

- Times grandes com cadencia de release definida
- Produtos que precisam de release branches para suporte
- Organizacoes com sprint-based workflow
- Quando TBD puro e arriscado mas GitFlow e complexo demais

**Fonte:** [Azure DevOps Blog -- Release Flow](https://devblogs.microsoft.com/devops/release-flow-how-we-do-branching-on-the-vsts-team/) | [Microsoft Learn -- How Microsoft develops with DevOps](https://learn.microsoft.com/en-us/devops/develop/how-microsoft-develops-devops)

---

### 1.6 Tabela Comparativa Completa

| Criterio | GitFlow | GitHub Flow | GitLab Flow | TBD | Release Flow |
|----------|---------|-------------|-------------|-----|--------------|
| **Complexidade** | Alta | Baixa | Media | Baixa | Media |
| **Branches permanentes** | 2 (main + develop) | 1 (main) | 2-4 (main + env) | 1 (trunk) | 1 (main) |
| **Release branches** | Sim | Nao | Opcional | Opcional | Sim (por sprint) |
| **Hotfix branches** | Sim (dedicadas) | Nao (feature branch) | Nao (fix em main, promove) | Nao (fix em trunk, cherry-pick) | Nao (cherry-pick) |
| **Deploy frequency** | Baixa-media | Alta (continuo) | Media-alta | Muito alta | Media (por sprint) |
| **Feature flags** | Opcional | Opcional | Opcional | Essencial | Recomendado |
| **Team size ideal** | 5-20 | 1-50+ | 5-50 | 10-10.000+ | 10-500+ |
| **Melhor para** | SDKs, mobile, versioned | SaaS, web apps | Multi-env, regulated | High-perf teams | Enterprise sprints |
| **Quem usa** | Projetos versionados | GitHub, startups | GitLab, enterprises | Google, Meta, Netflix | Microsoft, Azure |

---

## 2. Branch Naming Conventions

### 2.1 Prefixos Padrao da Industria

| Prefixo | Proposito | Exemplo |
|---------|-----------|---------|
| `feat/` ou `feature/` | Nova funcionalidade | `feat/oauth-google-integration` |
| `fix/` ou `bugfix/` | Correcao de bug | `fix/cart-total-calculation` |
| `hotfix/` | Correcao critica em producao | `hotfix/critical-security-patch` |
| `release/` | Preparacao de release | `release/v2.1.0` |
| `chore/` | Manutencao, tooling, deps | `chore/update-eslint-config` |
| `docs/` | Documentacao | `docs/api-authentication-guide` |
| `test/` | Testes | `test/add-payment-integration-tests` |
| `refactor/` | Refatoracao sem mudanca funcional | `refactor/extract-auth-service` |
| `perf/` | Melhoria de performance | `perf/optimize-image-loading` |
| `ci/` | Mudancas em CI/CD | `ci/add-staging-deploy-step` |
| `style/` | Formatacao, whitespace | `style/fix-linting-errors` |
| `revert/` | Reverter mudanca anterior | `revert/remove-broken-cache` |

### 2.2 Regras de Nomenclatura

```
<prefixo>/<descricao-em-kebab-case>
<prefixo>/<ticket-id>-<descricao>
<usuario>/<prefixo>/<descricao>
```

#### Regras Tecnicas

1. **Usar lowercase** -- `feat/login`, nunca `Feat/Login`
2. **Separar palavras com hifens** -- `feat/user-authentication`, nunca `feat/user_authentication`
3. **Evitar caracteres especiais** -- Apenas alfanumericos, hifens e barras
4. **Ser descritivo mas conciso** -- `fix/cart-total-rounding`, nao `fix/fixed-the-bug-where-cart-total-was-wrong`
5. **Incluir ticket quando possivel** -- `feat/JIRA-1234-oauth-integration`
6. **Maximo ~50 caracteres** apos o prefixo

### 2.3 Convencoes de Projetos Open Source

#### React (facebook/react)

O React usa branches simples sem prefixo formal rigido. O desenvolvimento principal ocorre na branch `main`. Pull requests de contribuidores externos seguem o padrao de fork + branch descritiva. O time interno do React trabalha diretamente em branches descritivas sem prefixo obrigatorio.

#### Next.js (vercel/next.js)

O Next.js segue um modelo similar ao GitHub Flow. A branch principal e `canary` (nao `main`), que serve como branch de desenvolvimento. Releases sao cortadas de `canary` para branches de release versionadas.

#### Kubernetes (kubernetes/kubernetes)

O Kubernetes adota um modelo rigoroso para contribuicoes. PRs devem ser pequenas e focadas (preferem 100 PRs pequenas a 10 monolitos). O titulo do PR deve ter 50 caracteres ou menos, e o corpo deve incluir "Fixes #12345" quando aplicavel. Branches de release sao mantidas para cada minor version (`release-1.28`, `release-1.29`).

### 2.4 Branch Naming com Ticket Reference

```bash
# Jira
feat/PROJ-1234-user-authentication
fix/PROJ-5678-cart-calculation

# GitHub Issues
feat/42-oauth-google
fix/108-memory-leak

# Linear
feat/ENG-234-redesign-dashboard
```

### 2.5 Automacao: Criar Branch a partir de Issue

No GitHub, e possivel criar branches automaticamente a partir de issues:

1. Abra a issue no GitHub
2. No sidebar direito, clique em "Create a branch"
3. O GitHub gera um nome baseado no titulo da issue (ex: `42-add-oauth-support`)
4. O branch e criado automaticamente no remote

Ferramentas como Linear e Jira tambem oferecem integracao nativa para criacao automatica de branches com o formato `<prefix>/<ticket-id>-<slug>`.

**Fonte:** [DEV Community -- Branch Naming Conventions](https://dev.to/varbsan/a-simplified-convention-for-naming-branches-and-commits-in-git-il4) | [Zignuts -- Master Git Branch Naming 2026](https://www.zignuts.com/blog/master-git-branch-naming-conventions)

---

## 3. Commit Conventions

### 3.1 Conventional Commits Specification (v1.0.0)

A especificacao Conventional Commits define uma convencao leve sobre mensagens de commit, fornecendo regras claras para criar um historico de commits explicito e legivel por maquina.

#### Formato

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

#### Types Obrigatorios

| Type | Proposito | SemVer |
|------|-----------|--------|
| `feat` | Nova funcionalidade para o usuario | MINOR |
| `fix` | Correcao de bug | PATCH |

#### Types Adicionais (Convencionais)

| Type | Proposito |
|------|-----------|
| `build` | Mudancas no sistema de build (webpack, npm, etc.) |
| `chore` | Manutencao que nao modifica src ou test |
| `ci` | Mudancas em configuracao de CI |
| `docs` | Apenas documentacao |
| `style` | Formatacao (whitespace, semicolons, etc.) |
| `refactor` | Refatoracao sem mudanca funcional |
| `perf` | Melhoria de performance |
| `test` | Adicionar ou corrigir testes |
| `revert` | Reverter commit anterior |

#### Breaking Changes

Breaking changes DEVEM ser indicadas de uma das formas:

```bash
# Opcao 1: ! antes do :
feat!: drop support for Node 14

# Opcao 2: footer BREAKING CHANGE
feat: allow provided config object to extend other configs

BREAKING CHANGE: `extends` key in config file is now used for extending configs

# Opcao 3: ambos combinados
feat(api)!: remove deprecated endpoints

BREAKING CHANGE: The /v1/users endpoint has been removed. Use /v2/users instead.
```

#### Exemplos Praticos

```bash
# Feature com escopo
feat(auth): add OAuth 2.0 Google provider

# Bug fix com referencia a issue
fix(cart): resolve rounding error in total calculation

Closes #1234

# Refatoracao sem impacto funcional
refactor(database): extract query builder into separate module

# Breaking change
feat(api)!: migrate to v3 response format

BREAKING CHANGE: All API responses now use { data, meta, errors }
instead of the flat response format.

Migration guide: https://docs.example.com/migration/v3

# Multi-paragrafo body
fix(auth): prevent session fixation attack

The session ID was not being regenerated after login,
making the application vulnerable to session fixation attacks.

This fix regenerates the session ID upon successful authentication
and invalidates the previous session.

Reviewed-by: Security Team
Refs: CVE-2024-12345
```

#### Mapeamento para SemVer

| Commit Type | Version Bump | Exemplo |
|-------------|-------------|---------|
| `fix:` | PATCH (0.0.X) | 1.2.3 -> 1.2.4 |
| `feat:` | MINOR (0.X.0) | 1.2.3 -> 1.3.0 |
| `BREAKING CHANGE` | MAJOR (X.0.0) | 1.2.3 -> 2.0.0 |
| `docs:`, `style:`, etc. | Nenhum | 1.2.3 -> 1.2.3 |

**Fonte:** [conventionalcommits.org v1.0.0](https://www.conventionalcommits.org/en/v1.0.0/) | [SEI/CMU -- Versioning with Git Tags and Conventional Commits](https://www.sei.cmu.edu/blog/versioning-with-git-tags-and-conventional-commits/)

---

### 3.2 Semantic Release -- Automacao Completa

O semantic-release automatiza todo o ciclo de release: determina a proxima versao, gera changelog e publica o pacote.

#### Instalacao e Configuracao

```bash
npm install --save-dev semantic-release \
  @semantic-release/commit-analyzer \
  @semantic-release/release-notes-generator \
  @semantic-release/changelog \
  @semantic-release/npm \
  @semantic-release/github \
  @semantic-release/git
```

#### Configuracao (`.releaserc.json`)

```json
{
  "branches": ["main"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    ["@semantic-release/changelog", {
      "changelogFile": "CHANGELOG.md"
    }],
    "@semantic-release/npm",
    ["@semantic-release/git", {
      "assets": ["CHANGELOG.md", "package.json"],
      "message": "chore(release): ${nextRelease.version}\n\n${nextRelease.notes}"
    }],
    "@semantic-release/github"
  ]
}
```

#### GitHub Actions Workflow

```yaml
name: Release
on:
  push:
    branches: [main]

permissions:
  contents: write
  issues: write
  pull-requests: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npx semantic-release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

**Fonte:** [semantic-release GitHub](https://github.com/semantic-release/semantic-release) | [npm -- semantic-release](https://www.npmjs.com/package/semantic-release)

---

### 3.3 Commitlint + Husky -- Enforcement Local

#### Instalacao

```bash
# Commitlint
npm install --save-dev @commitlint/cli @commitlint/config-conventional

# Husky
npm install --save-dev husky
npx husky init
```

#### Configuracao do Commitlint (`commitlint.config.js`)

```javascript
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    // Type deve ser um dos tipos convencionais
    'type-enum': [2, 'always', [
      'build', 'chore', 'ci', 'docs', 'feat', 'fix',
      'perf', 'refactor', 'revert', 'style', 'test'
    ]],
    // Subject nao pode ser vazio
    'subject-empty': [2, 'never'],
    // Subject com maximo 100 caracteres
    'subject-max-length': [2, 'always', 100],
    // Type nao pode ser vazio
    'type-empty': [2, 'never'],
    // Type em lowercase
    'type-case': [2, 'always', 'lower-case'],
    // Subject nao termina com ponto
    'subject-full-stop': [2, 'never', '.'],
  },
};
```

#### Hook do Husky (`.husky/commit-msg`)

```bash
npx --no -- commitlint --edit $1
```

#### Teste

```bash
# Falha: tipo invalido
git commit -m "foo: this will fail"
# Error: type must be one of [build, chore, ci, docs, feat, fix, ...]

# Sucesso: formato correto
git commit -m "feat(auth): add OAuth Google provider"
# Commit criado com sucesso
```

**Fonte:** [commitlint -- Local Setup Guide](https://commitlint.js.org/guides/local-setup.html) | [GitHub -- commitlint](https://github.com/conventional-changelog/commitlint/)

---

### 3.4 Changelog Generation

#### release-please (Google)

O release-please, criado pelo Google, automatiza a geracao de changelogs e criacao de releases baseadas em Conventional Commits. Diferente do semantic-release (que faz release automatico a cada merge), o release-please mantem Release PRs que acumulam mudancas.

```yaml
# .github/workflows/release-please.yml
name: release-please
on:
  push:
    branches: [main]

permissions:
  contents: write
  pull-requests: write

jobs:
  release-please:
    runs-on: ubuntu-latest
    steps:
      - uses: googleapis/release-please-action@v4
        with:
          release-type: node
```

**Como funciona:**

1. A cada push em `main`, o release-please analisa os commits
2. Cria/atualiza um Release PR com changelog organizado
3. Quando o Release PR e merged, cria a tag e GitHub Release
4. O CHANGELOG.md e atualizado automaticamente

#### standard-version (legacy)

```bash
# Gerar release sem publicar
npx standard-version

# Pre-release
npx standard-version --prerelease alpha
# 1.0.0 -> 1.0.1-alpha.0

# Release especifica
npx standard-version --release-as major
```

> **Nota:** O `standard-version` foi descontinuado em favor do release-please.

**Fonte:** [googleapis/release-please](https://github.com/googleapis/release-please) | [release-please-action](https://github.com/google-github-actions/release-please-action)

---

### 3.5 Signed Commits (GPG/SSH)

Commits assinados provam criptograficamente a identidade do autor. O GitHub exibe um badge "Verified" em commits assinados.

#### Configuracao GPG

```bash
# Gerar chave GPG (recomendado: 4096 bits)
gpg --full-generate-key

# Listar chaves
gpg --list-secret-keys --keyid-format=long

# Configurar Git
git config --global user.signingkey <KEY_ID>
git config --global commit.gpgsign true

# Exportar chave publica (adicionar ao GitHub)
gpg --armor --export <KEY_ID>
```

#### Configuracao SSH (Git 2.34+)

```bash
# Configurar formato de assinatura
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgsign true
```

#### Verificacao

```bash
# Verificar assinatura do ultimo commit
git log --show-signature -1

# Verificar todas as assinaturas
git log --show-signature
```

**Fonte:** [GitHub Docs -- Signing commits](https://docs.github.com/en/authentication/managing-commit-signature-verification/signing-commits) | [Git SCM -- Signing Your Work](https://git-scm.com/book/en/v2/Git-Tools-Signing-Your-Work)

---

### 3.6 Atomic Commits -- Best Practices

Um commit atomico e a menor unidade logica de mudanca. Cada commit deve:

1. **Representar UMA mudanca logica** -- Nao misturar feature + refactor + fix no mesmo commit
2. **Compilar e passar testes** -- Cada commit no historico deve ser buildable
3. **Ser revertivel independentemente** -- `git revert <sha>` deve funcionar sem efeitos colaterais
4. **Ter mensagem que descreve a intencao** -- "why" > "what"

#### Anti-patterns

```bash
# ERRADO: commit gigante com tudo misturado
git commit -m "various fixes and improvements"

# ERRADO: WIP commits
git commit -m "WIP"
git commit -m "WIP 2"
git commit -m "almost done"

# CERTO: commits atomicos e descritivos
git commit -m "feat(auth): add password reset email template"
git commit -m "feat(auth): implement password reset API endpoint"
git commit -m "test(auth): add password reset flow integration tests"
```

#### Tecnica: Interactive Rebase para Limpar Historico

```bash
# Antes de abrir PR, limpar commits WIP
git rebase -i HEAD~5

# Opcoes no editor:
# pick   abc1234 feat(auth): add reset template
# squash def5678 WIP
# squash ghi9012 fix typo
# pick   jkl3456 feat(auth): add reset endpoint
# pick   mno7890 test(auth): add reset tests
```

---

## 4. PR/MR Workflow

### 4.1 PR Templates

Um PR template padroniza a informacao fornecida em cada Pull Request, reduzindo back-and-forth e melhorando a qualidade do review.

#### Template Recomendado (`.github/PULL_REQUEST_TEMPLATE.md`)

```markdown
## Summary

<!-- Brief description of what this PR does and why -->

## Changes

- [ ] Change 1
- [ ] Change 2

## Type of Change

- [ ] Bug fix (non-breaking change that fixes an issue)
- [ ] New feature (non-breaking change that adds functionality)
- [ ] Breaking change (fix or feature that would cause existing functionality to change)
- [ ] Documentation update
- [ ] Refactoring (no functional changes)

## Testing

<!-- How was this tested? -->

- [ ] Unit tests added/updated
- [ ] Integration tests added/updated
- [ ] Manual testing performed

## Screenshots (if applicable)

<!-- Add screenshots here -->

## Checklist

- [ ] My code follows the project's style guidelines
- [ ] I have performed a self-review of my code
- [ ] I have added tests that prove my fix/feature works
- [ ] New and existing unit tests pass locally
- [ ] I have updated the documentation accordingly
- [ ] My changes generate no new warnings

## Breaking Changes

<!-- If this is a breaking change, describe the impact and migration path -->

## Related Issues

Closes #<!-- issue number -->
```

**Fonte:** [Microsoft Engineering Playbook -- PR Template](https://microsoft.github.io/code-with-engineering-playbook/code-reviews/pull-request-template/) | [Graphite -- PR Template Checklist](https://graphite.com/guides/comprehensive-checklist-github-pr-template)

---

### 4.2 Code Review Best Practices (Google Engineering Practices)

O Google publicou suas praticas de code review como documentacao open-source. Os principios fundamentais:

#### O Padrao do Code Review

> "O proposito primario do code review e garantir que a saude geral do codebase do Google esteja melhorando ao longo do tempo."
> -- [Google eng-practices](https://google.github.io/eng-practices/review/reviewer/standard.html)

#### Principios para Reviewers

| Principio | Detalhe |
|-----------|---------|
| **Progresso do desenvolvedor** | Desenvolvedores precisam conseguir avancar. Se nunca submeterem melhorias, o codebase nunca melhora. |
| **Decisoes tecnicas > preferencia pessoal** | Se o autor demonstra que varias abordagens sao igualmente validas, o reviewer deve aceitar a preferencia do autor. |
| **Evitar over-engineering** | Reviewers devem evitar que devs criem codigo mais generico que o necessario ou adicionem funcionalidade especulativa. |
| **Resposta rapida** | Maximo de 1 dia util para responder a um review request. |

#### O Que Verificar no Review

1. **Design** -- A mudanca esta bem projetada?
2. **Funcionalidade** -- O codigo faz o que o autor pretendia?
3. **Complexidade** -- Poderia ser mais simples?
4. **Testes** -- Testes apropriados (unit, integration, e2e)?
5. **Naming** -- Nomes claros para variaveis, funcoes, classes?
6. **Comentarios** -- Comentarios sao claros e necessarios?
7. **Style** -- Segue o style guide do projeto?
8. **Documentacao** -- Documentacao relevante foi atualizada?

#### Tamanho do PR

| Tamanho | LOC | Review Quality |
|---------|-----|----------------|
| Pequeno (ideal) | < 200 linhas | Alto |
| Medio | 200-400 linhas | Bom |
| Grande | 400-1000 linhas | Decrescente |
| Muito grande | > 1000 linhas | Review superficial provavel |

O Kubernetes project explicita: "We would prefer 100 small, obvious PRs over 10 unreviewable monoliths."

**Fonte:** [Google eng-practices -- Code Review](https://google.github.io/eng-practices/review/) | [Google eng-practices -- What to look for](https://google.github.io/eng-practices/review/reviewer/looking-for.html)

---

### 4.3 CODEOWNERS

O arquivo CODEOWNERS define automaticamente quem deve revisar mudancas em partes especificas do codebase.

#### Sintaxe (`.github/CODEOWNERS`)

```bash
# Owners globais (fallback para qualquer arquivo sem owner especifico)
* @org/core-team

# Frontend
/src/components/ @org/frontend-team
/src/pages/ @org/frontend-team
*.tsx @org/frontend-team

# Backend
/src/api/ @org/backend-team
/src/services/ @org/backend-team

# Infrastructure
/infra/ @org/devops-team
/.github/ @org/devops-team
Dockerfile @org/devops-team

# Database
/migrations/ @org/data-team @org/backend-team

# Documentacao
/docs/ @org/tech-writers
*.md @org/tech-writers

# Seguranca -- require review de security team
/src/auth/ @org/security-team
/src/crypto/ @org/security-team

# Configuracao critica -- multiple owners required
package.json @org/core-team @org/devops-team
tsconfig.json @org/core-team
```

#### Regras de Precedencia

A **ultima regra que faz match** tem precedencia. Isso significa que regras mais especificas devem vir depois das mais genericas.

#### Enforcement

Para tornar CODEOWNERS obrigatorio:
1. Ativar branch protection em `main`
2. Habilitar "Require review from Code Owners"
3. Code owners com write permission sao automaticamente solicitados para review

**Requisito:** Code owners devem ter **write permission** no repositorio.

**Fonte:** [GitHub Docs -- About code owners](https://docs.github.com/articles/about-code-owners) | [Aviator -- CODEOWNERS Guide](https://www.aviator.co/blog/a-modern-guide-to-codeowners/)

---

### 4.4 CI Checks como Merge Gates

Status checks obrigatorios garantem que codigo so e merged quando todos os gates passam:

```yaml
# Exemplo: GitHub Actions como status check
name: CI
on: [pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run lint

  typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run typecheck

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test -- --coverage
      - name: Check coverage threshold
        run: |
          COVERAGE=$(cat coverage/coverage-summary.json | jq '.total.lines.pct')
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "Coverage $COVERAGE% is below 80% threshold"
            exit 1
          fi

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm audit --audit-level=high
```

---

### 4.5 Merge Strategies: Squash vs Merge vs Rebase

#### Comparacao Detalhada

| Estrategia | Historico | Merge Commits | Quando Usar |
|------------|-----------|---------------|-------------|
| **Merge Commit** | Preserva todos os commits + merge commit | Sim | Quando o historico granular do PR importa |
| **Squash & Merge** | Um unico commit por PR | Nao | Main branch limpo, PRs com commits WIP |
| **Rebase & Merge** | Commits individuais sem merge commit | Nao | Historico linear, commits ja atomicos |

#### Merge Commit

```
main: A──B──────M──
                /
feature: C──D──E
```

- **Pros:** Preserva historico completo. Facil reverter o PR inteiro com `git revert -m 1 M`.
- **Contras:** Merge commits poluem o historico. Pode dificultar `git log`.

#### Squash & Merge

```
main: A──B──S(C+D+E)──
```

- **Pros:** Main branch limpo. Cada PR = 1 commit. Facil de entender o que cada PR fez.
- **Contras:** Perde granularidade dos commits individuais. Conflitos podem ser mais frequentes em branches long-lived.

#### Rebase & Merge

```
main: A──B──C'──D'──E'──
```

- **Pros:** Historico linear sem merge commits. Mantém commits individuais.
- **Contras:** Reescreve historico (novos SHAs). Conflitos precisam ser resolvidos por commit.

#### Recomendacao por Contexto

| Contexto | Estrategia Recomendada |
|----------|----------------------|
| Time pequeno, commits limpos | Rebase & Merge |
| Time grande, muitos PRs/dia | Squash & Merge |
| Open source, historico importa | Merge Commit |
| Projetos com audit trail rigido | Merge Commit |
| Monorepo com muitos contribuidores | Squash & Merge |

**Fonte:** [GitHub Docs -- About PR merges](https://docs.github.com/articles/about-pull-request-merges) | [Graphite -- Best merge strategy](https://graphite.com/blog/pull-request-merge-strategy) | [Atlassian -- Merging vs Rebasing](https://www.atlassian.com/git/tutorials/merging-vs-rebasing)

---

### 4.6 Auto-Merge e Merge Queue

#### Auto-Merge

GitHub permite configurar auto-merge em PRs que atendem todos os requisitos:

1. Habilitar "Allow auto-merge" nas settings do repositorio
2. Branch protection com pelo menos um check obrigatorio
3. Ao criar o PR, clicar "Enable auto-merge"
4. Quando todos os checks passarem e reviews forem aprovados, o merge acontece automaticamente

#### Dependabot Auto-Merge

```yaml
# .github/workflows/dependabot-auto-merge.yml
name: Dependabot auto-merge
on: pull_request

permissions:
  contents: write
  pull-requests: write

jobs:
  dependabot:
    runs-on: ubuntu-latest
    if: github.actor == 'dependabot[bot]'
    steps:
      - name: Approve and auto-merge patch updates
        run: gh pr merge --auto --squash "$PR_URL"
        env:
          PR_URL: ${{ github.event.pull_request.html_url }}
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

#### GitHub Merge Queue

O Merge Queue resolve o problema de PRs que passam nos checks individualmente mas quebram quando merged junto com outros PRs.

**Como funciona:**

1. Ao adicionar um PR a queue, o GitHub cria uma branch temporaria (`main/pr-N`)
2. Essa branch combina `main` + todos os PRs a frente na queue + o PR atual
3. CI roda nessa branch combinada
4. Se passar, o PR e merged. Se falhar, e removido da queue.

**Configuracao:**

- Disponivel em repositorios publicos (GitHub Free) e privados (GitHub Enterprise Cloud)
- Configuravel com concurrency limit (1-100 builds simultaneos)
- Integra-se com branch protection rules

**Fonte:** [GitHub Docs -- Managing a merge queue](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-a-merge-queue) | [GitHub Blog -- Merge Queue GA](https://github.blog/news-insights/product-news/github-merge-queue-is-generally-available/)

---

## 5. Monorepo Git Strategies

### 5.1 Git Worktrees -- Desenvolvimento Paralelo

Git worktrees permitem checkout de multiplas branches em diretorios separados simultaneamente, compartilhando o mesmo `.git`.

```bash
# Criar worktree para branch existente
git worktree add ../project-hotfix hotfix/critical-bug

# Criar worktree com nova branch
git worktree add ../project-feature -b feat/new-dashboard

# Listar worktrees
git worktree list
# /home/user/project          abc1234 [main]
# /home/user/project-hotfix   def5678 [hotfix/critical-bug]
# /home/user/project-feature  ghi9012 [feat/new-dashboard]

# Remover worktree
git worktree remove ../project-hotfix
```

#### Use Cases

| Cenario | Beneficio |
|---------|-----------|
| Hotfix urgente durante feature development | Nao precisa stash/commit WIP |
| Code review de outro PR | Checkout separado sem perder contexto |
| Rodar testes em outra branch | Nao interrompe desenvolvimento |
| Comparar comportamento entre branches | Ambas rodando simultaneamente |

#### Limitacao Importante

A mesma branch NAO pode estar checked out em mais de um worktree simultaneamente.

**Fonte:** [DataCamp -- Git Worktree Tutorial](https://www.datacamp.com/tutorial/git-worktree-tutorial) | [Andrew Lock -- Git Worktree](https://andrewlock.net/working-on-two-git-branches-at-once-with-git-worktree/)

---

### 5.2 Sparse Checkout -- Repos Grandes

Sparse checkout permite popular o working directory apenas com os arquivos/diretorios necessarios, dramaticamente reduzindo o tamanho do checkout em monorepos.

```bash
# Clone com sparse checkout
git clone --sparse --filter=blob:none https://github.com/org/monorepo.git
cd monorepo

# Adicionar apenas os diretorios necessarios
git sparse-checkout add packages/frontend
git sparse-checkout add packages/shared

# Listar diretorios no sparse checkout
git sparse-checkout list

# Verificar status
git sparse-checkout check-rules
```

#### Configuracao em CI

```yaml
# GitHub Actions com sparse checkout
steps:
  - uses: actions/checkout@v4
    with:
      sparse-checkout: |
        packages/api
        packages/shared
        package.json
      sparse-checkout-cone-mode: true
```

**Fonte:** [GitHub Blog -- Sparse Checkout](https://github.blog/open-source/git/bring-your-monorepo-down-to-size-with-sparse-checkout/) | [Git SCM -- git-sparse-checkout](https://git-scm.com/docs/git-sparse-checkout)

---

### 5.3 Git LFS (Large File Storage)

Git LFS armazena arquivos grandes fora do repositorio Git, substituindo-os por ponteiros leves.

```bash
# Instalar Git LFS
git lfs install

# Rastrear tipos de arquivo grandes
git lfs track "*.psd"
git lfs track "*.zip"
git lfs track "*.mp4"
git lfs track "assets/**/*.png"

# Verificar o que esta sendo rastreado
git lfs ls-files

# O .gitattributes e atualizado automaticamente
cat .gitattributes
# *.psd filter=lfs diff=lfs merge=lfs -text
# *.zip filter=lfs diff=lfs merge=lfs -text
```

#### Quando Usar LFS

| Arquivo | Usar LFS? | Motivo |
|---------|-----------|--------|
| Imagens de design (PSD, AI, Sketch) | Sim | Arquivos binarios grandes |
| Videos e assets de midia | Sim | Gigabytes de dados |
| Build artifacts (ZIP, TAR) | Sim | Nao devem estar no historico |
| Fontes (TTF, OTF, WOFF) | Depende | Se > 1MB, considere |
| Dados de ML (modelos, datasets) | Sim | Frequentemente > 100MB |
| Codigo fonte | Nao | Git lida bem com texto |

**Fonte:** [DEV Community -- Managing Large Repos with LFS](https://dev.to/arasosman/managing-large-repositories-with-git-lfs-and-sparse-checkout-2ek7)

---

### 5.4 Shallow Clones para CI

Shallow clones reduzem drasticamente o tempo de clone ao baixar apenas commits recentes.

```bash
# Clone com profundidade limitada
git clone --depth 1 https://github.com/org/repo.git

# Clone com mais contexto (ultimos 20 commits)
git clone --depth 20 https://github.com/org/repo.git

# Clone otimizado para CI (sem blobs desnecessarios)
git clone --depth 1 --filter=blob:none --sparse https://github.com/org/repo.git

# Partial clone (sem blobs > 1MB)
git clone --filter=blob:limit=1m https://github.com/org/repo.git
```

#### Impacto em Performance

| Tecnica | Reducao de Dados | Melhor Para |
|---------|-----------------|-------------|
| `--depth 1` | 70-90% | CI builds simples |
| `--filter=blob:none` | 60-80% | Monorepos |
| `--sparse` + `--filter` | 80-95% | Monorepos com sparse checkout |
| `--single-branch` | 30-50% | Repos com muitas branches |

#### GitLab CI Default

O GitLab Runner faz shallow clone por padrao com `GIT_DEPTH=20`. Pode ser customizado no `.gitlab-ci.yml`:

```yaml
variables:
  GIT_DEPTH: 5  # Apenas ultimos 5 commits
  GIT_STRATEGY: fetch  # Reusar repo existente
```

#### Limitacoes

- Nao permite acesso ao historico completo
- `git log` mostra apenas commits clonados
- `git blame` pode nao funcionar corretamente
- Nao e util se o build precisa de tags ou historico completo

**Fonte:** [DEV Community -- Improve git clone in CI](https://dev.to/saranshk/improve-git-clone-performance-in-a-ci-pipeline-52pf) | [GitLab Docs -- Improving monorepo performance](https://docs.gitlab.com/user/project/repository/monorepos/)

---

### 5.5 Git Submodules vs Subtrees

#### Comparacao

| Aspecto | Submodules | Subtrees |
|---------|-----------|----------|
| **Armazenamento** | Referencia (URL + commit SHA) | Copia completa dos arquivos |
| **Tamanho do repo** | Menor (apenas ponteiros) | Maior (inclui historico do sub-projeto) |
| **Setup para contribuidores** | Requer `git submodule init/update` | Transparente (arquivos ja estao la) |
| **Upstream contributions** | Facil (sub-repo e independente) | Dificil (mudancas misturadas no historico) |
| **Atualizacao** | `git submodule update --remote` | `git subtree pull` |
| **Complexidade** | Alta (muitos footguns) | Media |
| **CI/CD** | Precisa de flags especiais | Funciona out-of-the-box |
| **Melhor para** | Dependencias que voce mantem separadas | Codigo que quer "absorver" |

#### Submodules

```bash
# Adicionar submodule
git submodule add https://github.com/org/shared-lib.git libs/shared

# Clonar repo com submodules
git clone --recurse-submodules https://github.com/org/main-repo.git

# Atualizar submodules
git submodule update --remote --merge

# Inicializar submodules apos clone
git submodule init
git submodule update
```

#### Subtrees

```bash
# Adicionar subtree
git subtree add --prefix=libs/shared https://github.com/org/shared-lib.git main --squash

# Atualizar subtree
git subtree pull --prefix=libs/shared https://github.com/org/shared-lib.git main --squash

# Push mudancas de volta para o sub-repo
git subtree push --prefix=libs/shared https://github.com/org/shared-lib.git main
```

#### Recomendacao

| Cenario | Escolha |
|---------|---------|
| Dependencia externa que voce nao modifica | Submodule |
| Codigo compartilhado entre poucos repos | Subtree |
| Muitos contribuidores, setup simples | Subtree |
| Dependencia com releases independentes | Submodule |
| Quer "absorver" e esquecer da origem | Subtree |

**Fonte:** [Atlassian -- Git Subtree](https://www.atlassian.com/git/tutorials/git-subtree) | [GitProtect -- Subtree vs Submodule](https://gitprotect.io/blog/managing-git-projects-git-subtree-vs-submodule/)

---

## 6. Branch Protection & Security

### 6.1 GitHub Branch Protection Rules

Branch protection rules previnem force pushes, delecoes acidentais e merges sem aprovacao.

#### Configuracao Recomendada para `main`

| Regra | Valor Recomendado | Proposito |
|-------|-------------------|-----------|
| Require pull request reviews | 1-2 approvals | Garantir revisao humana |
| Dismiss stale reviews | Ativado | Re-review apos novos pushes |
| Require review from Code Owners | Ativado | Especialistas revisam suas areas |
| Require status checks to pass | Ativado | CI deve passar antes de merge |
| Require branches to be up to date | Ativado | Branch deve estar atualizada com main |
| Require signed commits | Opcional | Verificacao de identidade |
| Require linear history | Opcional | Evita merge commits |
| Include administrators | Ativado | Admins tambem seguem as regras |
| Restrict pushes | Ativado | Apenas merge via PR |
| Allow force pushes | Desativado | Nunca em branches protegidas |
| Allow deletions | Desativado | Prevenir delecao acidental |

### 6.2 GitHub Rulesets (Feature Mais Recente)

Rulesets sao a evolucao das branch protection rules, oferecendo mais flexibilidade e escalabilidade.

#### Diferencas Chave

| Aspecto | Branch Protection Rules | Rulesets |
|---------|------------------------|---------|
| **Escopo** | Uma branch por regra | Multiplas branches via pattern |
| **Sobreposicao** | Apenas uma regra por branch | Multiplas rulesets podem aplicar |
| **Conflito** | N/A | Regra mais restritiva vence |
| **Alcance** | Apenas branches | Branches + Tags |
| **Bypass** | Admins podem bypass | Bypass list configuravel |
| **Auditoria** | Basica | Audit log completo |
| **Import/Export** | Nao | Sim (JSON) |

#### Exemplo de Ruleset

```json
{
  "name": "Production Protection",
  "target": "branch",
  "enforcement": "active",
  "conditions": {
    "ref_name": {
      "include": ["~DEFAULT_BRANCH", "refs/heads/release/*"],
      "exclude": []
    }
  },
  "rules": [
    { "type": "deletion" },
    { "type": "non_fast_forward" },
    { "type": "required_signatures" },
    {
      "type": "pull_request",
      "parameters": {
        "required_approving_review_count": 2,
        "dismiss_stale_reviews_on_push": true,
        "require_code_owner_review": true,
        "require_last_push_approval": true
      }
    },
    {
      "type": "required_status_checks",
      "parameters": {
        "strict_required_status_checks_policy": true,
        "required_status_checks": [
          { "context": "CI / lint" },
          { "context": "CI / test" },
          { "context": "CI / build" }
        ]
      }
    }
  ],
  "bypass_actors": [
    {
      "actor_type": "Team",
      "actor_id": 12345,
      "bypass_mode": "pull_request"
    }
  ]
}
```

#### Required Reviewer Rule (GA em 2026)

O GitHub lancou a "Required Reviewer Rule" como Generally Available em fevereiro de 2026, permitindo exigir que reviewers especificos aprovem PRs baseados em file patterns, independente do CODEOWNERS.

**Fonte:** [GitHub Docs -- About rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets) | [GitHub Docs -- Branch protection](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches) | [GitHub Blog -- Required reviewer rule GA](https://github.blog/changelog/2026-02-17-required-reviewer-rule-is-now-generally-available/)

---

### 6.3 Signed Commits Enforcement

Para projetos com requisitos de seguranca elevados:

1. **Ativar "Require signed commits"** na branch protection
2. **Configurar verificacao** -- Commits sem assinatura verificavel serao rejeitados
3. **Suportar GPG e SSH** -- Ambos os metodos sao aceitos pelo GitHub
4. **Vigilant mode** -- No GitHub, o usuario pode ativar "vigilant mode" que marca todos os commits nao-assinados como "Unverified"

### 6.4 Push Restrictions

Restricoes de push controlam quem pode enviar commits diretamente para branches protegidas:

```
Settings > Branches > Branch Protection > Restrict who can push to matching branches
```

- **Adicionar times/usuarios especificos** que podem push direto
- **Ideal:** Apenas bots de CI (como dependabot ou release-please) podem push direto em `main`
- **Todos os humanos** devem passar pelo fluxo de PR

---

## 7. Release Management

### 7.1 Semantic Versioning (SemVer 2.0.0)

A especificacao SemVer, criada por Tom Preston-Werner (co-fundador do GitHub), define um sistema de versionamento que comunica a natureza das mudancas.

#### Formato

```
MAJOR.MINOR.PATCH[-pre-release][+build-metadata]
```

#### Regras

| Componente | Incrementa Quando | Exemplo |
|-----------|-------------------|---------|
| **MAJOR** | Mudancas incompativeis na API | 1.0.0 -> 2.0.0 |
| **MINOR** | Nova funcionalidade backward-compatible | 1.0.0 -> 1.1.0 |
| **PATCH** | Bug fix backward-compatible | 1.0.0 -> 1.0.1 |

#### Pre-release Versions

```
1.0.0-alpha       < 1.0.0-alpha.1    < 1.0.0-alpha.beta
1.0.0-beta        < 1.0.0-beta.2     < 1.0.0-beta.11
1.0.0-rc.1        < 1.0.0
```

- Identificadores numericos sao comparados como inteiros
- Identificadores alfanumericos sao comparados lexicalmente
- Pre-release versions tem precedencia menor que a versao normal

#### Build Metadata

```
1.0.0+20240101
1.0.0+build.123
1.0.0-alpha+001
```

Build metadata e IGNORADA na determinacao de precedencia de versao.

#### Regras Adicionais SemVer

1. Uma vez que uma versao e publicada, NAO pode ser modificada
2. MAJOR version 0 (0.y.z) e para desenvolvimento inicial -- API pode mudar a qualquer momento
3. Version 1.0.0 define a API publica
4. Patch version DEVE ser incrementada se apenas bug fixes backward-compatible
5. Minor version DEVE resetar patch para 0 quando incrementada
6. Major version DEVE resetar minor e patch para 0 quando incrementada

**Fonte:** [semver.org -- Semantic Versioning 2.0.0](https://semver.org/)

---

### 7.2 Release Branches Workflow

#### GitFlow Release Branch

```bash
# Criar release branch
git checkout develop
git checkout -b release/1.2.0

# Apenas bug fixes e metadata nesta branch
# Nada de features novas

# Finalizar release
git checkout main
git merge --no-ff release/1.2.0
git tag -a v1.2.0 -m "Release v1.2.0"

# Merge de volta para develop
git checkout develop
git merge --no-ff release/1.2.0

# Deletar release branch
git branch -d release/1.2.0
```

#### TBD/Release Flow Release Branch

```bash
# Criar release branch a partir do trunk
git checkout main
git checkout -b releases/2024-Q1

# Deploy da release branch
# Se hotfix necessario:
git checkout main
# Fix no main primeiro
git commit -m "fix: resolve payment race condition"

# Cherry-pick para release
git checkout releases/2024-Q1
git cherry-pick <sha-do-fix>
```

---

### 7.3 Hotfix Procedures

#### Fluxo de Hotfix (GitFlow)

```bash
# 1. Criar hotfix branch a partir de main
git checkout main
git checkout -b hotfix/1.2.1

# 2. Aplicar fix
git commit -m "fix: resolve critical auth bypass"

# 3. Merge em main e tag
git checkout main
git merge --no-ff hotfix/1.2.1
git tag -a v1.2.1 -m "Hotfix v1.2.1"

# 4. Merge em develop
git checkout develop
git merge --no-ff hotfix/1.2.1

# 5. Deletar hotfix branch
git branch -d hotfix/1.2.1
```

#### Fluxo de Hotfix (TBD com Cherry-Pick)

A best practice para times TBD e:

1. **Reproduzir o bug no trunk** 
2. **Fixar no trunk** com teste
3. **CI verifica** no trunk
4. **Cherry-pick** para a release branch
5. **CI verifica** na release branch

```bash
# Fix no trunk
git checkout main
git commit -m "fix: resolve payment timeout issue"
# SHA: abc1234

# Cherry-pick para release
git checkout releases/2024-Q1
git cherry-pick abc1234

# Se conflito, resolver e continuar
git cherry-pick --continue
```

**Fonte:** [trunkbaseddevelopment.com -- Branch for Release](https://trunkbaseddevelopment.com/branch-for-release/) | [2coffee -- Hotfixing with Cherry-pick](https://2coffee.dev/en/articles/hotfix-with-git-cherry-pick)

---

### 7.4 Release Notes Automation

#### release-please Output

Quando o Release PR e merged, o release-please gera automaticamente:

```markdown
## [2.1.0](https://github.com/org/repo/compare/v2.0.0...v2.1.0) (2026-04-11)

### Features

* **auth:** add OAuth 2.0 Google provider ([#123](https://github.com/org/repo/issues/123))
* **dashboard:** implement real-time analytics widget ([#145](https://github.com/org/repo/issues/145))

### Bug Fixes

* **cart:** resolve rounding error in total calculation ([#134](https://github.com/org/repo/issues/134))
* **api:** fix rate limiter bypass on batch endpoints ([#156](https://github.com/org/repo/issues/156))

### Performance Improvements

* **database:** optimize N+1 queries in user listing ([#167](https://github.com/org/repo/issues/167))
```

---

### 7.5 Tag Management

```bash
# Criar tag anotada (recomendado para releases)
git tag -a v1.2.0 -m "Release v1.2.0 - OAuth integration"

# Criar tag leve (para uso temporario)
git tag v1.2.0-rc.1

# Listar tags
git tag -l "v1.*"

# Push tags para remote
git push origin v1.2.0
git push origin --tags  # Todas as tags

# Deletar tag local e remota
git tag -d v1.2.0-rc.1
git push origin --delete v1.2.0-rc.1

# Criar branch a partir de tag (para hotfix)
git checkout -b hotfix/1.2.1 v1.2.0

# Ver detalhes de uma tag
git show v1.2.0
```

#### Convencoes de Tag

| Formato | Exemplo | Uso |
|---------|---------|-----|
| `v{MAJOR}.{MINOR}.{PATCH}` | `v1.2.3` | Release final |
| `v{M}.{m}.{p}-alpha.{N}` | `v1.2.3-alpha.1` | Alpha pre-release |
| `v{M}.{m}.{p}-beta.{N}` | `v1.2.3-beta.2` | Beta pre-release |
| `v{M}.{m}.{p}-rc.{N}` | `v1.2.3-rc.1` | Release candidate |

---

### 7.6 Pre-release / RC Versions

#### Workflow de Pre-release

```bash
# Desenvolvimento -> Alpha
git tag -a v2.0.0-alpha.1 -m "v2.0.0 Alpha 1"
# Testes internos, feedback

# Alpha -> Beta
git tag -a v2.0.0-beta.1 -m "v2.0.0 Beta 1"
# Testes mais amplos, beta testers

# Beta -> Release Candidate
git tag -a v2.0.0-rc.1 -m "v2.0.0 Release Candidate 1"
# Testes finais, apenas bug fixes

# RC -> Release
git tag -a v2.0.0 -m "v2.0.0 - Major release"
# Deploy em producao
```

#### Semantic Release com Pre-release Channels

```json
{
  "branches": [
    "main",
    { "name": "beta", "prerelease": true },
    { "name": "alpha", "prerelease": true }
  ]
}
```

Isso permite que merges em `beta` gerem versoes como `1.2.0-beta.1` e merges em `alpha` gerem `1.2.0-alpha.1`, enquanto merges em `main` geram releases estaveis.

---

## 8. Decision Frameworks

### 8.1 Matriz de Decisao: Escolhendo a Estrategia

```
                        Deployment Frequency
                    Low ◄──────────────────► High
                    │                         │
            Large   │  GitFlow    Release Flow│  TBD
Team Size   ────────┤           ┌─────────────┤
            Small   │  GitLab   │ GitHub Flow │  TBD
                    │  Flow     │             │
                    │           │             │
                    Low ◄───────┴─────────────► High
                        CI/CD Maturity
```

### 8.2 Checklist de Decisao

| Pergunta | Se SIM | Se NAO |
|----------|--------|--------|
| Voce faz deploy continuo (diario+)? | GitHub Flow ou TBD | GitFlow ou Release Flow |
| Precisa suportar multiplas versoes em producao? | GitFlow | GitHub Flow ou TBD |
| Time tem > 50 devs? | TBD ou Release Flow | Qualquer |
| Tem infraestrutura de feature flags? | TBD | GitFlow ou GitHub Flow |
| CI/CD pipeline roda em < 10 min? | TBD ou GitHub Flow | GitFlow |
| Precisa de aprovacao por ambiente (staging, prod)? | GitLab Flow | GitHub Flow |
| Produto e SaaS com versao unica? | GitHub Flow ou TBD | GitFlow |
| Produto e SDK/library versionado? | GitFlow | GitHub Flow |
| Cadencia de release e por sprint? | Release Flow | GitHub Flow ou TBD |

### 8.3 DORA Metrics e Branching Strategy

As metricas DORA (DevOps Research and Assessment), baseadas em pesquisa com 32.000+ profissionais, demonstram correlacao direta entre branching strategy e performance de entrega.

#### Metricas Core

| Metrica | Elite | High | Medium | Low |
|---------|-------|------|--------|-----|
| **Deployment Frequency** | Multiplas/dia | 1x/semana a 1x/mes | 1x/mes a 1x/6meses | < 1x/6meses |
| **Lead Time for Changes** | < 1 hora | 1 dia a 1 semana | 1 semana a 1 mes | > 1 mes |
| **Change Failure Rate** | 0-15% | 16-30% | 16-30% | > 30% |
| **Time to Restore Service** | < 1 hora | < 1 dia | 1 dia a 1 semana | > 1 semana |

#### Correlacao com Branching

| Performance Level | Branching Strategy Tipica |
|-------------------|--------------------------|
| **Elite** | Trunk-Based Development, GitHub Flow |
| **High** | GitHub Flow, Release Flow |
| **Medium** | GitLab Flow, GitFlow simplificado |
| **Low** | GitFlow complexo, branches long-lived |

> Times elite tem **2x mais probabilidade** de atingir metas organizacionais. A pesquisa mostra que short-lived branches (< 1 dia) e integracao frequente correlacionam com alta performance.

**Nota 2024-2025:** O relatorio DORA 2025 mudou de 4 niveis de performance para 7 perfis de time, incorporando sinais culturais e humanos alem das metricas de delivery.

**Fonte:** [Atlassian -- DORA Metrics](https://www.atlassian.com/devops/frameworks/dora-metrics) | [Octopus Deploy -- DORA Metrics 2024/25](https://octopus.com/devops/metrics/dora-metrics/) | [Google Cloud -- Four Keys](https://cloud.google.com/blog/products/devops-sre/using-the-four-keys-to-measure-your-devops-performance)

---

### 8.4 Setup Recomendado por Tipo de Projeto

#### SaaS Web App (time de 5-15 devs)

```
Strategy:     GitHub Flow
Merge:        Squash & Merge
Commits:      Conventional Commits + Commitlint
Release:      semantic-release (automatico a cada merge em main)
Protection:   1 approval, CI checks, CODEOWNERS
Branch Names: feat/, fix/, chore/, docs/
```

#### Mobile App (time de 10-30 devs)

```
Strategy:     GitFlow
Merge:        Merge Commit (historico completo)
Commits:      Conventional Commits + Commitlint
Release:      release-please (Release PRs)
Protection:   2 approvals, CI + E2E checks, signed commits
Branch Names: feature/, bugfix/, release/, hotfix/
```

#### Enterprise Platform (time de 50+ devs)

```
Strategy:     Trunk-Based Development
Merge:        Squash & Merge
Commits:      Conventional Commits + Commitlint
Release:      Release Flow (sprint-based branches)
Protection:   2 approvals, CI + security scans, CODEOWNERS, merge queue
Branch Names: feat/, fix/ (short-lived, < 1 day)
Feature Flags: LaunchDarkly / Unleash / custom
```

#### Open Source Library (contribuidores variados)

```
Strategy:     GitHub Flow (fork model)
Merge:        Squash & Merge
Commits:      Conventional Commits (enforced via CI)
Release:      release-please
Protection:   2 approvals (1 maintainer), CI + coverage, DCO sign-off
Branch Names: Contribuidores usam forks
```

#### Monorepo Multi-Package (time de 15-40 devs)

```
Strategy:     Trunk-Based Development
Merge:        Squash & Merge
Commits:      Conventional Commits com scopes por package
Release:      release-please (monorepo mode) ou changesets
Protection:   1-2 approvals, CI por package afetado, CODEOWNERS por diretorio
Optimization: Sparse checkout, shallow clone em CI, git worktrees
Branch Names: feat/, fix/ com scope: feat/api-rate-limiting
```

---

## 9. Fontes & Referencias

### Documentacao Oficial e Especificacoes

- [nvie.com -- A successful Git branching model (GitFlow original)](https://nvie.com/posts/a-successful-git-branching-model/)
- [Scott Chacon -- GitHub Flow](https://scottchacon.com/2011/08/31/github-flow/)
- [GitHub Docs -- GitHub Flow](https://docs.github.com/en/get-started/using-github/github-flow)
- [GitLab Docs -- Branching Strategies](https://docs.gitlab.com/user/project/repository/branches/strategies/)
- [trunkbaseddevelopment.com](https://trunkbaseddevelopment.com/)
- [Microsoft Learn -- How Microsoft develops with DevOps](https://learn.microsoft.com/en-us/devops/develop/how-microsoft-develops-devops)
- [Azure DevOps Blog -- Release Flow](https://devblogs.microsoft.com/devops/release-flow-how-we-do-branching-on-the-vsts-team/)
- [conventionalcommits.org -- Specification v1.0.0](https://www.conventionalcommits.org/en/v1.0.0/)
- [semver.org -- Semantic Versioning 2.0.0](https://semver.org/)

### Ferramentas e Projetos

- [semantic-release (GitHub)](https://github.com/semantic-release/semantic-release)
- [release-please (Google)](https://github.com/googleapis/release-please)
- [commitlint (GitHub)](https://github.com/conventional-changelog/commitlint/)
- [commitlint -- Local Setup Guide](https://commitlint.js.org/guides/local-setup.html)
- [release-please-action (GitHub Marketplace)](https://github.com/google-github-actions/release-please-action)

### Engineering Practices

- [Google eng-practices -- Code Review](https://google.github.io/eng-practices/review/)
- [Google eng-practices -- The Standard of Code Review](https://google.github.io/eng-practices/review/reviewer/standard.html)
- [Google eng-practices -- What to look for](https://google.github.io/eng-practices/review/reviewer/looking-for.html)
- [Paul Hammant -- Google's Scaled TBD](https://paulhammant.com/2013/05/06/googles-scaled-trunk-based-development/)
- [Paul Hammant -- Google's vs Facebook's TBD](https://paulhammant.com/2014/01/08/googles-vs-facebooks-trunk-based-development/)
- [Martin Fowler -- Feature Toggles](https://martinfowler.com/articles/feature-toggles.html)

### GitHub Features

- [GitHub Docs -- About rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets)
- [GitHub Docs -- About protected branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)
- [GitHub Docs -- About code owners](https://docs.github.com/articles/about-code-owners)
- [GitHub Docs -- About PR merges](https://docs.github.com/articles/about-pull-request-merges)
- [GitHub Docs -- Managing a merge queue](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-a-merge-queue)
- [GitHub Blog -- Merge Queue GA](https://github.blog/news-insights/product-news/github-merge-queue-is-generally-available/)
- [GitHub Blog -- Required reviewer rule GA](https://github.blog/changelog/2026-02-17-required-reviewer-rule-is-now-generally-available/)
- [GitHub Docs -- Signing commits](https://docs.github.com/en/authentication/managing-commit-signature-verification/signing-commits)

### Monorepo e Performance

- [GitHub Blog -- Sparse Checkout](https://github.blog/open-source/git/bring-your-monorepo-down-to-size-with-sparse-checkout/)
- [Git SCM -- git-sparse-checkout](https://git-scm.com/docs/git-sparse-checkout)
- [GitLab Docs -- Improving monorepo performance](https://docs.gitlab.com/user/project/repository/monorepos/)
- [Atlassian -- Git Subtree](https://www.atlassian.com/git/tutorials/git-subtree)
- [Atlassian -- Merging vs Rebasing](https://www.atlassian.com/git/tutorials/merging-vs-rebasing)
- [DataCamp -- Git Worktree Tutorial](https://www.datacamp.com/tutorial/git-worktree-tutorial)

### DORA e Metricas

- [Atlassian -- DORA Metrics](https://www.atlassian.com/devops/frameworks/dora-metrics)
- [Octopus Deploy -- DORA Metrics 2024/25](https://octopus.com/devops/metrics/dora-metrics/)
- [Google Cloud -- Four Keys](https://cloud.google.com/blog/products/devops-sre/using-the-four-keys-to-measure-your-devops-performance)

### Guias e Tutoriais

- [Atlassian -- Gitflow Workflow](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow)
- [Microsoft Engineering Playbook -- PR Template](https://microsoft.github.io/code-with-engineering-playbook/code-reviews/pull-request-template/)
- [Graphite -- PR Template Checklist](https://graphite.com/guides/comprehensive-checklist-github-pr-template)
- [Graphite -- Best merge strategy](https://graphite.com/blog/pull-request-merge-strategy)
- [Kubernetes Contributors -- Pull Requests](https://www.kubernetes.dev/docs/guide/pull-requests/)
- [Git SCM -- Signing Your Work](https://git-scm.com/book/en/v2/Git-Tools-Signing-Your-Work)
- [Aviator -- CODEOWNERS Guide](https://www.aviator.co/blog/a-modern-guide-to-codeowners/)
- [DEV Community -- Branch Naming Conventions](https://dev.to/varbsan/a-simplified-convention-for-naming-branches-and-commits-in-git-il4)

### Feature Flags

- [LaunchDarkly -- Feature Flags 101](https://launchdarkly.com/blog/what-are-feature-flags/)
- [Unleash -- TBD with Feature Flags](https://docs.getunleash.io/guides/trunk-based-development)
- [trunkbaseddevelopment.com -- Feature Flags](https://trunkbaseddevelopment.com/feature-flags/)

---

> **Pesquisa conduzida por:** Prism (research-orqx), Squad Research
> **Metodo:** WebSearch extensivo com verificacao cruzada de todas as afirmacoes
> **Total de fontes consultadas:** 45+ fontes verificadas em tempo real
> **Data:** 2026-04-11
