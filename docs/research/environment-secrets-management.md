# Environment Management, Secrets Management e Configuration-as-Code

> **Deep Research** | Atualizado: Abril 2026
> **Nivel de Profundidade:** DEEP DIVE (Pyramid Level 3)
> **Fontes:** 30+ fontes verificadas via WebSearch (Tier 2-4)

---

## Indice

1. [Arquitetura de Ambientes](#1-arquitetura-de-ambientes)
2. [Secrets Management](#2-secrets-management)
3. [Environment Variables — Melhores Praticas](#3-environment-variables--melhores-praticas)
4. [Feature Flags](#4-feature-flags)
5. [Configuration-as-Code](#5-configuration-as-code)
6. [CI/CD Environment Promotion](#6-cicd-environment-promotion)
7. [Convencoes Profissionais de Nomenclatura](#7-convencoes-profissionais-de-nomenclatura)
8. [Decision Frameworks](#8-decision-frameworks)

---

## 1. Arquitetura de Ambientes

### 1.1 Tiers Padrao de Ambientes

A arquitetura de ambientes segue uma progressao linear que espelha o ciclo de vida do software, do desenvolvimento ate a producao.

| Tier | Ambiente | Proposito | Dados | Acesso |
|------|----------|-----------|-------|--------|
| 0 | **Local** | Desenvolvimento individual | Mock/seed data | Desenvolvedor |
| 1 | **Dev** | Integracao continua | Sinteticos | Time de dev |
| 2 | **Staging** | Validacao pre-prod | Anonimizados de prod | QA + time |
| 3 | **Pre-Prod** | Testes de carga e seguranca | Copia sanitizada de prod | QA + SRE |
| 4 | **Production** | Usuarios reais | Dados reais | Todos (via app) |

**Quando usar cada tier:**

- **Startups (1-10 devs):** Local + Staging + Production (3 ambientes)
- **Scale-ups (10-50 devs):** Local + Dev + Staging + Production (4 ambientes)
- **Enterprise (50+ devs):** Todos os 5 tiers, com possibilidade de ambientes adicionais (sandbox, demo, hotfix)

### 1.2 Environment Parity — 12-Factor App

O principio de **Dev/Prod Parity** (Fator X do 12-Factor App) estabelece que os ambientes devem ser o mais similares possivel. Isso minimiza bugs que so aparecem em producao.

> *"The twelve-factor app is designed for continuous deployment by keeping the gap between development and production small."*
> -- [12factor.net/config](https://12factor.net/config)

**Os 3 gaps a minimizar:**

| Gap | Descricao | Pratica |
|-----|-----------|---------|
| **Time gap** | Tempo entre dev e deploy | Deploy em horas, nao semanas |
| **Personnel gap** | Quem escreveu vs quem deploya | Devs fazem deploy (DevOps) |
| **Tools gap** | Ferramentas diferentes por env | Mesma stack em todos os ambientes |

**Implementacao pratica:**

```yaml
# docker-compose.yml — garante paridade local/prod
services:
  app:
    build: .
    env_file: .env.local
    depends_on:
      - db
      - redis
  db:
    image: postgres:16  # mesma versao de prod
    volumes:
      - pgdata:/var/lib/postgresql/data
  redis:
    image: redis:7-alpine  # mesma versao de prod
```

**Regras de paridade:**

1. Mesma versao de banco de dados em todos os ambientes
2. Mesma versao de runtime (Node.js, Python, etc.)
3. Mesmos servicos de backing (Redis, S3, etc.) — evitar SQLite local vs PostgreSQL em prod
4. Mesma estrutura de environment variables (nomes identicos, valores diferentes)

### 1.3 Gerenciamento de Dados entre Ambientes

| Ambiente | Fonte de Dados | Regras |
|----------|----------------|--------|
| Local | Seed scripts / fixtures | Dados ficticios, nunca copiar prod |
| Dev | Seeds automatizados | Reset diario, dados sinteticos |
| Staging | Snapshot anonimizado de prod | PII removida (LGPD Art. 18), atualizado semanalmente |
| Pre-Prod | Copia sanitizada com volume real | Mesmo volume de prod para testes de carga |
| Production | Dados reais | Backup automatico, PITR habilitado |

**Sanitizacao de dados para staging (exemplo PostgreSQL):**

```sql
-- Anonimizar dados sensiveis ao copiar para staging
UPDATE users SET
  email = 'user_' || id || '@staging.local',
  phone = '+5511900000000',
  cpf = '000.000.000-00',
  name = 'User ' || id
WHERE environment = 'staging';
```

### 1.4 Preview / Ephemeral Environments

Preview environments sao ambientes temporarios criados automaticamente para cada pull request, permitindo testar mudancas isoladamente antes do merge.

**Comparativo de plataformas (2026):**

| Plataforma | Preview por PR | Banco Isolado | Cleanup Automatico | Custo |
|------------|:-:|:-:|:-:|-------|
| **Vercel** | Sim | Nao (mesmo banco) | Sim | Incluso no plano |
| **Netlify** | Sim | Nao | Sim | Incluso no plano |
| **Railway** | Sim | Sim (fork do banco) | Sim (ao deletar branch) | Pay-per-use |
| **Render** | Sim | Nao (manual) | Configuravel | Plano pago |
| **Northflank** | Sim | Sim | Sim | Plano pago |

> *"Feature branches get ephemeral environments with fresh databases for every PR, and compute is billed on actual usage so ephemeral preview environments cost almost nothing if reviewers close PRs promptly."*
> -- [Northflank Blog](https://northflank.com/blog/preview-environment-platforms)

**Railway — ambiente efemero com banco isolado:**

Railway e particularmente forte nesse cenario: qualquer branch nova no repositorio vinculado dispara um deploy dentro de um novo ambiente nomeado com o nome do branch, e o ambiente e deletado automaticamente quando o branch e deletado.

**Vercel — preview deployments:**

Cada PR no Vercel recebe uma URL unica de preview (`https://project-git-branch-user.vercel.app`). A variavel `VERCEL_ENV` retorna `"preview"` nesses deploys, permitindo logica condicional.

```typescript
// Detectar ambiente em runtime
const isPreview = process.env.VERCEL_ENV === 'preview';
const isProduction = process.env.VERCEL_ENV === 'production';

// Usar banco/API diferente em preview
const apiUrl = isPreview
  ? process.env.STAGING_API_URL
  : process.env.PRODUCTION_API_URL;
```

---

## 2. Secrets Management

### 2.1 Conceitos Fundamentais

**Secret** e qualquer credencial que, se exposta, compromete a seguranca do sistema: API keys, database credentials, tokens OAuth, certificados TLS, chaves de criptografia.

**Principios inegociaveis:**

1. Secrets NUNCA no codigo-fonte (hardcoded)
2. Secrets NUNCA no git (mesmo em branches privadas)
3. Rotacao automatica sempre que possivel
4. Principio de menor privilegio (least privilege)
5. Auditoria de acesso a cada secret

### 2.2 Comparativo de Ferramentas (2026)

| Criterio | HashiCorp Vault | AWS Secrets Manager | AWS SSM Parameter Store | Doppler | Infisical |
|----------|:-:|:-:|:-:|:-:|:-:|
| **Tipo** | Self-hosted / HCP Cloud | Managed (AWS) | Managed (AWS) | Cloud-only | Open Source / Cloud |
| **Licenca** | BSL (ex-open source) | Proprietaria | Proprietaria | Proprietaria | MIT (core) |
| **Dynamic Secrets** | Sim (maduro) | Nao (rotation-based) | Nao | Beta (2026) | Em desenvolvimento |
| **Auto-Rotation** | Sim (schedule-based) | Sim (Lambda nativo) | Nao | Sim | Sim |
| **Encryption-as-Service** | Sim (Transit engine) | Nao | Nao | Nao | Nao |
| **PKI/Certificados** | Sim | Nao | Nao | Nao | Sim |
| **Multi-Cloud** | Sim | Somente AWS | Somente AWS | Sim | Sim |
| **UI Amigavel** | Basica | Console AWS | Console AWS | Excelente | Boa |
| **CLI** | Sim | AWS CLI | AWS CLI | Sim (DX excelente) | Sim |
| **SDKs** | 10+ linguagens | AWS SDKs | AWS SDKs | 12+ linguagens | 10+ linguagens |
| **K8s Operator** | Sim | Nao (usa ESO) | Nao (usa ESO) | Sim | Sim |
| **Self-Hosting** | Sim | Nao | Nao | Nao | Sim |
| **Versioning** | Sim | Sim | Limitado | Sim | Sim |
| **Cross-Region** | Sim | Sim (replicacao) | Nao | N/A | Sim |
| **Max Secret Size** | Ilimitado | 64 KB | 4 KB (std) / 8 KB (adv) | Ilimitado | Ilimitado |

#### Pricing (2026)

| Ferramenta | Free Tier | Entrada Paga | Enterprise |
|------------|-----------|-------------|------------|
| **HashiCorp Vault** | Community Edition (self-hosted) | HCP Dev: ~$0.03/hr | HCP Standard: ~$1.58/hr + $0.128/client/hr |
| **AWS Secrets Manager** | Nenhum | $0.40/secret/mes + $0.05/10K API calls | Mesmo (pay-per-use) |
| **AWS SSM Parameter Store** | Standard: gratis | Advanced: $0.05/parametro/mes | Mesmo |
| **Doppler** | Ate 3 usuarios | Team: $12/user/mes | Enterprise: custom (~$50/dev/mes) |
| **Infisical** | Self-hosted ilimitado | Cloud Pro: $8/user/mes | Enterprise: custom |

> Fonte: [Top 5 Secrets Management Tools of 2026](https://guptadeepak.com/top-5-secrets-management-tools-hashicorp-vault-aws-doppler-infisical-and-azure-key-vault-compared/), [Infisical Pricing](https://infisical.com/pricing), [Doppler Vendr](https://www.vendr.com/marketplace/doppler), [AWS Secrets Manager Pricing](https://costbench.com/software/secrets-management/aws-secrets-manager/), [Vault Pricing](https://infisical.com/blog/hashicorp-vault-pricing)

### 2.3 Decision Framework — Qual Ferramenta Escolher

```
Voce esta 100% na AWS?
|-- SIM -> Precisa de rotation automatica?
|   |-- SIM -> AWS Secrets Manager
|   +-- NAO -> AWS SSM Parameter Store (gratis)
+-- NAO -> Precisa de dynamic secrets / PKI?
    |-- SIM -> HashiCorp Vault (HCP ou self-hosted)
    +-- NAO -> Prioridade e DX e velocidade?
        |-- SIM -> Doppler (cloud, zero infra)
        +-- NAO -> Prefere self-hosted e open source?
            |-- SIM -> Infisical
            +-- NAO -> Doppler
```

**Recomendacao por perfil (2026):**

| Perfil | Recomendacao | Justificativa |
|--------|-------------|---------------|
| Startup ate 5 devs | Doppler (free) ou Infisical (self-hosted) | Setup em minutos, DX excelente |
| Time 5-20 devs, multi-cloud | Infisical Cloud ou Doppler Team | Integracao com CI/CD, custo acessivel |
| Enterprise, compliance pesado | HashiCorp Vault (HCP Dedicated) | Dynamic secrets, PKI, audit granular |
| 100% AWS | SSM Parameter Store + Secrets Manager | Integracao nativa, zero overhead |

### 2.4 Secret Rotation — Automacao

A rotacao automatica de secrets e uma das praticas mais criticas para seguranca. Credenciais estaticas sao o vetor de ataque mais explorado.

**Padroes de rotacao:**

| Padrao | Descricao | Ferramenta |
|--------|-----------|------------|
| **Scheduled Rotation** | Rotaciona em intervalo fixo (30/60/90 dias) | AWS Secrets Manager, Vault, Doppler |
| **Dynamic Secrets** | Credencial gerada on-demand, com TTL curto | HashiCorp Vault |
| **Event-Driven** | Rotaciona apos evento (breach, offboarding) | Qualquer ferramenta |

**AWS Secrets Manager — rotation nativa para RDS:**

```python
# AWS Secrets Manager rotation com Lambda
# A AWS fornece Lambda functions pre-construidas para RDS, Redshift, DocumentDB

import boto3

client = boto3.client('secretsmanager')

# Habilitar rotation automatica a cada 30 dias
client.rotate_secret(
    SecretId='prod/db/main-credentials',
    RotationRules={
        'AutomaticallyAfterDays': 30,
        'Duration': '2h',
        'ScheduleExpression': 'rate(30 days)'
    }
)
```

> *"AWS provides pre-built rotation Lambda functions that handle the entire rotation workflow including updating the database with the new credentials and verifying connectivity before old credentials are invalidated."*
> -- [AWS Docs](https://aws.amazon.com/blogs/security/how-to-choose-the-right-aws-service-for-managing-secrets-and-configurations/)

**HashiCorp Vault — dynamic secrets para PostgreSQL:**

```bash
# Configurar database secrets engine
vault secrets enable database

# Configurar conexao (credenciais reais ficam no Vault, nao aqui)
vault write database/config/mydb \
  plugin_name=postgresql-database-plugin \
  connection_url="postgresql://{{username}}:{{password}}@db-host:5432/app" \
  allowed_roles="app-role" \
  username="vault_admin"

# Criar role com TTL de 1 hora
vault write database/roles/app-role \
  db_name=mydb \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' \
    VALID UNTIL '{{expiration}}'; \
    GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"

# Aplicacao solicita credencial temporaria
vault read database/creds/app-role
# Retorna: username=v-app-role-xyz, lease_duration=3600s
# Credencial expira automaticamente apos 1h
```

> Fonte: [HashiCorp Vault Database Secrets](https://developer.hashicorp.com/vault/docs/secrets/databases), [Auto Rotation HCP](https://developer.hashicorp.com/hcp/docs/vault-secrets/auto-rotation)

---

## 3. Environment Variables — Melhores Praticas

### 3.1 Convencoes de Nomenclatura

**Formato padrao: `PREFIX_SERVICE_KEY`**

```bash
# Padrao: SCREAMING_SNAKE_CASE
# Estrutura: [SCOPE]_[SERVICO]_[RECURSO]

# Database
DATABASE_URL=<connection-string>
DATABASE_POOL_SIZE=20
DATABASE_SSL_MODE=require

# APIs externas
STRIPE_SECRET_KEY=<your-stripe-secret-key>
STRIPE_WEBHOOK_SECRET=<your-webhook-secret>
SENDGRID_API_KEY=<your-sendgrid-key>

# Servicos internos
AUTH_JWT_SECRET=<your-jwt-secret-min-32-chars>
AUTH_JWT_EXPIRY=3600
AUTH_REFRESH_TOKEN_EXPIRY=604800

# Feature toggles
FEATURE_NEW_CHECKOUT=true
FEATURE_DARK_MODE=false

# Infraestrutura
REDIS_URL=<your-redis-url>
S3_BUCKET_NAME=my-app-uploads
S3_REGION=us-east-1
```

**Regras de nomenclatura:**

| Regra | Bom | Ruim |
|-------|-----|------|
| SCREAMING_SNAKE_CASE | `DATABASE_URL` | `databaseUrl`, `database-url` |
| Prefixo por servico | `STRIPE_SECRET_KEY` | `SECRET_KEY` |
| Sem abreviacoes obscuras | `DATABASE_POOL_SIZE` | `DB_PS` |
| Sem dados no nome | `STRIPE_SECRET_KEY` | `STRIPE_SK_LIVE_4242` |
| Sufixo de tipo quando ambiguo | `AUTH_PORT` (numero), `AUTH_ENABLED` (bool) | `AUTH` |

### 3.2 Gerenciamento de Arquivos .env

**Estrutura recomendada:**

```
project/
  .env.example       # Template com placeholders (COMMITADO)
  .env.local         # Overrides locais do dev (GITIGNORED)
  .env.development   # Defaults de dev (pode commitar se nao tiver secrets)
  .env.staging       # Valores de staging (GITIGNORED ou em secrets manager)
  .env.production    # NUNCA commitar — vive no secrets manager
  .env.test          # Valores para testes automatizados (pode commitar)
  .gitignore         # DEVE conter: .env*.local, .env.staging, .env.production
```

**Exemplo de `.env.example`:**

```bash
# .env.example — Template para novos desenvolvedores
# Copie para .env.local e preencha os valores

# === Database ===
DATABASE_URL=<your-database-connection-string>

# === Auth ===
AUTH_JWT_SECRET=<generate-with-openssl-rand-hex-32>
AUTH_JWT_EXPIRY=3600

# === External APIs ===
STRIPE_SECRET_KEY=<your-stripe-test-key>
STRIPE_WEBHOOK_SECRET=<your-webhook-secret>

# === Feature Flags ===
FEATURE_NEW_CHECKOUT=false
```

**Regras inegociaveis para `.env`:**

1. `.env` e `.env*.local` SEMPRE no `.gitignore`
2. `.env.example` SEMPRE commitado, com valores placeholder
3. Secrets NUNCA em `.env.example` (nem valores de teste reais)
4. Novos devs copiam `.env.example` para `.env.local` como primeiro passo

### 3.3 NEXT_PUBLIC_ — Perigos e Regras

No Next.js, variaveis prefixadas com `NEXT_PUBLIC_` sao incluidas no bundle JavaScript enviado ao browser. Isso significa que qualquer pessoa pode ve-las inspecionando o codigo-fonte da pagina.

**O que PODE ter NEXT_PUBLIC_:**

```bash
NEXT_PUBLIC_APP_URL=https://meuapp.com
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=<pk_live_or_test>  # publishable key e publica por design
NEXT_PUBLIC_SUPABASE_URL=https://xxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=<anon-key>              # anon key respeita RLS
NEXT_PUBLIC_GA_TRACKING_ID=G-XXXXXXX
NEXT_PUBLIC_SENTRY_DSN=https://xxx@sentry.io/123
```

**O que NUNCA pode ter NEXT_PUBLIC_:**

```bash
# PROIBIDO — expoe no browser!
# NEXT_PUBLIC_DATABASE_URL         -> acesso direto ao banco
# NEXT_PUBLIC_STRIPE_SECRET_KEY    -> pode fazer cobracas
# NEXT_PUBLIC_SUPABASE_SERVICE_ROLE -> bypassa RLS
# NEXT_PUBLIC_JWT_SECRET           -> pode forjar tokens
# NEXT_PUBLIC_AWS_SECRET_ACCESS_KEY -> acesso total a AWS
```

> *"In Next.js, only variables prefixed with NEXT_PUBLIC_ are available in browser code, and secret keys should never be prefixed with NEXT_PUBLIC_."*
> -- [Vercel Docs](https://vercel.com/docs/environment-variables)

**Server-only vs Client-safe:**

| Tipo | Prefixo | Onde Roda | Exemplo |
|------|---------|-----------|---------|
| Server-only | Sem prefixo | Server Components, API Routes, Middleware | `DATABASE_URL`, `STRIPE_SECRET_KEY` |
| Client-safe | `NEXT_PUBLIC_` | Browser + Server | `NEXT_PUBLIC_APP_URL`, `NEXT_PUBLIC_ANON_KEY` |

### 3.4 Validacao na Inicializacao

Validar environment variables no startup da aplicacao previne erros em runtime que sao dificeis de debugar.

**t3-env (recomendado para Next.js):**

```typescript
// src/env.ts — usando @t3-oss/env-nextjs + zod
import { createEnv } from "@t3-oss/env-nextjs";
import { z } from "zod";

export const env = createEnv({
  // Variaveis server-only
  server: {
    DATABASE_URL: z.string().url(),
    AUTH_JWT_SECRET: z.string().min(32),
    STRIPE_SECRET_KEY: z.string().startsWith("sk_"),
    STRIPE_WEBHOOK_SECRET: z.string().startsWith("whsec_"),
    NODE_ENV: z.enum(["development", "staging", "production"]),
    REDIS_URL: z.string().url().optional(),
  },

  // Variaveis publicas (browser)
  client: {
    NEXT_PUBLIC_APP_URL: z.string().url(),
    NEXT_PUBLIC_SUPABASE_URL: z.string().url(),
    NEXT_PUBLIC_SUPABASE_ANON_KEY: z.string().min(1),
    NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY: z.string().startsWith("pk_"),
  },

  // Necessario para Next.js — destructuring manual
  runtimeEnv: {
    DATABASE_URL: process.env.DATABASE_URL,
    AUTH_JWT_SECRET: process.env.AUTH_JWT_SECRET,
    STRIPE_SECRET_KEY: process.env.STRIPE_SECRET_KEY,
    STRIPE_WEBHOOK_SECRET: process.env.STRIPE_WEBHOOK_SECRET,
    NODE_ENV: process.env.NODE_ENV,
    REDIS_URL: process.env.REDIS_URL,
    NEXT_PUBLIC_APP_URL: process.env.NEXT_PUBLIC_APP_URL,
    NEXT_PUBLIC_SUPABASE_URL: process.env.NEXT_PUBLIC_SUPABASE_URL,
    NEXT_PUBLIC_SUPABASE_ANON_KEY: process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY,
    NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY: process.env.NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY,
  },
});

// Uso no codigo:
// import { env } from "@/env";
// const dbUrl = env.DATABASE_URL; // tipo seguro, validado
```

> Fonte: [T3 Env Docs](https://env.t3.gg/docs/nextjs), [Create T3 App](https://create.t3.gg/en/usage/env-variables)

**envalid (para Node.js generico):**

```typescript
// src/config.ts — usando envalid
import { cleanEnv, str, port, url, bool, num } from "envalid";

export const config = cleanEnv(process.env, {
  DATABASE_URL: url({ desc: "PostgreSQL connection string" }),
  PORT: port({ default: 3000 }),
  NODE_ENV: str({
    choices: ["development", "staging", "production"],
    default: "development"
  }),
  REDIS_URL: url({ default: undefined }),
  ENABLE_CACHE: bool({ default: true }),
  RATE_LIMIT_MAX: num({ default: 100 }),
  JWT_SECRET: str({ desc: "Secret for JWT signing (min 32 chars)" }),
});

// config e tipado e validado no startup
// Se faltar variavel obrigatoria, a app NAO inicia
```

**Zod standalone (para qualquer framework):**

```typescript
// src/env-schema.ts
import { z } from "zod";

const envSchema = z.object({
  DATABASE_URL: z.string().url(),
  PORT: z.coerce.number().default(3000),
  NODE_ENV: z.enum(["development", "staging", "production"]),
  JWT_SECRET: z.string().min(32, "JWT_SECRET must be at least 32 characters"),
  API_KEY: z.string().min(1),
});

// Valida e exporta — crash no startup se invalido
export const env = envSchema.parse(process.env);
export type Env = z.infer<typeof envSchema>;
```

### 3.5 Padroes Vercel e Supabase

**Vercel — env var scoping:**

```
Variavel X:
  Production: "valor_prod"   -> deploy de main
  Preview: "valor_staging"   -> deploy de PRs
  Development: "valor_dev"   -> vercel dev local
```

Variaveis no Vercel podem ser marcadas como **Sensitive** (valor oculto apos salvar) e filtradas por **branch** (e.g., so aplicar em branches com prefixo `feat/`).

**Supabase — secrets para Edge Functions:**

```bash
# Definir secrets para Edge Functions
supabase secrets set STRIPE_SECRET_KEY=<your-key>
supabase secrets set RESEND_API_KEY=<your-key>

# Listar secrets
supabase secrets list

# No codigo da Edge Function:
# Deno.env.get("STRIPE_SECRET_KEY");  // acessivel apenas server-side

# Variaveis pre-populadas automaticamente:
# SUPABASE_URL — API gateway
# SUPABASE_ANON_KEY — chave publica (respeita RLS)
# SUPABASE_SERVICE_ROLE_KEY — chave admin (bypassa RLS, NUNCA no browser)
```

> Fonte: [Supabase Edge Functions Secrets](https://supabase.com/docs/guides/functions/secrets), [Supabase Config Management](https://supabase.com/docs/guides/local-development/managing-config)

**Helper para validacao em Edge Functions:**

```typescript
// supabase/functions/_shared/env.ts
export function requireEnv(name: string): string {
  const value = Deno.env.get(name);
  if (!value) {
    throw new Error(`Missing required environment variable: ${name}`);
  }
  return value;
}

// Uso:
const stripeKey = requireEnv("STRIPE_SECRET_KEY");
```

---

## 4. Feature Flags

### 4.1 Conceitos Fundamentais

Feature flags (ou feature toggles) permitem ativar/desativar funcionalidades em runtime sem deploy. Existem diferentes tipos com diferentes ciclos de vida.

**Tipos de feature flags:**

| Tipo | Ciclo de Vida | Exemplo | Owner |
|------|--------------|---------|-------|
| **Release Flag** | Dias-semanas (remover apos 100%) | `flag_new_checkout` | Engenharia |
| **Experiment Flag** | Semanas-meses (A/B test) | `exp_pricing_page_v2` | Produto |
| **Ops Flag** | Permanente | `ops_enable_cache` | SRE/Ops |
| **Kill Switch** | Permanente | `kill_external_payment` | SRE/Ops |
| **Permission Flag** | Permanente | `perm_beta_access` | Produto |

> *"Release flags must die within 30 days of reaching 100% — enforce in CI."*
> -- [CodeWithSeb Feature Flags Guide](https://www.codewithseb.com/blog/feature-flags-progressive-rollouts-nextjs-guide)

### 4.2 Comparativo de Plataformas (2026)

| Criterio | LaunchDarkly | Unleash | GrowthBook | Statsig |
|----------|:-:|:-:|:-:|:-:|
| **Tipo** | SaaS | Open Source / Cloud | Open Source / Cloud | SaaS |
| **Licenca** | Proprietaria | Apache 2.0 | MIT (core) | Proprietaria |
| **Self-Hosting** | Nao | Sim | Sim | Nao |
| **SDKs** | 35+ linguagens | 15+ linguagens | 15+ linguagens | 20+ linguagens |
| **Boolean Flags** | Sim | Sim | Sim | Sim |
| **Multivariate** | Sim | Sim | Sim | Sim |
| **% Rollout** | Sim | Sim | Sim | Sim |
| **User Targeting** | Avancado | Basico-Medio | Medio | Avancado |
| **Experimentation** | Add-on pago | Nao nativo | Nativo (forte) | Nativo (forte) |
| **Approval Workflows** | Sim (Enterprise) | Sim (Enterprise) | Nao | Nao |
| **Audit Log** | Sim | Sim | Sim | Sim |
| **Governance/RBAC** | Avancado | Basico | Basico | Medio |
| **Data Warehouse Native** | Nao | Nao | Sim (BigQuery, Snowflake) | Sim |
| **Maturity** | Mais maduro (2014) | Maduro (2015) | Medio (2021) | Medio (2021) |

#### Pricing (2026)

| Plataforma | Free Tier | Entrada Paga | Enterprise |
|------------|-----------|-------------|------------|
| **LaunchDarkly** | Developer (limitado) | Foundation: $12/seat/mes | Custom ($15K-$150K+/ano) |
| **Unleash** | Open Source (ilimitado) | Enterprise: $75/seat/mes (min 5) | Custom (anual) |
| **GrowthBook** | Self-hosted (ilimitado) | Pro: por features, sem per-seat | Enterprise: custom |
| **Statsig** | Developer: 2M events/mes | Pro: 5M events/mes + $0.05/1K extra | Custom |

> Fonte: [LaunchDarkly Pricing](https://launchdarkly.com/pricing/), [Unleash Pricing](https://www.getunleash.io/pricing), [GrowthBook Pricing](https://www.growthbook.io/pricing), [Statsig Pricing](https://www.statsig.com/pricing)

**Decision Framework — Feature Flags:**

```
Precisa de experimentacao (A/B tests)?
|-- SIM -> Quer warehouse-native analytics?
|   |-- SIM -> GrowthBook (open source + warehouse)
|   +-- NAO -> Statsig (free tier generoso + experiments nativos)
+-- NAO -> Prioridade e governance/compliance?
    |-- SIM -> LaunchDarkly (enterprise-grade, mais maduro)
    +-- NAO -> Quer self-host?
        |-- SIM -> Unleash (open source, full control)
        +-- NAO -> Statsig (free unlimited flags)
```

### 4.3 Padroes de Rollout

**Boolean flag (on/off):**

```typescript
// Simples: ativado para todos ou ninguem
if (getFlag("feature_new_checkout")) {
  return <NewCheckout />;
}
return <OldCheckout />;
```

**Percentage rollout (canary):**

```typescript
// Gradual: 5% -> 25% -> 50% -> 100%
// Usa hash deterministico do userId (nao Math.random()!)
function isInRollout(userId: string, percentage: number): boolean {
  const hash = murmurhash3(userId + "feature_new_checkout");
  return (hash % 100) < percentage;
}
```

**Multivariate flag:**

```typescript
// Teste com multiplas variantes
const variant = getFlag("exp_pricing_layout");
// Retorna: "control" | "variant_a" | "variant_b"

switch (variant) {
  case "variant_a":
    return <PricingLayoutA />;
  case "variant_b":
    return <PricingLayoutB />;
  default:
    return <PricingLayoutControl />;
}
```

**Kill switch (circuit breaker):**

```typescript
// Kill switch para dependencia externa
// Se Stripe falhar, desabilita checkout temporariamente
const isPaymentEnabled = getFlag("kill_switch_payments");

if (!isPaymentEnabled) {
  return <MaintenancePage message="Pagamentos temporariamente indisponiveis" />;
}

// Kill switches devem SEMPRE defaultar para ON (fail-open)
// E serem acionaveis em < 30 segundos
```

### 4.4 Implementacao em Next.js

**Server Component com feature flag:**

```typescript
// app/checkout/page.tsx — Server Component
import { getFlag } from "@/lib/feature-flags";

export default async function CheckoutPage() {
  const useNewCheckout = await getFlag("feature_new_checkout");

  if (useNewCheckout) {
    return <NewCheckoutServer />;
  }
  return <LegacyCheckoutServer />;
}
```

**Middleware para routing A/B:**

```typescript
// middleware.ts — avaliacao no edge (zero flicker)
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";
import { evaluateFlag } from "@/lib/feature-flags-edge";

export async function middleware(request: NextRequest) {
  const userId = request.cookies.get("userId")?.value ?? "anonymous";
  const variant = await evaluateFlag("exp_landing_page", userId);

  if (variant === "variant_b" && request.nextUrl.pathname === "/") {
    return NextResponse.rewrite(new URL("/landing-b", request.url));
  }

  return NextResponse.next();
}

export const config = {
  matcher: ["/"],
};
```

**Client Component com Statsig (free):**

```typescript
// providers/statsig-provider.tsx
"use client";
import { StatsigProvider } from "statsig-react";

export function FeatureFlagProvider({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <StatsigProvider
      sdkKey={process.env.NEXT_PUBLIC_STATSIG_CLIENT_KEY!}
      user={{ userID: getUserId() }}
      waitForInitialization={true}
    >
      {children}
    </StatsigProvider>
  );
}

// components/NewFeature.tsx
"use client";
import { useGate } from "statsig-react";

export function ConditionalFeature() {
  const { value: isEnabled } = useGate("new_feature_gate");

  if (!isEnabled) return null;
  return <NewFeatureComponent />;
}
```

---

## 5. Configuration-as-Code

### 5.1 Principios

Configuration-as-Code (CaC) trata configuracoes de aplicacao como codigo versionado, revisado e testado — nao como valores soltos em dashboards ou arquivos nao rastreados.

**Principios centrais:**

1. **Versionado** — toda config mora no git (exceto secrets)
2. **Validado** — schema validation antes de aplicar
3. **Imutavel** — config gerada no build, nao editada em runtime
4. **Auditavel** — quem mudou, quando, por que (git log)
5. **Revisavel** — mudancas de config passam por PR review

### 5.2 Runtime vs Build-Time Config

| Aspecto | Build-Time | Runtime |
|---------|-----------|---------|
| **Quando resolve** | `next build` / `npm run build` | Quando a app inicia ou durante execucao |
| **Mutavel em prod** | Nao (requer redeploy) | Sim (sem redeploy) |
| **Performance** | Melhor (inlined no bundle) | Leitura adicional |
| **Uso** | URLs publicas, feature flags estaticos | Secrets, feature flags dinamicos |
| **Seguranca** | Visivel no bundle se client-side | Protegido no server |
| **Next.js** | `NEXT_PUBLIC_*` no `.env` | `serverRuntimeConfig` / env vars |

```typescript
// next.config.ts — separacao build-time vs runtime
const nextConfig = {
  // Build-time: inlined no bundle JavaScript
  env: {
    NEXT_PUBLIC_APP_VERSION: process.env.npm_package_version,
    NEXT_PUBLIC_BUILD_ID: process.env.VERCEL_GIT_COMMIT_SHA?.slice(0, 7),
  },

  // Runtime: lido no server a cada request
  serverRuntimeConfig: {
    databaseUrl: process.env.DATABASE_URL,
    jwtSecret: process.env.AUTH_JWT_SECRET,
  },

  // Runtime publico: lido no server mas acessivel no client via getServerSideProps
  publicRuntimeConfig: {
    apiUrl: process.env.NEXT_PUBLIC_API_URL,
  },
};
```

### 5.3 Schemas de Validacao

```typescript
// config/app-config.schema.ts
import { z } from "zod";

export const appConfigSchema = z.object({
  app: z.object({
    name: z.string(),
    version: z.string().regex(/^\d+\.\d+\.\d+$/),
    environment: z.enum(["development", "staging", "production"]),
  }),

  database: z.object({
    host: z.string(),
    port: z.number().min(1).max(65535),
    name: z.string(),
    poolSize: z.number().min(1).max(100).default(20),
    ssl: z.boolean().default(false),
  }),

  cache: z.object({
    enabled: z.boolean().default(true),
    ttlSeconds: z.number().min(0).default(3600),
    provider: z.enum(["redis", "memory", "none"]).default("memory"),
  }),

  rateLimit: z.object({
    windowMs: z.number().default(900000),     // 15 min
    maxRequests: z.number().default(100),
    authMaxRequests: z.number().default(5),    // mais restrito para auth
  }),

  features: z.object({
    newCheckout: z.boolean().default(false),
    darkMode: z.boolean().default(false),
    maintenanceMode: z.boolean().default(false),
  }),
});

export type AppConfig = z.infer<typeof appConfigSchema>;
```

```yaml
# config/app-config.production.yaml
app:
  name: "MyApp"
  version: "2.1.0"
  environment: "production"

database:
  host: "${DATABASE_HOST}"          # interpolacao de env var
  port: 5432
  name: "myapp_prod"
  poolSize: 50
  ssl: true

cache:
  enabled: true
  ttlSeconds: 3600
  provider: "redis"

rateLimit:
  windowMs: 900000
  maxRequests: 100
  authMaxRequests: 5

features:
  newCheckout: true
  darkMode: true
  maintenanceMode: false
```

### 5.4 Drift Detection

Configuration drift ocorre quando o estado real de um sistema diverge do estado declarado no codigo.

**Tipos de drift:**

| Tipo | Causa | Exemplo |
|------|-------|---------|
| **Manual change** | Alguem editou config direto no dashboard | Mudou env var no Vercel sem PR |
| **Partial deploy** | Deploy falhou no meio | Metade dos pods com config nova |
| **External mutation** | Servico externo mudou config | Terraform state vs AWS real |
| **Version skew** | Config de versao antiga rodando | Feature flag schema v1 vs v2 |

**Estrategias de deteccao:**

```yaml
# .github/workflows/drift-detection.yml
name: Config Drift Detection
on:
  schedule:
    - cron: '0 */6 * * *'   # a cada 6 horas
  workflow_dispatch:

jobs:
  detect-drift:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Check Terraform drift
        run: |
          terraform plan -detailed-exitcode -out=plan.out
          # Exit code 2 = drift detected
        continue-on-error: true

      - name: Compare Vercel env vars
        run: |
          vercel env pull .env.vercel-current --yes
          diff .env.expected .env.vercel-current || echo "DRIFT DETECTED"

      - name: Alert on drift
        if: failure()
        uses: slackapi/slack-github-action@v2
        with:
          payload: |
            {"text": "Configuration drift detected! Check workflow run."}
```

**GitOps — reconciliacao automatica:**

ArgoCD e Flux implementam deteccao de drift automatica para Kubernetes: comparam o estado desejado (no Git) com o estado real (no cluster) e podem corrigir automaticamente.

> *"Argo CD can automatically sync an application when it detects differences between the desired manifests in Git and the live state in the cluster."*
> -- [Spacelift Config Drift](https://spacelift.io/blog/what-is-configuration-drift)

**Ferramentas de drift detection (2025-2026):**

| Ferramenta | Escopo | Tipo |
|------------|--------|------|
| **Driftctl** | Terraform / AWS | Open source |
| **ArgoCD** | Kubernetes | Open source (GitOps) |
| **Flux** | Kubernetes | Open source (GitOps) |
| **Spacelift** | Terraform, Pulumi, CloudFormation | Comercial |
| **Puppet/Chef** | Infraestrutura generica | Open source / Comercial |
| **Terragrunt** | Terraform multi-env | Open source |
| **KubeDiff** | Kubernetes | Open source |

> Fonte: [AI Infra Link — Config Drift Detection 2025](https://www.ai-infra-link.com/mastering-config-drift-detection-top-open-source-tools-for-2025/)

---

## 6. CI/CD Environment Promotion

### 6.1 Fluxo de Promocao Multi-Ambiente

O padrao mais comum em GitHub Actions usa **environments** nativos com **protection rules** para controlar a promocao entre ambientes.

```
PR merge -> Dev (auto) -> Staging (auto + tests) -> Production (manual approval)
```

**Workflow completo:**

```yaml
# .github/workflows/deploy.yml
name: Deploy Pipeline
on:
  push:
    branches: [main]

jobs:
  # === FASE 1: Build e Testes ===
  build-and-test:
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
      - run: npm test -- --coverage

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: build
          path: .next/
          retention-days: 1

  # === FASE 2: Deploy Dev (automatico) ===
  deploy-dev:
    needs: build-and-test
    runs-on: ubuntu-latest
    environment:
      name: development
      url: https://dev.myapp.com
    steps:
      - uses: actions/checkout@v4
      - uses: actions/download-artifact@v4
        with:
          name: build
          path: .next/

      - name: Deploy to Dev
        run: npx vercel deploy --prod
        env:
          VERCEL_TOKEN: ${{ secrets.VERCEL_TOKEN }}

  # === FASE 3: Deploy Staging (automatico + smoke tests) ===
  deploy-staging:
    needs: deploy-dev
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.myapp.com
    steps:
      - uses: actions/checkout@v4
      - uses: actions/download-artifact@v4
        with:
          name: build
          path: .next/

      - name: Deploy to Staging
        run: npx vercel deploy --prod
        env:
          VERCEL_TOKEN: ${{ secrets.VERCEL_TOKEN }}

      - name: Run E2E Tests
        run: npx playwright test --config=playwright.staging.config.ts
        env:
          BASE_URL: https://staging.myapp.com

      - name: Run Smoke Tests
        run: |
          curl -f https://staging.myapp.com/api/health || exit 1
          curl -f https://staging.myapp.com/api/readiness || exit 1

  # === FASE 4: Deploy Production (approval manual) ===
  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production   # Requer approval manual (configurado no GitHub)
      url: https://myapp.com
    steps:
      - uses: actions/checkout@v4
      - uses: actions/download-artifact@v4
        with:
          name: build
          path: .next/

      - name: Deploy to Production
        run: npx vercel deploy --prod
        env:
          VERCEL_TOKEN: ${{ secrets.VERCEL_TOKEN }}

      - name: Post-deploy smoke test
        run: |
          sleep 30
          curl -f https://myapp.com/api/health || exit 1

      - name: Notify team
        if: success()
        run: |
          echo "Production deploy successful: ${{ github.sha }}"
```

> Fonte: [GitHub Docs Deployment Environments](https://docs.github.com/en/actions/concepts/workflows-and-actions/deployment-environments), [OneUptime Multi-Env Guide](https://oneuptime.com/blog/post/2026-01-25-deploy-multiple-environments-github-actions/view)

### 6.2 Approval Gates

**Configuracao no GitHub:**

1. Settings > Environments > Production
2. Required reviewers: adicionar 1+ pessoas
3. Wait timer: opcional (e.g., 15 min entre staging e prod)
4. Deployment branches: restringir a `main` apenas

```yaml
# Cada environment pode ter secrets exclusivos
# Settings > Environments > Production > Environment secrets
# - VERCEL_TOKEN (prod)
# - DATABASE_URL (prod)
# - STRIPE_SECRET_KEY (prod live)
```

**Custom deployment protection rules (GitHub Apps):**

Alem de approval manual, voce pode integrar gates automaticos que verificam status externo antes de permitir o deploy — por exemplo, verificar se o Datadog nao tem alertas ativos ou se o PagerDuty nao tem incidentes abertos.

### 6.3 Database Migration Coordination

Migracao de banco e o aspecto mais critico da promocao entre ambientes. Schemas nao podem ser "rollbackados" com um flip de switch como um deploy de aplicacao.

**Expand-Contract Pattern (recomendado):**

```
Fase 1 (EXPAND): Adicionar nova coluna/tabela (backward-compatible)
  -> Deploy app v2 que usa nova + velha estrutura
  -> Migrar dados para nova estrutura

Fase 2 (CONTRACT): Remover coluna/tabela antiga
  -> Deploy app v3 que so usa nova estrutura
  -> Drop da estrutura antiga
```

**Exemplo pratico:**

```sql
-- Fase 1: EXPAND — renomear coluna "name" para "full_name"
-- Migration 001: adicionar nova coluna
ALTER TABLE users ADD COLUMN full_name TEXT;
UPDATE users SET full_name = name;

-- Nesse ponto, app v1 (usa "name") e app v2 (usa "full_name") funcionam
-- Deploy app v2 que le/escreve em ambas colunas

-- Fase 2: CONTRACT — remover coluna antiga (so apos 100% na v2)
-- Migration 002: remover coluna antiga
ALTER TABLE users DROP COLUMN name;
```

> *"Database migrations must retain backward compatibility at all phases by employing an expand and contract model."*
> -- [Liquibase Blue-Green Deployments](https://www.liquibase.com/blog/blue-green-deployments-liquibase)

**Workflow de migracao no CI/CD:**

```yaml
  # Executar migracao ANTES do deploy da aplicacao
  run-migrations:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Run database migrations
        run: npx prisma migrate deploy
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}

      - name: Verify migration
        run: npx prisma migrate status
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}

  deploy-production:
    needs: run-migrations  # so deploya app APOS migracao OK
    # ...
```

**Ferramentas de migracao:**

| Ferramenta | Linguagem | Destaques |
|------------|-----------|-----------|
| **Prisma Migrate** | TypeScript/JS | Schema-first, type-safe |
| **Drizzle Kit** | TypeScript/JS | Push/pull, lightweight |
| **Flyway** | Java/Multi | Maduro, enterprise |
| **Liquibase** | Java/Multi | Changelogs, rollback |
| **Alembic** | Python | SQLAlchemy integration |
| **golang-migrate** | Go | CLI simples |

### 6.4 Rollback Procedures

| Tipo de Rollback | Como | Quando |
|-----------------|------|--------|
| **Instant rollback (Vercel)** | Promover deploy anterior via dashboard | Bug na app, sem mudanca de DB |
| **Feature flag rollback** | Desligar flag | Feature nova causando problemas |
| **Git revert + redeploy** | `git revert` + merge em main | Mudanca problematica identificada |
| **DB rollback** | Migration reversa (se possivel) | Schema change quebrou algo |
| **Full rollback** | Reverter app + DB + config | Incidente critico |

**Vercel instant rollback:**

```bash
# Listar deploys anteriores
vercel ls --prod

# Promover deploy anterior para producao
vercel promote <deployment-url>

# Ou via dashboard: Deployments > ... > Promote to Production
```

**Checklist pre-rollback:**

1. O rollback de app e compativel com o schema atual do banco?
2. Existem migrations que precisam ser revertidas?
3. Feature flags precisam ser desligados antes do rollback?
4. Cache precisa ser invalidado?
5. Filas de mensagens tem trabalhos incompativeis?

> *"Include a rollback test in your deployment runbook by flipping to green, running smoke tests, flipping back to blue, running smoke tests again, and validating the path before you need it in an incident."*
> -- [Octopus Deploy Blue-Green Best Practices](https://octopus.com/devops/software-deployments/blue-green-deployment-best-practices/)

---

## 7. Convencoes Profissionais de Nomenclatura

### 7.1 Environment Variables — Padroes por Dominio

```bash
# === Padrao Geral ===
# Formato: {SERVICO}_{RECURSO}_{PROPRIEDADE}
# Case: SCREAMING_SNAKE_CASE
# Separador: underscore (_)

# === Database ===
DATABASE_URL                    # Connection string completa
DATABASE_HOST                   # Hostname
DATABASE_PORT                   # Porta
DATABASE_NAME                   # Nome do banco
DATABASE_USER                   # Usuario
DATABASE_POOL_SIZE              # Pool de conexoes
DATABASE_SSL_MODE               # "require" | "prefer" | "disable"
DATABASE_DIRECT_URL             # Conexao direta (bypassa pooler)

# === Cache ===
REDIS_URL                       # Connection string
REDIS_HOST                      # Hostname
REDIS_PORT                      # Porta
REDIS_TLS_ENABLED               # Boolean

# === Auth ===
AUTH_SECRET                     # Secret principal (NextAuth/Auth.js)
AUTH_JWT_SECRET                  # JWT signing secret
AUTH_JWT_EXPIRY                  # TTL em segundos
AUTH_REFRESH_TOKEN_EXPIRY        # TTL refresh token
AUTH_GOOGLE_CLIENT_ID            # OAuth provider
AUTH_GOOGLE_CLIENT_SECRET        # OAuth provider
AUTH_GITHUB_CLIENT_ID            # OAuth provider
AUTH_GITHUB_CLIENT_SECRET        # OAuth provider

# === APIs Externas ===
STRIPE_SECRET_KEY               # Secret key (server-only)
STRIPE_PUBLISHABLE_KEY          # Publishable key (pode ser publica)
STRIPE_WEBHOOK_SECRET            # Webhook signing secret
RESEND_API_KEY                   # API key
OPENAI_API_KEY                   # API key
ANTHROPIC_API_KEY                # API key
SENTRY_DSN                      # Data Source Name
SENTRY_AUTH_TOKEN                # Token de release

# === Infraestrutura ===
AWS_ACCESS_KEY_ID               # IAM key
AWS_SECRET_ACCESS_KEY            # IAM secret
AWS_REGION                      # us-east-1
S3_BUCKET_NAME                  # Nome do bucket
CDN_URL                         # URL do CDN

# === App ===
APP_URL                         # URL publica da aplicacao
APP_PORT                        # Porta do servidor
APP_ENV                         # development | staging | production
APP_LOG_LEVEL                   # debug | info | warn | error
APP_CORS_ORIGINS                # Origens permitidas (comma-separated)
```

### 7.2 API Key Patterns — Formatos por Provider

Muitos providers usam prefixos padrao que facilitam a identificacao e validacao de keys.

| Provider | Prefixo Secret | Prefixo Public | Formato |
|----------|---------------|----------------|---------|
| Stripe | `sk_live_` / `sk_test_` | `pk_live_` / `pk_test_` | Alfanumerico |
| OpenAI | `sk-` | N/A | Alfanumerico |
| Anthropic | `sk-ant-` | N/A | Alfanumerico com hifens |
| Supabase | JWT (`eyJ...`) | JWT (`eyJ...`) | JWT (anon vs service_role) |
| Resend | `re_` | N/A | Alfanumerico |
| SendGrid | `SG.` | N/A | Alfanumerico com pontos |
| GitHub PAT | `ghp_` | N/A | Alfanumerico |
| GitHub App | `ghs_` | N/A | Alfanumerico |
| Vercel | `vercel_` | N/A | Alfanumerico |

**Usar prefixos nativos para validacao automatica:**

```typescript
// Validar formato de API keys com Zod
const envSchema = z.object({
  STRIPE_SECRET_KEY: z.string().regex(
    /^sk_(live|test)_/,
    "Stripe key must start with sk_live_ or sk_test_"
  ),
  OPENAI_API_KEY: z.string().startsWith("sk-"),
  RESEND_API_KEY: z.string().startsWith("re_"),
});
```

### 7.3 Service Account Naming

**Formato: `{purpose}-{service}-{environment}`**

```
# Google Cloud IAM Service Accounts
vm-webapp-prod@myproject.iam.gserviceaccount.com
wlif-cicd-staging@myproject.iam.gserviceaccount.com
sa-backup-prod@myproject.iam.gserviceaccount.com

# AWS IAM Users / Roles
cicd-github-actions-prod       # CI/CD pipeline
app-webapp-staging              # Aplicacao em staging
svc-backup-scheduler-prod       # Servico de backup

# Database Roles
app_readonly_prod               # Leitura apenas
app_readwrite_staging           # Leitura e escrita em staging
migration_admin_prod            # Migracao (privilegiado, uso temporario)
```

> Fonte: [Google Cloud Service Account Best Practices](https://docs.google.com/iam/docs/best-practices-service-accounts), [Azure Naming Convention](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-naming)

**Regras:**

| Regra | Exemplo Bom | Exemplo Ruim |
|-------|------------|-------------|
| Incluir environment | `app-web-prod` | `app-web` |
| Incluir proposito | `cicd-deploy-staging` | `deploy` |
| Nao incluir dados sensiveis | `stripe-webhook-handler` | `stripe-key-4242` |
| Kebab-case ou snake_case | `app-web-prod` | `AppWebProd` |
| Ser descritivo | `backup-scheduler-daily` | `cron1` |

**Prefixos recomendados para service accounts:**

| Prefixo | Proposito | Exemplo |
|---------|-----------|---------|
| `vm-` | Conta vinculada a VM | `vm-webapp-prod` |
| `sa-` | Service account generico | `sa-monitoring-prod` |
| `cicd-` | Pipeline CI/CD | `cicd-github-actions-prod` |
| `app-` | Aplicacao | `app-api-server-staging` |
| `svc-` | Servico interno | `svc-email-worker-prod` |
| `wlif-` | Workload Identity Federation | `wlif-gke-prod` |
| `admin-` | Conta administrativa | `admin-db-maintenance` |

### 7.4 Branch-to-Environment Mapping

```
main         -> Production
staging      -> Staging
develop      -> Development
feat/*       -> Preview (ephemeral)
hotfix/*     -> Preview -> Production (fast-track)
release/*    -> Staging -> Production
```

**Configuracao no Vercel:**

```json
{
  "git": {
    "deploymentEnabled": {
      "main": true,
      "staging": true
    }
  }
}
```

**GitHub Actions com mapeamento:**

```yaml
# Determinar environment baseado no branch
env:
  DEPLOY_ENV: ${{ github.ref == 'refs/heads/main' && 'production' ||
                  github.ref == 'refs/heads/staging' && 'staging' ||
                  'preview' }}
```

### 7.5 Resource Naming — Cloud Providers

**Formato geral: `{org}-{project}-{resource}-{environment}-{region}`**

```
# AWS Resources
myapp-api-lambda-prod-use1          # Lambda function
myapp-uploads-s3-prod-use1          # S3 bucket
myapp-cache-elasticache-staging     # ElastiCache
myapp-db-rds-prod-use1              # RDS instance

# GCP Resources
myapp-api-cloudrun-prod-us-central1
myapp-db-cloudsql-staging

# Supabase Projects
myapp-prod                           # Production project
myapp-staging                        # Staging project
myapp-dev                            # Development project
```

> Fonte: [AWS Naming Standards](https://knowledge.businesscompassllc.com/aws-account-and-resource-naming-standards-and-best-practices/), [Azure Cloud Adoption Framework](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-naming)

---

## 8. Decision Frameworks

### 8.1 Checklist: Setup de Novo Projeto

```
[ ] 1. Criar .env.example com todas as variaveis documentadas
[ ] 2. Adicionar .env* ao .gitignore (exceto .env.example)
[ ] 3. Configurar validacao com t3-env ou envalid
[ ] 4. Definir ambientes: local + staging + production (minimo)
[ ] 5. Escolher secrets manager (Doppler free para startups)
[ ] 6. Configurar preview deployments (Vercel ou Railway)
[ ] 7. Configurar GitHub Environments com protection rules
[ ] 8. Implementar kill switches para dependencias externas
[ ] 9. Documentar runbook de rollback
[ ] 10. Configurar secret scanning (GitHub native ou gitleaks)
```

### 8.2 Matriz de Maturidade

| Nivel | Praticas | Indicador |
|-------|----------|-----------|
| **L1 — Ad-hoc** | .env commitado, secrets hardcoded, sem validacao | "funciona na minha maquina" |
| **L2 — Basico** | .env.example, gitignore, variaveis no Vercel/Heroku | Deploy manual, sem gates |
| **L3 — Estruturado** | Validacao com Zod, secrets manager, CI/CD multi-env | Approval gates, preview deploys |
| **L4 — Avancado** | Config-as-code, drift detection, feature flags, rotation | Rollback automatico, canary deploys |
| **L5 — Elite** | Dynamic secrets, GitOps, zero-trust, observability | Mudancas config auditaveis end-to-end |

### 8.3 Quick Reference — Custo Mensal por Perfil

| Perfil | Secrets | Feature Flags | Hosting | Total Estimado |
|--------|---------|---------------|---------|----------------|
| Solo dev | Doppler Free | Statsig Free | Vercel Free | $0/mes |
| Startup (5 devs) | Infisical self-hosted | GrowthBook self-hosted | Vercel Pro ($20) | ~$20/mes |
| Scale-up (20 devs) | Doppler Team ($240) | Statsig Pro ($150) | Vercel Team ($400) | ~$790/mes |
| Enterprise (100 devs) | Vault HCP ($2K+) | LaunchDarkly ($5K+) | AWS/GCP ($10K+) | ~$17K+/mes |

### 8.4 Anti-Patterns — O Que NUNCA Fazer

| Anti-Pattern | Risco | Solucao |
|-------------|-------|---------|
| Commitar `.env` no git | Leak de credenciais | `.gitignore` + secret manager |
| Usar `NEXT_PUBLIC_` para secrets | Exposicao no browser | Server-only variables |
| Copiar banco de prod para local | Violacao LGPD | Seed scripts + dados sinteticos |
| Deploy direto em producao | Zero safety net | CI/CD com staging obrigatorio |
| Secrets hardcoded "so para testar" | Ficam para sempre | Usar secret manager desde dia 1 |
| Feature flag sem data de expiracao | Codigo morto acumulado | TTL de 30 dias em release flags |
| Rotacao manual de credenciais | Esquecimento, breach | Automacao via Secrets Manager/Vault |
| Mesmo secret em todos ambientes | Blast radius maximo | Um secret por ambiente |
| Editar config em prod via dashboard | Drift, sem audit trail | Config-as-code + PR review |

---

## Fontes Consultadas

### Secrets Management
- [Top 5 Secrets Management Tools of 2026 — Deepak Gupta](https://guptadeepak.com/top-5-secrets-management-tools-hashicorp-vault-aws-doppler-infisical-and-azure-key-vault-compared/)
- [AWS Secrets Manager vs HashiCorp Vault 2026 — Infisical](https://infisical.com/blog/aws-secrets-manager-vs-hashicorp-vault)
- [Infisical vs Doppler vs HashiCorp Vault — PkgPulse](https://www.pkgpulse.com/blog/infisical-vs-doppler-vs-hashicorp-vault-secrets-management-2026)
- [Vault vs Doppler 2025 — Doppler Blog](https://www.doppler.com/blog/vault-vs-doppler-a-2025-secrets-management-face-off)
- [HashiCorp Vault Pricing 2026 — Infisical](https://infisical.com/blog/hashicorp-vault-pricing)
- [AWS Secrets Manager Pricing — CostGoat](https://costgoat.com/pricing/aws-secrets-manager)
- [SSM Parameter Store vs Secrets Manager — Ran the Builder](https://ranthebuilder.cloud/blog/secrets-manager-vs-parameter-store-which-one-should-you-really-use/)
- [How to Choose AWS Service for Secrets — AWS Blog](https://aws.amazon.com/blogs/security/how-to-choose-the-right-aws-service-for-managing-secrets-and-configurations/)
- [Secrets Management Pricing 2026 — CyberSecTool](https://www.cybersectool.com/blog/secrets-management-pricing-breakdown-2026)
- [HashiCorp Vault Auto Rotation — HashiCorp Developer](https://developer.hashicorp.com/hcp/docs/vault-secrets/auto-rotation)
- [Vault Database Secrets Engine — HashiCorp Developer](https://developer.hashicorp.com/vault/docs/secrets/databases)
- [Automating API Key Rotation 2025 — AI Infra Link](https://www.ai-infra-link.com/automating-api-key-rotation-best-practices-for-enhanced-security-in-2025/)
- [Secret Rotation Patterns — I Am Raghuveer](https://www.iamraghuveer.com/posts/secret-rotation/)

### Environment Variables
- [12-Factor App Config — 12factor.net](https://12factor.net/config)
- [T3 Env — Next.js — t3.gg](https://env.t3.gg/docs/nextjs)
- [Create T3 App Environment Variables](https://create.t3.gg/en/usage/env-variables)
- [Environment Variable Management Best Practices 2026 — Env-Sentinel](https://www.envsentinel.dev/blog/environment-variable-management-tips-best-practices)
- [Environment Variables Best Practices — Configu](https://configu.com/blog/environment-variables-how-to-use-them-and-4-critical-best-practices/)
- [Vercel Environment Variables — Vercel Docs](https://vercel.com/docs/environment-variables)
- [Vercel Environments — Vercel Docs](https://vercel.com/docs/deployments/environments)
- [Supabase Edge Functions Secrets — Supabase Docs](https://supabase.com/docs/guides/functions/secrets)
- [Supabase Config Management — Supabase Docs](https://supabase.com/docs/guides/local-development/managing-config)
- [Next.js Env Validation with t3-oss — Sikandar Dev](https://medium.com/@the.sikandar.dev/next-js-env-validation-with-t3-oss-env-nextjs-0733778b1b73)

### Feature Flags
- [Feature Flags Progressive Rollouts Next.js — CodeWithSeb](https://www.codewithseb.com/blog/feature-flags-progressive-rollouts-nextjs-guide)
- [Feature Flags in Next.js Rollouts — Statsig](https://www.statsig.com/perspectives/feature-flags-nextjs-rollouts)
- [Feature Flags Best Practices — Design Revision](https://designrevision.com/blog/feature-flags-best-practices)
- [LaunchDarkly Pricing — LaunchDarkly](https://launchdarkly.com/pricing/)
- [Unleash Pricing — Unleash](https://www.getunleash.io/pricing)
- [GrowthBook Pricing — GrowthBook](https://www.growthbook.io/pricing)
- [Statsig Pricing — Statsig](https://www.statsig.com/pricing)
- [Open Source Feature Flag Tools Compared 2026 — FlagShark](https://flagshark.com/blog/open-source-feature-flag-tools-compared-2026/)
- [Top Feature Flag Tools Compared — Startupik](https://startupik.com/top-feature-flag-tools-compared-launchdarkly-vs-statsig-vs-growthbook/)
- [Implement Feature Flags in Next.js with Unleash — Unleash Docs](https://docs.getunleash.io/guides/implement-feature-flags-in-nextjs)
- [Feature Flag System with Next.js and Supabase — FreeCodeCamp](https://www.freecodecamp.org/news/how-to-build-a-production-ready-feature-flag-system-with-nextjs-and-supabase/)

### CI/CD e Deployment
- [Deploy to Multiple Environments GitHub Actions — OneUptime](https://oneuptime.com/blog/post/2026-01-25-deploy-multiple-environments-github-actions/view)
- [Deployment Gates GitHub Actions — OneUptime](https://oneuptime.com/blog/post/2025-12-20-deployment-gates-github-actions/view)
- [GitHub Deployment Environments — GitHub Docs](https://docs.github.com/en/actions/concepts/workflows-and-actions/deployment-environments)
- [GitHub Deployment Environments and Approval Gates — Silvana's Blog](https://devops.silvanasblog.com/blog/github-action-deployment-gates/)
- [Blue-Green Deployment Best Practices 2025 — Octopus Deploy](https://octopus.com/devops/software-deployments/blue-green-deployment-best-practices/)
- [Blue-Green Deployments Liquibase](https://www.liquibase.com/blog/blue-green-deployments-liquibase)
- [Blue-Green Deployments AWS ECS — FreeCodeCamp](https://www.freecodecamp.org/news/how-to-manage-blue-green-deployments-on-aws-ecs-with-database-migrations/)

### Configuration & Drift Detection
- [Config Drift Detection Open Source Tools 2025 — AI Infra Link](https://www.ai-infra-link.com/mastering-config-drift-detection-top-open-source-tools-for-2025/)
- [Configuration Drift — Spacelift](https://spacelift.io/blog/what-is-configuration-drift)
- [Configuration Drift — Aqua Security](https://www.aquasec.com/cloud-native-academy/vulnerability-management/configuration-drift/)
- [Configuration Drift — Puppet](https://www.puppet.com/blog/configuration-drift)

### Naming Conventions
- [Azure Naming Convention — Microsoft Cloud Adoption Framework](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-naming)
- [AWS Naming Standards — Business Compass](https://knowledge.businesscompassllc.com/aws-account-and-resource-naming-standards-and-best-practices/)
- [GCP Service Account Best Practices — Google Cloud](https://docs.google.com/iam/docs/best-practices-service-accounts)

### Preview Environments
- [10 Best Preview Environment Platforms 2026 — Northflank](https://northflank.com/blog/preview-environment-platforms)
- [Railway Environments Explained — DEV Community](https://dev.to/bean_bean/railway-environments-explained-branch-deployments-staging-and-zero-config-databases-2556)
- [Vercel vs Railway vs Render 2026 — Athenic](https://getathenic.com/blog/vercel-vs-railway-vs-render-ai-deployment)
