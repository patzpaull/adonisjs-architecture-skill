# Observability: OpenTelemetry & Prometheus Metrics

Production-grade observability for AdonisJS v6 APIs using a hybrid approach: `@adonisjs/otel` for distributed tracing and `prom-client` for Prometheus-compatible metrics.

The architectural rationale lives in `references/decisions.md` (Decision 20). This file is the implementation reference.

---

## Package Stack

```json
{
  "dependencies": {
    "@adonisjs/otel": "^1.2.0",
    "@opentelemetry/api": "^1.9.0",
    "@opentelemetry/exporter-trace-otlp-proto": "^0.211.0",
    "@opentelemetry/instrumentation-runtime-node": "^0.24.0",
    "prom-client": "^15.1.3"
  }
}
```

**Why not OTEL metrics?** `@adonisjs/otel` uses the OTEL metrics SDK but its Prometheus exporter requires a pull-model HTTP endpoint that conflicts with AdonisJS's server lifecycle. `prom-client` is battle-tested for Prometheus scraping and has zero friction — one registry, one `/metrics` route.

---

## Initialization Order (Critical)

OTEL auto-instrumentation must start **before** the AdonisJS Ignitor loads any app code. Create a top-level `otel.ts` and import it first in `bin/server.ts`:

```typescript
// otel.ts  (project root — NOT in app/)
import { diag, DiagConsoleLogger, DiagLogLevel } from '@opentelemetry/api'
import { init } from '@adonisjs/otel/init'

if (process.env.NODE_ENV === 'development') {
  diag.setLogger(new DiagConsoleLogger(), DiagLogLevel.INFO)
}

await init(import.meta.dirname)
```

```typescript
// bin/server.ts — otel.ts MUST be first
import '../otel.js'                          // <-- line 1, before everything
import { Ignitor, prettyPrintError } from '@adonisjs/core'

// ... rest of server bootstrap
```

If `otel.ts` is imported after Ignitor starts, HTTP and database auto-instrumentation will miss early traffic.

---

## OTEL Configuration

```typescript
// config/otel.ts
import { defineConfig } from '@adonisjs/otel'
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-proto'
import { RuntimeNodeInstrumentation } from '@opentelemetry/instrumentation-runtime-node'

export default defineConfig({
  enabled: true,

  traceExporter: new OTLPTraceExporter({
    url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT,
  }),

  // Disable OTEL metric readers — prom-client handles metrics
  metricReaders: [],

  samplingRatio: Number.parseFloat(process.env.OTEL_TRACES_SAMPLER_ARG || '1.0'),
  serviceName: process.env.APP_NAME || 'my-api',
  serviceVersion: process.env.APP_VERSION || '0.1.0',
  environment: process.env.APP_ENV || 'development',

  instrumentations: {
    '@opentelemetry/instrumentation-runtime-node': new RuntimeNodeInstrumentation(),
  },
})
```

**Environment variables:**

| Variable | Purpose | Default |
|----------|---------|---------|
| `OTEL_EXPORTER_OTLP_ENDPOINT` | Collector endpoint (Jaeger, Tempo, Datadog Agent) | — |
| `OTEL_TRACES_SAMPLER_ARG` | Sampling ratio `0.0`–`1.0` | `1.0` (100%) |
| `APP_NAME` | Service name in traces | `'my-api'` |
| `APP_VERSION` | Service version tag | `'0.1.0'` |
| `APP_ENV` | Environment tag (`production`, `staging`) | `'development'` |

---

## OtelService: Central Metrics & Tracing Interface

One static class owns the Prometheus registry, all metric definitions, and the span helper. Place it at `app/services/otel_service.ts`.

