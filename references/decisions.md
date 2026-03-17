# Architecture Decisions

Opinionated decisions for AdonisJS v6 API kit projects. Each decision includes the rationale and the rejected alternatives.

---

## Decision 1: Service-Oriented Architecture over Fat Controllers

**Decision:** All business logic lives in service classes. Controllers are thin HTTP adapters.

**Rationale:**
- Controllers that contain business logic cannot be reused from CLI commands, queue workers, or other entry points.
- Testing business logic through HTTP assertions is slow and brittle.
- Service classes with injected dependencies are trivially unit-testable.
- AdonisJS v6 controllers receive `HttpContext` — coupling business logic to HTTP concerns leaks infrastructure into your domain.

**Rejected Alternative:** Fat controllers (logic inline). Works for small apps but collapses under complexity. Also rejected "skinny controller, fat model" — models should own their schema and relationships, not orchestration.

---

## Decision 2: Repository Pattern for Data Access

**Decision:** All database queries go through repository classes. Services never call Lucid's query builder directly.

**Rationale:**
- Centralizes query logic — if you need to add a global scope or change a query, there's one place to look.
- Repositories provide a clean seam for testing. Mock the repository, not the ORM.
- Named methods (`findOverdue()`, `findByTenantWithBalance()`) are more expressive than inline `.where()` chains scattered across services.
- Prepares the codebase for potential ORM changes (unlikely but possible).

**Rejected Alternative:** Direct Lucid calls in services. Faster to write initially but creates query duplication and makes testing harder. Also rejected the "query scope only" approach — scopes are good for composable filters but don't replace the need for a data access boundary.

**Pragmatic Exception:** For trivial CRUD with no business logic (e.g., a settings table), a service can call `Model.findOrFail()` directly. The repository layer is for when queries have meaningful logic or are reused.

---

## Decision 3: Constructor Injection for Services, Method Injection for Controllers

**Decision:** Services use `@inject()` on the class for constructor injection. Controllers use `@inject()` on individual handler methods.

**Rationale:**
- AdonisJS v6 controllers are instantiated per-request. Method injection avoids creating dependencies the specific action doesn't need.
- Services are typically resolved once (or per-request via the container) and use all their dependencies across methods, so constructor injection is cleaner.
- Both patterns are first-class in AdonisJS v6's IoC container.

```typescript
// Controller — method injection
export default class InvoicesController {
  @inject()
  async store({ request, response }: HttpContext, invoiceService: InvoiceService) {
    // only this method needs InvoiceService
  }
}

// Service — constructor injection
@inject()
export default class InvoiceService {
  constructor(
    private invoiceRepo: InvoiceRepository,
    private paymentRepo: PaymentRepository
  ) {}
}
```

---

## Decision 4: Abstract Classes for Contracts, Not Interfaces

**Decision:** Use abstract classes instead of TypeScript interfaces when defining contracts for dependency injection.

**Rationale:**
- TypeScript interfaces are erased at compile time. The IoC container operates at runtime and cannot resolve an interface.
- Abstract classes exist at runtime and can be used as binding keys in the container.
- AdonisJS v6 docs explicitly recommend this approach for hexagonal architecture.

```typescript
// GOOD — abstract class exists at runtime
export abstract class PaymentGateway {
  abstract charge(amount: number, token: string): Promise<ChargeResult>
}

// BAD — interface is erased, container can't use it
export interface PaymentGateway {
  charge(amount: number, token: string): Promise<ChargeResult>
}
```

---

## Decision 5: Events for Side Effects, Not Inline Execution

**Decision:** When a business operation triggers side effects (notifications, audit logs, accounting entries, webhook calls), emit an event. Don't execute the side effect inline.

**Rationale:**
- Prevents service classes from becoming god classes that know about every downstream system.
- Side effects can be added/removed by registering/unregistering listeners — open/closed principle.
- Listeners can be made async or queued independently.
- Testing the core operation doesn't require mocking every side effect.

**When to break this rule:** If the side effect is part of the core operation's contract (e.g., "processing a payment MUST create a journal entry or the payment is invalid"), do it inline in the service within a transaction. If it's a "nice to have" consequence, use events.

---

## Decision 6: VineJS Validators Produce DTOs

