# AWS Cloud Infrastructure

> **Deep Research** -- Servicos essenciais, padroes de arquitetura e quando escalar alem do Vercel/Supabase.
> **Nivel:** DEEP DIVE | **Fontes:** 40+ (Tier 2-4) | **Verificado via WebSearch:** Sim

---

## Indice

1. [Servicos Core da AWS para Web Apps](#1-servicos-core-da-aws-para-web-apps)
2. [Padroes de Arquitetura Serverless](#2-padroes-de-arquitetura-serverless)
3. [Padroes de Auto-Scaling](#3-padroes-de-auto-scaling)
4. [Melhores Praticas de Seguranca AWS](#4-melhores-praticas-de-seguranca-aws)
5. [Estrategia Multi-Account](#5-estrategia-multi-account)
6. [Otimizacao de Custos](#6-otimizacao-de-custos)
7. [Quando Graduar do Vercel/Supabase](#7-quando-graduar-do-vercelsupabase)
8. [AWS Well-Architected Framework](#8-aws-well-architected-framework)

---

## 1. Servicos Core da AWS para Web Apps

### 1.1 Compute: Lambda vs ECS/Fargate vs EC2 vs App Runner

A escolha do servico de compute correto e uma das decisoes mais impactantes na arquitetura AWS. Cada servico atende a um perfil de workload diferente.

#### Tabela Comparativa de Compute

| Caracteristica | Lambda | ECS/Fargate | EC2 | App Runner |
|---------------|--------|-------------|-----|------------|
| **Modelo** | Serverless functions | Serverless containers | Virtual machines | Managed containers |
| **Max Execution** | 15 min | Ilimitado | Ilimitado | Ilimitado |
| **Cold Start** | 200ms-15s (sem otimizacao) | 30-60s (task startup) | N/A (always running) | Pausa/resume rapido |
| **Scaling** | Automatico (ms) | Automatico (minutos) | Auto Scaling Groups | Automatico |
| **Networking** | VPC opcional | VPC nativo (ENI por task) | VPC nativo | VPC opcional (public por default) |
| **Complexidade** | Baixa | Media-Alta | Alta | Baixa |
| **Preco Modelo** | Pay per invocation + duration | Pay per vCPU/memory/hora | Pay per instance/hora | Pay per vCPU/memory (pausa = so memory) |
| **Container Support** | Container images ate 10GB | Docker nativo | Docker manual | Docker nativo |

**Fontes:** [Thoughtful Architect](https://www.thoughtfularchitect.dev/posts/aws-compute-comparison), [AWS Decision Guide](https://docs.aws.amazon.com/decision-guides/latest/fargate-or-lambda/fargate-or-lambda.html), [cloudonaut](https://cloudonaut.io/fargate-vs-apprunner/)

#### Quando Usar Cada Servico

**AWS Lambda -- Ideal para:**
- Event-driven workloads (triggers de S3, DynamoDB Streams, SQS)
- APIs com trafego variavel ou bursty
- Microservicos stateless com execucao < 15 min
- Processamento assincrono (imagens, PDFs, webhooks)
- Workloads com periodos longos de inatividade

**ECS/Fargate -- Ideal para:**
- Aplicacoes containerizadas com execucao longa
- Microservicos que precisam de controle granular de recursos
- Workloads com trafego sustentado e previsivel
- ML inference e data processing
- Quando voce precisa de networking VPC completo

**EC2 -- Ideal para:**
- Aplicacoes legacy que precisam de controle total do OS
- Workloads com requisitos especificos de hardware (GPU, alta memoria)
- Quando voce precisa de instancias dedicadas ou bare metal
- Licenciamento de software que exige instancias fixas

**App Runner -- Ideal para:**
- Web apps simples e APIs HTTP sincronas
- Equipes que querem zero infraestrutura para gerenciar
- MVPs e prototipos rapidos
- Quando simplicidade e prioridade sobre controle
- Aplicacoes com trafego baixo/intermitente (scale-to-zero parcial: pausa container, paga so memoria)

**Fonte:** [Medium - Dashanka De Silva](https://dashankadesilva.medium.com/aws-app-runner-vs-ecs-vs-lambda-choosing-the-right-compute-option-36915d355cd5)

#### Decision Framework para Compute

```
Pergunta 1: Execucao dura mais de 15 minutos?
  SIM -> Container (ECS/Fargate) ou EC2
  NAO -> Continuar

Pergunta 2: Workload e event-driven ou HTTP API leve?
  SIM -> Lambda
  NAO -> Continuar

Pergunta 3: Precisa de controle de networking (VPC, security groups)?
  SIM -> ECS/Fargate
  NAO -> Continuar

Pergunta 4: Equipe tem experiencia com containers?
  SIM -> ECS/Fargate
  NAO -> App Runner

Pergunta 5: Precisa de controle total do OS?
  SIM -> EC2
  NAO -> ECS/Fargate
```

---

### 1.2 Storage: S3 vs EFS vs EBS

#### Tabela Comparativa de Storage

| Caracteristica | S3 | EBS | EFS |
|---------------|-----|-----|-----|
| **Tipo** | Object storage | Block storage | File storage (NFS) |
| **Acesso** | Via API (HTTP) | Single EC2 instance (Multi-Attach limitado) | Multiplas instancias simultaneas |
| **Durabilidade** | 99.999999999% (11 nines) | 99.999% | 99.999999999% |
| **Escalabilidade** | Ilimitada | Ate 64 TiB por volume | Elastico (petabytes) |
| **Latencia** | ~100ms (first byte) | Sub-millisecond | ~ms (varia por modo) |
| **Custo (us-east-1)** | $0.023/GB/mes (Standard) | $0.08/GB/mes (gp3) | $0.30/GB/mes (Standard) |
| **Use Case Principal** | Objetos, backups, static files, data lakes | Databases, boot volumes, IOPS intensivo | Shared file systems, CMS, containers |

**Fontes:** [SquareOps 2025](https://squareops.com/knowledge/aws-s3-vs-ebs-vs-efs-vs-glacier-which-storage-is-best-in-2025/), [NetApp](https://www.netapp.com/blog/ebs-efs-amazons3-best-cloud-storage-system/)

#### Classes de Storage do S3

| Classe | Custo/GB/mes | Acesso | Use Case |
|--------|-------------|--------|----------|
| S3 Standard | $0.023 | Frequente | Dados ativos, websites |
| S3 Intelligent-Tiering | $0.023 + monitoring fee | Automatico | Padroes de acesso imprevisiveis |
| S3 Standard-IA | $0.0125 | Infrequente | Backups, disaster recovery |
| S3 One Zone-IA | $0.01 | Infrequente, single AZ | Dados reproduziveis |
| S3 Glacier Instant | $0.004 | Raro, ms retrieval | Arquivos com acesso instantaneo |
| S3 Glacier Flexible | $0.0036 | Raro, min-hrs retrieval | Long-term archive |
| S3 Glacier Deep Archive | $0.00099 | Raríssimo, 12h retrieval | Compliance, 7-10 anos |

**Regra de ouro:** Escolha EBS para velocidade, EFS para acesso compartilhado e S3 para escala e custo-eficiencia.

**Fonte:** [CloudOptimo](https://www.cloudoptimo.com/blog/choosing-the-right-aws-storage-ebs-vs-efs-vs-s3-explained/)

---

### 1.3 Database: RDS vs DynamoDB vs Aurora vs ElastiCache

#### Tabela Comparativa de Databases

| Caracteristica | RDS | Aurora | DynamoDB | ElastiCache |
|---------------|-----|--------|----------|-------------|
| **Tipo** | Relacional (managed) | Relacional (cloud-native) | NoSQL (key-value/document) | In-memory cache |
| **Engines** | PostgreSQL, MySQL, MariaDB, Oracle, SQL Server | MySQL, PostgreSQL | Proprietario | Redis, Memcached |
| **Performance** | Padrao da engine | 3x MySQL, 5x PostgreSQL | Single-digit ms | 300-500 microseconds |
| **Scaling** | Vertical + Read Replicas | Auto-scaling replicas | Auto-scaling ilimitado (ate 10M req/s) | Vertical + cluster mode |
| **Storage Max** | 64 TiB | 128 TiB (auto-grow) | Ilimitado | Ate 500 nodes |
| **Serverless** | Nao | Aurora Serverless v2 | On-demand mode | Serverless cache (2023+) |
| **Multi-Region** | Manual | Global Database | Global Tables | Global Datastore |
| **Custo Inicial** | ~$13/mes (db.t3.micro) | ~$29/mes (minimo) | Pay per request ou provisioned | ~$13/mes (cache.t3.micro) |

**Fontes:** [Bytebase - RDS vs DynamoDB](https://www.bytebase.com/blog/rds-vs-dynamodb/), [Lushbinary - Aurora vs RDS 2026](https://lushbinary.com/blog/aws-aurora-vs-rds-cost-performance-architecture-guide-2026/), [Dynobase](https://dynobase.dev/dynamodb-vs-elasticache/)

#### Aurora DSQL (Novidade 2025)

Aurora DSQL entrou em GA em meados de 2025 como um banco SQL distribuido totalmente serverless com compatibilidade PostgreSQL. Projetado para aplicacoes que precisam de deployments multi-region active-active com consistencia forte.

**Fonte:** [DasRoot.net](https://dasroot.net/posts/2026/01/aws-database-services-rds-aurora-dynamodb-comparison/)

#### Decision Framework para Database

```
Dados relacionais com transacoes ACID?
  SIM -> RDS ou Aurora
    Budget limitado? -> RDS PostgreSQL
    Performance critica? -> Aurora
    Multi-region active-active? -> Aurora DSQL ou DynamoDB Global Tables
  NAO -> Continuar

Dados com schema flexivel, acesso por key?
  SIM -> DynamoDB
    Acesso previsivel? -> Provisioned capacity
    Acesso imprevisivel? -> On-demand

Precisa de cache para reduzir latencia?
  SIM -> ElastiCache Redis
    Session store? -> Redis
    Cache de queries? -> Redis
    Leaderboards/contadores? -> Redis
```

#### Database Savings Plans (2025)

A AWS lancou Database Savings Plans, um modelo de desconto baseado em commitment que se aplica a Aurora, RDS, DynamoDB, ElastiCache, DocumentDB, Neptune, Keyspaces, Timestream e DMS.

**Fonte:** [K21Academy](https://k21academy.com/aws-cloud/aws-database-service-amazon-rds-aurora-dynamodb-elasticache/)

---

### 1.4 Networking: VPC, Subnets, Security Groups, NACLs, ALB/NLB

#### Arquitetura de Rede Recomendada

```
VPC (10.0.0.0/16)
|
+-- Public Subnet (10.0.1.0/24) -- AZ-a
|   +-- ALB / NLB
|   +-- NAT Gateway
|   +-- Bastion Host (se necessario)
|
+-- Public Subnet (10.0.2.0/24) -- AZ-b
|   +-- ALB / NLB (multi-AZ)
|   +-- NAT Gateway (redundancia)
|
+-- Private Subnet (10.0.10.0/24) -- AZ-a
|   +-- Application servers (ECS tasks, EC2)
|   +-- Lambda (VPC-attached)
|
+-- Private Subnet (10.0.11.0/24) -- AZ-b
|   +-- Application servers (multi-AZ)
|
+-- Private Subnet (10.0.20.0/24) -- AZ-a
|   +-- Database (RDS primary)
|
+-- Private Subnet (10.0.21.0/24) -- AZ-b
    +-- Database (RDS standby)
```

**Fonte:** [InventiveHQ](https://inventivehq.com/knowledge-base/aws/aws-vpc-network-segmentation)

#### Security Groups vs NACLs

| Caracteristica | Security Groups | NACLs |
|---------------|----------------|-------|
| **Nivel** | Instance/ENI level | Subnet level |
| **Stateful?** | Sim (return traffic automatico) | Nao (regras explicitas in/out) |
| **Regras** | Apenas ALLOW | ALLOW e DENY |
| **Avaliacao** | Todas as regras juntas | Ordem numerica (primeira match) |
| **Default** | Deny tudo (inbound), Allow tudo (outbound) | Allow tudo (default NACL) |
| **Uso** | Controle primario de acesso | Camada adicional de protecao |

**Pratica recomendada:** Use Security Groups como controle primario. NACLs como camada extra para bloqueio subnet-level. Principio de defense in depth.

**Fontes:** [Jay Tillu - Medium](https://jaytillu.medium.com/understanding-amazon-vpc-security-with-subnets-nacl-and-security-groups-f2baf2f21efc), [AWS DDoS Best Practices](https://docs.aws.amazon.com/whitepapers/latest/aws-best-practices-ddos-resiliency/security-groups-and-network-acls-bp5.html)

#### ALB vs NLB

| Caracteristica | ALB (Application) | NLB (Network) |
|---------------|-------------------|---------------|
| **Camada OSI** | Layer 7 (HTTP/HTTPS) | Layer 4 (TCP/UDP/TLS) |
| **Routing** | Path-based, host-based, header-based | IP/Port based |
| **WebSocket** | Sim | Sim |
| **Performance** | Maior latencia (~ms) | Ultra-baixa latencia (~100us) |
| **Static IP** | Nao (DNS only) | Sim (Elastic IP por AZ) |
| **WAF** | Sim (integra com AWS WAF) | Nao |
| **Security Groups** | Sim (sempre) | Sim (desde agosto 2023) |
| **Use Case** | APIs REST, web apps, microservicos HTTP | gRPC, IoT, gaming, alta performance |

**Fontes:** [Anyshift - VPC Deep Dive](https://www.anyshift.io/blog/a-deep-dive-in-aws-resources-best-practices-to-adopt-vpc-networking), [Rearc - NLBs Get SGs](https://rearc.io/blog/nlbs-get-sgs)

---

### 1.5 CDN: CloudFront vs Cloudflare

| Caracteristica | CloudFront | Cloudflare |
|---------------|-----------|------------|
| **PoPs** | 450+ em 90 cidades | 335+ cidades em 125+ paises |
| **TTFB Medio** | ~42ms | ~38ms (27ms com Argo) |
| **DDoS** | Shield Standard (gratis) | Gratis em todos os planos |
| **WAF** | Pago (AWS WAF separado) | Incluso em planos pagos |
| **Edge Compute** | Lambda@Edge + CloudFront Functions | Cloudflare Workers |
| **Setup** | Requer expertise AWS | Troca de DNS nameservers |
| **Pricing** | $0.085-$0.170/GB (varia por regiao) | Bandwidth ilimitada em planos fixos |
| **Bot Management** | AWS WAF Bot Control (pago) | Incluso (ML-based) |
| **Free Tier** | 1 TB/mes (12 meses) | Plano gratis permanente |
| **Integracao AWS** | Nativa (S3, ALB, API Gateway) | Via configuracao manual |

**Fontes:** [CrawlWP 2026](https://crawlwp.com/aws-cloudfront-vs-cloudflare/), [CloudOptimo 2025](https://www.cloudoptimo.com/blog/cloudfront-vs-cloudflare-vs-akamai-choosing-the-right-cdn-in-2025/), [Vigilbase](https://vigilbase.com/cloudflare/vs/aws-cloudfront)

#### Pricing CloudFront (2025 Update)

No final de 2025, a AWS introduziu planos de pricing flat-rate para CloudFront ($0 a $1,000/mes) sem cobranças excedentes, tornando o modelo mais previsivel.

**Quando usar CloudFront:** Deep integration com AWS (S3 origins, Lambda@Edge, ALB).
**Quando usar Cloudflare:** Infraestrutura nao-AWS, orcamento limitado, simplicidade maxima, DDoS gratis.

---

### 1.6 DNS: Route 53

Amazon Route 53 e o servico de DNS gerenciado da AWS, altamente disponivel e escalavel.

#### Pricing Route 53

| Recurso | Custo |
|---------|-------|
| Hosted Zone | $0.50/zona/mes (primeiras 25) |
| Queries Standard | $0.40 por milhao (ate 1B queries) |
| Queries Latency/Geo | $0.60-$0.70 por milhao |
| Health Checks (AWS endpoints) | Gratis (ate 50) |
| Health Checks (non-AWS) | $0.50-$0.75 por health check/mes |
| Domain Registration | Varia ($12-$35/ano para .com) |

**Fonte:** [AWS Route 53 Pricing](https://aws.amazon.com/route53/pricing/), [Pump.co Guide](https://www.pump.co/blog/aws-route-53)

#### Routing Policies

| Policy | Use Case |
|--------|----------|
| **Simple** | Single resource, sem health check |
| **Weighted** | A/B testing, blue-green deployment (% de trafego) |
| **Latency-based** | Multi-region -- roteia para a regiao de menor latencia |
| **Geolocation** | Roteia por localizacao do usuario (pais, continente) |
| **Geoproximity** | Roteia por proximidade geografica com bias ajustavel |
| **Failover** | Active-passive -- failover automatico para backup |
| **Multi-value** | Load distribution com health checks (ate 8 records) |

**Destaque:** Alias records para servicos AWS (ALB, CloudFront, S3, API Gateway, etc.) sao gratuitos -- sem cobranca por query.

**Fonte:** [Builder.aws 2025 Guide](https://builder.aws.com/content/35T6YBNG2ywpuwZurZPnAZaJBY4/aws-route-53-the-complete-guide-2025-edition)

---

## 2. Padroes de Arquitetura Serverless

### 2.1 API Gateway + Lambda Patterns

O padrao mais fundamental da arquitetura serverless AWS.

```
Cliente -> API Gateway -> Lambda -> DynamoDB/RDS
                       -> Lambda -> S3
                       -> Lambda -> SQS -> Lambda (async)
```

#### Padroes de Integracao

| Padrao | Descricao | Use Case |
|--------|-----------|----------|
| **Sync REST** | API Gateway -> Lambda -> DB -> Response | CRUD APIs, dashboards |
| **Async Fire-and-Forget** | API Gateway -> SQS -> Lambda | Upload processing, emails |
| **Fan-Out** | SNS -> Multiplos SQS -> Multiplos Lambda | Notificacoes multicanal |
| **Orchestration** | Step Functions -> Multiplos Lambda | Order processing, aprovacoes |
| **Event-Driven** | EventBridge -> Lambda | Cross-service communication |
| **Streaming** | Kinesis/DynamoDB Streams -> Lambda | Real-time analytics |

**Fontes:** [Serverless Patterns - DEV](https://dev.to/aws-builders/serverless-patterns-4439), [AWS Compute Blog](https://aws.amazon.com/blogs/compute/serverless-icymi-q3-2025/)

#### API Gateway: REST vs HTTP API

| Caracteristica | REST API | HTTP API |
|---------------|----------|----------|
| **Custo** | $3.50 por milhao de requests | $1.00 por milhao de requests |
| **Latencia** | ~30ms overhead | ~10ms overhead |
| **Features** | API keys, usage plans, caching, WAF | JWT auth, CORS, OIDC |
| **WebSocket** | Nao (use WebSocket API) | Nao |
| **Recomendacao** | APIs complexas com rate limiting | APIs simples, custo otimizado |

**Recomendacao 2025:** Para a maioria dos novos projetos, use HTTP API. So use REST API se precisar de caching nativo, API keys, ou WAF integration.

---

### 2.2 Event-Driven Architecture

#### EventBridge vs SQS vs SNS vs Step Functions

| Servico | Tipo | Padrao | Use Case Principal |
|---------|------|--------|-------------------|
| **EventBridge** | Event bus | Many-to-many | Event routing com filtering e transformation |
| **SQS** | Queue | Point-to-point | Buffering, desacoplamento, retry |
| **SNS** | Topic | Pub/sub (fan-out) | Notificacoes, fan-out para multiplos consumers |
| **Step Functions** | Orchestrator | Stateful workflow | Processos complexos multi-step, saga pattern |

**Fontes:** [Serverless Architecture Patterns](https://systemdr.substack.com/p/serverless-architecture-patterns), [DEV - Event Driven Architecture](https://dev.to/brayanarrieta/building-event-driven-architectures-on-aws-a-modern-approach-to-scalability-and-decoupling-50lg)

#### Exemplo: Event-Driven com EventBridge

```
User Sign-Up Event
    |
    v
EventBridge (central bus)
    |
    +-> Rule: "billing" -> SQS -> Lambda (create billing account)
    +-> Rule: "notifications" -> SNS -> Email/SMS welcome
    +-> Rule: "analytics" -> Kinesis -> Lambda (track signup metrics)
    +-> Rule: "onboarding" -> Step Functions (multi-step flow)
```

**Novidade 2025:** A integracao SQS com Lambda agora oferece modo provisionado com event pollers configuraveis, cada um capaz de 1 MB/s throughput, 10 invocacoes concorrentes ou 10 polling calls/s.

**Fonte:** [AWS Compute Blog - re:Invent 2025](https://www.ranthebuilder.cloud/post/aws-re-invent-2025-my-serverless-agentic-ai-takeaways)

---

### 2.3 Serverless vs Containers: Decision Framework

| Criterio | Serverless (Lambda) | Containers (ECS/Fargate) |
|----------|--------------------|-----------------------|
| **Trafego** | Bursty, intermitente | Sustentado, previsivel |
| **Duty Cycle** | < 50% | > 50% |
| **Execucao** | < 15 min | Qualquer duracao |
| **Estado** | Stateless | Stateful possivel |
| **Custo Breakeven** | < 15 req/s sustentados | > 15 req/s sustentados |
| **Cold Start** | Preocupacao real | Nao se aplica (always running) |
| **Scaling** | Instantaneo (ms) | Minutos |

**Dados de custo concretos:**
- Processamento de 50K imagens: Containers = $4.80 vs Serverless = $380 (79x mais caro)
- Para < 15 req/s: Serverless e mais barato
- Para > 15 req/s sustentados: Containers sao mais custo-efetivos

**Tendencia 2025-2026:** 78% das equipes de engenharia rodam arquiteturas hibridas (serverless + containers), com reducao de 30-48% em custos.

**Fontes:** [The Cloud Standard](https://thecloudstandard.com/serverless-vs-containers/), [AWS FinOps Guide](https://aws.amazon.com/blogs/aws-cloud-financial-management/a-finops-guide-to-comparing-containers-and-serverless-functions-for-compute/), [ReadySetCloud](https://www.readysetcloud.io/blog/allen.helton/when-is-serverless-more-expensive/)

---

### 2.4 Cold Start Optimization

Cold start e o tempo que Lambda leva para inicializar um novo execution environment.

#### Tempos de Cold Start por Runtime (sem otimizacao)

| Runtime | Cold Start Tipico |
|---------|------------------|
| Python | 200-600ms |
| Node.js | 200-800ms |
| Go | 100-300ms |
| Java (Spring Boot) | 5-15 segundos |
| .NET | 400ms-2s |

#### Estrategias de Otimizacao

| Estrategia | Reducao | Aplicabilidade |
|-----------|---------|----------------|
| **SnapStart** | 4.3x (Java: 6.1s -> 1.4s) | Java, Python, .NET 8 (Native AOT) |
| **Graviton3 (ARM64)** | 15-20% cold start mais rapido | Node.js, Python |
| **Provisioned Concurrency** | Elimina cold start (pre-aquece) | Qualquer runtime (custo extra) |
| **Bundle otimizado** | 50% reducao em init | Todos (tree-shaking, minification) |
| **Lazy initialization** | Variavel | Todos (defer SDK init) |
| **Layers compartilhados** | Reducao de package size | Todos |

**SnapStart (2024-2025 Updates):**
- Originalmente so Java, agora suporta Python (nov 2024) e .NET 8 Native AOT
- Tira snapshot do environment inicializado e restaura na invocacao
- Java Spring Boot: 6.1s -> 1.4s (reducao de 4.3x)
- **Limitacao:** Nao suporta ARM64 (Graviton) -- apenas x86_64

**Graviton/ARM:**
- Lambda em Graviton2 (arm64): ~20% melhor price-performance
- Cold starts 13-24% mais rapidos na inicializacao
- Graviton4 disponivel em 2026 com melhorias adicionais

**Fontes:** [AWS Lambda Cold Start - Viprasol](https://viprasol.com/blog/aws-lambda-cold-start-optimization/), [Agile Soft Labs 2026](https://www.agilesoftlabs.com/blog/2026/02/aws-lambda-cold-start-7-proven-fixes), [Zircon.tech - Graviton4](https://zircon.tech/blog/aws-optimization-in-2026-graviton4-lambda-managed-instances-and-the-new-cost-playbook/)

---

### 2.5 Lambda@Edge vs CloudFront Functions

| Caracteristica | CloudFront Functions | Lambda@Edge |
|---------------|---------------------|-------------|
| **Runtime** | JavaScript restrito | Node.js, Python |
| **Execucao Max** | < 1ms | 5s (viewer) / 30s (origin) |
| **Memoria** | 2 MB | 128-10,240 MB |
| **Network Calls** | Nao | Sim |
| **Triggers** | Viewer request/response | Viewer + Origin request/response |
| **Custo** | $0.10 por milhao | $0.60 por milhao + duration |
| **Escala** | Milhoes de req/s | Milhares de req/s |

**Use Cases CloudFront Functions:**
- Cache key normalization
- URL rewrites e redirects
- Header manipulation
- A/B testing (simples)

**Use Cases Lambda@Edge:**
- Autenticacao/autorizacao complexa
- Server-side rendering no edge
- Image/video processing on-the-fly
- Acesso a DynamoDB/S3 no edge
- Personalizacao de conteudo

**Fonte:** [AWS Docs - Choosing Edge Functions](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/edge-functions-choosing.html), [Stormit](https://www.stormit.cloud/blog/cloudfront-functions-vs-lambda-at-edge/)

---

## 3. Padroes de Auto-Scaling

### 3.1 EC2 Auto Scaling Groups

#### Tipos de Scaling Policies

| Policy | Como Funciona | Quando Usar |
|--------|--------------|-------------|
| **Target Tracking** | Mantem metrica em um target (ex: CPU 60%) | Maioria dos casos -- set-and-forget |
| **Step Scaling** | Steps de scaling baseados em quao longe a metrica esta do threshold | Quando precisa de controle fino dos incrementos |
| **Simple Scaling** | Adiciona/remove N instancias quando threshold e ultrapassado | Legacy -- use Target Tracking ou Step |
| **Scheduled Scaling** | Escala em horarios predeterminados (cron) | Padroes previsiveis (horario comercial, Black Friday) |
| **Predictive Scaling** | ML analisa padroes historicos e provisiona proativamente | Workloads com padroes periodicos + tempo longo de init |

**Predictive Scaling:** Usa ML para analisar padroes de trafego historicos e provisionar instancias ANTES do pico. Funciona em conjunto com Target Tracking -- Predictive define capacidade minima, Target Tracking ajusta em tempo real.

**Fontes:** [AWS Predictive Scaling Docs](https://docs.aws.amazon.com/autoscaling/ec2/userguide/ec2-auto-scaling-predictive-scaling.html), [Sedai 2026](https://sedai.io/blog/understanding-aws-autoscaling-and-its-features)

#### Configuracao Recomendada

```yaml
# CDK/CloudFormation conceptual
AutoScalingGroup:
  MinSize: 2  # Minimo para HA (multi-AZ)
  MaxSize: 20
  DesiredCapacity: 2

  # Target Tracking (primario)
  TargetTrackingPolicy:
    TargetValue: 60  # CPU 60%
    PredefinedMetric: ASGAverageCPUUtilization

  # Predictive (complementar)
  PredictiveScalingPolicy:
    Mode: ForecastAndScale
    MaxCapacityBreachBehavior: HonorMaxCapacity

  # Scheduled (eventos conhecidos)
  ScheduledAction:
    Schedule: "cron(0 8 * * MON-FRI *)"  # Scale up 8am weekdays
    MinSize: 4
```

---

### 3.2 ECS Service Auto Scaling

```yaml
# ECS Service Auto Scaling
ServiceAutoScaling:
  MinTasks: 2
  MaxTasks: 50

  # Target Tracking por CPU
  TargetTrackingPolicy:
    TargetValue: 70
    PredefinedMetric: ECSServiceAverageCPUUtilization
    ScaleInCooldown: 300   # 5 min antes de scale-in
    ScaleOutCooldown: 60   # 1 min antes de scale-out

  # Step Scaling por custom metric (queue depth)
  StepScalingPolicy:
    MetricName: ApproximateNumberOfMessages
    Namespace: AWS/SQS
    Steps:
      - LowerBound: 100
        UpperBound: 500
        Adjustment: +2
      - LowerBound: 500
        Adjustment: +5
```

---

### 3.3 DynamoDB Auto-Scaling

| Modo | Descricao | Use Case |
|------|-----------|----------|
| **On-Demand** | Pay per request, sem provisioning | Trafego imprevisivel, novos projetos |
| **Provisioned + Auto Scaling** | Target utilization (tipico 60-70%) | Trafego previsivel, custo otimizado |

```yaml
# DynamoDB Auto Scaling
DynamoDBAutoScaling:
  ReadCapacity:
    MinCapacity: 5
    MaxCapacity: 1000
    TargetUtilization: 70%
  WriteCapacity:
    MinCapacity: 5
    MaxCapacity: 1000
    TargetUtilization: 70%
```

**Fonte:** [Flexera - AWS Auto Scaling](https://www.flexera.com/blog/finops/aws-autoscaling-scaling-ec2-ecs-rds-and-more/)

---

### 3.4 Aurora Auto-Scaling

Aurora Auto Scaling ajusta dinamicamente o numero de Read Replicas para lidar com picos de trafego de leitura.

```yaml
AuroraAutoScaling:
  MinReplicas: 1
  MaxReplicas: 15
  TargetValue: 70  # CPU das replicas
  ScaleInCooldown: 300
  ScaleOutCooldown: 300
```

**Aurora Serverless v2:** Escala automaticamente de 0.5 ACU ate 256 ACU, eliminando a necessidade de gerenciar replicas para muitos workloads.

---

### 3.5 Lambda Concurrency Management

| Tipo | Descricao | Custo |
|------|-----------|-------|
| **Unreserved** | Pool compartilhado (default 1000/conta) | Incluso |
| **Reserved** | Reserva N execucoes da conta | Sem custo extra |
| **Provisioned** | Pre-aquece N environments (elimina cold start) | $0.0000041667/GB-segundo |

```yaml
# Lambda Concurrency
LambdaConcurrency:
  AccountLimit: 1000  # Default (pode pedir aumento)
  ReservedConcurrency: 100  # Reserva para esta funcao
  ProvisionedConcurrency: 10  # Pre-aquecidas
  AutoScaling:  # Para Provisioned Concurrency
    MinCapacity: 5
    MaxCapacity: 50
    TargetUtilization: 70%
```

---

### 3.6 Quando Auto-Scaling NAO e Suficiente

Auto-scaling resolve problemas de capacidade, mas nao resolve problemas de arquitetura:

| Sinal | Problema Real | Solucao |
|-------|--------------|---------|
| Scaling mas latencia continua alta | Bottleneck no database | Adicionar cache (ElastiCache), read replicas |
| Custo de scaling maior que o revenue | Ineficiencia arquitetural | Refatorar para event-driven, otimizar queries |
| Scaling atinge limites da conta | Limites de servico | Arquitetura multi-region, request throttling |
| Escala mas deploys ficam lentos | Monolito grande | Migrar para microservicos |
| Scaling horizontal nao melhora throughput | Contencao de estado | Redesenhar data model, CQRS |

---

## 4. Melhores Praticas de Seguranca AWS

### 4.1 IAM: Policies, Roles e Least Privilege

#### Principios Fundamentais

1. **Nunca use root account** para operacoes do dia-a-dia
2. **MFA obrigatorio** em todas as contas (lesson #1 dos maiores breaches 2023-2025)
3. **Least privilege** -- conceda apenas as permissoes necessarias para a tarefa
4. **Roles > Users** -- use IAM Roles para servicos e aplicacoes
5. **Temporary credentials** -- use STS AssumeRole em vez de access keys

**Fontes:** [AWS IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html), [DEV - IAM 2026](https://dev.to/karaniph/aws-iam-security-best-practices-in-2026-a-complete-guide-o14)

#### Novidades 2025-2026

- **Access Analyzer unused access findings:** Identifica roles e policies com permissoes nao exercidas, combinando deteccao de acesso externo e auditoria de least-privilege.
- **Resource Control Policies (RCPs):** Introduzidas no final de 2024, complementam SCPs controlando resource-based policies diretamente, permitindo criar data perimeters.

**Fonte:** [AWS Security Blog - Least Privilege at Scale](https://aws.amazon.com/blogs/security/strategies-for-achieving-least-privilege-at-scale-part-1/)

#### Exemplo de IAM Policy (Least Privilege)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-app-uploads/*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "us-east-1"
        }
      }
    }
  ]
}
```

**Anti-pattern:** NUNCA use `"Action": "*"` ou `"Resource": "*"` em producao.

---

### 4.2 AWS Organizations e SCPs

**SCPs (Service Control Policies)** definem guardrails de permissao -- nao concedem permissoes, apenas limitam o maximo que pode ser concedido.

#### Exemplo de SCP: Bloquear regioes nao autorizadas

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyNonApprovedRegions",
      "Effect": "Deny",
      "NotAction": [
        "iam:*",
        "organizations:*",
        "sts:*",
        "support:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": [
            "us-east-1",
            "sa-east-1"
          ]
        }
      }
    }
  ]
}
```

**Boas praticas:**
- NUNCA attache SCPs na raiz da Organization sem testar extensivamente
- Crie uma OU de teste e mova contas uma a uma
- Use `service last accessed data` para refinar SCPs

**Fonte:** [AWS Organizations SCPs Docs](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html)

---

### 4.3 Secrets Manager vs SSM Parameter Store vs Vault

| Caracteristica | SSM Parameter Store | Secrets Manager | HashiCorp Vault |
|---------------|-------------------|----------------|----------------|
| **Custo** | Gratis (Standard) | $0.40/secret/mes + API | Open-source (infra propria) |
| **Rotacao Automatica** | Nao (requer Lambda custom) | Sim (nativo para RDS, Redshift) | Sim (dynamic secrets) |
| **Password Generation** | Nao | Sim (CLI/API) | Sim |
| **Multi-Cloud** | AWS only | AWS only | Multi-cloud e hybrid |
| **Complexidade** | Baixa | Baixa | Alta (self-managed) |
| **Limite de Tamanho** | 8KB (Standard) / 8KB (Advanced) | 64KB | Ilimitado |
| **Integracao AWS** | Nativa (CloudFormation, ECS, Lambda) | Nativa | Requer configuracao |

**Quando usar cada:**
- **SSM Parameter Store:** Configs simples, feature flags, key-value nao-sensivel. Gratis.
- **Secrets Manager:** Credenciais de DB com rotacao automatica, API keys que precisam rodar.
- **HashiCorp Vault:** Multi-cloud, dynamic secrets (JIT), 10K+ secrets (custo-efetivo em escala).

**Fontes:** [HackerNoon Comparison](https://hackernoon.com/aws-secrets-manager-vs-hashicorp-vault-vs-aws-parameter-store-bcbf60b0c0d1), [ScaleSec](https://scalesec.com/blog/a-comparison-of-secrets-managers-for-aws/)

---

### 4.4 KMS para Encryption

#### Tipos de KMS Keys

| Tipo | Gerenciamento | Custo | Use Case |
|------|--------------|-------|----------|
| **AWS Managed** | AWS gerencia ciclo de vida | Gratis | Default para servicos AWS |
| **Customer Managed** | Voce controla policies e rotacao | $1/key/mes + $0.03/10K requests | Compliance, audit requirements |
| **External Key Material** | Import your own key | $1/key/mes | Requisitos regulatorios especificos |

**Boas Praticas KMS:**
1. Crie keys separadas por servico (S3, RDS, Secrets Manager, EBS)
2. Mantenha administradores e usuarios de keys separados (separation of duties)
3. NUNCA use `"Principal": "*"` sem conditions em key policies
4. Use CloudTrail para auditar uso de keys
5. AWS suporta hybrid post-quantum TLS em KMS (ECDH + ML-KEM)

**Fontes:** [AWS KMS Best Practices](https://docs.aws.amazon.com/prescriptive-guidance/latest/aws-kms-best-practices/introduction.html), [Toc Consulting 2026](https://tocconsulting.fr/best-practices/kms-security)

---

### 4.5 WAF, Shield e Protecao DDoS

| Servico | Camada | Custo | Protege Contra |
|---------|--------|-------|---------------|
| **Shield Standard** | L3/L4 | Gratis | DDoS volumetrico comum |
| **Shield Advanced** | L3/L4/L7 | $3,000/mes + data fees | DDoS sofisticado + DRT team + cost protection |
| **WAF** | L7 | $5/web ACL + $1/rule + $0.60/milhao requests | SQLi, XSS, bots, rate limiting |

**Shield Advanced inclui:**
- Visibilidade de ataques em tempo real
- Acesso ao DDoS Response Team (DRT) 24/7
- Reembolso de custos de scaling causados por ataques
- Mitigacao automatica na camada de aplicacao via WAF rules

**WAF Managed Rules recomendadas:**
- AWS Core Rule Set (CRS) -- protecao contra OWASP Top 10
- SQL Database rule group
- Known Bad Inputs rule group
- Bot Control

**Fonte:** [DEV - AWS Security at Scale](https://dev.to/imsushant12/aws-waf-shield-and-guardduty-security-at-scale-41p7), [AWS WAF Best Practices](https://aws.github.io/aws-security-services-best-practices/guides/waf/using-waf-with-other-services/docs/)

---

### 4.6 GuardDuty, Security Hub, Config

| Servico | Funcao | Custo |
|---------|--------|-------|
| **GuardDuty** | Threat detection (analisa logs, VPC flow, DNS) | Pay per volume analisado |
| **Security Hub** | Dashboard centralizado de seguranca | $0.0010 per finding per check |
| **Config** | Compliance -- avalia configuracoes de recursos | $0.003 per item recorded |
| **CloudTrail** | Audit log de todas as API calls | Gratis (management events, 1 trail) |
| **Inspector** | Vulnerability scanning (EC2, ECR, Lambda) | Pay per scan |
| **Macie** | PII detection em S3 | Pay per GB scanned |

**Configuracao recomendada minima:**
1. GuardDuty em todas as contas e regioes
2. Security Hub com CIS AWS Foundations Benchmark v5.0.0 (outubro 2025)
3. Config com rules basicas (S3 public, SG open, encryption)
4. CloudTrail em todas as regioes
5. VPC Flow Logs habilitados em todas as VPCs

**Fontes:** [Toc Consulting - Security Hub 2026](https://tocconsulting.fr/best-practices/securityhub-security), [AWS GuardDuty Best Practices](https://aws.github.io/aws-security-services-best-practices/guides/guardduty/)

---

## 5. Estrategia Multi-Account

### 5.1 Por Que Multiplas Contas

Uma unica conta AWS e como um monolito -- qualquer erro afeta tudo. Multi-account oferece:

- **Isolamento de seguranca:** Blast radius limitado se uma conta for comprometida
- **Separacao de billing:** Custos claros por projeto/ambiente
- **Least privilege natural:** Desenvolvedores nao podem acidentalmente tocar em producao
- **Compliance:** Ambientes regulados separados dos demais
- **Limites de servico:** Cada conta tem seus proprios limites

**Fonte:** [AWS Whitepaper - Organizing Your Environment](https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/organizing-your-aws-environment.html)

---

### 5.2 Estrutura de OUs Recomendada

```
Root
|
+-- Security OU
|   +-- Log Archive Account (CloudTrail, Config, VPC Flow Logs)
|   +-- Security Tooling Account (GuardDuty, Security Hub, Inspector)
|
+-- Infrastructure OU
|   +-- Network Account (Transit Gateway, VPN, Direct Connect)
|   +-- Shared Services Account (CI/CD, containers registry, DNS)
|
+-- Workloads OU
|   +-- SDLC OU
|   |   +-- Dev Account
|   |   +-- Staging Account
|   |   +-- QA Account
|   |
|   +-- Production OU
|       +-- Prod Account
|       +-- DR Account
|
+-- Sandbox OU
|   +-- Developer Sandbox Accounts (experimentacao livre)
|
+-- Suspended OU
    +-- Contas desativadas (SCP: deny all)
```

**Fontes:** [AWS Organizations Best Practices](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_best-practices.html), [Spacelift - Multi-Account Strategy](https://spacelift.io/blog/aws-multi-account-strategy)

---

### 5.3 Cross-Account Access Patterns

| Padrao | Mecanismo | Use Case |
|--------|-----------|----------|
| **AssumeRole** | IAM Role com trust policy | Acesso temporario cross-account |
| **Resource-Based Policy** | Policy no recurso (S3, KMS, SNS) | Compartilhar recurso especifico |
| **AWS RAM** | Resource Access Manager | Compartilhar subnets, Transit Gateway |
| **SSO / Identity Center** | Centralized login | Acesso humano cross-account |

```json
// Trust Policy para cross-account role
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "unique-external-id"
        }
      }
    }
  ]
}
```

---

### 5.4 Consolidated Billing

- Todas as contas sob uma Organization compartilham volume discounts
- Reservas (RIs, Savings Plans) podem ser compartilhadas entre contas
- Use **Cost Allocation Tags** para rastrear custos por projeto/equipe/ambiente
- **AWS Cost Explorer** da Organization mostra gastos agregados e por conta

---

### 5.5 Casos Reais: Netflix e Airbnb

**Netflix:**
- Roda mais de 1000 microservicos loose-coupled na AWS
- Arquitetura multi-region para streaming global
- Adotou abordagem "Identity First" para seguranca cross-account
- Usa centenas de contas AWS com automacao pesada

**Airbnb:**
- Milhares de instancias EC2 com Kubernetes (EKS) para microservicos
- Centenas de microservicos em EKS (search, bookings, messaging, payments)
- Cada servico escala independentemente

**Fontes:** [Netflix Cloud Security Newsletter](https://www.cloudsecuritynewsletter.com/p/netflix-s-identity-first-aws-cloud-security-evolution), [Medium - Netflix and Airbnb on AWS](https://medium.com/@gajanandaadhikari/case-studies-scaling-globally-with-aws-lessons-from-netflix-and-airbnb-069e3fe7daf4)

---

## 6. Otimizacao de Custos

### 6.1 Reserved Instances vs Savings Plans vs Spot

| Opcao | Desconto | Flexibilidade | Commitment | Melhor Para |
|-------|----------|---------------|------------|-------------|
| **On-Demand** | 0% | Total | Nenhum | Testes, workloads imprevisiveis |
| **Spot Instances** | Ate 90% | Pode ser interrompido (2 min notice) | Nenhum | Batch processing, CI/CD, ML training |
| **Savings Plans (Compute)** | Ate 66% | EC2, Fargate, Lambda -- qualquer familia/regiao | 1 ou 3 anos | Maioria dos workloads |
| **Savings Plans (EC2 Instance)** | Ate 72% | Familia/regiao especifica | 1 ou 3 anos | Workloads estaveis em regiao fixa |
| **Reserved Instances (Standard)** | Ate 75% | Familia/tipo/regiao fixa | 1 ou 3 anos | DB workloads com alto desconto |
| **Reserved Instances (Convertible)** | 31-54% | Pode trocar familia de instancia | 1 ou 3 anos | Workloads em evolucao |

**Fontes:** [Finout - 5 Key Differences 2025](https://www.finout.io/blog/aws-savings-plans-vs-reserved-instances-5-key-differences-in-2025), [Holori 2026](https://holori.com/aws-savings-plans-vs-reserved-instances-which-should-you-choose-in-2026/)

#### Estrategia Recomendada 2025-2026

```
1. Compute Savings Plans para compute geral (EC2, Fargate, Lambda)
   -> Melhor balance entre desconto e flexibilidade

2. Reserved Instances para databases (RDS, ElastiCache)
   -> Desconto maior justifica o lock-in para workloads estaveis

3. Spot Instances para workloads tolerantes a interrupcao
   -> CI/CD runners, batch processing, ML training

4. On-Demand para tudo mais (dev, testes, picos)
```

**Mudanca 2025:** A partir de 1 de junho 2025, RIs e Savings Plans sao restritos ao uso de um unico cliente final, afetando MSPs e revendedores.

**Fonte:** [Nops - RI and SP Changes 2025](https://www.nops.io/blog/aws-reserved-instance-and-savings-plan-changes-for-2025/)

---

### 6.2 Right-Sizing e Ferramentas de Custo

| Ferramenta | Funcao |
|-----------|--------|
| **AWS Cost Explorer** | Visualizacao e analise de custos historicos e projecoes |
| **AWS Budgets** | Alertas quando custos excedem limites definidos |
| **AWS Compute Optimizer** | Recomendacoes de right-sizing para EC2, Lambda, EBS |
| **Trusted Advisor** | Checks de custo, seguranca, performance, fault tolerance |
| **Cost Allocation Tags** | Tags para categorizar custos por projeto/equipe/ambiente |
| **AWS Cost Anomaly Detection** | ML para detectar gastos anomalos |

**Cost Allocation Tags essenciais:**

```yaml
Tags:
  - Key: Project
    Value: "my-saas-app"
  - Key: Environment
    Value: "production"
  - Key: Team
    Value: "backend"
  - Key: CostCenter
    Value: "engineering"
```

---

### 6.3 Custos Reais por Escala de Usuarios

**Aviso:** Custos variam enormemente por tipo de aplicacao, padroes de acesso e arquitetura. Estes sao estimativas baseadas em web apps tipicos (API + DB + storage + CDN).

#### Estimativa: Web App SaaS Tipico

| Escala | Arquitetura Sugerida | Custo Mensal Estimado |
|--------|---------------------|----------------------|
| **10K MAU** | Lambda + API Gateway + RDS t3.small + S3 + CloudFront | $50-150/mes |
| **100K MAU** | Lambda/ECS Fargate + Aurora Serverless + ElastiCache + CloudFront | $500-1,500/mes |
| **1M MAU** | ECS Fargate (multi-AZ) + Aurora + ElastiCache cluster + CloudFront + WAF | $3,000-8,000/mes |
| **10M MAU** | ECS/EKS multi-region + Aurora Global + ElastiCache + CloudFront + Shield Advanced | $15,000-50,000+/mes |

**Custos ocultos frequentes:**
- **NAT Gateway:** $33/mes por AZ + $0.045/GB processado
- **Data Transfer:** $0.09/GB saindo da AWS (pode ser 5-50x mais caro que alternativas)
- **CloudWatch Logs:** Acumulam rapido se nao configurar retencao
- **Elastic IPs nao associadas:** $3.60/mes por IP ocioso (desde fev 2024)

**Fontes:** [Amazon Hidden Costs 2026](https://costbench.com/software/cloud-infrastructure/aws/hidden-costs/), [Flexera 2025 Report](https://www.appverticals.com/blog/aws-pricing-overview/)

---

### 6.4 Serverless vs Container Cost em Diferentes Escalas

```
Custo Mensal
    ^
    |           Containers (ECS/Fargate)
    |          /
    |         /
    |        /
    |       /    Serverless (Lambda)
    |      /    /
    |     /    /
    |    /    /
    |   /   /
    |  /  /
    | / /
    |//
    +--+----+----+----+-----> Requests/segundo
       5   15   50   100

    Breakeven: ~15 req/s sustentados
    Abaixo: Serverless mais barato
    Acima: Containers mais baratos
```

**Dado concreto:** Equipes que aplicam hybrid FinOps (serverless para bursty + containers para sustentado) alcancam 30-48% de reducao de custo comparado a usar apenas um modelo.

**Fonte:** [The Cloud Standard 2026](https://thecloudstandard.com/serverless-vs-containers/)

---

## 7. Quando Graduar do Vercel/Supabase

### 7.1 Sinais de que Voce Superou o PaaS

| Sinal | Sintoma | Impacto |
|-------|---------|---------|
| **Hacks constantes** | Engenheiros gastam tempo criando workarounds para limitacoes da plataforma | Produtividade cai |
| **Custos crescentes** | Conta Supabase vai de $25 para $500+/mes | ROI diminui |
| **Limites de funcoes** | Edge functions com timeout insuficiente, cold starts inaceitaveis | UX degradada |
| **Vendor lock-in** | Preso a versao especifica de Postgres, auth system, edge functions | Flexibilidade zero |
| **Compliance** | Requisitos de seguranca/compliance nao atendidos pelo PaaS | Risco legal |
| **Performance ceiling** | Otimizacoes esgotadas dentro das limitacoes do PaaS | Crescimento bloqueado |
| **Falta de observabilidade** | Nao consegue debugar problemas complexos | Incidentes demoram para resolver |

**Fontes:** [ClickIT - Migrate from Vercel to AWS](https://www.clickittech.com/ai/migrate-from-vercel-to-aws/), [Medium - Startup Scaling Playbook](https://medium.com/@Johannbeukes/the-startup-scaling-playbook-from-vercel-to-aws-and-when-to-switch-3cb81b7368a7)

---

### 7.2 Migration Paths

#### Vercel para AWS

| Componente Vercel | Equivalente AWS | Ferramenta de Migracao |
|------------------|-----------------|----------------------|
| Next.js hosting | ECS/Fargate + ALB ou Lambda + CloudFront | SST + OpenNext |
| Edge Functions | Lambda@Edge ou CloudFront Functions | Reescrita manual |
| Image Optimization | CloudFront + Lambda@Edge ou S3 + imgproxy | Configuracao |
| Analytics | CloudWatch + custom dashboards | Instrumentacao |
| Preview Deployments | CodePipeline + ALB routing | Configuracao CI/CD |
| CDN | CloudFront | Configuracao |
| DNS | Route 53 | Migracao de records |

**SST + OpenNext:** Framework open-source recomendado para migrar Next.js do Vercel para AWS. Da experiencia similar ao Vercel com controle total e custos menores.

**Economia real:** Custos de Vercel (~$2,000/mes) caindo para ~$500/mes apos migracao para ECS + CloudFront.

**Fonte:** [Encore Cloud - Migrate from Vercel](https://encore.cloud/resources/migrate-vercel-to-aws)

#### Supabase para AWS

| Componente Supabase | Equivalente AWS | Complexidade |
|--------------------|-----------------|-------------|
| PostgreSQL | RDS PostgreSQL ou Aurora | Media (pg_dump/pg_restore) |
| Auth (GoTrue) | Cognito ou Auth0 | Alta (requer migracao de usuarios) |
| Realtime | AppSync ou API Gateway WebSocket | Alta |
| Storage (S3-based) | S3 direto | Baixa |
| Edge Functions (Deno) | Lambda (Node.js/Python) | Media (reescrita) |
| PostgREST API | API Gateway + Lambda ou AppSync | Media |
| Row Level Security | RDS com policies customizadas | Alta (requer reescrita) |

**Migracao de DB Supabase -> RDS/Aurora:**
1. `pg_dump` do Supabase (incluindo schema + data)
2. Criar RDS PostgreSQL ou Aurora PostgreSQL
3. `pg_restore` no RDS
4. Reconfigurar connection strings na aplicacao
5. Migrar RLS policies para application-level ou DB-level
6. Testar extensivamente

**Fonte:** [Bytebase - Migrate Supabase to AWS](https://www.bytebase.com/blog/how-to-migrate-from-supabase-to-aws/), [FZF.dev - 8 Weeks Migration](https://fzf.dev/en/cloud-migration)

---

### 7.3 Abordagem Hibrida (Recomendada para Transicao)

A estrategia mais pragmatica: manter o frontend no Vercel e mover o backend para AWS.

```
[Vercel - Frontend]
    |
    | HTTPS API calls
    v
[API Gateway / ALB - AWS]
    |
    +-> Lambda / ECS (business logic)
    +-> RDS Aurora (database)
    +-> S3 (storage)
    +-> ElastiCache (cache)
    +-> SQS/EventBridge (async)
```

**Vantagens:**
- Mantem DX do Vercel para frontend (preview deploys, edge network, zero-config)
- Ganha controle total do backend (networking, scaling, security)
- Migracao incremental -- move servico por servico
- Custos de frontend no Vercel sao geralmente baixos

**Desvantagens:**
- Latencia extra de cross-origin API calls (mitigavel com Vercel Rewrites)
- Dois providers para gerenciar
- Build/deploy separados para frontend e backend

**Fonte:** [Vercel + AWS Partner Page](https://vercel.com/partners/aws), [Encore Cloud](https://encore.cloud/resources/migrate-vercel-to-aws)

---

### 7.4 Comparativo de Custo em Escala

| Escala | Vercel + Supabase | AWS (self-managed) | Economia |
|--------|------------------|-------------------|----------|
| **MVP/Hobby** | $0-25/mes | $50-100/mes | Vercel 2-4x mais barato |
| **10K MAU** | $45-75/mes | $50-150/mes | Similar |
| **100K MAU** | $300-600/mes | $500-1,500/mes | PaaS pode ser mais barato |
| **1M MAU** | $2,000-5,000/mes | $3,000-8,000/mes | Depende do uso |
| **10M MAU** | $10,000-30,000+/mes | $5,000-15,000/mes | AWS 2-3x mais barato |

**Ponto de inflexao:** Entre 100K-1M MAU, a economia da AWS comeca a justificar a complexidade operacional. Abaixo disso, o PaaS geralmente vale o premium pela simplicidade.

**Nota:** A maioria das equipes ve reducao de 30-50% em custos apos migracao, mas a equipe precisa ter capacidade operacional para gerenciar a infraestrutura.

**Fonte:** [ClickIT - Vercel Tax](https://www.clickittech.com/ai/migrate-from-vercel-to-aws/)

---

## 8. AWS Well-Architected Framework

### 8.1 Visao Geral dos 6 Pilares

O Well-Architected Framework da AWS e um guia abrangente para construir arquiteturas seguras, eficientes e sustentaveis na nuvem. Possui 6 pilares, cada um com principios de design e melhores praticas.

**Fonte:** [AWS Well-Architected](https://aws.amazon.com/architecture/well-architected/), [AWS Blog - 6 Pillars](https://aws.amazon.com/blogs/apn/the-6-pillars-of-the-aws-well-architected-framework/)

---

### 8.2 Pilar 1: Operational Excellence

**Foco:** Executar e monitorar sistemas para entregar valor de negocio e melhorar continuamente processos e procedimentos.

| Principio | Pratica |
|-----------|---------|
| Operate as code | Infrastructure as Code (CDK, Terraform, CloudFormation) |
| Make frequent, small changes | CI/CD com deploys frequentes e reversiveis |
| Anticipate failure | Runbooks, game days, chaos engineering |
| Learn from failures | Post-mortems sem blame, metricas de melhoria |
| Use managed services | Reduzir burden operacional com servicos gerenciados |

**Ferramentas AWS:**
- CloudFormation / CDK (IaC)
- Systems Manager (operacoes)
- CloudWatch (monitoring)
- X-Ray (tracing distribuido)
- EventBridge (automacao)

---

### 8.3 Pilar 2: Security

**Foco:** Proteger informacoes, sistemas e ativos enquanto entrega valor de negocio.

| Principio | Pratica |
|-----------|---------|
| Strong identity foundation | IAM, MFA, least privilege, centralized identity |
| Traceability | CloudTrail, VPC Flow Logs, Config |
| Apply security at all layers | WAF, Security Groups, NACLs, encryption |
| Automate security | GuardDuty, Security Hub, automated remediation |
| Protect data in transit and at rest | KMS, TLS, certificate management |
| Prepare for security events | Incident response plans, runbooks |

---

### 8.4 Pilar 3: Reliability

**Foco:** Garantir que um workload execute sua funcao pretendida corretamente e consistentemente.

| Principio | Pratica |
|-----------|---------|
| Recover from failure automatically | Auto Scaling, health checks, failover |
| Test recovery procedures | Chaos engineering, DR drills |
| Scale horizontally | Distribua carga entre multiplos recursos pequenos |
| Stop guessing capacity | Auto Scaling baseado em metricas reais |
| Manage change through automation | IaC, CI/CD, blue-green deployments |

**Metricas-chave:**
- RTO (Recovery Time Objective): Quanto tempo para recuperar
- RPO (Recovery Point Objective): Quanto dado pode perder

| Estrategia DR | RTO | RPO | Custo |
|--------------|-----|-----|-------|
| Backup & Restore | Horas | Horas | Baixo |
| Pilot Light | Minutos | Minutos | Medio |
| Warm Standby | Minutos | Segundos | Alto |
| Multi-Site Active-Active | Tempo real | Zero | Muito Alto |

---

### 8.5 Pilar 4: Performance Efficiency

**Foco:** Usar recursos computacionais eficientemente para atender requisitos do sistema.

| Principio | Pratica |
|-----------|---------|
| Democratize advanced technologies | Use servicos gerenciados (ML, analytics, databases) |
| Go global in minutes | Multi-region deployment, CloudFront, Global Accelerator |
| Use serverless architectures | Lambda, Fargate, Aurora Serverless, DynamoDB |
| Experiment more often | A/B test diferentes configuracoes |
| Consider mechanical sympathy | Escolha o tipo de recurso certo para o workload |

---

### 8.6 Pilar 5: Cost Optimization

**Foco:** Evitar gastos desnecessarios e entender onde o dinheiro e gasto.

| Principio | Pratica |
|-----------|---------|
| Implement cloud financial management | FinOps team, cost-aware culture |
| Adopt a consumption model | Pay for what you use (serverless, auto-scaling) |
| Measure overall efficiency | Custo por transacao, custo por usuario |
| Stop spending on undifferentiated heavy lifting | Use managed services |
| Analyze and attribute expenditure | Tags, Cost Explorer, Budgets |

---

### 8.7 Pilar 6: Sustainability

**Foco:** Minimizar impactos ambientais dos workloads na nuvem.

| Principio | Pratica |
|-----------|---------|
| Understand your impact | Medir carbon footprint (Customer Carbon Footprint Tool) |
| Establish sustainability goals | Metricas de eficiencia energetica |
| Maximize utilization | Right-size instancias, eliminar idle resources |
| Use efficient hardware | Graviton (ARM) -- ate 60% mais eficiente energeticamente |
| Use managed services | Escala compartilhada e mais eficiente |
| Reduce downstream impact | Otimizar data transfer, minimizar dados armazenados |

**AWS e Sustainability:**
- Meta de 100% energia renovavel ate 2025 (alcancada)
- Graviton processors: 60% mais eficientes energeticamente que x86 equivalentes
- Regioes AWS com menor carbon footprint: US West (Oregon), Europe (Ireland, Frankfurt)

**Fonte:** [AWS Well-Architected - Sustainability](https://docs.aws.amazon.com/wellarchitected/latest/framework/the-pillars-of-the-framework.html), [Tutorials Dojo - 6 Pillars](https://tutorialsdojo.com/aws-well-architected-framework-six-pillars/)

---

### 8.8 Well-Architected Review Process

O AWS Well-Architected Tool e um servico gratuito no console AWS que permite avaliar sua arquitetura contra os 6 pilares.

**Processo:**
1. **Definir workload** no Well-Architected Tool
2. **Responder perguntas** de cada pilar (weighted choices)
3. **Identificar High-Risk Issues (HRIs)** e Medium-Risk Issues (MRIs)
4. **Criar improvement plan** com acoes priorizadas
5. **Implementar melhorias** e reavaliar periodicamente
6. **Milestone reviews** para trackear progresso

**Recomendacao:** Faca um Well-Architected Review pelo menos 1x por ano ou antes de grandes mudancas arquiteturais.

---

## Apendice: Checklist de Decisao AWS

### Quando migrar para AWS?

```
[ ] Custos do PaaS excedem $500/mes consistentemente
[ ] Equipe tem pelo menos 1 pessoa com experiencia AWS
[ ] Requisitos de compliance nao atendidos pelo PaaS
[ ] Performance ceiling atingido no PaaS atual
[ ] Mais de 50% do tempo dos engenheiros e gasto em workarounds
[ ] Escala projetada > 100K MAU nos proximos 12 meses
```

### Stack AWS recomendada para web apps

```
Nivel 1 (MVP, < 10K MAU):
  Compute:  Lambda + API Gateway
  Database: RDS PostgreSQL (t3.micro/small)
  Storage:  S3
  CDN:      CloudFront
  DNS:      Route 53
  Secrets:  SSM Parameter Store

Nivel 2 (Growth, 10K-100K MAU):
  Compute:  Lambda + ECS Fargate (hibrido)
  Database: Aurora PostgreSQL Serverless v2
  Cache:    ElastiCache Redis
  Storage:  S3
  CDN:      CloudFront
  DNS:      Route 53
  Secrets:  Secrets Manager
  Security: WAF + GuardDuty + Security Hub

Nivel 3 (Scale, 100K-1M MAU):
  Compute:  ECS Fargate (multi-AZ) + Lambda (async)
  Database: Aurora PostgreSQL + Read Replicas
  Cache:    ElastiCache Redis Cluster
  Storage:  S3 + EFS (se necessario)
  CDN:      CloudFront + WAF
  DNS:      Route 53 (latency-based)
  Secrets:  Secrets Manager
  Security: WAF + Shield Advanced + GuardDuty + Security Hub
  Monitoring: CloudWatch + X-Ray + Datadog/New Relic

Nivel 4 (Enterprise, > 1M MAU):
  Compute:  EKS / ECS multi-region
  Database: Aurora Global Database + DynamoDB (hot data)
  Cache:    ElastiCache Global Datastore
  Storage:  S3 (multi-region replication)
  CDN:      CloudFront + Global Accelerator
  DNS:      Route 53 (failover + latency)
  Security: Full security stack + Shield Advanced + DRT
  Accounts: Multi-account (Organizations)
  DR:       Multi-region active-active ou warm standby
```

---

## Fontes Consolidadas

### Compute
- [Thoughtful Architect - AWS Compute Comparison](https://www.thoughtfularchitect.dev/posts/aws-compute-comparison)
- [AWS - Fargate or Lambda Decision Guide](https://docs.aws.amazon.com/decision-guides/latest/fargate-or-lambda/fargate-or-lambda.html)
- [cloudonaut - Fargate vs App Runner](https://cloudonaut.io/fargate-vs-apprunner/)
- [MTechZilla - Fargate vs ECS vs Lambda 2025](https://www.mtechzilla.com/blogs/aws-fargate-vs-ecs-vs-lambda)

### Storage
- [SquareOps - AWS Storage Guide 2025](https://squareops.com/knowledge/aws-s3-vs-ebs-vs-efs-vs-glacier-which-storage-is-best-in-2025/)
- [NetApp - EBS EFS S3 Comparison](https://www.netapp.com/blog/ebs-efs-amazons3-best-cloud-storage-system/)

### Database
- [Bytebase - RDS vs DynamoDB 2025](https://www.bytebase.com/blog/rds-vs-dynamodb/)
- [Bytebase - Aurora vs RDS 2025](https://www.bytebase.com/blog/aurora-vs-rds/)
- [Lushbinary - Aurora vs RDS 2026](https://lushbinary.com/blog/aws-aurora-vs-rds-cost-performance-architecture-guide-2026/)

### Networking
- [InventiveHQ - VPC Network Segmentation](https://inventivehq.com/knowledge-base/aws/aws-vpc-network-segmentation)
- [AWS - DDoS Best Practices](https://docs.aws.amazon.com/whitepapers/latest/aws-best-practices-ddos-resiliency/security-groups-and-network-acls-bp5.html)

### CDN
- [CrawlWP - CloudFront vs Cloudflare 2026](https://crawlwp.com/aws-cloudfront-vs-cloudflare/)
- [CloudOptimo - CDN Comparison 2025](https://www.cloudoptimo.com/blog/cloudfront-vs-cloudflare-vs-akamai-choosing-the-right-cdn-in-2025/)

### Serverless
- [AWS - Serverless Patterns](https://dev.to/aws-builders/serverless-patterns-4439)
- [AWS Compute Blog - re:Invent 2025](https://www.ranthebuilder.cloud/post/aws-re-invent-2025-my-serverless-agentic-ai-takeaways)
- [Viprasol - Lambda Cold Start 2026](https://viprasol.com/blog/aws-lambda-cold-start-optimization/)
- [AWS - Lambda@Edge vs CloudFront Functions](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/edge-functions-choosing.html)

### Auto-Scaling
- [AWS - Predictive Scaling](https://docs.aws.amazon.com/autoscaling/ec2/userguide/ec2-auto-scaling-predictive-scaling.html)
- [Sedai - AWS Auto Scaling 2026](https://sedai.io/blog/understanding-aws-autoscaling-and-its-features)

### Security
- [AWS IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [AWS KMS Best Practices](https://docs.aws.amazon.com/prescriptive-guidance/latest/aws-kms-best-practices/introduction.html)
- [AWS GuardDuty Best Practices](https://aws.github.io/aws-security-services-best-practices/guides/guardduty/)
- [AWS WAF Best Practices](https://aws.github.io/aws-security-services-best-practices/guides/waf/using-waf-with-other-services/docs/)

### Multi-Account
- [AWS - Organizing Your Environment](https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/organizing-your-aws-environment.html)
- [AWS - Organizations Best Practices](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_best-practices.html)
- [Spacelift - Multi-Account Strategy](https://spacelift.io/blog/aws-multi-account-strategy)

### Cost Optimization
- [Finout - Savings Plans vs RIs 2025](https://www.finout.io/blog/aws-savings-plans-vs-reserved-instances-5-key-differences-in-2025)
- [The Cloud Standard - Serverless vs Containers Cost 2026](https://thecloudstandard.com/serverless-vs-containers/)
- [AWS FinOps Guide](https://aws.amazon.com/blogs/aws-cloud-financial-management/a-finops-guide-to-comparing-containers-and-serverless-functions-for-compute/)

### Migration
- [ClickIT - Migrate from Vercel to AWS](https://www.clickittech.com/ai/migrate-from-vercel-to-aws/)
- [Bytebase - Migrate Supabase to AWS](https://www.bytebase.com/blog/how-to-migrate-from-supabase-to-aws/)
- [Encore Cloud - Migrate from Vercel](https://encore.cloud/resources/migrate-vercel-to-aws)

### Well-Architected
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [AWS Blog - 6 Pillars](https://aws.amazon.com/blogs/apn/the-6-pillars-of-the-aws-well-architected-framework/)
- [AWS Docs - Pillars of the Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/the-pillars-of-the-framework.html)

---

*Pesquisa conduzida por: research-orqx (Prism) | Squad: squad-research*
*Data: 2026-04-11 | Nivel: DEEP DIVE | Fontes: 40+ verificadas via WebSearch*
