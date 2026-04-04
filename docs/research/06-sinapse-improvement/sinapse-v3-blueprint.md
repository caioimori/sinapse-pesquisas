# SINAPSE-AI v3 Blueprint

> **Status:** LIVING DOCUMENT -- atualizado continuamente conforme novas pesquisas
> **Data de criacao:** 2026-04-04
> **Autor:** Prism (Research Operations Conductor) | squad-research
> **Fontes consolidadas:** gap-analysis.md (50 gaps), source-extractions.md (Claude Code internals), aiox-deep-analysis.md (AIOX patterns), multi-llm-squads-ux.md (Multi-LLM + UX), skills-ecosystem-analysis.md (Skills standard), hallucinations-prevention.md (Anti-hallucination)
> **Objetivo:** Single Source of Truth para todas as decisoes arquiteturais do SINAPSE v3

---

## 1. Visao

O SINAPSE-AI v3 sera um meta-framework de orquestracao de agentes AI **token-conscious, verification-first e multi-LLM** que transforma qualquer LLM CLI/IDE em um sistema de desenvolvimento orquestrado com squads especializadas, quality gates automaticos e memoria persistente. Diferente de colecoes de skills isoladas (Superpowers, Trail of Bits) ou frameworks single-IDE (AIOX), o SINAPSE v3 sera o primeiro framework a oferecer transpilacao completa -- agents, hooks, permissions e instrucoes -- para 8+ plataformas a partir de uma unica fonte de verdade, mantendo governance constitucional e anti-hallucination em todas elas.

---

## 2. Principios Arquiteturais

Sete principios que guiam TODA decisao de design no SINAPSE v3:

### P1. Token-First Design
Cada byte de instrucao tem custo financeiro e cognitivo. O overhead de instrucoes por turn deve ser inferior a 5K tokens. Conteudo que nao e relevante para o turn atual NAO deve ser carregado. Progressive disclosure em 3 niveis (metadata -> instructions -> resources) e o padrao universal.

**Base:** gap-analysis.md mostra 31K tokens/turn atual (excedendo 12K char limit em 69%). Claude Code trunca silenciosamente apos 4K chars/arquivo. 40% das instrucoes atuais nunca sao lidas pelo modelo.

### P2. Verification-First (Zero Trust para AI)
Nunca confiar em output de LLM sem verificacao. Toda geracao de codigo passa por 7 camadas de defesa: prompt engineering -> tool grounding (Read before Edit) -> type checking -> linting -> test execution -> code review (CodeRabbit) -> quality gates. Memory entries sao hints, nao ground truth.

**Base:** hallucinations-prevention.md demonstra que hallucination e matematicamente inevitavel. 19.7% dos packages sugeridos por LLMs sao fabricados. 29-45% do codigo AI contem vulnerabilidades.

### P3. Progressive Disclosure
Informacao e carregada em camadas crescentes de detalhe, apenas quando necessaria. CLAUDE.md contem resumo ultra-compacto. Rules carregam por path matching. Skills carregam on-demand. Knowledge bases carregam via tool use. Documentos grandes sao sharded em chunks de ~300 tokens.

**Base:** Agent Skills spec (agentskills.io) define 3 niveis: metadata (~100 tokens, sempre) -> instructions (<5K tokens, quando ativado) -> resources (ilimitado, sob demanda). Claude Code usa Deferred Tools (93% reducao).

### P4. Multi-LLM Compatibility
Claude Code e first-class citizen, mas toda configuracao e transpilavel para Codex CLI, Gemini CLI, Cursor, GitHub Copilot, Windsurf, Kimi Code e Kiro. AGENTS.md e o standard universal (24+ ferramentas). Configuracoes tool-specific sao aditivas, nunca subtrativas.

**Base:** multi-llm-squads-ux.md mapeou 8 LLM CLIs/IDEs com features detalhadas. AGENTS.md e stewarded pela Agentic AI Foundation (Linux Foundation).

### P5. Skills as Distribution Layer
Squads e agents sao o motor interno (YAML + tasks + workflows). Skills no formato SKILL.md sao a camada de distribuicao publica que alcanca 33 plataformas. Toda knowledge exportavel do SINAPSE deve ter uma representacao em SKILL.md padrao.

**Base:** skills-ecosystem-analysis.md mostra 1,060+ skills catalogadas, 33 plataformas suportando o formato, 134K stars no obra/superpowers.

### P6. Orchestration Over Isolation
O diferencial do SINAPSE nao e ter skills isoladas -- e ter um sistema orquestrado com governance constitucional, delegacao automatica, quality gates e memoria cross-agent. Nenhum competidor tem orquestracao cross-domain com squads.

**Base:** Analise comparativa mostra que SINAPSE e significativamente mais sofisticado que qualquer player. O gap esta na distribuicao, nao na capacidade.

### P7. Graceful Degradation
Toda feature avancada degrada graciosamente quando a plataforma nao suporta. Hooks degradam para instrucoes textuais. Agents degradam para rules. Memory degrada para CLAUDE.md/AGENTS.md. Code intelligence e sempre opcional.

**Base:** AIOX demonstra que compatibilidade real varia entre plataformas (Claude Code: full, Cursor: rules-only). O SINAPSE deve funcionar em todas, mesmo que com menos automacao.

---

## 3. Token Budget Architecture

### 3.1 O Problema Atual

| Componente | Tokens Atuais | Limite Real | Status |
|-----------|--------------|-------------|--------|
| Global CLAUDE.md | ~1,721 | 1,000 (4K chars) | EXCEDE |
| Project CLAUDE.md | ~3,349 | 1,000 (4K chars) | EXCEDE |
| Total CLAUDE.md | ~5,069 | 3,001 (12K chars) | EXCEDE 69% |
| Rules (project, 19 files) | ~16,995 | Conditional | SEM FILTERING |
| Rules (global, 13 files) | ~10,495 | Conditional | SEM FILTERING |
| Agent persona | ~800-1,500 | On-demand | OK |
| **TOTAL por turn** | **~31,000-32,000** | **<5,000 target** | **6x acima** |

### 3.2 Budget Target v3

| Componente | Budget Maximo | Carregamento |
|-----------|--------------|-------------|
| CLAUDE.md (global) | 800 tokens (3,200 chars) | Sempre |
| CLAUDE.md (project) | 800 tokens (3,200 chars) | Sempre |
| AGENTS.md (universal) | 500 tokens (2,000 chars) | Sempre (multi-LLM) |
| Rules ativas (path-matched) | 1,500 tokens | Condicional |
| Agent persona ativa | 800 tokens | Quando invocado |
| Memory (HOT tier) | 500 tokens | SessionStart |
| **TOTAL maximo por turn** | **~4,900 tokens** | -- |
| Skills (on-demand) | 1,500 tokens cada | Quando triggered |