```typescript
// app/services/otel_service.ts
import { trace, type Span } from '@opentelemetry/api'
import { Registry, Counter, Gauge, Histogram, collectDefaultMetrics } from 'prom-client'

export default class OtelService {
  private static readonly SERVICE_NAME = process.env.APP_NAME || 'my-api'

  // Tracer for manual span creation
  public static readonly tracer = trace.getTracer(this.SERVICE_NAME)

  // Single Prometheus registry — all metrics registered here
  public static readonly register = new Registry()

  // Static initializer: runs once at import time
  static {
    OtelService.register.setDefaultLabels({
      app: OtelService.SERVICE_NAME,
      env: process.env.NODE_ENV || 'development',
    })
    collectDefaultMetrics({ register: OtelService.register })
  }

  // --- Counters (ever-increasing event counts) ---
  public static readonly counters = {
    httpRequests: OtelService.createCounter(
      'http_requests_total',
      'Total HTTP requests',
      ['http_method', 'http_route', 'http_status_code', 'user_account_type']
    ),
    authEvents: OtelService.createCounter(
      'auth_events_total',
      'Authentication attempts',
      ['operation', 'status', 'user_account_type']
    ),
    exceptions: OtelService.createCounter(
      'app_exceptions_total',
      'Application exceptions',
      ['operation', 'error_type', 'http_route']
    ),
    dbOperations: OtelService.createCounter(
      'db_operations_total',
      'Database operations',
      ['connection', 'operation', 'query_type']
    ),
    cacheOperations: OtelService.createCounter(
      'cache_operations_total',
      'Cache operations',
      ['cache_operation', 'cache_store']
    ),
  }

  // --- Gauges (point-in-time state) ---
  public static readonly gauges = {
    queueDepth: OtelService.createGauge(
      'queue_depth_count',
      'Messages in queue',
      ['queue_name']
    ),
    activeUsers: OtelService.createGauge(
      'active_users_count',
      'Active users in current window',
      ['account_type']
    ),
  }

  // --- Histograms (latency distributions) ---
  public static readonly histograms = {
    httpRequestDuration: OtelService.createHistogram(
      'http_request_duration_seconds',
      'HTTP request-response cycle duration',
      ['http_method', 'http_route', 'http_status_code']
    ),
    dbQueryDuration: OtelService.createHistogram(
      'db_query_duration_seconds',
      'Database query execution time',
      ['connection', 'query_type']
    ),
    externalApiDuration: OtelService.createHistogram(
      'external_api_duration_seconds',
      'External API call latency',
      ['operation', 'status']
    ),
  }

  // --- Span helper ---
  public static async withSpan<T>(
    name: string,
    operation: (span: Span) => Promise<T>,
    attributes?: Record<string, string | number | boolean>
  ): Promise<T> {
    return this.tracer.startActiveSpan(name, async (span) => {
      if (attributes) span.setAttributes(attributes)
      const start = process.hrtime()
      try {
        const result = await operation(span)
        span.setStatus({ code: 0 })
        return result
      } catch (error) {
        span.recordException(error as Error)
        span.setStatus({ code: 2, message: (error as Error).message })
        this.counters.exceptions.add(1, {
          operation: name,
          error_type: (error as Error).constructor.name,
        })
        throw error
      } finally {
        const diff = process.hrtime(start)
        span.setAttribute('operation.duration_seconds', diff[0] + diff[1] / 1e9)
        span.end()
      }
    })
  }

  public static recordCache(op: 'hit' | 'miss' | 'set' | 'delete', store = 'default') {
    this.counters.cacheOperations.add(1, { cache_operation: op, cache_store: store })
  }

  // --- Private factory helpers ---
  private static createCounter(name: string, help: string, labelNames: string[]) {
    const counter = new Counter({ name, help, labelNames, registers: [this.register] })
    return {
      add: (val: number, labels?: Record<string, string | number>) =>
        counter.inc(this.cleanLabels(labels, labelNames), val),
    }
  }

  private static createGauge(name: string, help: string, labelNames: string[]) {
    const gauge = new Gauge({ name, help, labelNames, registers: [this.register] })
    return {
      set: (val: number, labels?: Record<string, string | number>) =>
        gauge.set(this.cleanLabels(labels, labelNames), val),
    }
  }

  private static createHistogram(name: string, help: string, labelNames: string[]) {
    const histogram = new Histogram({ name, help, labelNames, registers: [this.register] })
    return {
      record: (val: number, labels?: Record<string, string | number>) =>
        histogram.observe(this.cleanLabels(labels, labelNames), val),
    }
  }

  private static cleanLabels(
    labels: Record<string, string | number> | undefined,
    allowedLabels: string[]
  ): Record<string, string> {
    const clean: Record<string, string> = {}
    for (const label of allowedLabels) clean[label] = 'unknown'
    if (labels) {
      for (const [k, v] of Object.entries(labels)) {
        const key = k.replace(/\./g, '_')
        if (allowedLabels.includes(key)) clean[key] = String(v)
      }
    }
    return clean
  }
}
```

