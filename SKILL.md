---
name: adonisjs-architecture
description: >
  Opinionated architecture decisions, patterns, and project structure for AdonisJS v6 API kit applications.
  Use when the user asks about architecture decisions, project structure, pattern selection, separation of concerns,
  service layers, repository patterns, domain-driven design, or mentions how to organize, which pattern to use,
  best practices, or architecture. Also trigger when the user is creating controllers, services, repositories,
  actions, DTOs, validators, events, listeners, state machines, or any domain logic classes in an AdonisJS v6 project.
  Even if the user doesn't explicitly say "architecture," use this skill whenever structural or organizational
  decisions are being made in an AdonisJS v6 codebase.
---

# AdonisJS v6 API Architecture

Opinionated, service-oriented architecture for AdonisJS v6 API kit applications. TypeScript-first, separation of concerns everywhere, testable by default.

## Core Philosophy

1. **Controllers are HTTP adapters** — they translate HTTP into service calls and back. Zero business logic.
2. **Services orchestrate** — they coordinate repositories, emit events, enforce business rules.
3. **Repositories own data access** — all Lucid queries live here. Services never call `.query()` directly.
4. **Domain objects are pure** — calculations, value objects, and state machines have no DB or HTTP awareness.
5. **Events decouple side effects** — downstream consequences (notifications, audit logs, accounting entries) happen via listeners, not inline.
6. **DTOs cross boundaries** — data moves between layers via typed objects, never raw `request.all()`.

## When to Read Reference Files

**Instruction:** When this skill activates, immediately read the relevant reference file(s) for the task at hand using the Read tool *before* generating any output.

- **Architecture decisions & rationale** → Read `references/decisions.md` (Decisions 1–12 on structure, 13–18 on performance & memory safety)
- **Code examples for each layer** → Read `references/code-examples.md`
- **Testing patterns** → Read `references/testing.md`
- **State machine patterns** → Read `references/state-machines.md`
- **Performance & memory safety** → Read `references/performance.md`

## Project Structure

```
app/
├── controllers/           # Thin HTTP handlers (1 resource = 1 controller)
│   └── payments_controller.ts
├── services/              # Business orchestration
│   └── payment_service.ts
├── repositories/          # Data access via Lucid
│   └── invoice_repository.ts
├── domain/                # Pure business logic
│   ├── billing/           # Calculations, strategies
│   ├── accounting/        # Journal entry builders
│   └── value_objects/     # Money, DateRange, etc.
├── actions/               # Single-purpose commands (optional, for complex ops)
│   └── process_payment_action.ts
├── dtos/                  # Data Transfer Objects
│   └── payment_dto.ts
├── state_machines/        # Lifecycle definitions
│   └── invoice_state_machine.ts
├── events/                # Event classes
│   └── payment_received.ts
├── listeners/             # Event handlers
│   └── generate_journal_entry.ts
├── validators/            # VineJS request validation
│   └── store_payment_validator.ts
├── models/                # Lucid models — schema + relationships + scopes ONLY
│   └── invoice.ts
├── contracts/             # Abstract classes for swappable dependencies
│   └── payment_gateway.ts
├── exceptions/            # Domain-specific errors
│   └── insufficient_balance_exception.ts
├── policies/              # Bouncer authorization
│   └── invoice_policy.ts
├── middleware/             # HTTP middleware
│   └── tenant_middleware.ts
└── providers/             # Service providers for container bindings
    └── repository_provider.ts

start/
├── events.ts              # Event → Listener registrations (emitter.on calls)
├── routes.ts              # Route definitions
└── kernel.ts              # Global middleware registration

database/
├── migrations/            # Lucid migration files
├── factories/             # Model factories for testing/seeding
└── seeders/               # Database seeders
```

### File Naming

AdonisJS v6 uses **snake_case** for all files and directories. Follow this convention strictly:
- `payment_service.ts` not `PaymentService.ts`
- `store_payment_validator.ts` not `StorePaymentValidator.ts`
- Directories: `value_objects/` not `ValueObjects/`

### Barrel Exports

Do NOT use barrel files (`index.ts`) for re-exporting. AdonisJS v6 uses direct imports with the `#` subpath import alias. Every import should point to the exact file:

```typescript
// GOOD
import PaymentService from '#services/payment_service'
import { InvoiceRepository } from '#repositories/invoice_repository'

// BAD — no barrel files
import { PaymentService } from '#services'
```

## Layer Rules

### Controllers

```typescript
// app/controllers/payments_controller.ts
import type { HttpContext } from '@adonisjs/core/http'
import { inject } from '@adonisjs/core'
import PaymentService from '#services/payment_service'
import { storePaymentValidator } from '#validators/store_payment_validator'

export default class PaymentsController {
  @inject()
  async store({ request, response }: HttpContext, paymentService: PaymentService) {
    const data = await request.validateUsing(storePaymentValidator)
    const payment = await paymentService.processPayment(data)
    return response.created(payment)
  }
}
```

