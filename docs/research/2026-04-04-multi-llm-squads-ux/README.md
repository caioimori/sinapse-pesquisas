# Pesquisa DEFINITIVA: Multi-LLM Compatibility, Squad Creation Pipeline & CLI UX

> **Research Level:** DEFINITIVE (Nivel 4 - Research Depth Pyramid)
> **Data:** 2026-04-04
> **Pesquisador:** Prism (Research Operations Conductor)
> **Squad:** squad-research
> **Fontes:** 40+ fontes, Tiers 1-5
> **Objetivo:** Fundamentar decisoes arquiteturais para o SINAPSE-AI Framework

---

## Indice

- [PART 1: Multi-LLM Compatibility](#part-1-multi-llm-compatibility)
  - [1.1 Arquitetura de Cada LLM CLI/IDE](#11-arquitetura-de-cada-llm-cliide)
  - [1.2 Design do Framework Universal](#12-design-do-framework-universal)
  - [1.3 Sistema de Instalacao e Configuracao](#13-sistema-de-instalacao-e-configuracao)
- [PART 2: Squad Creation Pipeline](#part-2-squad-creation-pipeline)
  - [2.1 Analise da Estrutura Atual de Squads](#21-analise-da-estrutura-atual-de-squads)
  - [2.2 Best Practices para Criacao de Squads](#22-best-practices-para-criacao-de-squads)
  - [2.3 Marketplace e Distribuicao de Squads](#23-marketplace-e-distribuicao-de-squads)
- [PART 3: CLI UX & Branding](#part-3-cli-ux--branding)
  - [3.1 Best Practices de CLI UX](#31-best-practices-de-cli-ux)
  - [3.2 SINAPSE CLI Branding](#32-sinapse-cli-branding)
- [Recommendations Consolidadas](#recommendations-consolidadas)
- [Sources](#sources)

---

# PART 1: Multi-LLM Compatibility

## 1.1 Arquitetura de Cada LLM CLI/IDE

### Tabela Comparativa Consolidada

| Dimensao | Claude Code | Codex CLI | Gemini CLI | Cursor | GitHub Copilot | Windsurf | Kimi Code | Kiro (ex-Amazon Q) |
|----------|-------------|-----------|------------|--------|----------------|----------|-----------|---------------------|
| **Config File** | `.claude/settings.json` | `~/.codex/config.toml` | `~/.gemini/settings.json` | `.cursor/rules/*.mdc` | `.github/copilot-instructions.md` | `.windsurf/rules/*.md` | `~/.kimi/config.toml` | `.kiro/settings/cli.json` |
| **Instructions File** | `CLAUDE.md` | `AGENTS.md` | `GEMINI.md` | `.cursor/rules/*.mdc` | `.github/copilot-instructions.md` | `.windsurf/rules/*.md` | `AGENTS.md` (fallback) | `.kiro/steering/*.md` |
| **Agent System** | Sub-agents (Agent Teams) | Subagents via `[agents]` em config.toml | `.gemini/agents/*.md` (YAML frontmatter) | Nao nativo (via rules) | `.github/agents/*.agent.md` | Nao nativo | `--agent-file` flag | `.kiro/agents/` |
| **Memory** | `CLAUDE.md` + `/memory` | Transcripts locais + `codex resume` | `/memory` commands + GEMINI.md | Memories (auto-geradas por Cascade) | Copilot Spaces | Memories (auto) + global_rules.md | Session persistence | Steering files |
| **MCP Support** | Sim (nativo, robusto) | Sim (config.toml + `codex mcp`) | Sim (settings.json + agent-level) | Sim (v2.6+ MCP Apps) | Sim (agent profiles + repo settings) | Sim | Sim (ACP nativo) | Sim (mcp.json) |
| **Hooks/Events** | 5 eventos (PreToolUse, PostToolUse, UserPromptSubmit, PreCompact, Stop) | 5 eventos (SessionStart, PreToolUse, PostToolUse, UserPromptSubmit, Stop) | Nao documentado | Automations (v2.6+) | Nao documentado | Nao documentado | Nao documentado | Nao documentado |
| **Permission Model** | Deny/Allow rules em settings.json | 3 modos: Auto, Read-only, Full Access | Policy engine (policy.toml) | Nao granular | tools property em agent profiles | System-level rules | Sandbox mode | Trust boundary |
| **Context Window** | 1M tokens (Opus 4.6) | 192K tokens | 1M tokens (Gemini 3 Pro) | Varia por modelo | Varia por modelo | Varia por modelo | 256K tokens (K2.5) | Varia |
| **Open Source** | Nao | Sim (Rust) | Sim | Nao | Nao | Nao | Sim | Parcial (Q CLI era OSS, Kiro nao) |
| **Preco** | API pay-as-you-go ou Max $100-200/mes | API token rates | Free tier (1000 req/dia Flash) | $20/mes | Incluido no GitHub plan | $15/mes | Free tier generoso | Free tier AWS |

---

### 1.1.1 Claude Code (Referencia Principal)

**FINDING:** Claude Code e a plataforma com o sistema de configuracao mais maduro e granular entre todos os LLM CLIs.

**Arquitetura de Configuracao:**
- **5 camadas de hierarquia** (cascading de maior para menor prioridade): Managed Settings > Command Line > Local Project (.claude/settings.local.json) > Shared Project (.claude/settings.json) > User Defaults (~/.claude/settings.json)
- **CLAUDE.md** como arquivo de instrucoes do projeto, com hierarquia por diretorio (mais proximo ao arquivo editado tem prioridade)
- **Rules system** em `.claude/rules/*.md` com carregamento automatico
- **Hooks** em 5 eventos: PreToolUse, PostToolUse, UserPromptSubmit, PreCompact, Stop
- **Permission model** granular com deny/allow rules em settings.json
- **Agent Teams** (sub-agents) para paralelismo de tarefas

**IMPLICATION:** SINAPSE foi construido sobre Claude Code e aproveita 100% dessas capacidades. Qualquer camada de abstracacao multi-IDE deve preservar essa riqueza quando no Claude Code e degradar graciosamente nos outros.

**RECOMMENDATION:** Claude Code deve continuar como plataforma "first-class citizen". A compatibilidade com outros IDEs deve ser aditiva, nunca subtrativa.

---

### 1.1.2 OpenAI Codex CLI

**FINDING:** Codex CLI e o concorrente mais proximo em arquitetura ao Claude Code, com AGENTS.md, hooks, MCP e subagents.

**Arquitetura de Configuracao:**
- **config.toml** em `~/.codex/config.toml` como config principal
  - Suporta `[agents]` para definir roles de subagents
  - Suporta profiles via `--profile` flag
  - `project_doc_fallback_filenames` permite fallback para outros nomes alem de AGENTS.md
  - `project_doc_max_bytes = 65536` (64KB max por arquivo de instrucoes)
- **AGENTS.md** com hierarquia similar ao CLAUDE.md:
  - `~/.codex/AGENTS.override.md` > `~/.codex/AGENTS.md` (global)
  - Em cada diretorio do path: `AGENTS.override.md` > `AGENTS.md` > fallback names
  - Concatena todos os niveis encontrados
- **hooks.json** com 5 eventos: SessionStart, PreToolUse, PostToolUse, UserPromptSubmit, Stop
  - Discovery: `~/.codex/hooks.json` + `<repo>/.codex/hooks.json`
  - Matcher regex para filtrar eventos
  - Hooks concorrentes (nao bloqueantes entre si)
  - **LIMITACAO CRITICA: Hooks desativados no Windows**
- **MCP** configuravel via config.toml (STDIO ou streaming HTTP)
- **Subagents** para paralelismo, com addresses path-based (`/root/agent_a`)
- **3 modos de permissao:** Auto (default), Read-only, Full Access
- **Open source (Rust)** -- modelo: codex-mini-latest e GPT-5.3-Codex
- **SWE-bench:** 77.3% no Terminal-Bench 2.0

**IMPLICATION:** A arquitetura do Codex e MUITO similar ao Claude Code. O SINAPSE pode gerar config para ambos com transformacoes relativamente simples. O maior gap e a ausencia de hooks no Windows.

**RECOMMENDATION:** Codex CLI deve ser o segundo target de compatibilidade. A transformacao CLAUDE.md -> AGENTS.md e quase 1:1. Hooks precisarao de fallback no Windows.

---

### 1.1.3 Google Gemini CLI

**FINDING:** Gemini CLI tem o sistema de agents mais sofisticado entre os concorrentes, com YAML frontmatter completo e isolation model robusto.

**Arquitetura de Configuracao:**
- **settings.json** em `~/.gemini/settings.json` para config global
- **GEMINI.md** com 3-tier hierarchy:
  - **Global:** `~/.gemini/GEMINI.md`
  - **Workspace:** CWD + parent directories ate git root ou home
  - **JIT (Just-in-time):** Auto-scan de subdiretorios (limite configuravel: `context.discoveryMaxDirs = 200`)
- **Import syntax:** `@file.md` para modularizar GEMINI.md (paths relativos e absolutos)
- **context.fileName** em settings.json aceita array: `["AGENTS.md", "CONTEXT.md", "GEMINI.md"]`
- **Agents** em `.gemini/agents/*.md` com YAML frontmatter rico:
  ```yaml
  ---
  name: agent-slug
  description: What the agent does
  kind: local  # ou "remote" para Agent2Agent
  tools: [list]  # wildcards: *, mcp_*, mcp_server-name_*
  mcpServers: {inline MCP configs}
  model: gemini-3-flash-preview
  temperature: 0.2
  max_turns: 10
  timeout_mins: 10
  ---
  System prompt aqui...
  ```
- **Tool permissions** granulares com wildcards
- **Isolation model:** Cada subagent tem context loop independente
- **Invocacao:** Automatica (main agent roteia) ou forcada (`@agent-name`)
- **Policy engine:** `policy.toml` com regras por subagent
- **Memory:** `/memory show`, `/memory reload`, `/memory add`
- **Open source** -- modelo: Gemini 3 Pro (1M tokens context)

**IMPLICATION:** O Gemini CLI JA suporta o conceito de squads nativamente (agents em diretorio, com isolation e routing). O formato de agents do SINAPSE pode ser transpilado para `.gemini/agents/*.md` quase diretamente. O `context.fileName` aceitando array e um gift -- pode ler AGENTS.md diretamente.

**RECOMMENDATION:** Gemini CLI deve ser o terceiro target. O mapeamento de squads SINAPSE -> `.gemini/agents/` e natural. Usar `context.fileName: ["AGENTS.md", "GEMINI.md"]` para compatibilidade.

---

### 1.1.4 Cursor

**FINDING:** Cursor migrou do `.cursorrules` monolitico para um sistema modular `.cursor/rules/*.mdc` com 4 tipos de regras.

**Arquitetura de Configuracao:**
- **Legacy:** `.cursorrules` na raiz (deprecated, ainda funciona)
- **Novo:** `.cursor/rules/*.mdc` (MDC = Markdown Cursor)
- **4 tipos de regras** determinados pelo frontmatter:
  1. **Always** (`alwaysApply: true`) -- sempre injetado no contexto
  2. **Auto-Attach** (globs definidos, `alwaysApply: false`) -- injetado quando arquivo matching esta no contexto
  3. **Agent** (description presente, sem globs nem alwaysApply) -- AI consulta quando achar relevante
  4. **Manual** (nenhum dos acima) -- nunca automatico, precisa ser referenciado
- **Frontmatter MDC:**
  ```
  ---
  description: Short description of the rule
  globs: "*.ts, src/**/*.py"
  alwaysApply: true
  ---
  Rule content in markdown...
  ```
- **Escopo por diretorio:** Subdiretorios podem ter seus proprios `.cursor/rules/`
- **MCP:** Suportado desde v2.6 (Marco 2026), com MCP Apps em cloud sandbox
- **Automations:** Trigger-based workflows que usam MCPs configurados
- **Sem sistema nativo de agents** (usa rules para guiar comportamento)
- **Sem hooks nativos** (Automations sao o equivalente mais proximo)

**IMPLICATION:** A transformacao SINAPSE -> Cursor requer decomposicao. O CLAUDE.md monolitico precisa ser splitado em multiplos arquivos .mdc com frontmatter adequado. Rules com `paths:` no SINAPSE mapeiam para `globs:` no Cursor.

**RECOMMENDATION:** Criar gerador que decomponha as rules do SINAPSE em arquivos `.cursor/rules/*.mdc` individuais. Rules "always-on" usam `alwaysApply: true`, rules com paths usam `globs`. Agent personas ficam como type "Agent" com description.

---

### 1.1.5 GitHub Copilot

**FINDING:** Copilot evoluiu para suporte completo de custom agents (.agent.md) com MCP em Marco 2026, alem do ja existente copilot-instructions.md.

**Arquitetura de Configuracao:**
- **3 tipos de instrucoes:**
  1. **Repository-wide:** `.github/copilot-instructions.md` -- sempre incluido em todas as interacoes
  2. **Path-specific:** `.github/instructions/NAME.instructions.md` com glob patterns em frontmatter
  3. **Agent instructions:** `AGENTS.md`, `CLAUDE.md`, ou `GEMINI.md` na raiz (leitura por agentic mode)
- **Custom Agents** (Marco 2026):
  - Localizacao: `.github/agents/*.agent.md`
  - YAML frontmatter com: name, description, target, tools, model, mcp-servers, metadata
  - Tool aliases: execute, read, edit, search, agent, web, todo
  - MCP inline: `mcp-servers:` com config YAML completa
  - Max 30,000 caracteres por agent profile
  - Secrets via `${{ secrets.NAME }}` syntax
- **Copilot Spaces:** Workspaces curados de conhecimento para contexto
- **Prioridade:** Personal > Repository > Organization instructions
- **Auto-gerador:** Copilot cloud agent pode gerar `copilot-instructions.md` automaticamente

**IMPLICATION:** O Copilot agora suporta agents de forma similar ao SINAPSE. O `.github/agents/` e um target viavel para gerar agent definitions. O formato de tools (aliases como "execute", "read", "edit") e mais simples que o SINAPSE mas mapeavel.

**RECOMMENDATION:** Gerar `.github/agents/*.agent.md` a partir das definicoes de agents do SINAPSE. Mapear tasks para tool aliases do Copilot. Usar `.github/copilot-instructions.md` para instrucoes globais (equivalente ao CLAUDE.md).

---

### 1.1.6 Windsurf (Codeium)

**FINDING:** Windsurf tem sistema de rules com budget limitado (12K caracteres total) e memories auto-geradas pelo Cascade.

**Arquitetura de Configuracao:**
- **Rules:** `.windsurf/rules/*.md` em Markdown
  - Ativacao: `always_on`, conditional, ou manual
  - Discovery: todos os `.windsurf/rules/` no workspace e subdiretorios
  - Git-aware: busca ate git root
  - Deduplicacao automatica em multi-folder workspaces
- **Limits:**
  - Per-rule: 6,000 caracteres max
  - Total: 12,000 caracteres max (ativo simultaneamente)
  - Priorizacao quando excede: Global > Workspace, dentro de cada escopo `always_on` primeiro
  - **Rules que nao cabem no budget sao silenciosamente dropadas**
- **Memories:** Auto-geradas pelo Cascade, persistem entre sessoes
- **Global rules:** `~/.codeium/windsurf/memories/global_rules.md`
- **System-level rules:** Deploy global, nao editavel por end users (ideal para org-wide)
- **Context pipeline:** Rules > Memories > Open files > Indexed retrieval > Recent actions
- **MCP:** Suportado
- **Windsurf Rules Directory:** Catalogo curado de regras exemplo pela equipe Windsurf

**IMPLICATION:** O budget de 12K caracteres e uma limitacao SEVERA para o SINAPSE (que tem rules complexas e extensas). A geracao para Windsurf precisa de compressao agressiva -- priorizar as rules mais criticas.

**RECOMMENDATION:** Criar versao "compressed" das rules do SINAPSE para Windsurf. Priorizar: Constitution (top), agent authority, safe collaboration. Descartar regras de baixa prioridade. Alertar usuario sobre limitacao de budget.

---

### 1.1.7 Kimi Code CLI

**FINDING:** Kimi Code e um competidor emergente da Moonshot AI (China) com suporte a AGENTS.md e ACP (Agent Client Protocol).

**Arquitetura de Configuracao:**
- **config.toml** em `~/.kimi/config.toml`
- **Prioridade de context files:** `AGENTS.md` > `.cursorrules` > `KIMI.md`
- **Agent files:** YAML format com `--agent-file` flag
- **System prompts:** Markdown templates com `${VAR}` syntax
- **Built-in variables:** KIMI_NOW, KIMI_WORK_DIR, KIMI_WORK_DIR_LS, KIMI_AGENTS_MD, KIMI_SKILLS
- **ACP (Agent Client Protocol):** Suporte nativo para integracao com IDEs
- **Modelo:** K2.5 com 256K context, 100 tok/s output
- **Open source** em github.com/MoonshotAI/kimi-cli
- **Limitacao atual:** AGENTS.md nao e auto-loaded -- precisa ser referenciado manualmente (feature request aberto)

**IMPLICATION:** Kimi Code e relevante por ser popular na Asia e por suportar AGENTS.md como formato primario. Quando o auto-load for implementado, sera compativel out-of-the-box com AGENTS.md gerado pelo SINAPSE.

**RECOMMENDATION:** Monitorar evolucao. Nao priorizar como target imediato, mas garantir que AGENTS.md gerado pelo SINAPSE seja compativel.

---

### 1.1.8 Kiro CLI (ex-Amazon Q Developer)

**FINDING:** Amazon Q Developer CLI foi renomeado para Kiro CLI, com migracao de `.amazonq/` para `.kiro/`.

**Arquitetura de Configuracao:**
- **Migracao automatica** de `.amazonq/` para `.kiro/` na instalacao
- **Estrutura `.kiro/`:**
  - `settings/mcp.json` -- MCP servers
  - `settings/cli.json` -- Config global
  - `prompts/` -- Prompts salvos
  - `agents/` -- Custom agents (JSON format)
  - `steering/*.md` -- Steering rules (equivalente a rules)
- **Backward compatibility:** Le `.amazonq/` se `.kiro/` nao existe
- **Steering = Rules:** Formato Markdown em `.kiro/steering/`
- **Agent format:** JSON com schema reference, nao Markdown
- **MCP:** Suportado via mcp.json

**IMPLICATION:** Kiro e relevante para usuarios AWS mas tem formato proprio (JSON agents, steering em vez de rules). A compatibilidade requer transformacao mais significativa.

**RECOMMENDATION:** Baixa prioridade. Suportar via geracao de `.kiro/steering/` a partir das rules, mas agents em JSON sao um formato diferente que requer mapeamento custom.

---

### O Elefante na Sala: AGENTS.md como Standard Universal

**FINDING:** AGENTS.md emergiu como o standard universal para instrucoes de coding agents, ja adotado por 20,000+ repositorios e suportado por 24+ ferramentas. E stewarded pela Agentic AI Foundation (Linux Foundation).

**Detalhes Criticos:**
- **Formato:** Markdown puro, sem campos obrigatorios
- **Secoes comuns:** Project overview, Build/test commands, Code style, Testing, Security, Commit format
- **Hierarquia:** Nearest file wins (mais proximo ao arquivo editado)
- **Monorepos:** Suporta nested AGENTS.md por subprojeto
- **Governanca:** Linux Foundation (Agentic AI Foundation), co-criado por OpenAI, Google, Cursor, Factory
- **Ferramentas suportadas:** Codex, Jules, Cursor, Copilot, Windsurf, Kimi Code, Zed, JetBrains, Warp, e mais
- **Relacao com tool-specific files:**
  - Copilot le `AGENTS.md`, `CLAUDE.md`, ou `GEMINI.md` no agentic mode
  - Gemini CLI pode ser configurado para ler `AGENTS.md` via `context.fileName`
  - Codex CLI tem `project_doc_fallback_filenames` que pode incluir qualquer nome

**IMPLICATION:** O SINAPSE NAO PRECISA reinventar o formato de instrucoes. O AGENTS.md ja e o standard. A estrategia deve ser: gerar AGENTS.md como formato universal + gerar arquivos tool-specific para features avanacadas (hooks, rules granulares, agent definitions).

**RECOMMENDATION:** Adotar estrategia de "AGENTS.md como base + tool-specific overrides":
1. Gerar `AGENTS.md` com instrucoes universais (funciona em 24+ ferramentas)
2. Gerar `CLAUDE.md` com instrucoes avancadas Claude-specific (hooks, rules, deny/allow)
3. Gerar `.gemini/agents/*.md` para agent definitions no Gemini
4. Gerar `.cursor/rules/*.mdc` para rules no Cursor
5. Gerar `.github/agents/*.agent.md` para agents no Copilot
6. Tudo a partir de UMA fonte de verdade (o squad definition do SINAPSE)

---

## 1.2 Design do Framework Universal

### Arquitetura da Camada de Abstracao

Com base na pesquisa, a arquitetura proposta para o SINAPSE multi-IDE:

```
                    SINAPSE Source of Truth
                    =====================
                    squad.yaml
                    agents/*.md (SINAPSE format)
                    tasks/*.md
                    knowledge-base/*.md
                    rules/*.md
                    workflows/*.yaml
                           |
                           v
                    +-------------+
                    | Transpiler  |
                    | Engine      |
                    +-------------+
                           |
          +------+------+------+------+------+------+
          |      |      |      |      |      |      |
          v      v      v      v      v      v      v
       CLAUDE  AGENTS GEMINI CURSOR COPILOT WINDSURF KIRO
       .md     .md    .md    rules  agents  rules   steering
       rules/  hooks  agents .mdc   .agent  .md     .md
       hooks   .json  .md           .md
       settings
```

### Mapeamento de Features por Plataforma

| Feature SINAPSE | Claude Code | Codex CLI | Gemini CLI | Cursor | Copilot | Windsurf | Kimi Code | Kiro |
|-----------------|-------------|-----------|------------|--------|---------|----------|-----------|------|
| **Constitution/Global Rules** | CLAUDE.md | AGENTS.md | GEMINI.md | `.cursor/rules/` (alwaysApply) | copilot-instructions.md | global_rules.md | AGENTS.md | steering/*.md |
| **Agent Definitions** | Sub-agents (implicit) | [agents] config.toml | `.gemini/agents/*.md` | Rules type "Agent" | `.github/agents/*.agent.md` | N/A | `--agent-file` | `.kiro/agents/` (JSON) |
| **Agent Personas** | System prompt em CLAUDE.md | AGENTS.md | System prompt em agent.md | Rule content | Agent markdown content | N/A | System prompt templates | Agent JSON |
| **Task Definitions** | Instructions em rules | AGENTS.md sections | Agent frontmatter (tools) | Rule per-task | Tools property | N/A | N/A | N/A |
| **Hooks/Gates** | hooks (5 events) | hooks.json (5 events) | policy.toml | Automations | N/A | N/A | N/A | N/A |
| **Permission Control** | deny/allow rules | 3 approval modes | Tool wildcards | N/A | tools property | system rules | Sandbox | Trust boundary |
| **MCP Integration** | settings.json | config.toml | settings.json + agents | Built-in | Agent mcp-servers | Built-in | ACP | mcp.json |
| **Directory Scoping** | rules/ com paths | Override por diretorio | Nested GEMINI.md | `.cursor/rules/` por dir | `.github/instructions/` com globs | `.windsurf/rules/` por dir | N/A | N/A |

### Estrategia de Transpilacao

**Nivel 1: Universal (AGENTS.md)**
- Instrucoes do projeto (Constitution resumida)
- Convencoes de codigo
- Comandos de build/test
- Seguranca basica
- **Funciona em:** TODAS as ferramentas (24+)

**Nivel 2: Tool-Specific Instructions**
- CLAUDE.md com rules detalhadas e hierarquia
- GEMINI.md com imports e modularizacao
- `.cursor/rules/*.mdc` com frontmatter adequado
- `.github/copilot-instructions.md` + path-specific instructions
- `.windsurf/rules/*.md` (comprimido para 12K limit)
- **Funciona em:** Tool especifica com features avancadas

**Nivel 3: Agent Definitions**
- `.gemini/agents/*.md` com YAML frontmatter completo
- `.github/agents/*.agent.md` com tools e MCP
- Codex `[agents]` em config.toml
- **Funciona em:** Tools que suportam agents nativos

**Nivel 4: Hooks & Gates**
- `.claude/hooks/` com scripts CJS
- `.codex/hooks.json` com matchers
- Gemini `policy.toml`
- **Funciona em:** Claude Code e Codex CLI (mais maduros)

### Como AIOX Faz (Referencia)

O AIOX (SynkraAI) resolve o problema multi-IDE com:
- Config centralizada em `aiox.config.yaml`
- Suporte declarado para Claude Code (full), Codex CLI, Gemini CLI, Cursor
- Squad definitions em formato proprio
- Instalador que detecta e configura para IDE detectada
- **Limitacao:** Compatibilidade real varia -- Claude Code tem "full support", outros sao parciais

**FINDING:** O mercado ainda NAO tem uma solucao universal madura. AGENTS.md e o mais proximo de standard, mas cobre apenas a camada de instrucoes, nao agents, hooks ou permissions.

**RECOMMENDATION:** O SINAPSE tem oportunidade de ser o primeiro framework a oferecer transpilacao COMPLETA (nao apenas instrucoes) para multiplas plataformas. O diferencial e gerar agents, hooks e permissions alem de apenas o AGENTS.md.

---

## 1.3 Sistema de Instalacao e Configuracao

### Pesquisa de Referencia: Melhores Instaladores

**create-t3-app:**
- Usa `@clack/prompts` para UI interativa bonita
- Selecao condicional: so instala o que voce escolheu
- Modular: cada tecnologia e um "installer" independente
- Multi-language support na documentacao

**Astro CLI (`create astro`):**
- Walk-through guiado passo a passo
- Mensagens amigaveis e encorajadoras
- Templates selecionaveis
- Git init automatico

**@clack/prompts:**
- 80% menor que alternativas
- React-like components para terminal
- Criado por membro do Astro core team
- Suporta: text input, selectable lists, confirmations, spinners, progress
- Usado por: Astro, SvelteKit, create-t3-app

**AIOX Installer:**
- Detecta instalacoes existentes
- Instala/atualiza framework
- Configura agents e workflows
- Config via `aiox.config.yaml`

### Design do Instalador SINAPSE

Baseado na pesquisa, o instalador ideal para o SINAPSE:

```
sinapse init
  |
  v
[1. WELCOME]
  "SINAPSE AI Framework - Setup"
  Versao, links uteis
  |
  v
[2. LANGUAGE]
  > Portugues (pt-BR)
    English (en-US)
    Espanol (es)
  |
  v
[3. LLM TARGET]
  > Claude Code (full support - recommended)
    Codex CLI (full support)
    Gemini CLI (full support)
    Cursor (rules only)
    GitHub Copilot (agents + instructions)
    Windsurf (rules only, 12K limit)
    Kimi Code (AGENTS.md only)
    Kiro/Amazon Q (steering + agents)
    Multiple (generate for all selected)
  |
  v
[4. PROJECT TYPE]
  > Greenfield (novo projeto)
    Brownfield (projeto existente)
  |
  v
[5. FRAMEWORK TEMPLATE] (se greenfield)
  > Next.js (App Router)
    React + Vite
    Vue + Nuxt
    Node.js API
    Python FastAPI
    Custom
  |
  v
[6. SQUAD SELECTION]
  Core Squads (included):
  [x] Development (developer, qa, architect...)
  
  Optional Squads:
  [ ] Research & Intelligence (7 agents)
  [ ] Brand System (8 agents)
  [ ] Content Intelligence (6 agents)
  [ ] Commercial Systems (5 agents)
  [ ] Growth & Analytics (6 agents)
  ...
  |
  v
[7. CONFIG GENERATION]
  Gerando AGENTS.md...          done
  Gerando CLAUDE.md...          done
  Gerando .claude/rules/...     done (12 files)
  Gerando .claude/hooks/...     done (8 files)
  Gerando .gemini/agents/...    done (7 files)
  Gerando .cursor/rules/...     done (15 files)
  |
  v
[8. SUMMARY]
  SINAPSE instalado com sucesso!
  
  LLM Targets: Claude Code, Gemini CLI
  Squads: 3 (Development, Research, Brand)
  Agents: 22
  Rules: 15
  
  Proximos passos:
  - Rode `sinapse doctor` para verificar setup
  - Rode `sinapse graph --stats` para ver dashboard
```

### Modo Expert vs Guided

| Modo | Quando | Comportamento |
|------|--------|---------------|
| **Guided** (default) | Primeiro uso, usuarios novos | Todas as perguntas, explicacoes, defaults sugeridos |
| **Expert** | `sinapse init --expert` | Opcoes avancadas, menos explicacoes, config YAML direto |
| **Quick** | `sinapse init --quick` | Aceita todos os defaults, zero perguntas |
| **From Config** | `sinapse init --config sinapse.preset.yaml` | Le preset e aplica sem perguntas |

### Presets

```yaml
# sinapse.preset.yaml
name: "Startup SaaS"
language: pt-BR
llm_targets: [claude-code, cursor]
project_type: greenfield
framework: nextjs-app-router
squads:
  - development
  - research
  - brand
  - content
  - growth
options:
  documentation_first: true
  safe_collaboration: true
  security_checks: true
```

---

# PART 2: Squad Creation Pipeline

## 2.1 Analise da Estrutura Atual de Squads

### Dimensoes de um Squad SINAPSE (Atual)

Baseado no `squad-research` como referencia:

| Dimensao | squad-research | Benchmark |
|----------|----------------|-----------|
| **Agents** | 7 | Ideal: 3-7 (pesquisa confirma) |
| **Tasks** | 72 | Alto -- 10-15 por agent |
| **Workflows** | 6 | Adequado |
| **Knowledge Bases** | 13 | Robusto |
| **Checklists** | 3 | Minimo adequado |
| **Templates** | 6 | Adequado |
| **Total Files** | 109 | Complexo mas completo |

### Pesquisa: Numero Otimo de Agents por Squad

**FINDING:** Pesquisa do Google e da industria converge em 3-7 agents por workflow/squad como sweet spot.

**Dados de suporte:**
- Google Research: "Centralized coordination melhorou performance em 80.9% vs single agent em tarefas paralelizaveis"
- Industria: "Para tarefas completaveis por um unico agent, multi-agent adiciona 30-70% latencia e 2-5x custo sem melhoria de qualidade"
- Producao: "Sistemas com mais de 5-7 agents em um unico workflow sao dificeis de debugar, testar e manter"
- Anti-pattern: "Um workflow de 10 passos com 10 agents e quase sempre PIOR que 3 agents especializados com boundaries claros"

**IMPLICATION:** O squad-research com 7 agents esta no limite superior do sweet spot. Squads maiores (como os 8 do brand-system) podem ter overhead de coordenacao excessivo.

**RECOMMENDATION:**
- Manter squads entre 4-8 agents (incluindo orchestrator)
- Cada agent deve representar uma "capability genuinamente distinta"
- Preferir agents com 8-12 tasks cada (nem muito poucos nem muitos)
- Se um agent tem menos de 5 tasks, considerar merge com outro agent

### Framework de Qualidade para Squads

| Dimensao | Min | Ideal | Max | Justificativa |
|----------|-----|-------|-----|---------------|
| Agents | 3 | 5-7 | 10 | Sweet spot de coordenacao |
| Tasks per Agent | 5 | 8-12 | 15 | Especializacao sem sobrecarga |
| Workflows | 2 | 4-6 | 8 | Cobertura de cenarios |
| Knowledge Bases | 3 | 6-10 | 15 | Fundamento sem excesso |
| Checklists | 1 | 2-4 | 6 | Quality gates |
| Templates | 2 | 4-6 | 10 | Outputs padronizados |
| Cross-squad Connections | 1 | 3-5 | 8 | Integracao sem acoplamento |

---

## 2.2 Best Practices para Criacao de Squads

### Persona Design: Framework de 4 Camadas

Baseado na pesquisa de Mindra e industria, cada agent persona deve ter:

**Camada 1 -- Identity (Quem)**
```yaml
identity:
  name: "Sage"
  role: "Deep Research Scholar"
  archetype: "Scholar"
  scope: "Pesquisa profunda, multi-fonte, literature review"
  NOT: "Nao faz analise competitiva (delegar para Hawk)"
```
- Nome memoravel e unico
- Role clara e boundada
- Definir explicitamente o que o agent NAO faz (previne role drift)
- Archetype que comunica o "flavor" do agente

**Camada 2 -- Behavioral Constraints (O que pode/nao pode)**
```yaml
constraints:
  hard_stops:
    - "Nunca inventar dados -- citar fontes sempre"
    - "Nunca ultrapassar o escopo definido no brief"
  uncertainty_handling: "Declarar nivel de confianca (alto/medio/baixo)"
  escalation: "Se bloqueado, escalar para Prism (orchestrator)"
```
- Imperativas, nao sugestoes
- Hard stops inegociaveis
- Protocolo de incerteza explicito
- Caminhos de escalacao definidos

**Camada 3 -- Communication Style (Como fala)**
```yaml
communication:
  tone: "Academico mas acessivel"
  positive_example: "A analise sugere forte correlacao entre X e Y (r=0.87, n=1200)"
  negative_example: "Eu acho que X esta relacionado com Y"
  vocabulary:
    prefer: ["indica", "sugere", "correlaciona", "evidencia"]
    avoid: ["acho", "parece", "talvez", "provavelmente"]
  format: "Structured with headings, citations, confidence levels"
```
- Exemplos positivos E negativos (nao apenas descricoes vagas)
- Vocabulario especifico
- Formato de output definido

**Camada 4 -- Contextual Knowledge (O que sabe)**
```yaml
knowledge:
  static: "knowledge-base/*.md carregado no prompt"
  dynamic: "Runtime context via MCP e file access"
  domain: "Porter's Five Forces, SWOT, PESTEL, etc."
  separation: "Static knowledge no prompt, dynamic via tools"
```
- Separacao clara entre conhecimento estatico (prompt) e dinamico (runtime)
- Domain expertise documentada
- Referencias a knowledge bases especificas

### Anti-Patterns de Persona Design

| Anti-Pattern | Problema | Solucao |
|-------------|----------|---------|
| **Persona vaga** | "Voce e um assistente util" | Definir role, scope, boundaries explicitamente |
| **Sycophancy** | Agent concorda com tudo | "Nao valide informacao incorreta para evitar conflito" |
| **Scope creep** | Agent faz de tudo | Declarar limites strict com redirect patterns |
| **Persona drift** | Constraints abandonados em conversas longas | Reinforcement mid-prompt de regras chave |
| **Confidence miscalibration** | Agent parece 100% certo de tudo | Epistemic humility: niveis de certeza calibrados |

### Task Design: Principios

Cada task deve seguir:

1. **Inputs claros:** O que o agent recebe
2. **Outputs claros:** O que o agent entrega
3. **Pre-conditions:** O que precisa ser verdade antes
4. **Post-conditions:** O que deve ser verdade depois
5. **Execution modes:** Interactive, YOLO, Pre-Flight
6. **Quality gates:** Como validar o output
7. **Error handling:** O que fazer se falhar
8. **Escalation path:** Para quem escalar

### Knowledge Base Design

**Efetiva:**
- Referencia a frameworks e metodologias reconhecidos (Porter, SWOT, JTBD)
- Exemplos concretos, nao apenas teoria
- Heuristicas acionaveis ("Quando X, faca Y")
- Atualizada regularmente

**Inefetiva:**
- Copy-paste de Wikipedia sem contextualizacao
- Teoria pura sem heuristicas praticas
- Informacao desatualizada
- Knowledge que o LLM ja sabe de treinamento (redundante)

### Workflow Design

**Principio:** Workflows sao compostos por tasks conectadas, nao por agents conectados.

```yaml
workflow:
  name: deep-research-cycle
  trigger: "Nova demanda de pesquisa profunda"
  steps:
    - task: create-research-brief
      agent: research-orqx  # orchestrator cria o brief
      output: brief.md
    - task: conduct-deep-research
      agent: deep-researcher  # specialist executa
      input: brief.md
      output: research-raw.md
    - task: synthesize-research-report
      agent: data-synthesizer  # outro specialist refina
      input: research-raw.md
      output: report.md
    - task: validate-research
      agent: research-orqx  # orchestrator valida
      input: report.md
      gate: research-quality-checklist
```

---

## 2.3 Marketplace e Distribuicao de Squads

### Estado do Mercado em 2026

**FINDING:** O mercado de AI agent skills/packages explodiu em 2026. Skills.sh (Vercel) ja tem 83,627 skills com 8M+ installs. O formato e analogo ao npm para AI agents.

**Plataformas existentes:**
| Plataforma | Tamanho | Modelo | Nota |
|------------|---------|--------|------|
| Skills.sh (Vercel) | 83,627 skills | `npx skills add` | Suporta 18 AI agents |
| ClawHub (OpenClaw) | 3,200 skills | CLI com semantic search | Versionamento com rollback |
| npm (generic) | Packages tradicionais | `npm install` | Squad (Brady Gaster) usa npm |
| AIOX Squads | SynkraAI ecosystem | Instalador proprio | Squad Creator Free/Pro |

### Design do SINAPSE Squad Distribution

**Formato de Package:**

```
@sinapse/squad-research/
  squad.yaml          # Manifesto (metadata, agents, workflows)
  agents/             # Agent definitions (Markdown)
  tasks/              # Task definitions (Markdown)
  workflows/          # Workflow definitions (YAML)
  knowledge-base/     # Knowledge bases (Markdown)
  checklists/         # Quality checklists (Markdown)
  templates/          # Output templates (Markdown)
  README.md           # Documentacao humana
  CHANGELOG.md        # Historico de versoes
  package.json        # npm metadata + sinapse engine field
```

**Package.json:**
```json
{
  "name": "@sinapse/squad-research",
  "version": "1.0.0",
  "description": "World-class research & intelligence squad",
  "sinapse": {
    "type": "squad",
    "engine": ">=5.0.0",
    "agents": 7,
    "tasks": 72,
    "compatibility": {
      "claude-code": "full",
      "codex-cli": "full",
      "gemini-cli": "full",
      "cursor": "rules-only",
      "copilot": "agents-only"
    }
  },
  "keywords": ["sinapse", "squad", "research", "intelligence"]
}
```

**Instalacao:**
```bash
sinapse squad add @sinapse/squad-research
# ou
npx @sinapse/squad-research
```

### Versionamento e Compatibilidade

| Campo | Formato | Exemplo |
|-------|---------|---------|
| Squad version | SemVer | 1.2.0 |
| Engine compatibility | SemVer range | >=5.0.0 |
| LLM compatibility | Explicit map | { claude-code: "full", cursor: "rules-only" } |
| Breaking changes | MAJOR bump | 2.0.0 quando agent definitions mudam |

### Quality Metrics para Squads

**Pre-publicacao (automatizado):**
```
sinapse squad validate @sinapse/squad-research
  [x] squad.yaml valido (schema check)
  [x] Todos os agents referenciados existem
  [x] Todas as tasks referenciadas existem
  [x] Workflows conectam tasks existentes
  [x] Knowledge bases referenciadas existem
  [x] Min agents: 3 (tem 7) PASS
  [x] Min tasks: 15 (tem 72) PASS
  [x] Min workflows: 2 (tem 6) PASS
  [x] Sem broken references
  [x] Sem circular dependencies
  [x] README.md existe
  [x] CHANGELOG.md existe
  Score: 12/12 PASS
```

**Post-publicacao (community):**
- Stars/downloads (popularidade)
- Issue response time (manutencao)
- User reviews (qualidade percebida)
- Compatibility reports (funciona em quais IDEs)

### Community Squad Contribution

```bash
# Criar novo squad a partir de template
sinapse squad create my-squad --template=blank

# Desenvolver localmente
sinapse squad dev my-squad

# Validar antes de publicar
sinapse squad validate my-squad

# Publicar no registry
sinapse squad publish my-squad
```

---

# PART 3: CLI UX & Branding

## 3.1 Best Practices de CLI UX

### Principios de Design de CLI (Consolidado)

Baseado na pesquisa de Vercel, GitHub CLI, Heroku, Evil Martians e industria:

**1. Performance e Design (Vercel Philosophy)**
- "Performance IS design" -- nenhuma animacao bonita compensa load time lento
- Otimistic UI: mostrar estado "building" antes do servidor confirmar
- Skeleton screens em vez de spinners tradicionais
- Informacao densa e visivel at-a-glance

**2. Command Naming (GitHub CLI Pattern)**
```
gh <noun> <verb> [flags]
gh pr create --title "feat: ..."
gh issue list --state open
gh repo clone owner/repo
```
- Pattern: `<tool> <noun> <verb>`
- Nouns: categorias (pr, issue, repo, run)
- Verbs: acoes (create, list, view, delete, clone)
- Consistente e previsivel -- depois de aprender o pattern, voce adivinha comandos novos

**3. Output Colorido (Heroku + Better CLI)**
- Usar cores com parcimonia -- 2-3 cores max
- Reservar vermelho APENAS para erros
- Reservar amarelo para warnings
- Verde para sucesso
- Dim/bold para hierarquia visual
- **SEMPRE** oferecer `--no-color` e respeitar `NO_COLOR` env var
- Nao usar cor como UNICO meio de informacao (acessibilidade)

**4. Error Messages (Industria)**
```
ERROR [E001] Configuration file not found

  Expected: .sinapse/squad.yaml
  Location: /home/user/project/
  
  Resolution:
  1. Run 'sinapse init' to create configuration
  2. Or specify path: sinapse --config path/to/squad.yaml
  
  Docs: https://sinapse.ai/docs/errors/E001
```
- Codigo de erro unico
- Titulo descritivo
- O que era esperado vs o que encontrou
- Steps de resolucao concretos
- Link para documentacao

**5. Progress Indicators (Evil Martians)**

3 patterns para manter usuario informado:
| Pattern | Quando | Exemplo |
|---------|--------|---------|
| **Spinner** | Duracao desconhecida | `Analisando codebase...` |
| **X of Y** | Itens discretos | `Processando agent 3/7...` |
| **Progress bar** | Percentual conhecido | `[████████░░] 80% Installing dependencies` |

**6. Help Text Design**
```
USAGE
  sinapse <command> [options]

COMMANDS
  init        Initialize SINAPSE in current project
  squad       Manage squads (add, remove, list, validate)
  agent       Interact with agents (@agent-name)
  graph       Visualize dependencies and stats
  doctor      Diagnose setup issues

FLAGS
  --help      Show help for any command
  --version   Show SINAPSE version
  --no-color  Disable colored output
  --verbose   Show detailed output

EXAMPLES
  sinapse init                    Start guided setup
  sinapse squad add research      Install research squad
  sinapse agent @developer        Activate developer agent
  sinapse graph --deps            Show dependency tree

LEARN MORE
  Documentation: https://sinapse.ai/docs
  Issues:        https://github.com/sinapse-ai/core/issues
```

### Ferramentas de Construcao de CLI

| Ferramenta | Proposito | Usado Por | Nota |
|-----------|-----------|-----------|------|
| **@clack/prompts** | Prompts interativos bonitos | Astro, SvelteKit, T3 | 80% menor que alternativas |
| **Ink (React)** | UI completa em terminal | Claude Code, Gatsby, Yarn 2 | React para CLI |
| **oclif** | Framework CLI robusto | Heroku, Salesforce | Enterprise-grade |
| **Commander.js** | Parsing de comandos | npm CLI, Vue CLI | Leve e popular |
| **chalk** | Cores no terminal | Quase tudo | Standard de-facto |
| **ora** | Spinners | Muitos CLIs | Simples e bonito |

**RECOMMENDATION para SINAPSE:**
- **@clack/prompts** para o instalador interativo (bonito, leve)
- **Ink** para dashboard e outputs complexos (flexbox no terminal)
- **Commander.js** para parsing de comandos
- **chalk** para cores
- **ora** para spinners simples

---

## 3.2 SINAPSE CLI Branding

### Naming Convention: A Metafora da Orquestra

O SINAPSE ja usa a metafora de orquestra/sinfonia. Baseado na pesquisa sobre naming (Apple, GitHub CLI), consolidar:

**Principio Apple:** Nomes faceis de lembrar, pronunciar e soletrar. "Um aluno da 5a serie deveria conseguir soletrar e um avo deveria conseguir dizer."

**Hierarquia de Naming:**

| Nivel | Nome | Pattern | Exemplo |
|-------|------|---------|---------|
| **Framework** | SINAPSE | Acronimo memoravel | -- |
| **CLI** | `sinapse` | Lowercase, sem prefixo | `sinapse init` |
| **Squad** | Nome proprio | Substantivo descritivo | squad-research, squad-brand |
| **Orchestrator** | Nome + `-orqx` | Pattern consistente | research-orqx (Prism) |
| **Agent** | Nome proprio (persona) | Substantivo evocativo | Sage, Pulse, Hawk, Scope |
| **Task** | Verb-noun kebab | Acao clara | `conduct-deep-research` |
| **Workflow** | Noun-cycle | Ciclo nomeado | `deep-research-cycle` |
| **Command** | `*verb` | Asterisco + verbo | `*orchestrate`, `*sprint` |

### Command Structure para SINAPSE CLI

```
sinapse <noun> <verb> [options]
```

**Top-level nouns:**

| Noun | Descricao | Exemplos |
|------|-----------|---------|
| `init` | Inicializacao (standalone) | `sinapse init` |
| `squad` | Gerenciar squads | `sinapse squad add`, `sinapse squad list`, `sinapse squad validate` |
| `agent` | Interagir com agents | `sinapse agent @developer`, `sinapse agent list` |
| `graph` | Visualizar dependencias | `sinapse graph --deps`, `sinapse graph --stats` |
| `doctor` | Diagnostico | `sinapse doctor` |
| `config` | Configuracao | `sinapse config set`, `sinapse config get` |
| `generate` | Gerar configs para IDE | `sinapse generate --target cursor` |
| `story` | Gerenciar stories | `sinapse story create`, `sinapse story list` |
| `run` | Executar workflow | `sinapse run deep-research-cycle` |

### Experiencia de Greeting/Onboarding

**Primeira execucao (`sinapse` sem args):**
```
  ┌──────────────────────────────────────────────┐
  │                                              │
  │   S I N A P S E                              │
  │   AI-Orchestrated Development Framework      │
  │                                              │
  │   v5.0.0                                     │
  │                                              │
  │   Quick Start:                               │
  │   sinapse init      Setup new project        │
  │   sinapse doctor    Check configuration      │
  │   sinapse --help    Show all commands         │
  │                                              │
  │   sinapse.ai/docs                            │
  │                                              │
  └──────────────────────────────────────────────┘
```

Nota: ASCII art minimalista. Sem emojis no CLI output padrao (acessibilidade). Opcional via `--fancy`.

**Agent activation greeting:**
```
sinapse agent @developer

  Pixel (Developer) activated
  Story: 2.1 — IDE Detection
  Branch: caio/feat/ide-detection
  Status: Ready
  
  Commands: *help, *develop, *task, *exit
```

Breve, informativo, sem excesso. Mostrar contexto atual (story, branch, status).

### Status Display / Dashboard

```
sinapse status

  Project: sinapse-ai
  Branch:  caio/feat/multi-llm
  
  Squads Active:
    development  7 agents  OK
    research     7 agents  OK
    brand        8 agents  OK
  
  Stories:
    2.1  IDE Detection       Done
    2.2  Multi-LLM Support   InProgress  (3/8 tasks)
    2.3  Squad Marketplace   Ready
  
  Health:
    Config     OK
    MCP        OK (3 servers)
    Hooks      OK (8 active)
    CodeIntel  Fallback (provider unavailable)
```

### Como Transmitir Premium no CLI

Baseado na pesquisa de branding:

1. **Velocidade** -- Respostas instantaneas, zero lag desnecessario
2. **Densidade de informacao** -- Cada linha conta, zero filler
3. **Consistencia** -- Mesmo formato em TODOS os outputs
4. **Silencio inteligente** -- Sucesso e silencioso, apenas erros falam
5. **Confianca** -- Mensagens assertivas, sem "trying to...", sem "attempting..."
6. **Completude** -- Sempre mostrar proximo passo, nunca deixar o usuario perdido

**Evitar:**
- Mensagens excessivamente entusiastas ("Amazing! Your squad is ready!")
- ASCII art elaborado em operacoes rotineiras
- Emojis como substituto de informacao
- Spinners em operacoes que levam < 200ms
- Mensagens vagas ("Something went wrong")

---

# Recommendations Consolidadas

## R1: Estrategia Multi-LLM (Prioridade ALTA)

| Prioridade | Target | Nivel de Suporte | Justificativa |
|-----------|--------|-------------------|---------------|
| P0 | Claude Code | Full (100%) | Plataforma nativa, features completas |
| P1 | AGENTS.md (universal) | Base (70%) | 24+ ferramentas, standard da industria |
| P2 | Codex CLI | Full (95%) | Arquitetura mais similar, open source, hooks |
| P3 | Gemini CLI | Full (90%) | Agent system nativo, free tier, 1M context |
| P4 | Cursor | Rules only (60%) | Popular mas sem agents/hooks nativos |
| P5 | GitHub Copilot | Agents + Instructions (70%) | Custom agents desde Marco 2026 |
| P6 | Windsurf | Compressed rules (40%) | Budget limit de 12K caracteres |
| P7 | Kimi Code | AGENTS.md only (30%) | Emergente, auto-load pendente |
| P8 | Kiro | Steering + agents (40%) | Formato JSON proprio, nicho AWS |

**Acao imediata:** Implementar transpiler engine que gera output para P0-P3. P4-P8 em fases futuras.

## R2: AGENTS.md como Base Universal

Gerar `AGENTS.md` como output primario universal. Contem:
- Project overview e convencoes
- Constitution resumida (top 5 rules)
- Build/test commands
- Agent capabilities summary
- Security guidelines

Este arquivo funciona em TODAS as ferramentas. Files tool-specific adicionam features avancadas.

## R3: Squad Size Guidelines

- **4-8 agents** por squad (incluindo orchestrator)
- **8-12 tasks** por agent
- **4-6 workflows** por squad
- **Regra de ouro:** Se um agent tem < 5 tasks, merge. Se tem > 15, split.

## R4: Persona Design Mandatorio

Todo agent deve ter as 4 camadas:
1. Identity (quem + o que NAO faz)
2. Behavioral Constraints (hard stops + escalation)
3. Communication Style (exemplos positivos e negativos)
4. Contextual Knowledge (static vs dynamic)

## R5: CLI Stack Recomendado

- **@clack/prompts** para instalador interativo
- **Ink (React)** para dashboards
- **Commander.js** para parsing
- **chalk** para cores
- **sinapse <noun> <verb>** como naming pattern (estilo GitHub CLI)

## R6: Instalador em 3 Modos

- **Guided** (default): Walk-through completo
- **Expert**: Opcoes avancadas, menos hand-holding
- **Quick**: Zero perguntas, defaults sensatos

## R7: Squad Distribution via npm

- Package format com squad.yaml + agents/ + tasks/ + workflows/
- Publicacao via `sinapse squad publish`
- Instalacao via `sinapse squad add @scope/squad-name`
- Validacao automatica pre-publish

## R8: Hooks Cross-Platform

| Plataforma | Hooks Support | Estrategia |
|-----------|---------------|------------|
| Claude Code | Full (5 events) | Nativo |
| Codex CLI | Full (5 events) | Transpile hooks/ -> hooks.json (Windows: fallback) |
| Gemini CLI | policy.toml | Transpile permissions -> policy rules |
| Cursor | Automations | Best-effort mapping |
| Outros | Nenhum | Documentar como manual check |

---

# Sources

## Part 1: Multi-LLM Compatibility

### Claude Code
- [Claude Code Documentation](https://docs.anthropic.com/en/docs/claude-code)

### OpenAI Codex CLI
- [Codex CLI Features](https://developers.openai.com/codex/cli/features)
- [Codex CLI AGENTS.md Guide](https://developers.openai.com/codex/guides/agents-md)
- [Codex CLI Hooks](https://developers.openai.com/codex/hooks)
- [Codex CLI Advanced Configuration](https://developers.openai.com/codex/config-advanced)
- [Codex GitHub Repository](https://github.com/openai/codex)
- [Codex Agent Skills](https://developers.openai.com/codex/skills)
- [Unrolling the Codex Agent Loop](https://openai.com/index/unrolling-the-codex-agent-loop/)

### Google Gemini CLI
- [Gemini CLI GitHub Repository](https://github.com/google-gemini/gemini-cli)
- [GEMINI.md Context Files](https://geminicli.com/docs/cli/gemini-md/)
- [Gemini CLI Subagents](https://geminicli.com/docs/core/subagents/)
- [Gemini CLI Configuration](https://geminicli.com/docs/reference/configuration/)
- [Gemini CLI Subagents Comparison](https://www.morphllm.com/gemini-cli-subagents)
- [Google Blog: Introducing Gemini CLI](https://blog.google/innovation-and-ai/technology/developers-tools/introducing-gemini-cli-open-source-ai-agent/)

### Cursor
- [Cursor Rules Documentation](https://cursor.com/docs/context/rules)
- [Cursor Rules Guide 2026](https://www.claudemdeditor.com/cursor-rules-guide)
- [Cursor MCP Setup Guide](https://dev.to/serenitiesai/how-to-set-up-mcp-servers-in-cursor-ide-complete-guide-2026-5gdl)
- [Cursor Beta Features 2026](https://markaicode.com/cursor-beta-features-2026/)
- [awesome-cursor-rules-mdc Reference](https://github.com/sanjeed5/awesome-cursor-rules-mdc/blob/main/cursor-rules-reference.md)
- [Deep Dive into Cursor Rules](https://forum.cursor.com/t/a-deep-dive-into-cursor-rules-0-45/60721)

### GitHub Copilot
- [Custom Instructions for Copilot](https://docs.github.com/copilot/customizing-copilot/adding-custom-instructions-for-github-copilot)
- [Custom Agents Configuration](https://docs.github.com/en/copilot/reference/custom-agents-configuration)
- [Copilot Custom Instructions in VS Code](https://code.visualstudio.com/docs/copilot/customization/custom-instructions)
- [Copilot in Visual Studio March Update](https://github.blog/changelog/2026-04-02-github-copilot-in-visual-studio-march-update/)
- [Copilot Customization Handbook](https://copilot-academy.github.io/workshops/copilot-customization/copilot_customization_handbook)
- [Copilot Spaces](https://medium.com/arconsis/part-4-github-copilot-spaces-giving-copilot-real-understanding-of-your-project-a5dca2d5d97d)

### Windsurf
- [Windsurf Documentation](https://docs.windsurf.com/windsurf/getting-started)
- [Windsurf Cascade Memories](https://docs.windsurf.com/windsurf/cascade/memories)
- [Windsurf Rules Guide](https://design.dev/guides/windsurf-rules/)
- [Windsurf Context Management](https://iceberglakehouse.com/posts/2026-03-context-windsurf/)
- [Windsurf Rules Directory](https://windsurf.com/editor/directory)

### Kimi Code CLI
- [Kimi Code CLI GitHub](https://github.com/MoonshotAI/kimi-cli)
- [Kimi Code Website](https://www.kimi.com/code/en)
- [Kimi CLI Agents and Subagents](https://moonshotai.github.io/kimi-cli/en/customization/agents.html)
- [Kimi K2.5 Developer Guide](https://www.nxcode.io/resources/news/kimi-k2-5-developer-guide-kimi-code-cli-2026)

### Kiro CLI (ex-Amazon Q)
- [Kiro CLI](https://kiro.dev/cli/)
- [Migrating from Amazon Q Developer CLI](https://kiro.dev/docs/cli/migrating-from-q/)
- [Amazon Q Developer CLI GitHub](https://github.com/aws/amazon-q-developer-cli)

### AGENTS.md Standard
- [AGENTS.md Official Site](https://agents.md/)
- [AGENTS.md GitHub Repository](https://github.com/agentsmd/agents.md)
- [AGENTS.md InfoQ Coverage](https://www.infoq.com/news/2025/08/agents-md/)
- [AGENTS.md Deep Dive](https://prpm.dev/blog/agents-md-deep-dive)

### Multi-IDE / AIOX
- [AIOX Core GitHub](https://github.com/SynkraAI/aiox-core)
- [AIOX User Guide](https://github.com/SynkraAI/aiox-core/blob/main/docs/guides/user-guide.md)
- [AIOX Squads Guide](https://github.com/SynkraAI/aiox-core/blob/main/docs/guides/squads-guide.md)
- [Codex vs Claude Code 2026 Architecture Deep Dive](https://blakecrosley.com/blog/codex-vs-claude-code-2026)
- [Claude Code vs Codex vs Gemini 2026](https://www.educative.io/blog/claude-code-vs-codex-vs-gemini-code-assist)

## Part 2: Squad Creation Pipeline

### Multi-Agent Systems
- [Google Research: Scaling Agent Systems](https://research.google/blog/towards-a-science-of-scaling-agent-systems-when-and-why-agent-systems-work/)
- [Multi-Agent Systems Guide 2026](https://whatisagentic.ai/learn/multi-agent-systems/)
- [How to Build Multi-Agent Systems 2026](https://dev.to/eira-wexford/how-to-build-multi-agent-systems-complete-2026-guide-1io6)
- [Multi-Agent AI Orchestration](https://neomanex.com/posts/multi-agent-ai-systems-orchestration)
- [Best Practices for AI Agent Implementations](https://onereach.ai/blog/best-practices-for-ai-agent-implementations/)

### Persona Design
- [Designing AI Agent Personas (Mindra)](https://mindra.co/blog/designing-ai-agent-personas-system-prompts-enterprise)
- [How to Define an AI Agent Persona](https://thenewstack.io/how-to-define-an-ai-agent-persona-by-tweaking-llm-prompts/)
- [System Prompts for AI Agents Guide](https://converter.brightcoding.dev/blog/system-prompts-for-ai-agents-the-complete-2026-guide-to-building-powerful-safe-autonomous-systems)

### Squad Distribution
- [Agent Skills as npm 2026](https://www.buildmvpfast.com/blog/agent-skills-npm-ai-package-manager-2026)
- [Squad Framework (Brady Gaster)](https://github.com/bradygaster/squad)
- [Building AI Agent Squad for Repo](https://marcusfelling.com/blog/2026/building-an-ai-agent-squad-for-your-repo)

## Part 3: CLI UX & Branding

### CLI Design
- [Vercel Developer Experience as Design](https://blakecrosley.com/guides/design/vercel)
- [CLI UX Best Practices: Progress Displays (Evil Martians)](https://evilmartians.com/chronicles/cli-ux-best-practices-3-patterns-for-improving-progress-displays)
- [Heroku CLI Style Guide](https://devcenter.heroku.com/articles/cli-style-guide)
- [Colors and Formatting in CLI (Better CLI)](https://bettercli.org/design/using-colors-in-cli/)
- [Mastering CLI Design Best Practices](https://jsschools.com/programming/mastering-cli-design-best-practices-for-powerful-/)
- [Elevate Developer Experiences with CLI Design (Thoughtworks)](https://www.thoughtworks.com/en-us/insights/blog/engineering-effectiveness/elevate-developer-experiences-cli-design-guidelines)

### Terminal UI
- [Ink: React for CLI](https://github.com/vadimdemedes/ink)
- [Claude Code Terminal UI Internals](https://kotrotsos.medium.com/claude-code-internals-part-11-terminal-ui-542fe17db016)
- [Clack Prompts](https://www.clack.cc/)
- [Building CLI with Clack and oclif](https://codecryrepeat.hashnode.dev/learn-how-to-create-a-beautiful-cli-application-with-the-oclif-and-clackprompts)

### Branding & Naming
- [Apple's Naming Conventions (Brand Chemistry)](https://brandchemistry.co/p/apple-naming-conventions)
- [GitHub CLI Manual](https://cli.github.com/manual/)

### Installers
- [create-t3-app DeepWiki](https://deepwiki.com/t3-oss/create-t3-app)
- [Astro Install and Setup](https://docs.astro.build/en/install-and-setup/)

---

> Pesquisa conduzida por Prism (Research Operations Conductor), squad-research.
> Nivel: DEFINITIVE (40+ fontes, todos os tiers de credibilidade).
> Data: 2026-04-04.