### 3.3 CLAUDE.md v3 -- Conteudo Obrigatorio (max 3,200 chars cada)

**Global CLAUDE.md (~800 tokens):**
```
- User profile (Caio/Matheus -- product builders, nao git experts)
- Safe collaboration resumo (auto-branch, auto-sync, auto-PR)
- Git safety net (NUNCA main, NUNCA force push sem confirmar)
- Security essentials (secret scan, service_role never frontend)
- Agent invocation syntax (@agent-name, *command)
- Referencia: "Detalhes em skills on-demand"
```

**Project CLAUDE.md (~800 tokens):**
```
- Constitution resumo (10 artigos, 1 linha cada)
- Framework boundary (L1-L4, 1 linha cada)
- Documentation-First gate (1 paragrafo)
- Mandatory delegation gate (1 paragrafo)
- Agent roster (tabela compacta: role -> persona -> escopo)
- Anti-hallucination rules (Read before Edit, Uncertainty Protocol)
- Referencia: "Detalhes em rules/ com paths: filtering"
```

**O que SAI dos CLAUDE.md:**
- Code snippets genericos (error handling, template loading)
- Debugging instructions
- Environment setup
- Common commands
- Framework structure detalhada (vai para skill)
- Workflow execution detalhada (vai para skill)
- Security checklist detalhada (vai para skill)
- Tool examples (vai para skill)
- Toda duplicacao global/project

### 3.4 Rules v3 -- Todas com `paths:` Frontmatter

| Rule | paths: | Estimativa de carga |
|------|--------|---------------------|
| safe-collaboration.md | `.git/**`, `**/branch*` | 5% dos turns |
| security-data-protection.md | `.env*`, `**/supabase/**`, `**/deploy*` | 10% dos turns |
| mcp-usage.md | `.claude/settings*`, `**/mcp*` | 5% dos turns |
| workflow-execution.md | `docs/stories/**`, `**/workflow*` | 15% dos turns |
| story-lifecycle.md | `docs/stories/**` | 15% dos turns |
| squad-awareness.md | `squads/**`, `**/squad*` | 10% dos turns |
| mandatory-delegation.md | (skill, nao rule) | 0% |
| agent-authority.md | (skill, nao rule) | 0% |
| agent-handoff.md | (skill, nao rule) | 0% |
| documentation-first.md | `docs/**`, `src/**` (gate) | 20% dos turns |
| hook-governance.md | `.claude/hooks/**` | 3% dos turns |
| nsn-mode.md | (skill, nao rule) | 0% |
| cross-squad-routing.md | (skill, nao rule) | 0% |
| tool-examples.md | (skill, nao rule) | 0% |
| tool-response-filtering.md | (skill, nao rule) | 0% |
| ids-principles.md | (skill, nao rule) | 0% |
| coderabbit-integration.md | (skill, nao rule) | 0% |

**Rules que permanecem (7):** ~25K chars totais, mas apenas ~3-5K carregam por turn medio (path filtering reduz 70-80%).

**Rules que viram skills (12):** ~43K chars removidos do carregamento automatico.

### 3.5 Economia Projetada

| Otimizacao | Savings (tokens/turn) |
|-----------|----------------------|
| Reescrever CLAUDE.md dentro dos limites | Corrige truncation |
| Path filtering em rules | ~12,000 |
| Eliminar duplicacao global/project | ~750-1,000 |
| Eliminar duplicacao rules/CLAUDE.md | ~1,250 |
| Migrar 12 rules para skills | ~14,000 |
| Remover conteudo irrelevante | ~500-750 |
| **TOTAL** | **~28,500-29,000** |

**Reducao de ~31K para ~3-5K tokens/turn = ~84-90% de economia.**

---

## 4. Skills System Design

### 4.1 Formato de Skill SINAPSE

Baseado na spec oficial agentskills.io + anthropics/skills:

```
.sinapse/skills/{skill-name}/
  SKILL.md          # OBRIGATORIO: frontmatter YAML + instrucoes
  scripts/           # Opcional: codigo executavel
  references/        # Opcional: documentacao detalhada
  assets/            # Opcional: templates, recursos
```

**Frontmatter padrao:**
```yaml
---
name: sinapse-{domain}-{skill}
description: >
  [O que faz]. [Quando usar -- ser "pushy" com keywords de ativacao].
  Use when: [triggers explicitos].
license: UNLICENSED
compatibility: SINAPSE framework. Works standalone for basic usage.
metadata:
  author: sinapse-ai
  version: "1.0"
  squad: "{squad-name}"
  category: "{category}"
---
```

**Corpo do SKILL.md (<500 linhas):**
```markdown
# {Skill Title}

## When to Use
[Decision tree com triggers explicitos]

## Process
[Steps imperativas, nao declarativas]

## Examples
[Input/output concretos]

## References
- [Detailed guide](references/guide.md)
- [Templates](assets/templates/)
```

### 4.2 Categorias de Skills

| Categoria | Skills | Tipo |
|-----------|--------|------|
| **Framework Core** | constitution-detail, agent-authority, mandatory-delegation, agent-handoff | Contextual |
| **Workflow** | story-lifecycle, workflow-execution, spec-pipeline, qa-loop | Contextual |
| **Security** | security-data-protection, secret-scanning, pre-deploy-gates | Contextual |
| **Quality** | coderabbit-integration, code-review, testing-patterns | Contextual |
| **DevOps** | safe-collaboration, mcp-usage, ci-cd-templates | Contextual |
| **Research** | deep-research, competitive-analysis, market-sizing | Squad-specific |
| **Brand** | brand-audit, visual-identity, tone-of-voice | Squad-specific |
| **Content** | editorial-calendar, content-strategy | Squad-specific |
| **Anti-Hallucination** | verification-patterns, import-validation, vibe-and-verify | Contextual |
| **Tooling** | tool-examples, tool-response-filtering, ids-principles | Contextual |
| **Cross-Squad** | cross-squad-routing, nsn-mode | Contextual |
| **Onboarding** | getting-started, project-setup, squad-creation | Contextual |

### 4.3 Migracao: Rules -> Skills

**Criterio de decisao:**
- Se a rule e um GATE (bloqueia operacao) -> permanece como rule
- Se a rule e INFORMACIONAL (detalha como fazer) -> vira skill
- Se a rule e CONTEXTUAL (so relevante em certos paths) -> rule com paths: filtering

**Plano de migracao:**

