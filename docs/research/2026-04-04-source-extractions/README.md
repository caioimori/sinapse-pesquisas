# Source Extractions — Raw Data from Repos

> **Date:** 2026-04-04
> **Purpose:** Raw extracted data from key repositories for SINAPSE framework improvement

---

## 1. Piebald System Prompts Catalog (8,197 stars)

**Repo:** `Piebald-AI/claude-code-system-prompts`
**Description:** All parts of Claude Code's system prompt, 24+ builtin tool descriptions, sub agent prompts, utility prompts. Updated per CC version.

### 1.1 Agent Prompts (33 prompts)

| Prompt | Purpose |
|--------|---------|
| `agent-creation-architect` | Creates new custom agents |
| `agent-hook` | Hook condition evaluation for agents |
| `auto-mode-rule-reviewer` | Reviews rules for auto mode |
| `bash-command-description-writer` | Writes descriptions for bash commands |
| `bash-command-prefix-detection` | Detects command prefixes |
| `batch-slash-command` | Batch processing of slash commands |
| `claude-guide-agent` | Guide agent for Claude Code help |
| `claudemd-creation` | Creates CLAUDE.md files |
| `coding-session-title-generator` | Generates session titles |
| `conversation-summarization` | Summarizes conversations for compaction |
| `determine-which-memory-files-to-attach` | Memory file relevance scoring |
| `dream-memory-consolidation` | 4-phase memory consolidation (Orient/Gather/Consolidate/Prune) |
| `explore` | Codebase exploration agent |
| `general-purpose` | General purpose subagent |
| `hook-condition-evaluator-stop` | Evaluates stop conditions |
| `plan-mode-enhanced` | Enhanced planning mode |
| `prompt-suggestion-generator-v2` | Suggests next prompts |
| `quick-git-commit` | Quick commit creation |
| `quick-pr-creation` | Quick PR creation |
| `recent-message-summarization` | Summarizes recent messages |
| `review-pr-slash-command` | PR review command |
| `schedule-slash-command` | Schedule/cron command |
| `security-monitor-first-part` | Security monitoring (part 1) |
| `security-monitor-second-part` | Security monitoring (part 2) |
| `security-review-slash-command` | Security review command |
| `session-memory-update-instructions` | How to update session memory |
| `session-search-assistant` | Search across sessions |
| `session-title-and-branch-generation` | Title + branch naming |
| `status-line-setup` | Status line configuration |
| `verification-specialist` | Verification/testing specialist |
| `webfetch-summarizer` | Summarizes web fetch results |
| `worker-fork-execution` | Forked worker subprocess |

### 1.2 System Prompts (75+ prompts)

Key categories:
- **Doing Tasks** (11 prompts): ambitious-tasks, help-feedback, minimize-file-creation, no-compatibility-hacks, no-premature-abstractions, no-time-estimates, no-unnecessary-additions, no-unnecessary-error-handling, read-before-modifying, security, software-engineering-focus
- **Tool Usage** (12 prompts): create-files, delegate-exploration, direct-search, edit-files, read-files, reserve-bash, search-content, search-files, skill-invocation, subagent-guidance, task-management
- **Memory** (4 prompts): agent-memory-instructions, description-part, memory-description-of-user-feedback, memory-file-contents
- **Modes** (6 prompts): auto-mode, buddy-mode, learning-mode, minimal-mode, plan-mode-phase-four, remote-plan-mode-ultraplan
- **Worker/Team** (3 prompts): worker-instructions, teammate-communication, fork-usage-guidelines
- **Efficiency** (2 prompts): output-efficiency, tone-and-style
- **Context** (3 prompts): compaction-summary, partial-compaction-instructions, scratchpad-directory
- **Insights** (5 prompts): at-a-glance-summary, friction-analysis, on-the-horizon, session-facets, suggestions
- **Security** (2 prompts): censoring-malicious, executing-actions-with-care

### 1.3 Skills (14 prompts)

| Skill | Purpose |
|-------|---------|
| `agent-design-patterns` | Patterns for creating agents |
| `build-with-claude-api` | Building with Claude API |
| `build-with-claude-api-reference-guide` | API reference guide |
| `computer-use-mcp` | Computer use via MCP |
| `create-verifier-skills` | Creating verification skills |
| `debugging` | Debug assistance |
| `init-claudemd-and-skill-setup` | Initialize CLAUDE.md + skills |
| `loop-slash-command` | Loop/recurring execution |
| `simplify` | Code simplification/cleanup |
| `stuck-slash-command` | Help when stuck |
| `update-claude-code-config` | Config updates |
| `update-config-7-step-verification` | 7-step config verification |
| `verify-cli-changes-example` | CLI verification example |
| `verify-skill` | Verification skill |

