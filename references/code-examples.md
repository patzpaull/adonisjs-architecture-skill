# Code Examples

Complete, working examples for each architectural layer. Copy-paste ready.

---

## 1. Controller Example

```typescript
// app/controllers/invoices_controller.ts
import type { HttpContext } from '@adonisjs/core/http'
import { inject } from '@adonisjs/core'
import InvoiceService from '#services/invoice_service'
import {
  storeInvoiceValidator,
  updateInvoiceValidator,
} from '#validators/invoice_validator'

export default class InvoicesController {
  @inject()
  async index({ request, response }: HttpContext, invoiceService: InvoiceService) {
    const page = request.input('page', 1)
    const limit = request.input('limit', 20)
    const invoices = await invoiceService.list(page, limit)
    return response.ok(invoices)
  }

  @inject()
  async show({ params, response }: HttpContext, invoiceService: InvoiceService) {
    const invoice = await invoiceService.findOrFail(params.id)
    return response.ok(invoice)
  }

  @inject()
  async store({ request, response }: HttpContext, invoiceService: InvoiceService) {
    const data = await request.validateUsing(storeInvoiceValidator)
    const invoice = await invoiceService.create(data)
    return response.created(invoice)
  }

  @inject()
  async update({ params, request, response }: HttpContext, invoiceService: InvoiceService) {
    const data = await request.validateUsing(updateInvoiceValidator)
    const invoice = await invoiceService.update(params.id, data)
    return response.ok(invoice)
  }
}
```

---

## 2. Service Example

```typescript
// app/services/invoice_service.ts
import { inject } from '@adonisjs/core'
import db from '@adonisjs/lucid/services/db'
import emitter from '@adonisjs/core/services/emitter'
import { InvoiceRepository } from '#repositories/invoice_repository'
import { CustomerRepository } from '#repositories/customer_repository'
import InvoiceCreated from '#events/invoice_created'
import InvoiceOverdue from '#events/invoice_overdue'
import type { StoreInvoiceDto, UpdateInvoiceDto } from '#dtos/invoice_dto'
import { InvoiceStatus } from '#models/invoice'
import InvalidInvoiceStateException from '#exceptions/invalid_invoice_state_exception'

@inject()
export default class InvoiceService {
  constructor(
    private invoiceRepo: InvoiceRepository,
    private customerRepo: CustomerRepository
  ) {}

  async list(page: number, limit: number) {
    return this.invoiceRepo.paginate(page, limit)
  }

  async findOrFail(id: string) {
    return this.invoiceRepo.findOrFail(id)
  }

  async create(data: StoreInvoiceDto) {
    return db.transaction(async (trx) => {
      const customer = await this.customerRepo.findOrFail(data.customerId, trx)
      const invoice = await this.invoiceRepo.create(
        {
          customerId: customer.id,
          amount: data.amount,
          dueDate: data.dueDate,
          status: InvoiceStatus.DRAFT,
          lineItems: data.lineItems,
        },
        trx
      )
      await emitter.emit(new InvoiceCreated(invoice))
      return invoice
    })
  }

  async update(id: string, data: UpdateInvoiceDto) {
    const invoice = await this.invoiceRepo.findOrFail(id)
    if (invoice.status !== InvoiceStatus.DRAFT) {
      throw new InvalidInvoiceStateException(id, invoice.status, InvoiceStatus.DRAFT)
    }
    return this.invoiceRepo.update(invoice, data)
  }

  async markOverdue(invoiceId: string) {
    return db.transaction(async (trx) => {
      const invoice = await this.invoiceRepo.findOrFail(invoiceId, trx)
      if (invoice.status !== InvoiceStatus.SENT) {
        throw new InvalidInvoiceStateException(invoiceId, invoice.status, InvoiceStatus.SENT)
      }
      const updated = await this.invoiceRepo.updateStatus(invoice, InvoiceStatus.OVERDUE, trx)
      await emitter.emit(new InvoiceOverdue(updated))
      return updated
    })
  }
}
```

---

## 3. Repository Example