| Rule Atual | Destino v3 | Justificativa |
|-----------|-----------|---------------|
| documentation-first.md | Rule (gate) | Bloqueia codigo sem story |
| safe-collaboration.md | Rule (com paths:) | Gate de git safety |
| security-data-protection.md | Rule (com paths:) + Skill (checklist detalhada) | Gate + detalhes |
| mandatory-delegation.md | Skill | Informacional, nao gate |
| agent-authority.md | Skill | Informacional, nao gate |
| agent-handoff.md | Skill | Protocolo invocado quando necessario |
| workflow-execution.md | Rule (com paths:) | Relevante quando stories sao editadas |
| story-lifecycle.md | Rule (com paths:) | Relevante quando stories sao editadas |
| hook-governance.md | Rule (com paths:) | Relevante quando hooks sao editados |
| squad-awareness.md | Rule (com paths:) | Relevante quando squads sao usadas |
| mcp-usage.md | Rule (com paths:) | Relevante quando MCP configs mudam |
| nsn-mode.md | Skill | Protocolo invocado quando agent trava |
| cross-squad-routing.md | Skill | Invocado quando multi-squad |
| tool-examples.md | Skill | Referencia on-demand |
| tool-response-filtering.md | Skill | Referencia on-demand |
| ids-principles.md | Skill | Principios consultados on-demand |
| coderabbit-integration.md | Skill | Invocado pre-commit |

### 4.4 Cross-Platform Compatibility

Skills no formato SKILL.md padrao funcionam em 33 plataformas sem modificacao. Para distribuicao publica:

```
sinapse-ai/skills/           # Repositorio publico
  marketplace.json           # Plugin manifest
  skills/
    sinapse-research-deep/SKILL.md
    sinapse-qa-gate/SKILL.md
    sinapse-story-creation/SKILL.md
    ...
```

**Instalacao:**
```bash
# Claude Code
/plugin marketplace add sinapse-ai/skills
/plugin install research-skills@sinapse-marketplace

# Codex CLI
/skills install sinapse-research-deep

# Gemini CLI / Cursor / Copilot
# SKILL.md e lido automaticamente se presente no workspace
```

---

## 5. Memory Architecture v2

### 5.1 Modelo de 3 Tiers (HOT/WARM/COLD)

Baseado no AIOX Memory Intelligence System + Claude Code autoDream:

| Tier | Attention Score | Comportamento | Token Budget | Decay |
|------|----------------|---------------|-------------|-------|
| **HOT** | > 0.7 | Sincronizado com MEMORY.md, sempre disponivel | ~500 tokens | 0.01/dia |
| **WARM** | 0.3-0.7 | Carregado no SessionStart se budget permitir | ~2,000 tokens | 0.1/dia |
| **COLD** | < 0.3 | Acessivel via `*recall` ou Grep | Sob demanda | 0.5/dia |
| **ARCHIVE** | < 0.1 (90+ dias) | Movido para `.old/` | N/A | N/A |

**Formula de attention scoring:**
```
attention_score = base_relevance x recency_factor x access_modifier x confidence
```

### 5.2 4 Tipos de Memoria (Cognitive Sectors)

| Sector | Descricao | TTL Base | Exemplo |
|--------|-----------|----------|---------|
| **Episodic** | "O que aconteceu" | 7 dias | "Sessao de ontem descobriu bug no auth flow" |
| **Semantic** | "O que sabemos" | 365 dias | "Projeto usa Supabase + Next.js 15" |
| **Procedural** | "Como fazer" | 30 dias | "Para deploy: npm run build && vercel deploy" |
| **Reflective** | "O que aprendemos" | Infinito | "RLS sempre deve ser testado com usuario diferente" |

### 5.3 Auto-Consolidation (Dream Protocol)

Baseado no autoDream do Claude Code (4 fases):

```
Phase 1 -- Orient
- ls o diretorio de memoria
- Ler indice
- Scan rapido dos topic files

Phase 2 -- Gather Recent Signal
- Daily logs desde ultima consolidacao
- Memorias existentes que mudaram
- Busca em transcripts (grep JSONL narrowly)

Phase 3 -- Consolidate
- Escrever/atualizar memory files
- Merge novos sinais em arquivos existentes
- Converter datas relativas para absolutas
- Deletar fatos contraditos (manter mais recente)

Phase 4 -- Prune and Index
- Atualizar indice (max ~200 linhas, ~25KB)
- Cada entrada: uma linha, ~150 chars
- Remover ponteiros stale
- Demotar entradas verbose
- Resolver contradicoes
```

**Triggers:**
- >= 24h desde ultima consolidacao
- >= 5 sessoes novas
- Nenhuma consolidacao ativa
- >= 10min desde ultimo scan

**Implementacao v3:** Hook `SessionEnd` que dispara consolidacao como worker fork. Alternativa: comando `/dream` para consolidacao manual.

### 5.4 Memory as Hints (NAO Ground Truth)

Regra NON-NEGOTIABLE para todos os agentes:

```
Memory entries are hints, not ground truth.
ALWAYS verify against actual codebase before acting on memory.
Only update memory AFTER successful actions (Strict Write Discipline).
```

### 5.5 Size Limits

| Componente | Limite |
|-----------|--------|
| MEMORY.md index | ~200 linhas, ~25KB |
| Cada entrada de indice | ~150 chars |
| Topic file individual | ~5KB |
| Total de memorias | ~50KB |
| Memory budget por turn | ~500 tokens (HOT) + ~2,000 tokens (WARM) |

---

## 6. Agent Architecture v2

### 6.1 Dois Niveis de Agentes

| Nivel | Agentes | Carregamento | Disponibilidade |
|-------|---------|-------------|-----------------|
| **Framework (core)** | 10-12 agentes | Persona sob demanda, roster sempre visivel | Todo projeto SINAPSE |
| **Squad (domain)** | 165+ agentes | Carregados quando squad e invocada | Squads instaladas |

**Framework agents (fixos):**

| Role | Persona | Escopo |
|------|---------|--------|
| @developer | Pixel | Implementacao de codigo |
| @quality-gate | Litmus | Testes e qualidade |
| @architect | Stratum | Arquitetura e design tecnico |
| @project-lead | Beacon | Product Management, epic orchestration |
| @product-lead | Axis | Product Owner, story validation |
| @sprint-lead | Sync | Scrum Master, story creation |
| @analyst | Scope | Pesquisa e analise |
| @data-engineer | Tensor | Database design |
| @ux-design-expert | Mosaic | UX/UI design |
| @devops | Pipeline | CI/CD, git push (EXCLUSIVO) |

**DECISAO:** Personas sao UNICAS e consistentes em todos os projetos. Eliminar a duplicidade atual (Pixel/Dex, Litmus/Quinn, etc.).

### 6.2 Worker Fork Model

Para tarefas paralelas ou isoladas, usar o Worker Fork Model do Claude Code em vez de sub-agents completos:

```
Worker Fork Rules (non-negotiable):
1. Worker E o fork. NAO spawna sub-agents
2. NAO conversa ou pergunta
3. USA tools diretamente (Bash, Read, Write)
4. Se modifica arquivos, commita antes de reportar
5. NAO emite texto entre tool calls
6. Fica dentro do escopo da diretiva
7. Report de max 500 palavras
8. Response DEVE comecar com "Scope:"
9. REPORTA fatos estruturados, entao para

Output format:
  Scope: <echo back assigned scope>
  Result: <answer or key findings>
  Key files: <relevant file paths>
  Files changed: <list with commit hash>
  Issues: <list if any>
```

**Quando usar cada modelo:**

| Cenario | Modelo | Custo Estimado |
|---------|--------|---------------|
| Mudanca de role na conversa | Persona switch | ~0 tokens extras |
| Tarefa paralela isolada | Worker fork | ~20K tokens + 500 palavras report |
| Tarefa sequencial simples | Persona switch | ~0 tokens extras |
| Pesquisa independente | Worker fork | ~20K tokens |
| Review de codigo | Worker fork | ~20K tokens |

### 6.3 Persona Design (4 Camadas)

Cada agent persona segue o modelo de 4 camadas:

**Camada 1 -- Identity (Quem)**
```yaml
identity:
  name: "Pixel"
  role: "Full-Stack Developer"
  archetype: "Craftsman"
  scope: "Implementacao, testes, refatoracao"
  NOT: "NAO faz push (delegar @devops), NAO valida stories (delegar @product-lead)"
```

**Camada 2 -- Behavioral Constraints (O que pode/nao pode)**
```yaml
constraints:
  hard_stops:
    - "NUNCA editar arquivo sem ler primeiro"
    - "NUNCA push sem delegar a @devops"
  uncertainty_handling: "Se nao tiver certeza, verificar com Read/Grep antes de agir"
  escalation: "Se bloqueado, escalar para orchestrator"
```

**Camada 3 -- Communication Style (Como fala)**
```yaml
communication:
  tone: "Tecnico mas acessivel"
  positive_example: "Implementei o endpoint /api/users com validacao Zod e testes unitarios"
  negative_example: "Fiz la o negocio"
  format: "Structured updates with file list and test results"
```

**Camada 4 -- Contextual Knowledge (O que sabe)**
```yaml
knowledge:
  static: "knowledge-base/*.md no squad"
  dynamic: "Runtime context via Read/Grep"
  domain: "TypeScript, React, Next.js, Supabase, etc."
```

### 6.4 Squad Sweet Spot

| Dimensao | Min | Ideal | Max |
|----------|-----|-------|-----|
| Agents por squad | 3 | 5-7 | 10 |
| Tasks por agent | 5 | 8-12 | 15 |
| Workflows por squad | 2 | 4-6 | 8 |
| Knowledge bases | 3 | 6-10 | 15 |
| Cross-squad connections | 1 | 3-5 | 8 |

### 6.5 Agent Selection Decision Tree

```
Request recebido
  |
  +-- E sobre codigo/implementacao? --> @developer
  |
  +-- E sobre arquitetura/tech choice? --> @architect
  |
  +-- E sobre qualidade/testes? --> @quality-gate
  |
  +-- E sobre criar story? --> @sprint-lead
  |
  +-- E sobre validar story? --> @product-lead
  |
  +-- E sobre epic/requirements? --> @project-lead
  |
  +-- E sobre database/schema? --> @data-engineer
  |
  +-- E sobre UX/UI? --> @ux-design-expert
  |
  +-- E sobre pesquisa/analise? --> @analyst
  |
  +-- E sobre git push/PR/deploy? --> @devops
  |
  +-- E sobre dominio de squad? --> Squad orchestrator
  |
  +-- Nao sei --> @sinapse-orqx diagnostica
```

### 6.6 Cost-Aware Delegation

| Custo da delegacao | Decisao |
|-------------------|---------|
| Persona switch (~0 tokens) | SEMPRE preferir para tarefas sequenciais |
| Worker fork (~20K tokens) | Apenas para tarefas paralelas ou isolamento necessario |
| Sub-agent completo (~30K+ tokens) | Evitar; usar worker fork |

---

## 7. Planning Pipeline v2

### 7.1 Tres Tracks Adaptativos

| Track | Criterio | Pipeline | Prompts |
|-------|----------|----------|---------|
| **Fast-track** | Bug fix trivial, typo, 1-3 files | Story auto-criada e auto-validada, implement direto | 1-3 |
| **Standard** | Feature, refactor, task definida | SDC completo: Create -> Validate -> Implement -> QA | 5-15 |
| **Heavy** | Initiative nova, multiple stories, alta complexidade | Spec Pipeline -> SDC por story -> QA Loop | 20-50+ |

**Criterios de selecao automatica:**

| Sinal | Fast-track | Standard | Heavy |
|-------|-----------|----------|-------|
| Files afetados | 1-3 | 4-15 | 16+ |
| Complexidade estimada | S | M-L | XL |
| Scope claro? | Sim | Sim | Precisa spec |
| Risk | Baixo | Medio | Alto |
| Dependencias externas | 0 | 0-2 | 3+ |

### 7.2 Execution Modes

| Mode | Descricao | Quando usar | Prompts |
|------|-----------|-------------|---------|
| **YOLO** | Autonomo, decisoes automaticas, decision log | Tasks simples com scope claro | 0-1 |
| **Interactive** | Checkpoints em decisoes, feedback do usuario | Default para features | 5-10 |
| **Pre-Flight** | Planejamento completo upfront antes de implementar | Tasks complexas ou arriscadas | 10-15 |

### 7.3 Quality Gates (3 Camadas)

| Camada | Tempo | Ferramenta | Decisao |
|--------|-------|-----------|---------|
| **Pre-commit** | ~30s | Type check + lint + secret scan | BLOCK se falhar |
| **PR Review** | ~5min | CodeRabbit + test suite + dependency audit | BLOCK se CRITICAL |
| **Human Review** | Variavel | Developer review + stakeholder approval | APPROVE / REQUEST CHANGES |

### 7.4 Self-Healing Loop

```
1. Agent gera codigo
2. Type check + lint executam automaticamente
3. Se falhar -> error output volta ao agent
4. Agent corrige baseado nos erros
5. Testes re-executam
6. Repete ate max 3 iteracoes (pre-commit) ou 5 iteracoes (QA Loop)
7. Se max atingido -> escala para humano
```

---

## 8. Anti-Hallucination Architecture

### 8.1 Verification-First Principle

```
NUNCA confie em output de LLM sem verificacao.
SEMPRE verifique contra ground truth (arquivos, testes, types).
SEMPRE audite com ferramentas automaticas.
```

### 8.2 Modelo de Defesa em 7 Camadas