**Decision:** Request validation happens at the controller boundary using VineJS. Validators produce typed output that acts as the DTO for the service layer.

**Rationale:**
- Single validation point — data is validated once, at entry.
- VineJS infers TypeScript types from the schema, giving you a typed DTO for free.
- Services receive pre-validated data and can trust it without re-checking.

```typescript
// app/validators/store_payment_validator.ts
import vine from '@vinejs/vine'

export const storePaymentValidator = vine.compile(
  vine.object({
    invoice_id: vine.string().uuid(),
    amount: vine.number().positive(),
    method: vine.enum(['cash', 'mobile_money', 'bank_transfer']),
    reference: vine.string().optional(),
  })
)

// In controller — validated output is the DTO
const data = await request.validateUsing(storePaymentValidator)
// `data` is typed as { invoice_id: string, amount: number, method: 'cash' | ... }
const payment = await paymentService.processPayment(data)
```

**When to use explicit DTO classes:** When you need to transform or enrich validated data before passing to the service (e.g., resolving a user from a token, attaching tenant context). Create a simple class or interface in `app/dtos/`.

---

## Decision 7: Database Transactions Managed at the Service Layer

**Decision:** Transaction boundaries are defined in service methods, not in repositories or controllers.

**Rationale:**
- Services know the business operation's scope — what needs to be atomic.
- Repositories handle individual queries; they don't know whether they're part of a larger operation.
- Controllers shouldn't manage transactions — that's business logic.

```typescript
// In a service
import db from '@adonisjs/lucid/services/db'

async processPayment(data: StorePaymentDto) {
  return db.transaction(async (trx) => {
    const invoice = await this.invoiceRepo.findOrFail(data.invoiceId, trx)
    const payment = await this.paymentRepo.create({ ...data }, trx)
    await this.invoiceRepo.applyPayment(invoice, payment.amount, trx)
    return payment
  })
}
```

**Pattern:** Pass the transaction object (`trx`) to repository methods so they can use it. Repositories should accept an optional transaction parameter.

---

## Decision 8: Domain Exceptions over Generic Errors

**Decision:** Throw domain-specific exception classes for business rule violations. Let the AdonisJS exception handler convert them to HTTP responses.

**Rationale:**
- `throw new InsufficientBalanceException(wallet)` is more expressive than `throw new Error('Insufficient balance')`.
- Custom exceptions can carry structured data (the wallet, the requested amount, etc.).
- The global exception handler maps exception types to HTTP status codes, keeping controllers clean.

```typescript
// app/exceptions/insufficient_balance_exception.ts
import { Exception } from '@adonisjs/core/exceptions'

export default class InsufficientBalanceException extends Exception {
  static status = 422
  static code = 'E_INSUFFICIENT_BALANCE'

  constructor(
    public walletId: string,
    public available: number,
    public requested: number
  ) {
    super(`Wallet ${walletId} has ${available} but ${requested} was requested`)
  }
}
```

---

## Decision 9: No Business Logic in Models

**Decision:** Lucid models define schema (columns, relationships, hooks for data integrity) and query scopes. Business logic lives in services and domain objects.

**Rationale:**
- Models that contain business methods become impossible to test without a database.
- "Where does this logic live?" is always answered: services for orchestration, domain objects for calculations.
- Query scopes on models are acceptable because they're composable query fragments, not business logic.

**What belongs on a model:**
- Column definitions and casts
- Relationships (`hasMany`, `belongsTo`, etc.)
- Computed properties for data formatting (`get fullName()`)
- Query scopes for reusable filters (`scopeActive`, `scopeOverdue`)
- Lifecycle hooks ONLY for data integrity (e.g., generating a UUID before create)

**What does NOT belong on a model:**
- Payment processing logic
- Notification triggers
- State transition logic
- Anything that coordinates with other models

---

## Decision 10: Strategy Pattern for Dual Business Models

**Decision:** When the same operation has different implementations per business model (e.g., installment billing vs. subscription billing), use the strategy pattern.

**Rationale:**
- Avoids `if (model === 'installment') { ... } else { ... }` scattered everywhere.
- New business models can be added by creating a new strategy class.
- The correct strategy can be resolved via the IoC container based on the contract type.

