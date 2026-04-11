# Cloudflare Ecosystem e Edge Computing Patterns para Aplicacoes Web Modernas

> **Nivel de Profundidade:** DEEP DIVE (Nivel 3)
> **Fontes:** 40+ fontes verificadas via WebSearch, Tiers 2-4
> **Data da Pesquisa:** Abril 2026
> **Pesquisador:** Prism (research-orqx) via WebSearch

---

## Indice

1. [Cloudflare Workers — Computacao no Edge](#1-cloudflare-workers--computacao-no-edge)
2. [Cloudflare R2 — Object Storage](#2-cloudflare-r2--object-storage)
3. [Cloudflare D1 — Database no Edge](#3-cloudflare-d1--database-no-edge)
4. [Cloudflare WAF e Seguranca](#4-cloudflare-waf-e-seguranca)
5. [Cloudflare CDN e Performance](#5-cloudflare-cdn-e-performance)
6. [Stack Cloudflare + Vercel + Supabase](#6-stack-cloudflare--vercel--supabase)
7. [Cloudflare Pages](#7-cloudflare-pages)
8. [Edge Computing Decision Framework](#8-edge-computing-decision-framework)
9. [Fontes e Referencias](#9-fontes-e-referencias)

---

## 1. Cloudflare Workers — Computacao no Edge

### 1.1 Arquitetura e Modelo de V8 Isolates

Cloudflare Workers utiliza o V8 engine — o mesmo motor usado pelo Chromium e Node.js — para orquestrar **isolates**: contextos leves que fornecem ao codigo variaveis acessiveis e um ambiente seguro de execucao. Uma unica instancia do runtime pode executar centenas ou milhares de isolates, alternando entre eles de forma transparente.

**Diferenca fundamental em relacao a containers:**

| Caracteristica | V8 Isolates (Workers) | Containers (Lambda) | VMs Tradicionais |
|---|---|---|---|
| Tempo de startup | < 1ms | 100-1000ms | Segundos a minutos |
| Memoria por instancia | ~128 MB | 128 MB - 10 GB | GB+ |
| Overhead de isolamento | Minimo (V8 sandbox) | Medio (cgroups/namespaces) | Alto (hypervisor) |
| Densidade por servidor | Milhares de isolates | Dezenas de containers | Unidades de VMs |
| Cold start | Praticamente zero | Significativo | Muito alto |

Fonte: [How Workers works - Cloudflare Docs](https://developers.cloudflare.com/workers/reference/how-workers-works/)

**Evolucao em 2026 — Dynamic Workers:** A Cloudflare lancou o Dynamic Worker Loader em open beta, oferecendo sandboxing baseado em V8 isolates para execucao de codigo gerado por IA. Os isolates iniciam em poucos milissegundos e usam poucos megabytes de memoria, sendo aproximadamente **100x mais rapidos para inicializar** e **10-100x mais eficientes em memoria** que containers tipicos.

Fonte: [Cloudflare Dynamic Workers Open Beta - InfoQ](https://www.infoq.com/news/2026/04/cloudflare-dynamic-workers-beta/)

### 1.2 Caracteristicas de Performance

#### Cold Start
Workers elimina cold starts por design. Enquanto AWS Lambda pode levar 100-1000ms para cold start, Workers inicia em sub-1ms. Isso ocorre porque isolates nao precisam carregar um sistema operacional ou inicializar um runtime completo.

Fonte: [Eliminating cold starts with Cloudflare Workers](https://blog.cloudflare.com/eliminating-cold-starts-with-cloudflare-workers/)

#### CPU Time Limits

| Plano | CPU Time por Request | CPU Time Maximo (opt-in) |
|---|---|---|
| Free | 10ms | 10ms |
| Paid (padrao) | 30 segundos | 30 segundos |
| Paid (configurado) | Ate 5 minutos (300,000ms) | Via `cpu_ms` no wrangler.toml |

**Nota importante:** CPU time e o tempo que a CPU efetivamente gasta fazendo trabalho. Se um Worker faz um sub-request e aguarda resposta, esse tempo de espera **nao conta** como CPU time.

Fonte: [Workers Limits - Cloudflare Docs](https://developers.cloudflare.com/workers/platform/limits/), [Higher CPU limits changelog](https://developers.cloudflare.com/changelog/post/2025-03-25-higher-cpu-limits/)

#### Memoria
Cada isolate pode consumir ate **128 MB** de memoria, incluindo o JavaScript heap e alocacoes WebAssembly. Quando um isolate excede 128 MB, o runtime permite que requests em andamento completem e cria um novo isolate para requests subsequentes.

Fonte: [Workers Limits - Cloudflare Docs](https://developers.cloudflare.com/workers/platform/limits/)

### 1.3 Workers vs Lambda vs Vercel Edge Functions — Comparacao Detalhada

| Dimensao | Cloudflare Workers | AWS Lambda | Vercel Edge Functions |
|---|---|---|---|
| **Runtime** | V8 Isolates | Containers | V8 Isolates (via Workers) |
| **Localizacoes** | 300+ PoPs globais | Regioes AWS especificas | ~30 regioes (Vercel Edge Network) |
| **Cold start** | < 1ms | 100-1000ms | ~50ms |
| **CPU time** | 10ms (free) / 30s (paid) / 5min (opt-in) | 15 minutos | 30 segundos |
| **Memoria** | 128 MB | Ate 10 GB | 128 MB |
| **Linguagens** | JS/TS, Python, Rust (WASM) | Node.js, Python, Java, Go, .NET, etc | JS/TS |
| **Latencia P50 global** | 10-30ms | 50-200ms | 10-30ms (perto da regiao), 50-150ms (global free/pro) |
| **Preco base** | $5/mes (10M requests) | Pay-per-use ($0.20/1M requests + compute) | Incluso no plano Vercel |
| **Preco 10M requests/mes** | ~$5 | ~$17 (Lambda@Edge) | Incluso no Pro ($20/mes) |
| **Melhor para** | APIs, middleware, latencia global | Processamento pesado, ecossistema AWS | Apps Next.js, server components |

**Performance relativa:** Workers roda **210% mais rapido** que AWS Lambda@Edge e **298% mais rapido** que Lambda standard.

**Mudanca de pricing da AWS (Agosto 2025):** A AWS comecou a cobrar pela fase INIT do Lambda, o que pode aumentar o gasto com Lambda em **10-50%** para funcoes com logica pesada de startup.

Fontes: [Lambda@Edge vs Workers vs Vercel Edge](https://prabhatgiri.com/blogs/lambdaedge-vs-cloudflare-workers-vs-vercel-edge-latency-limits-and-cost-in-2025/), [Cloudflare vs Vercel vs Netlify Edge Performance 2026](https://dev.to/dataformathub/cloudflare-vs-vercel-vs-netlify-the-truth-about-edge-performance-2026-50h0)

### 1.4 Cron Triggers para Tarefas Agendadas

Cron Triggers permitem executar Workers em horarios programados usando expressoes cron com 5 campos. Quando invocado por um Cron Trigger, o runtime inicia um `ScheduledEvent` tratado pela funcao `scheduled`.

```javascript
// wrangler.toml
// [triggers]
// crons = ["*/5 * * * *", "0 0 * * *"]

export default {
  async scheduled(event, env, ctx) {
    // event.cron contem a expressao cron que disparou
    // event.scheduledTime contem o timestamp agendado
    switch (event.cron) {
      case "*/5 * * * *":
        await checkHealthEndpoints(env);
        break;
      case "0 0 * * *":
        await generateDailyReport(env);
        break;
    }
  },
};
```

**Caracteristicas importantes:**
- Executam em UTC
- Podem ser combinados com Workflows para tarefas multi-step de longa duracao
- Armazenam as 100 invocacoes mais recentes para debugging
- Testaveis localmente com `wrangler dev --test-scheduled`
- Sem custo adicional — disponivel tanto no free tier quanto no paid
- Suportam Python Workers desde abril 2025

Fonte: [Cron Triggers - Cloudflare Docs](https://developers.cloudflare.com/workers/configuration/cron-triggers/)

### 1.5 Durable Objects para Computacao Stateful no Edge

Durable Objects sao um tipo especial de Cloudflare Worker que combina **compute com storage**. Cada Durable Object tem um nome globalmente unico, permitindo enviar requests de qualquer lugar do mundo para um objeto especifico.

**Casos de uso primarios:**

| Caso de Uso | Descricao | Exemplo |
|---|---|---|
| Real-time collaboration | Coordenacao entre multiplos clientes | Google Docs-style co-editing, whiteboards compartilhados |
| Chat e gaming | WebSockets + estado compartilhado | Salas de chat, jogos multiplayer |
| IoT | Controle de dispositivos em tempo real | Dashboards ao vivo, controle remoto |
| Event orchestration | Alarms + estado em memoria + storage duravel | Filas, workflows, pipelines de dados |
| Counters e rate limiting | Estado consistente globalmente | Contadores de API, rate limiting por usuario |

**Modelo de consistencia:** Durable Objects garantem **strong consistency** — todas as operacoes no mesmo objeto sao serializadas, eliminando race conditions.

```javascript
export class ChatRoom {
  constructor(state, env) {
    this.state = state;
    this.sessions = [];
  }

  async fetch(request) {
    const pair = new WebSocketPair();
    const [client, server] = Object.values(pair);
    this.state.acceptWebSocket(server);
    this.sessions.push(server);

    return new Response(null, {
      status: 101,
      webSocket: client,
    });
  }

  async webSocketMessage(ws, message) {
    // Broadcast para todos os participantes
    for (const session of this.sessions) {
      if (session !== ws) {
        session.send(message);
      }
    }
  }
}
```

Fonte: [Durable Objects - Cloudflare Docs](https://developers.cloudflare.com/durable-objects/), [Introducing Durable Objects](https://blog.cloudflare.com/introducing-workers-durable-objects/)

### 1.6 Workers KV vs D1 vs R2 — Comparacao de Storage

| Dimensao | Workers KV | D1 | R2 |
|---|---|---|---|
| **Tipo** | Key-Value distribuido | SQLite relacional | Object Storage (S3-compatible) |
| **Consistencia** | Eventually consistent (~60s) | Strong consistency (primary) | Strong consistency |
| **Melhor para** | Config, sessoes, cache | Dados relacionais, queries SQL | Arquivos, midias, blobs |
| **Tamanho max do valor** | 25 MB | 10 GB por database | 5 TB por objeto |
| **Write throughput** | 1 write/s por key | 500-2K writes/s | Alta (paralelo) |
| **Read throughput** | Milhares/s por key | Alta (read replicas) | Alta (CDN integrado) |
| **Free tier storage** | 1 GB | 5 GB | 10 GB |
| **Free tier reads** | 100K/dia | 25M rows/mes | 10M/mes |
| **Free tier writes** | 1K/dia | 50K rows/mes | 1M/mes |
| **Queries** | Get/Put/List/Delete | SQL completo (SQLite) | S3 API |
| **Localizacao** | 300+ PoPs (replicated) | Primary + read replicas | Distribuido |

**Recomendacao de uso combinado:** Uma aplicacao pode usar KV para dados de sessao, R2 para arquivos e midias, e D1 (ou Hyperdrive com Postgres externo) para dados relacionais.

Fonte: [Choosing a storage product - Cloudflare Docs](https://developers.cloudflare.com/workers/platform/storage-options/), [Cloudflare KV, R2, D1 Tutorial](https://blog.birdor.com/cloudflare-tutorial-part-6-kv-r2-d1-storage/)

### 1.7 Service Bindings para Microservicos no Edge

Service bindings permitem que um Worker chame outro Worker diretamente, sem overhead de rede. Ambos os Workers rodam na **mesma thread do mesmo servidor Cloudflare** por padrao.

**Beneficios:**
- **Zero overhead de latencia** entre Workers
- **Sem custo adicional** — dividir funcionalidade em multiplos Workers nao gera custos extras
- **Dois modos de comunicacao:** RPC (recomendado) e HTTP

```javascript
// Worker A — chama Worker B via RPC
export default {
  async fetch(request, env) {
    // env.AUTH_SERVICE e o service binding
    const user = await env.AUTH_SERVICE.validateToken(token);
    const data = await env.DATA_SERVICE.getUserData(user.id);
    return Response.json(data);
  },
};
```

```toml
# wrangler.toml do Worker A
[[services]]
binding = "AUTH_SERVICE"
service = "auth-worker"
entrypoint = "AuthService"

[[services]]
binding = "DATA_SERVICE"
service = "data-worker"
entrypoint = "DataService"
```

Fonte: [Service Bindings - Cloudflare Docs](https://developers.cloudflare.com/workers/runtime-apis/bindings/service-bindings/), [Introducing Worker Services](https://blog.cloudflare.com/introducing-worker-services/)

### 1.8 Empresas Usando Workers em Producao

#### Shopify
Shopify utiliza Cloudflare Workers para sua plataforma **Hydrogen/Oxygen** — o framework eCommerce baseado em Remix. Workers alimentam:
- Storefronts personalizados de merchants
- Website de marketing global com escalabilidade instantanea
- Deploy de campanhas globais sem delays de startup

Fonte: [Shopify Case Study - Cloudflare](https://www.cloudflare.com/case-studies/shopify/)

#### Discord
Discord economiza centenas de milhares em custos de hardware e bandwidth usando Cloudflare, enquanto oferece maior performance e seguranca aos usuarios.

Fonte: [Discord Case Study - Cloudflare](https://www.cloudflare.com/case-studies/discord/)

#### Workers AI (Adocao em 2026)
Workers AI teve um aumento de **4.000% year-over-year** em inference requests no Q1 2026, indicando adocao massiva do ecossistema.

Fonte: [Cloudflare Workers V8 Isolates AI Agents](https://www.kunalganglani.com/blog/cloudflare-workers-v8-isolates-ai-agents)

### 1.9 Pricing Detalhado

| Recurso | Free Tier | Paid ($5/mes) |
|---|---|---|
| Requests | 100,000/dia | 10 milhoes/mes inclusos |
| CPU time | 10ms/invocacao | 30s padrao, ate 5min (opt-in) |
| Requests extras | N/A | $0.30 por milhao extra |
| CPU time extra | N/A | $0.02 por milhao de CPU-ms |
| Workers KV reads | 100,000/dia | 10 milhoes/mes |
| Workers KV writes | 1,000/dia | 1 milhao/mes |
| Durable Objects | Nao incluso | Incluso (storage billing desde Jan 2026) |
| Cron Triggers | Incluso | Incluso |

Fonte: [Workers Pricing - Cloudflare Docs](https://developers.cloudflare.com/workers/platform/pricing/)

---

## 2. Cloudflare R2 — Object Storage

### 2.1 Zero Egress Fees — A Vantagem Competitiva

A proposta de valor central do R2 e **zero egress fees**. Egress diretamente do R2 — via Workers API, S3 API ou dominios r2.dev — nao incorre em taxas de transferencia de dados.

**Comparacao de custos para 10 TB armazenados + 50 TB transferencia mensal:**

| Provedor | Storage | Egress | Total Mensal |
|---|---|---|---|
| **Cloudflare R2** | $150 | $0 | **$150** |
| **AWS S3** | $230 | $4,500 | **$4,730** |
| **Google Cloud Storage** | $200 | $5,000 | **~$5,200** |
| **Azure Blob** | $200 | $4,350 | **~$4,550** |

**Economia de 97% em relacao ao S3 para workloads com alta transferencia.**

Fonte: [R2 Pricing - Cloudflare Docs](https://developers.cloudflare.com/r2/pricing/), [R2 vs S3 Deep Dive](https://yconsulting.substack.com/p/cloudflare-r2-vs-the-big-3-a-deep)

### 2.2 R2 vs S3 vs Vercel Blob vs Supabase Storage

| Dimensao | Cloudflare R2 | AWS S3 | Vercel Blob | Supabase Storage |
|---|---|---|---|---|
| **Egress** | Zero | $0.09/GB | Incluso no plano | Incluso no plano |
| **S3-compatible** | Sim | Nativo | Nao | Nao |
| **Free tier storage** | 10 GB | Nenhum (12 meses trial) | 1 GB | 1 GB |
| **CDN integrado** | Sim (Cloudflare CDN) | CloudFront (custo extra) | Sim | Sim |
| **Auth integrada** | Nao | IAM | Nao | Sim (RLS) |
| **Realtime events** | Nao | S3 Event Notifications | Nao | Sim (Realtime) |
| **Melhor para** | Alto volume de egress, midia | Ecossistema AWS completo | Apps Vercel simples | Backend integrado |
| **Storage classes** | Standard + Infrequent Access | 7+ classes | Unica | Unica |

**Recomendacao hibrida:** Usar R2 para arquivos publicos com alto trafego (zero egress) e Supabase Storage para arquivos privados que precisam de RLS e permissoes por usuario.

Fonte: [Supabase vs R2 2026](https://www.buildmvpfast.com/compare/supabase-vs-r2), [R2 vs S3 Comparison](https://www.pump.co/blog/cloudflare-vs-s3)

### 2.3 Integracao com Next.js — Presigned URLs

O padrao recomendado para uploads de arquivo em Next.js com R2 usa presigned URLs, onde o arquivo nunca toca o servidor — o servidor apenas assina o request.

```typescript
// app/api/upload/route.ts — API Route para gerar presigned URL
import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";

const S3 = new S3Client({
  region: "auto",
  endpoint: `https://${process.env.CF_ACCOUNT_ID}.r2.cloudflarestorage.com`,
  credentials: {
    accessKeyId: process.env.R2_ACCESS_KEY_ID!,
    secretAccessKey: process.env.R2_SECRET_ACCESS_KEY!,
  },
});

export async function POST(request: Request) {
  const { filename, contentType } = await request.json();

  const command = new PutObjectCommand({
    Bucket: process.env.R2_BUCKET_NAME,
    Key: `uploads/${Date.now()}-${filename}`,
    ContentType: contentType,
  });

  const signedUrl = await getSignedUrl(S3, command, {
    expiresIn: 3600, // 1 hora
  });

  return Response.json({ uploadUrl: signedUrl });
}
```

```typescript
// Componente React — Upload via presigned URL
async function uploadFile(file: File) {
  // 1. Obter presigned URL do servidor
  const { uploadUrl } = await fetch("/api/upload", {
    method: "POST",
    body: JSON.stringify({
      filename: file.name,
      contentType: file.type,
    }),
  }).then((r) => r.json());

  // 2. Upload direto para R2 (sem tocar o servidor)
  await fetch(uploadUrl, {
    method: "PUT",
    body: file,
    headers: { "Content-Type": file.type },
  });
}
```

**Multipart uploads:** Para arquivos acima de 5 MB, usar `@aws-sdk/lib-storage` que gerencia multipart automaticamente. Para arquivos acima de 100 MB em redes instaveis, multipart com resume e recomendado com chunks de 5-64 MB.

Fonte: [Upload Files to R2 in Next.js](https://www.buildwithmatija.com/blog/how-to-upload-files-to-cloudflare-r2-nextjs), [Presigned URLs - R2 Docs](https://developers.cloudflare.com/r2/api/s3/presigned-urls/)

### 2.4 Estrategias de CDN Caching com R2

R2 integra-se nativamente com o CDN da Cloudflare. Estrategias recomendadas:

1. **Custom domain com cache:** Configurar um dominio customizado apontando para o bucket R2 permite controle total de cache headers via Cache Rules
2. **Cache-Control headers:** Definir `Cache-Control: public, max-age=31536000, immutable` para assets estaticos (imagens com hash no nome)
3. **Tiered Cache:** Ativar para reduzir hits no origin — se o conteudo nao esta no edge local, o sistema checa "super caches" regionais antes de ir ao R2
4. **Infrequent Access class:** Para dados acessados raramente, reduz custo de storage

---

## 3. Cloudflare D1 — Database no Edge

### 3.1 SQLite no Edge — Casos de Uso e Limitacoes

D1 e o banco de dados relacional da Cloudflare, construido sobre SQLite. Mantido como "o primeiro banco SQL queryable" do ecossistema Workers.

#### Quando D1 Faz Sentido

| Cenario | D1 e Adequado? | Motivo |
|---|---|---|
| Blog / CMS | Sim | Read-heavy, dados estruturados simples |
| Catalogo de produtos | Sim | Muitas leituras, poucas escritas |
| Perfis de usuario | Sim | Read-heavy, queries simples |
| SaaS multi-tenant | Sim | 1 database por tenant (ate 50K databases/conta) |
| eCommerce transacional | Depende | Escritas moderadas podem funcionar |
| Analytics real-time | Nao | Write-heavy, precisa de database otimizado para append |
| Sistema financeiro | Nao | Requer ACID completo, multi-writer |
| App com 100K+ writes/s | Nao | SQLite tem single-writer |

#### Limitacoes Concretas

| Limitacao | Valor |
|---|---|
| Tamanho maximo do database | 10 GB |
| Databases por conta | 50,000 |
| Write throughput | 500-2K writes/s sustentado |
| Concorrencia de escrita | Single writer (WAL mode: reads concorrentes) |
| Export/TCP externo | Nao disponivel nativamente |
| Migration tooling | Limitado comparado ao ecossistema Postgres |

Fonte: [D1 Limits - Cloudflare Docs](https://developers.cloudflare.com/d1/platform/limits/), [SQLite Edge Production Ready](https://www.sitepoint.com/sqlite-edge-production-readiness-2026/)

### 3.2 D1 vs PostgreSQL Tradicional

| Dimensao | Cloudflare D1 | PostgreSQL (Supabase/Neon) |
|---|---|---|
| **Motor** | SQLite | PostgreSQL |
| **Latencia para reads** | Sub-10ms (warm, co-located) | 20-100ms (depende da regiao) |
| **Write throughput** | 500-2K/s | 10K-100K/s |
| **Tamanho maximo** | 10 GB por database | Ilimitado (pratica: TB+) |
| **Features avancadas** | SQL basico | JSONB, full-text search, extensions, RLS |
| **Multi-tenant** | 1 DB por tenant (natural) | Schema-based ou RLS |
| **Custo** | Free ate 5 GB + 25M reads | $0-25/mes (Supabase free/pro) |
| **Lock-in** | Alto (sem export facil) | Baixo (Postgres padrao) |
| **Ecossistema** | Limitado | Enorme (ORMs, tools, extensions) |

### 3.3 Read Replicas e Sharding

D1 mantém um database primario em uma localizacao com read replicas distribuidas estrategicamente. Para escalar alem de 10 GB, desenvolvedores implementam sharding manual — por exemplo, um database D1 por tenant ou por regiao.

**Exemplo de sharding por tenant:**
```javascript
export default {
  async fetch(request, env) {
    const tenantId = getTenantId(request);
    // Cada tenant tem seu proprio database D1
    const db = env[`DB_${tenantId}`];
    const results = await db
      .prepare("SELECT * FROM products WHERE active = 1")
      .all();
    return Response.json(results);
  },
};
```

Fonte: [Scaling D1 from 10GB to 500GB](https://medium.com/@tristantrommer/scaling-cloudflare-d1-from-10-gb-to-500-gb-with-manual-database-sharding-4e95d6deb742)

### 3.4 Migration Patterns

```javascript
// migrations/0001_create_users.sql
CREATE TABLE IF NOT EXISTS users (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  email TEXT NOT NULL UNIQUE,
  name TEXT NOT NULL,
  created_at TEXT DEFAULT (datetime('now'))
);

CREATE INDEX idx_users_email ON users(email);
```

```bash
# Aplicar migrations com wrangler
npx wrangler d1 migrations apply MY_DATABASE
```

---

## 4. Cloudflare WAF e Seguranca

### 4.1 Web Application Firewall (WAF)

O WAF da Cloudflare inclui managed rulesets pre-configurados para protecao contra ameacas conhecidas, incluindo OWASP Top 10. A suite de seguranca de aplicacoes abrange WAF, Rate Limiting, DDoS L7, API Shield, Bot Management e seguranca client-side.

**Niveis de WAF por plano:**

| Feature | Free | Pro ($20/mes) | Business ($200/mes) | Enterprise |
|---|---|---|---|---|
| Managed rulesets | Basico | Sim | Sim | Sim + custom |
| Custom rules | 5 | 20 | 100 | 1000+ |
| Rate limiting rules | 1 | 10 | 50 | Custom |
| Firewall Analytics | Basico | Detalhado | Detalhado | Avancado |
| WAF ML scoring | Nao | Nao | Sim | Sim |

Fonte: [WAF Overview - Cloudflare Docs](https://developers.cloudflare.com/waf/)

### 4.2 DDoS Protection

A protecao DDoS da Cloudflare opera em camada 3/4 (rede) e camada 7 (aplicacao):

- **Free tier:** Protecao DDoS unmetered — sem limites de volume
- **Managed rulesets L7:** Regras pre-configuradas que detectam vetores de ataque DDoS conhecidos
- **Defesa proativa:** Configuracoes de threshold customizaveis por dominio

**Todas as protecoes DDoS sao gratuitas** em todos os planos, incluindo o free. Isso e um diferencial significativo — AWS WAF e AWS Shield Advanced custam a partir de $3,000/mes.

Fonte: [DDoS Protection - Cloudflare Docs](https://developers.cloudflare.com/ddos-protection/best-practices/proactive-defense/)

### 4.3 Rate Limiting

Rate limiting rules permitem definir limites de taxa para requests que correspondem a uma expressao, e a acao a executar quando esses limites sao atingidos.

```javascript
// Exemplo de configuracao via API
// Rate limit: 100 requests por minuto por IP no endpoint /api/*
{
  "expression": "(http.request.uri.path matches \"^/api/\")",
  "action": "block",
  "ratelimit": {
    "characteristics": ["ip.src"],
    "period": 60,
    "requests_per_period": 100,
    "mitigation_timeout": 600
  }
}
```

**Casos de uso:**
- Proteger endpoints de login contra brute-force (ex: 5 tentativas/15min)
- Limitar chamadas de API por cliente
- Mitigar ataques volumetricos (mass assignment, data exfiltration)

**Novidade 2025:** Unmetered rate limiting para clientes self-serve — sem custo adicional por request bloqueado.

Fonte: [Rate Limiting Rules - Cloudflare Docs](https://developers.cloudflare.com/waf/rate-limiting-rules/)

### 4.4 Bot Management

O sistema de Bot Management atribui um **bot score de 1 a 99** para cada request:

| Score Range | Classificacao | Acao Tipica |
|---|---|---|
| 1-29 | Provavel bot | Block ou challenge |
| 30-49 | Suspeito | Managed challenge |
| 50-79 | Possivelmente humano | Monitorar |
| 80-99 | Provavel humano | Allow |

Acoes disponiveis baseadas no score: block, allow, rate limit, managed challenge, JS challenge, interactive challenge.

Fonte: [Bot Management - Cloudflare Reference Architecture](https://developers.cloudflare.com/reference-architecture/diagrams/bots/bot-management/)

### 4.5 Page Shield (Client-Side Security)

Page Shield (renomeado para "Client-Side Security") monitora scripts, conexoes e cookies carregados pelos visitantes do site.

**Capacidades:**
- **Script monitoring:** Detecta scripts de terceiros maliciosos ou comprometidos (ataques Magecart-style)
- **Connection monitoring:** Monitora conexoes externas feitas pelo browser
- **Cookie monitoring:** Rastreia cookies first e third-party
- **AI detection cascading:** Combina graph neural networks e LLMs, reduzindo falsos positivos em ate **200x**
- **Content Security Policies:** Geracao automatica de CSP headers

**Novidade junho 2025:** Client-Side Security disponibilizado para **todos os clientes, incluindo free tier**. Free e Pro recebem script monitoring, connection monitoring e cookie monitoring.

Fonte: [Client-Side Security - Cloudflare Docs](https://developers.cloudflare.com/client-side-security/), [Client-Side Security Open to Everyone](https://blog.cloudflare.com/client-side-security-open-to-everyone/)

### 4.6 Zero Trust Access (Cloudflare Access + Tunnel)

Cloudflare Zero Trust combina dois componentes:

**Cloudflare Access:** Proxy identity-aware que autentica usuarios antes de conceder acesso. Integra com provedores de identidade existentes (Azure AD, Okta, Google, GitHub).

**Cloudflare Tunnel:** `cloudflared` cria conexoes outbound-only da infraestrutura para o edge da Cloudflare, **eliminando regras de firewall inbound** e a necessidade de VPNs tradicionais.

```yaml
# config.yml do cloudflared
tunnel: meu-tunnel-id
credentials-file: /root/.cloudflared/credentials.json

ingress:
  - hostname: app.meudominio.com
    service: http://localhost:3000
  - hostname: api.meudominio.com
    service: http://localhost:8080
  - service: http_status:404
```

**Beneficios:**
- Sem portas abertas na internet
- Autenticacao por aplicacao (nao por rede)
- Suporta IPs privados e CIDR ranges
- Substitui VPNs corporativas

Fonte: [Cloudflare Tunnel - Docs](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/), [How to Configure Zero Trust](https://oneuptime.com/blog/post/2026-01-25-cloudflare-zero-trust/view)

---

## 5. Cloudflare CDN e Performance

### 5.1 Cache Rules e Cache Keys

Cloudflare oferece controle granular sobre caching atraves de Cache Rules:

| Tipo de Regra | Descricao | Exemplo |
|---|---|---|
| **Cache Everything** | Cacheia inclusive HTML | Landing pages estaticas |
| **Bypass Cache** | Nao cacheia | APIs autenticadas, dashboards |
| **Custom Cache Key** | Inclui query params, headers, cookies no cache key | Personalizacao por pais/lingua |
| **Edge TTL** | Tempo que o edge mantem o cache | 1h para API, 1 ano para assets |
| **Browser TTL** | Tempo que o browser mantem o cache | 30 dias para imagens |
| **Tiered Cache** | Hierarquia de cache (edge -> regional -> origin) | Reduz carga no origin |

**Tiered Cache** cria uma hierarquia onde, se o conteudo nao esta no edge local, o sistema checa "super caches" regionais antes de solicitar ao origin server. Isso aumenta dramaticamente as cache hit rates e reduz a carga no origin.

Fonte: [CDN Reference Architecture - Cloudflare Docs](https://developers.cloudflare.com/reference-architecture/architectures/cdn/)

### 5.2 Image Optimization

Cloudflare oferece otimizacao de imagens em tempo real no edge:

| Feature | Plano | Descricao |
|---|---|---|
| **Polish** | Pro+ | Compressao automatica (lossy/lossless) |
| **WebP/AVIF conversion** | Pro+ | Converte para formatos modernos automaticamente |
| **Image Resizing** | Business+ / Workers | Redimensiona on-the-fly baseado em device/viewport |
| **Mirage** | Pro+ | Lazy loading inteligente para imagens |

A otimizacao automatica converte imagens para formatos modernos (WebP, AVIF), redimensiona baseado no device e viewport, e comprime sem perda visivel de qualidade — tudo no edge sem trabalho manual.

Fonte: [Cloudflare CDN Review 2025](https://www.logicpin.com/technology/is-cloudflare-still-the-best-choice-for-cdn-performance-optimization-and-website-security-in-2025/)

### 5.3 Early Hints e HTTP/3

#### Early Hints (HTTP 103)
Early Hints usa o status code HTTP 103 como primeira resposta ao cliente, com "hints" sendo headers HTTP que indicam recursos que o browser deve comecar a carregar enquanto o servidor prepara a resposta final.

**Resultado:** Melhoria de mais de **30% no Largest Contentful Paint (LCP)** medido pelo Web Page Test.

```javascript
// Exemplo em Cloudflare Workers
export default {
  async fetch(request) {
    // Retorna Early Hints antes da resposta completa
    return new Response(htmlContent, {
      headers: {
        Link: '</styles.css>; rel=preload; as=style, </app.js>; rel=preload; as=script',
      },
    });
  },
};
```

**Nota:** Early Hints so funciona sobre HTTP/2 e HTTP/3.

Fonte: [Early Hints - Cloudflare Blog](https://blog.cloudflare.com/early-hints/), [Early Hints Performance](https://blog.cloudflare.com/early-hints-performance/)

#### HTTP/3 (QUIC)
- Reduz latencia em **30% no mobile** segundo report Akamai 2025
- QUIC combina handshakes de transporte e encriptacao, resultando em ate **10% mais rapido no TTFB**
- Cloudflare open-sourced `tokio-quiche` — biblioteca Rust assincrona para QUIC/HTTP/3

Fonte: [Cloudflare tokio-quiche - InfoQ](https://www.infoq.com/news/2025/12/quic-http3-rust/)

### 5.4 Argo Smart Routing

Argo Smart Routing usa dados proprietarios em tempo real sobre congestionamento de rede para enviar requests e responses apenas pelas rotas mais rapidas e confiaveis.

**Resultado medio:** **30% mais rapido** na performance de web apps.

**Como funciona:**
1. Cloudflare monitora condicoes de rede em tempo real entre todos os 300+ PoPs
2. Quando um request chega, Argo determina a rota otima baseado em latencia atual
3. O request e roteado pela rede privada da Cloudflare em vez da internet publica
4. A resposta retorna pela rota mais rapida disponivel

**Pricing:** Custo adicional baseado em trafego roteado (~$5/mes por 1 GB).

Fonte: [Argo Smart Routing - Cloudflare Docs](https://developers.cloudflare.com/argo-smart-routing/)

### 5.5 Load Balancing

Cloudflare Load Balancing distribui trafego entre endpoints, reduzindo latencia e melhorando a experiencia do usuario.

**Features:**
- **Traffic steering:** Por latencia, regiao geografica, ou coordenadas GPS
- **Health checks:** Monitoramento em intervalos configuraveis, multiplos data centers
- **Failover automatico:** Redireciona trafego quando endpoint esta unhealthy
- **Custom rules:** Comportamento baseado em caracteristicas do request
- **Monitor Groups (2025):** Combinam multiplos health monitors em um grupo logico para verificacoes mais sofisticadas

**Pricing:** $5/mes por dominio + $0.50 por milhao de health-checked requests.

Fonte: [Load Balancing - Cloudflare Docs](https://developers.cloudflare.com/load-balancing/)

---

## 6. Stack Cloudflare + Vercel + Supabase

### 6.1 Cloudflare como DNS + WAF + CDN com Vercel Hosting

A combinacao mais comum: Cloudflare fornece seguranca e CDN, Vercel hospeda a aplicacao Next.js, Supabase fornece backend.

```
[Usuario] --> [Cloudflare Edge]
                |
                |- DNS Resolution
                |- WAF (filtra ataques)
                |- DDoS Protection
                |- CDN (assets estaticos)
                |- Rate Limiting
                |
                v
           [Vercel Edge Network]
                |
                |- Next.js App (SSR/SSG/ISR)
                |- Edge Functions
                |- API Routes
                |
                v
           [Supabase]
                |
                |- PostgreSQL Database
                |- Auth (JWT)
                |- Realtime
                |- Storage
                |- Edge Functions
```

**Configuracao de DNS:** Os registros DNS apontam para a infraestrutura do Vercel, com Cloudflare atuando como DNS provider e proxy reverso.

**Cuidado importante:** Vercel desaconselha usar reverse proxies como Cloudflare na frente do Vercel quando Bot Protection esta ativado, pois isso **degrada significativamente** a precisao de deteccao e performance. A recomendacao e usar Cloudflare para DNS only, ou aceitar o trade-off.

Fonte: [Vercel + Cloudflare Super Team](https://www.getfishtank.com/insights/how-does-vercel-and-cloudflare-create-a-modern-infrastructure-super-team), [Should I use Cloudflare in front of Vercel?](https://vercel.com/kb/guide/cloudflare-with-vercel)

### 6.2 Configuracoes Recomendadas

#### Opcao A: DNS-Only (Recomendada pela Vercel)

```
Cloudflare DNS (proxy OFF, grey cloud) --> Vercel
```

- Cloudflare fornece apenas DNS
- Vercel cuida de CDN, SSL e Edge
- Bot Protection do Vercel funciona corretamente
- Melhor para apps Next.js que dependem de features Vercel

#### Opcao B: Full Proxy (Melhor Seguranca)

```
Cloudflare DNS + WAF + CDN (proxy ON, orange cloud) --> Vercel
```

- Cloudflare cuida de seguranca (WAF, DDoS, rate limiting)
- Cache estrategico no Cloudflare para assets estaticos
- **Trade-off:** Bot Protection do Vercel nao funciona bem
- Melhor para apps que precisam de seguranca enterprise

#### Opcao C: Hibrida

```
dominio.com         --> Cloudflare proxy ON  --> Vercel (landing, marketing)
app.dominio.com     --> Cloudflare proxy OFF --> Vercel (app interativo)
api.dominio.com     --> Cloudflare proxy ON  --> Supabase/Workers (API)
```

### 6.3 Edge Middleware — Vercel Edge vs Cloudflare Workers

| Aspecto | Vercel Edge Middleware | Cloudflare Workers |
|---|---|---|
| **Posicao** | Antes do framework (Next.js) | Independente do framework |
| **Runtime** | V8 Isolates | V8 Isolates |
| **Integracao Next.js** | Nativa (middleware.ts) | Via fetch/service bindings |
| **Casos de uso** | Auth, redirects, A/B testing, i18n | APIs independentes, microservicos, edge compute |
| **Storage** | Vercel KV, Vercel Blob | KV, D1, R2, Durable Objects |
| **Ideal quando** | Feature do app Next.js | Logica independente do framework |

### 6.4 Supabase Atras do Cloudflare

**Caching de queries Supabase no edge com Workers + KV:**

```javascript
// Worker que cacheia queries Supabase no KV
export default {
  async fetch(request, env) {
    const cacheKey = new URL(request.url).pathname;
    
    // 1. Tentar cache KV primeiro
    const cached = await env.CACHE.get(cacheKey, "json");
    if (cached) {
      return Response.json(cached, {
        headers: { "X-Cache": "HIT" },
      });
    }

    // 2. Buscar no Supabase
    const { createClient } = await import("@supabase/supabase-js");
    const supabase = createClient(
      env.SUPABASE_URL,
      env.SUPABASE_ANON_KEY
    );
    
    const { data, error } = await supabase
      .from("products")
      .select("*")
      .eq("active", true);

    if (error) {
      return Response.json({ error: error.message }, { status: 500 });
    }

    // 3. Cachear no KV (TTL: 5 minutos)
    await env.CACHE.put(cacheKey, JSON.stringify(data), {
      expirationTtl: 300,
    });

    return Response.json(data, {
      headers: { "X-Cache": "MISS" },
    });
  },
};
```

**Cuidado com auth e caching:** Se o CDN cacheia uma resposta com token de auth e serve para outro usuario, esse usuario pode ser autenticado como a pessoa errada. Desde `@supabase/ssr v0.10.0`, a biblioteca automaticamente passa cache headers necessarios (Cache-Control, Expires, Pragma) para prevenir isso.

**Para buckets privados do Supabase Storage:** Permissoes sao verificadas por usuario, entao caching no CDN resulta em cache miss para cada usuario diferente.

Fonte: [Cache Supabase at Edge - egghead.io](https://egghead.io/courses/cache-supabase-data-at-the-edge-with-cloudflare-workers-and-kv-storage-883c7959), [Supabase Storage CDN - Docs](https://supabase.com/docs/guides/storage/cdn/fundamentals)

### 6.5 Estrategias de Otimizacao de Custo

**Stack de custo zero para MVP/Side Projects:**

| Componente | Servico | Free Tier |
|---|---|---|
| Frontend | Cloudflare Pages | Unlimited bandwidth, 500 builds/mes |
| Backend compute | Cloudflare Workers | 100K requests/dia |
| Database | Supabase | 500 MB Postgres, 50K MAU auth |
| Object storage | Cloudflare R2 | 10 GB storage, 10M reads/mes |
| Cache | Workers KV | 100K reads/dia |
| DNS + CDN + WAF | Cloudflare | Incluso free tier |
| **Total** | | **$0/mes** |

**Stack para crescimento (1,000+ usuarios):**

| Componente | Servico | Custo |
|---|---|---|
| Frontend | Vercel Pro | $20/mes |
| Backend compute | Cloudflare Workers Paid | $5/mes |
| Database | Supabase Pro | $25/mes |
| Object storage | Cloudflare R2 | $0.015/GB/mes |
| Cache | Workers KV | Incluso no Workers Paid |
| DNS + WAF | Cloudflare | Free |
| **Total estimado** | | **$50-75/mes** |

Fonte: [Free Full-Stack App with Cloudflare + Supabase](https://dev.to/hexshift/how-to-host-a-scalable-full-stack-app-for-free-using-cloudflare-pages-workers-and-supabase-2ke5), [MVP Tech Stack 2025](https://www.startupbricks.in/blog/ultimate-tech-stack-for-your-mvp-in-2025)

---

## 7. Cloudflare Pages

### 7.1 Pages vs Vercel para Static/JAMstack

| Dimensao | Cloudflare Pages | Vercel |
|---|---|---|
| **Bandwidth** | Ilimitado (free) | 100 GB/mes (free), 1 TB (pro) |
| **Builds** | 500/mes (free) | 6000/mes (free) |
| **Concurrent builds** | 1 (free), 5 (pro) | 1 (free), 1 (pro) |
| **Preview deployments** | Ilimitado | Ilimitado |
| **Custom domains** | Ilimitado | Ilimitado |
| **Edge functions** | Via Workers (integrado) | Edge Functions nativas |
| **Framework support** | Generico (qualquer framework) | Otimizado para Next.js |
| **Git integration** | GitHub, GitLab | GitHub, GitLab, Bitbucket |
| **Preco Pro** | $5/mes (Workers Paid) | $20/mes |
| **Melhor para** | Static sites, custo otimizado | Next.js, DX premium |

**Cloudflare Pages vence em custo** — especialmente para projetos com alto trafego, gracas ao bandwidth ilimitado.

**Vercel vence em DX para Next.js** — integracao nativa, preview deployments inteligentes, analytics integrado.

Fonte: [Vercel vs Cloudflare Pages Comparison](https://www.freetiers.com/blog/vercel-vs-cloudflare-pages-comparison), [Vercel vs Netlify vs Cloudflare Pages 2025](https://www.ai-infra-link.com/vercel-vs-netlify-vs-cloudflare-pages-2025-comparison-for-developers/)

### 7.2 Functions (Server-Side) no Pages

Cloudflare Pages Functions sao Workers integrados ao Pages. Basta criar arquivos na pasta `functions/` do projeto:

```
my-project/
  functions/
    api/
      users.ts       --> GET/POST /api/users
      users/
        [id].ts      --> GET/PUT/DELETE /api/users/:id
    _middleware.ts    --> Middleware global
  public/
    index.html
```

```typescript
// functions/api/users.ts
export const onRequestGet: PagesFunction<Env> = async (context) => {
  const users = await context.env.DB.prepare(
    "SELECT * FROM users LIMIT 50"
  ).all();
  return Response.json(users.results);
};

export const onRequestPost: PagesFunction<Env> = async (context) => {
  const body = await context.request.json();
  await context.env.DB.prepare(
    "INSERT INTO users (name, email) VALUES (?, ?)"
  ).bind(body.name, body.email).run();
  return Response.json({ success: true }, { status: 201 });
};
```

### 7.3 Quando Usar Pages vs Workers vs Ambos

| Cenario | Recomendacao |
|---|---|
| Site estatico com blog | Pages only |
| Site estatico + API simples | Pages + Functions |
| API REST independente | Workers only |
| App full-stack edge-native | Pages (frontend) + Workers (backend via service bindings) |
| Microservicos no edge | Workers only (com service bindings) |
| Tarefas agendadas | Workers (Cron Triggers) |
| Computacao stateful | Workers + Durable Objects |

**Smart Placement:** Feature que automaticamente roda Workers compute-heavy mais perto das fontes de dados (database) em vez de forcar tudo para o edge. Isso resolve o problema de latencia quando o Worker precisa consultar um database em uma regiao especifica.

---

## 8. Edge Computing Decision Framework

### 8.1 Quando Computar no Edge vs Origin

O principio central e **"Data Gravity"** — se os dados necessarios para a logica vivem no edge, a logica tambem deve estar la. Caso contrario, o padrao "Chatty Edge" vai aumentar latencia.

#### Decision Matrix

| Fator | Edge (Cloudflare Workers) | Origin (Server tradicional) |
|---|---|---|
| **Dados necessarios** | Disponiveis no edge (KV, cache) | Database centralizado |
| **Latencia critica** | Sim — precisa de sub-50ms global | Toleravel — 200ms aceitavel |
| **Complexidade de logica** | Simples a media | Complexa, transacional |
| **Estado necessario** | Stateless ou Durable Objects | Stateful com ACID |
| **CPU time** | < 30s | Minutos a horas |
| **Memoria** | < 128 MB | GB+ |
| **Dependencias** | Poucas, leves | Muitas, pesadas |
| **Escala** | Global, milhoes de requests | Regional, milhares de requests |

#### O que Pertence a Cada Camada

```
EDGE (Cloudflare Workers)
|-- Autenticacao / validacao de JWT
|-- Rate limiting
|-- A/B testing / feature flags
|-- Redirects / rewrites
|-- Personalizacao por geo/device
|-- Cache invalidation
|-- API routing / gateway
|-- Image optimization
|-- Bot detection
|-- Headers manipulation

ORIGIN (Server / Supabase / AWS)
|-- Business logic complexa
|-- Transacoes ACID
|-- Queries complexas ao database
|-- Processamento batch
|-- Integracao com servicos internos
|-- Machine learning (modelos grandes)
|-- File processing pesado
|-- Geracoes de PDF/relatorios
```

**Performance:** Edge reduz TTFB em **60-80%** para usuarios globais. SSR tradicional no origin resulta em TTFB de 200-800ms, enquanto Edge-Side Rendering atinge **20-50ms** TTFB.

Fonte: [Edge vs Origin 2026](https://dailydevpost.com/blog/edge-vs-origin-business-logic-cdn), [Edge Computing Web Performance Architecture 2026](https://www.digitalapplied.com/blog/edge-computing-2026-web-performance-architecture)

### 8.2 Data Locality Considerations

```
                    [Usuario no Brasil]
                           |
                    [Cloudflare PoP SP]
                     /              \
            [Edge Logic]        [Cache Hit?]
                |                    |
                |               [Sim] --> Resposta imediata (~5ms)
                |               [Nao] --> Tiered Cache regional
                |                              |
                |                    [Hit?] --> Resposta (~20ms)
                |                    [Miss] --> Origin
                |                              |
                v                              v
         [Supabase/DB]  <--  [Origin Server]
         (us-east-1)         (us-east-1)
              |
         Resposta (~150ms RTT Brasil->US)
```

**Estrategia de mitigacao:** Usar Workers KV ou D1 read replicas para dados frequentemente acessados, reduzindo a necessidade de roundtrips ao origin.

### 8.3 Edge Caching Strategies

| Estrategia | TTL | Use Case | Invalidacao |
|---|---|---|---|
| **Imutavel** | 1 ano | Assets com hash (app.a1b2c3.js) | Nunca (novo hash = novo arquivo) |
| **Stale-while-revalidate** | 1h, SWR 24h | Dados semi-dinamicos | Background revalidation |
| **Short TTL** | 60s | Dados com atualizacao frequente | TTL expira naturalmente |
| **Cache tags** | Variavel | Conteudo por categoria | Purge por tag |
| **No cache** | 0 | Dados pessoais, auth | N/A |
| **Tiered** | Variavel | Qualquer conteudo | Propaga do origin |

```javascript
// Exemplo: Stale-while-revalidate no Workers
export default {
  async fetch(request, env) {
    const cache = caches.default;
    let response = await cache.match(request);

    if (response) {
      // Revalidar em background se proximo do vencimento
      const age = parseInt(response.headers.get("Age") || "0");
      if (age > 3000) {
        // > 50 minutos de 1h TTL
        event.waitUntil(revalidateAndCache(request, cache));
      }
      return response;
    }

    // Cache miss — buscar do origin
    response = await fetch(request);
    response = new Response(response.body, response);
    response.headers.set("Cache-Control", "public, max-age=3600");
    event.waitUntil(cache.put(request, response.clone()));
    return response;
  },
};
```

### 8.4 Global Deployment Patterns

#### Pattern 1: Edge-First (Cloudflare Workers)
```
Toda logica no edge. Database no edge (D1) ou cacheado (KV).
- Latencia: 10-30ms global
- Complexidade: Limitada pelo runtime
- Custo: Muito baixo
- Ideal para: APIs simples, sites estaticos com personalizacao
```

#### Pattern 2: Edge + Origin Hibrido
```
Edge: Auth, routing, caching, personalizacao
Origin: Business logic, database, processamento
- Latencia: 20-100ms (depende de cache hit rate)
- Complexidade: Ilimitada
- Custo: Moderado
- Ideal para: SaaS, eCommerce, apps full-stack
```

#### Pattern 3: Origin-First com Edge Caching
```
Toda logica no origin. Edge so para CDN e seguranca.
- Latencia: 100-500ms (com CDN cache reduz para sub-100ms)
- Complexidade: Ilimitada
- Custo: Maior (compute centralizado)
- Ideal para: Apps enterprise, sistemas legados
```

### 8.5 Anti-Patterns de Edge Computing

| Anti-Pattern | Problema | Solucao |
|---|---|---|
| **Chatty Edge** | Worker faz 10+ requests ao origin por invocacao | Mover logica para o origin ou cachear dados no KV |
| **Fat Edge** | Worker com 128 MB de logica complexa | Dividir em microservicos com service bindings |
| **Stateful Edge sem Durable Objects** | Tentar manter estado em Workers regulares | Usar Durable Objects para estado |
| **Edge database para writes pesados** | D1 com 10K+ writes/s | Usar Postgres/Supabase centralizado |
| **Cache sem invalidacao** | Dados stale servidos indefinidamente | Implementar TTL e purge strategies |
| **Global deploy sem considerar data locality** | Edge longe do database | Usar Smart Placement ou Hyperdrive |

### 8.6 Latency Optimization Checklist

```
[ ] DNS: Cloudflare DNS (anycast, < 10ms resolution)
[ ] TLS: HTTP/3 + QUIC habilitado (reduz handshake)
[ ] Cache: Tiered Cache ativado (reduz origin hits)
[ ] Early Hints: Ativado para preload de CSS/JS criticos
[ ] Images: Polish + WebP/AVIF conversion ativados
[ ] Argo: Smart Routing ativado (30% mais rapido, custo extra)
[ ] Workers: Auth/routing no edge (elimina roundtrip ao origin)
[ ] KV: Dados frequentes cacheados no edge (sub-ms reads)
[ ] Compression: Brotli habilitado
[ ] Smart Placement: Ativado para Workers que consultam databases
```

---

## 9. Fontes e Referencias

### Documentacao Oficial Cloudflare
- [How Workers Works](https://developers.cloudflare.com/workers/reference/how-workers-works/)
- [Workers Pricing](https://developers.cloudflare.com/workers/platform/pricing/)
- [Workers Limits](https://developers.cloudflare.com/workers/platform/limits/)
- [Cron Triggers](https://developers.cloudflare.com/workers/configuration/cron-triggers/)
- [Durable Objects](https://developers.cloudflare.com/durable-objects/)
- [Service Bindings](https://developers.cloudflare.com/workers/runtime-apis/bindings/service-bindings/)
- [Storage Options](https://developers.cloudflare.com/workers/platform/storage-options/)
- [R2 Pricing](https://developers.cloudflare.com/r2/pricing/)
- [R2 Presigned URLs](https://developers.cloudflare.com/r2/api/s3/presigned-urls/)
- [D1 Overview](https://developers.cloudflare.com/d1/)
- [D1 Limits](https://developers.cloudflare.com/d1/platform/limits/)
- [WAF Overview](https://developers.cloudflare.com/waf/)
- [Rate Limiting Rules](https://developers.cloudflare.com/waf/rate-limiting-rules/)
- [DDoS Protection](https://developers.cloudflare.com/ddos-protection/best-practices/proactive-defense/)
- [Client-Side Security](https://developers.cloudflare.com/client-side-security/)
- [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/)
- [Argo Smart Routing](https://developers.cloudflare.com/argo-smart-routing/)
- [Load Balancing](https://developers.cloudflare.com/load-balancing/)
- [Early Hints](https://developers.cloudflare.com/cache/advanced-configuration/early-hints/)
- [CDN Reference Architecture](https://developers.cloudflare.com/reference-architecture/architectures/cdn/)
- [Bot Management Reference Architecture](https://developers.cloudflare.com/reference-architecture/diagrams/bots/bot-management/)

### Blog Posts e Anuncios Cloudflare
- [Eliminating Cold Starts](https://blog.cloudflare.com/eliminating-cold-starts-with-cloudflare-workers/)
- [Introducing Durable Objects](https://blog.cloudflare.com/introducing-workers-durable-objects/)
- [Introducing Worker Services](https://blog.cloudflare.com/introducing-worker-services/)
- [Early Hints Performance](https://blog.cloudflare.com/early-hints-performance/)
- [Client-Side Security Open to Everyone](https://blog.cloudflare.com/client-side-security-open-to-everyone/)
- [Unmetered Rate Limiting](https://blog.cloudflare.com/unmetered-ratelimiting/)
- [tokio-quiche Open Source](https://www.infoq.com/news/2025/12/quic-http3-rust/)

### Case Studies
- [Shopify Case Study](https://www.cloudflare.com/case-studies/shopify/)
- [Discord Case Study](https://www.cloudflare.com/case-studies/discord/)
- [Cloudflare Case Studies for SaaS](https://www.clodo.dev/saas-product-startups-cloudflare-case-studies)

### Comparacoes e Analises
- [Lambda@Edge vs Workers vs Vercel Edge 2025](https://prabhatgiri.com/blogs/lambdaedge-vs-cloudflare-workers-vs-vercel-edge-latency-limits-and-cost-in-2025/)
- [Edge Performance Comparison 2026](https://dev.to/dataformathub/cloudflare-vs-vercel-vs-netlify-the-truth-about-edge-performance-2026-50h0)
- [Dynamic Workers Open Beta - InfoQ](https://www.infoq.com/news/2026/04/cloudflare-dynamic-workers-beta/)
- [R2 vs S3 Deep Dive](https://yconsulting.substack.com/p/cloudflare-r2-vs-the-big-3-a-deep)
- [R2 vs S3 Comparison](https://www.digitalapplied.com/blog/cloudflare-r2-vs-aws-s3-comparison)
- [Supabase vs R2 2026](https://www.buildmvpfast.com/compare/supabase-vs-r2)
- [Vercel vs Cloudflare Pages](https://www.freetiers.com/blog/vercel-vs-cloudflare-pages-comparison)
- [Edge vs Origin 2026](https://dailydevpost.com/blog/edge-vs-origin-business-logic-cdn)
- [SQLite Edge Production Ready](https://www.sitepoint.com/sqlite-edge-production-readiness-2026/)
- [Scaling D1 via Sharding](https://medium.com/@tristantrommer/scaling-cloudflare-d1-from-10-gb-to-500-gb-with-manual-database-sharding-4e95d6deb742)

### Integracoes e Tutorials
- [Vercel + Cloudflare Architecture](https://www.getfishtank.com/insights/how-does-vercel-and-cloudflare-create-a-modern-infrastructure-super-team)
- [Cloudflare with Vercel - Vercel KB](https://vercel.com/kb/guide/cloudflare-with-vercel)
- [Upload Files to R2 in Next.js](https://www.buildwithmatija.com/blog/how-to-upload-files-to-cloudflare-r2-nextjs)
- [Cache Supabase at Edge](https://egghead.io/courses/cache-supabase-data-at-the-edge-with-cloudflare-workers-and-kv-storage-883c7959)
- [Free Full-Stack with Cloudflare + Supabase](https://dev.to/hexshift/how-to-host-a-scalable-full-stack-app-for-free-using-cloudflare-pages-workers-and-supabase-2ke5)
- [Workers + Supabase Integration](https://developers.cloudflare.com/workers/databases/third-party-integrations/supabase/)
- [Zero Trust Configuration](https://oneuptime.com/blog/post/2026-01-25-cloudflare-zero-trust/view)

### Performance e Otimizacao
- [Cloudflare CDN Review 2025](https://www.logicpin.com/technology/is-cloudflare-still-the-best-choice-for-cdn-performance-optimization-and-website-security-in-2025/)
- [Edge Computing Architecture 2026](https://www.digitalapplied.com/blog/edge-computing-2026-web-performance-architecture)
- [Performance Optimization Playbook](https://blog.birdor.com/cloudflare-tutorial-part-8-performance-playbook/)
- [Higher CPU Limits Changelog](https://developers.cloudflare.com/changelog/post/2025-03-25-higher-cpu-limits/)

---

> **Pesquisa conduzida por:** Prism (research-orqx) — Squad Research
> **Metodologia:** Deep Dive (Nivel 3) com 40+ fontes verificadas via WebSearch
> **Confianca:** Alta — todas as claims verificadas contra documentacao oficial e fontes independentes
> **Proximos passos recomendados:** Aplicar decision framework na arquitetura do projeto ativo, definir quais servicos Cloudflare serao adotados baseado no perfil de uso
