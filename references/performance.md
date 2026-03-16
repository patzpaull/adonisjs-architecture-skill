# Performance & Memory Safety

Quick-reference for avoiding performance pitfalls and memory leaks in AdonisJS v6 API applications using the service-oriented architecture.

The detailed rationale for each rule lives in `references/decisions.md` (Decisions 13–18). This file is the operational checklist.

---

## Pre-Flight Checklist

Run through this before merging any feature branch:

- [ ] All stateless services and repositories registered as singletons in a provider
- [ ] No `emitter.on()` calls outside of `start/events.ts`
- [ ] No external I/O (HTTP calls, email, file uploads) inside `db.transaction()` blocks
- [ ] No model instances stored as class properties on singleton services
- [ ] No unbounded `Map` or `Set` caches on singleton services
- [ ] Repository `findOrFail()` methods do NOT eagerly preload relationships
- [ ] Heavy listener work (email, webhooks, PDFs) dispatched to a job queue, not inline

---

## Singleton Registration Pattern

```typescript
// app/providers/app_provider.ts
import type { ApplicationService } from '@adonisjs/core/types'

export default class AppProvider {
  constructor(protected app: ApplicationService) {}

  register() {
    // Repositories
    this.app.container.singleton(
      (await import('#repositories/invoice_repository')).InvoiceRepository,
      () => new (await import('#repositories/invoice_repository')).InvoiceRepository()
    )

    // Services that depend on repos
    this.app.container.singleton(
      (await import('#services/invoice_service')).default,
      async (resolver) => {
        const { InvoiceRepository } = await import('#repositories/invoice_repository')
        const { ContractRepository } = await import('#repositories/contract_repository')
        const invoiceRepo = await resolver.make(InvoiceRepository)
        const contractRepo = await resolver.make(ContractRepository)
        const { default: InvoiceService } = await import('#services/invoice_service')
        return new InvoiceService(invoiceRepo, contractRepo)
      }
    )
  }
}
```

**Simpler alternative** — if your services use `@inject()` and all their dependencies are also singletons, the container can auto-resolve the tree. Just register the leaf dependencies as singletons:

```typescript
register() {
  this.app.container.singleton(InvoiceRepository, () => new InvoiceRepository())
  this.app.container.singleton(ContractRepository, () => new ContractRepository())
  // InvoiceService will auto-resolve with @inject(), getting singleton repos
}
```

> **Note:** This only works if `InvoiceService` is also marked as a singleton. Otherwise the service is still recreated per-request, even though its dependencies are shared.

---

## Transaction Safety Patterns

### Pattern 1: DB-only transaction, events after

```typescript
async processPayment(data: StorePaymentDto) {
  // Transaction holds a connection — keep it fast
  const payment = await db.transaction(async (trx) => {
    const invoice = await this.invoiceRepo.findOrFail(data.invoiceId, trx)
    const payment = await this.paymentRepo.create(data, trx)
    await this.invoiceRepo.applyPayment(invoice, payment.amount, trx)
    return payment
  })
  // Connection released — safe to do slow work
  await emitter.emit(new PaymentReceived(payment))
  return payment
}
```

### Pattern 2: External call first, then short transaction

```typescript
async processPayment(data: StorePaymentDto) {
  // External call — no connection held
  const charge = await this.gateway.charge(data.amount, data.token)

  // Short DB transaction
  const payment = await db.transaction(async (trx) => {
    return this.paymentRepo.create({ ...data, chargeId: charge.id }, trx)
  })

  await emitter.emit(new PaymentReceived(payment))
  return payment
}
```

### Pattern 3: Saga for multi-step external + DB

When you need to coordinate multiple external calls with DB writes, use a saga pattern with compensation:

```typescript
async processPayment(data: StorePaymentDto) {
  // Step 1: Charge externally
  const charge = await this.gateway.charge(data.amount, data.token)

  try {
    // Step 2: Record in DB
    const payment = await db.transaction(async (trx) => {
      return this.paymentRepo.create({ ...data, chargeId: charge.id }, trx)
    })
    return payment
  } catch (error) {
    // Compensate: refund the charge if DB write failed
    await this.gateway.refund(charge.id)
    throw error
  }
}
```

---

## Event Listener Performance

### Categorize every listener

| Category | Execution | Examples |
|----------|-----------|---------|
| **Critical & fast** | Inline (awaited) | Update balance, record audit log, transition state |
| **Critical & slow** | Queued with retry | Generate journal entry, sync to accounting system |
| **Non-critical** | Queued, fire-and-forget | Send email, call webhook, generate PDF |

### Measuring listener impact

Add timing to identify slow listeners:

```typescript
// Temporary diagnostic — remove after profiling
import emitter from '@adonisjs/core/services/emitter'

emitter.on('*', (event) => {
  const start = performance.now()
  return () => {
    const duration = performance.now() - start
    if (duration > 100) {
      console.warn(`Slow listener: ${event.constructor.name} took ${duration.toFixed(0)}ms`)
    }
  }
})
```

---

## Memory Leak Detection

### Symptoms

- Process RSS memory grows monotonically (never levels off)
- `MaxListenersExceededWarning` in logs
- Response times degrade over hours without increased load
- OOM kills in production after extended uptime

### Common Causes in This Architecture

| Cause | Where to look | Fix |
|-------|--------------|-----|
| Dynamic listener registration | `emitter.on()` outside `start/events.ts` | Move to static registration |
| Model refs in singletons | Service class properties | Keep models as local variables |
| Unbounded caches | `Map`/`Set` on singleton services | Use Redis or LRU with max size |
| Unresolved promises | `Promise` chains in listeners that never settle | Add timeouts, catch rejections |
| Unreleased DB connections | Long transactions, missing `trx` cleanup | Keep transactions short, add timeouts |

### Diagnostic: Check for Dynamic Listeners

```bash
# Search for emitter.on outside of start/events.ts
grep -rn "emitter\.on\|emitter\.once" app/ --include="*.ts"
# Should return zero results — all registration belongs in start/events.ts
```

### Diagnostic: Monitor Connection Pool

```typescript
// In a health check endpoint or diagnostic route
import db from '@adonisjs/lucid/services/db'

router.get('/health/db', async ({ response }) => {
  const pool = db.connection().pool
  return response.ok({
    used: pool.numUsed(),
    free: pool.numFree(),
    pending: pool.numPendingAcquires(),
    total: pool.numUsed() + pool.numFree(),
  })
})
```

If `pending` is consistently > 0, transactions are holding connections too long.

---

## Performance Benchmarks to Watch

When load testing your API, track these metrics:

- **P99 response time**: Should not degrade over a 1-hour sustained load test. If it does, suspect memory leak or connection pool exhaustion.
- **DB connection pool utilization**: `numUsed / total` should stay below 80% under normal load. If it hits 100%, transactions are too long or too frequent.
- **Heap used**: Should stabilize after warmup. If it grows linearly, you have a memory leak.
- **Event loop lag**: Should stay under 50ms. If it spikes, you have CPU-bound work blocking the event loop (move it to a worker thread or queue).
- **GC pause time**: Occasional short pauses are normal. Frequent long pauses (>100ms) suggest too many long-lived objects (likely model caching in singletons).