```typescript
// app/contracts/billing_strategy.ts
export abstract class BillingStrategy {
  abstract generateInvoice(contract: Contract): Promise<Invoice>
  abstract calculatePenalty(invoice: Invoice): Promise<number>
  abstract handlePayment(payment: Payment): Promise<void>
}

// app/services/billing/installment_billing_strategy.ts
export class InstallmentBillingStrategy extends BillingStrategy {
  async generateInvoice(contract: Contract) { /* ... */ }
  async calculatePenalty(invoice: Invoice) { /* ... */ }
  async handlePayment(payment: Payment) { /* ... */ }
}

// app/services/billing/subscription_billing_strategy.ts
export class SubscriptionBillingStrategy extends BillingStrategy {
  // ...
}
```

Resolve contextually:
```typescript
// In a service or provider
const strategy = contract.type === 'installment'
  ? await app.container.make(InstallmentBillingStrategy)
  : await app.container.make(SubscriptionBillingStrategy)
```

---

## Decision 11: Subpath Imports for Everything

**Decision:** Always use AdonisJS v6 subpath imports (`#services/`, `#models/`, etc.) instead of relative paths.

**Rationale:**
- Eliminates `../../../` import chains.
- Refactoring file locations doesn't break imports.
- Consistent, scannable import blocks.

Configure in `package.json`:
```json
{
  "imports": {
    "#controllers/*": "./app/controllers/*.js",
    "#services/*": "./app/services/*.js",
    "#repositories/*": "./app/repositories/*.js",
    "#models/*": "./app/models/*.js",
    "#domain/*": "./app/domain/*.js",
    "#actions/*": "./app/actions/*.js",
    "#dtos/*": "./app/dtos/*.js",
    "#events/*": "./app/events/*.js",
    "#listeners/*": "./app/listeners/*.js",
    "#validators/*": "./app/validators/*.js",
    "#exceptions/*": "./app/exceptions/*.js",
    "#state_machines/*": "./app/state_machines/*.js",
    "#contracts/*": "./app/contracts/*.js",
    "#policies/*": "./app/policies/*.js"
  }
}
```

---

## Decision 12: Explicit Error Handling over Silent Failures

**Decision:** Operations should throw on failure. Never return `null` to indicate an error condition.

**Rationale:**
- `null` returns force callers to null-check everything, leading to defensive coding.
- Exceptions carry context about what went wrong.
- AdonisJS's exception handler provides a centralized place to format error responses.

```typescript
// GOOD
async findActiveContract(tenantId: string): Promise<Contract> {
  const contract = await Contract.query()
    .where('tenant_id', tenantId)
    .where('status', 'active')
    .first()

  if (!contract) {
    throw new ContractNotFoundException(tenantId)
  }

  return contract
}

// BAD
async findActiveContract(tenantId: string): Promise<Contract | null> {
  return Contract.query()
    .where('tenant_id', tenantId)
    .where('status', 'active')
    .first()
}
```

**Exception:** Repositories may return `null` when the caller genuinely needs to handle "not found" as a non-exceptional case (e.g., `findExistingWallet()` where not having a wallet triggers wallet creation).

---

## Decision 13: Register Stateless Services as Singletons

**Decision:** Stateless services and repositories MUST be registered as singletons in a service provider. Do not rely on the container's default auto-resolution for frequently-used classes.

**Rationale:**
- AdonisJS v6 controllers are instantiated per-request. When you use `@inject()`, the container resolves the entire dependency tree on every request.
- If `PaymentService` depends on 4 repositories, and each repository is auto-resolved (the default), the container creates and discards 5 objects per request — multiplied across every endpoint.
- Stateless services and repositories (no request-specific state held between method calls) are safe as singletons. There's no reason to recreate them.
- Without singleton registration, a deep dependency graph causes unnecessary GC pressure under load.

