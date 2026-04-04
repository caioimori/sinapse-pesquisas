# Deep Dive: AI Development Frameworks, NPM Publishing e Convencoes de Projeto

> **Nivel:** DEEP DIVE (Research Depth Pyramid L3)
> **Data:** 2026-04-04
> **Pesquisador:** Prism (Research Orchestrator, squad-research)
> **Objetivo:** Fornecer inteligencia completa para construir e publicar o SINAPSE-AI como o melhor AI development framework no NPM

---

## SUMARIO EXECUTIVO

Este documento consolida pesquisa exaustiva em tres dominios criticos para o SINAPSE-AI:

1. **Frameworks de AI Development** -- Analise comparativa de 15+ frameworks incluindo BMAD-METHOD, AIOX-Core, Claude Code nativo, Cursor, Windsurf, Aider, OpenHands, SWE-agent, Devin, Cline, Continue, Bolt.new, v0, Lovable e orquestradores emergentes.

2. **NPM Publishing** -- Guia completo para publicacao profissional: package.json, CLI tools, monorepo, scoped packages, TypeScript, CI/CD, seguranca, e testes pre-publicacao.

3. **Convencoes de Projeto** -- Nomenclatura de arquivos, estrutura de diretorios, gerenciamento de .env, .gitignore, organizacao de monorepo, e sistemas de scaffolding.

**FINDING principal:** O mercado de AI development frameworks explodiu em 2025-2026. O BMAD-METHOD lidera com 43.5K stars e inspirou diretamente o AIOX-Core (fork). Claude Code nativo atingiu 84.6K stars. Nenhum framework combina orquestracao multi-agente + multi-squad + CLI-first + observability da forma que o SINAPSE faz.

**IMPLICATION:** Existe uma janela de oportunidade para o SINAPSE se posicionar como o framework mais completo, desde que a publicacao no NPM seja profissional e a documentacao seja de classe mundial.

**RECOMMENDATION:** Publicar via `npx sinapse-ai init` com installer interativo, scoped como `@sinapse-ai/core`, usando changesets para versionamento e GitHub Actions para CI/CD.

---

# PARTE 1: ANALISE COMPARATIVA DE FRAMEWORKS

## 1.1 BMAD-METHOD (Breakthrough Method for Agile AI Driven Development)