**Design rules:**
- One static class. No instances. Imported directly where needed.
- `withSpan()` handles span lifecycle, exception recording, and duration tagging automatically.
- All label names declared up front — `cleanLabels()` ensures Prometheus never receives unexpected label keys.
- `collectDefaultMetrics()` gives you Node.js process metrics (heap, CPU, GC, event loop lag) for free.

---

## Middleware Chain

Two separate middleware pipelines must be registered in the right order.

### Server Middleware (`start/kernel.ts`)

```typescript
server.use([
  () => import('#middleware/container_bindings_middleware'),
  () => import('@adonisjs/cors/cors_middleware'),
  () => import('#middleware/http_metric_middleware'),  // Records Prometheus metrics
])
```

### Router Middleware (`start/kernel.ts`)

```typescript
router.use([
  () => import('@adonisjs/otel/otel_middleware'),   // Creates OTEL span
  () => import('#middleware/otel_enrich'),           // Enriches span with request/user context
  () => import('@adonisjs/core/bodyparser_middleware'),
])
```

**Order matters:** `otel_middleware` creates the span, `otel_enrich` reads it. `http_metric_middleware` runs as a server middleware so it wraps the entire router pipeline, capturing the true end-to-end duration.

### HTTP Metrics Middleware

```typescript
// app/middleware/http_metric_middleware.ts
import type { HttpContext } from '@adonisjs/core/http'
import type { NextFn } from '@adonisjs/core/types/http'
import { performance } from 'node:perf_hooks'
import OtelService from '#services/otel_service'

const IGNORED_ROUTES = ['/metrics', '/health', '/favicon.ico']

export default class HttpMetricMiddleware {
  async handle(ctx: HttpContext, next: NextFn) {
    const start = performance.now()
    try {
      await next()
    } finally {
      const route = ctx.route?.pattern || ctx.request.url()
      if (IGNORED_ROUTES.includes(route)) return

      const duration = performance.now() - start
      const labels = {
        http_method: ctx.request.method(),
        http_route: route,
        http_status_code: ctx.response.response.statusCode,
        user_account_type: (ctx as any).keycloakUser?.account_type || 'anonymous',
      }

      OtelService.counters.httpRequests.add(1, labels)
      OtelService.histograms.httpRequestDuration.record(duration / 1000, labels)
    }
  }
}
```

### OTEL Span Enrichment Middleware

```typescript
// app/middleware/otel_enrich.ts
import { trace, context } from '@opentelemetry/api'
import type { HttpContext } from '@adonisjs/core/http'

export default class OtelEnrichMiddleware {
  async handle(ctx: HttpContext, next: () => Promise<void>) {
    const span = trace.getSpan(context.active())

    if (span) {
      span.setAttribute('http.method', ctx.request.method())
      span.setAttribute('http.url', ctx.request.url())
      span.setAttribute('http.client_ip', ctx.request.ip())
      span.setAttribute('http.user_agent', ctx.request.header('user-agent') || 'unknown')
    }

    try {
      await next()
    } finally {
      if (span) {
        span.setAttribute('http.status_code', ctx.response.getStatus())

        // Populate user attributes after auth middleware has run
        const user = (ctx as any).keycloakUser
        if (user) {
          span.setAttributes({
            'user.id': String(user.id),
            'user.email': user.email || 'unknown',
            'user.account_type': user.account_type || 'unknown',
            'client.id': user.client_id || 'unknown',
          })
        }
      }
    }
  }
}
```

