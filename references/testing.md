# Testing Patterns

Testing patterns for an AdonisJS v6 API application using Japa test runner.

---

## Test Organization

```
tests/
├── bootstrap.ts              # Test setup (migrations, etc.)
├── unit/                     # Pure logic, no HTTP, no DB
│   ├── domain/
│   │   └── installment_calculator.spec.ts
│   └── value_objects/
│       └── money.spec.ts
├── functional/               # HTTP tests against running app
│   ├── invoices/
│   │   ├── list.spec.ts
│   │   ├── create.spec.ts
│   │   └── update.spec.ts
│   └── payments/
│       └── process.spec.ts
└── integration/              # Service tests with real DB
    ├── services/
    │   └── invoice_service.spec.ts
    └── repositories/
        └── invoice_repository.spec.ts
```

**Rule:** Unit tests have zero I/O. Integration tests hit the database. Functional tests make HTTP requests.

---

## Unit Tests (Domain Objects)

```typescript
// tests/unit/domain/installment_calculator.spec.ts
import { test } from '@japa/runner'
import { InstallmentCalculator } from '#domain/billing/installment_calculator'
import { DateTime } from 'luxon'

test.group('InstallmentCalculator', () => {
  test('calculates correct number of installments', ({ assert }) => {
    const result = InstallmentCalculator.calculate(
      12000,
      0,
      12,
      DateTime.fromISO('2025-01-01')
    )
    assert.equal(result.installments.length, 12)
  })

  test('zero interest produces equal installments', ({ assert }) => {
    const result = InstallmentCalculator.calculate(
      12000,
      0,
      12,
      DateTime.fromISO('2025-01-01')
    )
    for (const installment of result.installments) {
      assert.equal(installment.total, 1000)
      assert.equal(installment.interest, 0)
    }
  })

  test('total amount with interest exceeds principal', ({ assert }) => {
    const result = InstallmentCalculator.calculate(
      12000,
      12,
      12,
      DateTime.fromISO('2025-01-01')
    )
    assert.isAbove(result.totalAmount, 12000)
    assert.isAbove(result.totalInterest, 0)
  })

  test('final installment has zero remaining balance', ({ assert }) => {
    const result = InstallmentCalculator.calculate(
      12000,
      12,
      12,
      DateTime.fromISO('2025-01-01')
    )
    const last = result.installments[result.installments.length - 1]
    assert.equal(last.remainingBalance, 0)
  })
})
```

---

## Unit Tests (Value Objects)

```typescript
// tests/unit/value_objects/money.spec.ts
import { test } from '@japa/runner'
import { Money } from '#domain/value_objects/money'

test.group('Money', () => {
  test('creates from amount', ({ assert }) => {
    const m = Money.fromAmount(100.50)
    assert.equal(m.amount, 100.5)
  })

  test('adds two money values', ({ assert }) => {
    const a = Money.fromAmount(10)
    const b = Money.fromAmount(20)
    assert.equal(a.add(b).amount, 30)
  })

  test('throws on different currencies', ({ assert }) => {
    const tzs = Money.fromAmount(100, 'TZS')
    const usd = Money.fromAmount(100, 'USD')
    assert.throws(() => tzs.add(usd), /different currencies/)
  })

  test('handles precision correctly', ({ assert }) => {
    const a = Money.fromAmount(0.1)
    const b = Money.fromAmount(0.2)
    assert.equal(a.add(b).amount, 0.3)
  })
})
```

---

## Functional Tests (HTTP Endpoints)