**Source:** [GitHub - BMAD-METHOD](https://github.com/bmad-code-org/BMAD-METHOD)

| Dimensao | Detalhe |
|----------|---------|
| **Stars/Forks** | 43.5K stars, 5.2K forks |
| **Versao** | v6.2.2 (marco 2026) |
| **Linguagem** | JavaScript (88.1%), HTML, CSS, Python |
| **Licenca** | MIT |
| **Runtime** | Node.js v20+ |
| **Instalacao** | `npx bmad-method install` |

### Arquitetura

- **Padrao:** Modular com ecosystem de modulos (BMM Core, BMB Builder, TEA Test, BMGD Game Dev, CIS Creative)
- **Agentes:** 12+ roles especializados (PM, Architect, Developer, UX, QA, Analyst)
- **Party Mode:** Multiplos agentes colaboram numa unica sessao
- **Scale-Adaptive Intelligence:** Ajusta profundidade de planejamento baseado na complexidade do projeto
- **Ajuda contextual:** `bmad-help` skill da orientacao de proximo passo

### Sistema de Agentes

Os agentes sao definidos como personas com prompts estruturados. Cada agente tem expertise de dominio e workflows pre-definidos. A comunicacao entre agentes acontece via arquivos markdown com checkboxes -- criando audit trail.

### Workflows

34+ workflow templates cobrindo:
- Analysis (brainstorming, requirements)
- Planning (PRD, roadmap)
- Architecture (system design, tech selection)
- Implementation (coding, testing, deployment)

### Configuracao

Arquivos de configuracao incluem `.npmrc`, `package.json`, `eslint.config.mjs`, `prettier.config.mjs`, `.coderabbit.yaml`. Estrutura de pastas: `.augment/`, `.claude-plugin/`, `src/`, `docs/`, `tools/`, `website/`.

### FINDING

BMAD e o framework mais popular do segmento e a inspiracao direta do AIOX-Core e indiretamente do SINAPSE. Seu diferencial e a filosofia de "AI como colaborador expert, nao substituto" e a escala adaptativa.

### IMPLICATION

O SINAPSE precisa se diferenciar do BMAD em dimensoes claras -- multi-squad e a principal. O BMAD nao tem conceito de squads temticos.

### RECOMMENDATION

Referenciar BMAD explicitamente como inspiracao na documentacao do SINAPSE. Posicionar o diferencial em: multi-squad orchestration, Constitution formal, e observability dashboard.

---

## 1.2 AIOX-Core (Synkra AIOS)

**Source:** [GitHub - AIOX-Core](https://github.com/SynkraAI/aiox-core)

| Dimensao | Detalhe |
|----------|---------|
| **Stars/Forks** | ~839+ commits (stars nao divulgados amplamente) |
| **Origem** | Fork/derivado do BMAD-METHOD |
| **Linguagem** | JavaScript |
| **Runtime** | Node.js >= 18.0.0 (v20+ recomendado) |
| **Instalacao** | `npx aiox-core init <project-name>` |

### Arquitetura

- **Padrao:** CLI-First (CLI First -> Observability Second -> UI Third)
- **Inovacao 1 - Agentic Planning:** Agentes dedicados (analyst, PM, architect) colaboram com usuarios para criar PRD e documentos de arquitetura
- **Inovacao 2 - Contextualized Development:** Scrum master transforma planos em stories hyperspecificas com contexto completo embutido
- **Installer interativo:** Usa `@clack/prompts` para UX rica no terminal

### Compatibilidade IDE

| IDE/CLI | Hook Parity | Impacto |
|---------|-------------|---------|
| Claude Code | Completo | Automacao maxima |
| Gemini CLI | Alto | Boa cobertura pre/post-tool |
| Codex CLI | Parcial | Depende de AGENTS.md, MCP, workflow |
| Cursor | Nenhum | Automacao reduzida, baseado em rules |
| GitHub Copilot | Nenhum | Repository instructions + MCP focus |
| AntiGravity | Workflow-based | Integracao via workflows |

### Extensibilidade

Permite criar agentes customizados e "Squads" para dominios alem de software: escrita criativa, estrategia de negocios, saude, educacao.

### FINDING

O AIOX-Core e o framework mais proximo do SINAPSE em conceito. Ambos derivam do BMAD, ambos usam CLI-first, ambos tem conceito de squads. A diferenca e que o SINAPSE tem Constitution formal, hooks de governance, e um ecosistema de 20+ squads.

### IMPLICATION

O AIOX-Core e o concorrente direto mais proximo. O SINAPSE precisa demonstrar superioridade em: depth de squads, governance, quality gates, e production-readiness.

### RECOMMENDATION

Estudar o installer interativo do AIOX (@clack/prompts) como referencia para o installer do SINAPSE. Documentar diferenciais claros vs AIOX na pagina do NPM.

---

## 1.3 Claude Code (Nativo - Anthropic)

**Source:** [Claude Code Overview](https://code.claude.com/docs/en/overview) | [Claude Code Product](https://claude.com/product/claude-code)

| Dimensao | Detalhe |
|----------|---------|
| **Stars** | 84.6K stars, 7.2K forks |
| **Versao** | v2.0.74+ (dez 2025) |
| **Plataforma** | Terminal-native, extensions para VS Code, JetBrains |
| **Modelo** | Claude Opus 4, Sonnet 4 |

### Capabilities Nativas (Built-in)

| Feature | Descricao |
|---------|-----------|
| **CLAUDE.md** | Arquivo markdown no root do projeto que Claude le no inicio de cada sessao. Define coding standards, decisoes de arquitetura, libraries preferidas |
| **Plan Mode** | Exploracao read-only antes de implementacao. Planos armazenados em `.claude/plans/` |
| **Subagents** | Agentes especializados para subtasks (desde jul 2025) |
| **Agent Skills** | Sistema modular de capacidades extensiveis (desde out 2025) com metadata, core instructions, e nested resources |
| **Auto Memory** | Salva aprendizados (build commands, debugging insights) entre sessoes |
| **Hooks** | PreToolUse, PostToolUse, UserPromptSubmit, PreCompact -- extensiveis |
| **Native SDK** | TypeScript e Python SDKs oficiais |
| **Slack Integration** | Integracao nativa com Slack |
| **MCP (Model Context Protocol)** | Protocolo para conectar tools externas |
| **Context Window** | Ate 200K tokens por sessao (Opus 4) |

### O Que Frameworks Adicionam (vs Nativo)

O Claude Code nativo fornece primitivas poderosas. Frameworks como SINAPSE adicionam:

1. **Personas de agente estruturadas** -- Claude Code nao tem conceito de "agente com personalidade e dominio"
2. **Workflows multi-step** -- O Plan Mode e simples; frameworks definem workflows complexos
3. **Multi-squad orchestration** -- Claude Code nao tem conceito de squads
4. **Quality gates formais** -- O nativo nao tem QA loops automatizados
5. **Constitution e governance** -- Nao existe enforcement de regras arquiteturais
6. **Story lifecycle** -- Nao existe conceito de story-driven development
7. **Cross-agent delegation** -- Subagents sao simples vs delegation matrix formal

### FINDING

Claude Code e a plataforma mais popular (84.6K stars) e fornece as primitivas que frameworks estendem. O ecossistema de extensoes (claude-skills com 5.2K stars, oh-my-claudecode trending) esta explodindo.

### IMPLICATION

O SINAPSE opera SOBRE o Claude Code, usando seus hooks, CLAUDE.md, e subagents como fundacao. A proposta de valor e a camada de orquestracao, governance, e squads -- nao substituir o Claude Code.

### RECOMMENDATION

Posicionar SINAPSE como "the enterprise orchestration layer for Claude Code" na comunicacao. Explorar compatibilidade com Gemini CLI e Codex CLI como o AIOX faz.

---

## 1.4 Cursor (Regras e Contexto)

**Source:** [Cursor Rules Docs](https://cursor.com/docs/context/rules) | [awesome-cursorrules](https://github.com/PatrickJS/awesome-cursorrules)

### Sistema de Regras

Cursor usa tres tipos de regras para injecao de contexto:

| Tipo | Local | Escopo |
|------|-------|--------|
| **Project Rules** | `.cursor/rules/*.mdc` | Version-controlled, scoped ao projeto |
| **User Rules** | Settings globais | Pessoal, sempre aplicado |
| **Legacy** | `.cursorrules` (root) | Deprecated, ainda suportado |

### Formato .mdc (Frontmatter)

Cada arquivo `.mdc` tem tres campos de frontmatter que controlam ativacao:

| Campo | Funcao |
|-------|--------|
| `description` | Descricao para o agente decidir relevancia |
| `globs` | Patterns de arquivos que ativam a regra (ex: `*.tsx`) |
| `alwaysApply` | Se `true`, aplica em toda conversa |

### Padrao de Contexto

Antes de cada prompt chegar ao modelo, Cursor verifica quais regras aplicam e prepende o conteudo ao context window. Isso e similar ao sistema de `.claude/rules/` do SINAPSE.

### FINDING

O padrao Cursor de regras por arquivo com glob patterns e muito elegante. O SINAPSE ja implementa algo similar com `.claude/rules/` e frontmatter de paths. A diferenca e que SINAPSE tem rules com enforcement (hooks que bloqueiam), enquanto Cursor rules sao advisory.

### RECOMMENDATION

Manter o sistema atual de rules do SINAPSE. Considerar adicionar um campo `globs` no frontmatter das rules para paridade com Cursor.

---

## 1.5 Windsurf / Codeium (Cascade)

**Source:** [Windsurf Cascade](https://windsurf.com/cascade) | [Windsurf Docs](https://docs.windsurf.com/windsurf/cascade/cascade)

### Arquitetura Cascade

| Feature | Descricao |
|---------|-----------|
| **AI Flows** | Paradigma que permite trabalho autonomo E colaborativo |
| **Persistent Agent** | Le codebase, constroi modelo mental, executa planos multi-step |
| **Planning System** | Agente de planejamento especializado refina plano long-term enquanto modelo principal toma acoes short-term |
| **Todo Lists** | Cascade cria todo lists dentro da conversa para tasks complexas |
| **Context Tracking** | Rastreia edits, commands, clipboard, terminal para inferir intent |
| **SWE-1.5** | Modelo proprietario otimizado para multi-file refactoring |
| **MCP Support** | Figma, Slack, Stripe, PostgreSQL, Playwright |

### FINDING

Windsurf e o competidor mais sofisticado em termos de "flow" -- a experiencia de trabalhar COM o AI sem interrupcoes. O conceito de "planning agent separado do execution agent" e inteligente.

### RECOMMENDATION

Considerar pattern de "planning agent" separado para o SINAPSE em sessoes longas.

---

## 1.6 Aider

**Source:** [Aider.chat](https://aider.chat/) | [GitHub - Aider](https://github.com/Aider-AI/aider)

| Dimensao | Detalhe |
|----------|---------|
| **Stars/Forks** | 42.8K stars, 4.1K forks |
| **Linguagem** | Python (80%) |
| **Commits** | 13.1K+ |
| **Downloads** | 5.7M+ PyPI, 15B tokens/semana |
| **Modelos** | Claude 3.7 Sonnet, DeepSeek R1, GPT-4o, modelos locais |

### Diferenciais

| Feature | Descricao |
|---------|-----------|
| **Repository Map** | Mapa de funcoes, classes, e relacoes do codebase inteiro |
| **Auto Git Commits** | Cada mudanca recebe commit automatico com mensagem descritiva |
| **Chat Modes** | Code, Architect, Ask, Help |
| **Auto Lint/Test** | Testa e faz lint apos cada mudanca |
| **100+ Linguagens** | Suporte amplo via tree-sitter |
| **`/undo`** | Reverte ultimo commit instantaneamente |

### FINDING

Aider e o rei do "pair programming no terminal" com a melhor integracao git de todos os frameworks. O conceito de Repository Map (usando tree-sitter para mapear funcoes e relacoes) e brilhante para dar contexto ao LLM.

### RECOMMENDATION

Estudar o sistema de Repository Map do Aider como possivel feature futura do SINAPSE (code intelligence enrichment).

---

## 1.7 OpenHands (ex-OpenDevin)

**Source:** [GitHub - OpenHands](https://github.com/OpenHands/OpenHands) | [openhands.dev](https://openhands.dev/)

| Dimensao | Detalhe |
|----------|---------|
| **Stars/Forks** | 70.5K stars, 8.8K forks |
| **Linguagem** | Python (72.7%), TypeScript (25.5%) |
| **Commits** | 6.4K+ |
| **SWEBench** | 77.6 |
| **Licenca** | MIT (core), proprietary (enterprise) |

### Arquitetura

| Componente | Descricao |
|-----------|-----------|
| **SDK** | Python library composavel -- "engine" de todo o sistema |
| **CLI** | Interface terminal simples |
| **GUI** | REST API + React SPA |
| **Cloud** | Slack, Jira, Linear integracoes |
| **Event Stream** | Abstracao de actions/observations formando perception-action loop |
| **Multi-Agent** | `AgentDelegateAction` permite delegar subtasks entre agentes |
| **CodeActAgent** | Agente generalista default |
| **BrowsingAgent** | Agente especializado em web browsing |

### Clientes Enterprise

TikTok, VMware, Roche, Amazon, Netflix, Mastercard, Red Hat, MongoDB, Apple, NVIDIA, Google.

### FINDING

OpenHands e o framework open-source mais bem-sucedido em adocao enterprise. O padrao SDK-first (composable library) e o modelo de distribuicao mais forte.

### RECOMMENDATION

Considerar oferecer o SINAPSE como SDK alem de CLI tool -- permitindo integracao programatica.

---

## 1.8 SWE-agent

**Source:** [GitHub - SWE-agent](https://github.com/SWE-agent/SWE-agent) | [Paper NeurIPS 2024](https://arxiv.org/abs/2405.15793)

### Evolucao

| Versao | Status |
|--------|--------|
| **SWE-agent original** | Paper NeurIPS 2024, custom Agent-Computer Interfaces (ACIs) |
| **mini-SWE-agent** | Recomendado atualmente, ~100 linhas Python, mesma performance |
| **SWE-AF (AgentField)** | Fleet autonoma: PM, architects, coders, reviewers, testers |

### Conceito-Chave: Agent-Computer Interfaces (ACIs)

O insight central e que LMs sao um NOVO tipo de end-user que precisa de interfaces especializadas -- nao interfaces humanas. O ACI custom melhora significativamente a capacidade de criar/editar codigo, navegar repositorios, e executar testes.

### FINDING

O conceito de ACI e fundamental: agentes de AI nao devem usar as mesmas interfaces que humanos. O SINAPSE ja faz isso implicitamente com hooks e rules -- mas poderia ser mais explicito.

---

## 1.9 Devin (Cognition Labs)

**Source:** [Cognition.ai](https://cognition.ai/) | [Devin Wikipedia](https://en.wikipedia.org/wiki/Devin_AI)

| Dimensao | Detalhe |
|----------|---------|
| **Tipo** | Closed-source, SaaS |
| **Preco** | $20/mes (Devin 2.0, reduzido de $500) |
| **ARR** | ~$73M (jun 2025) |
| **Modelo** | Proprietario (GPT-4 scale) |

### Arquitetura

| Modulo | Funcao |
|--------|--------|
| **Planner** | Analisa instrucoes NL, cria plano sequencial |
| **Core AI Models** | LLM pre-treinado em code + NL |
| **Devin Wiki** | Documentacao automatica do codebase |
| **Devin Search** | Busca semantica no codigo |
| **Cloud IDE** | Agent-native, multiplos Devins em paralelo |
| **Interactive Planning** | Roadmaps clicaveis e editaveis |

### FINDING

Devin 2.0 demonstra que pricing accessivel ($20/mes) + cloud IDE e um modelo viavel. O crescimento de $1M para $73M ARR em 9 meses e impressionante.

### IMPLICATION

O SINAPSE nao compete diretamente com Devin (que e SaaS closed-source), mas o modelo de "multiple agents in parallel" e inspirador.

---

## 1.10 Cline e Continue

**Source:** [Cline.bot](https://cline.bot) | [GitHub - Cline](https://github.com/cline/cline)

### Cline

| Feature | Descricao |
|---------|-----------|
| **Plataforma** | VS Code extension |
| **Downloads** | ~2M em 6 meses |
| **Arquitetura** | Plan/Act modes |
| **MCP** | Integracao nativa com Model Context Protocol |
| **Context Engineering** | Dynamic Context Management, AST-Based Analysis, Narrative Integrity, Memory Bank |
| **Open-source** | Sim |

### Continue

| Feature | Descricao |
|---------|-----------|
| **Plataforma** | VS Code extension -> CLI (mid-2025 pivot) |
| **Indexacao** | Local vector database (Lancet protocol) para semantic search |
| **Pivot 2025** | De IDE extension para "Continuous AI" -- CLI para code enforcement em PRs |

### FINDING

Cline e o lider de open-source VS Code AI agents. O conceito de "Memory Bank" (captura informacao essencial sem input explicito) e relevante para o SINAPSE.

---

## 1.11 Bolt.new / v0 / Lovable

**Source:** [AI-Driven Prototyping Comparison](https://addyo.substack.com/p/ai-driven-prototyping-v0-bolt-and)

| Tool | Foco | Backend | Database | Preco |
|------|------|---------|----------|-------|
| **Bolt.new** | Full-stack apps | Next.js scaffolding | Supabase | Premium |
| **v0** | UI components | Nao (Vercel frontend) | Nao | Freemium |
| **Lovable** | Full-stack MVPs | React + Supabase | Supabase built-in | $25/mes |

### FINDING

Estas ferramentas focam em SCAFFOLDING rapido -- gerar apps inteiros de prompts. O SINAPSE foca em ORQUESTRACAO de desenvolvimento profissional. Sao complementares, nao concorrentes.

---

## 1.12 Orquestradores Emergentes do Ecossistema Claude Code (2026)

| Framework | Stars | Descricao |
|-----------|-------|-----------|
| **claude-skills** | 5.2K | 220+ skills e agent plugins |
| **oh-my-claudecode** | Trending #1 | Zero-config orchestration, 3-5x speedup |
| **Everything-Claude-Code** | 3.7K (1 dia) | 112 agents, 16 orchestrators, 146 skills |
| **Ruflo** | -- | Multi-agent swarms, RAG, Claude/Codex |
| **Claude Colony** | -- | Multi-agent coordination |
| **ClaudeSwarm** | -- | Swarm intelligence para Claude |

### FINDING

O ecossistema de extensoes Claude Code esta em explosao. O SINAPSE precisa se diferenciar como framework PROFISSIONAL com governance, nao apenas "mais uma colecao de agents".

---

## 1.13 Tabela Comparativa Consolidada

| Framework | Stars | Tipo | Agentes | Multi-Agent | Workflows | CLI | Config Format | Distribuicao |
|-----------|-------|------|---------|-------------|-----------|-----|---------------|-------------|
| **BMAD-METHOD** | 43.5K | Open, MIT | 12+ | Party Mode | 34+ templates | npx | YAML/MD | NPM |
| **AIOX-Core** | N/A | Open | 6+ | Squad concept | Story-driven | npx | YAML/MD | NPM |
| **Claude Code** | 84.6K | Anthropic | Subagents | Basic delegation | Plan Mode | Native | CLAUDE.md | Built-in |
| **Aider** | 42.8K | Open | Single | Nao | 4 chat modes | pip | .aider.conf | PyPI |
| **OpenHands** | 70.5K | Open, MIT | CodeAct+others | AgentDelegate | SDK-based | pip/docker | YAML | PyPI/Docker |
| **SWE-agent** | ~15K | Open | Single | Nao (SWE-AF sim) | ACI-based | pip | Config files | PyPI |
| **Devin** | N/A | Closed, SaaS | Multiple parallel | Cloud IDE | Planner-based | Web | N/A | SaaS |
| **Cline** | ~10K | Open | Single | Nao | Plan/Act | VS Code ext | Settings | Marketplace |
| **Windsurf** | N/A | Proprietario | Cascade | Nao | AI Flows | IDE | Settings | IDE |
| **Cursor** | N/A | Proprietario | Single | Nao | Rules-based | IDE | .mdc rules | IDE |
| **SINAPSE** | -- | Em dev | 20+ squads, 100+ agents | Multi-squad + delegation matrix | SDC, QA Loop, Spec, Brownfield | CLI-first | YAML/MD + Constitution | NPM (planejado) |

---

# PARTE 2: NPM PUBLISHING -- GUIA COMPLETO

## 2.1 Fundamentos do NPM Publishing

### package.json Essencial

```json
{
  "name": "@sinapse-ai/core",
  "version": "1.0.0",
  "description": "AI-Orchestrated System for Full Stack Development",
  "main": "./dist/index.cjs",
  "module": "./dist/index.mjs",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "import": {
        "types": "./dist/index.d.ts",
        "default": "./dist/index.mjs"
      },
      "require": {
        "types": "./dist/index.d.cts",
        "default": "./dist/index.cjs"
      }
    }
  },
  "bin": {
    "sinapse": "./dist/cli/index.mjs"
  },
  "files": [
    "dist/**/*",
    "templates/**/*",
    "README.md",
    "LICENSE"
  ],
  "engines": {
    "node": ">=18.0.0"
  },
  "publishConfig": {
    "access": "public"
  },
  "repository": {
    "type": "git",
    "url": "https://github.com/sinapse-ai/sinapse-ai"
  },
  "keywords": [
    "ai", "agents", "orchestration", "claude-code",
    "development-framework", "multi-agent", "cli"
  ],
  "license": "MIT",
  "author": "Caio Imori <email>"
}
```

### Campos Criticos Explicados

| Campo | Funcao | Nota |
|-------|--------|------|
| `name` | Nome do pacote no registry | `@sinapse-ai/core` para scoped package |
| `version` | Versao semver | Major.Minor.Patch |
| `main` | Entry point CommonJS | Para `require()` |
| `module` | Entry point ESM | Para `import` |
| `types` | Entry point TypeScript declarations | Para autocomplete/type-checking |
| `exports` | Conditional exports (moderno) | Substitui main/module em Node 12+ |
| `bin` | CLI executaveis | Mapeamento comando -> arquivo |
| `files` | Allowlist de arquivos publicados | Mais seguro que `.npmignore` |
| `engines` | Requisitos de runtime | `node >= 18` |
| `publishConfig` | Config de publicacao | `"access": "public"` para scoped packages |

---

## 2.2 CLI Tools no NPM

### Estrutura Minima

```
sinapse-ai/
  bin/
    sinapse.mjs          # #!/usr/bin/env node
  dist/
    cli/
      index.mjs          # Entry point compilado
    lib/
      ...
  package.json
```

### O Shebang

```javascript
#!/usr/bin/env node
// Este shebang DEVE ser a primeira linha do arquivo
// Em *nix: diz ao OS para usar node como interpretador
// Em Windows: npm le e ignora como comentario
```

### O Campo `bin`

```json
// Comando unico (mesmo nome do pacote)
"bin": "./dist/cli/index.mjs"

// Multiplos comandos
"bin": {
  "sinapse": "./dist/cli/index.mjs",
  "sinapse-init": "./dist/cli/init.mjs"
}
```

### Quando instalado globalmente (`npm install -g`):
- npm cria symlink entre o comando e o script
- O comando fica disponivel no PATH do sistema
- `npm link` faz o mesmo localmente para desenvolvimento

### Libraries Populares para CLIs

| Library | Funcao | Uso |
|---------|--------|-----|
| **commander** | Argument parsing | Definir comandos, flags, opcoes |
| **yargs** | Argument parsing (alternativa) | API fluente, completions |
| **@clack/prompts** | UI interativa moderna | Prompts bonitos no terminal (AIOX usa) |
| **inquirer** | UI interativa classica | Prompts, selecao, checkboxes |
| **chalk** | Cores no terminal | Output colorido |
| **ora** | Spinners | Loading indicators |
| **boxen** | Caixas no terminal | Destaque de mensagens |
| **listr2** | Task lists | Progresso de tarefas sequenciais |
| **figlet** | ASCII art | Banners de boas-vindas |
| **commander + zod** | Parsing + validacao | Validacao type-safe de argumentos |

---

## 2.3 Scoped Packages (@scope/package)

**Source:** [NPM Docs - Scoped Packages](https://docs.npmjs.com/creating-and-publishing-scoped-public-packages/)

### Configuracao

```bash
# Criar organizacao no npmjs.com primeiro
npm init --scope=@sinapse-ai

# Publicar como publico (scoped packages sao privados por default!)
npm publish --access public

# Versoes subsequentes nao precisam do --access
npm publish
```

### Vantagens de Scoped Packages

1. **Namespace protegido** -- Ninguem pode publicar `@sinapse-ai/qualquer-coisa`
2. **Organizacao logica** -- `@sinapse-ai/core`, `@sinapse-ai/cli`, `@sinapse-ai/agents`
3. **Visibilidade** -- Sinaliza que e um ecossistema, nao um pacote isolado
4. **Monorepo-friendly** -- Cada pacote do workspace pode ter seu scope

### Recomendacao para SINAPSE

| Pacote | Proposito |
|--------|-----------|
| `sinapse-ai` | Pacote principal (sem scope, para `npx sinapse-ai init`) |
| `@sinapse-ai/core` | Core framework (agentes, tasks, workflows) |
| `@sinapse-ai/cli` | CLI tools |
| `@sinapse-ai/agents` | Agent definitions |
| `@sinapse-ai/squads` | Squad definitions |

---

## 2.4 TypeScript Publishing

**Source:** [TypeScript Publishing Docs](https://www.typescriptlang.org/docs/handbook/declaration-files/publishing.html) | [Liran Tal - TS in 2025](https://lirantal.com/blog/typescript-in-2025-with-esm-and-cjs-npm-publishing)

### Estado em 2025

Publicar TypeScript com suporte dual ESM/CJS ainda e complexo. As opcoes:

| Approach | Tool | Pros | Cons |
|----------|------|------|------|
| **tsup** | Bundler | Controle fino, dual output | Config necessaria |
| **tshy** | High-level build | Simples, usa tsc | Menos controle |
| **ESM-only** | tsc | Mais simples | Quebra CommonJS consumers |
| **JSR** | Registry moderno | TS-first, ESM-only | Ecossistema menor |

### tsconfig.json para Libraries

```json
{
  "compilerOptions": {
    "target": "ES2024",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "outDir": "./dist",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

### tsup.config.ts (Dual Build)

```typescript
import { defineConfig } from 'tsup';

export default defineConfig({
  entry: ['src/index.ts', 'src/cli/index.ts'],
  format: ['cjs', 'esm'],
  dts: true,
  splitting: false,
  sourcemap: true,
  clean: true,
  outExtension({ format }) {
    return {
      js: format === 'cjs' ? '.cjs' : '.mjs',
    };
  },
});
```

---

## 2.5 .npmignore vs .gitignore vs `files`

**Source:** [NPM Blog - Publishing What You Mean](https://blog.npmjs.org/post/165769683050/publishing-what-you-mean-to-publish.html)

### Hierarquia de Decisao

```
files field em package.json (ALLOWLIST)  <-- MAIS SEGURO, RECOMENDADO
     |
     v
.npmignore (BLOCKLIST)                   <-- Se nao tem files field
     |
     v
.gitignore (FALLBACK)                    <-- Se nao tem .npmignore
```

### RECOMENDACAO: Usar `files` (Allowlist)

```json
{
  "files": [
    "dist/**/*",
    "templates/**/*",
    "bin/**/*",
    "README.md",
    "LICENSE",
    "CHANGELOG.md"
  ]
}
```

**Vantagens:**
- Allowlist e mais seguro que blocklist (nao publica acidentalmente)
- Facil de auditar (so o que esta listado vai pro NPM)
- `npm pack --dry-run` mostra exatamente o que sera publicado

### Arquivos SEMPRE Incluidos (nao precisa listar)

- `package.json`
- `README.md` (ou README)
- `LICENSE` (ou LICENCE)
- O arquivo referenciado em `main`

### Arquivos SEMPRE Excluidos

- `.git/`
- `node_modules/`
- `.npmrc`
- `package-lock.json`
- `.gitignore`, `.npmignore`

---

## 2.6 NPM Scripts para Publishing

**Source:** [NPM Docs - Scripts](https://docs.npmjs.com/cli/v11/using-npm/scripts/)

### Lifecycle Hooks

| Script | Quando Executa | Uso |
|--------|----------------|-----|
| `prepare` | Apos `npm install` E antes de `npm publish` | Build do pacote |
| `prepublishOnly` | APENAS antes de `npm publish` | Testes, lint, verificacoes |
| `prepack` | Antes de `npm pack` e `npm publish` | Build final |
| `postpack` | Apos `npm pack` | Cleanup |

### Scripts Recomendados

```json
{
  "scripts": {
    "build": "tsup",
    "clean": "rimraf dist",
    "lint": "eslint src/",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "prepublishOnly": "npm run clean && npm run lint && npm run typecheck && npm run test && npm run build",
    "prepack": "npm run build",
    "version": "conventional-changelog -p angular -i CHANGELOG.md -s && git add CHANGELOG.md",
    "pack:check": "npm pack --dry-run",
    "pub:dry": "npm publish --dry-run"
  }
}
```

---

## 2.7 Versionamento Semantico (SemVer)

### Formato: MAJOR.MINOR.PATCH

| Componente | Quando Incrementar | Exemplo |
|-----------|-------------------|---------|
| **MAJOR** | Breaking changes (API incompativel) | 1.0.0 -> 2.0.0 |
| **MINOR** | Nova funcionalidade (backward compatible) | 1.0.0 -> 1.1.0 |
| **PATCH** | Bug fix (backward compatible) | 1.0.0 -> 1.0.1 |
| **Pre-release** | Versao instavel | 1.0.0-beta.1 |

### Conventional Commits -> SemVer

| Tipo de Commit | Versao Bump |
|---------------|-------------|
| `fix:` | PATCH |
| `feat:` | MINOR |
| `feat!:` ou `BREAKING CHANGE:` | MAJOR |
| `docs:`, `chore:`, `style:` | Nenhum (nao publica nova versao) |

---

## 2.8 Changelog Automatico

**Source:** [conventional-changelog](https://github.com/conventional-changelog/conventional-changelog) | [semantic-release](https://github.com/semantic-release/semantic-release)

### Opcao 1: Semantic Release (Full Automation)

```yaml
# .github/workflows/release.yml
name: Release
on:
  push:
    branches: [main]
jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      id-token: write
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          registry-url: 'https://registry.npmjs.org'
      - run: npm ci
      - run: npx semantic-release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

### Opcao 2: Changesets (Monorepo-Friendly)

```yaml
# .github/workflows/release.yml
name: Release
on:
  push:
    branches: [main]
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm ci
      - uses: changesets/action@v1
        with:
          publish: npm run release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

### RECOMENDACAO

Para o SINAPSE (monorepo com multiplos packages): **Changesets** e a melhor opcao. Para single-package: **Semantic Release** e mais simples.

---

## 2.9 CI/CD para NPM Publishing

### GitHub Actions Workflow Completo

```yaml
name: CI/CD
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18, 20, 22]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck
      - run: npm test

  publish:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    permissions:
      contents: write
      id-token: write  # Para OIDC trusted publishing
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          registry-url: 'https://registry.npmjs.org'
      - run: npm ci
      - run: npm run build
      - run: npm publish --provenance
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

### Seguranca no CI/CD

| Pratica | Descricao |
|---------|-----------|
| **OIDC Trusted Publishing** | Sem tokens long-lived; GitHub e identity provider para NPM |
| **Provenance** | `npm publish --provenance` assina o pacote com Sigstore |
| **2FA** | FIDO-based 2FA obrigatorio (TOTP deprecated) |
| **Granular Tokens** | NPM Granular Access Tokens (Legacy Tokens sunset em 2025) |
| **npm ci** | Usa lockfile estrito, falha se inconsistente |
| **npm audit** | Verificacao de vulnerabilidades antes de publicar |
| **Disable postinstall** | Scripts de lifecycle sao vetor de ataque #1 |

---

## 2.10 Testes Pre-Publicacao

**Source:** [NPM Pack Testing](https://dev.to/scooperdev/use-npm-pack-to-test-your-packages-locally-486e)

### Workflow de Verificacao

```bash
# 1. Ver o que sera publicado (sem criar arquivo)
npm pack --dry-run

# 2. Criar tarball local
npm pack
# Output: sinapse-ai-1.0.0.tgz

# 3. Instalar tarball num projeto de teste
cd /tmp/test-project && npm init -y
npm install /path/to/sinapse-ai-1.0.0.tgz

# 4. Testar CLI
npx sinapse --version

# 5. Simular publicacao
npm publish --dry-run

# 6. Alternativa: npm link (para dev rapido)
cd /path/to/sinapse-ai
npm link
cd /path/to/test-project
npm link sinapse-ai
```

### Comparacao de Metodos

| Metodo | Acuracia | Velocidade | Quando Usar |
|--------|----------|-----------|-------------|
| `npm pack` | Alta (simula real) | Media | Pre-publicacao final |
| `npm link` | Baixa (symlink, ignora `files`) | Alta | Desenvolvimento ativo |
| `yalc` | Alta | Alta | Alternativa a npm link |
| `verdaccio` | Maxima (registry local) | Baixa | CI/CD testing |

---

## 2.11 README para NPM

### Estrutura Recomendada

```markdown
# sinapse-ai

> AI-Orchestrated System for Full Stack Development

[![npm version](https://badge.fury.io/js/sinapse-ai.svg)](...)
[![CI](https://github.com/.../actions/workflows/ci.yml/badge.svg)](...)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](...)
[![Downloads](https://img.shields.io/npm/dm/sinapse-ai.svg)](...)

## Quick Start (5-10 linhas de codigo/comando)

## Features (lista concisa)

## Installation (instrucoes claras)

## Usage (exemplos praticos)

## Configuration (referencia)

## Documentation (link para docs completos)

## Contributing

## License
```

### Badges Essenciais (via shields.io)

| Badge | URL Pattern |
|-------|------------|
| NPM version | `https://badge.fury.io/js/{package}.svg` |
| Downloads | `https://img.shields.io/npm/dm/{package}.svg` |
| CI status | `https://github.com/{org}/{repo}/actions/workflows/ci.yml/badge.svg` |
| License | `https://img.shields.io/badge/License-MIT-yellow.svg` |
| TypeScript | `https://img.shields.io/badge/TypeScript-Ready-blue.svg` |
| Node version | `https://img.shields.io/node/v/{package}.svg` |

---

## 2.12 Modelo de Scaffolding CLI (Referencia)

### Como create-next-app Funciona

1. User executa `npx create-next-app my-app`
2. npx baixa e executa o pacote
3. CLI usa Inquirer/prompts para coletar opcoes (TypeScript? ESLint? Tailwind?)
4. CLI copia templates para o diretorio de destino
5. CLI instala dependencias
6. CLI exibe instrucoes de proximo passo

### Modelo para SINAPSE

```bash
npx sinapse-ai init my-project
# Perguntas interativas:
# 1. Qual tipo de projeto? (Next.js / React / Node.js API / Library)
# 2. Quais squads habilitar? (development, research, commercial, ...)
# 3. Instalar hooks de governance? (sim/nao)
# 4. Configurar GitHub integration? (sim/nao)
# 5. Package manager? (npm / pnpm / yarn)
```

---

# PARTE 3: CONVENCOES DE PROJETO

## 3.1 Nomenclatura de Arquivos

**Source:** [TypeScript Blackbook](http://unional.github.io/typescript-blackbook/docs/guidelines/files_and_folders/naming-convention/) | [Biome Rules](https://biomejs.dev/linter/rules/use-filenaming-convention/)

### Consenso do Ecossistema

| Tipo de Arquivo | Convencao | Exemplo |
|----------------|-----------|---------|
| **Componentes React** | PascalCase | `UserProfile.tsx` |
| **Utilities, modules** | kebab-case | `string-utils.ts` |
| **Classes** | PascalCase | `EventEmitter.ts` |
| **Configuracao** | kebab-case | `eslint.config.mjs` |
| **Testes** | Match do arquivo + `.test` | `string-utils.test.ts` |
| **Tipos/Interfaces** | PascalCase | `UserTypes.ts` |
| **Constants** | kebab-case ou UPPER_CASE | `api-constants.ts` |
| **Hooks React** | camelCase com `use` prefix | `useAuth.ts` |

### Regra Geral

**kebab-case para arquivos, PascalCase para o que e exportado como componente/classe.** Isso evita problemas cross-OS (Linux e case-sensitive, Windows nao).

---

## 3.2 Estrutura de Diretorios

### Pattern: Monorepo Moderno

```
sinapse-ai/
  apps/               # Aplicacoes (dashboard, docs site)
    dashboard/
    docs/
  packages/           # Packages NPM publicaveis
    core/             # @sinapse-ai/core
    cli/              # @sinapse-ai/cli
    agents/           # @sinapse-ai/agents
    squads/           # @sinapse-ai/squads
    shared/           # @sinapse-ai/shared (utilities)
  templates/          # Templates de scaffold
  docs/               # Documentacao do framework
  scripts/            # Build scripts, CI helpers
  .github/            # GitHub Actions, templates
  .changeset/         # Changesets config
  turbo.json          # Turborepo config
  pnpm-workspace.yaml # Workspace definition
  package.json        # Root package.json
  tsconfig.json       # Root TS config
```

### Pattern: Feature-Based (Dentro de Cada Package)

```
packages/core/
  src/
    agents/           # Agent system
      agent-loader.ts
      agent-registry.ts
    tasks/            # Task system
      task-runner.ts
      task-validator.ts
    workflows/        # Workflow orchestration
      workflow-engine.ts
    hooks/            # Hook system
      hook-runner.ts
    config/           # Configuration
      config-loader.ts
    utils/            # Shared utilities
      file-utils.ts
    index.ts          # Public API
  tests/
    agents/
    tasks/
  package.json
  tsconfig.json
```

---

## 3.3 Gerenciamento de .env

**Source:** [dotenv-flow](https://github.com/kerimdzhanov/dotenv-flow)

### Hierarquia de Arquivos

```
.env                  # Defaults para todos os ambientes (NAO committed)
.env.local            # Override local (NAO committed)
.env.development      # Variaveis de desenvolvimento
.env.test             # Variaveis de teste
.env.production       # Variaveis de producao
.env.development.local # Override local de dev (NAO committed)
.env.example          # Template com placeholders (COMMITTED)
```

### .gitignore para .env

```gitignore
# Environment variables
.env
.env.local
.env.*.local
.env.development
.env.production
# MAS: .env.example e COMMITTED
!.env.example
```

### Best Practices

1. **Sempre** criar `.env.example` com todas as variaveis necessarias (sem valores reais)
2. **Nunca** colocar `NEXT_PUBLIC_*` com secrets (sao publicas no frontend)
3. **Validar** variaveis no startup (usar zod ou joi para schema validation)
4. **Centralizar** acesso via config object, nao `process.env` espalhado
5. **Producao**: Usar secrets manager (AWS Secrets Manager, Vault), nao .env

---

## 3.4 .gitignore Best Practices

### Template para Projeto TypeScript/Node.js

```gitignore
# Dependencies
node_modules/
.pnp
.pnp.js

# Build outputs
dist/
build/
.next/
out/
*.tsbuildinfo

# Environment
.env
.env.local
.env.*.local

# IDE
.idea/
.vscode/settings.json
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# Debug logs
npm-debug.log*
yarn-debug.log*
yarn-error.log*
pnpm-debug.log*

# Testing
coverage/
.nyc_output/

# Cache
.eslintcache
.turbo/
.vite/

# Misc
*.tgz
.vercel
```

### Regras Importantes

1. **SEMPRE** commitar lock files (`package-lock.json`, `pnpm-lock.yaml`)
2. **NUNCA** commitar `node_modules/`
3. **NUNCA** commitar arquivos de build (`dist/`)
4. **NUNCA** commitar secrets (`.env`, chaves, tokens)
5. Usar GitHub's Node.gitignore como base e adaptar

---

## 3.5 Organizacao de Monorepo

**Source:** [Monorepo Tools](https://monorepo.tools/) | [Turborepo](https://turbo.build/)

### pnpm-workspace.yaml

```yaml
packages:
  - 'packages/*'
  - 'apps/*'
```

### turbo.json

```json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"]
    },
    "test": {
      "dependsOn": ["build"]
    },
    "lint": {},
    "typecheck": {
      "dependsOn": ["^build"]
    }
  }
}
```

### Recomendacao: Turborepo + pnpm

| Tool | Papel |
|------|-------|
| **pnpm** | Package manager (workspaces nativos, eficiente em disco) |
| **Turborepo** | Task orchestration (caching, paralelismo) |
| **Changesets** | Versionamento e publishing de multiplos packages |

---

## 3.6 Documentacao e Config Files

### Arquivos na Raiz

| Arquivo | Proposito | Committed? |
|---------|-----------|-----------|
| `README.md` | Documentacao principal | Sim |
| `CONTRIBUTING.md` | Guia para contribuidores | Sim |
| `CHANGELOG.md` | Historico de mudancas (auto-gerado) | Sim |
| `LICENSE` | Licenca do projeto | Sim |
| `CODE_OF_CONDUCT.md` | Codigo de conduta | Sim |
| `.editorconfig` | Config cross-editor | Sim |
| `tsconfig.json` | Config TypeScript root | Sim |
| `eslint.config.mjs` | Config ESLint (flat config) | Sim |
| `prettier.config.mjs` | Config Prettier | Sim |
| `.gitignore` | Exclusoes do git | Sim |
| `.npmrc` | Config npm (ex: `auto-install-peers=true`) | Sim |
| `.nvmrc` | Versao Node.js | Sim |

---

# CONCLUSOES E RECOMENDACOES ESTRATEGICAS

## Posicionamento

O SINAPSE ocupa um espaco unico no mercado:

| Dimensao | SINAPSE vs Concorrentes |
|----------|------------------------|
| **Multi-squad** | Unico framework com 20+ squads tematicos |
| **Constitution** | Unico com governance formal e enforcement via hooks |
| **Quality gates** | QA Loop, 4 workflows formais (SDC, QA Loop, Spec Pipeline, Brownfield) |
| **Story-driven** | Documentation-first como principio inegociavel |
| **Safe collaboration** | Auto-branch, auto-sync, auto-resolve, auto-PR |

## Plano de Publicacao Recomendado

### Fase 1: MVP (Single Package)

```bash
npx sinapse-ai init my-project
```

Publicar como `sinapse-ai` (unscoped) no NPM com:
- CLI installer interativo (@clack/prompts)
- Templates para Next.js, Node.js API, Library
- Agent definitions embutidos
- Squad definitions embutidos
- Hooks de governance

### Fase 2: Monorepo Split

```
@sinapse-ai/core       # Engine central
@sinapse-ai/cli        # CLI tools
@sinapse-ai/agents     # Agent definitions
@sinapse-ai/squads     # Squad marketplace
@sinapse-ai/hooks      # Governance hooks
```

### Fase 3: Ecosystem

- Squad marketplace (community-contributed squads)
- Plugin system para extensoes
- Dashboard de observability
- VSCode extension

## Stack Tecnica Recomendada

| Componente | Tool |
|-----------|------|
| Linguagem | TypeScript |
| Build | tsup (dual ESM/CJS) |
| Package Manager | pnpm |
| Monorepo | Turborepo |
| Versionamento | Changesets |
| CI/CD | GitHub Actions + OIDC trusted publishing |
| Testes | Vitest |
| Lint | ESLint flat config + Biome |
| CLI UI | @clack/prompts |
| Argument Parsing | commander + zod |

---

# FONTES

## Frameworks
- [BMAD-METHOD GitHub](https://github.com/bmad-code-org/BMAD-METHOD)
- [AIOX-Core GitHub](https://github.com/SynkraAI/aiox-core)
- [Claude Code Overview](https://code.claude.com/docs/en/overview)
- [Claude Code Product](https://claude.com/product/claude-code)
- [Cursor Rules Docs](https://cursor.com/docs/context/rules)
- [Windsurf Cascade](https://windsurf.com/cascade)
- [Aider GitHub](https://github.com/Aider-AI/aider)
- [OpenHands GitHub](https://github.com/OpenHands/OpenHands)
- [SWE-agent GitHub](https://github.com/SWE-agent/SWE-agent)
- [Cognition AI - Devin](https://cognition.ai/)
- [Cline GitHub](https://github.com/cline/cline)
- [awesome-claude-code](https://github.com/hesreallyhim/awesome-claude-code)
- [claude-skills](https://github.com/alirezarezvani/claude-skills)

## NPM Publishing
- [NPM Docs - Scoped Packages](https://docs.npmjs.com/creating-and-publishing-scoped-public-packages/)
- [NPM Docs - Scripts](https://docs.npmjs.com/cli/v11/using-npm/scripts/)
- [Snyk - Modern npm Package](https://snyk.io/blog/best-practices-create-modern-npm-package/)
- [TypeScript Publishing Docs](https://www.typescriptlang.org/docs/handbook/declaration-files/publishing.html)
- [Liran Tal - TS in 2025](https://lirantal.com/blog/typescript-in-2025-with-esm-and-cjs-npm-publishing)
- [semantic-release](https://github.com/semantic-release/semantic-release)
- [Changesets](https://github.com/changesets/changesets)
- [NPM Security - OWASP](https://cheatsheetseries.owasp.org/cheatsheets/NPM_Security_Cheat_Sheet.html)
- [NPM Provenance - Sigstore](https://dev.to/dataformathub/npm-security-2025-why-provenance-and-sigstore-change-everything-2m7j)

## Convencoes
- [TypeScript Blackbook - Naming](http://unional.github.io/typescript-blackbook/docs/guidelines/files_and_folders/naming-convention/)
- [Monorepo Tools](https://monorepo.tools/)
- [Turborepo Docs](https://turbo.build/)
- [dotenv-flow](https://github.com/kerimdzhanov/dotenv-flow)
- [GitHub Node.gitignore](https://github.com/github/gitignore/blob/main/Node.gitignore)

## Metricas de Comunidade (2026)
- [AI Dev Tool Rankings - LogRocket](https://blog.logrocket.com/ai-dev-tool-power-rankings/)
- [Best AI Coding Agents - faros.ai](https://www.faros.ai/blog/best-ai-coding-agents-2026)
- [Agentic Framework Explosion - Pebblous](https://blog.pebblous.ai/blog/agentic-framework-explosion/en/index.html)

---

*Pesquisa conduzida por Prism (Research Operations Conductor, squad-research)*
*Nivel: DEEP DIVE (L3) | Fontes: 30+ | Tempo: multi-hora*