```typescript
// app/repositories/invoice_repository.ts
import Invoice from '#models/invoice'
import { InvoiceStatus } from '#models/invoice'
import type { TransactionClientContract } from '@adonisjs/lucid/types/database'
import type { DateTime } from 'luxon'

export class InvoiceRepository {
  async findOrFail(id: string, trx?: TransactionClientContract) {
    const query = Invoice.query()
    if (trx) query.useTransaction(trx)
    return query.where('id', id).firstOrFail()
  }

  async paginate(page: number, limit: number) {
    return Invoice.query()
      .orderBy('created_at', 'desc')
      .paginate(page, limit)
  }

  async findByCustomer(customerId: string) {
    return Invoice.query()
      .where('customer_id', customerId)
      .orderBy('created_at', 'desc')
  }

  async findOverdue() {
    return Invoice.query()
      .where('status', InvoiceStatus.OVERDUE)
      .orderBy('due_date', 'asc')
  }

  async findSentDueBefore(date: DateTime) {
    return Invoice.query()
      .where('status', InvoiceStatus.SENT)
      .where('due_date', '<', date.toSQL())
  }

  async create(data: Partial<Invoice>, trx?: TransactionClientContract) {
    const invoice = new Invoice()
    invoice.fill(data)
    if (trx) invoice.useTransaction(trx)
    await invoice.save()
    return invoice
  }

  async update(invoice: Invoice, data: Partial<Invoice>, trx?: TransactionClientContract) {
    invoice.merge(data)
    if (trx) invoice.useTransaction(trx)
    await invoice.save()
    return invoice
  }

  async updateStatus(invoice: Invoice, status: InvoiceStatus, trx?: TransactionClientContract) {
    return this.update(invoice, { status }, trx)
  }

  async sumUnpaidByCustomer(customerId: string): Promise<number> {
    const result = await Invoice.query()
      .where('customer_id', customerId)
      .whereIn('status', [InvoiceStatus.SENT, InvoiceStatus.OVERDUE])
      .sum('amount as total')
      .first()

    return Number(result?.$extras.total ?? 0)
  }
}
```

---

## 4. DTO Examples

DTOs live in `app/dtos/` organised by domain subdirectory. Three types are used:
- **Input DTOs** — data flowing into a service (from validator + controller-attached context)
- **Response DTOs** — output shapes when the Lucid model is insufficient
- **Nested DTOs** — sub-shapes used inside input or response DTOs

### Folder structure

```
app/dtos/
├── payments/
│   ├── record_payment_dto.ts
│   └── payment_response_dto.ts
├── invoices/
│   └── invoice_dto.ts
└── tasks/
    └── task_dto.ts
```

### Input DTO — with controller-attached context

```typescript
// app/dtos/payments/record_payment_dto.ts
import type { PaymentMethod } from '#enums/payment_method'

export interface RecordPaymentDto {
  // From request body (validated by VineJS)
  invoiceId: string
  amount: number
  method: PaymentMethod
  reference?: string
  // Attached by the controller from auth context — never from request.body()
  recordedByUserId: string
  tenantId: string
}
```

**Controller building the enriched DTO:**

```typescript
// app/controllers/payments_controller.ts
async store({ request, auth, response }: HttpContext) {
  const data = await request.validateUsing(storePaymentValidator)

  const dto: RecordPaymentDto = {
    ...data,
    recordedByUserId: auth.user!.id,
    tenantId: auth.user!.tenantId,
  }

  const payment = await this.paymentService.recordPayment(dto)
  return response.created(payment)
}
```

### Input DTO — with nested shape

```typescript
// app/dtos/invoices/invoice_dto.ts

export interface LineItemDto {
  description: string
  quantity: number
  unitPrice: number
}

export interface StoreInvoiceDto {
  customerId: string
  amount: number
  dueDate: string          // ISO date string — convert to DateTime inside the service
  lineItems?: LineItemDto[]
  // Context:
  createdByUserId: string
}

export interface UpdateInvoiceDto {
  amount?: number
  dueDate?: string
  lineItems?: LineItemDto[]
}
```

### Response DTO — when model shape is insufficient

Return a plain DTO from the service (never from the controller) when:
- The response includes computed fields not stored in the DB
- Fields are aggregated from multiple models
- A subset or projection of the model is needed

```typescript
// app/dtos/payments/payment_response_dto.ts

export interface PaymentResponseDto {
  id: string
  invoiceId: string
  amount: number
  method: string
  reference: string | null
  paidAt: string             // ISO string — Luxon DateTime serialised here, not in controller
  invoiceBalance: number     // computed: invoice.amount - total payments
  receiptUrl: string         // constructed at serialization time
}
```

