# State Machine Patterns

Patterns for implementing state machines in AdonisJS v6 for managing entity lifecycles (invoices, customers, etc.).

---

## Approach: Lightweight Custom State Machines

AdonisJS v6 doesn't have a built-in state machine package like Laravel's Spatie Model States. Instead, use a lightweight custom implementation that:

1. Defines valid transitions in a declarative map
2. Enforces transitions at the service/domain layer
3. Emits events on state changes
4. Keeps the state machine logic separate from the model

---

## State Machine Class

```typescript
// app/state_machines/state_machine.ts
import StateMachineException from '#exceptions/state_machine_exception'

export interface StateTransition<S extends string> {
  from: S
  to: S
}

export abstract class StateMachine<S extends string> {
  protected abstract transitions: StateTransition<S>[]

  canTransition(from: S, to: S): boolean {
    return this.transitions.some((t) => t.from === from && t.to === to)
  }

  getAllowedTransitions(from: S): S[] {
    return this.transitions.filter((t) => t.from === from).map((t) => t.to)
  }

  assertCanTransition(from: S, to: S, entityId?: string): void {
    if (!this.canTransition(from, to)) {
      const allowed = this.getAllowedTransitions(from)
      throw new StateMachineException(from, to, allowed, entityId)
    }
  }
}
```

---

## Invoice State Machine

```typescript
// app/state_machines/invoice_state_machine.ts
import { StateMachine } from '#state_machines/state_machine'
import { InvoiceStatus } from '#models/invoice'

export class InvoiceStateMachine extends StateMachine<InvoiceStatus> {
  protected transitions = [
    // Draft can be sent or cancelled
    { from: InvoiceStatus.DRAFT, to: InvoiceStatus.SENT },
    { from: InvoiceStatus.DRAFT, to: InvoiceStatus.CANCELLED },

    // Sent can be paid, go overdue, or be cancelled
    { from: InvoiceStatus.SENT, to: InvoiceStatus.PAID },
    { from: InvoiceStatus.SENT, to: InvoiceStatus.OVERDUE },
    { from: InvoiceStatus.SENT, to: InvoiceStatus.CANCELLED },

    // Overdue can be paid or cancelled
    { from: InvoiceStatus.OVERDUE, to: InvoiceStatus.PAID },
    { from: InvoiceStatus.OVERDUE, to: InvoiceStatus.CANCELLED },
  ]
}

// Export a singleton instance
export const invoiceStateMachine = new InvoiceStateMachine()
```

---

## Customer State Machine

```typescript
// app/state_machines/customer_state_machine.ts
import { StateMachine } from '#state_machines/state_machine'
import { CustomerStatus } from '#models/customer'

export class CustomerStateMachine extends StateMachine<CustomerStatus> {
  protected transitions = [
    { from: CustomerStatus.ACTIVE, to: CustomerStatus.INACTIVE },
    { from: CustomerStatus.ACTIVE, to: CustomerStatus.SUSPENDED },
    { from: CustomerStatus.INACTIVE, to: CustomerStatus.ACTIVE },
    { from: CustomerStatus.SUSPENDED, to: CustomerStatus.ACTIVE },
    { from: CustomerStatus.SUSPENDED, to: CustomerStatus.INACTIVE },
  ]
}

export const customerStateMachine = new CustomerStateMachine()
```

---

## State Machine Exception

```typescript
// app/exceptions/state_machine_exception.ts
import { Exception } from '@adonisjs/core/exceptions'

export default class StateMachineException extends Exception {
  static status = 422
  static code = 'E_INVALID_STATE_TRANSITION'

  constructor(from: string, to: string, allowed: string[], entityId?: string) {
    const entity = entityId ? ` for entity ${entityId}` : ''
    const allowedStr = allowed.length > 0 ? allowed.join(', ') : 'none'
    super(
      `Cannot transition from "${from}" to "${to}"${entity}. Allowed transitions from "${from}": [${allowedStr}]`
    )
  }
}
```

---

## Using State Machines in Services

```typescript
// app/services/invoice_service.ts
import { inject } from '@adonisjs/core'
import db from '@adonisjs/lucid/services/db'
import emitter from '@adonisjs/core/services/emitter'
import { InvoiceRepository } from '#repositories/invoice_repository'
import { invoiceStateMachine } from '#state_machines/invoice_state_machine'
import { InvoiceStatus } from '#models/invoice'
import InvoiceStatusChanged from '#events/invoice_status_changed'

@inject()
export default class InvoiceService {
  constructor(private invoiceRepo: InvoiceRepository) {}

  async transitionStatus(invoiceId: string, newStatus: InvoiceStatus) {
    return db.transaction(async (trx) => {
      const invoice = await this.invoiceRepo.findOrFail(invoiceId, trx)
      const oldStatus = invoice.status

      invoiceStateMachine.assertCanTransition(oldStatus, newStatus, invoiceId)

      const updated = await this.invoiceRepo.updateStatus(invoice, newStatus, trx)

      await emitter.emit(new InvoiceStatusChanged(updated, oldStatus, newStatus))

      return updated
    })
  }

  async send(invoiceId: string) {
    return this.transitionStatus(invoiceId, InvoiceStatus.SENT)
  }

  async markPaid(invoiceId: string) {
    return this.transitionStatus(invoiceId, InvoiceStatus.PAID)
  }

  async markOverdue(invoiceId: string) {
    return this.transitionStatus(invoiceId, InvoiceStatus.OVERDUE)
  }

  async cancel(invoiceId: string) {
    return this.transitionStatus(invoiceId, InvoiceStatus.CANCELLED)
  }
}
```

