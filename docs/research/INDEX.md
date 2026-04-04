# SINAPSE Deep Research Initiative — Master Index

> **Date:** 2026-04-04
> **Status:** Phase 1 COMPLETE | Phase 2 COMPLETE | Phase 3 (Blueprint) PENDING
> **Total Output:** 10,868 lines across 10 documents
> **Agents Used:** 8 parallel research agents
> **Sources Analyzed:** 6 repos, 15+ websites, 200+ system prompts

---

## How to Navigate This Research

1. **Start here** — Read this INDEX for overview
2. **For Claude Code internals** — Read Streams 1 + Architecture Deep Dive + Source Extractions
3. **For token optimization** — Read Stream 2 + Gap Analysis (Section 1)
4. **For infrastructure** — Read Stream 4 (Supabase, Vercel, Security)
5. **For SINAPSE improvements** — Read Gap Analysis (50 gaps prioritized)
6. **For organization standards** — Read Naming & Scaffolding
7. **For AIOX comparison** — Read AIOX Deep Analysis + Source Extractions (Section 6)

---

## Phase 1 — Collection (COMPLETE)

### Stream 1: Claude Code Internals & Source Code
**File:** [`2026-04-04-claude-code-internals/README.md`](2026-04-04-claude-code-internals/README.md)
**Lines:** 1,026 | **Sections:** 28
**Covers:** Leak event, codebase scale (1,902 files, 512K LOC), agent loop, system prompt engineering, tool system (24+ tools), permission system (5 modes), hook system (9 lifecycle events), memory architecture (Dream consolidation), compaction (4 messages preserved, 10K threshold), MCP (6 transport types), config (3-source merge), sessions, cost tracking (per-model pricing), sandbox (3 isolation modes), security (9,707-line bash parser), multi-agent/coordinator, feature flags (44 hidden), anti-distillation, bootstrap (12-phase sequence), services (130 modules), skills (14 bundled), bridge/IDE (31 modules)

### Stream 2: Token Economy & AI/LLM Fundamentals
**File:** [`token-economy-ai-fundamentals.md`](token-economy-ai-fundamentals.md)
**Lines:** 1,225 | **Sections:** 20+
**Covers:** Tokenization (BPE), cost per operation, context window strategies, compaction mechanics, file format efficiency (YAML 20-30% more efficient than JSON), prompt caching ($1.50 vs $15.00/M tokens), batch vs sequential operations, rule injection cost (~500-800 tokens per 100 lines), agent overhead (min 20K tokens per spawn), memory optimization, LLM fundamentals (transformers, attention, RLHF, Constitutional AI), tool use, chain-of-thought, multi-agent patterns (ReAct, Plan-and-Execute), self-learning, second brain concepts, LLM comparison (Claude vs GPT vs Gemini)

### Stream 3: Frameworks, NPM & Project Organization
**File:** [`ai-frameworks-npm-publishing-deep-dive.md`](ai-frameworks-npm-publishing-deep-dive.md)
**Lines:** 1,371 | **Sections:** 20+
**Covers:** 15 frameworks compared (BMAD 43.5K stars, OpenHands 70.5K, Aider 42.8K, AIOX 2.6K), NPM publishing (semver, CI/CD with OIDC, scoped packages, Changesets for monorepo, tsup for dual ESM/CJS), file naming conventions, directory structures (5 paradigms), .env management (T3+Zod), .gitignore, monorepo patterns (Turborepo), template/scaffold systems

### Stream 4: Infrastructure, Security & Collaboration
**File:** [`infraestrutura-profissional-cybersecurity-colaboracao.md`](infraestrutura-profissional-cybersecurity-colaboracao.md)
**Lines:** 2,218 | **Sections:** 30+
**Covers:** Supabase complete (11 components, auth+MFA, RLS with 99.99% improvement examples, Edge Functions, Realtime, Storage, Vault, performance, pricing), Vercel complete (3-layer architecture, Fluid Compute, deploy, Edge, CI/CD, analytics), OWASP Top 10 2025, auth security, API security, XSS/CSRF/CSP, supply chain attacks, LGPD compliance, pen testing (ZAP vs Burp), incident response, git workflows (Git Flow vs GitHub Flow vs Trunk-Based), CI/CD pipelines, testing pyramid, Obsidian second brain + RAG with pgvector