**Service building the response DTO:**

```typescript
// app/services/payment_service.ts
async recordPayment(dto: RecordPaymentDto): Promise<PaymentResponseDto> {
  return db.transaction(async (trx) => {
    const invoice = await this.invoiceRepo.findOrFail(dto.invoiceId, trx)
    const payment = await this.paymentRepo.create(dto, trx)
    const balance = await this.invoiceRepo.remainingBalance(invoice.id, trx)

    return {
      id: payment.id,
      invoiceId: payment.invoiceId,
      amount: payment.amount,
      method: payment.method,
      reference: payment.reference ?? null,
      paidAt: payment.paidAt.toISO()!,
      invoiceBalance: balance,
      receiptUrl: `/receipts/${payment.id}`,
    }
  })
}
```

### Complex input DTO — real-world task completion

```typescript
// app/dtos/tasks/task_dto.ts
import type { ACCondition } from '#enums/ac_condition'
import type { TaskStatus } from '#enums/task_status'

export interface TaskWorkCompleteDto {
  taskId: number
  taskWorkId: number
  status: TaskStatus
  note?: string | null
  reasonForReschedule?: string | null
  acCondition?: ACCondition | null
  completedAt?: Date | null
  newRescheduledDate?: Date | null
  wifiModuleId?: number | null
  iduSn?: string | null
  oduSn?: string | null
  iduModel?: string | null
  oduModel?: string | null
  images: { type: 'before' | 'after' | 'longladder'; image: string }[]
  sendNotification?: boolean | null
  taskQuestions?: { taskQuestionId: number; taskQuestionChoiceId: number }[]
  taskSpareParts?: { sparePartId: number; quantity: number }[]
  taskMaterials?: { name: string; value: number; unit: string; category: string }[]
  // Context:
  completedByUserId: string
}

export interface TaskCreateDto {
  customerContractId: number | null
  description: string | null
  iduSn: string | null
  iduModel: string | null
  oduSn: string | null
  oduModel: string | null
  assetTypeId: number | null
  // Context:
  assignedByUserId: string
}
```

### `package.json` subpath import

```json
{
  "imports": {
    "#dtos/*": "./app/dtos/*.js"
  }
}
```

Usage:

```typescript
import type { RecordPaymentDto } from '#dtos/payments/record_payment_dto'
import type { StoreInvoiceDto, LineItemDto } from '#dtos/invoices/invoice_dto'
import type { TaskWorkCompleteDto } from '#dtos/tasks/task_dto'
```

---

## 5. Validator Example

```typescript
// app/validators/invoice_validator.ts
import vine from '@vinejs/vine'

export const storeInvoiceValidator = vine.compile(
  vine.object({
    customerId: vine.string().uuid(),
    amount: vine.number().positive(),
    dueDate: vine.date({ formats: ['YYYY-MM-DD'] }),
    lineItems: vine
      .array(
        vine.object({
          description: vine.string().trim().maxLength(255),
          quantity: vine.number().positive(),
          unitPrice: vine.number().min(0),
        })
      )
      .optional(),
  })
)

export const updateInvoiceValidator = vine.compile(
  vine.object({
    amount: vine.number().positive().optional(),
    dueDate: vine.date({ formats: ['YYYY-MM-DD'] }).optional(),
    lineItems: vine
      .array(
        vine.object({
          description: vine.string().trim().maxLength(255),
          quantity: vine.number().positive(),
          unitPrice: vine.number().min(0),
        })
      )
      .optional(),
  })
)

// app/validators/payment_validator.ts
export const recordPaymentValidator = vine.compile(
  vine.object({
    invoiceId: vine.string().uuid(),
    amount: vine.number().positive(),
    method: vine.enum(Object.values(PaymentMethod)),
    reference: vine.string().optional(),
    paidAt: vine.date({ formats: ['YYYY-MM-DD'] }).optional(),
  })
)
```

---

## 6. Event & Listener Example

```typescript
// app/events/invoice_created.ts
import type Invoice from '#models/invoice'

export default class InvoiceCreated {
  constructor(public invoice: Invoice) {}
}

// app/events/invoice_overdue.ts
import type Invoice from '#models/invoice'

export default class InvoiceOverdue {
  constructor(public invoice: Invoice) {}
}

// app/events/payment_received.ts
import type Payment from '#models/payment'

export default class PaymentReceived {
  constructor(public payment: Payment) {}
}
```