```typescript
// app/providers/app_provider.ts
import type { ApplicationService } from '@adonisjs/core/types'
import { InvoiceRepository } from '#repositories/invoice_repository'
import { ContractRepository } from '#repositories/contract_repository'
import { WalletRepository } from '#repositories/wallet_repository'
import InvoiceService from '#services/invoice_service'
import PaymentService from '#services/payment_service'

export default class AppProvider {
  constructor(protected app: ApplicationService) {}

  register() {
    // Repositories — stateless, safe as singletons
    this.app.container.singleton(InvoiceRepository, () => new InvoiceRepository())
    this.app.container.singleton(ContractRepository, () => new ContractRepository())
    this.app.container.singleton(WalletRepository, () => new WalletRepository())

    // Services — stateless, depend on singleton repos
    this.app.container.singleton(InvoiceService, async (resolver) => {
      const invoiceRepo = await resolver.make(InvoiceRepository)
      const contractRepo = await resolver.make(ContractRepository)
      return new InvoiceService(invoiceRepo, contractRepo)
    })

    this.app.container.singleton(PaymentService, async (resolver) => {
      const invoiceRepo = await resolver.make(InvoiceRepository)
      const walletRepo = await resolver.make(WalletRepository)
      return new PaymentService(invoiceRepo, walletRepo)
    })
  }
}
```

**What qualifies as stateless:**
- The class holds no mutable instance properties that change between calls
- Constructor dependencies are themselves stateless or singletons
- Methods receive all request-specific data via parameters, not instance state

**What must NOT be a singleton:**
- Anything that holds a reference to `HttpContext`, the current request, or the current user
- Classes that accumulate state across calls (caches that grow unbounded, connection holders)

---

## Decision 14: Never Do External I/O Inside Database Transactions

**Decision:** Database transactions must contain only database operations. External I/O (API calls, email, file uploads, queue dispatches) happens after the transaction commits, typically via events.

**Rationale:**
- Each `db.transaction(async (trx) => { ... })` holds a database connection from the pool for the entire callback duration.
- AdonisJS's default connection pool is small (typically 10 connections). If 10 concurrent requests each hold a connection while waiting on a slow external API, the pool is exhausted and all subsequent requests queue up.
- External calls can fail or timeout. If the transaction rolls back after a successful external side effect (e.g., money was charged but DB update failed), you have an inconsistency that's very hard to recover from.

```typescript
// GOOD — DB only inside transaction, side effects after
async processPayment(data: StorePaymentDto) {
  const payment = await db.transaction(async (trx) => {
    const invoice = await this.invoiceRepo.findOrFail(data.invoiceId, trx)
    const payment = await this.paymentRepo.create(data, trx)
    await this.invoiceRepo.applyPayment(invoice, payment.amount, trx)
    return payment
  })

  // External I/O happens AFTER commit, via event
  await emitter.emit(new PaymentReceived(payment))
  return payment
}

// BAD — external API call holds the DB connection hostage
async processPayment(data: StorePaymentDto) {
  return db.transaction(async (trx) => {
    const invoice = await this.invoiceRepo.findOrFail(data.invoiceId, trx)
    const result = await this.paymentGateway.charge(data.amount) // BLOCKS CONNECTION
    const payment = await this.paymentRepo.create({ ...data, chargeId: result.id }, trx)
    await this.invoiceRepo.applyPayment(invoice, payment.amount, trx)
    await this.notificationService.sendReceipt(payment) // BLOCKS CONNECTION AGAIN
    return payment
  })
}
```

**Pattern for external-then-DB workflows:** If you must call an external API and then record the result, do the external call first, then open a short transaction for the DB writes:

```typescript
async processPayment(data: StorePaymentDto) {
  // Step 1: External call (no transaction held)
  const chargeResult = await this.paymentGateway.charge(data.amount, data.token)

  // Step 2: Short transaction for DB writes only
  const payment = await db.transaction(async (trx) => {
    const invoice = await this.invoiceRepo.findOrFail(data.invoiceId, trx)
    const payment = await this.paymentRepo.create(
      { ...data, chargeId: chargeResult.id },
      trx
    )
    await this.invoiceRepo.applyPayment(invoice, payment.amount, trx)
    return payment
  })

  // Step 3: Side effects after commit
  await emitter.emit(new PaymentReceived(payment))
  return payment
}
```

---

## Decision 15: Event Listeners Must Be Non-Blocking for I/O-Heavy Work

**Decision:** AdonisJS's emitter runs listeners sequentially and awaits them. Listeners that perform heavy I/O (email, webhooks, PDF generation, external API calls) must dispatch to a background job queue instead of doing the work inline.