**Rules:**
- Validate input via VineJS validators
- Call exactly ONE service method
- Return response — no business logic, no direct DB queries
- Use `@inject()` for method injection of services
- One controller per resource, standard REST actions only

### Services

```typescript
// app/services/payment_service.ts
import { inject } from '@adonisjs/core'
import emitter from '@adonisjs/core/services/emitter'
import { InvoiceRepository } from '#repositories/invoice_repository'
import { WalletRepository } from '#repositories/wallet_repository'
import PaymentReceived from '#events/payment_received'

@inject()
export default class PaymentService {
  constructor(
    private invoiceRepo: InvoiceRepository,
    private walletRepo: WalletRepository
  ) {}

  async processPayment(data: StorePaymentDto) {
    // Orchestrate: validate business rules, coordinate repos, emit events
    const invoice = await this.invoiceRepo.findOrFail(data.invoiceId)
    // ... business rule checks ...
    const payment = await this.invoiceRepo.recordPayment(invoice, data.amount)
    await emitter.emit(new PaymentReceived(payment))
    return payment
  }
}
```

**Rules:**
- Use constructor injection via `@inject()` decorator
- Coordinate multiple repositories — this is the orchestration layer
- Emit events for side effects — don't handle them inline
- Throw domain exceptions for business rule violations
- Services may call other services when orchestrating cross-domain workflows
- Keep methods focused: one business operation per method

### Repositories

```typescript
// app/repositories/invoice_repository.ts
import Invoice from '#models/invoice'

export class InvoiceRepository {
  async findOrFail(id: string) {
    return Invoice.findOrFail(id)
  }

  async findOverdue(tenantId: string) {
    return Invoice.query()
      .where('tenant_id', tenantId)
      .where('status', 'overdue')
      .orderBy('due_date', 'asc')
  }

  async recordPayment(invoice: Invoice, amount: number) {
    // All query logic stays here
  }
}
```

**Rules:**
- All Lucid query builder calls live here
- Return models or serialized data — never raw query results
- Name methods after what they retrieve, not how: `findOverdue()` not `queryByStatusAndDate()`
- No business logic — just data access and persistence
- **Pagination**: use Lucid's `.paginate(page, limit)` in the repository. Return the `ModelPaginatorContract` directly to the service/controller. Never pass raw `page`/`limit` into a service — let the repository own pagination.

```typescript
async list(page: number, limit: number) {
  return Invoice.query().orderBy('created_at', 'desc').paginate(page, limit)
}
```

### Domain Objects

Pure TypeScript classes/functions with no framework dependencies:

```typescript
// app/domain/billing/installment_calculator.ts
export class InstallmentCalculator {
  static calculate(principal: number, rate: number, months: number): InstallmentSchedule {
    // Pure math — no DB, no HTTP, no framework imports
  }
}
```

### DTOs

```typescript
// app/dtos/payment_dto.ts
export interface StorePaymentDto {
  invoiceId: string
  amount: number
  method: PaymentMethod
  reference?: string
}
```

**Rules:**
- Plain interfaces or classes — no decorators, no Lucid
- Used to move data between layers
- Validators produce DTOs, services consume them

### Response Serialization

Return Lucid models directly from services when the full model shape is acceptable. Use `serializeAs` on model columns to rename or hide fields in JSON output:

```typescript
// app/models/invoice.ts
@column({ serializeAs: null })         // hidden from JSON response
declare internalNote: string | null

@column({ serializeAs: 'customer_id' }) // renamed in JSON response
declare customerId: string
```

When the response shape differs significantly from the model (e.g., computed fields, nested aggregates), return a plain DTO object from the service instead of the model. Never add serialization logic to controllers.

### Events & Listeners

```typescript
// app/events/payment_received.ts
import Payment from '#models/payment'

export default class PaymentReceived {
  constructor(public payment: Payment) {}
}

// app/listeners/generate_journal_entry.ts
import PaymentReceived from '#events/payment_received'
import { JournalEntryBuilder } from '#domain/accounting/journal_entry_builder'

export default class GenerateJournalEntry {
  async handle(event: PaymentReceived) {
    const builder = new JournalEntryBuilder()
    await builder.forPayment(event.payment).save()
  }
}
```

Register in `start/events.ts`:
```typescript
import emitter from '@adonisjs/core/services/emitter'
const PaymentReceived = () => import('#events/payment_received')
const GenerateJournalEntry = () => import('#listeners/generate_journal_entry')

emitter.on(PaymentReceived, [GenerateJournalEntry])
```

### Authorization (Bouncer)

Authorization lives at the **controller layer**, called before the service. Policies are plain classes with no DI required.

```typescript
// app/policies/invoice_policy.ts
import User from '#models/user'
import Invoice from '#models/invoice'
import { BasePolicy } from '@adonisjs/bouncer'
import { AuthorizerResponse } from '@adonisjs/bouncer/types'

export default class InvoicePolicy extends BasePolicy {
  view(user: User, invoice: Invoice): AuthorizerResponse {
    return user.id === invoice.userId
  }

  update(user: User, invoice: Invoice): AuthorizerResponse {
    return user.id === invoice.userId && invoice.status === 'DRAFT'
  }
}
```