```typescript
// app/listeners/send_invoice_email.ts
import type InvoiceCreated from '#events/invoice_created'
import mail from '@adonisjs/mail/services/main'

export default class SendInvoiceEmail {
  async handle(event: InvoiceCreated) {
    const { invoice } = event
    await invoice.load('customer')

    await mail.send((message) => {
      message
        .to(invoice.customer.email)
        .subject(`Invoice #${invoice.number} — ${invoice.customer.name}`)
        .htmlView('emails/invoice_created', { invoice })
    })
  }
}
```

```typescript
// app/listeners/send_overdue_reminder.ts
import type InvoiceOverdue from '#events/invoice_overdue'
import mail from '@adonisjs/mail/services/main'

export default class SendOverdueReminder {
  async handle(event: InvoiceOverdue) {
    const { invoice } = event
    await invoice.load('customer')

    await mail.send((message) => {
      message
        .to(invoice.customer.email)
        .subject(`Overdue: Invoice #${invoice.number}`)
        .htmlView('emails/invoice_overdue', { invoice })
    })
  }
}
```

```typescript
// start/events.ts
import emitter from '@adonisjs/core/services/emitter'

const InvoiceCreated = () => import('#events/invoice_created')
const InvoiceOverdue = () => import('#events/invoice_overdue')
const PaymentReceived = () => import('#events/payment_received')

const SendInvoiceEmail = () => import('#listeners/send_invoice_email')
const SendOverdueReminder = () => import('#listeners/send_overdue_reminder')
const UpdateInvoiceBalance = () => import('#listeners/update_invoice_balance')

emitter.on(InvoiceCreated, [SendInvoiceEmail])
emitter.on(InvoiceOverdue, [SendOverdueReminder])
emitter.on(PaymentReceived, [UpdateInvoiceBalance])
```

---

## 7. Domain Object Example

```typescript
// app/domain/billing/invoice_calculator.ts
export interface LineItem {
  description: string
  quantity: number
  unitPrice: number
}

export interface InvoiceTotals {
  subtotal: number
  taxAmount: number
  total: number
}

export class InvoiceCalculator {
  /**
   * Calculate invoice totals from line items and a tax rate.
   * Pure function — no DB, no framework dependencies.
   */
  static calculate(lineItems: LineItem[], taxRatePercent: number = 0): InvoiceTotals {
    const subtotal = lineItems.reduce((sum, item) => {
      return sum + Math.round(item.quantity * item.unitPrice * 100) / 100
    }, 0)

    const taxAmount = Math.round(subtotal * (taxRatePercent / 100) * 100) / 100
    const total = Math.round((subtotal + taxAmount) * 100) / 100

    return { subtotal, taxAmount, total }
  }

  static lineItemTotal(item: LineItem): number {
    return Math.round(item.quantity * item.unitPrice * 100) / 100
  }
}
```

---

## 8. Value Object Example

```typescript
// app/domain/value_objects/money.ts
export class Money {
  private constructor(
    private readonly cents: number,
    public readonly currency: string = 'TZS'
  ) {}

  static fromAmount(amount: number, currency = 'TZS'): Money {
    return new Money(Math.round(amount * 100), currency)
  }

  static fromCents(cents: number, currency = 'TZS'): Money {
    return new Money(cents, currency)
  }

  static zero(currency = 'TZS'): Money {
    return new Money(0, currency)
  }

  get amount(): number {
    return this.cents / 100
  }

  add(other: Money): Money {
    this.assertSameCurrency(other)
    return new Money(this.cents + other.cents, this.currency)
  }

  subtract(other: Money): Money {
    this.assertSameCurrency(other)
    return new Money(this.cents - other.cents, this.currency)
  }

  multiply(factor: number): Money {
    return new Money(Math.round(this.cents * factor), this.currency)
  }

  isPositive(): boolean { return this.cents > 0 }
  isZero(): boolean { return this.cents === 0 }
  isNegative(): boolean { return this.cents < 0 }

  greaterThan(other: Money): boolean {
    this.assertSameCurrency(other)
    return this.cents > other.cents
  }

