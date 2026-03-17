# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.2.0] - 2026-03-17

### Added

- Domain-subdirectory structure for `app/dtos/` in `SKILL.md` project tree (`payments/`, `invoices/`, `tasks/` subdirectories with example files)
- Expanded `### DTOs` section in `SKILL.md` with the three DTO type categories (input, response, nested), naming convention table, and import rules
- Expanded `references/code-examples.md` section 4 into a full DTO reference covering: domain folder structure, input DTO with controller-attached context, nested DTO (line items), response DTO with service builder example, complex real-world task completion DTO (derived from production `TaskWorkCompletePayload`), and `#dtos/*` subpath import configuration
- Replaced Decision 6 in `references/decisions.md` with an expanded decision covering `app/dtos/` domain organisation, the three DTO categories, naming conventions table, decision criteria for VineJS inference vs explicit DTO files, rejected alternatives (flat dtos/, inline service DTOs)

## [1.1.0] - 2026-03-17

### Added

- `references/observability.md` — production-grade OpenTelemetry and Prometheus metrics reference derived from `bbtz-3party-api`: hybrid tracing (`@adonisjs/otel` + OTLP) + metrics (`prom-client`) approach, `otel.ts` initialization order, `config/otel.ts` template, `OtelService` static class pattern (registry, counters, gauges, histograms, `withSpan()` helper), HTTP metrics middleware, span enrichment middleware, Lucid `db:query` event listener instrumentation, Prometheus `/metrics` endpoint, instrumentation decision guide, exception handler integration, and production checklist
- Decision 20 in `references/decisions.md` — hybrid tracing + Prometheus metrics rationale, rejected alternatives (OTEL metrics SDK, log-based metrics), and implementation summary
- `references/observability.md` added to `SKILL.md` reference routing table
- Skill trigger description extended to activate on observability, OTEL, Prometheus, tracing, metrics, instrumentation, and `/metrics` endpoint queries
- `start/otel_metrics.ts` and `config/otel.ts` / `otel.ts` added to project structure in `SKILL.md`
- Decision count updated from 18 to 19 (multi-tenancy) and 20 (observability) in `SKILL.md` reference note

## [1.0.0] - 2026-03-17

### Added

- Core `SKILL.md` with opinionated service-oriented architecture for AdonisJS v6 API applications
- Full project structure definition covering controllers, services, repositories, domain, actions, DTOs, state machines, events, listeners, validators, models, contracts, exceptions, policies, middleware, and providers
- Layer rules for controllers (HTTP adapters), services (orchestration), repositories (data access), domain objects (pure logic), DTOs, events & listeners, and authorization (Bouncer)
- Dependency injection strategy using AdonisJS v6 IoC container with `@inject()` decorator
- Singleton registration guidance for stateless services and repositories
- Abstract class pattern for swappable dependencies (e.g., payment gateways)
- Key conventions: one class per file, snake_case file names, `#` subpath imports, explicit return types, no `any` types
- Enum convention: PascalCase name, SCREAMING_SNAKE_CASE keys and values
- Response serialization guidance using Lucid `serializeAs` and plain DTOs
- Authorization pattern: Bouncer policies at controller boundary before service calls
- `references/decisions.md` — 18 architectural decisions with rationale, trade-offs, and GOOD/BAD examples
- `references/code-examples.md` — runnable examples for every architecture layer
- `references/performance.md` — memory safety rules, transaction patterns, singleton registration, benchmarks
- `references/state-machines.md` — lightweight state machine pattern with guarded transitions and testing
- `references/testing.md` — unit, integration, and functional test patterns using Japa
- Performance & memory safety rules: singleton registration, no I/O inside transactions, queued listener work, no cached model instances in singletons, lean repository defaults, boot-time-only event listener registration
- Action vs Service guidance: when to extract a service method into a dedicated action class
- `README.md` with installation instructions, compatibility table, and feature overview
- MIT `LICENSE`