```typescript
// tests/functional/invoices/create.spec.ts
import { test } from '@japa/runner'
import testUtils from '@adonisjs/core/services/test_utils'
import { UserFactory } from '#database/factories/user_factory'
import { ContractFactory } from '#database/factories/contract_factory'

test.group('POST /api/v1/invoices', (group) => {
  group.each.setup(() => testUtils.db().withGlobalTransaction())

  test('creates an invoice with valid data', async ({ client, assert }) => {
    const user = await UserFactory.create()
    const contract = await ContractFactory.merge({ tenantId: user.id }).create()

    const response = await client
      .post('/api/v1/invoices')
      .loginAs(user)
      .json({
        contractId: contract.id,
        amount: 50000,
        dueDate: '2025-06-15',
      })

    response.assertStatus(201)
    response.assertBodyContains({
      contractId: contract.id,
      amount: 50000,
      status: 'draft',
    })
  })

  test('returns 422 for invalid amount', async ({ client }) => {
    const user = await UserFactory.create()
    const contract = await ContractFactory.merge({ tenantId: user.id }).create()

    const response = await client
      .post('/api/v1/invoices')
      .loginAs(user)
      .json({
        contractId: contract.id,
        amount: -100,
        dueDate: '2025-06-15',
      })

    response.assertStatus(422)
  })

  test('returns 422 for missing contract', async ({ client }) => {
    const user = await UserFactory.create()

    const response = await client
      .post('/api/v1/invoices')
      .loginAs(user)
      .json({
        contractId: '00000000-0000-0000-0000-000000000000',
        amount: 50000,
        dueDate: '2025-06-15',
      })

    response.assertStatus(422)
  })

  test('returns 401 without authentication', async ({ client }) => {
    const response = await client.post('/api/v1/invoices').json({
      contractId: 'some-id',
      amount: 50000,
      dueDate: '2025-06-15',
    })

    response.assertStatus(401)
  })
})
```

---

## Integration Tests (Services)

```typescript
// tests/integration/services/invoice_service.spec.ts
import { test } from '@japa/runner'
import testUtils from '@adonisjs/core/services/test_utils'
import app from '@adonisjs/core/services/app'
import InvoiceService from '#services/invoice_service'
import { ContractFactory } from '#database/factories/contract_factory'
import InvalidInvoiceStateException from '#exceptions/invalid_invoice_state_exception'

test.group('InvoiceService', (group) => {
  group.each.setup(() => testUtils.db().withGlobalTransaction())

  test('creates invoice in draft status', async ({ assert }) => {
    const contract = await ContractFactory.create()
    const service = await app.container.make(InvoiceService)

    const invoice = await service.create({
      contractId: contract.id,
      amount: 50000,
      dueDate: DateTime.fromISO('2025-06-15'),
    })

    assert.equal(invoice.status, 'draft')
    assert.equal(invoice.contractId, contract.id)
  })

  test('prevents update of non-draft invoice', async ({ assert }) => {
    const contract = await ContractFactory.create()
    const service = await app.container.make(InvoiceService)

    const invoice = await service.create({
      contractId: contract.id,
      amount: 50000,
      dueDate: DateTime.fromISO('2025-06-15'),
    })

    // Manually move to pending to test the guard
    invoice.status = 'pending'
    await invoice.save()

    await assert.rejects(
      () => service.update(invoice.id, { amount: 60000 }),
      InvalidInvoiceStateException
    )
  })
})
```

---

## Integration Tests (Repositories)

```typescript
// tests/integration/repositories/invoice_repository.spec.ts
import { test } from '@japa/runner'
import testUtils from '@adonisjs/core/services/test_utils'
import { InvoiceRepository } from '#repositories/invoice_repository'
import { InvoiceFactory } from '#database/factories/invoice_factory'
import { DateTime } from 'luxon'

test.group('InvoiceRepository', (group) => {
  group.each.setup(() => testUtils.db().withGlobalTransaction())

  const repo = new InvoiceRepository()

  test('findOverdueByTenant returns only overdue invoices', async ({ assert }) => {
    const tenantId = 'tenant-1'

    await InvoiceFactory.merge({ tenantId, status: 'overdue' }).createMany(3)
    await InvoiceFactory.merge({ tenantId, status: 'pending' }).createMany(2)
    await InvoiceFactory.merge({ tenantId: 'other', status: 'overdue' }).create()

    const results = await repo.findOverdueByTenant(tenantId)
    assert.equal(results.length, 3)
    results.forEach((inv) => assert.equal(inv.status, 'overdue'))
  })

  test('sumUnpaidByContract sums pending and overdue', async ({ assert }) => {
    const contractId = 'contract-1'

    await InvoiceFactory.merge({ contractId, status: 'pending', amount: 1000 }).create()
    await InvoiceFactory.merge({ contractId, status: 'overdue', amount: 2000 }).create()
    await InvoiceFactory.merge({ contractId, status: 'paid', amount: 5000 }).create()

    const total = await repo.sumUnpaidByContract(contractId)
    assert.equal(total, 3000)
  })
})
```