  equals(other: Money): boolean {
    return this.cents === other.cents && this.currency === other.currency
  }

  private assertSameCurrency(other: Money) {
    if (this.currency !== other.currency) {
      throw new Error(`Cannot operate on different currencies: ${this.currency} vs ${other.currency}`)
    }
  }
}
```

---

## 9. Exception Example

```typescript
// app/exceptions/invalid_invoice_state_exception.ts
import { Exception } from '@adonisjs/core/exceptions'
import type { InvoiceStatus } from '#models/invoice'

export default class InvalidInvoiceStateException extends Exception {
  static status = 422
  static code = 'E_INVALID_INVOICE_STATE'

  constructor(invoiceId: string, currentState: InvoiceStatus, expectedState: InvoiceStatus) {
    super(
      `Invoice ${invoiceId} is in "${currentState}" state but must be in "${expectedState}" to perform this operation`
    )
  }
}

// app/exceptions/customer_not_found_exception.ts
import { Exception } from '@adonisjs/core/exceptions'

export default class CustomerNotFoundException extends Exception {
  static status = 404
  static code = 'E_CUSTOMER_NOT_FOUND'

  constructor(identifier: string) {
    super(`Customer "${identifier}" was not found`)
  }
}
```

---

## 10. Action Example

```typescript
// app/actions/record_invoice_payment_action.ts
import { inject } from '@adonisjs/core'
import db from '@adonisjs/lucid/services/db'
import emitter from '@adonisjs/core/services/emitter'
import { InvoiceRepository } from '#repositories/invoice_repository'
import { PaymentRepository } from '#repositories/payment_repository'
import { InvoiceStatus } from '#models/invoice'
import PaymentReceived from '#events/payment_received'
import InvalidInvoiceStateException from '#exceptions/invalid_invoice_state_exception'
import type { RecordPaymentDto } from '#dtos/payment_dto'

/**
 * Records a payment against an invoice and transitions invoice status.
 * Extracted from PaymentService because the partial-payment and
 * status-transition logic is complex enough to warrant its own class.
 */
@inject()
export default class RecordInvoicePaymentAction {
  constructor(
    private invoiceRepo: InvoiceRepository,
    private paymentRepo: PaymentRepository
  ) {}

  async execute(data: RecordPaymentDto) {
    return db.transaction(async (trx) => {
      const invoice = await this.invoiceRepo.findOrFail(data.invoiceId, trx)

      if (
        invoice.status !== InvoiceStatus.SENT &&
        invoice.status !== InvoiceStatus.OVERDUE
      ) {
        throw new InvalidInvoiceStateException(
          data.invoiceId,
          invoice.status,
          InvoiceStatus.SENT
        )
      }

      const payment = await this.paymentRepo.create(
        {
          invoiceId: invoice.id,
          amount: data.amount,
          method: data.method,
          reference: data.reference ?? null,
          paidAt: data.paidAt ?? DateTime.now(),
        },
        trx
      )

      const newBalanceDue = invoice.balanceDue - data.amount
      const newStatus = newBalanceDue <= 0 ? InvoiceStatus.PAID : invoice.status

      await this.invoiceRepo.update(
        invoice,
        { balanceDue: Math.max(0, newBalanceDue), status: newStatus },
        trx
      )

      await emitter.emit(new PaymentReceived(payment))
      return payment
    })
  }
}
```

---

## 11. Service Provider Example

```typescript
// app/providers/repository_provider.ts
import type { ApplicationService } from '@adonisjs/core/types'
import { InvoiceRepository } from '#repositories/invoice_repository'
import { CustomerRepository } from '#repositories/customer_repository'
import { PaymentRepository } from '#repositories/payment_repository'
import InvoiceService from '#services/invoice_service'
import PaymentService from '#services/payment_service'

export default class RepositoryProvider {
  constructor(protected app: ApplicationService) {}

  register() {
    // Repositories — stateless, register as singletons
    this.app.container.singleton(InvoiceRepository, () => new InvoiceRepository())
    this.app.container.singleton(CustomerRepository, () => new CustomerRepository())
    this.app.container.singleton(PaymentRepository, () => new PaymentRepository())

    // Services — stateless, depend on singleton repos
    this.app.container.singleton(InvoiceService, async (resolver) => {
      return new InvoiceService(
        await resolver.make(InvoiceRepository),
        await resolver.make(CustomerRepository)
      )
    })

    this.app.container.singleton(PaymentService, async (resolver) => {
      return new PaymentService(
        await resolver.make(InvoiceRepository),
        await resolver.make(PaymentRepository)
      )
    })
  }

