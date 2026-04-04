# Pesquisa Definitiva: Infraestrutura Profissional, Cybersecurity e Colaboracao em Software

> **Nivel:** DEFINITIVE (Research Depth Pyramid Level 4)
> **Audiencia:** Iniciantes que querem construir aplicacoes production-grade (de landing pages a fintechs)
> **Data:** 2026-04-04
> **Pesquisador:** Prism (Research Orchestrator) via Sage (Deep Research Scholar)
> **Fontes:** 30+ buscas, 15+ paginas completas, Tier 2-4

---

## SUMARIO EXECUTIVO

Este documento consolida pesquisa exaustiva sobre cinco pilares fundamentais para construir aplicacoes web profissionais: **Supabase** (backend-as-a-service), **Vercel** (deployment e hosting), **Cybersecurity** (seguranca de aplicacoes web), **Colaboracao em Software Engineering** (Git, CI/CD, code review) e **Obsidian como Second Brain para desenvolvimento com AI**. Cada secao contem arquitetura, exemplos praticos, padroes recomendados e erros comuns a evitar.

---

# PARTE 1: SUPABASE -- DEEP DIVE COMPLETO

## 1.1 Arquitetura

Supabase e uma plataforma de desenvolvimento baseada em PostgreSQL que combina multiplos servicos open-source atraves de um API Gateway (Kong). Em 2025, a empresa atingiu avaliacao de $5B e se reposicionou de "alternativa ao Firebase" para "plataforma de desenvolvimento Postgres".

### Componentes Principais

| Componente | Tecnologia | Funcao |
|-----------|------------|--------|
| **PostgreSQL** | Postgres 15+ | Banco de dados core -- acesso direto com privilegios totais |
| **GoTrue** | Go | Autenticacao JWT, gerenciamento de usuarios, emissao de tokens |
| **PostgREST** | Haskell | Transforma o schema do banco em REST API automaticamente |
| **Realtime** | Elixir/Phoenix | WebSockets para Broadcast, Presence e Postgres Changes |
| **Storage** | Node.js | Armazenamento de objetos compativel com S3, metadados no Postgres |
| **pg_graphql** | Rust (Extension) | API GraphQL gerada automaticamente do schema |
| **pg_meta** | Node.js | API RESTful para gerenciamento do banco (tabelas, roles, queries) |
| **Supavisor** | Elixir | Connection pooler cloud-native e multi-tenant |
| **Kong** | Lua/NGINX | API Gateway que roteia requests para todos os servicos |
| **Edge Functions** | Deno | Runtime TypeScript serverless para funcoes customizadas |
| **Studio** | TypeScript | Dashboard open-source para administracao |

### Fluxo de Request

```
Cliente → Kong (API Gateway) → [GoTrue | PostgREST | Realtime | Storage | Functions]
                                         ↓
                                    PostgreSQL
```

**Principio de design:** Cada componente funciona independentemente (isolamento), mas se complementa via APIs e webhooks (integracao). Voce pode usar qualquer componente separadamente ou auto-hospedar toda a stack.

**FINDING:** Supabase nao abstrai o PostgreSQL -- voce tem acesso direto com privilegios totais.
**IMPLICATION:** Diferente do Firebase, voce pode usar SQL puro, stored procedures, triggers e todas as extensoes do Postgres.
**RECOMMENDATION:** Aprenda SQL e PostgreSQL fundamentais antes de usar Supabase. Isso lhe dara controle total sobre o banco.

