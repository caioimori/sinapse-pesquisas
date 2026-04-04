# SINAPSE Deep Research Initiative — Master Index

> **Status:** Phase 1 COMPLETE | Phase 2 COMPLETE | Phase 2.5 COMPLETE | Phase 3 (Blueprint) PENDING
> **Total Output:** 18,958 lines across 15 research documents
> **Agents Used:** 12 parallel research agents across 4 phases
> **Repository:** github.com/caioimori/caioimori-pesquisas (private)

---

## How to Navigate

```
docs/research/
├── INDEX.md                    ← VOCE ESTA AQUI
├── RESEARCH-STANDARD.md        ← Padrao de qualidade obrigatorio
├── EXECUTION-PLAN.md           ← Plano completo + como retomar
│
├── 01-claude-code/             ← Tudo sobre Claude Code
├── 02-token-economy-ai/        ← Tokens, AI, LLM, ML, alucinacoes
├── 03-frameworks-comparison/    ← AIOX, BMad, multi-LLM, NPM
├── 04-software-engineering/     ← Full-stack, animacoes, naming
├── 05-infrastructure/           ← Supabase, Vercel, security
├── 06-sinapse-improvement/      ← Gap analysis, roadmap
├── 07-skills-agents-swarm/      ← Skills, swarm, AGI
└── sources/                     ← Catalogos de repos
```

---

## 01-claude-code/ — Claude Code Internals

| File | Lines | Focus |
|------|-------|-------|
| [internals.md](01-claude-code/internals.md) | 1,026 | Leak event, 1,902 files, agent loop, system prompt, 24+ tools, permissions, hooks, memory, compaction, MCP, config, sessions, cost tracking, sandbox, security, multi-agent, 44 feature flags, bootstrap (12 phases), services (130 modules), skills, bridge (31 modules) |
| [architecture-deep-dive.md](01-claude-code/architecture-deep-dive.md) | 1,299 | KAIROS daemon, autoDream memory (4-phase), 3-layer compaction, prompt cache optimization (14 cache-break vectors), anti-distillation, undercover mode, model codenames (Capybara, Fennec=Opus 4.6) |
| [source-extractions.md](01-claude-code/source-extractions.md) | 298 | Raw data: Piebald catalog (33 agent prompts, 75+ system prompts, 14 skills, 30+ reminders, 24+ tools), Dream memory full text, Worker fork prompt, compaction template, pricing tables, architecture constants |

## 02-token-economy-ai/ — Tokens, AI & Machine Learning

| File | Lines | Focus |
|------|-------|-------|
| [token-economy-ai-fundamentals.md](02-token-economy-ai/token-economy-ai-fundamentals.md) | 1,225 | Tokenization (BPE), cost per operation, context strategies, YAML 20-30% more efficient than JSON, prompt caching ($1.50 vs $15/M), rule cost (~500-800 tokens/100 lines), agent overhead (min 20K/spawn), LLM fundamentals (transformers, attention, RLHF, Constitutional AI), multi-agent patterns, self-learning, second brain, LLM comparison |
| [hallucinations-prevention.md](02-token-economy-ai/hallucinations-prevention.md) | 918 | 3 root causes, code hallucinations (19.7% phantom packages), 6 prevention strategies (CoVe, RAG 40-96% reduction), detection (semantic entropy, HaluGate), Verification-First Architecture (7-layer defense), SINAPSE anti-hallucination patterns, vibe coding (92% adoption, "Vibe & Verify") |

## 03-frameworks-comparison/ — Frameworks & Multi-LLM

| File | Lines | Focus |
|------|-------|-------|
| [frameworks-npm-publishing.md](03-frameworks-comparison/frameworks-npm-publishing.md) | 1,371 | 15 frameworks (BMAD 43.5K stars, OpenHands 70.5K, Aider 42.8K), NPM publishing (semver, OIDC, Changesets, tsup), .env management (T3+Zod), monorepo (Turborepo) |
| [aiox-deep-analysis.md](03-frameworks-comparison/aiox-deep-analysis.md) | 1,417 | Memory Intelligence (4 layers, HOT/WARM/COLD, 73% token reduction), 14 workflows, 12 agents, 3-layer quality gates, L1-L4 architecture, IDE sync (6 IDEs), 11 security domains, NPM distribution |
| [multi-llm-squads-ux.md](03-frameworks-comparison/multi-llm-squads-ux.md) | 1,250 | 7 LLM CLIs (Claude Code, Codex, Gemini, Cursor, Copilot, Windsurf, Amazon Q), AGENTS.md universal standard (Linux Foundation, 20K+ repos), squad creation (4-8 agents sweet spot), persona design 4-layer, CLI UX (Vercel pattern) |

## 04-software-engineering/ — Full-Stack & Standards

| File | Lines | Focus |
|------|-------|-------|
| [fullstack-engineering-animations.md](04-software-engineering/fullstack-engineering-animations.md) | 3,073 | Architecture patterns (Clean, DDD, Hexagonal, Serverless + decision trees), SOLID + TypeScript, design patterns, Next.js App Router, TanStack Query + Zustand, testing (Vitest/Playwright/MSW), CI/CD, Disney's 12 principles, GSAP, Framer Motion, React Three Fiber, GLSL shaders, post-processing, 7 project templates (LP → Fintech → Mobile) |
| [naming-organization-scaffolding.md](04-software-engineering/naming-organization-scaffolding.md) | 1,174 | Naming golden rule, 5 directory paradigms (Screaming Architecture), scaffolding (Yeoman/Plop/Hygen), T3 Env + Zod, .gitignore template, GitHub Actions, Conventional Commits, Atomic Design + Tailwind 4, docs-as-code, productization, 3 validation checklists |

