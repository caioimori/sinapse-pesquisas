# TOKEN ECONOMY: Os 10 Mandamentos (6W Framework)

> **Fonte:** Documento compartilhado pelo usuário (2026-04-05)
> **Dados:** 16 fontes primárias, 1,020 sessões analisadas, 48.6M tokens mapeados
> **Resultado:** 57.4K → 7.7K tokens/sessão = 86.6% redução

---

## A Fórmula do Desperdício

```
overhead x turns/sessão x sessões = DESPERDÍCIO TOTAL

57.4K x 30 avg x 2,000 = 3.4 BILHÕES tok/mês
 7.7K x 30 avg x 2,000 = 462M tok/mês

ECONOMIA: 86.6% = ~$44,370/mês (Opus @ $15/1M input)
```

## 11 Fases de Ingestão (EAGER vs LAZY)

| Fase | O que | Estratégia | Tokens | Controlável? |
|------|-------|------------|--------|-------------|
| F0 | Managed Policy | EAGER (fixo) | ~500 | NÃO |
| F1 | User Settings | EAGER (fixo) | ~200 | NÃO |
| F2 | Project Settings | EAGER (fixo) | ~300 | NÃO |
| F3 | CLAUDE.md | EAGER (fixo) | ~3,000 | SIM (slim) |
| F4 | Rules sem globs | EAGER | ~14,000 | SIM (add globs) |
| F4 | Rules com globs | LAZY (JIT) | 0 | -- |
| F5 | Auto-memory | EAGER (fixo) | ~2,900 | SIM (max 200L) |
| F6 | Skills discovery | EAGER (meta) | ~2,000 | SIM (metadata) |
| F6 | Skills body | LAZY (invoke) | 0 | -- |
| F7 | Commands discovery | EAGER (desc) | ~22,900 | SIM (delete) |
| F8 | Agents discovery | EAGER (desc) | ~14,200 | SIM (delete) |
| F9 | MCP tool schemas | EAGER | ~5,000 | SIM (disable) |
| F10 | Git status | EAGER (fixo) | ~800 | NÃO |
| F11 | System prompt assembly | EAGER (fixo) | ~1,500 | NÃO |
| | **TOTAL sem otimizar** | | **~57,400** | |
| | **TOTAL otimizado** | | **~7,700** | |

**Zona de Custo Fixo (~8.7K):** Fases 0-5. Aceite. Otimize CLAUDE.md e memory.
**Zona de Custo Variável (0-50K+):** Fases 6-9. **AQUI ESTÁ O OURO. 85% da economia.**

## Os 10 Mandamentos

### I. CONTEXT = CACHE GERENCIADO
Cada token no system prompt é cobrado N vezes (N = turns). 47.6K overhead × 30 turns = 1.4M tokens desperdiçados/sessão. "Lost in the Middle": 50K lixo é PIOR que 8K limpo.

### II. CONHECE AS 11 FASES
Fases 0-5: custo fixo baixo (~8.7K). Não mexer. Fases 6-9: custo variável (0-50K+). FOCO AQUI.

### III. PROGRESSIVE DISCLOSURE EM 3 CAMADAS
- Layer 1 (EAGER): metadata ~100 tok/skill
- Layer 2 (LAZY): body ~2,000 tok/skill — só quando invocado
- Layer 3 (LAZY): references — só quando Claude decide ler
- 20 skills: 40K eager → 2K metadata + 4-6K invocados = 85-90% economia

### IV. FILTRA SCHEMAS DINAMICAMENTE
Lazy-Loaded Tools: 427K tokens/dia economizados por instância. Desabilitar MCPs não usados.

### V. PROMPT CACHING
System prompt cacheado após turn 1. Sessões longas > muitas curtas. Hooks dinâmicos podem QUEBRAR o cache prefix.

### VI. COMPACTAR PROATIVAMENTE
Auto-compaction dispara a 83.5% do context window. LOSSY. A 50%: backup estruturado. A 70%: /compact controlado. NUNCA chegar em 83.5%.

### VII. ISOLAR VERBOSO EM SUBAGENTS
SEM subagent: 50 files × 2K = 100K tokens no contexto. COM subagent: Agent(Explore) = 1.5K tokens no parent. ECONOMIA: 98.5%. Regra: output esperado > 5K tokens? Delegue.

### VIII. CADA ARQUIVO = TOKEN TAX
- 762 commands × ~30 avg = 22,860 tok/sessão
- 254 agents × ~56 avg = 14,224 tok/sessão
- 15 rules × ~930 avg = 13,950 tok/sessão
- Audit loop 14 dias. Delete ratio > 80% = saudável.

### IX. INSTRUÇÃO ≠ ENFORCEMENT
Rule de 2,000 tok que INSTRUI → validator de 20 linhas: 0 tok. Enforcement → pre-commit hooks (0 tokens). 14K tokens de rules globais → 0-2K JIT + validators.

### X. MEDIRÁS, OU NÃO GERENCIARÁS
- Overhead controlável: TARGET < 8K tokens
- Commands existentes: TARGET < 80
- Agents existentes: TARGET < 30
- Usage ratio: TARGET > 50%
- Ciclo: Dia 1 audit → identificar 0-uso → deletar → Dia 14 repetir

## Targets SINAPSE (Aplicar)

| Métrica | Atual SINAPSE | Target |
|---------|--------------|--------|
| Commands | 7 + ~20 squads | < 80 |
| Agents registrados | ~20 orqx | < 30 |
| Rules sem globs | 18/19 | 0 |
| Overhead estimado | ~30K+ | < 8K |
| CLAUDE.md | ~20K chars | < 2K chars (~500 tok) |
| Memory | sem limite | max 200 linhas |

## Audit Prompt (Usar a cada 14 dias)

Analisar sessões em `~/.claude/projects/` extraindo: sessões interativas vs headless, slash commands invocados, skill invocations, agent subagent_type, shell runners, commits por skill/feature. Gerar relatório com top skills, top agents, commands/agents/skills NUNCA usados, recomendação de cleanup com economia estimada.