### 1.4 System Reminders (30+ reminders)

Runtime-injected contextual messages:
- `agent-mention` — When agent is mentioned
- `compact-file-reference` — File refs during compaction
- `exited-plan-mode` — After leaving plan mode
- `file-exists-but-empty` — Empty file warning
- `file-modified-by-user-or-linter` — External file change
- `file-opened-in-ide` — IDE file open notification
- `hook-blocking-error` — Hook block message
- `hook-stopped-continuation` — Hook stopped action
- `invoked-skills` — Active skills list
- `lines-selected-in-ide` — IDE selection context
- `malware-analysis-after-read` — Security scan after read
- `memory-file-contents` — Memory file injection
- `plan-mode-is-active-5-phase` — 5-phase plan mode
- `plan-mode-is-active-iterative` — Iterative plan mode
- `plan-mode-is-active-subagent` — Subagent plan mode
- `session-continuation` — Session resume
- `task-tools-reminder` — Task tools nudge
- `team-coordination` — Team coordination
- `todowrite-reminder` — Todo reminder
- `token-usage` — Usage stats
- `usd-budget` — Budget tracking
- `verify-plan-reminder` — Plan verification nudge

### 1.5 Tool Descriptions (24+ tools)

Full tool list from extracted prompts:
AskUserQuestion, Bash, Computer, Config, CronCreate, Edit, EnterPlanMode, EnterWorktree, ExitPlanMode, ExitWorktree, Glob, Grep, LSP, NotebookEdit, PowerShell, ReadFile, SendMessageTool, Skill, Sleep, Task, TaskCreate, TeamDelete, TeammateTool, TodoWrite, ToolSearch, WebFetch, WebSearch, Write

### 1.6 Data References (16 data files)

- Agent SDK patterns (Python + TypeScript)
- Agent SDK reference (Python + TypeScript)
- Claude API reference (C, curl, Go, Java, PHP, Python, Ruby, TypeScript)
- Claude model catalog
- Files API reference
- GitHub Actions workflow
- HTTP error codes reference
- Live documentation sources
- Message batches API reference
- **Prompt caching design optimization** ← KEY for token economy
- Session memory template
- Streaming reference

---

## 2. Dream Memory Consolidation (Full Text)

```
Phase 1 — Orient
- ls the memory directory
- Read index file
- Skim existing topic files

Phase 2 — Gather recent signal
- Daily logs (logs/YYYY/MM/YYYY-MM-DD.md)
- Existing memories that drifted
- Transcript search (grep JSONL narrowly)

Phase 3 — Consolidate
- Write/update memory files
- Merge new signal into existing files
- Convert relative dates to absolute
- Delete contradicted facts

Phase 4 — Prune and index
- Update index under max lines + 25KB
- Each entry: one line, ~150 chars
- Remove stale pointers
- Demote verbose entries
- Add new important memories
- Resolve contradictions
```

---

## 3. Worker Fork Execution (Full System Prompt)

```
RULES (non-negotiable):
1. You ARE the fork. Do NOT spawn sub-agents
2. Do NOT converse or ask questions
3. USE tools directly: Bash, Read, Write
4. If modify files, commit before reporting
5. Do NOT emit text between tool calls
6. Stay within directive scope
7. Keep report under 500 words
8. Response MUST begin with "Scope:"
9. REPORT structured facts, then stop

Output format:
  Scope: <echo back assigned scope>
  Result: <answer or key findings>
  Key files: <relevant file paths>
  Files changed: <list with commit hash>
  Issues: <list if any>
```

---

## 4. Context Compaction Summary (Full Text)

```
Write a continuation summary including:
1. Task Overview — core request and success criteria
2. Current State — what completed, files modified, outputs
3. Important Discoveries — constraints, decisions, errors, failed approaches
4. Next Steps — specific actions, blockers, priority order
5. Context to Preserve — preferences, domain details, promises

Wrap in <summary></summary> tags.
```

---

## 5. Claw-Code Architecture Constants

From Rust port analysis:

| Constant | Value | Purpose |
|----------|-------|---------|
| `preserve_recent_messages` | 4 | Messages kept verbatim during compaction |
| `max_estimated_tokens` | 10,000 | Threshold to trigger compaction |
| `MAX_INSTRUCTION_FILE_CHARS` | 4,000 | Max chars per CLAUDE.md file |
| `MAX_TOTAL_INSTRUCTION_CHARS` | 12,000 | Total budget for all instruction files |
| Token estimation | `len / 4 + 1` | Rough token estimate from char count |

### Pricing (per million tokens)

| Model | Input | Output | Cache Create | Cache Read |
|-------|-------|--------|-------------|------------|
| Opus 4.6 | $15.00 | $75.00 | $18.75 | $1.50 |
| Sonnet 4.6 | $15.00 | $75.00 | $18.75 | $1.50 |
| Haiku 4.5 | $1.00 | $5.00 | $1.25 | $0.10 |

### MCP Transport Types
Stdio, SSE, HTTP, WebSocket, SDK, ManagedProxy

### Permission Modes
ReadOnly, WorkspaceWrite, DangerFullAccess, Prompt, Allow

### Hook Exit Codes
0 = Allow, 2 = Deny/Block, Other = Warn (treated as allow)

### Config Sources (merge priority)
User → Project → Local (Local overrides Project overrides User)

---

## 6. AIOX Framework Comparison

**Repo:** `SynkraAI/aiox-core` (2,583 stars, 2,647 files)

### Architecture Similarities with SINAPSE

| Feature | AIOX | SINAPSE |
|---------|------|---------|
| Constitution | Yes (.aiox-core/constitution.md) | Yes (.sinapse-ai/constitution.md) |
| Agent count | 12 | 10 (framework) + 186 (squads) |
| Workflows | 14 pre-built | 4 primary + custom |
| CLI | aiox CLI (JavaScript) | sinapse CLI (JavaScript) |
| NPM distribution | npm install -g aiox-core | npm install -g sinapse-ai |
| Quality gates | 3-layer | Story-driven + QA gate |
| Memory system | 4-layer (73% token reduction) | Auto memory + MEMORY.md |
| Squads | Dynamic creation | 18 specialized squads |
| Agent personas | Named (Dex, Quinn, Aria...) | Named (same pattern) |
| Story-driven dev | Yes (Article III) | Yes (Article III) |
| Documentation hub | 61 articles at academialendaria.ai | Internal docs |

### AIOX-Specific Features Worth Studying

1. **Memory Intelligence System** — 4 layers: Capture → Storage → Retrieval → Evolution (73% token reduction)
2. **HOT/WARM/COLD tier** — Memory temperature-based retrieval
3. **11 Security Domains** — API keys, env vars, sandboxing, etc.
4. **NPX Installer** — `npx @synkra/aiox-install` for easy onboarding
5. **Squad Creator Agent** — Meta-agent that creates new squads dynamically
6. **Agent Selection Guide** — Decision tree for choosing the right agent
7. **Learning Paths** — Vertical learning tracks from problem to result
8. **Community Contribution Flow** — RFC + PR workflow for community squads

### AIOX File Structure (2,647 files)

```
.aiox-core/
├── cli/commands/          # CLI commands (config, generate, manifest, mcp, metrics, migrate, qa, validate, workers)
├── constitution.md        # Core principles
├── core-config.yaml       # Configuration
├── core/
│   ├── code-intel/        # Code intelligence (providers, helpers, enricher)
│   ├── config/            # Config system (loader, resolver, schemas, templates)
│   ├── doctor/            # Health checks (15+ checks)
│   ├── elicitation/       # Interactive elicitation engine
│   ├── graph-dashboard/   # Dependency visualization
│   └── docs/              # Internal docs
├── development/
│   ├── agents/            # Agent definitions
│   ├── tasks/             # Task definitions
│   ├── templates/         # Document templates
│   ├── checklists/        # Quality checklists
│   └── workflows/         # Workflow definitions
└── infrastructure/        # CI/CD templates
```

---

## 7. Key Sources

| Source | URL | Type |
|--------|-----|------|
| Piebald System Prompts | github.com/Piebald-AI/claude-code-system-prompts | Extracted prompts catalog |
| Claw-Code (Rust port) | github.com/ultraworkers/claw-code | Clean-room rewrite |
| Claude Code Unpacked | github.com/h26liu/claude-code-unpacked | Architecture analysis |
| AIOX-Core | github.com/SynkraAI/aiox-core | Competing framework |
| AIOX Academy | aiox.academialendaria.ai/materiais | Documentation hub |
| System Prompts Leaks | github.com/asgeirtj/system_prompts_leaks | Multi-LLM prompts |