**Rationale:**
- When you `await emitter.emit(new PaymentReceived(payment))`, all registered listeners are executed in sequence. If you have 4 listeners and the third one calls an email API that takes 2 seconds, your entire endpoint is blocked for those 2 seconds.
- Accumulated listener latency directly impacts response time. 4 listeners × 500ms each = 2 seconds added to every request.
- A failed listener (e.g., email service is down) can throw an exception that bubbles up to the controller, failing the entire request even though the core operation succeeded.

```typescript
// BAD — blocks the response while sending email
export default class SendInvoiceNotification {
  async handle(event: InvoiceCreated) {
    await mail.send((message) => {
      message
        .to(event.invoice.contract.tenant.email)
        .subject(`Invoice #${event.invoice.number}`)
        .htmlView('emails/invoice_created', { invoice: event.invoice })
    })
  }
}

// GOOD — dispatches to queue, returns immediately
export default class SendInvoiceNotification {
  async handle(event: InvoiceCreated) {
    await queue.dispatch(SendInvoiceEmailJob, {
      invoiceId: event.invoice.id,
    })
  }
}
```

**Guidelines for listener categorization:**

- **Inline (fast, critical):** Updating a balance, recording an audit log to DB, transitioning state. These are fast DB writes that must happen as part of the operation's contract.
- **Queued (slow, non-critical):** Sending emails, calling webhooks, generating PDFs, syncing to external systems. Failure shouldn't break the core operation.

---

## Decision 16: Never Cache Model Instances in Singletons

**Decision:** Singleton services must never store references to Lucid model instances as instance properties. Models should only exist within the scope of a single method call.

**Rationale:**
- Lucid model instances hold references to their query client, loaded relationships, dirty attributes, and the underlying connection. A single model instance can keep a surprisingly large object graph alive in memory.
- If a singleton service stores a model reference (e.g., `this.lastProcessedInvoice = invoice`), that reference persists for the lifetime of the process. The garbage collector can never reclaim it.
- Over time, accumulated model references in singletons constitute a classic Node.js memory leak — memory usage grows monotonically until the process crashes or is restarted.
- Loaded relationships compound the problem: an invoice that preloaded its contract, which preloaded its tenant, which preloaded its address — one reference holds the entire chain in memory.

```typescript
// BAD — leaks memory, one model per call held forever
@inject()
export default class ContractService {
  private lastActivated?: Contract  // MEMORY LEAK

  async activate(contractId: string) {
    const contract = await this.contractRepo.findOrFail(contractId)
    contract.status = 'active'
    await contract.save()
    this.lastActivated = contract  // This reference never dies
    return contract
  }
}

// BAD — cache grows without bound
@inject()
export default class InvoiceService {
  private cache = new Map<string, Invoice>()  // MEMORY LEAK

  async findOrFail(id: string) {
    if (this.cache.has(id)) return this.cache.get(id)!
    const invoice = await this.invoiceRepo.findOrFail(id)
    this.cache.set(id, invoice)  // Grows forever, stale data
    return invoice
  }
}

// GOOD — models are local variables, GC'd after each call
@inject()
export default class ContractService {
  async activate(contractId: string) {
    const contract = await this.contractRepo.findOrFail(contractId)
    contract.status = 'active'
    await contract.save()
    return contract  // Caller holds the reference, not the service
  }
}
```

**If you need caching**, use a dedicated cache layer (Redis, AdonisJS cache) with serialized data (plain objects or JSON), not live model instances. Cache the data, not the ORM object.

---

## Decision 17: Repositories Must Not Eagerly Load Relationships by Default

**Decision:** Repository methods return lean model instances by default. Relationship preloading must be explicitly requested by the caller.

**Rationale:**
- A `findOrFail()` that always preloads 3 relationships fetches far more data than most callers need. If 8 out of 10 callers only need the invoice itself, you're running 3 unnecessary JOINs or subqueries on 80% of calls.
- Unnecessary preloading increases memory pressure — each loaded relationship instantiates additional model instances.
- Over-fetching masks N+1 query problems instead of solving them. The correct fix for N+1 is explicit preloading at the call site that needs it, not blanket preloading everywhere.

```typescript
// GOOD — lean by default, preload on demand
export class InvoiceRepository {
  async findOrFail(id: string, trx?: TransactionClientContract) {
    const query = Invoice.query()
    if (trx) query.useTransaction(trx)
    return query.where('id', id).firstOrFail()
  }

