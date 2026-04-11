# Database Selection & Architecture

> **Deep Research** -- Framework decisional completo para escolha de banco de dados por caso de uso.
> Pesquisa verificada via WebSearch. Termos tecnicos mantidos em ingles.

---

## Indice

1. [Bancos Relacionais (SQL)](#1-bancos-relacionais-sql)
2. [Bancos NoSQL](#2-bancos-nosql)
3. [In-Memory / Cache](#3-in-memory--cache)
4. [Search Engines](#4-search-engines)
5. [Vector Databases (AI/ML)](#5-vector-databases-aiml)
6. [Time-Series Databases](#6-time-series-databases)
7. [Graph Databases](#7-graph-databases)
8. [Framework de Decisao](#8-framework-de-decisao)

---

## 1. Bancos Relacionais (SQL)

### 1.1 PostgreSQL

PostgreSQL ultrapassou o MySQL em 2025 como o banco mais utilizado entre desenvolvedores profissionais, consolidando a posicao de banco #1 mais desejado no Stack Overflow Survey 2026, com adocao crescendo 12% ano a ano. Em termos de market share entre developers, PostgreSQL atingiu 45.55% contra 41.09% do MySQL.

**Pontos Fortes:**

| Caracteristica | Detalhe |
|----------------|---------|
| Query Optimizer | Superior ao MySQL; Postgres 18 adicionou async I/O com melhoria de 2-3x em sequential scans |
| Tipos avancados | JSONB, arrays, ranges, composite types, hstore |
| Extensibilidade | Object-relational (ORDBMS) com table inheritance, user-defined types |
| Standards compliance | Segue o padrao SQL mais fielmente que qualquer outro banco open-source |
| Licenca | BSD/MIT-like, sem restricoes de distribuicao |
| ACID | Completo, com serializable isolation |

**Pontos Fracos:**

| Limitacao | Contexto |
|-----------|----------|
| Simple reads | MySQL 8.4 supera PostgreSQL 17 em 20-30% em SELECT simples (sysbench OLTP read-only) |
| Curva de aprendizado | Configuracao de autovacuum, tuning de shared_buffers, work_mem requerem expertise |
| Replicacao nativa | Logical replication evolui, mas sharding nativo ainda nao e first-class (diferente de Vitess/CockroachDB) |
| Write-heavy extremo | Em cenarios como Uber-scale, MySQL com sharding dedicado pode ter vantagem |

**Quando usar:** Praticamente qualquer projeto novo. PostgreSQL cobre ~90% dos casos de uso de banco de dados. E a recomendacao default.

#### Extensions Essenciais

```
-- PostGIS: dados geoespaciais
CREATE EXTENSION postgis;
SELECT ST_Distance(
  ST_MakePoint(-46.6333, -23.5505)::geography,  -- Sao Paulo
  ST_MakePoint(-43.1729, -22.9068)::geography    -- Rio de Janeiro
);
-- Retorna distancia em metros (~357 km)

-- pgvector: embeddings e busca vetorial para AI/ML
CREATE EXTENSION vector;
CREATE TABLE documents (
  id SERIAL PRIMARY KEY,
  content TEXT,
  embedding vector(1536)  -- dimensao do OpenAI text-embedding-3-small
);
CREATE INDEX ON documents USING ivfflat (embedding vector_cosine_ops)
  WITH (lists = 100);

-- pg_cron: agendamento interno
CREATE EXTENSION pg_cron;
SELECT cron.schedule('cleanup-old-sessions', '0 3 * * *',
  $$DELETE FROM sessions WHERE expires_at < NOW()$$
);
```

**PostGIS** e maduro e amplamente usado em producao por governos, empresas de logistica e plataformas de mapeamento. Transforma o PostgreSQL em um motor de dados geoespaciais completo.

**pgvector** e atualmente a extensao estrela do ecossistema PostgreSQL para AI/ML. Suporta similarity search, semantic search, RAG, recommendation systems e NLP. A extensao **pgvectorscale** do Timescale complementa com melhor performance em escala.

**pg_cron** permite agendar jobs diretamente dentro do PostgreSQL usando cron syntax familiar, sem scheduler externo. Limitacao: jobs rodam em uma unica conexao -- jobs longos bloqueiam execucoes subsequentes.

Outras extensions notaveis: **TimescaleDB** (time-series), **pgai** (workflows de AI), **pg_partman** (particionamento automatico), **pg_stat_statements** (analise de queries).

> A maioria das extensions funciona bem em conjunto -- pgvector + TimescaleDB + PostGIS podem coexistir no mesmo banco.

#### Servicos Gerenciados

| Servico | Modelo | Diferencial | Preco Inicial | Melhor Para |
|---------|--------|-------------|---------------|-------------|
| **Supabase** | BaaS (Backend-as-a-Service) | Auth + Realtime + Storage + APIs auto-geradas + SQL editor | Free tier (500MB, shared CPU) | MVPs, startups, apps full-stack |
| **Neon** | Serverless Database | Separacao compute/storage, branching, scale-to-zero | Free tier; $0.35/GB-month storage | Dev/staging, workloads variaveis |
| **AWS RDS** | Managed Database | Battle-tested, IAM, compliance enterprise | ~$15/mes (db.t3.micro) | Empresas no ecossistema AWS |
| **Aurora** | AWS-managed, engine propria | 5x throughput do MySQL, 3x do PostgreSQL padrao | ~$30/mes (serverless v2 min) | Alta disponibilidade, enterprise |

**Supabase** adquiriu OrioleDB para buscar arquitetura de storage desacoplado. Inclui auth (email, OAuth), real-time updates, file storage e auto-generated APIs. Para startups construindo MVP, elimina semanas de desenvolvimento backend.

**Neon** foi adquirido pela Databricks em 2025 por ~$1 bilhao. Apos a aquisicao, cortou precos: storage de $1.75 para $0.35/GB-month, compute -25%. Scale-to-zero significa custo zero quando o banco esta ocioso. Ideal para workloads intermitentes.

**AWS RDS/Aurora**: escolha quando ja esta na AWS, precisa de compliance enterprise (SOC 2, HIPAA, PCI DSS), ou requer controles granulares de IAM e networking. Aurora Serverless v2 escala automaticamente, mas e mais caro que Neon/Supabase para workloads pequenos.

---

### 1.2 MySQL

MySQL ainda mantem DB-Engines score de 858.34 contra 680.08 do PostgreSQL em termos absolutos de presenca no mercado, refletindo decadas de instalacoes existentes.

**Pontos Fortes:**

| Caracteristica | Detalhe |
|----------------|---------|
| Simple reads | 20-30% mais rapido que PostgreSQL em SELECT simples (sysbench) |
| Ecossistema LAMP | Stack padrao para milhoes de aplicacoes web |
| Facilidade de setup | Configuracao inicial mais simples |
| Write-heavy | Vantagem em cenarios de escrita intensiva em escala extrema |

**Quando Preferir MySQL sobre PostgreSQL:**

- Aplicacoes web legacy no stack LAMP/LEMP
- Workloads read-heavy com queries simples (blogs, CMS, e-commerce basico)
- Equipes com experiencia consolidada em MySQL
- Cenarios onde sharding horizontal via Vitess e necessario

#### PlanetScale e Vitess

**Vitess** e um sistema open-source de database clustering para MySQL, desenvolvido no YouTube para escalar o banco principal para petabytes em 70.000 nodes em 20 data centers. Hoje e usado por Slack, HubSpot, Blizzard, Etsy, GitHub e Block.

```
# Vitess permite sharding horizontal com minima mudanca na aplicacao
# Tabelas sao divididas entre multiplas instancias MySQL
# A aplicacao continua usando queries SQL padrao
```

**PlanetScale** e a plataforma gerenciada baseada em Vitess, oferecendo:

- Safe migrations sem downtime (schema changes non-blocking)
- Branching de banco de dados (similar a git branches para schema)
- Horizontal sharding gerenciado
- Em 2025, lancou suporte a **PostgreSQL** alem do MySQL

> PlanetScale foca em empresas com 10.000+ clientes que precisam de escala horizontal. Revenue estimada: $105M em 2026.

---

### 1.3 SQLite

SQLite esta vivendo um renascimento em 2025-2026. Tecnologias como Turso, Cloudflare D1, LibSQL e Litestream convergiram para resolver limitacoes reais do SQLite preservando sua simplicidade.

**Quando SQLite e Surpreendentemente Bom:**

| Cenario | Por que funciona |
|---------|-----------------|
| Aplicacoes single-tenant | Sem contenção de acesso |
| Read-heavy com poucos writers | WAL mode resolve a maioria dos problemas |
| Edge computing | Latencia de leitura single-digit milliseconds (dados locais) |
| Embedded em mobile/desktop | Zero-config, zero-maintenance |
| Prototipos e MVPs | Sem infraestrutura necessaria |
| CI/CD test databases | Instancia efemera, rapida |

#### Edge Databases baseados em SQLite

| Plataforma | Abordagem | Diferencial |
|------------|-----------|-------------|
| **Turso/libSQL** | Fork do SQLite com client/server protocol | Read replicas perto dos usuarios, 4x write throughput do SQLite padrao, embedded replicas com sync automatico |
| **Cloudflare D1** | SQLite dentro do Workers runtime | Replicacao gerenciada e invisivel, binding direto (sem connection string), GA desde abril 2024 |
| **LiteFS** | Replicacao via FUSE no filesystem | Intercepta WAL, transparente para a aplicacao, ideal para Fly.io |

**Performance no Edge:** Embedded replicas (Turso) e FUSE replicas (LiteFS) entregam leituras em single-digit milliseconds porque os dados sao locais. PostgreSQL gerenciado cross-region fica em 30-80ms. Para aplicacoes read-heavy, isso representa 3-10x reducao de latencia.

**Limitacoes reais do SQLite para producao:**

- Apenas um writer por vez (sem write concurrency)
- Sem suporte nativo a roles/permissions no nivel de banco
- Sem replicacao built-in (depende de ferramentas externas)
- Maximo 281 TB por banco (raramente um problema)

---

### 1.4 CockroachDB

CockroachDB entrega **serializable isolation** -- o gold standard de consistencia -- enquanto mantem alta disponibilidade. Usa o protocolo Raft consensus para garantir strong consistency, onde cada write deve ser concordado por maioria das replicas.

**Pontos Fortes:**

| Caracteristica | Detalhe |
|----------------|---------|
| Distribuido nativamente | Multi-region, multi-cloud sem proxy layer |
| Consistencia global | Serializable isolation + Raft consensus |
| SQL padrao | Wire-compatible com PostgreSQL |
| Sobrevive a falhas | Zero downtime durante falhas de node/datacenter |
| Data locality | Dados posicionados conforme regulamentacao (LGPD, GDPR) |

**Casos de Uso Primarios:**

1. **Fintech & Pagamentos**: bancos e fintechs com 50M+ usuarios precisando de uptime continuo
2. **AI-Driven Workloads**: CockroachDB 25.2 introduziu vector indexing (C-SPANN) compativel com PostgreSQL
3. **Multi-Region Global**: posicionamento inteligente de dados por localidade e regulamentacao

**CockroachDB 25.2** trouxe melhorias de ate 50% em throughput com leader leases, buffered writes e WAL failover.

**Quando usar:** Quando voce precisa de SQL distribuido com consistencia forte e zero downtime em escala global. Nao e para MVPs ou projetos pequenos -- a complexidade operacional e o custo sao significativamente maiores que PostgreSQL simples.

```sql
-- Exemplo: tabela com data locality por regiao
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email STRING NOT NULL,
  region STRING NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now()
) LOCALITY REGIONAL BY ROW AS region;

-- Dados de usuarios brasileiros ficam no datacenter do Brasil
-- Dados de europeus ficam na Europa (GDPR compliance)
```

---

## 2. Bancos NoSQL

### 2.1 MongoDB

MongoDB e construido no principio de que "dados acessados juntos devem ser armazenados juntos". O modelo de documentos (BSON/JSON) elimina a necessidade de JOINs na maioria dos casos.

**Pontos Fortes:**

| Caracteristica | Detalhe |
|----------------|---------|
| Schema flexivel | Documentos podem ter estruturas diferentes na mesma collection |
| Developer experience | Queries naturais em JSON, SDKs excelentes |
| Atlas (managed) | Multi-cloud, serverless, vector search integrado |
| Horizontal scaling | Sharding nativo e maduro |
| Aggregation pipeline | Framework poderoso para transformacao de dados |

**Quando Documentos Superam Tabelas:**

- Dados com schema variavel (catalogo de produtos com atributos diferentes)
- Hierarquias profundas (comentarios aninhados, configuracoes complexas)
- Prototipacao rapida onde o schema ainda esta evoluindo
- Content management systems com tipos de conteudo heterogeneos
- Real-time analytics com aggregation pipeline

**Anti-Patterns Comuns (EVITAR):**

| Anti-Pattern | Problema | Solucao |
|-------------|----------|---------|
| Unbounded arrays | Documento pode ultrapassar 16MB (limite BSON) | Separar em documentos referenciados |
| $lookup excessivo | Performance equivalente a JOINs ruins -- fragmenta dados | Denormalizar dados acessados juntos |
| Massive collections | Centenas de collections pequenas | Consolidar em collections maiores com discriminator field |
| Bloated documents | Documentos com campos nunca lidos juntos | Separar em sub-documents ou collections |
| Queries sem indice | Full collection scan | Criar indices compostos alinhados com access patterns |
| Case-insensitive sem indice | Query lenta sem cobertura | Criar collation index |

```javascript
// ANTI-PATTERN: array ilimitado de reviews no documento do produto
{
  _id: "product-123",
  name: "Widget",
  reviews: [/* pode crescer indefinidamente ate 16MB */]
}

// CORRETO: reviews como collection separada
// Collection: reviews
{
  _id: "review-456",
  product_id: "product-123",
  user_id: "user-789",
  rating: 5,
  text: "Excelente produto",
  created_at: ISODate("2025-01-15")
}
```

**MongoDB Atlas em 2025-2026:** inclui vector search para AI features, serverless instances para workloads variaveis, multi-cloud para compliance, e Atlas Stream Processing para materialized views em real-time.

---

### 2.2 DynamoDB

DynamoDB e o banco NoSQL gerenciado da AWS, projetado para single-digit millisecond latency em qualquer escala.

**Single-Table Design:**

O passo mais critico e identificar **todos os access patterns antes de projetar as keys**. Nao e "get orders" -- e "get all orders for a customer in the last 30 days".

```javascript
// Exemplo de single-table design
// PK = tipo de entidade + ID
// SK = ordenacao/filtro

// Acesso: "Todos os pedidos de um cliente nos ultimos 30 dias"
{
  PK: "CUSTOMER#cust-123",
  SK: "ORDER#2025-03-15#order-456",
  // ... atributos do pedido
}

// Acesso: "Detalhes de um produto especifico"
{
  PK: "PRODUCT#prod-789",
  SK: "METADATA",
  // ... atributos do produto
}

// GSI1: busca por status do pedido
// GSI1PK: "STATUS#PENDING"
// GSI1SK: "2025-03-15#order-456"
```

**Modelo de Custo:**

| Modo | Quando Usar | Custo Aproximado |
|------|-------------|-----------------|
| On-Demand | Trafico imprevisivel, startups | $0.25/1M reads, $1.25/1M writes |
| Provisioned + Auto Scaling | Trafico previsivel | Ate 77% mais barato com reserved capacity |
| Database Savings Plans (2025) | Comprometimento de 1-3 anos | 12-18% desconto cross-mode |

**Novidades 2025-2026:**
- Multi-attribute composite keys em GSIs (ate 4 atributos por PK/SK, sem concatenacao manual)
- Database Savings Plans (dezembro 2025)

**Armadilhas de Custo:**
- GSIs sao o **#1 multiplicador de custo oculto** -- cada GSI duplica dados e consome write capacity
- Projection ALL em GSIs infla storage desnecessariamente
- On-demand pode ficar caro em workloads constantes de alto volume

**Quando Escolher DynamoDB:**
- Ja esta na AWS e precisa de integracoes nativas (Lambda, API Gateway, EventBridge)
- Latencia single-digit millisecond e requisito hard
- Access patterns sao bem definidos e nao mudam frequentemente
- Escala previsivel de milhoes a bilhoes de items

**Quando NAO Escolher:**
- Queries ad-hoc sao frequentes (SQL e muito mais flexivel)
- Access patterns mudam constantemente
- Precisa de JOINs complexos ou aggregations
- Equipe nao tem experiencia com modelagem NoSQL

---

### 2.3 Cassandra / ScyllaDB

Para cenarios de **extreme write throughput** distribuido globalmente.

| Aspecto | Cassandra | ScyllaDB |
|---------|-----------|----------|
| Linguagem | Java | C++20 (Seastar framework) |
| Arquitetura | Thread-per-core (JVM) | Shared-nothing, shard-per-core |
| Throughput | Baseline | 2-5x maior (benchmarks independentes; ate 10x em testes internos) |
| Latencia | Mais variavel (GC pauses da JVM) | Mais consistente e menor (sem GC) |
| Memoria | 25-30% mais uso | 25-30% mais eficiente |
| CQL | Nativo | Compativel |
| Ecossistema | Maior comunidade, mais ferramentas | Crescente, compativel com tools Cassandra |

**ScyllaDB** alcanca 6.43 milhoes de operacoes/segundo em uma unica instancia c7gn.16xlarge (Graviton 3E). Em pipeline mode, chega a 10M QPS (SET) e 15M QPS (GET).

**Quando Usar Wide-Column:**
- Write throughput extremo (IoT sensors, event logging, time-series de alto volume)
- Distribuicao multi-datacenter com eventual consistency aceitavel
- Dados imutaveis ou append-only (logs, eventos, metricas)
- Escala horizontal para petabytes

**Quando NAO Usar:**
- Queries complexas com JOINs
- Transacoes ACID cross-partition
- Dados que precisam de atualizacao frequente (update-heavy)
- Projetos pequenos (overhead operacional nao se justifica)

---

### 2.4 Firestore vs Supabase Realtime

| Aspecto | Firestore | Supabase Realtime |
|---------|-----------|-------------------|
| Banco subjacente | NoSQL (documentos) | PostgreSQL |
| Real-time | Nativo, built from the ground up | PostgreSQL LISTEN/NOTIFY + WebSockets |
| Offline support | Maduro (iOS/Android SDKs com cache local) | Em evolucao, ainda nao equivalente |
| Query model | Hierarquia de collections/documents | SQL completo com filtros |
| Mobile SDKs | Maduros, battle-tested | Crescentes, menos nativos |
| Vendor lock-in | Alto (Google Cloud) | Baixo (PostgreSQL padrao, self-host possivel) |
| Precificacao | Pay-per-read/write | Planos fixos + compute |

**Firestore** mantem vantagem para apps **mobile-first, offline-capable** onde os SDKs maduros de iOS/Android fornecem capacidades que Supabase ainda nao equivale.

**Supabase Realtime** oferece SQL filters completos em live queries, dando controle mais preciso sobre o que e sincronizado em real-time.

**Tendencia 2026:** Convergencia -- Firebase lancou **Data Connect** (integracao com PostgreSQL) e **Firebase AI Logic** (Gemini). O gap entre as duas plataformas esta diminuindo.

**Recomendacao:**
- App mobile nativo com requisito de offline-first: **Firestore**
- Web app ou API com real-time: **Supabase Realtime**
- Precisa de SQL + real-time: **Supabase**
- Ja esta no Google Cloud com Flutter: **Firestore**

---

## 3. In-Memory / Cache

### 3.1 Redis

Redis e o padrao de facto para caching, pub/sub, filas de mensagens e estruturas de dados in-memory.

**Padroes de Caching:**

```javascript
// 1. Cache-Aside (mais comum)
async function getUser(userId) {
  const cached = await redis.get(`user:${userId}`);
  if (cached) return JSON.parse(cached);

  const user = await db.query('SELECT * FROM users WHERE id = $1', [userId]);
  await redis.set(`user:${userId}`, JSON.stringify(user), 'EX', 3600);
  return user;
}

// 2. Stale-While-Revalidate
async function getData(key) {
  const cached = await redis.get(key);
  if (cached) {
    // Retorna imediatamente, revalida em background
    refreshInBackground(key);
    return JSON.parse(cached);
  }
  return fetchAndCache(key);
}

// 3. Rate Limiting com sorted sets
async function rateLimit(userId, limit = 100, windowSecs = 60) {
  const now = Date.now();
  const key = `rate:${userId}`;

  await redis.multi()
    .zadd(key, now, `${now}`)
    .zremrangebyscore(key, 0, now - windowSecs * 1000)
    .expire(key, windowSecs)
    .exec();

  const count = await redis.zcard(key);
  return count <= limit;
}
```

**Estruturas de Dados Alem de Key-Value:**

| Estrutura | Caso de Uso |
|-----------|-------------|
| Strings | Cache simples, contadores, flags |
| Hashes | Session data, objetos parciais |
| Lists | Filas de tarefas, feeds recentes |
| Sets | Tags, membros unicos, intersecoes |
| Sorted Sets | Leaderboards, rate limiting, scheduling |
| Streams | Event sourcing, logs em real-time, message queues |
| Pub/Sub | Notificacoes, real-time updates, coordenacao entre servicos |
| HyperLogLog | Contagem de unicos aproximada (cardinalidade) |

**Redis Streams** sao particularmente poderosos para event sourcing e message queues com consumer groups, oferecendo persistencia (diferente de Pub/Sub que e fire-and-forget).

#### Upstash (Serverless Redis)

Upstash resolve o desafio de usar Redis em ambientes serverless (Vercel Edge Functions, Cloudflare Workers) com uma HTTP REST API e o SDK `@upstash/redis`.

```typescript
import { Redis } from '@upstash/redis';

const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL,
  token: process.env.UPSTASH_REDIS_REST_TOKEN,
});

// Funciona em Edge Functions (sem TCP connection necessaria)
await redis.set('session:abc', { userId: 123 }, { ex: 3600 });
const session = await redis.get('session:abc');
```

**Pricing Upstash (2025):**
- Free tier: 500K commands/mes, 200GB bandwidth
- Pay-as-you-go: $0.2 por 100K commands
- Fixed plans disponiveis desde 2025

**Quando usar Upstash:** Edge functions, serverless APIs com trafico variavel, rate limiting, session data, feature flags.

---

### 3.2 Memcached

Memcached e a escolha quando **simplicidade maxima** e o requisito.

| Aspecto | Memcached | Redis |
|---------|-----------|-------|
| Modelo de dados | Apenas key-value (strings) | 10+ estruturas de dados |
| Threading | Multi-threaded nativo | Predominantemente single-threaded |
| Persistencia | Nenhuma (cache puro) | RDB + AOF |
| Pub/Sub | Nao | Sim |
| Overhead de memoria | Menor | Maior (metadata por key) |
| Caso de uso ideal | Cache puro de HTML, API responses, sessions | Cache + filas + pub/sub + analytics |

**Quando Memcached e a Melhor Escolha:**
- Cache puro de key-value strings (fragments HTML, API responses)
- Ambiente com multiplos cores onde multi-threading nativo e vantajoso
- Orcamento de memoria limitado (menor overhead por chave)
- Nao precisa de persistencia, pub/sub ou estruturas complexas

**Para a maioria dos projetos modernos, Redis oferece o melhor balanco** de performance, features e flexibilidade. Memcached se justifica em cenarios de cache puro em escala onde multi-threading e memoria eficiente sao criticos.

---

### 3.3 Dragonfly

Dragonfly e um drop-in replacement para Redis construido do zero em C++ com arquitetura multi-threaded shared-nothing.

**Numeros de Performance:**

| Metrica | Redis | Dragonfly |
|---------|-------|-----------|
| Throughput | ~1M QPS | 6.43M QPS (single instance) |
| Pipeline mode | ~3M QPS | 10M SET / 15M GET QPS |
| Memoria | Baseline | 25-30% menor uso |
| Custo de infra | Baseline | Ate 80% reducao |
| Compatibilidade | -- | 100% Redis + Valkey + Memcached APIs |

**Quando Considerar Dragonfly:**
- Workload Redis que exige escalar para milhoes de QPS
- Quer reduzir numero de nodes (um Dragonfly substitui multiplos Redis)
- Compatibilidade total e requisito (API drop-in)
- Eficiencia de memoria e custo sao prioridades

**Riscos:**
- Ecossistema menor que Redis (menos ferramentas de monitoramento/backup)
- Comunidade e tooling ainda em crescimento
- Redis/Valkey tem decadas de battle-testing em producao

---

## 4. Search Engines

### 4.1 Comparativo: Elasticsearch vs OpenSearch vs Meilisearch vs Typesense

| Aspecto | Elasticsearch | OpenSearch | Meilisearch | Typesense |
|---------|--------------|------------|-------------|-----------|
| Licenca | Elastic License 2.0 / AGPL | Apache 2.0 | MIT | GPL-3 |
| Linguagem | Java | Java (fork do ES) | Rust | C++ |
| RAM minima | 8GB+ | 8GB+ | 2GB+ (sub-1GB baseline) | 1GB+ |
| Setup | Complexo (shards, JVM, cluster) | Similar ao ES | Simples (1 binario) | Simples |
| Typo tolerance | Via fuzzy queries | Via fuzzy queries | Nativo, automatico | Nativo, automatico |
| Faceted search | Aggregations (poderoso) | Aggregations | Nativo | Nativo |
| Escala | Petabytes, multi-cluster | Petabytes | Ate ~10M docs eficiente | Replicated cluster (Raft) |
| Operacao | Equipe dedicada recomendada | Similar ao ES | Minimal ops | Minimal ops |
| Vendor lock-in | Elastic Cloud | AWS OpenSearch | Self-host ou Cloud | Self-host ou Cloud |

**Quando Usar Cada Um:**

| Cenario | Recomendacao |
|---------|-------------|
| Logs + observabilidade enterprise | Elasticsearch ou OpenSearch |
| Busca e-commerce com facets complexos | Elasticsearch |
| Evitar vendor lock-in, feature parity com ES | OpenSearch |
| Busca instantanea para developers (setup rapido) | Meilisearch |
| Catalogo de produtos, site search (< 10M docs) | Meilisearch ou Typesense |
| High-availability sem sharding | Typesense |

#### Algolia (Managed Search)

Algolia e a solucao managed de busca com foco em developer experience e search relevance.

**Modelo de Pricing:**

| Plano | Custo | Caracteristicas |
|-------|-------|-----------------|
| Grow | Pay-as-you-go, sem base mensal | Busca basica |
| Grow Plus | $0.75 / 1K searches | AI features (NeuralSearch) |
| Premium | Contrato anual, custom | SLAs enterprise |
| Elevate | Contrato anual, custom | Full AI suite |

Add-ons: AI recommendations ($0.60/1K requests), crawler ($0.80/1K requests).

**Programa Startup:** $10,000 em creditos gratuitos.

**Quando Algolia vale o custo:** busca mission-critical onde relevance e velocidade justificam o preco premium, e a equipe nao quer operar infraestrutura de busca.

---

### 4.2 PostgreSQL Full-Text Search vs Search Engine Dedicado

**Casos reais de PostgreSQL FTS substituindo Elasticsearch:**
- GitHub migrou busca de Elasticsearch para PostgreSQL
- Reddit reduziu infraestrutura de busca de 6 servicos para 1
- Discord eliminou cluster Elasticsearch depois que PostgreSQL provou ser 3x mais rapido
- GitLab consolidou tudo no PostgreSQL

```sql
-- PostgreSQL Full-Text Search basico
SELECT title, ts_rank(search_vector, query) AS rank
FROM articles,
  to_tsquery('portuguese', 'banco & dados') AS query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 20;

-- Criar indice GIN para performance
CREATE INDEX idx_articles_search ON articles USING gin(search_vector);
```

**Quando PostgreSQL FTS e Suficiente:**

| Criterio | PostgreSQL FTS | Search Engine Dedicado |
|----------|---------------|----------------------|
| Ate ~1M documentos | Excelente | Overkill |
| Typo tolerance | Limitado (trigram) | Nativo |
| Busca "lambo" → "Lamborghini" | Nao funciona nativamente | Funciona |
| Facets/aggregations complexos | Limitado | Nativo |
| Consistencia transacional | ACID garantido | Eventualmente consistente |
| Ops overhead | Zero (mesmo banco) | Cluster adicional |
| Ranking sofisticado | Basico (ts_rank) | BM25, learning to rank |

**Extensao pg_search** implementa BM25 (mesmo algoritmo do Elasticsearch) dentro do PostgreSQL, preenchendo parte do gap.

**Regra pratica:** Comece com PostgreSQL FTS. Migre para search engine dedicado quando typo tolerance, ranking sofisticado ou facets complexos se tornam requisitos criticos.

---

## 5. Vector Databases (AI/ML)

### 5.1 Comparativo de Vector Databases

| Database | Tipo | Linguagem | Max Vetores | Latencia (p95, 10M vecs) | Custo Mensal |
|----------|------|-----------|-------------|--------------------------|--------------|
| **pgvector** | Extension PostgreSQL | C | ~100M (pratico) | Variavel | $0 incremental (ou $30-80 instancia dedicada) |
| **Pinecone** | Managed serverless | Proprietario | Bilhoes | ~45ms | $70-300+ (5M+ vecs: $500-1500) |
| **Weaviate** | Open-source | Go | Bilhoes | ~30ms | Self-host ou managed |
| **Qdrant** | Open-source | Rust | Bilhoes | ~22ms | Self-host ou managed |
| **Chroma** | Open-source | Python | Milhoes | ~50ms | Self-host (lightweight) |

**pgvector vs Databases Dedicados:**

```sql
-- pgvector: busca vetorial dentro do PostgreSQL
SELECT id, content,
  1 - (embedding <=> query_embedding) AS similarity
FROM documents
ORDER BY embedding <=> query_embedding
LIMIT 5;

-- Vantagem: JOIN com dados relacionais na mesma query
SELECT d.content, u.name, d.created_at
FROM documents d
JOIN users u ON d.author_id = u.id
ORDER BY d.embedding <=> $1
LIMIT 5;
```

**Quando pgvector e Suficiente:**
- Ate ~10-100M vetores
- Ja usa PostgreSQL (zero infra adicional)
- Precisa de consistencia transacional (embedding + metadata atomico)
- Queries combinam similarity search + filtros SQL
- Equipe pequena sem capacity para operar infra adicional

**Quando Migrar para Database Dedicado:**
- Mais de 100M vetores
- Latencia sub-10ms e requisito hard
- Volume de queries de similarity search e muito alto
- Precisa de features avancadas (hybrid search, multi-tenancy nativo)

**pgvectorscale** (Timescale) melhora significativamente a performance do pgvector: 471 QPS a 99% recall vs 41 QPS do Qdrant a 50M vetores em certos benchmarks.

---

### 5.2 Padroes de Arquitetura RAG

Em 2026, RAG evoluiu de tecnica experimental para **arquitetura critica de producao**. 72% das empresas rodam RAG pipelines em producao (vs 8% em Q1 2024).

**Padroes RAG em 2026:**

| Padrao | Complexidade | Quando Usar |
|--------|-------------|-------------|
| **Naive RAG** | Baixa | Prototipos, Q&A simples |
| **Hybrid RAG** | Media | Producao (baseline 2026 -- 85% das empresas) |
| **Graph RAG** | Alta | Multi-hop reasoning, knowledge bases complexas |
| **Agentic RAG** | Alta | Routing dinamico entre ferramentas/databases |

```python
# Naive RAG simplificado
from openai import OpenAI

# 1. Chunking e embedding dos documentos
chunks = split_document(document, chunk_size=512)
embeddings = embed(chunks)  # text-embedding-3-small

# 2. Armazenar no vector database
for chunk, embedding in zip(chunks, embeddings):
    db.execute("""
        INSERT INTO documents (content, embedding)
        VALUES (%s, %s)
    """, [chunk, embedding])

# 3. Query: buscar contexto relevante
query_embedding = embed([user_question])[0]
context = db.execute("""
    SELECT content FROM documents
    ORDER BY embedding <=> %s
    LIMIT 5
""", [query_embedding])

# 4. Gerar resposta com contexto
response = openai.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": f"Contexto:\n{context}"},
        {"role": "user", "content": user_question}
    ]
)
```

**Tecnicas Avancadas (2026):**
- **Context-aware chunking:** usar modelo para detectar limites semanticos (nao cortar em tamanho fixo)
- **Hybrid search:** combinar BM25 (keyword) + vector similarity
- **Re-ranking:** buscar top 100 candidatos, re-ranker seleciona top 5-10 para o LLM
- **Metadata filtering:** filtrar por data, autor, tipo antes da busca vetorial

---

## 6. Time-Series Databases

### 6.1 TimescaleDB vs InfluxDB vs QuestDB

| Aspecto | TimescaleDB | InfluxDB 3.0 | QuestDB |
|---------|-------------|--------------|---------|
| Arquitetura | Extension PostgreSQL | Engine reescrita em Rust | Engine dedicada (Java + C++ + Rust) |
| SQL | PostgreSQL completo | InfluxQL + SQL (limitado) | SQL + InfluxDB Line Protocol |
| Ingestao | Baseline | Competitivo | **12-36x mais rapido** que InfluxDB, 6-13x mais rapido que TimescaleDB |
| Queries complexas | Bom (PostgreSQL) | Limitado | **43-418x mais rapido** que InfluxDB, 16-20x mais rapido que TimescaleDB |
| Storage engine | Hypercore (row + columnar hibrido) | Apache Parquet | Open formats (Parquet, Iceberg roadmap) |
| Continuous aggregates | Nativo (materialized views) | Limitado no Core | Em desenvolvimento |
| Retencao | Configuravel | 72h no Core (limitacao!) | Configuravel |
| Integracao PostgreSQL | Nativa (e uma extension) | Nenhuma | PGWire protocol |

**Performance destacada do QuestDB:**
- Ingestao: 12-36x mais rapido que InfluxDB 3 Core
- Queries analiticas complexas: 43-418x mais rapido que InfluxDB 3 Core
- Novidade 2026: N-dimensional arrays (ideal para order books L2, vetores, embeddings)

```sql
-- TimescaleDB: criar hypertable (particionamento automatico por tempo)
CREATE TABLE metrics (
  time TIMESTAMPTZ NOT NULL,
  device_id TEXT,
  temperature DOUBLE PRECISION,
  humidity DOUBLE PRECISION
);

SELECT create_hypertable('metrics', 'time');

-- Continuous aggregate para dashboard
CREATE MATERIALIZED VIEW hourly_avg
WITH (timescaledb.continuous) AS
SELECT time_bucket('1 hour', time) AS bucket,
       device_id,
       AVG(temperature) AS avg_temp,
       MAX(temperature) AS max_temp
FROM metrics
GROUP BY bucket, device_id;
```

**Quando Usar Cada Um:**

| Cenario | Recomendacao |
|---------|-------------|
| Ja usa PostgreSQL, quer adicionar time-series | TimescaleDB |
| Ingestao de alto volume e prioridade #1 | QuestDB |
| Queries analiticas complexas | QuestDB |
| Ecossistema IoT com Telegraf/Grafana | InfluxDB ou QuestDB |
| Quer SQL padrao com time-series | TimescaleDB ou QuestDB |
| Precisa de open formats (Parquet, Iceberg) | QuestDB |

**InfluxDB 3.0 Core** tem limitacoes significativas: retencao de 72h, maximo 5 databases. A comunidade ficou decepcionada com essas restricoes no tier gratuito. O tier Enterprise remove essas limitacoes.

**TigerData** (novo nome da Timescale) expandiu o alcance do PostgreSQL para time-series, mantendo a vantagem de ser "apenas mais uma extension no seu PostgreSQL existente".

---

## 7. Graph Databases

### 7.1 Neo4j vs Amazon Neptune

| Aspecto | Neo4j | Amazon Neptune |
|---------|-------|----------------|
| Foco | Pure graph, especialista | Multi-model (property graph + RDF) |
| Query language | Cypher (alinhado com ISO GQL) | Gremlin, SPARQL, openCypher (subset) |
| Deployment | Cloud, on-premises, hibrido | AWS-only (fully managed) |
| Consistencia | ACID completo | Imediata no writer, eventual nos replicas |
| Performance | Superior em traversal puro | Boa, mas storage desacoplado adiciona latencia |
| Custo | Community Edition gratuita (GPLv3) | Pay-as-you-go (instancias + storage + I/O) |
| Ecossistema | 10+ anos, comunidade madura | Integracoes AWS (SageMaker, CloudWatch) |

**Quando Relational JOINs Nao Sao Suficientes:**

```cypher
// Neo4j Cypher: encontrar amigos de amigos que compraram produto similar
MATCH (me:User {id: 'user-123'})-[:FRIENDS_WITH*2]-(fof:User)
      -[:PURCHASED]->(p:Product)<-[:SIMILAR_TO]-(rec:Product)
WHERE NOT (me)-[:PURCHASED]->(rec)
RETURN rec.name, COUNT(fof) AS recommendation_score
ORDER BY recommendation_score DESC
LIMIT 10;

// Em SQL, isso seria 4+ JOINs com subqueries e performance degradando
// exponencialmente conforme a profundidade de relacao aumenta
```

**Casos de Uso para Graph Databases:**

| Caso de Uso | Por que Graph | Exemplo |
|-------------|--------------|---------|
| Redes sociais | Relacoes N:N com profundidade variavel | Amigos de amigos, conexoes |
| Recommendation engines | Traversal de relacoes comprador→produto→similar | Netflix, Amazon |
| Deteccao de fraude | Padroes de conexao entre entidades suspeitas | Transacoes bancarias |
| Knowledge graphs | Ontologias com relacoes semanticas | Wikis, bases de conhecimento |
| Gerenciamento de rede | Dependencias de infra (servidor→switch→rack) | Telecom, DevOps |
| Genealogia/compliance | Hierarquias com profundidade desconhecida | Cadeia societaria, AML |

**Recomendacao:**
- **Neo4j** para pure graph workloads, performance maxima de traversal, deployment flexivel
- **Neptune** para equipes na AWS que precisam de RDF/SPARQL ou querem managed service sem ops

**Quando NAO usar Graph Database:**
- Dados sao predominantemente tabulares com relacoes simples (1:N, N:M com 1-2 niveis)
- PostgreSQL com `RECURSIVE CTEs` resolve o problema
- Volume de dados e pequeno (< 1M nodes) -- PostgreSQL com JOINs funciona bem
- Equipe nao tem experiencia com modelagem de grafos

---

## 8. Framework de Decisao

### 8.1 Regra "Start with PostgreSQL"

**Por que PostgreSQL cobre ~90% dos casos:**

| Necessidade | PostgreSQL Resolve? | Como |
|-------------|--------------------|------|
| Dados relacionais | Sim | SQL padrao, ACID |
| Dados semi-estruturados | Sim | JSONB com indices GIN |
| Full-text search | Sim | tsvector + GIN + pg_search |
| Geoespacial | Sim | PostGIS |
| Time-series | Sim | TimescaleDB extension |
| Vector/embeddings (AI) | Sim | pgvector / pgvectorscale |
| Cache simples | Parcial | UNLOGGED tables (nao substitui Redis) |
| Pub/Sub basico | Sim | LISTEN/NOTIFY |
| Job scheduling | Sim | pg_cron |
| Graph queries simples | Parcial | RECURSIVE CTEs (nao substitui Neo4j para deep traversal) |

**A sequencia recomendada:**

```
1. Comece com PostgreSQL para TUDO
2. Adicione Redis quando precisar de cache sub-millisecond
3. Adicione search engine quando precisar de typo tolerance / facets complexos
4. Adicione vector database dedicado quando ultrapassar 100M embeddings
5. Adicione time-series dedicado quando ingestao > 1M pontos/segundo
6. Adicione graph database quando traversal > 3 niveis de profundidade
7. Adicione NoSQL (Mongo/Dynamo) quando schema flexibility E escala horizontal sao ambos criticos
```

---

### 8.2 Matriz de Decisao Completa

| Criterio | PostgreSQL | MySQL | MongoDB | DynamoDB | Redis | Elasticsearch | Neo4j |
|----------|-----------|-------|---------|----------|-------|---------------|-------|
| **Modelo de dados** | Relacional + JSON | Relacional | Documentos | Key-Value + Documents | Key-Value + Structures | Documentos (search) | Grafos |
| **Consistencia** | ACID (serializable) | ACID | Eventual (tunable) | Eventual (tunable) | Eventual | Eventual | ACID |
| **Escala horizontal** | Limitada (Citus, logical rep) | Vitess/PlanetScale | Nativo (sharding) | Nativo (ilimitado) | Cluster (limitado) | Nativo (shards) | Limitada |
| **Latencia tipica** | 1-10ms | 1-10ms | 1-10ms | 1-5ms | <1ms | 10-100ms | 1-50ms |
| **Queries ad-hoc** | Excelente | Bom | Bom (aggregation) | Ruim | N/A | Excelente (search) | Bom (Cypher) |
| **Developer experience** | Excelente | Boa | Excelente | Curva ingreme | Excelente | Complexo | Boa |
| **Custo (entrada)** | Free (self-host) | Free (self-host) | Free (self-host) / $57/mes Atlas | Pay-per-use (~$0) | Free (self-host) | Free (self-host) | Free (Community) |
| **Managed services** | Supabase, Neon, RDS | PlanetScale, RDS | Atlas | AWS nativo | Upstash, ElastiCache | Elastic Cloud | Aura |

---

### 8.3 Quando Adicionar Databases Especializados

| Sinal | Database a Adicionar | Alternativa PostgreSQL |
|-------|---------------------|----------------------|
| Cache miss rate > 20%, latencia p99 > 100ms | Redis | UNLOGGED tables (limitado) |
| Usuarios reclamam de busca (typos, relevancia) | Meilisearch/Typesense | pg_search + trigram |
| RAG pipeline com > 50M embeddings | Qdrant/Pinecone | pgvector (ate ~100M) |
| Ingestao de metricas > 500K pontos/segundo | QuestDB/TimescaleDB | TimescaleDB extension |
| Relacoes com > 3 niveis de profundidade frequentes | Neo4j | RECURSIVE CTEs |
| Write throughput > 100K writes/segundo | Cassandra/ScyllaDB | Particionamento + replicas |
| Mobile offline-first com sync | Firestore | Supabase (limitado offline) |
| Single-digit ms com escala AWS nativa | DynamoDB | RDS/Aurora |

---

### 8.4 Padroes de Arquitetura Multi-Database (Polyglot Persistence)

**Exemplos do Mundo Real:**

| Empresa | Stack de Databases | Razao |
|---------|-------------------|-------|
| Netflix | Cassandra + Redis/Dynomite + MySQL + Elasticsearch | Global replication + cache + transactions + search |
| Uber | MySQL (sharded) + Redis + Cassandra | Transactions + geospatial cache + trip logs |
| Airbnb | MySQL + Redis + Elasticsearch + DynamoDB | Core data + cache + search + sessions |

**Padrao Recomendado para Startups:**

```
Estagio 1 (MVP): PostgreSQL (tudo)
Estagio 2 (Product-Market Fit): PostgreSQL + Redis
Estagio 3 (Escala): PostgreSQL + Redis + Search Engine
Estagio 4 (AI features): PostgreSQL + Redis + Search + Vector DB
Estagio 5 (Global): + CDN + Edge cache + Read replicas
```

**Desafios de Polyglot Persistence:**
- Consistencia entre bancos (distributed transactions sao complexas)
- Monitoramento e observabilidade multiplicados
- Equipe precisa de expertise em multiplas tecnologias
- Backup, restore e disaster recovery para cada banco
- Custo operacional cresce linearmente com cada banco adicional

---

### 8.5 Comparativo de Custo por Escala

| Escala | PostgreSQL (Supabase) | PostgreSQL (Neon) | MongoDB Atlas | DynamoDB On-Demand | Redis (Upstash) |
|--------|----------------------|-------------------|---------------|-------------------|-----------------|
| **10K records, baixo trafico** | Free tier | Free tier | Free tier (512MB) | ~$1/mes | Free tier |
| **100K records, moderado** | $25/mes (Pro) | ~$10/mes | $57/mes (M10) | ~$10-30/mes | ~$5/mes |
| **1M records, consistente** | $25-75/mes | ~$30-80/mes | $57-200/mes | ~$50-150/mes | ~$20/mes |
| **10M records, alto trafico** | $75-300/mes | ~$100-300/mes | $200-800/mes | ~$200-1000/mes | ~$50-200/mes |
| **100M records, enterprise** | $300-1500/mes | Custom | $800-5000/mes | ~$1000-10000/mes | ~$200-1000/mes |

> Valores aproximados baseados em precificacao publica de 2025-2026. Custos reais dependem de read/write patterns, storage, network egress e features utilizadas.

**Observacoes de custo:**
- DynamoDB parece barato ate adicionar GSIs e modo on-demand em alto volume
- MongoDB Atlas tem custo previsivel mas clusters maiores sao caros
- Neon e mais barato que Supabase para workloads intermitentes (scale-to-zero)
- Para trafico constante 24/7, Aurora pode ser mais previsivel que Neon
- Upstash e o mais barato para cache serverless em baixo-medio volume

---

### 8.6 Caminhos de Migracao

| De | Para | Dificuldade | Ferramentas | Notas |
|----|------|-------------|-------------|-------|
| MySQL → PostgreSQL | Media | pgloader, AWS DMS | Ajustar tipos de dados, syntax differences |
| MongoDB → PostgreSQL | Alta | Mongrel, custom ETL | Denormalize → normalize; repensar schema |
| PostgreSQL → CockroachDB | Baixa | pg_dump + import | Wire-compatible; ajustar features nao suportadas |
| Firestore → Supabase | Media | Custom migration scripts | NoSQL → SQL; redesign schema |
| Redis → Dragonfly | Muito Baixa | Trocar endpoint | API 100% compativel |
| Elasticsearch → Meilisearch | Media | Re-index via API | Simplificar mappings; perder features avancadas |
| SQLite → Turso/libSQL | Muito Baixa | Upload do arquivo .db | Fork compativel |
| PostgreSQL → Neon | Muito Baixa | pg_dump/restore ou logical replication | PostgreSQL padrao |
| pgvector → Pinecone | Media | Export embeddings + re-import | Perder JOINs com dados relacionais |
| DynamoDB → PostgreSQL | Alta | AWS DMS, custom ETL | Redesign completo de access patterns → schema relacional |

---

### 8.7 Checklist de Selecao de Banco de Dados

Use esta checklist para cada novo projeto:

```
[ ] 1. Quais sao os access patterns primarios? (CRUD, search, analytics, real-time)
[ ] 2. Qual o volume esperado? (registros, reads/sec, writes/sec)
[ ] 3. Consistencia e critica? (financeiro = sim, analytics = nao)
[ ] 4. Precisa de escala horizontal? (multi-region, sharding)
[ ] 5. Schema e estavel ou evolui frequentemente?
[ ] 6. Latencia maxima aceitavel? (sub-ms, <10ms, <100ms)
[ ] 7. Equipe tem experiencia com qual tecnologia?
[ ] 8. Orcamento para managed services?
[ ] 9. Requisitos regulatorios? (LGPD, GDPR, data locality)
[ ] 10. Precisa de offline-first? (mobile apps)
```

**Se respondeu "nao sei" para a maioria: comece com PostgreSQL.**

---

## Fontes

### Bancos Relacionais
- [PostgreSQL vs MySQL: Which Database Should You Choose in 2026? -- Bytebase](https://www.bytebase.com/blog/postgres-vs-mysql/)
- [PostgreSQL vs MySQL in 2026 -- DEV Community](https://dev.to/philip_mcclarence_2ef9475/postgresql-vs-mysql-in-2026-performance-features-and-when-to-use-each-3g7e)
- [Best PostgreSQL Hosting in 2026 -- DEV Community](https://dev.to/philip_mcclarence_2ef9475/best-postgresql-hosting-in-2026-rds-vs-supabase-vs-neon-vs-self-hosted-5fkp)
- [Neon vs Supabase -- Bytebase](https://www.bytebase.com/blog/neon-vs-supabase/)
- [Supabase vs AWS RDS -- Leanware](https://www.leanware.co/insights/supabase-vs-aws-rds)
- [SQLite Renaissance -- DEV Community](https://dev.to/pockit_tools/the-sqlite-renaissance-why-the-worlds-most-deployed-database-is-taking-over-production-in-2026-3jcc)
- [Distributed SQLite: LibSQL and Turso -- DEV Community](https://dev.to/dataformathub/distributed-sqlite-why-libsql-and-turso-are-the-new-standard-in-2026-58fk)
- [Post-PostgreSQL: Is SQLite on the Edge Production Ready? -- SitePoint](https://www.sitepoint.com/sqlite-edge-production-readiness-2026/)
- [CockroachDB Momentum 2026 -- PR Newswire](https://www.prnewswire.com/news-releases/cockroach-labs-accelerates-momentum-into-2026-as-enterprises-rebuild-for-ai-scale-resilience-302660764.html)
- [PlanetScale -- Vitess](https://planetscale.com/vitess)
- [PlanetScale Extends to PostgreSQL -- InfoQ](https://www.infoq.com/news/2025/10/planetscale-metal-postgres/)

### Extensions PostgreSQL
- [7 PostgreSQL Extensions 2026 -- DEV Community](https://dev.to/finny_collins/7-postgresql-extensions-that-will-supercharge-your-database-in-2026-1ab6)
- [Top 12 PostgreSQL Extensions -- SQLFlash](https://sqlflash.ai/article/20250918_top_12_postgresql_extensions/)
- [pgvector Guide -- Tiger Data](https://www.tigerdata.com/learn/postgresql-extensions-pgvector)
- [PostgreSQL Extensions 2025 -- Aiven](https://aiven.io/blog/postgresql-extensions-you-need-to-know)

### NoSQL
- [MongoDB Schema Design Anti-Patterns -- MongoDB Docs](https://www.mongodb.com/docs/manual/data-modeling/design-antipatterns/)
- [MongoDB in 2025-2026 -- Calmops](https://calmops.com/database/mongodb/mongodb-trends-2025-2026/)
- [DynamoDB Database Design in 2026 -- SimpleAWS](https://newsletter.simpleaws.dev/p/dynamodb-database-design-in-2026)
- [DynamoDB Pricing -- Cloud Burn](https://cloudburn.io/blog/amazon-dynamodb-pricing)
- [ScyllaDB vs Cassandra -- ScyllaDB](https://www.scylladb.com/compare/scylladb-vs-apache-cassandra/)
- [ScyllaDB: Not Just a Faster Cassandra -- ScyllaDB](https://www.scylladb.com/2025/09/02/beyond-apache-cassandra/)
- [Firebase vs Supabase 2026 -- Pockit](https://pockit.tools/blog/supabase-vs-firebase-2026-comparison/)
- [Firebase vs Supabase Realtime -- Ably](https://ably.com/compare/firebase-vs-supabase)

### In-Memory / Cache
- [Upstash vs Redis Cloud 2026 -- BuildMVPFast](https://www.buildmvpfast.com/compare/upstash-vs-redis-cloud)
- [Redis vs Dragonfly -- OneUptime](https://oneuptime.com/blog/post/2026-01-21-redis-vs-dragonfly/view)
- [Dragonfly vs Redis -- DragonflyDB](https://www.dragonflydb.io/dragonfly-vs-redis)
- [Redis vs Memcached -- ScaleGrid](https://scalegrid.io/blog/redis-vs-memcached/)
- [Redis vs Memcached -- AWS](https://aws.amazon.com/elasticache/redis-vs-memcached/)

### Search Engines
- [Elasticsearch Alternatives 2026 -- Meilisearch](https://www.meilisearch.com/blog/elasticsearch-alternatives)
- [Typesense vs Algolia vs Elasticsearch vs Meilisearch -- Typesense](https://typesense.org/typesense-vs-algolia-vs-elasticsearch-vs-meilisearch/)
- [Algolia Pricing -- Algolia](https://www.algolia.com/pricing)
- [PostgreSQL FTS vs Dedicated Search -- Nomadz](https://nomadz.pl/en/blog/postgres-full-text-search-or-meilisearch-vs-typesense)
- [Postgres FTS vs Elasticsearch -- Xata](https://xata.io/blog/postgres-full-text-search-postgres-vs-elasticsearch)
- [Postgres FTS vs the rest -- Supabase](https://supabase.com/blog/postgres-full-text-search-vs-the-rest)

### Vector Databases
- [Best Vector Databases 2026 -- Firecrawl](https://www.firecrawl.dev/blog/best-vector-databases)
- [Vector DB Comparison 2026 -- 4xxi](https://4xxi.com/articles/vector-database-comparison/)
- [Vector Databases for RAG -- Cloudmagazin](https://www.cloudmagazin.com/en/2026/04/02/vector-databases-rag-pinecone-weaviate-qdrant-pgvector-comparison/)
- [RAG Architecture Patterns -- Calmops](https://calmops.com/architecture/rag-architecture-retrieval-augmented-generation/)
- [10 RAG Architectures 2026 -- Techment](https://www.techment.com/blogs/rag-architectures-enterprise-use-cases-2026/)

### Time-Series
- [TimescaleDB vs InfluxDB vs QuestDB -- QuestDB](https://questdb.com/blog/comparing-influxdb-timescaledb-questdb-time-series-databases/)
- [TimescaleDB vs QuestDB Benchmark 2026 -- QuestDB](https://questdb.com/blog/timescaledb-vs-questdb-comparison/)
- [Best Time-Series Databases 2026 -- Tiger Data](https://www.tigerdata.com/learn/the-best-time-series-databases-compared)

### Graph Databases
- [AWS Neptune vs Neo4j -- PuppyGraph](https://www.puppygraph.com/blog/aws-neptune-vs-neo4j)
- [Neo4j vs Neptune -- Brilworks](https://www.brilworks.com/blog/neptune-vs-neo4j/)

### Arquitetura Multi-Database
- [Polyglot Persistence -- Martin Fowler](https://martinfowler.com/bliki/PolyglotPersistence.html)
- [Polyglot Persistence Revolution -- Medium](https://medium.com/@vitinhotoinho/the-polyglot-persistence-revolution-high-scale-data-architecture-in-real-scale-systems-7cb87d93a25a)
- [When to Use PostgreSQL vs MongoDB vs DynamoDB -- Ajit Singh](https://singhajit.com/postgresql-vs-mongodb-vs-dynamodb/)
- [Why PostgreSQL Top Choice 2025 -- Yugabyte](https://www.yugabyte.com/blog/postgresql-top-choice-in-2025/)

---

> **Pesquisa conduzida por:** Prism (Research Orchestrator) -- squad-research
> **Metodo:** DEEP DIVE (Research Depth Pyramid Level 3)
> **Fontes:** 40+ fontes verificadas via WebSearch, Tier 3-4 (Industry + Market)
> **Data:** Abril 2026