### Stream 5: Source Extractions (Raw Data)
**File:** [`2026-04-04-source-extractions/README.md`](2026-04-04-source-extractions/README.md)
**Lines:** 298 | **Sections:** 7
**Covers:** Piebald catalog complete inventory (33 agent prompts, 75+ system prompts, 14 skills, 30+ system reminders, 24+ tool descriptions, 16 data references), Dream memory consolidation full 4-phase text, Worker fork execution full system prompt, Context compaction summary template, Architecture constants (pricing tables, char limits, config sources), AIOX comparison table, all source URLs

### Supplementary: Architecture Deep Dive
**File:** [`claude-code-architecture-deep-dive.md`](claude-code-architecture-deep-dive.md)
**Lines:** 1,299 | **Sections:** 15+
**Covers:** KAIROS autonomous daemon (150+ references, 15s blocking budget), autoDream memory consolidation (4-phase with triple gate), 3-layer compaction (Micro/Auto/Full), prompt cache optimization (14 cache-break vectors), anti-distillation mechanisms, undercover mode, 44 feature flags with codenames (Capybara, Fennec=Opus 4.6, Tengu, Numbat)

---

## Phase 2 — Synthesis (COMPLETE)

### Gap Analysis — SINAPSE vs Claude Code vs AIOX
**File:** [`2026-04-04-sinapse-gap-analysis/README.md`](2026-04-04-sinapse-gap-analysis/README.md)
**Lines:** 733 | **Gaps Found:** 50 | **Dimensions:** 12
**Covers:** Token economy (CRITICAL: ~40% CLAUDE.md truncated, 17K tokens overhead from unfiltered rules), context engineering (no static/dynamic separation, no document sharding), memory system (no auto-consolidation, no relevance scoring, no HOT/WARM/COLD tiers), planning pipeline (rigid vs adaptive, missing fast-track), agent architecture (20K+ tokens per spawn, 175 agents vs value), skills system (ABSENT — biggest opportunity), hooks (5 of 9 lifecycle events used), security (no AST bash parser, incomplete secret scanning), NPM distribution (no npx installer, no onboarding flow), naming inconsistencies (different personas per project), project scaffolding (no sinapse init), productization (no multi-tenancy, no usage tracking)

**Priority Matrix:** 4 P0 (immediate), 18 P1 (next sprint), 16 P2 (backlog), 12 P3 (future)
**Sprint 0 "Token Diet":** ~14,000 tokens/turn savings (45% reduction), effort S/M

### AIOX Deep Analysis
**File:** [`2026-04-04-aiox-deep-analysis/README.md`](2026-04-04-aiox-deep-analysis/README.md)
**Lines:** 1,417 | **Domains:** 8 | **Recommendations:** 13
**Covers:** Memory Intelligence System (4 sub-layers: Capture/Storage/Retrieval/Evolution, attention scoring formula, HOT/WARM/COLD/ARCHIVE tiers, cognitive sectors, progressive disclosure for 73% token reduction, self-learning with confidence scoring), 14+ workflows (SDC 4-phase, QA Loop max 5, Spec Pipeline 6-phase, Brownfield 10-phase), 12 agents with ADE commands, 3-layer quality gates (pre-commit 30s, PR automation 5min, human review), L1-L4 architecture model, IDE sync for 6 IDEs, squad marketplace, 11 security domains, NPM distribution (package.json, binaries, installation flow)

