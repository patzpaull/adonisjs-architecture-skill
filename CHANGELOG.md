# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
