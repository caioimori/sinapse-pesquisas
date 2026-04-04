# Claude Code Internals — Comprehensive Research Document

> **Research Date:** 2026-04-04
> **Sources:** claw-code clean-room rewrite, leaked source analyses, architecture deep dives, competing frameworks
> **Purpose:** Understanding Claude Code's production architecture for SINAPSE framework improvement

---

## Table of Contents

1. [The Leak Event](#1-the-leak-event)
2. [Codebase Scale & Structure](#2-codebase-scale--structure)
3. [Core Architecture — The Agent Loop](#3-core-architecture--the-agent-loop)
4. [System Prompt & Context Engineering](#4-system-prompt--context-engineering)
5. [Tool System](#5-tool-system)
6. [Permission System](#6-permission-system)
7. [Hook System](#7-hook-system)
8. [Memory Architecture](#8-memory-architecture)
9. [Compaction System](#9-compaction-system)
10. [MCP Integration](#10-mcp-integration)
11. [Configuration System](#11-configuration-system)
12. [Session Management](#12-session-management)
13. [Cost & Usage Tracking](#13-cost--usage-tracking)
14. [Sandbox System](#14-sandbox-system)
15. [Security Architecture](#15-security-architecture)
16. [Multi-Agent & Coordinator Mode](#16-multi-agent--coordinator-mode)
17. [Unreleased Systems (Feature-Flagged)](#17-unreleased-systems-feature-flagged)
18. [Anti-Distillation & Undercover Mode](#18-anti-distillation--undercover-mode)
19. [Bootstrap & Startup Sequence](#19-bootstrap--startup-sequence)
20. [Services Layer](#20-services-layer)
21. [Skills System](#21-skills-system)
22. [Bridge System (IDE Integration)](#22-bridge-system-ide-integration)
23. [Complete Subsystem Inventory](#23-complete-subsystem-inventory)
24. [Complete Command Inventory](#24-complete-command-inventory)
25. [Complete Tool Inventory](#25-complete-tool-inventory)
26. [Competing Frameworks Analysis](#26-competing-frameworks-analysis)
27. [Architectural Lessons for SINAPSE](#27-architectural-lessons-for-sinapse)
28. [Sources](#28-sources)

---

## 1. The Leak Event

On **March 31, 2026**, Anthropic accidentally published Claude Code's entire source code inside npm package v2.1.88. A **59.8 MB JavaScript source map file** (.map), intended for internal debugging, was inadvertently included due to a missing `*.map` entry in `.npmignore`.

**Root cause:** Claude Code is built on Bun (which Anthropic acquired in late 2025). Bun generates source maps by default. The release team failed to exclude debugging artifacts.

**Scale of exposure:**
- **512,000+ lines** of TypeScript
- **1,902 source files**
- **207 commands**
- **184 tools**
- **35 subsystems**
- **44 hidden feature flags** covering 20+ unshipped features

**Community response:** A clean-room rewrite (ultraworkers/claw-code) hit **50,000 GitHub stars in 2 hours** — the fastest-growing repository in GitHub history. It currently has **163,000 stars** and **101,000 forks**.

**Anthropic response:** DMCA takedown requests were issued across thousands of GitHub repositories. Anthropic confirmed it was a packaging error, not a security breach — no customer data or credentials were exposed.

---

## 2. Codebase Scale & Structure

### Root-Level Files (18)
- `main.tsx` — 785KB entry point with custom React terminal renderer
- `QueryEngine.ts` — 46,000-line LLM API engine
- `Tool.ts` — Type definitions
- `commands.ts` — Command registry
- `tools.ts` — Tool registry
- `context.ts` — System/user context
- `cost-tracker.ts` — Cost tracking
- `query.ts` — Query processing
- `Task.ts`, `tasks.ts` — Task system
- `history.ts`, `ink.ts`, `setup.ts`, and others

### 35 Subsystems

| Subsystem | Module Count | Purpose |
|-----------|-------------|---------|
| **components** | 389 | UI components (React + Ink terminal) |
| **services** | 130 | Backend services (analytics, API, OAuth, MCP, plugins, memory) |
| **hooks** | 104 | Event hooks (notifications, tool permissions, file suggestions) |
| **commands** | ~85 files | Slash commands (/commit, /review, /compact, /mcp, /memory, etc.) |
| **tools** | ~184 modules | Agent tools (BashTool, FileReadTool, AgentTool, etc.) |
| **bridge** | 31 | IDE integration (REPL, JWT, messaging, polling) |
| **assistant** | - | Assistant logic |
| **bootstrap** | - | 12-phase startup sequence |
| **buddy** | - | Digital pet system (unreleased) |
| **cli** | - | CLI entry, transports (HybridTransport, SSE, WebSocket) |
| **constants** | - | Global constants |
| **coordinator** | 1 | Multi-agent coordination mode |
| **context** | - | Context management |
| **entrypoints** | - | cli, init, mcp, sdk (with controlSchemas, coreSchemas, coreTypes) |
| **keybindings** | - | Keyboard shortcuts |
| **memdir** | 8 | Memory directory (findRelevantMemories, memoryScan, memoryAge, etc.) |
| **migrations** | - | Data migrations |
| **moreright** | - | Extended rights/permissions |
| **native-ts** | - | Native TypeScript integration |
| **outputStyles** | - | Output formatting |
| **plugins** | - | Plugin architecture |
| **query** | - | Query processing |
| **remote** | - | Remote operations |
| **schemas** | - | Data schemas |
| **screens** | - | Screen rendering |
| **server** | - | Server mode |
| **skills** | 20 | Skill system (batch, loop, remember, simplify, etc.) |
| **state** | - | State management |
| **tasks** | - | Task management |
| **types** | - | Type definitions |
| **upstreamproxy** | - | Upstream proxy |
| **utils** | - | Utilities |
| **vim** | - | Vim mode |
| **voice** | - | Voice input (Deepgram Nova 3) |

### Technical Stack
- **Runtime:** Bun
- **Language:** TypeScript (strict mode)
- **UI:** React + Ink (terminal renderer with game-engine-style optimization)
- **Authentication:** OAuth 2.0
- **LSP:** Language Server Protocol integration
- **MCP:** Model Context Protocol integration

---

## 3. Core Architecture — The Agent Loop

The fundamental agent loop is deliberately minimal (~20 lines). The 512,000 lines represent the **supporting infrastructure**, not control logic.

### Loop Pattern (from claw-code Rust port: `conversation.rs`)

```
1. User Input → Add as message to session
2. API Call → ApiClient::stream() with current context
3. Message Assembly → Collect text + tool-use events
4. Tool Detection → Extract pending tool invocations
5. Permission Authorization → Check against PermissionPolicy
6. Hook Execution → Run PreToolUse/PostToolUse via HookRunner
7. Tool Execution → Delegate to ToolExecutor::execute()
8. Result Recording → Store tool results as messages
9. Loop → Repeat until no pending tools or max_iterations
```

**Key insight:** "Overengineering control flow (state machines, DAG orchestration) is misplaced effort. Keep iteration patterns austere and invest in the ecosystem surrounding them."

### ConversationRuntime (Rust port)

The `ConversationRuntime<C, T>` is generic over:
- `C: ApiClient` — The API client implementation
- `T: ToolExecutor` — The tool executor implementation

Components:
- **StaticToolExecutor** — Registry-based, stores boxed function handlers in BTreeMap
- **UsageTracker** — Cumulative token metrics across turns
- **Session** — Conversation history with compaction capabilities

---

## 4. System Prompt & Context Engineering

### Prompt Construction (from `prompt.rs`)

**Constants:**
- `MAX_INSTRUCTION_FILE_CHARS`: **4,000** characters per file
- `MAX_TOTAL_INSTRUCTION_CHARS`: **12,000** characters total
- `FRONTIER_MODEL_NAME`: "Opus 4.6"
- `SYSTEM_PROMPT_DYNAMIC_BOUNDARY`: Separates static/dynamic content for cache optimization

**Instruction File Discovery:**
Traverses ancestor directories searching for:
1. `CLAUDE.md` (or `CLAW.md` in the port)
2. `CLAUDE.local.md`
3. `.claude/CLAUDE.md`
4. `.claude/instructions.md`

Deduplication via content hash comparison. Files truncated within character budgets.

### Cache Optimization Strategy

**The architectural constraint around which the entire product is built:**

```
[Static System Prompt] → CACHED (shared across all users)
  ↓
[SYSTEM_PROMPT_DYNAMIC_BOUNDARY]
  ↓
[Dynamic Content: CLAUDE.md, git status, date] → NOT CACHED (session-specific)
```

- Static content first, dynamic content last
- All Claude Code users share the same system prompt cache
- The date goes in the message, not the prompt (would break cache)
- **14 tracked cache-break vectors** monitored for invalidation
- Adding an MCP tool, putting a timestamp in system prompt, switching models — each invalidates the entire cache and **5x costs for that turn**

**Cost impact:**
- Without caching: $50-100 per long Opus session (100 turns)
- With caching: $10-19
- A/B testing showed "~1.2% output token reduction vs qualitative 'be concise'" — leading to explicit constraints: "keep text between tool calls to <=25 words"

### Prompt Sections (build order)
1. Intro / identity
2. System guidelines
3. Project context (working directory, git status)
4. Instruction files (CLAUDE.md hierarchy)
5. Runtime configuration
6. Tool definitions

---

## 5. Tool System

### Scale
- **~40 registered agent tools** in the main registry
- **184 total tool modules** (including sub-modules, UIs, prompts, utilities)
- **29,000 lines** of base tool definitions

### Architecture Principles

Each tool is **self-contained** with:
- Input schema (typed parameters, validation logic)
- Permission level (per-tool, not blanket)
- Execution logic (isolated execution contexts)
- Output formatting
- UI component (React + Ink)
- Prompt template

**Key insight:** "Agents with 5-8 well-described, purpose-built tools consistently outperform agents with a single omnibus tool — even when the omnibus tool is technically more capable."

### Tool Categories

| Category | Tools | Purpose |
|----------|-------|---------|
| **File Operations** | FileReadTool, FileWriteTool, FileEditTool, GlobTool, GrepTool | Read/write/search files |
| **Shell Execution** | BashTool (16 modules), PowerShellTool (14 modules) | Command execution with security |
| **Agent System** | AgentTool (20 modules) | Sub-agent spawning and management |
| **MCP** | MCPTool, ListMcpResourcesTool, ReadMcpResourceTool, McpAuthTool | MCP protocol integration |
| **Task Management** | TaskCreateTool, TaskGetTool, TaskListTool, TaskUpdateTool, TaskStopTool, TaskOutputTool | Background task management |
| **Team** | TeamCreateTool, TeamDeleteTool | Multi-agent team management |
| **Communication** | SendMessageTool, AskUserQuestionTool | User/agent communication |
| **Web** | WebFetchTool, WebSearchTool | Web access |
| **IDE** | LSPTool (6 modules), NotebookEditTool | IDE integration |
| **Planning** | EnterPlanModeTool, ExitPlanModeTool | Plan mode management |
| **Worktree** | EnterWorktreeTool, ExitWorktreeTool | Git worktree isolation |
| **Config** | ConfigTool, SkillTool | Configuration and skills |
| **Scheduling** | CronCreateTool, CronDeleteTool, CronListTool, RemoteTriggerTool | Scheduled execution |
| **Memory** | TodoWriteTool | Task tracking |
| **Utility** | ToolSearchTool, SleepTool, BriefTool, SyntheticOutputTool | Misc utilities |

### AgentTool — Sub-Agent Architecture

Sub-agents are **first-class citizens** of the same tool registry:
- `AgentTool.tsx` — Core agent spawning
- `forkSubagent.ts` — Fork model for sub-agent creation
- `runAgent.ts` — Agent execution runtime
- `resumeAgent.ts` — Agent resumption
- `agentMemory.ts` / `agentMemorySnapshot.ts` — Agent-specific memory
- Built-in agents: `generalPurposeAgent`, `planAgent`, `verificationAgent`, `exploreAgent`, `clawCodeGuideAgent`

### Safety Through Proximity

"Safety rules appear in the immediate context of the action they govern" — embedded directly in tool descriptions rather than separate policy files. This makes them harder for LLMs to overlook.

---

## 6. Permission System

### Permission Modes (from `permissions.rs`)

```
ReadOnly → WorkspaceWrite → DangerFullAccess → Prompt → Allow
```

| Mode | Description |
|------|-------------|
| `ReadOnly` | Planning phase only, minimal access |
| `WorkspaceWrite` | File modification capabilities (auto-apply edits) |
| `DangerFullAccess` | Unrestricted operations (no confirmation) |
| `Prompt` | Interactive user decision-making |
| `Allow` | Unconditional access |

### Authorization Logic

1. Grants access if current mode is `Allow` or equals/exceeds required mode
2. Triggers prompting when escalating from `WorkspaceWrite` to `DangerFullAccess`
3. Returns denial with descriptive reasoning otherwise
4. Tools default to requiring `DangerFullAccess` unless explicitly configured
5. Missing prompters deny escalation requests

### Tiered Model Routing for Permissions

Not all permission decisions require frontier-class inference:
- **Permission checks:** Claude Haiku (cheapest model) pre-screens dangerous commands
- **Frustration detection:** Regex patterns in code, no LLM call
- **Context compression:** MicroCompact uses zero API calls; escalates only when necessary

---

## 7. Hook System

### Hook Events (from `hooks.rs`)

| Event | Timing |
|-------|--------|
| `PreToolUse` | Before tool invocation |
| `PostToolUse` | After tool completion |

### Exit Code Protocol

| Code | Meaning | Effect |
|------|---------|--------|
| **0** | Allow | Operation proceeds; stdout captured as optional message |
| **2** | Deny | Operation blocked; prevents execution |
| **Other** | Warning | Generates warning but permits execution |

### Data Flow

Commands receive **JSON payload via stdin** containing:
- Hook event name
- Tool name and input (parsed + raw JSON)
- Tool output (post-execution only)
- Error status flag

**Environment variables** also set: `HOOK_EVENT`, `HOOK_TOOL_NAME`, `HOOK_TOOL_INPUT`, `HOOK_TOOL_IS_ERROR`, `HOOK_TOOL_OUTPUT`

### Platform Support
- **Windows:** `cmd /C`
- **Unix-like:** `sh -lc`

### Hook Categories (from TypeScript source — 104 modules)

**Notification hooks** (`notifs/`):
- Auto mode unavailability, deprecation, fast mode
- IDE status, MCP connectivity, model migration
- Plugin auto-update, rate limit, settings errors
- Startup, teammate shutdown

**Tool permission hooks:**
- Coordinator handler, interactive handler, swarm worker handler
- Permission logging

**File suggestions hooks:**
- File suggestions, unified suggestions

---

## 8. Memory Architecture

### Three-Layer Self-Healing Memory

Claude Code uses a **lightweight `MEMORY.md` index** (~150 characters per line, max 200 lines / ~25KB) that acts as a pointer system, never storing actual raw data.

**Design principles:**
- Memory treated as **hints**, not ground truth
- Agent explicitly instructed to **verify against actual codebase** before acting
- **Strict Write Discipline** — updates only after successful actions (prevents cementing errors)

### Memory Directory (memdir — 8 modules)

| Module | Purpose |
|--------|---------|
| `findRelevantMemories.ts` | Locate pertinent memories |
| `memdir.ts` | Core entry point |
| `memoryAge.ts` | Temporal tracking |
| `memoryScan.ts` | Scanning across storage |
| `memoryTypes.ts` | Type definitions |
| `paths.ts` | File path management |
| `teamMemPaths.ts` | Team memory paths |
| `teamMemPrompts.ts` | Team memory prompts |

### autoDream — Memory Consolidation

**4-phase consolidation process:**
1. **Orient** — Read MEMORY.md, scan existing memory files
2. **Gather** — Check logs for outdated/contradictory memories
3. **Consolidate** — Merge observations, resolve conflicts
4. **Prune** — Maintain <=200 lines / ~25KB limit

**Trigger conditions (ALL required):**
- >= 24 hours since last consolidation
- >= 5 new sessions minimum
- No active consolidation process
- >= 10 minutes since last scan

**Implementation:** Runs as a forked sub-agent in `services/autoDream/` during idle periods. Uses triple gate (24-hour time gate, 5+ session accumulation, file-based advisory lock) with mtime-based state rollback.

---

## 9. Compaction System

### Three-Layer Context Compression (from `compact.rs` and analysis)

| Layer | Strategy | API Calls | Details |
|-------|----------|-----------|---------|
| **MicroCompact** | Local cache editing, removes old tool outputs | Zero | No API cost |
| **AutoCompact** | Near-ceiling compression | 1 per trigger | 13K token buffer, up to 20K-token summaries, 3-failure circuit breaker |
| **Full Compact** | Complete conversation compression | 1 | Summary + recently accessed files (5K tokens/file cap), active plans, skill schemas; post-compression: ~50K tokens |

### CompactionConfig (from Rust port)
- `preserve_recent_messages`: **4** (default)
- `max_estimated_tokens`: **10,000** (default)

### Key Functions
- `should_compact()` — Checks if compactable messages exceed count AND token thresholds
- `compact_session()` — Removes older messages while preserving recent context
- `summarize_messages()` — Generates structured summaries with: message counts, tool usage, recent requests, pending work, key files, timelines
- `merge_compact_summaries()` — Multi-stage compaction preserving "Previously compacted context" vs "Newly compacted context"

### Token Estimation
Messages estimated at **length / 4 + 1** characters per token.

### Specialized Inference
- `infer_pending_work()` — Identifies TODO/next/pending keywords
- `collect_key_files()` — Extracts code file references (.rs, .ts, .js, .json, .md, .tsx)
- `infer_current_work()` — Captures most recent substantive message

### Critical Vulnerabilities in Compaction
- **MCP tool results bypass microcompaction** — Only tools in `COMPACTABLE_TOOLS` eligible
- **Read operations frozen** — Tools with `maxResultSizeChars: Infinity` exempted via `seenIds`
- **Compaction laundering** — Autocompact prompt preserves "all user messages that are not tool results" — injected instructions can survive as "user directives"
- **Bug impact:** Before 3-failure circuit breaker, continuous autocompaction retry burned **~250,000 API calls per day globally**

---

## 10. MCP Integration

### Tool Naming Convention
```
mcp__{server}__{tool}
```
Both components normalized: non-alphanumeric replaced with underscores, consecutive underscores collapsed for "claude.ai" prefixed names.

Example: tool "weather tool" on "claude.ai Example Server" becomes `mcp__claude_ai_Example_Server__weather_tool`

### Transport Types (6)

| Transport | Description |
|-----------|-------------|
| **Stdio** | Local process execution with command + arguments |
| **SSE** | Server-Sent Events with URL and headers |
| **HTTP** | Remote endpoints with optional OAuth |
| **WebSocket** | Real-time bidirectional communication |
| **SDK** | Built-in implementations (returns no signature) |
| **ManagedProxy** | Anthropic-hosted proxy with URL and ID |

### Architecture Principle
"Claude Code isn't built on top of MCP. It IS MCP — every capability, including Computer Use, runs as a tool call."

Computer Use is implemented as `@ant/computer-use-mcp` — a dedicated MCP server, not special-cased functionality.

### Configuration Hashing
`scoped_mcp_config_hash()` generates stable identifiers using FNV-1a-like hash. Hash **ignores configuration scope** (user vs. local) — identical configs from different sources produce matching hashes.

### CCR Proxy URL Processing
`unwrap_ccr_proxy_url()` extracts underlying MCP URLs from Anthropic's CCR proxy wrappers by parsing `mcp_url` query parameter.

---

## 11. Configuration System

### Three Sources with Precedence (from `config.rs`)

| Source | Priority | Location |
|--------|----------|----------|
| **User** | Lowest | `~/.claude/settings.json` (or legacy `.claw.json`) |
| **Project** | Medium | `.claude/settings.json` (in project root) |
| **Local** | Highest | `.claude/settings.local.json` (environment-specific) |

### Discovery Sequence (5 locations)
1. User legacy `.claw.json`
2. User `settings.json` in config home
3. Project legacy `.claw.json`
4. Project `settings.json` in `.claw/` subdirectory
5. Local `settings.local.json`

### Deep Merging
Objects recursively merged across config sources rather than replaced wholesale, enabling incremental configuration composition.

### RuntimeConfig Contains
- Merged configuration map across all sources
- Feature-specific configs: hooks, plugins, MCP servers, OAuth, sandbox settings

### Sandbox Configuration
- Filesystem isolation modes: `Off`, `WorkspaceOnly`, `AllowList`
- Namespace restrictions
- Network isolation options

---

## 12. Session Management

### Session Format (from `session.rs`)

**MessageRole enum:** `System`, `User`, `Assistant`, `Tool`

**ContentBlock types:**
- **Text** — Simple string content
- **ToolUse** — ID, name, JSON input
- **ToolResult** — Output, error status, reference to originating tool use

**Session struct:** Version number + message history, serializable to JSON.

### Persistence
- `save_to_path()` — JSON to disk
- `load_from_path()` — Reconstruct from file
- `to_json()` / `from_json()` — Native <-> JSON conversion

### Usage Tracking per Message
Optional token usage metrics: input_tokens, output_tokens, cache_creation_input_tokens, cache_read_input_tokens

---

## 13. Cost & Usage Tracking

### Model Pricing (from `usage.rs`)

| Model | Input (per M tokens) | Output (per M tokens) | Cache Creation | Cache Read |
|-------|----------------------|----------------------|----------------|------------|
| **Haiku** | $1.00 | $5.00 | - | - |
| **Sonnet** | $15.00 | $75.00 | $18.75 | $1.50 |
| **Opus** | $15.00 | $75.00 | $18.75 | $1.50 |

### Token Tracking
- Per-turn tracking: input, output, cache creation, cache read
- Cumulative tracking across conversation turns
- Formatted summary lines with cost breakdowns

### Model Name Normalization
`pricing_for_model()` normalizes model names and returns appropriate pricing. Unknown models fall back to default pricing.

---

## 14. Sandbox System

### Filesystem Isolation Modes (from `sandbox.rs`)

| Mode | Description |
|------|-------------|
| `Off` | No isolation |
| `WorkspaceOnly` | Restricts access to workspace (default) |
| `AllowList` | Only specified mount paths permitted |

### Implementation
- Linux: Uses `unshare`-based command with isolated namespaces
- Custom HOME/TMPDIR paths
- Container environment detection (`.dockerenv`, `/run/.containerenv`, env vars, `/proc/1/cgroup`)
- Sandbox status checks: Linux platform, `unshare` availability, config validity

---

## 15. Security Architecture

### Bash Validation System
- **9,707 lines** across 3 files
- **23+ numbered security validators**
- Uses **tree-sitter WASM parser** to build AST of every command before execution

### Bash Security Checks Include
- Zsh builtin blocking
- Zero-width space injection defense
- Unicode injection protection
- IFS attacks
- Token bypasses
- Redirect validation
- ANSI-C quoting gap detection
- Sed edit parsing
- Destructive command warnings
- Read-only mode validation
- Path validation

### Known Vulnerabilities (post-leak)

**Parser Differential (CVE):**
Three different parsers (`splitCommand_DEPRECATED`, `tryParseShellCommand`, `ParsedCommand.parse`) have conflicting edge-case behavior. `shell-quote`'s BAREWORD regex treats CR as token boundary; bash IFS does not include CR. Attackers crafting commands with embedded carriage returns can get validator approval while bash interprets differently.

**Subcommand Limit Bypass:**
Commands with >50 subcommands override security analysis and simply prompt the user. Fix exists in code but not enabled in public builds.

**Early-Allow Short Circuits:**
Validators like `validateGitCommit` return "allow", bypassing all subsequent checks. Documented prior exploit: validateGitCommit → bashCommandIsSafe short-circuits → validateRedirections NEVER runs → ~/.bashrc overwritten.

**Permission Discard Logic:**
`echo "payload" > ~/.bashrc` passes if `Bash(echo:*)` is permitted due to warning discard logic.

### Four-Stage Context Pipeline Vulnerabilities
1. Tool Result Budgeting
2. Microcompact
3. Context Collapse
4. Autocompact

**Critical exemptions enabling persistence attacks** — MCP tool results bypass microcompaction, read operations frozen via `seenIds`.

---

## 16. Multi-Agent & Coordinator Mode

### Three Execution Models
1. **Fork model** — Forked sub-agents
2. **Teammate model** — Collaborative agents
3. **Worktree model** — Git worktree isolation

### Coordinator Mode
- Task distribution across parallel workers
- Permission Queue (Mailbox): Workers request authorization for dangerous operations
- Atomic Claim Mechanism: `createResolveOnce` prevents duplicate handling
- Team Memory: Shared agent workspace

### Cache Sharing Economics
Sub-agents pay only for unique instructions by sharing byte-identical context copies that share the KV cache. Without cache sharing, multi-agent parallelism incurs massive token penalties.

### Orchestration Pattern
1. Parallel research (workers investigate simultaneously)
2. Synthesis (coordinator consolidates, writes specs)
3. Parallel implementation (workers execute per-spec changes)
4. Verification (separate testing workers)

Workers operate in **isolated git worktrees** (avoiding merge conflicts).

---

## 17. Unreleased Systems (Feature-Flagged)

### KAIROS — Autonomous Daemon Mode
Referenced **150+ times** in source. Named after Ancient Greek "at the right time."

Features:
- Autonomous operation via periodic `<tick>` prompts
- **15-second blocking budget** per decision cycle
- **Append-only daily log files** for audit trails
- GitHub webhooks, 5-minute cron cycles
- Background memory consolidation via `/dream`
- Exclusive tools: `SendUserFileTool`, `PushNotificationTool`, `SubscribePRTool`
- Push notifications to user

### ULTRAPLAN — Remote Planning
- Offloads complex planning to **Cloud Container Runtime running Opus 4.6**
- **30-minute planning window**
- Polls every 3 seconds with "teleport sentinel" result retrieval
- Browser UI for live monitoring (approve/reject plan)

### BUDDY — Digital Pet System
- Tamagotchi-style companion with **18 species**
- Gacha mechanics, **1% shiny odds**
- Rarity tiers, procedurally generated stats
- Planned teaser: April 1-7, 2026; full launch: May 2026
- Species names hex-encoded to hide from internal build scanners

### TungstenTool — Internal Terminal Control
- Keystroke and screen-capture for virtual terminals
- Gated by `USER_TYPE === 'ant'` at build time
- Not available in public builds

### Feature Flags (44 total)
Notable flags:
- `PROACTIVE`
- `VOICE_MODE`
- `BRIDGE_MODE`
- `KAIROS`
- `anti_distillation: ['fake_tools']`
- `tengu_penguins_off` (Fast Mode kill-switch)
- `tengu_anti_distill_fake_tool_injection`

### Model Codenames

| Codename | Maps To |
|----------|---------|
| Tengu | Claude Code project codename |
| Capybara | New model family (possibly "Mythos"), currently v8 with 29-30% false claims rate |
| Fennec | Opus 4.6 |
| Numbat | Unreleased model |
| Opus 4.7, Sonnet 4.8 | In development (found in forbidden strings) |

---

## 18. Anti-Distillation & Undercover Mode

### Anti-Distillation (2 layers)

**Layer 1 — Fake Tool Injection:**
When `ANTI_DISTILLATION_CC` flag is enabled, decoy tool definitions injected into system prompt. Requires 4 simultaneous conditions: compile-time flag, CLI entry point, first-party provider, GrowthBook killswitch.

**Layer 2 — Connector-Text Summarization:**
Server returns cryptographically signed summaries rather than full reasoning chains. Only activates for internal Anthropic users (`USER_TYPE === 'ant'`).

### Undercover Mode
- `undercover.ts` — Identity masking for external repositories
- Enabled by default for Anthropic employees in non-allowlisted repos
- Strips `Co-Authored-By` attribution
- Forbids mentioning internal codenames ("Capybara", "Tengu")
- Suppresses "Claude Code" references
- **NO force-OFF mechanism** documented in code
- Allowlist of 22 private repos leaked (includes `anthropics/casino`, `anthropics/forge-web`, `anthropics/claude-for-hiring`)
- Dead-code-eliminated from external builds

---

## 19. Bootstrap & Startup Sequence

### 12-Phase Startup (from `bootstrap.rs`)

1. `CliEntry` — CLI argument parsing
2. `FastPathVersion` — Version check
3. `StartupProfiler` — Performance profiling
4. `SystemPromptFastPath` — System prompt pre-computation
5. `ChromeMcpFastPath` — Chrome MCP fast path
6. `DaemonWorkerFastPath` — Daemon worker initialization
7. `BridgeFastPath` — IDE bridge initialization
8. `DaemonFastPath` — Daemon mode fast path
9. `BackgroundSessionFastPath` — Background session
10. `TemplateFastPath` — Template processing
11. `EnvironmentRunnerFastPath` — Environment runner
12. `MainRuntime` — Full runtime initialization

Phases are deduplicated (no phase can appear twice in plan).

---

## 20. Services Layer

### 130 Modules Across Key Areas

**Agent Services:** AgentSummary

**Documentation:** MagicDocs (documentation generation), PromptSuggestion (speculation)

**Session Memory:** SessionMemory, sessionMemoryUtils, prompts

**Analytics Infrastructure:**
- Datadog integration
- GrowthBook (feature flags / A/B testing)
- First-party event logging + exporter
- Data sink with killswitch

**API Services:**
- adminRequests, bootstrap, client
- Error handling (errorUtils, errors)
- Prompt dumping (dumpPrompts)
- Usage tracking (emptyUsage)

**Other services:** OAuth, MCP services, plugin operations, settings sync, team memory, policy limits

---

## 21. Skills System

### 20 Modules

**Bundled Skills:**
- `batch.ts` — Batch processing
- `clawApi.ts` / `clawApiContent.ts` — API integration
- `clawInChrome.ts` — Chrome integration
- `debug.ts` — Debugging
- `keybindings.ts` — Keyboard shortcuts
- `loop.ts` — Loop execution (recurring tasks)
- `loremIpsum.ts` — Sample text
- `remember.ts` — State persistence
- `scheduleRemoteAgents.ts` — Remote agent scheduling
- `simplify.ts` — Code simplification
- `skillify.ts` — Skill conversion
- `stuck.ts` — Problem-solving assistance
- `updateConfig.ts` — Config updates
- `verify.ts` / `verifyContent.ts` — Verification

**Infrastructure:**
- `bundledSkills.ts` — Aggregation
- `loadSkillsDir.ts` — Dynamic directory loading
- `mcpSkillBuilders.ts` — MCP skill builders

---

## 22. Bridge System (IDE Integration)

### 31 Modules

**Core:** bridgeApi, bridgeConfig, bridgeMain, bridgeMessaging, bridgeUI
**Security:** bridgePermissionCallbacks, jwtUtils
**Session:** codeSessionApi, createSession
**Communication:** inboundAttachments, inboundMessages, replBridge, remoteBridgeCore
**Configuration:** pollConfig, pollConfigDefaults, envLessBridgeConfig
**Debug:** bridgeDebug, debugUtils
**State:** bridgeEnabled, bridgePointer, bridgeStatusUtil
**Performance:** capacityWake, flushGate

---

## 23. Complete Subsystem Inventory

All 35 subsystems from the TypeScript codebase:

1. assistant, 2. bootstrap, 3. bridge (31 modules), 4. buddy, 5. cli,
6. commands, 7. components (389 modules), 8. constants, 9. coordinator (1 module),
10. context, 11. entrypoints, 12. hooks (104 modules), 13. keybindings,
14. memdir (8 modules), 15. migrations, 16. moreright, 17. native-ts,
18. outputStyles, 19. plugins, 20. query, 21. remote, 22. schemas,
23. screens, 24. server, 25. services (130 modules), 26. skills (20 modules),
27. state, 28. tasks, 29. tools, 30. types, 31. upstreamproxy,
32. utils, 33. vim, 34. voice, 35. (root-level files)

---

## 24. Complete Command Inventory

### Key Commands (99 identified from snapshot)

**Core workflow:** help, status, model, permissions, clear, compact, cost, diff, effort, exit, export, fast, feedback, files, memory, plan, review, resume, rewind, session, skills, stats, tasks, usage, version

**Git/GitHub:** branch, commit, commit-push-pr, pr_comments

**Configuration:** config, hooks, keybindings, output-style, permissions, privacy-settings, rate-limit-options, sandbox-toggle, theme, vim, color

**IDE/Integration:** bridge, bridge-kick, chrome, desktop, ide, mobile, remote-env, remote-setup, terminal-setup

**Account:** login, logout, extra-usage, install, install-github-app, install-slack-app, upgrade

**Advanced:** agents, autofix-pr, bughunter, doctor, heapdump, insights, issue, plugin, security-review, share, stickers, summary, tag, teleport, thinkback, thinkback-play, ultraplan, voice

**Internal/Debug:** ant-trace, break-cache, ctx_viz, debug-tool-call, env, good-claw, init-verifiers, mock-limits, oauth-refresh, onboarding, perf-issue, release-notes, reset-limits, statusline

---

## 25. Complete Tool Inventory

### 184 Tool Modules

**Agent Tools (20):** AgentTool, UI, agentColorManager, agentDisplay, agentMemory, agentMemorySnapshot, agentToolUtils, built-in agents (codeGuide, explore, generalPurpose, plan, statuslineSetup, verification), builtInAgents, constants, forkSubagent, loadAgentsDir, prompt, resumeAgent, runAgent

**Bash Tools (16):** BashTool, BashToolResultMessage, UI, bashCommandHelpers, bashPermissions, bashSecurity, commandSemantics, commentLabel, destructiveCommandWarning, modeValidation, pathValidation, prompt, readOnlyValidation, sedEditParser, sedValidation, shouldUseSandbox

**PowerShell Tools (14):** PowerShellTool, UI, clmTypes, commandSemantics, commonParameters, destructiveCommandWarning, gitSafety, modeValidation, pathValidation, powershellPermissions, powershellSecurity, prompt, readOnlyValidation, toolName

**File Tools (14):** FileReadTool (5), FileWriteTool (3), FileEditTool (6)

**MCP Tools (10):** MCPTool (4), ListMcpResourcesTool (3), ReadMcpResourceTool (3)

**Task Tools (17):** TaskCreateTool (3), TaskGetTool (3), TaskListTool (3), TaskUpdateTool (3), TaskStopTool (3), TaskOutputTool (2)

**Team Tools (8):** TeamCreateTool (4), TeamDeleteTool (4)

**Planning Tools (8):** EnterPlanModeTool (4), ExitPlanModeTool (4)

**Worktree Tools (8):** EnterWorktreeTool (4), ExitWorktreeTool (4)

**Search Tools (6):** GlobTool (3), GrepTool (3)

**Web Tools (8):** WebFetchTool (5), WebSearchTool (3)

**LSP Tools (6):** LSPTool, UI, formatters, prompt, schemas, symbolContext

**Other Tools:** BriefTool (5), ConfigTool (5), SkillTool (4), SendMessageTool (4), NotebookEditTool (4), ScheduleCronTool (5), RemoteTriggerTool (3), TodoWriteTool (3), ToolSearchTool (3), AskUserQuestionTool (2), REPLTool (2), SyntheticOutputTool (1), SleepTool (1), McpAuthTool (1)

**Shared:** gitOperationTracking, spawnMultiAgent, TestingPermissionTool, utils

---

## 26. Competing Frameworks Analysis

### BMAD Method (Breakthrough Method for Agile AI-Driven Development)

**Repository:** bmad-code-org/BMAD-METHOD
**Version:** v6 (Stable)
**License:** MIT

**Core concept:** Spec-Driven Development (SDD) with human-in-the-loop governance. Structured YAML-based workflows orchestrating specialized AI agents.

**Agent roles:** Product Manager, Architect, Developer, UX Designer, Scrum Master, QA, and more (12+ total)

**Key innovations in v6:**
- **Document sharding** — Large docs split into focused pieces (~300 tokens vs ~5,000), reducing token consumption by 74-90%
- **Party Mode** — Multiple agent personas collaborate within single sessions
- **Claude Code native implementation** — Uses Skills, Commands, Hooks, Memory, Files
- **Scale-adaptive intelligence** — Adjusts complexity from bug fixes to enterprise systems
- 34+ documented workflows

**Ecosystem modules:** BMad Builder (custom agents), Test Architect (TEA), Game Dev Studio, Creative Intelligence Suite

**Comparison with SINAPSE:**
- Similar agent-based workflow approach
- BMAD focuses heavily on token optimization through sharding
- Less emphasis on git safety/collaboration than SINAPSE
- More modular/pluggable architecture

---

### AIOX-Core (Synkra AIOS)

**Repository:** SynkraAI/aiox-core
**Version:** 4.2.11
**Stars:** 2.6k
**License:** MIT

**Core concept:** AI-Orchestrated System for Full Stack Development with CLI-first architecture.

**Architecture hierarchy:** CLI First -> Observability Second -> UI Third

**Two-phase development model:**
1. **Agentic Planning Phase** — Analyst, PM, Architect collaborate for PRD/architecture
2. **Contextual Development Phase** — Scrum Master creates hyper-detailed stories with complete context

**Agent system:** @analyst, @pm, @architect, @dev, @qa, @sm

**IDE Hook Parity:**

| IDE/CLI | Hook Parity |
|---------|-------------|
| Claude Code | Complete |
| Gemini CLI | High |
| Codex CLI | Partial |
| Cursor | None (Rules + MCP) |
| GitHub Copilot | None (Instruction-based) |

**Multi-domain squads:** Software dev, creative writing, business strategy, health/wellness, education

**Comparison with SINAPSE:**
- VERY similar architecture to SINAPSE (both are AI-orchestrated systems)
- Similar agent roles and naming patterns
- SINAPSE has more sophisticated framework protection (L1-L4 layers)
- SINAPSE has stronger constitutional governance model
- AIOX has broader IDE support beyond Claude Code

---

### Ruflo — Multi-Agent Swarm Platform

**Repository:** ruvnet/ruflo
**Stars:** 1,173

**Core concept:** Agent orchestration for Claude Code — 60+ agent swarm with native MCP integration.

**Key claims:**
- 75% reduction in Claude API costs
- Agents for planning, coding, testing, security — all running in parallel while sharing memory
- Native integration via MCP (commands usable directly in Claude Code sessions)

---

### Claude Flow / Shipyard — Multi-Agent Orchestration

Other frameworks emerging in the multi-agent space for Claude Code, focusing on:
- Parallel task execution
- Shared memory patterns
- Cost optimization through cache sharing
- Git worktree isolation

---

## 27. Architectural Lessons for SINAPSE

### 7 Key Lessons from Claude Code's Production Architecture

**1. Minimal Core Loop, Maximum Infrastructure**
The agent loop is ~20 lines. The 512K lines are supporting infrastructure. SINAPSE should keep its orchestration loop simple and invest in tools, permissions, and context management.

**2. Safety Through Proximity**
Embed constraints directly in tool descriptions, not separate policy files. SINAPSE's hook system aligns well with this — but consider embedding more safety rules directly into agent prompts.

**3. Structured Tools Over Generic Commands**
Every frequently-executed shell command should become a dedicated, typed, gated tool. SINAPSE could benefit from formalizing more tool definitions beyond what Claude Code provides natively.

**4. Context Engineering as Competitive Advantage**
- Separate static from dynamic prompt content for cache optimization
- Track cache-break vectors
- Build tiered compression strategies (MicroCompact -> AutoCompact -> Full Compact)
- SINAPSE should monitor which context elements correlate with agent success

**5. Memory as Indexed Hints**
- Lightweight index with verification-on-retrieval
- Consolidation and cleanup lifecycle
- Skeptical memory prevents confident recommendations of outdated info
- SINAPSE's MEMORY.md pattern aligns well

**6. Cache-Sharing for Multi-Agent**
- Byte-identical context copies share KV cache
- Sub-agents pay only for unique instructions
- SINAPSE should design agent prompts to maximize cache sharing across agents

**7. Tiered Model Routing**
- Not all decisions need frontier models
- Use Haiku for permission checks, regex for frustration detection
- Zero-API-call compression as first tier
- SINAPSE could route simpler decisions to cheaper models

### Additional SINAPSE Improvements

**From BMAD:** Document sharding for large specs/PRDs (~300 tokens per shard vs ~5,000 full doc). SINAPSE's story system could benefit from this approach.

**From AIOX:** Multi-IDE support beyond Claude Code. As the ecosystem grows, SINAPSE should consider Gemini CLI and Codex CLI compatibility.

**From claw-code analysis:** The hook system is production-validated. SINAPSE's hook governance rules are well-aligned with Claude Code's design. The PreToolUse/PostToolUse pattern with exit codes 0/2 is exactly what SINAPSE implements.

---

## 28. Sources

### Primary Sources — Clean-Room Rewrites
- [ultraworkers/claw-code](https://github.com/ultraworkers/claw-code) — 163K stars, Rust port with reference data snapshots
- [ultraworkers/claw-code-parity](https://github.com/ultraworkers/claw-code-parity) — 5.4K stars, parity analysis

### Leak Analysis Articles
- [VentureBeat — Claude Code's source code appears to have leaked](https://venturebeat.com/technology/claude-codes-source-code-appears-to-have-leaked-heres-what-we-know)
- [The Hacker News — Claude Code Source Leaked via npm Packaging Error](https://thehackernews.com/2026/04/claude-code-tleaked-via-npm-packaging.html)
- [Sabrina.dev — Comprehensive Analysis of Claude Code Source Leak](https://www.sabrina.dev/p/claude-code-source-leak-analysis)
- [Alex Kim — The Claude Code Source Leak: fake tools, frustration regexes, undercover mode](https://alex000kim.com/posts/2026-03-31-claude-code-source-leak/)
- [Low Code Agency — Claude Code Source Code Leaked? Here's what it contains](https://www.lowcode.agency/blog/claude-code-source-code-leaked)
- [ClaudeFast — Claude Code Source Leak: Everything Found](https://claudefa.st/blog/guide/mechanics/claude-code-source-leak)
- [DEV Community — Claude Code's Entire Source Code Just Leaked](https://dev.to/evan-dong/claude-codes-entire-source-code-just-leaked-512000-lines-exposed-3139)
- [Particula Tech — Claude Code Source Leak: 7 Agent Architecture Lessons](https://particula.tech/blog/claude-code-source-leak-agent-architecture-lessons)
- [Straiker — With Great Agency Comes Great Responsibility](https://www.straiker.ai/blog/claude-code-source-leak-with-great-agency-comes-great-responsibility)
- [The Register — Claude Code's source reveals extent of system access](https://www.theregister.com/2026/04/01/claude_code_source_leak_privacy_nightmare/)
- [SecurityWeek — Critical Vulnerability in Claude Code](https://www.securityweek.com/critical-vulnerability-in-claude-code-emerges-days-after-source-leak/)
- [DEV Community — The Great Claude Code Leak of 2026](https://dev.to/varshithvhegde/the-great-claude-code-leak-of-2026-accident-incompetence-or-the-best-pr-stunt-in-ai-history-3igm)
- [ModemGuides — Claude Code Leak Architecture Analysis](https://www.modemguides.com/blogs/ai-news/claude-code-leak-architecture-analysis)
- [DEV Community — What Claude Code's Leaked Architecture Reveals About Building Production MCP Servers](https://dev.to/shekharp1536/what-claude-codes-leaked-architecture-reveals-about-building-production-mcp-servers-2026-10on)

### Competing Frameworks
- [BMAD-METHOD](https://github.com/bmad-code-org/BMAD-METHOD) — Breakthrough Method for Agile AI-Driven Development
- [BMAD Method Docs](https://docs.bmad-method.org/)
- [SynkraAI/aiox-core](https://github.com/SynkraAI/aiox-core) — AI-Orchestrated System for Full Stack Development
- [ruvnet/ruflo](https://github.com/ruvnet/ruflo) — Multi-agent swarm orchestration for Claude Code

### Security Analysis
- [Straiker — Context Pipeline Vulnerabilities](https://www.straiker.ai/blog/claude-code-source-leak-with-great-agency-comes-great-responsibility)
- [SecurityWeek — Critical Vulnerability](https://www.securityweek.com/critical-vulnerability-in-claude-code-emerges-days-after-source-leak/)
- [The Register — System Access Concerns](https://www.theregister.com/2026/04/01/claude_code_source_leak_privacy_nightmare/)

---

*Research compiled 2026-04-04 for SINAPSE AI framework improvement.*
