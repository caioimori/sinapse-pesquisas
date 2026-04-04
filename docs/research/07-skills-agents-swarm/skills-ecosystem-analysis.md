# Analise do Ecossistema de Skills & Plugins para Coding Agents

**Data:** 2026-04-04
**Nivel de Profundidade:** DEEP DIVE (nivel 3)
**Pesquisador:** Prism (Research Operations Conductor)
**Squad:** squad-research
**Fontes:** 15+ repositorios, 5 awesome lists, spec oficial agentskills.io, dados de 33+ plataformas

---

## Sumario Executivo

O ecossistema de Agent Skills atingiu massa critica em abril de 2026. A Anthropic publicou o repositorio `anthropics/skills` (110K stars) com 17 skills oficiais e uma especificacao aberta em agentskills.io. Em paralelo, a comunidade ja produziu 1.060+ skills catalogadas, com 33 plataformas/clientes suportando o formato. O projeto `obra/superpowers` (134K stars) emergiu como a referencia comunitaria dominante para metodologia de desenvolvimento com agents. O formato SKILL.md -- pasta com frontmatter YAML + Markdown -- tornou-se o padrao de facto cross-platform, suportado por Claude Code, Codex, Gemini CLI, Cursor, VS Code, GitHub Copilot, e 27+ outros clientes.

**Implicacao para SINAPSE:** O formato de skills esta maduro e padronizado. SINAPSE deve adotar o formato Agent Skills (SKILL.md) como camada de distribuicao publica de suas squads e agents, mantendo a arquitetura proprietaria (YAML + tasks + workflows) como motor interno.

---

## 1. Formato Oficial de Skills (anthropics/skills)

### 1.1 Estrutura de Arquivos

```
skill-name/
  SKILL.md          # OBRIGATORIO: metadados + instrucoes
  scripts/           # Opcional: codigo executavel
  references/        # Opcional: documentacao complementar
  assets/            # Opcional: templates, recursos estaticos
  LICENSE.txt        # Opcional: licenca
```

**Regra fundamental:** O nome da pasta DEVE coincidir com o campo `name` no frontmatter.

### 1.2 Frontmatter YAML (Especificacao agentskills.io)

| Campo | Obrigatorio | Restricoes |
|-------|-------------|------------|
| `name` | Sim | Max 64 chars. Lowercase + hyphens. Sem hyphens consecutivos. |
| `description` | Sim | Max 1024 chars. Descreve O QUE faz E QUANDO usar. |
| `license` | Nao | Nome da licenca ou referencia a arquivo. |
| `compatibility` | Nao | Max 500 chars. Requisitos de ambiente. |
| `metadata` | Nao | Map arbitrario string->string. |
| `allowed-tools` | Nao | Experimental. Lista de tools pre-aprovadas. |

**Exemplo minimo:**
```yaml
---
name: pdf-processing
description: Extract PDF text, fill forms, merge files. Use when handling PDFs.
---
```

**Exemplo completo:**
```yaml
---
name: pdf-processing
description: Extract PDF text, fill forms, merge files. Use when handling PDFs.
license: Apache-2.0
compatibility: Requires Python 3.14+ and uv
metadata:
  author: example-org
  version: "1.0"
allowed-tools: Bash(git:*) Read
---
```

### 1.3 Progressive Disclosure (3 niveis)

Este e o pattern mais importante do formato:

| Nivel | O que carrega | Quando | Tamanho ideal |
|-------|---------------|--------|---------------|
| 1. Metadata | `name` + `description` | Sempre (startup) | ~100 tokens |
| 2. Instructions | Body completo do SKILL.md | Quando a skill e ativada | <5000 tokens (~500 linhas) |
| 3. Resources | Arquivos em scripts/, references/, assets/ | Sob demanda | Ilimitado |

**Regra critica:** Manter SKILL.md abaixo de 500 linhas. Mover material de referencia detalhado para arquivos separados.

### 1.4 Catalogo de Skills Oficiais (Anthropic)

A Anthropic publicou 17 skills organizadas em 3 plugins:

**Plugin: document-skills (source-available, nao open source)**
- `docx` -- Criacao/edicao de Word com tracked changes
- `pdf` -- Manipulacao completa de PDF (extract, merge, fill forms)
- `pptx` -- Criacao/edicao de PowerPoint
- `xlsx` -- Criacao/edicao de Excel com formulas e charts

**Plugin: example-skills (Apache 2.0)**
- `algorithmic-art` -- Arte generativa com p5.js e seeded randomness
- `brand-guidelines` -- Aplicacao da identidade visual Anthropic
- `canvas-design` -- Design visual em PNG/PDF
- `doc-coauthoring` -- Co-autoria de documentos
- `frontend-design` -- Interfaces production-grade sem "AI slop"
- `internal-comms` -- Comunicacoes internas (status reports, newsletters)
- `mcp-builder` -- Guia para criar MCP servers
- `skill-creator` -- Meta-skill para criar novas skills
- `slack-gif-creator` -- GIFs animados otimizados para Slack
- `theme-factory` -- Criacao de temas visuais
- `web-artifacts-builder` -- Artefatos HTML interativos
- `webapp-testing` -- Testes de webapps com Playwright

**Plugin: claude-api**
- `claude-api` -- Documentacao e SDK da API Claude (Python, TS, Java, Go, Ruby, C#, PHP)

### 1.5 Sistema de Plugins (Marketplace)

O arquivo `.claude-plugin/marketplace.json` define como skills sao agrupadas e distribuidas:

```json
{
  "name": "anthropic-agent-skills",
  "plugins": [
    {
      "name": "document-skills",
      "description": "...",
      "source": "./",
      "strict": false,
      "skills": ["./skills/xlsx", "./skills/docx", ...]
    }
  ]
}
```

**Instalacao via Claude Code:**
```bash
/plugin marketplace add anthropics/skills
/plugin install document-skills@anthropic-agent-skills
```

### 1.6 Padroes de Qualidade Observados

Analisando as skills oficiais, os padroes de qualidade sao:

1. **Description "pushy"**: A Anthropic recomenda descriptions "um pouco pushy" para combater a tendencia de under-triggering. Exemplo: incluir keywords especificos que o user pode mencionar.

2. **Instrucoes imperativas**: Uso de forma imperativa (nao declarativa) nas instrucoes.

3. **Exemplos concretos**: Toda skill oficial inclui exemplos de input/output.

4. **Referencia a templates**: Skills complexas referenciam templates em sub-pastas (`templates/`, `references/`).

5. **Scripts como black boxes**: Scripts em `scripts/` devem ser executados com `--help` primeiro, nao lidos no contexto.

6. **Decision trees**: Skills complexas (como `webapp-testing`) incluem arvores de decisao para o agent.

7. **Sem "inventions"**: Skills devem ensinar patterns existentes, nao inventar novos.

---

## 2. Formato Codex (OpenAI)

### 2.1 AGENTS.md vs SKILL.md

O Codex usa dois mecanismos complementares:

| Arquivo | Proposito | Formato |
|---------|-----------|---------|
| `AGENTS.md` | Instrucoes globais do repositorio (equivalente a CLAUDE.md) | Markdown puro |
| `.codex/skills/*/SKILL.md` | Skills especificas com frontmatter | Mesmo formato Agent Skills |

O `AGENTS.md` do Codex e extenso (focado em Rust/codex-rs) e funciona como um CLAUDE.md detalhado com convencoes de codigo, patterns de teste, e workflow.

### 2.2 Skills Codex Nativas

O Codex inclui 3 skills internas:

1. **babysit-pr** -- Monitoramento continuo de PRs (review comments, CI, mergeability). Inclui scripts Python, agentes YAML, e heuristicas de referencia. E a skill mais sofisticada analisada -- 200+ linhas com polling loop, retry logic, e stop conditions.

2. **remote-tests** -- Execucao de testes em ambientes Docker remotos.

3. **test-tui** -- Testes interativos da TUI do Codex.

### 2.3 Formato de Agent YAML (Codex-especifico)

```yaml
interface:
  display_name: "PR Babysitter"
  short_description: "Watch PR review comments, CI, and merge conflicts"
  default_prompt: "Babysit the current PR: monitor reviewer comments..."
```

Este formato YAML e especifico do Codex para definir prompts default de agentes. Nao faz parte do spec Agent Skills.

### 2.4 Convergencia Claude Code / Codex

**FINDING:** Ambas as plataformas convergem para o mesmo formato SKILL.md com frontmatter YAML. O Codex usa `.codex/skills/` enquanto Claude Code usa o sistema de plugins. A especificacao em agentskills.io e aceita por ambos.

**IMPLICATION:** Skills escritas no formato padrao funcionam em ambas as plataformas sem modificacao.

---

## 3. Ecossistema Comunitario

### 3.1 Numeros do Ecossistema

| Metrica | Valor | Fonte |
|---------|-------|-------|
| Skills catalogadas (VoltAgent/awesome-agent-skills) | 1.060+ | GitHub |
| Dev teams contribuindo | 38+ | officialskills.sh |
| Categorias | 11+ | officialskills.sh |
| Skills oficiais de vendors | 307 | officialskills.sh |
| Skills comunitarias | 144+ | officialskills.sh |
| Plataformas que suportam o formato | 33 | agentskills.io/clients |

### 3.2 Repositorios Mais Populares (por stars)

| Repositorio | Stars | Descricao |
|-------------|-------|-----------|
| `obra/superpowers` | 134.347 | Framework completo de desenvolvimento com agents |
| `anthropics/skills` | 110.197 | Skills oficiais Anthropic + spec |
| `anthropics/claude-code` | 108.327 | CLI oficial do Claude Code |
| `ComposioHQ/awesome-claude-skills` | 50.983 | Curadoria de skills |
| `hesreallyhim/awesome-claude-code` | 36.301 | Curadoria geral Claude Code |
| `VoltAgent/awesome-agent-skills` | 14.045 | Catalogo cross-platform |
| `travisvn/awesome-claude-skills` | 10.493 | Curadoria de skills |
| `alirezarezvani/claude-skills` | 9.263 | 220+ skills multi-dominio |
| `trailofbits/skills` | 4.274 | 73 skills de seguranca |
| `expo/skills` | 1.606 | Skills oficiais Expo |

### 3.3 obra/superpowers -- Referencia Comunitaria Dominante

Com 134K stars, `obra/superpowers` e o projeto comunitario mais influente. Nao e apenas uma colecao de skills -- e uma **metodologia de desenvolvimento completa**.

**Skills incluidas (14):**
- `brainstorming` -- Ideacao estruturada
- `dispatching-parallel-agents` -- Orquestracao de agents paralelos
- `executing-plans` -- Execucao de planos de implementacao
- `finishing-a-development-branch` -- Finalizacao de branches
- `receiving-code-review` -- Recebimento de code review
- `requesting-code-review` -- Solicitacao de code review
- `subagent-driven-development` -- Desenvolvimento com sub-agentes
- `systematic-debugging` -- Debug sistematico
- `test-driven-development` -- TDD
- `using-git-worktrees` -- Worktrees para paralelismo
- `verification-before-completion` -- Verificacao pre-entrega
- `writing-plans` -- Escrita de planos de implementacao
- `writing-skills` -- Meta-skill para criar skills
- `using-superpowers` -- Guia de uso do framework

**Metodologia:**
1. Entender o que o user quer (spec)
2. Mostrar spec em chunks digeriveis
3. Criar implementation plan para "junior engineer entusiastico"
4. Subagent-driven development (agent por task)
5. Review em 2 estagios (spec compliance + code quality)

**Compatibilidade cross-platform:**
- Claude Code (marketplace oficial)
- Codex (instrucao de fetch + install)
- Cursor (marketplace)
- OpenCode (instrucao de fetch + install)
- GitHub Copilot CLI
- Gemini CLI

### 3.4 Trail of Bits -- Skills de Seguranca

Com 73 skills, e o maior contribuidor single-domain:
- Vulnerability scanners por blockchain (Solana, Algorand, Cairo, Cosmos, Substrate, TON)
- CodeQL/Semgrep analysis
- Differential review
- Constant-time analysis
- Entry point analyzer
- Firebase APK scanner
- DWARF expert
- BurpSuite parser

### 3.5 Categorias de Skills no Ecossistema

| Categoria | Exemplos | Volume |
|-----------|----------|--------|
| Desenvolvimento Web | React, Next.js, frontend-design, Vercel, Netlify | Alto |
| DevOps/Infra | Terraform, Azure, Docker, CI/CD | Alto (133 so Microsoft) |
| Documentos | DOCX, PDF, PPTX, XLSX | Medio |
| Seguranca | Trail of Bits, auditing, vulnerability scanning | Alto (73+) |
| AI/ML | Hugging Face, fal.ai, model training | Medio |
| Product Management | Dean Peters (46), Pawel Huryn (65) | Alto |
| Marketing | SEO, copywriting, A/B testing, CRO | Medio (35) |
| Design | Canvas, algorithmic art, theme factory | Baixo |
| Data/Analytics | DuckDB, ClickHouse, market analysis | Medio |
| Mobile | Expo, iOS simulator, Firebender (Android) | Baixo |
| Web3/Blockchain | Binance, smart contracts, token auditing | Medio |

### 3.6 Distribuicao de Qualidade

| Tier | Descricao | % Estimado |
|------|-----------|------------|
| **S-tier** | Oficial Anthropic + Trail of Bits + Superpowers | ~5% |
| **A-tier** | Vendors oficiais (Vercel, Netlify, Expo, Microsoft) | ~15% |
| **B-tier** | Comunidade forte (bem documentado, testado) | ~25% |
| **C-tier** | Comunidade basica (funcional mas sem polish) | ~35% |
| **D-tier** | Low-effort / AI-generated bulk | ~20% |

---

## 4. Plataformas que Suportam Agent Skills

### 4.1 Catalogo Completo de Clientes (33 plataformas)

| Plataforma | Tipo | Empresa |
|------------|------|---------|
| Claude Code | Terminal agent | Anthropic |
| Claude.ai | Web agent | Anthropic |
| OpenAI Codex | Cloud agent | OpenAI |
| VS Code | IDE | Microsoft |
| GitHub Copilot | IDE integration | Microsoft/GitHub |
| Cursor | AI IDE | Anysphere |
| Gemini CLI | Terminal agent | Google |
| Junie | IDE agent | JetBrains |
| OpenCode | Terminal agent | SST |
| OpenHands | Cloud agent platform | All Hands AI |
| Roo Code | IDE extension | Roo Code Inc |
| Goose | Terminal agent | Block |
| Amp | Terminal agent | Sourcegraph |
| Letta | Stateful agent platform | Letta |
| Firebender | Android IDE agent | Firebender |
| Factory | AI dev platform | Factory |
| Autohand Code CLI | Terminal agent | Autohand |
| Mux | Parallel agent platform | Coder |
| Piebald | Desktop/web agent | Piebald |
| pi | Minimal terminal agent | badlogic |
| Databricks Genie Code | Data agent | Databricks |
| Agentman | Healthcare agent | Agentman |
| TRAE | AI IDE | ByteDance |
| Spring AI | Framework | VMware/Pivotal |
| Mistral Vibe | Terminal agent | Mistral AI |
| Command Code | AI IDE | Command Code |
| Ona | Cloud agent platform | Ona |
| VT Code | Terminal agent | vinhnx |
| Qodo | Code quality agent | Qodo |
| Laravel Boost | Framework skills | Laravel |
| Emdash | Desktop parallel agent | General Action |
| Snowflake Cortex Code | Data agent | Snowflake |
| Kiro | AI IDE | AWS |

### 4.2 Convergencia Cross-Platform

O formato Agent Skills (SKILL.md) e suportado por TODAS as 33 plataformas listadas. A convergencia e notavel:

- **Terminal agents** (Claude Code, Gemini CLI, Codex, OpenCode, Goose, Amp) -- todos usam SKILL.md
- **IDEs** (Cursor, VS Code, TRAE, Kiro) -- todos suportam SKILL.md
- **Cloud platforms** (OpenHands, Factory, Mux, Ona) -- todos suportam SKILL.md
- **Frameworks** (Spring AI, Laravel Boost) -- integram SKILL.md nativamente

---

## 5. Taxonomia: Skills vs Plugins vs MCP Tools

### 5.1 Definicoes

| Conceito | O que e | Quando usar | Formato |
|----------|---------|-------------|---------|
| **Skill** | Pasta com instrucoes + recursos que ensinam o agent a fazer algo | Quando o agent precisa de conhecimento/workflow especializado | SKILL.md + pasta |
| **Plugin** | Agrupamento de skills para distribuicao | Quando voce quer distribuir N skills como um pacote | marketplace.json |
| **MCP Tool** | Servidor que expoe ferramentas via Model Context Protocol | Quando o agent precisa de acoes programaticas (APIs, DB, etc.) | Servidor MCP |
| **CLAUDE.md / AGENTS.md** | Instrucoes globais do projeto | Convencoes de codigo, rules, workflows do projeto | Markdown |
| **Hook** | Script que intercepta eventos do lifecycle | Pre-commit, pre-tool-use, validacoes automaticas | Script (CJS/Python/Bash) |

### 5.2 Relacao entre Camadas

```
CLAUDE.md/AGENTS.md (project-level instructions)
    |
    +-- Skills (domain expertise, loaded on-demand)
    |       |
    |       +-- scripts/ (executable code)
    |       +-- references/ (documentation)
    |
    +-- MCP Tools (programmatic actions)
    |       |
    |       +-- APIs, databases, external services
    |
    +-- Hooks (lifecycle enforcement)
            |
            +-- Pre-commit, pre-tool-use, validation
```

### 5.3 Quando usar cada um

| Necessidade | Solucao |
|-------------|---------|
| Ensinar o agent a criar PDFs | Skill |
| Dar acesso ao agent ao Supabase | MCP Tool |
| Garantir que ninguem comite secrets | Hook |
| Definir convencoes de codigo do projeto | CLAUDE.md |
| Distribuir um pacote de skills | Plugin marketplace |

---

## 6. Melhores Skills Comunitarias -- Patterns de Sucesso

### 6.1 O que torna uma skill efetiva

Analisando as skills de maior qualidade (Anthropic oficiais, obra/superpowers, Trail of Bits):

1. **Description como trigger**: A description nao e so documentacao -- e o mecanismo primario de ativacao. Skills com descriptions vagas sao sub-utilizadas.

2. **Progressive disclosure disciplinado**: SKILL.md conciso (<500 linhas) com referencias a arquivos detalhados. Nao poluir o contexto.

3. **Decision trees**: Skills complexas incluem fluxogramas de decisao (when to use, when NOT to use).

4. **Scripts executaveis**: Tarefas deterministicas/repetitivas em scripts, nao em instrucoes textuais.

5. **Exemplos concretos**: Input/output esperado documentado.

6. **Self-referencing**: Skills que referenciam outras skills (como superpowers faz) criam um ecossistema coeso.

7. **Cross-platform compatibility**: Skills que funcionam em multiplas plataformas (superpowers suporta 6+).

### 6.2 Anti-patterns observados

1. **SKILL.md gigantes** (1000+ linhas sem references/) -- polui contexto
2. **Descriptions genericas** ("Helps with X") -- nao triggera
3. **Instrucoes declarativas** em vez de imperativas
4. **Sem exemplos** -- agent nao sabe o formato esperado
5. **Dependencias implicitas** -- nao documenta requisitos
6. **Bulk-generated skills** -- conteudo generico sem value real

---

## 7. Implicacoes para SINAPSE

### 7.1 FINDING: O formato SKILL.md e o padrao de facto universal

**IMPLICATION:** SINAPSE pode distribuir conhecimento de squads como skills padrao, alcancando 33+ plataformas sem modificacao.

**RECOMMENDATION:** Criar uma camada de exportacao que converte agents/tasks SINAPSE em SKILL.md padrao para distribuicao publica.

### 7.2 FINDING: Progressive disclosure em 3 niveis e critico

**IMPLICATION:** A arquitetura atual do SINAPSE (agent.md + tasks/ + knowledge-base/) ja segue este pattern naturalmente.

**RECOMMENDATION:** Mapear a estrutura SINAPSE para o formato Agent Skills:
```
Agent.md body compacto    -> SKILL.md (nivel 2)
Tasks detalhadas          -> references/tasks/
Knowledge bases           -> references/kb/
Templates                 -> assets/templates/
```

### 7.3 FINDING: Superpowers domina com 134K stars usando uma metodologia, nao so skills isoladas

**IMPLICATION:** O diferencial de SINAPSE nao e ter skills -- e ter um sistema orquestrado (squads + agents + workflows + constitution). Isso e significativamente mais sofisticado que qualquer coisa no mercado.

**RECOMMENDATION:** Posicionar SINAPSE como "Superpowers on steroids" -- nao apenas skills, mas um sistema orquestrado com governance, quality gates, e delegacao automatica.

### 7.4 FINDING: Nenhum competidor tem sistema de squads + orchestration

**IMPLICATION:** O mercado tem skills isoladas (1 skill = 1 dominio) mas nao tem orquestracao cross-domain. SINAPSE ja resolve isso com squads.

**RECOMMENDATION:** O marketplace SINAPSE deve oferecer:
1. Skills individuais (compativel com todos os 33 clients)
2. Squads completas (diferencial SINAPSE, requer SINAPSE runtime)
3. Workflows cross-squad (nivel mais alto, exclusivo SINAPSE)

### 7.5 FINDING: Skill descriptions "pushy" sao necessarias

**IMPLICATION:** Skills com descriptions conservadoras nao sao ativadas pelo LLM.

**RECOMMENDATION:** Todas as skills SINAPSE exportadas devem ter descriptions explicitamente "pushy" com keywords de ativacao, seguindo a recomendacao oficial da Anthropic.

### 7.6 FINDING: O campo `allowed-tools` e experimental mas estrategico

**IMPLICATION:** Permite que skills definam quais tools podem usar sem pedir permissao. Importante para skills que executam scripts.

**RECOMMENDATION:** Monitorar evolucao deste campo. Quando estabilizar, incluir em todas as skills SINAPSE que usam scripts.

### 7.7 Formato Recomendado para Skills SINAPSE

```yaml
---
name: sinapse-{squad}-{skill}
description: >
  [O que faz]. [Quando usar -- ser pushy].
  Use when: [lista de triggers explicitos].
license: UNLICENSED
compatibility: Requires SINAPSE framework for full features. Works standalone for basic usage.
metadata:
  author: sinapse-ai
  version: "1.0"
  squad: "{squad-name}"
  agent: "{agent-name}"
  sinapse-version: "5.1"
---

# {Skill Title}

[Instrucoes concisas, <500 linhas]

## When to Use
[Decision tree]

## Process
[Steps imperatives]

## Examples
[Input/output concretos]

## References
- [Detail guide](references/guide.md)
- [Templates](assets/templates/)
```

### 7.8 Skills que SINAPSE Deve Exportar por Default

| Squad | Skills Candidatas | Prioridade |
|-------|-------------------|------------|
| squad-research | deep-research, competitive-analysis, market-sizing | Alta |
| squad-development | story-creation, qa-gate, code-review | Alta |
| squad-brand | brand-audit, visual-identity, tone-of-voice | Media |
| squad-content | editorial-calendar, content-strategy | Media |
| squad-copy | headline-writing, persuasion-framework | Media |
| squad-growth | seo-audit, analytics-setup | Media |
| squad-commercial | battle-card, sales-playbook | Baixa |

### 7.9 Distribuicao Recomendada

1. **GitHub marketplace:** `sinapse-ai/skills` com marketplace.json
2. **Plugin packs:** Um plugin por squad (ex: `research-skills@sinapse-marketplace`)
3. **officialskills.sh:** Registrar skills SINAPSE no diretorio oficial
4. **Cross-platform:** Testar em Claude Code, Codex, Gemini CLI, Cursor

---

## 8. Analise Comparativa: SINAPSE vs Ecossistema

| Dimensao | SINAPSE | Superpowers | Anthropic Skills | Trail of Bits |
|----------|---------|-------------|------------------|---------------|
| **Escopo** | Multi-squad orchestration | Dev methodology | Skills isoladas | Security domain |
| **Orquestracao** | Constitution + agents + workflows | Plan -> subagent -> review | Nenhuma | Nenhuma |
| **Quality gates** | Automaticos (hooks + qa-gate) | 2-stage review | Manual | Nenhum |
| **Cross-domain** | Sim (20+ squads) | Nao (so dev) | Nao | Nao |
| **Formato** | YAML + MD proprietario | SKILL.md padrao | SKILL.md padrao | SKILL.md padrao |
| **Distribuicao** | Local (ainda) | Marketplace 6+ platforms | Marketplace Claude | Marketplace Claude |
| **Governance** | Constitution formal | Nenhuma | Nenhuma | Nenhuma |
| **Memory/Context** | Agent memory + scratchpad | Nenhum | Nenhum | Nenhum |

**Conclusao:** SINAPSE e significativamente mais sofisticado que qualquer player no ecossistema. O gap esta na distribuicao -- outros ja estao em 33 plataformas enquanto SINAPSE e local.

---

## 9. Recomendacoes Estrategicas

### Curto Prazo (1-2 semanas)
1. Criar repositorio `sinapse-ai/skills` com formato Agent Skills padrao
2. Exportar 3-5 skills das squads mais maduras (research, development)
3. Registrar marketplace no Claude Code

### Medio Prazo (1-2 meses)
4. Criar layer de exportacao automatica (agent SINAPSE -> SKILL.md)
5. Publicar no officialskills.sh
6. Testar cross-platform (Codex, Gemini CLI, Cursor)
7. Criar meta-skill `sinapse-methodology` (equivalente ao superpowers)

### Longo Prazo (3-6 meses)
8. Plugin marketplace completo com todas as squads
9. Skills com `allowed-tools` quando o campo estabilizar
10. Sistema de eval automatico (usando pattern do skill-creator da Anthropic)
11. Contribuir para agentskills.io spec com propostas de orquestracao

---

## 10. Referencias

### Repositorios Analisados
- `anthropics/skills` -- 110K stars -- https://github.com/anthropics/skills
- `anthropics/claude-code` -- 108K stars -- https://github.com/anthropics/claude-code
- `openai/codex` -- https://github.com/openai/codex
- `obra/superpowers` -- 134K stars -- https://github.com/obra/superpowers
- `trailofbits/skills` -- 4.2K stars -- https://github.com/trailofbits/skills
- `expo/skills` -- 1.6K stars -- https://github.com/expo/skills
- `alirezarezvani/claude-skills` -- 9.2K stars -- https://github.com/alirezarezvani/claude-skills

### Awesome Lists Consultadas
- `VoltAgent/awesome-agent-skills` -- 14K stars -- https://github.com/VoltAgent/awesome-agent-skills
- `ComposioHQ/awesome-claude-skills` -- 51K stars -- https://github.com/ComposioHQ/awesome-claude-skills
- `hesreallyhim/awesome-claude-code` -- 36K stars -- https://github.com/hesreallyhim/awesome-claude-code
- `travisvn/awesome-claude-skills` -- 10K stars -- https://github.com/travisvn/awesome-claude-skills

### Especificacoes e Documentacao
- Agent Skills Spec: https://agentskills.io/specification
- Agent Skills Client Showcase: https://agentskills.io/clients
- Official Skills Directory: https://officialskills.sh
- Claude Skills Support: https://support.claude.com/en/articles/12512176-what-are-skills
- Anthropic Blog Post: https://anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills

### Referencia Metodologica
- Porter, M. (1980). Competitive Strategy. Free Press. -- Framework de analise competitiva aplicado ao mapeamento do ecossistema.
- Christensen, C. (1997). The Innovator's Dilemma. HBS Press. -- Analise de como o formato padrao disrupta solucoes proprietarias.
- Moore, G. (1991). Crossing the Chasm. HarperBusiness. -- O ecossistema de skills esta no "early majority" do ciclo de adocao.

---

*Pesquisa conduzida por Prism (Research Operations Conductor) -- squad-research*
*Nivel: DEEP DIVE | Fontes: 15+ repos, 5 awesome lists, 33 plataformas | Tier 3-4 (Industry + Market)*
