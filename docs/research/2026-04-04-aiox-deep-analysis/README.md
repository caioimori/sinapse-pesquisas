# AIOX-Core Deep Analysis para Melhoria do SINAPSE-AI

> **Data:** 2026-04-04
> **Analista:** Prism (Research Orchestrator) via squad-research
> **Fontes:** GitHub (SynkraAI/aiox-core), documentacao publica (aiox.academialendaria.ai)
> **Versao AIOX analisada:** 5.0.3 (MIT License)
> **Objetivo:** Extrair TUDO que seja util para evolucao do SINAPSE-AI

---

## Indice

1. [Visao Geral do AIOX](#1-visao-geral-do-aiox)
2. [Sistema de Memoria (Deep Dive)](#2-sistema-de-memoria-deep-dive)
3. [Sistema de Workflows (Deep Dive)](#3-sistema-de-workflows-deep-dive)
4. [Sistema de Agentes (Deep Dive)](#4-sistema-de-agentes-deep-dive)
5. [Quality Gates (Deep Dive)](#5-quality-gates-deep-dive)
6. [Arquitetura do Sistema (Deep Dive)](#6-arquitetura-do-sistema-deep-dive)
7. [Sistema de Squads (Deep Dive)](#7-sistema-de-squads-deep-dive)
8. [Seguranca (Deep Dive)](#8-seguranca-deep-dive)
9. [Distribuicao NPM e CLI](#9-distribuicao-npm-e-cli)
10. [Constitution e Principios](#10-constitution-e-principios)
11. [Autonomous Development Engine (ADE)](#11-autonomous-development-engine-ade)
12. [Diferencas-Chave AIOX vs SINAPSE](#12-diferencas-chave-aiox-vs-sinapse)
13. [Recomendacoes para SINAPSE-AI](#13-recomendacoes-para-sinapse-ai)

---

## 1. Visao Geral do AIOX

**Nome completo:** AIOX Squad - Artificial Intelligence Orchestration eXperience

**Descricao:** Framework open source de orquestracao de IA que coordena agentes especializados, workflows e experiencia CLI First para qualquer dominio.

**Premissa central:** "Devolvendo as pessoas o poder de criar" -- framework que devolve o controle a quem tem coragem de construir.

### Duas Inovacoes-Chave

1. **Planejamento Agentico:** Agentes dedicados (analyst, pm, architect) colaboram para criar PRDs e documentos de arquitetura detalhados e consistentes. Engenharia avancada de prompts + refinamento human-in-the-loop produzem especificacoes que vao muito alem da geracao generica de tarefas.

2. **Desenvolvimento Contextualizado por Engenharia:** O agente SM (Scrum Master) transforma os planos detalhados em historias de desenvolvimento hiperdetalhadas que contem TUDO que o agente dev precisa -- contexto completo, detalhes de implementacao e orientacao arquitetural incorporada diretamente nos arquivos de historias.

**Resultado:** Elimina tanto a inconsistencia de planejamento quanto a perda de contexto -- os maiores problemas no desenvolvimento assistido por IA.

### Premissa Arquitetural: CLI First

```
CLI First --> Observability Second --> UI Third
```

| Camada | Prioridade | Foco | Exemplos |
|--------|-----------|------|----------|
| CLI | Maxima | Onde a inteligencia vive. Toda execucao, decisoes e automacao | Agentes, workflows, comandos |
| Observability | Secundaria | Observar e monitorar o CLI em tempo real | Dashboard SSE, logs, metricas |
| UI | Terciaria | Gestao pontual e visualizacoes | Kanban, settings, story management |

**Principios derivados:**
- A CLI e a fonte da verdade -- dashboards apenas observam
- Funcionalidades novas devem funcionar 100% via CLI antes de ter UI
- A UI nunca deve ser requisito para operacao do sistema
- Observabilidade serve para entender o que o CLI esta fazendo, nao para controla-lo

### Compatibilidade Multi-IDE

| IDE/CLI | Paridade de Hooks vs Claude | Impacto Pratico |
|---------|---------------------------|-----------------|
| Claude Code | Completa (referencia) | Automacao maxima |
| Gemini CLI | Alta (eventos nativos) | Cobertura forte |
| Codex CLI | Parcial/limitada | Depende de AGENTS.md, /skills, MCP |
| Cursor | Sem lifecycle hooks | Menor automacao, foco em regras + MCP |
| GitHub Copilot | Sem lifecycle hooks | Foco em instrucoes de repositorio + MCP |
| AntiGravity | Workflow-based | Integracao por workflows, nao hooks |

### Ativacao de Agentes por IDE

- **Claude Code:** `/agent-name`
- **Gemini CLI:** `/aiox-menu` --> `/aiox-<agent>`
- **Codex CLI:** `/skills` --> `aiox-<agent-id>`
- **Cursor/Copilot/AntiGravity:** limites e workarounds em `docs/ide-integration.md`

---

## 2. Sistema de Memoria (Deep Dive)

### Arquitetura de Duas Camadas Independentes

O sistema de memoria do AIOX opera em **duas camadas independentes que NAO se comunicam entre si**:

#### Camada 1: Claude Code Nativo
- Gerencia: `CLAUDE.md` (3 niveis de hierarquia), `rules/*.md`, `MEMORY.md` (primeiras 200 linhas injetadas no system prompt)
- Storage: `~/.claude/projects/.../memory/` e `.claude/agent-memory/`
- Auto-salva transcripts como arquivos `.jsonl`
- **NAO** tem session-digest ou memory-flush automatico ao fechar

#### Camada 2: AIOX Framework
- Gerencia: Gotchas, snapshots, timeline, session state via modulos JavaScript
- Storage: `.aiox/` directory (gitignored, estado runtime)
- Fornece continuidade inter-agente e crash recovery

### Modulos de Memoria Core

#### Gotchas Memory (`gotchas-memory.js`)
- Auto-captura erros apos 3 ocorrencias identicas em 24 horas
- Entrada manual via comando `*gotcha {description}`
- Persiste em `.aiox/gotchas.json`, `.aiox/gotchas.md`, `.aiox/error-tracking.json`
- Injeta gotchas relevantes no contexto de task via `getContextForTask()`

#### Context Snapshot (`context-snapshot.js`)
- Captura: git state, story ID, agente, working directory
- Limites: max 50 snapshots, auto-cleanup apos 7 dias
- Habilita recuperacao mid-task

#### File Evolution Tracker
- Rastreia modificacoes de arquivo ao longo do tempo
- Detecta potenciais conflitos via `detectDrift()`
- Persiste indice de evolucao em `.aiox/file-evolution/`

#### Timeline Manager (fachada unificada)
- Agrega eventos de snapshots, file evolution, entradas manuais
- Normaliza e ordena cronologicamente
- Limites: 5000 entradas, retencao de 90 dias
- Auto-sync a cada 60 segundos

### Pipeline de Carregamento de Contexto de Sessao (3-Tier)

| Tier | Criticidade | Tempo | O que carrega |
|------|------------|-------|---------------|
| Tier 1 | Critical | 80ms | Configuracao do agente via `AgentConfigLoader` |
| Tier 2 | High | 120ms (paralelo) | Permission mode + Git state detection |
| Tier 3 | Best-effort | 180ms (paralelo) | Session type, project status, branch, modified files, active story, previous agent handoff |

### Memory Intelligence System (MIS) -- Modulo Pro

O MIS e a camada inteligente de memoria, implementada no repositorio privado `aiox-pro` seguindo modelo "Open Core". O `aiox-core` fornece apenas extension points.

#### Arquitetura de 4 Camadas

**Camada 1: Capture**
- Intercepta eventos de sessao via Claude Code hooks
- Extrai conhecimento estruturado
- Componentes:
  - **PreCompact Hook:** Dispara quando contexto se aproxima dos limites, triggering `session-digest.js`
  - **Stop Hook:** Executa flush final quando sessao encerra
  - **PostToolUseFailure Hook:** Captura gotcha aprimorada em falhas de tool
  - **Extractors:** Session digest, gotcha, e correction extractors classificam memorias por tier, sector e scope

**Camada 2: Storage**
- Memorias armazenadas como Markdown com YAML frontmatter em `.aiox/memories/`
- Formato do frontmatter:

```yaml
id: mem-YYYY-MM-DD-NNN
type: episodic | semantic | procedural | reflective
tier: session | daily | durable
agent: shared | dev | qa | architect
confidence: 0.0-1.0
attention_score: 0.0-1.0 (calculado)
source: user-correction | session-digest | auto-gotcha | manual | heuristic
related_memories: [id-list]
evidence_count: integer
```

- Estrategia dual-storage: memoria nativa Claude Code + storage indexado MIS para retrieval inteligente

**Camada 3: Retrieval**
- Progressive disclosure minimiza consumo de tokens em 3 niveis:
  1. **Index Scan** (~50 tokens): Busca por metadata, retorna titulos e IDs
  2. **Context Retrieval** (~200 tokens): Resumos e evidencias de memorias correspondentes
  3. **Full Detail** (~1000+ tokens): Conteudo completo da memoria sob demanda

- **Token Budget:** Core-only: ~500 tokens; Com pro + progressive retrieval: ~2,700 tokens (~73% reducao)

**Camada 4: Evolution**
- Descoberta automatica de regras e validacao:
  - **Self-learner:** Extrai heuristicas de correcoes do usuario e padroes repetidos
  - **Confidence Scoring:** Formula combina evidence count, recency e access frequency
  - **Rule Proposer:** Gera propostas de mudanca apenas quando confidence excede 0.9 com 5+ evidencias
  - **Gate:** Todas as modificacoes requerem aprovacao explicita do usuario; rejeicoes diminuem confidence

### Attention Scoring e Tiers HOT/WARM/COLD

**Formula:**
```
attention_score = base_relevance x recency_factor x access_modifier x confidence
```

**Atribuicao de Tier:**

| Tier | Score | Comportamento | Token Budget |
|------|-------|--------------|-------------|
| HOT | > 0.7 | Sincronizado com MEMORY.md nativo, sempre disponivel | ~500 tokens |
| WARM | 0.3-0.7 | Carregado durante inicializacao do pipeline se budget permitir | ~2000 tokens |
| COLD | < 0.3 | Acessivel apenas via comando explicito `*recall` | Sob demanda |
| ARCHIVE | < 0.1 (90+ dias) | Movido para `.old/` directory | N/A |

**Taxas de Decay por Tier:**

| Tier de Duracao | Taxa de Decay |
|-----------------|--------------|
| Session | 0.5/dia |
| Daily | 0.1/dia |
| Durable | 0.01/dia |

### Cognitive Sectors (4 Tipos de Memoria)

| Sector | Descricao | TTL Base | Notas |
|--------|-----------|----------|-------|
| Episodic | "O que aconteceu" | 7 dias | Access modifier aplicado |
| Semantic | "O que sabemos" | 365 dias | Fatos permanecem validos indefinidamente |
| Procedural | "Como fazer" | 30 dias | Modificado por timestamp de ultimo uso |
| Reflective | "O que aprendemos" | Infinito | Insights meta-cognitivos persistem |

### Agent Scoping -- Preferencias de Sector por Agente

| Agente | Sectors Preferidos | Foco |
|--------|-------------------|------|
| @dev | Procedural + Semantic | Padroes, APIs |
| @qa | Reflective + Episodic | Licoes, resultados de teste |
| @architect | Semantic + Reflective | Arquitetura, decisoes de design |
| @pm/@po | Episodic + Semantic | Eventos, requisitos |
| @sm | Procedural + Episodic | Processo, eventos de sprint |

### Self-Learning e Evolution

O sistema aprende atraves de 3 mecanismos:

1. **Correction Tracking:** Correcoes do usuario capturadas com confidence inicial de 0.3
2. **Pattern Recognition:** Sinais repetidos em 5+ sessoes aumentam confidence
3. **Heuristic Extraction:** Quando confidence excede threshold, propostas sao geradas

**Fluxo de exemplo:** Correcao do usuario recebe 0.3 confidence --> Apos aparecer 5+ vezes em sessoes diferentes, confidence atinge 0.95 --> Sistema propoe adicionar regra ao CLAUDE.md --> Aprovacao do usuario aplica; rejeicao diminui confidence e registra reasoning.

### Memory Loader API

6 metodos primarios:

1. `loadForAgent(agentId, options)` -- Metodo primario de ativacao
2. `queryMemories(agentId, options)` -- Filtragem avancada
3. `getHotMemories(agentId, options)` -- Apenas alta atencao (score > 0.7)
4. `getWarmMemories(agentId, options)` -- Atencao moderada (0.3-0.7)
5. `searchByTags(agentId, tags, options)` -- Retrieval por tags
6. `getRecentMemories(agentId, days, options)` -- Filtragem por tempo

### Token Budget Management

- **Default:** 2000 tokens por ativacao
- **Custom:** Configuravel em agent config (`memoryBudget: 3000`)
- O sistema nunca excede o limite configurado e deixa buffer para overhead do sistema

### Performance Targets

- Memory load (Tier 2): < 200ms tipico
- Timeout safeguard: 500ms max
- Metricas rastreadas: `result.metrics.loaders.memories` (status, duration, timestamps)

### Persistent Storage Map

```
.aiox/
  session-state.json          (TTL: 1 hora)
  gotchas.json + .md
  error-tracking.json
  snapshots/{id}.json         (Max 50, 7 dias)
  timeline/unified-timeline.json
  file-evolution/evolution-index.json

docs/stories/
  .session-state.yaml         (Epic-level, no TTL)
```

### Crash Detection e Recovery

`SessionState.detectCrash()` dispara se:
- Ultima atividade > 30 minutos atras
- Ultima acao NAO foi PAUSE/COMPLETED/ABORT

Opcoes de recuperacao: CONTINUE, REVIEW, RESTART, DISCARD

### Gaps Criticos Identificados no Sistema de Memoria

1. **Sem session-digest:** Conhecimento perdido quando sessao do Claude Code fecha
2. **Sem memory-flush:** Modulos AIOX nao tem persistencia automatica EOF
3. **Camadas desconectadas:** Claude MEMORY.md e AIOX gotchas nunca sincronizam
4. **Hooks nao conectados:** `session-start.js` mapeia para null (sem evento Claude Code); `session-end.js` mapeia para Stop mas nao configurado

### Mapeamento de Eventos (Abstracao Cross-CLI)

| Evento AIOX | Claude Code | Gemini | Status |
|-------------|-------------|--------|--------|
| sessionStart | null | SessionStart | Gemini only |
| sessionEnd | Stop (unmapped) | SessionEnd | Gemini only |
| beforeAgent | PreToolUse | BeforeAgent | Ambos |
| afterTool | PostToolUse | AfterTool | Ambos |

---

## 3. Sistema de Workflows (Deep Dive)

### Lista Completa dos 14+ Workflows

#### Greenfield Workflows (Do zero)

| # | Workflow | Descricao | Agentes |
|---|---------|-----------|---------|
| 1 | Story Development Cycle (SDC) | Workflow primario de 4 fases: SM --> PO --> Dev --> QA | River, Pax, Dex, Quinn |
| 2 | Greenfield Full-Stack | Apps full-stack do zero com 9 agentes | Todos |
| 3 | Greenfield Service | Novos servicos backend do zero | Subset backend |
| 4 | Greenfield UI | Novas interfaces frontend do zero | Subset frontend |

#### Brownfield Workflows (Projetos existentes)

| # | Workflow | Descricao | Agentes |
|---|---------|-----------|---------|
| 5 | Brownfield Discovery | Avaliacao de divida tecnica em 10 fases | Aria, Dara, Uma, Quinn |
| 6 | Brownfield Full-Stack | Adicionar features a projetos full-stack existentes | Todos |
| 7 | Brownfield Service | Adicionar servicos a projetos existentes | Subset backend |
| 8 | Brownfield UI | Adicionar UI a projetos existentes | Subset frontend |

#### Quality e Requirements Workflows

| # | Workflow | Descricao |
|---|---------|-----------|
| 9 | QA Loop | Ciclo iterativo review-fix com max 5 iteracoes |
| 10 | Spec Pipeline | Requisitos vagos --> specs executaveis em 6 fases |
| 11 | Design System Build & Quality | Build de design system com controles de qualidade |
| 12 | Epic Orchestration | Orquestracao de epics e multi-story delivery |
| 13 | Auto-Worktree | Isolamento de branches via Git worktrees |
| 14 | Development Cycle | Ciclo geral de desenvolvimento |

#### Arquivos de Workflow (GitHub)

```
.aiox-core/development/workflows/
  auto-worktree.yaml
  brownfield-discovery.yaml
  brownfield-fullstack.yaml
  brownfield-service.yaml
  brownfield-ui.yaml
  design-system-build-quality.yaml
  development-cycle.yaml
  epic-orchestration.yaml
  greenfield-fullstack.yaml
  greenfield-service.yaml
  greenfield-ui.yaml
  qa-loop.yaml
  spec-pipeline.yaml
  story-development-cycle.yaml
```

### Story Development Cycle (SDC) -- Detalhamento Completo

O SDC e o workflow central do AIOX, automatizando o processo completo de desenvolvimento de story em 4 fases.

#### Fase 1: Story Creation (@sm - River)
- **Task:** `create-next-story.md`
- **Output Status:** Draft
- **Criterios de sucesso:**
  - Titulo descritivo
  - Acceptance criteria definidos
  - Escopo claro
  - Dependencias identificadas

#### Fase 2: Story Validation (@po - Pax)
- **Task:** `validate-next-story.md`
- **Input:** Output da Fase 1
- **10-Point Checklist:**
  1. Titulo claro e objetivo
  2. Descricao completa
  3. Acceptance criteria testaveis (formato Given/When/Then preferido)
  4. Escopo bem definido (listas IN/OUT)
  5. Dependencias mapeadas
  6. Estimativa de complexidade
  7. Clareza de valor de negocio
  8. Riscos documentados
  9. Criterios de Done claros
  10. Alinhamento com PRD/Epic
- **Decisao:** GO (>=7 pontos) ou NO-GO (fixes obrigatorios listados)

#### Fase 3: Implementation (@dev - Dex)
- **Task:** `dev-develop-story.md` + CodeRabbit Self-Healing
- **Modos de execucao:**
  - **YOLO:** Autonomo (0-1 prompts), gera decision logs
  - **Interactive:** Default, com checkpoints (5-10 prompts)
  - **Pre-Flight:** Planejamento completo upfront (10-15 prompts)
- **Workflow interno:** Read task --> Implement + subtasks --> Write tests --> Validate --> Mark complete --> Update file list --> Repeat
- **CodeRabbit Self-Healing Loop:**
  - Executa review automatizado
  - Max 2 iteracoes em issues CRITICAL
  - Documenta issues HIGH
  - Para em findings CRITICAL persistentes
- **Transicoes:** Ready --> In Progress --> In Review

#### Fase 4: QA Review (@qa - Quinn)
- **Task:** `qa-gate.md`
- **7 Quality Gate Checks:**
  1. Code review standards
  2. Unit test adequacy
  3. Acceptance criteria fulfillment
  4. Regression prevention
  5. Performance compliance
  6. OWASP basics
  7. Documentation updates
- **Decisoes:**
  - **PASS:** Todos checks passaram, sem issues HIGH --> Status: Done
  - **CONCERNS:** Issues nao-bloqueantes presentes --> Approve com observacoes
  - **FAIL:** Issues HIGH/CRITICAL --> Retorna ao Dev (In Progress)
  - **WAIVED:** Issues explicitamente aceitas --> Approve com documentacao

#### Status Flow

```
Draft --> Ready --> In Progress --> In Review --> Done
         ^                              |
         |_____ (PO rejects) ___________|
                                        |
In Progress <-- (QA rejects) ----------|
```

### QA Loop -- Ciclo Iterativo

```
@qa review --> verdict --> @dev fixes --> re-review (max 5 iteracoes)
```

**Comandos:**
- `*qa-loop {storyId}` -- Iniciar loop
- `*qa-loop-review` -- Resumir do review
- `*qa-loop-fix` -- Resumir do fix
- `*stop-qa-loop` -- Pausar, salvar estado
- `*resume-qa-loop` -- Resumir do estado
- `*escalate-qa-loop` -- Forcar escalacao

**Verdicts:** APPROVE, REJECT, BLOCKED

**Triggers de escalacao:** `max_iterations_reached`, `verdict_blocked`, `fix_failure`, `manual_escalate`

### Spec Pipeline -- Pre-Implementation (6 Fases)

| Fase | Agente | Output | Skip Se |
|------|--------|--------|---------|
| 1. Gather | @pm | requirements.json | Nunca |
| 2. Assess | @architect | complexity.json | source=simple |
| 3. Research | @analyst | research.json | Classe SIMPLE |
| 4. Write Spec | @pm | spec.md | Nunca |
| 5. Critique | @qa | critique.json | Nunca |
| 6. Plan | @architect | implementation.yaml | Se APPROVED |

**Classes de complexidade:**

| Score | Classe | Fases |
|-------|--------|-------|
| <= 8 | SIMPLE | gather --> spec --> critique (3) |
| 9-15 | STANDARD | Todas 6 fases |
| >= 16 | COMPLEX | 6 fases + ciclo de revisao |

**5 Dimensoes de complexidade (scored 1-5):**
- Scope (arquivos afetados)
- Integration (APIs externas)
- Infrastructure (mudancas necessarias)
- Knowledge (familiaridade do time)
- Risk (nivel de criticidade)

### Brownfield Discovery -- 10 Fases

**Coleta de dados (Fases 1-3):**
- Fase 1: @architect --> `system-architecture.md`
- Fase 2: @data-engineer --> `SCHEMA.md` + `DB-AUDIT.md`
- Fase 3: @ux-design-expert --> `frontend-spec.md`

**Draft e Validacao (Fases 4-7):**
- Fase 4: @architect --> `technical-debt-DRAFT.md`
- Fase 5: @data-engineer --> `db-specialist-review.md`
- Fase 6: @ux-design-expert --> `ux-specialist-review.md`
- Fase 7: @qa --> `qa-review.md` (QA Gate)

**Finalizacao (Fases 8-10):**
- Fase 8: @architect --> `technical-debt-assessment.md` (final)
- Fase 9: @analyst --> `TECHNICAL-DEBT-REPORT.md` (executivo)
- Fase 10: @pm --> Epic + stories prontas para desenvolvimento

---

## 4. Sistema de Agentes (Deep Dive)

### Todos os 12 Agentes

#### Agentes Meta

| ID | Nome | Archetype | Role |
|----|------|-----------|------|
| @aios-master | Orion | Orchestrator | Orquestrador principal e governanca do framework |
| @aios-orchestrator | (unnamed) | Orchestrator | Orquestrador de workflow e coordenacao de equipe |

#### Agentes de Planejamento (Interface Web)

| ID | Nome | Archetype | Role |
|----|------|-----------|------|
| @analyst | Atlas | Researcher | Pesquisa de mercado e analise competitiva |
| @pm | Morgan | Product Manager | PRDs, epics e spec pipeline |
| @architect | Aria | Designer | Arquitetura de sistema e decisoes tecnicas |
| @ux-design-expert | Uma | Designer | UX/UI Design e Design Systems |

#### Agentes de Desenvolvimento (IDE)

| ID | Nome | Archetype | Role |
|----|------|-----------|------|
| @sm | River | Facilitator | Scrum Master, stories e sprint planning |
| @dev | Dex | Builder | Full Stack Developer, implementacao |
| @qa | Quinn | Guardian | Quality Assurance, review e testes |
| @po | Pax | Balancer | Product Owner, backlog e validacao |
| @data-engineer | Dara | Engineer | Schemas, RLS, migrations, queries |
| @devops | Gage | Deployer | CI/CD, git push (EXCLUSIVO), releases |

#### Agente Utilitario

| ID | Nome | Role |
|----|------|------|
| @squad-creator | (unnamed) | Criacao de novos squads de agentes |

### Formato de Definicao de Agente

Agentes sao definidos como arquivos Markdown em `.aiox-core/development/agents/`:

```
.aiox-core/development/agents/
  aiox-master.md
  analyst.md
  analyst/          (subdiretorio com tasks?)
  architect.md
  architect/
  data-engineer.md
  data-engineer/
  dev.md
  dev/
  devops.md
  devops/
  pm.md
  pm/
  po.md
  po/
  qa.md
  qa/
  sm.md
  sm/
  squad-creator.md
  ux-design-expert.md
  ux/
```

### Comandos por Agente

#### @sm (River) - Scrum Master
- `*draft` -- Criar story
- `*story-checklist` -- Checklist de story

#### @po (Pax) - Product Owner
- `*validate-story-draft` -- Validar story com 10-point checklist
- `*backlog-review` -- Revisar backlog

#### @dev (Dex) - Developer
- `*develop` -- Implementar story
- `*run-tests` -- Rodar testes
- `*apply-qa-fixes` -- Aplicar fixes de QA
- `*execute-subtask` -- Executar subtask (ADE)
- `*track-attempt` -- Rastrear tentativa
- `*rollback` -- Rollback
- `*capture-insights` -- Capturar insights
- `*list-gotchas` -- Listar gotchas
- `*apply-qa-fix` -- Aplicar fix de QA

#### @qa (Quinn) - Quality
- `*review` -- Revisar codigo
- `*gate` -- Quality gate
- `*code-review` -- Code review
- `*critique-spec` -- Criticar spec (ADE)
- `*review-build` -- Revisar build (ADE)
- `*request-fix` -- Solicitar fix
- `*verify-fix` -- Verificar fix

#### @pm (Morgan) - Product Manager
- `*gather-requirements` -- Coletar requisitos (ADE)
- `*write-spec` -- Escrever spec (ADE)

#### @architect (Aria) - Architect
- `*assess-complexity` -- Avaliar complexidade (ADE)
- `*create-plan` -- Criar plano (ADE)
- `*create-context` -- Criar contexto (ADE)
- `*map-codebase` -- Mapear codebase (ADE)

#### @analyst (Atlas) - Analyst
- `*research-deps` -- Pesquisar dependencias (ADE)
- `*extract-patterns` -- Extrair padroes (ADE)

#### @devops (Gage) - DevOps
- `*create-worktree` -- Criar worktree (ADE)
- `*list-worktrees` -- Listar worktrees
- `*merge-worktree` -- Merge worktree
- `*cleanup-worktrees` -- Cleanup worktrees
- `*inventory-assets` -- Inventariar assets
- `*analyze-paths` -- Analisar paths
- `*migrate-agent` -- Migrar agente
- `*migrate-batch` -- Migracao em lote

### Delegation Matrix

| Autoridade | Agente Exclusivo |
|-----------|-----------------|
| git push | @devops |
| PR creation | @devops |
| Release/Tag | @devops |
| Story creation | @sm, @po |
| Architecture decisions | @architect |
| Quality verdicts | @qa |

---

## 5. Quality Gates (Deep Dive)

### Sistema de 3 Camadas

#### Camada 1: Pre-commit (~30s)

- Checks automatizados locais antes do commit
- **Tools:** ESLint, Jest, TypeScript
- **Configuracao:**
  - Lint: fail on 'error', timeout 60s
  - Test: timeout 300s, cobertura minima 80%
  - Typecheck: timeout 120s
- Roda com fail-fast habilitado

#### Camada 2: PR Automation (~5min)

- Code review assistido por IA em pull requests
- **Tools:** CodeRabbit (AI code review) + Quinn/@qa (agente QA automatizado)
- **Severidades:**
  - CRITICAL: bloqueia
  - HIGH: avisa
  - MEDIUM: documenta
  - LOW: ignora
- CodeRabbit timeout: 15 minutos
- Foco: seguranca, erros de logica, performance, issues arquiteturais
- Quinn reviews: test coverage, edge cases, error handling, acceptance criteria

#### Camada 3: Human Review (Variavel)

- Sign-off estrategico por reviewers designados
- **Estrategias de assignment:** auto, manual, ou round-robin
- **Default reviewer:** @architect
- **Sign-off expiry:** 24 horas
- **Checklist cobre:** Arquitetura, Seguranca, Qualidade, Business

### CodeRabbit Self-Healing Loop

1. Executa review automatizado pos-commit/pre-push
2. Identifica issues por severidade
3. Issues CRITICAL: agente @dev tenta corrigir automaticamente (max 2 iteracoes)
4. Issues HIGH: documentadas para revisao
5. Issues CRITICAL persistentes: workflow para

### CLI Commands

```bash
aiox qa run --layer=[1-3]   # Executar camada especifica
aiox qa status               # Verificar status
aiox qa report               # Gerar relatorios
aiox qa configure            # Ajustar configuracoes
```

---

## 6. Arquitetura do Sistema (Deep Dive)

### Estrutura de Diretorios (Root)

```
aiox-core/
  .aiox-core/            # Framework core (L1/L2)
    cli/
    constitution.md
    core-config.yaml
    core/                # Core modules
      code-intel/
      config/
      doctor/
      elicitation/
      events/
      execution/
      graph-dashboard/
      health-check/
      ideation/
      ids/
      manifest/
      mcp/
      memory/
      migration/
      orchestration/
      permissions/
      quality-gates/
      registry/
      session/
      synapse/
      ui/
      utils/
    data/
    development/         # Agents, tasks, workflows, templates
      agents/            # 12 agent definitions
      agent-teams/
      checklists/
      data/
      scripts/
      tasks/
      templates/
      workflows/         # 14+ workflow definitions
    docs/
    elicitation/
    hooks/
    infrastructure/
    install-manifest.yaml
    manifests/
    monitor/
    presets/
    product/
    project-config.yaml
    quality/
    schemas/
    scripts/
    utils/
    workflow-intelligence/
  .aiox/                 # Runtime state (gitignored)
  .claude/               # Claude Code integration
  .codex/                # Codex CLI integration
  .cursor/               # Cursor integration
  .gemini/               # Gemini CLI integration
  .antigravity/          # AntiGravity integration
  .docker/               # Docker configs
  .github/               # GitHub Actions
  .husky/                # Git hooks
  bin/                   # CLI entry points
    aiox.js
    aiox-core --> aiox.js
    aiox-minimal.js
    aiox-graph.js
    aiox-ids.js
    aiox-init.js
    modules/
    utils/
  docs/                  # Documentation
  packages/              # Monorepo workspace packages
  pro/                   # Pro module
  scripts/               # Build/utility scripts
  squads/                # Community squads
    _example/
    claude-code-mastery/
  tests/                 # Test suites
```

### Modelo de Camadas (L1-L4)

| Camada | Mutabilidade | Paths | Notas |
|--------|-------------|-------|-------|
| L1 Framework Core | NEVER modify | `.aiox-core/core/`, `.aiox-core/constitution.md`, `bin/aiox.js` | Protegido por deny rules |
| L2 Framework Templates | NEVER modify | `.aiox-core/development/tasks/`, `templates/`, `checklists/`, `workflows/`, `infrastructure/` | Extend-only |
| L3 Project Config | Mutable (excecoes) | `.aiox-core/data/`, `agents/*/MEMORY.md`, `core-config.yaml` | Allow rules permitem |
| L4 Project Runtime | ALWAYS modify | `docs/stories/`, `packages/`, `squads/`, `tests/` | Trabalho do projeto |

### Tech Stack Completo

#### Runtime e Linguagens
- **Node.js 18.0.0+** (Active LTS, v20+ recomendado)
- **npm 9.0.0+**
- **JavaScript ES2022** (CommonJS)
- **TypeScript 5.9.3** (apenas .d.ts, nao compilado)
- ESM migration e full TypeScript planejados para Q2 2026

#### CLI e UX Interativo
- `@clack/prompts` (^0.11.0) -- Prompts modernos com animacoes
- `chalk` (^4.1.2) -- Estilizacao de terminal
- `picocolors` (^1.1.1) -- Alternativa leve (14x menor)
- `ora` (^5.4.1) -- Spinners de terminal
- `commander` (^12.1.0) -- Framework CLI com parsing de argumentos
- `inquirer` (^8.2.6) -- Prompts interativos e wizards

#### File System e Processamento
- `fs-extra` (^11.3.2) -- Operacoes de arquivo baseadas em Promise
- `glob` (^10.4.4) -- Matching de arquivo baseado em pattern
- `js-yaml` (^4.1.0) -- Parsing YAML
- `handlebars` (^4.7.8) -- Templates
- `@kayvan/markdown-tree-parser` (^1.5.0) -- Markdown para AST

#### Validacao e Versionamento
- `validator` (^13.15.15) -- Validadores de string
- `semver` (^7.7.2) -- Parsing de versao semantica
- `ajv` (^8.17.1) + `ajv-formats` -- Validacao JSON Schema

#### Processo e Execucao
- `execa` (^5.1.1) -- Gerenciamento de child process
- `chokidar` (^3.5.3) -- File watching
- `proper-lockfile` (^4.1.2) -- File locking

#### Testes e Qualidade
- `jest` (^30.2.0) -- Framework de testes (80% coverage threshold)
- `eslint` (^9.38.0) -- Linting
- `prettier` (^3.5.3) -- Formatacao
- `husky` (^9.1.7) -- Git hooks
- `lint-staged` (^16.1.1) -- Linting de staged files
- `semantic-release` (^25.0.2) -- Release automatizado

#### Integracoes Externas MCP
- clickup-direct
- context7
- exa-direct
- desktop-commander
- docker-mcp
- ide

#### CLI Tools Externas
- GitHub CLI 2.x+
- Railway CLI 3.x+
- Supabase CLI 1.x+
- Git 2.30+

#### Performance
- Bundle size: ~5MB (minified, production)
- Cold start: ~200ms; warm start: ~50ms
- Baseline memory: 30MB (Node.js + AIOX core)

### IDE Sync System

O AIOX mantem sincronizacao de agentes para multiplas IDEs:

```bash
npm run sync:ide              # Sync para todas as IDEs
npm run sync:ide:claude       # Sync apenas Claude Code
npm run sync:ide:codex        # Sync apenas Codex
npm run sync:ide:gemini       # Sync apenas Gemini
npm run sync:ide:cursor       # Sync apenas Cursor
npm run sync:ide:github-copilot
npm run sync:ide:antigravity
```

Validacao:
```bash
npm run validate:claude-sync && npm run validate:claude-integration
npm run validate:codex-sync && npm run validate:codex-integration
npm run validate:gemini-sync && npm run validate:gemini-integration
npm run validate:parity       # Valida paridade multi-IDE
```

---

## 7. Sistema de Squads (Deep Dive)

### Conceito

Squads sao equipes modulares de agentes IA para qualquer dominio. Permitem estender o AIOX alem do desenvolvimento de software para escrita criativa, estrategia de negocios, saude, educacao etc.

### Estrutura de um Squad

```
squads/seu-squad/
  config.yaml           # Configuracao do squad
  agents/               # Agentes especializados
  tasks/                # Fluxos de trabalho
  templates/            # Templates
  checklists/           # Checklists de validacao
  data/                 # Base de conhecimento
  README.md             # Documentacao
  user-guide.md         # Guia do usuario
```

### Squads Disponiveis no Repo

```
squads/
  _example/             # Template de exemplo
  claude-code-mastery/  # Squad de dominio Claude Code
```

### Squads Externos

- **hybrid-ops** (https://github.com/SynkraAI/aiox-hybrid-ops-pedro-valerio) -- Operacoes hibridas humano-agente

### AIOX Pro -- Squads Avancados

O modulo Pro oferece squads especializados com capacidades expandidas, disponiveis para membros do AIOX Cohort Advanced.

### Learning Paths Relacionados

1. **Data Engineer Trail:** De "preciso de uma tabela com RLS" ao schema validado
2. **DevOps Trail:** CI/CD, quality gates e deploy automatizado
3. **Squad Creator Trail:** Construa squads de agentes do zero

---

## 8. Seguranca (Deep Dive)

### 11 Dominios de Seguranca

#### 1. API Key Management
- Armazenar keys em environment variables ou secret managers, NUNCA no codigo
- Rotacao: 90 dias para AI provider keys, 30 dias para JWT secrets
- Validar keys no startup com regex e requisitos de comprimento minimo

#### 2. Environment Variables e Secrets
- `.env` em desenvolvimento (adicionado ao `.gitignore`)
- Isolamento de ambiente: prevenir keys de producao em dev
- Criptografia de arquivos de secrets com AES-256-CBC
- Configuracao separada para dev vs producao

#### 3. File e Directory Permissions
- Diretorios criticos: `700` (owner only)
- Arquivos de configuracao: `600`
- Bloquear acesso a paths sensiveis (`/etc`, `/var`, `/usr`, `.ssh`, `.aws`)
- Validar permissoes via scripts automatizados

#### 4. Sandboxing e Isolamento
- MCP servers em containers Docker com `no-new-privileges:true`
- Drop all capabilities por default
- Resource limits: 50% CPU max, 512MB memory
- Read-only root filesystem com dirs temporarios (noexec, nosuid, nodev)

#### 5. Input Validation
- Sanitizar file paths contra directory traversal
- Validar project names (alfanumerico, dashes, underscores, max 64 chars)
- JSON Schema para validacao de configuracao
- Limites de comprimento e filtragem de caracteres

#### 6. Injection Protection
- `execFile()` com argumentos parametrizados em vez de string interpolation
- SQL governance hooks bloqueando CREATE TABLE, DROP TABLE, DELETE, UPDATE, INSERT sem aprovacao
- Read-protection hooks bloqueando reads parciais em arquivos criticos
- Prevencao de prototype pollution bloqueando `__proto__`, `constructor`, `prototype`

#### 7. Logging e Auditing
- Audit logging para autenticacao, acesso a arquivo, execucao de comando
- Formato JSON com timestamp, actor, action, context, result, security approval status
- Redacao de dados sensiveis (API keys, passwords, tokens) antes de escrever logs
- Retencao de 90 dias com compressao e arquivamento

#### 8. Permission Modes
- 3 modos controlam autonomia do agente: **Explore, Ask, Auto**
- Producao default: "Ask" mode requerendo confirmacao do usuario
- Desenvolvimento: "Auto" mode para iteracao mais rapida
- Claude Hooks fornecem validacao pre-execucao

#### 9. Production vs Development Configuration
- Producao: debug off, rate limiting enabled, TLS 1.2+ required, strict validation, 1h session timeout
- Desenvolvimento: debug on, CORS: `*`, rate limiting relaxado, 24h session timeout
- Validacao via script checando NODE_ENV, debug flags, e presenca de certificado

#### 10. Network Security
- CORS para dominios de producao especificos
- TLS 1.2+ com cipher suites especificas
- Security headers (HSTS, CSP, X-Frame-Options: DENY)
- Rate limit: 1000 requests por 15 minutos por default

#### 11. Vulnerability Reporting
- GitHub Security Advisories, NAO issues publicas
- Timeline: 24h acknowledgment, 72h assessment, 30 dias patch
- Security Hall of Fame para responsible disclosures

### Defense in Depth

```
External: Network firewall, WAF, TLS termination, rate limiting
  Application: Permission modes, input validation, command sanitization
    Execution: Sandboxing, process isolation, resource limits, hooks
      Data: Encryption at rest, secure storage, audit logging
```

### Arquitetura Multi-Layer de Seguranca

5 camadas integradas:
1. **Application Layer** -- Funcionalidade core
2. **Authentication Layer** -- Verificacao JWT
3. **Input Validation Layer** -- Sanitizacao e prevencao
4. **Rate Limiting Layer** -- Protecao DOS
5. **Network Layer** -- Seguranca de transporte

### Rate Limiting Strategy

| Operacao | Janela | Limite | Proposito |
|----------|--------|--------|-----------|
| API Calls | 15 min | 1000 | Protecao geral |
| Authentication | 15 min | 5 | Prevencao brute force |
| Installation | 1 hora | 10 | Prevencao de abuso |
| Meta-Agent | 1 min | 30 | Protecao de recurso |
| File Operations | 1 min | 100 | Protecao de filesystem |

### Criptografia
- **At rest:** AES-256-GCM com IVs aleatorios de 16 bytes
- **Em transito:** TLS 1.2+ com cipher suites fortes

### OWASP Top 10 Compliance
Todas as 10 categorias enderecadas com implementacoes especificas.

---

## 9. Distribuicao NPM e CLI

### Package.json Key Info

```json
{
  "name": "aiox-core",
  "version": "5.0.3",
  "license": "MIT",
  "bin": {
    "aiox": "bin/aiox.js",
    "aiox-core": "bin/aiox.js",
    "aiox-minimal": "bin/aiox-minimal.js",
    "aiox-graph": "bin/aiox-graph.js"
  },
  "workspaces": ["packages/*"],
  "engines": {
    "node": ">=18.0.0",
    "npm": ">=9.0.0"
  }
}
```

### Binarios CLI

| Binario | Funcao |
|---------|--------|
| `aiox` / `aiox-core` | CLI principal (bin/aiox.js) |
| `aiox-minimal` | CLI minimal (bin/aiox-minimal.js) |
| `aiox-graph` | Visualizacao de dependencias (bin/aiox-graph.js) |
| `aiox-ids` | IDS system (bin/aiox-ids.js) |
| `aiox-init` | Inicializacao (bin/aiox-init.js) |

### O que e Publicado no NPM (files field)

```
bin/
scripts/
packages/
.aiox-core/
.claude/CLAUDE.md
.claude/rules/
.claude/hooks/
pro/license/
pro/squads/
pro/pro-config.yaml
pro/feature-registry.yaml
pro/package.json
pro/README.md
docs/guides/
docs/installation/
docs/examples/
docs/community/
docs/legal/
docs/security/
docs/aiox-workflows/
docs/aiox-agent-flows/
docs/framework/
docs/en/ docs/es/ docs/pt/ docs/zh/
docs/*.md
README.md
LICENSE
```

### Fluxo de Instalacao

#### Novo projeto:
```bash
npx aiox-core init meu-projeto
```

#### Projeto existente:
```bash
cd seu-projeto
npx aiox-core install
```

#### Atualizacao:
```bash
npx aiox-core@latest install
```

O instalador:
- Detecta instalacao existente automaticamente
- Atualiza apenas arquivos que mudaram
- Cria backups `.bak` para customizacoes
- Preserva configuracoes especificas do projeto

### Comandos CLI NPX

```bash
npx aiox-core init <nome-projeto> [--force] [--skip-install] [--template <nome>]
npx aiox-core install [--force] [--quiet] [--dry-run]
npx aiox-core --version
npx aiox-core info
npx aiox-core doctor [--fix]
npx aiox-core update
npx aiox-core uninstall
```

### Scripts NPM Notaveis

```bash
npm run validate:structure     # Source tree guardian
npm run validate:agents        # Validar agentes
npm run sync:ide               # Sincronizar todas IDEs
npm run validate:parity        # Validar paridade multi-IDE
npm run validate:publish       # Validar antes de publicar
npm run generate:manifest      # Gerar install manifest
npm run validate:manifest      # Validar manifest
```

### Lint-Staged Configuration

```json
{
  "*.{js,mjs,cjs,ts}": ["eslint --fix --cache", "prettier --write"],
  "*.md": ["prettier --write", "node scripts/semantic-lint.js --staged"],
  ".aiox-core/development/agents/*.md": ["npm run sync:ide"]
}
```

Nota: Alteracoes em arquivos de agentes automaticamente triggerem sync de IDE.

---

## 10. Constitution e Principios

### Principios Fundamentais

| Artigo | Principio | Severidade |
|--------|-----------|------------|
| I | CLI First | NON-NEGOTIABLE |
| II | Agent Authority | NON-NEGOTIABLE |
| III | Story-Driven Development | MUST |
| IV | No Invention | MUST |
| V | Quality First | MUST |

#### I. CLI First (NON-NEGOTIABLE)
- Toda funcionalidade nova DEVE funcionar 100% via CLI antes de UI
- Dashboards apenas observam, NUNCA controlam
- UI NUNCA e requisito para operacao do sistema
- Hierarquia: CLI > Observability > UI

#### II. Agent Authority (NON-NEGOTIABLE)
- Cada agente tem autoridades exclusivas inviolaveis
- Apenas @devops pode `git push`, criar PRs, releases
- Agentes DEVEM delegar quando fora do escopo
- Nenhum agente pode assumir autoridade de outro

#### III. Story-Driven Development (MUST)
- Nenhum codigo sem story associada
- Stories DEVEM ter acceptance criteria claros
- Progresso rastreado via checkboxes na story
- File List mantida atualizada

#### IV. No Invention (MUST)
- Todo statement em spec.md DEVE rastrear para FR-*, NFR-*, CON-*, ou finding de research
- NAO adicionar features nao presentes nos requisitos
- NAO assumir detalhes de implementacao nao pesquisados
- NAO especificar tecnologias nao validadas

#### V. Quality First (MUST)
- `npm run lint` sem erros
- `npm run typecheck` sem erros
- `npm test` sem falhas
- `npm run build` com sucesso
- CodeRabbit sem issues CRITICAL
- Story status "Done" ou "Ready for Review"

---

## 11. Autonomous Development Engine (ADE)

O ADE e o sistema para desenvolvimento autonomo que transforma requisitos em codigo funcional. Composto por 7 Epics:

| Epic | Nome | Descricao |
|------|------|-----------|
| 1 | Worktree Manager | Isolamento de branches via Git worktrees |
| 2 | Migration V2-->V3 | Migracao para formato autoClaude V3 |
| 3 | Spec Pipeline | Transforma requisitos em specs executaveis |
| 4 | Execution Engine | Executa specs com 13 steps + self-critique |
| 5 | Recovery System | Recuperacao automatica de falhas |
| 6 | QA Evolution | Review estruturado em 10 fases |
| 7 | Memory Layer | Memoria persistente de padroes e insights |

### Fluxo Principal

```
User Request --> Spec Pipeline --> Execution Engine --> QA Review --> Working Code
                                        |
                                Recovery System
                                        |
                                 Memory Layer
```

### Quick Start ADE

```bash
# 1. Criar spec a partir de requisito
@pm *gather-requirements
@architect *assess-complexity
@analyst *research-deps
@pm *write-spec
@qa *critique-spec

# 2. Executar spec aprovada
@architect *create-plan
@architect *create-context
@dev *execute-subtask 1.1

# 3. QA Review
@qa *review-build STORY-42
```

---

## 12. Diferencas-Chave AIOX vs SINAPSE

### Nomenclatura

| Conceito | AIOX | SINAPSE |
|----------|------|---------|
| Framework name | aiox-core | sinapse-ai |
| Config dir | .aiox-core/ | .sinapse-ai/ |
| Runtime dir | .aiox/ | .sinapse/ |
| Master agent | @aios-master (Orion) | @sinapse-orqx (Imperator) |
| Dev agent | @dev (Dex) | @developer (Pixel) |
| QA agent | @qa (Quinn) | @quality-gate (Litmus) |
| Architect | @architect (Aria) | @architect (Stratum) |
| PM | @pm (Morgan) | @project-lead (Beacon) |
| PO | @po (Pax) | @product-lead (Axis) |
| SM | @sm (River) | @sprint-lead (Sync) |
| DevOps | @devops (Gage) | @devops (Pipeline) |
| Data Engineer | @data-engineer (Dara) | @data-engineer (Tensor) |
| UX | @ux-design-expert (Uma) | @ux-design-expert (Mosaic) |
| Analyst | @analyst (Atlas) | @analyst (Scope) |

### Diferencas Arquiteturais

| Aspecto | AIOX | SINAPSE |
|---------|------|---------|
| Licenca | MIT (open source) | UNLICENSED (privado) |
| Distribuicao | NPM package (`npx aiox-core`) | Integrado diretamente nos projetos |
| Multi-IDE | Claude, Gemini, Codex, Cursor, Copilot, AntiGravity | Claude Code (primario) |
| Squads | Built-in com marketplace community | Built-in com foco em dominos de negocio |
| Pro module | Separado (`@aiox-fullstack/pro`) | N/A (tudo integrado) |
| Memoria | 2 camadas desconectadas + Pro MIS | Obsidian-based (Ars Contexta) |
| Constitution | 5 artigos | 10 artigos |
| Doc-first | Article III: Story-Driven (MUST) | Article III: Documentation-First (NON-NEGOTIABLE) |
| CLI tools | aiox, aiox-minimal, aiox-graph, aiox-ids | sinapse (CLI) |
| Workflows | 14+ workflows YAML | 4 primary workflows |
| Testing | Jest 30, 80% coverage | Jest, project-specific |
| Hooks | Cross-CLI abstraction layer | Claude Code hooks |
| User safety | Nao mencionado explicitamente | Safe Collaboration (NON-NEGOTIABLE) para nao-git-experts |
| Delegation | Agent Authority (NON-NEGOTIABLE) | Mandatory Delegation (NON-NEGOTIABLE) |
| Security articles | 11 dominios | 25 deployment blockers + LGPD |

### O que AIOX Tem que SINAPSE NAO Tem

1. **Multi-IDE support** com IDE sync system e paridade de hooks
2. **NPM distribution** para instalacao em qualquer projeto
3. **Gotchas Memory** -- auto-captura de erros recorrentes
4. **Context Snapshot** com crash detection e recovery
5. **File Evolution Tracker** para detectar drift
6. **Timeline Manager** unificado com 5000 entradas
7. **Memory Intelligence System (MIS)** com 4 camadas, attention scoring, self-learning
8. **Autonomous Development Engine (ADE)** com 7 epics
9. **Worktree Manager** para isolamento de branches
10. **Spec Pipeline** completo com 6 fases e classes de complexidade
11. **Permission Modes** (Explore, Ask, Auto) para controle de autonomia
12. **50+ commands** em 9 categorias incluindo self-modification
13. **Squad marketplace** community com templates
14. **Learning Paths** (Data Engineer, DevOps, Squad Creator)
15. **Session-level crash detection** (`SessionState.detectCrash()`)
16. **Progressive disclosure** para retrieval de memoria (50/200/1000+ tokens)
17. **Cognitive sectors** (Episodic, Semantic, Procedural, Reflective)
18. **Decay functions** com taxas diferenciadas por tier
19. **Self-learning** com confidence scoring e rule proposals
20. **IDE-agnostic event mapping** (cross-CLI abstraction)

### O que SINAPSE Tem que AIOX NAO Tem

1. **Constitution mais completa** (10 artigos vs 5)
2. **Documentation-First como NON-NEGOTIABLE** (AIOX e apenas MUST)
3. **Mandatory Delegation explicitamente NON-NEGOTIABLE** com regra "mesmo quando explicitamente pedido"
4. **Safe Collaboration** como principio fundamental para usuarios nao-tecnicoss
5. **Security & Data Protection** com 25 deployment blockers e LGPD compliance
6. **Terminal Bus** para comunicacao cross-terminal
7. **NSN Mode** (Never Say Never) -- protocolo de resiliencia
8. **Obsidian Knowledge Persistence** (Ars Contexta) com cronjob diario
9. **Cross-Squad Routing** patterns predefinidos (brand_launch, go_to_market, etc.)
10. **Hook Governance** documentacao formal com exit code protocol
11. **Agent Handoff Protocol** com compaction limits e scratchpad
12. **Squads de negocio** especializados (brand, commercial, content, copy, finance, growth, etc.)

---

## 13. Recomendacoes para SINAPSE-AI

### PRIORIDADE ALTA -- Implementar Imediatamente

#### R1: Memory Intelligence System (MIS)
**O que:** Implementar sistema de memoria com 4 camadas (Capture, Storage, Retrieval, Evolution).
**Por que:** A maior diferenca tecnica entre AIOX e SINAPSE. O MIS do AIOX resolve o problema de perda de contexto entre sessoes com attention scoring, progressive disclosure e self-learning. O SINAPSE atualmente depende do Obsidian (Ars Contexta) que e um cronjob diario -- nao e real-time.
**Como implementar:**
- Criar modulo de Gotchas Memory (auto-captura de erros recorrentes)
- Implementar Context Snapshot com crash detection
- Adicionar File Evolution Tracker
- Usar formato Markdown + YAML frontmatter para memorias
- Attention scoring: `base_relevance x recency_factor x access_modifier x confidence`
- Tiers HOT (>0.7)/WARM (0.3-0.7)/COLD (<0.3) com token budgets
- Progressive disclosure: Index Scan (~50t) --> Context (~200t) --> Full (~1000+t)
- Cognitive sectors por agente (dev=Procedural+Semantic, qa=Reflective+Episodic)

#### R2: Crash Detection e Recovery
**O que:** Implementar `SessionState.detectCrash()` que detecta se sessao anterior terminou abruptamente.
**Por que:** Sessoes interrompidas sao comuns. Sem recovery, todo contexto e perdido.
**Como:** Detectar se ultima atividade > 30min e ultima acao != PAUSE/COMPLETED/ABORT. Oferecer opcoes: CONTINUE, REVIEW, RESTART, DISCARD.

#### R3: Gotchas Memory
**O que:** Sistema que auto-captura erros apos 3 ocorrencias identicas em 24h.
**Por que:** Evita que agentes repitam os mesmos erros. Aprendizado automatico sem intervencao do usuario.
**Como:** Hook em PostToolUseFailure, persistir em `.sinapse/gotchas.json`, injetar gotchas relevantes no contexto de cada task.

### PRIORIDADE MEDIA -- Implementar em Breve

#### R4: Permission Modes (Explore, Ask, Auto)
**O que:** 3 modos de autonomia para agentes.
**Por que:** Permite controlar quanto de confirmacao o usuario precisa dar. "Auto" para desenvolvimento rapido, "Ask" para producao.
**Como:** Configuracao em `core-config.yaml`, hooks validam antes de cada execucao.

#### R5: Spec Pipeline Completo
**O que:** 6 fases para transformar requisitos vagos em specs executaveis com classes de complexidade (SIMPLE/STANDARD/COMPLEX).
**Por que:** O SINAPSE tem o conceito mas nao as 5 dimensoes de complexidade pontuadas (Scope, Integration, Infrastructure, Knowledge, Risk).
**Como:** Implementar scoring e skip logic baseado em complexidade.

#### R6: Self-Learning com Confidence Scoring
**O que:** Sistema que aprende com correcoes do usuario e propoe novas regras quando confidence > 0.9 com 5+ evidencias.
**Por que:** Melhoria continua automatica do framework baseada em uso real.
**Como:** Correction tracking (confidence 0.3 inicial) --> Pattern recognition (5+ sessoes) --> Heuristic extraction --> User approval gate.

#### R7: Execution Modes (YOLO, Interactive, Pre-Flight)
**O que:** 3 modos de execucao para o agente developer.
**Por que:** YOLO permite execucao completamente autonoma (0-1 prompts) com decision logs. Util para tasks simples e batch processing.
**Como:** Flag no `*develop` command; YOLO gera `decision-log-{story-id}.md` automaticamente.

#### R8: IDE Sync System
**O que:** Sistema de sincronizacao automatica de agentes para multiplas IDEs.
**Por que:** Embora SINAPSE foque em Claude Code, ter paridade com Gemini/Codex expande alcance.
**Como:** Scripts de sync que geram configuracoes especificas por IDE a partir de uma fonte unica.

### PRIORIDADE BAIXA -- Considerar para Futuro

#### R9: NPM Distribution
**O que:** Distribuir SINAPSE como pacote NPM para instalacao em qualquer projeto.
**Por que:** Democratiza o framework. Qualquer pessoa pode usar com `npx sinapse-ai install`.
**Nota:** Requer revisao de licenciamento (atualmente UNLICENSED).

#### R10: Timeline Manager Unificado
**O que:** Fachada unificada que agrega eventos de snapshots, file evolution e entradas manuais.
**Por que:** Historico unificado facilita debugging e auditoria.
**Como:** 5000 entradas max, 90 dias retencao, auto-sync a cada 60s.

#### R11: Agent Self-Modification Commands
**O que:** Comandos como `*improve-self`, `*evolve`, `*adapt`, `*optimize-performance`.
**Por que:** Permite que agentes se otimizem baseado em uso.
**Nota:** Requer gates fortes para evitar drift indesejado.

#### R12: Worktree Manager
**O que:** Isolamento de branches via Git worktrees para desenvolvimento paralelo.
**Por que:** Permite trabalho simultaneo em multiplas stories sem conflito.
**Como:** Comandos `*create-worktree`, `*merge-worktree`, `*cleanup-worktrees`.

#### R13: Token Budget por Agente
**O que:** Cada agente tem um budget de tokens configuravel para memoria.
**Por que:** Controle fino de quanto contexto cada agente recebe.
**Como:** Campo `memoryBudget` na configuracao do agente, default 2000 tokens.

---

## Apendice A: Referencia de Comandos AIOX (50+)

### 9 Categorias

| Categoria | Qtd | Exemplos |
|-----------|-----|----------|
| Core | 4 | `*help`, `*status`, `*config`, `*version` |
| Agent Management | 7 | `*create-agent`, `*list-agents`, `*activate`, `*deactivate`, `*modify-agent`, `*delete-agent`, `*clone-agent` |
| Task Operations | 5 | `*create-task`, `*list-tasks`, `*run-task`, `*schedule-task`, `*modify-task` |
| Workflow | 5 | `*create-workflow`, `*list-workflows`, `*run-workflow`, `*stop-workflow`, `*workflow-status` |
| Code Generation | 4 | `*generate-component`, `*generate-api`, `*generate-tests`, `*generate-documentation` |
| Analysis | 5 | `*analyze-framework`, `*analyze-code`, `*improve-code-quality`, `*suggest-refactoring`, `*detect-patterns` |
| Memory | 4 | `*memory`, `*learn`, `*remember`, `*forget` |
| Self-Modification | 4 | `*improve-self`, `*evolve`, `*adapt`, `*optimize-performance` |
| System | 6+ | `*backup`, `*restore`, `*update`, `*uninstall`, `*doctor`, `*export`, `*import`, `*benchmark`, `*debug`, `*plugin` |

## Apendice B: Mapeamento de Eventos Cross-CLI

| Evento AIOX | Claude Code | Gemini CLI | Codex CLI |
|-------------|-------------|------------|-----------|
| sessionStart | null | SessionStart | N/A |
| sessionEnd | Stop (unmapped) | SessionEnd | N/A |
| beforeAgent | PreToolUse | BeforeAgent | N/A |
| afterTool | PostToolUse | AfterTool | N/A |

## Apendice C: Links de Referencia

- **GitHub:** https://github.com/SynkraAI/aiox-core
- **NPM:** https://www.npmjs.com/package/aiox-core
- **Documentacao:** https://aiox.academialendaria.ai
- **Versao analisada:** 5.0.3
- **Licenca:** MIT

---

> **Analise conduzida por Prism (Research Orchestrator)**
> Squad: squad-research | SINAPSE-AI
> Data: 2026-04-04