  async boot() {}
}
```

---

## 12. Route Definition Example

```typescript
// start/routes.ts
import router from '@adonisjs/core/services/router'
import { middleware } from '#start/kernel'

const CustomersController = () => import('#controllers/customers_controller')
const InvoicesController = () => import('#controllers/invoices_controller')
const PaymentsController = () => import('#controllers/payments_controller')

router
  .group(() => {
    router.resource('customers', CustomersController).apiOnly()
    router.resource('invoices', InvoicesController).apiOnly()
    router.resource('payments', PaymentsController).apiOnly().except(['destroy'])
  })
  .prefix('api/v1')
  .middleware(middleware.auth({ guards: ['api'] }))
```

---

## 13. Model Example (Schema + Relationships + Scopes Only)

```typescript
// app/models/invoice.ts
import { DateTime } from 'luxon'
import { BaseModel, column, belongsTo, hasMany, scope } from '@adonisjs/lucid/orm'
import type { BelongsTo, HasMany } from '@adonisjs/lucid/types/relations'
import Customer from '#models/customer'
import Payment from '#models/payment'

export enum InvoiceStatus {
  DRAFT = 'DRAFT',
  SENT = 'SENT',
  PAID = 'PAID',
  OVERDUE = 'OVERDUE',
  CANCELLED = 'CANCELLED',
}

export default class Invoice extends BaseModel {
  @column({ isPrimary: true })
  declare id: string

  @column()
  declare customerId: string

  @column()
  declare number: string

  @column()
  declare amount: number

  @column()
  declare balanceDue: number

  @column()
  declare status: InvoiceStatus

  @column.date()
  declare dueDate: DateTime

  @column.dateTime({ autoCreate: true })
  declare createdAt: DateTime

  @column.dateTime({ autoCreate: true, autoUpdate: true })
  declare updatedAt: DateTime

  // --- Relationships ---

  @belongsTo(() => Customer)
  declare customer: BelongsTo<typeof Customer>

  @hasMany(() => Payment)
  declare payments: HasMany<typeof Payment>

  // --- Query Scopes ---

  static unpaid = scope((query) => {
    query.whereIn('status', [InvoiceStatus.SENT, InvoiceStatus.OVERDUE])
  })

  static overdue = scope((query) => {
    query.where('status', InvoiceStatus.OVERDUE)
  })

  static dueBefore = scope((query, date: DateTime) => {
    query.where('due_date', '<', date.toSQL()!)
  })
}
```

```typescript
// app/models/customer.ts
import { DateTime } from 'luxon'
import { BaseModel, column, hasMany } from '@adonisjs/lucid/orm'
import type { HasMany } from '@adonisjs/lucid/types/relations'
import Invoice from '#models/invoice'

export enum CustomerStatus {
  ACTIVE = 'ACTIVE',
  INACTIVE = 'INACTIVE',
  SUSPENDED = 'SUSPENDED',
}

export default class Customer extends BaseModel {
  @column({ isPrimary: true })
  declare id: string

  @column()
  declare name: string

  @column()
  declare email: string

  @column()
  declare phone: string | null

  @column()
  declare status: CustomerStatus

  @column.dateTime({ autoCreate: true })
  declare createdAt: DateTime

  @column.dateTime({ autoCreate: true, autoUpdate: true })
  declare updatedAt: DateTime

  // --- Relationships ---

  @hasMany(() => Invoice)
  declare invoices: HasMany<typeof Invoice>
}
```

```typescript
// app/models/payment.ts
import { DateTime } from 'luxon'
import { BaseModel, column, belongsTo } from '@adonisjs/lucid/orm'
import type { BelongsTo } from '@adonisjs/lucid/types/relations'
import Invoice from '#models/invoice'

export enum PaymentMethod {
  CASH = 'CASH',
  BANK_TRANSFER = 'BANK_TRANSFER',
  CARD = 'CARD',
  MOBILE_MONEY = 'MOBILE_MONEY',
}

export default class Payment extends BaseModel {
  @column({ isPrimary: true })
  declare id: string