  async findWithContract(id: string) {
    return Invoice.query()
      .where('id', id)
      .preload('contract')
      .firstOrFail()
  }

  async findWithFullDetails(id: string) {
    return Invoice.query()
      .where('id', id)
      .preload('contract', (q) => q.preload('tenant'))
      .preload('payments')
      .preload('lineItems')
      .firstOrFail()
  }
}

// BAD — every caller pays for all preloads
export class InvoiceRepository {
  async findOrFail(id: string) {
    return Invoice.query()
      .where('id', id)
      .preload('contract', (q) => q.preload('tenant'))
      .preload('payments')
      .preload('lineItems')
      .firstOrFail()  // 4 queries when caller only needed the invoice
  }
}
```

**Alternative pattern — preload callback:**

```typescript
async findOrFail(
  id: string,
  preload?: (query: ModelQueryBuilderContract<typeof Invoice>) => void,
  trx?: TransactionClientContract
) {
  const query = Invoice.query().where('id', id)
  if (trx) query.useTransaction(trx)
  if (preload) preload(query)
  return query.firstOrFail()
}

// Usage
const invoice = await this.invoiceRepo.findOrFail(id, (q) => {
  q.preload('contract').preload('payments')
})
```

---

## Decision 18: Static Event Listener Registration Only

**Decision:** All event listeners MUST be registered statically in `start/events.ts` at application boot. Never register listeners dynamically inside request handlers, loops, or service methods.

**Rationale:**
- Dynamic listener registration is the most common source of memory leaks in Node.js event-driven applications.
- If a listener is registered inside a request handler, every incoming request adds a new listener. After 10,000 requests, you have 10,000 duplicate listeners — each holding a closure reference that prevents garbage collection.
- Node.js will emit a warning at 11 listeners (`MaxListenersExceededWarning`), but by then the damage is already accumulating.

```typescript
// GOOD — registered once at boot in start/events.ts
import emitter from '@adonisjs/core/services/emitter'
const PaymentReceived = () => import('#events/payment_received')
const GenerateJournalEntry = () => import('#listeners/generate_journal_entry')

emitter.on(PaymentReceived, [GenerateJournalEntry])

// BAD — registered on every request, classic memory leak
export default class PaymentsController {
  async store({ request, response }: HttpContext) {
    // This adds a NEW listener every request!
    emitter.on(PaymentReceived, [GenerateJournalEntry])  // MEMORY LEAK

    const payment = await paymentService.processPayment(data)
    return response.created(payment)
  }
}

// BAD — registered inside a service constructor (if service is not singleton, leaks)
export default class PaymentService {
  constructor() {
    // If this service is auto-resolved per request, this leaks
    emitter.on(SomeEvent, [SomeListener])  // MEMORY LEAK
  }
}
```

---

## Decision 19: Multi-Tenancy via Middleware + Repository Scoping

**Decision:** Resolve tenant context in middleware, store it on `HttpContext`, and scope every repository query using an explicit `tenantId` parameter. Never rely on implicit global state for tenant isolation.

**Rationale:** Multi-tenant data isolation is a security boundary — a missed `WHERE tenant_id = ?` clause leaks data across tenants. Making `tenantId` an explicit parameter to repository methods makes accidental omission a compile-time error (TypeScript will warn), not a runtime data leak.

**Pattern:**

```typescript
// app/middleware/tenant_middleware.ts
import type { HttpContext } from '@adonisjs/core/http'

export default class TenantMiddleware {
  async handle({ auth, tenantId }: HttpContext, next: () => Promise<void>) {
    // Resolve tenant from authenticated user (or subdomain, header, etc.)
    const user = await auth.authenticate()
    // Attach tenantId to context for downstream use
    ;(ctx as any).tenantId = user.tenantId
    await next()
  }
}
```

Extend `HttpContext` to carry `tenantId`:

```typescript
// contracts/http.ts  (or start/context.ts)
import { HttpContext } from '@adonisjs/core/http'