### Naming, Organization & Scaffolding
**File:** [`2026-04-04-naming-organization-scaffolding/README.md`](2026-04-04-naming-organization-scaffolding/README.md)
**Lines:** 1,174 | **Sections:** 12
**Covers:** Naming golden rule (file reflects export), 5 directory paradigms (Screaming Architecture recommended), scaffolding comparison (Yeoman vs Plop vs Hygen), T3 Env + Zod for .env validation, comprehensive .gitignore template, GitHub Actions CI/CD pipelines, Conventional Commits spec, PR/issue templates, Atomic Design + Tailwind CSS 4, docs-as-code (Docusaurus vs VitePress vs Mintlify), productization structure (multi-tenancy, billing, marketplace), 3 validation checklists

---

## Key Sources

| Source | Stars | What We Extracted |
|--------|-------|-------------------|
| [Piebald-AI/claude-code-system-prompts](https://github.com/Piebald-AI/claude-code-system-prompts) | 8,197 | 200+ system prompts, 24+ tool descriptions, agent prompts |
| [ultraworkers/claw-code](https://github.com/ultraworkers/claw-code) | 163,357 | Full Rust implementation (compact, prompt, conversation, hooks, MCP, permissions, session, usage) |
| [h26liu/claude-code-unpacked](https://github.com/h26liu/claude-code-unpacked) | — | Architecture analysis, CLI rebuild, leaked tools inventory |
| [SynkraAI/aiox-core](https://github.com/SynkraAI/aiox-core) | 2,583 | Framework comparison (2,647 files), constitution, 12 agents, 14 workflows |
| [aiox.academialendaria.ai](https://aiox.academialendaria.ai/materiais) | — | 61 articles: memory (4 layers), security (11 domains), workflows, squads |
| [asgeirtj/system_prompts_leaks](https://github.com/asgeirtj/system_prompts_leaks) | — | Multi-LLM system prompts (GPT-5, Claude, Gemini, Grok) |

---

## Quick Reference: Top 20 Discoveries

### CRITICAL (act immediately)
1. **~40% do CLAUDE.md e truncado silenciosamente** — limite 4K/arquivo, temos 13K
2. **18/19 rules carregam SEMPRE** — ~17K tokens de overhead fixo desnecessario
3. **Duplicacao massiva** — global/project CLAUDE.md + rules repetem conteudo
4. **Skills system AUSENTE** — a maior oportunidade de economia (14K tokens/turn quando movido)

### HIGH (next sprint)
5. **Compaction preserves only 4 messages** — Claude Code's internal limit
6. **Cache read 10x cheaper** ($1.50 vs $15/M) — structure for cache hits
7. **Dream Memory Consolidation** — 4-phase automated (Orient/Gather/Consolidate/Prune)
8. **AIOX Memory Intelligence** — HOT/WARM/COLD tiers, 73% token reduction
9. **Worker Fork Model** — isolated workers with 500-word structured reports
10. **Planning pipeline rigid** — needs fast-track for trivial fixes
11. **Auto-sync not enforced** — safe-collaboration rule exists but no SessionStart hook
12. **NPX installer missing** — blocks adoption

### MEDIUM (backlog)
13. **200+ modular prompt fragments** in Claude Code — conditional loading
14. **Document sharding** (BMAD) — 70-80% savings per document
15. **Deferred Tools pattern** — 93% reduction in tool descriptions
16. **9 hook lifecycle events** — we use 5
17. **Agent personas inconsistent** — different names per project
18. **No usage tracking** — can't identify inefficient workflows

### STRATEGIC (future)
19. **KAIROS daemon** — autonomous agent with GitHub webhooks
20. **44 feature flags** — progressive rollout system

---

## Pending: Phase 3 — Blueprint

- [ ] SINAPSE-AI v3 Blueprint (consolidated architecture)
- [ ] Sprint 0 "Token Diet" execution plan
- [ ] Improvement Roadmap with epics/stories
- [ ] Reorganize research into subfolders by topic

---

*Research Initiative completed 2026-04-04. Total: 10,868 lines, 10 documents, 50 gaps identified, 6 repos analyzed.*