  @column()
  declare invoiceId: string

  @column()
  declare amount: number

  @column()
  declare method: PaymentMethod

  @column()
  declare reference: string | null

  @column.dateTime()
  declare paidAt: DateTime

  @column.dateTime({ autoCreate: true })
  declare createdAt: DateTime

  @column.dateTime({ autoCreate: true, autoUpdate: true })
  declare updatedAt: DateTime

  // --- Relationships ---

  @belongsTo(() => Invoice)
  declare invoice: BelongsTo<typeof Invoice>
}
```

---

## 14. Configuration: package.json Subpath Imports

AdonisJS v6 uses Node.js subpath imports (`imports` field in `package.json`) for the `#` alias. No TypeScript path aliases needed — Node resolves these natively.

```json
{
  "name": "my-crm-api",
  "type": "module",
  "imports": {
    "#controllers/*": "./app/controllers/*.js",
    "#services/*": "./app/services/*.js",
    "#repositories/*": "./app/repositories/*.js",
    "#models/*": "./app/models/*.js",
    "#validators/*": "./app/validators/*.js",
    "#dtos/*": "./app/dtos/*.js",
    "#events/*": "./app/events/*.js",
    "#listeners/*": "./app/listeners/*.js",
    "#domain/*": "./app/domain/*.js",
    "#actions/*": "./app/actions/*.js",
    "#state_machines/*": "./app/state_machines/*.js",
    "#contracts/*": "./app/contracts/*.js",
    "#exceptions/*": "./app/exceptions/*.js",
    "#policies/*": "./app/policies/*.js",
    "#middleware/*": "./app/middleware/*.js",
    "#providers/*": "./app/providers/*.js",
    "#config/*": "./config/*.js",
    "#database/*": "./database/*.js"
  }
}
```

---

## 15. Configuration: tsconfig.json

```json
{
  "extends": "@adonisjs/tsconfig/tsconfig.app.json",
  "compilerOptions": {
    "rootDir": "./",
    "outDir": "./build",
    "paths": {
      "#controllers/*": ["./app/controllers/*.js"],
      "#services/*": ["./app/services/*.js"],
      "#repositories/*": ["./app/repositories/*.js"],
      "#models/*": ["./app/models/*.js"],
      "#validators/*": ["./app/validators/*.js"],
      "#dtos/*": ["./app/dtos/*.js"],
      "#events/*": ["./app/events/*.js"],
      "#listeners/*": ["./app/listeners/*.js"],
      "#domain/*": ["./app/domain/*.js"],
      "#actions/*": ["./app/actions/*.js"],
      "#state_machines/*": ["./app/state_machines/*.js"],
      "#contracts/*": ["./app/contracts/*.js"],
      "#exceptions/*": ["./app/exceptions/*.js"],
      "#policies/*": ["./app/policies/*.js"],
      "#middleware/*": ["./app/middleware/*.js"],
      "#providers/*": ["./app/providers/*.js"],
      "#config/*": ["./config/*.js"],
      "#database/*": ["./database/*.js"]
    }
  }
}
```

---

## 16. Configuration: adonisrc.ts (Provider Registration)

```typescript
// adonisrc.ts
import { defineConfig } from '@adonisjs/core/app'

export default defineConfig({
  typescript: true,

  providers: [
    () => import('@adonisjs/core/providers/app_provider'),
    () => import('@adonisjs/core/providers/hash_provider'),
    {
      file: () => import('@adonisjs/core/providers/repl_provider'),
      environment: ['repl'],
    },
    () => import('@adonisjs/core/providers/vinejs_provider'),
    () => import('@adonisjs/lucid/database_provider'),
    () => import('@adonisjs/auth/auth_provider'),

    // Custom providers — register repositories and services here
    () => import('#providers/repository_provider'),
  ],

  preloads: [
    () => import('#start/routes'),
    () => import('#start/events'),
    {
      file: () => import('#start/kernel'),
      environment: ['web', 'test'],
    },
  ],

  commands: [
    () => import('@adonisjs/core/commands'),
    () => import('@adonisjs/lucid/commands'),
  ],
})
```

---

## 17. Queue Integration Example

When a listener needs to do slow work (send email, generate PDF, call webhook), dispatch a job instead of doing the work inline.

