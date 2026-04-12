# 🏗️ Software Project Team

A team of specialized AI agents for building production-grade software projects. Built on Vertical Slice Architecture + Clean Architecture + Mediator pattern with .NET, PostgreSQL, RabbitMQ, Redis, Elasticsearch, and more.

## Installation

```bash
/team install https://github.com/mkurak/agent-workshop-software-project-team.git
```

> Requires [Agent Team Manager](https://github.com/mkurak/agent-workshop-agent-team-manager-skill) to be installed first.

## Agents

### 🧠 API Agent (`api-agent`)

The brain of every project. Owns all business logic across Domain, Application, Infrastructure, and API layers. Other services (Socket, Worker, Consumers) are bridges — they call the API, never contain logic.

**Core Principles:**
1. API is the brain — all logic lives here
2. Endpoints are bridges — parse HTTP, call Mediator, return response
3. Handlers are the single logic point — no try-catch, throw exceptions
4. Vertical Slice organization — feature = Command + Handler + Validator
5. Interface in Application, Implementation in Infrastructure
6. Fire-and-forget producer — email/log to RMQ, don't wait
7. Entity never returned as response — always map to DTO
8. Every Command gets a Validator — no exceptions

**Knowledge Base:**
- Logging Strategy (virtual debugging — log every step, no performance concern)
- Error Handling (exception hierarchy → HTTP status mapping)
- Naming Conventions (files, namespaces, endpoints)
- Architecture Layers (Domain, Application, Infrastructure, API)
- Workflows (new feature, query, migration, endpoint)
- RMQ Topology (consumer declares own topology, idempotent)
- Pagination (cursor-based infinite scroll, optional count)
- Soft Delete (ISoftDeletable + global query filter)
- Audit Trail (two-layer: IAuditableEntity + AuditLog table)
- Dynamic Settings (DB + Redis, no redeploy for config changes)
- Caching Strategy (pipeline behavior + Redis, TTL from settings)
- File Storage (MinIO/S3 compatible, entity-based paths)
- Notification Pattern (HTTP instant + RMQ async, decision table)
- Idempotency (X-Idempotency-Key + Redis SETNX)
- Concurrency Handling (optimistic locking + RowVersion)

*More agents coming soon: Socket Agent, Worker Agent, Flutter Agent, React Agent, Infrastructure Agent*

## Recommended Companion Skills

These skills work great with this team:

- [Brainstorm Skill](https://github.com/mkurak/agent-workshop-brainstorm-skill) — structured brainstorming with persistent state
- [Rule Skill](https://github.com/mkurak/agent-workshop-rule-skill) — coding rule management with guided wizard

## Architecture Overview

```
┌─────────────────────────────────────────┐
│                  API                     │  ← Brain (all logic)
│  Domain → Application → Infrastructure  │
│  Minimal API Endpoints (bridges)        │
└──────────┬──────────┬───────────────────┘
           │          │
    ┌──────┴──┐  ┌────┴────┐
    │ Socket  │  │ Worker  │  ← Bridges (HTTP to API)
    │ SignalR │  │  Cronos │
    └─────────┘  └─────────┘
           │
    ┌──────┴──────────────────┐
    │      RabbitMQ           │  ← Async messaging
    ├─────────┬───────────────┤
    │LogIngest│  MailSender   │  ← Consumers
    │→Elastic │  →SMTP        │
    └─────────┴───────────────┘
```

## Tech Stack

| Layer | Technology |
|-------|-----------|
| API | .NET 9, Minimal API, Mediator (source gen) |
| Database | PostgreSQL 17, EF Core 9 |
| Messaging | RabbitMQ 3 (fanout exchanges) |
| Cache | Redis 7 |
| Logging | Serilog → RMQ → Elasticsearch 8 + Kibana |
| Email | MailSender consumer → Mailpit (dev) |
| Auth | JWT HS256, BCrypt, X-Internal-Token |
| Storage | MinIO (dev) / S3 (prod) |
| Infrastructure | Docker Compose, dotnet watch |

## Key Rules

- **Everything runs in Docker** — no local SDK installations
- **Minimal API only** — no Controllers
- **martinothamar/Mediator** — source generator, not MediatR
- **LogIngest logs to console only** — prevents infinite RMQ loop
- **X-Internal-Token** for system-to-system auth

## License

MIT