```
Layer 1: Prompt Engineering
  - Anti-hallucination rules no CLAUDE.md
  - "Se nao tiver certeza, diga explicitamente"
  - Restricao de conhecimento externo
  - Chain-of-Thought para consistencia

Layer 2: Tool Grounding (Read before Edit)
  - NUNCA editar arquivo sem ler primeiro
  - NUNCA importar sem verificar package.json
  - NUNCA referenciar path sem verificar existencia
  - Grep para patterns antes de sugerir mudancas

Layer 3: Type Checking
  - TypeScript compiler em modo strict
  - Execucao automatica apos cada Edit
  - Captura phantom APIs e tipos incorretos

Layer 4: Linting
  - ESLint com rules do projeto
  - Captura patterns incorretos
  - Execucao automatica

Layer 5: Test Execution
  - Testes rodam apos mudancas significativas
  - TDAD: testes primeiro, AI implementa
  - Captura erros de logica (a camada mais importante)

Layer 6: Code Review (AI)
  - CodeRabbit como first-pass automatico
  - Self-healing loop max 2 iteracoes em CRITICAL
  - Documenta issues HIGH

Layer 7: Quality Gates (Human + Automation)
  - Pre-commit gate (30s): type + lint + secrets
  - PR gate (5min): CodeRabbit + tests + deps
  - Human review para logica de negocio
```

### 8.3 Rules Anti-Hallucination para CLAUDE.md

```markdown
## Anti-Hallucination (NON-NEGOTIABLE)

### Read Before Write
- NUNCA edite arquivo sem ler primeiro via Read tool
- NUNCA importe package sem verificar existencia via Grep/Glob
- NUNCA referencie path sem verificar existencia

### Uncertainty Protocol
- Se nao tiver certeza, declare explicitamente
- Preferir "nao sei" a fabricar resposta
- Se nao encontrar evidencia no codebase, diga isso

### Memory as Hints
- Entries de memoria sao HINTS, nao ground truth
- SEMPRE verificar contra codebase real antes de agir
- Atualizar memoria apenas APOS acoes bem-sucedidas
```

### 8.4 Hook Patterns Anti-Hallucination

| Hook | Evento | Acao |
|------|--------|------|
| verify-imports.cjs | PreToolUse (Write/Edit) | Verifica imports contra package.json |
| verify-file-paths.cjs | PreToolUse (Write/Edit) | Verifica paths referenciados existem |
| dependency-audit.cjs | PreToolUse (Bash: npm/pip install) | Verifica packages no registry (anti-slopsquatting) |
| type-check-on-edit.cjs | PostToolUse (Write/Edit) | Executa type checker apos edicao |
| secret-scan.cjs | PreToolUse (Write/Edit) | Bloqueia commits com secrets |

### 8.5 Metodologia "Vibe & Verify"

Padrao oficial do SINAPSE para desenvolvimento assistido por AI:

```
VIBE: AI gera codigo via prompts naturais
  |
VERIFY: Verificacao sistematica em 7 camadas
  |-> Type checking
  |-> Linting
  |-> Test execution
  |-> Security scanning (secret scan + dep audit)
  |-> Code review (CodeRabbit)
  |-> Human review de logica critica
  |-> Quality gates automaticos
```

**Calibracao por risco:**
- Prototipos, MVPs, tools internas -> Vibe coding agressivo
- Producao com dados sensiveis -> Engineering rigor completo + AI

---

## 9. Multi-LLM Compatibility

### 9.1 Plataformas Target (por prioridade)

