# Supabase at Scale — Production Patterns, Advanced Configurations & Enterprise Usage

> **Tipo:** Deep Research | **Nivel:** DEEP DIVE  
> **Data:** 2026-04-11 | **Autor:** Prism (research-orqx)  
> **Fontes:** 40+ fontes verificadas via WebSearch  
> **Idioma:** Portugues (termos tecnicos em ingles)

---

## Indice

1. [Supabase Production Architecture](#1-supabase-production-architecture)
2. [Database Scaling](#2-database-scaling)
3. [RLS Advanced](#3-rls-advanced)
4. [Multi-Tenant Architecture](#4-multi-tenant-architecture)
5. [Supabase + Vercel Integration](#5-supabase--vercel-integration)
6. [Supabase Branching & Environments](#6-supabase-branching--environments)
7. [Supabase Security Hardening](#7-supabase-security-hardening)
8. [Supabase vs Alternativas](#8-supabase-vs-alternativas)

---

## 1. Supabase Production Architecture

### 1.1 Visao Geral da Arquitetura

O Supabase e uma plataforma open-source que combina PostgreSQL, Auth, Storage, Realtime e Edge Functions em um unico backend-as-a-service. Em producao, cada componente opera de forma independente e escala separadamente.

**Stack de producao:**

| Componente | Tecnologia | Funcao |
|-----------|-----------|--------|
| Database | PostgreSQL 15+ | Banco relacional primario |
| Connection Pooling | Supavisor (Elixir) | Pool de conexoes cloud-native |
| Auth | GoTrue (Go) | Autenticacao e autorizacao |
| Realtime | Elixir/Phoenix | WebSockets, Broadcast, Presence |
| Storage | S3-compatible | Armazenamento de arquivos + CDN |
| Edge Functions | Deno Deploy | Funcoes serverless na edge |
| API Gateway | PostgREST | REST API automatica do schema |
| Dashboard | Next.js | Interface de gerenciamento |

Fonte: [Supabase Architecture](https://supabase.com/docs/guides/functions/architecture)

### 1.2 Connection Pooling com Supavisor

O Supavisor e o connection pooler cloud-native do Supabase, construido em Elixir, capaz de gerenciar milhoes de conexoes de clientes para pools de conexoes PostgreSQL nativas.

**Arquitetura do Supavisor:**

- Projetado para operar como um cluster de alta disponibilidade
- Configuracao de tenants armazenada em um PostgreSQL separado de alta disponibilidade
- Quando um client se conecta, o tenant pool e iniciado e todas as conexoes com o database sao estabelecidas
- O PID do processo e distribuido para todos os nos do cluster e armazenado em um key-value store in-memory
- Em caso de queda de no, pool processes sao monitorados e clientes reconectam automaticamente

**Modos de conexao (pos-fevereiro 2025):**

| Modo | Porta | Uso | Prepared Statements | Ideal Para |
|------|-------|-----|---------------------|------------|
| Transaction Mode | 6543 | Pool compartilhado entre clients | Nao suporta | Serverless, APIs, Edge Functions |
| Session Mode | 5432 | Conexao exclusiva por client | Suporta | ORMs, long-lived connections |

Fonte: [Supabase Connection Management](https://supabase.com/docs/guides/database/connection-management)

**Mudanca critica (28/02/2025):** O Supavisor descontinuou o Session Mode na porta 6543. Apos esta data, a porta 6543 suporta apenas Transaction Mode, enquanto a porta 5432 continua com Session Mode.

**Configuracao recomendada para producao:**

```sql
-- Verificar pool size atual
SHOW max_connections;

-- Pool size padrao no Supavisor: 15
-- Se definir pool_size = 30, o Supavisor pode abrir ate 30 conexoes server-side
-- Essas conexoes sao compartilhadas entre Session Mode (5432) e Transaction Mode (6543)
```

**Quando usar cada modo:**

| Cenario | Modo Recomendado | Motivo |
|---------|-----------------|--------|
| Next.js API Routes | Transaction | Conexoes curtas, alto throughput |
| Edge Functions | Transaction | Conexoes efemeras |
| ORM (Prisma, Drizzle) | Session (se usar prepared statements) | Compatibilidade com prepared statements |
| Migrations | Direct (sem pooler) | DDL requer conexao direta |
| Dashboard queries | Transaction | Consultas ad-hoc |

Fonte: [Supavisor FAQ](https://supabase.com/docs/guides/troubleshooting/supavisor-faq-YyP5tI)

### 1.3 Edge Functions

Edge Functions do Supabase sao funcoes TypeScript executadas globalmente na edge usando Deno Deploy.

**Arquitetura de execucao:**

1. O CLI empacota a funcao e dependencias em um arquivo ESZip (formato compacto do Deno)
2. O bundle e enviado para o backend do Supabase
3. Uma URL unica e gerada para a funcao
4. Multiplos isolates podem rodar simultaneamente no mesmo edge location

**Limites de producao:**

| Parametro | Free | Pro |
|----------|------|-----|
| CPU Time por request | 2s | 2s |
| Request idle timeout | 150s | 150s |
| Background tasks (apos resposta) | 150s | 400s |
| Invocacoes/mes | 500.000 | 2.000.000 |
| Memory por isolate | 256 MB | 256 MB |

Fonte: [Edge Functions Limits](https://supabase.com/docs/guides/functions/limits)

**Performance de cold start:**

- Execucoes iniciais sao rapidas (milissegundos) gracas ao formato ESZip compacto
- Isolates permanecem ativos por um periodo (dependente do plano) para atender requests subsequentes
- Storage S3-compatible pode ser montado como persistent file storage com ate **97% de reducao no cold start**

Fonte: [Persistent Storage for Edge Functions](https://supabase.com/blog/persistent-storage-for-faster-edge-functions)

### 1.4 Realtime Scaling

O Supabase Realtime oferece tres features principais: Broadcast, Presence e Postgres Changes, todos via WebSockets.

**Benchmarks oficiais (cluster AWS, 2-6 nos):**

Os benchmarks sao conduzidos usando k6 contra um Realtime Cluster deployado na AWS. Metricas coletadas incluem message throughput, latency percentiles, CPU/memory utilization e connection success rates.

**Limites configuraveis:**

| Parametro | Configuravel | Padrao |
|----------|-------------|--------|
| Max concurrent clients | Sim | Depende do plano |
| Max events per second | Sim | Depende do plano |
| Max channels por conexao | Sim | Limite por plano |
| Max joins por segundo | Sim | Limite por plano |
| Message byte size | Nao | 1 MB |
| Presence calls por client | Sim | 5 por 30s window |

Fonte: [Realtime Limits](https://supabase.com/docs/guides/realtime/limits)

**Consideracoes de performance para Postgres Changes:**

- Cada evento de mudanca precisa ser verificado contra as permissoes do usuario subscrito
- Para uso em escala, considere usar uma tabela "publica" separada sem RLS e com filtros
- Use Broadcast direto para cenarios de alta frequencia (chat, gaming)

Fonte: [Realtime Benchmarks](https://supabase.com/docs/guides/realtime/benchmarks)

### 1.5 Storage CDN

O Supabase Storage e um object store S3-compatible com CDN global.

**Capacidades em producao:**

| Feature | Detalhes |
|---------|----------|
| CDN Global | 285+ cidades, reducao de latencia |
| S3 Compatibility | AWS Signature Version 4, qualquer S3 client |
| Image Transformations | On-the-fly via imgproxy |
| Smart CDN Caching | Cache de imagens transformadas |
| Signed URLs | Acesso privado com tempo de expiracao |
| RLS em Storage | Policies no nivel do bucket/objeto |

Fonte: [Supabase Storage](https://supabase.com/docs/guides/storage), [S3 Compatibility](https://supabase.com/docs/guides/storage/s3/compatibility)

**Exemplo de image transformation:**

```javascript
const { data } = supabase.storage
  .from('avatars')
  .getPublicUrl('profile.jpg', {
    transform: {
      width: 200,
      height: 200,
      resize: 'cover',
      format: 'webp',
      quality: 80
    }
  });
```

---

## 2. Database Scaling

### 2.1 Read Replicas

Read Replicas sao databases adicionais mantidas em sincronia com o Primary database via replicacao assincrona.

**Mudanca critica (04/04/2025):** O routing de requests Data API mudou de Round-Robin para **Geo-routing**, direcionando requests para o database mais proximo.

**Quando usar Read Replicas vs Compute maior:**

| Cenario | Solucao | Motivo |
|---------|---------|--------|
| Alto volume de leitura | Read Replicas | Distribui carga de SELECT |
| Queries complexas pesadas | Compute maior | CPU/RAM para queries individuais |
| Usuarios globais | Read Replicas em multiplas regioes | Reduz latencia |
| Writes pesados | Compute maior | Replicas nao aceitam writes |
| Analytics/reporting | Read Replica dedicada | Isola carga analitica |

Fonte: [Read Replicas](https://supabase.com/docs/guides/platform/read-replicas), [Read Replicas vs Bigger Compute](https://supabase.com/blog/read-replicas-vs-bigger-compute)

**Configuracao com Supabase client:**

```typescript
import { createClient } from '@supabase/supabase-js';

// Client principal (leitura + escrita)
const supabase = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_ANON_KEY!
);

// Para Read Replica, use o endpoint especifico
// O Geo-routing automatico direciona para a replica mais proxima
// Cada Read Replica tem seu proprio endpoint de API
```

### 2.2 Table Partitioning

Particionamento divide tabelas grandes em segmentos menores para melhor performance.

**Estrategias de particionamento no PostgreSQL:**

| Tipo | Uso | Exemplo |
|------|-----|---------|
| Range | Dados temporais | Logs por mes, orders por ano |
| List | Categorias discretas | Dados por regiao, por tenant |
| Hash | Distribuicao uniforme | Sharding por user_id |

**Exemplo pratico — particionamento por data:**

```sql
-- Tabela pai particionada por range
CREATE TABLE events (
    id          BIGINT GENERATED ALWAYS AS IDENTITY,
    tenant_id   UUID NOT NULL,
    event_type  TEXT NOT NULL,
    payload     JSONB,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- Particoes mensais
CREATE TABLE events_2025_01 PARTITION OF events
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

CREATE TABLE events_2025_02 PARTITION OF events
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');

-- Indice na particao (criado automaticamente nas particoes filhas)
CREATE INDEX idx_events_tenant_created
    ON events (tenant_id, created_at);
```

### 2.3 Index Strategies

Os indices sao a ferramenta mais poderosa para otimizacao de queries no PostgreSQL.

**Tipos de indice e quando usar:**

| Tipo | Caso de Uso | Operadores Suportados | Exemplo |
|------|------------|----------------------|---------|
| B-Tree | Igualdade, range, sorting | `=`, `<`, `>`, `BETWEEN`, `IN`, `ORDER BY` | `CREATE INDEX idx ON users (email)` |
| GIN | Arrays, JSONB, full-text | `@>`, `?`, `?&`, `@@` | `CREATE INDEX idx ON products USING gin (tags)` |
| GiST | Geometria, ranges, full-text | `<<`, `>>`, `&&`, `@>` | `CREATE INDEX idx ON locations USING gist (coordinates)` |
| BRIN | Dados naturalmente ordenados | `<`, `>`, `=` | `CREATE INDEX idx ON logs USING brin (created_at)` |
| Hash | Apenas igualdade | `=` | `CREATE INDEX idx ON sessions USING hash (token)` |

Fonte: [PostgreSQL Index Types](https://www.postgresql.org/docs/current/indexes-types.html)

**Indices avancados para producao:**

```sql
-- Partial Index: indexa apenas linhas relevantes
CREATE INDEX idx_active_users ON users (email)
    WHERE status = 'active';

-- Covering Index (INCLUDE): evita table lookup
CREATE INDEX idx_orders_lookup ON orders (user_id, created_at)
    INCLUDE (total, status);

-- Expression Index: indexa resultado de funcao
CREATE INDEX idx_users_lower_email ON users (LOWER(email));

-- Composite Index: ordena por multiplas colunas
CREATE INDEX idx_products_category_price ON products (category_id, price DESC);
```

**Regras de ouro para indices:**

1. Cada indice desacelera writes (INSERT, UPDATE, DELETE)
2. Monitore com `pg_stat_user_indexes` para detectar indices nao utilizados
3. Use `EXPLAIN ANALYZE` antes e depois de criar um indice
4. Indices em colunas de baixa cardinalidade (ex: boolean) raramente ajudam
5. Composite indexes: coloque a coluna mais seletiva primeiro

### 2.4 Query Optimization

**Ferramentas de diagnostico:**

```sql
-- Habilitar pg_stat_statements para rastrear queries lentas
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Top 10 queries por tempo total
SELECT
    calls,
    mean_exec_time::numeric(10,2) AS avg_ms,
    total_exec_time::numeric(10,2) AS total_ms,
    rows,
    query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Analisar plano de execucao
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders
WHERE user_id = 'uuid-here'
  AND created_at > NOW() - INTERVAL '30 days';
```

**Patterns de otimizacao:**

```sql
-- ANTI-PATTERN: CTE materializado desnecessario (PostgreSQL 12+)
-- CTEs sao inlined por padrao quando referenciados apenas uma vez
WITH user_orders AS (
    SELECT * FROM orders WHERE user_id = $1
)
SELECT * FROM user_orders WHERE status = 'pending';

-- MELHOR: O PostgreSQL 12+ ja faz inline automaticamente
-- Para forcar materializacao quando necessario:
WITH user_orders AS MATERIALIZED (
    SELECT * FROM orders WHERE user_id = $1
)
SELECT * FROM user_orders WHERE status = 'pending';

-- Materialized Views para dashboards
CREATE MATERIALIZED VIEW mv_daily_revenue AS
SELECT
    DATE_TRUNC('day', created_at) AS day,
    tenant_id,
    SUM(total) AS revenue,
    COUNT(*) AS order_count
FROM orders
WHERE status = 'completed'
GROUP BY 1, 2;

-- Indice unico necessario para REFRESH CONCURRENTLY
CREATE UNIQUE INDEX idx_mv_daily_revenue
    ON mv_daily_revenue (day, tenant_id);

-- Refresh sem lock (requer unique index)
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_revenue;
```

Fonte: [PostgreSQL CTE Optimization](https://hakibenita.com/be-careful-with-cte-in-postgre-sql)

### 2.5 Vacuum Tuning

O autovacuum e essencial para a saude do PostgreSQL em producao. Configuracoes default frequentemente nao sao suficientes para workloads pesados.

**Parametros recomendados para producao:**

| Parametro | Default | Producao | Motivo |
|----------|---------|----------|--------|
| `autovacuum_max_workers` | 3 | 5 | Mais workers para tabelas paralelas |
| `autovacuum_naptime` | 1min | 45s | Checa mais frequentemente |
| `vacuum_cost_limit` | 200 | 2000 | Vacuum mais agressivo |
| `vacuum_cost_delay` | 2ms | 5ms | Balanco entre velocidade e I/O |
| `autovacuum_vacuum_scale_factor` | 0.2 | 0.05 | Dispara com 5% de dead tuples (vs 20%) |
| `log_autovacuum_min_duration` | -1 | 200ms | Loga vacuums lentos |

Fonte: [Tuning Autovacuum](https://www.cybertec-postgresql.com/en/tuning-autovacuum-postgresql/)

**Tuning por tabela (para tabelas com alto volume de writes):**

```sql
-- Tabela com muitos updates: vacuum mais agressivo
ALTER TABLE events SET (
    autovacuum_vacuum_scale_factor = 0.01,     -- 1% de dead tuples
    autovacuum_vacuum_threshold = 1000,         -- minimo 1000 dead tuples
    autovacuum_analyze_scale_factor = 0.02,     -- re-analyze com 2%
    autovacuum_vacuum_cost_delay = 2            -- mais rapido
);
```

**Monitoramento de vacuum:**

```sql
-- Verificar dead tuples e ultimo vacuum
SELECT
    relname,
    n_dead_tup,
    n_live_tup,
    ROUND(n_dead_tup::numeric / NULLIF(n_live_tup, 0) * 100, 2) AS dead_pct,
    last_vacuum,
    last_autovacuum,
    last_analyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;

-- Bloqueadores do vacuum: transacoes longas
SELECT
    pid,
    age(NOW(), xact_start) AS duration,
    query,
    state
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start ASC
LIMIT 5;
```

Fonte: [Percona Autovacuum Tuning](https://www.percona.com/blog/tuning-autovacuum-in-postgresql-and-autovacuum-internals/)

---

## 3. RLS Advanced

### 3.1 Performance Impact

Row Level Security (RLS) pode ter **impacto massivo** na performance se nao otimizado corretamente. O database avalia policies para cada query, e sem otimizacao, isso pode adicionar overhead significativo.

**Resultado real de otimizacao:**

| Cenario | Sem otimizacao | Com otimizacao | Melhoria |
|---------|---------------|----------------|----------|
| SELECT com RLS em 1M rows | >3 minutos (timeout) | 2ms | 90.000x |
| auth.uid() sem indice | Seq Scan | Index Scan | 100x+ |
| Funcao em policy sem SELECT wrapper | Chamada por row | Cached (initPlan) | 1000x+ |

Fonte: [RLS Performance and Best Practices](https://supabase.com/docs/guides/troubleshooting/rls-performance-and-best-practices-Z5Jjwv)

### 3.2 Otimizacoes Criticas

**Regra #1: SEMPRE indexe colunas referenciadas em RLS policies**

```sql
-- OBRIGATORIO: indice na coluna usada no policy
CREATE INDEX idx_orders_user_id ON orders USING btree (user_id);
CREATE INDEX idx_orders_tenant_id ON orders USING btree (tenant_id);
```

**Regra #2: Use o SELECT wrapper pattern para funcoes**

```sql
-- ANTI-PATTERN: funcao chamada para cada row
CREATE POLICY "user_access" ON orders
    FOR ALL USING (
        auth.uid() = user_id  -- auth.uid() executado para CADA row
    );

-- CORRETO: SELECT wrapper permite cache via initPlan
CREATE POLICY "user_access" ON orders
    FOR ALL USING (
        (SELECT auth.uid()) = user_id  -- Cached, executado UMA vez
    );
```

**Regra #3: Evite subqueries complexas — use security definer functions**

```sql
-- ANTI-PATTERN: subquery na policy (executada por row)
CREATE POLICY "team_access" ON documents
    FOR ALL USING (
        team_id IN (
            SELECT team_id FROM team_members WHERE user_id = auth.uid()
        )
    );

-- CORRETO: security definer function com cache
CREATE OR REPLACE FUNCTION get_user_team_ids()
RETURNS UUID[]
LANGUAGE sql
SECURITY DEFINER
STABLE  -- Indica que o resultado nao muda dentro da transacao
SET search_path = ''
AS $$
    SELECT ARRAY(
        SELECT team_id FROM public.team_members
        WHERE user_id = (SELECT auth.uid())
    );
$$;

CREATE POLICY "team_access" ON documents
    FOR ALL USING (
        team_id = ANY((SELECT get_user_team_ids()))
    );
```

Fonte: [Supabase RLS Best Practices](https://makerkit.dev/blog/tutorials/supabase-rls-best-practices), [Optimizing RLS Performance](https://medium.com/@antstack/optimizing-rls-performance-with-supabase-postgres-fa4e2b6e196d)

### 3.3 Multi-Tenant RLS Patterns

**Pattern 1: Tenant via JWT Custom Claims (recomendado)**

```sql
-- 1. Adicionar tenant_id como custom claim no JWT via Auth Hook
-- 2. Acessar claim no RLS policy

CREATE POLICY "tenant_isolation" ON orders
    FOR ALL USING (
        tenant_id = (
            SELECT (auth.jwt() -> 'app_metadata' ->> 'tenant_id')::uuid
        )
    );

-- Indice obrigatorio
CREATE INDEX idx_orders_tenant ON orders (tenant_id);
```

**Pattern 2: Hierarquia de roles (admin > manager > member)**

```sql
CREATE OR REPLACE FUNCTION get_user_role(p_tenant_id UUID)
RETURNS TEXT
LANGUAGE sql
SECURITY DEFINER
STABLE
SET search_path = ''
AS $$
    SELECT role FROM public.tenant_members
    WHERE user_id = (SELECT auth.uid())
      AND tenant_id = p_tenant_id
    LIMIT 1;
$$;

-- Policy com hierarquia
CREATE POLICY "role_based_access" ON sensitive_data
    FOR SELECT USING (
        (SELECT get_user_role(tenant_id)) IN ('admin', 'manager')
    );

CREATE POLICY "role_based_write" ON sensitive_data
    FOR INSERT WITH CHECK (
        (SELECT get_user_role(tenant_id)) = 'admin'
    );
```

### 3.4 Testing Strategies com pgTAP

O Supabase suporta testes de database com pgTAP. RLS policies falham silenciosamente (nao lancam erros, apenas filtram dados), o que torna testes essenciais.

```sql
-- supabase/tests/rls_orders_test.sql

BEGIN;
SELECT plan(5);

-- Setup: criar usuarios de teste
SELECT tests.create_supabase_user('user_a', 'usera@test.com');
SELECT tests.create_supabase_user('user_b', 'userb@test.com');

-- Inserir dados com usuarios diferentes
SELECT tests.authenticate_as('user_a');
INSERT INTO orders (user_id, total, status)
VALUES ((SELECT auth.uid()), 100.00, 'pending');

SELECT tests.authenticate_as('user_b');
INSERT INTO orders (user_id, total, status)
VALUES ((SELECT auth.uid()), 200.00, 'completed');

-- Teste 1: User A so ve seus proprios pedidos
SELECT tests.authenticate_as('user_a');
SELECT is(
    (SELECT COUNT(*) FROM orders)::int,
    1,
    'User A should only see their own orders'
);

-- Teste 2: User B so ve seus proprios pedidos
SELECT tests.authenticate_as('user_b');
SELECT is(
    (SELECT COUNT(*) FROM orders)::int,
    1,
    'User B should only see their own orders'
);

-- Teste 3: User A nao pode atualizar pedidos de User B
SELECT tests.authenticate_as('user_a');
UPDATE orders SET status = 'cancelled'
WHERE user_id != (SELECT auth.uid());
SELECT is(
    (SELECT COUNT(*) FROM orders WHERE status = 'cancelled')::int,
    0,
    'User A should not be able to update User B orders'
);

-- Teste 4: Verificar que policies existem
SELECT policies_are(
    'public',
    'orders',
    ARRAY['user_access', 'user_insert', 'user_update']
);

-- Teste 5: Verificar roles da policy
SELECT policy_roles_are('public', 'orders', 'user_access', ARRAY['authenticated']);

SELECT * FROM finish();
ROLLBACK;
```

Fonte: [Testing with pgTAP](https://supabase.com/docs/guides/database/testing), [Testing RLS Policies with pgTAP](https://blair-devmode.medium.com/testing-row-level-security-rls-policies-in-postgresql-with-pgtap-a-supabase-example-b435c1852602)

**Executar testes:**

```bash
# Localmente
supabase test db

# Em CI/CD
supabase db test --linked
```

### 3.5 Anti-Patterns de RLS

| Anti-Pattern | Problema | Solucao |
|-------------|----------|---------|
| `auth.uid()` sem SELECT wrapper | Executado para cada row | `(SELECT auth.uid())` |
| Subquery na policy sem indice | Seq Scan na subquery | Criar indice + security definer function |
| Policy muito permissiva com OR | Desabilita index scan | Separar em policies especificas |
| Desabilitar RLS "temporariamente" | Nunca e re-habilitado | Nunca desabilitar em producao |
| Usar `service_role` no frontend | Bypassa RLS completamente | Apenas no servidor |
| Nao testar policies | Falhas silenciosas | pgTAP obrigatorio |

---

## 4. Multi-Tenant Architecture

### 4.1 Comparativo de Abordagens

| Criterio | Row-Level (RLS) | Schema-per-Tenant | Database-per-Tenant |
|----------|----------------|-------------------|---------------------|
| **Isolamento de dados** | Logico (policy) | Logico (schema) | Fisico (DB separado) |
| **Complexidade de setup** | Baixa | Media | Alta |
| **Migrations** | Uma migracao para todos | Uma por schema | Uma por database |
| **Performance de query** | Depende de indice + RLS | Sem overhead de RLS | Sem overhead |
| **Backup/restore por tenant** | Complexo | Medio | Simples |
| **Custo** | Baixo (1 DB) | Medio (1 DB, N schemas) | Alto (N DBs) |
| **Compliance (audit trail)** | Requer config adicional | Natural por schema | Natural por DB |
| **Max tenants pratico** | 10.000+ | 100-1.000 | 10-100 |
| **Supabase support nativo** | Sim (RLS built-in) | Parcial | Nao (1 projeto = 1 DB) |

### 4.2 Quando Usar Cada Abordagem

**Decision Framework:**

```
Quantos tenants voce tera?
|
+-- < 20 tenants (enterprise B2B, regulamentado)
|   -> Database-per-Tenant
|   -> Motivo: isolamento maximo, compliance, backup individual
|
+-- 20-500 tenants (B2B SaaS mid-market)
|   -> Schema-per-Tenant
|   -> Motivo: isolamento bom, migrations gerenciaveis
|
+-- 500+ tenants (B2B SaaS, B2C)
    -> Row-Level Security (RLS)
    -> Motivo: escala, custo, simplicidade operacional
```

Fonte: [Multi-Tenant Architecture](https://dev.to/blackie360/-enforcing-row-level-security-in-supabase-a-deep-dive-into-lockins-multi-tenant-architecture-4hd2)

### 4.3 Implementacao Row-Level (RLS) — Padrao Supabase

```sql
-- 1. Schema base
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT UNIQUE NOT NULL,
    plan TEXT DEFAULT 'free',
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE tenant_members (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE,
    user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
    role TEXT NOT NULL DEFAULT 'member' CHECK (role IN ('owner','admin','member','viewer')),
    UNIQUE(tenant_id, user_id)
);

-- 2. Toda tabela de negocio tem tenant_id
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE NOT NULL,
    name TEXT NOT NULL,
    created_by UUID REFERENCES auth.users(id),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 3. Indices obrigatorios
CREATE INDEX idx_tenant_members_user ON tenant_members (user_id);
CREATE INDEX idx_tenant_members_tenant ON tenant_members (tenant_id);
CREATE INDEX idx_projects_tenant ON projects (tenant_id);

-- 4. Funcao helper com cache
CREATE OR REPLACE FUNCTION get_user_tenant_ids()
RETURNS UUID[]
LANGUAGE sql
SECURITY DEFINER
STABLE
SET search_path = ''
AS $$
    SELECT COALESCE(
        ARRAY(SELECT tenant_id FROM public.tenant_members WHERE user_id = (SELECT auth.uid())),
        '{}'::UUID[]
    );
$$;

-- 5. RLS policies
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;

CREATE POLICY "tenant_isolation" ON projects
    FOR ALL USING (
        tenant_id = ANY((SELECT get_user_tenant_ids()))
    );

CREATE POLICY "tenant_insert" ON projects
    FOR INSERT WITH CHECK (
        tenant_id = ANY((SELECT get_user_tenant_ids()))
    );
```

### 4.4 Implementacao Schema-per-Tenant

```sql
-- Funcao para criar schema de novo tenant
CREATE OR REPLACE FUNCTION create_tenant_schema(p_tenant_slug TEXT)
RETURNS VOID
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
BEGIN
    -- Criar schema
    EXECUTE format('CREATE SCHEMA IF NOT EXISTS %I', p_tenant_slug);

    -- Criar tabelas no schema do tenant
    EXECUTE format('
        CREATE TABLE %I.projects (
            id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
            name TEXT NOT NULL,
            created_by UUID REFERENCES auth.users(id),
            created_at TIMESTAMPTZ DEFAULT NOW()
        )', p_tenant_slug);

    -- Grants
    EXECUTE format('GRANT USAGE ON SCHEMA %I TO authenticated', p_tenant_slug);
    EXECUTE format('GRANT ALL ON ALL TABLES IN SCHEMA %I TO authenticated', p_tenant_slug);
END;
$$;
```

Fonte: [Supabase Multi-Tenancy CRM Integration](https://www.stacksync.com/blog/supabase-multi-tenancy-crm-integration)

---

## 5. Supabase + Vercel Integration

### 5.1 SSR Auth com Next.js

O pacote `@supabase/ssr` gerencia autenticacao server-side usando cookies.

**Distincao critica: `getUser()` vs `getSession()`:**

| Metodo | Validacao | Uso Seguro |
|--------|----------|------------|
| `supabase.auth.getUser()` | Envia request ao Supabase Auth server, revalida token | Server Components, API Routes, Middleware |
| `supabase.auth.getSession()` | Le do cookie local, SEM revalidacao | Apenas client components (nunca para protecao) |

Fonte: [Supabase SSR Advanced Guide](https://supabase.com/docs/guides/auth/server-side/advanced-guide)

**Setup completo para Next.js App Router:**

```typescript
// lib/supabase/server.ts
import { createServerClient } from '@supabase/ssr';
import { cookies } from 'next/headers';

export async function createClient() {
  const cookieStore = await cookies();

  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return cookieStore.getAll();
        },
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options)
            );
          } catch {
            // Server Component — cookies somente leitura
          }
        },
      },
    }
  );
}
```

```typescript
// lib/supabase/client.ts
import { createBrowserClient } from '@supabase/ssr';

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  );
}
```

Fonte: [Creating a Supabase Client for SSR](https://supabase.com/docs/guides/auth/server-side/creating-a-client)

### 5.2 Edge Middleware para Auth

```typescript
// middleware.ts
import { createServerClient } from '@supabase/ssr';
import { NextResponse, type NextRequest } from 'next/server';

export async function middleware(request: NextRequest) {
  let supabaseResponse = NextResponse.next({ request });

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return request.cookies.getAll();
        },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value }) =>
            request.cookies.set(name, value)
          );
          supabaseResponse = NextResponse.next({ request });
          cookiesToSet.forEach(({ name, value, options }) =>
            supabaseResponse.cookies.set(name, value, options)
          );
        },
      },
    }
  );

  // IMPORTANTE: usar getUser(), NAO getSession()
  const { data: { user } } = await supabase.auth.getUser();

  if (!user && request.nextUrl.pathname.startsWith('/dashboard')) {
    const url = request.nextUrl.clone();
    url.pathname = '/login';
    return NextResponse.redirect(url);
  }

  return supabaseResponse;
}

export const config = {
  matcher: ['/dashboard/:path*', '/api/:path*'],
};
```

Fonte: [Setting up Server-Side Auth for Next.js](https://supabase.com/docs/guides/auth/server-side/nextjs)

### 5.3 Caching (ISR/SSG) com Supabase

**Problema critico de seguranca:** Se o CDN (Vercel Edge, Cloudflare) fizer cache de respostas autenticadas e servir para outro usuario, esse usuario recebera o token cacheado e sera autenticado como a pessoa errada.

**Solucao (a partir de `@supabase/ssr` v0.10.0):**

A biblioteca automaticamente passa headers de cache necessarios (Cache-Control, Expires, Pragma) ao callback `setAll` quando um token refresh ocorre.

```typescript
// Para rotas autenticadas: DESABILITAR cache
export const dynamic = 'force-dynamic';
// OU
export const revalidate = 0;

// Para dados publicos: usar ISR normalmente
export const revalidate = 3600; // revalidar a cada 1 hora

export default async function PublicPage() {
  const supabase = await createClient();
  const { data: posts } = await supabase
    .from('posts')
    .select('*')
    .eq('published', true);

  return <PostList posts={posts} />;
}
```

Fonte: [Supabase SSR Advanced Guide](https://supabase.com/docs/guides/auth/server-side/advanced-guide)

### 5.4 Rate Limiting

```typescript
// Usando Vercel KV para rate limiting com Supabase
import { Ratelimit } from '@upstash/ratelimit';
import { kv } from '@vercel/kv';

const ratelimit = new Ratelimit({
  redis: kv,
  limiter: Ratelimit.slidingWindow(10, '10 s'), // 10 requests por 10s
  analytics: true,
});

// No middleware ou API route
export async function POST(request: Request) {
  const ip = request.headers.get('x-forwarded-for') ?? '127.0.0.1';
  const { success, limit, reset, remaining } = await ratelimit.limit(ip);

  if (!success) {
    return new Response('Too Many Requests', {
      status: 429,
      headers: {
        'X-RateLimit-Limit': limit.toString(),
        'X-RateLimit-Remaining': remaining.toString(),
        'X-RateLimit-Reset': reset.toString(),
      },
    });
  }

  // Processar request normalmente...
}
```

### 5.5 Env Vars Management

**Regras para variaveis de ambiente com Vercel + Supabase:**

| Variavel | Prefixo | Onde usar | Segura? |
|---------|---------|----------|---------|
| `NEXT_PUBLIC_SUPABASE_URL` | `NEXT_PUBLIC_` | Client + Server | Sim (publica por design) |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | `NEXT_PUBLIC_` | Client + Server | Sim (respeita RLS) |
| `SUPABASE_SERVICE_ROLE_KEY` | Sem prefixo | Server ONLY | NAO expor no client |
| `SUPABASE_DB_URL` | Sem prefixo | Server ONLY | NAO expor no client |

**Configuracao na Vercel:**

```bash
# Via CLI
vercel env add SUPABASE_SERVICE_ROLE_KEY production
vercel env add NEXT_PUBLIC_SUPABASE_URL production preview development

# Ou via Vercel Marketplace (auto-configura as variaveis)
```

### 5.6 Edge Functions vs Vercel Functions

**Limitacao importante:** Vercel nao suporta nativamente o runtime Deno, criando incompatibilidade com as Edge Functions do Supabase (que sao baseadas em Deno). A melhor abordagem e desacoplar as Edge Functions do Supabase do Vercel e integrar via API calls.

```typescript
// Chamar Supabase Edge Function a partir de Vercel API Route
const { data, error } = await supabase.functions.invoke('process-payment', {
  body: { orderId: '123', amount: 9900 },
});
```

Fonte: [Supabase for Vercel](https://vercel.com/marketplace/supabase)

---

## 6. Supabase Branching & Environments

### 6.1 Database Branching

Supabase Branching cria ambientes separados a partir do projeto principal, permitindo testar mudancas sem afetar producao.

**Tipos de branch:**

| Tipo | Duracao | Auto-pause | Auto-delete | Uso |
|------|---------|-----------|-------------|-----|
| Preview | Efemero | Sim (apos inatividade) | Sim (quando PR merge/close) | Testar features isoladas |
| Persistent | Longa duracao | Nao | Nao | staging, QA, development |

Fonte: [Supabase Branching](https://supabase.com/docs/guides/deployment/branching)

**Integracao com GitHub:**

- Supabase monitora commits, branches e pull requests
- Branching automatico pode ser habilitado: criar branch no GitHub = criar branch no Supabase
- Ao mergear na branch principal, Supabase executa automaticamente o deployment workflow

Fonte: [GitHub Integration](https://supabase.com/docs/guides/deployment/branching/github-integration)

### 6.2 Migration Workflow (Local -> Staging -> Prod)

**Workflow recomendado:**

```
Local Development          Staging               Production
     |                        |                      |
     |-- supabase db diff --> |                      |
     |   (gera migration)     |                      |
     |                        |-- CI/CD push ------> |
     |                        |   (GitHub Actions)    |
     |                        |                      |
     v                        v                      v
  supabase/migrations/    Linked project         Auto-deploy
  seed.sql                  (staging)           via merge to main
```

**Comandos essenciais:**

```bash
# 1. Desenvolvimento local
supabase start                      # Inicia stack local
supabase db diff --local            # Diff entre schema local e migrations
supabase migration new add_feature  # Criar migration vazia
supabase db reset                   # Reset local + aplica todas as migrations

# 2. Testar migrations
supabase db push --linked           # Push para projeto staging
supabase db test --linked           # Rodar testes pgTAP no staging

# 3. Deployment via CI (GitHub Actions)
# supabase db push --linked e executado no CI quando merge para main
```

Fonte: [Managing Environments](https://supabase.com/docs/guides/deployment/managing-environments), [Database Migrations](https://supabase.com/docs/guides/deployment/database-migrations)

### 6.3 Seed Data

**Melhor pratica:** Incluir apenas INSERT statements no seed file, evitar DDL (schema statements).

```sql
-- supabase/seed.sql

-- Dados de referencia
INSERT INTO public.plans (id, name, price, features) VALUES
    ('free', 'Free', 0, '{"max_projects": 3}'),
    ('pro', 'Pro', 2500, '{"max_projects": 50, "analytics": true}'),
    ('enterprise', 'Enterprise', 9900, '{"max_projects": -1, "sso": true}');

-- Dados de teste (apenas para desenvolvimento)
INSERT INTO auth.users (id, email, raw_user_meta_data) VALUES
    ('d0fc3f88-1234-5678-9abc-def012345678', 'test@example.com', '{"name": "Test User"}');

INSERT INTO public.tenants (id, name, slug) VALUES
    ('a1b2c3d4-1234-5678-9abc-def012345678', 'Acme Corp', 'acme');
```

Fonte: [Seeding Your Database](https://supabase.com/docs/guides/local-development/seeding-your-database)

### 6.4 CI/CD com GitHub Actions

```yaml
# .github/workflows/supabase-deploy.yml
name: Deploy Supabase Migrations

on:
  push:
    branches: [main]
    paths:
      - 'supabase/migrations/**'

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: supabase/setup-cli@v1
        with:
          version: latest

      - name: Link Supabase project
        run: supabase link --project-ref ${{ secrets.SUPABASE_PROJECT_REF }}
        env:
          SUPABASE_ACCESS_TOKEN: ${{ secrets.SUPABASE_ACCESS_TOKEN }}

      - name: Run database tests
        run: supabase test db --linked

      - name: Deploy migrations
        run: supabase db push --linked
        env:
          SUPABASE_ACCESS_TOKEN: ${{ secrets.SUPABASE_ACCESS_TOKEN }}
```

**Regras de seguranca para producao:**

1. **Nunca** altere o database via Dashboard em producao — tudo via migrations
2. **Nunca** compartilhe senhas de producao (especialmente postgres)
3. Use approval workflows no CI para prevenir migrations acidentais
4. Mantenha migrations em version control (git)

Fonte: [Local Development Overview](https://supabase.com/docs/guides/local-development/overview)

---

## 7. Supabase Security Hardening

### 7.1 Custom JWT e Auth Hooks

O Custom Access Token Hook roda antes de cada emissao de token JWT e permite adicionar, remover ou alterar claims.

**Implementacao de RBAC via Auth Hook:**

```sql
-- Funcao que adiciona custom claims ao JWT
CREATE OR REPLACE FUNCTION custom_access_token_hook(event JSONB)
RETURNS JSONB
LANGUAGE plpgsql
STABLE
AS $$
DECLARE
    claims JSONB;
    user_role TEXT;
    user_tenant_id UUID;
BEGIN
    claims := event -> 'claims';

    -- Buscar role e tenant do usuario
    SELECT role, tenant_id INTO user_role, user_tenant_id
    FROM public.tenant_members
    WHERE user_id = (event ->> 'user_id')::UUID
    LIMIT 1;

    -- Adicionar claims customizados
    IF user_role IS NOT NULL THEN
        claims := jsonb_set(claims, '{app_metadata,role}', to_jsonb(user_role));
        claims := jsonb_set(claims, '{app_metadata,tenant_id}', to_jsonb(user_tenant_id::text));
    END IF;

    -- Retornar evento modificado
    event := jsonb_set(event, '{claims}', claims);
    RETURN event;
END;
$$;

-- Conceder permissao
GRANT EXECUTE ON FUNCTION custom_access_token_hook TO supabase_auth_admin;
REVOKE EXECUTE ON FUNCTION custom_access_token_hook FROM authenticated, anon, public;

-- Habilitar no Dashboard: Authentication > Hooks > Custom Access Token
```

Fonte: [Custom Access Token Hook](https://supabase.com/docs/guides/auth/auth-hooks/custom-access-token-hook)

**Limites dos hooks:**

| Tipo | Timeout | Nota |
|------|---------|------|
| Postgres Hook | 2 segundos | Executado no database |
| HTTP Hook | 5 segundos | Executado via Edge Function |

**Claims obrigatorios no JWT:**

`iss`, `aud`, `exp`, `iat`, `sub`, `role`, `aal`, `session_id`, `email`, `phone`, `is_anonymous`

Fonte: [JWT Claims Reference](https://supabase.com/docs/guides/auth/jwt-fields)

### 7.2 RBAC (Role-Based Access Control)

O Supabase esta desenvolvendo a extensao `supabase_rbac` para uma abordagem Postgres-first.

**Pattern atual de RBAC com custom claims:**

```sql
-- Acessar role no RLS policy
CREATE POLICY "admin_only" ON admin_settings
    FOR ALL USING (
        (SELECT (auth.jwt() -> 'app_metadata' ->> 'role')) = 'admin'
    );

-- Policy com hierarquia de roles
CREATE POLICY "manager_and_above" ON reports
    FOR SELECT USING (
        (SELECT (auth.jwt() -> 'app_metadata' ->> 'role'))
        IN ('admin', 'manager')
    );
```

Fonte: [Custom Claims & RBAC](https://supabase.com/docs/guides/database/postgres/custom-claims-and-role-based-access-control-rbac)

### 7.3 Rate Limiting

O Supabase Auth enforcea rate limits em endpoints de autenticacao para prevenir abuso.

**Limites configuraveis (Dashboard > Authentication > Rate Limits):**

| Endpoint | Default | Configuravel |
|---------|---------|-------------|
| Sign up | Sim | Sim |
| Sign in | Sim | Sim |
| Token refresh | Sim | Sim |
| Password recovery | Sim | Sim |

Para rate limiting na aplicacao, implemente no nivel do middleware (ver secao 5.4).

Fonte: [Auth Rate Limits](https://supabase.com/docs/guides/auth/rate-limits)

### 7.4 Network Restrictions

```bash
# Restringir acesso ao database para IPs especificos
# Via CLI
supabase network-restrictions update \
  --db-allow-cidr 203.0.113.0/24 \
  --db-allow-cidr 2001:db8::/32

# Verificar restricoes atuais
supabase network-restrictions get
```

**Consideracoes:**

- Restricoes sao aplicadas ANTES do trafego chegar ao database
- Necessario adicionar CIDRs IPv4 E IPv6 se conexoes diretas resolverem para IPv6
- Novas restricoes SUBSTITUEM as anteriores (nao sao aditivas)

Fonte: [Network Restrictions](https://supabase.com/docs/guides/platform/network-restrictions)

### 7.5 SSL Enforcement

```sql
-- Verificar se SSL esta enforced
-- Dashboard: Database > Settings > SSL Enforcement

-- Por padrao, Supabase permite conexoes SSL e non-SSL
-- Para maxima seguranca, enforce SSL:
-- Dashboard: Settings > Database > SSL Configuration > Enforce SSL
```

Fonte: [Platform Security](https://supabase.com/docs/guides/security/platform-security)

### 7.6 Custom Domains

```bash
# Configurar custom domain
# Dashboard: Settings > Custom Domains

# Supabase emite SSL via multiplas CAs:
# - Let's Encrypt
# - Google Trust Services
# - SSL.com
# Processo leva ate 30 minutos
```

Fonte: [Custom Domains](https://supabase.com/docs/guides/platform/custom-domains)

### 7.7 PGAudit

PGAudit estende o logging nativo do PostgreSQL para rastreamento seletivo de atividades.

```sql
-- Habilitar extensao
CREATE EXTENSION IF NOT EXISTS pgaudit;

-- Configurar logging por sessao
ALTER SYSTEM SET pgaudit.log = 'write, ddl';
-- write = INSERT, UPDATE, DELETE
-- ddl = CREATE, ALTER, DROP

-- Logging por usuario especifico
ALTER ROLE app_user SET pgaudit.log = 'all';

-- Logging por objeto (mais preciso, menos volume)
-- Recomendado para producao
ALTER ROLE auditor SET pgaudit.role = 'auditor';
GRANT SELECT ON sensitive_data TO auditor;
-- Agora apenas SELECTs em sensitive_data serao logados
```

**Alerta de producao:** PGAudit pode gerar ENORME volume de logs. Usar logging global sem restricoes pode desacelerar o database e esgotar espaco em disco. Prefira session, user ou object logging.

Fonte: [PGAudit](https://supabase.com/docs/guides/database/extensions/pgaudit)

### 7.8 Vault (Secrets Management)

O Vault e uma extensao PostgreSQL para armazenamento seguro de segredos usando Transparent Column Encryption (TCE).

```sql
-- Inserir segredo no Vault
-- ATENCAO: desabilitar statement logging antes
ALTER SYSTEM SET log_statement = 'none';
SELECT pg_reload_conf();

-- Armazenar segredo
SELECT vault.create_secret(
    'sk_live_abc123def456',    -- segredo
    'stripe_api_key',          -- nome unico
    'Chave de API do Stripe'   -- descricao
);

-- Recuperar segredo (decriptado)
SELECT * FROM vault.decrypted_secrets
WHERE name = 'stripe_api_key';

-- Usar em funcao
CREATE OR REPLACE FUNCTION call_stripe_api(p_endpoint TEXT)
RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
DECLARE
    api_key TEXT;
BEGIN
    SELECT decrypted_secret INTO api_key
    FROM vault.decrypted_secrets
    WHERE name = 'stripe_api_key';

    -- Usar api_key na chamada HTTP via pg_net
    RETURN (
        SELECT content::jsonb FROM net.http_get(
            p_endpoint,
            headers := jsonb_build_object(
                'Authorization', 'Bearer ' || api_key
            )
        )
    );
END;
$$;

-- Re-habilitar logging
ALTER SYSTEM SET log_statement = 'all';
SELECT pg_reload_conf();
```

**Tecnologia de criptografia:**

- Authenticated Encryption with Associated Data (AEAD) via pgsodium
- Chaves de criptografia gerenciadas separadamente dos dados
- Dados criptografados em disco e em dumps do database

Fonte: [Supabase Vault](https://supabase.com/docs/guides/database/vault), [Vault Feature](https://supabase.com/features/vault)

### 7.9 Security Hardening Checklist (2025-2026)

Baseado nas mudancas de seguranca anunciadas pelo Supabase:

| Item | Status | Acao |
|------|--------|------|
| Push protection para secret keys | Novo | Bloqueia commits com chaves expostas |
| pg_graphql desabilitado por default | Novo | Reduz superficie de ataque |
| Splinter security checks | Novo | Detecta weak password policies, privileges excessivos |
| Hardened security environments | Novo | Defaults estritos obrigatorios |
| `supabase_rbac` extension | Em desenvolvimento | RBAC nativo Postgres-first |
| MFA enforcement | Recomendado | Ativar para todas as contas admin |
| SSL enforcement | Recomendado | Forcar TLS em todas as conexoes |
| Network restrictions | Recomendado | Whitelist de IPs para acesso ao DB |

Fonte: [Supabase Security Retro 2025](https://supaexplorer.com/dev-notes/supabase-security-2025-whats-new-and-how-to-stay-secure.html)

---

## 8. Supabase vs Alternativas

### 8.1 Comparativo Geral

| Criterio | Supabase | Firebase | Neon | PlanetScale | Turso |
|----------|----------|----------|------|-------------|-------|
| **Database** | PostgreSQL | Firestore (NoSQL) | PostgreSQL | MySQL (Vitess) | libSQL (SQLite fork) |
| **Modelo** | BaaS completo | BaaS completo | Database-only | Database-only | Database-only |
| **Open Source** | Sim | Nao | Sim | Parcial | Sim |
| **Self-Host** | Sim (Docker) | Nao | Nao | Nao | Sim |
| **Auth built-in** | Sim | Sim | Nao | Nao | Nao |
| **Storage built-in** | Sim | Sim | Nao | Nao | Nao |
| **Realtime built-in** | Sim | Sim | Nao | Nao | Nao |
| **Edge Functions** | Sim (Deno) | Sim (Node.js) | Nao | Nao | Nao |
| **SQL Support** | Full SQL | NoSQL queries | Full SQL | Full SQL | SQL (SQLite dialect) |
| **Branching** | Sim (Git-based) | Nao | Sim (CoW, instantaneo) | Sim (Git-like) | Nao |
| **Scale-to-Zero** | Nao (pausa em free) | Nao | Sim | Nao | Sim |
| **Read Replicas** | Sim | Automatico | Sim | Sim (3 AZs default) | Sim (edge embedded) |
| **Vendor Lock-in** | Baixo | Alto | Medio | Medio | Baixo |

Fontes: [Supabase vs Firebase](https://supabase.com/alternatives/supabase-vs-firebase), [Neon vs Supabase](https://www.bytebase.com/blog/neon-vs-supabase/), [Best Database Software for Startups](https://makerkit.dev/blog/tutorials/best-database-software-startups)

### 8.2 Analise de Custos por Escala

#### Supabase Pricing (2025-2026)

| Plano | Preco/mes | Database | MAU Auth | Storage | Inclui |
|-------|----------|----------|----------|---------|--------|
| Free | $0 | 500 MB | 50K | 1 GB | 2 projetos, pausa em 7d |
| Pro | $25 | 8 GB | 100K | 100 GB | + $10 compute credit |
| Team | $599 | 8 GB | 100K | 100 GB | SSO, SOC 2, compliance |
| Enterprise | Custom | Custom | Custom | Custom | HIPAA, dedicated support |

**Compute add-ons:**

| Instancia | CPU | RAM | Preco/mes | Conexoes diretas |
|-----------|-----|-----|----------|-----------------|
| Nano (free) | Shared | 0.5 GB | $0 | 60 |
| Micro | 2-core ARM | 1 GB | $10 | 60 |
| Small | 2-core ARM | 2 GB | $15 | 90 |
| Medium | 2-core ARM | 4 GB | $60 | 120 |
| Large | 2-core ARM | 8 GB | $110 | 160 |
| XL | 4-core ARM | 16 GB | $210 | 240 |
| 2XL | 8-core ARM | 32 GB | $410 | 380 |
| 4XL | 16-core ARM | 64 GB | $810 | 480 |
| 8XL | 32-core ARM | 128 GB | $1.540 | 490 |
| 12XL | 48-core ARM | 192 GB | $2.310 | 500 |
| 16XL | 64-core ARM | 256 GB | $3.080 | 500 |

Fonte: [Supabase Pricing](https://supabase.com/pricing), [Compute and Disk](https://supabase.com/docs/guides/platform/compute-and-disk)

**Custos adicionais que surpreendem:**

| Item | Custo |
|------|-------|
| Bandwidth (egress) | $0.09/GB |
| Auth MAU (acima do incluso) | $0.00325/usuario |
| Read Replicas | Mesmo preco da instancia compute |
| Custom Domain | Incluso no Team+ |
| PITR (Point-in-Time Recovery) | Add-on pago |

Fonte: [Supabase Pricing Breakdown](https://www.metacto.com/blogs/the-true-cost-of-supabase-a-comprehensive-guide-to-pricing-integration-and-maintenance)

#### Firebase Pricing

| Plano | Firestore Reads | Firestore Writes | Storage | Auth |
|-------|----------------|-----------------|---------|------|
| Spark (Free) | 50K/dia | 20K/dia | 5 GB | 10K MAU (Phone) |
| Blaze (Pay-as-you-go) | $0.06/100K | $0.18/100K | $0.026/GB | $0.06/verification (Phone) |

**Risco:** Custos escalam com leituras/escritas. Um app com 100K DAU fazendo 10 reads/sessao = 1M reads/dia = ~$18/dia = ~$540/mes so em reads.

Fonte: [Firebase vs Supabase Pricing](https://www.getmonetizely.com/articles/supabase-vs-firebase-vs-planetscale-which-backend-as-a-service-is-right-for-your-budget)

#### Neon Pricing

| Plano | Compute | Storage | Branching | Scale-to-Zero |
|-------|---------|---------|-----------|---------------|
| Free | 100 CU-hours | 0.5 GB | Sim | Sim |
| Launch ($19/mo) | 100 CU-hours | 10 GB | Sim | Sim |
| Scale ($69/mo) | 750 CU-hours | 50 GB | Sim | Sim |
| Business ($700/mo) | 1000 CU-hours | 500 GB | Sim | Sim |

Preco por uso: $0.14/CU-hour compute, $0.35/GB-month storage (80% reducao em 2025).

Fonte: [Neon Pricing](https://neon.com/pricing), [Neon Database Review](https://www.getautonoma.com/blog/neon-database)

#### PlanetScale Pricing

| Plano | Preco/mes | Storage | Rows Read | Rows Written |
|-------|----------|---------|-----------|-------------|
| Hobby (removido) | Era $0 | - | - | - |
| Scaler | $29/mo | 10 GB | 1B/mo | 10M/mo |
| Scaler Pro | $39+/mo | 10 GB+ | Custom | Custom |
| Enterprise | Custom | Custom | Custom | Custom |

**Nota:** PlanetScale removeu o free tier em 2024 e agora tambem oferece PostgreSQL via Metal infrastructure com NVMe SSDs.

Fonte: [PlanetScale Pricing](https://planetscale.com/pricing)

#### Turso Pricing

| Plano | Preco/mes | Storage | Databases | Row Reads/mes |
|-------|----------|---------|-----------|-------------|
| Starter (Free) | $0 | 5 GB | 100 | 500M |
| Developer | $4.99 | 9 GB | Unlimited | 1B |
| Scaler | $24.92 | 24 GB | Unlimited | 5B |
| Pro | $416.58 | 50 GB | Unlimited | 25B |

**Diferencial:** Embedded replicas permitem sync local para reads com latencia zero. Ideal para apps read-heavy e edge computing.

Fonte: [Turso Pricing](https://turso.tech/pricing)

### 8.3 Estimativa de Custo por Escala de Usuarios

| Escala | Supabase | Firebase | Neon | PlanetScale | Turso |
|--------|----------|----------|------|-------------|-------|
| **1K MAU** | $25 (Pro) | ~$0-5 (Spark/Blaze) | $0-19 | $29+ | $0-5 |
| **10K MAU** | $25-60 | ~$30-80 | $19-69 | $39+ | $5-25 |
| **100K MAU** | $110-410 | ~$200-800 | $69-700 | $100+ | $25-417 |
| **1M MAU** | $810-3.080+ | ~$2.000-8.000+ | $700+ | $500+ | Custom |

**Notas:**
- Supabase: inclui Auth + Storage + Realtime + DB. Custo principal e compute
- Firebase: custos escalam com reads/writes, dificil prever
- Neon: scale-to-zero economiza em workloads intermitentes; mais barato para bursty
- PlanetScale: foco em reliability, sem free tier, preco premium por estabilidade
- Turso: mais barato para multi-tenant com muitos databases pequenos na edge

### 8.4 Decision Framework

```
Voce precisa de auth + storage + realtime + DB em uma plataforma?
|
+-- Sim
|   |
|   +-- Prefere SQL/relacional?
|   |   -> SUPABASE
|   |
|   +-- Prefere NoSQL/mobile-first?
|       -> FIREBASE
|
+-- Nao (so preciso de database)
    |
    +-- PostgreSQL?
    |   |
    |   +-- Serverless com scale-to-zero e branching?
    |   |   -> NEON
    |   |
    |   +-- BaaS completo mas so quero o DB?
    |       -> SUPABASE (ignora os extras)
    |
    +-- MySQL com sharding horizontal?
    |   -> PLANETSCALE
    |
    +-- SQLite na edge com embedded replicas?
        -> TURSO
```

**Matriz de decisao detalhada:**

| Se voce precisa de... | Escolha | Motivo |
|----------------------|---------|--------|
| Prototipar rapido, stack completo | Supabase | Auth+DB+Storage+Realtime integrados |
| Push notifications mobile | Firebase | Suporte nativo, FCM |
| Joins complexos, transacoes ACID | Supabase ou Neon | PostgreSQL completo |
| Schema branching instantaneo | Neon | Copy-on-Write, custo zero |
| Horizontal sharding automatico | PlanetScale | Vitess, non-blocking migrations |
| Read-heavy na edge (IoT, mobile sync) | Turso | Embedded replicas, latencia zero |
| Self-hosting (controle total) | Supabase | Docker, open-source completo |
| Evitar vendor lock-in | Supabase ou Turso | Open-source, standard SQL |
| Compliance (HIPAA, SOC 2) | Supabase Team/Enterprise ou Neon Business | Certificacoes disponíveis |
| Workloads imprevisíveis/bursty | Neon | Scale-to-zero economiza custo |
| Uptime maximo, SLA rigoroso | PlanetScale | Track record superior vs Neon |
| Multi-tenant com 10K+ tenants | Turso (se read-heavy) ou Supabase (RLS) | Turso: 1 DB por tenant nativo |

Fontes: [Best Database Software for Startups](https://makerkit.dev/blog/tutorials/best-database-software-startups), [Supabase vs Neon](https://www.bytebase.com/blog/neon-vs-supabase/), [Supabase vs PlanetScale](https://www.leanware.co/insights/supabase-vs-planetscale)

---

## Apendice: Fontes Consultadas

### Documentacao Oficial
- [Supabase Docs — Connection Management](https://supabase.com/docs/guides/database/connection-management)
- [Supabase Docs — Read Replicas](https://supabase.com/docs/guides/platform/read-replicas)
- [Supabase Docs — Edge Functions](https://supabase.com/docs/guides/functions)
- [Supabase Docs — Edge Functions Architecture](https://supabase.com/docs/guides/functions/architecture)
- [Supabase Docs — Edge Functions Limits](https://supabase.com/docs/guides/functions/limits)
- [Supabase Docs — Realtime Limits](https://supabase.com/docs/guides/realtime/limits)
- [Supabase Docs — Realtime Benchmarks](https://supabase.com/docs/guides/realtime/benchmarks)
- [Supabase Docs — Storage](https://supabase.com/docs/guides/storage)
- [Supabase Docs — S3 Compatibility](https://supabase.com/docs/guides/storage/s3/compatibility)
- [Supabase Docs — Image Transformations](https://supabase.com/docs/guides/storage/serving/image-transformations)
- [Supabase Docs — RLS](https://supabase.com/docs/guides/database/postgres/row-level-security)
- [Supabase Docs — RLS Performance](https://supabase.com/docs/guides/troubleshooting/rls-performance-and-best-practices-Z5Jjwv)
- [Supabase Docs — Custom Claims & RBAC](https://supabase.com/docs/guides/database/postgres/custom-claims-and-role-based-access-control-rbac)
- [Supabase Docs — Custom Access Token Hook](https://supabase.com/docs/guides/auth/auth-hooks/custom-access-token-hook)
- [Supabase Docs — JWT Claims Reference](https://supabase.com/docs/guides/auth/jwt-fields)
- [Supabase Docs — Auth Rate Limits](https://supabase.com/docs/guides/auth/rate-limits)
- [Supabase Docs — SSR Advanced Guide](https://supabase.com/docs/guides/auth/server-side/advanced-guide)
- [Supabase Docs — Creating SSR Client](https://supabase.com/docs/guides/auth/server-side/creating-a-client)
- [Supabase Docs — Next.js Auth](https://supabase.com/docs/guides/auth/server-side/nextjs)
- [Supabase Docs — Branching](https://supabase.com/docs/guides/deployment/branching)
- [Supabase Docs — Managing Environments](https://supabase.com/docs/guides/deployment/managing-environments)
- [Supabase Docs — Database Migrations](https://supabase.com/docs/guides/deployment/database-migrations)
- [Supabase Docs — Seeding](https://supabase.com/docs/guides/local-development/seeding-your-database)
- [Supabase Docs — Network Restrictions](https://supabase.com/docs/guides/platform/network-restrictions)
- [Supabase Docs — Custom Domains](https://supabase.com/docs/guides/platform/custom-domains)
- [Supabase Docs — Platform Security](https://supabase.com/docs/guides/security/platform-security)
- [Supabase Docs — PGAudit](https://supabase.com/docs/guides/database/extensions/pgaudit)
- [Supabase Docs — Vault](https://supabase.com/docs/guides/database/vault)
- [Supabase Docs — Testing](https://supabase.com/docs/guides/database/testing)
- [Supabase Docs — Compute and Disk](https://supabase.com/docs/guides/platform/compute-and-disk)
- [Supabase Pricing](https://supabase.com/pricing)
- [Supabase Docs — Supavisor FAQ](https://supabase.com/docs/guides/troubleshooting/supavisor-faq-YyP5tI)

### Blog Posts e Artigos
- [Supabase Blog — Read Replicas vs Bigger Compute](https://supabase.com/blog/read-replicas-vs-bigger-compute)
- [Supabase Blog — Persistent Storage for Edge Functions](https://supabase.com/blog/persistent-storage-for-faster-edge-functions)
- [Supabase Blog — Storage Image Resizing and Smart CDN](https://supabase.com/blog/storage-image-resizing-smart-cdn)
- [Supabase Blog — Supabase Local Dev](https://supabase.com/blog/supabase-local-dev)
- [Supabase Blog — Vault](https://supabase.com/blog/supabase-vault)
- [MakerKit — Supabase RLS Best Practices](https://makerkit.dev/blog/tutorials/supabase-rls-best-practices)
- [MakerKit — Best Database Software for Startups](https://makerkit.dev/blog/tutorials/best-database-software-startups)
- [AntStack — Optimizing RLS Performance](https://medium.com/@antstack/optimizing-rls-performance-with-supabase-postgres-fa4e2b6e196d)
- [SupaExplorer — Security Retro 2025](https://supaexplorer.com/dev-notes/supabase-security-2025-whats-new-and-how-to-stay-secure.html)
- [Metacto — Supabase Pricing Breakdown](https://www.metacto.com/blogs/the-true-cost-of-supabase-a-comprehensive-guide-to-pricing-integration-and-maintenance)
- [DesignRevision — Supabase Pricing Real Costs](https://designrevision.com/blog/supabase-pricing)
- [Bytebase — Neon vs Supabase](https://www.bytebase.com/blog/neon-vs-supabase/)
- [Bytebase — Supabase vs Firebase](https://www.bytebase.com/blog/supabase-vs-firebase/)
- [Leanware — Supabase vs PlanetScale](https://www.leanware.co/insights/supabase-vs-planetscale)
- [Leanware — Supabase vs Firebase Guide](https://www.leanware.co/insights/supabase-vs-firebase-complete-comparison-guide)
- [Leanware — Supabase Best Practices](https://www.leanware.co/insights/supabase-best-practices)
- [DEV.to — Firebase vs Supabase 2025](https://dev.to/dev_tips/firebase-vs-supabase-in-2025-which-one-actually-scales-with-you-2374)
- [DEV.to — RLS Multi-Tenant Architecture](https://dev.to/blackie360/-enforcing-row-level-security-in-supabase-a-deep-dive-into-lockins-multi-tenant-architecture-4hd2)
- [Stacksync — Supabase Multi-Tenancy CRM](https://www.stacksync.com/blog/supabase-multi-tenancy-crm-integration)
- [Blair Jordan — Testing RLS with pgTAP](https://blair-devmode.medium.com/testing-row-level-security-rls-policies-in-postgresql-with-pgtap-a-supabase-example-b435c1852602)
- [Haki Benita — CTE in PostgreSQL](https://hakibenita.com/be-careful-with-cte-in-postgre-sql)
- [Cybertec — Tuning Autovacuum](https://www.cybertec-postgresql.com/en/tuning-autovacuum-postgresql/)
- [Percona — Autovacuum Internals](https://www.percona.com/blog/tuning-autovacuum-in-postgresql-and-autovacuum-internals/)
- [PostgreSQL Docs — Index Types](https://www.postgresql.org/docs/current/indexes-types.html)
- [PGAnalyze — GIN Index](https://pganalyze.com/blog/gin-index)
- [Neon — PostgreSQL Index Types](https://neon.com/postgresql/postgresql-indexes/postgresql-index-types)
- [Neon Pricing](https://neon.com/pricing)
- [PlanetScale Pricing](https://planetscale.com/pricing)
- [Turso Pricing](https://turso.tech/pricing)
- [Turso — Embedded Replicas](https://turso.tech/blog/introducing-embedded-replicas-deploy-turso-anywhere-2085aa0dc242)
- [Deno — Supabase Functions on Deno Deploy](https://deno.com/blog/supabase-functions-on-deno-deploy)
- [GitHub — Supavisor Repository](https://github.com/supabase/supavisor)
- [Vercel — Supabase Marketplace](https://vercel.com/marketplace/supabase)
- [Supabase GitHub Discussion — Connection Pooler Session Mode Deprecation](https://github.com/orgs/supabase/discussions/32755)