---

## Manual Service Instrumentation

Use `OtelService.withSpan()` to wrap operations that deserve their own trace span. Prefer this on:
- Service methods that call external APIs
- Business operations with meaningful latency (task assignment, payment processing)
- Any operation you want to appear as a named node in your trace waterfall

```typescript
// app/services/site_check_assignment_service.ts
import OtelService from '#services/otel_service'

export default class SiteCheckAssignmentService {
  async assignBEC(task: SiteCheckTask): Promise<Assignment> {
    return OtelService.withSpan(
      'SiteCheckAssignmentService.assignBEC',
      async (span) => {
        span.setAttribute('task.id', task.id)
        span.setAttribute('task.type', task.type)

        const candidates = await this.candidateRepo.findAvailable(task)
        if (!candidates.length) {
          span.addEvent('no_candidates_found')
          throw new NoCandidatesException(task.id)
        }

        const assignment = await this.assignmentRepo.create(task, candidates[0])
        OtelService.counters.taskAssignments.add(1, {
          operation: 'assign_bec',
          status: 'success',
        })
        return assignment
      }
    )
  }
}
```

**Rules:**
- Span names follow `ClassName.methodName` convention for easy filtering.
- Set attributes on the span for queryable context (`task.id`, `task.type`). Keep values scalar (string/number/bool).
- `withSpan()` automatically records exceptions and sets span status — don't duplicate that.
- Do not wrap every method. Instrument operations that have observable latency or failure modes worth tracing.

---

## Database Metrics via Lucid Events

Register global event listeners on the Lucid emitter at boot to capture all query telemetry. Place this in `start/otel_metrics.ts` and import it in `adonisrc.ts`.

```typescript
// start/otel_metrics.ts
import db from '@adonisjs/lucid/services/db'
import OtelService from '#services/otel_service'

// Per-query counter and latency
db.emitter.on('db:query', (query: any) => {
  const sql = (query.sql || '').trim().toUpperCase()
  const queryType = sql.startsWith('SELECT') ? 'SELECT'
    : sql.startsWith('INSERT') ? 'INSERT'
    : sql.startsWith('UPDATE') ? 'UPDATE'
    : sql.startsWith('DELETE') ? 'DELETE'
    : 'OTHER'

  OtelService.counters.dbOperations.add(1, {
    connection: query.connection || 'default',
    operation: 'query',
    query_type: queryType,
  })

  if (query.duration) {
    OtelService.histograms.dbQueryDuration.record(query.duration / 1000, {
      connection: query.connection || 'default',
      query_type: queryType,
    })
  }
})

// DB error tracking
db.emitter.on('db:query:error', () => {
  OtelService.counters.exceptions.add(1, {
    operation: 'database_query',
    error_type: 'DatabaseError',
    http_route: 'database',
  })
})
```

Register in `adonisrc.ts`:

```typescript
// adonisrc.ts
preloads: [
  () => import('#start/routes'),
  () => import('#start/kernel'),
  () => import('#start/events'),
  () => import('#start/otel_metrics'),  // Boot-time telemetry hooks
],
```

**Gauge updates** — for metrics that reflect current system state (queue depth, active tasks), use `setInterval` inside the same file. Keep the interval conservative (30+ seconds) to avoid adding query pressure:

```typescript
setInterval(async () => {
  try {
    const [{ count }] = await db.from('jobs').where('status', 'pending').count('* as count')
    OtelService.gauges.queueDepth.set(Number(count), { queue_name: 'default' })
  } catch {
    // gauge misses are non-fatal
  }
}, 30_000)
```

---

## Metrics Endpoint

Expose Prometheus metrics at `GET /metrics`. This route should bypass authentication middleware.

```typescript
// start/routes.ts
router.get('/metrics', async ({ response }) => {
  response.header('Content-Type', OtelService.register.contentType)
  return response.send(await OtelService.register.metrics())
})
```

Configure Prometheus to scrape this endpoint. In production, restrict access by network policy rather than application-level auth (Prometheus scraper needs unauthenticated access).

---

## Project Structure

Add these files to the standard project layout:

```
otel.ts                        # Root-level OTEL init — imported first in bin/server.ts
config/
└── otel.ts                    # @adonisjs/otel configuration
app/
├── services/
│   └── otel_service.ts        # Tracer + Prometheus registry + all metric definitions
└── middleware/
    ├── http_metric_middleware.ts   # Server middleware: Prometheus HTTP metrics
    └── otel_enrich.ts             # Router middleware: span enrichment with user/request data
start/
└── otel_metrics.ts            # DB query listeners + periodic gauge updates
```

---

## What to Instrument

### Always instrument

| Layer | What | How |
|-------|------|-----|
| HTTP | All requests (method, route, status, duration) | `http_metric_middleware` |
| HTTP | Active span enrichment (user, IP, UA) | `otel_enrich` |
| DB | Query count and latency by type | `start/otel_metrics.ts` |
| Exceptions | All uncaught errors | Exception handler + `withSpan()` auto-records |
| Auth | Login/token verification attempts and latency | Auth middleware |

### Instrument selectively (via `withSpan`)

- External API calls with measurable latency
- Business operations with business-level failure modes (task assignment, payment processing)
- Queue dispatch operations

### Do NOT instrument

- Simple repository `findOrFail()` / CRUD calls — the DB query listener already captures these at the database level
- Pure domain object methods (calculations, state machine transitions) — they're CPU-only and fast
- Internal utility functions

---

## Exception Handler Integration

Record every unhandled exception in the global exception handler:

```typescript
// app/exceptions/handler.ts
import { HttpExceptionHandler } from '@adonisjs/core/http'
import type { HttpContext } from '@adonisjs/core/http'
import OtelService from '#services/otel_service'

export default class HttpExceptionHandler extends HttpExceptionHandler {
  async handle(error: unknown, ctx: HttpContext) {
    OtelService.counters.exceptions.add(1, {
      operation: 'http_handler',
      error_type: (error as any).constructor?.name || 'Error',
      http_route: ctx.route?.pattern || ctx.request.url(),
    })
    return super.handle(error, ctx)
  }

  async report(error: unknown, ctx: HttpContext) {
    OtelService.counters.exceptions.add(1, {
      operation: 'http_report',
      error_type: (error as any).constructor?.name || 'Error',
      http_route: ctx.route?.pattern || ctx.request.url(),
    })
    return super.report(error, ctx)
  }
}
```

---

## Checklist

Before shipping observability to production:

- [ ] `otel.ts` is the first import in `bin/server.ts`
- [ ] `OTEL_EXPORTER_OTLP_ENDPOINT` is set in the environment
- [ ] `http_metric_middleware` registered as a **server** middleware (not router)
- [ ] `@adonisjs/otel/otel_middleware` registered as a **router** middleware, before `otel_enrich`
- [ ] `start/otel_metrics.ts` added to `preloads` in `adonisrc.ts`
- [ ] `/metrics` route is accessible to the Prometheus scraper
- [ ] Sampling ratio configured for production (`OTEL_TRACES_SAMPLER_ARG=0.1` for 10% is typical at scale)
- [ ] No SQL query contents logged in spans (PII and secrets can appear in queries)