| Prioridade | Plataforma | Paridade | Formato Principal |
|-----------|-----------|----------|-------------------|
| **P1** | Claude Code | Full (referencia) | CLAUDE.md + rules/ + hooks/ |
| **P2** | Codex CLI | Full | AGENTS.md + hooks.json |
| **P3** | Gemini CLI | Full | GEMINI.md + .gemini/agents/ |
| **P4** | Cursor | Rules only | .cursor/rules/*.mdc |
| **P4** | GitHub Copilot | Agents + instructions | .github/agents/ + copilot-instructions.md |
| **P5** | Windsurf | Rules only (12K limit) | .windsurf/rules/*.md |
| **P5** | Kimi Code | AGENTS.md only | AGENTS.md |
| **P5** | Kiro | Steering + agents | .kiro/steering/*.md |

### 9.2 Arquitetura de Transpilacao

```
SINAPSE Source of Truth
=======================
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
  +----+----+----+----+----+----+----+
  |    |    |    |    |    |    |    |
  v    v    v    v    v    v    v    v
CLAUDE AGENTS GEMINI CURSOR COPILOT WIND  KIMI KIRO
.md    .md   .md    rules  agents  rules  .md  steer
rules  hooks agents .mdc   .agent  .md
hooks  .json .md           .md
```

### 9.3 Estrategia de 4 Niveis

**Nivel 1: Universal (AGENTS.md)** -- funciona em 24+ ferramentas
- Constitution resumida
- Convencoes de codigo
- Build/test commands
- Seguranca basica

**Nivel 2: Tool-Specific Instructions** -- features avancadas
- CLAUDE.md com rules detalhadas
- GEMINI.md com @imports
- .cursor/rules/*.mdc com frontmatter
- .windsurf/rules/*.md (comprimido para 12K)

**Nivel 3: Agent Definitions** -- tools que suportam agents
- .gemini/agents/*.md
- .github/agents/*.agent.md
- Codex [agents] em config.toml

**Nivel 4: Hooks & Gates** -- enforcement automatico
- .claude/hooks/ (CJS scripts)
- .codex/hooks.json
- Gemini policy.toml

### 9.4 Mapeamento de Features

| Feature SINAPSE | Claude Code | Codex CLI | Gemini CLI | Cursor |
|-----------------|-------------|-----------|------------|--------|
| Constitution | CLAUDE.md | AGENTS.md | GEMINI.md | alwaysApply rules |
| Agent Definitions | Sub-agents | [agents] config | .gemini/agents/ | Agent rules |
| Hooks/Gates | 5 events | 5 events | policy.toml | Automations |
| Permissions | deny/allow | 3 modes | Tool wildcards | N/A |
| MCP | settings.json | config.toml | settings.json | Built-in |
| Memory | MEMORY.md | codex resume | /memory | Memories |

---

## 10. Installation & Onboarding

### 10.1 Flow do Instalador

```bash
npx sinapse-ai init
```

```
[1. WELCOME]
  "SINAPSE AI Framework v3 -- Setup"
  |
[2. LANGUAGE]
  > Portugues (pt-BR) | English (en-US) | Espanol (es)
  |
[3. LLM TARGET]
  > Claude Code (full -- recommended)
    Codex CLI (full)
    Gemini CLI (full)
    Cursor (rules only)
    GitHub Copilot (agents + instructions)
    Windsurf (rules only, 12K limit)
    Multiple (generate for all selected)
  |
[4. PROJECT TYPE]
  > Greenfield (novo projeto) | Brownfield (existente)
  |
[5. TEMPLATE] (se greenfield)
  > Next.js App Router | React + Vite | Node.js API | Python FastAPI | Custom
  |
[6. SQUAD SELECTION]
  Core: [x] Development (always included)
  Optional: [ ] Research | [ ] Brand | [ ] Content | [ ] Growth | ...
  |
[7. CONFIG GENERATION]
  Gerando configs para targets selecionados...
  |
[8. SUMMARY]
  Instalado! Rode `sinapse doctor` para verificar.
```

### 10.2 Modos de Instalacao

| Modo | Flag | Comportamento |
|------|------|---------------|
| Guided (default) | -- | Todas as perguntas, explicacoes |
| Expert | `--expert` | Opcoes avancadas, menos explicacoes |
| Quick | `--quick` | Aceita defaults, zero perguntas |
| From Preset | `--config preset.yaml` | Le preset e aplica |

### 10.3 Presets

```yaml
# sinapse.preset.yaml
name: "Startup SaaS"
language: pt-BR
llm_targets: [claude-code, cursor]
project_type: greenfield
framework: nextjs-app-router
squads: [development, research, brand, content, growth]
options:
  documentation_first: true
  safe_collaboration: true
  security_checks: true
```

### 10.4 Tecnologia do Instalador

- **UI:** @clack/prompts (80% menor que alternativas, usado por Astro/SvelteKit/T3)
- **Distribuicao:** npm (npx sinapse-ai init)
- **Deteccao:** Auto-detecta projeto existente, IDE instalada, git config

---

## 11. CLI UX & Branding

### 11.1 Metafora Central: A Orquestra

SINAPSE como uma orquestra onde cada agent e um musico, cada squad e uma secao, e o orchestrator e o maestro. A CLI reflete isso:

- `sinapse` = o maestro
- `sinapse squad` = secoes da orquestra
- `sinapse doctor` = afinacao
- `sinapse graph` = partitura visual

### 11.2 Convencoes de Comandos

```bash
# Framework
sinapse init                    # Setup
sinapse doctor                  # Health check
sinapse graph --deps            # Dependency visualization
sinapse graph --stats           # Entity stats

# Squads
sinapse squad add <name>        # Install squad
sinapse squad list              # List installed squads
sinapse squad validate <name>   # Quality check

# Skills
sinapse skill add <name>        # Install skill
sinapse skill list              # List skills
sinapse skill export <format>   # Export to SKILL.md format

# Config
sinapse config transpile        # Generate configs for all targets
sinapse config validate         # Validate configs
```

### 11.3 Agent Invocation

```
@developer          # Ativar agent
*help               # Comandos do agent
*develop            # Iniciar implementacao
*exit               # Sair do agent
```

### 11.4 Status Displays

Mensagens claras, sem jargao git:
- "Atualizando seu projeto... X mudancas novas."
- "Criada area segura para trabalhar: `caio/feat/xxx`"
- "Enviei para revisao. Matheus precisa aprovar no GitHub."
- "BLOQUEADO: encontrei API key em config.js. Removendo antes de salvar."

### 11.5 Error Messages

Pattern: O que aconteceu -> Por que -> Como resolver
```
BLOQUEADO: Story nao encontrada para docs/stories/
  Motivo: Constitution Article III exige story antes de implementacao
  Resolucao: Use @sprint-lead *draft para criar a story primeiro
```

---

## 12. Hooks Architecture v2

### 12.1 Todos os 9 Lifecycle Events

| Evento | Implementado v2 | Target v3 | Uso |
|--------|-----------------|-----------|-----|
| **SessionStart** | NAO | SIM | Auto-sync git, carregar memoria, verificar branch |
| **SessionEnd** | NAO | SIM | Memory consolidation, salvar scratchpad |
| **PreToolUse** | SIM (5 hooks) | SIM (expandir) | Security gates, budget check, delegation enforcement |
| **PostToolUse** | NAO | SIM | Type check apos Edit, gotcha capture em failures |
| **UserPromptSubmit** | SIM (1 hook) | SIM | Context injection |
| **PreCompact** | SIM (1 hook) | SIM | Session digest antes de compaction |
| **Stop** | NAO | SIM | Cleanup, salvar estado, memory flush |
| **Notification** | NAO | P3 | Informar mudancas externas |
| **SubagentStop** | NAO | P3 | Capturar resultado de workers |

### 12.2 Hooks Prioritarios v3

| Hook | Evento | Acao | Prioridade |
|------|--------|------|-----------|
| auto-sync.cjs | SessionStart | git fetch + branch check + memory load | P1 |
| memory-flush.cjs | Stop | Consolidar memoria, salvar scratchpad | P1 |
| verify-imports.cjs | PreToolUse (Write/Edit) | Validar imports contra package.json | P1 |
| secret-scan.cjs | PreToolUse (Write/Edit) | Bloquear secrets no codigo | P1 |
| type-check.cjs | PostToolUse (Write/Edit) | Type check apos edicao | P2 |
| token-budget.cjs | PreToolUse | Warning se instrucoes excedem 80% budget | P3 |
| dep-audit.cjs | PreToolUse (Bash: npm install) | Anti-slopsquatting | P1 |

### 12.3 Design Principles de Hooks

1. **Fail-open** -- Se hook crashar, exit 0 (allow)
2. **Fast** -- < 5 segundos por hook
3. **Silent on success** -- Output apenas em block ou warning
4. **Deterministic** -- Mesmo input, mesmo output
5. **No side effects** -- Hooks leem estado, nao modificam

---

## 13. Security Architecture

### 13.1 Verification-First para Code Generation

Toda geracao de codigo passa pelas 7 camadas de defesa (Secao 8.2).

### 13.2 Secret Scanning

- Hook PreToolUse em Write/Edit verifica patterns de secrets
- Integracao com gitleaks para scan completo pre-push
- Padroes detectados: .env files, API keys, tokens, private keys, DB connection strings, webhook URLs

### 13.3 Pre-Deploy Gates Automatizados (Tier 1)

Os 10 absolute blockers do Tier 1 devem ser automatizados como hooks:

| # | Blocker | Automacao |
|---|---------|-----------|
| 1 | Tabela sem RLS | SQL query check |
| 2 | API keys hardcoded | Hook: secret-scan |
| 3 | service_role no frontend | Grep hook |
| 4 | Sem MFA em admin | Manual (checklist) |
| 5 | APIs sem auth | Code review (CodeRabbit) |
| 6 | SQL string concatenation | Hook: sql-governance |
| 7 | Critical/high vuln deps | npm audit hook |
| 8 | Secrets no codebase | gitleaks hook |
| 9 | Credenciais default | Code review |
| 10 | Sem TLS | Deploy config check |

### 13.4 LGPD Compliance Tooling

Skill dedicada `sinapse-security-lgpd` com:
- Checklist de compliance (17 items do Tier 2)
- Templates de politica de privacidade
- Template de resposta a incidentes
- Guia de direitos do titular

---

## 14. File & Naming Standards

### 14.1 Naming Conventions

| Tipo | Padrao | Exemplo |
|------|--------|---------|
| Files/directories | kebab-case | `user-profile.ts`, `squad-research/` |
| React components | PascalCase | `UserProfile.tsx` |
| TypeScript types | PascalCase | `UserProfile`, `ApiResponse` |
| Variables/functions | camelCase | `getUserProfile()` |
| Constants | SCREAMING_SNAKE | `MAX_RETRY_COUNT` |
| Agent personas | PascalCase (single name) | Pixel, Litmus, Stratum |
| Skills | kebab-case com prefixo | `sinapse-security-lgpd` |
| Tasks | verb-object-qualifier | `create-story-next`, `validate-story-draft` |

### 14.2 Screaming Architecture (Top-Level)

```
project-root/
  docs/
    stories/         # Story files
    prd/             # Product requirements
    architecture/    # Architecture docs
    guides/          # User guides
  src/               # Application code
  tests/             # Test files
  squads/            # Installed squads
  .claude/           # Claude Code config
    settings.json
    CLAUDE.md
    rules/
    hooks/
  .sinapse/          # SINAPSE runtime (gitignored)
    handoffs/
    scratchpad/
    memories/
  .sinapse-ai/       # SINAPSE framework (L1-L2)
```

### 14.3 .env Management

- .env files SEMPRE em .gitignore
- .env.example com placeholders SEMPRE presente
- Validacao via T3 Env + Zod schema
- NEXT_PUBLIC_* sao PUBLIC -- nunca secrets

### 14.4 .gitignore Template

```gitignore
# SINAPSE Runtime
.sinapse/handoffs/
.sinapse/scratchpad/
.sinapse/memories/

# Secrets
.env
.env.local
.env.*.local

# Dependencies
node_modules/

# Build
dist/
.next/
out/

# OS
.DS_Store
Thumbs.db
```

---

## 15. Project Templates

### 15.1 Templates Disponíveis

| Template | Stack | Security Target | Performance Target |
|----------|-------|----------------|-------------------|
| **Landing Page** | Next.js + Tailwind + Framer Motion | HTTPS, CSP headers | LCP < 2.5s, CLS < 0.1 |
| **SaaS** | Next.js + Supabase + Stripe | RLS, MFA, LGPD | TTFB < 200ms |
| **E-commerce** | Next.js + Supabase + Stripe | PCI compliance, RLS | LCP < 2.5s |
| **Fintech** | Next.js + Supabase + edge functions | SOC2, encryption at rest, MFA | 99.9% uptime |
| **Portfolio** | Next.js + MDX + Tailwind | HTTPS, CSP | LCP < 1.5s |
| **Blog** | Next.js + MDX + Supabase | Auth, CSP | LCP < 2s |
| **Mobile** | React Native + Expo + Supabase | RLS, secure storage | 60fps |

### 15.2 Cada Template Inclui

- Folder structure scaffolded
- CLAUDE.md pre-configurado
- Rules relevantes
- .env.example com schema Zod
- CI/CD workflow (GitHub Actions)
- .gitignore completo
- README template
- First story de exemplo

---

## 16. Roadmap (Sprints)

### Sprint 0 -- Token Diet (P0, 1-2 dias)

**Objetivo:** Eliminar truncation silenciosa e reduzir overhead em 45%.

| Task | Descricao | Esforco |
|------|-----------|---------|
| 0.1 | Reescrever Global CLAUDE.md <= 3,200 chars | S |
| 0.2 | Reescrever Project CLAUDE.md <= 3,200 chars | S |
| 0.3 | Adicionar paths: frontmatter a 18 rules | S |
| 0.4 | Eliminar duplicacao entre CLAUDE.md e rules | M |
| 0.5 | Verificar total <= 12K chars | S |
| 0.6 | Unificar naming de personas (eliminar Dex/Quinn/etc) | S |

**Resultado:** ~14,000 tokens/turn economizados. Instrucoes criticas nao sao mais truncadas.

### Sprint 1 -- Skills System + Memory v2 (P1, 1 semana)

| Task | Descricao | Esforco |
|------|-----------|---------|
| 1.1 | Definir formato de skill SINAPSE (baseado em SKILL.md spec) | S |
| 1.2 | Migrar 12 rules para skills | M |
| 1.3 | Implementar skill loading on-demand | M |
| 1.4 | Implementar Memory 3-tier (HOT/WARM/COLD) | L |
| 1.5 | Implementar Dream Protocol (SessionEnd hook) | M |
| 1.6 | Adicionar "Memory as Hints" ao CLAUDE.md | S |
| 1.7 | Definir size limits para memoria | S |

**Resultado:** ~14,000 tokens adicionais economizados. Memoria inteligente com auto-consolidacao.

### Sprint 2 -- Multi-LLM + Installation (P1, 1 semana)

| Task | Descricao | Esforco |
|------|-----------|---------|
| 2.1 | Gerar AGENTS.md universal a partir de SINAPSE config | M |
| 2.2 | Transpiler para Codex CLI (AGENTS.md + hooks.json) | M |
| 2.3 | Transpiler para Gemini CLI (.gemini/agents/) | M |
| 2.4 | Transpiler para Cursor (.cursor/rules/*.mdc) | M |
| 2.5 | Transpiler para Copilot (.github/agents/) | M |
| 2.6 | Implementar `sinapse init` com @clack/prompts | L |
| 2.7 | Criar presets (Startup SaaS, Portfolio, etc.) | M |

**Resultado:** SINAPSE funcional em 5 plataformas. Onboarding em < 5 minutos.

### Sprint 3 -- Quality Gates + Security (P1, 1 semana)

| Task | Descricao | Esforco |
|------|-----------|---------|
| 3.1 | Hook SessionStart: auto-sync git + memoria | M |
| 3.2 | Hook PostToolUse: type check apos Edit | M |
| 3.3 | Hook PreToolUse: verify-imports (anti-slopsquatting) | M |
| 3.4 | Hook PreToolUse: secret-scan (gitleaks) | M |
| 3.5 | Automatizar Tier 1 pre-deploy gates | L |
| 3.6 | Implementar pipeline adaptativo (fast/standard/heavy) | M |
| 3.7 | Anti-hallucination rules no CLAUDE.md template | S |

**Resultado:** Quality gates automaticos em 7 camadas. Pipeline adaptativo por complexidade.

### Sprint 4 -- Templates + Documentation (P2, ongoing)

| Task | Descricao | Esforco |
|------|-----------|---------|
| 4.1 | Criar 7 project templates | L |
| 4.2 | Publicar skills no marketplace (sinapse-ai/skills) | M |
| 4.3 | Site de documentacao (Docusaurus/Nextra) | XL |
| 4.4 | Getting Started guide | M |
| 4.5 | Agent reference | M |
| 4.6 | Workflow reference | M |
| 4.7 | Usage tracking/observability | L |

**Resultado:** Documentacao publica, templates prontos, marketplace de skills.

---

## 17. Metricas de Sucesso

### 17.1 Metricas Tecnicas

| Metrica | Estado Atual | Target v3 | Metodo de Medicao |
|---------|-------------|-----------|-------------------|
| Token overhead per turn | ~31,000 | < 5,000 | Contagem de chars em CLAUDE.md + rules ativas |
| Rules carregadas por turn | 19/19 (100%) | 3-5/19 (~20%) | Path filtering hit rate |
| CLAUDE.md truncation | 40% perdido | 0% perdido | Verificar chars <= 4K/arquivo, 12K total |
| Memory token usage | Desconhecido | ~500-2,500 por turn | HOT + WARM budget tracking |
| Custo de instrucoes/turn | ~$0.48 sem cache | ~$0.075 sem cache | (tokens x pricing) |

### 17.2 Metricas de Produto

| Metrica | Target | Metodo de Medicao |
|---------|--------|-------------------|
| Story completion rate | > 90% | Stories Done / Stories Created |
| QA pass rate (first attempt) | > 70% | QA PASS / QA total |
| Time to first productive session (novo usuario) | < 15 min | Cronometro: install -> first commit |
| Cross-LLM compatibility score | >= 5 plataformas | Testes de transpilacao bem-sucedidos |
| Hallucination incidents per sprint | Tracking | Issues marcadas como hallucination |

### 17.3 Metricas de Ecossistema

| Metrica | Target v3 | Metodo |
|---------|-----------|--------|
| Skills publicadas | >= 15 | sinapse-ai/skills repo |
| Plataformas com transpilacao testada | >= 5 | CI/CD de transpilacao |
| Documentacao (paginas) | >= 30 | Site de docs |

---

## Apendice A: Mapeamento Gap -> Sprint

| Gap # | Descricao | Sprint | Status |
|-------|-----------|--------|--------|
| 1.2 | CLAUDE.md excede limites | Sprint 0 | TODO |
| 1.3 | Rules sem path filtering | Sprint 0 | TODO |
| 1.4 | Duplicacao global/project | Sprint 0 | TODO |
| 1.5 | Duplicacao rules/CLAUDE.md | Sprint 0 | TODO |
| 1.6 | Conteudo irrelevante | Sprint 0 | TODO |
| 10.1 | Naming inconsistente | Sprint 0 | TODO |
| 6.1 | Skills system | Sprint 1 | TODO |
| 3.1 | Memory consolidation | Sprint 1 | TODO |
| 3.2 | Memory relevance scoring | Sprint 1 | TODO |
| 3.3 | Memory size limits | Sprint 1 | TODO |
| 3.4 | Memory as hints | Sprint 1 | TODO |
| 2.1 | Multi-LLM transpilation | Sprint 2 | TODO |
| 9.1 | Installer NPX | Sprint 2 | TODO |
| 9.2 | Onboarding flow | Sprint 2 | TODO |
| 7.3 | Auto-sync SessionStart | Sprint 3 | TODO |
| 8.2 | Secret scanning | Sprint 3 | TODO |
| 8.4 | Pre-deploy gates | Sprint 3 | TODO |
| 4.1 | Pipeline adaptativo | Sprint 3 | TODO |
| 5.1 | Worker fork model | Sprint 3 | TODO |
| 11.1 | Project templates | Sprint 4 | TODO |
| 12.2 | Usage tracking | Sprint 4 | TODO |

---

## Apendice B: Decisoes Arquiteturais (ADRs)

### ADR-001: SKILL.md como formato de distribuicao
**Decisao:** Adotar SKILL.md (agentskills.io spec) como formato de distribuicao publica de skills SINAPSE.
**Razao:** 33 plataformas suportam o formato. Standard stewarded pela Anthropic.
**Consequencia:** Skills SINAPSE funcionam em Claude Code, Codex, Gemini CLI, Cursor e 29+ outras ferramentas.

### ADR-002: AGENTS.md como standard universal
**Decisao:** Gerar AGENTS.md como camada base de instrucoes, alem do CLAUDE.md.
**Razao:** AGENTS.md e stewarded pela Agentic AI Foundation (Linux Foundation), suportado por 24+ ferramentas.
**Consequencia:** Qualquer ferramenta que le AGENTS.md obtem instrucoes basicas do SINAPSE sem configuracao adicional.

### ADR-003: Personas unicas por role
**Decisao:** Uma persona por role, consistente em todos os projetos (Pixel, Litmus, Stratum, etc.).
**Razao:** Duplicidade atual (Pixel/Dex) desperdiça tokens e causa confusao.
**Consequencia:** Global e project CLAUDE.md usam os mesmos nomes. Eliminacao de ~200-300 tokens de tabela duplicada.

### ADR-004: Memory 3-tier model
**Decisao:** Adotar modelo HOT/WARM/COLD inspirado no AIOX MIS.
**Razao:** Reducao de 50-73% no consumo de tokens de memoria. Previne degradacao por memorias stale.
**Consequencia:** Memorias com baixo attention score nao poluem o contexto.

### ADR-005: Pipeline adaptativo (3 tracks)
**Decisao:** Substituir pipeline rigido (tudo passa pelo SDC completo) por 3 tracks adaptativos.
**Razao:** Bug fix de 1 linha nao deve custar 10K tokens em overhead de story.
**Consequencia:** Fast-track para fixes triviais economiza ~5-10K tokens/task.

### ADR-006: Verification-First como principio constitucional
**Decisao:** Elevar Verification-First a principio constitucional (Article XI).
**Razao:** Hallucination e matematicamente inevitavel. 19.7% de packages fabricados. 29-45% de vulnerabilidades em codigo AI.
**Consequencia:** 7 camadas de defesa obrigatorias em todo projeto SINAPSE.

---

*Blueprint compilado 2026-04-04 por Prism (Research Operations Conductor) | squad-research*
*Fontes: 6 documentos de pesquisa, 50+ gaps identificados, 40+ fontes primarias*
*Status: LIVING DOCUMENT -- atualizado continuamente*
