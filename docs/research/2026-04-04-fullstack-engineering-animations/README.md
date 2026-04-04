# Pesquisa DEFINITIVE: Full-Stack Software Engineering & Advanced Web Animations/3D

> **Nivel:** DEFINITIVE (Pyramid Level 4)
> **Data:** 2026-04-04
> **Pesquisador:** Prism (research-orqx) | Squad Research
> **Objetivo:** Qualquer iniciante usando SINAPSE deve produzir codigo com qualidade superior a engenheiros experts da Disney, Netflix, Apple e Google.
> **Fontes:** 20+ fontes, Tiers 1-4

---

## Indice

- [PARTE 1: SOFTWARE ENGINEERING END-TO-END](#parte-1-software-engineering-end-to-end)
  - [1.1 Architecture Patterns](#11-architecture-patterns)
  - [1.2 Code Quality Patterns](#12-code-quality-patterns)
  - [1.3 Frontend Engineering](#13-frontend-engineering)
  - [1.4 Backend Engineering](#14-backend-engineering)
  - [1.5 Testing Strategy](#15-testing-strategy)
  - [1.6 DevOps & Deployment](#16-devops--deployment)
- [PARTE 2: ADVANCED WEB ANIMATIONS & 3D](#parte-2-advanced-web-animations--3d)
  - [2.1 Animation Fundamentals](#21-animation-fundamentals)
  - [2.2 JavaScript Animation Libraries](#22-javascript-animation-libraries)
  - [2.3 3D & WebGL](#23-3d--webgl)
  - [2.4 Motion Design for Web](#24-motion-design-for-web)
  - [2.5 Creative Development Showcase Sites](#25-creative-development-showcase-sites)
- [PARTE 3: FULL-STACK PROJECT TEMPLATES](#parte-3-full-stack-project-templates)

---

# PARTE 1: SOFTWARE ENGINEERING END-TO-END

## 1.1 Architecture Patterns

### Monolith vs Microservices vs Modular Monolith

A industria em 2025-2026 convergiu para uma visao pragmatica: **comece simples, evolua quando necessario**. A melhor arquitetura e a que se encaixa na necessidade atual, nao a que funcionou para Netflix ou Amazon.

#### Monolith Tradicional

**Quando usar:**
- Equipe de 1-5 desenvolvedores
- MVP ou prototipo rapido
- Dominio de negocio simples e bem compreendido
- Time-to-market e prioridade absoluta

**Vantagens:**
- Deploy simples (um artefato)
- Debugging direto (stack trace unico)
- Sem overhead de comunicacao entre servicos
- Ideal para validacao de produto

**Desvantagens:**
- Acoplamento tende a crescer sem disciplina
- Scaling e tudo-ou-nada
- Um deploy quebrado afeta tudo

#### Modular Monolith (O Padrao de Ouro para 2025-2026)

O **Modular Monolith** emergiu como o padrao dominante. Combina a simplicidade do monolito com as boas praticas de microservices. A aplicacao roda como um unico sistema, mas e claramente dividida em modulos independentes com fronteiras bem definidas.

**Quando usar:**
- Equipe de 3-15 desenvolvedores
- SaaS, plataformas, aplicacoes de medio porte
- Quando voce quer modularidade sem complexidade distribuida
- Caminho evolutivo — modulos podem virar microservices depois

**Estrutura:**

```
src/
  modules/
    auth/
      domain/        # Entidades, Value Objects
      application/   # Use Cases, Services
      infrastructure/ # Repos, APIs externas
      presentation/  # Controllers, Routes
      index.ts       # Public API do modulo
    billing/
      ...
    notifications/
      ...
  shared/
    kernel/          # Tipos compartilhados, Events
    infrastructure/  # Database, Logging, Config
```

**Principios-chave:**
1. Cada modulo expoe APENAS uma public API (barrel export)
2. Modulos NUNCA importam diretamente de outro modulo — usam a public API
3. Comunicacao entre modulos via Domain Events (desacoplamento)
4. Cada modulo pode ter seu proprio schema de banco (schema-per-module)
5. Um modulo pode ser extraido para microservice quando necessario

**Casos reais:** Shopify comecou como monolito e evoluiu para modular monolith. Extraem microservices apenas para necessidades especificas (checkout, deteccao de fraude). O GitHub permanece como um monolito Ruby on Rails servindo milhoes de desenvolvedores.

#### Microservices

**Quando usar:**
- Equipe de 10+ desenvolvedores (beneficios so aparecem acima desse numero)
- Necessidade real de scaling independente por servico
- Deploys independentes e frequentes por times diferentes
- Dominios de negocio muito distintos com ciclos de vida diferentes

**Custo real:**
- Service discovery, load balancing, circuit breakers
- Distributed tracing (OpenTelemetry), centralized logging
- Event bus/message broker (Kafka, RabbitMQ, SQS)
- Consistency eventual (saga pattern, compensating transactions)
- DevOps overhead significativo (Kubernetes, service mesh)

**FINDING:** Microservices trazem alto custo de coordenacao, deployment e seguranca. Muitas grandes empresas estao retornando para modular monoliths.

**IMPLICATION:** Para a maioria dos projetos SINAPSE (equipes pequenas, time-to-market rapido), microservices sao prematuros.

**RECOMMENDATION:** Comece SEMPRE com Modular Monolith. Extraia microservices apenas quando houver necessidade comprovada de scaling independente.

#### Decision Tree

```
Projeto novo?
  |
  +-- Equipe <= 5 devs?
  |     +-- SIM --> Monolith (com boa estrutura de pastas)
  |     +-- NAO --> Modular Monolith
  |
  +-- Equipe > 10 devs?
  |     +-- Times independentes com dominios distintos?
  |           +-- SIM --> Microservices (seletivo)
  |           +-- NAO --> Modular Monolith
  |
  +-- Necessidade de scaling independente comprovada?
        +-- SIM --> Extrair modulo especifico para microservice
        +-- NAO --> Manter no Modular Monolith
```

### Clean Architecture (Uncle Bob)

Clean Architecture organiza o codigo em camadas concentricas onde a **regra de dependencia** aponta sempre para dentro — camadas externas dependem das internas, nunca o contrario.

```
        +---------------------------------------+
        |           Frameworks & Drivers        |  (Express, Next.js, Prisma, Supabase)
        |   +-------------------------------+   |
        |   |     Interface Adapters        |   |  (Controllers, Presenters, Gateways)
        |   |   +-----------------------+   |   |
        |   |   |   Application Layer   |   |   |  (Use Cases, Application Services)
        |   |   |   +---------------+   |   |   |
        |   |   |   |    Domain     |   |   |   |  (Entities, Value Objects, Domain Events)
        |   |   |   +---------------+   |   |   |
        |   |   +-----------------------+   |   |
        |   +-------------------------------+   |
        +---------------------------------------+
```

**Camadas em TypeScript:**

```typescript
// === DOMAIN LAYER (centro) ===
// Entidades puras, sem dependencias externas
class Order {
  constructor(
    public readonly id: OrderId,
    public readonly items: OrderItem[],
    public readonly status: OrderStatus,
  ) {}

  get total(): Money {
    return this.items.reduce((sum, item) => sum.add(item.subtotal), Money.zero());
  }

  confirm(): Order {
    if (this.status !== OrderStatus.Pending) {
      throw new DomainError('Only pending orders can be confirmed');
    }
    return new Order(this.id, this.items, OrderStatus.Confirmed);
  }
}

// === APPLICATION LAYER ===
// Use Cases orquestram a logica de negocio
interface OrderRepository {
  findById(id: OrderId): Promise<Order | null>;
  save(order: Order): Promise<void>;
}

class ConfirmOrderUseCase {
  constructor(
    private readonly orderRepo: OrderRepository,
    private readonly eventBus: EventBus,
  ) {}

  async execute(orderId: string): Promise<Result<void, AppError>> {
    const order = await this.orderRepo.findById(new OrderId(orderId));
    if (!order) return err(new NotFoundError('Order'));

    const confirmed = order.confirm();
    await this.orderRepo.save(confirmed);
    await this.eventBus.publish(new OrderConfirmedEvent(confirmed));

    return ok(undefined);
  }
}

// === INFRASTRUCTURE LAYER (borda) ===
// Implementacao concreta do repository
class PrismaOrderRepository implements OrderRepository {
  constructor(private readonly prisma: PrismaClient) {}

  async findById(id: OrderId): Promise<Order | null> {
    const data = await this.prisma.order.findUnique({
      where: { id: id.value },
      include: { items: true },
    });
    return data ? OrderMapper.toDomain(data) : null;
  }

  async save(order: Order): Promise<void> {
    await this.prisma.order.upsert({
      where: { id: order.id.value },
      create: OrderMapper.toPersistence(order),
      update: OrderMapper.toPersistence(order),
    });
  }
}
```

### Hexagonal Architecture (Ports & Adapters)

Hexagonal Architecture (ou Ports & Adapters) separa o core business logic de preocupacoes externas. **Ports** sao contratos (interfaces) que definem como o dominio comunica. **Adapters** sao implementacoes substituiveis.

```
                  Driving Adapters (entrada)
                  [REST API] [GraphQL] [CLI]
                       |         |       |
                       v         v       v
                  +----[PORTS (interfaces)]----+
                  |                            |
                  |      APPLICATION CORE      |
                  |    (Domain + Use Cases)     |
                  |                            |
                  +----[PORTS (interfaces)]----+
                       |         |       |
                       v         v       v
                  Driven Adapters (saida)
                  [Postgres] [Redis] [S3] [Email]
```

**Driving Adapters** (inbound): mapeiam dados externos para o formato interno e invocam a aplicacao (ex: um controller REST que chama um use case).

**Driven Adapters** (outbound): implementam as interfaces de port e sao chamados pela aplicacao (ex: um repository Postgres que implementa `OrderRepository`).

**Vantagem critica:** Testing se torna trivial — fake adapters permitem testar em isolamento. Trocar Postgres por MongoDB, Express por Fastify, sem tocar na logica de negocio.

### Domain-Driven Design (DDD)

DDD e a abordagem para modelar software complexo alinhando o codigo ao dominio de negocio.

#### Strategic DDD (comece por aqui)

1. **Bounded Contexts** — fronteiras dentro das quais um modelo de dominio e valido e consistente. Cada contexto tem sua propria Ubiquitous Language.

```
[Bounded Context: Orders]     [Bounded Context: Inventory]
  - Order (entity)              - Product (entity)
  - OrderItem (entity)          - Stock (entity)
  - OrderPlaced (event)   -->   - ReserveStock (command)
```

2. **Context Mapping** — como bounded contexts se comunicam:
   - **Shared Kernel**: codigo/modelos compartilhados
   - **Customer-Supplier**: um contexto serve o outro
   - **Anti-Corruption Layer**: traduz entre modelos
   - **Published Language**: contrato formal (API schema)

3. **Ubiquitous Language** — vocabulario compartilhado entre devs e negocio. Se o negocio diz "confirmar pedido", o codigo deve ter `order.confirm()`, nao `order.updateStatus(3)`.

#### Tactical DDD

1. **Entities** — objetos com identidade unica que persiste ao longo do tempo
2. **Value Objects** — objetos imutaveis definidos por seus atributos (Money, Email, Address)
3. **Aggregates** — cluster de entidades tratado como unidade para mudancas de dados

```typescript
// Value Object
class Money {
  constructor(
    public readonly amount: number,
    public readonly currency: Currency,
  ) {
    if (amount < 0) throw new DomainError('Money cannot be negative');
  }

  add(other: Money): Money {
    if (this.currency !== other.currency) throw new DomainError('Currency mismatch');
    return new Money(this.amount + other.amount, this.currency);
  }

  equals(other: Money): boolean {
    return this.amount === other.amount && this.currency === other.currency;
  }
}

// Aggregate Root
class Order {
  private _items: OrderItem[] = [];
  private _domainEvents: DomainEvent[] = [];

  // Aggregate controla acesso ao estado interno
  addItem(product: ProductId, quantity: number, price: Money): void {
    const item = new OrderItem(product, quantity, price);
    this._items.push(item);
    this._domainEvents.push(new OrderItemAddedEvent(this.id, item));
  }

  // Invariantes sao protegidas no aggregate
  confirm(): void {
    if (this._items.length === 0) {
      throw new DomainError('Cannot confirm order without items');
    }
    this._status = OrderStatus.Confirmed;
    this._domainEvents.push(new OrderConfirmedEvent(this.id, this.total));
  }
}
```

4. **Domain Events** — registram que algo aconteceu no dominio
5. **Repositories** — interface para persistir/recuperar aggregates
6. **Domain Services** — logica que nao pertence a nenhuma entity/value object

**RECOMMENDATION:** Strategic DDD (Bounded Contexts) deve vir PRIMEIRO. Tactical patterns (Aggregates, Value Objects) vem depois. Nao use DDD para CRUDs simples — o overhead nao se justifica.

### Event-Driven Architecture

#### Domain Events (local)

Comunicacao assincrona DENTRO de um bounded context ou entre modulos:

```typescript
// Event
class OrderConfirmedEvent {
  constructor(
    public readonly orderId: string,
    public readonly total: number,
    public readonly occurredAt: Date = new Date(),
  ) {}
}

// Event Bus (in-process)
class EventBus {
  private handlers = new Map<string, Function[]>();

  subscribe(eventName: string, handler: Function): void {
    const existing = this.handlers.get(eventName) || [];
    this.handlers.set(eventName, [...existing, handler]);
  }

  async publish(event: DomainEvent): Promise<void> {
    const handlers = this.handlers.get(event.constructor.name) || [];
    await Promise.all(handlers.map(h => h(event)));
  }
}
```

#### CQRS (Command Query Responsibility Segregation)

Separa modelos de **escrita** (Commands) e **leitura** (Queries):

```typescript
// Command side — otimizado para consistencia
class CreateOrderCommand {
  constructor(
    public readonly customerId: string,
    public readonly items: Array<{ productId: string; quantity: number }>,
  ) {}
}

class CreateOrderHandler {
  async handle(command: CreateOrderCommand): Promise<Result<string, AppError>> {
    // Logica de dominio complexa, validacoes, eventos
    const order = Order.create(command.customerId, command.items);
    await this.orderRepo.save(order);
    return ok(order.id);
  }
}

// Query side — otimizado para performance de leitura
class GetOrderSummaryQuery {
  constructor(public readonly orderId: string) {}
}

class GetOrderSummaryHandler {
  async handle(query: GetOrderSummaryQuery): Promise<OrderSummaryDTO> {
    // View otimizada, pode usar cache, materialized views, etc.
    return this.readDb.query(`SELECT ... FROM order_summary WHERE id = $1`, [query.orderId]);
  }
}
```

#### Event Sourcing

Em vez de salvar o estado atual, salva todos os eventos que levaram ao estado:

```typescript
// Event Store
interface EventStore {
  append(streamId: string, events: DomainEvent[]): Promise<void>;
  getStream(streamId: string): Promise<DomainEvent[]>;
}

// Reconstruir estado a partir de eventos
function rehydrate(events: DomainEvent[]): Order {
  let order = Order.empty();
  for (const event of events) {
    order = order.apply(event);
  }
  return order;
}
```

**Quando usar Event Sourcing:** audit trail completo, analytics comportamental, read models flexiveis. **Quando NAO usar:** CRUDs simples, equipes sem experiencia em sistemas distribuidos.

### Serverless Patterns

```
Vercel Edge Functions — Para logica leve proximo ao usuario (auth, redirects, A/B testing)
Supabase Edge Functions — Para logica backend que precisa de acesso ao DB
AWS Lambda — Para jobs pesados, processamento async
```

**Padroes uteis:**
- **API Gateway + Lambda**: cada endpoint e uma funcao isolada
- **Fan-out/Fan-in**: uma funcao dispara varias em paralelo e consolida resultados
- **Event-driven processing**: SQS/EventBridge trigger funcoes
- **Scheduled jobs**: cron via CloudWatch Events ou Supabase pg_cron

### JAMstack / Edge-First

```
Static (CDN) + Edge (Middleware, Edge Functions) + API (Serverless, BaaS)
```

O modelo mental do Next.js App Router em 2026: "O que o usuario deve ver IMEDIATAMENTE e o que pode esperar?" — essa e a essencia de partial prerendering e streaming.

### Decision Tree por Tipo de Projeto

| Tipo de Projeto | Arquitetura Recomendada |
|----------------|------------------------|
| Landing Page | JAMstack (SSG + Edge) |
| SaaS B2B | Modular Monolith + Clean Architecture |
| E-commerce | Modular Monolith + CQRS (leitura otimizada) |
| Fintech | Modular Monolith + DDD + Event Sourcing |
| Marketplace | Modular Monolith evoluindo para microservices seletivos |
| Blog/Content | JAMstack (SSG + ISR) |
| Real-time App | Event-Driven + WebSockets + Edge |

---

## 1.2 Code Quality Patterns

### SOLID Principles em TypeScript

#### S — Single Responsibility Principle

```typescript
// ERRADO: classe faz tudo
class UserService {
  createUser(data: UserData) { /* cria usuario */ }
  sendWelcomeEmail(user: User) { /* envia email */ }
  generateReport(users: User[]) { /* gera relatorio */ }
}

// CORRETO: cada classe tem uma responsabilidade
class UserCreator {
  constructor(
    private readonly userRepo: UserRepository,
    private readonly eventBus: EventBus,
  ) {}

  async create(data: UserData): Promise<Result<User, ValidationError>> {
    const user = User.create(data);
    await this.userRepo.save(user);
    await this.eventBus.publish(new UserCreatedEvent(user));
    return ok(user);
  }
}

class WelcomeEmailSender {
  constructor(private readonly emailService: EmailService) {}

  async onUserCreated(event: UserCreatedEvent): Promise<void> {
    await this.emailService.send(WelcomeEmail.for(event.user));
  }
}
```

#### O — Open/Closed Principle

```typescript
// Aberto para extensao, fechado para modificacao
interface PaymentProcessor {
  process(payment: Payment): Promise<Result<PaymentResult, PaymentError>>;
}

class StripeProcessor implements PaymentProcessor {
  async process(payment: Payment): Promise<Result<PaymentResult, PaymentError>> {
    // Implementacao Stripe
  }
}

class PaddleProcessor implements PaymentProcessor {
  async process(payment: Payment): Promise<Result<PaymentResult, PaymentError>> {
    // Implementacao Paddle — adicionado sem modificar codigo existente
  }
}

// Factory seleciona o processador correto
class PaymentProcessorFactory {
  private processors = new Map<string, PaymentProcessor>();

  register(type: string, processor: PaymentProcessor): void {
    this.processors.set(type, processor);
  }

  get(type: string): PaymentProcessor {
    const processor = this.processors.get(type);
    if (!processor) throw new Error(`Unknown payment type: ${type}`);
    return processor;
  }
}
```

#### L — Liskov Substitution Principle

```typescript
// Subtipos devem ser substituiveis por seus tipos base
abstract class Shape {
  abstract area(): number;
}

class Rectangle extends Shape {
  constructor(protected width: number, protected height: number) { super(); }
  area(): number { return this.width * this.height; }
}

class Square extends Shape {
  constructor(private side: number) { super(); }
  area(): number { return this.side * this.side; }
}
// Square NAO estende Rectangle — evita violacao de LSP
```

#### I — Interface Segregation Principle

```typescript
// ERRADO: interface gorda
interface Worker {
  work(): void;
  eat(): void;
  sleep(): void;
}

// CORRETO: interfaces segregadas
interface Workable { work(): void; }
interface Feedable { eat(): void; }
interface Restable { sleep(): void; }

class HumanWorker implements Workable, Feedable, Restable {
  work() { /* ... */ }
  eat() { /* ... */ }
  sleep() { /* ... */ }
}

class RobotWorker implements Workable {
  work() { /* ... */ }
  // Nao precisa implementar eat() ou sleep()
}
```

#### D — Dependency Inversion Principle

```typescript
// ERRADO: high-level depende de low-level
class OrderService {
  private db = new PostgresDatabase(); // acoplamento direto
}

// CORRETO: ambos dependem de abstracoes
interface Database {
  query<T>(sql: string, params?: unknown[]): Promise<T[]>;
}

class OrderService {
  constructor(private readonly db: Database) {} // injecao de dependencia
}

// Na composicao root (container DI):
const db: Database = new PostgresDatabase(config);
const orderService = new OrderService(db);
```

### Design Patterns Essenciais para Web Moderno

#### Strategy Pattern

```typescript
// Selecionar algoritmo em runtime
interface CacheStrategy {
  get<T>(key: string): Promise<T | null>;
  set<T>(key: string, value: T, ttl?: number): Promise<void>;
}

class RedisCacheStrategy implements CacheStrategy { /* ... */ }
class InMemoryCacheStrategy implements CacheStrategy { /* ... */ }
class NoCacheStrategy implements CacheStrategy {
  async get() { return null; }
  async set() { /* no-op */ }
}

class DataFetcher {
  constructor(private cache: CacheStrategy) {}

  async fetch<T>(key: string, fetcher: () => Promise<T>): Promise<T> {
    const cached = await this.cache.get<T>(key);
    if (cached) return cached;
    const data = await fetcher();
    await this.cache.set(key, data);
    return data;
  }
}
```

#### Repository Pattern

```typescript
interface Repository<T, ID> {
  findById(id: ID): Promise<T | null>;
  findAll(filter?: Partial<T>): Promise<T[]>;
  save(entity: T): Promise<void>;
  delete(id: ID): Promise<void>;
}

class UserRepository implements Repository<User, UserId> {
  constructor(private readonly supabase: SupabaseClient) {}

  async findById(id: UserId): Promise<User | null> {
    const { data, error } = await this.supabase
      .from('users')
      .select('*')
      .eq('id', id.value)
      .single();
    if (error || !data) return null;
    return UserMapper.toDomain(data);
  }
  // ...
}
```

#### Observer Pattern (Event Emitter)

```typescript
type EventHandler<T = unknown> = (payload: T) => void | Promise<void>;

class TypedEventEmitter {
  private handlers = new Map<string, Set<EventHandler>>();

  on<T>(event: string, handler: EventHandler<T>): () => void {
    if (!this.handlers.has(event)) {
      this.handlers.set(event, new Set());
    }
    this.handlers.get(event)!.add(handler as EventHandler);
    return () => this.handlers.get(event)?.delete(handler as EventHandler);
  }

  async emit<T>(event: string, payload: T): Promise<void> {
    const handlers = this.handlers.get(event);
    if (!handlers) return;
    await Promise.allSettled([...handlers].map(h => h(payload)));
  }
}
```

#### Middleware Pattern

```typescript
type Middleware<T> = (context: T, next: () => Promise<void>) => Promise<void>;

class Pipeline<T> {
  private middlewares: Middleware<T>[] = [];

  use(middleware: Middleware<T>): this {
    this.middlewares.push(middleware);
    return this;
  }

  async execute(context: T): Promise<void> {
    let index = 0;
    const next = async (): Promise<void> => {
      if (index < this.middlewares.length) {
        const middleware = this.middlewares[index++];
        await middleware(context, next);
      }
    };
    await next();
  }
}
```

### TypeScript Best Practices

#### Strict Mode (obrigatorio)

```json
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "forceConsistentCasingInFileNames": true
  }
}
```

#### Branded Types

```typescript
// Evitar confundir IDs de tipos diferentes
type UserId = string & { readonly __brand: 'UserId' };
type OrderId = string & { readonly __brand: 'OrderId' };

function createUserId(value: string): UserId { return value as UserId; }
function createOrderId(value: string): OrderId { return value as OrderId; }

// Agora TypeScript impede: findOrder(userId) -- erro de tipo!
function findOrder(id: OrderId): Promise<Order | null> { /* ... */ }
```

#### Discriminated Unions

```typescript
type ApiResponse<T> =
  | { status: 'success'; data: T }
  | { status: 'error'; error: { code: string; message: string } }
  | { status: 'loading' };

function handleResponse<T>(response: ApiResponse<T>) {
  switch (response.status) {
    case 'success':
      return response.data; // TypeScript sabe que data existe aqui
    case 'error':
      throw new Error(response.error.message);
    case 'loading':
      return null;
  }
}
```

#### Zod Schemas (validacao runtime)

```typescript
import { z } from 'zod';

const CreateUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(2).max(100),
  role: z.enum(['admin', 'user', 'viewer']),
  metadata: z.record(z.string()).optional(),
});

type CreateUserInput = z.infer<typeof CreateUserSchema>;
// Tipo inferido automaticamente do schema — single source of truth
```

### Error Handling com Result Type

```typescript
// Result type — inspirado em Rust
type Result<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E };

function ok<T>(value: T): Result<T, never> {
  return { ok: true, value };
}

function err<E>(error: E): Result<never, E> {
  return { ok: false, error };
}

// Uso: impossivel acessar valor sem tratar erro
async function createUser(input: CreateUserInput): Promise<Result<User, ValidationError | ConflictError>> {
  const existing = await userRepo.findByEmail(input.email);
  if (existing) return err(new ConflictError('Email already in use'));

  const validation = CreateUserSchema.safeParse(input);
  if (!validation.success) return err(new ValidationError(validation.error));

  const user = User.create(validation.data);
  await userRepo.save(user);
  return ok(user);
}

// Consumidor DEVE verificar antes de usar
const result = await createUser(data);
if (!result.ok) {
  // TypeScript sabe que result.error existe
  logger.error('Failed to create user', { error: result.error });
  return;
}
// TypeScript sabe que result.value e User
console.log(result.value.name);
```

**Biblioteca recomendada:** `neverthrow` — Result type ergonomico com chaining, async support e match patterns.

### Logging Patterns

```typescript
// Structured logging com correlation IDs
interface Logger {
  info(message: string, context?: Record<string, unknown>): void;
  warn(message: string, context?: Record<string, unknown>): void;
  error(message: string, context?: Record<string, unknown>): void;
}

// Correlation ID para rastrear requests end-to-end
class CorrelatedLogger implements Logger {
  constructor(
    private readonly logger: Logger,
    private readonly correlationId: string,
  ) {}

  info(message: string, context?: Record<string, unknown>): void {
    this.logger.info(message, { ...context, correlationId: this.correlationId });
  }
  // ...
}

// Middleware para Next.js
export function middleware(request: NextRequest) {
  const correlationId = request.headers.get('x-correlation-id') || crypto.randomUUID();
  const response = NextResponse.next();
  response.headers.set('x-correlation-id', correlationId);
  return response;
}
```

---

## 1.3 Frontend Engineering

### Next.js App Router Architecture (2025-2026)

O App Router e a arquitetura padrao para aplicacoes React em producao. O modelo server-first muda fundamentalmente como voce estrutura, renderiza e entrega aplicacoes web.

#### Mental Model: Server-Client Boundary

**Regra:** Todo componente e Server Component por padrao. Roda APENAS no servidor, ZERO JavaScript enviado ao browser, acesso direto a banco de dados, file system e variaveis de ambiente.

```typescript
// Server Component (default) — sem 'use client'
async function ProductList() {
  const products = await db.product.findMany(); // acesso direto ao DB
  return (
    <div>
      {products.map(p => (
        <ProductCard key={p.id} product={p} />
      ))}
    </div>
  );
}

// Client Component — precisa de 'use client'
'use client';
function AddToCartButton({ productId }: { productId: string }) {
  const [isPending, startTransition] = useTransition();
  return (
    <button onClick={() => startTransition(() => addToCart(productId))}>
      {isPending ? 'Adding...' : 'Add to Cart'}
    </button>
  );
}
```

**Insight arquitetural:** Client Components podem renderizar Server Components passados como `children`. Isso permite manter o client boundary o menor possivel:

```tsx
// Layout com client wrapper minimo
'use client';
function InteractiveLayout({ children }: { children: React.ReactNode }) {
  const [isOpen, setIsOpen] = useState(false);
  return (
    <div>
      <button onClick={() => setIsOpen(!isOpen)}>Toggle</button>
      {isOpen && children} {/* children sao Server Components! */}
    </div>
  );
}
```

#### Estrutura de Pastas Recomendada

```
app/
  (auth)/
    login/page.tsx
    register/page.tsx
    layout.tsx           # Layout compartilhado de auth
  (dashboard)/
    dashboard/page.tsx
    settings/page.tsx
    layout.tsx           # Layout com sidebar
  api/
    webhooks/
      stripe/route.ts
  layout.tsx             # Root layout
  error.tsx              # Global error boundary
  loading.tsx            # Global loading
  not-found.tsx

src/
  components/
    ui/                  # Design system (Button, Input, Card)
    features/            # Feature-specific components
    layouts/             # Layout components
  lib/
    actions/             # Server Actions
    db/                  # Database client, queries
    auth/                # Auth utilities
    validations/         # Zod schemas
  hooks/                 # Custom hooks
  types/                 # TypeScript types
  utils/                 # Pure utility functions
```

#### Server Actions

```typescript
// lib/actions/order.ts
'use server';

import { revalidatePath } from 'next/cache';
import { z } from 'zod';

const CreateOrderSchema = z.object({
  items: z.array(z.object({
    productId: z.string().uuid(),
    quantity: z.number().int().positive(),
  })),
});

export async function createOrder(formData: FormData) {
  const session = await getSession();
  if (!session) throw new Error('Unauthorized');

  const raw = Object.fromEntries(formData);
  const parsed = CreateOrderSchema.safeParse(JSON.parse(raw.data as string));
  if (!parsed.success) return { error: parsed.error.flatten() };

  const order = await db.order.create({
    data: { userId: session.userId, items: parsed.data.items },
  });

  revalidatePath('/orders');
  return { success: true, orderId: order.id };
}
```

### State Management (2025-2026)

A abordagem moderna usa **dois tools** para 90% dos casos:

| Tipo de Estado | Ferramenta | Quando |
|---------------|-----------|--------|
| Server State | **TanStack Query** | Dados vindos de APIs, cache, revalidacao |
| Client State | **Zustand** | UI state leve, global sem boilerplate |
| URL State | **nuqs** | Filtros, paginacao, tabs, search params |
| Form State | **React Hook Form** | Formularios complexos |
| React State | `useState/useReducer` | Estado local simples do componente |

```typescript
// TanStack Query — server state
const { data, isLoading, error } = useQuery({
  queryKey: ['products', filters],
  queryFn: () => fetchProducts(filters),
  staleTime: 5 * 60 * 1000, // 5 min
});

// Zustand — client state
const useCartStore = create<CartStore>((set) => ({
  items: [],
  addItem: (item) => set((state) => ({ items: [...state.items, item] })),
  removeItem: (id) => set((state) => ({ items: state.items.filter(i => i.id !== id) })),
  total: 0,
}));

// nuqs — URL state
const [search, setSearch] = useQueryState('search', parseAsString.withDefault(''));
const [page, setPage] = useQueryState('page', parseAsInteger.withDefault(1));
```

**FINDING:** Aplicacoes usando TanStack Query mostram 40-60% menos requests de rede comparado com implementacoes Redux para as mesmas features.

**RECOMMENDATION:** Stack padrao: TanStack Query + nuqs + Zustand. Redux APENAS em projetos legados existentes.

### Component Architecture

#### Compound Components

```tsx
// Componentes que trabalham juntos como uma unidade
function Tabs({ children, defaultValue }: TabsProps) {
  const [active, setActive] = useState(defaultValue);
  return (
    <TabsContext.Provider value={{ active, setActive }}>
      {children}
    </TabsContext.Provider>
  );
}

Tabs.List = function TabsList({ children }: { children: React.ReactNode }) {
  return <div role="tablist">{children}</div>;
};

Tabs.Trigger = function TabsTrigger({ value, children }: TabsTriggerProps) {
  const { active, setActive } = useTabsContext();
  return (
    <button
      role="tab"
      aria-selected={active === value}
      onClick={() => setActive(value)}
    >
      {children}
    </button>
  );
};

Tabs.Content = function TabsContent({ value, children }: TabsContentProps) {
  const { active } = useTabsContext();
  if (active !== value) return null;
  return <div role="tabpanel">{children}</div>;
};

// Uso
<Tabs defaultValue="tab1">
  <Tabs.List>
    <Tabs.Trigger value="tab1">Tab 1</Tabs.Trigger>
    <Tabs.Trigger value="tab2">Tab 2</Tabs.Trigger>
  </Tabs.List>
  <Tabs.Content value="tab1">Content 1</Tabs.Content>
  <Tabs.Content value="tab2">Content 2</Tabs.Content>
</Tabs>
```

#### Headless Components

Separar logica de rendering. Bibliotecas como **Radix UI** e **Ark UI** fornecem logica complexa sem styling, deixando a apresentacao visual completamente por sua conta.

```tsx
// Headless hook — logica sem UI
function useToggle(initial = false) {
  const [isOpen, setIsOpen] = useState(initial);
  return {
    isOpen,
    open: () => setIsOpen(true),
    close: () => setIsOpen(false),
    toggle: () => setIsOpen(prev => !prev),
    getToggleProps: () => ({
      onClick: () => setIsOpen(prev => !prev),
      'aria-expanded': isOpen,
    }),
    getContentProps: () => ({
      hidden: !isOpen,
    }),
  };
}
```

### Performance (Core Web Vitals)

**Metricas alvo:**
- **LCP** (Largest Contentful Paint): < 2.5s
- **INP** (Interaction to Next Paint): < 200ms
- **CLS** (Cumulative Layout Shift): < 0.1

#### Otimizacoes criticas:

**1. Bundle Splitting:**
```tsx
// Route-based splitting (automatico no Next.js)
// Component-level lazy loading para componentes pesados
const HeavyChart = dynamic(() => import('./HeavyChart'), {
  loading: () => <ChartSkeleton />,
  ssr: false, // Nao renderizar no servidor se usa APIs de browser
});
```

**2. Image Optimization:**
```tsx
import Image from 'next/image';

<Image
  src="/hero.webp"
  width={1200}
  height={600}
  alt="Hero image"
  priority        // LCP image — carrega imediatamente
  sizes="(max-width: 768px) 100vw, 50vw"
/>
```

**3. Font Optimization:**
```tsx
import { Inter } from 'next/font/google';
const inter = Inter({ subsets: ['latin'], display: 'swap' });
```

**4. Tree-shaking:**
```typescript
// Importar APENAS o que precisa
import { debounce } from 'lodash-es'; // NAO: import _ from 'lodash'
```

**5. Prefetching:**
```tsx
<Link href="/products" prefetch={true}>Products</Link>
```

### Acessibilidade (WCAG 2.2)

WCAG 2.2 (outubro 2025) e o padrao atual. 94.8% dos sites ainda falham — ser acessivel e diferencial competitivo.

**Checklist essencial:**

```tsx
// 1. Semantic HTML primeiro, ARIA so quando necessario
<nav aria-label="Main navigation">
  <ul>
    <li><a href="/">Home</a></li>
  </ul>
</nav>

// 2. Keyboard navigation
<button onKeyDown={(e) => {
  if (e.key === 'Enter' || e.key === ' ') handleAction();
}}>

// 3. Focus management
const dialogRef = useRef<HTMLDialogElement>(null);
useEffect(() => {
  if (isOpen) dialogRef.current?.focus();
}, [isOpen]);

// 4. Screen reader text
<span className="sr-only">Close dialog</span>

// 5. Color contrast — ratio minimo 4.5:1 para texto normal
// 6. Touch targets — minimo 44x44px (WCAG 2.2 novo criterio)
// 7. Reduced motion
@media (prefers-reduced-motion: reduce) {
  * { animation-duration: 0.01ms !important; }
}
```

### Form Handling (React Hook Form + Zod + Server Actions)

```tsx
'use client';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

// Schema compartilhado entre client e server
const ContactSchema = z.object({
  name: z.string().min(2, 'Nome muito curto'),
  email: z.string().email('Email invalido'),
  message: z.string().min(10, 'Mensagem muito curta'),
});

type ContactFormData = z.infer<typeof ContactSchema>;

export function ContactForm() {
  const { register, handleSubmit, formState: { errors, isSubmitting } } = useForm<ContactFormData>({
    resolver: zodResolver(ContactSchema),
  });

  async function onSubmit(data: ContactFormData) {
    const formData = new FormData();
    Object.entries(data).forEach(([key, value]) => formData.append(key, value));
    await submitContact(formData); // Server Action
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('name')} aria-invalid={!!errors.name} />
      {errors.name && <span role="alert">{errors.name.message}</span>}

      <input {...register('email')} type="email" aria-invalid={!!errors.email} />
      {errors.email && <span role="alert">{errors.email.message}</span>}

      <textarea {...register('message')} aria-invalid={!!errors.message} />
      {errors.message && <span role="alert">{errors.message.message}</span>}

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Enviando...' : 'Enviar'}
      </button>
    </form>
  );
}
```

---

## 1.4 Backend Engineering

### API Design — REST vs GraphQL vs tRPC

| Criterio | REST | GraphQL | tRPC |
|----------|------|---------|------|
| **Melhor para** | APIs publicas, multi-linguagem | UIs complexas, dados flexiveis | Full-stack TypeScript |
| **Type Safety** | Manual (OpenAPI/Swagger) | Schema-based | End-to-end automatico |
| **Learning curve** | Baixa | Media | Baixa (se TypeScript) |
| **Over/Under fetching** | Comum | Resolvido | N/A (RPC) |
| **Caching** | HTTP nativo | Complexo | Customizado |
| **Quando usar** | API publica, backward compatibility | Multi-client com dados variados | Monorepo TypeScript, Next.js |

**RECOMMENDATION para SINAPSE:**
- **Projetos internos/SaaS:** tRPC (type safety end-to-end, zero codegen)
- **APIs publicas:** REST com OpenAPI schema
- **Apps com dados complexos e multiplos clientes:** GraphQL

#### tRPC com Next.js

```typescript
// server/routers/product.ts
import { router, publicProcedure, protectedProcedure } from '../trpc';
import { z } from 'zod';

export const productRouter = router({
  list: publicProcedure
    .input(z.object({
      cursor: z.string().optional(),
      limit: z.number().min(1).max(50).default(20),
      category: z.string().optional(),
    }))
    .query(async ({ input }) => {
      const items = await db.product.findMany({
        take: input.limit + 1,
        cursor: input.cursor ? { id: input.cursor } : undefined,
        where: input.category ? { category: input.category } : undefined,
      });
      // Pagination cursor
      let nextCursor: string | undefined;
      if (items.length > input.limit) {
        const nextItem = items.pop();
        nextCursor = nextItem?.id;
      }
      return { items, nextCursor };
    }),

  create: protectedProcedure
    .input(CreateProductSchema)
    .mutation(async ({ input, ctx }) => {
      return db.product.create({ data: { ...input, createdBy: ctx.userId } });
    }),
});
```

### Authentication Patterns

#### Supabase Auth (recomendado para SINAPSE)

```typescript
// lib/auth.ts
import { createServerClient } from '@supabase/ssr';
import { cookies } from 'next/headers';

export async function getSession() {
  const cookieStore = await cookies();
  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll: () => cookieStore.getAll(),
        setAll: (cookiesToSet) => {
          cookiesToSet.forEach(({ name, value, options }) =>
            cookieStore.set(name, value, options)
          );
        },
      },
    }
  );

  const { data: { session } } = await supabase.auth.getSession();
  return session;
}
```

#### Row Level Security (RLS) com RBAC

```sql
-- Habilitar RLS em TODAS as tabelas com dados de usuario
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Policy: usuarios veem apenas seus pedidos
CREATE POLICY "users_own_orders"
ON orders FOR ALL
USING (auth.uid() = user_id);

-- RBAC via custom claims no JWT
CREATE POLICY "admin_all_orders"
ON orders FOR ALL
USING (
  (auth.jwt() -> 'app_metadata' ->> 'role') = 'admin'
);

-- Policy multi-tenant
CREATE POLICY "tenant_isolation"
ON orders FOR ALL
USING (
  tenant_id = (auth.jwt() -> 'app_metadata' ->> 'tenant_id')::uuid
);
```

### Caching Strategies

```
Browser Cache (Cache-Control headers)
  |
  v
CDN Cache (Vercel Edge, Cloudflare)
  |
  v
Application Cache (Redis / In-memory)
  |
  v
Database Query Cache (Connection pooling, prepared statements)
```

**ISR + CDN:**
```typescript
// Next.js — Incremental Static Regeneration
export const revalidate = 60; // Revalida a cada 60 segundos

// Stale-while-revalidate pattern
// s-maxage=60 (CDN cache 60s)
// stale-while-revalidate=300 (serve stale por ate 5min enquanto revalida)
```

**Redis Cache:**
```typescript
class CachedProductService {
  constructor(
    private readonly redis: Redis,
    private readonly productRepo: ProductRepository,
  ) {}

  async getProduct(id: string): Promise<Product | null> {
    const cached = await this.redis.get(`product:${id}`);
    if (cached) return JSON.parse(cached);

    const product = await this.productRepo.findById(id);
    if (product) {
      await this.redis.setex(`product:${id}`, 300, JSON.stringify(product));
    }
    return product;
  }

  async invalidateProduct(id: string): Promise<void> {
    await this.redis.del(`product:${id}`);
    // Se usando CDN, purge tambem
    await this.cdnPurge(`/api/products/${id}`);
  }
}
```

### Rate Limiting

```typescript
import { Ratelimit } from '@upstash/ratelimit';
import { Redis } from '@upstash/redis';

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(10, '10s'), // 10 requests por 10s
  analytics: true,
});

// Em middleware Next.js
export async function middleware(request: NextRequest) {
  const ip = request.ip ?? '127.0.0.1';
  const { success, limit, remaining, reset } = await ratelimit.limit(ip);

  if (!success) {
    return new NextResponse('Too Many Requests', {
      status: 429,
      headers: {
        'X-RateLimit-Limit': limit.toString(),
        'X-RateLimit-Remaining': remaining.toString(),
        'X-RateLimit-Reset': reset.toString(),
      },
    });
  }
}
```

---

## 1.5 Testing Strategy

### Testing Pyramid (2025-2026)

```
           /\
          /  \
         / E2E \         ~10% — Playwright
        /________\       Fluxos criticos do usuario
       /          \
      / Integration \    ~30% — Vitest + MSW
     /______________\    API + componentes com dados
    /                \
   /     Unit Tests   \  ~60% — Vitest
  /____________________\ Logica de negocio, utils, hooks
```

### Stack Recomendada

| Camada | Ferramenta | Uso |
|--------|-----------|-----|
| Unit | **Vitest** | Funcoes puras, hooks, utils, domain logic |
| Component | **Vitest + Testing Library** | Componentes React com interacao |
| Integration | **Vitest + MSW** | API integration com mocks HTTP |
| E2E | **Playwright** | Fluxos criticos em browser real |
| Visual | **Storybook + Chromatic** | Regression visual de componentes |

### Exemplos Praticos

#### Unit Test (Vitest)

```typescript
import { describe, it, expect } from 'vitest';

describe('Money', () => {
  it('deve somar valores da mesma moeda', () => {
    const a = new Money(100, 'BRL');
    const b = new Money(50, 'BRL');
    const result = a.add(b);
    expect(result.amount).toBe(150);
    expect(result.currency).toBe('BRL');
  });

  it('deve rejeitar soma de moedas diferentes', () => {
    const brl = new Money(100, 'BRL');
    const usd = new Money(50, 'USD');
    expect(() => brl.add(usd)).toThrow('Currency mismatch');
  });

  it('deve rejeitar valores negativos', () => {
    expect(() => new Money(-1, 'BRL')).toThrow('Money cannot be negative');
  });
});
```

#### Component Test (Vitest + Testing Library)

```typescript
import { render, screen, userEvent } from '@testing-library/react';

describe('AddToCartButton', () => {
  it('deve adicionar item ao carrinho ao clicar', async () => {
    const user = userEvent.setup();
    const onAdd = vi.fn();

    render(<AddToCartButton productId="123" onAdd={onAdd} />);

    await user.click(screen.getByRole('button', { name: /add to cart/i }));

    expect(onAdd).toHaveBeenCalledWith('123');
  });
});
```

#### Integration Test (MSW)

```typescript
import { http, HttpResponse } from 'msw';
import { setupServer } from 'msw/node';

const server = setupServer(
  http.get('/api/products', () => {
    return HttpResponse.json([
      { id: '1', name: 'Product 1', price: 29.99 },
      { id: '2', name: 'Product 2', price: 49.99 },
    ]);
  }),
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

describe('ProductList', () => {
  it('deve exibir produtos da API', async () => {
    render(<ProductList />);
    expect(await screen.findByText('Product 1')).toBeInTheDocument();
    expect(await screen.findByText('Product 2')).toBeInTheDocument();
  });

  it('deve exibir erro quando API falha', async () => {
    server.use(
      http.get('/api/products', () => {
        return new HttpResponse(null, { status: 500 });
      }),
    );
    render(<ProductList />);
    expect(await screen.findByText(/error/i)).toBeInTheDocument();
  });
});
```

#### E2E Test (Playwright)

```typescript
import { test, expect } from '@playwright/test';

test.describe('Checkout Flow', () => {
  test('usuario deve completar compra', async ({ page }) => {
    await page.goto('/products');

    // Adicionar ao carrinho
    await page.click('[data-testid="product-1"] button');
    await expect(page.locator('[data-testid="cart-count"]')).toHaveText('1');

    // Ir para checkout
    await page.click('[data-testid="cart-icon"]');
    await page.click('text=Checkout');

    // Preencher dados
    await page.fill('[name="email"]', 'test@example.com');
    await page.fill('[name="card"]', '4242424242424242');
    await page.click('text=Pay');

    // Confirmacao
    await expect(page).toHaveURL(/\/order\/confirmation/);
    await expect(page.locator('h1')).toContainText('Order Confirmed');
  });
});
```

**MSW reuse:** Os mesmos mocks podem ser reutilizados entre Vitest, Playwright, Storybook e React Native.

---

## 1.6 DevOps & Deployment

### Pipeline CI/CD

```yaml
# .github/workflows/ci.yml
name: CI/CD Pipeline

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile

      # Lint -> Typecheck -> Test (em paralelo quando possivel)
      - run: pnpm lint
      - run: pnpm typecheck
      - run: pnpm test -- --coverage
      - run: pnpm build

  e2e:
    needs: quality
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - run: pnpm install --frozen-lockfile
      - run: npx playwright install --with-deps
      - run: pnpm test:e2e

  deploy-preview:
    needs: quality
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}

  deploy-production:
    needs: [quality, e2e]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-args: '--prod'
```

### Preview Deployments

Vercel gera uma URL unica de preview para cada PR — um deployment completo production-equivalent com suas proprias serverless functions, URL propria e acesso a variaveis de ambiente de staging. Isso permite:

- QA em ambiente real antes do merge
- Compartilhar com stakeholders para feedback
- Visual regression testing automatizado
- Performance testing por PR

### Feature Flags

```typescript
// Vercel Edge Config ou custom
import { get } from '@vercel/edge-config';

export async function isFeatureEnabled(flag: string, userId?: string): Promise<boolean> {
  const config = await get(flag);
  if (!config) return false;

  // Staged rollout
  if (config.percentage && userId) {
    const hash = hashString(userId + flag);
    return (hash % 100) < config.percentage;
  }

  return config.enabled === true;
}

// Uso em Server Component
async function ProductPage() {
  const showNewCheckout = await isFeatureEnabled('new-checkout');
  return showNewCheckout ? <NewCheckout /> : <LegacyCheckout />;
}
```

### Monitoring

| Ferramenta | Funcao |
|-----------|--------|
| **Sentry** | Error tracking, performance monitoring |
| **Vercel Analytics** | Core Web Vitals, Speed Insights |
| **LogRocket** / **PostHog** | Session replay, product analytics |
| **Uptime Robot** / **Better Uptime** | Uptime monitoring, alertas |
| **PlanetScale** / **Supabase Dashboard** | Database metrics |

---

# PARTE 2: ADVANCED WEB ANIMATIONS & 3D

## 2.1 Animation Fundamentals

### Performance Tiers de Animacao Web

| Tier | Propriedades | Performance | Uso |
|------|-------------|-------------|-----|
| **S-Tier** | `transform`, `opacity` | Hardware accelerated (compositor) | SEMPRE preferir |
| **A-Tier** | `filter`, `clip-path`, `background-color` | Precisa paint, mas sem layout | Aceitavel |
| **D-Tier** | `width`, `height`, `padding`, `margin`, `top`, `left` | Layout + paint + composite | EVITAR (mata FPS) |

**Regra de ouro:** Anime APENAS `transform` e `opacity` quando possivel. Essas propriedades rodam na GPU compositor thread, nao bloqueiam o main thread.

```css
/* BOM — hardware accelerated */
.element {
  transition: transform 0.3s ease, opacity 0.3s ease;
  will-change: transform; /* hint para o browser */
}
.element:hover {
  transform: translateY(-4px) scale(1.02);
  opacity: 0.9;
}

/* RUIM — causa layout thrashing */
.element:hover {
  top: -4px;      /* layout recalculation */
  width: 102%;    /* layout recalculation */
}
```

### 12 Principios de Animacao da Disney Aplicados a Web

1. **Squash & Stretch** — Botoes que "espremem" ao clicar
2. **Anticipation** — Hover state antes da acao principal
3. **Staging** — Direcionar atencao com animacao seletiva
4. **Straight Ahead & Pose to Pose** — Keyframes bem definidos
5. **Follow Through & Overlapping** — Elementos que "seguem" apos parada
6. **Ease In / Ease Out** — NUNCA use `linear` para UI
7. **Arcs** — Movimentos naturais seguem arcos, nao linhas retas
8. **Secondary Action** — Badge que pulsa enquanto conteudo aparece
9. **Timing** — 150-400ms e o sweet spot. < 150ms imperceptivel, > 400ms lento
10. **Exaggeration** — Enfatizar para comunicar
11. **Solid Drawing** — 3D consistente, sombras coerentes
12. **Appeal** — Personality que engaja

### Timing & Easing

```css
/* Easing recomendado por tipo de acao */
:root {
  /* Entrada — comecar rapido, desacelerar */
  --ease-out: cubic-bezier(0.16, 1, 0.3, 1);

  /* Saida — comecar devagar, acelerar */
  --ease-in: cubic-bezier(0.55, 0.055, 0.675, 0.19);

  /* Entrada + saida */
  --ease-in-out: cubic-bezier(0.87, 0, 0.13, 1);

  /* Spring-like — overshoot sutil */
  --ease-spring: cubic-bezier(0.34, 1.56, 0.64, 1);

  /* Duracoes por contexto */
  --duration-instant: 100ms;   /* Feedback imediato (hover) */
  --duration-fast: 200ms;      /* Micro-interacoes */
  --duration-normal: 300ms;    /* Transicoes padrao */
  --duration-slow: 500ms;      /* Transicoes complexas */
  --duration-enter: 400ms;     /* Elementos entrando */
  --duration-exit: 200ms;      /* Elementos saindo (mais rapido!) */
}
```

### View Transitions API

A View Transitions API e nativa do browser e permite animar transicoes entre views "tirando screenshot" da view antiga e crossfading para a nova.

```typescript
// Next.js com View Transitions
'use client';
import { useRouter } from 'next/navigation';

function NavigationLink({ href, children }: { href: string; children: React.ReactNode }) {
  const router = useRouter();

  const handleClick = (e: React.MouseEvent) => {
    e.preventDefault();
    if (!document.startViewTransition) {
      router.push(href);
      return;
    }
    document.startViewTransition(() => {
      router.push(href);
    });
  };

  return <a href={href} onClick={handleClick}>{children}</a>;
}
```

```css
/* CSS para View Transitions */
::view-transition-old(root) {
  animation: fade-out 0.25s ease-out;
}

::view-transition-new(root) {
  animation: fade-in 0.25s ease-in;
}

/* Transicao especifica para imagem de produto */
.product-image {
  view-transition-name: product-hero;
}
```

### Scroll-Driven Animations (CSS nativo)

```css
/* CSS Scroll Timeline — suportado em Chrome 115+ */
@keyframes reveal {
  from { opacity: 0; transform: translateY(50px); }
  to { opacity: 1; transform: translateY(0); }
}

.scroll-reveal {
  animation: reveal linear;
  animation-timeline: view();
  animation-range: entry 0% entry 100%;
}
```

---

## 2.2 JavaScript Animation Libraries

### Decision Matrix

| Criterio | GSAP | Framer Motion | Motion One | React Spring | Lenis |
|----------|------|--------------|------------|-------------|-------|
| **Bundle** | ~23KB gzip | ~32KB gzip | ~3.8KB gzip | ~16KB | ~5KB |
| **React integration** | Manual (refs) | Nativo | Ok | Nativo | Hook |
| **Scroll animations** | ScrollTrigger (melhor) | whileInView | IntersectionObserver | Basico | Smooth scroll |
| **Timeline** | Excelente | Basico | Basico | Nao | N/A |
| **Performance** | Melhor (bypassa React) | Bom | Excelente | Bom | Excelente |
| **Learning curve** | Alta | Baixa (React devs) | Baixa | Media | Baixa |
| **Melhor para** | Websites criativos | UI de produto | Micro-interacoes | Physics-based | Smooth scroll |

### GSAP (GreenSock Animation Platform)

O motor de animacao mais poderoso para web. Ideal para websites criativos, portfolios, agencias.

```typescript
import gsap from 'gsap';
import { ScrollTrigger } from 'gsap/ScrollTrigger';
import { SplitText } from 'gsap/SplitText';

gsap.registerPlugin(ScrollTrigger, SplitText);

// Timeline complexo
function heroAnimation() {
  const tl = gsap.timeline({
    scrollTrigger: {
      trigger: '.hero',
      start: 'top top',
      end: '+=100%',
      scrub: 1,
      pin: true,
    },
  });

  tl.from('.hero-title', { y: 100, opacity: 0, duration: 1 })
    .from('.hero-subtitle', { y: 50, opacity: 0, duration: 0.8 }, '-=0.5')
    .from('.hero-cta', { scale: 0, duration: 0.5 }, '-=0.3')
    .to('.hero-bg', { scale: 1.2, duration: 2 }, 0);

  return tl;
}

// SplitText — animar caractere por caractere
function textReveal(element: string) {
  const split = new SplitText(element, { type: 'chars,words' });

  gsap.from(split.chars, {
    opacity: 0,
    y: 20,
    rotateX: -90,
    stagger: 0.02,
    duration: 0.8,
    ease: 'back.out(1.7)',
    scrollTrigger: {
      trigger: element,
      start: 'top 80%',
    },
  });
}

// DrawSVG — animar tracos SVG
gsap.from('.svg-path', {
  drawSVG: 0,
  duration: 2,
  ease: 'power2.inOut',
});
```

### Framer Motion (Motion)

Integrado nativamente com React. Ideal para produtos e SaaS.

```tsx
import { motion, AnimatePresence } from 'framer-motion';

// Layout animations (FLIP automatico)
function ProductGrid({ products }: { products: Product[] }) {
  return (
    <motion.div layout className="grid grid-cols-3 gap-4">
      <AnimatePresence>
        {products.map(product => (
          <motion.div
            key={product.id}
            layout
            initial={{ opacity: 0, scale: 0.8 }}
            animate={{ opacity: 1, scale: 1 }}
            exit={{ opacity: 0, scale: 0.8 }}
            transition={{ type: 'spring', stiffness: 300, damping: 25 }}
          >
            <ProductCard product={product} />
          </motion.div>
        ))}
      </AnimatePresence>
    </motion.div>
  );
}

// Gestures
function DraggableCard() {
  return (
    <motion.div
      drag="x"
      dragConstraints={{ left: -100, right: 100 }}
      whileDrag={{ scale: 1.1 }}
      whileHover={{ y: -5 }}
      whileTap={{ scale: 0.95 }}
    >
      Drag me
    </motion.div>
  );
}

// Scroll-triggered animations
function ScrollReveal({ children }: { children: React.ReactNode }) {
  return (
    <motion.div
      initial={{ opacity: 0, y: 50 }}
      whileInView={{ opacity: 1, y: 0 }}
      viewport={{ once: true, margin: '-100px' }}
      transition={{ duration: 0.6, ease: [0.16, 1, 0.3, 1] }}
    >
      {children}
    </motion.div>
  );
}
```

### Lenis (Smooth Scroll)

```typescript
'use client';
import Lenis from 'lenis';
import { useEffect } from 'react';

export function SmoothScrollProvider({ children }: { children: React.ReactNode }) {
  useEffect(() => {
    const lenis = new Lenis({
      duration: 1.2,
      easing: (t) => Math.min(1, 1.001 - Math.pow(2, -10 * t)),
      orientation: 'vertical',
      smoothWheel: true,
    });

    function raf(time: number) {
      lenis.raf(time);
      requestAnimationFrame(raf);
    }
    requestAnimationFrame(raf);

    // Sincronizar com GSAP ScrollTrigger
    lenis.on('scroll', ScrollTrigger.update);
    gsap.ticker.add((time) => lenis.raf(time * 1000));
    gsap.ticker.lagSmoothing(0);

    return () => lenis.destroy();
  }, []);

  return <>{children}</>;
}
```

**RECOMMENDATION para SINAPSE:**
- **Produtos/SaaS:** Framer Motion (integra melhor com React)
- **Websites criativos/portfolios:** GSAP + Lenis + ScrollTrigger
- **Ambos:** Lenis para smooth scroll, combina com qualquer library

---

## 2.3 3D & WebGL

### Three.js + React Three Fiber (R3F)

R3F e a abordagem declarativa para Three.js em React. O ecosistema pmndrs (Poimandres) fornece tudo que voce precisa.

```
@react-three/fiber    — Core (Three.js em React)
@react-three/drei     — Helpers e abstractions (100+ utilitarios)
@react-three/postprocessing — Efeitos visuais (bloom, DOF, etc.)
@react-three/rapier   — Physics engine
leva                  — GUI de debug para tweaking ao vivo
r3f-perf              — Performance monitoring
```

#### Setup Basico

```tsx
import { Canvas } from '@react-three/fiber';
import { OrbitControls, Environment, Float } from '@react-three/drei';

function Scene() {
  return (
    <Canvas
      camera={{ position: [0, 0, 5], fov: 45 }}
      dpr={[1, 2]}                    // Device pixel ratio adaptativo
      performance={{ min: 0.5 }}       // Degradacao graceful de performance
      gl={{ antialias: true, alpha: true }}
    >
      <ambientLight intensity={0.5} />
      <directionalLight position={[10, 10, 5]} intensity={1} castShadow />

      <Float speed={2} rotationIntensity={0.5} floatIntensity={1}>
        <mesh>
          <torusKnotGeometry args={[1, 0.3, 128, 16]} />
          <meshStandardMaterial color="#6366f1" roughness={0.1} metalness={0.8} />
        </mesh>
      </Float>

      <OrbitControls enableDamping dampingFactor={0.05} />
      <Environment preset="city" />
    </Canvas>
  );
}
```

#### Performance Optimization

```tsx
import { useFrame } from '@react-three/fiber';
import { Instances, Instance, Detailed } from '@react-three/drei';

// 1. INSTANCING — milhares de objetos em um unico draw call
function ParticleField({ count = 1000 }) {
  const positions = useMemo(() => {
    return Array.from({ length: count }, () => [
      (Math.random() - 0.5) * 20,
      (Math.random() - 0.5) * 20,
      (Math.random() - 0.5) * 20,
    ] as [number, number, number]);
  }, [count]);

  return (
    <Instances limit={count}>
      <sphereGeometry args={[0.05, 8, 8]} />
      <meshBasicMaterial color="#ffffff" />
      {positions.map((pos, i) => (
        <Instance key={i} position={pos} />
      ))}
    </Instances>
  );
}

// 2. LOD (Level of Detail) — melhora 30-40% em cenas grandes
function AdaptiveModel({ url }: { url: string }) {
  return (
    <Detailed distances={[0, 50, 100]}>
      <HighDetailModel url={url} />    {/* Proximo */}
      <MediumDetailModel url={url} />  {/* Medio */}
      <LowDetailModel url={url} />     {/* Distante */}
    </Detailed>
  );
}

// 3. ON-DEMAND RENDERING — renderiza apenas quando necessario
<Canvas frameloop="demand">
  {/* Apenas re-renderiza quando estado muda */}
</Canvas>
```

**Limites de performance:**
- Maximo ~1000 draw calls (idealmente < 200)
- Texturas: 512x512 ou 1024x1024 para a maioria dos casos (4K e overkill)
- Efeitos de post-processing: maximo 1-2 em dispositivos mid-range
- Usar DRACO compression para modelos 3D (reduz 80-90% do tamanho)

### Shaders (GLSL)

Shaders sao programas que rodam na GPU. **Vertex shader** transforma posicoes, **fragment shader** define cores.

```glsl
// Vertex Shader — displacing vertices com noise
uniform float uTime;
uniform float uAmplitude;

varying vec2 vUv;
varying float vDisplacement;

// Simplex noise (importar da biblioteca)
#pragma glslify: snoise = require(glsl-noise/simplex/3d)

void main() {
  vUv = uv;

  // Calcular displacement baseado em noise
  float noise = snoise(vec3(position.x * 2.0, position.y * 2.0, uTime * 0.5));
  vDisplacement = noise * uAmplitude;

  // Deslocar vertice ao longo da normal
  vec3 newPosition = position + normal * vDisplacement;

  gl_Position = projectionMatrix * modelViewMatrix * vec4(newPosition, 1.0);
}
```

```glsl
// Fragment Shader — cor baseada no displacement
uniform float uTime;
uniform vec3 uColorA;
uniform vec3 uColorB;

varying vec2 vUv;
varying float vDisplacement;

void main() {
  // Misturar cores baseado no displacement
  float mixFactor = (vDisplacement + 1.0) * 0.5;
  vec3 color = mix(uColorA, uColorB, mixFactor);

  // Adicionar efeito de borda
  float edge = smoothstep(0.0, 0.1, vDisplacement);
  color += vec3(0.2, 0.5, 1.0) * (1.0 - edge) * 0.3;

  gl_FragColor = vec4(color, 1.0);
}
```

```tsx
// Uso em React Three Fiber
import { shaderMaterial } from '@react-three/drei';

const WaveMaterial = shaderMaterial(
  { uTime: 0, uAmplitude: 0.3, uColorA: [0.2, 0.1, 0.5], uColorB: [0.9, 0.3, 0.2] },
  vertexShader,
  fragmentShader,
);

function AnimatedSphere() {
  const materialRef = useRef<any>();

  useFrame(({ clock }) => {
    if (materialRef.current) {
      materialRef.current.uTime = clock.elapsedTime;
    }
  });

  return (
    <mesh>
      <sphereGeometry args={[1, 64, 64]} />
      <waveMaterial ref={materialRef} />
    </mesh>
  );
}
```

### Post-Processing

```tsx
import { EffectComposer, Bloom, ChromaticAberration, DepthOfField } from '@react-three/postprocessing';
import { BlendFunction } from 'postprocessing';

function Effects() {
  return (
    <EffectComposer>
      {/* Bloom — brilho em materiais com luminance > 1 */}
      <Bloom
        luminanceThreshold={1}
        luminanceSmoothing={0.9}
        intensity={0.5}
      />

      {/* Chromatic Aberration — dispersao RGB */}
      <ChromaticAberration
        offset={[0.002, 0.002]}
        blendFunction={BlendFunction.NORMAL}
      />

      {/* Depth of Field — foco seletivo */}
      <DepthOfField
        focusDistance={0.01}
        focalLength={0.02}
        bokehScale={3}
      />
    </EffectComposer>
  );
}
```

### WebGPU (Futuro)

Desde Three.js r171 (setembro 2025), WebGPU e production-ready:
```typescript
import * as THREE from 'three/webgpu'; // Fallback automatico para WebGL 2
```

Compute shaders destravam ganhos de 10-100x para particle systems e physics. WebGPU e o sucessor do WebGL, suportado em Chrome, Edge e Safari.

---

## 2.4 Motion Design for Web

### Micro-interactions

Gartner previu que ate o final de 2025, 75% das aplicacoes customer-facing incorporariam micro-interactions como pratica padrao de UI/UX.

```tsx
// Botao com feedback haptico
function ButtonWithFeedback({ children, onClick }: ButtonProps) {
  return (
    <motion.button
      whileHover={{ scale: 1.02, y: -2 }}
      whileTap={{ scale: 0.98 }}
      transition={{ type: 'spring', stiffness: 400, damping: 17 }}
      onClick={onClick}
      className="relative overflow-hidden"
    >
      {children}
      {/* Ripple effect on click */}
      <motion.span
        className="absolute inset-0 bg-white/20 rounded-full"
        initial={{ scale: 0, opacity: 1 }}
        animate={{ scale: 4, opacity: 0 }}
        transition={{ duration: 0.6 }}
      />
    </motion.button>
  );
}

// Loading skeleton com shimmer
function Skeleton({ className }: { className?: string }) {
  return (
    <div className={cn('animate-pulse rounded-md bg-muted', className)}>
      <div className="bg-gradient-to-r from-transparent via-white/20 to-transparent animate-shimmer" />
    </div>
  );
}
```

### Page Transitions

```tsx
// Layout com AnimatePresence para transicoes de rota
'use client';
import { AnimatePresence, motion } from 'framer-motion';
import { usePathname } from 'next/navigation';

export function PageTransition({ children }: { children: React.ReactNode }) {
  const pathname = usePathname();

  return (
    <AnimatePresence mode="wait">
      <motion.div
        key={pathname}
        initial={{ opacity: 0, y: 20 }}
        animate={{ opacity: 1, y: 0 }}
        exit={{ opacity: 0, y: -20 }}
        transition={{ duration: 0.3, ease: [0.16, 1, 0.3, 1] }}
      >
        {children}
      </motion.div>
    </AnimatePresence>
  );
}
```

### Text Animations

```tsx
// Reveal caractere por caractere
function TextReveal({ text, className }: { text: string; className?: string }) {
  return (
    <motion.span className={className}>
      {text.split('').map((char, index) => (
        <motion.span
          key={index}
          initial={{ opacity: 0, y: 20 }}
          whileInView={{ opacity: 1, y: 0 }}
          viewport={{ once: true }}
          transition={{ delay: index * 0.03, duration: 0.4 }}
          style={{ display: 'inline-block' }}
        >
          {char === ' ' ? '\u00A0' : char}
        </motion.span>
      ))}
    </motion.span>
  );
}

// Text scramble effect
function useTextScramble(text: string) {
  const [displayed, setDisplayed] = useState('');
  const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!@#$%';

  useEffect(() => {
    let frame = 0;
    const totalFrames = text.length * 3;

    const interval = setInterval(() => {
      setDisplayed(
        text.split('').map((char, i) => {
          if (frame / 3 > i) return char;
          return chars[Math.floor(Math.random() * chars.length)];
        }).join('')
      );
      frame++;
      if (frame > totalFrames) clearInterval(interval);
    }, 30);

    return () => clearInterval(interval);
  }, [text]);

  return displayed;
}
```

### Parallax Effects

```tsx
// Mouse-driven parallax
function MouseParallax({ children, depth = 20 }: { children: React.ReactNode; depth?: number }) {
  const [position, setPosition] = useState({ x: 0, y: 0 });

  useEffect(() => {
    const handleMouseMove = (e: MouseEvent) => {
      const x = (e.clientX / window.innerWidth - 0.5) * depth;
      const y = (e.clientY / window.innerHeight - 0.5) * depth;
      setPosition({ x, y });
    };
    window.addEventListener('mousemove', handleMouseMove);
    return () => window.removeEventListener('mousemove', handleMouseMove);
  }, [depth]);

  return (
    <motion.div
      animate={{ x: position.x, y: position.y }}
      transition={{ type: 'spring', stiffness: 50, damping: 20 }}
    >
      {children}
    </motion.div>
  );
}

// Scroll-driven parallax com GSAP
function setupParallax() {
  gsap.utils.toArray<HTMLElement>('[data-speed]').forEach(el => {
    const speed = parseFloat(el.dataset.speed || '0');
    gsap.to(el, {
      yPercent: speed * 100,
      ease: 'none',
      scrollTrigger: {
        trigger: el,
        start: 'top bottom',
        end: 'bottom top',
        scrub: true,
      },
    });
  });
}
```

### SVG Animation

```tsx
// Path drawing animation
function DrawSVG({ d, duration = 2 }: { d: string; duration?: number }) {
  const pathRef = useRef<SVGPathElement>(null);

  useEffect(() => {
    const path = pathRef.current;
    if (!path) return;
    const length = path.getTotalLength();
    path.style.strokeDasharray = `${length}`;
    path.style.strokeDashoffset = `${length}`;

    gsap.to(path, {
      strokeDashoffset: 0,
      duration,
      ease: 'power2.inOut',
      scrollTrigger: { trigger: path, start: 'top 80%' },
    });
  }, [d, duration]);

  return (
    <svg viewBox="0 0 100 100">
      <path ref={pathRef} d={d} fill="none" stroke="currentColor" strokeWidth="2" />
    </svg>
  );
}

// SVG Morphing com Framer Motion
function MorphIcon({ isOpen }: { isOpen: boolean }) {
  return (
    <svg viewBox="0 0 24 24" width={24} height={24}>
      <motion.path
        d={isOpen ? 'M6 18L18 6M6 6l12 12' : 'M4 6h16M4 12h16M4 18h16'}
        fill="none"
        stroke="currentColor"
        strokeWidth={2}
        animate={{ d: isOpen ? 'M6 18L18 6M6 6l12 12' : 'M4 6h16M4 12h16M4 18h16' }}
        transition={{ duration: 0.3 }}
      />
    </svg>
  );
}
```

### Reveal Animations (Stagger)

```tsx
// Container que revela filhos com stagger
const containerVariants = {
  hidden: { opacity: 0 },
  visible: {
    opacity: 1,
    transition: {
      staggerChildren: 0.1,
      delayChildren: 0.2,
    },
  },
};

const itemVariants = {
  hidden: { opacity: 0, y: 30 },
  visible: {
    opacity: 1,
    y: 0,
    transition: { duration: 0.5, ease: [0.16, 1, 0.3, 1] },
  },
};

function StaggerList({ items }: { items: Item[] }) {
  return (
    <motion.ul
      variants={containerVariants}
      initial="hidden"
      whileInView="visible"
      viewport={{ once: true, margin: '-50px' }}
    >
      {items.map(item => (
        <motion.li key={item.id} variants={itemVariants}>
          {item.name}
        </motion.li>
      ))}
    </motion.ul>
  );
}
```

---

## 2.5 Creative Development Showcase Sites

### O que Ganha Premios (Awwwards, FWA, CSS Design Awards)

**Criterios Awwwards:**
- Design (40%)
- Usability (30%)
- Creativity (20%)
- Content (10%)

**Tecnologias dominantes em sites premiados:**
- Three.js / R3F para 3D
- GSAP + ScrollTrigger para animacoes complexas
- Lenis para smooth scroll
- Shaders customizados (GLSL) para efeitos visuais unicos
- WebGPU para performance em cenas complexas

**Padroes recorrentes:**
1. **Scroll storytelling** — sticky sections com animacoes que respondem ao scroll
2. **3D environments interativos** — portfolio de Bruno Simon (Site of the Month Jan 2026) com mundo 3D navegavel por veiculo
3. **Gamificacao** — visitantes sao participantes, nao observadores passivos
4. **Tipografia dinamica** — texto que reage a cursor, scroll ou dados
5. **Sound design** — audio ambiente sincronizado com interacoes
6. **Performance impecavel** — sites premiados DEVEM carregar rapido apesar da complexidade

**Equilibrio critico:** Beauty vs Usability. Sites premiados nao sacrificam usabilidade por estetica. O Awwwards peso de 30% em Usability garante isso.

**RECOMMENDATION:** Para portfolios/agencias que buscam premios, a stack e: Next.js + GSAP + Three.js/R3F + Lenis + Shaders customizados. Framer Motion para elementos de UI, GSAP para scroll-driven e timeline complexo.

---

# PARTE 3: FULL-STACK PROJECT TEMPLATES

## Template 1: Landing Page (Estatica, Animada, Conversao)

### Tech Stack
- **Framework:** Next.js (App Router, SSG)
- **Styling:** Tailwind CSS + CSS Variables
- **Animations:** GSAP + ScrollTrigger + Lenis
- **CMS:** Sanity ou Payload CMS (conteudo editavel)
- **Analytics:** PostHog ou Plausible
- **Deploy:** Vercel (Edge)

### Estrutura

```
app/
  page.tsx                # Homepage (Server Component)
  layout.tsx
  globals.css
src/
  components/
    sections/             # Hero, Features, Pricing, CTA, Footer
    ui/                   # Button, Input, Card, Badge
    animations/           # ScrollReveal, TextReveal, Parallax
  lib/
    cms/                  # CMS client, queries
    analytics/            # Event tracking
  hooks/
    useInView.ts
    useMousePosition.ts
  utils/
    cn.ts                 # classNames utility
```

### Padroes-chave
- SSG para performance maxima (LCP < 1s)
- Preload de fontes e imagens hero
- Animations com `prefers-reduced-motion` fallback
- Structured data (JSON-LD) para SEO
- A/B testing com feature flags para CTAs

### Performance Targets
- LCP < 1.5s | INP < 100ms | CLS < 0.05
- Lighthouse Score > 95

### Seguranca
- Rate limiting em forms de contato
- Input validation (Zod) em todos os forms
- CORS restrito
- CSP headers

---

## Template 2: SaaS Application (Auth, Dashboard, Billing, Multi-tenant)

### Tech Stack
- **Framework:** Next.js (App Router)
- **Backend:** tRPC + Supabase
- **Auth:** Supabase Auth + RLS
- **State:** TanStack Query + Zustand
- **Forms:** React Hook Form + Zod
- **UI:** shadcn/ui + Tailwind CSS
- **Billing:** Stripe (subscriptions)
- **Email:** Resend
- **Monitoring:** Sentry + Vercel Analytics
- **Deploy:** Vercel + Supabase Cloud

### Estrutura

```
app/
  (marketing)/            # Landing, pricing, blog (publico)
    page.tsx
    pricing/page.tsx
  (auth)/                 # Login, register, reset password
    login/page.tsx
    register/page.tsx
  (dashboard)/            # App autenticado
    dashboard/page.tsx
    settings/
      profile/page.tsx
      billing/page.tsx
      team/page.tsx
    [tenantSlug]/          # Multi-tenant routes
      page.tsx
  api/
    webhooks/
      stripe/route.ts
    trpc/[trpc]/route.ts
src/
  server/
    routers/              # tRPC routers por dominio
    db/                   # Prisma/Drizzle schema
    services/             # Business logic
  modules/
    auth/                 # Auth hooks, guards, middleware
    billing/              # Stripe integration, plans, usage
    tenants/              # Multi-tenant logic
    notifications/        # Email, push, in-app
  components/
    ui/                   # shadcn/ui components
    features/             # Feature-specific components
    layouts/              # Dashboard layout, sidebar
  lib/
    validations/          # Zod schemas compartilhados
    utils/
```

### Padroes-chave
- Modular Monolith (um modulo por dominio)
- RLS em TODAS as tabelas (tenant isolation)
- Webhook handling robusto (idempotency, retry)
- Subscription lifecycle completo (trial, upgrade, downgrade, cancel)
- Audit logging para compliance

### Performance Targets
- LCP < 2.5s | INP < 200ms | CLS < 0.1
- API response < 200ms (p95)
- Dashboard load < 3s

### Seguranca
- RLS habilitado em todas as tabelas
- service_role NUNCA no frontend
- Rate limiting em auth endpoints (5/15min)
- Input validation em todas as rotas
- CORS restrito a dominio proprio
- MFA para admins
- Audit log de acoes criticas
- Secrets em environment variables (nunca hardcoded)

---

## Template 3: E-commerce / Marketplace

### Tech Stack
- **Framework:** Next.js (App Router, ISR para produtos)
- **Backend:** tRPC + Supabase
- **Search:** Algolia ou Meilisearch
- **Payments:** Stripe Connect (marketplace) ou Stripe Checkout
- **State:** TanStack Query + Zustand (cart)
- **Images:** Cloudinary ou Supabase Storage
- **Email:** Resend
- **Deploy:** Vercel + Supabase Cloud

### Estrutura

```
app/
  (storefront)/
    page.tsx               # Home com featured products (ISR)
    products/
      [slug]/page.tsx      # Product detail (ISR, revalidate: 60)
    categories/[slug]/
    cart/page.tsx
    checkout/page.tsx
  (seller)/                # Seller dashboard (marketplace)
    seller/
      products/page.tsx
      orders/page.tsx
      analytics/page.tsx
  (admin)/
    admin/
      products/page.tsx
      orders/page.tsx
      sellers/page.tsx
src/
  modules/
    catalog/               # Products, categories, search
    cart/                   # Cart state, calculations
    checkout/              # Order creation, payment flow
    orders/                # Order management, tracking
    reviews/               # Product reviews, ratings
    sellers/               # Seller onboarding, payouts (marketplace)
  components/
    product/               # ProductCard, ProductGrid, ProductDetail
    cart/                  # CartDrawer, CartItem, CartSummary
    checkout/              # CheckoutForm, PaymentForm, AddressForm
```

### Padroes-chave
- ISR para paginas de produto (cache 60s, revalidate on-demand)
- Search otimizado com faceted filtering
- Cart persistido em localStorage + synced com server
- Checkout atomico (reservation pattern para stock)
- Webhooks Stripe para payment confirmation
- Image optimization com responsive srcset

### Performance Targets
- Product page LCP < 2s (ISR)
- Search results < 100ms
- Checkout < 3 steps

### Seguranca
- PCI compliance via Stripe (nunca processar dados de cartao)
- RLS para seller isolation
- Rate limiting em checkout
- Fraud detection (Stripe Radar)
- LGPD: consentimento, direitos do titular, politica de privacidade

---

## Template 4: Fintech (Transacoes, KYC, Compliance, Real-time)

### Tech Stack
- **Framework:** Next.js (App Router)
- **Backend:** tRPC + Supabase + Redis
- **Auth:** Supabase Auth + MFA obrigatorio
- **Real-time:** Supabase Realtime + WebSockets
- **Queue:** BullMQ (Redis-based) para jobs async
- **KYC:** Plaid ou Persona
- **Payments:** Stripe Treasury ou banking API
- **Monitoring:** Sentry + Datadog
- **Deploy:** Vercel + Supabase + AWS (compliance)

### Estrutura

```
app/
  (public)/
    page.tsx
  (auth)/
    login/page.tsx          # Com MFA obrigatorio
    kyc/page.tsx            # KYC verification flow
  (app)/
    dashboard/page.tsx      # Portfolio overview, real-time balances
    transactions/page.tsx   # Transaction history
    transfers/page.tsx      # Send/receive money
    settings/
      security/page.tsx     # MFA, session management
src/
  modules/
    auth/                   # Auth + MFA + session management
    kyc/                    # KYC verification, document upload
    accounts/               # Account balances, statements
    transactions/           # Transaction processing, history
    transfers/              # Money movement, scheduling
    compliance/             # AML checks, reporting, audit
    notifications/          # Real-time alerts, push
  server/
    events/                 # Event Sourcing store
    sagas/                  # Long-running transaction sagas
```

### Padroes-chave
- **Event Sourcing** para TODAS as transacoes (audit trail completo)
- **Saga pattern** para transferencias (compensating transactions)
- **CQRS** para separar writes (consistencia) de reads (performance)
- **MFA obrigatorio** para todas as contas
- **Idempotency keys** em todas as operacoes financeiras
- **Real-time** updates via WebSocket para balances e transacoes
- **Encryption at rest** para dados sensiveis (pgcrypto)
- **Rate limiting agressivo** em endpoints financeiros

### Performance Targets
- Transaction processing < 500ms
- Real-time update latency < 100ms
- Dashboard LCP < 2s
- 99.99% uptime

### Seguranca (CRITICA)
- Todos os 25 deployment blockers da Constitution
- MFA obrigatorio (Artigo X)
- Encryption at rest e in transit
- KYC/AML compliance
- Audit log imutavel (event sourcing)
- Session timeout agressivo (15min inatividade)
- IP whitelisting para operacoes criticas
- Penetration testing periodico
- SOC 2 Type II compliance
- LGPD total (DPO, consentimento, direitos, notificacao)

---

## Template 5: Portfolio / Agency (Criativo, Animado, Case Studies)

### Tech Stack
- **Framework:** Next.js (App Router, SSG + ISR)
- **Styling:** Tailwind CSS + CSS Variables + Custom Properties
- **Animations:** GSAP + ScrollTrigger + Lenis + Framer Motion
- **3D:** React Three Fiber + @react-three/drei
- **Shaders:** Custom GLSL
- **CMS:** Sanity (para case studies)
- **Deploy:** Vercel

### Estrutura

```
app/
  page.tsx                  # Hero 3D + scroll-driven narrative
  work/
    page.tsx                # Case studies grid
    [slug]/page.tsx         # Case study detail (ISR)
  about/page.tsx
  contact/page.tsx
src/
  components/
    canvas/                 # Three.js scenes
      HeroScene.tsx
      BackgroundParticles.tsx
      TransitionEffect.tsx
    sections/               # Page sections com animacoes
      Hero.tsx
      WorkGrid.tsx
      About.tsx
    animations/
      TextReveal.tsx
      MagneticButton.tsx
      CursorFollower.tsx
      ParallaxImage.tsx
    ui/
  shaders/
    noise.vert
    noise.frag
    distortion.vert
    distortion.frag
  lib/
    gsap/                   # GSAP setup, ScrollTrigger config
    lenis/                  # Smooth scroll provider
    cms/                    # Sanity client
  hooks/
    useScrollProgress.ts
    useMousePosition.ts
    useSmoothTransform.ts
```

### Padroes-chave
- 3D scene no hero com LOD e on-demand rendering
- GSAP ScrollTrigger para scroll-driven storytelling
- Lenis para smooth scroll em toda a pagina
- Custom cursor que reage a elementos interativos
- Page transitions com AnimatePresence ou View Transitions API
- Case studies com imagens lazy-loaded e progressive loading
- `prefers-reduced-motion` fallback para TODAS as animacoes
- Preload de assets criticos (fontes, texturas 3D)

### Performance Targets
- LCP < 2.5s (com 3D hero)
- INP < 150ms
- Smooth 60fps em scroll animations
- Lighthouse Performance > 85 (creative sites sacrificam um pouco)

### Seguranca
- Rate limiting no form de contato
- Input validation
- CSP headers (especialmente para WebGL)
- Sem dados de usuario sensiveis

---

## Template 6: Blog / Content Platform (CMS, SEO, Social Sharing)

### Tech Stack
- **Framework:** Next.js (App Router, SSG + ISR)
- **CMS:** Sanity ou Payload CMS
- **MDX:** next-mdx-remote (para conteudo tecnico)
- **Search:** Algolia DocSearch ou built-in
- **Comments:** Giscus (GitHub-based) ou custom
- **Analytics:** Plausible ou PostHog
- **Deploy:** Vercel

### Estrutura

```
app/
  page.tsx                  # Homepage com featured posts (SSG)
  blog/
    page.tsx                # Blog listing com paginacao (ISR)
    [slug]/page.tsx         # Post detail (SSG, revalidate on publish)
  categories/[slug]/page.tsx
  tags/[slug]/page.tsx
  feed.xml/route.ts         # RSS feed
  sitemap.xml/route.ts      # Dynamic sitemap
src/
  components/
    blog/
      PostCard.tsx
      PostContent.tsx
      TableOfContents.tsx
      ShareButtons.tsx
      RelatedPosts.tsx
    mdx/                    # Custom MDX components
      CodeBlock.tsx
      Callout.tsx
      Image.tsx
  lib/
    cms/                    # CMS client, queries
    seo/                    # Metadata helpers, JSON-LD
    search/                 # Search indexing
```

### Padroes-chave
- SSG para posts (build time) + ISR on-demand revalidation no publish
- Structured data (JSON-LD) em cada post para rich snippets
- Open Graph + Twitter cards automaticos
- RSS feed e sitemap.xml dinamicos
- Table of Contents gerado do conteudo
- Code syntax highlighting com rehype-pretty-code
- Image optimization automatica
- Reading time estimado

### Performance Targets
- LCP < 1.5s (SSG)
- INP < 100ms
- Lighthouse SEO > 98
- Lighthouse Performance > 95

### SEO Essencial
```tsx
// Metadata dinamico por post
export async function generateMetadata({ params }: PostPageProps): Promise<Metadata> {
  const post = await getPost(params.slug);
  return {
    title: post.title,
    description: post.excerpt,
    openGraph: {
      title: post.title,
      description: post.excerpt,
      type: 'article',
      publishedTime: post.publishedAt,
      authors: [post.author.name],
      images: [{ url: post.coverImage, width: 1200, height: 630 }],
    },
    twitter: { card: 'summary_large_image' },
  };
}

// JSON-LD para rich snippets
const jsonLd = {
  '@context': 'https://schema.org',
  '@type': 'Article',
  headline: post.title,
  datePublished: post.publishedAt,
  author: { '@type': 'Person', name: post.author.name },
};
```

---

## Template 7: Mobile App (React Native, Cross-platform, Offline-first)

### Tech Stack
- **Framework:** Expo (managed workflow) + React Native
- **Navigation:** Expo Router (file-based)
- **Backend:** Supabase
- **State:** TanStack Query + Zustand + MMKV
- **UI:** NativeWind (Tailwind for RN) + React Native Reanimated
- **Animations:** React Native Reanimated + Gesture Handler
- **Offline:** WatermelonDB ou Supabase offline sync
- **Push:** Expo Notifications
- **Deploy:** EAS Build + EAS Submit

### Estrutura

```
app/
  (tabs)/
    index.tsx              # Home tab
    search.tsx             # Search tab
    profile.tsx            # Profile tab
  (auth)/
    login.tsx
    register.tsx
  (modals)/
    [id].tsx               # Detail modal
  _layout.tsx              # Root layout

src/
  components/
    ui/                    # Design system components
    features/              # Feature-specific
  modules/
    auth/
    sync/                  # Offline sync logic
    notifications/
  lib/
    supabase/              # Supabase client
    storage/               # MMKV, AsyncStorage
  hooks/
    useOfflineSync.ts
    usePushNotifications.ts
```

### Padroes-chave
- **Offline-first:** dados sincronizados localmente, sync quando online
- **Optimistic updates:** UI atualiza imediatamente, reconcilia com server depois
- **Skeleton loading:** usar esqueletos em vez de spinners
- **Gesture-driven navigation:** swipe, drag, pinch naturais
- **Deep linking:** cada tela acessivel via URL
- **Background sync:** sincronizar dados em background
- **Push notifications:** setup com Expo Notifications + server-side triggers

### Performance Targets
- App start < 2s (cold start)
- Screen transition < 300ms
- Offline mode funcional para features core
- 60fps em todas as animacoes

### Seguranca
- Biometric authentication (FaceID, fingerprint)
- Certificate pinning
- Encrypted local storage (MMKV)
- Secure token storage (Expo SecureStore)
- No secrets in JS bundle

---

## Tabela Comparativa de Templates

| Template | Stack Principal | Banco | Auth | Animations | Deploy |
|----------|----------------|-------|------|------------|--------|
| Landing Page | Next.js SSG | N/A (CMS) | N/A | GSAP + Lenis | Vercel |
| SaaS | Next.js + tRPC | Supabase | Supabase Auth + RLS | Framer Motion | Vercel + Supabase |
| E-commerce | Next.js + tRPC | Supabase | Supabase Auth | Framer Motion | Vercel + Supabase |
| Fintech | Next.js + tRPC | Supabase + Redis | Supabase Auth + MFA | Framer Motion | Vercel + AWS |
| Portfolio | Next.js SSG | Sanity | N/A | GSAP + R3F + Lenis | Vercel |
| Blog | Next.js SSG/ISR | Sanity/Payload | Opcional | Framer Motion (leve) | Vercel |
| Mobile | Expo + RN | Supabase | Supabase Auth | Reanimated | EAS |

---

## Fontes

### Architecture & Software Engineering
- [Modular Monolith vs Microservices 2025](https://medium.com/@the_atomic_architect/architecture-patterns-that-actually-scale-in-2025-the-only-three-you-need-89d1488c60a7)
- [Microservices vs Monoliths Decision Framework](https://blogs.justenougharchitecture.com/microservices-vs-monoliths-vs-modular-monoliths-a-2025-decision-framework/)
- [ByteByteGo: Monolith vs Microservices vs Modular Monoliths](https://blog.bytebytego.com/p/monolith-vs-microservices-vs-modular)
- [Hexagonal Architecture and Clean Architecture](https://dev.to/dyarleniber/hexagonal-architecture-and-clean-architecture-with-examples-48oi)
- [Clean Architecture and DDD in Practice 2025](https://wojciechowski.app/en/articles/clean-architecture-domain-driven-design-2025)
- [TypeScript for Domain-Driven Design](https://dev.to/shafayeat/typescript-for-domain-driven-design-ddd-6kk)
- [DDD Bounded Contexts and Aggregates](https://www.usefulfunctions.co.uk/2025/11/06/domain-driven-design-bounded-contexts-and-aggregates/)
- [SOLID Principles TypeScript Guide (Strapi)](https://strapi.io/blog/solid-design-principles-javascript-typescript-guide)
- [Design Patterns in TypeScript (Refactoring Guru)](https://refactoring.guru/design-patterns/typescript)
- [Event-Driven Architecture, Event Sourcing, and CQRS](https://dev.to/yasmine_ddec94f4d4/event-driven-architecture-event-sourcing-and-cqrs-how-they-work-together-1bp1)

### Frontend & Next.js
- [Next.js App Router Best Practices 2026](https://ztabs.co/blog/nextjs-app-router-best-practices)
- [Next.js Architecture 2026 — Server-First Patterns](https://www.yogijs.tech/blog/nextjs-project-architecture-app-router)
- [Next.js App Router Patterns That Actually Matter](https://dev.to/teguh_coding/nextjs-app-router-the-patterns-that-actually-matter-in-2026-146)
- [React State Management 2025](https://www.developerway.com/posts/react-state-management-2025)
- [Redux vs TanStack Query & Zustand 2025](https://www.bugragulculer.com/blog/good-bye-redux-how-react-query-and-zustand-re-wired-state-management-in-25)
- [21 React Design Patterns](https://dev.to/perssondennis/21-fantastic-react-design-patterns-and-when-to-use-them-7bb)
- [Headless Component Pattern (Martin Fowler)](https://martinfowler.com/articles/headless-component.html)
- [Frontend Performance Checklist 2025](https://strapi.io/blog/frontend-performance-checklist)
- [WCAG 2.2 Accessibility Guide](https://inhaq.com/blog/accessibility-for-design-engineers-building-inclusive-uis.html)

### Backend & APIs
- [API Design Principles 2026: REST vs gRPC vs GraphQL vs tRPC](https://ruchitsuthar.com/blog/software-craftsmanship/api-design-principles-rest-grpc-graphql/)
- [REST vs GraphQL vs tRPC Guide 2026](https://dev.to/dataformathub/rest-vs-graphql-vs-trpc-the-ultimate-api-design-guide-for-2026-8n3)
- [Supabase RLS Documentation](https://supabase.com/docs/guides/database/postgres/row-level-security)
- [Supabase RBAC with Custom Claims](https://supabase.com/docs/guides/database/postgres/custom-claims-and-role-based-access-control-rbac)
- [Redis Caching Strategies: Next.js Production](https://www.digitalapplied.com/blog/redis-caching-strategies-nextjs-production)
- [TypeScript Error Handling with Result Types](https://typescript.tv/best-practices/error-handling-with-result-types/)

### Testing
- [Test Like a Pro 2025: Vitest, Playwright](https://javascript.plainenglish.io/test-like-a-pro-in-2025-how-i-transformed-my-javascript-projects-with-vitest-playwright-and-more-9616cfb72e9b)
- [Testing Libraries Compared: Vitest vs Jest vs Playwright 2026](https://www.pkgpulse.com/blog/testing-libraries-compared)
- [Mock Service Worker](https://mswjs.io/)
- [React Hook Form + Zod + Server Actions](https://nehalist.io/react-hook-form-with-nextjs-server-actions/)

### Animations & Motion
- [Web Animation Performance Tier List](https://motion.dev/magazine/web-animation-performance-tier-list)
- [View Transitions 2025 Update (Chrome)](https://developer.chrome.com/blog/view-transitions-in-2025)
- [Animations on the Web (Complete Guide)](https://www.benedikt-sperl.de/blog/2026-01-13-animations-on-the-web)
- [GSAP vs Motion Guide 2026](https://satishkumar.xyz/blogs/gsap-vs-motion-guide-2026)
- [Best React Scroll Animation Libraries 2025](https://zoer.ai/posts/zoer/best-react-scroll-animation-libraries-2025)
- [Lenis Smooth Scroll](https://www.lenis.dev/)
- [Motion UI Trends 2025](https://www.betasofttechnology.com/motion-ui-trends-and-micro-interactions/)
- [SVG Animation Encyclopedia 2025](https://www.svgai.org/blog/research/svg-animation-encyclopedia-complete-guide)

### 3D & WebGL
- [Building Efficient Three.js Scenes (Codrops)](https://tympanus.net/codrops/2025/02/11/building-efficient-three-js-scenes-optimize-performance-while-maintaining-quality/)
- [100 Three.js Tips That Improve Performance](https://www.utsubo.com/blog/threejs-best-practices-100-tips)
- [Scaling React-Three-Fiber Applications](https://gitnation.com/contents/scaling-react-three-fiber-applications-beyond-the-hello-world)
- [React Three Fiber Performance Guide](https://r3f.docs.pmnd.rs/advanced/scaling-performance)
- [WebGPU Three.js Migration Guide 2026](https://www.utsubo.com/blog/webgpu-threejs-migration-guide)
- [Animate WebGL Shaders with GSAP (Codrops)](https://tympanus.net/codrops/2025/10/08/how-to-animate-webgl-shaders-with-gsap-ripples-reveals-and-dynamic-blur-effects/)
- [The Book of Shaders: Noise](https://thebookofshaders.com/11/)
- [React Postprocessing (pmndrs)](https://github.com/pmndrs/react-postprocessing)
- [@react-three/drei](https://github.com/pmndrs/drei)

### Creative Development & Awards
- [Awwwards Annual Awards 2025](https://www.awwwards.com/annual-awards/)
- [Award-Winning Web Design Guide](https://www.utsubo.com/blog/award-winning-website-design-guide)
- [How to Become a Creative Developer 2026](https://www.creativedevjobs.com/blog/how-to-become-a-creative-developer)

### SaaS & Multi-tenant
- [Multi-Tenant SaaS Architecture Cloud 2025](https://isitdev.com/multi-tenant-saas-architecture-cloud-2025/)
- [Build Multi-Tenant SaaS (Logto)](https://blog.logto.io/build-multi-tenant-saas-application)
- [Top RBAC Providers for Multi-Tenant SaaS 2025](https://workos.com/blog/top-rbac-providers-for-multi-tenant-saas-2025)

### DevOps
- [CI/CD with Vercel](https://circleci.com/blog/ci-cd-with-vercel/)
- [CI/CD Pipeline with Vercel + GitHub](https://pablodiazt.com/tech-stack/ci-cd-vercel-github)
- [Vercel Preview Deployments](https://vercel.com/products/previews)

---

*Pesquisa conduzida por Prism (research-orqx) | Squad Research | SINAPSE AI*
*Nivel: DEFINITIVE (Pyramid Level 4) | 30+ fontes consultadas | Tiers 1-4*
