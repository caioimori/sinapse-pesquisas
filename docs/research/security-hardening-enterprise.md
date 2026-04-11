# Security Hardening Enterprise

> Deep research sobre praticas avancadas de seguranca para sistemas em producao, indo alem do basico OWASP.
> Pesquisa conduzida com verificacao via WebSearch. Todas as afirmacoes possuem fontes citadas.

**Ultima atualizacao:** 2026-04-11
**Nivel de profundidade:** DEEP DIVE (Tier 3 — Research Depth Pyramid)
**Linhas:** ~2000+

---

## Indice

1. [OWASP Top 10 (2025) Deep Dive](#1-owasp-top-10-2025-deep-dive)
2. [Supply Chain Security](#2-supply-chain-security)
3. [SAST / DAST / SCA](#3-sast--dast--sca)
4. [Authentication & Authorization Architecture](#4-authentication--authorization-architecture)
5. [API Security](#5-api-security)
6. [Secrets Management Advanced](#6-secrets-management-advanced)
7. [Zero Trust Architecture](#7-zero-trust-architecture)
8. [Compliance Frameworks](#8-compliance-frameworks)
9. [Incident Response](#9-incident-response)
10. [Security Checklist por Tipo de Projeto](#10-security-checklist-por-tipo-de-projeto)

---

## 1. OWASP Top 10 (2025) Deep Dive

O OWASP Top 10:2025 recebeu atualizacoes significativas em relacao a versao 2021. Duas categorias novas foram adicionadas e a filosofia mudou de sintomas para causas-raiz.

Fonte: [OWASP Top 10:2025 Official](https://owasp.org/Top10/2025/)

### 1.1 Lista Completa OWASP Top 10:2025

| Posicao | Categoria | Mudanca vs 2021 |
|---------|-----------|-----------------|
| A01 | Broken Access Control | Manteve #1 (absorveu SSRF) |
| A02 | Security Misconfiguration | Subiu de #5 para #2 |
| A03 | Software Supply Chain Failures | **NOVA** (expandiu "Vulnerable Components") |
| A04 | Cryptographic Failures | Caiu de #2 para #4 |
| A05 | Injection | Caiu de #3 para #5 |
| A06 | Insecure Design | Caiu de #4 para #6 |
| A07 | Authentication Failures | Renomeada (antes "Identification and Auth") |
| A08 | Software or Data Integrity Failures | Manteve posicao similar |
| A09 | Security Logging and Alerting Failures | Renomeada |
| A10 | Mishandling of Exceptional Conditions | **NOVA** |

Fonte: [OWASP Top 10:2025 Introduction](https://owasp.org/Top10/2025/0x00_2025-Introduction/), [GitLab Blog](https://about.gitlab.com/blog/2025-owasp-top-10-whats-changed-and-why-it-matters/)

---

### 1.2 A01:2025 — Broken Access Control

**Status:** #1 mais critico. 100% das aplicacoes testadas tinham alguma forma de broken access control. 3.73% das aplicacoes tinham CWEs mapeados nesta categoria.

Fonte: [OWASP A01:2025](https://owasp.org/Top10/2025/A01_2025-Broken_Access_Control/)

**Novidade 2025:** SSRF (Server-Side Request Forgery) foi consolidado dentro desta categoria como uma falha de autorizacao.

**Exemplos reais de ataques:**
- **Instagram IDOR (2019):** Pesquisadores descobriram que era possivel acessar fotos privadas de qualquer usuario manipulando IDs de objeto na API
- **Ticketmaster (2024):** 560 milhoes de registros expostos por falta de MFA em contas Snowflake — falha de controle de acesso
- **Change Healthcare (2024):** 190 milhoes de registros — servidor sem MFA permitiu acesso inicial

Fonte: [IntelligenceX Blog](https://blog.intelligencex.org/broken-access-control-owasp-a01-2025-complete-guide)

**Padroes de prevencao:**

```javascript
// PROIBIDO: Confiar em parametros do client-side
app.get('/api/users/:id/profile', (req, res) => {
  const profile = await db.getUserProfile(req.params.id); // IDOR!
  res.json(profile);
});

// CORRETO: Validar ownership no server-side
app.get('/api/users/:id/profile', authenticate, (req, res) => {
  // Verificar se o usuario autenticado TEM acesso ao recurso solicitado
  if (req.params.id !== req.user.id && !req.user.roles.includes('admin')) {
    return res.status(403).json({ error: 'Forbidden' });
  }
  const profile = await db.getUserProfile(req.params.id);
  res.json(profile);
});
```

```javascript
// PROIBIDO: Controle de acesso no frontend
if (user.role === 'admin') {
  showAdminPanel(); // Atacante pode burlar via DevTools
}

// CORRETO: Controle de acesso no backend + frontend
// Backend middleware
function requireRole(role) {
  return (req, res, next) => {
    if (!req.user || !req.user.roles.includes(role)) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }
    next();
  };
}

app.get('/api/admin/users', authenticate, requireRole('admin'), getUsers);
```

```sql
-- Supabase RLS: Usuarios so veem seus proprios dados
ALTER TABLE user_profiles ENABLE ROW LEVEL SECURITY;

CREATE POLICY "users_read_own_data"
ON user_profiles FOR SELECT
USING (auth.uid() = user_id);

CREATE POLICY "users_update_own_data"
ON user_profiles FOR UPDATE
USING (auth.uid() = user_id)
WITH CHECK (auth.uid() = user_id);
```

**Checklist de prevencao A01:**
- [ ] Deny by default — negar acesso a tudo, liberar explicitamente
- [ ] Validar autorizacao no server-side para TODA requisicao
- [ ] Implementar RBAC ou ABAC com testes de unidade
- [ ] Desabilitar directory listing no servidor web
- [ ] Rate-limit em APIs para dificultar enumeracao
- [ ] Logar todas as falhas de autorizacao
- [ ] Testes automatizados de access control (unit + integration)

---

### 1.3 A04:2025 — Cryptographic Failures

**Exemplos comuns:**
- Dados sensiveis trafegando sem TLS (HTTP em vez de HTTPS)
- Uso de algoritmos obsoletos (MD5, SHA-1, DES)
- Chaves de criptografia hardcoded no codigo
- Dados sensiveis em repouso sem criptografia

**Encryption at rest:**

```javascript
// Node.js — Criptografia AES-256-GCM para dados em repouso
import { createCipheriv, createDecipheriv, randomBytes } from 'crypto';

function encrypt(plaintext, key) {
  const iv = randomBytes(16);
  const cipher = createCipheriv('aes-256-gcm', key, iv);
  const encrypted = Buffer.concat([
    cipher.update(plaintext, 'utf8'),
    cipher.final()
  ]);
  const authTag = cipher.getAuthTag();
  return { iv, encrypted, authTag };
}

function decrypt(encrypted, key, iv, authTag) {
  const decipher = createDecipheriv('aes-256-gcm', key, iv);
  decipher.setAuthTag(authTag);
  return Buffer.concat([
    decipher.update(encrypted),
    decipher.final()
  ]).toString('utf8');
}
```

```sql
-- PostgreSQL: Criptografia com pgcrypto
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Criptografar dados sensiveis
INSERT INTO sensitive_data (user_id, ssn_encrypted)
VALUES (
  $1,
  pgp_sym_encrypt($2, current_setting('app.encryption_key'))
);

-- Descriptografar
SELECT pgp_sym_decrypt(ssn_encrypted, current_setting('app.encryption_key'))
FROM sensitive_data WHERE user_id = $1;
```

**Encryption in transit:**

```javascript
// Forcar HTTPS com HSTS
app.use((req, res, next) => {
  res.setHeader(
    'Strict-Transport-Security',
    'max-age=63072000; includeSubDomains; preload'
  );
  next();
});

// Redirect HTTP para HTTPS
app.use((req, res, next) => {
  if (!req.secure && req.headers['x-forwarded-proto'] !== 'https') {
    return res.redirect(301, `https://${req.hostname}${req.url}`);
  }
  next();
});
```

**Key management best practices:**
- NUNCA armazenar chaves no codigo ou em .env em producao
- Usar servicos gerenciados: AWS KMS, GCP KMS, HashiCorp Vault
- Rotacionar chaves a cada 90 dias (minimo)
- Separar chaves por ambiente (dev, staging, prod)
- Manter audit log de uso de chaves

---

### 1.4 A05:2025 — Injection

**Tipos principais:** SQL Injection, NoSQL Injection, LDAP Injection, OS Command Injection, XSS.

```javascript
// SQL INJECTION — PROIBIDO
const query = `SELECT * FROM users WHERE email = '${userInput}'`;
// Atacante: ' OR '1'='1' -- => retorna TODOS os usuarios

// SQL INJECTION — CORRETO: Parameterized queries
const query = 'SELECT * FROM users WHERE email = $1';
const result = await db.query(query, [userInput]);

// Supabase: Ja parametrizado por padrao
const { data } = await supabase
  .from('users')
  .select('*')
  .eq('email', userInput);
```

```javascript
// NoSQL INJECTION — PROIBIDO (MongoDB)
db.users.find({ username: req.body.username });
// Atacante envia: { "username": { "$gt": "" } }

// NoSQL INJECTION — CORRETO
const username = String(req.body.username); // Forcar tipo string
db.users.find({ username });
```

```javascript
// OS COMMAND INJECTION — PROIBIDO
const { exec } = require('child_process');
exec(`ping ${userInput}`, callback);
// Atacante: "8.8.8.8; rm -rf /"

// OS COMMAND INJECTION — CORRETO
const { execFile } = require('child_process');
execFile('ping', ['-c', '4', userInput], callback);
// execFile nao usa shell, prevenindo injection
```

---

### 1.5 A02:2025 — Security Misconfiguration

**Subiu de #5 para #2.** Afeta 3% das aplicacoes testadas. Inclui:

- Configuracoes default nao alteradas
- Features desnecessarias habilitadas (portas, servicos, paginas)
- Stack traces expostos em producao
- Headers de seguranca ausentes
- Permissoes de cloud services excessivamente abertas

```javascript
// PROIBIDO: Stack traces em producao
app.use((err, req, res, next) => {
  res.status(500).json({ error: err.stack }); // Expoe internos!
});

// CORRETO: Error handling seguro
app.use((err, req, res, next) => {
  // Logar detalhes internamente
  logger.error('Internal error', {
    error: err.message,
    stack: err.stack,
    requestId: req.id
  });
  // Retornar mensagem generica ao usuario
  res.status(500).json({
    error: 'Internal server error',
    requestId: req.id  // para suporte identificar o erro
  });
});
```

```javascript
// Next.js: Desabilitar headers reveladores
// next.config.js
module.exports = {
  poweredByHeader: false,  // Remove "X-Powered-By: Next.js"
  headers: async () => [{
    source: '/:path*',
    headers: [
      { key: 'X-Frame-Options', value: 'DENY' },
      { key: 'X-Content-Type-Options', value: 'nosniff' },
      { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
    ],
  }],
};
```

---

### 1.6 A07:2025 — Authentication Failures

Includes: credential stuffing, session fixation, weak credentials, missing MFA.

**Licao dos maiores breaches:** A ausencia de MFA foi a causa raiz dos maiores vazamentos de 2023-2025. Change Healthcare (190M registros) e Ticketmaster/Snowflake (560M registros) foram comprometidos por falta de MFA.

Fonte: [HIPAA Journal](https://www.hipaajournal.com/biggest-healthcare-data-breaches-2024/), [Axios](https://www.axios.com/2025/01/28/ticketmaster-advance-auto-parts-data-breaches-victims)

```javascript
// Protecao contra credential stuffing
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutos
  max: 5,                      // 5 tentativas
  message: { error: 'Too many login attempts. Try again later.' },
  standardHeaders: true,
  legacyHeaders: false,
  keyGenerator: (req) => req.body.email || req.ip,  // Por email OU IP
});

app.post('/api/auth/login', authLimiter, loginHandler);
```

---

### 1.7 A08:2025 — Software and Data Integrity Failures

Abrange supply chain attacks e seguranca de CI/CD pipelines.

**Exemplos reais:**
- **SolarWinds (2020):** Atacantes injetaram backdoor no build system, afetando 18.000+ organizacoes
- **Codecov (2021):** Script de upload do bash comprometido, roubando credenciais de CI/CD
- **event-stream (2018):** Novo maintainer adicionou malware ao pacote npm com 2M downloads/semana

**Prevencao:**

```yaml
# GitHub Actions: Pinning de actions por hash (nao tag)
# PROIBIDO: Usar tags mutaveis
# - uses: actions/checkout@v4

# CORRETO: Usar hash do commit
- uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11 # v4.1.1
```

```html
<!-- Verificar integridade de scripts externos -->
<!-- PROIBIDO -->
<script src="https://cdn.example.com/analytics.js"></script>

<!-- CORRETO: Subresource Integrity (SRI) -->
<script
  src="https://cdn.example.com/analytics.js"
  integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxy9rx7HNQlGYl1kPzQho1wx4JwY8w"
  crossorigin="anonymous"
></script>
```

---

## 2. Supply Chain Security

### 2.1 Ataques Notaveis e Licoes

| Incidente | Ano | Impacto | Causa Raiz | Licao |
|-----------|-----|---------|------------|-------|
| event-stream | 2018 | 2M downloads/semana | Social engineering de maintainer | Auditar novos maintainers |
| ua-parser-js | 2021 | 8M downloads/semana | Conta comprometida | Exigir MFA em todos os maintainers |
| colors.js/faker.js | 2022 | 20M+ downloads/semana | Sabotagem pelo maintainer | Dependencia de single-maintainer e risco |
| XZ Utils | 2024 | Backdoor em compressao | Social engineering multi-ano | Revisao de codigo mesmo de contribuidores confiados |
| chalk/debug (npm) | 2025 | 18 pacotes comprometidos | Phishing de maintainer | Phishing-resistant MFA obrigatorio |

Fonte: [Rescana Analysis](https://www.rescana.com/post/in-depth-analysis-supply-chain-poisoning-of-popular-npm-packages-exploiting-event-stream-ua-parser), [CISA Alert Sep 2025](https://www.cisa.gov/news-events/alerts/2025/09/23/widespread-supply-chain-compromise-impacting-npm-ecosystem), [Palo Alto Networks](https://www.paloaltonetworks.com/blog/cloud-security/npm-supply-chain-attack/)

**Em setembro de 2025,** o ecossistema JavaScript sofreu o maior comprometimento npm da historia, quando uma conta de maintainer foi hijacked via phishing, injetando codigo malicioso em 18 pacotes populares incluindo chalk e debug.

### 2.2 Ferramentas de Dependency Scanning

| Ferramenta | Tipo | Foco | Preco | Diferencial |
|------------|------|------|-------|-------------|
| npm audit | Built-in | CVEs conhecidos | Gratis | Integrado ao npm CLI |
| Snyk | SCA + SAST | CVEs + licencas + malware | Freemium ($25/dev/mes) | Fix automatico via PR |
| Socket.dev | Behavioral | Malware + comportamento suspeito | Freemium | 70+ sinais de risco de supply chain |
| Dependabot | Updates | CVEs + atualizacoes | Gratis (GitHub) | Nativo no GitHub |
| Renovate | Updates | CVEs + atualizacoes | Gratis | 30+ package managers, multi-plataforma |

Fonte: [Socket.dev](https://socket.dev/), [Snyk State of Secrets](https://snyk.io/articles/state-of-secrets/)

**Socket.dev** diferencia-se por fazer deep package inspection para detectar comportamento malicioso ANTES da instalacao, monitorando sinais como: codigo ofuscado, atividade de rede suspeita, acesso ao filesystem, execucao de shell commands, e install scripts. O registro npm agora inclui links de analise Socket diretamente nas paginas de pacotes.

### 2.3 Dependabot vs Renovate

| Aspecto | Dependabot | Renovate |
|---------|------------|---------|
| Plataformas | GitHub only | GitHub, GitLab, Bitbucket, Azure DevOps |
| Package managers | 14 | 30+ |
| Agrupamento de PRs | Manual (definir grupos) | Automatico por padrao |
| Automerge | Via GitHub Actions (workaround) | Built-in (`"automerge": true`) |
| Regex managers | Nao | Sim (Dockerfiles, Makefiles, CI) |
| Preco | Gratis | Gratis |
| Melhor para | GitHub-only, setup minimo | Multi-plataforma, controle avancado |

Fonte: [Renovate Docs Bot Comparison](https://docs.renovatebot.com/bot-comparison/), [TurboStarter Blog](https://www.turbostarter.dev/blog/renovate-vs-dependabot-whats-the-best-tool-to-automate-your-dependency-updates)

### 2.4 Lockfile Integrity e Dependency Pinning

```json
// package.json — PROIBIDO: Ranges amplos
{
  "dependencies": {
    "express": "^4.0.0",
    "lodash": "*"
  }
}
```

```json
// package.json — CORRETO: Versoes exatas ou ranges minimos
{
  "dependencies": {
    "express": "4.21.2",
    "lodash": "~4.17.21"
  }
}
```

```bash
# Verificar integridade do lockfile no CI
npm ci  # Usa EXATAMENTE o que esta no package-lock.json (falha se divergir)
# NUNCA usar "npm install" no CI — ele pode atualizar o lockfile
```

### 2.5 Software Bill of Materials (SBOM)

O SBOM e um inventario completo de todos os componentes, bibliotecas e dependencias de um software. Essencial para rastreabilidade e resposta rapida a vulnerabilidades.

Fonte: [GitHub SBOM Guide](https://github.com/resources/articles/what-is-an-sbom-software-bill-of-materials), [npm sbom docs](https://docs.npmjs.com/cli/v9/commands/npm-sbom/)

```bash
# Gerar SBOM em formato SPDX
npm sbom --sbom-format spdx

# Gerar SBOM em formato CycloneDX
npm sbom --sbom-format cyclonedx

# GitHub: Exportar automaticamente via Dependency Graph
# Settings > Code security and analysis > Dependency graph
```

**Caso real:** Quando o ataque Shai-Hulud comprometeu 500+ pacotes npm em setembro 2025, organizacoes com SBOMs atualizados identificaram rapidamente se tinham versoes afetadas. Organizacoes sem SBOM precisaram de triagem manual demorada.

### 2.6 npm Provenance

npm provenance permite verificar a origem de um pacote — que ele foi construido a partir de um repositorio especifico, em um CI especifico, com um commit especifico.

```bash
# Publicar com provenance (requer CI como GitHub Actions)
npm publish --provenance

# Verificar provenance de um pacote
npm audit signatures
```

---

## 3. SAST / DAST / SCA

### 3.1 Comparacao de Ferramentas SAST

| Aspecto | Semgrep | CodeQL | Snyk Code | SonarQube |
|---------|---------|--------|-----------|-----------|
| Abordagem | Pattern matching semantico | Banco de dados de codigo | AI/ML | Regras estaticas |
| Precisao | 82% (12% false positive) | 88% (5% false positive) | 85% (8% false positive) | Variavel |
| Velocidade | Muito rapida | Lenta (analise profunda) | Rapida | Media |
| Customizacao | YAML simples (aprender em 1 dia) | QL (curva alta) | Limitada | Media |
| Regras | 3000+ | 2500+ | AI-trained | 6500+ (85% qualidade, 15% seguranca) |
| Preco | OSS gratis, Pro $40/dev/mes | Gratis no GitHub | Gratis ate 100 scans/mes | Community gratis, $150+/dev/ano |
| Melhor para | Customizacao rapida | Profundidade maxima | Facilidade + SCA | Qualidade + seguranca juntos |

Fonte: [sanj.dev comparison](https://sanj.dev/post/ai-code-security-tools-comparison), [DryRun Security Showdown](https://www.dryrun.security/blog/dryrun-security-vs-semgrep-sonarqube-codeql-and-snyk---c-security-analysis-showdown), [StackHawk SAST tools](https://www.stackhawk.com/blog/best-sast-tools-comparison/)

### 3.2 Comparacao de Ferramentas DAST

| Aspecto | OWASP ZAP | Burp Suite Pro | Nuclei |
|---------|-----------|----------------|--------|
| Preco | Gratis (Apache 2.0) | ~$475/usuario/ano | Gratis (OSS) |
| Deteccao | OWASP Top 10, issues comuns | Complexas, multi-step | Template-based, baixo false positive |
| CI/CD | GitHub Actions, Docker, GitLab CI nativos | Docker, requer licenca enterprise | CLI simples, facil de integrar |
| Manual testing | Basico | Excelente (Intruder, Repeater) | Nao e foco |
| Melhor para | CI/CD automatizado, custo zero | Pentesting manual expert | Scans customizados, rapidos |

Fonte: [AppSec Santa ZAP vs Burp](https://appsecsanta.com/burp-suite-vs-zap), [Rafter DAST Comparison](https://rafter.so/blog/dast-tools-comparison)

**Recomendacao:** Usar ZAP no CI/CD para scans automatizados + Burp Suite para pentesting manual. Nuclei para verificacoes customizadas.

### 3.3 Software Composition Analysis (SCA)

| Aspecto | Snyk Open Source | Socket.dev | npm audit |
|---------|-----------------|------------|-----------|
| CVEs conhecidos | Sim (database proprio) | Sim | Sim (GitHub Advisory DB) |
| Malware detection | Limitado | Sim (70+ sinais) | Nao |
| License compliance | Sim | Parcial | Nao |
| Fix automatico | Sim (PRs com fix) | Nao | `npm audit fix` |
| Profundidade | Transitive deps | Behavioral analysis | Transitive deps |
| Preco | Gratis ate 200 testes/mes | Freemium | Gratis |

### 3.4 Integracao CI/CD — GitHub Actions

```yaml
# .github/workflows/security.yml
name: Security Scanning

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]
  schedule:
    - cron: '0 6 * * 1'  # Toda segunda as 6h

jobs:
  # SAST com Semgrep
  semgrep:
    runs-on: ubuntu-latest
    container:
      image: semgrep/semgrep
    steps:
      - uses: actions/checkout@v4
      - run: semgrep scan --config auto --sarif --output semgrep.sarif
      - uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: semgrep.sarif

  # SCA com npm audit
  dependency-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm audit --audit-level=high

  # Secret scanning com gitleaks
  secrets:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  # DAST com OWASP ZAP (em staging)
  dast:
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: zaproxy/action-full-scan@v0.12.0
        with:
          target: 'https://staging.myapp.com'
          rules_file_name: 'zap-rules.tsv'
```

Fonte: [Semgrep CI configs](https://semgrep.dev/docs/semgrep-ci/sample-ci-configs), [Gitleaks GitHub](https://github.com/gitleaks/gitleaks)

### 3.5 Quando Usar Cada Ferramenta

| Fase | Ferramenta | Objetivo |
|------|-----------|----------|
| Pre-commit | gitleaks, TruffleHog | Detectar secrets antes do commit |
| Pull Request | Semgrep, Snyk Code | SAST em diff do PR |
| Pull Request | Socket.dev, npm audit | SCA em dependencias alteradas |
| Merge to main | CodeQL, SonarQube | Analise profunda completa |
| Staging | OWASP ZAP, Nuclei | DAST contra aplicacao rodando |
| Pre-release | Burp Suite (manual) | Pentesting antes de major releases |
| Continuous | Snyk Monitor, Dependabot | Monitorar novas CVEs em deps existentes |

---

## 4. Authentication & Authorization Architecture

### 4.1 OAuth 2.0 / OIDC — Fluxos Detalhados

**OAuth 2.1** esta sendo finalizado como a evolucao do OAuth 2.0, tornando PKCE obrigatorio para TODOS os clientes.

Fonte: [OAuth 2.1 spec](https://oauth.net/2.1/), [Auth0 PKCE docs](https://auth0.com/docs/get-started/authentication-and-authorization-flow/authorization-code-flow-with-pkce)

| Fluxo | Quando Usar | PKCE? | Seguranca |
|-------|-------------|-------|-----------|
| Authorization Code + PKCE | SPAs, Mobile, SSR | Obrigatorio | Alta |
| Client Credentials | Machine-to-machine (M2M) | N/A | Alta (com secret seguro) |
| Device Authorization | Smart TVs, IoT | N/A | Media |
| Implicit (DEPRECATED) | NUNCA usar | N/A | Baixa — descontinuado |
| ROPC (DEPRECATED) | NUNCA usar | N/A | Baixa — descontinuado |

**PKCE Flow detalhado:**

```
1. Client gera code_verifier (random string 43-128 chars)
2. Client calcula code_challenge = SHA256(code_verifier)
3. Client redireciona para /authorize com code_challenge
4. User autentica no Authorization Server
5. Auth Server retorna authorization_code via redirect
6. Client troca code + code_verifier por tokens em /token
7. Auth Server valida: SHA256(code_verifier) === code_challenge armazenado
8. Auth Server emite access_token + refresh_token
```

```javascript
// Implementacao PKCE em JavaScript
import { randomBytes, createHash } from 'crypto';

function generateCodeVerifier() {
  return randomBytes(32)
    .toString('base64url')
    .slice(0, 128);
}

function generateCodeChallenge(verifier) {
  return createHash('sha256')
    .update(verifier)
    .digest('base64url');
}

// Uso
const codeVerifier = generateCodeVerifier();
const codeChallenge = generateCodeChallenge(codeVerifier);

// Enviar na requisicao de autorizacao
const authUrl = new URL('https://auth.example.com/authorize');
authUrl.searchParams.set('response_type', 'code');
authUrl.searchParams.set('client_id', CLIENT_ID);
authUrl.searchParams.set('redirect_uri', REDIRECT_URI);
authUrl.searchParams.set('code_challenge', codeChallenge);
authUrl.searchParams.set('code_challenge_method', 'S256');
authUrl.searchParams.set('scope', 'openid profile email');
```

### 4.2 JWT Best Practices

Fonte: [Auth0 Token Best Practices](https://auth0.com/docs/secure/tokens/token-best-practices), [SkyCloak JWT Lifecycle](https://skycloak.io/blog/jwt-token-lifecycle-management-expiration-refresh-revocation-strategies/)

| Pratica | Recomendacao |
|---------|-------------|
| Algoritmo | RS256 (assimetrico) — servidor assina com private key, clientes validam com public key |
| Expiracao access token | 15-30 minutos |
| Expiracao refresh token | 7-14 dias |
| Armazenamento (browser) | httpOnly cookie com secure, sameSite strict |
| Armazenamento (NUNCA) | localStorage (vulneravel a XSS) |
| Payload | NUNCA incluir dados sensiveis (senhas, PII) |
| Claim jti | Usar para detectar reuso de refresh tokens |
| Key rotation | Rotacionar signing keys a cada 90 dias |

**Token Rotation Pattern:**

```javascript
// Servidor: Implementacao de refresh token rotation
async function refreshTokens(oldRefreshToken) {
  // 1. Verificar se o refresh token existe e esta valido
  const tokenRecord = await db.findRefreshToken(oldRefreshToken);

  if (!tokenRecord) {
    // Token nao encontrado — possivel reuso!
    // Revogar TODA a familia de tokens do usuario
    await db.revokeAllTokensForUser(tokenRecord.userId);
    throw new Error('Refresh token reuse detected');
  }

  if (tokenRecord.used) {
    // Token ja foi usado — REPLAY ATTACK!
    await db.revokeAllTokensForUser(tokenRecord.userId);
    throw new Error('Refresh token reuse detected');
  }

  // 2. Marcar o token antigo como usado
  await db.markTokenAsUsed(oldRefreshToken);

  // 3. Gerar novos tokens
  const newAccessToken = generateAccessToken(tokenRecord.userId);
  const newRefreshToken = generateRefreshToken();

  // 4. Armazenar novo refresh token (mesma familia)
  await db.storeRefreshToken({
    token: newRefreshToken,
    userId: tokenRecord.userId,
    family: tokenRecord.family,
    used: false,
    expiresAt: new Date(Date.now() + 14 * 24 * 60 * 60 * 1000)
  });

  return { accessToken: newAccessToken, refreshToken: newRefreshToken };
}
```

### 4.3 Session Management

| Aspecto | Stateless (JWT) | Stateful (Server Sessions) |
|---------|-----------------|---------------------------|
| Storage | Token no client | Session ID no client, dados no servidor |
| Escala | Excelente (sem estado no server) | Requer session store distribuido (Redis) |
| Revogacao | Dificil (blacklist ou short TTL) | Facil (deletar do store) |
| Tamanho | Cresce com claims | Cookie pequeno (session ID) |
| Melhor para | APIs, microservices | Apps tradicionais, requisitos de revogacao imediata |

```javascript
// Cookie de sessao seguro
app.use(session({
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: {
    secure: true,         // Apenas HTTPS
    httpOnly: true,       // Inacessivel via JavaScript
    sameSite: 'strict',   // Protecao CSRF
    maxAge: 30 * 60 * 1000, // 30 minutos
    path: '/',
    domain: '.myapp.com'
  },
  store: new RedisStore({ client: redisClient }),
  name: '__session',      // Nome nao-padrao (nao "connect.sid")
}));
```

### 4.4 Passkeys / WebAuthn

**Estado em 2025:** Passkeys se tornaram mainstream. 69% dos usuarios tem pelo menos uma passkey. 48% dos top 100 sites suportam passkeys. Microsoft reportou 1 milhao de registros diarios — crescimento de 350% desde 2024.

Fonte: [1Password State of Passkeys 2025](https://www.1password.community/blog/random-but-memorable/the-state-of-passkeys-in-2025/163464), [AuthSignal Passwordless 2025](https://www.authsignal.com/blog/articles/passwordless-authentication-in-2025-the-year-passkeys-went-mainstream)

**Numeros-chave:**
- Taxa de sucesso de login com passkeys: 93% (vs 63% com metodo tradicional)
- Mercado de autenticacao passwordless: $24.1 bilhoes em 2025, projecao $55.7 bilhoes em 2030
- Tempo de implementacao: de 6 meses para 2-3 sprints com SDKs modernos
- WebAuthn Level 3 spec: Working Draft publicado em janeiro 2025

```javascript
// Registro de passkey (WebAuthn) — Server-side
import {
  generateRegistrationOptions,
  verifyRegistrationResponse
} from '@simplewebauthn/server';

const rpName = 'My App';
const rpID = 'myapp.com';
const origin = 'https://myapp.com';

// 1. Gerar opcoes de registro
const options = await generateRegistrationOptions({
  rpName,
  rpID,
  userID: user.id,
  userName: user.email,
  attestationType: 'none',
  authenticatorSelection: {
    residentKey: 'preferred',
    userVerification: 'preferred',
  },
});

// 2. Verificar resposta do client
const verification = await verifyRegistrationResponse({
  response: attestationResponse,
  expectedChallenge: options.challenge,
  expectedOrigin: origin,
  expectedRPID: rpID,
});

if (verification.verified) {
  // Armazenar credencial no banco
  await db.storeCredential({
    credentialID: verification.registrationInfo.credentialID,
    publicKey: verification.registrationInfo.credentialPublicKey,
    counter: verification.registrationInfo.counter,
    userId: user.id,
  });
}
```

### 4.5 MFA Implementation Patterns

| Metodo | Seguranca | UX | Phishing-resistant? |
|--------|-----------|-----|---------------------|
| SMS OTP | Baixa (SIM swap) | Facil | Nao |
| TOTP (Google Auth) | Media | Media | Nao |
| Push notification | Media-Alta | Boa | Parcial (MFA fatigue) |
| Hardware key (YubiKey) | Alta | Media | Sim |
| Passkeys/WebAuthn | Alta | Excelente | Sim |

**Recomendacao 2025:** Passkeys como metodo primario, TOTP como fallback. SMS somente como ultimo recurso.

### 4.6 RBAC vs ABAC vs ReBAC

Fonte: [Oso RBAC vs ABAC vs ReBAC](https://www.osohq.com/learn/rbac-vs-abac-vs-rebac-what-is-the-best-access-policy-paradigm), [Permit.io comparison](https://www.permit.io/blog/rbac-vs-abac-and-rebac-choosing-the-right-authorization-model)

| Modelo | Descricao | Melhor Para | Complexidade |
|--------|-----------|-------------|--------------|
| RBAC | Permissoes atreladas a roles (Admin, Editor, Viewer) | Organizacoes simples, poucos papeis | Baixa |
| ABAC | Decisao baseada em atributos (user dept + resource sensitivity + time + location) | Organizacoes complexas, regras dinamicas | Alta |
| ReBAC | Acesso baseado em relacoes entre entidades (user owns doc, org contains project) | Hierarquias, dados interconectados | Media-Alta |
| Hibrido | Combina RBAC + ABAC + ReBAC | Producao real (evolucao gradual) | Variavel |

**Tendencia 2025:** Abordagem hibrida e o padrao. Comece com RBAC, adicione ABAC quando precisar de regras dinamicas, use ReBAC para hierarquias de dados. Cedar (Amazon) foi adotado como linguagem de autorizacao para AI agents no Bedrock AgentCore.

```javascript
// Exemplo RBAC simples
const permissions = {
  admin: ['create', 'read', 'update', 'delete', 'manage_users'],
  editor: ['create', 'read', 'update'],
  viewer: ['read'],
};

function checkPermission(userRole, action) {
  return permissions[userRole]?.includes(action) ?? false;
}

// Exemplo ABAC
function checkAccess({ user, resource, action, environment }) {
  // Regra: Editores podem editar apenas recursos do seu departamento,
  // durante horario comercial
  if (action === 'edit') {
    return (
      user.role === 'editor' &&
      user.department === resource.department &&
      environment.hour >= 9 &&
      environment.hour <= 18
    );
  }
  return false;
}
```

### 4.7 Supabase Auth Deep Dive

Fonte: [Supabase Auth Architecture](https://supabase.com/docs/guides/auth/architecture), [Supabase Custom Claims](https://supabase.com/docs/guides/database/postgres/custom-claims-and-role-based-access-control-rbac), [Supabase Auth Hooks](https://supabase.com/docs/guides/auth/auth-hooks)

**Arquitetura:** Supabase Auth e um fork do GoTrue (Netlify), um servidor de API JWT escrito em Go. Emite JWTs que sao usados diretamente nas RLS policies do PostgreSQL.

**Custom Claims via Auth Hooks:**

```sql
-- Funcao de Auth Hook para adicionar custom claims ao JWT
CREATE OR REPLACE FUNCTION public.custom_access_token_hook(event jsonb)
RETURNS jsonb
LANGUAGE plpgsql
STABLE
AS $$
DECLARE
  claims jsonb;
  user_role text;
BEGIN
  -- Buscar o role do usuario
  SELECT role INTO user_role
  FROM public.user_roles
  WHERE user_id = (event->>'user_id')::uuid;

  -- Adicionar claim ao JWT
  claims := event->'claims';
  IF user_role IS NOT NULL THEN
    claims := jsonb_set(claims, '{user_role}', to_jsonb(user_role));
  ELSE
    claims := jsonb_set(claims, '{user_role}', '"viewer"');
  END IF;

  -- Retornar o evento modificado
  event := jsonb_set(event, '{claims}', claims);
  RETURN event;
END;
$$;

-- Dar permissao ao supabase_auth_admin
GRANT EXECUTE ON FUNCTION public.custom_access_token_hook TO supabase_auth_admin;
GRANT ALL ON TABLE public.user_roles TO supabase_auth_admin;
REVOKE EXECUTE ON FUNCTION public.custom_access_token_hook FROM authenticated, anon, public;
```

```sql
-- Usar o custom claim na RLS policy
CREATE POLICY "admins_full_access"
ON public.sensitive_data
FOR ALL
USING (
  (auth.jwt()->>'user_role') = 'admin'
);

CREATE POLICY "viewers_read_only"
ON public.sensitive_data
FOR SELECT
USING (
  auth.uid() IS NOT NULL
);
```

**Chaves Supabase — Regras de Uso:**

| Chave | Onde Usar | RLS | Seguranca |
|-------|----------|-----|-----------|
| `anon` key | Frontend/Client | Respeita RLS | Seguro com RLS ativo |
| `service_role` key | Server ONLY (Edge Functions, backend) | Bypassa RLS | NUNCA expor no frontend |
| `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` | Frontend (novo formato 2025) | Respeita RLS | Seguro com RLS ativo |

---

## 5. API Security

### 5.1 Rate Limiting Patterns

Fonte: [API7.ai Rate Limiting Guide](https://api7.ai/blog/rate-limiting-guide-algorithms-best-practices), [Arcjet Rate Limiting](https://blog.arcjet.com/rate-limiting-algorithms-token-bucket-vs-sliding-window-vs-fixed-window/), [Zuplo Best Practices 2025](https://zuplo.com/learning-center/10-best-practices-for-api-rate-limiting-in-2025)

| Algoritmo | Como Funciona | Burst | Memoria | Melhor Para |
|-----------|---------------|-------|---------|-------------|
| Fixed Window | Contador por janela de tempo fixa | Pico na borda | Baixa | APIs simples |
| Sliding Window | Janela deslizante com timestamps | Suave | Alta | APIs publicas |
| Token Bucket | Tokens recarregam a taxa constante | Controlado | Media | Developer APIs (padrao recomendado) |
| Leaky Bucket | Requisicoes processadas a taxa fixa | Nenhum | Baixa | Streaming, processamento uniforme |

```javascript
// Token Bucket com Redis (distribuido)
import { Ratelimit } from '@upstash/ratelimit';
import { Redis } from '@upstash/redis';

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.tokenBucket(10, '10 s', 20),
  // 10 tokens/10s, burst max 20
  analytics: true,
  prefix: 'api',
});

// Middleware
async function rateLimitMiddleware(req, res, next) {
  const identifier = req.user?.id || req.ip;
  const { success, limit, remaining, reset } = await ratelimit.limit(identifier);

  res.setHeader('X-RateLimit-Limit', limit);
  res.setHeader('X-RateLimit-Remaining', remaining);
  res.setHeader('X-RateLimit-Reset', reset);

  if (!success) {
    return res.status(429).json({
      error: 'Too Many Requests',
      retryAfter: Math.ceil((reset - Date.now()) / 1000),
    });
  }
  next();
}
```

**Limites recomendados por endpoint (2025):**

| Endpoint | Limite | Janela | Justificativa |
|----------|--------|--------|---------------|
| Login/Register | 5 req | 15 min | Anti brute-force |
| Reset de credenciais | 3 req | 1 hora | Anti abuso |
| API geral (auth) | 100 req | 1 min | Uso normal |
| API geral (public) | 30 req | 1 min | Prevenir abuso |
| Upload | 10 req | 1 hora | Recursos custosos |
| Webhook | 1000 req | 1 min | Volume alto esperado |

### 5.2 Input Validation com Zod

```typescript
import { z } from 'zod';

// Schema de validacao robusto
const CreateUserSchema = z.object({
  email: z.string()
    .email('Invalid email format')
    .max(255)
    .transform(v => v.toLowerCase().trim()),
  name: z.string()
    .min(2, 'Name too short')
    .max(100, 'Name too long')
    .regex(/^[a-zA-Z\s\-']+$/, 'Invalid characters in name'),
  secret: z.string()
    .min(12, 'Must be at least 12 characters')
    .regex(/[A-Z]/, 'Must contain uppercase')
    .regex(/[a-z]/, 'Must contain lowercase')
    .regex(/[0-9]/, 'Must contain number')
    .regex(/[^A-Za-z0-9]/, 'Must contain special character'),
  age: z.number()
    .int()
    .min(13, 'Must be at least 13')
    .max(120)
    .optional(),
  role: z.enum(['viewer', 'editor', 'admin'])
    .default('viewer'),
});

// Uso no endpoint
app.post('/api/users', async (req, res) => {
  const result = CreateUserSchema.safeParse(req.body);

  if (!result.success) {
    return res.status(400).json({
      error: 'Validation failed',
      details: result.error.flatten().fieldErrors,
    });
  }

  // result.data esta validado e tipado
  const user = await createUser(result.data);
  res.json(user);
});
```

### 5.3 CORS Configuration

```javascript
// PROIBIDO em producao
app.use(cors({ origin: '*' }));

// CORRETO: Whitelist explicito
const allowedOrigins = [
  'https://myapp.com',
  'https://www.myapp.com',
  'https://admin.myapp.com',
];

app.use(cors({
  origin: (origin, callback) => {
    // Permitir requests sem origin (mobile apps, Postman)
    if (!origin) return callback(null, true);

    if (allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'));
    }
  },
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  allowedHeaders: ['Content-Type', 'Authorization', 'X-Request-ID'],
  exposedHeaders: ['X-RateLimit-Limit', 'X-RateLimit-Remaining'],
  maxAge: 86400, // Cache preflight por 24h
}));
```

### 5.4 Security Headers

Fonte: [MDN CSP Guide](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CSP), [dotCMS Security Headers Best Practices](https://www.dotcms.com/blog/security-headers-best-practices), [Barrion Security Headers Guide](https://barrion.io/blog/security-headers-guide)

```javascript
import helmet from 'helmet';

app.use(helmet({
  // Content Security Policy
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],  // Usar nonce para inline scripts
      styleSrc: ["'self'", "'unsafe-inline'"],     // CSS inline necessario em muitos frameworks
      imgSrc: ["'self'", 'data:', 'https://cdn.myapp.com'],
      connectSrc: ["'self'", 'https://api.myapp.com', 'https://*.supabase.co'],
      fontSrc: ["'self'", 'https://fonts.gstatic.com'],
      objectSrc: ["'none'"],
      mediaSrc: ["'none'"],
      frameSrc: ["'none'"],
      baseUri: ["'self'"],
      formAction: ["'self'"],
      frameAncestors: ["'none'"],
      upgradeInsecureRequests: [],
    },
    reportOnly: false,  // Usar true primeiro para testar
  },
  // HSTS
  strictTransportSecurity: {
    maxAge: 63072000,        // 2 anos
    includeSubDomains: true,
    preload: true,
  },
  // Outros headers
  xFrameOptions: { action: 'deny' },
  xContentTypeOptions: true,        // nosniff
  referrerPolicy: { policy: 'strict-origin-when-cross-origin' },
  xXssProtection: false,            // Deprecado — CSP e substituto
}));

// Permissions-Policy (separado do helmet)
app.use((req, res, next) => {
  res.setHeader('Permissions-Policy',
    'camera=(), microphone=(), geolocation=(), payment=(self)'
  );
  next();
});
```

**Tabela de Security Headers essenciais:**

| Header | Funcao | Valor Recomendado |
|--------|--------|-------------------|
| Content-Security-Policy | Previne XSS, data injection | Strict CSP com nonces |
| Strict-Transport-Security | Forca HTTPS | max-age=63072000; includeSubDomains; preload |
| X-Frame-Options | Previne clickjacking | DENY |
| X-Content-Type-Options | Previne MIME sniffing | nosniff |
| Referrer-Policy | Controla envio de referer | strict-origin-when-cross-origin |
| Permissions-Policy | Restringe features do browser | camera=(), microphone=(), geolocation=() |

### 5.5 GraphQL Security

**Estatistica alarmante:** Aproximadamente 80% das APIs GraphQL permanecem vulneraveis a ataques DoS em 2025.

Fonte: [GraphQL.org Security](https://graphql.org/learn/security/), [OWASP GraphQL Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/GraphQL_Cheat_Sheet.html), [MarkAICode GraphQL DoS 2025](https://markaicode.com/graphql-api-dos-vulnerabilities-2025/)

**Problemas especificos do GraphQL:**

```graphql
# Ataque de profundidade (depth attack)
query MaliciousQuery {
  user(id: 1) {
    friends {
      friends {
        friends {
          friends {
            friends {
              id
            }
          }
        }
      }
    }
  }
}
```

```graphql
# Ataque de complexidade (alias attack)
query AliasBomb {
  a1: users(first: 1000) { id name email }
  a2: users(first: 1000) { id name email }
  a3: users(first: 1000) { id name email }
}
# 100 aliases = 100.000 registros
```

**Protecoes obrigatorias:**

```javascript
import depthLimit from 'graphql-depth-limit';
import { createComplexityLimitRule } from 'graphql-validation-complexity';

const server = new ApolloServer({
  schema,
  validationRules: [
    // 1. Limitar profundidade
    depthLimit(7),  // Maximo 7 niveis

    // 2. Limitar complexidade
    createComplexityLimitRule(1000, {
      scalarCost: 1,
      objectCost: 2,
      listFactor: 10,
    }),
  ],

  // 3. Desabilitar introspection em producao
  introspection: process.env.NODE_ENV !== 'production',

  // 4. Persisted queries (somente queries pre-aprovadas)
  persistedQueries: {
    cache: new InMemoryLRUCache(),
  },

  // 5. Timeout global
  plugins: [
    {
      requestDidStart: () => ({
        executionDidStart: () => {
          const timeout = setTimeout(() => {
            throw new Error('Query timeout');
          }, 10000); // 10 segundos
          return () => clearTimeout(timeout);
        },
      }),
    },
  ],
});
```

---

## 6. Secrets Management Advanced

### 6.1 Secret Scanning — Ferramentas

Fonte: [Gitleaks GitHub](https://github.com/gitleaks/gitleaks), [TruffleHog GitHub](https://github.com/trufflesecurity/trufflehog), [Snyk State of Secrets 2025](https://snyk.io/articles/state-of-secrets/)

**Dado alarmante:** 28 milhoes de credenciais vazaram no GitHub em 2025.

| Ferramenta | Stars | Linguagem | Diferencial | False Positives |
|------------|-------|-----------|-------------|-----------------|
| Gitleaks | ~25,700 | Go | Velocidade, regras TOML customizaveis | Altos (regex sem verificacao) |
| TruffleHog | ~17,000+ | Go | Verifica se secrets sao ATIVOS (live) | Baixos (--only-verified) |
| git-secrets | ~13,200 | Shell | Leve, foco AWS | Baixos (escopo limitado) |
| Kingfisher (2025) | Novo | — | 2-5x mais rapido que Gitleaks, blast radius | Baixos |
| GitHub Secret Scanning | Built-in | — | Push protection, pattern config | Baixos |

### 6.2 Pre-commit Hooks para Secrets

```yaml
# .pre-commit-config.yaml — TruffleHog
repos:
  - repo: https://github.com/trufflesecurity/trufflehog
    rev: v3.88.0
    hooks:
      - id: trufflehog
        entry: trufflehog git file://. --since-commit HEAD --only-verified --fail
        language: system
```

```yaml
# Alternativa: Gitleaks
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.21.2
    hooks:
      - id: gitleaks
```

```toml
# .gitleaks.toml — Configuracao customizada
title = "Custom Gitleaks Config"

[allowlist]
  paths = [
    '''.env\.example''',
    '''test/fixtures/''',
  ]

[[rules]]
  id = "supabase-service-role"
  description = "Supabase Service Role Key"
  regex = '''eyJhbGciOi[A-Za-z0-9_-]{20,}\.[A-Za-z0-9_-]{50,}'''
  tags = ["supabase", "key"]
```

### 6.3 GitHub Secret Scanning e Push Protection

Fonte: [GitHub Push Protection docs](https://docs.github.com/en/code-security/secret-scanning/enabling-secret-scanning-features/enabling-push-protection-for-your-repository), [GitHub Changelog Aug 2025](https://github.blog/changelog/2025-08-19-secret-scanning-configuring-patterns-in-push-protection-is-now-generally-available/)

**Novidade 2025:** Configuracao de quais patterns sao protegidos por push protection agora esta Generally Available (GA). Organizacoes podem customizar quais padroes bloqueiam push no nivel Enterprise ou Organization.

```bash
# Verificar se push protection esta ativo
gh api repos/OWNER/REPO --jq '.security_and_analysis.secret_scanning_push_protection.status'
# Esperado: "enabled"
```

### 6.4 HashiCorp Vault — Dynamic Secrets

Fonte: [Vault Database Secrets](https://developer.hashicorp.com/vault/tutorials/db-credentials/database-secrets), [Vault Auto-Rotation](https://developer.hashicorp.com/hcp/docs/vault-secrets/auto-rotation)

**Dynamic secrets** sao credenciais geradas on-demand com TTL curto. Quando o TTL expira, Vault automaticamente revoga a credencial. Isso elimina o risco de credenciais estaticas comprometidas.

```bash
# Configurar database secrets engine
vault secrets enable database

# Configurar conexao PostgreSQL
vault write database/config/mydb \
  plugin_name=postgresql-database-plugin \
  allowed_roles="readonly,readwrite" \
  username="vault_admin"

# Criar role de leitura (TTL 1 hora, max 24 horas)
vault write database/roles/readonly \
  db_name=mydb \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN VALID UNTIL '{{expiration}}'; \
    GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  revocation_statements="REVOKE ALL PRIVILEGES ON ALL TABLES IN SCHEMA public FROM \"{{name}}\"; \
    DROP ROLE IF EXISTS \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"

# Obter credenciais dinamicas
vault read database/creds/readonly
# Retorna: username=v-token-readonly-abc123, lease_duration=1h
```

**Auto-rotation de root credentials:**

```bash
# Rotacao automatica a cada sabado a meia-noite
vault write database/config/mydb \
  rotation_schedule="0 0 * * SAT" \
  rotation_window="1h"
```

### 6.5 Vercel + Supabase Secrets Management

Fonte: [Supabase Environment Variables](https://supabase.com/docs/guides/functions/secrets), [Vercel Supabase Integration](https://vercel.com/marketplace/supabase)

**Padroes recomendados:**

| Variavel | Prefixo | Exposicao | Uso |
|----------|---------|-----------|-----|
| NEXT_PUBLIC_SUPABASE_URL | NEXT_PUBLIC_ | Client-side | OK (URL publica) |
| NEXT_PUBLIC_SUPABASE_ANON_KEY | NEXT_PUBLIC_ | Client-side | OK (com RLS ativo) |
| SUPABASE_SERVICE_ROLE_KEY | SEM prefixo | Server-side only | API routes, Edge Functions |
| DATABASE_URL | SEM prefixo | Server-side only | Conexao direta ao DB |
| SUPABASE_JWT_SECRET | SEM prefixo | Server-side only | Verificacao de tokens |

```bash
# Vercel: Escopo por ambiente
# Production: Usa as credenciais de producao
# Preview: Usa credenciais de staging (branch previews)
# Development: Usa credenciais locais

# Verificar variaveis (sem expor valores)
vercel env ls

# Supabase Edge Functions: Secrets seguros
supabase secrets set MY_API_KEY=value_here
supabase secrets list  # Mostra nomes, nunca valores
```

**Regra critica:** NUNCA usar `NEXT_PUBLIC_` para secrets. Variaveis com esse prefixo sao incluidas no bundle JavaScript do client e visiveis para qualquer usuario.

---

## 7. Zero Trust Architecture

### 7.1 Principios Fundamentais

Fonte: [Cloudflare Zero Trust Implementation](https://developers.cloudflare.com/reference-architecture/implementation-guides/zero-trust/), [Wikipedia Zero Trust](https://en.wikipedia.org/wiki/Zero_trust_architecture), [BeyondCorp](https://www.beyondcorp.com/)

**"Never trust, always verify"** — Nenhum usuario, dispositivo ou servico e confiavel por padrao, mesmo dentro da rede corporativa.

| Principio | Descricao |
|-----------|-----------|
| Least Privilege | Acesso minimo necessario para cada tarefa |
| Microsegmentation | Rede dividida em segmentos isolados |
| Continuous Verification | Validar identidade e contexto em CADA requisicao |
| Assume Breach | Projetar assumindo que o atacante ja esta dentro |
| Device Trust | Verificar saude do dispositivo alem da identidade do usuario |

### 7.2 BeyondCorp (Google)

Desenvolvido apos a Operation Aurora (ataque APT chines em 2009), BeyondCorp elimina a nocao de rede interna confiavel.

**Principios-chave:**
- Todas as aplicacoes sao deployadas na internet publica
- Acesso determinado por identidade do usuario + estado do dispositivo + contexto
- Sem VPN corporativa — acesso direto via proxy de identidade
- Cada requisicao e avaliada individualmente

### 7.3 Cloudflare Zero Trust

Cloudflare oferece uma stack completa de Zero Trust:

| Produto | Funcao |
|---------|--------|
| Cloudflare Access | Identity-aware proxy para apps internas |
| Cloudflare Tunnel | Conexao segura sem expor portas |
| Cloudflare Gateway | DNS filtering e secure web gateway |
| Cloudflare Browser Isolation | Isolar execucao de paginas web |
| Cloudflare CASB | Monitorar SaaS para misconfigurations |

```bash
# Cloudflare Tunnel: Expor servico interno sem abrir portas
cloudflared tunnel create my-app
cloudflared tunnel route dns my-app internal.myapp.com
```

```yaml
# Configuracao do tunnel — config.yml
tunnel: <TUNNEL_ID>
credentials-file: /root/.cloudflared/<TUNNEL_ID>.json
ingress:
  - hostname: internal.myapp.com
    service: http://localhost:3000
  - service: http_status:404
```

### 7.4 Network Segmentation com Vercel/Supabase

| Camada | Protecao |
|--------|----------|
| Vercel Edge Network | DDoS protection, WAF, geo-blocking |
| Vercel Secure Compute | Funcoes em VPC isolada |
| Supabase Network Restrictions | Whitelist de IPs para acesso ao DB |
| Supabase SSL Enforcement | TLS obrigatorio para todas as conexoes |
| Supabase Branch Databases | Ambientes isolados por branch |

```sql
-- Supabase: Restringir acesso ao banco por IP
-- Settings > Database > Network Restrictions
-- Adicionar APENAS os IPs da sua infraestrutura:
-- - Vercel Edge Functions IPs
-- - Seu IP de desenvolvimento
-- - IPs do CI/CD
```

### 7.5 mTLS (Mutual TLS)

Fonte: [GitGuardian mTLS Guide](https://blog.gitguardian.com/mutual-tls-mtls-authentication/), [Smallstep mTLS Implementation](https://smallstep.com/docs/mtls/)

No mTLS, tanto o cliente quanto o servidor apresentam certificados digitais, garantindo autenticacao bidirecional. Elimina necessidade de API keys ou tokens para identidade de servicos.

**Implementacao com service mesh:**
- **Istio:** mTLS automatico entre todos os pods (PeerAuthentication STRICT)
- **Linkerd:** mTLS habilitado por padrao em todas as conexoes
- **Consul Connect:** mTLS com rotacao automatica de certificados

```yaml
# Istio: Forcar mTLS em todo o namespace
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT
```

---

## 8. Compliance Frameworks

### 8.1 LGPD — Lei Geral de Protecao de Dados (Brasil)

Fonte: [ICLG Data Protection Brazil](https://iclg.com/practice-areas/data-protection-laws-and-regulations/brazil), [ComplyDog LGPD Guide](https://complydog.com/blog/brazil-lgpd-complete-data-protection-compliance-guide-saas), [SecurePrivacy LGPD Checklist](https://secureprivacy.ai/blog/lgpd-compliance-requirements)

| Requisito | Artigo | Descricao | Acao Necessaria |
|-----------|--------|-----------|-----------------|
| Base legal | Art. 7 | Definir base legal para cada tratamento | Documentar base legal (consentimento, contrato, interesse legit.) |
| Consentimento | Art. 8 | Consentimento livre, informado, inequivoco | Formulario opt-in explicito |
| Direitos do titular | Art. 18 | Acesso, correcao, exclusao, portabilidade | Portal de direitos do titular (15 dias para responder) |
| DPO/Encarregado | Art. 41 | Nomear DPO | Designar e publicar dados de contato |
| RIPD | Art. 38 | Relatorio de Impacto a Protecao de Dados | DPIA antes de tratamentos de risco |
| Transferencia int'l | Art. 33 | SCCs obrigatorias (agosto 2025) | Implementar 24 clausulas ANPD |
| Notificacao de incidente | Art. 48 | Notificar ANPD + titulares | Dentro de 72 horas |
| Criancas | Art. 14 | Consentimento dos pais | Mecanismo especifico para menores |
| Politica de privacidade | Art. 9 | Publicar politica acessivel | Pagina publica com linguagem clara |

**Penalidades:** Multa de ate 2% do faturamento no Brasil, limitada a R$ 50 milhoes por infracao. ANPD tem demonstrado disposicao para agir preventivamente (suspensoes da Meta e X Corp).

**Deadline critico 2025:** A partir de 23 de agosto de 2025, transferencias internacionais de dados so sao validas com SCCs implementadas ou outros mecanismos aprovados pela ANPD.

Fonte: [Mayer Brown SCCs](https://www.mayerbrown.com/en/insights/publications/2025/08/end-of-grace-period-implementation-of-brazils-standard-contractual-clauses-in-international-transfers-of-personal-data)

### 8.2 SOC 2 Type II

| Aspecto | Descricao |
|---------|-----------|
| O que e | Atestacao de controles de seguranca por auditor independente |
| Periodo | Type I: ponto no tempo / Type II: 3-12 meses de observacao |
| 5 Trust Principles | Security (obrigatorio), Availability, Processing Integrity, Confidentiality, Privacy |
| Quem precisa | SaaS B2B, qualquer empresa que processa dados de clientes enterprise |
| Tempo para obter | 6-12 meses (primeiro) / 3-6 meses (renovacao) |
| Custo auditor | $30K-$100K+ dependendo do escopo |

**Controles essenciais:**
- Access control (MFA, RBAC, revisao periodica)
- Change management (CI/CD com aprovacoes)
- Incident response plan documentado e testado
- Vulnerability management (scans regulares)
- Vendor management (avaliar seguranca de terceiros)
- Encryption (in transit + at rest)
- Logging e monitoring

### 8.3 ISO 27001

| Aspecto | Descricao |
|---------|-----------|
| O que e | Standard internacional para Information Security Management System (ISMS) |
| Certificacao | Auditor acreditado certifica por 3 anos (auditorias anuais) |
| Anexo A | 93 controles em 4 categorias (Organizacionais, Pessoas, Fisicos, Tecnologicos) |
| Quem precisa | Empresas que vendem para Europa, governo, ou grandes corporacoes |
| Custo | $15K-$70K (certificacao) + custos internos |
| Revisao 2022 | ISO 27001:2022 — atualizada com cloud, threat intelligence, data masking |

### 8.4 PCI DSS 4.0

Fonte: [PCI DSS Blog](https://blog.pcisecuritystandards.org/countdown-to-pci-dss-v4.0), [Basis Theory PCI 2025](https://blog.basistheory.com/pci-requirements-in-2025)

**A partir de 1 de abril de 2025,** todos os 51 requisitos "future-dated" do PCI DSS 4.0 se tornaram obrigatorios.

**Mudancas criticas:**
- MFA obrigatorio para TODO acesso ao ambiente de dados de cartao
- Scripts de pagina de checkout devem ser inventariados e justificados (Req 6.4, 11.6)
- Criptografia robusta obrigatoria (especialmente whole-disk encryption)
- Escopo do assessment documentado e revisado anualmente (ou semestralmente para TPSPs)
- Gestao de seguranca de third-party vendors reforcada

### 8.5 HIPAA

Fonte: [HIPAA Security Rule NPRM](https://www.hhs.gov/hipaa/for-professionals/security/hipaa-security-rule-nprm/factsheet/index.html), [HIPAA Journal Updates](https://www.hipaajournal.com/hipaa-updates-hipaa-changes/)

**Proposta de atualizacao (janeiro 2025):**
- Eliminacao da distincao "required" vs "addressable" — TUDO obrigatorio
- Criptografia de ePHI em transito E em repouso obrigatoria
- MFA obrigatorio
- Vulnerability scans semestrais, pentest anual
- Inventario de ativos e mapa de rede atualizado anualmente
- Restauracao de servicos em ate 72 horas apos incidente

### 8.6 Compliance Automation Tools

Fonte: [SecureLeap SOC 2 Tools Guide 2025](https://www.secureleap.tech/blog/soc-2-tools-vanta-drata-secureframe-guide-2025), [Comp AI Vanta vs Drata](https://trycomp.ai/vanta-vs-drata)

| Ferramenta | Frameworks | Integracoes | Preco (startup) | Diferencial |
|------------|-----------|-------------|-----------------|-------------|
| Vanta | SOC 2, ISO 27001, HIPAA, PCI, GDPR | 375+ | ~$10K/ano | Setup rapido, breadth |
| Drata | SOC 2, ISO 27001, HIPAA, PCI, DORA, NIS2 | 300+ | ~$7.5K/ano | Customizacao, CI/CD monitoring |
| Secureframe | SOC 2, ISO 27001, HIPAA, PCI | 150+ | ~$8K/ano | Simplicidade |
| Sprinto | SOC 2, ISO 27001, HIPAA, GDPR | 200+ | ~$6K/ano | Preco competitivo |

**49% das organizacoes ja usam ferramentas de automacao de compliance (PwC 2025).**

---

## 9. Incident Response

### 9.1 Template de Plano de Resposta a Incidentes

Fonte: [Atlassian Postmortem Templates](https://www.atlassian.com/incident-management/postmortem/templates), [Rootly SRE Best Practices 2025](https://rootly.com/sre/2025-sre-incident-management-best-practices-checklist)

**Classificacao de Severidade:**

| Severidade | Criterio | Tempo de Resposta | Exemplos |
|------------|----------|-------------------|----------|
| SEV-1 (Critical) | Servico completamente indisponivel ou breach confirmado | 15 minutos | DB down, dados vazados, ransomware |
| SEV-2 (High) | Degradacao severa ou vulnerabilidade explorada | 1 hora | API lenta, feature core quebrada |
| SEV-3 (Medium) | Degradacao parcial, workaround disponivel | 4 horas | Feature secundaria com bug |
| SEV-4 (Low) | Issue menor, sem impacto em usuarios | 24 horas | Typo em UI, warning no log |

**Fases de Resposta:**

```
Fase 1: Deteccao e Triagem (0-15 min)
  - Alerta recebido e reconhecido
  - Severidade classificada
  - Incident Commander designado
  - Canal de comunicacao criado (#incident-YYYY-MM-DD)
  - Stakeholders notificados conforme severidade

Fase 2: Contencao (15-60 min)
  - Impacto avaliado (quantos usuarios, quais dados)
  - Acoes de contencao executadas (isolamento, rollback)
  - Evidencias preservadas (logs, snapshots)
  - Comunicacao inicial enviada (interna)

Fase 3: Erradicacao (1-24h)
  - Causa raiz identificada
  - Vulnerabilidade corrigida ou mitigada
  - Sistemas afetados verificados
  - Credenciais comprometidas rotacionadas

Fase 4: Recuperacao (variavel)
  - Servicos restaurados
  - Monitoramento intensificado
  - Dados restaurados de backup (se necessario)
  - Validacao com usuarios

Fase 5: Pos-Incidente (48-72h)
  - Postmortem agendado (dentro de 48-72 horas)
  - Timeline documentada
  - Action items definidos com owners
  - Comunicacao final enviada
```

### 9.2 Procedimentos de Escalacao

```
SEV-1: On-call -> Incident Commander -> VP Engineering -> CEO -> Juridico
        |                                    |
        +-- Supabase Support (se DB)         +-- ANPD (se breach de dados pessoais)

SEV-2: On-call -> Tech Lead -> Engineering Manager
SEV-3: On-call -> Tech Lead
SEV-4: Backlog (proximo sprint)
```

### 9.3 Templates de Comunicacao

**Comunicacao interna (SEV-1/SEV-2):**

```
INCIDENTE EM ANDAMENTO

Severidade: SEV-1
Inicio: 2025-XX-XX HH:MM UTC
Status: Investigando / Contido / Resolvido
Impacto: [Descricao do impacto para usuarios]
Incident Commander: [Nome]
Proxima atualizacao: em [30/60] minutos

Timeline:
- HH:MM — Alerta detectado por [sistema]
- HH:MM — Equipe de resposta acionada
- HH:MM — [Acao tomada]
```

**Comunicacao externa (breach — LGPD):**

```
NOTIFICACAO DE INCIDENTE DE SEGURANCA

Prezado(a) [Titular],

Identificamos um incidente de seguranca em [data] que pode ter afetado
seus dados pessoais.

Dados potencialmente afetados: [lista]
Periodo do incidente: [data inicio] a [data fim]
Medidas tomadas: [descricao]
Recomendacoes: [o que o usuario deve fazer]

Para mais informacoes ou exercer seus direitos (Art. 18 LGPD),
entre em contato: [email DPO]
```

### 9.4 Processo de Postmortem Blameless

Fonte: [DevSeatIt Blameless Postmortem](https://devseatit.com/sre-practices/blameless-postmortem/), [FireHydrant Template](https://firehydrant.com/blog/incident-retrospective-postmortem-template/)

**Regras fundamentais:**
1. Sem apontar dedos — todos fizeram o melhor com as informacoes disponiveis
2. Foco em fatos, nao opinioes
3. Perguntar "por que" 5 vezes (5 Whys) para encontrar causa raiz
4. Documentar aprendizados, nao julgamentos
5. Action items com donos, prioridades e prazos

**Template de Postmortem:**

```
POSTMORTEM — [Titulo do Incidente]

Metadata:
  Data: YYYY-MM-DD
  Autores: [Nomes]
  Status: Draft | Review | Final
  Severidade: SEV-X
  Duracao: X horas Y minutos
  Incident Commander: [Nome]

Resumo Executivo:
  [2-3 frases descrevendo o que aconteceu, impacto, e resolucao]

Impacto:
  Usuarios afetados: X
  Receita perdida: R$ X
  Duracao do impacto: Xh Xm
  Dados expostos: [sim/nao, quais]

Timeline:
  HH:MM UTC — Primeiro alerta
  HH:MM UTC — Equipe acionada
  HH:MM UTC — Causa identificada
  HH:MM UTC — Fix aplicado
  HH:MM UTC — Servico restaurado

Analise de Causa Raiz (5 Whys):
  1. Por que o servico caiu? -> Porque o DB ficou sem conexoes
  2. Por que ficou sem conexoes? -> Porque um query lento bloqueou o pool
  3. Por que o query era lento? -> Porque faltava indice na tabela X
  4. Por que faltava indice? -> Porque o migration nao foi revisado
  5. Por que nao foi revisado? -> Porque nao temos checklist de review de DB

Action Items:
  | ID | Acao                           | Dono       | Prioridade | Prazo     |
  |----|--------------------------------|------------|------------|-----------|
  | 1  | Adicionar indice na tabela X   | @dev       | P0         | 2 dias    |
  | 2  | Criar checklist de review de DB| @architect | P1         | 1 semana  |
  | 3  | Alertar quando pool > 80%      | @devops    | P1         | 3 dias    |
```

### 9.5 Notificacao de Breach — LGPD

**Prazo:** 72 horas apos tomar conhecimento do incidente.

**Para ANPD, comunicar:**
1. Natureza dos dados pessoais afetados
2. Numero de titulares afetados (estimativa)
3. Medidas tecnicas e de seguranca utilizadas
4. Riscos gerados pelo incidente
5. Medidas adotadas para reverter ou mitigar
6. Motivo do atraso, se comunicacao nao for imediata

---

## 10. Security Checklist por Tipo de Projeto

### 10.1 MVP / Startup (Budget: $0-500/mes)

**Objetivo:** Seguranca viavel minima sem bloquear velocidade.

| Categoria | Ferramenta/Acao | Custo |
|-----------|-----------------|-------|
| Auth | Supabase Auth (built-in MFA) | Gratis |
| RLS | Habilitar em TODAS as tabelas | Gratis |
| SAST | Semgrep OSS no CI | Gratis |
| SCA | npm audit + Dependabot | Gratis |
| Secrets | gitleaks pre-commit hook | Gratis |
| HTTPS | Vercel (automatico) | Gratis |
| Headers | helmet.js | Gratis |
| Rate limiting | Upstash Ratelimit | Free tier |
| Monitoring | Sentry free tier | Gratis |
| Compliance | Politica de privacidade (LGPD) | Gratis |
| **Total** | | **$0/mes** |

**Checklist MVP:**
- [ ] RLS ativado em todas as tabelas com dados de usuario
- [ ] service_role NUNCA no frontend
- [ ] .env em .gitignore + .env.example existe
- [ ] npm audit sem vulnerabilidades critical/high
- [ ] gitleaks rodando no pre-commit
- [ ] HTTPS forcado
- [ ] Headers de seguranca basicos (helmet)
- [ ] Rate limiting em endpoints de autenticacao
- [ ] Politica de privacidade publicada
- [ ] Backup do banco configurado

---

### 10.2 Growth Stage (Budget: $500-5,000/mes)

**Objetivo:** Escalar seguranca junto com a base de usuarios.

| Categoria | Ferramenta/Acao | Custo |
|-----------|-----------------|-------|
| Auth | Passkeys + MFA | Incluido no auth provider |
| SAST | Semgrep Pro ou Snyk Code | ~$200-500/mes |
| DAST | OWASP ZAP no CI/CD | Gratis |
| SCA | Snyk Open Source + Socket.dev | ~$200/mes |
| Secrets | TruffleHog + GitHub Secret Scanning | Gratis |
| WAF | Cloudflare Pro + rate limiting | ~$200/mes |
| Monitoring | Sentry Pro + Datadog APM | ~$300-500/mes |
| Compliance | Vanta ou Drata (SOC 2 prep) | ~$625-833/mes |
| Pentest | Anual com consultoria | ~$250/mes (diluido) |
| DPO | Consultor LGPD | ~$500/mes |
| **Total** | | **~$2,500-3,500/mes** |

**Checklist Growth:**
- [ ] Tudo do MVP +
- [ ] Passkeys como metodo primario de auth
- [ ] SAST automatizado em todo PR
- [ ] DAST rodando contra staging semanal
- [ ] Dependency monitoring continuo (Snyk Monitor)
- [ ] WAF configurado com regras customizadas
- [ ] Logging centralizado com alertas
- [ ] Incident response plan documentado
- [ ] Pentest anual
- [ ] SOC 2 Type II em progresso
- [ ] DPO designado (LGPD)
- [ ] Data retention policy definida
- [ ] SBOM gerado a cada release

---

### 10.3 Enterprise (Budget: $5,000-50,000+/mes)

**Objetivo:** Programa completo de seguranca com compliance multi-framework.

| Categoria | Ferramenta/Acao | Custo |
|-----------|-----------------|-------|
| Auth | Enterprise SSO (SAML/OIDC) + Passkeys | $1K-5K/mes |
| IAM | WorkOS ou Auth0 Enterprise | $1K-3K/mes |
| SAST/DAST/SCA | Snyk Enterprise ou plataforma ASPM | $2K-5K/mes |
| Secrets | HashiCorp Vault managed | $1K-3K/mes |
| Zero Trust | Cloudflare Zero Trust ou Zscaler | $2K-10K/mes |
| SIEM | Datadog Security ou Splunk Cloud | $3K-10K/mes |
| Compliance | Vanta Enterprise (SOC 2 + ISO 27001 + HIPAA + PCI) | $2.5K-6.5K/mes |
| Bug Bounty | HackerOne ou Bugcrowd | $1K-5K/mes |
| Pentest | Trimestral com firma tier-1 | $2K-8K/mes (diluido) |
| Security team | CISO + Security Engineer | $15K-30K/mes |
| **Total** | | **$30K-85K+/mes** |

**Checklist Enterprise:**
- [ ] Tudo do Growth +
- [ ] Zero Trust implementado
- [ ] mTLS entre servicos
- [ ] Dynamic secrets (Vault)
- [ ] SOC 2 Type II certificado
- [ ] ISO 27001 certificado
- [ ] PCI DSS compliance (se processa pagamentos)
- [ ] HIPAA compliance (se processa dados de saude)
- [ ] SIEM com correlacao de eventos 24/7
- [ ] Red team exercises semestrais
- [ ] Bug bounty program ativo
- [ ] Vendor security assessments
- [ ] Business continuity plan testado
- [ ] Disaster recovery testado
- [ ] Security awareness training trimestral
- [ ] Threat modeling para features criticas
- [ ] Data classification policy implementada
- [ ] Network segmentation completa
- [ ] Incident response testado com tabletop exercises

---

## Decision Framework — Escolha de Ferramentas

### Por Fase do Projeto

```
MVP (0-$500/mes):
  Auth: Supabase Auth
  SAST: Semgrep OSS
  SCA: npm audit + Dependabot
  Secrets: gitleaks
  Infra: Vercel + Supabase (security built-in)

Growth ($500-5K/mes):
  Auth: Supabase Auth + Passkeys
  SAST: Semgrep Pro ou Snyk Code
  SCA: Snyk + Socket.dev
  DAST: OWASP ZAP
  Secrets: TruffleHog + GitHub Secret Scanning
  Compliance: Vanta ou Drata
  Infra: Cloudflare Pro + Vercel + Supabase

Enterprise ($5K+/mes):
  Auth: Enterprise SSO + WorkOS
  SAST/DAST/SCA: Snyk Enterprise (plataforma unica)
  Secrets: HashiCorp Vault
  Zero Trust: Cloudflare ZT ou Zscaler
  SIEM: Datadog Security
  Compliance: Vanta Enterprise (multi-framework)
  Infra: Cloudflare Enterprise + Vercel Enterprise + Supabase Pro
```

### Por Tipo de Ameaca

| Ameaca | Ferramenta Primaria | Ferramenta Secundaria |
|--------|--------------------|-----------------------|
| Injection (SQLi, XSS) | Semgrep/CodeQL (SAST) | OWASP ZAP (DAST) |
| Supply chain | Socket.dev + Snyk | npm audit + lockfile audit |
| Credential theft | Passkeys + MFA | Rate limiting + monitoring |
| Secret leaks | gitleaks/TruffleHog | GitHub Secret Scanning |
| DDoS/Abuse | Cloudflare WAF | Rate limiting (Upstash) |
| Insider threat | RBAC + audit logs | SIEM + behavior analytics |
| Misconfigurations | Semgrep (config rules) | Cloud security posture (CSPM) |

---

## Fontes Consolidadas

### OWASP
- [OWASP Top 10:2025](https://owasp.org/Top10/2025/)
- [OWASP A01:2025 Broken Access Control](https://owasp.org/Top10/2025/A01_2025-Broken_Access_Control/)
- [OWASP GraphQL Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/GraphQL_Cheat_Sheet.html)
- [OWASP CSP Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Content_Security_Policy_Cheat_Sheet.html)

### Supply Chain
- [CISA npm Supply Chain Alert (Sep 2025)](https://www.cisa.gov/news-events/alerts/2025/09/23/widespread-supply-chain-compromise-impacting-npm-ecosystem)
- [Palo Alto Networks npm Attack Analysis](https://www.paloaltonetworks.com/blog/cloud-security/npm-supply-chain-attack/)
- [Rescana Analysis — event-stream, ua-parser-js](https://www.rescana.com/post/in-depth-analysis-supply-chain-poisoning-of-popular-npm-packages-exploiting-event-stream-ua-parser)
- [Snyk State of Secrets 2025](https://snyk.io/articles/state-of-secrets/)

### Ferramentas SAST/DAST/SCA
- [sanj.dev AI Code Security Comparison](https://sanj.dev/post/ai-code-security-tools-comparison)
- [StackHawk SAST Tools 2025](https://www.stackhawk.com/blog/best-sast-tools-comparison/)
- [AppSec Santa Burp vs ZAP](https://appsecsanta.com/burp-suite-vs-zap)
- [Rafter DAST Comparison](https://rafter.so/blog/dast-tools-comparison)
- [Semgrep CI Configuration](https://semgrep.dev/docs/semgrep-ci/sample-ci-configs)

### Authentication
- [Auth0 PKCE Flow](https://auth0.com/docs/get-started/authentication-and-authorization-flow/authorization-code-flow-with-pkce)
- [OAuth 2.1 Spec](https://oauth.net/2.1/)
- [Auth0 Token Best Practices](https://auth0.com/docs/secure/tokens/token-best-practices)
- [1Password State of Passkeys 2025](https://www.1password.community/blog/random-but-memorable/the-state-of-passkeys-in-2025/163464)
- [AuthSignal Passwordless 2025](https://www.authsignal.com/blog/articles/passwordless-authentication-in-2025-the-year-passkeys-went-mainstream)

### Access Control
- [Oso RBAC vs ABAC vs ReBAC](https://www.osohq.com/learn/rbac-vs-abac-vs-rebac-what-is-the-best-access-policy-paradigm)
- [Permit.io Authorization Models](https://www.permit.io/blog/rbac-vs-abac-and-rebac-choosing-the-right-authorization-model)

### Supabase
- [Supabase Auth Architecture](https://supabase.com/docs/guides/auth/architecture)
- [Supabase Custom Claims and RBAC](https://supabase.com/docs/guides/database/postgres/custom-claims-and-role-based-access-control-rbac)
- [Supabase Auth Hooks](https://supabase.com/docs/guides/auth/auth-hooks)
- [Supabase Environment Variables](https://supabase.com/docs/guides/functions/secrets)

### API Security
- [API7 Rate Limiting Guide](https://api7.ai/blog/rate-limiting-guide-algorithms-best-practices)
- [Arcjet Rate Limiting Algorithms](https://blog.arcjet.com/rate-limiting-algorithms-token-bucket-vs-sliding-window-vs-fixed-window/)
- [Zuplo Rate Limiting Best Practices 2025](https://zuplo.com/learning-center/10-best-practices-for-api-rate-limiting-in-2025)
- [MDN Content Security Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CSP)
- [GraphQL.org Security](https://graphql.org/learn/security/)

### Secrets Management
- [Gitleaks GitHub](https://github.com/gitleaks/gitleaks)
- [TruffleHog GitHub](https://github.com/trufflesecurity/trufflehog)
- [GitHub Push Protection](https://docs.github.com/en/code-security/secret-scanning/enabling-secret-scanning-features/enabling-push-protection-for-your-repository)
- [HashiCorp Vault Database Secrets](https://developer.hashicorp.com/vault/tutorials/db-credentials/database-secrets)

### Zero Trust
- [Cloudflare Zero Trust Implementation](https://developers.cloudflare.com/reference-architecture/implementation-guides/zero-trust/)
- [Cloudflare Zero Trust for Startups](https://developers.cloudflare.com/reference-architecture/design-guides/zero-trust-for-startups/)
- [BeyondCorp](https://www.beyondcorp.com/)
- [GitGuardian mTLS Guide](https://blog.gitguardian.com/mutual-tls-mtls-authentication/)

### Compliance
- [ICLG Brazil Data Protection](https://iclg.com/practice-areas/data-protection-laws-and-regulations/brazil)
- [ComplyDog LGPD Guide](https://complydog.com/blog/brazil-lgpd-complete-data-protection-compliance-guide-saas)
- [Mayer Brown SCCs Brazil](https://www.mayerbrown.com/en/insights/publications/2025/08/end-of-grace-period-implementation-of-brazils-standard-contractual-clauses-in-international-transfers-of-personal-data)
- [PCI DSS 4.0 Countdown](https://blog.pcisecuritystandards.org/countdown-to-pci-dss-v4.0)
- [HIPAA Security Rule NPRM](https://www.hhs.gov/hipaa/for-professionals/security/hipaa-security-rule-nprm/factsheet/index.html)
- [SecureLeap SOC 2 Guide 2025](https://www.secureleap.tech/blog/soc-2-tools-vanta-drata-secureframe-guide-2025)

### Incident Response
- [Atlassian Postmortem Templates](https://www.atlassian.com/incident-management/postmortem/templates)
- [Rootly SRE Best Practices 2025](https://rootly.com/sre/2025-sre-incident-management-best-practices-checklist)
- [FireHydrant Postmortem Template](https://firehydrant.com/blog/incident-retrospective-postmortem-template/)
- [DevSeatIt Blameless Postmortem](https://devseatit.com/sre-practices/blameless-postmortem/)

### Dependency Management
- [Renovate Bot Comparison](https://docs.renovatebot.com/bot-comparison/)
- [Socket.dev npm Security](https://socket.dev/)
- [npm SBOM docs](https://docs.npmjs.com/cli/v9/commands/npm-sbom/)

### Data Breaches (Licoes)
- [HIPAA Journal Biggest Breaches 2024](https://www.hipaajournal.com/biggest-healthcare-data-breaches-2024/)
- [Axios Data Breach Victims 2024](https://www.axios.com/2025/01/28/ticketmaster-advance-auto-parts-data-breaches-victims)
