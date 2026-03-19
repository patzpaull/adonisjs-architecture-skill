# AdonisJS v6 Architecture Skill

An opinionated architecture skill for Claude Code that guides AdonisJS v6 API application development. Covers service-oriented architecture, repository pattern, DI strategy, state machines, performance rules, and testing patterns.

## Compatibility

- AdonisJS v6 (Node.js 20+)
- TypeScript 5+
- VineJS (validation)
- Lucid ORM (data access)

## Installation

### Option 1 — Claude Code skills directory (recommended)

```bash
# Copy into your user-level Claude skills directory
cp -r adonisjs-architecture-skill ~/.claude/skills/adonisjs-architecture

# Or clone directly
git clone https://github.com/omakei/adonisjs-architecture-skill ~/.claude/skills/adonisjs-architecture
```

### Option 2 — Project-local skill

```bash
# Add to your project's .claude/skills directory
mkdir -p .claude/skills
cp -r adonisjs-architecture-skill .claude/skills/adonisjs-architecture
```

## What It Covers

| File | Contents |
|------|----------|
| `SKILL.md` | Core architecture rules, layer responsibilities, DI strategy, conventions, DTO organisation |
| `code-examples.md` | Runnable examples for every layer (controllers, services, repos, DTOs, events, state machines, config) |
| `decisions.md` | 20 architectural decisions with rationale, trade-offs, and GOOD/BAD examples |
| `performance.md` | Memory safety rules, transaction patterns, singleton registration, benchmarks |
| `state-machines.md` | Lightweight state machine pattern with guarded transitions and testing |
| `testing.md` | Unit, integration, and functional test patterns using Japa |
| `observability.md` | OpenTelemetry + Prometheus hybrid tracing and metrics (OtelService, HTTP metrics middleware, DB query instrumentation, `/metrics` endpoint) |

## When the Skill Activates

Claude will use this skill automatically when you're working on AdonisJS v6 code and:

- Asking about architecture, project structure, or pattern selection
- Creating controllers, services, repositories, actions, DTOs, validators, events, or listeners
- Making structural or organizational decisions in an AdonisJS v6 codebase
- Asking about separation of concerns, DI, state machines, or domain-driven design
- Working with observability, OpenTelemetry, Prometheus metrics, tracing, or instrumentation

## Key Principles

1. **Controllers are HTTP adapters** — zero business logic
2. **Services orchestrate** — coordinate repos, emit events, enforce rules
3. **Repositories own data access** — all Lucid queries live here
4. **Domain objects are pure** — no DB, no HTTP awareness
5. **Events decouple side effects** — downstream consequences go via listeners
6. **DTOs cross layer boundaries** — never pass raw `request.all()`; organise by domain in `app/dtos/`
