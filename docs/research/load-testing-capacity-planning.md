# Load Testing & Capacity Planning

> **Deep Research** | Pesquisa aprofundada sobre como validar que seu sistema aguenta trafego real e como planejar crescimento.
> **Data:** Abril 2026 | **Fontes:** 40+ fontes verificadas via WebSearch

---

## Indice

1. [Fundamentos de Load Testing](#1-fundamentos-de-load-testing)
2. [Comparacao de Ferramentas](#2-comparacao-de-ferramentas-de-load-testing)
3. [k6 Deep Dive](#3-k6-deep-dive)
4. [Testando Arquiteturas Especificas](#4-testando-arquiteturas-especificas)
5. [Capacity Planning](#5-capacity-planning)
6. [Padroes de Auto-Scaling](#6-padroes-de-auto-scaling)
7. [Performance Budgets](#7-performance-budgets)
8. [Chaos Engineering](#8-chaos-engineering)
9. [Benchmarks do Mundo Real](#9-benchmarks-do-mundo-real)
10. [Decision Framework: MVP vs Enterprise](#10-decision-framework-mvp-vs-enterprise-scale)

---

## 1. Fundamentos de Load Testing

### 1.1 Tipos de Teste de Carga

Cada tipo de teste de carga responde a uma pergunta diferente sobre o sistema. A escolha do tipo correto e crucial para obter insights acionaveis.

| Tipo | Objetivo | Padrao de Carga | Pergunta que Responde |
|------|----------|-----------------|----------------------|
| **Load Test** | Validar performance sob carga esperada | Ramp-up gradual ate carga-alvo, manter, ramp-down | "O sistema performa bem sob trafego normal?" |
| **Stress Test** | Descobrir limites do sistema | Carga alem do esperado (1.5-2x) | "O que acontece quando ultrapassamos a carga normal?" |
| **Spike Test** | Validar resiliencia a picos subitos | Aumento instantaneo massivo, queda rapida | "O sistema sobrevive a um pico repentino (Black Friday, viral)?" |
| **Soak/Endurance Test** | Detectar degradacao ao longo do tempo | Carga constante por horas (4-24h) | "Existem memory leaks, connection leaks, degradacao gradual?" |
| **Breakpoint Test** | Encontrar o ponto de ruptura | Aumento progressivo continuo ate falha | "Qual e a capacidade maxima absoluta do sistema?" |

> **Fonte:** [Grafana Labs - Types of Load Testing](https://grafana.com/load-testing/types-of-load-testing/), [Locust Cloud - 5 Essential Load Test Profiles](https://www.locust.cloud/blog/5-essential-load-test-profiles/)

**Smoke Test** (variacao): Um teste rapido com 1-2 VUs para verificar que o script funciona antes de executar o teste completo. Sempre execute um smoke test antes de qualquer teste de carga real.

### 1.2 Metricas-Chave

As metricas de load testing se dividem em quatro categorias fundamentais:

#### Throughput (Vazao)

- **RPS (Requests Per Second)**: Numero de requisicoes processadas por segundo
- **TPS (Transactions Per Second)**: Numero de transacoes completas por segundo (pode incluir multiplas requisicoes)
- **Regra**: Se o throughput nao cresce linearmente com a carga, existe um bottleneck

#### Latencia (Tempo de Resposta)

| Percentil | Significado | Alvo Tipico (API) |
|-----------|-------------|-------------------|
| **p50 (mediana)** | Metade dos usuarios experimenta isso ou menos | < 200ms |
| **p95** | 95% dos usuarios — o "caso quase-pior" | < 500ms |
| **p99** | 99% dos usuarios — tail latency | < 1000ms |
| **p99.9** | 1 em 1000 — critico para SLAs | < 2000ms |

> **NUNCA use media (average) como metrica principal.** Medias escondem tail latency. Um p50 de 100ms com p99 de 5000ms indica problemas serios que a media de 150ms nao mostraria.
> **Fonte:** [TestGuild - Load Testing Best Practices](https://testguild.com/best-load-testing/)

#### Error Rate (Taxa de Erro)

- **Taxa aceitavel em producao**: < 0.1% (1 erro a cada 1000 requisicoes)
- **Distinguir tipos**: HTTP 4xx (erro do cliente) vs 5xx (erro do servidor)
- **Timeouts**: Contar separadamente — sao os erros mais impactantes para UX

#### Saturacao

- **CPU**: Acima de 70% sustentado = risco de degradacao
- **Memoria**: Crescimento monotono = memory leak
- **Conexoes de banco**: Perto do limite = pool exhaustion iminente
- **Disk I/O**: IOPS no limite = bottleneck de storage

### 1.3 Metodologia de Teste

A abordagem recomendada segue quatro fases:

```
Fase 1: BASELINE        Fase 2: TARGET         Fase 3: STRESS         Fase 4: REPORT
Medir performance       Testar com carga       Testar alem do         Documentar e
atual sem carga         esperada               esperado               decidir
significativa           (producao real)        (1.5-3x target)        
                                                                      
Smoke test              Load test              Stress test            Analise de
1-5 VUs                 VUs = usuarios         Spike test             resultados
5 min                   reais estimados        Soak test              Recomendacoes
                        10-30 min              30-60+ min             Plano de acao
```

### 1.4 Erros Comuns e Anti-Patterns

| Anti-Pattern | Problema | Solucao |
|-------------|----------|---------|
| **Carga insuficiente** | Testar com 100 usuarios e assumir que 10.000 funcionara igual | Usar dados reais de producao para estimar carga; testar com multiplos do target |
| **Ignorar think time** | VUs enviando requests sem pausa = carga irreal 10-100x maior que real | Adicionar delays realisticos entre acoes (2-10s entre cliques) |
| **Testar em laboratorio perfeito** | Rede local, dados minimos, sem concorrencia real | Testar de regioes diferentes, com dados realisticos, incluindo background load |
| **Usar media como metrica** | p50=100ms, p99=5s, media=150ms esconde o problema | Sempre usar percentis (p50, p95, p99) |
| **Ignorar Coordinated Omission** | Quando o servidor esta lento, o cliente envia MENOS requests, mascarando a latencia real | Usar ferramentas que corrigem isso (k6, wrk2) |
| **Nao correlacionar dados dinamicos** | Scripts com tokens/sessoes hardcoded = requests falhando silenciosamente | Extrair tokens dinamicos, usar parametrizacao |
| **Testar so a API** | API rapida mas banco lento, ou CDN mascara o problema | Testar end-to-end incluindo banco, cache, servicos externos |
| **Nao considerar operacoes async** | Filas (Kafka, RabbitMQ) acumulando backlog invisivel | Monitorar profundidade de filas, lag de consumers |

> **Fonte:** [Expert-Soft - Load Testing Best Practices 2025](https://expert-soft.com/blog/load-testing-best-practices/), [RadView - Common Load Testing Mistakes](https://www.radview.com/blog/common-load-testing-mistakes-fix-production)

---

## 2. Comparacao de Ferramentas de Load Testing

### 2.1 Tabela Comparativa

| Ferramenta | Linguagem | Protocolos | Cloud Option | Preco (OSS) | Learning Curve | Stars (GitHub) |
|-----------|-----------|-----------|-------------|-------------|----------------|---------------|
| **k6** | JavaScript/TypeScript | HTTP, WebSocket, gRPC, Browser | Grafana Cloud k6 | Gratuito | Baixa | 29.9k+ |
| **Artillery** | YAML + JS/TS | HTTP, WebSocket, Socket.io, Playwright | Artillery Cloud (AWS Fargate) | Gratuito | Baixa | 8k+ |
| **Locust** | Python | HTTP (extensivel) | Locust Cloud | Gratuito | Baixa | 25k+ |
| **JMeter** | Java (GUI/XML) | HTTP, JDBC, LDAP, JMS, FTP, SMTP | BlazeMeter, OctoPerf | Gratuito | Media-Alta | 8.5k+ |
| **Gatling** | Scala/Java/JS | HTTP, WebSocket, JMS, SSE | Gatling Enterprise | Gratuito (OSS) | Media | 6.5k+ |
| **wrk** | Lua (scripts) | HTTP apenas | Nao | Gratuito | Baixa | 38k+ |
| **wrk2** | Lua (scripts) | HTTP apenas | Nao | Gratuito | Baixa | 4k+ |

> **Fonte:** [Vervali - Best Load Testing Tools 2026](https://www.vervali.com/blog/best-load-testing-tools-in-2026-definitive-guide-to-jmeter-gatling-k6-loadrunner-locust-blazemeter-neoload-artillery-and-more/)

### 2.2 k6 (Grafana)

**O favorito da comunidade dev moderna.** JavaScript/TypeScript nativo, arquitetura em Go para alta performance, integracao nativa com Grafana.

- **Vantagens**: Developer-friendly, TypeScript nativo (v1.0+, maio 2025), extensoes em Go (xk6), thresholds como SLOs, integracao CI/CD nativa, k6 Browser para testes de browser, Kubernetes Operator (v1.0 GA, setembro 2025)
- **Desvantagens**: Sem GUI para criacao de testes (k6 Studio mitiga isso), distribuido requer Kubernetes ou Cloud
- **Melhor para**: Equipes de desenvolvimento modernas, CI/CD pipelines, projetos JavaScript/TypeScript

> **Fonte:** [GitHub - grafana/k6](https://github.com/grafana/k6), [k6.io](https://k6.io/)

### 2.3 Artillery

**YAML-first com integracao Playwright unica.** Ideal para quem quer load testing + browser testing.

- **Vantagens**: Configuracao YAML simples, Playwright nativo para browser load testing, metricas de Web Vitals automaticas (LCP, FCP, TTFB), teste distribuido via AWS Fargate/Azure Container Instances, Node.js ecosystem
- **Desvantagens**: Menos extensivel que k6, comunidade menor, cloud apenas AWS/Azure
- **Melhor para**: Times Node.js, testes que precisam simular browser real, YAML-first teams

> **Fonte:** [Artillery.io - Playwright Integration](https://www.artillery.io/docs/reference/engines/playwright), [GitHub - artilleryio/artillery](https://github.com/artilleryio/artillery)

### 2.4 Locust

**Python puro com dashboard web real-time.** Event-based (gevent), milhares de usuarios por processo.

- **Vantagens**: Python puro (qualquer lib Python disponivel), dashboard web real-time, distribuido nativo (master/worker), async Python suportado, arquitetura plugavel
- **Desvantagens**: Performance inferior a Go-based (k6), HTTP-focused por padrao (extensivel mas requer trabalho)
- **Melhor para**: Times Python, data science teams, quem precisa de logica de teste complexa

> **Fonte:** [Locust.io](https://locust.io/), [GitHub - locustio/locust](https://github.com/locustio/locust)

### 2.5 JMeter

**O veterano enterprise.** 25+ anos, GUI completa, suporte a quase todos os protocolos.

- **Vantagens**: GUI intuitiva para nao-programadores, protocolos extensivos (HTTP, JDBC, LDAP, JMS, FTP, SMTP), ecossistema de plugins enorme, documentacao extensa, teste distribuido nativo
- **Desvantagens**: Baseado em threads Java (pesado em recursos), GUI lenta sob carga, XML como formato de teste (dificil de versionar), arquitetura datada
- **Melhor para**: QA teams enterprise, testes multi-protocolo, ambientes legacy

> **Fonte:** [BlazeMeter - Gatling vs JMeter](https://www.blazemeter.com/blog/gatling-vs-jmeter), [TestLeaf - Top 5 Load Testing Tools 2025](https://www.testleaf.com/blog/5-best-load-testing-tools-in-2025/)

### 2.6 Gatling

**Alto throughput com reports HTML detalhados.** Arquitetura async nao-bloqueante em Scala.

- **Vantagens**: Eficiente em recursos (async, nao-bloqueante), reports HTML detalhados out-of-the-box, "load testing as code" (Scala/Java/JS), alta concorrencia com poucos recursos
- **Desvantagens**: Teste distribuido apenas na versao Enterprise (paga), comunidade menor que JMeter/k6, requer conhecimento de codigo
- **Melhor para**: Times JVM, alta concorrencia com poucos recursos, reports detalhados sem setup adicional

> **Fonte:** [BrowserStack - JMeter vs Gatling](https://www.browserstack.com/guide/jmeter-vs-gatling)

### 2.7 wrk e wrk2

**Benchmarking HTTP leve e preciso.** Ferramentas de linha de comando para medicao de latencia.

- **wrk**: Benchmarking HTTP multithreaded em C. Extremamente leve, scripts Lua para customizacao. Limitacao: sofre de Coordinated Omission, reportando latencia menor que a real
- **wrk2**: Fork do wrk que corrige Coordinated Omission com throughput constante e HdrHistograms. Produz medicoes precisas ate o 99.9999% percentil. A flag `-R` define o throughput alvo em RPS

> **Diferenca critica:** wrk mede "quanto tempo ESTA requisicao levou", wrk2 mede "quanto tempo o USUARIO esperou desde quando deveria ter recebido a resposta". wrk2 e a medicao correta para SLAs.
> **Fonte:** [GitHub - giltene/wrk2](https://github.com/giltene/wrk2), [GitHub - wg/wrk](https://github.com/wg/wrk)

### 2.8 Recomendacao por Cenario

| Cenario | Ferramenta Recomendada | Justificativa |
|---------|----------------------|---------------|
| MVP/Startup (JS/TS stack) | **k6** | Developer-friendly, CI/CD nativo, gratuito |
| Browser load testing | **Artillery + Playwright** | Unico com Playwright nativo + metricas Web Vitals |
| Data science / ML team | **Locust** | Python nativo, integracao com ecossistema |
| Enterprise multi-protocolo | **JMeter** | Protocolos extensivos, GUI, ecossistema maduro |
| Alta concorrencia JVM | **Gatling** | Async eficiente, reports detalhados |
| Benchmark rapido HTTP | **wrk2** | Preciso, leve, correcao de Coordinated Omission |
| CI/CD pipeline | **k6** | Thresholds como SLOs, exit codes, GitHub Actions |

---

## 3. k6 Deep Dive

### 3.1 Instalacao e Setup

```bash
# macOS
brew install k6

# Windows (Chocolatey)
choco install k6

# Windows (winget)
winget install k6 --source winget

# Linux (Debian/Ubuntu)
sudo gpg -k
sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg \
  --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" \
  | sudo tee /etc/apt/sources.list.d/k6.list
sudo apt-get update && sudo apt-get install k6

# Docker
docker run --rm -i grafana/k6 run - <script.js

# Verificar instalacao
k6 version
```

> **Fonte:** [k6.io/docs](https://k6.io/docs/), [k6 Complete Guide](https://agmazon.com/blog/articles/technology/202602/k6-load-testing-complete-guide-en.html)

### 3.2 Escrevendo Test Scripts

#### Script Basico com Thresholds e Checks

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

// Configuracao: define cenarios, thresholds (SLOs), e stages
export const options = {
  // Stages definem o padrao de carga (ramp-up, plateau, ramp-down)
  stages: [
    { duration: '2m', target: 20 },   // Ramp-up: 0 -> 20 VUs em 2 minutos
    { duration: '5m', target: 20 },   // Plateau: manter 20 VUs por 5 minutos
    { duration: '2m', target: 50 },   // Ramp-up: 20 -> 50 VUs em 2 minutos
    { duration: '5m', target: 50 },   // Plateau: manter 50 VUs por 5 minutos
    { duration: '2m', target: 0 },    // Ramp-down: 50 -> 0 VUs em 2 minutos
  ],

  // Thresholds: criterios de PASS/FAIL (codifica seus SLOs)
  thresholds: {
    http_req_failed: ['rate<0.01'],           // < 1% de erros HTTP
    http_req_duration: ['p(95)<500'],          // p95 < 500ms
    http_req_duration: ['p(99)<1000'],         // p99 < 1000ms
    'http_req_duration{name:homepage}': ['p(95)<300'],  // Threshold por endpoint
  },
};

export default function () {
  // Checks: validacoes funcionais (nao causam falha no teste)
  const res = http.get('https://api.exemplo.com/health', {
    tags: { name: 'homepage' },
  });

  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
    'body contains expected': (r) => r.body.includes('ok'),
  });

  // Think time: simula tempo real entre acoes do usuario
  sleep(Math.random() * 3 + 1); // 1-4 segundos aleatorio
}
```

#### Multiplos Scenarios

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  scenarios: {
    // Cenario 1: Navegacao (maioria dos usuarios)
    browsing: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '2m', target: 100 },
        { duration: '5m', target: 100 },
        { duration: '2m', target: 0 },
      ],
      exec: 'browsingFlow',
      gracefulRampDown: '30s',
    },

    // Cenario 2: Compra (10% dos usuarios)
    purchasing: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '2m', target: 10 },
        { duration: '5m', target: 10 },
        { duration: '2m', target: 0 },
      ],
      exec: 'purchaseFlow',
      gracefulRampDown: '30s',
    },

    // Cenario 3: Carga constante em RPS
    api_constant_rate: {
      executor: 'constant-arrival-rate',
      rate: 50,             // 50 iteracoes por timeUnit
      timeUnit: '1s',       // = 50 RPS
      duration: '5m',
      preAllocatedVUs: 20,
      maxVUs: 100,
      exec: 'apiCheck',
    },
  },

  thresholds: {
    'http_req_duration{scenario:browsing}': ['p(95)<800'],
    'http_req_duration{scenario:purchasing}': ['p(95)<1200'],
    'http_req_duration{scenario:api_constant_rate}': ['p(95)<300'],
  },
};

export function browsingFlow() {
  http.get('https://api.exemplo.com/products');
  sleep(2);
  http.get('https://api.exemplo.com/products/123');
  sleep(3);
}

export function purchaseFlow() {
  // Credenciais devem vir de arquivo externo ou variavel de ambiente
  // Nunca hardcode senhas em scripts de teste
  const loginRes = http.post('https://api.exemplo.com/auth/login', JSON.stringify({
    email: `testuser_${__VU}@example.com`,
    password: __ENV.TEST_USER_PASSWORD,  // via variavel de ambiente
  }), { headers: { 'Content-Type': 'application/json' } });

  const token = loginRes.json('access_token');

  http.post('https://api.exemplo.com/cart/add', JSON.stringify({
    product_id: 123,
    quantity: 1,
  }), {
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`,
    },
  });
  sleep(1);
}

export function apiCheck() {
  http.get('https://api.exemplo.com/health');
}
```

> **Fonte:** [Grafana k6 Documentation - Scenarios](https://grafana.com/docs/k6/latest/using-k6/scenarios/advanced-examples/), [k6 - Write Your First Test](https://grafana.com/docs/k6/latest/get-started/write-your-first-test/)

### 3.3 Data Parameterization

```javascript
import http from 'k6/http';
import { SharedArray } from 'k6/data';
import papaparse from 'https://jslib.k6.io/papaparse/5.1.1/index.js';

// SharedArray: carrega dados UMA vez e compartilha entre TODOS os VUs
// (sem isso, cada VU carrega uma copia completa na memoria)

// Opcao 1: JSON
const usersJSON = new SharedArray('users', function () {
  return JSON.parse(open('./data/users.json'));
});

// Opcao 2: CSV com Papa Parse
const usersCSV = new SharedArray('csv-users', function () {
  return papaparse.parse(open('./data/users.csv'), { header: true }).data;
});

export default function () {
  // Selecionar usuario aleatorio
  const user = usersJSON[Math.floor(Math.random() * usersJSON.length)];

  const res = http.post('https://api.exemplo.com/auth/login', JSON.stringify({
    email: user.email,
    password: user.password,  // vem do arquivo de dados externo
  }), {
    headers: { 'Content-Type': 'application/json' },
  });
}
```

> **Fonte:** [Grafana k6 - Data Parameterization](https://grafana.com/docs/k6/latest/examples/data-parameterization/), [Grafana k6 - SharedArray](https://grafana.com/docs/k6/latest/javascript-api/k6-data/sharedarray/)

### 3.4 Custom Metrics

```javascript
import http from 'k6/http';
import { Counter, Gauge, Rate, Trend } from 'k6/metrics';

// 4 tipos de metricas customizadas:
const loginDuration = new Trend('login_duration');     // Min, max, avg, percentis
const loginFailRate = new Rate('login_fail_rate');      // Proporcao boolean (sucesso/falha)
const loginCount = new Counter('login_count');          // Contador incremental
const activeUsers = new Gauge('active_users');           // Ultimo valor, min, max

export const options = {
  thresholds: {
    login_duration: ['p(95)<800'],     // p95 de login < 800ms
    login_fail_rate: ['rate<0.05'],    // < 5% de falhas no login
  },
};

export default function () {
  const start = Date.now();
  const res = http.post('https://api.exemplo.com/auth/login', JSON.stringify({
    email: `user_${__VU}@test.com`,
    password: __ENV.TEST_PASSWORD,  // via env var: k6 run -e TEST_PASSWORD=xxx script.js
  }), { headers: { 'Content-Type': 'application/json' } });

  const duration = Date.now() - start;

  // Registrar metricas
  loginDuration.add(duration);
  loginFailRate.add(res.status !== 200);
  loginCount.add(1);
  activeUsers.add(__VU); // numero do VU atual
}
```

> **Fonte:** [Grafana k6 - Create Custom Metrics](https://grafana.com/docs/k6/latest/using-k6/metrics/create-custom-metrics/)

### 3.5 Teste Distribuido com k6 Cloud

```bash
# Login no Grafana Cloud k6
k6 cloud login --token YOUR_API_TOKEN

# Executar teste no Cloud (de multiplas regioes)
k6 cloud run script.js

# Executar localmente mas enviar resultados para o Cloud
k6 run --out cloud script.js
```

Para Kubernetes, o k6 Operator (v1.0 GA, setembro 2025) permite execucao declarativa:

```yaml
# k6-test.yaml - TestRun CRD
apiVersion: k6.io/v1alpha1
kind: TestRun
metadata:
  name: k6-load-test
spec:
  parallelism: 4          # 4 pods executando em paralelo
  script:
    configMap:
      name: k6-test-script
      file: script.js
  arguments: --out cloud
```

> **Fonte:** [k6.io](https://k6.io/), [k6 Complete Guide](https://agmazon.com/blog/articles/technology/202602/k6-load-testing-complete-guide-en.html)

### 3.6 Integracao CI/CD (GitHub Actions)

```yaml
# .github/workflows/load-test.yml
name: Load Test

on:
  pull_request:
    branches: [main]
  # Ou agendado (cron)
  schedule:
    - cron: '0 6 * * 1' # Toda segunda-feira as 6h

jobs:
  load-test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup k6
        uses: grafana/setup-k6-action@v1

      - name: Run Load Test
        uses: grafana/run-k6-action@v1
        with:
          path: tests/load/api-load-test.js
          # Para enviar resultados ao Grafana Cloud:
          # cloud-run-locally: true
        env:
          K6_CLOUD_TOKEN: ${{ secrets.K6_CLOUD_TOKEN }}

      - name: Upload Results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: k6-results
          path: results/
```

> **Nota:** A action oficial `grafana/run-k6-action` so funciona em runners Linux. Para Windows ou macOS, instale k6 manualmente no pipeline.
> **Fonte:** [k6 Blog - Load Testing Using GitHub Actions](https://k6.io/blog/load-testing-using-github-actions/), [Grafana Labs - k6 and GitHub Actions](https://grafana.com/blog/2024/07/15/performance-testing-with-grafana-k6-and-github-actions/)

### 3.7 Dashboard com Grafana

A stack recomendada para visualizacao e: **k6 -> InfluxDB/Prometheus -> Grafana**.

```bash
# Opcao 1: Enviar metricas para InfluxDB v2
k6 run --out influxdb=http://localhost:8086/k6 script.js

# Opcao 2: Expor metricas para Prometheus (via extensao)
# Requer xk6-output-prometheus-remote
k6 run --out experimental-prometheus-rw script.js

# Opcao 3: Output nativo JSON/CSV para analise posterior
k6 run --out json=results.json script.js
k6 run --out csv=results.csv script.js
```

**Setup com Docker Compose (k6 + InfluxDB + Grafana):**

```yaml
# docker-compose.yml
version: '3'
services:
  influxdb:
    image: influxdb:2.7
    ports:
      - "8086:8086"
    environment:
      - DOCKER_INFLUXDB_INIT_MODE=setup
      - DOCKER_INFLUXDB_INIT_USERNAME=admin
      - DOCKER_INFLUXDB_INIT_PASSWORD=${INFLUXDB_PASSWORD}
      - DOCKER_INFLUXDB_INIT_ORG=k6
      - DOCKER_INFLUXDB_INIT_BUCKET=k6

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_AUTH_ANONYMOUS_ENABLED=true
    depends_on:
      - influxdb
```

**Dashboards pre-configurados disponiveis:** [k6 Dashboard (ID 21704)](https://grafana.com/grafana/dashboards/21704-k6-dashboard/) e [k6 Dashboard (ID 16072)](https://grafana.com/grafana/dashboards/16072-k6/).

> **Fonte:** [Grafana k6 - InfluxDB Output](https://grafana.com/docs/k6/latest/results-output/real-time/influxdb/), [Dev.to - k6 + Docker + Prometheus + Grafana](https://dev.to/leading-edje/open-source-load-testing-with-k6-docker-prometheus-and-grafana-5ej6)

---

## 4. Testando Arquiteturas Especificas

### 4.1 Vercel (Serverless)

#### Limites e Consideracoes

| Parametro | Hobby | Pro | Enterprise |
|-----------|-------|-----|-----------|
| Concurrent executions | 30.000 | 30.000 | 100.000+ |
| Function timeout | 60s | 300s | 900s |
| Fast Data Transfer | 100 GB/mes | 1 TB/mes | Custom |
| GB-hours (compute) | 4h CPU ativo/mes | ~1.000 GB-hours | Custom |

#### Cold Starts

Vercel reporta zero cold starts para **99.37% de todas as requisicoes**. A tecnologia Fluid Compute (habilitada por padrao desde abril 2025) reduz cold starts via:
- Bytecode caching automatico
- Pre-warming de funcoes em deployments de producao
- Funcoes arquivadas reativam em ~1s extra no primeiro cold start

**Benchmarks de latencia:**

| Tipo | P50 Latencia | Cold Start |
|------|-------------|------------|
| Edge Runtime | ~106ms | < 50ms |
| Serverless (warm) | ~246ms | N/A |
| Serverless (cold) | ~859ms | 100ms-1s+ |

> **Fonte:** [Vercel - Scale to One: Fluid Compute](https://vercel.com/blog/scale-to-one-how-fluid-solves-cold-starts), [Vercel - Fluid Compute Docs](https://vercel.com/docs/fluid-compute)

#### Politica de Load Testing na Vercel

A Vercel **requer notificacao previa** antes de executar load tests. Sem aviso, IPs serao bloqueados por trafego anormal. Para testar:

1. Contactar suporte Vercel antes do teste
2. Usar Fluid Compute com warm-up de 2-3 minutos antes de medir
3. Testar sustained load e spikes separadamente
4. Monitorar consumo de GB-hours (billing)

> **Fonte:** [Vercel KB - Load Testing Policy](https://vercel.com/kb/guide/what-s-vercel-s-policy-regarding-load-testing-deployments)

### 4.2 Supabase

#### Connection Limits por Compute Size

| Instancia | CPU | Memoria | Max DB Connections | Pooler Max Clients | Preco Aprox. |
|-----------|-----|---------|-------------------|-------------------|-------------|
| Micro | 2-core (shared) | 1 GB | 60 | 200 | Incluso no Pro |
| Small | 2-core (shared) | 2 GB | 90 | 400 | ~$50/mes |
| Medium | 2-core (shared) | 4 GB | 120 | 600 | ~$75/mes |
| Large | 2-core (dedicated) | 8 GB | 160 | 800 | ~$110/mes |
| XL | 4-core (dedicated) | 16 GB | 240 | 1.000 | ~$210/mes |
| 2XL | 8-core (dedicated) | 32 GB | 380 | 1.500 | ~$410/mes |
| 4XL | 16-core (dedicated) | 64 GB | 480 | 3.000 | ~$810/mes |
| 8XL | 32-core (dedicated) | 128 GB | 490 | 6.000 | ~$1.610/mes |
| 12XL | 48-core (dedicated) | 192 GB | 500 | 9.000 | Custom |
| 16XL | 64-core (dedicated) | 256 GB | 500 | 12.000 | Custom |

> **Importante:** Limites do Supavisor (pooler) sao hardcoded e nao podem ser alterados sem upgrade de compute.
> **Fonte:** [Supabase Docs - Compute and Disk](https://supabase.com/docs/guides/platform/compute-and-disk)

#### RLS Performance Sob Carga

RLS (Row Level Security) tem impacto de performance especialmente com Realtime:
- Cada change event e verificado contra TODOS os subscribers
- 100 usuarios subscritos + 1 INSERT = **100 reads** (um por usuario)
- Bottleneck de banco pode atrasar mensagens ate timeout
- **Recomendacao**: Indexar as colunas usadas nas RLS policies (ex: `user_id`)

#### Realtime Limits

O cluster Realtime suporta **milhoes de conexoes concorrentes**, porem limits sao configuraveis por projeto. Benchmarks oficiais mostram 10.000+ conexoes concorrentes sem problemas.

> **Fonte:** [Supabase Docs - Realtime Limits](https://supabase.com/docs/guides/realtime/limits), [Supabase Docs - Realtime Benchmarks](https://supabase.com/docs/guides/realtime/benchmarks)

### 4.3 Next.js API Routes

#### Pontos Criticos de Load Testing

```javascript
// k6 script para testar ISR cache warming
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  scenarios: {
    // Testar ISR: primeira requisicao gera, subsequentes servem cache
    isr_warming: {
      executor: 'per-vu-iterations',
      vus: 50,
      iterations: 1, // Cada VU faz 1 request — testa cache miss
      maxDuration: '30s',
    },
    // Testar API routes sob carga
    api_load: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '1m', target: 100 },
        { duration: '3m', target: 100 },
        { duration: '1m', target: 0 },
      ],
      exec: 'apiRouteTest',
    },
  },
};

export default function () {
  // Testar pagina ISR — primeira chamada e lenta, cache subsequente e rapido
  const res = http.get('https://seu-app.vercel.app/products/123');
  check(res, {
    'ISR page loads': (r) => r.status === 200,
    'response under 2s': (r) => r.timings.duration < 2000,
    'cache header present': (r) => r.headers['X-Vercel-Cache'] !== undefined,
  });
}

export function apiRouteTest() {
  const res = http.get('https://seu-app.vercel.app/api/products');
  check(res, {
    'API returns 200': (r) => r.status === 200,
    'API under 500ms': (r) => r.timings.duration < 500,
  });
}
```

### 4.4 PostgreSQL Sob Carga

#### Benchmark com pgbench

```bash
# Inicializar banco de teste (escala 100 = ~1.6GB de dados)
pgbench -i -s 100 -h localhost -U postgres testdb

# Executar benchmark: 100 clientes, 4 threads, 5 minutos
pgbench -c 100 -j 4 -T 300 -h localhost -U postgres testdb

# Benchmark read-only
pgbench -c 100 -j 4 -T 300 -S -h localhost -U postgres testdb
```

**Resultados de referencia (hardware moderado):** 400 clientes, 40 threads, 30 minutos: ~8M transacoes, ~4.487 TPS, latencia media de 89ms.

> **Fonte:** [PostgreSQL Docs - pgbench](https://www.postgresql.org/docs/current/pgbench.html)

### 4.5 WebSocket Testing (Supabase Realtime)

```javascript
// k6 script para testar WebSocket (Supabase Realtime)
import ws from 'k6/ws';
import { check } from 'k6';

export const options = {
  vus: 100,
  duration: '5m',
  thresholds: {
    ws_connecting: ['p(95)<1000'],   // Conexao WebSocket < 1s
    ws_msgs_received: ['count>100'], // Recebeu mensagens
  },
};

export default function () {
  const url = `wss://${__ENV.SUPABASE_PROJECT}.supabase.co/realtime/v1/websocket?apikey=${__ENV.SUPABASE_ANON_KEY}`;

  const res = ws.connect(url, {}, function (socket) {
    socket.on('open', () => {
      // Subscrever ao canal
      socket.send(JSON.stringify({
        topic: 'realtime:public:messages',
        event: 'phx_join',
        payload: {},
        ref: '1',
      }));
    });

    socket.on('message', (msg) => {
      // Processar mensagens recebidas
    });

    // Manter conexao por 30 segundos
    socket.setTimeout(function () {
      socket.close();
    }, 30000);
  });

  check(res, {
    'WebSocket connected': (r) => r && r.status === 101,
  });
}
```

---

## 5. Capacity Planning

### 5.1 Metodologia

O capacity planning segue um ciclo de quatro etapas:

```
MEASURE              MODEL                PREDICT              PROVISION
Coletar dados        Criar modelo de      Projetar demanda     Provisionar
reais de uso         capacidade atual     futura com base      infraestrutura
                                          em crescimento       com margem
                                          e sazonalidade       de seguranca

Metricas atuais      Req/s por recurso    Projecao 3-12 meses  Headroom de 30-50%
Picos historicos     Bottlenecks          Cenarios otimista/    Plano de escala
Padroes sazonais     Pontos de saturacao  pessimista/provavel   Alertas proativos
```

> **Fonte:** [Kir Shatrov - Capacity Planning for Web Apps](https://kirshatrov.com/posts/capacity-planning-for-web-apps/), [ByteByteGo - Capacity Planning](https://blog.bytebytego.com/p/capacity-planning)

### 5.2 Traffic Estimation: MAU para Peak RPS

A conversao de MAU (Monthly Active Users) para RPS (Requests Per Second) segue uma cadeia de estimativas:

```
MAU = 100.000 usuarios

Passo 1: DAU (Daily Active Users)
  DAU = MAU / 5 = 20.000 (regra geral: ~20% dos MAU)

Passo 2: Requests por dia
  Acoes por usuario/dia = 10 (paginas, cliques, API calls)
  Total requests/dia = 20.000 x 10 = 200.000

Passo 3: RPS medio
  RPS medio = 200.000 / 86.400 = ~2.3 RPS
  (Simplificacao: dividir por 100.000 para calculo rapido)

Passo 4: Peak RPS (regra 80/20)
  80% do trafego ocorre em 20% do tempo (Pareto)
  Peak RPS = RPS medio x 5 = ~11.5 RPS
  (Em eventos especiais: x10-20 do medio)

Passo 5: Headroom
  Provisionar para: Peak RPS x 1.5 = ~17 RPS
  (50% de margem para picos inesperados)
```

#### Tabela de Referencia Rapida

| MAU | DAU (20%) | Req/dia | RPS Medio | Peak RPS (5x) | Provision (1.5x) |
|-----|-----------|---------|-----------|---------------|-------------------|
| 1.000 | 200 | 2.000 | 0.02 | 0.1 | 0.15 |
| 10.000 | 2.000 | 20.000 | 0.23 | 1.2 | 1.7 |
| 100.000 | 20.000 | 200.000 | 2.3 | 11.5 | 17 |
| 1.000.000 | 200.000 | 2.000.000 | 23 | 115 | 173 |
| 10.000.000 | 2.000.000 | 20.000.000 | 231 | 1.157 | 1.736 |

> **Nota:** Esses numeros sao estimativas conservadoras. Apps com alto engajamento (redes sociais, chat) podem ter DAU/MAU de 50%+ e 50+ acoes/dia.
> **Fonte:** [Dev.to - How I Calculate Capacity for Systems Design](https://dev.to/zeeshanali0704/how-i-calculate-capacity-for-systems-design-399o), [OneUpTime - Peak Capacity Planning](https://oneuptime.com/blog/post/2026-01-30-peak-capacity-planning/view)

### 5.3 Database Capacity Planning

#### Conexoes

```
Regra de ouro para pool sizing:
  Pool Size = (Numero de cores do CPU) * 2 + 1

Exemplo: CPU 4 cores
  Pool Size ideal = 4 * 2 + 1 = 9 conexoes

Para Supabase com PostgREST:
  PostgREST pool = 40% das max_connections
  Restante: Auth server, Realtime, migrations, admin
```

#### Storage

```
Estimativa de crescimento de banco:
  Tamanho medio por registro = 500 bytes (tipico)
  Novos registros/dia = DAU x acoes-com-write x 0.3 (30% geram escrita)

  Exemplo (100K MAU):
    20.000 DAU x 10 acoes x 0.3 = 60.000 novos registros/dia
    60.000 x 500 bytes = 30 MB/dia
    30 MB x 365 = ~11 GB/ano (dados puros)
    Com indices e overhead: ~33 GB/ano (3x)
```

#### IOPS

| Operacao | IOPS Tipicas | Quando se torna bottleneck |
|----------|-------------|---------------------------|
| Leitura sequencial | Baixo | Raramente |
| Leitura aleatoria | Alto | Queries sem indice em tabelas grandes |
| Escrita | Medio-Alto | Muitos INSERTs/UPDATEs simultaneos |
| WAL writes | Medio | Replicacao sob carga alta |

### 5.4 CDN Capacity Considerations

- **Cache Hit Ratio alvo**: > 90% para assets estaticos, > 70% para ISR
- **Bandwidth**: Vercel Pro inclui 1 TB/mes; cada GB excedente custa ~$0.15
- **Edge locations**: Mais regioes = menor latencia global mas maior custo
- **Cache invalidation**: Revalidacao ISR via `revalidateTag()` ou `revalidatePath()` e eficiente; purge total de CDN e caro

### 5.5 Cost Modeling

| Escala (MAU) | Vercel | Supabase | Total Estimado/Mes |
|-------------|--------|---------|-------------------|
| 1.000 | $0 (Hobby) | $0 (Free) | $0 |
| 10.000 | $20 (Pro) | $25 (Pro) | $45 |
| 50.000 | $20 + ~$30 overage | $25 + Small ($50) | ~$125 |
| 100.000 | $20 + ~$80 overage | $25 + Medium ($75) | ~$200 |
| 500.000 | $20 + ~$300 overage | $25 + Large ($110) | ~$455 |
| 1.000.000 | Enterprise (custom) | $25 + XL ($210) | $500-2.000+ |

> **Nota:** Custos sao estimativas baseadas em uso tipico. Workloads pesados (muitos Edge Functions, queries complexas, storage alto) podem aumentar significativamente. Vercel cobra por GB-hours, invocations, e bandwidth separadamente.

---

## 6. Padroes de Auto-Scaling

### 6.1 Horizontal vs Vertical Scaling

| Aspecto | Vertical (Scale Up) | Horizontal (Scale Out) |
|---------|---------------------|----------------------|
| **O que muda** | CPU/RAM da instancia | Numero de instancias |
| **Downtime** | Sim (restart necessario) | Nao (adiciona instancias) |
| **Limite** | Teto de hardware | Virtualmente ilimitado |
| **Complexidade** | Baixa | Alta (load balancer, sessoes, estado) |
| **Custo** | Linear/Exponencial | Linear |
| **Quando usar** | Database (primeiro recurso), cache | Application servers, APIs stateless |

**Decision Framework:**

```
O componente e stateless? (sem sessao, sem estado local)
  SIM -> Horizontal scaling (adicionar instancias)
  NAO -> Primeiro tentar tornar stateless, depois:
         - Se banco de dados -> Vertical + Read Replicas
         - Se cache -> Horizontal (Redis Cluster)
         - Se sessao -> Externalizar sessao (Redis), depois horizontal
```

> **Fonte:** [BlowStack - Horizontal vs Vertical Scaling in AWS](https://blowstack.com/blog/horizontal-scaling-and-vertical-scaling-in-aws)

### 6.2 Auto-Scaling Triggers

| Trigger | Metrica | Threshold Tipico | Melhor Para |
|---------|---------|------------------|-------------|
| CPU | `CPUUtilization` | > 70% (scale out), < 30% (scale in) | Application servers |
| Memoria | `MemoryUtilization` | > 80% | Aplicacoes memory-intensive |
| Request count | `RequestCountPerTarget` | > 1000 req/min/instancia | APIs com carga previsivel |
| Queue depth | `ApproximateNumberOfMessages` | > 100 mensagens | Workers de fila |
| Latencia | `TargetResponseTime` | p95 > 500ms | SLA-driven services |
| Custom | Metrica de negocio | Variavel | Logica especifica |

### 6.3 Vercel Auto-Scaling

Vercel e **auto-scaling by design** (serverless). Nao ha configuracao explicita:

- Funcoes escalam automaticamente com a demanda
- Limite de concurrent executions: 30K (Pro) / 100K+ (Enterprise)
- **Fluid Compute** (2025): Instancias "quentes" sao reutilizadas, reduzindo cold starts
- **Nao ha scale-to-zero delay perceptivel** para 99.37% das requisicoes
- **Controle**: Apenas via funcao `maxDuration` e selecao de regiao (single vs multi)

### 6.4 Supabase Scaling

| Acao de Scaling | Como | Downtime |
|----------------|------|----------|
| Upgrade de compute | Dashboard > Project > Compute | Minimo (< 2 min) |
| Read replicas | Dashboard > Database > Read Replicas | Zero |
| Connection pooling | Supavisor (automatico) | Zero |
| Point-in-Time Recovery | Disponivel em planos Pro+ | N/A |

### 6.5 AWS Auto-Scaling Patterns

#### EC2 Auto Scaling

```yaml
# Exemplo: AWS Auto Scaling Group com target tracking
AutoScalingGroup:
  MinSize: 2
  MaxSize: 20
  DesiredCapacity: 4
  TargetTrackingScaling:
    TargetValue: 70.0                    # CPU alvo de 70%
    PredefinedMetricSpecification:
      PredefinedMetricType: ASGAverageCPUUtilization
    ScaleInCooldown: 300                 # 5 min antes de scale-in
    ScaleOutCooldown: 60                 # 1 min antes de scale-out
```

#### Lambda Concurrency

```
Lambda scaling:
  - Burst concurrency: 500-3000 (varia por regiao) instantaneamente
  - Apos burst: +500 instancias/minuto
  - Reserved concurrency: Garantir minimo para funcao critica
  - Provisioned concurrency: Pre-aquecer instancias (elimina cold starts)
    - Auto Scaling via Application Auto Scaling: escala provisioned
      concurrency baseado em schedule ou utilization
```

> **Fonte:** [AWS Docs - Lambda Concurrency](https://docs.aws.amazon.com/lambda/latest/dg/lambda-concurrency.html), [OlioApps - Lambda Scaling](https://www.olioapps.com/blog/how-we-solved-aws-lambda-scaling-with-auto-scaling-and-provisioned-concurrency)

### 6.6 Database Scaling

| Estrategia | Complexidade | Quando Usar |
|-----------|-------------|-------------|
| **Connection Pooling** (PgBouncer/Supavisor) | Baixa | Primeiro recurso -- sempre |
| **Read Replicas** | Media | Leituras > 70% do workload |
| **Vertical Scaling** | Baixa | Antes de horizontal para DB |
| **Partitioning** | Media-Alta | Tabelas > 100M registros |
| **Sharding** | Alta | > 1TB de dados, multi-tenant global |
| **CQRS** | Alta | Workloads de leitura e escrita muito diferentes |

### 6.7 Caching como Scaling

Cache e a forma mais custo-efetiva de escalar. Cada cache hit e uma requisicao que NAO chega ao backend/banco.

| Layer | Ferramenta | Latencia | Quando Usar |
|-------|-----------|----------|-------------|
| **Browser** | Cache-Control headers | 0ms (local) | Assets estaticos, fontes |
| **CDN** | Vercel Edge, CloudFront | 5-50ms | HTML, imagens, APIs publicas |
| **ISR** | Next.js ISR | 0ms (cache hit) | Paginas semi-dinamicas |
| **Application** | Redis, Memcached | 1-2ms | Sessoes, resultados de query, rate limiting |
| **Database** | Query cache, pg_stat | 0-5ms overhead | Queries frequentes identicas |

**Redis como scaling layer:**

Redis reduz latencia de API de centenas de milissegundos para **1-2ms** -- uma melhoria de 100x+. Combinado com ISR do Next.js, cria multiplas camadas de cache que protegem o banco de dados de carga excessiva.

> **Fonte:** [DigitalApplied - Redis Caching Strategies Next.js](https://www.digitalapplied.com/blog/redis-caching-strategies-nextjs-production), [Dev.to - Scaling Next.js with Redis](https://dev.to/rafalsz/scaling-nextjs-with-redis-cache-handler-55lh)

---

## 7. Performance Budgets

### 7.1 Web Performance Budgets (Core Web Vitals 2025)

Google usa Core Web Vitals como fator de ranking. Apenas **47% dos sites** atendem aos thresholds atualmente.

| Metrica | Bom | Precisa Melhorar | Ruim | O que Mede |
|---------|-----|-----------------|------|-----------|
| **LCP** (Largest Contentful Paint) | <= 2.5s | 2.5-4.0s | > 4.0s | Velocidade de carregamento |
| **INP** (Interaction to Next Paint) | <= 200ms | 200-500ms | > 500ms | Responsividade (substituiu FID em marco 2024) |
| **CLS** (Cumulative Layout Shift) | <= 0.1 | 0.1-0.25 | > 0.25 | Estabilidade visual |
| **TTFB** (Time to First Byte) | <= 200ms | 200-500ms | > 500ms | Responsividade do servidor |

> **Fonte:** [aTeamSoftSolutions - Core Web Vitals 2025](https://www.ateamsoftsolutions.com/core-web-vitals-optimization-guide-2025-showing-lcp-inp-cls-metrics-and-performance-improvement-strategies-for-web-applications/), [Google Developers - Core Web Vitals](https://developers.google.com/search/docs/appearance/core-web-vitals)

### 7.2 API Performance Budgets

| Categoria | p50 Target | p95 Target | p99 Target |
|-----------|-----------|-----------|-----------|
| **Health check** | < 10ms | < 50ms | < 100ms |
| **CRUD simples** | < 50ms | < 200ms | < 500ms |
| **Busca com filtros** | < 100ms | < 500ms | < 1000ms |
| **Aggregacao/Report** | < 500ms | < 2000ms | < 5000ms |
| **Upload/processamento** | < 1000ms | < 5000ms | < 10000ms |

### 7.3 Database Query Performance Budgets

| Tipo de Query | Target | Alerta | Critico |
|--------------|--------|--------|---------|
| Lookup por PK/indice | < 5ms | > 20ms | > 100ms |
| Query filtrada (indexada) | < 50ms | > 200ms | > 500ms |
| JOIN simples (2-3 tabelas) | < 100ms | > 500ms | > 1000ms |
| Aggregacao (COUNT, SUM) | < 200ms | > 1000ms | > 3000ms |
| Full table scan | EVITAR | Sempre | Sempre |

### 7.4 Monitoramento de Budget Compliance em CI/CD

```yaml
# Exemplo: Lighthouse CI com budgets
# .lighthouserc.js
module.exports = {
  ci: {
    collect: {
      url: ['https://seu-app.vercel.app'],
      numberOfRuns: 3,
    },
    assert: {
      assertions: {
        'largest-contentful-paint': ['error', { maxNumericValue: 2500 }],
        'interactive': ['error', { maxNumericValue: 3800 }],
        'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }],
        'total-byte-weight': ['warning', { maxNumericValue: 500000 }], // 500KB
      },
    },
  },
};
```

```yaml
# GitHub Action para Lighthouse CI
name: Performance Budget
on: [pull_request]
jobs:
  lighthouse:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Lighthouse CI
        uses: treosh/lighthouse-ci-action@v12
        with:
          configPath: '.lighthouserc.js'
          uploadArtifacts: true
```

```javascript
// k6 thresholds como performance budgets
export const options = {
  thresholds: {
    // API budgets
    'http_req_duration{type:crud}': ['p(95)<200'],
    'http_req_duration{type:search}': ['p(95)<500'],
    'http_req_duration{type:report}': ['p(95)<2000'],
    // Error budget
    http_req_failed: ['rate<0.001'],  // 99.9% success rate (SLO)
    // Throughput budget
    http_reqs: ['rate>100'],           // Minimo 100 RPS
  },
};
```

---

## 8. Chaos Engineering

### 8.1 Principios

Chaos Engineering e a disciplina de experimentar em sistemas distribuidos para construir confianca na capacidade do sistema de resistir a condicoes turbulentas em producao. Nasceu na Netflix com o Chaos Monkey (2010) e evoluiu para um campo estruturado.

**Principios fundamentais (principlesofchaos.org):**

1. **Definir "estado estavel"**: Entender o comportamento normal do sistema antes de injetar falhas
2. **Hipotese sobre estado estavel**: "O sistema continuara operando normalmente quando X falhar"
3. **Variar eventos do mundo real**: Simular falhas que realmente acontecem (rede, disco, processo)
4. **Executar em producao**: Testar no ambiente que importa (apos praticar em staging)
5. **Automatizar para rodar continuamente**: Chaos como parte do pipeline, nao evento unico
6. **Minimizar blast radius**: Comecar pequeno, limitar impacto

> **Fonte:** [Principles of Chaos Engineering](https://principlesofchaos.org/)

### 8.2 Ferramentas

| Ferramenta | Tipo | Melhor Para | Preco |
|-----------|------|-------------|-------|
| **Chaos Monkey** | OSS (Netflix) | Terminar instancias aleatoriamente | Gratuito |
| **Gremlin** | SaaS (comercial) | Multiplos tipos de ataque, enterprise | Pago |
| **LitmusChaos** | OSS (CNCF Sandbox) | Kubernetes-native, cloud-native | Gratuito |
| **AWS FIS** | Managed (AWS) | Infra AWS, cenarios pre-definidos | Pay-per-use |
| **Chaos Mesh** | OSS (CNCF) | Kubernetes, granularidade fina | Gratuito |

**Detalhes:**

- **Chaos Monkey**: Pioneiro historico. Limitacoes: poucos tipos de ataque, requer Spinnaker, abordagem aleatoria sem controle fino
- **Gremlin**: Primeiro SaaS comercial de chaos. Tipos de ataque: CPU, memoria, disco, rede, processo, DNS, estado. Dashboard completo, guardrails de seguranca
- **LitmusChaos**: Open source, CNCF Sandbox. Workflows CRD para Kubernetes. ChaosHub com experimentos pre-construidos. Integracao com Argo para GitOps
- **AWS FIS**: Gerenciado pela AWS. Cenarios pre-definidos (AZ failure, cross-region connectivity). Integracao com CloudWatch Alarms como stop conditions. Cenarios Library para bootstrap rapido

> **Fonte:** [Harness - Top Chaos Engineering Tools](https://www.harness.io/blog/chaos-engineering-tools), [Steadybit - Chaos Engineering Tools 2025 Guide](https://steadybit.com/blog/top-chaos-engineering-tools-worth-knowing-about-2025-guide/), [Gremlin - Tools Comparison](https://www.gremlin.com/community/tutorials/chaos-engineering-tools-comparison)

### 8.3 GameDay Exercises

Um GameDay e um exercicio planejado onde a equipe simula falhas e pratica resposta a incidentes.

**Estrutura de um GameDay:**

```
ANTES (Planejamento):
  1. Definir objetivo claro: "Testar resposta quando DB primario cai"
  2. Identificar participantes: engenheiros, SRE, PM
  3. Definir guardrails: stop conditions, rollback plan
  4. Preparar runbooks e dashboards
  5. Notificar stakeholders

DURANTE (Execucao):
  1. Comecar em ambiente de staging (primeira vez)
  2. Injetar falha conforme cenario planejado
  3. Observar: alarmes dispararam? Time notado? Em quanto tempo?
  4. Executar resposta: time segue runbook? Funciona?
  5. Escalar se necessario (blast radius controlado)

DEPOIS (Retrospectiva):
  1. O que funcionou? O que nao funcionou?
  2. Alarmes foram uteis? Faltou algum?
  3. Runbook estava atualizado? Passos faltando?
  4. Quanto tempo levou para detectar e resolver?
  5. Documentar melhorias e tracking de acoes
```

> **GameDay e progressivo:** Comece em dev/staging, evolua para pre-production, so depois producao. Com pratica, tempo de resposta cai de horas para minutos.
> **Fonte:** [AWS Blog - FIS Team Game Days](https://aws.amazon.com/blogs/mt/learn-from-aws-fault-injection-service-team-approach-to-game-days/), [AWS Well-Architected - Game Days](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_testing_resiliency_game_days_resiliency.html)

### 8.4 Failure Injection Patterns

| Pattern | O que Simula | Ferramenta Exemplo |
|---------|-------------|-------------------|
| **Process kill** | Crash de servico | Chaos Monkey, Gremlin |
| **Network latency** | Degradacao de rede | tc/netem, Gremlin, Chaos Mesh |
| **Packet loss** | Perda de pacotes | tc/netem, AWS FIS |
| **DNS failure** | Falha de resolucao DNS | Gremlin, LitmusChaos |
| **Disk fill** | Disco cheio | Gremlin, stress-ng |
| **CPU stress** | Saturacao de CPU | stress-ng, Gremlin |
| **AZ failure** | Queda de Availability Zone | AWS FIS |
| **Dependency timeout** | Servico externo lento/indisponivel | Toxiproxy, Gremlin |
| **Clock skew** | Dessincronia de relogio | Chaos Mesh |

### 8.5 Modelo de Maturidade

| Nivel | Descricao | Pre-requisitos | Acoes |
|-------|-----------|----------------|-------|
| **0 - Nenhum** | Sem pratica de chaos | N/A | Comecar com observabilidade |
| **1 - Observabilidade** | Monitoring e alertas funcionais | Logs, metricas, dashboards | Implementar APM, definir SLOs |
| **2 - Manual** | GameDays manuais em staging | Observabilidade funcional, runbooks | GameDays trimestrais em staging |
| **3 - Automatizado** | Chaos automatizado em staging | GameDays regulares, confianca do time | LitmusChaos/Gremlin em staging CI |
| **4 - Producao** | Chaos automatizado em producao | Meses de pratica em staging | Chaos em prod com guardrails |
| **5 - Continuo** | Chaos como parte do pipeline | Cultura de resiliencia | Chaos em cada deploy |

> **Quando comecar:** Quando voce tem observabilidade funcional (alarmes que funcionam, dashboards que mostram estado real) e pelo menos runbooks basicos. Nao comece chaos sem saber como detectar que algo deu errado.
> **Fonte:** [O'Reilly - Chaos Maturity Model](https://www.oreilly.com/library/view/chaos-engineering/9781491988459/ch09.html), [Harness - Chaos Engineering Maturity Model](https://www.harness.io/resources/the-chaos-engineering-maturity-model)

---

## 9. Benchmarks do Mundo Real

### 9.1 Vercel Serverless Performance (2025)

| Metrica | Edge Runtime | Serverless (Warm) | Serverless (Cold) |
|---------|-------------|-------------------|-------------------|
| P50 Latencia | ~106ms | ~246ms | ~859ms |
| Cold Start | < 50ms | 100ms-1s | 100ms-1s+ |
| Multi-Region P50 | 20-50ms (enterprise) | N/A | N/A |
| Single Region P50 | 10-30ms (perto) | ~200ms | ~800ms |

- Fluid Compute elimina cold starts para 99.37% dos requests
- Edge Functions sao ~9x mais rapidas em cold start que serverless tradicional
- Vercel unificou Edge Middleware e Edge Functions sob "Vercel Functions" em mid-2025

> **Fonte:** [OpenStatus - Vercel Edge vs Serverless Latency](https://www.openstatus.dev/blog/monitoring-latency-vercel-edge-vs-serverless), [ByteIota - Edge Functions vs Serverless 2025](https://byteiota.com/edge-functions-vs-serverless-the-2025-performance-battle/)

### 9.2 Supabase Performance

| Componente | Benchmark | Compute Size |
|-----------|-----------|-------------|
| Realtime concurrent connections | 10.000+ sem degradacao | Variavel |
| PostgREST API | Limitado por pool size e DB | Proporcional ao compute |
| Auth | Rate limited por design | Configuravel |
| Storage | CDN-backed, fast reads | Depende de CDN |

**RLS impact:** Cada subscriber verificado individualmente. 100 subscribers x 1 INSERT = 100 DB reads para verificacao de permissao.

> **Fonte:** [Supabase Docs - Realtime Benchmarks](https://supabase.com/docs/guides/realtime/benchmarks), [Supabase Docs - Performance Tuning](https://supabase.com/docs/guides/platform/performance)

### 9.3 PostgreSQL Benchmarks

| Configuracao | Clients | TPS | Latencia Media |
|-------------|---------|-----|----------------|
| pgbench default, 30 min | 400 | ~4.487 | 89ms |
| Read-only, 10 min | 100 | ~15.000+ | < 10ms |
| Custom workload (OLTP) | 200 | ~8.000-12.000 | 20-50ms |

> **Nota:** Resultados variam drasticamente com hardware, configuracao, e complexidade de queries. pgbench e um benchmark sintetico -- queries reais podem ser muito diferentes.
> **Fonte:** [PostgreSQL Docs - pgbench](https://www.postgresql.org/docs/current/pgbench.html), [Severalnines - Benchmarking PostgreSQL](https://severalnines.com/blog/benchmarking-postgresql-performance/)

### 9.4 Cost vs Performance por Escala

| Escala | Infra Tipica | Custo/Mes | RPS Suportado | Latencia p95 |
|--------|-------------|-----------|--------------|-------------|
| **1K MAU** | Vercel Hobby + Supabase Free | $0 | ~1 RPS | < 300ms |
| **10K MAU** | Vercel Pro + Supabase Pro (Micro) | ~$45 | ~5 RPS | < 300ms |
| **100K MAU** | Vercel Pro + Supabase Pro (Medium) | ~$200 | ~50 RPS | < 500ms |
| **1M MAU** | Vercel Enterprise + Supabase XL+ | $500-2.000 | ~500 RPS | < 500ms |
| **10M MAU** | Custom infra (AWS/GCP) + CDN global | $5.000-50.000 | ~5.000 RPS | < 300ms |

> **Trade-off principal:** Vercel+Supabase sao otimos para developer experience e tempo-to-market, mas o custo por request sobe com escala. Apos ~500K MAU, vale investigar infra propria ou hibrida.

---

## 10. Decision Framework: MVP vs Enterprise Scale

### 10.1 Quando Vercel + Supabase Basta

Vercel + Supabase e suficiente quando:

- MAU < 100.000 (com uso tipico)
- Peak RPS < 50
- Database < 10GB
- Equipe < 10 devs
- Nao requer compliance especifico (HIPAA, SOC2 nivel avancado)
- Workload e predominantemente leitura (> 80%)
- Nao precisa de background jobs complexos (filas longas, ML inference)

### 10.2 Sinais de que Voce Precisa Mais Infra

| Sinal | Evidencia | Acao |
|-------|-----------|------|
| Connection pool exhaustion | Erros 503, "too many connections" | Upgrade compute Supabase ou adicionar read replicas |
| Cold start impactando UX | p95 > 2s em funcoes criticas | Migrar para Fluid Compute ou Provisioned Concurrency |
| Custo de bandwidth > budget | Bill Vercel subindo exponencialmente | Avaliar CDN dedicado (CloudFront, Cloudflare) |
| Background jobs falhando | Timeouts de 300s nao sao suficientes | Adicionar queue (BullMQ, AWS SQS) + workers dedicados |
| Latencia global > 500ms | Usuarios em regioes distantes reclamando | Multi-region (Enterprise Vercel) ou edge deployment |
| Database > 50GB | Queries ficando lentas, storage caro | Partitioning, archiving, ou migration |
| Compliance obrigatorio | HIPAA, PCI-DSS, dados sensiveis | Enterprise plans ou infra propria com audit trail |

### 10.3 Scaling Roadmap

```
STAGE 1: MVP (0 - 10K MAU)
==========================================
Infra:   Vercel Free/Pro + Supabase Free/Pro
Custo:   $0 - $45/mes
Load:    < 5 RPS
Testing: Smoke test manual com k6
Scaling: Nenhum -- tudo automatico
Focus:   Product-market fit, velocidade de iteracao

         |
         v

STAGE 2: GROWTH (10K - 100K MAU)
==========================================
Infra:   Vercel Pro + Supabase Pro (Small/Medium)
Custo:   $75 - $200/mes
Load:    5 - 50 RPS
Testing: k6 no CI/CD (load test semanal)
Scaling: Upgrade compute Supabase conforme necessidade
         Redis cache para queries frequentes
         CDN otimizado (cache headers)
Focus:   Performance, monitoring, alertas basicos
Adicionar: APM (Sentry, Datadog), error tracking

         |
         v

STAGE 3: SCALE (100K - 1M MAU)
==========================================
Infra:   Vercel Pro/Enterprise + Supabase Large/XL
         + Redis (Upstash ou dedicado)
         + Queue system (BullMQ/SQS)
Custo:   $500 - $2.000/mes
Load:    50 - 500 RPS
Testing: k6 no CI/CD (load test em cada PR)
         Soak tests mensais
         Chaos engineering em staging
Scaling: Read replicas Supabase
         Multi-region (se global)
         Background workers dedicados
Focus:   Resiliencia, observabilidade completa, SLOs
Adicionar: Grafana stack, runbooks, on-call rotation

         |
         v

STAGE 4: ENTERPRISE (1M+ MAU)
==========================================
Infra:   Hibrido: Vercel (frontend) + AWS/GCP (backend)
         + PostgreSQL gerenciado (RDS/Cloud SQL)
         + Redis Cluster
         + CDN dedicado (CloudFront/Cloudflare)
         + Message queue (SQS/Kafka)
         + Container orchestration (ECS/K8s)
Custo:   $5.000 - $50.000+/mes
Load:    500+ RPS
Testing: Load testing continuo
         Chaos engineering em producao
         GameDays trimestrais
         Performance regression em cada deploy
Scaling: Auto-scaling horizontal (application)
         Database sharding/CQRS
         Edge computing global
         Multi-AZ/Multi-region
Focus:   Disponibilidade (99.99%), compliance, custo/req
Adicionar: SRE team, incident management, DR plan
```

### 10.4 Budget Allocation por Stage

| Stage | Infra (%) | Monitoring (%) | Testing (%) | Pessoas (%) |
|-------|-----------|---------------|-------------|-------------|
| MVP | 10% | 5% | 5% | 80% |
| Growth | 20% | 10% | 10% | 60% |
| Scale | 30% | 15% | 15% | 40% |
| Enterprise | 35% | 20% | 15% | 30% |

> **Nota:** "Pessoas" inclui tempo de engenharia. No MVP, 80% do orcamento vai para construir features. Na Enterprise, a infra e operational excellence consomem a maior parte.

### 10.5 Checklist de Load Testing por Stage

#### MVP (obrigatorio antes de lancar)
- [ ] Smoke test basico (k6, 1-5 VUs, 1 min)
- [ ] Verificar cold starts aceitaveis (< 3s)
- [ ] Testar endpoints criticos (auth, main flow)
- [ ] Verificar rate limits da plataforma

#### Growth (obrigatorio antes de cada release major)
- [ ] Load test com carga esperada (k6, baseline + 2x)
- [ ] Stress test ate 3x da carga esperada
- [ ] Monitorar connection pool usage
- [ ] Verificar cache hit ratios
- [ ] k6 no CI/CD com thresholds de SLO

#### Scale (continuo)
- [ ] Load test em cada PR (smoke no CI)
- [ ] Load test semanal completo (load + stress)
- [ ] Soak test mensal (4-8h)
- [ ] Chaos engineering em staging (mensal)
- [ ] Database benchmark trimestral
- [ ] Performance budget enforcement no CI

#### Enterprise (continuo + proativo)
- [ ] Tudo acima, mais:
- [ ] Chaos engineering em producao (semanal)
- [ ] GameDay trimestral
- [ ] Capacity planning revisado mensalmente
- [ ] Multi-region failover testing
- [ ] Disaster recovery drill semestral

---

## Fontes

### Ferramentas e Documentacao Oficial
- [Grafana k6](https://k6.io/) | [GitHub](https://github.com/grafana/k6)
- [Artillery.io](https://www.artillery.io/) | [GitHub](https://github.com/artilleryio/artillery)
- [Locust](https://locust.io/) | [GitHub](https://github.com/locustio/locust)
- [wrk2](https://github.com/giltene/wrk2) | [wrk](https://github.com/wg/wrk)
- [PostgreSQL pgbench](https://www.postgresql.org/docs/current/pgbench.html)

### Plataformas
- [Vercel Docs - Fluid Compute](https://vercel.com/docs/fluid-compute)
- [Vercel - Load Testing Policy](https://vercel.com/kb/guide/what-s-vercel-s-policy-regarding-load-testing-deployments)
- [Vercel Pricing](https://vercel.com/pricing)
- [Supabase Docs - Compute and Disk](https://supabase.com/docs/guides/platform/compute-and-disk)
- [Supabase Docs - Connection Management](https://supabase.com/docs/guides/database/connection-management)
- [Supabase Docs - Realtime Limits](https://supabase.com/docs/guides/realtime/limits)
- [Supabase Docs - Realtime Benchmarks](https://supabase.com/docs/guides/realtime/benchmarks)

### Metodologia e Melhores Praticas
- [Grafana Labs - Types of Load Testing](https://grafana.com/load-testing/types-of-load-testing/)
- [Locust Cloud - 5 Essential Load Test Profiles](https://www.locust.cloud/blog/5-essential-load-test-profiles/)
- [TestGuild - Load Testing Best Practices](https://testguild.com/best-load-testing/)
- [Expert-Soft - Load Testing Best Practices 2025](https://expert-soft.com/blog/load-testing-best-practices/)
- [Principles of Chaos Engineering](https://principlesofchaos.org/)
- [ByteByteGo - Capacity Planning](https://blog.bytebytego.com/p/capacity-planning)
- [Kir Shatrov - Capacity Planning for Web Apps](https://kirshatrov.com/posts/capacity-planning-for-web-apps/)

### Performance e Benchmarks
- [OpenStatus - Vercel Edge vs Serverless](https://www.openstatus.dev/blog/monitoring-latency-vercel-edge-vs-serverless)
- [Vercel Blog - Fluid Compute](https://vercel.com/blog/scale-to-one-how-fluid-solves-cold-starts)
- [Google Developers - Core Web Vitals](https://developers.google.com/search/docs/appearance/core-web-vitals)
- [aTeamSoftSolutions - Core Web Vitals 2025](https://www.ateamsoftsolutions.com/core-web-vitals-optimization-guide-2025-showing-lcp-inp-cls-metrics-and-performance-improvement-strategies-for-web-applications/)

### Chaos Engineering
- [O'Reilly - Chaos Maturity Model](https://www.oreilly.com/library/view/chaos-engineering/9781491988459/ch09.html)
- [Harness - Chaos Engineering Maturity Model](https://www.harness.io/resources/the-chaos-engineering-maturity-model)
- [Harness - Top Chaos Engineering Tools](https://www.harness.io/blog/chaos-engineering-tools)
- [Gremlin - Tools Comparison](https://www.gremlin.com/community/tutorials/chaos-engineering-tools-comparison)
- [Steadybit - Chaos Engineering Tools 2025](https://steadybit.com/blog/top-chaos-engineering-tools-worth-knowing-about-2025-guide/)
- [AWS - FIS GameDays](https://aws.amazon.com/blogs/mt/learn-from-aws-fault-injection-service-team-approach-to-game-days/)
- [AWS Well-Architected - Game Days](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_testing_resiliency_game_days_resiliency.html)

### Scaling e Caching
- [DigitalApplied - Redis Caching Next.js](https://www.digitalapplied.com/blog/redis-caching-strategies-nextjs-production)
- [AWS Docs - Lambda Concurrency](https://docs.aws.amazon.com/lambda/latest/dg/lambda-concurrency.html)
- [BlowStack - Horizontal vs Vertical Scaling AWS](https://blowstack.com/blog/horizontal-scaling-and-vertical-scaling-in-aws)
- [Dev.to - Scaling Next.js with Redis](https://dev.to/rafalsz/scaling-nextjs-with-redis-cache-handler-55lh)

### Comparativos
- [Vervali - Best Load Testing Tools 2026](https://www.vervali.com/blog/best-load-testing-tools-in-2026-definitive-guide-to-jmeter-gatling-k6-loadrunner-locust-blazemeter-neoload-artillery-and-more/)
- [BlazeMeter - Gatling vs JMeter](https://www.blazemeter.com/blog/gatling-vs-jmeter)
- [BrowserStack - JMeter vs Gatling](https://www.browserstack.com/guide/jmeter-vs-gatling)
- [TestLeaf - Top 5 Load Testing Tools 2025](https://www.testleaf.com/blog/5-best-load-testing-tools-in-2025/)

---

> **Pesquisa conduzida por:** Prism (Research Operations Conductor) | squad-research
> **Nivel de profundidade:** DEEP DIVE (Research Depth Pyramid Level 3)
> **Fontes consultadas:** 40+ fontes, Tiers 2-4
> **Verificacao:** Todas as afirmacoes factuais verificadas via WebSearch (abril 2026)