> Fontes: [Supabase Architecture Docs](https://supabase.com/docs/guides/getting-started/architecture) | [DeepWiki Supabase Overview](https://deepwiki.com/supabase/supabase/1-overview)

---

## 1.2 Autenticacao (Auth)

GoTrue e o servico de autenticacao do Supabase -- um fork do projeto original do Netlify. Ele gerencia usuarios, emite JWTs e integra com Row Level Security do PostgreSQL.

### Metodos Suportados

| Metodo | Descricao | Quando Usar |
|--------|-----------|-------------|
| **Email + Senha** | Login tradicional com email e password | Apps com cadastro proprio |
| **Magic Link** | Link enviado por email, clique para logar | Experiencia sem senha |
| **OTP (One-Time Password)** | Codigo enviado por email ou SMS | Verificacao rapida |
| **OAuth / Social Login** | Google, GitHub, Apple, Discord, etc. | Reduzir friccao no cadastro |
| **SSO (SAML)** | Single Sign-On empresarial | Clientes enterprise (B2B) |
| **Phone Auth** | Verificacao via SMS | Apps mobile-first |
| **Custom OAuth/OIDC** | Qualquer provider compativel | Integracao com identity providers especificos |

### MFA (Multi-Factor Authentication)

Supabase implementa MFA via dois metodos:

1. **App Authenticator (TOTP)** -- Time-based One-Time Password (Google Authenticator, Authy)
2. **Phone Messaging** -- Codigo gerado pelo Supabase Auth e enviado por SMS

Voce pode criar fluxos onde MFA e:
- **Opcional** -- usuario escolhe ativar
- **Obrigatorio para todos** -- forcado no login
- **Obrigatorio para grupos especificos** -- ex: apenas admins

### Fluxo JWT

```
1. Usuario faz login (email/senha, OAuth, magic link, etc.)
2. GoTrue valida credenciais contra auth.users no Postgres
3. GoTrue emite Access Token (JWT) + Refresh Token
4. Cliente inclui JWT em todas as requests subsequentes
5. PostgREST/Realtime/Storage validam o JWT
6. RLS policies usam auth.uid() e auth.jwt() para autorizar
```

### Identity Linking

Supabase automaticamente vincula identidades com o mesmo email a um unico usuario. Se alguem se cadastra com Google e depois tenta com GitHub (mesmo email), ambos apontam para o mesmo usuario.

### Configuracao de Email Templates

Templates de email (confirmacao, magic link, reset de senha) sao customizaveis no Dashboard. Para producao, configure um SMTP customizado (ex: Resend, SendGrid) em vez do SMTP padrao do Supabase que tem limites baixos.

**FINDING:** Magic Links expiram em 1 hora e so podem ser solicitados a cada 60 segundos por padrao.
**IMPLICATION:** Em producao, voce precisa configurar rate limits e timeouts adequados.
**RECOMMENDATION:** Use OAuth social login como metodo primario para reduzir friccao, e email/senha como fallback. Ative MFA obrigatorio para contas admin.

> Fontes: [Supabase Auth Docs](https://supabase.com/docs/guides/auth) | [Supabase MFA Docs](https://supabase.com/docs/guides/auth/auth-mfa) | [Auth Architecture](https://supabase.com/docs/guides/auth/architecture)

---

## 1.3 Row Level Security (RLS)

RLS e a feature mais critica do Supabase. Ele implementa autorizacao no nivel do banco de dados, funcionando como clausulas WHERE implicitas em toda query.

### Por que RLS e Obrigatorio

Em janeiro de 2025, 170+ apps construidos com Lovable foram encontrados com bancos expostos (CVE-2025-48757) porque desenvolvedores nao ativaram RLS. **RLS nao e opcional -- e a diferenca entre uma app segura e um data breach.**

### Como Ativar

```sql
-- Ativar RLS na tabela
ALTER TABLE public.profiles ENABLE ROW LEVEL SECURITY;

-- IMPORTANTE: Sem policies, NINGUEM acessa os dados (nem usuarios autenticados)
-- Voce precisa criar policies explicitas
```

### Sintaxe de Policies

| Operacao | Clausula | Funcao |
|----------|----------|--------|
| **SELECT** | `USING` | Filtra quais linhas o usuario pode LER |
| **INSERT** | `WITH CHECK` | Valida se o usuario pode INSERIR a nova linha |
| **UPDATE** | `USING` + `WITH CHECK` | USING filtra linhas existentes, WITH CHECK valida linhas modificadas |
| **DELETE** | `USING` | Filtra quais linhas o usuario pode DELETAR |

### Exemplos Praticos

```sql
-- 1. Usuarios so veem seus proprios dados
CREATE POLICY "users_own_data" ON profiles
FOR SELECT TO authenticated
USING ((SELECT auth.uid()) = user_id);

-- 2. Usuarios so podem inserir dados proprios
CREATE POLICY "users_insert_own" ON profiles
FOR INSERT TO authenticated
WITH CHECK ((SELECT auth.uid()) = user_id);

-- 3. Usuarios so podem atualizar dados proprios
CREATE POLICY "users_update_own" ON profiles
FOR UPDATE TO authenticated
USING ((SELECT auth.uid()) = user_id)
WITH CHECK ((SELECT auth.uid()) = user_id);

-- 4. Dados publicos visiveis para todos
CREATE POLICY "public_read" ON posts
FOR SELECT
USING (published = true);

-- 5. Multi-tenancy: usuarios so veem dados da sua organizacao
CREATE POLICY "tenant_isolation" ON orders
FOR ALL TO authenticated
USING (
  org_id IN (
    SELECT org_id FROM org_members
    WHERE user_id = (SELECT auth.uid())
  )
);

-- 6. Admin pode ver tudo
CREATE POLICY "admin_full_access" ON profiles
FOR ALL TO authenticated
USING (
  (SELECT auth.jwt() -> 'app_metadata' ->> 'role') = 'admin'
);
```

### Helper Functions

| Funcao | Retorna | Uso |
|--------|---------|-----|
| `auth.uid()` | UUID do usuario autenticado | Filtrar dados por usuario |
| `auth.jwt()` | JSON do token JWT completo | Acessar roles, metadata, claims |
| `auth.role()` | Role do PostgreSQL (anon/authenticated) | Diferenciar usuarios logados de anonimos |

### Erros Comuns

1. **Esquecer de ativar RLS** -- tabela fica completamente exposta via API
2. **Ativar RLS sem criar policies** -- ninguem acessa nada (deny-all padrao)
3. **Testar no SQL Editor** -- o SQL Editor BYPASSA RLS. Teste sempre via SDK do cliente
4. **Nao indexar colunas usadas em policies** -- causa full table scans lentos
5. **Usar `auth.uid()` sem `SELECT`** -- `(SELECT auth.uid()) = user_id` e ate 95% mais rapido que `auth.uid() = user_id` porque cacheia o resultado

### Otimizacao de Performance de RLS

| Tecnica | Melhoria Medida | Exemplo |
|---------|-----------------|---------|
| Indexar colunas de policies | 99.94% | `CREATE INDEX idx_user ON table(user_id)` |
| Envolver funcoes em SELECT | 94.97% | `(SELECT auth.uid()) = user_id` |
| Filtrar queries explicitamente | 94.74% | `.eq('user_id', userId)` no SDK |
| Usar security definer functions | 99.993% | `CREATE FUNCTION private.has_role()` |
| Minimizar JOINs em policies | 99.78% | Usar subqueries com IN |
| Especificar roles com TO | 99.78% | `FOR SELECT TO authenticated` |

### Views e RLS

Views bypassam RLS por padrao. No PostgreSQL 15+, ative:

```sql
CREATE VIEW my_view WITH (security_invoker = true)
AS SELECT * FROM my_table;
```

**FINDING:** RLS impacta performance significativamente sem otimizacao adequada.
**IMPLICATION:** Apps em producao podem ficar lentas se policies nao forem otimizadas.
**RECOMMENDATION:** Sempre indexe colunas referenciadas em policies, envolva funcoes em SELECT, e adicione filtros explicitos nas queries do SDK.

> Fontes: [Supabase RLS Docs](https://supabase.com/docs/guides/database/postgres/row-level-security) | [RLS Performance Best Practices](https://supabase.com/docs/guides/troubleshooting/rls-performance-and-best-practices-Z5Jjwv) | [VibeAppScanner RLS Guide 2026](https://vibeappscanner.com/supabase-row-level-security)

---

## 1.4 Edge Functions

Edge Functions sao funcoes server-side TypeScript executadas globalmente na edge, rodando no Deno runtime.

### Quando Usar

- Endpoints HTTP autenticados ou publicos com baixa latencia
- Webhooks (Stripe, GitHub, etc.)
- Geracao de imagens ou Open Graph sob demanda
- Inferencia AI e orquestracao de LLMs
- Emails transacionais
- Bots de mensagens (Slack, Discord, Telegram)
- Logica que nao pode rodar no cliente (secrets, validacao server-side)

### Arquitetura

```
1. Request chega ao Gateway
2. Gateway valida JWT, aplica rate-limiting
3. Funcao executa no node regional mais proximo
4. Novo V8 isolate e criado por invocacao (sandboxed)
5. Funcao acessa Supabase APIs ou servicos terceiros
6. Resposta retorna via gateway
```

### Criando uma Edge Function

```bash
# Criar nova funcao
supabase functions new my-function

# Estrutura criada:
# supabase/functions/my-function/index.ts
```

```typescript
// supabase/functions/my-function/index.ts
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'

Deno.serve(async (req) => {
  const supabase = createClient(
    Deno.env.get('SUPABASE_URL')!,
    Deno.env.get('SUPABASE_ANON_KEY')!,
    { global: { headers: { Authorization: req.headers.get('Authorization')! } } }
  )

  const { data, error } = await supabase.from('profiles').select('*')

  return new Response(JSON.stringify({ data, error }), {
    headers: { 'Content-Type': 'application/json' },
  })
})
```

### Deploy

```bash
# Deploy local para producao
supabase functions deploy my-function

# O CLI empacota a funcao em formato ESZip
# URL gerada: https://<project>.supabase.co/functions/v1/my-function
```

### Secrets

```bash
# Definir secrets (nao ficam no codigo)
supabase secrets set STRIPE_KEY=sk_live_xxx
supabase secrets set RESEND_API_KEY=re_xxx

# Acessar na funcao
const stripeKey = Deno.env.get('STRIPE_KEY')
```

### CORS

```typescript
const corsHeaders = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Headers': 'authorization, x-client-info, apikey, content-type',
}

// Tratar preflight
if (req.method === 'OPTIONS') {
  return new Response('ok', { headers: corsHeaders })
}
```

**FINDING:** Cold starts sao rapidos (milissegundos) gracas ao formato ESZip compacto e overhead minimo do Deno.
**IMPLICATION:** Edge Functions sao viaveis para endpoints criticos sem preocupacao com cold start.
**RECOMMENDATION:** Use Edge Functions para logica server-side leve. Para workloads pesados, use background workers.

> Fontes: [Edge Functions Docs](https://supabase.com/docs/guides/functions) | [Edge Functions Architecture](https://supabase.com/docs/guides/functions/architecture) | [Deno Edge Functions Feature](https://supabase.com/features/deno-edge-functions)

---

## 1.5 Design de Schema para Apps Comuns

### Padrao SaaS Multi-Tenant

Dois approaches principais:

**1. Shared Table com tenant_id (Recomendado para maioria)**

```sql
-- Tabela de organizacoes (tenants)
CREATE TABLE organizations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  slug TEXT UNIQUE NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Membros da organizacao
CREATE TABLE org_members (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  role TEXT CHECK (role IN ('owner', 'admin', 'member', 'viewer')),
  UNIQUE(org_id, user_id)
);

-- Qualquer tabela de dados do tenant
CREATE TABLE projects (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- RLS: isolamento por tenant
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;
CREATE POLICY "tenant_isolation" ON projects
FOR ALL TO authenticated
USING (
  org_id IN (
    SELECT org_id FROM org_members
    WHERE user_id = (SELECT auth.uid())
  )
);
```

**2. Schema-Per-Tenant (Enterprise com isolamento total)**

Cada tenant recebe um schema PostgreSQL dedicado. Mais seguro mas mais complexo de manter. Use quando regulamentacao exige isolamento fisico de dados.

### Padrao Fintech

```sql
-- Contas financeiras
CREATE TABLE accounts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users(id),
  type TEXT CHECK (type IN ('checking', 'savings', 'investment')),
  balance DECIMAL(15,2) DEFAULT 0.00,
  currency TEXT DEFAULT 'BRL',
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Transacoes (append-only, nunca deletar)
CREATE TABLE transactions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id UUID REFERENCES accounts(id),
  type TEXT CHECK (type IN ('credit', 'debit', 'transfer')),
  amount DECIMAL(15,2) NOT NULL,
  description TEXT,
  metadata JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Audit log (append-only)
CREATE TABLE audit_log (
  id BIGSERIAL PRIMARY KEY,
  table_name TEXT NOT NULL,
  record_id UUID NOT NULL,
  action TEXT CHECK (action IN ('INSERT', 'UPDATE', 'DELETE')),
  old_data JSONB,
  new_data JSONB,
  changed_by UUID REFERENCES auth.users(id),
  changed_at TIMESTAMPTZ DEFAULT now()
);
```

### Soft Deletes

```sql
-- Adicionar coluna de soft delete
ALTER TABLE profiles ADD COLUMN deleted_at TIMESTAMPTZ;

-- View que filtra deletados
CREATE VIEW active_profiles AS
SELECT * FROM profiles WHERE deleted_at IS NULL;

-- Ou usar RULE que converte DELETE em UPDATE
CREATE RULE soft_delete AS ON DELETE TO profiles
DO INSTEAD UPDATE profiles SET deleted_at = now()
WHERE id = OLD.id;
```

### Audit Logs com Triggers

```sql
-- Funcao generica de auditoria
CREATE OR REPLACE FUNCTION audit_trigger_func()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO audit_log (table_name, record_id, action, old_data, new_data, changed_by)
  VALUES (
    TG_TABLE_NAME,
    COALESCE(NEW.id, OLD.id),
    TG_OP,
    CASE WHEN TG_OP IN ('UPDATE', 'DELETE') THEN row_to_json(OLD) END,
    CASE WHEN TG_OP IN ('INSERT', 'UPDATE') THEN row_to_json(NEW) END,
    auth.uid()
  );
  RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Aplicar a qualquer tabela
CREATE TRIGGER audit_profiles
AFTER INSERT OR UPDATE OR DELETE ON profiles
FOR EACH ROW EXECUTE FUNCTION audit_trigger_func();
```

> Fontes: [Supabase Multi-Tenancy Discussion](https://github.com/orgs/supabase/discussions/1615) | [Soft Deletes Docs](https://supabase.com/docs/guides/troubleshooting/soft-deletes-with-supabase-js) | [Postgres Auditing](https://supabase.com/blog/postgres-audit)

---

## 1.6 Migrations e Branching

### Workflow de Migrations

```bash
# 1. Iniciar Supabase localmente
supabase init
supabase start

# 2. Criar migration
supabase migration new create_profiles_table

# 3. Editar o arquivo gerado em supabase/migrations/
# 4. Aplicar localmente
supabase db reset  # Re-aplica todas as migrations do zero

# 5. Deploy para producao
supabase db push
```

### Database Branching

Supabase Branching cria ambientes separados que derivam do projeto principal:

- **Preview Branches** -- efemeras, criadas para cada PR, pausadas apos inatividade, deletadas quando PR e fechado
- **Persistent Branches** -- long-lived, para staging, QA ou desenvolvimento, nao pausam automaticamente

Com integracao GitHub, migrations no diretorio `supabase/migrations/` sao aplicadas automaticamente. Dados de producao NAO sao copiados para branches de preview.

### Gerenciamento de Ambientes

```
Local (supabase start) → Preview (branch por PR) → Staging (persistent) → Production
```

> Fontes: [Branching Docs](https://supabase.com/docs/guides/deployment/branching) | [Database Migrations](https://supabase.com/docs/guides/deployment/database-migrations) | [Managing Environments](https://supabase.com/docs/guides/deployment/managing-environments)

---

## 1.7 Realtime

### Tres Features Principais

| Feature | Descricao | Use Case |
|---------|-----------|----------|
| **Broadcast** | Mensagens low-latency entre clientes | Chat, cursores colaborativos, gaming |
| **Presence** | Rastrear estado compartilhado entre clientes | "Online agora", usuarios ativos |
| **Postgres Changes** | Stream de mudancas no banco via WAL | Dashboards ao vivo, sync automatico |

### Exemplo: Chat em Tempo Real

```typescript
const channel = supabase.channel('room-1')

// Broadcast: enviar mensagem
channel.send({
  type: 'broadcast',
  event: 'message',
  payload: { user: 'Caio', text: 'Ola!' }
})

// Broadcast: receber mensagens
channel.on('broadcast', { event: 'message' }, (payload) => {
  console.log('Nova mensagem:', payload)
})

// Presence: rastrear quem esta online
channel.on('presence', { event: 'sync' }, () => {
  const state = channel.presenceState()
  console.log('Online:', Object.keys(state))
})

// Postgres Changes: escutar mudancas na tabela
channel.on('postgres_changes', {
  event: 'INSERT',
  schema: 'public',
  table: 'messages'
}, (payload) => {
  console.log('Nova mensagem no banco:', payload.new)
})

channel.subscribe()
```

### Canais Publicos vs Privados

Canais podem ser publicos (qualquer um se inscreve) ou privados (requerem autenticacao). Para dados sensiveis, use canais privados com RLS aplicado.

> Fontes: [Realtime Docs](https://supabase.com/docs/guides/realtime) | [Broadcast Docs](https://supabase.com/docs/guides/realtime/broadcast) | [Postgres Changes](https://supabase.com/docs/guides/realtime/postgres-changes)

---

## 1.8 Storage

### Conceitos

- **Buckets** -- containers para arquivos (como pastas S3)
- **Objetos** -- os arquivos em si
- **Policies** -- RLS aplicado na tabela `storage.objects`

### Tipos de Bucket

| Tipo | Acesso | Quando Usar |
|------|--------|-------------|
| **Public** | Qualquer um pode ler via URL | Imagens de perfil, assets publicos |
| **Private** | Acesso controlado por RLS | Documentos, uploads de usuarios |

### Exemplo: Upload de Imagem de Perfil

```typescript
// Upload
const { data, error } = await supabase.storage
  .from('avatars')
  .upload(`${userId}/avatar.png`, file, {
    cacheControl: '3600',
    upsert: true
  })

// URL publica (bucket publico)
const { data: { publicUrl } } = supabase.storage
  .from('avatars')
  .getPublicUrl(`${userId}/avatar.png`)

// URL assinada (bucket privado, expira em 1h)
const { data: { signedUrl } } = await supabase.storage
  .from('documents')
  .createSignedUrl(`${userId}/contract.pdf`, 3600)
```

### Policies de Storage

```sql
-- Usuarios podem fazer upload apenas na sua pasta
CREATE POLICY "users_upload_own" ON storage.objects
FOR INSERT TO authenticated
WITH CHECK (
  bucket_id = 'avatars' AND
  (storage.foldername(name))[1] = auth.uid()::text
);

-- Qualquer um pode ler avatares (bucket publico)
CREATE POLICY "public_read_avatars" ON storage.objects
FOR SELECT
USING (bucket_id = 'avatars');
```

### Restricoes por Bucket

Ao criar um bucket, voce pode definir:
- `allowedMimeTypes`: ex. `['image/png', 'image/jpeg']`
- `maxFileSize`: ex. `5242880` (5MB)

> Fontes: [Storage Docs](https://supabase.com/docs/guides/storage) | [Storage Access Control](https://supabase.com/docs/guides/storage/security/access-control) | [Storage Buckets](https://supabase.com/docs/guides/storage/buckets/fundamentals)

---

## 1.9 Vault (Gerenciamento de Secrets)

Vault e uma extensao do PostgreSQL para armazenar secrets encriptados diretamente no banco.

### Como Funciona

- **Encriptacao em repouso** -- secrets sao armazenados em disco de forma encriptada
- **AEAD (Authenticated Encryption with Associated Data)** -- dados sao encriptados E assinados (nao podem ser forjados)
- **Separacao de chaves** -- a chave de encriptacao NUNCA fica junto dos dados, e gerenciada pelo backend do Supabase

### Uso

```sql
-- Criar um secret
SELECT vault.create_secret('sk_live_abc123', 'stripe_key', 'Chave do Stripe');

-- Ler secrets (view que decripta no momento da query)
SELECT * FROM vault.decrypted_secrets WHERE name = 'stripe_key';

-- Atualizar secret
SELECT vault.update_secret(
  'uuid-do-secret',
  'novo_valor',
  'stripe_key',
  'Descricao atualizada'
);
```

### Cuidado Critico em Producao

Quando voce insere secrets via `INSERT` SQL, esses statements sao logados por padrao nos logs do Supabase. **Desative statement logging ao usar o Vault** para evitar que secrets apaream em plaintext nos logs.

> Fontes: [Vault Docs](https://supabase.com/docs/guides/database/vault) | [Vault Feature Page](https://supabase.com/features/vault) | [Vault Blog Post](https://supabase.com/blog/supabase-vault)

---

## 1.10 Performance

### Indexacao

| Tipo de Index | Quando Usar | Exemplo |
|--------------|-------------|---------|
| **B-tree** (padrao) | Comparacoes de igualdade e range | `CREATE INDEX idx ON table(column)` |
| **Hash** | Apenas igualdade exata | `CREATE INDEX idx ON table USING hash(column)` |
| **GIN** | Arrays, JSONB, full-text search | `CREATE INDEX idx ON table USING gin(metadata)` |
| **GiST** | Dados geometricos, ranges, proximity | `CREATE INDEX idx ON table USING gist(location)` |
| **BRIN** | Colunas que crescem monotonicamente (created_at) | 10x menor que B-tree equivalente |

### Partial Indexes

```sql
-- Indexar apenas pedidos ativos (muito menor que index completo)
CREATE INDEX idx_active_orders ON orders(created_at)
WHERE status = 'active';
```

### Composite Indexes

```sql
-- Para queries que filtram por ambas colunas
CREATE INDEX idx_org_user ON org_members(org_id, user_id);
```

### EXPLAIN ANALYZE

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 'abc-123';
-- Procure por: Sequential Scan (ruim), Index Scan (bom)
-- Se aparecer Seq Scan em tabela grande, crie um index
```

### Connection Pooling (Supavisor)

Supavisor e o connection pooler cloud-native do Supabase. Regra geral:

- Se usa bastante a API REST (PostgREST), nao suba o pool size acima de 40% das Max Connections
- Caso contrario, pode usar 80% para o pool
- Para serverless/edge functions, **sempre** use Supavisor (modo transaction) em vez de conexao direta

### pg_stat_statements

Habilitado por padrao em todo projeto Supabase. Use para encontrar queries ineficientes:

```sql
SELECT query, calls, total_exec_time, mean_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

> Fontes: [Query Optimization](https://supabase.com/docs/guides/database/query-optimization) | [Performance Tuning](https://supabase.com/docs/guides/platform/performance) | [Connection Management](https://supabase.com/docs/guides/database/connection-management)

---

## 1.11 Security Hardening

### SSL/TLS

Por padrao, Supabase permite conexoes SSL e nao-SSL. Para producao, **force SSL**:
- HTTP APIs (PostgREST, Storage, Auth) ja forcam SSL automaticamente
- Conexoes diretas ao PostgreSQL precisam de configuracao explicita
- Necessario para conformidade SOC 2 e HIPAA

### Network Restrictions

Restrinja quais IPs podem se conectar ao seu banco:
- Aplicado a conexoes diretas e pooler
- NAO se aplica a HTTP APIs
- Mesmo sem restricao de IP, credenciais validas ainda sao necessarias

### API Keys

| Chave | Onde Usar | Seguranca |
|-------|-----------|-----------|
| `anon` (publishable) | Frontend/Cliente | Respeita RLS policies |
| `service_role` (secret) | Backend APENAS | **BYPASSA RLS** -- acesso total |

**Regras de ouro:**
1. NUNCA exponha `service_role` no frontend
2. Armazene chaves em variaveis de ambiente
3. Rotacione chaves periodicamente
4. Use chaves diferentes para cada ambiente (dev/staging/prod)

### Alertas Automaticos (2025+)

Se voce criar tabelas com RLS desativado, Supabase envia alertas por email e mostra avisos no Dashboard. Push protection bloqueia commits contendo chaves secretas.

> Fontes: [Securing Your API](https://supabase.com/docs/guides/api/securing-your-api) | [API Keys](https://supabase.com/docs/guides/api/api-keys) | [Platform Security](https://supabase.com/docs/guides/security/platform-security) | [Security Retro 2025](https://supabase.com/blog/supabase-security-2025-retro)

---

## 1.12 Monitoramento

### Ferramentas Integradas

| Ferramenta | Funcao |
|-----------|--------|
| **pg_stat_statements** | Identifica queries lentas (habilitado por padrao) |
| **Logflare Endpoints** | Queries SQL nos logs para analise |
| **Supabase Reports** | Dashboards dedicados para servidores |
| **Metrics API** | Stream de metricas Prometheus-compativel |
| **OpenTelemetry** | Export para plataformas de observabilidade |

### Integracao com Ferramentas Externas

- **Grafana** -- Dashboard JSON pre-construido com 200+ graficos (`supabase-grafana`)
- **Datadog** -- Integracao nativa disponivel
- Qualquer stack Prometheus-compativel

### Comandos de Inspecao

```sql
-- Queries mais lentas
SELECT * FROM pg_stat_statements ORDER BY mean_exec_time DESC LIMIT 10;

-- Conexoes ativas
SELECT * FROM pg_stat_activity WHERE state = 'active';

-- Tabelas sem index sendo escaneadas
SELECT relname, seq_scan, idx_scan
FROM pg_stat_user_tables
WHERE seq_scan > idx_scan
ORDER BY seq_scan DESC;
```

> Fontes: [Debugging and Monitoring](https://supabase.com/docs/guides/database/inspect) | [Logs & Analytics](https://supabase.com/features/logs-analytics) | [Metrics API](https://supabase.com/docs/guides/telemetry/metrics)

---

## 1.13 Pricing

### Planos (2025-2026)

| Plano | Preco | Database | MAUs | Storage | Observacoes |
|-------|-------|----------|------|---------|-------------|
| **Free** | $0/mes | 500 MB | 50K | 1 GB | 2 projetos, pausa apos 7 dias inativo |
| **Pro** | $25/mes + uso | 8 GB | 100K | 100 GB | $10 credito compute, spend cap ativado |
| **Team** | $599/mes | Pro + extras | Pro + extras | Pro + extras | Colaboracao de equipe |
| **Enterprise** | Custom | Custom | Custom | Custom | SLA, compliance, suporte dedicado |

### Quando Fazer Upgrade

- **Free → Pro**: Quando seu app precisa estar online 24/7 (free pausa apos 7 dias)
- **Pro → Team**: Quando voce tem equipe e precisa de collaboration features
- **Team → Enterprise**: Quando precisa de SLA, compliance (SOC 2, HIPAA)

### Custos que Crescem Rapido

| Recurso | Custo Adicional | Observacao |
|---------|----------------|-----------|
| MAUs | $3.25 / 1.000 usuarios | Principal driver de custo em escala |
| Bandwidth | Variavel por plano | Monitorar via Dashboard |
| Compute scaling | Conforme plan | Escala automatica pode surprender |

**FINDING:** O plano Pro a $25/mes e o backend minimo viavel para qualquer produto que voce pretende cobrar.
**IMPLICATION:** O custo e menor que um jantar, mas MAUs podem escalar rapidamente.
**RECOMMENDATION:** Comece no Free para aprender, migre para Pro assim que for para producao. Ative spend cap no Pro para evitar surpresas na fatura.

> Fontes: [Supabase Pricing](https://supabase.com/pricing) | [UI Bakery Pricing Breakdown](https://uibakery.io/blog/supabase-pricing) | [Metacto Pricing Guide](https://www.metacto.com/blogs/the-true-cost-of-supabase-a-comprehensive-guide-to-pricing-integration-and-maintenance)

---

## 1.14 TypeScript Types Gerados Automaticamente

```bash
# Gerar types do projeto remoto
npx supabase gen types typescript --project-id "seu-project-id" > database.types.ts

# Gerar types do banco local
npx supabase gen types typescript --local > database.types.ts
```

### Usando os Types

```typescript
import { createClient } from '@supabase/supabase-js'
import { Database } from './database.types'

const supabase = createClient<Database>(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
)

// Agora voce tem autocomplete e type safety em todas as queries
const { data } = await supabase
  .from('profiles')    // TypeScript conhece suas tabelas
  .select('id, name')  // Autocomplete nos campos
  .eq('id', userId)    // Type-safe
```

### Automatizando via GitHub Actions

```yaml
# .github/workflows/gen-types.yml
name: Generate Types
on:
  schedule:
    - cron: '0 0 * * *' # Diariamente
jobs:
  generate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npx supabase gen types typescript --project-id ${{ secrets.SUPABASE_PROJECT_ID }} > database.types.ts
      - uses: stefanzweifel/git-auto-commit-action@v5
        with:
          commit_message: 'chore: update database types'
```

> Fontes: [Generating Types](https://supabase.com/docs/guides/api/rest/generating-types) | [TypeScript Support](https://supabase.com/docs/reference/javascript/typescript-support)

---

# PARTE 2: VERCEL -- DEEP DIVE COMPLETO

## 2.1 Arquitetura

Vercel opera em tres camadas de compute:

| Camada | O que e | Quando Usar |
|--------|---------|-------------|
| **Static (SSG/ISR)** | Paginas pre-renderizadas servidas via CDN | Conteudo que muda raramente |
| **Serverless Functions** | Funcoes Node.js que escalam automaticamente | API routes, SSR, data fetching |
| **Edge Functions** | Funcoes leves no V8 runtime, globalmente distribuidas | Middleware, autenticacao, personalizacao |

### Fluid Compute (2025+)

Vercel introduziu "Fluid Compute" que permite execucao concorrente na mesma instancia de funcao, reduzindo cold starts, latencia e custos. Em vez de criar multiplas instancias isoladas, uma unica instancia reutiliza tempo ocioso para processar novas requests.

### Fluxo de Request

```
Usuario → Edge Network (CDN) → [Cache Hit?]
  ├── Sim → Retorna resposta cached
  └── Nao → Roteamento para:
       ├── Static Asset → CDN edge node
       ├── Serverless Function → Regiao especifica (default: iad1)
       └── Edge Function → Node mais proximo do usuario
```

> Fontes: [Vercel Functions](https://vercel.com/docs/functions) | [How Vercel Works Guide](https://www.techtic.com/blog/how-vercel-works-beginner-friendly-guide/)

---

## 2.2 Next.js Deployment

### Estrategias de Renderizacao

| Estrategia | Sigla | Quando Renderiza | Quando Usar |
|-----------|-------|------------------|-------------|
| **Static Site Generation** | SSG | No build time | Blog posts, documentacao, marketing pages |
| **Incremental Static Regeneration** | ISR | Background apos revalidate | E-commerce, conteudo que muda periodicamente |
| **Server-Side Rendering** | SSR | A cada request | Dados unicos por usuario, tempo real |
| **Streaming** | -- | Progressivo | UIs complexas, AI responses |

### App Router e Server Components

```typescript
// app/page.tsx - Server Component por padrao (SSG)
export default async function Page() {
  const data = await fetch('https://api.example.com/data')
  return <div>{data}</div>
}

// ISR: revalida a cada 60 segundos
export default async function Page() {
  const data = await fetch('https://api.example.com/data', {
    next: { revalidate: 60 }
  })
  return <div>{data}</div>
}

// SSR: sem cache (dynamic)
export const dynamic = 'force-dynamic'
export default async function Page() {
  const data = await fetch('https://api.example.com/data', {
    cache: 'no-store'
  })
  return <div>{data}</div>
}
```

### Deploy Automatico

Ao conectar um repositorio GitHub, Vercel automaticamente:
1. Detecta Next.js e configura build
2. Cria preview deployment para cada PR
3. Deploy para producao ao mergear na branch principal
4. Configura SSL, CDN, e otimizacoes automaticamente

> Fontes: [Next.js on Vercel](https://vercel.com/docs/frameworks/full-stack/nextjs) | [Rendering Strategy Guide](https://vercel.com/blog/how-to-choose-the-best-rendering-strategy-for-your-app) | [Enterprise Next.js Architecture](https://slashdev.io/us/blog/enterprise-nextjs-architecture-on-vercel-ssg-isr-rsc)

---

## 2.3 Environment Variables

### Escopos

| Ambiente | Quando Aplica | Exemplo |
|----------|--------------|---------|
| **Development** | `vercel dev` localmente | DB_URL do banco local |
| **Preview** | Deploy de branches que nao sao producao | DB_URL do banco staging |
| **Production** | Deploy da branch principal | DB_URL do banco producao |

### Variaveis Sensiveis

Variaveis marcadas como "sensitive" tem protecoes adicionais: valores ficam ocultos no Dashboard. So podem ser criadas em Preview e Production.

### CLI

```bash
# Puxar variaveis para .env.local
vercel env pull

# Adicionar variavel
vercel env add STRIPE_KEY production

# Listar variaveis
vercel env ls
```

### NEXT_PUBLIC_ (CUIDADO)

Variaveis com prefixo `NEXT_PUBLIC_` sao expostas no bundle do cliente. **NUNCA** coloque secrets nelas:

```
NEXT_PUBLIC_SUPABASE_URL=https://xxx.supabase.co     # OK - e publico
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJhb...               # OK - anon key respeita RLS
SUPABASE_SERVICE_ROLE_KEY=eyJhb...                    # Sem NEXT_PUBLIC_ - so server
```

### Edge Config

Vercel Edge Config permite configuracao dinamica na edge (feature flags, A/B testing) sem redeploy.

> Fontes: [Vercel Environment Variables](https://vercel.com/docs/environment-variables) | [Vercel Environments](https://vercel.com/docs/deployments/environments) | [Sensitive Variables](https://vercel.com/docs/environment-variables/sensitive-environment-variables)

---

## 2.4 Edge Functions e Middleware

### Middleware

Middleware executa ANTES do cache e do routing, ideal para:
- Autenticacao e autorizacao
- Redirects baseados em geo-localizacao
- A/B testing
- Feature flags

```typescript
// middleware.ts (na raiz do projeto)
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  // Redirecionar usuarios nao autenticados
  const token = request.cookies.get('sb-token')
  if (!token && request.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url))
  }
  return NextResponse.next()
}

export const config = {
  matcher: ['/dashboard/:path*']
}
```

**Limite:** Middleware pode usar no maximo 50ms de CPU time em media.

### Edge API Routes

```typescript
// app/api/hello/route.ts
export const runtime = 'edge' // Executa na Edge globalmente

export async function GET(request: Request) {
  return new Response('Hello from the Edge!')
}
```

> Fontes: [Edge Functions Docs](https://vercel.com/docs/functions/runtimes/edge/edge-functions.rsc) | [Vercel Edge Explained](https://upstash.com/blog/vercel-edge)

---

## 2.5 Cron Jobs

```json
// vercel.json
{
  "crons": [
    {
      "path": "/api/cleanup",
      "schedule": "0 0 * * *"
    },
    {
      "path": "/api/send-digest",
      "schedule": "0 9 * * 1"
    }
  ]
}
```

### Seguranca

Proteja cron endpoints com `CRON_SECRET`:

```typescript
// app/api/cleanup/route.ts
export async function GET(request: Request) {
  const authHeader = request.headers.get('authorization')
  if (authHeader !== `Bearer ${process.env.CRON_SECRET}`) {
    return new Response('Unauthorized', { status: 401 })
  }
  // Executar limpeza...
}
```

> Fontes: [Cron Jobs Docs](https://vercel.com/docs/cron-jobs) | [Cron Jobs Quickstart](https://vercel.com/docs/cron-jobs/quickstart)

---

## 2.6 Analytics e Speed Insights

### Web Analytics

Metricas de visitantes, page views e referrers. Ativado com um script leve no frontend.

### Speed Insights

Coleta metricas reais dos dispositivos dos usuarios (Real User Monitoring):

| Metrica | O que mede |
|---------|-----------|
| **FCP** (First Contentful Paint) | Tempo ate primeiro conteudo visivel |
| **LCP** (Largest Contentful Paint) | Tempo ate maior elemento visivel |
| **CLS** (Cumulative Layout Shift) | Estabilidade visual (sem "pulos") |
| **INP** (Interaction to Next Paint) | Responsividade a interacoes |
| **TTFB** (Time to First Byte) | Tempo ate primeiro byte do servidor |

```bash
npm install @vercel/speed-insights
```

```typescript
// app/layout.tsx
import { SpeedInsights } from '@vercel/speed-insights/next'

export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        {children}
        <SpeedInsights />
      </body>
    </html>
  )
}
```

> Fontes: [Speed Insights](https://vercel.com/docs/speed-insights) | [Vercel Observability](https://vercel.com/products/observability)

---

## 2.7 Storage Options (2025+)

### Mudanca Importante

`@vercel/postgres` e `@vercel/kv` foram **descontinuados**. Agora use providers do Marketplace:

| Necessidade | Provider Recomendado | Pacote |
|------------|---------------------|--------|
| **PostgreSQL** | Neon, Supabase | `@neondatabase/serverless` |
| **Key-Value (Redis)** | Upstash Redis | `@upstash/redis` |
| **Blob Storage** | Vercel Blob (mantido) | `@vercel/blob` |
| **Edge Config** | Vercel Edge Config (mantido) | `@vercel/edge-config` |

### Vercel Blob

Upload direto do browser ate 5 TB:

```typescript
import { put } from '@vercel/blob'

const blob = await put('avatar.png', file, {
  access: 'public',
})
console.log(blob.url)
```

> Fontes: [Vercel Storage Overview](https://vercel.com/docs/storage) | [Marketplace Storage](https://vercel.com/docs/marketplace-storage)

---

## 2.8 Performance e Otimizacao

### Image Optimization

```typescript
import Image from 'next/image'

// Automaticamente: resize, compress, serve em WebP/AVIF
<Image
  src="/hero.jpg"
  width={1200}
  height={600}
  alt="Hero"
  priority // Carrega imediatamente (acima do fold)
/>
```

### Font Optimization

```typescript
import { Inter } from 'next/font/google'
const inter = Inter({ subsets: ['latin'] })

// next/font: carrega apenas subsets necessarios
// Elimina layout shift de fontes
// Self-hosted automaticamente (sem request a Google)
```

### Bundle Analysis

```bash
npm install @next/bundle-analyzer

# next.config.ts
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
})
module.exports = withBundleAnalyzer(nextConfig)

# Executar analise
ANALYZE=true npm run build
```

### optimizePackageImports

```typescript
// next.config.ts
module.exports = {
  experimental: {
    optimizePackageImports: ['lodash', 'date-fns', '@heroicons/react']
  }
}
```

Resultado em caso real: -40% no build time e payload JS inicial menor.

> Fontes: [Image Optimization](https://vercel.com/docs/image-optimization) | [Vercel Optimization Guide](https://dev.to/pipipi-dev/vercel-optimization-reducing-build-time-and-improving-response-2eji)

---

## 2.9 Security Headers

### Content Security Policy (CSP)

```typescript
// middleware.ts
import { NextResponse } from 'next/server'

export function middleware(request) {
  const nonce = Buffer.from(crypto.randomUUID()).toString('base64')
  const cspHeader = `
    default-src 'self';
    script-src 'self' 'nonce-${nonce}';
    style-src 'self' 'unsafe-inline';
    img-src 'self' blob: data:;
    font-src 'self';
    connect-src 'self' https://*.supabase.co;
    frame-ancestors 'none';
    form-action 'self';
    upgrade-insecure-requests;
  `
  const response = NextResponse.next()
  response.headers.set('Content-Security-Policy', cspHeader.replace(/\n/g, ''))
  response.headers.set('x-nonce', nonce)
  return response
}
```

### Headers de Seguranca Essenciais

```json
// next.config.js ou vercel.json
{
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        { "key": "X-Frame-Options", "value": "DENY" },
        { "key": "X-Content-Type-Options", "value": "nosniff" },
        { "key": "Referrer-Policy", "value": "strict-origin-when-cross-origin" },
        { "key": "Strict-Transport-Security", "value": "max-age=63072000; includeSubDomains; preload" },
        { "key": "Permissions-Policy", "value": "camera=(), microphone=(), geolocation=()" }
      ]
    }
  ]
}
```

### CORS

```json
// vercel.json
{
  "headers": [
    {
      "source": "/api/(.*)",
      "headers": [
        { "key": "Access-Control-Allow-Credentials", "value": "true" },
        { "key": "Access-Control-Allow-Origin", "value": "https://meusite.com" },
        { "key": "Access-Control-Allow-Methods", "value": "GET,POST,PUT,DELETE,OPTIONS" },
        { "key": "Access-Control-Allow-Headers", "value": "Authorization, Content-Type" }
      ]
    }
  ]
}
```

**NUNCA use `Access-Control-Allow-Origin: *` em producao.**

> Fontes: [Vercel Security Headers](https://vercel.com/docs/headers/security-headers) | [Vercel CORS Guide](https://vercel.com/kb/guide/how-to-enable-cors) | [Next.js CSP Guide](https://nextjs.org/docs/pages/guides/content-security-policy)

---

## 2.10 Pricing

### Planos (2025-2026)

| Plano | Preco | Bandwidth | Edge Requests | Funcoes | Observacoes |
|-------|-------|-----------|---------------|---------|-------------|
| **Hobby** | $0/mes | 100 GB | 1M | Limitado | NAO comercial, Fair Use |
| **Pro** | $20/user/mes | 1 TB | 10M | Escala | $20 flexible spending credit |
| **Enterprise** | ~$45K/ano | Custom | Custom | Custom | SLA, compliance |

### Mudancas de Setembro 2025

Vercel reestruturou pricing com billing baseado em creditos. Pro ganhou features que antes eram Enterprise.

### Dicas para Economizar

1. Use ISR em vez de SSR onde possivel (menos invocacoes de funcao)
2. Otimize imagens (reduz bandwidth)
3. Configure cache headers adequados
4. Use `optimizePackageImports` para reduzir bundle
5. Monitore com Speed Insights para identificar desperdicio

> Fontes: [Vercel Pricing](https://vercel.com/pricing) | [Vercel Limits](https://vercel.com/docs/limits) | [Pricing Breakdown 2026](https://checkthat.ai/brands/vercel/pricing)

---

# PARTE 3: CYBERSECURITY PARA WEB APPS

## 3.1 OWASP Top 10 (2025)

O OWASP Top 10 e o padrao global de conscientizacao sobre seguranca de aplicacoes web. A edicao 2025 traz mudancas significativas:

| # | Vulnerabilidade | Novidade 2025 | Prevalencia |
|---|----------------|---------------|-------------|
| **A01** | Broken Access Control | Mantido no #1 desde 2021 | 94% das apps testadas |
| **A02** | Security Misconfiguration | Subiu de #5 para #2 | 3% das apps testadas |
| **A03** | Software Supply Chain Failures | **EXPANDIDO** (era "Vulnerable Components") | Ataques npm massivos em 2025 |
| **A04** | Cryptographic Failures | Mantido | Algoritmos fracos, keys expostas |
| **A05** | Injection | Mantido | SQL, NoSQL, OS command, LDAP |
| **A06** | Insecure Design | Mantido | Falhas de design, nao implementacao |
| **A07** | Authentication Failures | Mantido | Brute force, session hijacking |
| **A08** | Software or Data Integrity Failures | Mantido | CI/CD inseguro, updates nao verificados |
| **A09** | Security Logging and Alerting Failures | Renomeado | Logs insuficientes, sem alertas |
| **A10** | Mishandling of Exceptional Conditions | **NOVO** | Fail-open, stack traces expostas |

### A01: Broken Access Control

**O que e:** Usuario acessa recursos que nao deveria (IDOR, escalacao de privilegios, manipulacao de JWT).

```typescript
// ERRADO: Confia no ID do parametro sem verificar autorizacao
app.get('/api/users/:id', async (req, res) => {
  const user = await db.getUser(req.params.id)
  res.json(user)
})

// CORRETO: Verifica que o usuario so acessa seus proprios dados
app.get('/api/users/:id', async (req, res) => {
  if (req.params.id !== req.user.id && !req.user.isAdmin) {
    return res.status(403).json({ error: 'Forbidden' })
  }
  const user = await db.getUser(req.params.id)
  res.json(user)
})
```

### A03: Software Supply Chain Failures (CRITICO em 2025)

Em setembro de 2025, 18 pacotes npm populares com 2.6 BILHOES de downloads semanais foram comprometidos (debug, chalk, e outros). Em marco de 2026, o pacote Axios (100M+ downloads semanais) sofreu ataque de supply chain por hackers norte-coreanos.

**Protecao:**

```bash
# 1. Auditar dependencias
npm audit --audit-level=high

# 2. Usar lockfiles E audita-los
# package-lock.json pode conter versoes maliciosas invisiveis no package.json

# 3. Pinar versoes exatas
# package.json: "axios": "1.7.2" (nao "^1.7.2")

# 4. Cooldown de 7 dias para novas versoes
# Rejeitar qualquer versao publicada ha menos de 7 dias

# 5. Verificar integridade
npm ci  # Em vez de npm install (respeita lockfile exatamente)
```

### A05: Injection

```javascript
// PROIBIDO: Concatenacao de strings em SQL
const query = `SELECT * FROM users WHERE name = '${userInput}'`

// OBRIGATORIO: Queries parametrizadas
const query = 'SELECT * FROM users WHERE name = $1'
const result = await db.query(query, [userInput])

// Supabase: ja parametrizado automaticamente
const { data } = await supabase.from('users').select('*').eq('name', userInput)
```

### A10: Mishandling of Exceptional Conditions (NOVO 2025)

```typescript
// ERRADO: Expoe stack trace e detalhes internos
app.get('/api/data', async (req, res) => {
  try {
    const data = await fetchData()
    res.json(data)
  } catch (error) {
    res.status(500).json({ error: error.stack }) // NUNCA faca isso
  }
})

// CORRETO: Fail-closed, log interno, mensagem generica
app.get('/api/data', async (req, res) => {
  try {
    const data = await fetchData()
    res.json(data)
  } catch (error) {
    console.error('Internal error:', error) // Log detalhado interno
    res.status(500).json({ error: 'Internal server error' }) // Mensagem generica
  }
})
```

### Ferramentas de Deteccao

| Tipo | Ferramenta | Detecta |
|------|-----------|---------|
| **SAST** | Semgrep, SonarQube | Injection, crypto failures, auth issues (no codigo) |
| **SCA** | Snyk, Dependabot | Componentes vulneraveis |
| **DAST** | OWASP ZAP, Burp Suite | Broken access, misconfiguration, SSRF (em runtime) |

> Fontes: [OWASP Top 10 2025](https://owasp.org/Top10/2025/) | [SecComply Analysis](https://seccomply.net/resources/blog/owasp-top-10-2025) | [GitLab OWASP Changes](https://about.gitlab.com/blog/2025-owasp-top-10-whats-changed-and-why-it-matters/)

---

## 3.2 Seguranca de Autenticacao

### Password Hashing

| Algoritmo | Recomendacao | Nota |
|-----------|-------------|------|
| **bcrypt** | Recomendado | O mais usado em Node.js |
| **Argon2id** | Ideal | Mais moderno, resistente a GPU |
| **PBKDF2** | Aceitavel | Quando bcrypt/argon2 nao disponiveis |
| **MD5/SHA1** | PROIBIDO | Inseguro, nao usar para senhas |

### JWT Best Practices

| Pratica | Detalhe |
|---------|---------|
| **Storage** | httpOnly cookies (NUNCA localStorage) |
| **Expiracao** | Access token: 5-15 min. Refresh token: 7-30 dias |
| **Algoritmo** | EdDSA (mais seguro) ou ES256. Evitar RS256 em novos projetos |
| **Secret** | Minimo 64 caracteres, gerado com fonte criptografica segura |
| **Transporte** | Somente HTTPS |
| **Validacao** | Sempre validar signature, exp, iss, aud no server |

### Session Management

- Gerar session IDs com pelo menos 64 bits de entropia
- Regenerar ID apos autenticacao (previne session fixation)
- Invalidar sessao tanto client-side quanto server-side no logout
- Definir timeout de inatividade

> Fontes: [JWT Best Practices 2025](https://jwt.app/blog/jwt-best-practices/) | [JWT Security Guide](https://guptadeepak.com/understanding-jwt-from-basics-to-advanced-security/) | [Auth Security Guide](https://clerk.com/articles/authentication-security-in-web-applications)

---

## 3.3 XSS, CSRF e Content Security Policy

### XSS (Cross-Site Scripting)

| Tipo | Descricao | Prevencao |
|------|-----------|-----------|
| **Reflected** | Payload no URL, refletido na resposta | Sanitizar input, encodar output |
| **Stored** | Payload salvo no banco, exibido para outros | Sanitizar antes de salvar, CSP |
| **DOM-based** | Manipulacao do DOM no cliente | Evitar innerHTML, usar textContent |

**Regra de ouro:** Sanitize input on arrival, encode output on render.

### CSRF (Cross-Site Request Forgery)

**Protecoes combinadas:**

1. **SameSite cookies** -- `SameSite=Strict` (ideal) ou `SameSite=Lax` (minimo)
2. **Anti-CSRF tokens** -- token unico por sessao, validado no server
3. **Verificacao de Origin/Referer header**

```typescript
// Configuracao de cookie seguro
res.cookie('session', token, {
  httpOnly: true,      // Nao acessivel via JavaScript
  secure: true,        // Somente HTTPS
  sameSite: 'strict',  // Previne CSRF
  maxAge: 3600000,     // 1 hora
  path: '/',
})
```

### Content Security Policy (CSP)

CSP e uma segunda linha de defesa contra XSS. Define quais fontes de conteudo sao permitidas:

```
Content-Security-Policy:
  default-src 'self';
  script-src 'self' 'nonce-abc123';
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: https:;
  connect-src 'self' https://*.supabase.co;
  frame-ancestors 'none';
```

**Dica:** Comece com `Content-Security-Policy-Report-Only` para monitorar violacoes sem bloquear conteudo.

> Fontes: [OWASP XSS Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html) | [OWASP CSRF Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html) | [PortSwigger XSS Guide](https://portswigger.net/web-security/cross-site-scripting)

---

## 3.4 Supply Chain Security (npm)

### Ataques Reais de 2025-2026

| Data | Pacote | Downloads Semanais | Vetor |
|------|--------|--------------------|-------|
| Set/2025 | debug, chalk + 16 outros | 2.6B total | Phishing de maintainers |
| Mar/2026 | Axios | 100M+ | Roubo de credenciais (APT norte-coreano) |

### Defesas Obrigatorias

```bash
# 1. Sempre use lockfile
npm ci  # NUNCA npm install em CI

# 2. Audite regularmente
npm audit --audit-level=high

# 3. Pinar versoes exatas no package.json
"dependencies": {
  "axios": "1.7.2"  # Exato, sem ^
}

# 4. Configure Dependabot
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"

# 5. Gitleaks para detectar secrets
npx gitleaks detect

# 6. Cooldown de 7 dias antes de adotar novas versoes
```

> Fontes: [CISA npm Alert](https://www.cisa.gov/news-events/alerts/2025/09/23/widespread-supply-chain-compromise-impacting-npm-ecosystem) | [Axios Supply Chain Attack](https://snyk.io/blog/axios-npm-package-compromised-supply-chain-attack-delivers-cross-platform/) | [Defending npm Attacks](https://www.armorcode.com/blog/defending-against-npm-supply-chain-attacks-a-practical-guide)

---

## 3.5 LGPD (Lei Geral de Protecao de Dados)

### Requisitos Tecnicos para Desenvolvedores

| Artigo | Requisito | Implementacao Tecnica |
|--------|-----------|----------------------|
| **Art. 7-8** | Consentimento explicito | Formulario opt-in antes de processar dados pessoais |
| **Art. 9** | Politica de privacidade | Pagina publica acessivel com detalhes do processamento |
| **Art. 14** | Dados de criancas | Consentimento dos pais obrigatorio |
| **Art. 18** | Direitos do titular | Portal para acesso, correcao e exclusao de dados |
| **Art. 33** | Transferencia internacional | Standard Contractual Clauses (SCCs) |
| **Art. 38** | RIPD | Data Protection Impact Assessment (DPIA) proativo |
| **Art. 41** | DPO (Encarregado) | Designar responsavel por dados pessoais |
| **Art. 46** | Medidas tecnicas | Encriptacao, RLS, acesso minimo |
| **Art. 48** | Notificacao de breach | 3 dias uteis para ANPD + titulares |

### Checklist Tecnico LGPD

```
[ ] Consentimento: formulario com opt-in explicito (nao pre-marcado)
[ ] Portal de direitos: endpoint para exportar/deletar dados do usuario
[ ] DPO designado e contactavel publicamente
[ ] Politica de privacidade publicada e acessivel
[ ] Breach notification: procedimento documentado (3 dias uteis)
[ ] Retencao de dados: periodos definidos e documentados
[ ] Logs de acesso a dados pessoais (auditoria)
[ ] Encriptacao em transito (TLS) e em repouso
[ ] DPIA realizado para processamentos de alto risco
[ ] Consentimento parental para dados de menores
```

### Prioridades de Fiscalizacao 2025-2026

A ANPD priorizou: dados de criancas, AI/biometria e data scraping. Empresas nesses setores devem esperar inspecoes.

> Fontes: [LGPD Brazil Info](https://lgpd-brazil.info/) | [DLA Piper Data Protection Brazil](https://www.dlapiperdataprotection.com/index.html?t=law&c=BR) | [ICLG Brazil Data Protection 2025-2026](https://iclg.com/practice-areas/data-protection-laws-and-regulations/brazil)

---

## 3.6 Penetration Testing

### Ferramentas Principais

| Ferramenta | Tipo | Preco | Quando Usar |
|-----------|------|-------|-------------|
| **OWASP ZAP** | DAST | Gratuito | CI/CD automation, budget limitado |
| **Burp Suite Pro** | DAST | Pago | Pen testing profissional, engagements complexos |
| **Semgrep** | SAST | Gratuito/Pago | Analise de codigo estatica, custom rules |
| **Snyk** | SCA | Gratuito/Pago | Dependencias vulneraveis |
| **Nuclei** | Vuln Scanner | Gratuito | Templates de deteccao, community-driven |

### Recomendacao Pratica

Para iniciantes, comece com:
1. **OWASP ZAP** em modo automatico contra seu app
2. **npm audit** para dependencias
3. **Semgrep** para SAST basico
4. Evolua para Burp Suite conforme amadurecer

> Fontes: [Burp vs ZAP Comparison](https://www.softwaresecured.com/post/burp-versus-zap) | [ZAP vs Burp Features](https://www.pynt.io/learning-hub/burp-suite-guides/burp-suite-vs-zap-features-key-differences-limitations)

---

## 3.7 Incident Response

### Playbook Basico para Web Apps

```
1. DETECCAO
   - Alerta do Sentry/monitoring
   - Relato de usuario
   - Scan automatizado detecta anomalia

2. CONTENCAO (Primeiros 30 minutos)
   - Preservar logs e evidencias
   - Isolar sistema comprometido
   - Revogar tokens/sessoes comprometidos
   - Ativar MFA forcado se credenciais vazaram

3. ERRADICACAO
   - Comparar codigo atual com versao known-good
   - Resetar senhas de contas admin comprometidas
   - Revogar API keys expostas
   - Corrigir vulnerabilidade explorada

4. RECUPERACAO
   - Restaurar de backup verified (se necessario)
   - Redeploy com fix aplicado
   - Monitorar intensivamente por 48h

5. NOTIFICACAO (LGPD: 3 dias uteis)
   - Notificar ANPD
   - Notificar titulares afetados
   - Documentar tudo para compliance

6. LICOES APRENDIDAS
   - Post-mortem sem culpados
   - Atualizar playbook
   - Implementar controles preventivos
```

### Documento Vivo

Playbooks devem ser revisados anualmente e imediatamente apos qualquer incidente real ou exercicio tabletop.

> Fontes: [FRSecure Web App Response Playbook](https://frsecure.com/web-application-attack-response-playbook/) | [CISA Incident Response Playbooks](https://www.cisa.gov/sites/default/files/2024-08/Federal_Government_Cybersecurity_Incident_and_Vulnerability_Response_Playbooks_508C.pdf) | [Incident Response Framework 2025](https://broadchannel.org/incident-response-framework-guide/)

---

## 3.8 Compliance: SOC 2 e ISO 27001

### Comparacao Rapida

| Aspecto | SOC 2 | ISO 27001 |
|---------|-------|-----------|
| **Origem** | EUA (AICPA) | Global (ISO/IEC) |
| **Resultado** | Relatorio de attestation | Certificacao |
| **Timeline** | Type I: 2-3 meses | 6-18 meses |
| **Custo** | $20K-$100K+ | $30K-$200K+ |
| **Foco geografico** | America do Norte | Global (Europa, Asia, LatAm) |
| **Overlap** | ~70-80% com ISO 27001 | ~70-80% com SOC 2 |

### Quando Fazer

| Situacao | Framework |
|----------|-----------|
| Vendendo para empresas americanas | SOC 2 primeiro |
| Expansao internacional | ISO 27001 |
| Ambos os mercados | Busque simultaneamente (~70% overlap) |
| Startup early-stage | Comece com SOC 2 Type I (mais rapido) |

**FINDING:** Compliance remove friccao comercial. Compradores enterprise usam SOC 2/ISO 27001 como "sinais rapidos de confianca" em vendor reviews.
**IMPLICATION:** Sem compliance, deals travam em security questionnaires.
**RECOMMENDATION:** Se vende B2B SaaS para empresas, priorize SOC 2 Type I. E um investimento que se paga em deals fechados mais rapido.

> Fontes: [SOC 2 vs ISO 27001 TrustCloud](https://www.trustcloud.ai/iso-27001/choose-soc-2-and-iso-27001/) | [SOC 2 for Startups](https://www.ispartnersllc.com/blog/soc-2-for-startups/) | [Compliance for Startups](https://www.uprootsecurity.com/blog/compliance-for-startups)

---

# PARTE 4: COLABORACAO EM SOFTWARE ENGINEERING

## 4.1 Git Workflows

### Tres Abordagens Principais

| Workflow | Branches | Complexidade | Ideal Para |
|----------|----------|-------------|------------|
| **Git Flow** | main, develop, feature, release, hotfix | Alta | Software versionado, releases estruturados |
| **GitHub Flow** | main + feature branches | Baixa | SaaS, deploy continuo, equipes pequenas |
| **Trunk-Based** | main (trunk) + feature flags | Minima | CI/CD maduros, equipes senior |

### GitHub Flow (Recomendado para a maioria em 2025)

```
1. main e SEMPRE deployavel
2. Crie branch a partir de main: feat/nova-feature
3. Faca commits pequenos e frequentes
4. Abra Pull Request
5. Code review + CI/CD checks
6. Merge para main (squash ou merge commit)
7. Deploy automatico
8. Delete branch
```

### Branch Protection (Obrigatorio)

```yaml
# Configuracoes recomendadas no GitHub:
- Require pull request reviews before merging (1+ reviewer)
- Require status checks to pass (lint, tests, type-check)
- Require branches to be up to date before merging
- Do not allow bypassing the above settings
- Restrict who can push to matching branches
```

> Fontes: [Git Workflows Comparison](https://devtoolhub.com/git-workflows-gitflow-githubflow-trunk-based/) | [Choosing Git Strategy 2025](https://abdullah.ranktriz.com/blog/29) | [Trunk vs GitFlow](https://graphite.com/guides/trunk-vs-gitflow)

---

## 4.2 Code Review

### Melhores Praticas

| Pratica | Detalhe |
|---------|---------|
| PRs pequenos | Maximo ~400 linhas de mudanca significativa |
| Review em < 24h | Nao deixe PRs esperando dias |
| Feedback construtivo | "E se fizessemos X?" vs "Isso esta errado" |
| Foco no que importa | Humanos: arquitetura, tradeoffs. Automacao: formatacao, lint |
| Checklist claro | O que verificar em cada review |

### Ferramentas Automatizadas (2025)

| Camada | Ferramenta | Funcao |
|--------|-----------|--------|
| **Formatacao** | Prettier, Black | Auto-fix estilo de codigo |
| **Linting** | ESLint, Biome | Regras de qualidade |
| **Type Check** | TypeScript | Seguranca de tipos |
| **SAST** | Semgrep, CodeQL | Analise semantica de seguranca |
| **AI Review** | CodeRabbit, Copilot | Design issues, complexidade, sugestoes |

### O que Humanos Devem Revisar (Automacao nao pega)

- Decisoes arquiteturais e tradeoffs
- Naming e clareza de abstracoes
- Edge cases nao cobertos por testes
- Performance implications
- Seguranca (logica de autorizacao)
- Alinhamento com requirements da story

> Fontes: [Code Review Best Practices 2025](https://group107.com/blog/code-review-best-practices/) | [AI Code Review Tools](https://dev.to/heraldofsolace/the-6-best-ai-code-review-tools-for-pull-requests-in-2025-4n43) | [CodeRabbit](https://www.coderabbit.ai/)

---

## 4.3 CI/CD com GitHub Actions

### Pipeline de Producao

```yaml
# .github/workflows/ci.yml
name: CI Pipeline
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  lint-and-typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck

  test:
    runs-on: ubuntu-latest
    needs: lint-and-typecheck
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      - run: npm ci
      - run: npm test -- --coverage

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm audit --audit-level=high
      - uses: returntocorp/semgrep-action@v1

  deploy-preview:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    needs: [lint-and-typecheck, test]
    steps:
      - uses: actions/checkout@v4
      # Vercel auto-deploys preview para PRs

  deploy-production:
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    needs: [lint-and-typecheck, test, security]
    environment: production
    steps:
      - uses: actions/checkout@v4
      # Vercel auto-deploys para producao ao merge em main
```

### Boas Praticas (2025)

1. **Lint e test primeiro** -- falhe rapido antes de gastar recursos com build
2. **Use `npm ci`** -- respeita lockfile exatamente (reproducibilidade)
3. **Cache agressivo** -- `actions/setup-node` com `cache: 'npm'`
4. **Environments protegidos** -- producao requer aprovacao manual
5. **Secrets no GitHub** -- NUNCA hardcode em workflows
6. **Nomes descritivos** -- workflows modulares e reutilizaveis

### Performance de GitHub Actions (2025)

Em agosto de 2025, GitHub triplicou capacidade de scheduling: 71M jobs/dia, queue times 62% menores, cold starts sub-segundo para runners cached.

> Fontes: [GitHub Actions CI/CD Tutorial](https://tech-insider.org/github-actions-ci-cd-pipeline-tutorial-2026/) | [Frontend CI/CD Pipeline 2025](https://feature-sliced.design/blog/frontend-cicd-pipeline-guide) | [GitHub Copilot Actions Best Practices](https://github.com/github/awesome-copilot/blob/main/instructions/github-actions-ci-cd-best-practices.instructions.md)

---

## 4.4 Testing Strategies

### Piramide de Testes

```
         /\
        /E2E\          10% -- Poucos, caros, lentos
       /------\
      /Integracao\      20% -- Moderados, medem interacao
     /------------\
    / Unit Tests   \    70% -- Muitos, rapidos, baratos
   /________________\
```

### Quando Usar Cada Tipo

| Tipo | O que testa | Ferramentas | Frequencia |
|------|------------|-------------|------------|
| **Unit** | Funcoes individuais, logica pura | Jest, Vitest | A cada commit |
| **Integration** | Interacao entre modulos, API + DB | Jest + Supertest, Testing Library | A cada PR |
| **E2E** | Jornadas criticas do usuario | Playwright, Cypress | Antes de merge em main |

### Distribuicao Recomendada

- **70% Unit** -- testa logica de negocio, validacoes, transformacoes
- **20% Integration** -- testa que componentes trabalham juntos corretamente
- **10% E2E** -- testa apenas jornadas criticas (login, checkout, etc.)

### Ajustes por Contexto

Para prototipos rapidos: reduza E2E, foque em unit. Para fintechs: aumente integration e E2E (seguranca critica). Para apps safety-critical: teste tudo extensivamente.

> Fontes: [Testing Pyramid Guide](https://circleci.com/blog/testing-pyramid/) | [Software Testing Pyramid 2025](https://www.devzery.com/post/software-testing-pyramid-guide-2025) | [Integration vs E2E Playbook](https://dev.to/michael_burry_00/integration-vs-e2e-system-testing-a-practical-testing-pyramid-playbook-with-real-ci-pipelines-1del)

---

## 4.5 Monitoring em Producao

### Stack Recomendado

| Camada | Ferramenta | Funcao |
|--------|-----------|--------|
| **Error Tracking** | Sentry | Captura exceptions, agrupa em issues |
| **Performance** | Sentry + Vercel Speed Insights | Metricas de performance real |
| **Uptime** | Sentry Uptime / BetterStack | Monitora disponibilidade |
| **Logs** | Vercel Logs / Supabase Logs | Logs centralizados |
| **Database** | pg_stat_statements / Supabase Reports | Performance de queries |

### Sentry Setup Basico

```typescript
// sentry.client.config.ts
import * as Sentry from '@sentry/nextjs'

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  tracesSampleRate: 0.1,  // 10% das transacoes (ajuste conforme volume)
  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0,  // 100% replay quando ha erro
})
```

### O que Monitorar

| Metrica | Threshold | Acao |
|---------|-----------|------|
| Error rate | > 1% | Investigar imediatamente |
| P95 response time | > 2s | Otimizar queries/endpoints |
| Uptime | < 99.9% | Resolver causa raiz |
| Core Web Vitals | LCP > 2.5s, CLS > 0.1 | Otimizar frontend |

### Sentry AI (2025)

Sentry Seer, o agente AI de debugging, detecta issues criticas automaticamente e sugere fixes antes do merge, alem de monitorar AI agent workflows (invocacoes, tool executions, token usage).

> Fontes: [Sentry Guide 2025](https://www.baytechconsulting.com/blog/sentry-io-comprehensive-guide-2025) | [Next.js Production Monitoring](https://eastondev.com/blog/en/posts/dev/20251220-nextjs-production-monitoring/) | [Sentry.io](https://sentry.io/)

---

## 4.6 Pair Programming com AI

### Modelo de Uso Otimo (2025)

| Ferramenta | Quando Usar | Porcentagem do Trabalho |
|-----------|-------------|------------------------|
| **GitHub Copilot** | Boilerplate, completions, regex, padroes repetitivos | 80% (dia a dia) |
| **Claude Code** | Arquitetura, debugging complexo, refactoring grande, strategic thinking | 20% (pensamento profundo) |

### Boas Praticas

1. **Trate AI como junior developer** -- produz drafts, precisa de revisao
2. **Gere codigo incrementalmente** -- nao cole blocos enormes de uma vez
3. **Teste apos cada integracao** -- rode unit tests a cada mudanca
4. **Use version control agressivamente** -- commite frequentemente para poder reverter
5. **Especifique business rules explicitamente** -- AI nao conhece suas regras de negocio
6. **Nao confie cegamente** -- valide seguranca, performance e edge cases

### Estatisticas de Adocao

- 65% dos devs usam AI tools semanalmente (Stack Overflow 2025)
- 41% de todo codigo e AI-generated ou AI-assisted
- Ganho de produtividade: 25-40% quando usado corretamente

> Fontes: [AI Coding Best Practices 2025](https://dev.to/ranndy360/ai-coding-best-practices-in-2025-4eel) | [AI Pair Programming Good Bad Ugly](https://www.builder.io/blog/ai-pair-programming) | [AI Pair Programming with Claude](https://felo.ai/blog/ai-pair-programming-claude-code/)

---

# PARTE 5: OBSIDIAN E SECOND BRAIN PARA DESENVOLVIMENTO COM AI

## 5.1 Obsidian como Second Brain

### Por que Obsidian

| Vantagem | Detalhe |
|----------|---------|
| **Gratuito** | Core app e gratis, plugins community sao gratuitos |
| **Local-first** | Arquivos Markdown no seu disco, voce e dono dos dados |
| **Formato universal** | .md legivel por qualquer editor |
| **Linkavel** | `[[nota]]` cria conexoes entre ideias (graph view) |
| **AI-friendly** | Claude Code pode ler/escrever diretamente nos arquivos |

### Metodo Zettelkasten para Devs

O Zettelkasten ("caixa de notas") e um metodo de gestao de conhecimento onde cada nota contem UMA ideia atomica e se conecta a outras notas.

Para desenvolvedores:
- **Atoms** -- notas pequenas e focadas: uma API especifica, um pattern, uma ferramenta aprendida
- **Links** -- cada atom se conecta a atoms relacionados via `[[wikilinks]]`
- **Emergence** -- padroes e insights emergem das conexoes, nao de hierarquias rigidas

### Estrutura de Vault para Devs

```
vault/
├── 000-Inbox/          # Captura rapida, nao processado
├── 100-Projects/       # Projetos ativos
│   ├── meu-saas/
│   └── freelance-x/
├── 200-Areas/          # Areas de responsabilidade
│   ├── programming/
│   ├── security/
│   └── infrastructure/
├── 300-Resources/      # Material de referencia
│   ├── supabase/
│   ├── vercel/
│   └── react/
├── 400-Archive/        # Concluido ou inativo
├── CLAUDE.md           # Contexto para Claude Code
└── memory.md           # Memoria entre sessoes
```

> Fontes: [Ultimate Zettelkasten in Obsidian](https://medium.com/@anjanj/the-ultimate-zettelkasten-system-how-i-built-a-second-brain-in-obsidian-95c29f89c6f7) | [Obsidian History](https://www.taskade.com/blog/obsidian-history) | [Zettelkasten Connected Second Brain](https://www.obsibrain.com/blog/zettelkasten-how-to-build-a-connected-second-brain-that-actually-grows-with-you)

---

## 5.2 Obsidian + AI: Construindo um "Jarvis"

### Arquitetura: Claude Code Dentro do Vault

Em vez de ir ate a AI, voce coloca a AI DENTRO do seu sistema de conhecimento.

```
1. Abra terminal na raiz do seu Obsidian vault
2. Execute: claude
3. Claude le automaticamente CLAUDE.md e memory.md
4. Claude tem acesso a TODOS os seus arquivos de conhecimento
5. Claude pode ler, escrever e modificar notas
```

### CLAUDE.md -- Contexto Persistente

```markdown
# CLAUDE.md

## Quem eu sou
- Nome: Caio Imori
- Projetos ativos: [lista de projetos]
- Stack principal: Next.js, Supabase, TypeScript

## Estrutura do vault
- 000-Inbox: captura rapida
- 100-Projects: projetos ativos
- 200-Areas: areas de expertise
- 300-Resources: material de referencia

## Regras de escrita
- Notas em portugues
- Use [[wikilinks]] para conectar ideias
- Cada nota = uma ideia atomica
- Tags: #tipo/conceito, #tipo/projeto, #tipo/ferramenta

## Preferencias
- Prefiro exemplos praticos a teoria abstrata
- Formato de code blocks com linguagem especificada
- Nao use emojis em notas tecnicas
```

### memory.md -- Continuidade Entre Sessoes

```markdown
# memory.md

## Ultima sessao (2026-04-03)
- Trabalhei no projeto SaaS: implementei RLS policies
- Decidi usar shared-table multi-tenancy com tenant_id
- Proximo passo: implementar Edge Functions para webhooks Stripe

## Decisoes importantes
- Stack: Next.js 15 + Supabase + Vercel
- Autenticacao: Google OAuth + email/senha como fallback
- MFA obrigatorio para admins

## Padroes aprendidos
- Sempre envolver auth.uid() em SELECT em policies RLS
- Usar npm ci em CI/CD (nao npm install)
```

**Atualize ao final de cada sessao:** "Salve tudo que preciso para a proxima vez em memory.md"

### Skills como Slash Commands

Apos completar workflows com sucesso, converta em SOPs reutilizaveis:

```markdown
# /pesquisar-tecnologia

## Quando usar
Quando preciso avaliar uma nova tecnologia para um projeto

## Passos
1. Buscar documentacao oficial
2. Verificar GitHub stars, issues abertas, frequencia de commits
3. Checar vulnerabilidades conhecidas (npm audit, Snyk)
4. Procurar case studies de uso em producao
5. Avaliar: learning curve, community, maturidade, lock-in
6. Criar nota em 300-Resources com conclusoes
```

### Custo

- Obsidian: Gratuito
- Claude Pro: ~R$100/mes
- MCP servers: Gratuitos, self-hosted
- **Total: ~R$100/mes** para sistema integrado com memoria persistente

> Fontes: [AI Second Brain Obsidian + Claude](https://noahvnct.substack.com/p/how-to-build-your-ai-second-brain) | [Obsidian Claude Code Workflow](https://jamesdonnelly.dev/blog/obsidian-claude-code-workflow/) | [Second Brain GTD GitHub](https://github.com/sean-esk/second-brain-gtd)

---

## 5.3 RAG (Retrieval Augmented Generation)

### O que e

RAG e a tecnica de enriquecer respostas de LLMs com dados de uma base de conhecimento externa. Em vez de depender apenas do treinamento do modelo, voce fornece contexto relevante junto com a pergunta.

### Como Funciona

```
1. Usuario faz pergunta
2. Sistema busca documentos relevantes na base de conhecimento
3. Documentos relevantes + pergunta sao enviados ao LLM
4. LLM gera resposta fundamentada nos documentos
5. Resposta pode citar fontes especificas
```

### Arquitetura Pratica com Obsidian

```
Obsidian Vault (Markdown files)
       ↓
Indexacao (embeddings via OpenAI/local model)
       ↓
Vector Database (Chroma, Pinecone, pgvector no Supabase)
       ↓
Retrieval (busca por similaridade semantica)
       ↓
Augmented Prompt (contexto + pergunta)
       ↓
LLM (Claude, GPT) gera resposta
```

### pgvector no Supabase

Supabase tem suporte nativo a pgvector, permitindo armazenar embeddings diretamente no Postgres:

```sql
-- Habilitar extensao
CREATE EXTENSION IF NOT EXISTS vector;

-- Tabela de documentos com embeddings
CREATE TABLE documents (
  id BIGSERIAL PRIMARY KEY,
  content TEXT,
  metadata JSONB,
  embedding VECTOR(1536)  -- dimensao do modelo de embedding
);

-- Busca por similaridade
SELECT content, metadata,
  1 - (embedding <=> query_embedding) AS similarity
FROM documents
ORDER BY embedding <=> query_embedding
LIMIT 5;
```

### Beneficios

| Beneficio | Detalhe |
|-----------|---------|
| **Reduce alucinacoes** | Respostas ancoradas em fontes verificaveis |
| **Conhecimento atualizado** | Nao depende do treinamento do modelo |
| **Privacidade** | Dados ficam locais ou no seu banco |
| **Citacao de fontes** | AI pode referenciar documentos especificos |
| **Sem retraining** | Atualize a base de conhecimento, nao o modelo |

### Estado em 2025

RAG evoluiu de simples text retrieval para integracao multimodal (imagens, audio), real-time e autonoma. E considerado "infraestrutura indispensavel" para AI enterprise.

> Fontes: [AWS RAG Explanation](https://aws.amazon.com/what-is/retrieval-augmented-generation/) | [NVIDIA RAG Guide](https://blogs.nvidia.com/blog/what-is-retrieval-augmented-generation/) | [RAG State 2025](https://www.ayadata.ai/the-state-of-retrieval-augmented-generation-rag-in-2025-and-beyond/) | [RAG Review 2025](https://ragflow.io/blog/rag-review-2025-from-rag-to-context)

---

# APENDICE: CHECKLIST DE PRODUCAO

## Antes de Deployar Qualquer App

### Supabase
- [ ] RLS ativado em TODAS as tabelas com dados de usuarios
- [ ] service_role NAO exposta no frontend
- [ ] Queries parametrizadas (sem string interpolation)
- [ ] Indexes em colunas usadas em RLS policies
- [ ] auth.uid() envolvido em SELECT em policies
- [ ] SSL enforced para conexoes diretas ao banco
- [ ] Email templates customizados (SMTP proprio)
- [ ] Types TypeScript gerados e atualizados

### Vercel
- [ ] Environment variables separadas por ambiente (dev/preview/prod)
- [ ] NEXT_PUBLIC_ nao contem secrets
- [ ] Security headers configurados (CSP, HSTS, X-Frame-Options)
- [ ] CORS restrito a origens conhecidas
- [ ] Image optimization com next/image
- [ ] Font optimization com next/font
- [ ] Bundle analysis realizada

### Seguranca
- [ ] OWASP Top 10 revisado para cada feature
- [ ] npm audit sem vulnerabilidades critical/high
- [ ] Dependencias pinadas em versoes exatas
- [ ] .env em .gitignore, .env.example existe
- [ ] Rate limiting em todos os endpoints publicos
- [ ] Input validation com Zod/schema em todas as entradas
- [ ] Error handling: fail-closed, mensagens genericas para usuario
- [ ] Sentry configurado para error tracking
- [ ] JWT em httpOnly cookies (nao localStorage)
- [ ] MFA ativado para contas admin

### LGPD
- [ ] Consentimento com opt-in explicito
- [ ] Portal de direitos do titular (acesso/exclusao)
- [ ] Politica de privacidade publicada
- [ ] Procedimento de breach notification documentado
- [ ] DPO designado

### CI/CD
- [ ] Pipeline: lint → typecheck → test → security audit → deploy
- [ ] Branch protection em main (review obrigatorio)
- [ ] Secrets no GitHub Secrets (nao hardcoded)
- [ ] Deploy automatico via Vercel + GitHub integration
- [ ] Environment protection para producao

---

> **Pesquisa produzida por:** Prism (Research Orchestrator), squad-research
> **Nivel de profundidade:** DEFINITIVE (Level 4 - Research Depth Pyramid)
> **Methodology:** Insight Crystallization Engine (FINDING + IMPLICATION + RECOMMENDATION)
> **Total de fontes consultadas:** 50+ (Tier 2-4 da Source Credibility Matrix)
> **Data de producao:** 2026-04-04

-- Prism, iluminando o caminho