---

## Testing with Dependency Swaps

```typescript
// tests/functional/payments/process.spec.ts
import { test } from '@japa/runner'
import testUtils from '@adonisjs/core/services/test_utils'
import app from '@adonisjs/core/services/app'
import { PaymentGateway } from '#contracts/payment_gateway'

test.group('Payment Processing', (group) => {
  group.each.setup(() => testUtils.db().withGlobalTransaction())

  test('uses fake payment gateway in tests', async ({ assert }) => {
    // Swap the real gateway for a fake
    app.container.swap(PaymentGateway, () => {
      return {
        async charge(amount: number, token: string) {
          return { success: true, chargeId: 'fake-charge-123' }
        },
        async refund(chargeId: string) {
          return { success: true }
        },
      }
    })

    // ... run your test ...

    // Restore after test
    app.container.restore(PaymentGateway)
  })
})
```

---

## Test Bootstrap Setup

```typescript
// tests/bootstrap.ts
import { assert } from '@japa/assert'
import { apiClient } from '@japa/api-client'
import app from '@adonisjs/core/services/app'
import testUtils from '@adonisjs/core/services/test_utils'

export const plugins = [assert(), apiClient()]

export const runnerHooks = {
  setup: [() => testUtils.db().migrate()],
  teardown: [() => testUtils.db().truncate()],
}

export const configureSuite = {
  unit: (suite) => {},
  functional: (suite) => {
    suite.setup(() => testUtils.httpServer().start())
  },
  integration: (suite) => {},
}
```

---

## Factory Definitions

Factories live in `database/factories/` and follow the naming convention `{model_name}_factory.ts`.

```typescript
// database/factories/invoice_factory.ts
import Invoice from '#models/invoice'
import { InvoiceStatus } from '#models/invoice'
import factory from '@adonisjs/lucid/factories'

export const InvoiceFactory = factory
  .define(Invoice, ({ faker }) => {
    return {
      customerId: faker.string.uuid(),
      amount: faker.number.int({ min: 100, max: 100000 }),
      status: InvoiceStatus.DRAFT,
      dueDate: faker.date.future(),
    }
  })
  .build()
```

```typescript
// database/factories/user_factory.ts
import User from '#models/user'
import factory from '@adonisjs/lucid/factories'

export const UserFactory = factory
  .define(User, ({ faker }) => {
    return {
      email: faker.internet.email(),
      password: 'secret',
      tenantId: faker.string.uuid(),
    }
  })
  .build()
```

Import path uses the `#database` subpath alias:

```typescript
// package.json imports
"#database/*": "./database/*.js"
```

```typescript
// Usage in tests
import { InvoiceFactory } from '#database/factories/invoice_factory'

const draft = await InvoiceFactory.create()
const paid  = await InvoiceFactory.merge({ status: InvoiceStatus.PAID }).create()
const bulk  = await InvoiceFactory.createMany(5)
```

---

## Key Testing Principles

1. **Unit tests** for domain objects and value objects — fast, no I/O.
2. **Integration tests** for services and repositories — real DB via transactions (rolled back).
3. **Functional tests** for HTTP endpoints — full stack through the router.
4. **Use factories** for test data — never hand-construct models in tests.
5. **Use `withGlobalTransaction()`** — each test runs in a transaction that's rolled back.
6. **Swap dependencies** via `app.container.swap()` for external services (payment gateways, email, etc.).
7. **Test behavior, not implementation** — assert on outcomes, not on which methods were called.
