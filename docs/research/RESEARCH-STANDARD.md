# SINAPSE Research Standard — Padrão de Qualidade Máxima

> **Este documento define o padrão obrigatório para TODA pesquisa realizada neste repositório.**
> Quando o usuário pede "faça pesquisa de X", este padrão é aplicado AUTOMATICAMENTE.

---

## 1. Estrutura Obrigatória de Toda Pesquisa

Toda pesquisa DEVE conter estas seções:

```markdown
# {Título da Pesquisa}

> **Data:** YYYY-MM-DD
> **Autor:** {Agent(s)} via SINAPSE Research Initiative
> **Fontes:** {Número} fontes consultadas
> **Objetivo:** {Uma frase}

---

## Índice
(Todas as seções numeradas)

## 1. Panorama Geral
(Visão 360° do assunto)

## 2-N. Seções Temáticas
(Cobertura profunda de cada ângulo)

## N+1. Referências Históricas & Mundiais
### Pessoas
(As maiores referências de cada sub-área)
### Livros "Bíblias"
(Os livros definitivos — a leitura obrigatória)
### Papers & Artigos Seminais
(Artigos que mudaram o campo)

## N+2. Fontes & Links
(Todas as fontes com URLs verificáveis)

## N+3. Checklist de Completude
- [ ] Cobriu todos os ângulos do assunto?
- [ ] Listou referências históricas/mundiais?
- [ ] Listou livros "bíblias"?
- [ ] Fechou todas as lacunas possíveis?
- [ ] Citou fontes verificáveis?
```

---

## 2. Requisitos de Profundidade

### Pessoas Referência (OBRIGATÓRIO)
Para CADA sub-área do assunto, listar:
- **Nome completo** da pessoa
- **Por que é referência** (1 frase)
- **Contribuição principal** (livro, framework, empresa, descoberta)

Exemplo:
> **Robert C. Martin (Uncle Bob)** — Pai do Clean Code e dos princípios SOLID. Autor de "Clean Code" (2008) e "Clean Architecture" (2017). Transformou a forma como a indústria pensa sobre qualidade de código.

### Livros "Bíblias" (OBRIGATÓRIO)
Para CADA sub-área, listar os livros definitivos:
- **Título** + **Autor** + **Ano**
- **Por que é "bíblia"** (1 frase)
- **O que extrair** (conceitos-chave para o SINAPSE)

Exemplo:
> **"Design Patterns: Elements of Reusable Object-Oriented Software"** — Erich Gamma, Richard Helm, Ralph Johnson, John Vlissides (GoF, 1994). A obra que definiu padrões de design para toda a indústria de software. Extrair: Strategy, Factory, Observer, Decorator para uso em agent definitions.

### Cobertura 360° (OBRIGATÓRIO)
Toda pesquisa DEVE cobrir:
- **O quê** — Definição e conceitos fundamentais
- **Por quê** — Por que isso importa para o SINAPSE
- **Como** — Implementação prática, patterns, exemplos
- **Quem** — Referências históricas e mundiais
- **Onde** — Fontes, livros, papers, repos
- **Quando** — Timeline histórica, evolução do campo
- **Riscos** — O que pode dar errado, armadilhas
- **Tendências** — Para onde o campo está indo

---

## 3. Níveis de Pesquisa

| Nível | Quando usar | Profundidade |
|-------|-------------|-------------|
| **Quick** | Pergunta pontual | 1-2 páginas, 5-10 fontes |
| **Standard** | Sub-tema específico | 5-15 páginas, 15-30 fontes |
| **Deep** | Área completa | 20-50 páginas, 30-60 fontes |
| **Definitive** | Fonte da verdade | 50+ páginas, 60+ fontes, referências completas |

**Padrão default: Definitive** (a menos que especificado)

---

## 4. Organização de Arquivos

```
docs/research/
├── INDEX.md                          ← Master index (sempre atualizado)
├── RESEARCH-STANDARD.md              ← Este arquivo
├── YYYY-MM-DD-{slug}/
│   └── README.md                     ← Pesquisa principal
├── {slug-descritivo}.md              ← Pesquisas standalone
└── ...
```

Naming: `YYYY-MM-DD-{tema-em-kebab-case}/README.md`

---

## 5. Processo de Revisão

Antes de considerar uma pesquisa "completa":
1. **Verificar** — Todas as fontes são acessíveis?
2. **Completar** — Há lacunas não cobertas?
3. **Referenciar** — Pessoas e livros estão listados?
4. **Indexar** — INDEX.md foi atualizado?
5. **Commitar** — Pesquisa foi salva no Git?

---

## 6. Propósito das Pesquisas

Estas pesquisas servem como **fonte da verdade** para:
- Aprimorar o SINAPSE-AI framework
- Criar novas squads, agents, subagents, workers
- Criar clones cognitivos baseados em especialistas
- Gerar knowledge bases para agents especializados
- Treinar a equipe (Caio e Matheus) nos fundamentos
- Tomar decisões arquiteturais e de produto

**Qualidade > velocidade.** Melhor uma pesquisa completa do que três superficiais.

---

*SINAPSE Research Standard v1.0 — 2026-04-04*