---

## Status Change Event

```typescript
// app/events/invoice_status_changed.ts
import type Invoice from '#models/invoice'
import { InvoiceStatus } from '#models/invoice'

export default class InvoiceStatusChanged {
  constructor(
    public invoice: Invoice,
    public from: InvoiceStatus,
    public to: InvoiceStatus
  ) {}
}
```

Register listeners that respond to specific transitions:

```typescript
// app/listeners/handle_invoice_overdue.ts
import type InvoiceStatusChanged from '#events/invoice_status_changed'
import { InvoiceStatus } from '#models/invoice'

export default class HandleInvoiceOverdue {
  async handle(event: InvoiceStatusChanged) {
    if (event.to !== InvoiceStatus.OVERDUE) return

    // Apply late fee, send notification, etc.
  }
}
```

---

## State Machines with Guards (Advanced)

For transitions that require preconditions:

```typescript
// app/state_machines/guarded_state_machine.ts
import StateMachineException from '#exceptions/state_machine_exception'

export interface GuardedTransition<S extends string> {
  from: S
  to: S
  guard?: () => Promise<boolean> | boolean
}

export abstract class GuardedStateMachine<S extends string> {
  protected abstract transitions: GuardedTransition<S>[]

  async canTransition(from: S, to: S): Promise<boolean> {
    const transition = this.transitions.find((t) => t.from === from && t.to === to)
    if (!transition) return false
    if (!transition.guard) return true
    return transition.guard()
  }

  async assertCanTransition(from: S, to: S, entityId?: string): Promise<void> {
    const can = await this.canTransition(from, to)
    if (!can) {
      const allowed = this.transitions.filter((t) => t.from === from).map((t) => t.to)
      throw new StateMachineException(from, to, allowed, entityId)
    }
  }
}
```

Example with a guard on invoice cancellation:

```typescript
// app/state_machines/invoice_state_machine.ts (guarded variant)
import { GuardedStateMachine } from '#state_machines/guarded_state_machine'
import { InvoiceRepository } from '#repositories/invoice_repository'
import { InvoiceStatus } from '#models/invoice'

export class InvoiceStateMachine extends GuardedStateMachine<InvoiceStatus> {
  constructor(private invoiceRepo: InvoiceRepository) {
    super()
  }

  protected transitions = [
    { from: InvoiceStatus.DRAFT, to: InvoiceStatus.SENT },
    {
      from: InvoiceStatus.SENT,
      to: InvoiceStatus.CANCELLED,
      guard: async () => {
        // Only allow cancellation if no payments have been recorded
        return true // actual check would query payment repo
      },
    },
    { from: InvoiceStatus.SENT, to: InvoiceStatus.PAID },
    { from: InvoiceStatus.SENT, to: InvoiceStatus.OVERDUE },
    { from: InvoiceStatus.OVERDUE, to: InvoiceStatus.PAID },
    { from: InvoiceStatus.OVERDUE, to: InvoiceStatus.CANCELLED },
  ]
}
```

---

## Testing State Machines

```typescript
// tests/unit/state_machines/invoice_state_machine.spec.ts
import { test } from '@japa/runner'
import { invoiceStateMachine } from '#state_machines/invoice_state_machine'
import { InvoiceStatus } from '#models/invoice'
import StateMachineException from '#exceptions/state_machine_exception'

test.group('InvoiceStateMachine', () => {
  test('allows DRAFT to SENT', ({ assert }) => {
    assert.isTrue(invoiceStateMachine.canTransition(InvoiceStatus.DRAFT, InvoiceStatus.SENT))
  })

  test('prevents DRAFT to PAID', ({ assert }) => {
    assert.isFalse(invoiceStateMachine.canTransition(InvoiceStatus.DRAFT, InvoiceStatus.PAID))
  })

  test('prevents PAID to SENT (no going back)', ({ assert }) => {
    assert.isFalse(invoiceStateMachine.canTransition(InvoiceStatus.PAID, InvoiceStatus.SENT))
  })

  test('throws on invalid transition', ({ assert }) => {
    assert.throws(
      () => invoiceStateMachine.assertCanTransition(InvoiceStatus.DRAFT, InvoiceStatus.PAID, 'inv-123'),
      StateMachineException
    )
  })

  test('lists allowed transitions from OVERDUE', ({ assert }) => {
    const allowed = invoiceStateMachine.getAllowedTransitions(InvoiceStatus.OVERDUE)
    assert.deepEqual(allowed.sort(), [InvoiceStatus.CANCELLED, InvoiceStatus.PAID].sort())
  })
})
```

---

## Key Principles

1. **State machines are domain objects** — no DB, no HTTP. They validate transitions only.
2. **Services call state machines** — the service handles the DB update and event emission.
3. **One state machine per entity lifecycle** — don't combine invoice and customer states.
4. **Enum values in transitions** — always use enum members, never raw strings, so refactors are caught by the compiler.
5. **Events announce transitions** — listeners react to state changes, not the state machine itself.
6. **Test state machines in isolation** — unit tests for the transition map, integration tests for the full flow.
