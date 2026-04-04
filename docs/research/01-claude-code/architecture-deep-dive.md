# Claude Code: Arquitetura Interna -- Pesquisa DEFINITIVE

> **Nivel:** DEFINITIVE (Research Depth Pyramid L4)
> **Data:** 2026-04-04
> **Pesquisador:** Prism (Research Orchestrator)
> **Fontes:** 30+ queries, 15+ paginas analisadas, Tier 1-4
> **Trigger:** Leak do source code via npm sourcemap em 31/03/2026

---

## INDICE

1. [O Leak: Contexto e Cronologia](#1-o-leak-contexto-e-cronologia)
2. [Arquitetura Geral e Boot Sequence](#2-arquitetura-geral-e-boot-sequence)
3. [Sistema de Tools](#3-sistema-de-tools)
4. [Query Engine e Agentic Loop](#4-query-engine-e-agentic-loop)
5. [Sistema de Agentes e Subagents](#5-sistema-de-agentes-e-subagents)
6. [Agent Teams e Coordinator Mode](#6-agent-teams-e-coordinator-mode)
7. [Sistema de Hooks](#7-sistema-de-hooks)
8. [CLAUDE.md e System Prompt Construction](#8-claudemd-e-system-prompt-construction)
9. [Memory e AutoDream](#9-memory-e-autodream)
10. [Context Management e Compaction](#10-context-management-e-compaction)
11. [Skills e Plugins](#11-skills-e-plugins)
12. [MCP (Model Context Protocol)](#12-mcp-model-context-protocol)
13. [Settings e Permissions](#13-settings-e-permissions)
14. [Sandboxing](#14-sandboxing)
15. [Worktree Isolation](#15-worktree-isolation)
16. [Prompt Caching e Otimizacao](#16-prompt-caching-e-otimizacao)
17. [Deferred Tools e Tool Search](#17-deferred-tools-e-tool-search)
18. [IDE Integrations e Bridge Mode](#18-ide-integrations-e-bridge-mode)
19. [Features Ocultas: KAIROS, Undercover, Anti-Distillation](#19-features-ocultas-kairos-undercover-anti-distillation)
20. [Feature Flags e Codenames Internos](#20-feature-flags-e-codenames-internos)
21. [Token Counting e Cost Tracking](#21-token-counting-e-cost-tracking)
22. [Seguranca: Bash Security e Permissoes](#22-seguranca-bash-security-e-permissoes)
23. [File Formats e Token Efficiency](#23-file-formats-e-token-efficiency)
24. [CLI vs Desktop vs Web](#24-cli-vs-desktop-vs-web)
25. [Implicacoes para SINAPSE](#25-implicacoes-para-sinapse)

---

## 1. O Leak: Contexto e Cronologia

### O que aconteceu

Em **31/03/2026**, um arquivo de source map de 59.8 MB (`.map`) foi acidentalmente incluido na versao **2.1.88** do pacote npm `@anthropic-ai/claude-code`. O arquivo continha **512.000+ linhas de TypeScript nao-ofuscado** em **1.906 arquivos**.

### Quem descobriu

O pesquisador de seguranca **Chaofan Shou** (`@Fried_rice`), estagiario na Solayer Labs, publicou a descoberta no X as 4:23 AM ET. O post acumulou 28.8 milhoes de visualizacoes.

### Causa raiz

Alguem esqueceu de adicionar `*.map` ao `.npmignore`. O bundler Bun gera source maps por default, e a build de producao nao desativou explicitamente.

### Resposta da Anthropic

- Confirmou como "release packaging issue caused by human error, not a security breach"
- Submeteu DMCA takedown requests, removendo 8.000+ repos do GitHub
- O source permanece acessivel em mirrors e rewrites (claw-code atingiu 100K stars -- recorde do GitHub)

### Versoes comprometidas com malware

Entre 00:21 e 03:29 UTC de 31/03, threat actors injetaram RAT (Remote Access Trojan) via versoes maliciosas do axios (1.14.1 e 0.30.4) em repos fakes de Claude Code.

**FINDING:** O leak mais significativo da historia de AI coding tools, expondo 44 feature flags, codenames internos, e features nao-lancadas.

**IMPLICATION:** Competidores tiveram acesso ao roadmap completo da Anthropic. A arquitetura revelada e de sofisticacao excepcional.

**RECOMMENDATION:** Estudar a arquitetura para informar decisoes no SINAPSE, especialmente os patterns de agent orchestration, caching, e tool system.

---

## 2. Arquitetura Geral e Boot Sequence

### Estrutura de Diretories do Source

```
src-rust/crates/
  cli/          # Entry point principal (main.tsx, 785KB)
  tools/        # 40+ tool implementations
  core/         # System prompts, constants, permissions
  api/          # API integration, beta headers
  coordinator/  # Multi-agent orchestration
  services/autoDream/  # Memory consolidation engine
  bridge/       # claude.ai integration
  migrations/   # Model version transitions
```

### 5 Camadas Arquiteturais

| Camada | Funcao | Componentes |
|--------|--------|-------------|
| **Agent Loop** | Task decomposition, tool selection | Tool calls, file reads, retries |
| **Context & Memory** | Persistencia de conhecimento | CLAUDE.md, auto-memory, history |
| **Execution Surface** | Acao no sistema | Read, Edit, Bash, subagents, worktrees |
| **Governance & Safety** | Limites de permissao | Permission modes, hooks, sandboxing |
| **Extensibility** | Novas capacidades | Skills, plugins, MCP, Agent SDK |

### Boot Sequence (3 camadas aninhadas)

**Fase 1 -- CLI Entrypoint (`cli.tsx`)**
- Fast-path para `--version`, `--daemon-worker` sem carregar modulos pesados
- Imports dinamicos estrategicos (zero-module fast paths)
- Mutacoes de environment antes da avaliacao de modulos

**Fase 2 -- Parallel Prefetch (`main.tsx` top-level)**
Tres operacoes concorrentes durante ~135ms da chain de imports:
- `startMdmRawRead()` -- MDM policy queries (20-40ms via plutil/reg query)
- `startKeychainPrefetch()` -- OAuth e API keys (~65ms no macOS)
- Resultados cacheados antes de serem necessarios

**Fase 3 -- Commander Parsing**
1. `eagerLoadSettings()` -- flags `--settings` antes do Commander
2. `Commander.parse()` -- resolve cwd, permission mode, model, session
3. `init()` -- aplica env vars, carrega MDM settings
4. `runMigrations()` -- schema atualmente na versao 11

**Fase 4 -- Setup Orchestration (`setup.ts`)**
Ordem critica:
1. Check Node.js >= 18
2. Session ID customizado (opcional)
3. UDS messaging server (habilita hook injection)
4. Teammate/swarm snapshot
5. **`setCwd(cwd)`** -- DEVE preceder hooks
6. **`captureHooksConfigSnapshot()`** -- le `.claude/settings.json` do cwd
7. File watcher initialization
8. Worktree creation + tmux session (opcional)
9. Background jobs: session memory, command prefetch, plugin hooks
10. `initSinks()` -- analytics
11. `logEvent('tengu_started')` -- beacon de confiabilidade
12. API key prefetch
13. Permission safety gates (root/sudo, sandbox)

**Fase 5 -- Global State (`bootstrap/state.ts`)**
Estado centralizado por sessao:
- Identity: `sessionId`, `originalCwd`, `projectRoot`, `cwd`
- Usage: `totalCostUSD`, `modelUsage`, token counters
- Flags: `isInteractive`, `sessionBypassPermissionsMode`
- **"Sticky latches"**: `afkModeHeaderLatched`, `fastModeHeaderLatched` previnem cache busts ao trocar settings mid-session

**Fase 6 -- Ink Render (`replLauncher.tsx`)**
- TUI baseada em React/Ink
- `ThemeProvider` wrapper automatico
- `startDeferredPrefetches()` apos primeiro render (user init, MCP, model capabilities)

### Bare Mode (`--bare` / `CLAUDE_CODE_SIMPLE`)
Para uso scriptado/SDK, elimina: UDS server, teammate snapshot, session memory, plugin hooks, attribution, deferred prefetches. Otimizado para latencia em CI.

---

## 3. Sistema de Tools

### Interface Core: `Tool<Input, Output, P>`

Tres generics: Zod input schema, output type, progress event shape.

Membros obrigatorios:
- `name` -- identificador primario
- `aliases` -- nomes legados
- `inputSchema` -- schema Zod (source of truth)
- `maxResultSizeChars` -- trigger para persistencia em disco
- `call()` -- async, recebe args, context, permission function, parent message
- `checkPermissions()` -- validacao antes da execucao
- `isConcurrencySafe(input)` -- pode rodar em paralelo?
- `isReadOnly(input)` -- read-only?
- `isDestructive?()` -- flag opcional

### `buildTool()` Factory

Defaults fail-closed:
```
isConcurrencySafe: () => false    // assume state mutation
isReadOnly: () => false
isDestructive: () => false
checkPermissions: () => Promise.resolve({ behavior: 'allow' })
```

### Pipeline de Registro (3 tiers)

1. **`getAllBaseTools()`** -- catalogo exaustivo com feature flags e gating por env var (dead-code elimination do Bun)
2. **`getTools()`** -- filtro por modo (simple, REPL, deny rules, `isEnabled()`)
3. **`assembleToolPool()`** -- built-ins alfabeticos como prefixo, MCP tools alfabeticos depois. Separacao preserva cache breakpoints.

### Catalogo de Tools (60+, ~20 habilitadas por default)

| Categoria | Tools |
|-----------|-------|
| **Execucao** | BashTool, PowerShellTool, REPLTool, AgentTool |
| **Arquivo** | FileReadTool, FileEditTool, FileWriteTool, GlobTool, GrepTool |
| **Web** | WebFetchTool, WebSearchTool, WebBrowserTool |
| **Dev** | LSPTool, NotebookEditTool, MCPTool, EnterWorktreeTool |
| **Controle** | TaskCreateTool, ScheduleCronTool, TeamCreateTool, RemoteTriggerTool |
| **KAIROS-only** | SendUserFile, PushNotification, SubscribePR |
| **Internal-only** | ConfigTool, TungstenTool, SuggestBackgroundPRTool |
| **Navegacao** | EnterPlanModeTool, ExitPlanModeV2Tool, BriefTool |
| **Agente** | AgentTool, SendMessageTool, TaskStopTool, TaskOutputTool |
| **MCP** | ListMcpResourcesTool, ReadMcpResourceTool, MonitorTool, McpAuthTool |
| **Interacao** | AskUserQuestionTool, SkillTool, TodoWriteTool |

### Pipeline de Execucao (`toolExecution.ts`)

`runToolUse()` orquestra:
1. **Zod validation** -- `inputSchema.safeParse(input)`
2. **Semantic validation** -- path traversal, size limits
3. **Speculative classifier** -- Bash commands iniciam security check antes dos hooks
4. **`backfillObservableInput`** -- clone raso para hooks e canUseTool
5. **PreToolUse hooks** -- async generators yielding progress/permission
6. **`canUseTool()`** -- gate principal de permissao
7. **`tool.call()`** -- execucao real
8. **PostToolUse hooks** -- apos completar
9. **Result serialization** -- processamento de budget de tamanho

**3 copias de input mantidas:**
1. API-bound original -- para cache/serializacao
2. Backfilled observable clone -- para hooks e canUseTool
3. Hook-updated call input -- potencialmente modificado para execucao

### Orquestracao de Concorrencia

`partitionToolCalls()` agrupa tools consecutivas safe:
- Tools com `isConcurrencySafe(input) === true` rodam em paralelo
- Non-safe tools quebram batches e rodam serialmente
- Teto: `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` (default 10)
- Modifiers enfileirados, aplicados apos batch completar

### Bash Tool: Tratamento Especial

- **Unica tool cujos erros abortam siblings** (dependency chains implicitas)
- Inicia security classifier especulativamente antes dos hooks
- Campo `_simulatedSedEdit` stripped antes da execucao
- 23 security checks numerados em `bashSecurity.ts`
- 18 builtins Zsh bloqueados
- Defesa contra Zsh equals expansion (`=curl`)
- Prevencao de Unicode zero-width space injection
- Defesa contra IFS null-byte injection

---

## 4. Query Engine e Agentic Loop

### Estrutura do Query Engine

`QueryEngine.ts` -- **46.000 linhas**, arquivo unico que concentra toda interacao com LLM API.

4 camadas:
1. `QueryEngine.submitMessage()` -- valida e constroi system prompt
2. `query()`/`queryLoop()` -- `while(true)` agentic loop
3. `queryModel`/`callModel` -- wrapper da API Anthropic com streaming
4. Stop hooks + token budget logic

### Fluxo do Agentic Loop

```
User input
  -> Claude responde com texto, tool calls, ou ambos
  -> Tools executam com validacao de permissao
  -> Resultados alimentam Claude
  -> Claude decide: mais tools ou resposta final
  -> Loop continua ate texto final sem tool calls
```

### Streaming

- Server-Sent Events reconstruidos em `AssistantMessage`
- Usage mutada in-place na ultima mensagem apos `message_delta`
- **Tombstone messages**: se fallback mid-stream, mensagens parciais marcadas como tombstone

### Retry Logic (`withRetry()`)

- Ate 10 retries com exponential backoff + jitter
- **529 (Overloaded):** Apenas foreground retenta; background bail imediatamente
- **Opus Fallback:** Apos 3 consecutive 529s em Opus, dispara `FallbackTriggeredError`
- **OAuth 401:** Forca token refresh
- **Context Overflow 400:** Computa novo `maxTokensOverride`
- **Persistent Mode:** Retry indefinido com cap de 30min e heartbeat messages
- **Socket Errors:** Chama `disableKeepAlive()` para conexoes stale

### 7 Razoes de Continuacao

1. `max_output_tokens_escalate`
2. `max_output_tokens_recovery`
3. `reactive_compact_retry`
4. `collapse_drain_retry`
5. `stop_hook_blocking`
6. `token_budget_continuation`
7. Tool-use loop implicito

### Token Budget e Stop Hooks

- Nudge messages quando consumo < 90%
- Early stop apos 3+ continuacoes quando deltas caem abaixo de 500 tokens
- Stop hooks (shell scripts) podem bloquear continuacao

---

## 5. Sistema de Agentes e Subagents

### Definicao: `AgentDefinition` (Discriminated Union)

3 tipos por `source` field:

**Built-In** (`source: 'built-in'`):
- System prompts dinamicos via `getSystemPrompt({toolUseContext})`
- Nao podem ser overridden por user files
- Tipos: general-purpose, Explore, Plan, verification, fork, statusline-setup

**Custom** (`source: userSettings | projectSettings | policySettings`):
- Carregados de `.claude/agents/*.md`
- System prompt no corpo markdown, config no frontmatter YAML

**Plugin** (`source: 'plugin'`):
- Via `--plugin-dir`
- Tratados como admin-trusted para MCP

### Built-In Agents

| Agent | Model | Tools | Modo | Notas |
|-------|-------|-------|------|-------|
| general-purpose | default | All (`['*']`) | sync/async | Delegacao padrao |
| Explore | haiku (ext) / inherit (ant) | Read-only | sync | `omitClaudeMd: true`, poupa ~5-15 Gtok/semana |
| Plan | inherit | Read-only | sync | `omitClaudeMd: true` |
| verification | inherit | No Edit/Write; permite /tmp scripts | async (forcado) | `background: true`; termina com VERDICT |
| fork | inherit | All + cache-identical tools | experimental | Paralelizacao com cache sharing |

### Execucao: Sync vs Async

6 condicoes forcam async:
1. `run_in_background === true`
2. Agent definition `background: true`
3. Coordinator mode ativo
4. Fork experiment (`FORK_SUBAGENT` gate)
5. KAIROS assistant mode
6. Proactive mode

**Sync path**: Build prompt -> worktree opcional -> `await runAgent(params)` -> cleanup
**Async path**: Registra no AppState -> retorna `async_launched` -> roda detached -> notificacao na idle do parent

### Fork Path (Experimental)

Inovacao chave: children usam texto placeholder identico para cada `tool_result`, garantindo prefixos byte-identical para prompt caching.

**Guards recursivos:** `querySource === 'agent:builtin:fork'` e scan por `<fork-boilerplate>` tag.

### Frontmatter de Custom Agents

```yaml
---
name: security-reviewer
description: Reviews code for security vulnerabilities
model: opus
tools: [Read, Grep, Glob, Bash]
disallowedTools: [Edit, Write]
permissionMode: plan
maxTurns: 20
background: false
isolation: worktree
memory: project
mcpServers: [memory-server]
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/security-check.sh"
skills: [security-review]
initialPrompt: "Review the staged changes for vulnerabilities"
effort: high
requiredMcpServers: [security-scanner]
---
You are a senior security engineer.
Review for injection flaws, auth issues, secret exposure.
```

---

## 6. Agent Teams e Coordinator Mode

### Agent Teams

Um session atua como team lead, coordenando teammates que trabalham independentemente com context windows proprios.

**Comunicacao:** Task files em disco + `SendMessageTool` (nao ha memoria compartilhada).

**SendMessageTool routing:**

| Target | Tipo | Comportamento |
|--------|------|---------------|
| teammate name | string | Escrito no mailbox; auto-resumes se parado |
| `"*"` | string | Broadcast para todos exceto sender |
| `"team-lead"` | shutdown_response | Approve/reject graceful shutdown |
| `"uds:<path>"` | string | Unix domain socket cross-session |
| `"bridge:<session-id>"` | string | Remote Control via Anthropic servers |

### Execucao

- **tmux/iTerm2 split panes**: cada teammate recebe seu proprio pane
- **In-process mode**: funciona em qualquer terminal
- `spawnMultiAgent.ts` propaga: agent-id, agent-name, team-name, agent-color, parent-session-id, CLI flags

### Coordinator Mode (`COORDINATOR_MODE` flag)

Implementado via system prompt, nao codigo:

4 fases:
1. **Research** -- workers investigam codebase em paralelo
2. **Synthesis** -- coordinator le findings, crafta specs
3. **Implementation** -- workers executam mudancas por spec
4. **Verification** -- testa mudancas

Instrucoes: "Parallelism is your superpower" e "Do not rubber-stamp weak work."

---

## 7. Sistema de Hooks

### 26 Hook Events

| Evento | Quando | Blockable |
|--------|--------|-----------|
| `SessionStart` | Sessao inicia/resume | Nao |
| `InstructionsLoaded` | CLAUDE.md/rules carregado | Nao |
| `UserPromptSubmit` | Usuario submete prompt | Sim |
| `PreToolUse` | Antes de tool executar | Sim |
| `PermissionRequest` | Dialog de permissao | Sim |
| `PermissionDenied` | Auto mode nega tool | Nao |
| `PostToolUse` | Apos tool succeeder | Nao |
| `PostToolUseFailure` | Apos tool falhar | Nao |
| `Notification` | CC envia notificacao | Nao |
| `SubagentStart` | Subagent spawned | Nao |
| `SubagentStop` | Subagent termina | Sim |
| `TaskCreated` | Task criada | Sim |
| `TaskCompleted` | Task completa | Sim |
| `Stop` | Claude termina resposta | Sim |
| `StopFailure` | Turn termina por erro API | Nao |
| `TeammateIdle` | Teammate fica idle | Sim |
| `ConfigChange` | Config muda | Sim |
| `CwdChanged` | Working dir muda | Nao |
| `FileChanged` | Arquivo monitorado muda | Nao |
| `WorktreeCreate` | Worktree criada | Sim |
| `WorktreeRemove` | Worktree removida | Nao |
| `PreCompact` | Antes de compaction | Nao |
| `PostCompact` | Apos compaction | Nao |
| `Elicitation` | MCP pede input | Sim |
| `ElicitationResult` | User responde MCP | Sim |
| `SessionEnd` | Sessao termina | Nao |

### 4 Tipos de Handler

**command** (95% do uso):
```json
{
  "type": "command",
  "command": "/path/to/script.sh",
  "timeout": 600,
  "statusMessage": "Running validation..."
}
```

**http**:
```json
{
  "type": "http",
  "url": "http://localhost:8080/hooks",
  "headers": { "Authorization": "Bearer $MY_TOKEN" },
  "timeout": 30
}
```

**prompt** (single-turn LLM):
```json
{
  "type": "prompt",
  "prompt": "Is this code secure? $ARGUMENTS",
  "model": "claude-opus",
  "timeout": 30
}
```

**agent** (multi-turn subagent com tool access):
```json
{
  "type": "agent",
  "prompt": "Verify this operation: $ARGUMENTS",
  "timeout": 60
}
```

**Precedencia:** agent > prompt > shell hooks

### Exit Codes

| Codigo | Significado | Efeito |
|--------|-------------|--------|
| 0 | Sucesso | Parse JSON do stdout |
| 2 | Block | Operacao negada, stderr mostrado |
| Outro | Non-blocking error | stderr em verbose mode |

### Decisoes PreToolUse

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow|deny|ask|defer",
    "permissionDecisionReason": "string",
    "updatedInput": { "field": "new_value" },
    "additionalContext": "string"
  }
}
```

### Matcher Syntax

```json
{ "matcher": "Bash" }              // exact match
{ "matcher": "Edit|Write" }        // either tool
{ "matcher": "mcp__memory__.*" }   // regex
{ "if": "Bash(git *)" }            // permission rule syntax
```

### Scopes de Config

```
Managed > Plugin > Skill/Agent > Local > Project > User
```

---

## 8. CLAUDE.md e System Prompt Construction

### 6 Camadas de Assemblagem

1. **Priority Resolver** (`systemPrompt.ts`) -- waterfall decision tree
2. **Content Factory** (`prompts.ts`) -- static + dynamic sections
3. **Section Registry** (`systemPromptSections.ts`) -- memoization control
4. **CLAUDE.md Loader** (`claudemd.ts`) -- multi-scope file discovery
5. **Memory System** (`memdir/memdir.ts`) -- auto-memory injection
6. **Cache Boundary** -- global vs session-scoped split

### Priority Waterfall

1. **Override mode** (loop) -- substitui tudo
2. **Coordinator mode** -- prompt de coordinator + appended
3. **Agent mode** -- instrucoes agent-specific
4. **Custom prompt** -- `--system-prompt` flag
5. **Default** -- full interactive session prompt

`appendSystemPrompt` e o UNICO que sempre appends (exceto override mode).

### Cache Boundary: `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__`

**Antes (static, globally-cacheable):**
- Intro (identity, URL guard, cyber-risk)
- System (markdown, permission modes, injection warnings)
- Doing Tasks (YAGNI, security, code-style)
- Actions (reversibility, blast-radius)
- Using Tools (hierarchy, parallelism)
- Tone & Style (no emojis, file:line refs)
- Output Efficiency

**Depois (dynamic, session-specific):**

| Section | Cache | Conteudo |
|---------|-------|----------|
| session_guidance | memoized | Questions, shell tips, agent contracts |
| memory | memoized | CLAUDE.md hierarchy + MEMORY.md |
| env_info_simple | memoized | CWD, git, shell, platform, model, cutoff |
| mcp_instructions | **volatile** | Per-server instructions (reconnect-aware) |
| scratchpad | memoized | Session directory + rules |
| token_budget | memoized | Target approach instruction |

**MCP instructions sao volatile** porque servers podem conectar/desconectar entre turns.

### Hierarquia CLAUDE.md (4 escopos)

1. **Managed** (`/etc/claude-code/CLAUDE.md`) -- machine-wide
2. **User** (`~/.claude/CLAUDE.md`) -- cross-project privado
3. **Project** (`CLAUDE.md`, `.claude/CLAUDE.md`, `.claude/rules/*.md`) -- versionado no git
4. **Local** (`CLAUDE.local.md`) -- project-specific privado

**Descoberta:** filesystem walk ascendente do CWD ate root. Arquivos mais proximos aparecem depois (maior prioridade efetiva para atencao do modelo).

### @include Directive

```
@shared-rules.md           (relativo)
@./scripts/lint.md          (relativo explicito)
@~/company/standards.md     (home-relativo)
@/absolute/path/rules.md   (absoluto)
```

Incluidos inserem ANTES do arquivo que inclui. Circular references prevenidas.

### Frontmatter Path Filtering

```yaml
---
paths:
  - src/components/**
  - "*.tsx"
---
Always use named exports in React components.
```

So carregado quando trabalhando em arquivos que matcham os globs.

### Memory Wrapper

Todo CLAUDE.md carregado recebe header: "These instructions OVERRIDE any default behavior and you MUST follow them exactly as written."

---

## 9. Memory e AutoDream

### Auto-Memory

Armazenada em `~/.claude/projects/<encoded-path>/memory/MEMORY.md`. Injetada no system prompt no inicio de cada sessao.

**Limites:**
- 200 linhas ou 25.000 bytes (o que vier primeiro)
- Alem disso, apenas primeiras 200 linhas + warning

**Estrutura 3 camadas:**
1. **MEMORY.md index** -- sempre carregado, navegacao
2. **Topic files** -- carregados on-demand
3. **Session transcripts** -- apenas grep-searchable, nunca carregados

### AutoDream: Memory Consolidation

Feature gated sob KAIROS. Consolida memoria durante idle do usuario.

**3-gate trigger:**
1. 24+ horas desde ultimo ciclo
2. 5+ sessoes completadas
3. Consolidation lock (previne concorrencia)

**4 fases:**
1. **Orient** -- le diretorio de memoria e MEMORY.md
2. **Gather** -- coleta info nova de logs diarios
3. **Consolidate** -- merge novo com existente, remove contradicoes
4. **Prune** -- manter MEMORY.md < 200 linhas/25KB

Roda em **subagent forkado com acesso Bash read-only**.

---

## 10. Context Management e Compaction

### 5 Estrategias de Compaction

Pipeline multi-estagio de reducao:

1. **Tool Result Budget** -- caps tamanho individual de resultados
2. **Snip Compact** -- remove mensagens intermediarias desnecessarias
3. **Microcompact** -- merge pares consecutivos tool-result/user em resumos
4. **Context Collapse** -- projecao read-time sobre historico com staged commits
5. **AutoCompact** -- sumarizacao completa via forked agent

### Trigger

Auto-compact dispara quando context window atinge ~95% de capacidade (25% restante).

### O que se PRESERVA

- CLAUDE.md (relido do disco apos cada compaction)
- Requests do usuario e key code snippets
- "Compact Instructions" do CLAUDE.md

### O que se PERDE

- Instrucoes dadas em conversacao (nao em CLAUDE.md)
- Error messages, line numbers, variable values, stack traces
- Reasoning por tras de decisoes (porque escolheu abordagem A vs B)
- Detalhes especificos de debugging

### Compact Instructions

Secao no CLAUDE.md que controla o que preservar:

```markdown
## Compact instructions
- Preserve code paths and unresolved security questions
- Preserve diff summaries and failed test output
```

### PreCompact/PostCompact Hooks

- `PreCompact` dispara ANTES da compaction (nao blockable)
- `PostCompact` dispara DEPOIS da compaction
- Permitem backup de contexto, notificacoes, etc.

### Bug documentado no source

Comentario de 10/03/2026: "1,279 sessions had 50+ consecutive failures...wasting ~250K API calls/day globally." Mitigacao: `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3`.

---

## 11. Skills e Plugins

### Skills (`SKILL.md`)

Formato:
```yaml
---
name: explain-code
description: Explains code with visual diagrams
disable-model-invocation: true    # Apenas usuario pode invocar
user-invocable: false             # Apenas Claude pode invocar
allowed-tools: [Read, Grep, Glob] # Restrict tools
---
Instructions for the skill...
$ARGUMENTS for dynamic values
$0, $1 for indexed arguments
```

**Invocacao:** `/skill-name` ou automaticamente quando relevante.

**Organizacao:** Cada skill e um diretorio com SKILL.md + arquivos de suporte. Arquivos extras nao carregam no contexto automaticamente.

### Plugins

Bundles de skills + agents + hooks + MCP + LSP.

```
.claude-plugin/
  plugin.json        # Manifest obrigatorio
skills/              # SKILL.md files
agents/              # Subagent definitions
commands/            # Legacy command files
hooks/               # hooks.json
.mcp.json            # MCP server configs
.lsp.json            # LSP server configs
```

**Manifest fields:** name, version, description, author, commands, agents, skills, hooks, mcpServers, outputStyles, lspServers.

**Restricao:** Plugin-shipped agents NAO suportam hooks, mcpServers, ou permissionMode no frontmatter. Para isso, mover para `.claude/agents/`.

**Namespacing:** Skills de plugins recebem prefixo `plugin-name:skill-name`.

---

## 12. MCP (Model Context Protocol)

### 3 Transport Types

| Transport | Uso | Recomendacao |
|-----------|-----|-------------|
| **stdio** | Processos locais | Acesso direto ao sistema |
| **HTTP** | Servidores remotos | Recomendado para cloud |
| **SSE** | Remote (legado) | Deprecated em favor de HTTP |

### Tool Discovery e Deferred Loading

- MCP tools sao **deferred por default**
- Apenas nomes de tools consomem contexto ate uso
- Tool Search descobre tools on-demand
- `ENABLE_TOOL_SEARCH=auto` para threshold-based loading (10% do context window)

### Config Example

```json
{
  "mcpServers": {
    "memory": {
      "command": "npx",
      "args": ["-y", "@anthropic-ai/mcp-memory"],
      "transport": "stdio"
    },
    "remote-api": {
      "url": "https://api.example.com/mcp",
      "transport": "http",
      "headers": { "Authorization": "Bearer ${MCP_TOKEN}" }
    }
  }
}
```

### MCP Instructions

`getMcpInstructions()` itera servers conectados, extraindo campos `instructions`:
```
# MCP Server Instructions

## [ServerName]
[instructions text]
```

Experimental: `mcpInstructionsDelta` entrega instrucoes como objetos persistentes (evita prompt-cache busts de servers late-connecting).

---

## 13. Settings e Permissions

### Hierarquia de Configuracao

```
Managed > CLI flags > Local > Project > User > Defaults
```

| Scope | Arquivo | Compartilhamento |
|-------|---------|-----------------|
| Managed | `/etc/claude-code/managed-settings.json` | Org-wide, admin |
| User | `~/.claude/settings.json` | Privado, cross-project |
| Project | `.claude/settings.json` | Git-shared, time |
| Local | `.claude/settings.local.json` | Privado, gitignored |

### Permission Rules

```json
{
  "permissions": {
    "allow": ["Bash(git diff *)", "Read(.)", "Edit(src/*)"],
    "ask": ["Bash(npm run *)", "WebFetch(*)"],
    "deny": ["Bash(rm *)", "Read(.env*)"]
  }
}
```

**Avaliacao:** deny > ask > allow (first match wins).

**Syntax:** `Tool` ou `Tool(specifier)` com glob patterns.

### 6 Permission Modes

| Modo | Sem Pedir | Uso |
|------|-----------|-----|
| `default` | Read only | Trabalho sensivel |
| `acceptEdits` | Read + edit | Iteracao com gate em commands |
| `plan` | Read, plan, explore | Design antes de modificar |
| `auto` | Tudo (com classifier) | Tasks longas, Team/Enterprise |
| `bypassPermissions` | Tudo, sem checks | Apenas VMs de teste |
| `dontAsk` | Apenas pre-approved | Policy-driven lockdown |

**Auto mode:** Requer Sonnet 4.6+, Team/Enterprise. Classifier separado revisa cada acao. Regras broad (`Bash(*)`) descartadas; narrow (`Bash(npm test)`) mantidas.

### Managed Enterprise Settings

Nao podem ser overridden. Opcoes exclusivas:
- `allowManagedHooksOnly`
- `allowManagedMcpServersOnly`
- `allowManagedPermissionRulesOnly`

`managed-settings.d/` para fragments (merge alfabetico, ultimo vence).

---

## 14. Sandboxing

### Primitivas OS-Level

| Plataforma | Tecnologia |
|-----------|-----------|
| macOS | Seatbelt |
| Linux/WSL2 | bubblewrap |
| Windows (nativo) | Planejado |

### Isolamento

- **Filesystem:** Read/write no CWD; bloqueia fora dele
- **Network:** Acesso via unix domain socket proxy, que enforcea domain allowlists
- **Child processes:** herdam restricoes do sandbox

### Config

```json
{
  "sandbox": {
    "enabled": true,
    "autoAllowBashIfSandboxed": true,
    "allowUnsandboxedCommands": false,
    "excludedCommands": ["docker"],
    "filesystem": {
      "allowWrite": ["/tmp/build"],
      "denyRead": ["~/.aws/credentials"]
    },
    "network": {
      "allowedDomains": ["github.com", "*.npmjs.org"],
      "allowLocalBinding": true
    }
  }
}
```

**Impacto:** Reduz permission prompts em 84% no uso interno.

**Limitacoes honestas:** Domain allowlists permitem exfiltracao potencial; domain fronting pode bypass; unix sockets podem dar acesso ao host.

---

## 15. Worktree Isolation

### Mecanismo

`isolation: worktree` no frontmatter de agent cria git worktree temporaria.

**Slug:** `agent-{earlyAgentId.slice(0,8)}`

### Cleanup

- Sem mudancas git-tracked: worktree + branch deletados automaticamente
- Com mudancas: branch retida para inspecao/merge
- Worktrees orfas por crash: limpas no startup apos `cleanupPeriodDays`

### Blast Radius Control

Cada agent tem codebase completo isolado. Agent A pode reescrever `src/auth.ts` enquanto Agent B reescreve o mesmo arquivo com abordagem diferente. Review e merge sao do usuario.

---

## 16. Prompt Caching e Otimizacao

### Principio

Prompt caching funciona por **prefix matching**. A API cacheia tudo do inicio ate cada `cache_control` breakpoint.

### Estrategia

- Conteudo estatico PRIMEIRO, dinamico ULTIMO
- `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__` separa cacheable de session-specific
- Tool definitions separadas alfabeticamente (built-in primeiro, MCP depois) para preservar cache breakpoints

### 14 Cache-Break Vectors Monitorados

`promptCacheBreakDetection.ts` rastreia 14 vetores de invalidacao:
- Adicionar MCP tool
- Timestamp no system prompt
- Trocar modelo mid-session
- Mudar imagens no prompt
- Modificar tool settings

### "Sticky Latches"

`afkModeHeaderLatched` e `fastModeHeaderLatched` previnem cache busts quando usuario toggle settings mid-session. Uma vez latchado, nao destrava.

### Custo de Cache

Na Anthropic: "We run alerts on our prompt cache hit rate and declare SEVs if they're too low."

### Fork Children e Cache

Fork children usam placeholder identico para `tool_result` blocks, garantindo prefixos byte-identical. "Spawning five forked agents costs barely more than 1."

---

## 17. Deferred Tools e Tool Search

### Problema Original

Tool definitions consumiam ~14-16K tokens no system prompt.

### Solucao: Deferred Loading

Desde v2.1.69, TODAS as built-in tools (Bash, Read, Edit, Write, etc.) sao deferred via ToolSearch.

**Reducao:** ~14-16K tokens -> ~968 tokens (93% reducao).

### Como Funciona

1. Claude ve apenas ToolSearch + non-deferred tools
2. Quando precisa de tool, busca via keyword ou `select:mcp__tool_name`
3. Retorna 3-5 `tool_reference` blocks mais relevantes
4. References expandidas automaticamente em todo o historico

### Config

```
ENABLE_TOOL_SEARCH=auto   # threshold-based (10% do context window)
ENABLE_TOOL_SEARCH=true   # sempre deferred
ENABLE_TOOL_SEARCH=false  # tudo upfront
```

---

## 18. IDE Integrations e Bridge Mode

### VS Code Extension

- Panel nativo com inline edits e IDE-style diff reviews
- Review/edit de planos antes de aceitar
- Auto-accept edits
- @-mention de arquivos com line ranges
- Multiple conversations em tabs separados

### JetBrains Plugin

- Roda CLI no terminal integrado da IDE
- Usa diff viewer da IDE para review
- Orquestra CLI (nao substitui)

### Bridge Protocol (Bidirectional Communication)

Layer de messaging bidirecional conectando CLI local com claude.ai/code, iOS, Android.

**Arquitetura:**
- CLI local faz HTTPS requests outbound only (sem portas inbound)
- Registra com Anthropic API e faz polling
- Dispositivos remotos routeiam mensagens via server
- Todo trafego sobre TLS

**NAO e** tunnel de rede (como ngrok). Forward structured application messages (prompts, tool results, status updates).

### Remote Control

- Sessao roda na maquina local
- Web/mobile sao apenas "janela" para sessao local
- Arquivos e MCP servers nunca saem da maquina
- Apenas chat messages e tool results passam pelo bridge

---

## 19. Features Ocultas: KAIROS, Undercover, Anti-Distillation

### KAIROS (Always-On Autonomous Agent)

Referenciado 150+ vezes no source. Gated sob flags `PROACTIVE` / `KAIROS`.

**Conceito:** Daemon persistente que recebe `<tick>` prompts periodicos perguntando "anything worth doing right now?"

**Especificacoes:**
- 15-second blocking budget (acoes que interromperiam usuario > 15s sao deferidas)
- Append-only daily logs (audit trail, previne self-erase)
- 3 tools exclusivas: SendUserFile, PushNotification, SubscribePR
- GitHub webhook monitoring sem intervencao
- Push notifications para usuarios
- `/dream` skill para memory distillation
- Cron refresh de 5 minutos

### Undercover Mode

Ativado via `CLAUDE_CODE_UNDERCOVER=1` ou automaticamente quando `USER_TYPE === 'ant'` (Anthropic employee) em repo nao-interno.

**Funcao:** Ocultar TODA evidencia de AI ao contribuir para open-source.

**Strip:**
- Co-Authored-By attribution
- Codenames internos (Capybara, Tengu, Fennec)
- Versoes nao-lancadas (opus-4-7, sonnet-4-8)
- Repos internos, tooling, Slack channels
- A frase "Claude Code"

**Critico:** "There is NO force-OFF" -- defaults para coberto quando incerto. Dead-code-eliminated em builds externas.

### Anti-Distillation (2 camadas)

**Layer 1 -- Fake Tools:**
Flag `ANTI_DISTILLATION_CC` envia `anti_distillation: ['fake_tools']` no request API. Servidor injeta tool schemas falsos no system prompt. Objetivo: poluir training data de competidores gravando API traffic.

Requer 4 condicoes: compile-time flag + cli entrypoint + first-party API + GrowthBook flag true.

**Layer 2 -- CONNECTOR_TEXT:**
Servidor bufferiza texto entre tool calls, sumariza com assinatura criptografica. Texto original restauravel via signature. Apenas para `USER_TYPE === 'ant'`.

**Contexto:** Em fev/2026, Anthropic documentou campanhas de distilacao por DeepSeek, Moonshot, e MiniMax (16M exchanges, 24K contas fraudulentas).

---

## 20. Feature Flags e Codenames Internos

### 44 Feature Flags (Compile-Time + Runtime)

**Compile-Time (Bun constant-folding):**
- `PROACTIVE` / `KAIROS` -- always-on assistant
- `COORDINATOR_MODE` -- multi-agent orchestration
- `BUDDY` -- companion pet system (Tamagotchi)
- `BRIDGE_MODE` -- claude.ai remote control
- `VOICE_MODE` -- comandos de voz
- `WORKFLOW_SCRIPTS` -- scripts de workflow
- `DAEMON` -- daemon mode
- `FORK_SUBAGENT` -- fork parallelization
- `ANTI_DISTILLATION_CC` -- fake tool injection
- `NATIVE_CLIENT_ATTESTATION` -- binary verification

**Runtime (GrowthBook, prefixo `tengu_`):**
- `tengu_anti_distill_fake_tool_injection`
- `tengu_penguins_off` (kill-switch Fast Mode)
- `tengu_attribution_header`
- `tengu_ultraplan_model`
- `tengu_scratch` (scratchpad directory)

### Codenames Internos

| Codename | Mapa Para |
|----------|-----------|
| **Tengu** | Claude Code (project codename, centenas de referencias) |
| **Capybara** | Nova familia de modelos (possivelmente Mythos) |
| **Fennec** | Opus 4.6 |
| **Numbat** | Modelo nao-lancado |
| **Penguin Mode** | Fast Mode |
| **Chicago** | Computer Use |

### Features Nao-Lancadas (20+)

- Background agents 24/7
- Um Claude orquestrando multiplos worker Claudes
- Cron scheduling
- Full voice command mode
- Browser control via Playwright
- Agents que dormem e se auto-retomam
- **ULTRAPLAN** -- planning remoto via Cloud Container Runtime, Opus 4.6, 30min, polling 3s, browser approval UI

### April Fools: BUDDY

`buddy/companion.ts` -- sistema Tamagotchi:
- 18 especies com tiers de raridade
- 1% probabilidade shiny
- Stats RPG (DEBUGGING, SNARK)
- Gerado deterministicamente do user ID via Mulberry32 PRNG
- Nomes de especies ofuscados com `String.fromCharCode()`

---

## 21. Token Counting e Cost Tracking

### Tracking

- `/cost` mostra uso de tokens na sessao
- Status line configuravel para display continuo
- `--plan` flag para calculos de quota (max20, pro, max5)
- Janela de quota de 5 horas

### Custos Escalam com Contexto

Cada mensagem inclui historico completo. Mensagem N envia tudo de 1 a N-1 + novo prompt.

### Otimizacoes (40-70% reducao)

1. **Model routing:** opusplan usa Opus para planning, Sonnet para implementacao
2. **Deferred tools:** 93% reducao em tool definition tokens
3. **Prompt caching:** prefix matching economiza billing
4. **Image cropping:** 200x200 = 54 tokens vs 1000x1000 = 1334 tokens (25x)
5. **Compact instructions:** "Just the code, no commentary"
6. **`/clear` entre tasks:** limpa historico irrelevante
7. **Skills on-demand:** nao carregar tudo upfront
8. **Fork agents:** "barely more than 1x cost" para parallelismo

---

## 22. Seguranca: Bash Security e Permissoes

### 23 Security Checks (`bashSecurity.ts`)

Incluem:
- 18 Zsh builtins bloqueados
- Zsh equals expansion (`=curl` permission bypass)
- Unicode zero-width space injection
- IFS null-byte injection
- Malformed token bypass (HackerOne finding)
- URL-encoded path traversal
- Unicode normalization attacks
- Backslash injection
- Case-sensitivity attacks

### 5-Layer Permission Gauntlet

1. **Tool-level checks** (Bash valida comandos destrutivos)
2. **Settings allowlist/denylist** (glob patterns)
3. **Sandbox policy** (path, command, network)
4. **Permission mode** (default, acceptEdits, plan, auto, bypass)
5. **Hook overrides** (PreToolUse)

### Client Attestation (DRM)

- `NATIVE_CLIENT_ATTESTATION` flag gates binary verification
- `CLAUDE_CODE_ATTRIBUTION_HEADER` controla header injection
- Placeholder `cch=00000` substituido pelo Zig-level HTTP stack do Bun antes da transmissao
- Attestation ABAIXO do JavaScript previne runtime patching e monkey-patching

### Frustration Detection (`userPromptKeywords.ts`)

Regex detectando estado emocional do usuario (profanidade, exasperacao). Prioriza eficiencia computacional sobre analise semantica.

---

## 23. File Formats e Token Efficiency

### Comparativo

| Formato | Economia vs JSON | Accuracy | Melhor Para |
|---------|-----------------|----------|-------------|
| Markdown | -16% tokens | Alta | Instrucoes, docs, CLAUDE.md |
| YAML | - | 62% (nested, vs 50% JSON) | Config, frontmatter |
| JSON | Baseline | 50% (nested) | Schemas, APIs |

### Recomendacao

- **CLAUDE.md:** Markdown (alinhado com linguagem natural)
- **Config:** YAML frontmatter + Markdown body
- **Skills/Agents:** YAML frontmatter em `.md` files
- **Settings:** JSON (padrao do ecossistema)
- **Limite:** < 300 linhas por arquivo

---

## 24. CLI vs Desktop vs Web

| Aspecto | CLI | Desktop | Web |
|---------|-----|---------|-----|
| **Base** | TypeScript + React/Ink TUI | App nativo com GUI | Cloud-based |
| **Execucao** | Local, terminal | Local com visual | Totalmente cloud |
| **Privacidade** | Arquivos locais, apenas API calls | Proximo do CLI | Cloud-based |
| **Browser** | Nao incluso | Browser built-in para testes | N/A |
| **Controle** | Maximo | Medio | Menor |
| **Worktrees** | Suporte nativo | Suporte built-in | N/A |
| **MCP** | Todos os transports | Todos os transports | Limitado |
| **Melhor para** | Power users, CI/CD | Visual workflows | Acesso mobile |

---

## 25. Implicacoes para SINAPSE

### FINDING

A arquitetura do Claude Code revela patterns de sofisticacao excepcional em orquestracao multi-agent, cache management, e governance. Os sistemas de hooks, skills, e agents sao quase identicos aos conceitos que o SINAPSE implementa.

### IMPLICATION

1. **Nosso hook system e equivalente ao deles** -- validacao de que o approach esta correto
2. **Deferred tools e a maior oportunidade** -- podemos economizar 93% de tokens em tool definitions
3. **A separacao static/dynamic no system prompt** e critica para custos
4. **Worktree isolation** para subagents e um pattern comprovado que devemos adotar
5. **Prompt cache stability** (sticky latches, alphabetical tool ordering) e engineering sutil mas de alto impacto
6. **Memory architecture em 3 camadas** (index + topics + transcripts) e superior ao flat file
7. **AutoDream-like consolidation** previne memory bloat

### RECOMMENDATION

1. Implementar **deferred tool loading** no SINAPSE imediatamente
2. Adotar **cache boundary pattern** (`DYNAMIC_BOUNDARY`) na construcao de system prompts
3. Explorar **worktree isolation** para agent squads paralelos
4. Implementar **Compact Instructions** nos CLAUDE.md do SINAPSE
5. Considerar **3-layer memory** (index -> topics -> grep-only transcripts)
6. Manter **skills < 300 linhas** e carregar on-demand
7. Usar **YAML frontmatter + Markdown body** como formato padrao

---

## FONTES

### Leak e Analise

- [Alex Kim - Source Leak Analysis](https://alex000kim.com/posts/2026-03-31-claude-code-source-leak/)
- [ClaudeFast - Everything Found](https://claudefa.st/blog/guide/mechanics/claude-code-source-leak)
- [Engineer's Codex - Diving into Source](https://read.engineerscodex.com/p/diving-into-claude-codes-source-code)
- [Kuber.studio - Full Breakdown](https://kuber.studio/blog/AI/Claude-Code's-Entire-Source-Code-Got-Leaked-via-a-Sourcemap-in-npm,-Let's-Talk-About-it)
- [Markdown Engineering - Source Deep Dive Course](https://www.markdown.engineering/learn-claude-code/)
- [Latent Space - AINews](https://www.latent.space/p/ainews-the-claude-code-source-leak)
- [DEV.to - Agentic Loop Architecture](https://dev.to/oldeucryptoboi/inside-claude-codes-architecture-the-agentic-loop-that-codes-for-you-cmk)
- [Penligent - Architecture Behind Tools, Memory, Hooks, MCP](https://www.penligent.ai/hackinglabs/inside-claude-code-the-architecture-behind-tools-memory-hooks-and-mcp/)

### Cobertura de Noticias

- [VentureBeat - Source Leak](https://venturebeat.com/technology/claude-codes-source-code-appears-to-have-leaked-heres-what-we-know/)
- [Cybernews - Fastest Growing Repo](https://cybernews.com/tech/claude-code-leak-spawns-fastest-github-repo/)
- [The Register - Source Code Exposed](https://www.theregister.com/2026/03/31/anthropic_claude_code_source_code/)
- [Hacker News - Source Leaked via npm](https://thehackernews.com/2026/04/claude-code-tleaked-via-npm-packaging.html)
- [TechCrunch - DMCA Takedowns](https://techcrunch.com/2026/04/01/anthropic-took-down-thousands-of-github-repos-trying-to-yank-its-leaked-source-code-a-move-the-company-says-was-an-accident/)
- [VentureBeat - Enterprise Security Actions](https://venturebeat.com/security/claude-code-512000-line-source-leak-attack-paths-audit-security-leaders/)
- [Nick Spisak Tweet](https://x.com/NickSpisak_/status/2039025241825939768)

### Documentacao Oficial

- [Claude Code Docs - How It Works](https://code.claude.com/docs/en/how-claude-code-works)
- [Claude Code Docs - Hooks Reference](https://code.claude.com/docs/en/hooks)
- [Claude Code Docs - Skills](https://code.claude.com/docs/en/skills)
- [Claude Code Docs - MCP](https://code.claude.com/docs/en/mcp)
- [Claude Code Docs - Settings](https://code.claude.com/docs/en/settings)
- [Claude Code Docs - Memory](https://code.claude.com/docs/en/memory)
- [Claude Code Docs - Sandboxing](https://code.claude.com/docs/en/sandboxing)
- [Claude Code Docs - Subagents](https://code.claude.com/docs/en/sub-agents)
- [Claude Code Docs - Agent Teams](https://code.claude.com/docs/en/agent-teams)
- [Claude Code Docs - Remote Control](https://code.claude.com/docs/en/remote-control)
- [Claude Code Docs - Compaction](https://platform.claude.com/docs/en/build-with-claude/compaction)
- [Claude Code Docs - Cost Management](https://code.claude.com/docs/en/costs)
- [Claude API - Tool Search](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool)
- [Claude API - Agent SDK](https://platform.claude.com/docs/en/agent-sdk/overview)
- [Claude API - Prompt Caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)

### Features e Anti-Distillation

- [Cybernews - Hidden Features](https://cybernews.com/security/anthropic-claude-source-code-discovered-features/)
- [WinBuzzer - Anti-Distillation Traps](https://winbuzzer.com/2026/04/01/claude-code-source-leak-anti-distillation-traps-undercover-mode-xcxwbn/)
- [MindStudio - 8 Hidden Features](https://www.mindstudio.ai/blog/claude-code-source-code-leak-8-hidden-features)
- [Anthropic Engineering - Sandboxing](https://www.anthropic.com/engineering/claude-code-sandboxing)

### Community Analysis

- [GitHub - System Prompts Leaks](https://github.com/asgeirtj/system_prompts_leaks)
- [GitHub - Claurst (Rust Rewrite + Breakdown)](https://github.com/Kuberwastaken/claurst)
- [GitHub - Claude Code Analysis](https://github.com/ComeOnOliver/claude-code-analysis)
- [GitHub - Deep Dive Claude Code](https://github.com/waiterxiaoyy/Deep-Dive-Claude-Code)
- [Addy Osmani - Claude Code Swarms](https://addyosmani.com/blog/claude-code-agent-teams/)
- [ClaudeCodeCamp - Prompt Caching](https://www.claudecodecamp.com/p/how-prompt-caching-actually-works-in-claude-code)

---

*Pesquisa conduzida por Prism (Research Orchestrator) -- squad-research*
*Nivel: DEFINITIVE (L4) | 30+ queries | 15+ paginas full-fetch | Tier 1-4 sources*
*Data: 2026-04-04*
