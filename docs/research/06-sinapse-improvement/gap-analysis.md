# SINAPSE-AI Gap Analysis — Cirurgica

> **Data:** 2026-04-04
> **Autor:** Prism (Research Orchestrator) + Sage (Deep Research)
> **Fontes:** Claude Code leaked source (v2.1.88), claw-code clean-room rewrite, AIOX-Core v4.2.11, BMAD Method v6, audit interno do SINAPSE
> **Objetivo:** Identificar gaps entre o SINAPSE-AI e a arquitetura interna do Claude Code, propondo otimizacoes com estimativas concretas

---

## Indice

1. [Token Economy Gaps (CRITICAL)](#1-token-economy-gaps-critical)
2. [Context Engineering Gaps](#2-context-engineering-gaps)
3. [Memory System Gaps](#3-memory-system-gaps)
4. [Planning Pipeline Gaps](#4-planning-pipeline-gaps)
5. [Agent Architecture Gaps](#5-agent-architecture-gaps)
6. [Skills & Commands Gaps](#6-skills--commands-gaps)
7. [Hooks & Governance Gaps](#7-hooks--governance-gaps)
8. [Security & Quality Gaps](#8-security--quality-gaps)
9. [NPM Distribution & Installation Gaps](#9-npm-distribution--installation-gaps)
10. [Naming, Organization & Perfectionism Gaps](#10-naming-organization--perfectionism-gaps)
11. [Project Scaffolding Gaps](#11-project-scaffolding-gaps)
12. [Productization Gaps](#12-productization-gaps)

---

## 1. Token Economy Gaps (CRITICAL)

### 1.1 Overhead Total Estimado do SINAPSE Atual

O Claude Code impoe limites rigidos que o SINAPSE esta estourando consistentemente.

**Limites do Claude Code:**
- `MAX_TOTAL_INSTRUCTION_CHARS = 12,000` (todos os CLAUDE.md combinados)
- `MAX_INSTRUCTION_FILE_CHARS = 4,000` (por arquivo)
- Formula de token: `chars / 4 + 1`

**SINAPSE — Medicao real (bytes ~ chars para ASCII):**

| Fonte | Chars | Tokens estimados | Status |
|-------|-------|-----------------|--------|
| Global CLAUDE.md (`~/.claude/CLAUDE.md`) | 6,880 | ~1,721 | **EXCEDE** limite de 4,000 chars/arquivo |
| Project CLAUDE.md (`.claude/CLAUDE.md`) | 13,393 | ~3,349 | **EXCEDE** limite de 4,000 chars/arquivo |
| **Total CLAUDE.md** | **20,273** | **~5,069** | **EXCEDE 12K limit em 69%** |

O Claude Code trunca silenciosamente apos 4,000 chars por arquivo e 12,000 total. Isso significa que **~40% do conteudo dos CLAUDE.md do SINAPSE nunca e lido pelo modelo**.

**Rules — Todas carregam em toda mensagem (sem path filtering):**

| Rule File | Chars | Tokens |
|-----------|-------|--------|
| security-data-protection.md | 8,348 | ~2,088 |
| safe-collaboration.md | 6,201 | ~1,551 |
| mcp-usage.md | 5,867 | ~1,467 |
| workflow-execution.md | 4,971 | ~1,243 |
| story-lifecycle.md | 4,543 | ~1,136 |
| squad-awareness.md | 3,990 | ~998 |
| mandatory-delegation.md | 3,842 | ~961 |
| ids-principles.md | 3,681 | ~921 |
| agent-handoff.md | 3,602 | ~901 |
| agent-authority.md | 3,380 | ~846 |
| tool-examples.md | 3,177 | ~795 |
| documentation-first.md | 2,799 | ~700 |
| coderabbit-integration.md | 2,659 | ~665 |
| hook-governance.md | 2,399 | ~600 |
| tool-response-filtering.md | 2,233 | ~559 |
| nsn-mode.md | 1,952 | ~489 |
| cross-squad-routing.md | 1,939 | ~485 |
| security-scanning.md | 1,727 | ~432 |
| agent-memory-imports.md | 630 | ~158 |
| **TOTAL (19 rules)** | **~67,939** | **~16,995** |

**Global rules (13 rules, mesma sessao):**

| Total chars | Total tokens |
|-------------|-------------|
| ~41,976 | ~10,495 |

**NOTA IMPORTANTE:** Rules com `paths:` frontmatter (como `story-lifecycle.md` e `mcp-usage.md`) teoricamente so carregam quando arquivos correspondentes sao editados. Porem, `mcp-usage.md` tem `paths: **/*` — que e equivalente a carregar sempre. Das 19 rules do projeto, apenas `story-lifecycle.md` tem path filtering real (`docs/stories/**`). As demais carregam em TODA mensagem.

**Overhead total estimado por mensagem:**

| Componente | Tokens |
|-----------|--------|
| CLAUDE.md (global + project, truncado) | ~3,001 (12K chars / 4) |
| Rules (project, sem filtering) | ~16,995 |
| Rules (global) | ~10,495 |
| Hooks (5, overhead minimo por serem scripts) | ~50 |
| Agent persona (quando ativo) | ~800-1,500 |
| **TOTAL ESTIMADO** | **~31,000-32,000** |

Para comparacao: o Claude Code internamente usa ~200 prompt fragments carregados condicionalmente, com tool descriptions usando Deferred Tools (reducao de 14-16K para 968 tokens = 93% menos).

**Custo financeiro:**
- Com cache: ~$0.045/M tokens de cache read (Opus)
- Sem cache (dynamic content): ~$15/M tokens
- 32K tokens de instrucoes por turn = ~$0.48/turn sem cache, ~$0.0014/turn com cache
- Uma sessao de 50 turns com tudo dynamic: **~$24 so em instrucoes**
- Com cache sharing: **~$0.07 so em instrucoes**

O problema: CLAUDE.md e rules vao na secao DYNAMIC do prompt (apos o `SYSTEM_PROMPT_DYNAMIC_BOUNDARY`), entao NAO compartilham cache entre usuarios/sessoes.

### 1.2 Gap: Conteudo Truncado Silenciosamente

- **Current State:** CLAUDE.md global tem 6,880 chars (limite: 4,000). Project CLAUDE.md tem 13,393 chars (limite: 4,000). Total: 20,273 chars (limite: 12,000). Tudo alem de 4K/arquivo e descartado.
- **Target State:** Cada arquivo <= 4,000 chars. Total <= 12,000 chars. Conteudo critico no topo de cada arquivo. Informacao restante movida para skills on-demand.
- **Gap Severity:** CRITICAL
- **Estimated Effort:** M (reescrever CLAUDE.md files)
- **Estimated Token Savings:** ~8,273 chars eliminados (conteudo que ja era ignorado). Ganho real: garantia de que instrucoes criticas nao sao cortadas.
- **Priority:** P0

### 1.3 Gap: Rules Sem Path Filtering

- **Current State:** 18 de 19 rules do projeto carregam em toda mensagem (~67K chars = ~17K tokens). Apenas `story-lifecycle.md` tem path filtering real. `mcp-usage.md` tem `paths: **/*` que e inutil.
- **Target State:** Cada rule com `paths:` frontmatter preciso. Exemplo: `safe-collaboration.md` so carrega quando `.git/` ou branches sao tocados. `security-data-protection.md` so quando `.env`, `supabase/`, ou deploy configs sao editados.
- **Gap Severity:** CRITICAL
- **Estimated Effort:** S (adicionar frontmatter a 18 arquivos)
- **Estimated Token Savings:** Estimativa conservadora: se 70% das rules nao sao relevantes em um turn medio, economia de ~12K tokens/turn (~$0.18/turn sem cache).
- **Priority:** P0

### 1.4 Gap: Duplicacao entre Global e Project CLAUDE.md

- **Current State:** Constitution table, agent table, agent commands, framework structure, boundary rules — todos duplicados entre global e project CLAUDE.md. Secoes inteiras sao identicas.
- **Target State:** Global CLAUDE.md contem APENAS regras que se aplicam a TODO projeto (safe collaboration, git safety, user profile). Project CLAUDE.md contem APENAS regras especificas do projeto. Zero duplicacao.
- **Gap Severity:** HIGH
- **Estimated Effort:** M
- **Estimated Token Savings:** ~3,000-4,000 chars eliminados = ~750-1,000 tokens/turn
- **Priority:** P0

### 1.5 Gap: Rules Duplicam Conteudo do CLAUDE.md

- **Current State:** `documentation-first.md` (rule) repete quase textualmente o que esta no CLAUDE.md. `mandatory-delegation.md` repete a delegation matrix. `agent-authority.md` repete tabelas de agentes.
- **Target State:** CLAUDE.md contem resumo ultra-compacto (1-2 linhas por conceito). Rules contem detalhes completos. Sem sobreposicao.
- **Gap Severity:** HIGH
- **Estimated Effort:** M
- **Estimated Token Savings:** ~5,000 chars = ~1,250 tokens/turn
- **Priority:** P0

### 1.6 Gap: Secoes Irrelevantes nos CLAUDE.md

- **Current State:** CLAUDE.md do projeto contem code snippets (error handling, template loading, agent command handling), debugging instructions, common commands — informacao que o modelo ja sabe ou que raramente e necessaria.
- **Target State:** Remover: code examples genericos, debugging instructions, environment setup, common commands. Manter apenas: regras de comportamento, constraints, e decisoes arquiteturais unicas do SINAPSE.
- **Gap Severity:** MEDIUM
- **Estimated Effort:** S
- **Estimated Token Savings:** ~2,000-3,000 chars = ~500-750 tokens/turn
- **Priority:** P1

### 1.7 Resumo de Economia Potencial (Token Economy)

| Otimizacao | Savings (tokens/turn) | Priority |
|-----------|----------------------|----------|
| Path filtering em rules | ~12,000 | P0 |
| Eliminar duplicacao global/project | ~750-1,000 | P0 |
| Eliminar duplicacao rules/CLAUDE.md | ~1,250 | P0 |
| Respeitar limite 4K/arquivo | (evita truncation) | P0 |
| Remover conteudo irrelevante | ~500-750 | P1 |
| **TOTAL** | **~14,500-15,000** | — |

Isso representa uma reducao de **~45-47%** do overhead atual por turn.

---

## 2. Context Engineering Gaps

### 2.1 Gap: Ausencia de Separacao Static/Dynamic

- **Current State:** O SINAPSE nao diferencia entre conteudo que poderia ser cacheado (estatico) e conteudo que muda por sessao (dinamico). Tudo vai na mesma "pilha" de instrucoes.
- **Target State:** Instruir o Claude Code a colocar conteudo estavel (constitution, agent roles, core principles) no system prompt estatico e conteudo variavel (current story, branch status, active agent) na zona dinamica. Na pratica, isso significa: mover o maximo possivel para o CLAUDE.md (que e cacheable entre sessions do mesmo projeto) e minimizar rules que mudam frequentemente.
- **Gap Severity:** HIGH
- **Estimated Effort:** M
- **Estimated Token Savings:** Se 60% do conteudo atual puder ser cacheado: de ~$0.48/turn para ~$0.19/turn (reducao de ~60% no custo de instrucoes).
- **Priority:** P1

**Nota tecnica:** O `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` do Claude Code separa o system prompt builtin (cacheable, compartilhado entre todos os usuarios) do conteudo user-supplied (CLAUDE.md, rules). O conteudo do SINAPSE esta TODO na zona dinamica. A unica forma de cachear e manter o mesmo conteudo entre turns da mesma sessao — o que ja acontece naturalmente para CLAUDE.md, mas NAO para rules que poderiam ser filtradas.

### 2.2 Gap: Ausencia de Document Sharding

- **Current State:** Stories, PRDs, e specs sao documentos monoliticos carregados integralmente. Uma story de 200 linhas consome ~1,500 tokens mesmo quando apenas uma secao (AC, File List) e relevante.
- **Target State:** Seguir o padrao BMAD de document sharding: quebrar documentos grandes em shards de ~300 tokens com metadata de roteamento. Carregar apenas o shard relevante para a task atual.
- **Gap Severity:** MEDIUM
- **Estimated Effort:** L (requer mudanca no story template + tooling)
- **Estimated Token Savings:** ~70-80% por documento carregado (de ~1,500 para ~300-450 tokens por story reference)
- **Priority:** P2

### 2.3 Gap: Ausencia de Deferred Tool Pattern

- **Current State:** Quando MCPs estao ativos, todas as tool descriptions sao carregadas no prompt. Cada MCP tool adiciona ~200-500 chars ao prompt.
- **Target State:** O Claude Code usa Deferred Tools internamente: tool descriptions sao comprimidas de 14-16K tokens para 968 tokens (93% reducao). SINAPSE deveria implementar um registry leve de tools no CLAUDE.md que apenas lista nomes + 1-line descriptions, com detalhes carregados on-demand via skill ou hook.
- **Gap Severity:** MEDIUM
- **Estimated Effort:** M (depende de como MCP tools sao configurados)
- **Estimated Token Savings:** Depende do numero de MCPs ativos. Com 5 MCPs (~15 tools): ~3,000-5,000 tokens salvos.
- **Priority:** P2

**Nota:** O SINAPSE ja tem `tool-examples.md` e `tool-response-filtering.md` que parcialmente atendem isso, mas nao sao deferred — sao carregados sempre.

### 2.4 Gap: System Reminders Nao Utilizados

- **Current State:** O SINAPSE nao usa system reminders contextuais. O Claude Code injeta 30+ tipos de system reminders em momentos especificos (quando um arquivo e modificado externamente, quando o plan mode esta ativo, quando memory files sao relevantes, etc.).
- **Target State:** Implementar um mecanismo de system reminders via hooks que injeta contexto relevante apenas quando necessario. Exemplo: quando o usuario menciona um agente, injetar apenas a persona desse agente (nao de todos os 175).
- **Gap Severity:** LOW (o hook system atual ja pode fazer parcialmente isso)
- **Estimated Effort:** L
- **Estimated Token Savings:** Dificil estimar; depende da implementacao
- **Priority:** P3

---

## 3. Memory System Gaps

### 3.1 Gap: Ausencia de Memory Consolidation Automatica

- **Current State:** O SINAPSE usa MEMORY.md index + topic files (Ars Contexta cronjob as 23h BRT via Haiku). E um cronjob externo ao Claude Code, dependente de trigger manual ou cron do OS.
- **Target State:** O Claude Code tem `autoDream` — 4-phase consolidation (Orient/Gather/Consolidate/Prune) que roda automaticamente como forked sub-agent durante idle periods. Trigger: >= 24h desde ultima consolidacao, >= 5 sessoes novas, nenhuma consolidacao ativa, >= 10min desde ultimo scan. Triple gate com mtime-based state rollback.
- **Gap Severity:** HIGH
- **Estimated Effort:** L (requer implementar como hook SessionEnd ou comando /dream)
- **Estimated Token Savings:** Indireto — previne memoria desatualizada que causa decisoes erradas e re-trabalho.
- **Priority:** P1

### 3.2 Gap: Ausencia de Memory Relevance Scoring

- **Current State:** O SINAPSE carrega MEMORY.md integralmente ou nao carrega. Nao ha scoring de relevancia — o modelo decide sozinho o que e relevante no conteudo carregado.
- **Target State:** O Claude Code tem `findRelevantMemories.ts` e `determine-which-memory-files-to-attach` — um prompt especializado que avalia quais memory files sao relevantes para o contexto atual antes de anexa-los. O AIOX tem 4 camadas (HOT/WARM/COLD) com scoring temporal e de frequencia, alegando 73% de reducao de tokens.
- **Gap Severity:** HIGH
- **Estimated Effort:** L (requer implementar scoring no hook ou pre-prompt)
- **Estimated Token Savings:** ~50-73% do conteudo de memoria carregado
- **Priority:** P1

### 3.3 Gap: Ausencia de Memory Size Limits

- **Current State:** Nao ha limite definido para MEMORY.md ou topic files. Pode crescer indefinidamente.
- **Target State:** O Claude Code limita a ~200 lines / ~25KB. Index entries tem ~150 chars max. Consolidation prune mantem dentro do budget.
- **Gap Severity:** MEDIUM
- **Estimated Effort:** S (definir limites, implementar pruning no Ars Contexta)
- **Estimated Token Savings:** Previne degradacao ao longo do tempo
- **Priority:** P1

### 3.4 Gap: Ausencia de "Memory as Hints" Pattern

- **Current State:** O SINAPSE trata memoria como fonte de verdade. Nao ha instrucao explicita para verificar contra o codebase real antes de agir.
- **Target State:** O Claude Code instrui explicitamente: "Memory entries are hints, not ground truth. Always verify against actual codebase before acting." Tambem usa "Strict Write Discipline" — updates de memoria so apos acoes bem-sucedidas (previne cimentar erros).
- **Gap Severity:** MEDIUM
- **Estimated Effort:** S (adicionar instrucao ao CLAUDE.md e ao protocol de memoria)
- **Estimated Token Savings:** Indireto — previne erros por memoria desatualizada
- **Priority:** P1

### 3.5 Gap: Ausencia de Contradiction Resolution

- **Current State:** Se duas entradas de memoria contradizem, nao ha mecanismo para detectar e resolver.
- **Target State:** Phase 3 (Consolidate) do autoDream detecta contradicoes e resolve mantendo a mais recente + logging da resolucao.
- **Gap Severity:** LOW
- **Estimated Effort:** M
- **Estimated Token Savings:** Marginal
- **Priority:** P2

---

## 4. Planning Pipeline Gaps

### 4.1 Gap: Pipeline Rigido vs Adaptativo

- **Current State:** O SINAPSE tem 4 workflows fixos (SDC, QA Loop, Spec Pipeline, Brownfield) com fases rigidas. Todo trabalho DEVE passar por Epic -> Story -> Validate -> Implement -> QA. Mesmo um bugfix de 1 linha requer story + validacao.
- **Target State:** O AIOX tem 14 workflows com adaptacao por complexidade. O Claude Code internamente usa "Scale-adaptive intelligence" — ajusta complexidade de bug fixes a enterprise systems. O BMAD tem 34+ workflows. O SINAPSE deveria ter: (1) fast-track para fixes triviais (story auto-validada), (2) standard track para features, (3) heavy track para initiatives complexas.
- **Gap Severity:** HIGH
- **Estimated Effort:** M (definir criterios de fast-track, ajustar hooks)
- **Estimated Token Savings:** ~5,000-10,000 tokens por fast-track task (evita carregar story templates + validation rules)
- **Priority:** P1

### 4.2 Gap: Ausencia de Execution Modes Formais

- **Current State:** Story lifecycle menciona 3 modes (YOLO, Interactive, Pre-Flight) mas nao ha mecanismo formal de selecao ou enforcement.
- **Target State:** Definir criterios claros de selecao automatica baseados em complexidade estimada. O Claude Code usa `plan-mode-enhanced` com 5 fases + `remote-plan-mode-ultraplan` para planejamento complexo.
- **Gap Severity:** MEDIUM
- **Estimated Effort:** M
- **Estimated Token Savings:** Indireto — melhora eficiencia de execucao
- **Priority:** P2

### 4.3 Gap: Spec Pipeline Sem Pruning

- **Current State:** Spec Pipeline gera 6 outputs (requirements.json, complexity.json, research.json, spec.md, critique.json, implementation.yaml) que podem totalizar 5-10K tokens. Nao ha pruning apos aprovacao.
- **Target State:** Apos aprovacao, comprimir spec pipeline outputs em um artifact unico de ~500-1,000 tokens com apenas as decisoes e constraints. Artifacts intermediarios arquivados.
- **Gap Severity:** LOW
- **Estimated Effort:** S
- **Estimated Token Savings:** ~3,000-5,000 tokens por spec pipeline completo
- **Priority:** P2

### 4.4 Gap: Ausencia de Story Quality Scoring Automatico

- **Current State:** Validacao de story e manual (10-point checklist, @product-lead). Nao ha scoring automatico pre-validacao.
- **Target State:** Hook ou skill que roda checklist automaticamente na criacao da story, gerando score e flags para @product-lead revisar. Reduz overhead de validacao manual.
- **Gap Severity:** LOW
- **Estimated Effort:** M
- **Estimated Token Savings:** ~2,000 tokens por ciclo de validacao (menos back-and-forth)
- **Priority:** P3

---

## 5. Agent Architecture Gaps

### 5.1 Gap: Custo Proibitivo de Delegacao via Sub-Agent

- **Current State:** O SINAPSE delega trabalho entre agentes como mudanca de persona no mesmo context. Quando usa SubAgent real (AgentTool), o custo minimo e ~20K tokens por spawn (context replication + agent instructions).
- **Target State:** O Claude Code usa Worker Fork Model: workers isolados que retornam structured reports de 500 palavras. O coordenador NAO precisa do contexto completo do worker. SINAPSE deveria usar: (1) persona switching para mudancas leves (ja faz), (2) worker forks apenas para tarefas paralelas ou isoladas, (3) NUNCA sub-agent para tarefas sequenciais simples.
- **Gap Severity:** HIGH
- **Estimated Effort:** M (documentar quando usar cada modelo + ajustar delegation rules)
- **Estimated Token Savings:** ~15,000+ tokens por delegacao desnecessaria evitada
- **Priority:** P1

### 5.2 Gap: 175 Agentes — Overhead de Definicao

- **Current State:** 175 agentes definidos em YAML/Markdown. Cada persona tem ~100-200 linhas. Apenas 10-12 sao usados em desenvolvimento (framework agents). Os outros 165 sao squad specialists que raramente ativam.
- **Target State:** O AIOX tem 12 agentes core. O Claude Code tem ~8 built-in agents. A maioria dos 175 agentes SINAPSE sao domain specialists que deveriam ser carregados on-demand via skill, nao pre-definidos. Manter: 10-12 framework agents. Mover: 165 squad agents para skills carregaveis.
- **Gap Severity:** MEDIUM (agentes nao carregam ate serem invocados, mas o squad-awareness.md lista todos)
- **Estimated Effort:** L
- **Estimated Token Savings:** ~1,000 tokens (squad-awareness.md atual tem ~4K chars listando todos os squads)
- **Priority:** P2

### 5.3 Gap: Agent Handoff Protocol Verboso

- **Current State:** Handoff protocol gera artifacts de ~500 tokens com formato YAML detalhado. Limit de 3 retained summaries. Scratchpad protocol adiciona mais overhead.
- **Target State:** Worker Fork Model do Claude Code: structured report de 500 palavras (~600 tokens) como resultado FINAL, sem artifacts intermediarios retidos. O coordenador descarta tudo exceto o report.
- **Gap Severity:** LOW (handoff protocol funciona, so e verbose)
- **Estimated Effort:** S
- **Estimated Token Savings:** ~300-500 tokens por switch
- **Priority:** P2

### 5.4 Gap: Ausencia de Agent Selection Decision Tree

- **Current State:** Delegacao depende do modelo entender a delegation matrix (tabela no CLAUDE.md + rules). Nao ha decision tree formal.
- **Target State:** O AIOX tem um Agent Selection Decision Tree explicito. O Claude Code tem `agent-creation-architect` prompt que avalia quando criar novos agentes. SINAPSE deveria ter um flowchart simples: "request type -> agent" com fallback rules.
- **Gap Severity:** LOW
- **Estimated Effort:** S
- **Estimated Token Savings:** Marginal (melhora precisao, nao token count)
- **Priority:** P3

### 5.5 Gap: Ausencia de Cache Sharing Between Agents

- **Current State:** Quando agentes sao invocados como sub-agents, cada um recebe copia completa do contexto. Nao ha otimizacao de cache sharing.
- **Target State:** O Claude Code implementa cache sharing: "byte-identical context copies share the KV cache." Sub-agents pagam apenas pelas instrucoes unicas. SINAPSE deveria maximizar instrucoes compartilhadas (constitution, project rules) e minimizar instrucoes unicas por agente.
- **Gap Severity:** MEDIUM
- **Estimated Effort:** M
- **Estimated Token Savings:** Potencialmente ~50-80% do custo de sub-agent spawn se contexto base for identico
- **Priority:** P2

---

## 6. Skills & Commands Gaps

### 6.1 Gap: Ausencia de Skills System

- **Current State:** O SINAPSE nao tem skills. Tasks sao definidas em Markdown files que o agente le e executa. Nao ha mecanismo de carregamento on-demand de comportamento especializado.
- **Target State:** O Claude Code tem 14 skills (batch, debug, loop, simplify, stuck, verify, etc.) que funcionam como "prompt bundles" carregados on-demand via Skill tool. O SINAPSE deveria converter: (1) knowledge bases em skills, (2) protocolos complexos em skills, (3) tasks repetitivas em skills.
- **Gap Severity:** HIGH
- **Estimated Effort:** L (definir formato de skill, converter conteudo existente)
- **Estimated Token Savings:** Enorme — mover conteudo de rules (always-loaded) para skills (on-demand). Exemplo: `security-data-protection.md` (8,348 chars) como skill = ~2,088 tokens salvos quando seguranca nao e relevante.
- **Priority:** P1

**Classificacao proposta — Global vs Contextual:**

| Always-Loaded (Global) | On-Demand (Skill) |
|------------------------|-------------------|
| Constitution (resumo) | security-data-protection |
| Agent authority (resumo) | safe-collaboration |
| Documentation-first gate | mcp-usage |
| Mandatory delegation | workflow-execution (details) |
| | story-lifecycle (details) |
| | ids-principles |
| | coderabbit-integration |
| | squad-awareness |
| | hook-governance |
| | nsn-mode |
| | cross-squad-routing |
| | tool-examples |
| | tool-response-filtering |

Global: ~4-5 rules (~10-12K chars = ~2,500-3,000 tokens)
Skills: ~14 rules (~55-58K chars = ~14,000 tokens salvos quando nao relevantes)

### 6.2 Gap: Custom Commands Sem Otimizacao

- **Current State:** 7 custom commands (checkpoint, context, inbox, msg, resume, session, sessions). Cada comando e um script que carrega contexto adicional.
- **Target State:** O Claude Code tem ~99 commands identificados, organizados por categoria (core, git, config, IDE, account, advanced). SINAPSE commands sao poucos mas uteis. O gap e menor aqui — a otimizacao seria garantir que commands nao injetem contexto desnecessario no prompt.
- **Gap Severity:** LOW
- **Estimated Effort:** S
- **Estimated Token Savings:** Depende de implementacao
- **Priority:** P3

### 6.3 Gap: Ausencia de Skill de Verificacao/Debug

- **Current State:** Nao ha skill equivalente a `verify`, `debug`, ou `stuck` do Claude Code.
- **Target State:** Skills que: (1) verificam se a implementacao atende os AC da story, (2) ajudam quando o agente esta travado, (3) debugam erros comuns.
- **Gap Severity:** MEDIUM
- **Estimated Effort:** M
- **Estimated Token Savings:** Indireto — reduz ciclos de tentativa-e-erro
- **Priority:** P2

---

## 7. Hooks & Governance Gaps

### 7.1 Gap: Hooks Limitados a 2 de 9 Lifecycle Events

- **Current State:** 5 hooks ativos, todos em PreToolUse, PostToolUse, UserPromptSubmit, ou PreCompact. Nao ha hooks em: SessionStart, SessionEnd, Notification, Stop, SubagentStop.
- **Target State:** O Claude Code suporta 9 lifecycle events. SINAPSE deveria usar:
  - `SessionStart` — auto-sync git, carregar branch context, verificar memoria
  - `SessionEnd` — trigger memory consolidation, salvar scratchpad
  - `Notification` — informar sobre mudancas externas
  - `Stop` — cleanup, salvar estado
  - `SubagentStop` — capturar resultado de sub-agents
- **Gap Severity:** MEDIUM
- **Estimated Effort:** M (implementar novos hooks)
- **Estimated Token Savings:** Indireto — melhor lifecycle management
- **Priority:** P2

### 7.2 Gap: Hook de Enforcement Nao Verifica Budget de Tokens

- **Current State:** Hooks verificam: git push authority, dangerous SQL, delegation enforcement, architecture-first, write path, story gate, slug validation, mind-clone governance, read protection. NENHUM verifica se o contexto esta dentro do budget de tokens.
- **Target State:** Hook PreToolUse que estima token count de instrucoes e emite WARNING se exceder 80% do budget implcito (~10K tokens de instrucoes). Previne degradacao silenciosa.
- **Gap Severity:** LOW
- **Estimated Effort:** M
- **Estimated Token Savings:** Prevencao de overflow
- **Priority:** P3

### 7.3 Gap: Ausencia de Auto-Sync no Inicio de Sessao

- **Current State:** `safe-collaboration.md` define que auto-sync DEVE acontecer no inicio de toda sessao. Mas nao ha hook SessionStart implementado que force isso. Depende do agente "lembrar" de fazer.
- **Target State:** Hook `SessionStart` que automaticamente executa: `git fetch origin`, verifica divergencia, cria branch se em main.
- **Gap Severity:** HIGH
- **Estimated Effort:** M
- **Estimated Token Savings:** Indireto — previne conflitos e re-trabalho
- **Priority:** P1

### 7.4 Gap: Hook Performance Nao Monitorado

- **Current State:** Hook governance define que hooks devem completar em <5 segundos, mas nao ha monitoramento.
- **Target State:** Logging de tempo de execucao de cada hook. Alertas se hooks excederem threshold.
- **Gap Severity:** LOW
- **Estimated Effort:** S
- **Estimated Token Savings:** Nenhum direto
- **Priority:** P3

---

## 8. Security & Quality Gaps

### 8.1 Gap: Ausencia de Bash Parser com AST

- **Current State:** O SINAPSE tem `sql-governance.py` para SQL e `enforce-git-push-authority.sh` para git push. Validacao de bash commands depende de pattern matching simples.
- **Target State:** O Claude Code tem 9,707 linhas de bash validation com tree-sitter WASM parser que constroi AST de cada comando antes de execucao. 23+ validators numerados cobrem: zero-width space injection, Unicode injection, IFS attacks, redirect validation, destructive command warnings, etc.
- **Gap Severity:** HIGH (mas mitigado pelo sandbox do Claude Code)
- **Estimated Effort:** XL (implementar parser AST e inviavel; melhor approach: usar o sandbox nativo do Claude Code + hooks pontuais)
- **Estimated Token Savings:** Nenhum
- **Priority:** P2

### 8.2 Gap: Secret Scanning Incompleto

- **Current State:** `safe-collaboration.md` define que pre-commit deve ter secret scan, e `security-scanning.md` define rules. Mas a implementacao depende de hooks que fazem regex matching basico.
- **Target State:** Integrar `gitleaks` ou `truffleHog` como hook PreToolUse em Write/Edit. O Claude Code tem `security-monitor-first-part` e `security-monitor-second-part` como prompts dedicados.
- **Gap Severity:** MEDIUM
- **Estimated Effort:** M
- **Estimated Token Savings:** Nenhum direto
- **Priority:** P1

### 8.3 Gap: CodeRabbit Integration Nao Automatica

- **Current State:** `coderabbit-integration.md` define rules mas depende de invocacao manual via WSL. Nao ha hook automatico pre-commit ou pre-push.
- **Target State:** Hook PostToolUse em `git commit` que automaticamente roda CodeRabbit review. Self-healing loop max 2 iteracoes (ja definido na story-lifecycle).
- **Gap Severity:** MEDIUM
- **Estimated Effort:** M
- **Estimated Token Savings:** Indireto — previne commits de baixa qualidade
- **Priority:** P2

### 8.4 Gap: Pre-Deploy Gate Extenso Mas Nao Automatizado

- **Current State:** `security-data-protection.md` define 25 deployment blockers em 3 tiers. Todos sao checklists manuais — nenhum e enforced por hooks.
- **Target State:** Automatizar pelo menos Tier 1 (10 absolute blockers) como hooks pre-push. Exemplo: RLS check via SQL query, service_role grep, npm audit, gitleaks scan.
- **Gap Severity:** HIGH
- **Estimated Effort:** L
- **Estimated Token Savings:** Nenhum
- **Priority:** P1

---

## 9. NPM Distribution & Installation Gaps

### 9.1 Gap: Ausencia de Installer NPX

- **Current State:** O `sinapse-ai` npm package existe mas nao tem installer interativo. Instalacao requer clonar repo e configurar manualmente.
- **Target State:** O AIOX tem `npx @synkra/aiox-install` que: (1) detecta projeto existente vs novo, (2) configura .claude/settings.json, (3) gera CLAUDE.md, (4) instala hooks, (5) configura squads. O SINAPSE deveria ter `npx sinapse-ai init` com experiencia similar.
- **Gap Severity:** HIGH
- **Estimated Effort:** L
- **Estimated Token Savings:** Nenhum
- **Priority:** P1

### 9.2 Gap: Onboarding Flow Inexistente

- **Current State:** Nao ha guia de primeiro uso. Usuarios precisam entender a Constitution, os agentes, os workflows, e as rules antes de comecar.
- **Target State:** Wizard de onboarding interativo que: (1) pergunta o tipo de projeto, (2) sugere squads relevantes, (3) configura rules minimas, (4) cria primeira epic/story de exemplo, (5) roda primeiro build/test.
- **Gap Severity:** HIGH
- **Estimated Effort:** L
- **Estimated Token Savings:** Nenhum
- **Priority:** P1

### 9.3 Gap: Documentacao Limitada vs AIOX (61 artigos)

- **Current State:** O SINAPSE tem docs espalhados (PRDs, architecture, guides, stories) sem indice centralizado. Nao ha site de documentacao publico.
- **Target State:** O AIOX tem 61 artigos documentados. O BMAD tem docs.bmad-method.org. SINAPSE deveria ter: site de docs (Docusaurus/Nextra), getting started guide, agent reference, workflow reference, API reference.
- **Gap Severity:** MEDIUM
- **Estimated Effort:** XL
- **Estimated Token Savings:** Nenhum
- **Priority:** P2

---

## 10. Naming, Organization & Perfectionism Gaps

### 10.1 Gap: Inconsistencia de Naming entre Projetos

- **Current State:** Agentes do framework tem personas diferentes por projeto. No global CLAUDE.md: Pixel, Litmus, Stratum, Beacon, Axis, Sync, Scope, Mosaic, Pipeline, Tensor. No project CLAUDE.md: Dex, Quinn, Aria, Morgan, Pax, River, Alex, Uma, Gage, Dara. Sao nomes DIFERENTES para os mesmos roles.
- **Target State:** Um unico set de personas por role, consistente em todos os projetos. Se a persona muda por projeto, documentar claramente o mapeamento.
- **Gap Severity:** MEDIUM
- **Estimated Effort:** S
- **Estimated Token Savings:** ~200-300 tokens (elimina tabela duplicada com nomes diferentes)
- **Priority:** P1

### 10.2 Gap: Directory Structure Inconsistente

- **Current State:** Squads usam `.sinapse/squad-{name}/` com subfolders: `agents/`, `tasks/`, `workflows/`, `knowledge-base/`, `checklists/`, `templates/`, `preferences/`. O framework usa `.sinapse-ai/development/` com: `tasks/`, `templates/`, `checklists/`, `workflows/`. Nomes de folders nao sao identicos (`knowledge-base` vs ausente no framework).
- **Target State:** Padronizar naming de directories entre squads e framework. Usar singular consistente ou plural consistente. Adicionar `knowledge-base/` ao framework se squads usam.
- **Gap Severity:** LOW
- **Estimated Effort:** S
- **Estimated Token Savings:** Nenhum
- **Priority:** P3

### 10.3 Gap: Task Naming Nao Padronizado

- **Current State:** Tasks usam mix de: `verb-noun.md` (create-next-story), `noun-verb-noun.md` (orchestrate-research-pipeline), `verb-adjective-noun.md` (validate-next-story). 1,229 tasks com naming inconsistente.
- **Target State:** Padrao unico: `{verb}-{object}-{qualifier}.md`. Exemplos: `create-story-next`, `validate-story-draft`, `orchestrate-pipeline-research`.
- **Gap Severity:** LOW
- **Estimated Effort:** XL (1,229 files para renomear — mas low priority)
- **Estimated Token Savings:** Nenhum
- **Priority:** P3

### 10.4 Gap: Knowledge Bases Nao Versionados

- **Current State:** 204 knowledge bases sem versionamento ou metadata de freshness.
- **Target State:** Cada KB com `last_verified:` date e `version:` field. Consolidation process verifica freshness periodicamente.
- **Gap Severity:** LOW
- **Estimated Effort:** M
- **Estimated Token Savings:** Nenhum
- **Priority:** P3

---

## 11. Project Scaffolding Gaps

### 11.1 Gap: Ausencia de Template de Projeto

- **Current State:** Criar um novo projeto SINAPSE requer: copiar files do repo principal, ajustar configs, configurar squads manualmente. Nao ha `sinapse init` que scaffolde um projeto novo.
- **Target State:** `sinapse init` que: (1) cria .claude/ com settings.json, CLAUDE.md, rules/, hooks/, (2) cria .sinapse-ai/ com structure minima, (3) cria docs/ com folders padrao, (4) configura .gitignore, (5) configura branch protection, (6) cria PR template.
- **Gap Severity:** HIGH
- **Estimated Effort:** L
- **Estimated Token Savings:** Nenhum
- **Priority:** P1 (blocker para adocao)

### 11.2 Gap: .gitignore Nao Padronizado

- **Current State:** Nao ha template .gitignore oficial do SINAPSE que cubra: .sinapse/handoffs/, .sinapse/scratchpad/, .env, node_modules, runtime artifacts.
- **Target State:** Template .gitignore incluido no `sinapse init` com secoes comentadas para SINAPSE runtime, Node.js, secrets, build artifacts.
- **Gap Severity:** MEDIUM
- **Estimated Effort:** S
- **Estimated Token Savings:** Nenhum
- **Priority:** P1

### 11.3 Gap: CI/CD Templates Nao Distribuidos

- **Current State:** `.sinapse-ai/infrastructure/` existe mas nao e incluido automaticamente em novos projetos.
- **Target State:** `sinapse init` pergunta se quer CI/CD e gera: GitHub Actions workflow para lint/test/build, Vercel/Railway deploy config, Dependabot config.
- **Gap Severity:** MEDIUM
- **Estimated Effort:** M
- **Estimated Token Savings:** Nenhum
- **Priority:** P2

### 11.4 Gap: Ausencia de PR Template

- **Current State:** PRs criados por @devops nao usam template padronizado persistido no repo.
- **Target State:** `.github/PULL_REQUEST_TEMPLATE.md` incluido no scaffold com: Summary, Story ID, Test Plan, Screenshots (if UI), Checklist.
- **Gap Severity:** LOW
- **Estimated Effort:** S
- **Estimated Token Savings:** Nenhum
- **Priority:** P2

---

## 12. Productization Gaps

### 12.1 Gap: Ausencia de Multi-Tenancy

- **Current State:** O SINAPSE e single-user/single-org. Nao ha conceito de "workspace" ou "team" alem de Caio/Matheus.
- **Target State:** Para ser plataforma: user accounts, workspaces, team management, shared squads, permissions per workspace.
- **Gap Severity:** MEDIUM (para productization; N/A se continuar interno)
- **Estimated Effort:** XL
- **Estimated Token Savings:** Nenhum
- **Priority:** P3

### 12.2 Gap: Ausencia de Usage Tracking/Billing

- **Current State:** O Claude Code rastreia custo internamente (cost-tracker.ts com pricing por modelo). O SINAPSE nao rastreia uso por agente, por squad, ou por task.
- **Target State:** Logging de: tokens consumidos por agente, por squad, por task. Dashboard mostrando custo por workflow. Permite identificar workflows ineficientes.
- **Gap Severity:** HIGH (para otimizacao)
- **Estimated Effort:** L
- **Estimated Token Savings:** Indireto — permite identificar e corrigir desperdicio
- **Priority:** P1

### 12.3 Gap: Ausencia de Dashboard/Observability

- **Current State:** `sinapse graph` visualiza dependencias mas nao mostra: metricas de uso, custos, performance de agentes, taxa de sucesso de tasks, tempo medio por workflow.
- **Target State:** Dashboard (CLI ou web) mostrando: cost per session, agent utilization, story velocity, QA pass rate, compaction frequency, memory health.
- **Gap Severity:** MEDIUM
- **Estimated Effort:** L
- **Estimated Token Savings:** Indireto
- **Priority:** P2

### 12.4 Gap: Ausencia de Marketplace de Squads

- **Current State:** Squads sao distribuidos no npm package principal. Nao ha como instalar squads individuais ou squads de terceiros.
- **Target State:** Registry de squads (`sinapse squad install @sinapse/squad-research`). Permite: squads oficiais, squads community, squads privados por org.
- **Gap Severity:** LOW (premature para o momento)
- **Estimated Effort:** XL
- **Estimated Token Savings:** Nenhum
- **Priority:** P3

### 12.5 Gap: Ausencia de Squad Creator Meta-Agent

- **Current State:** Squads sao criados manualmente seguindo templates. O AIOX tem um "Squad Creator" meta-agent que gera squads a partir de descricao.
- **Target State:** Ja existe parcialmente (squad-creator Craft no manifest). Formalizar como skill disponivel para @sinapse-orqx.
- **Gap Severity:** LOW
- **Estimated Effort:** M
- **Estimated Token Savings:** Nenhum
- **Priority:** P3

---

## Matriz de Prioridade Consolidada

### P0 — Imediato (bloqueiam eficacia atual)

| # | Gap | Severity | Effort | Token Savings |
|---|-----|----------|--------|--------------|
| 1.2 | CLAUDE.md excede limites (truncation silenciosa) | CRITICAL | M | Corrige truncation |
| 1.3 | Rules sem path filtering | CRITICAL | S | ~12,000/turn |
| 1.4 | Duplicacao global/project CLAUDE.md | HIGH | M | ~750-1,000/turn |
| 1.5 | Duplicacao rules/CLAUDE.md | HIGH | M | ~1,250/turn |

**Economia P0 total: ~14,000-14,250 tokens/turn**

### P1 — Proximo sprint

| # | Gap | Severity | Effort | Token Savings |
|---|-----|----------|--------|--------------|
| 1.6 | Conteudo irrelevante nos CLAUDE.md | MEDIUM | S | ~500-750/turn |
| 2.1 | Separacao static/dynamic | HIGH | M | ~60% custo instrucoes |
| 3.1 | Memory consolidation automatica | HIGH | L | Indireto |
| 3.2 | Memory relevance scoring | HIGH | L | ~50-73% memoria |
| 3.3 | Memory size limits | MEDIUM | S | Prevencao |
| 3.4 | Memory as hints | MEDIUM | S | Indireto |
| 4.1 | Pipeline rigido vs adaptativo | HIGH | M | ~5-10K/task |
| 5.1 | Custo de delegacao via sub-agent | HIGH | M | ~15K/delegacao |
| 6.1 | Ausencia de skills system | HIGH | L | ~14,000/turn |
| 7.3 | Auto-sync via SessionStart hook | HIGH | M | Indireto |
| 8.2 | Secret scanning incompleto | MEDIUM | M | Nenhum |
| 8.4 | Pre-deploy gate nao automatizado | HIGH | L | Nenhum |
| 9.1 | Installer NPX | HIGH | L | Nenhum |
| 9.2 | Onboarding flow | HIGH | L | Nenhum |
| 10.1 | Naming inconsistente de agentes | MEDIUM | S | ~200-300/turn |
| 11.1 | Template de projeto | HIGH | L | Nenhum |
| 11.2 | .gitignore padronizado | MEDIUM | S | Nenhum |
| 12.2 | Usage tracking | HIGH | L | Indireto |

### P2 — Backlog

| # | Gap | Severity | Effort |
|---|-----|----------|--------|
| 2.2 | Document sharding | MEDIUM | L |
| 2.3 | Deferred tool pattern | MEDIUM | M |
| 3.5 | Contradiction resolution | LOW | M |
| 4.2 | Execution modes formais | MEDIUM | M |
| 4.3 | Spec pipeline pruning | LOW | S |
| 5.2 | 175 agentes → skills | MEDIUM | L |
| 5.3 | Handoff protocol verboso | LOW | S |
| 5.5 | Cache sharing entre agentes | MEDIUM | M |
| 6.3 | Skills de verificacao/debug | MEDIUM | M |
| 7.1 | Hooks em lifecycle events adicionais | MEDIUM | M |
| 8.1 | Bash parser com AST | HIGH | XL |
| 8.3 | CodeRabbit automation | MEDIUM | M |
| 9.3 | Documentacao publica | MEDIUM | XL |
| 11.3 | CI/CD templates | MEDIUM | M |
| 11.4 | PR template | LOW | S |
| 12.3 | Dashboard/observability | MEDIUM | L |

### P3 — Futuro

| # | Gap | Severity | Effort |
|---|-----|----------|--------|
| 2.4 | System reminders contextuais | LOW | L |
| 4.4 | Story quality scoring automatico | LOW | M |
| 5.4 | Agent selection decision tree | LOW | S |
| 6.2 | Command optimization | LOW | S |
| 7.2 | Hook de budget de tokens | LOW | M |
| 7.4 | Hook performance monitoring | LOW | S |
| 10.2 | Directory structure consistente | LOW | S |
| 10.3 | Task naming padronizado | LOW | XL |
| 10.4 | KB versionamento | LOW | M |
| 12.1 | Multi-tenancy | MEDIUM | XL |
| 12.4 | Marketplace de squads | LOW | XL |
| 12.5 | Squad creator meta-agent | LOW | M |

---

## Proximo Passo Recomendado

**Sprint 0 (P0) — "Token Diet":**

1. Reescrever CLAUDE.md global para <= 4,000 chars (eliminar duplicacao, manter apenas global rules)
2. Reescrever CLAUDE.md project para <= 4,000 chars (eliminar duplicacao, manter apenas project-specific)
3. Adicionar `paths:` frontmatter a TODAS as 18 rules sem filtering
4. Eliminar duplicacao entre rules e CLAUDE.md
5. Verificar que total de CLAUDE.md files <= 12,000 chars

**Resultado esperado do Sprint 0:** Reducao de ~45% no overhead por turn, eliminacao de truncation silenciosa, melhora na aderencia do modelo as instrucoes (instrucoes criticas nao sao mais cortadas).

---

*Analise compilada 2026-04-04 por Prism (Research Orchestrator) com dados de Sage (Deep Researcher).*
*Fontes: Claude Code leaked source, claw-code, AIOX-Core, BMAD Method, audit interno SINAPSE.*
