# Observability & Monitoring -- Stack Completa para Aplicacoes Web Modernas

> **Nivel de pesquisa:** DEEP DIVE | **Fontes:** 40+ verificadas via WebSearch | **Escopo:** MVP ate Enterprise
> **Ultima atualizacao:** Abril 2026

---

## Indice

1. [Os Tres Pilares da Observability](#1-os-tres-pilares-da-observability)
2. [Logging Stack](#2-logging-stack)
3. [Metricas & Dashboards](#3-metricas--dashboards)
4. [Application Performance Monitoring (APM)](#4-application-performance-monitoring-apm)
5. [Alerting](#5-alerting)
6. [Infrastructure Monitoring](#6-infrastructure-monitoring)
7. [Observability para Serverless](#7-observability-para-serverless)
8. [Observability Maturity Model](#8-observability-maturity-model)
9. [Otimizacao de Custos](#9-otimizacao-de-custos)

---

## 1. Os Tres Pilares da Observability

Observability e a capacidade de entender o estado interno de um sistema a partir dos dados que ele emite. Os tres pilares classicos -- **Logs**, **Metrics** e **Traces** -- fornecem perspectivas complementares sobre a saude e o comportamento de sistemas distribuidos.

> **Principio fundamental:** Metricas alertam sobre problemas, traces mostram o caminho de execucao, e logs fornecem o contexto necessario para resolver.
> Fonte: [IBM -- Three Pillars of Observability](https://www.ibm.com/think/insights/observability-pillars)

### 1.1 Logs -- Registro Estruturado de Eventos

Logs sao registros historicos de eventos do sistema -- podem ser texto puro, binario ou estruturados com metadados.

**Structured Logging** e a pratica de emitir logs em formato parseavel (JSON) em vez de strings livres. Isso permite filtragem, busca e correlacao automatizada.

#### Log Levels (RFC 5424)

| Level | Uso | Quando usar |
|-------|-----|-------------|
| `fatal` | Sistema parou | Crash irrecuperavel |
| `error` | Operacao falhou | Exception capturada, request falhado |
| `warn` | Potencial problema | Deprecation, retry necessario |
| `info` | Evento de negocio | Requisicao processada, usuario criado |
| `debug` | Detalhe tecnico | Query SQL, payload de request |
| `trace` | Granularidade maxima | Entrada/saida de funcao |

**Regra pratica:** Em producao, use `info` como nivel minimo. `debug` e `trace` apenas sob demanda (via feature flag ou sampling).

#### Exemplo -- Structured Log com Pino (Node.js)

```javascript
import pino from 'pino';

const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  formatters: {
    level: (label) => ({ level: label }),
  },
  timestamp: pino.stdTimeFunctions.isoTime,
});

// Log estruturado com contexto
logger.info({
  event: 'order.created',
  orderId: '12345',
  userId: 'usr_abc',
  amount: 99.90,
  correlationId: req.headers['x-correlation-id'],
}, 'Pedido criado com sucesso');
```

**Output JSON:**
```json
{
  "level": "info",
  "time": "2026-04-11T14:30:00.000Z",
  "event": "order.created",
  "orderId": "12345",
  "userId": "usr_abc",
  "amount": 99.90,
  "correlationId": "550e8400-e29b-41d4-a716-446655440000",
  "msg": "Pedido criado com sucesso"
}
```

#### Correlation IDs

Um Correlation ID e um identificador unico adicionado na primeira interacao (request de entrada) que acompanha o request em todos os servicos que ele percorre. Funciona como o "fio invisivel" que conecta microservicos.

Fonte: [Microsoft Engineering Playbook -- Correlation IDs](https://microsoft.github.io/code-with-engineering-playbook/observability/correlation-id/)

**Implementacao recomendada:**

```javascript
// Middleware Express para Correlation ID
import { randomUUID } from 'crypto';

function correlationMiddleware(req, res, next) {
  // Usa o ID existente ou gera um novo
  const correlationId = req.headers['x-correlation-id'] || randomUUID();
  req.correlationId = correlationId;
  res.setHeader('x-correlation-id', correlationId);
  
  // Injeta no logger de request
  req.log = logger.child({ correlationId });
  next();
}
```

**Boas praticas:**
- Gerar o Correlation ID no API Gateway ou no primeiro ponto de entrada
- Propagar via header HTTP padronizado (`X-Correlation-ID`)
- Incluir em TODOS os logs emitidos durante o processamento do request
- OpenTelemetry faz isso automaticamente via `TraceId`

Fonte: [Last9 -- Trace ID vs Correlation ID](https://last9.io/blog/correlation-id-vs-trace-id/)

### 1.2 Metrics -- Medidas Numericas de Performance

Metrics sao medidas numericas de performance e comportamento do sistema ao longo do tempo, como uso de CPU, tempo de resposta ou taxa de erros.

#### Golden Signals (Google SRE Book)

O Google SRE Book define os **Four Golden Signals** -- se voce so pode monitorar 4 metricas de um sistema user-facing, foque nestas:

| Signal | O que mede | Exemplo |
|--------|-----------|---------|
| **Latency** | Tempo para processar um request | p50=45ms, p99=200ms |
| **Traffic** | Volume de demanda no sistema | 1.200 req/s |
| **Errors** | Taxa de requests que falham | 0.5% de respostas 5xx |
| **Saturation** | Quao perto da capacidade maxima | CPU a 78%, memoria a 65% |

> Fonte: [Google SRE Book -- Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)

#### RED Method (para microservicos)

Criado por Tom Wilkie (Grafana Labs), mede como um microservico se comporta do ponto de vista do caller:

| Metrica | Descricao |
|---------|-----------|
| **Rate** | Requests por segundo |
| **Errors** | Requests que falharam por segundo |
| **Duration** | Distribuicao de tempo de resposta (histograma) |

Fonte: [Last9 -- RED Method](https://last9.io/blog/monitoring-with-red-method/)

#### USE Method (para infraestrutura)

Criado por Brendan Gregg, foca em recursos de infraestrutura:

| Metrica | Descricao |
|---------|-----------|
| **Utilization** | Porcentagem do recurso em uso |
| **Saturation** | Fila de trabalho pendente |
| **Errors** | Eventos de erro do recurso |

**Quando usar cada metodo:**

| Metodo | Melhor para | Foco |
|--------|------------|------|
| Golden Signals | Servicos user-facing | Experiencia do usuario |
| RED | Microservicos | Comportamento do servico |
| USE | Infraestrutura (CPU, disco, rede) | Saude dos recursos |

### 1.3 Traces -- Rastreamento Distribuido

Traces representam o caminho completo de um request individual atraves de um sistema distribuido, identificando gargalos, dependencias e causas raiz de problemas.

#### OpenTelemetry -- O Padrao da Industria

OpenTelemetry (OTel) emergiu como o padrao de facto para observability em 2025, unificando metricas, logs e traces em um protocolo comum (OTLP). E o segundo projeto mais ativo da CNCF, atras apenas do Kubernetes.

Fonte: [CNCF -- From Chaos to Clarity: How OpenTelemetry Unified Observability](https://www.cncf.io/blog/2025/11/27/from-chaos-to-clarity-how-opentelemetry-unified-observability-across-clouds/)

**Status em 2025-2026:**
- Traces, Metrics, Logs: **GA (Generally Available)** em todas as linguagens principais
- Profiling: GA planejado para mid-2025
- Google Cloud adotou OTLP nativamente para trace ingestion
- Grandes empresas (bancos, aerolineas) adotaram amplamente

Fonte: [The New Stack -- Observability in 2025: OpenTelemetry and AI](https://thenewstack.io/observability-in-2025-opentelemetry-and-ai-to-fill-in-gaps/)

#### Conceitos Fundamentais de Tracing

```
[Trace]
  |
  |-- [Span: API Gateway] (50ms)
  |     |
  |     |-- [Span: Auth Service] (15ms)
  |     |
  |     |-- [Span: Order Service] (30ms)
  |           |
  |           |-- [Span: Database Query] (8ms)
  |           |
  |           |-- [Span: Payment Service] (18ms)
```

- **Trace:** O ciclo de vida completo de um request
- **Span:** Uma unidade de trabalho dentro de um trace (com nome, duracao, status, atributos)
- **Context Propagation:** Mecanismo que conecta spans entre servicos (via headers HTTP)

#### Setup OpenTelemetry para Node.js

```javascript
// instrumentation.js -- Deve ser importado ANTES de qualquer outro modulo
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-http');
const { OTLPMetricExporter } = require('@opentelemetry/exporter-metrics-otlp-http');
const { PeriodicExportingMetricReader } = require('@opentelemetry/sdk-metrics');

const sdk = new NodeSDK({
  serviceName: 'meu-servico-api',
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT || 'http://localhost:4318/v1/traces',
  }),
  metricReader: new PeriodicExportingMetricReader({
    exporter: new OTLPMetricExporter({
      url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT || 'http://localhost:4318/v1/metrics',
    }),
    exportIntervalMillis: 60000,
  }),
  instrumentations: [getNodeAutoInstrumentations()],
});

sdk.start();
```

**Zero-code instrumentation** (sem modificar codigo):
```bash
node --require @opentelemetry/auto-instrumentations-node/register app.js
```

Fonte: [OpenTelemetry -- Node.js Getting Started](https://opentelemetry.io/docs/languages/js/getting-started/nodejs/)

---

## 2. Logging Stack

### 2.1 Bibliotecas de Logging para Node.js

#### Comparacao: Pino vs Winston vs Bunyan

| Criterio | Pino | Winston | Bunyan |
|----------|------|---------|--------|
| **Performance** | 5-10x mais rapido que Winston | Moderada | Moderada |
| **Downloads/semana** | ~7M | ~12M (mais popular) | ~1.5M |
| **Formato nativo** | JSON (structured) | Flexivel (JSON, text, custom) | JSON (structured) |
| **Abordagem** | Async logging, separacao de escrita/formatacao | Sincrono, flexivel | Sincrono |
| **Transports** | Via pino-pretty, pino-elasticsearch, etc. | Extensivo (file, console, HTTP, etc.) | Streams |
| **OpenTelemetry** | Suporte nativo | Via plugin | Nao |
| **Manutencao** | Ativa | Ativa | **Descontinuado** -- repo recomenda Pino |
| **Melhor para** | APIs de alta performance, microservicos | Aplicacoes com routing complexo, legacy | **Nao usar em novos projetos** |

Fonte: [Better Stack -- Best Node.js Logging Libraries](https://betterstack.com/community/guides/logging/best-nodejs-logging-libraries/), [Dash0 -- Top 5 Node.js Logging Frameworks 2025](https://www.dash0.com/faq/the-top-5-best-node-js-and-javascript-logging-frameworks-in-2025-a-complete-guide)

**Recomendacao:** Use **Pino** para projetos novos. Performance superior, structured logging nativo e suporte OpenTelemetry. Winston continua valido para projetos existentes que ja o utilizam.

#### Setup Pino com Next.js

```javascript
// lib/logger.ts
import pino from 'pino';

export const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  transport: process.env.NODE_ENV === 'development'
    ? { target: 'pino-pretty', options: { colorize: true } }
    : undefined,
  // Em producao, JSON puro para log aggregation
  formatters: {
    level: (label) => ({ level: label }),
    bindings: (bindings) => ({
      service: 'minha-app',
      env: process.env.NODE_ENV,
      pid: bindings.pid,
      hostname: bindings.hostname,
    }),
  },
  timestamp: pino.stdTimeFunctions.isoTime,
});
```

### 2.2 Plataformas de Log Aggregation

#### Comparacao Detalhada

| Criterio | Datadog Logs | Grafana Loki | ELK Stack | Axiom | BetterStack |
|----------|-------------|-------------|-----------|-------|-------------|
| **Modelo** | SaaS gerenciado | Open source / Cloud | Open source / Cloud | SaaS | SaaS |
| **Indexacao** | Full-text | Apenas labels (metadata) | Full-text (Elasticsearch) | Colunar com compressao 95%+ | Full-text |
| **Query Language** | Datadog Query | LogQL (semelhante a PromQL) | KQL / Lucene | APL (Analytics Processing Language) | SQL-like |
| **Integracao** | 800+ integracoes nativas | Grafana + Prometheus + Tempo | Kibana + Logstash/Beats | API REST, S3, Vercel | Vercel, Heroku, Docker |
| **Melhor para** | Empresas que querem tudo-em-um | Equipes que ja usam Grafana/Prometheus | Analytics complexos, busca avancada | Volume alto com custo baixo | Startups, MVPs |
| **Free tier** | 14 dias trial | 50 GB logs/mes (Cloud) | Self-hosted: gratis | 500 GB/mes ingest, 30 dias retencao | 3 GB/mes |
| **Preco estimado** | $0.10/GB ingest + $0.05/GB retencao | $0.50/GB (Cloud) | Self-hosted: infra. Elastic Cloud: $95/mo+ | $25/mo para 1TB | $29/mo por responder |
| **Vendor lock-in** | Alto | Baixo (open source) | Baixo (open source) | Moderado | Moderado |

Fontes: [Grafana -- Grafana vs Datadog](https://grafana.com/compare/grafana-vs-datadog/), [ChaosSearch -- Grafana + Loki to Replace Datadog](https://www.chaossearch.io/blog/why-organizations-use-grafana-loki-to-replace-datadog), [SigNoz -- Datadog vs Grafana 2026](https://signoz.io/blog/datadog-vs-grafana/)

**Decision framework:**

| Situacao | Recomendacao |
|----------|-------------|
| MVP/Startup com budget limitado | **Axiom** (500 GB free) ou **BetterStack** |
| Equipe tecnica, ja usa Prometheus/Grafana | **Grafana Loki** |
| Empresa que quer solucao all-in-one | **Datadog** |
| Necessidade de busca full-text complexa | **ELK Stack** |
| Volume altissimo, foco em custo | **Axiom** (compressao 95%+) |

### 2.3 Log Retention e Otimizacao de Custo

| Tipo de log | Retencao recomendada | Justificativa |
|------------|---------------------|---------------|
| Error/Fatal | 90 dias - 1 ano | Debugging, compliance |
| Warn | 30-90 dias | Analise de tendencias |
| Info (negocios) | 30-90 dias | Metricas de negocio |
| Debug | 7-14 dias | Troubleshooting ativo |
| Trace (verbose) | 1-3 dias | Apenas durante investigacao |

**Estrategias de reducao de custo:**
1. **Sampling:** Coletar 10-20% dos logs de `info` em producao; 100% de `error`/`warn`
2. **Filtragem na origem:** Descartar logs de health checks, bots, e requests estaticos antes de enviar
3. **Compressao:** Usar Axiom ou Loki (indexacao por labels = indices menores)
4. **Tiering:** Mover logs antigos para cold storage (S3, GCS) apos periodo ativo

### 2.4 Vercel Log Drains

Desde setembro 2025, Vercel renomeou "Log Drains" para **Vercel Drains**, expandindo alem de logs para incluir OpenTelemetry traces, Web Analytics events e Speed Insights metrics.

Fonte: [Vercel Blog -- Introducing Vercel Drains](https://vercel.com/blog/introducing-vercel-drains)

**Configuracao:**
1. Dashboard Vercel > Team Settings > Drains > Add Drain
2. Selecionar dados: Logs, Traces, Speed Insights, Analytics
3. Configurar destino (Datadog, Axiom, BetterStack, endpoint custom)
4. Definir sampling rate por environment
5. Formatos suportados: JSON, NDJSON, Syslog

**Disponibilidade:** Planos Pro e Enterprise apenas.

Fonte: [Vercel Docs -- Working with Drains](https://vercel.com/docs/drains)

### 2.5 Supabase Logs API

Supabase oferece logging integrado powered by **Logflare**, cobrindo todos os servicos:

- **API Gateway:** Requests, responses, status codes
- **PostgreSQL:** Queries, connections, erros
- **Storage:** Uploads, downloads, erros de acesso
- **Edge Functions:** Invocacoes, duracoes, exceptions
- **Auth:** Login attempts, token refresh, erros

**Acesso:** Dashboard > Logs ou via API REST. Suporte a OpenTelemetry para exportar telemetria para qualquer plataforma compativel (Datadog, Honeycomb, Grafana).

Fonte: [Supabase -- New Observability Features](https://supabase.com/blog/new-observability-features-in-supabase), [Supabase Docs -- Logs & Analytics](https://supabase.com/features/logs-analytics)

---

## 3. Metricas & Dashboards

### 3.1 Prometheus + Grafana -- O Padrao Open Source

**Prometheus** e um sistema de monitoramento e alerting open source que coleta metricas via scraping de endpoints HTTP e armazena em um banco de dados time-series. **Grafana** e a plataforma de visualizacao que consome dados do Prometheus.

Fonte: [Grafana Docs -- Getting Started with Prometheus](https://grafana.com/docs/grafana/latest/fundamentals/getting-started/first-dashboards/get-started-grafana-prometheus/)

**Arquitetura basica:**

```
[Aplicacao] --expoe /metrics--> [Prometheus] --queries PromQL--> [Grafana Dashboard]
     |                              |
     |                        [Alertmanager]
     |                              |
[Node Exporter]              [PagerDuty/Slack]
```

**Setup com Docker Compose:**

```yaml
# docker-compose.monitoring.yml
version: '3.8'
services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-data:/var/lib/grafana

  node-exporter:
    image: prom/node-exporter:latest
    ports:
      - "9100:9100"

volumes:
  grafana-data:
```

Fonte: [Dev.to -- Monitoring Stack with Prometheus and Grafana](https://dev.to/durrello/building-a-complete-monitoring-stack-with-prometheus-and-grafana-using-docker-36h8)

### 3.2 Datadog -- Plataforma All-in-One

Datadog e a solucao SaaS mais adotada para observability enterprise, oferecendo metricas, logs, traces, APM e seguranca em uma plataforma unificada.

**Modelo de pricing (2025-2026):**

| Produto | Preco/host/mes (anual) | Preco/host/mes (on-demand) |
|---------|----------------------|---------------------------|
| Infrastructure (Pro) | $15 | $18 |
| Infrastructure (Enterprise) | $23 | $27 |
| APM | $31 | $36 |
| APM Pro | $36 | $41 |
| APM Enterprise | $40 | $47 |
| Log Management | $0.10/GB ingest | + $0.05/GB retencao |
| RUM | $1.50/1000 sessions | - |

**Billing model:** High-water mark -- mede o numero de hosts a cada hora, descarta o top 1% das horas com maior uso, e cobra pelo proximo maior valor.

Fonte: [Datadog Pricing](https://www.datadoghq.com/pricing/), [Finout -- Datadog Pricing Breakdown 2025](https://www.finout.io/blog/datadog-pricing-explained), [OpenObserve -- Datadog Pricing Hidden Costs](https://openobserve.ai/blog/datadog-pricing/)

**Alerta sobre custos:** Datadog pode se tornar extremamente caro conforme volume de dados e hosts crescem. E comum startups reportarem contas de $10K-50K/mes apos escalar. Monitorar custos e definir budgets e essencial.

### 3.3 Vercel Analytics & Speed Insights

**Vercel Speed Insights** monitora Core Web Vitals com dados de usuarios reais (Real User Monitoring):

| Metrica | O que mede | Meta "Good" |
|---------|-----------|-------------|
| **LCP** (Largest Contentful Paint) | Velocidade de carregamento | < 2.5s |
| **INP** (Interaction to Next Paint) | Responsividade | < 200ms |
| **CLS** (Cumulative Layout Shift) | Estabilidade visual | < 0.1 |
| **FCP** (First Contentful Paint) | Primeiro conteudo visivel | < 1.8s |
| **TTFB** (Time to First Byte) | Tempo ate primeiro byte | < 800ms |

Fonte: [Vercel Docs -- Speed Insights Metrics](https://vercel.com/docs/speed-insights/metrics)

**Setup:**

```bash
npm install @vercel/speed-insights
```

```tsx
// app/layout.tsx
import { SpeedInsights } from '@vercel/speed-insights/next';

export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        {children}
        <SpeedInsights />
      </body>
    </html>
  );
}
```

**Vercel Analytics** (separado do Speed Insights) fornece dados de trafego: page views, visitantes unicos, referrers, paises -- sem cookies, privacy-friendly.

### 3.4 Supabase Dashboard -- Monitoring Integrado

Supabase oferece dashboards nativos para cada servico:

| Servico | Metricas disponiveis |
|---------|---------------------|
| **Database** | Conexoes ativas, queries/s, latencia, tamanho do banco, cache hit ratio |
| **Auth** | Signups, logins, erros de autenticacao |
| **Storage** | Bandwidth, objects, tamanho total |
| **Edge Functions** | Invocacoes, duracoes, erros, distribuicao regional |
| **Realtime** | Conexoes WebSocket, mensagens/s |

**API de Metricas:** `/v1/projects/{ref}/metrics` retorna metricas em formato Prometheus/OpenMetrics, permitindo integracao com Grafana ou qualquer sistema OTel-compativel.

Fonte: [Supabase Docs -- Metrics API](https://supabase.com/docs/guides/telemetry/metrics)

### 3.5 Custom Metrics e KPIs de Negocio

Alem de metricas tecnicas, e critico monitorar KPIs de negocio que refletem impacto real:

| Categoria | Exemplos de metricas |
|-----------|---------------------|
| **Revenue** | MRR, conversoes/hora, carrinho abandonado |
| **Engagement** | DAU/MAU, tempo de sessao, feature adoption |
| **Reliability** | Uptime, error rate percebida pelo usuario |
| **Efficiency** | Custo por request, custo por usuario ativo |

**Implementacao com Prometheus custom metrics:**

```javascript
import { Counter, Histogram } from 'prom-client';

// Metrica de negocio: pedidos criados
const ordersCreated = new Counter({
  name: 'business_orders_created_total',
  help: 'Total de pedidos criados',
  labelNames: ['plan', 'source'],
});

// Metrica de negocio: tempo de checkout
const checkoutDuration = new Histogram({
  name: 'business_checkout_duration_seconds',
  help: 'Tempo do fluxo de checkout',
  buckets: [1, 2, 5, 10, 30, 60],
});

// Uso
ordersCreated.inc({ plan: 'pro', source: 'web' });
const end = checkoutDuration.startTimer();
await processCheckout();
end();
```

### 3.6 SLIs, SLOs, SLAs -- Definindo e Monitorando Confiabilidade

#### Definicoes

| Conceito | O que e | Exemplo |
|----------|---------|---------|
| **SLI** (Service Level Indicator) | Metrica que voce mede. Geralmente uma proporcao: eventos bons / eventos totais | 99.7% dos requests completam em <200ms |
| **SLO** (Service Level Objective) | Meta interna que a engenharia se compromete a atingir | 99.9% disponibilidade em janelas de 30 dias |
| **SLA** (Service Level Agreement) | Contrato externo com o cliente, com penalidades se violado | 99.5% uptime, com creditos se violado |
| **Error Budget** | Margem de indisponibilidade permitida pelo SLO | SLO 99.9% = 0.1% error budget = ~43 min/mes |

Fonte: [incident.io -- SLO, SLA, SLI Guide](https://incident.io/blog/slo-sla-sli), [Rootly -- SLA vs SLO vs SLI](https://rootly.com/blog/sla-vs-slo-vs-sli-the-full-breakdown-for-reliable-systems)

#### Error Budget -- Como Funciona

```
Error Budget = 1 - SLO

Exemplo: SLO = 99.9% availability
  Error Budget = 0.1% = 43.2 minutos/mes

Se voce consumiu 30 minutos de downtime:
  Budget restante = 13.2 minutos
  Status: Pode fazer deploy (budget OK)

Se voce consumiu 50 minutos de downtime:
  Budget restante = -6.8 minutos (ESTOUROU)
  Status: Freeze de deploys ate o budget resetar
```

Fonte: [Nobl9 -- Complete Guide to Error Budgets](https://www.nobl9.com/resources/a-complete-guide-to-error-budgets-setting-up-slos-slis-and-slas-to-maintain-reliability)

#### Tabela de Noves

| Disponibilidade | Downtime/ano | Downtime/mes | Downtime/semana |
|-----------------|-------------|-------------|-----------------|
| 99% (dois noves) | 3.65 dias | 7.3 horas | 1.68 horas |
| 99.9% (tres noves) | 8.76 horas | 43.2 min | 10.08 min |
| 99.95% | 4.38 horas | 21.6 min | 5.04 min |
| 99.99% (quatro noves) | 52.6 min | 4.32 min | 1.01 min |
| 99.999% (cinco noves) | 5.26 min | 25.9 seg | 6.05 seg |

**Recomendacao por tipo de servico:**

| Tipo | SLO recomendado | Justificativa |
|------|----------------|---------------|
| MVP / side project | 99% | Foco em velocidade de desenvolvimento |
| SaaS em crescimento | 99.9% | Equilibrio entre agilidade e confiabilidade |
| Financeiro / saude | 99.99% | Regulacao e impacto critico |
| Infraestrutura (DNS, CDN) | 99.99%+ | Dependencia de outros servicos |

---

## 4. Application Performance Monitoring (APM)

### 4.1 Datadog APM vs New Relic vs Sentry

| Criterio | Sentry | Datadog APM | New Relic |
|----------|--------|------------|-----------|
| **Foco principal** | Error tracking + performance | Full-stack APM + infra | Full-stack APM |
| **Melhor para** | Capturar e diagnosticar erros de codigo | Visao unificada de infra + apps + seguranca | APM profundo com pricing flexivel |
| **Error tracking** | Excelente (foco primario) | Bom (parte da plataforma) | Bom |
| **Distributed tracing** | Basico | Excelente | Excelente |
| **Infra monitoring** | Nao | Sim (nativo) | Sim (nativo) |
| **Session Replay** | Sim | Sim (RUM) | Sim |
| **Source maps** | Excelente | Bom | Bom |
| **Free tier** | 5.000 errors/mes, 1 usuario | 14 dias trial | 100 GB/mes, 1 usuario |
| **Preco inicial** | $26/mes (Team) | $31/host/mes | $0.40/GB apos free tier |
| **Integracao Next.js** | Excelente (SDK dedicado) | Boa | Boa |
| **Setup complexity** | Wizard automatico | Agente + config | Agente + config |

Fonte: [Better Stack -- Datadog vs Sentry 2026](https://betterstack.com/community/comparisons/datadog-vs-sentry/), [Apptension -- Sentry vs Datadog vs New Relic](https://apptension.com/guides/best-saas-error-monitoring-and-observability-tools-sentry-vs-datadog-vs-new-relic), [SigNoz -- Datadog vs Sentry 2026](https://signoz.io/comparisons/datadog-vs-sentry/)

**Decision framework:**

| Seu problema principal | Ferramenta recomendada |
|-----------------------|----------------------|
| "Erros em producao sem contexto" | **Sentry** primeiro |
| "Latencia alta, nao sei onde" | **Datadog APM** ou **New Relic** primeiro |
| "Preciso ver tudo: infra + app + logs" | **Datadog** (all-in-one) |
| "Budget limitado, preciso comecar" | **Sentry free** + **New Relic free** |

### 4.2 Sentry -- Setup Detalhado para Next.js

**Instalacao:**

```bash
npx @sentry/wizard@latest -i nextjs
```

O wizard cria automaticamente:
- `sentry.client.config.ts` -- Configuracao client-side
- `sentry.server.config.ts` -- Configuracao server-side
- `sentry.edge.config.ts` -- Configuracao edge runtime
- `next.config.js` atualizado com `withSentryConfig`

Fonte: [Sentry Docs -- Next.js](https://docs.sentry.io/platforms/javascript/guides/nextjs/)

**Configuracao recomendada:**

```typescript
// sentry.client.config.ts
import * as Sentry from '@sentry/nextjs';

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  environment: process.env.NODE_ENV,
  
  // Sampling
  tracesSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,
  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0,
  
  // Filtros
  ignoreErrors: [
    'ResizeObserver loop limit exceeded',
    'Non-Error promise rejection captured',
  ],
  
  // Release tracking
  release: process.env.SENTRY_RELEASE || process.env.VERCEL_GIT_COMMIT_SHA,
  
  integrations: [
    Sentry.replayIntegration(),
    Sentry.browserTracingIntegration(),
  ],
});
```

**Source Maps:** Sentry espera que source maps sejam enviados ANTES de erros ocorrerem. Se voce enviar apos um erro, ele NAO sera retroativamente mapeado.

Fonte: [Sentry Blog -- Setting Up Next.js Source Maps](https://blog.sentry.io/setting-up-next-js-source-maps-sentry/)

**Tunnel Route (recomendado):** Rotear requests do Sentry via rewrite do Next.js evita bloqueio por ad blockers:

```javascript
// next.config.js
const sentryConfig = {
  tunnelRoute: '/monitoring',
  widenClientFileUpload: true,
  disableLogger: true,
};
```

### 4.3 RUM vs Synthetic Monitoring

| Aspecto | Real User Monitoring (RUM) | Synthetic Monitoring |
|---------|--------------------------|---------------------|
| **Dados de** | Usuarios reais, dispositivos reais | Testes automatizados em "laboratorio" |
| **Quando monitora** | Apenas quando ha trafego real | 24/7, sob demanda ou schedule |
| **Cobertura** | Diversidade real (devices, redes, geolocalizacoes) | Controlada (locations e devices pre-definidos) |
| **Deteccao pre-producao** | Nao | Sim -- pega problemas antes dos usuarios |
| **Impacto de negocio** | Alto -- mostra experiencia real | Baixo -- testes artificiais |
| **Custo** | Baseado em sessoes/events | Baseado em checks/min |
| **Exemplos** | Vercel Speed Insights, Datadog RUM, Sentry | Datadog Synthetic, Checkly, Pingdom |
| **Melhor para** | Tendencias de longo prazo, segmentacao por publico | Regression testing, SLA monitoring, pre-deploy |

Fonte: [MDN -- RUM vs Synthetic Monitoring](https://developer.mozilla.org/en-US/docs/Web/Performance/Guides/Rum-vs-Synthetic), [ip-label -- Synthetic vs RUM 2025](https://ip-label.com/synthetic-monitoring-vs-real-user-monitoring/)

**Best practice:** Combine ambos -- synthetics para confiabilidade proativa, RUM para verdade de campo e impacto de negocio.

### 4.4 Performance Profiling em Producao

Profiling em producao identifica gargalos de CPU, memoria e I/O em codigo real com trafego real.

**Ferramentas:**
- **Sentry Profiling:** Integrado ao error tracking, mostra flamegraphs por transacao
- **Datadog Continuous Profiler:** Incluso no APM, suporta Node.js, Python, Go, Java
- **Pyroscope (Grafana):** Open source, continuous profiling integrado ao Grafana
- **Node.js built-in:** `--prof` flag ou `node --inspect` + Chrome DevTools

**Cuidado:** Profiling continuo adiciona overhead (tipicamente 1-5% de CPU). Use sampling em producao.

---

## 5. Alerting

### 5.1 Alert Fatigue -- O Maior Inimigo

Alert fatigue ocorre quando engenheiros recebem tantos alertas que param de prestar atencao. Alertas criticos se perdem no ruido.

**Dados de 2025:**
- Engenheiro on-call medio recebe ~50 alertas/semana
- Apenas 2-5% requerem intervencao humana
- AI-powered alert management pode reduzir ruido em 90%+

Fonte: [incident.io -- 2025 Guide to Preventing Alert Fatigue](https://incident.io/blog/2025-guide-to-preventing-alert-fatigue-for-modern-on-call-teams), [OneUptime -- Alert Fatigue](https://oneuptime.com/blog/post/2026-03-05-alert-fatigue-ai-on-call/view)

**Regras para alertas saudaveis:**

1. **Actionable:** Se o alerta dispara e ninguem pode tomar uma acao, ele NAO deveria existir
2. **Contexto:** Cada alerta deve linkar a um runbook com passos de resolucao
3. **Agrupamento:** Agrupar alertas relacionados em um incidente (ex: 33 alertas -> 1 incidente "DB connectivity issue")
4. **Dynamic thresholds:** Usar baselines adaptivos em vez de thresholds estaticos
5. **Revisao regular:** Podar alertas ruidosos mensalmente
6. **Ownership claro:** Cada servico tem um dono definido, cada tipo de alerta tem um path de escalacao

### 5.2 Ferramentas de Alerting e Incident Management

| Criterio | PagerDuty | Opsgenie | Grafana OnCall / IRM |
|----------|-----------|----------|---------------------|
| **Status (2025)** | Ativo, lider de mercado | **Descontinuado** -- venda encerra Jun/2025, suporte ate Abr/2027 | OnCall OSS em maintenance mode; substituido por **Grafana Cloud IRM** |
| **Preco** | Free ate 5 usuarios, depois $21-59/user/mes | N/A (descontinuado) | $19/mes plataforma + $20/user/mes (Cloud IRM) |
| **Integracoes** | 700+ (maior ecossistema) | Forte com Jira/Atlassian | Nativo com Grafana/Prometheus/Loki |
| **Automacao** | Incident Workflows (no-code) | Basic | Nativo com Grafana stack |
| **Melhor para** | Empresas grandes, muitas integracoes | **Migrar para outra solucao** | Equipes que ja usam Grafana |

Fonte: [TechnologyMatch -- PagerDuty vs Opsgenie vs Grafana OnCall](https://technologymatch.com/blog/pagerduty-vs-opsgenie-vs-grafana-oncall-which-incident-management-tool-is-right-for-your-team), [OneUptime -- Opsgenie Alternatives](https://oneuptime.com/blog/post/2026-02-21-10-best-opsgenie-alternatives/view)

**ALERTA IMPORTANTE:** Opsgenie sera descontinuado. Se voce o utiliza, planeje migracao ate abril de 2027. Alternativas: PagerDuty, Grafana Cloud IRM, incident.io, BetterStack.

**Alternativas modernas emergentes:**
- **incident.io** -- Foco em incident management + on-call, UX moderna
- **BetterStack** -- On-call + uptime + logs em um pacote
- **Rootly** -- AI-powered incident management

### 5.3 Escalation Policies

Uma policy de escalacao bem definida garante que alertas nao ignorados cheguem a quem pode resolver:

```
Nivel 1 (0 min):     Engenheiro on-call primario
Nivel 2 (15 min):    Engenheiro on-call secundario (backup)
Nivel 3 (30 min):    Tech Lead do time
Nivel 4 (60 min):    Engineering Manager
Nivel 5 (120 min):   VP of Engineering / CTO
```

**Boas praticas de on-call:**
- Rotacao de 1 semana (mais longo cria estresse; mais curto cria caos de planejamento)
- Compensacao justa para quem esta on-call
- Post-incident review (blameless) para aprender
- Dias de folga apos periodos de on-call intenso

### 5.4 Runbooks e Automacao de Resposta a Incidentes

Um runbook e mais que documentacao -- e um guia tatico que conduz responders pelo tratamento de um incidente especifico, do trigger a resolucao.

Fonte: [Rootly -- Incident Response Runbooks](https://rootly.com/incident-response/runbooks)

**Template de Runbook:**

```markdown
# Runbook: API Response Time > 2s

## Severity: P2
## Owner: Backend Team
## Last Updated: 2026-04-01

### Sintomas
- Alerta: `api_latency_p99 > 2000ms` por 5 minutos
- Usuarios reportando lentidao

### Diagnostico
1. Verificar Grafana dashboard: [link]
2. Checar pg_stat_activity: `SELECT * FROM pg_stat_activity WHERE state = 'active';`
3. Verificar CPU/memoria do host: `top` ou dashboard de infra
4. Checar connection pool: pgBouncer stats

### Mitigacao
1. Se query lenta: Identificar query com `pg_stat_statements`, criar index
2. Se CPU saturada: Escalar horizontalmente (adicionar replicas)
3. Se connection pool esgotado: Aumentar `max_connections` no pgBouncer
4. Se deploy recente causou: Rollback via `vercel rollback`

### Resolucao
- Documentar causa raiz no post-mortem
- Criar ticket para fix permanente
```

**Tendencia 2025-2026:** Runbooks estao evoluindo de documentos estaticos para **Intelligent Runbooks** com triggers automaticos e AI agents que interpretam sinais, entendem dependencias e agem adaptativamente.

Fonte: [Harness -- Runbook Automation](https://developer.harness.io/docs/ai-sre/runbooks/), [ilert -- Runbooks are History](https://www.ilert.com/blog/runbooks-are-history)

---

## 6. Infrastructure Monitoring

### 6.1 Server Metrics (CPU, Memoria, Disco, Rede)

As metricas basicas de servidor seguem o USE Method:

| Recurso | Utilizacao | Saturacao | Erros |
|---------|-----------|-----------|-------|
| **CPU** | % de uso por core | Load average, run queue | Interrupts de hardware |
| **Memoria** | % RAM usada, swap | OOM kills, page faults | ECC errors |
| **Disco** | % de espaco usado, IOPS | I/O wait, queue depth | Read/write errors |
| **Rede** | Bandwidth utilizada | Packet drops, retransmits | Interface errors |

**Ferramentas:**
- **Node Exporter** (Prometheus): Expoe metricas de host Linux
- **Windows Exporter**: Equivalente para Windows
- **Datadog Agent**: Coleta automaticamente com 800+ integracoes

### 6.2 Container Monitoring (Docker, Kubernetes)

| Ferramenta | Tipo | Melhor para |
|-----------|------|-------------|
| **Prometheus + Grafana** | Open source | Equipes tecnicas, Kubernetes nativo |
| **Datadog** | SaaS | Enterprise, all-in-one, 900+ integracoes |
| **Dynatrace** | SaaS | AI-powered (Davis AI), auto-discovery |
| **Sysdig** | SaaS/OSS | Seguranca + monitoring de containers |
| **Lens** | Desktop | Kubernetes IDE para devs, cluster management |
| **SigNoz** | OSS | OpenTelemetry-nativo, self-hosted option |

Fonte: [KloudFuse -- Kubernetes Monitoring Tools 2025](https://www.kloudfuse.com/blog/kubernetes-monitoring-tools-in-2025-our-top-10-picks), [Last9 -- Container Monitoring Tools](https://last9.io/blog/best-container-monitoring-tools/)

**Para Kubernetes, o padrao e kube-prometheus-stack (Helm chart):**

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack
```

Inclui: Prometheus, Alertmanager, Grafana, Node Exporter, kube-state-metrics.

### 6.3 Database Monitoring (PostgreSQL)

PostgreSQL oferece views de sistema poderosas para monitoramento:

#### pg_stat_activity -- Sessoes Ativas

```sql
-- Queries ativas em execucao
SELECT pid, now() - pg_stat_activity.query_start AS duration,
       query, state, wait_event_type, wait_event
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.query_start) > interval '5 seconds'
  AND state = 'active'
ORDER BY duration DESC;
```

#### pg_stat_statements -- Performance Historica

```sql
-- Top 10 queries por tempo total
SELECT query,
       calls,
       total_exec_time / 1000 AS total_seconds,
       mean_exec_time AS avg_ms,
       rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

#### Connection Pool (PgBouncer)

```sql
-- Metricas do pool (via PgBouncer admin console)
SHOW POOLS;
-- Mostra: database, user, active, waiting, server_active, server_idle, pool_mode
```

Fonte: [Instaclustr -- Mastering pg_stat_activity](https://www.instaclustr.com/blog/mastering-pg-stat-activity-for-real-time-monitoring-in-postgresql/), [Uptrace -- PostgreSQL Monitoring Tools](https://uptrace.dev/tools/postgresql-monitoring-tools)

**Metricas criticas de PostgreSQL para alertar:**

| Metrica | Threshold de alerta | Impacto |
|---------|---------------------|---------|
| Conexoes ativas / max_connections | > 80% | Novas conexoes rejeitadas |
| Cache hit ratio | < 95% | Queries lendo do disco |
| Replication lag | > 10s | Replicas inconsistentes |
| Dead tuples ratio | > 20% | VACUUM necessario |
| Long-running queries | > 30s | Locks, bloqueios |
| Transaction wraparound age | > 1B | Risco de data loss |

**Ferramentas especializadas:**
- **pganalyze:** Coleta automaticamente pg_stat_statements, identifica queries lentas, indexes faltantes
- **Supabase Dashboard:** Metricas nativas para projetos Supabase
- **Datadog PostgreSQL integration:** Dashboards pre-configurados

### 6.4 CDN e Edge Monitoring

| Metrica | Descricao | Meta |
|---------|-----------|------|
| Cache hit ratio | % de requests servidos do cache | > 90% |
| Origin request rate | Requests que vao ao servidor de origem | Minimizar |
| Bandwidth saved | Volume de dados servidos pelo cache | Maximizar |
| TTFB por regiao | Tempo ate primeiro byte por geolocalizacao | < 100ms |
| Error rate (4xx, 5xx) | Erros por edge location | < 0.1% |

**Monitoramento combinado:** RUM + synthetic tests (Lighthouse, WebPageTest) + CDN analytics para monitorar cache hit ratio, bandwidth saved e origin request reduction.

Fonte: [Dev.to -- Cloudflare vs Vercel vs Netlify Edge Performance 2026](https://dev.to/dataformathub/cloudflare-vs-vercel-vs-netlify-the-truth-about-edge-performance-2026-50h0)

---

## 7. Observability para Serverless

### 7.1 Desafios Unicos do Serverless

Serverless simplifica o desenvolvimento mas introduz desafios novos para observability:

| Desafio | Causa | Impacto |
|---------|-------|---------|
| **Efemeralidade** | Funcoes existem apenas durante a execucao | Telemetria deve ser exportada rapidamente |
| **Cold starts** | Inicializacao de novos containers | Latencia imprevisivel |
| **Distribuicao** | Centenas de funcoes independentes | Dificil rastrear requests end-to-end |
| **Context propagation** | Cada invocacao e isolada | Trace continuity requer ferramentas especializadas |
| **No server access** | Nao ha servidor para instalar agentes | Instrumentacao via SDK/layers |
| **Flushing** | Funcao pode terminar antes de exportar telemetria | Dados perdidos |

Fonte: [MirakiTech -- Serverless Observability Tools 2025](https://mirakitech.com/en-us/blogs/serverless-observability-and-monitoring-tools-in-2025), [Grafana -- Lambda OpenTelemetry Guide](https://grafana.com/blog/2025/04/15/aws-lambda-opentelemetry-and-grafana-cloud-a-guide-to-serverless-observability-considerations/)

### 7.2 Cold Start Tracking e Metricas de Funcoes

| Metrica | O que medir | Por que importa |
|---------|-------------|-----------------|
| **Cold start duration** | Tempo de inicializacao do container | Impacta UX na primeira request |
| **Function duration** | Tempo de execucao total (incluindo cold start) | Custo e performance |
| **Memory usage** | Pico de memoria vs alocada | Over-provisioning = desperdicio; under = OOM |
| **Init duration** | Tempo de import dos modulos | Otimizar imports reduz cold starts |
| **Concurrency** | Invocacoes simultaneas | Pode causar throttling |

**Reducao de cold starts (Vercel 2025):**
- **Fluid Compute:** Bytecode caching e predictive instance warming
- Edge Functions: Sub-50ms cold starts (vs 100-300ms de serverless tradicional)

Fonte: [SparkCo -- Vercel vs Cloudflare Edge Deployment](https://sparkco.ai/blog/vercel-vs-cloudflare-edge-deployment-deep-dive)

### 7.3 Vercel + Datadog Integration

A integracao Vercel-Datadog (via Vercel Drains, antigo Log Drains) fornece:

1. **Log Collection:** Logs de runtime, build e static via Vercel Drains
2. **Traces:** OpenTelemetry traces direto para Datadog APM, sem instrumentacao custom
3. **Performance Monitoring:** Visualizar performance de funcoes Vercel com code-level insights
4. **Metrics from Logs:** Metricas geradas a partir de logs sao armazenadas por 15 meses sem custo adicional

Fonte: [Datadog Docs -- Vercel Integration](https://docs.datadoghq.com/integrations/vercel/), [Vercel Marketplace -- Datadog](https://vercel.com/marketplace/datadog)

**Setup:**
1. Instalar integracao via [Vercel Marketplace](https://vercel.com/marketplace/datadog)
2. Conectar conta Datadog (API key + Application key)
3. Selecionar projetos para monitorar
4. Dados comecam a fluir automaticamente

### 7.4 Supabase Edge Functions Monitoring

Edge Functions do Supabase emitem logs e metricas acessiveis no dashboard:

- **Invocacoes:** Request/response data, headers, body, status codes, duracao
- **Filtros:** Por data, hora, status code, nivel de log
- **Logs:** Platform events, uncaught exceptions, mensagens de log customizadas
- **Reports:** Performance de funcoes, padroes de execucao, distribuicao regional

**Export OpenTelemetry:** Supabase suporta export de telemetria para qualquer ferramenta OTel-compativel.

Fonte: [Supabase Docs -- Edge Functions Logging](https://supabase.com/docs/guides/functions/logging)

---

## 8. Observability Maturity Model

### Modelo de Maturidade em 5 Niveis

| Nivel | Nome | Descricao | Ferramentas | Custo estimado/mes |
|-------|------|-----------|-------------|-------------------|
| **0** | Ausente | Sem monitoramento. Issues descobertos por usuarios reportando | Nenhuma | $0 |
| **1** | Basico | Error tracking + uptime checks. Reativo -- so sabe quando quebra | Sentry free + UptimeRobot | $0-10 |
| **2** | Padrao | Logs centralizados + metricas basicas + alertas simples. Pode diagnosticar problemas | Sentry + Axiom/Loki + Grafana + Vercel Analytics | $25-100 |
| **3** | Avancado | Distributed tracing + APM + SLOs definidos. Correlacao entre sinais | Sentry + Datadog/New Relic + OpenTelemetry + PagerDuty | $200-1.000 |
| **4** | Elite | Full observability + chaos engineering + auto-remediation. Proativo e preditivo | Stack completa + Chaos tools + AI-driven alerting | $1.000-5.000+ |

Fonte: [Middleware -- Observability Maturity Model](https://middleware.io/blog/observability-maturity-model/), [AWS -- Observability Maturity Model](https://aws-observability.github.io/observability-best-practices/guides/observability-maturity-model/)

### Quando Subir de Nivel

| Trigger para upgrade | De | Para |
|---------------------|-----|------|
| Primeiro usuario pagante | 0 | 1 |
| Revenue > $1K MRR ou 100+ usuarios ativos | 1 | 2 |
| Equipe > 3 devs ou microservicos | 2 | 3 |
| Revenue > $50K MRR ou SLA contratual | 3 | 4 |
| Operacao critica (financeiro, saude) | 3 | 4 |

### Detalhamento por Nivel

#### Nivel 0: Ausente (MVP)

- Nenhuma ferramenta de monitoring
- Erros descobertos por usuarios ("ta fora do ar!")
- Debugging via `console.log` em producao
- **Risco:** Alto. Problemas podem persistir por horas/dias sem deteccao

#### Nivel 1: Basico (Primeiros usuarios)

**Setup minimo:**

```
Sentry (free) -----> Error tracking + alertas por email
UptimeRobot (free) -> Ping check a cada 5 min
Vercel Analytics ---> Core Web Vitals basicos
```

**O que voce ganha:**
- Saber quando erros ocorrem em producao
- Saber quando o site cai
- Web Vitals basicos

**O que falta:**
- Sem logs centralizados (so no Vercel dashboard)
- Sem metricas custom
- Sem tracing

#### Nivel 2: Padrao (Crescimento)

**Stack recomendada:**

```
Sentry (Team $26/mo) ------> Errors + Performance
Axiom (free 500GB) ---------> Logs centralizados
Grafana Cloud (free) -------> Dashboards + metricas
Vercel Speed Insights ------> Core Web Vitals RUM
Checkly ou BetterUptime ----> Synthetic monitoring
```

**O que voce ganha:**
- Logs pesquisaveis de todos os servicos
- Dashboards com metricas de negocio e tecnicas
- Alertas que fazem sentido
- Capacidade de investigar incidentes

#### Nivel 3: Avancado (Escala)

**Stack recomendada:**

```
Sentry (Business) -----------> Errors + Session Replay + Profiling
Datadog ou New Relic ---------> APM + Distributed Tracing + Infra
OpenTelemetry ----------------> Instrumentacao padronizada
PagerDuty ou Grafana IRM -----> On-call + escalation
Grafana + Prometheus ---------> Custom dashboards + SLO tracking
```

**O que voce ganha:**
- Tracing end-to-end de requests
- SLOs definidos e monitorados com error budgets
- Correlacao automatica entre logs, metricas e traces
- On-call rotation profissional
- Performance profiling em producao

#### Nivel 4: Elite (Operacao critica)

**Adiciona ao Nivel 3:**

```
Chaos Engineering (Gremlin, LitmusChaos) -> Validacao proativa de resiliencia
AI-powered anomaly detection ------------> Deteccao antecipada de problemas
Auto-remediation ------------------------> Self-healing automatico
Business observability ------------------> Metricas de negocio correlacionadas
```

**O que voce ganha:**
- Sistemas que se auto-reparam (restart servicos, rollback deploys)
- Deteccao preditiva antes que usuarios sejam impactados
- Validacao continua de resiliencia via chaos experiments
- Observability como vantagem competitiva

Fonte: [DevOps Institute -- Chaos Engineering Observability](https://www.devopsinstitute.com/the-practice-of-chaos-engineering-observability/), [Dev.to -- Autonomous SRE](https://dev.to/vaib/autonomous-sre-revolutionizing-reliability-with-ai-automation-and-chaos-engineering-5c7g)

---

## 9. Otimizacao de Custos

### 9.1 Free Tier de Cada Ferramenta

| Ferramenta | Free Tier | Limite principal | Melhor para |
|-----------|-----------|-----------------|-------------|
| **Sentry** | 5.000 errors/mes, 1 usuario | Compartilhado entre errors e transactions | Error tracking MVP |
| **New Relic** | 100 GB/mes ingest, 1 usuario | Data ingest para de funcionar ao exceder | APM + metricas MVP |
| **Grafana Cloud** | 10K metricas, 50 GB logs, 50 GB traces | 14 dias retencao, 3 usuarios | Dashboards + metricas |
| **Axiom** | 500 GB/mes ingest, 30 dias retencao | - | Logs em volume |
| **BetterStack** | 3 GB logs, 100K exceptions, 10 monitors | Inclui incident management | All-in-one MVP |
| **UptimeRobot** | 50 monitors, checks a cada 5 min | Sem SSL monitoring no free | Uptime basico |
| **Checkly** | 5 checks, 1 regiao | API e browser checks | Synthetic monitoring |
| **Vercel Analytics** | Incluso no plano Hobby | Limitado em dados historicos | Web Vitals |
| **Supabase** | Logs no dashboard | 1 dia retencao no free | Dev/staging |

Fonte: [Sentry Pricing](https://sentry.io/pricing/), [New Relic Free Tier](https://newrelic.com/pricing/free-tier), [Grafana Pricing](https://grafana.com/pricing/), [Axiom Pricing](https://axiom.co/pricing)

### 9.2 Open Source vs Managed Service

| Aspecto | Open Source (self-hosted) | Managed Service (SaaS) |
|---------|------------------------|----------------------|
| **Custo direto** | Infra (servers, storage) | Subscription + data volume |
| **Custo oculto** | Manutencao, upgrades, on-call para o monitoring | Nenhum |
| **Escalabilidade** | Voce gerencia | Automatica |
| **Controle** | Total | Limitado ao que o vendor oferece |
| **Vendor lock-in** | Zero | Medio a alto |
| **Time-to-value** | Semanas | Horas |
| **Melhor para** | Equipes grandes com SRE dedicado | Equipes pequenas-medias, startups |

**Regra pratica:** Se voce tem < 5 engenheiros, use managed services. O custo de manter infra de monitoring e maior que o subscription.

### 9.3 Estrategias de Reducao de Custo

1. **Sampling inteligente:**
   - Tail-based sampling: Reter 5-10% dos traces no baseline, 100% para errors/slow requests
   - Log sampling: 100% de error/warn, 10-20% de info em producao

2. **Filtragem na origem:**
   - Descartar logs de health checks (`/health`, `/ready`)
   - Filtrar requests de bots/crawlers
   - Remover campos desnecessarios antes de enviar

3. **Retencao tiered:**
   - Hot: 7-14 dias (busca rapida, custo alto)
   - Warm: 30-90 dias (busca moderada, custo medio)
   - Cold: 1+ ano (S3/GCS, custo minimo, acesso lento)

4. **Cardinalidade controlada:**
   - Evitar labels de alta cardinalidade em metricas (ex: user_id como label)
   - Agregar metricas antes de enviar ao backend

5. **Adaptive Telemetry (Grafana):**
   - Classificacao inteligente de telemetria por valor
   - Recomendacoes automaticas para agregar, amostrar ou descartar dados de baixo valor

Fonte: [Mezmo -- Observability Cost Reduction Guide](https://www.mezmo.com/learn-observability/observability-cost-reduction-a-practical-guide), [Grafana -- Managing Observability Costs at Scale](https://grafana.com/blog/managing-observability-costs-at-scale-a-look-at-the-latest-cost-management-features-in-grafana-cloud/)

### 9.4 Stack Recomendada por Budget

#### Budget: $0/mes (Free Tier Only)

```
Error tracking:     Sentry free (5K errors/mo)
Logs:               Axiom free (500 GB/mo) ou Vercel dashboard
Uptime:             UptimeRobot free (50 monitors)
Web Vitals:         Vercel Speed Insights (incluso)
Dashboards:         Grafana Cloud free (10K metricas)

Total: $0/mes
Nivel de maturidade: 1 (Basico)
```

**Limitacoes:** Sem tracing, sem APM, retencao curta, sem on-call.

#### Budget: ~$50/mes

```
Error tracking:     Sentry Team ($26/mo)
Logs:               Axiom free (500 GB/mo)
Uptime:             BetterStack free + monitoring
Web Vitals:         Vercel Speed Insights
Dashboards:         Grafana Cloud free
Synthetic:          Checkly free (5 checks)

Total: ~$30-50/mes
Nivel de maturidade: 2 (Padrao)
```

**O que ganha sobre $0:** Mais erros tracked, source maps, team features, alertas melhores.

#### Budget: ~$200/mes

```
Error tracking:     Sentry Team ($26/mo)
APM + Logs:         New Relic ($0.40/GB apos 100GB free)
                    OU Datadog ($15/host/mo x 2-3 hosts)
Uptime + On-call:   BetterStack ($29/mo)
Web Vitals:         Vercel Speed Insights
Dashboards:         Grafana Cloud Pro ($29/mo)
Synthetic:          Checkly Starter ($30/mo)

Total: ~$150-250/mes
Nivel de maturidade: 2-3 (Padrao a Avancado)
```

**O que ganha:** APM com tracing, logs com retencao longa, on-call profissional.

#### Budget: $500+/mes

```
Error tracking:     Sentry Business ($80/mo)
APM + Infra:        Datadog Pro ($15-31/host/mo x 5-10 hosts)
                    OU New Relic Pro
Logs:               Incluso no Datadog ou Grafana Cloud
On-call:            PagerDuty Professional ($21-41/user/mo)
Web Vitals:         Vercel Speed Insights + Datadog RUM
Dashboards:         Grafana Cloud (ou Datadog nativo)
Synthetic:          Datadog Synthetic ou Checkly
SLO tracking:       Nobl9 ou Datadog SLO

Total: $500-2.000/mes
Nivel de maturidade: 3-4 (Avancado a Elite)
```

**O que ganha:** Full-stack observability, distributed tracing completo, SLOs com error budgets, on-call profissional com escalacao, profiling em producao.

---

## Apendice A: Glossario

| Termo | Definicao |
|-------|-----------|
| **APM** | Application Performance Monitoring -- monitoramento de performance de aplicacoes |
| **CNCF** | Cloud Native Computing Foundation |
| **CLS** | Cumulative Layout Shift -- metrica de estabilidade visual |
| **FCP** | First Contentful Paint -- primeiro conteudo visivel |
| **INP** | Interaction to Next Paint -- responsividade |
| **LCP** | Largest Contentful Paint -- velocidade de carregamento |
| **MTTR** | Mean Time To Resolution -- tempo medio de resolucao |
| **MTTA** | Mean Time To Acknowledge -- tempo medio de reconhecimento |
| **OTLP** | OpenTelemetry Protocol -- protocolo padrao de telemetria |
| **PromQL** | Prometheus Query Language |
| **RUM** | Real User Monitoring -- dados de usuarios reais |
| **SLI/SLO/SLA** | Service Level Indicator / Objective / Agreement |
| **SRE** | Site Reliability Engineering |
| **TTFB** | Time to First Byte -- tempo ate primeiro byte |
| **USE** | Utilization, Saturation, Errors -- metodo de monitoramento de infra |

## Apendice B: Checklist de Implementacao

### Nivel 1 -- Baseline (Dia 1)

- [ ] Instalar Sentry (free) com source maps
- [ ] Configurar UptimeRobot para endpoint principal
- [ ] Adicionar `@vercel/speed-insights` ao layout
- [ ] Configurar alerta de email para erros criticos
- [ ] Verificar que `.env` NAO esta nos logs

### Nivel 2 -- Standard (Semana 1-2)

- [ ] Configurar structured logging com Pino
- [ ] Setup Axiom ou Loki para log aggregation
- [ ] Criar Vercel Drain para enviar logs ao aggregator
- [ ] Implementar Correlation IDs no middleware
- [ ] Dashboard Grafana com metricas basicas (latencia, errors, traffic)
- [ ] Alertas para: error rate > 1%, latencia p99 > 2s, uptime < 99.5%
- [ ] Documentar runbook para top 3 incidentes mais comuns

### Nivel 3 -- Advanced (Mes 1-3)

- [ ] Instrumentar com OpenTelemetry (traces + metricas)
- [ ] APM ativo (Datadog ou New Relic)
- [ ] SLOs definidos para servicos criticos
- [ ] Error budgets calculados e monitorados
- [ ] On-call rotation configurada (PagerDuty ou Grafana IRM)
- [ ] PostgreSQL monitoring (pg_stat_statements, connection pool)
- [ ] Synthetic monitoring para fluxos criticos
- [ ] Business metrics no dashboard (conversoes, revenue)

---

## Fontes Principais

1. [IBM -- Three Pillars of Observability](https://www.ibm.com/think/insights/observability-pillars)
2. [Google SRE Book -- Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
3. [CNCF -- OpenTelemetry](https://www.cncf.io/projects/opentelemetry/)
4. [OpenTelemetry Docs -- Node.js](https://opentelemetry.io/docs/languages/js/getting-started/nodejs/)
5. [Dash0 -- Top 5 Node.js Logging Frameworks 2025](https://www.dash0.com/faq/the-top-5-best-node-js-and-javascript-logging-frameworks-in-2025-a-complete-guide)
6. [Better Stack -- Node.js Logging Libraries](https://betterstack.com/community/guides/logging/best-nodejs-logging-libraries/)
7. [Grafana Labs -- Grafana vs Datadog](https://grafana.com/compare/grafana-vs-datadog/)
8. [SigNoz -- Datadog vs Grafana 2026](https://signoz.io/blog/datadog-vs-grafana/)
9. [Datadog Pricing](https://www.datadoghq.com/pricing/)
10. [Finout -- Datadog Pricing Breakdown](https://www.finout.io/blog/datadog-pricing-explained)
11. [Vercel Docs -- Speed Insights](https://vercel.com/docs/speed-insights)
12. [Vercel Blog -- Introducing Vercel Drains](https://vercel.com/blog/introducing-vercel-drains)
13. [Supabase -- New Observability Features](https://supabase.com/blog/new-observability-features-in-supabase)
14. [Supabase Docs -- Metrics API](https://supabase.com/docs/guides/telemetry/metrics)
15. [incident.io -- SLO, SLA, SLI Guide](https://incident.io/blog/slo-sla-sli)
16. [Nobl9 -- Error Budgets Guide](https://www.nobl9.com/resources/a-complete-guide-to-error-budgets-setting-up-slos-slis-and-slas-to-maintain-reliability)
17. [Better Stack -- Datadog vs Sentry 2026](https://betterstack.com/community/comparisons/datadog-vs-sentry/)
18. [Apptension -- Sentry vs Datadog vs New Relic](https://apptension.com/guides/best-saas-error-monitoring-and-observability-tools-sentry-vs-datadog-vs-new-relic)
19. [Sentry Docs -- Next.js](https://docs.sentry.io/platforms/javascript/guides/nextjs/)
20. [Sentry Pricing](https://sentry.io/pricing/)
21. [New Relic Free Tier](https://newrelic.com/pricing/free-tier)
22. [Grafana Cloud Pricing](https://grafana.com/pricing/)
23. [Axiom Pricing](https://axiom.co/pricing)
24. [MDN -- RUM vs Synthetic Monitoring](https://developer.mozilla.org/en-US/docs/Web/Performance/Guides/Rum-vs-Synthetic)
25. [incident.io -- Alert Fatigue 2025 Guide](https://incident.io/blog/2025-guide-to-preventing-alert-fatigue-for-modern-on-call-teams)
26. [TechnologyMatch -- PagerDuty vs Opsgenie vs Grafana OnCall](https://technologymatch.com/blog/pagerduty-vs-opsgenie-vs-grafana-oncall-which-incident-management-tool-is-right-for-your-team)
27. [Rootly -- Incident Response Runbooks](https://rootly.com/incident-response/runbooks)
28. [Microsoft Engineering Playbook -- Correlation IDs](https://microsoft.github.io/code-with-engineering-playbook/observability/correlation-id/)
29. [Last9 -- RED Method](https://last9.io/blog/monitoring-with-red-method/)
30. [Instaclustr -- pg_stat_activity](https://www.instaclustr.com/blog/mastering-pg-stat-activity-for-real-time-monitoring-in-postgresql/)
31. [Grafana -- Lambda OpenTelemetry Guide](https://grafana.com/blog/2025/04/15/aws-lambda-opentelemetry-and-grafana-cloud-a-guide-to-serverless-observability-considerations/)
32. [Middleware -- Observability Maturity Model](https://middleware.io/blog/observability-maturity-model/)
33. [AWS -- Observability Best Practices](https://aws-observability.github.io/observability-best-practices/guides/observability-maturity-model/)
34. [Mezmo -- Observability Cost Reduction](https://www.mezmo.com/learn-observability/observability-cost-reduction-a-practical-guide)
35. [Grafana -- Managing Observability Costs at Scale](https://grafana.com/blog/managing-observability-costs-at-scale-a-look-at-the-latest-cost-management-features-in-grafana-cloud/)
36. [The New Stack -- OpenTelemetry and AI in 2025](https://thenewstack.io/observability-in-2025-opentelemetry-and-ai-to-fill-in-gaps/)
37. [Datadog Docs -- Vercel Integration](https://docs.datadoghq.com/integrations/vercel/)
38. [ChaosSearch -- Grafana Loki vs Datadog](https://www.chaossearch.io/blog/why-organizations-use-grafana-loki-to-replace-datadog)
39. [KloudFuse -- Kubernetes Monitoring Tools 2025](https://www.kloudfuse.com/blog/kubernetes-monitoring-tools-in-2025-our-top-10-picks)
40. [DevOps Institute -- Chaos Engineering Observability](https://www.devopsinstitute.com/the-practice-of-chaos-engineering-observability/)