Call `bouncer.authorize()` in the controller, **before** calling the service:

```typescript
// app/controllers/invoices_controller.ts
@inject()
async update({ params, request, response, bouncer }: HttpContext, invoiceService: InvoiceService) {
  const invoice = await invoiceService.findOrFail(params.id)
  await bouncer.with(InvoicePolicy).authorize('update', invoice)  // throws 403 if denied
  const data = await request.validateUsing(updateInvoiceValidator)
  const updated = await invoiceService.update(params.id, data)
  return response.ok(updated)
}
```

**Rules:**
- Authorization checks happen in controllers, not services — services assume the caller is authorized.
- Policies extend `BasePolicy` and use the `AuthorizerResponse` return type.
- Never pass `bouncer` into a service; keep authorization at the HTTP boundary.

## Dependency Injection Strategy

Use AdonisJS v6's built-in IoC container with the `@inject()` decorator:

- **Controllers**: Use method injection — `@inject()` on the handler method
- **Services**: Use constructor injection — `@inject()` on the class. **Register as singletons** in a provider (they're stateless).
- **Repositories**: **Register as singletons** in a provider — they're stateless and reconstructing them per-request wastes allocations.

For abstractions (e.g., swappable payment gateways), use abstract classes + contextual bindings:

```typescript
// app/contracts/payment_gateway.ts
export abstract class PaymentGateway {
  abstract charge(amount: number, token: string): Promise<ChargeResult>
  abstract refund(chargeId: string): Promise<RefundResult>
}
```

Bind in a provider:
```typescript
// app/providers/payment_provider.ts
import type { ApplicationService } from '@adonisjs/core/types'
import { PaymentGateway } from '#contracts/payment_gateway'
import { StripeGateway } from '#services/gateways/stripe_gateway'

export default class PaymentProvider {
  constructor(protected app: ApplicationService) {}

  register() {
    this.app.container.bind(PaymentGateway, () => new StripeGateway())
  }
}
```

## Key Conventions

1. **One class per file.** No exceptions.
2. **snake_case file names.** Enforced by AdonisJS v6 convention.
3. **Subpath imports.** Always use `#` aliases: `#services/`, `#models/`, `#repositories/`.
4. **Explicit return types** on all public service/repository methods.
5. **No `any` types.** Use `unknown` if type is truly unknown, then narrow.
6. **Database transactions** are managed at the service layer, not in repositories.
7. **Validation** happens once, at the controller boundary, via VineJS.
8. **Error handling**: throw typed domain exceptions; let AdonisJS exception handler format the HTTP response.
9. **Enums use ALL_CAPS for both keys and values.** The enum name is PascalCase; keys and string values are SCREAMING_SNAKE_CASE. This makes enum values unmistakable in code and consistent with database column values.

```typescript
// GOOD
export enum InvoiceStatus {
  DRAFT = 'DRAFT',
  SENT = 'SENT',
  PAID = 'PAID',
  OVERDUE = 'OVERDUE',
}

// BAD — mixed casing leaks into DB values and string comparisons
export enum InvoiceStatus {
  Draft = 'draft',
  Sent = 'sent',
}
```

Always use `Object.values(MyEnum)` in VineJS validators to keep enum values as the single source of truth:

```typescript
method: vine.enum(Object.values(PaymentMethod)),
```

## Performance & Memory Safety Rules

10. **Register stateless services/repositories as singletons.** Auto-resolution creates and GC's identical objects per request. See Decision 13.
11. **Never do external I/O inside `db.transaction()`.** Transactions hold DB connections. Slow external calls exhaust the pool. See Decision 14.
12. **Queue heavy listener work.** `emitter.emit()` awaits all listeners sequentially. Email, webhooks, PDF generation must go to a job queue. See Decision 15.
13. **Never cache model instances in singletons.** Models hold connection/relationship references. One forgotten ref = unbounded memory growth. See Decision 16.
14. **Repositories must not eagerly preload.** Return lean models by default. Offer explicit `findWithRelations()` variants. See Decision 17.
15. **Event listeners registered once at boot only.** Never inside request handlers or constructors. Dynamic registration = memory leak. See Decision 18.

## When to Use Actions vs Services

- **Service**: Orchestrates a domain workflow. May coordinate multiple repos, call external APIs, emit events. Has multiple related methods. E.g., `PaymentService` with `processPayment()`, `refundPayment()`, `allocateOverpayment()`.
- **Action**: Single-purpose, invokable class for complex operations that deserve their own file. Use when a service method grows beyond ~50 lines or involves intricate logic. E.g., `ProcessInstallmentPaymentAction`.

Actions are optional — start with services. Extract to actions when complexity demands it.