## 05-infrastructure/ — Infra, Security & DevOps

| File | Lines | Focus |
|------|-------|-------|
| [infra-cybersecurity-colaboracao.md](05-infrastructure/infra-cybersecurity-colaboracao.md) | 2,218 | Supabase (11 components, auth+MFA, RLS, Edge Functions, Realtime, Vault), Vercel (3-layer, Fluid Compute), OWASP Top 10 2025, auth security, XSS/CSRF/CSP, supply chain, LGPD, pen testing, git workflows, testing pyramid, Obsidian + RAG with pgvector |

## 06-sinapse-improvement/ — Gap Analysis & Roadmap

| File | Lines | Focus |
|------|-------|-------|
| [gap-analysis.md](06-sinapse-improvement/gap-analysis.md) | 733 | **50 gaps across 12 dimensions.** CRITICAL: ~40% CLAUDE.md truncated, 17K tokens unfiltered rules. Sprint 0 "Token Diet" = 14K tokens/turn savings (45%). Priority matrix: 4 P0, 18 P1, 16 P2, 12 P3 |

## 07-skills-agents-swarm/ — Skills, Agents, Swarm & AGI

| File | Lines | Focus |
|------|-------|-------|
| [skills-ecosystem-analysis.md](07-skills-agents-swarm/skills-ecosystem-analysis.md) | 595 | SKILL.md universal standard (33 platforms), anthropics/skills (110K stars), obra/superpowers (134K stars), 1,060+ skills catalogued, Progressive Disclosure pattern, SINAPSE is a category above all competitors |
| [llm-files-swarm-agi.md](07-skills-agents-swarm/llm-files-swarm-agi.md) | 1,794 | File maps for 7 LLM CLIs, swarm history (Reynolds 1986 → OpenAI Swarm 2025), 9 frameworks (CrewAI, AutoGen, LangGraph, MetaGPT), 7 orchestration patterns, AGI levels (OpenAI 5, DeepMind 6), 11 historical figures, timeline predictions (~2033), 7 fundamental books |

## sources/ — Reference Catalogs

| File | Lines | Focus |
|------|-------|-------|
| [github-repos-ecosystem.md](sources/github-repos-ecosystem.md) | 60 | 18 curated repos (anthropics/skills, openai/codex, awesome lists, community skills) |

---

## Key Sources

| Source | Stars | What We Extracted |
|--------|-------|-------------------|
| [Piebald-AI/claude-code-system-prompts](https://github.com/Piebald-AI/claude-code-system-prompts) | 8,197 | 200+ system prompts |
| [ultraworkers/claw-code](https://github.com/ultraworkers/claw-code) | 163,357 | Full Rust port |
| [h26liu/claude-code-unpacked](https://github.com/h26liu/claude-code-unpacked) | — | Architecture analysis |
| [SynkraAI/aiox-core](https://github.com/SynkraAI/aiox-core) | 2,583 | Competing framework |
| [anthropics/skills](https://github.com/anthropics/skills) | 110,000+ | Official skills repo |
| [obra/superpowers](https://github.com/obra/superpowers) | 134,000+ | Community skills |
| [aiox.academialendaria.ai](https://aiox.academialendaria.ai/materiais) | — | 61 articles |

---

## Top 20 Actionable Discoveries

### CRITICAL (P0)
1. ~40% do CLAUDE.md é truncado (limite 4K/arquivo, temos 13K)
2. 18/19 rules carregam SEMPRE (~17K tokens overhead)
3. Skills system AUSENTE (maior oportunidade: 14K tokens/turn)
4. Duplicação massiva entre CLAUDE.md e rules

### HIGH (P1)
5. SKILL.md é o formato universal (33 plataformas) — adotar
6. AGENTS.md é standard da Linux Foundation — gerar
7. Dream Memory Consolidation (4-phase automated)
8. AIOX Memory Intelligence (73% token reduction)
9. Worker Fork Model (500-word structured reports)
10. Planning pipeline precisa fast-track
11. Auto-sync não enforced (falta SessionStart hook)
12. NPX installer missing (blocks adoption)

### MEDIUM (P2)
13. Deferred Tools (93% reduction in tool descriptions)
14. Document sharding (70-80% savings)
15. 4-8 agents per squad is sweet spot
16. Verification-First Architecture (7-layer defense)
17. Persona design 4-layer model
18. CLI UX: speed > delight (Vercel pattern)

### STRATEGIC (P3)
19. KAIROS autonomous daemon (future vision)
20. AGI preparation (progressive autonomy)

---

## Meta Documents

| File | Purpose |
|------|---------|
| [INDEX.md](INDEX.md) | This file — master navigation |
| [RESEARCH-STANDARD.md](RESEARCH-STANDARD.md) | Quality template (auto-applied) |
| [EXECUTION-PLAN.md](EXECUTION-PLAN.md) | Full plan + session continuity |

---

*SINAPSE Deep Research Initiative — 2026-04-04*
*18,958 lines | 15 documents | 50 gaps | 12 agents | 8 commits*