declare module '@adonisjs/core/http' {
  interface HttpContext {
    tenantId: string
  }
}
```

Controllers extract `tenantId` from context and pass it to services:

```typescript
// GOOD — tenantId flows explicitly through layers
@inject()
async index({ tenantId, request, response }: HttpContext, invoiceService: InvoiceService) {
  const page = request.input('page', 1)
  const limit = request.input('limit', 20)
  const invoices = await invoiceService.list(tenantId, page, limit)
  return response.ok(invoices)
}
```

Repositories accept `tenantId` as an explicit parameter — never read it from a singleton or global:

```typescript
// GOOD — explicit scoping, no accidental cross-tenant leak
export class InvoiceRepository {
  async list(tenantId: string, page: number, limit: number) {
    return Invoice.query()
      .where('tenant_id', tenantId)
      .orderBy('created_at', 'desc')
      .paginate(page, limit)
  }

  async findOrFail(tenantId: string, id: string) {
    return Invoice.query()
      .where('tenant_id', tenantId)
      .where('id', id)
      .firstOrFail()
  }
}

// BAD — implicit tenant from singleton, breaks test isolation and concurrent requests
export class InvoiceRepository {
  private tenantId: string  // NEVER store tenant on instance

  setTenant(id: string) { this.tenantId = id }  // race condition in singleton

  async list() {
    return Invoice.query().where('tenant_id', this.tenantId)  // WRONG
  }
}
```

**Rejected alternatives:**
- *Async local storage / CLS*: Implicit and error-prone — context gets lost across `await` boundaries without careful setup. Explicit parameters are safer and more testable.
- *Storing tenant on the repository singleton*: Race condition — multiple concurrent requests share the singleton and overwrite each other's tenant.

**Trade-offs:** Every repository method signature carries `tenantId`. This is intentional — it's a forcing function that makes tenant scoping impossible to forget.

**Diagnostic:** If you see `MaxListenersExceededWarning` in your logs, you almost certainly have a dynamic listener registration somewhere. Search your codebase for `emitter.on` outside of `start/events.ts`.

---

## Decision 20: Hybrid Tracing (OTEL) + Prometheus Metrics (prom-client)

**Decision:** Use `@adonisjs/otel` exclusively for distributed tracing (exported via OTLP). Use `prom-client` directly for all metrics, exposed at `GET /metrics` for Prometheus scraping. Do not use the OTEL metrics SDK.

**Rationale:**
- The OTEL metrics SDK's Prometheus exporter requires managing its own HTTP listener, which conflicts with AdonisJS's server lifecycle and complicates startup ordering.
- `prom-client` is the de-facto standard for Prometheus in Node.js. It has a simpler API, a single `Registry` object, and zero lifecycle friction — just expose `registry.metrics()` on a route.
- Tracing and metrics serve different purposes: traces answer "what happened in this request?", metrics answer "how is the system behaving over time?". Using the best-in-class tool for each is more pragmatic than forcing both through OTEL.
- Sampling (reducing trace volume for high-traffic endpoints) does not affect metrics — counters and histograms capture everything regardless of whether the request was sampled for tracing.

**Rejected Alternative:** Using OTEL metrics SDK with `@opentelemetry/exporter-metrics-otlp-proto`. Works end-to-end if your backend speaks OTLP (Grafana Cloud, Datadog, etc.), but adds complexity when Prometheus scraping is already in place. Revisit if the team standardizes on a pure OTLP pipeline.

**Rejected Alternative:** Logging metrics to stdout and parsing with a log shipper. Brittle, high cardinality, and adds latency to the metrics pipeline.

**Implementation summary:**
1. `otel.ts` (project root) — imports `@adonisjs/otel/init` with `metricReaders: []` (metrics disabled in OTEL config).
2. `app/services/otel_service.ts` — static class owning the Prometheus `Registry`, all `Counter`/`Gauge`/`Histogram` definitions, the OTEL `tracer`, and the `withSpan()` helper.
3. `app/middleware/http_metric_middleware.ts` — server middleware recording per-request Prometheus metrics.
4. `app/middleware/otel_enrich.ts` — router middleware enriching the active OTEL span with user/request attributes.
5. `start/otel_metrics.ts` — boot-time Lucid `db:query` event listeners for database telemetry.

See `references/observability.md` for full implementation details.