```typescript
// app/jobs/send_invoice_email_job.ts
import { JobContract } from '@ioc:Rocketseat/Bull'
import Invoice from '#models/invoice'
import mail from '@adonisjs/mail/services/main'

export interface SendInvoiceEmailPayload {
  invoiceId: string
}

export default class SendInvoiceEmailJob implements JobContract {
  public key = 'SendInvoiceEmailJob'

  async handle(job: { data: SendInvoiceEmailPayload }) {
    const invoice = await Invoice.query()
      .where('id', job.data.invoiceId)
      .preload('customer')
      .firstOrFail()

    await mail.send((message) => {
      message
        .to(invoice.customer.email)
        .subject(`Invoice #${invoice.number}`)
        .htmlView('emails/invoice', { invoice })
    })
  }
}
```

```typescript
// app/listeners/send_invoice_email.ts — dispatches to queue, returns immediately
import type InvoiceCreated from '#events/invoice_created'
import Bull from '@ioc:Rocketseat/Bull'

export default class SendInvoiceEmail {
  async handle(event: InvoiceCreated) {
    await Bull.add('SendInvoiceEmailJob', { invoiceId: event.invoice.id })
  }
}
```

> **Note:** The example above uses `@rocketseat/adonis-bull`. For vanilla AdonisJS v6 you can use `bullmq` directly. The pattern is identical: listeners dispatch to queue, jobs do the heavy lifting.

---

## 18. Global Exception Handler

```typescript
// app/exceptions/handler.ts
import app from '@adonisjs/core/services/app'
import { HttpContext, ExceptionHandler } from '@adonisjs/core/http'
import { errors as lucidErrors } from '@adonisjs/lucid'
import DomainException from '#exceptions/domain_exception'

export default class HttpExceptionHandler extends ExceptionHandler {
  protected debug = !app.inProduction

  async handle(error: unknown, ctx: HttpContext) {
    if (error instanceof DomainException) {
      return ctx.response.status(error.status).send({
        error: error.code,
        message: error.message,
      })
    }

    if (error instanceof lucidErrors.E_ROW_NOT_FOUND) {
      return ctx.response.status(404).send({
        error: 'NOT_FOUND',
        message: 'Resource not found',
      })
    }

    return super.handle(error, ctx)
  }

  async report(error: unknown, ctx: HttpContext) {
    return super.report(error, ctx)
  }
}
```

```typescript
// app/exceptions/domain_exception.ts
import { Exception } from '@adonisjs/core/exceptions'

export default abstract class DomainException extends Exception {
  abstract code: string
}
```

Throwing `new InvalidInvoiceStateException(id, current, expected)` in any service automatically returns:

```json
{ "error": "E_INVALID_INVOICE_STATE", "message": "Invoice abc is in \"draft\" state but must be in \"sent\"..." }
```

---

## 19. Pagination Example

Pagination is owned by the **repository**. The service passes `page`/`limit` through; the controller reads them from the request.

```typescript
// app/repositories/invoice_repository.ts
import Invoice from '#models/invoice'

export class InvoiceRepository {
  async list(tenantId: string, page: number, limit: number) {
    return Invoice.query()
      .where('tenant_id', tenantId)
      .orderBy('created_at', 'desc')
      .paginate(page, limit)
  }
}
```

```typescript
// app/services/invoice_service.ts
@inject()
export default class InvoiceService {
  constructor(private invoiceRepo: InvoiceRepository) {}

  async list(tenantId: string, page: number, limit: number) {
    return this.invoiceRepo.list(tenantId, page, limit)
  }
}
```

```typescript
// app/controllers/invoices_controller.ts
@inject()
async index({ request, response, auth }: HttpContext, invoiceService: InvoiceService) {
  const page = request.input('page', 1)
  const limit = request.input('limit', 20)
  const invoices = await invoiceService.list(auth.user!.tenantId, page, limit)
  return response.ok(invoices)
}
```

The paginator serializes to:

```json
{
  "meta": {
    "total": 150,
    "per_page": 20,
    "current_page": 1,
    "last_page": 8,
    "first_page": 1,
    "first_page_url": "/?page=1",
    "last_page_url": "/?page=8",
    "next_page_url": "/?page=2",
    "previous_page_url": null
  },
  "data": [...]
}
```

with HTTP status `422`.
