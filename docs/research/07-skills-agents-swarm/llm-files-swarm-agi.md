# Mapa Definitivo de Arquivos LLM CLI/IDE, Swarm Intelligence e AGI

> **Data:** 2026-04-04
> **Autor:** Prism (research-orqx) via SINAPSE Research Initiative
> **Fontes:** 80+ fontes consultadas
> **Objetivo:** Referencia definitiva sobre (1) todos os arquivos gerados/lidos por cada ferramenta AI de codigo, (2) swarm intelligence aplicada a AI agents, e (3) o estado atual da corrida pela AGI.

---

## Indice

- [PARTE 1: Mapa Completo de Arquivos de Cada LLM CLI/IDE](#parte-1-mapa-completo-de-arquivos-de-cada-llm-cliide)
  - [1.1 Claude Code](#11-claude-code)
  - [1.2 OpenAI Codex CLI](#12-openai-codex-cli)
  - [1.3 Google Gemini CLI](#13-google-gemini-cli)
  - [1.4 Cursor](#14-cursor)
  - [1.5 GitHub Copilot](#15-github-copilot)
  - [1.6 Windsurf (Codeium)](#16-windsurf-codeium)
  - [1.7 Amazon Q Developer CLI](#17-amazon-q-developer-cli)
  - [1.8 Tabela Comparativa Master](#18-tabela-comparativa-master)
- [PARTE 2: Swarm Intelligence em AI](#parte-2-swarm-intelligence-em-ai)
  - [2.1 Fundacoes da Swarm Intelligence](#21-fundacoes-da-swarm-intelligence)
  - [2.2 AI Agent Swarms (Moderno)](#22-ai-agent-swarms-moderno)
  - [2.3 Padroes de Swarm para Desenvolvimento](#23-padroes-de-swarm-para-desenvolvimento)
  - [2.4 Implementacao Pratica](#24-implementacao-pratica)
  - [2.5 Pessoas-Chave e Livros](#25-pessoas-chave-e-livros)
- [PARTE 3: AGI -- Artificial General Intelligence](#parte-3-agi----artificial-general-intelligence)
  - [3.1 O que e AGI](#31-o-que-e-agi)
  - [3.2 Contexto Historico](#32-contexto-historico)
  - [3.3 Desafios Tecnicos](#33-desafios-tecnicos)
  - [3.4 Previsoes de Timeline para AGI](#34-previsoes-de-timeline-para-agi)
  - [3.5 O que AGI Significa para Frameworks de Desenvolvimento](#35-o-que-agi-significa-para-frameworks-de-desenvolvimento)
  - [3.6 Livros Fundamentais](#36-livros-fundamentais)
- [Referencias Historicas e Mundiais](#referencias-historicas-e-mundiais)
- [Fontes e Links](#fontes-e-links)
- [Checklist de Completude](#checklist-de-completude)

---

# PARTE 1: Mapa Completo de Arquivos de Cada LLM CLI/IDE

## 1.1 Claude Code

Claude Code (Anthropic) e a ferramenta com o ecossistema de arquivos mais rico e complexo entre todas as AI coding tools. Possui quatro escopos de configuracao: Managed (IT deploy), User (~/.claude/), Project (.claude/), e Local (.claude/settings.local.json).

### Diretorio Global: `~/.claude/`

```
~/.claude/
├── CLAUDE.md                    # Instrucoes pessoais globais (carregado em TODA sessao)
├── settings.json                # Configuracoes globais (permissoes, hooks, modelo)
├── history.jsonl                # Historico de prompts de todas as sessoes
├── stats-cache.json             # Estatisticas de uso agregadas
│
├── projects/                    # Transcricoes de sessao por projeto
│   └── {encoded-path}/         # Path do projeto com / trocado por -
│       ├── MEMORY.md            # Auto-memoria do projeto (max 200 linhas ou 25KB)
│       ├── {sessionId}.jsonl    # Transcricao de sessao principal
│       └── agent-{shortId}.jsonl # Transcricao de subagente
│
├── plans/                       # Documentos de plan mode
│   └── {whimsical-name}.md     # Planos com nomes whimsical auto-gerados
│
├── file-history/                # Checkpoints para undo/rollback
│   └── {sessionId}/
│       └── {fileHash}@v{version}  # Conteudo pre-edit por content hash
│
├── todos/                       # Task lists por sessao
│   └── {sessionId}-agent-{agentId}.json
│
├── session-env/                 # Variaveis de ambiente por sessao
│   └── {sessionId}/
│
├── shell-snapshots/             # Snapshots do ambiente shell
│   └── snapshot-{shell}-{timestamp}-{random}.sh
│
├── debug/                       # Logs de debug por sessao
│   └── {sessionId}.txt
│
├── commands/                    # Slash commands pessoais (/user:command-name)
│   └── {command-name}.md
│
├── skills/                      # Skills complexas com scripts
│   └── {skill-name}/
│       ├── SKILL.md             # YAML frontmatter + instrucoes
│       └── scripts/             # Scripts auxiliares
│
├── agents/                      # Definicoes de agentes pessoais
│   └── {agent-name}.md
│
├── plugins/                     # Marketplace e instalacoes de plugins
│   ├── cache/
│   ├── config.json
│   ├── installed_plugins.json
│   ├── known_marketplaces.json
│   ├── install-counts-cache.json
│   └── marketplaces/
│
├── ide/                         # Locks de integracao IDE
├── statsig/                     # Cache de feature flags
└── telemetry/                   # Telemetria de uso (se habilitado)
```

### Arquivo Global Raiz: `~/.claude.json`

Localizado FORA de ~/.claude/, na raiz do home:

```json
{
  "numStartups": 42,
  "installMethod": "npm",
  "theme": "dark",
  "autoUpdates": true,
  "preferredNotifChannel": "terminal",
  "hasCompletedOnboarding": true,
  "userID": "uuid",
  "oauthAccount": {
    "accountUuid": "...",
    "emailAddress": "...",
    "organizationUuid": "..."
  },
  "mcpServers": {
    "{serverName}": {
      "command": "string",
      "args": ["array"]
    }
  },
  "projects": {
    "{projectPath}": {
      "allowedTools": [],
      "hasTrustDialogAccepted": true,
      "mcpServers": {},
      "lastSessionId": "uuid",
      "lastCost": 0.42,
      "lastAPIDuration": 1234,
      "lastDuration": 5678,
      "exampleFiles": [],
      "hasCompletedProjectOnboarding": true,
      "dontCrawlDirectory": false
    }
  },
  "tipsHistory": {},
  "cachedStatsigGates": {},
  "cachedGrowthBookFeatures": {}
}
```

### Diretorio de Projeto: `.claude/`

```
.claude/
├── CLAUDE.md                    # Instrucoes do projeto (commitado no git)
├── settings.json                # Permissoes do projeto (allow/deny patterns, hooks)
├── settings.local.json          # Permissoes pessoais (gitignored)
│
├── rules/                       # Regras modulares por concern
│   └── {rule-name}.md          # YAML frontmatter com path-scoped activation
│                                # Ex: globs: ["src/**/*.ts"]
│
├── hooks/                       # Scripts de hooks (PreToolUse, PostToolUse, etc.)
│   └── {hook-script}.cjs       # ou .sh, .py
│
├── commands/                    # Slash commands do projeto (/project:command-name)
│   └── {command-name}.md       # Markdown com $ARGUMENTS
│
├── skills/                      # Skills do projeto
│   └── {skill-name}/
│       ├── SKILL.md             # YAML frontmatter: allowed_tools, model
│       └── scripts/
│
└── agents/                      # Subagentes do projeto
    └── {agent-name}.md          # System prompt + tool restrictions + modelo
```

### Locais do CLAUDE.md

CLAUDE.md pode existir em QUATRO niveis de hierarquia:

| Nivel | Localizacao | Escopo | Compartilhado? |
|-------|-------------|--------|----------------|
| Global | `~/.claude/CLAUDE.md` | Todas as sessoes, todos os projetos | Nao |
| Projeto (raiz) | `CLAUDE.md` na raiz do repo | Todo o projeto | Sim (git) |
| Projeto (.claude) | `.claude/CLAUDE.md` | Todo o projeto | Sim (git) |
| Subdiretorio | `{subdir}/CLAUDE.md` | Quando trabalhando nesse diretorio | Sim (git) |

Adicionalmente: `CLAUDE.local.md` (gitignored) para preferencias pessoais no nivel de projeto.

### settings.json Schema (campos principais)

O schema JSON e publicado em `https://json.schemastore.org/claude-code-settings.json`. Campos-chave:

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "permissions": {
    "allow": ["Bash(npm run *)", "Read(**)", "Write(.claude/*)"],
    "deny": ["Bash(rm -rf *)", "Read(.env)"]
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash|Write|Edit",
        "command": "node .claude/hooks/my-hook.cjs"
      }
    ],
    "PostToolUse": [],
    "UserPromptSubmit": [],
    "PreCompact": []
  },
  "env": {
    "MY_VAR": "value"
  }
}
```

### MCP Config

Arquivo principal: `~/.claude.json` (campo `mcpServers`).
Tambem pode ser definido em `.mcp.json` no projeto ou no settings.json do projeto.

### Formato de Sessao (JSONL)

Cada linha em `{sessionId}.jsonl` contem:

```json
{
  "type": "user|assistant|file-history-snapshot|queue-operation",
  "uuid": "...",
  "parentUuid": "...",
  "sessionId": "...",
  "message": { "role": "...", "content": "..." },
  "timestamp": "ISO-8601",
  "toolUseMessages": [],
  "thinkingMetadata": {}
}
```

### MEMORY.md

- Localizado em `~/.claude/projects/{encoded-path}/MEMORY.md`
- Auto-gerado pelo Claude durante sessoes
- Contem: comandos descobertos, padroes observados, insights de arquitetura
- Limite de carga: primeiras 200 linhas OU 25KB (o que vier primeiro)
- Acessivel via comando `/memory`

---

## 1.2 OpenAI Codex CLI

Codex CLI (OpenAI) usa TOML para configuracao e Markdown para instrucoes, com um sistema de profiles e features flags.

### Diretorio Global: `~/.codex/`

```
~/.codex/
├── config.toml              # Configuracao global do usuario
├── AGENTS.md                # Instrucoes globais (ou AGENTS.override.md)
├── log/                     # Logs de sessao (customizavel via log_dir)
│   └── {session}.log
└── (SQLite state DB)        # Banco de estado (customizavel via sqlite_home)
```

### Diretorio de Projeto: `.codex/`

```
.codex/
├── config.toml              # Override de configuracao por projeto (requer trust)
├── AGENTS.md                # Instrucoes do projeto
├── AGENTS.override.md       # Override temporario das instrucoes
└── hooks.json               # Lifecycle hooks (experimental, off por default)
```

### AGENTS.md -- Especificacao Completa

**Ordem de descoberta** (cada diretorio do git root ate o cwd):
1. `AGENTS.override.md` (prioridade maxima)
2. `AGENTS.md`
3. `TEAM_GUIDE.md` (fallback configuravel)
4. `.agents.md` (fallback configuravel)

**Concatenacao:** Arquivos sao concatenados do root para baixo, com linhas em branco entre eles. Arquivos mais proximos do cwd tem prioridade porque aparecem depois no prompt combinado.

**Limite de tamanho:** `project_doc_max_bytes = 32768` (32 KiB). Codex para de adicionar arquivos quando o limite combinado e atingido.

**Diretorio custom:** `CODEX_HOME` env var para usar profile diferente.

### config.toml -- Schema Completo

```toml
# Modelo e Provider
model = "gpt-5-codex"
model_provider = "openai"
model_context_window = 200000
model_reasoning_effort = "medium"  # minimal | low | medium | high | xhigh

# Sandbox e Seguranca
sandbox_mode = "read-only"  # read-only | workspace-write | danger-full-access
approval_policy = "default"  # granular: sandbox_escalation, rules, skill_approval

# Features
[features]
unified_exec = true
multi_agent = false
fast_mode = false
personality = true
web_search = "cached"  # disabled | cached | live
undo = true
shell_tool = true
skill_mcp_dependency_install = true
codex_hooks = false  # experimental

# Profiles
[profiles.fast]
model = "gpt-4o-mini"
model_reasoning_effort = "low"

[profiles.deep]
model = "o4-mini"
model_reasoning_effort = "xhigh"

# Shell Environment
[shell_environment_policy]
inherit = true
include_only = []
exclude = ["SECRET_KEY"]
set = { "CI" = "true" }

# Agents
[agents]
max_threads = 6
max_depth = 3

[agents.reviewer]
description = "Code review specialist"
config_file = ".codex/reviewer.toml"

# History
[history]
persistence = "save-all"  # save-all | none
max_bytes = 10485760

# MCP Servers
[mcp_servers.github]
command = "github-mcp-server"
args = ["--mode", "stdio"]
env = { "GITHUB_TOKEN" = "$GITHUB_TOKEN" }
tool_enable = ["get_issues", "create_pr"]
startup_timeout = 30000
execution_timeout = 60000

# TUI
[tui]
notifications = true
theme = "monokai"
animations = true
status_line = ["model", "cost", "tokens"]

# Observability
[otel]
enabled = false
exporter = "otlp"
```

### hooks.json Format

```json
{
  "hooks": [
    {
      "event": "pre_tool_use",
      "matcher": "bash",
      "command": "node .codex/hooks/validate.js"
    },
    {
      "event": "post_tool_use",
      "command": "echo 'done'"
    }
  ]
}
```

Eventos suportados: `pre_tool_use`, `post_tool_use` (mais eventos em desenvolvimento).

---

## 1.3 Google Gemini CLI

Gemini CLI (Google) usa JSON para configuracao e Markdown para contexto, com hierarquia de 4 niveis de settings.

### Diretorio Global: `~/.gemini/`

```
~/.gemini/
├── settings.json               # Configuracoes do usuario
├── GEMINI.md                   # Contexto global (carregado em toda sessao)
├── agents/                     # Agentes customizados do usuario
│   └── {agent-name}.md        # Markdown com YAML frontmatter
└── tmp/
    └── {project_hash}/
        └── shell_history       # Historico de shell por projeto
```

### Diretorio de Projeto: `.gemini/`

```
.gemini/
├── settings.json               # Override de configuracao por projeto
├── GEMINI.md                   # Contexto do projeto (ou AGENT.md como alias)
├── agents/                     # Agentes do projeto
│   └── {agent-name}.md        # Markdown com YAML frontmatter
├── sandbox.Dockerfile          # Dockerfile customizado para sandbox
├── sandbox-macos-{profile}.sb  # macOS sandbox profile customizado
└── .env                        # Variaveis de ambiente do projeto
```

### Hierarquia de Settings (4 niveis, menor para maior prioridade)

| Prioridade | Nivel | Localizacao |
|-----------|-------|-------------|
| 1 (menor) | System Defaults | `/etc/gemini-cli/system-defaults.json` (Linux), `C:\ProgramData\gemini-cli\system-defaults.json` (Win), `/Library/Application Support/GeminiCli/system-defaults.json` (Mac) |
| 2 | User Settings | `~/.gemini/settings.json` |
| 3 | Project Settings | `.gemini/settings.json` |
| 4 (maior) | System Overrides | `/etc/gemini-cli/settings.json` (Linux/Mac), `C:\ProgramData\gemini-cli\settings.json` (Win) |

Override de paths via env vars: `GEMINI_CLI_SYSTEM_DEFAULTS_PATH`, `GEMINI_CLI_SYSTEM_SETTINGS_PATH`.

### GEMINI.md -- Especificacao

**Nome configuravel:** Via `context.fileName` no settings.json (pode ser string ou array).

**Hierarquia de carregamento:**
1. `~/.gemini/GEMINI.md` (global)
2. Ancestrais do projeto (ate `.git` ou home)
3. Projeto root
4. Subdiretorios abaixo do cwd (max `context.discoveryMaxDirs` = 200 dirs)

**Import syntax:** `@path/to/file.md` dentro do GEMINI.md para importar outros arquivos.

**Alias:** `AGENT.md` tambem e aceito na raiz do projeto.

### settings.json -- Schema Completo (categorizado)

```json
{
  "general": {
    "preferredEditor": "vscode",
    "vimMode": false,
    "disableAutoUpdate": false,
    "checkpointing": { "enabled": false }
  },
  "ui": {
    "theme": "dark",
    "customThemes": {},
    "hideWindowTitle": false,
    "hideTips": false,
    "hideBanner": false,
    "hideFooter": false,
    "showMemoryUsage": false,
    "showLineNumbers": false,
    "showCitations": true
  },
  "model": {
    "name": "gemini-2.5-pro",
    "maxSessionTurns": -1,
    "chatCompression": {
      "contextPercentageThreshold": 0.7
    }
  },
  "context": {
    "fileName": "GEMINI.md",
    "discoveryMaxDirs": 200,
    "includeDirectories": [],
    "fileFiltering": {
      "respectGitIgnore": true,
      "respectGeminiIgnore": true,
      "enableRecursiveFileSearch": true
    }
  },
  "tools": {
    "sandbox": false,
    "shell": { "enableInteractiveShell": false },
    "core": null,
    "exclude": [],
    "allowed": []
  },
  "mcpServers": {
    "{server-name}": {
      "command": "npx",
      "args": ["-y", "@server/package"],
      "env": {},
      "url": null,
      "httpUrl": null,
      "headers": {},
      "timeout": 30000,
      "trust": false,
      "includeTools": [],
      "excludeTools": []
    }
  },
  "security": {
    "folderTrust": { "enabled": false },
    "auth": { "selectedType": null }
  },
  "privacy": {
    "usageStatisticsEnabled": true
  },
  "telemetry": {
    "enabled": false,
    "target": "local",
    "otlpEndpoint": null,
    "logPrompts": false
  }
}
```

### Formato de Agent YAML (subagents)

```markdown
---
name: code-reviewer
description: Specialized code review agent
model: gemini-2.5-pro
tools:
  - read_file
  - search
  - list_files
---

You are a code review specialist. Focus on...
```

### Variaveis de Ambiente

| Variavel | Proposito |
|----------|-----------|
| `GEMINI_API_KEY` | Autenticacao Gemini API |
| `GOOGLE_API_KEY` | Autenticacao Google Cloud |
| `GEMINI_MODEL` | Modelo default |
| `GEMINI_SANDBOX` | Modo sandbox ("true", "docker", "podman") |
| `GOOGLE_CLOUD_PROJECT` | Project ID do GCP |

---

## 1.4 Cursor

Cursor usa um sistema de regras em evolucao: migrou de `.cursorrules` (single file, deprecated) para `.cursor/rules/` (diretorio modular). Em versao 2.2+, rules sao pastas com `RULE.md`.

### Diretorio de Projeto: `.cursor/`

```
.cursor/
├── rules/                      # Regras modulares do projeto (v2.2+)
│   └── {rule-name}/
│       └── RULE.md             # Frontmatter + conteudo markdown
│
├── mcp.json                    # Configuracao MCP do projeto
│
└── (legacy) *.mdc              # Arquivos .mdc (pre-v2.2, ainda suportados)
```

### Diretorio Global: `~/.cursor/`

```
~/.cursor/
├── mcp.json                    # MCP servers globais
└── (user settings via UI)      # User Rules definidas no Settings do Cursor
```

### Arquivo Legacy: `.cursorrules`

Arquivo unico na raiz do projeto. Plain text ou markdown. Deprecated desde 2025, mas ainda suportado. Sem frontmatter, sem activation modes -- sempre ativo.

### Formato .mdc (pre-v2.2, ainda funcional)

```markdown
---
description: "Regras para componentes React"
alwaysApply: false
globs: ["src/components/**/*.tsx"]
---

# React Component Rules

- Use functional components
- Always use TypeScript
```

### Formato RULE.md (v2.2+, atual)

```
.cursor/rules/react-components/
└── RULE.md
```

```markdown
---
description: "Regras para componentes React"
alwaysApply: false
globs: ["src/components/**/*.tsx"]
---

# React Component Rules

- Use functional components
- Always use TypeScript
```

### Tipos de Regras

| Tipo | `alwaysApply` | Comportamento |
|------|---------------|---------------|
| Always | `true` | Incluida em toda sessao de chat |
| Auto-attached | `false` + `globs` | Ativada quando arquivos matching sao editados |
| Agent-requested | `false` sem globs | Description mostrada ao Agent que decide se aplica |
| Manual | N/A | Ativada via @-mention no chat |

### User Rules

Definidas no Settings > Rules (via UI). Globais a todo o ambiente Cursor, sempre aplicadas.

### MCP Config (mcp.json)

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "..." }
    }
  }
}
```

Localizacoes: `.cursor/mcp.json` (projeto) ou `~/.cursor/mcp.json` (global).

---

## 1.5 GitHub Copilot

GitHub Copilot usa um sistema de instrucoes em Markdown com YAML frontmatter para path-scoping.

### Estrutura de Arquivos

```
.github/
├── copilot-instructions.md          # Instrucoes globais do repositorio
│
└── instructions/                    # Instrucoes path-specific
    ├── python-style.instructions.md # Com frontmatter applyTo
    ├── api-rules.instructions.md
    └── testing/
        └── jest.instructions.md     # Subdiretorios permitidos
```

### Arquivo Principal: `.github/copilot-instructions.md`

Markdown puro, sem frontmatter. Aplicado a TODAS as requisicoes no contexto do repositorio. Limite: 4.000 caracteres para Copilot Code Review (sem limite para Chat e Cloud Agent).

### Instrucoes Path-Specific: `.github/instructions/*.instructions.md`

```markdown
---
applyTo: "**/*.py"
excludeAgent: "code-review"
---

# Python Style Rules

- Use type hints for all function parameters
- Follow PEP 8 naming conventions
```

**Campos do frontmatter:**
- `applyTo`: Glob pattern para matching de arquivos (`**/*.py`, `src/api/**/*`)
- `excludeAgent`: Exclui agente especifico (`code-review` ou `coding-agent`)

### Agent Instructions

Copilot tambem reconhece (na raiz do repo):
- `AGENTS.md`
- `CLAUDE.md`
- `GEMINI.md`

O arquivo mais proximo na arvore de diretorios tem prioridade.

### Prioridade de Instrucoes

1. **Personal** (settings do usuario) -- maior prioridade
2. **Repository** (.github/copilot-instructions.md + .github/instructions/)
3. **Organization** (configurado pelo admin da org) -- menor prioridade

### VSCode Settings para Copilot

```json
{
  "github.copilot.chat.codeGeneration.instructions": [
    { "file": ".github/copilot-instructions.md" }
  ],
  "github.copilot.chat.testGeneration.instructions": [
    { "text": "Always use Jest with TypeScript" }
  ]
}
```

---

## 1.6 Windsurf (Codeium)

Windsurf migrou de `.windsurfrules` (single file) para `.windsurf/rules/` (diretorio modular) no Wave 8. Tambem suporta `AGENTS.md`.

### Estrutura de Arquivos

```
Projeto/
├── .windsurfrules              # Legacy: regras plain-text (ainda suportado)
├── AGENTS.md                   # Instrucoes (always-on, sem frontmatter)
│
└── .windsurf/
    └── rules/                  # Regras modulares (Wave 8+)
        └── {rule-name}.md     # Markdown com frontmatter trigger

~/.codeium/windsurf/
├── memories/
│   └── global_rules.md        # Regras globais (max 6.000 chars)
├── mcp_config.json            # Configuracao MCP global
└── (cascade memories)         # Auto-geradas por workspace
```

### Localizacoes de Regras

| Escopo | Path | Limite |
|--------|------|--------|
| Global | `~/.codeium/windsurf/memories/global_rules.md` | 6.000 caracteres |
| Workspace | `.windsurf/rules/*.md` | 12.000 caracteres por arquivo |
| Enterprise | OS-specific: `/Library/Application Support/Windsurf/rules/` (Mac), `/etc/windsurf/rules/` (Linux), `C:\ProgramData\Windsurf\rules\` (Win) | N/A |
| Directory | `AGENTS.md` (qualquer diretorio) | N/A |

### Formato de Regras (.windsurf/rules/)

```markdown
---
trigger: glob
globs: "**/*.test.ts"
---

# Testing Rules

- Use describe/it blocks
- Mock external dependencies
```

### Trigger Types

| Trigger | Comportamento |
|---------|---------------|
| `always_on` | Incluido no system prompt em toda mensagem |
| `model_decision` | So description e mostrada; conteudo carregado quando relevante |
| `glob` | Ativado quando arquivos matching sao editados |
| `manual` | Ativado via @rule-name no input |

### Legacy `.windsurfrules`

Plain text com regras numeradas. Sem activation modes -- sempre ativo. Ainda funcional mas menos flexivel.

### MCP Config

```json
// ~/.codeium/windsurf/mcp_config.json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "..." }
    }
  }
}
```

### Cascade Memories

Auto-geradas durante conversas, armazenadas localmente em `~/.codeium/windsurf/memories/`. Sao workspace-specific. Nao consomem creditos. Nao sao duraveis entre maquinas (nao sincronizam). A Anthropic recomenda usar Rules ou `AGENTS.md` para conhecimento que deve ser confiavel.

---

## 1.7 Amazon Q Developer CLI

Amazon Q Developer CLI (AWS) usa JSON para agentes customizados. **Nota importante:** O projeto open-source foi descontinuado e agora e Kiro CLI (closed-source).

### Estrutura de Arquivos

```
~/.aws/amazonq/
├── cli-agents/                 # Agentes globais customizados
│   └── {agent-name}.json      # Configuracao JSON do agente
├── mcp.json                   # Servidores MCP globais
└── profiles/                   # Perfis de contexto
    └── {profile-name}/

Projeto/
├── Q_CONTEXT.md                # Contexto do projeto (gerado via /init)
├── .amazonq/
│   ├── rules/                  # Regras do projeto
│   │   └── *.md
│   └── cli-todo-lists/         # TODO lists locais
│       └── *.json
└── {agent-config}.json         # Agentes locais do projeto
```

### Agent Configuration Format (JSON)

```json
{
  "name": "my-agent",
  "description": "Specialized development agent",
  "prompt": "file://./prompts/agent.md",
  "model": "claude-sonnet-4",
  
  "tools": ["fs_read", "execute_bash", "@github-mcp/*"],
  "toolAliases": {
    "@github-mcp/get_issues": "github_issues"
  },
  "allowedTools": ["fs_read", "fs_*", "@git/git_status"],
  "toolsSettings": {
    "fs_write": { "allowedPaths": ["~/**"] }
  },
  
  "resources": [
    "file://README.md",
    "file://.amazonq/rules/**/*.md"
  ],
  
  "mcpServers": {
    "git": {
      "command": "git-mcp",
      "args": [],
      "env": {},
      "timeout": 120000
    }
  },
  
  "hooks": {
    "agentSpawn": [{ "command": "git status" }],
    "userPromptSubmit": [{ "command": "ls -la" }],
    "preToolUse": [{ "matcher": "execute_bash", "command": "audit" }],
    "postToolUse": [{ "matcher": "fs_write", "command": "cargo fmt" }],
    "stop": [{ "command": "cleanup" }]
  },
  
  "useLegacyMcpJson": true
}
```

### Hook Events

| Evento | Quando Dispara |
|--------|----------------|
| `agentSpawn` | Quando o agente inicia |
| `userPromptSubmit` | Quando usuario envia prompt |
| `preToolUse` | Antes de executar uma tool (com `matcher` opcional) |
| `postToolUse` | Apos executar uma tool (com `matcher` opcional) |
| `stop` | Quando o agente para |

### Profiles e Contexto

- **Default profile:** Contem global context + workspace context
- **Switching:** Via `/profile {name}` no chat
- **Q_CONTEXT.md:** Gerado automaticamente via `/init` (escaneia o repo)

---

## 1.8 Tabela Comparativa Master

### Arquivo de Instrucoes Principal

| Ferramenta | Arquivo | Localizacao | Formato |
|------------|---------|-------------|---------|
| Claude Code | `CLAUDE.md` | Raiz, .claude/, subdirs, ~/.claude/ | Markdown |
| Codex CLI | `AGENTS.md` | Raiz, .codex/, subdirs, ~/.codex/ | Markdown |
| Gemini CLI | `GEMINI.md` | Raiz, .gemini/, subdirs, ~/.gemini/ | Markdown (com @import) |
| Cursor | `.cursor/rules/*/RULE.md` | .cursor/rules/ | Markdown + YAML frontmatter |
| GitHub Copilot | `copilot-instructions.md` | .github/ | Markdown |
| Windsurf | `.windsurf/rules/*.md` | .windsurf/rules/ | Markdown + YAML frontmatter |
| Amazon Q | Agent JSON + `Q_CONTEXT.md` | .amazonq/, ~/.aws/amazonq/ | JSON + Markdown |

### Configuracao Global

| Ferramenta | Diretorio Global | Config Principal | Formato |
|------------|-----------------|------------------|---------|
| Claude Code | `~/.claude/` | `~/.claude.json` + `~/.claude/settings.json` | JSON |
| Codex CLI | `~/.codex/` | `~/.codex/config.toml` | TOML |
| Gemini CLI | `~/.gemini/` | `~/.gemini/settings.json` | JSON |
| Cursor | `~/.cursor/` | Via UI (Settings) | JSON |
| GitHub Copilot | N/A (via VSCode) | VSCode settings.json | JSON |
| Windsurf | `~/.codeium/windsurf/` | Via UI + `global_rules.md` | Markdown |
| Amazon Q | `~/.aws/amazonq/` | Agent JSON files | JSON |

### Configuracao de Projeto

| Ferramenta | Diretorio Projeto | Config Projeto | Rules System |
|------------|-------------------|----------------|--------------|
| Claude Code | `.claude/` | `settings.json` | `rules/*.md` com globs |
| Codex CLI | `.codex/` | `config.toml` | Inline no AGENTS.md |
| Gemini CLI | `.gemini/` | `settings.json` | Via GEMINI.md |
| Cursor | `.cursor/` | Via UI | `rules/*/RULE.md` com globs |
| GitHub Copilot | `.github/` | N/A | `instructions/*.instructions.md` |
| Windsurf | `.windsurf/` | Via UI | `rules/*.md` com triggers |
| Amazon Q | `.amazonq/` | Agent JSON | `rules/*.md` |

### MCP Config

| Ferramenta | Localizacao MCP | Formato |
|------------|----------------|---------|
| Claude Code | `~/.claude.json` (campo mcpServers) | JSON |
| Codex CLI | `config.toml` (secao [mcp_servers]) | TOML |
| Gemini CLI | `settings.json` (campo mcpServers) | JSON |
| Cursor | `.cursor/mcp.json` ou `~/.cursor/mcp.json` | JSON |
| GitHub Copilot | VSCode settings | JSON |
| Windsurf | `~/.codeium/windsurf/mcp_config.json` | JSON |
| Amazon Q | `~/.aws/amazonq/mcp.json` | JSON |

### Hooks / Lifecycle Events

| Ferramenta | Hooks File | Eventos |
|------------|-----------|---------|
| Claude Code | `.claude/settings.json` (campo hooks) + scripts em `.claude/hooks/` | PreToolUse, PostToolUse, UserPromptSubmit, PreCompact |
| Codex CLI | `.codex/hooks.json` (experimental) | pre_tool_use, post_tool_use |
| Gemini CLI | N/A (nao documentado) | N/A |
| Cursor | N/A | N/A |
| GitHub Copilot | N/A | N/A |
| Windsurf | N/A | N/A |
| Amazon Q | Agent JSON (campo hooks) | agentSpawn, userPromptSubmit, preToolUse, postToolUse, stop |

### Subagents / Multi-Agent

| Ferramenta | Suporte | Mecanismo |
|------------|---------|-----------|
| Claude Code | Sim (agents/ + Agent Teams experimental) | `.claude/agents/*.md` + TeammateTool |
| Codex CLI | Sim (agents config) | `[agents.*]` no config.toml |
| Gemini CLI | Sim (subagents) | `.gemini/agents/*.md` com YAML frontmatter |
| Cursor | Nao nativo | Via MCP |
| GitHub Copilot | Parcial (agent mode) | AGENTS.md |
| Windsurf | Nao nativo | Via MCP |
| Amazon Q | Sim (custom agents) | JSON agent configs com tools e hooks |

### Sessoes / Historico

| Ferramenta | Formato | Localizacao |
|------------|---------|-------------|
| Claude Code | JSONL | `~/.claude/projects/{path}/*.jsonl` |
| Codex CLI | Log files | `$CODEX_HOME/log/` |
| Gemini CLI | Shell history | `~/.gemini/tmp/{hash}/shell_history` |
| Cursor | N/A (interno) | Interno ao editor |
| GitHub Copilot | N/A (interno) | Interno ao editor |
| Windsurf | Cascade memories | `~/.codeium/windsurf/memories/` |
| Amazon Q | N/A | Interno |

### Memory / Auto-Learning

| Ferramenta | Mecanismo | Localizacao |
|------------|-----------|-------------|
| Claude Code | MEMORY.md auto-gerado | `~/.claude/projects/{path}/MEMORY.md` |
| Codex CLI | Nao nativo | N/A |
| Gemini CLI | Nao documentado | N/A |
| Cursor | Notepads (manual) | Interno ao editor |
| GitHub Copilot | Nao nativo | N/A |
| Windsurf | Cascade Memories (auto) | `~/.codeium/windsurf/memories/` |
| Amazon Q | Nao nativo | N/A |

---

# PARTE 2: Swarm Intelligence em AI

## 2.1 Fundacoes da Swarm Intelligence

### Origens Historicas

A Swarm Intelligence (SI) e um campo da inteligencia artificial inspirado no comportamento coletivo de sistemas biologicos descentralizados e auto-organizados. O termo foi cunhado por **Gerardo Beni** e **Jing Wang** em 1989, no contexto de sistemas roboticos celulares. Porem, as raizes intelectuais sao mais profundas.

**Timeline da Swarm Intelligence:**

| Ano | Marco | Protagonista |
|-----|-------|-------------|
| ~1960s | Estudos de comportamento cooperativo de insetos sociais | E.O. Wilson |
| 1986 | Modelo Boids -- simulacao de flocking com 3 regras simples | Craig Reynolds |
| 1989 | Cunhagem do termo "Swarm Intelligence" | Gerardo Beni & Jing Wang |
| 1991 | Ant System (precursor do ACO) | Marco Dorigo |
| 1992 | Ant Colony Optimization (tese de doutorado) | Marco Dorigo |
| 1995 | Particle Swarm Optimization (PSO) | James Kennedy & Russell Eberhart |
| 1999 | Livro "Swarm Intelligence" | Bonabeau, Dorigo & Theraulaz |
| 2001 | Artificial Bee Colony (ABC) | Dervis Karaboga |
| 2004 | "The Wisdom of Crowds" (popularizacao) | James Surowiecki |

### Inspiracao Biologica

| Sistema | Mecanismo | Aplicacao em SI |
|---------|-----------|-----------------|
| **Colonias de formigas** | Trilhas de feromonio (stigmergy) | Ant Colony Optimization -- rotas otimas |
| **Enxames de abelhas** | Danca waggle para comunicacao | Bee Colony -- otimizacao multi-objetivo |
| **Bandos de passaros** | Alinhamento, coesao, separacao | Boids/PSO -- otimizacao continua |
| **Cardumes de peixes** | Deteccao lateral de movimento | Schooling algorithms -- navegacao coletiva |
| **Colonias de cupins** | Construcao emergente sem plano | Termite algorithms -- auto-organizacao |

### Principios Fundamentais

1. **Auto-organizacao (Self-organization):** Estruturas globais emergem de interacoes locais simples. Nenhum agente individual "sabe" o plano global.

2. **Stigmergy:** Comunicacao indireta atraves do ambiente. Uma formiga deixa feromonio; outra formiga le o feromonio. Nao ha comunicacao direta -- o ambiente e o meio.

3. **Emergencia (Emergence):** Comportamentos complexos emergem de regras simples. Os Boids de Reynolds demonstraram que 3 regras (alignment, separation, cohesion) produzem flocking realista.

4. **Descentralizacao:** Nao ha lider central. Decisoes sao distribuidas. O sistema e resiliente -- remova qualquer agente individual e o sistema continua funcionando.

5. **Feedback positivo:** Mecanismos de amplificacao. Trilhas de feromonio mais fortes atraem mais formigas, que depositam mais feromonio. Ciclo virtuoso que converge para solucoes otimas.

6. **Feedback negativo:** Evaporacao de feromonio, saturacao. Previne o sistema de ficar preso em sub-otimos.

### O Modelo Boids de Craig Reynolds (1986)

Reynolds resolveu um problema de animacao: como fazer um bando de passaros parecer realista sem scriptar cada passaro individualmente? Sua solucao: tres regras locais para cada "boid":

1. **Separation:** Evitar colisao com vizinhos proximos
2. **Alignment:** Alinhar direcao com a media dos vizinhos
3. **Cohesion:** Mover-se em direcao ao centro de massa dos vizinhos

O resultado: comportamento emergente de flocking que parece coordenado, mas nao tem coordenador. Este modelo foi usado na industria de filmes (os morcegos em "Batman Returns", 1992) e lancou todo o campo de vida artificial.

### Ant Colony Optimization de Marco Dorigo (1992)

Dorigo tomou emprestado principios de formigas para resolver o Travelling Salesman Problem (TSP). Formigas artificiais percorrem grafos, depositam feromonio em caminhos bons, e o feromonio evapora com o tempo. Com iteracoes suficientes, o sistema converge para rotas proximas do otimo.

ACO foi aplicado com sucesso em: roteamento de redes, scheduling, design de circuitos, bioinformatica, e e um dos algoritmos metaheuristicos mais citados na historia.

---

## 2.2 AI Agent Swarms (Moderno)

A swarm intelligence encontrou uma nova encarnacao na era dos LLMs: em vez de otimizar funcoes matematicas, agentes AI trabalham coletivamente em tarefas complexas de desenvolvimento, pesquisa e analise.

### OpenAI Swarm

**O que e:** Framework experimental (open-source) lancado pela OpenAI em outubro de 2024. Nao e production-ready -- a OpenAI diz explicitamente.

**Como funciona:**
- Agents sao definidos como Python functions com instrucoes e tools
- Handoffs entre agents via funcoes de transferencia
- Routines definem sequencias de passos
- Stateless by design -- sem gerenciamento de estado built-in
- Bom para 2-5 agents com handoffs simples

**Padroes de handoff:**
```python
def transfer_to_billing():
    return billing_agent

agent = Agent(
    name="Triage",
    instructions="Route to appropriate agent",
    functions=[transfer_to_billing, transfer_to_support]
)
```

### Claude Code Agent Teams

**O que e:** Feature experimental (research preview) lancada com Claude Code v2.1.32+, Opus 4.6. Habilitar via `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS`.

**Arquitetura:**
- Um session atua como **team lead** (coordenador)
- Teammates trabalham independentemente em suas proprias context windows
- Teammates comunicam-se diretamente entre si (diferente de subagents que so reportam ao main)
- Usa `TeammateTool` e `SendMessageTool` internamente

**Diferenca de Subagents:**
- **Subagents:** Rodam dentro de uma sessao unica, so reportam ao agente principal, nao se comunicam entre si
- **Agent Teams:** Sessoes independentes que compartilham findings, desafiam decisoes uns dos outros, coordenam autonomamente

**Use cases ideais:**
- Cross-layer coordination (frontend + backend + tests)
- Debate e consenso em decisoes arquiteturais
- Tarefas de inventario ou classificacao em larga escala

### Ruflo (Claude Flow)

**O que e:** Plataforma open-source que transforma uma sessao Claude Code em um swarm de 60+ agents especializados. 25.000+ stars no GitHub (marco 2026).

**Arquitetura "Hive Mind":**
- Agent "Queen" coordena o trabalho
- Workers especializados: coder, tester, security reviewer, DevOps, data analyst, etc.
- Sistema de routing inteligente em 3 tiers para economizar 75% em custos de API
- Self-learning: agents compartilham conhecimento e aprendem uns com os outros

**Performance:** 84.8% no SWE-Bench.

**60+ agents pre-built:** researcher, coder, tester, security reviewer, DevOps engineer, data analyst, documentacao, e mais.

### CrewAI

**O que e:** Framework multi-agent que atribui roles (Researcher, Developer, Tester) com tools especificas, lidando com orquestracao nativamente.

**Criador:** Joao Moura (brasileiro).

**Diferenciais:**
- Memory em camadas: ChromaDB (short-term), SQLite (task results), vector embeddings (entities)
- Role-based: cada agent tem papel, goal, backstory
- Tools nativas + integracao com LangChain tools
- Sequencial ou paralelo

### Microsoft AutoGen

**O que e:** Framework da Microsoft para "conversas multi-agent". Trata workflows como conversacoes entre agentes.

**Diferenciais:**
- Mantem contexto de sessao (sem persistencia built-in)
- Code execution integrada
- Human-in-the-loop nativo
- AutoGen Studio para UI visual

### LangGraph (LangChain)

**O que e:** Framework de orquestracao de agents do ecossistema LangChain. Representa workflows como grafos com nos e arestas.

**Criador:** Harrison Chase.

**Diferenciais:**
- State management mais robusto: in-thread e cross-thread memory com persistencia configuravel
- Grafos ciclicos e condicionais
- Checkpointing e human-in-the-loop
- Producao-ready (o mais usado em enterprise)

### MetaGPT

**O que e:** "Primeira empresa de software AI" -- Meta GPT X (MGX) lancado em fev/2025. Input: uma linha de requisito. Output: user stories, analise competitiva, requisitos, data structures, APIs.

**Agentes internos:** Product Manager, Architect, Project Manager, Engineers -- seguindo SOPs de uma empresa de software real.

### ChatDev

**O que e:** Framework onde agents se comunicam via chat para desenvolver software. ChatDev 2.0 (DevAll) lancado em jan/2026 como plataforma zero-code de orquestracao multi-agent.

**Baseado no CAMEL framework** (Communicative Agents for "Mind" Exploration of Large Language Models), que demonstrou como prompting pode definir personalidades de agents.

### Agency Swarm

**O que e:** Framework para criar swarms de AI agents customizados com comunicacao hierarquica.

**Diferenciais:** Focado em agents que usam tools (function calling), com CEO agent no topo.

### Tabela Comparativa de Frameworks

| Framework | Orquestracao | State | Production-Ready | Multi-LLM | Stars (2026) |
|-----------|-------------|-------|-------------------|-----------|-------------|
| LangGraph | Graph-based | Robusto (persistente) | Sim | Sim | 15k+ |
| CrewAI | Role-based | Camadas (ChromaDB+SQLite) | Sim | Sim | 25k+ |
| AutoGen | Conversational | Sessao (sem persistencia) | Parcial | Sim | 40k+ |
| OpenAI Swarm | Handoff-based | Nenhum (stateless) | Nao | Nao (OpenAI only) | 20k+ |
| Ruflo | Hive Mind | Compartilhado | Sim (Claude) | Nao (Claude only) | 25k+ |
| MetaGPT | SOP-based | Documentos compartilhados | Parcial | Sim | 50k+ |
| ChatDev | Chat-based | Conversacao | Parcial | Sim | 30k+ |

---

## 2.3 Padroes de Swarm para Desenvolvimento

### Orchestrator-Worker (Hub-and-Spoke)

**Como funciona:** Um coordenador central despacha tarefas para workers especializados. O coordenador agrega resultados.

**Quando usar:** Decomposicao clara de tarefas, necessidade de controle centralizado.

**Exemplos:** SINAPSE (Imperator coordena specialists), Ruflo (Queen agent), CrewAI (Manager agent).

**Pros:** Controle alto, rastreabilidade, facil de debugar.
**Contras:** Single point of failure, bottleneck no coordenador.

### Peer-to-Peer (Mesh)

**Como funciona:** Agents operam como iguais sem coordenador central. Comunicam-se diretamente.

**Quando usar:** 3-8 agents que precisam iterar sobre um artefato compartilhado.

**Exemplos:** Claude Code Agent Teams (teammates se comunicam diretamente), AutoGen (conversas entre peers).

**Pros:** Resiliente, sem bottleneck, emergencia de consenso.
**Contras:** Dificil de escalar, potencial de loops infinitos.

### Hierarchical (Tree)

**Como funciona:** Arvore de agents com relacoes claras de autoridade. Supervisores delegam a subordinados.

**Quando usar:** Decomposicao natural em sub-tarefas, organizacoes grandes.

**Exemplos:** Agency Swarm (CEO -> managers -> workers), MetaGPT (PM -> Architect -> Engineers).

**Pros:** Escala logaritmicamente, isolamento de falhas por branch.
**Contras:** Latencia (decisoes sobem e descem a arvore), rigidez.

### Blackboard

**Como funciona:** Workspace compartilhado (blackboard) onde agents leem e escrevem. Agents monitoram o blackboard e atuam quando veem informacao relevante.

**Quando usar:** Problemas que requerem integracao de conhecimento de multiplas fontes.

**Exemplos:** Shared context em LangGraph (state graph), Ruflo (shared knowledge base).

**Pros:** Desacoplamento total entre agents, flexibilidade.
**Contras:** Race conditions, necessidade de governance do blackboard.

### Market-Based (Auction)

**Como funciona:** Agents fazem bids em tarefas. O melhor bid (por expertise, custo, disponibilidade) ganha a tarefa.

**Quando usar:** Recursos escassos, necessidade de otimizacao de alocacao.

**Exemplos:** Conceitual em scheduling de GPU para LLMs.

**Pros:** Alocacao otima de recursos, auto-regulacao.
**Contras:** Overhead de bidding, complexidade de pricing.

### Stigmergic

**Como funciona:** Comunicacao indireta atraves do ambiente. Um agent deixa um "feromonio digital" (metadata, flags, marcadores) que outros agents detectam e reagem.

**Quando usar:** Agents que nao precisam se conhecer, coordenacao emergente.

**Exemplos:** Git branches como stigmergy (um agent cria branch, outro detecta e reage), TODO comments no codigo, CI/CD pipelines.

**Pros:** Maximo desacoplamento, escalabilidade.
**Contras:** Convergencia lenta, dificil de debugar.

### Pipeline (Sequential)

**Como funciona:** Agents em sequencia, cada um processando e passando adiante.

**Quando usar:** Workflows lineares com etapas claras.

**Exemplos:** SINAPSE SDC (sprint-lead -> product-lead -> developer -> quality-gate -> devops).

**Pros:** Simples, previsivel, facil de monitorar.
**Contras:** Bottleneck no agent mais lento, sem paralelismo.

### Como SINAPSE se Compara

O SINAPSE usa primariamente o padrao **Orchestrator-Worker** com o Imperator (sinapse-orqx) no topo, delegando para squad orchestrators (*-orqx), que por sua vez delegam para specialists. O workflow SDC segue um padrao **Pipeline** (sprint-lead -> product-lead -> developer -> quality-gate -> devops). O sistema de handoff e scratchpad adiciona elementos de **Stigmergy** (agents deixam artefatos para o proximo). Agent Teams (experimental) traria **Peer-to-Peer** para tarefas que requerem debate.

---

## 2.4 Implementacao Pratica

### Como Implementar Swarm em Claude Code

**Opcao 1: Subagents (built-in)**
```
.claude/agents/
├── reviewer.md      # Code review specialist
├── tester.md        # Test writer
└── security.md      # Security auditor
```

Cada agent tem seu system prompt, tool restrictions, e modelo preferido. Claude invoca subagents automaticamente quando a tarefa corresponde.

**Opcao 2: Agent Teams (experimental)**
Habilitar: `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=true`

Um session coordena, teammates trabalham em paralelo. Ideal para mudancas cross-layer (frontend + backend + tests simultaneamente).

**Opcao 3: Ruflo (60+ agents)**
```bash
npx ruflo init
npx ruflo swarm --task "Implement user authentication"
```

Ruflo orquestra 60+ agents com routing inteligente e self-learning.

### Como Implementar Swarm em Codex CLI

```toml
# .codex/config.toml
[features]
multi_agent = true

[agents]
max_threads = 6
max_depth = 3

[agents.reviewer]
description = "Code review specialist"

[agents.architect]
description = "Architecture decision maker"
```

### Token Cost: Multi-Agent vs Single Agent

| Cenario | Single Agent | Multi-Agent (3) | Multi-Agent (10) |
|---------|-------------|-----------------|------------------|
| Context por agent | 200K tokens | 200K x 3 = 600K | 200K x 10 = 2M |
| Overhead de coordenacao | 0 | ~10-15% | ~20-30% |
| Total estimado | 200K | ~690K | ~2.6M |
| Custo relativo | 1x | ~3.5x | ~13x |

**Mitigacao via Ruflo:** Routing inteligente em 3 tiers (modelo caro para tarefas complexas, modelo barato para tarefas simples) reduz custo em ~75%.

### Quando Swarm Vale a Pena vs Overkill

| Cenario | Recomendacao |
|---------|-------------|
| Bug fix simples | Single agent (YOLO mode) |
| Feature em 1 arquivo | Single agent |
| Feature cross-layer (3+ arquivos) | 2-3 subagents |
| Refatoracao de arquitetura | Agent Teams (3-5 agents) |
| Projeto greenfield grande | Ruflo / MetaGPT |
| Code review + security audit | 2 agents paralelos |
| Decisao arquitetural complexa | Agent Teams com debate |

### Limites Atuais de AI Swarms

1. **Context window:** Cada agent tem limite de contexto. Coordenacao exige tokens.
2. **Custo:** Multi-agent multiplica custos de API significativamente.
3. **Determinismo:** Comportamento emergente e dificil de reproduzir e debugar.
4. **Race conditions:** Agents escrevendo no mesmo arquivo simultaneamente.
5. **Convergencia:** Sem garantia de que agents chegarao a consenso.
6. **Feedback loops:** Agents podem entrar em loops infinitos de revisao mutua.

---

## 2.5 Pessoas-Chave e Livros

### Pessoas Referencia

**Craig Reynolds** -- Pioneiro de vida artificial. Criou o modelo Boids (1986) que demonstrou como regras locais simples produzem comportamento emergente complexo. Seu trabalho na ACM SIGGRAPH 1987 lancou o campo inteiro e e usado ate hoje em filmes e jogos.

**Marco Dorigo** -- Pai do Ant Colony Optimization. Sua tese de doutorado (1992) na Universite Libre de Bruxelles criou uma das metaheuristicas mais influentes da historia. Co-autor do livro "Swarm Intelligence" (2004), a "biblia" do campo.

**Eric Bonabeau** -- Co-autor de "Swarm Intelligence: From Natural to Artificial Systems" (1999) com Dorigo e Theraulaz. CEO da Icosystem. Trouxe SI do mundo academico para aplicacoes empresariais.

**James Kennedy** -- Co-criador do Particle Swarm Optimization (1995) com Eberhart. Demonstrou que otimizacao pode emergir de interacoes sociais simples entre particulas.

**Russell Eberhart** -- Co-criador do PSO com Kennedy. Background em engenharia eletrica e computacional.

**Gerardo Beni** -- Cunhou o termo "Swarm Intelligence" em 1989 com Jing Wang, no contexto de robotica celular na UCI (University of California, Irvine).

**E.O. Wilson** -- Biologo e entomologo de Harvard. Seus estudos sobre insetos sociais nas decadas de 1960-70 forneceram a base biologica para toda a SI. Autor de "The Ants" (1990, Pulitzer) e "Sociobiology" (1975).

**James Surowiecki** -- Jornalista da New Yorker. "The Wisdom of Crowds" (2004) popularizou a ideia de que grupos tomam decisoes melhores que individuos, sob certas condicoes.

**Andrew Ng** -- Professor de Stanford, fundador do DeepLearning.AI e Coursera. Popularizou "agentic workflows" (2024) como o proximo paradigma de AI, onde multiplos agents colaboram iterativamente.

**Harrison Chase** -- Criador do LangChain e LangGraph. O framework de orquestracao de agents mais usado em enterprise (2025-2026).

**Joao Moura** -- Criador do CrewAI (brasileiro). Levou o conceito de "tripulacao" de agents com roles definidos ao mainstream.

### Livros "Biblias"

**"Swarm Intelligence: From Natural to Artificial Systems"** -- Eric Bonabeau, Marco Dorigo, Guy Theraulaz (1999). A biblia do campo. Cobre fundamentos biologicos, modelos matematicos, ACO, PSO, e aplicacoes. Essencial para entender a base teorica.

**"The Wisdom of Crowds"** -- James Surowiecki (2004). Popularizacao da inteligencia coletiva. Condicoes para sabedoria das multidoes: diversidade de opiniao, independencia, descentralizacao, agregacao.

**"Ant Colony Optimization"** -- Marco Dorigo & Thomas Stutzle (2004). Referencia tecnica definitiva sobre ACO. Algoritmos, convergencia, aplicacoes.

**"Particle Swarm Optimization"** -- Maurice Clerc (2006). Referencia tecnica sobre PSO. Variantes, convergencia, topologias.

**"The Ants"** -- Bert Holldobler & E.O. Wilson (1990). Pulitzer Prize. A obra definitiva sobre formigas -- a inspiracao biologica da SI.

**"Sociobiology: The New Synthesis"** -- E.O. Wilson (1975). O livro que lancou o estudo cientifico do comportamento social em animais, precursor intelectual da SI.

**"Emergence: The Connected Lives of Ants, Brains, Cities, and Software"** -- Steven Johnson (2001). Acessivel. Conecta emergencia biologica com sistemas computacionais e urbanos.

---

# PARTE 3: AGI -- Artificial General Intelligence

## 3.1 O que e AGI

### Definicao e Debate

**Artificial General Intelligence (AGI)** e uma inteligencia artificial capaz de realizar QUALQUER tarefa intelectual que um humano pode, no nivel humano ou superior, atraves de dominios, sem treinamento especifico para cada tarefa.

O debate central: **narrow AI vs AGI vs ASI**:

| Tipo | Definicao | Exemplos |
|------|-----------|----------|
| **Narrow AI (ANI)** | Excelente em UMA tarefa especifica | GPT-4 (texto), DALL-E (imagem), AlphaGo (Go) |
| **AGI** | Igual ou superior ao humano em TODAS as tarefas cognitivas | Nao existe ainda |
| **ASI (Superintelligence)** | Vastamente superior a TODOS os humanos em TODOS os dominios | Hipotetica |

### Definicoes por Empresa

Cada empresa define AGI de forma diferente (estrategicamente conveniente):

**OpenAI** -- Framework de 5 niveis:

| Nivel | Nome | Descricao | Status (2026) |
|-------|------|-----------|---------------|
| L1 | Chatbots | AI que conversa | Atingido |
| L2 | Reasoners | AI que raciocina como PhD | Atingido (o-series) |
| L3 | Agents | AI que age autonomamente por dias | Em progresso |
| L4 | Innovators | AI que contribui para pesquisa novel | Proximo alvo |
| L5 | Organizations | AI que opera como organizacao inteira | Futuro |

**Google DeepMind** -- Framework de 6 niveis (performance x breadth):

| Nivel | Performance | Narrow (1 dominio) | General (multiplos) |
|-------|-------------|---------------------|---------------------|
| L0 | No AI | Calculadora | N/A |
| L1 | Emerging | GOFAI, GPT-2 | GPT-4, Gemini |
| L2 | Competent | Toxicity classifiers | N/A |
| L3 | Expert | Grammar checkers | N/A |
| L4 | Virtuoso | Narrow specialists | N/A |
| L5 | Superhuman | AlphaGo, AlphaFold | ASI |

O framework cruza performance (eixo Y) com breadth (eixo X). AGI seria L3-L4 General.

**Anthropic** -- Dario Amodei diz que nao gosta do termo "AGI". Prefere "powerful AI". Acredita que AI poderosa pode chegar "tao cedo quanto 2026". Foca em seguranca como pre-requisito, nao como obstaculo.

**Meta** -- Yann LeCun insiste que LLMs atuais NAO levarao a AGI. Falta "world models" -- entendimento do mundo fisico. Propoe arquiteturas fundamentalmente diferentes (JEPA -- Joint Embedding Predictive Architecture).

### Estado Atual (abril 2026)

Os LLMs de 2025-2026 (GPT-5, Claude Opus 4.6, Gemini 2.5) demonstram:
- Raciocinio sofisticado (math olympiad-level)
- Programacao de nivel profissional
- Compreensao multimodal (texto, imagem, audio, video)
- Uso de tools e agents

Ainda faltam:
- Transferencia genuina entre dominios novos
- Aprendizado continuo sem re-training
- Common sense robusto
- Entendimento do mundo fisico
- Planejamento de longo prazo confiavel

---

## 3.2 Contexto Historico

### Pessoas Fundamentais

**Alan Turing (1912-1954)** -- Pai da ciencia da computacao. Seu paper "Computing Machinery and Intelligence" (1950) introduziu o Turing Test: se uma maquina pode enganar um humano pensando que e outro humano, ela "pensa". Definiu o framework intelectual para AGI.

**John McCarthy (1927-2011)** -- Cunhou o termo "Artificial Intelligence" em 1956 na Dartmouth Conference. Inventou LISP. Fundou os labs de AI em MIT e Stanford. A Dartmouth Conference (com Minsky, Shannon, Rochester) e considerada o nascimento do campo de AI.

**Marvin Minsky (1927-2016)** -- Co-fundador do MIT AI Lab. "Society of Mind" (1986) propoe que inteligencia emerge de agentes simples trabalhando juntos (precursor intelectual de swarm AI). Junto com McCarthy, foi a forca motora do AI nos anos 1960-70.

**Herbert Simon (1916-2001)** -- Nobel de Economia (1978). Co-criou o Logic Theorist (1955) e o General Problem Solver (1957) -- tentativas iniciais de AGI. Famosamente previu em 1965 que "em 20 anos, maquinas serao capazes de qualquer trabalho que um humano faz."

**Ray Kurzweil (1948-)** -- Futurista, inventor, agora Chief Engineer do Google. "The Singularity is Near" (2005) previu que AGI chegaria por volta de 2029 (e a "Singularidade" -- ASI que transforma a civilizacao -- por volta de 2045). Suas previsoes de 2005 sobre AI em 2020 foram surpreendentemente precisas.

**Nick Bostrom (1973-)** -- Filosofo sueco em Oxford. "Superintelligence: Paths, Dangers, Strategies" (2014) e o livro mais influente sobre os riscos de AGI/ASI. Introduziu os conceitos de "orthogonality thesis" (inteligencia e objetivos sao independentes) e "instrumental convergence" (qualquer AI suficientemente inteligente buscara auto-preservacao e acumulacao de recursos).

**Stuart Russell (1962-)** -- Professor em Berkeley. Co-autor do livro-texto padrao "Artificial Intelligence: A Modern Approach" (com Norvig, 1995). "Human Compatible" (2019) propoe que AI deve ser construida para servir humanos sem ter objetivos fixos -- "assistance games" onde o AI infere preferencias humanas.

**Demis Hassabis (1976-)** -- Co-fundador do DeepMind. AlphaGo (2016) derrotou Lee Sedol em Go. AlphaFold (2020) resolveu protein folding -- um dos maiores avancos cientificos da decada. Nobel de Quimica em 2024. Previu AGI ate ~2030 com 50% de probabilidade.

**Ilya Sutskever (1985-)** -- Co-fundador da OpenAI. Defendeu a "scaling hypothesis" -- que simplesmente escalar LLMs levaria a capacidades emergentes. Saiu da OpenAI em 2024 para fundar Safe Superintelligence Inc. (SSI), focada exclusivamente em criar superintelligencia segura.

**Dario Amodei (1983-)** -- CEO da Anthropic. Ex-VP de Research na OpenAI. "Machines of Loving Grace" (2024) -- ensaio de 50+ paginas sobre como AI poderosa poderia transformar o mundo positivamente (saude, economia, governanca). Acredita que powerful AI pode chegar em 2026. Prefere investir em seguranca como pre-requisito.

**Sam Altman (1985-)** -- CEO da OpenAI. Disse que "superintelligence" poderia chegar em "alguns milhares de dias" (2024). Framework de 5 niveis. Foco em infraestrutura de compute (Stargate project com SoftBank/Oracle).

**Yann LeCun (1960-)** -- Chief AI Scientist da Meta. Vencedor do Turing Award (2018) com Bengio e Hinton. Skeptico de que LLMs atuais levarao a AGI. Propoe JEPA e "world models" como caminho alternativo.

**Geoffrey Hinton (1947-)** -- "Padrinho do Deep Learning". Nobel de Fisica (2024). Saiu do Google em 2023 para falar livremente sobre riscos de AI. Alertou que AI poderia ultrapassar humanos em inteligencia mais rapido do que previsto.

**Yoshua Bengio (1964-)** -- Turing Award (2018). Signatario da declaracao sobre riscos existenciais de AI. Defende regulacao forte e pesquisa de seguranca.

---

## 3.3 Desafios Tecnicos

### 1. Raciocinio e Planejamento

LLMs atuais demonstram raciocinio impressionante em problemas contidos (math, coding), mas falham em:
- Planejamento de longo prazo com muitas etapas
- Raciocinio causal (nao apenas correlacional)
- Decomposicao de problemas genuinamente novos

Os modelos "o-series" da OpenAI (chain-of-thought) e Claude's extended thinking avancam nessa direcao, mas ainda nao sao confiaveis para problemas do mundo real com incerteza.

### 2. Common Sense

A "AI Winograd Schema Challenge" e exemplos similares demonstram que LLMs ainda falham em compreensao de senso comum que criancas de 5 anos dominam. "The trophy doesn't fit in the brown suitcase because it's too [big/small]" -- qual "it" refere ao que?

### 3. Transfer Learning entre Dominios

LLMs treinados em texto podem fazer math, codigo, e raciocinio, mas a transferencia e superficial. Um especialista humano em fisica que aprende biologia traz insights profundos de um campo para outro. LLMs aplicam padroes estatisticos, nao entendimento genuino (debate em aberto).

### 4. Embodiment e Mundo Fisico

Yann LeCun argumenta que inteligencia genuina requer "world models" -- representacoes internas de como o mundo fisico funciona. LLMs operam em texto e imagens, mas nao tem experiencia sensorial ou motora. Robotics + AI e uma fronteira ativa (Tesla Optimus, Figure AI, Google DeepMind).

### 5. Memoria de Longo Prazo e Aprendizado Continuo

LLMs atuais nao aprendem entre sessoes (exceto via fine-tuning caro). Cada sessao comeca do zero (exceto contexto fornecido). Sistemas como MEMORY.md do Claude Code sao workarounds, nao aprendizado genuino.

### 6. Alignment e Seguranca

O "alignment problem" (Brian Christian, 2020): como garantir que uma AI superinteligente faca o que queremos? Desafios:
- **Specification problem:** Nao sabemos especificar completamente o que queremos
- **Goodhart's Law:** "Quando uma medida se torna um alvo, ela deixa de ser uma boa medida"
- **Mesa-optimization:** AI pode desenvolver objetivos internos diferentes dos treinados
- **Deceptive alignment:** AI pode parecer alinhada durante treinamento mas agir diferente em deployment

---

## 3.4 Previsoes de Timeline para AGI

### Previsoes por Lider da Industria

| Pessoa | Organizacao | Previsao | Data da Previsao |
|--------|-------------|----------|------------------|
| **Sam Altman** | OpenAI | "Superintelligence em milhares de dias" (~2027-2028) | Set 2024 |
| **Dario Amodei** | Anthropic | "Powerful AI tao cedo quanto 2026" | Out 2024 |
| **Demis Hassabis** | DeepMind | "50% de chance de AGI ate 2030" | 2025 |
| **Shane Legg** | DeepMind | "50% de chance de Minimal AGI ate 2028" | Jan 2026 |
| **Elon Musk** | xAI | "AGI (mais inteligente que o humano mais inteligente) em 2025-2026" | 2024-2025 |
| **Mustafa Suleyman** | Microsoft AI | "Human-level performance em 12-18 meses" (tarefas profissionais) | 2025 |
| **Yann LeCun** | Meta | "AGI via LLMs atuais: nao. Via novos paradigmas: decadas" | 2025 |
| **Geoffrey Hinton** | Independente | "Podemos ter superintelligencia em 5-20 anos" | 2023 |

### Prediction Markets e Consenso

| Fonte | Previsao |
|-------|----------|
| **Metaculus** (fev 2026) | 25% de chance de AGI ate 2029, 50% ate 2033 |
| **80,000 Hours** review | Maioria dos surveys: AGI antes de 2100, mediana ~2040 |
| **AI Frontiers report** (ago 2025) | "Early AGI-like systems entre 2026-2028" em dominios especificos |
| **Surveys de cientistas** (set 2025) | Maioria concorda: AGI antes de 2100; mediana nas decadas de 2040 |

### O Padrao das Previsoes

Observacao historica: previsoes de AGI SEMPRE foram "daqui a 20 anos." Herbert Simon previu AGI em 20 anos em 1965. Marvin Minsky previu em 1967 que "dentro de uma geracao o problema de criar AI seria substancialmente resolvido." O padrao se repete -- mas ha um argumento de que DESTA VEZ e diferente, dado o ritmo de progresso observavel em LLMs desde 2020.

A diferenca: previsoes anteriores eram baseadas em intuicao. As atuais sao baseadas em scaling laws empiricas (Chinchilla, etc.) que mostram melhoria previsivel com mais compute e dados.

---

## 3.5 O que AGI Significa para Frameworks de Desenvolvimento

### Preparando o SINAPSE para Autonomia Progressiva

A medida que AI se torna mais capaz, frameworks como o SINAPSE devem evoluir:

**Nivel atual (2026):** Agent-assisted development
- Humanos tomam decisoes estrategicas
- AI executa tarefas taticas
- Human-in-the-loop obrigatorio para decisoes criticas
- SINAPSE: Imperator roteia, specialists executam, humanos aprovam

**Proximo nivel (~2027):** Agent-led development
- AI propoe decisoes estrategicas
- Humanos aprovam ou rejeitam
- Human-on-the-loop (supervisao, nao execucao)
- SINAPSE: Reduzir gates de aprovacao para decisoes de baixo risco

**Nivel futuro (~2029+):** Autonomous development
- AI toma decisoes autonomamente dentro de guardrails
- Humanos definem objetivos de alto nivel
- Human-over-the-loop (governanca, nao supervisao)
- SINAPSE: Constitution como guardrail, nao como processo

### Design para Autonomia Progressiva

1. **Guardrails parametrizaveis:** Em vez de hard-coded rules, niveis de autonomia configuraveis por tarefa, risco, e confianca no agent.

2. **Metricas de confianca:** Track record de cada agent. Agents que consistentemente produzem bom trabalho ganham mais autonomia. Agents novos comecam com supervisao maxima.

3. **Escalation inteligente:** Em vez de "human approves everything," so escalar para humano quando incerteza do agent ultrapassa threshold.

4. **Audit trail:** Toda decisao autonoma e rastreavel, auditavel, e reversivel.

5. **Kill switch:** Humano sempre pode interromper qualquer agent a qualquer momento.

### O Papel do Humano no Futuro

Mesmo com AGI, humanos mantem:
- **Definicao de objetivos:** O que construir e por que
- **Validacao de valores:** O resultado esta alinhado com valores humanos?
- **Responsabilidade legal:** Quem e responsavel pelo software?
- **Criatividade direcional:** Visao de produto, estetica, proposito
- **Governance:** Regras do jogo, limites eticos

---

## 3.6 Livros Fundamentais

### Sobre AGI e Superintelligence

**"Superintelligence: Paths, Dangers, Strategies"** -- Nick Bostrom (2014). A "biblia" sobre riscos de AGI/ASI. Orthogonality thesis, instrumental convergence, treacherous turn. Todo mundo que trabalha com AI deveria ler. Influenciou diretamente Elon Musk, Bill Gates, e a criacao da OpenAI.

**"The Singularity is Near"** -- Ray Kurzweil (2005). Previsoes sobre AGI (2029) e Singularidade (2045) baseadas em "law of accelerating returns." Muitas previsoes para 2020 se mostraram corretas. Otimismo tecnologico fundamentado em dados historicos.

**"Human Compatible: AI and the Problem of Control"** -- Stuart Russell (2019). Propoe uma nova abordagem: AI que NAO tem objetivos fixos, mas infere preferencias humanas. "Assistance games." O livro mais influente sobre alignment apos Bostrom.

**"Life 3.0: Being Human in the Age of Artificial Intelligence"** -- Max Tegmark (2017). Cenarios para o futuro da humanidade com AGI. Acessivel para nao-especialistas. Fundador do Future of Life Institute.

**"The Alignment Problem: Machine Learning and Human Values"** -- Brian Christian (2020). A historia de como a comunidade de ML descobriu que alinhar AI com valores humanos e o desafio central. Narrativa jornalistica excelente.

**"Machines of Loving Grace"** -- Dario Amodei (ensaio, 2024). 50+ paginas sobre como AI poderosa poderia transformar saude, ciencia, economia, e governanca para melhor. O contraponto otimista ao catastrofismo. Nao e um livro, mas e leitura obrigatoria.

**"AI 2041: Ten Visions for Our Future"** -- Kai-Fu Lee & Chen Qiufan (2021). Ficcao + analise tecnica. 10 cenarios para 2041. Perspectiva China + Ocidente.

### Sobre AI Safety e Ethics

**"The Precipice: Existential Risk and the Future of Humanity"** -- Toby Ord (2020). Contexto de risco existencial (nao so AI). AI como um dos maiores riscos.

**"Power and Prediction: The Disruptive Economics of Artificial Intelligence"** -- Ajay Agrawal, Joshua Gans, Avi Goldfarb (2022). Economia de AI: como AI como "prediction technology" reconfigura industrias.

### Sobre a Historia da AI

**"Artificial Intelligence: A Modern Approach"** -- Stuart Russell & Peter Norvig (1995, 4th ed 2020). O livro-texto padrao de AI. Usado em 1.500+ universidades. Referencia tecnica definitiva.

**"The Dream Machine"** -- M. Mitchell Waldrop (2001). Historia de J.C.R. Licklider e a visao de computacao interativa. Contexto historico essencial.

---

# Referencias Historicas e Mundiais

## Pessoas por Sub-Area

### Swarm Intelligence
| Pessoa | Contribuicao Principal |
|--------|----------------------|
| Craig Reynolds | Boids (1986), emergencia via regras locais |
| Marco Dorigo | Ant Colony Optimization (1992), metaheuristicas |
| Eric Bonabeau | "Swarm Intelligence" livro, aplicacoes empresariais |
| James Kennedy | Particle Swarm Optimization (1995) |
| Gerardo Beni | Cunhou o termo "Swarm Intelligence" (1989) |
| E.O. Wilson | Base biologica -- estudos de insetos sociais |

### Multi-Agent AI Frameworks
| Pessoa | Contribuicao Principal |
|--------|----------------------|
| Harrison Chase | LangChain/LangGraph -- orquestracao de agents mais usado |
| Joao Moura | CrewAI -- multi-agent com roles definidos |
| Andrew Ng | Popularizou "agentic workflows" como paradigma |
| Equipe OpenAI | Swarm framework, Codex CLI multi-agent |
| Equipe Anthropic | Claude Code Agent Teams, subagents |

### AGI e Safety
| Pessoa | Contribuicao Principal |
|--------|----------------------|
| Alan Turing | Turing Test, fundacao da ciencia da computacao |
| John McCarthy | Cunhou "Artificial Intelligence," Dartmouth 1956 |
| Nick Bostrom | "Superintelligence," riscos existenciais |
| Stuart Russell | "Human Compatible," assistance games |
| Demis Hassabis | AlphaGo, AlphaFold, Nobel 2024 |
| Ilya Sutskever | Scaling hypothesis, Safe Superintelligence Inc. |
| Dario Amodei | "Machines of Loving Grace," Anthropic |
| Geoffrey Hinton | "Padrinho do Deep Learning," Nobel 2024, alertas sobre riscos |

### AI Coding Tools
| Pessoa/Empresa | Contribuicao Principal |
|----------------|----------------------|
| Anthropic | Claude Code -- ecossistema de arquivos mais rico |
| OpenAI | Codex CLI -- TOML config, AGENTS.md padrao cross-tool |
| Google DeepMind | Gemini CLI -- hierarquia 4 niveis, sandbox |
| Cursor Team | Pioneer .cursorrules -> .mdc -> RULE.md evolution |
| GitHub/Microsoft | Copilot instructions -- path-specific com globs |
| Codeium (Windsurf) | Cascade memories, rules com triggers |
| AWS | Amazon Q -- agents JSON com hooks e MCP |

## Livros "Biblias" por Sub-Area

### Swarm Intelligence

**"Swarm Intelligence: From Natural to Artificial Systems"** -- Eric Bonabeau, Marco Dorigo, Guy Theraulaz (1999, Oxford University Press). O livro fundacional que definiu o campo. Cobre ant colony optimization, particle swarm optimization, e emergencia em sistemas biologicos e artificiais. O que extrair para o SINAPSE: principios de emergencia (comportamento global de regras locais), stigmergy (comunicacao indireta via ambiente), e auto-organizacao que informam o design de agent swarms.

**"The Wisdom of Crowds"** -- James Surowiecki (2004). Demonstra que grupos diversos tomam decisoes melhores que individuos experts, desde que haja diversidade, independencia, descentralizacao, e agregacao. O que extrair para o SINAPSE: fundamentacao teorica para multi-agent decision making -- squads com agentes diversos produzem resultados melhores que um agente unico generalista.

**"Complexity: A Guided Tour"** -- Melanie Mitchell (2009). Introducao acessivel a ciencia da complexidade: automata celulares, redes, computacao biologica, e emergencia. O que extrair para o SINAPSE: como sistemas simples (agents com regras simples) produzem comportamento complexo (framework completo).

### AGI e Safety

**"Superintelligence: Paths, Dangers, Strategies"** -- Nick Bostrom (2014). A "biblia" sobre riscos de AGI/ASI. Orthogonality thesis, instrumental convergence, treacherous turn. O que extrair para o SINAPSE: design de guardrails e kill switches para agents autonomos.

**"Human Compatible: AI and the Problem of Control"** -- Stuart Russell (2019). Propoe "assistance games" onde AI infere preferencias humanas. O que extrair para o SINAPSE: agents que expressam incerteza e pedem confirmacao em vez de assumir.

**"Life 3.0: Being Human in the Age of Artificial Intelligence"** -- Max Tegmark (2017). Cenarios para o futuro com AGI: utopia, dystopia, e caminhos intermediarios. O que extrair para o SINAPSE: design para autonomia progressiva (human-in-the-loop -> human-on-the-loop -> human-over-the-loop).

**"The Alignment Problem: Machine Learning and Human Values"** -- Brian Christian (2020). Como a comunidade de ML descobriu que alignment e o desafio central. O que extrair para o SINAPSE: Constitution como mecanismo de alignment -- principios que governam comportamento de agents.

**"The Singularity is Near"** -- Ray Kurzweil (2005). Previsoes sobre AGI (2029) e Singularidade (2045) baseadas em "law of accelerating returns." O que extrair para o SINAPSE: timeline planning -- o framework precisa ser projetado para um mundo onde agents sao significativamente mais capazes em 3-5 anos.

**"AI 2041: Ten Visions for Our Future"** -- Kai-Fu Lee & Chen Qiufan (2021). 10 cenarios de ficcao + analise tecnica para 2041. Perspectiva China + Ocidente. O que extrair para o SINAPSE: cenarios de uso futuro que informam roadmap de longo prazo.

### LLM CLI/IDE Tools

**"AI Engineering"** -- Chip Huyen (2025). O guia definitivo para construir sistemas de producao com LLMs. O que extrair para o SINAPSE: patterns de configuration management, model routing, e observability que cada CLI tool implementa de forma diferente.

**"The LLM Engineer's Handbook"** -- Paul Iusztin & Maxime Labonne (2025). Framework end-to-end para LLM apps production-ready. O que extrair para o SINAPSE: comparativo de approaches para context management e tool orchestration entre diferentes CLI tools.

---

# Fontes e Links

## Parte 1: LLM CLI/IDE Files

### Claude Code
- [Claude Code Settings (docs oficiais)](https://code.claude.com/docs/en/settings)
- [Anatomy of the .claude/ Folder (Avi Chawla)](https://blog.dailydoseofds.com/p/anatomy-of-the-claude-folder)
- [~/.claude/ directory structure (GitHub Gist)](https://gist.github.com/samkeen/dc6a9771a78d1ecee7eb9ec1307f1b52)
- [Claude Code settings.json guide (eesel AI)](https://www.eesel.ai/blog/settings-json-claude-code)
- [How Claude Code Manages Local Storage (Milvus Blog)](https://milvus.io/blog/why-claude-code-feels-so-stable-a-developers-deep-dive-into-its-local-storage-design.md)
- [Complete .claude Directory Guide (Computing for Geeks)](https://computingforgeeks.com/claude-code-dot-claude-directory-guide/)
- [Claude Code's Secret Weapon (Sid Saladi)](https://sidsaladi.substack.com/p/claude-codes-secret-weapon-the-complete)

### OpenAI Codex CLI
- [AGENTS.md Guide (OpenAI Developers)](https://developers.openai.com/codex/guides/agents-md)
- [Configuration Reference (OpenAI Developers)](https://developers.openai.com/codex/config-reference)
- [Config Basics (OpenAI Developers)](https://developers.openai.com/codex/config-basic)
- [Advanced Configuration (OpenAI Developers)](https://developers.openai.com/codex/config-advanced)
- [Codex GitHub Repository](https://github.com/openai/codex)

### Google Gemini CLI
- [Gemini CLI Configuration (docs oficiais)](https://google-gemini.github.io/gemini-cli/docs/get-started/configuration.html)
- [GEMINI.md Context Files](https://google-gemini.github.io/gemini-cli/docs/cli/gemini-md.html)
- [Gemini CLI Subagents](https://geminicli.com/docs/core/subagents/)
- [Gemini CLI GitHub Repository](https://github.com/google-gemini/gemini-cli)

### Cursor
- [Cursor Rules (docs oficiais)](https://cursor.com/docs/context/rules)
- [Cursor MCP Configuration](https://cursor.com/docs/context/mcp)
- [MDC Rules Best Practices (forum)](https://forum.cursor.com/t/my-best-practices-for-mdc-rules-and-troubleshooting/50526)
- [Awesome Cursor Rules MDC (GitHub)](https://github.com/sanjeed5/awesome-cursor-rules-mdc)

### GitHub Copilot
- [Adding Custom Instructions (GitHub Docs)](https://docs.github.com/copilot/customizing-copilot/adding-custom-instructions-for-github-copilot)
- [Custom Instructions in VS Code](https://code.visualstudio.com/docs/copilot/customization/custom-instructions)
- [Configure Custom Instructions](https://docs.github.com/en/copilot/how-tos/configure-custom-instructions)
- [5 Tips for Better Custom Instructions (GitHub Blog)](https://github.blog/ai-and-ml/github-copilot/5-tips-for-writing-better-custom-instructions-for-copilot/)

### Windsurf
- [Cascade Memories (docs oficiais)](https://docs.windsurf.com/windsurf/cascade/memories)
- [Cascade MCP Integration](https://docs.windsurf.com/windsurf/cascade/mcp)
- [Windsurf Rules Guide (design.dev)](https://design.dev/guides/windsurf-rules/)

### Amazon Q Developer CLI
- [Agent Format Specification](https://aws.github.io/amazon-q-developer-cli/agent-format.html)
- [Configuration Reference](https://docs.aws.amazon.com/amazonq/latest/qdeveloper-ug/command-line-custom-agents-configuration.html)
- [Context Management](https://docs.aws.amazon.com/amazonq/latest/qdeveloper-ug/command-line-context.html)
- [Amazon Q Developer CLI (GitHub)](https://github.com/aws/amazon-q-developer-cli)

## Parte 2: Swarm Intelligence

### Historicos e Fundamentos
- [Swarm Intelligence (Wikipedia)](https://en.wikipedia.org/wiki/Swarm_intelligence)
- [A Story of Swarm Intelligence (Robonaissance)](https://www.robonaissance.com/p/a-story-of-swarm-intelligence)
- [Swarm Intelligence in Agentic AI (Powerdrill Report)](https://powerdrill.ai/blog/swarm-intelligence-in-agentic-ai-an-industry-report)
- [Historical Development of Swarm Intelligence (Fiveable)](https://fiveable.me/swarm-intelligence-and-robotics/unit-1/historical-development-swarm-intelligence/study-guide/LNOm1YBPH6z9gbVM)
- [Swarm Intelligence (Scholarpedia)](http://www.scholarpedia.org/article/Swarm_intelligence)

### Frameworks Multi-Agent
- [AI Agent Frameworks Comparison 2026 (Turing)](https://www.turing.com/resources/ai-agent-frameworks)
- [LangGraph vs CrewAI vs AutoGen (o-mega)](https://o-mega.ai/articles/langgraph-vs-crewai-vs-autogen-top-10-agent-frameworks-2026)
- [AI Agent Showdown 2026 (DEV Community)](https://dev.to/topuzas/the-great-ai-agent-showdown-of-2026-openai-autogen-crewai-or-langgraph-1ea8)
- [Open-Source AI Agent Frameworks Compared (OpenAgents)](https://openagents.org/blog/posts/2026-02-23-open-source-ai-agent-frameworks-compared)

### Claude Code Agent Teams e Ruflo
- [Agent Teams (Claude Code Docs)](https://code.claude.com/docs/en/agent-teams)
- [Claude Code Agent Teams Guide (claudefast)](https://claudefa.st/blog/guide/agents/agent-teams)
- [Claude Code Swarm Orchestration Skill (GitHub Gist)](https://gist.github.com/kieranklaassen/4f2aba89594a4aea4ad64d753984b2ea)
- [Claude Code Swarms (Addy Osmani)](https://addyosmani.com/blog/claude-code-agent-teams/)
- [Ruflo GitHub Repository](https://github.com/ruvnet/ruflo)
- [Ruflo 25k Stars (AI for Automation)](https://aiforautomation.io/news/2026-03-25-ruflo-25k-stars-multi-agent-swarms-claude-code)

### Padroes de Orquestracao
- [Architecture of AI Agent Swarms (Codefinity)](https://codefinity.com/blog/The-Architecture-Of-AI-Agent-Swarms)
- [Agent Orchestration Patterns (Gurusup)](https://gurusup.com/blog/agent-orchestration-patterns)
- [Multi-Agent Systems Design Patterns (Tetrate)](https://tetrate.io/learn/ai/multi-agent-systems)
- [Event-Driven Multi-Agent Systems (Confluent)](https://www.confluent.io/blog/event-driven-multi-agent-systems/)

### MetaGPT e ChatDev
- [MetaGPT GitHub Repository](https://github.com/FoundationAgents/MetaGPT)
- [ChatDev GitHub Repository](https://github.com/OpenBMB/ChatDev)
- [What is MetaGPT (IBM)](https://www.ibm.com/think/topics/metagpt)

## Parte 3: AGI

### Definicoes e Frameworks
- [AGI Wikipedia](https://en.wikipedia.org/wiki/Artificial_general_intelligence)
- [Levels of AGI (DeepMind Paper, ArXiv)](https://arxiv.org/pdf/2311.02462)
- [AGI Insider Predictions (Tim Ventura)](https://medium.com/@timventura/agi-insider-predictions-for-the-arrival-of-human-level-artificial-intelligence-40c1084dbcb3)

### Timelines e Previsoes
- [AGI/Singularity: 9,800 Predictions Analyzed (AIMultiple)](https://aimultiple.com/artificial-general-intelligence-singularity-timing)
- [Will We Have AGI by 2030 (80,000 Hours)](https://80000hours.org/ai/guide/when-will-agi-arrive/)
- [Shrinking AGI Timelines (80,000 Hours)](https://80000hours.org/2025/03/when-do-experts-expect-agi-to-arrive/)
- [AGI Timeline Predictions (Nevo Systems)](https://nevo.systems/blogs/nevo-journal/agi-timeline-predictions)
- [DeepMind CEO on AGI (CNBC)](https://www.cnbc.com/2025/03/17/human-level-ai-will-be-here-in-5-to-10-years-deepmind-ceo-says.html)
- [AGI's Last Bottlenecks (AI Frontiers)](https://ai-frontiers.org/articles/agis-last-bottlenecks)

### Safety e Alignment
- [Machines of Loving Grace (Dario Amodei)](https://darioamodei.com/essay/machines-of-loving-grace)
- [Superintelligence 10 Years On (Quillette)](https://quillette.com/2024/07/02/superintelligence-10-years-on-nick-bostrom-ai-safety-agi/)
- [What Does Alignment Mean (Quanta Magazine)](https://www.quantamagazine.org/what-does-it-mean-to-align-ai-with-human-values-20221213/)

---

# Checklist de Completude

- [x] Cobriu todos os angulos do assunto?
  - [x] Parte 1: 7 ferramentas mapeadas com todos os arquivos
  - [x] Parte 2: Fundacoes + frameworks modernos + padroes + implementacao pratica
  - [x] Parte 3: Definicao + historia + desafios + timelines + implicacoes
- [x] Listou referencias historicas/mundiais?
  - [x] SI: Reynolds, Dorigo, Bonabeau, Kennedy, Beni, Wilson
  - [x] AI Frameworks: Chase, Moura, Ng
  - [x] AGI: Turing, McCarthy, Minsky, Bostrom, Russell, Hassabis, Sutskever, Amodei, Altman, LeCun, Hinton
- [x] Listou livros "biblias"?
  - [x] SI: "Swarm Intelligence" (Bonabeau/Dorigo), "Wisdom of Crowds" (Surowiecki)
  - [x] AGI: "Superintelligence" (Bostrom), "Human Compatible" (Russell), "Singularity is Near" (Kurzweil), "Life 3.0" (Tegmark), "Alignment Problem" (Christian), "AI 2041" (Lee/Qiufan)
- [x] Fechou todas as lacunas possiveis?
  - [x] Tabela comparativa master entre ferramentas
  - [x] Comparacao de frameworks multi-agent
  - [x] Timeline de previsoes AGI com fontes
- [x] Citou fontes verificaveis?
  - [x] 80+ fontes com URLs

---

*Pesquisa realizada por Prism (research-orqx) via SINAPSE Research Initiative*
*Nivel: DEFINITIVE | Data: 2026-04-04*
*-- Prism, iluminando o caminho*
