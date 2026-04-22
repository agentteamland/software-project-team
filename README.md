# 🏗️ Software Project Team

A team of 13 specialized AI agents for building production-grade software projects. Built on Vertical Slice Architecture + Clean Architecture + Mediator pattern with .NET, PostgreSQL, RabbitMQ, Redis, Elasticsearch, and more.

## Installation

```bash
/team install https://github.com/agentteamland/software-project-team.git
```

> Requires [Agent Team Manager](https://github.com/agentteamland/team-manager) to be installed first.
> Dependency: [Core](https://github.com/agentteamland/core) (auto-installed via team.json).

## Agents (13)

### 🧠 API Agent (`api-agent`) — 18 children

The brain of every project. All business logic lives in Domain + Application layers. Other services are bridges.

**Topics:** Architecture layers, audit trail, caching (pipeline behavior + Redis), concurrency (optimistic locking), dynamic settings (DB + Redis), error handling, file storage (MinIO/S3), idempotency, logging (virtual debug), multi-tenancy (User + Profile + Tenant), naming conventions, notification (HTTP + RMQ), pagination (cursor-based), RMQ topology, seed data, soft delete, user-facing strings (API↔UI i18n contract), workflows.

---

### ⚡ Socket Agent (`socket-agent`) — 17 children

SignalR WebSocket bridge. Real-time communication between clients and API. No business logic.

**Topics:** Hub design, auth (JWT query string), internal endpoints, API client pattern, group management, connection tracking, reconnection handling, CORS, rate limiting, event conventions, data boundary, multi-device, presence/typing, heartbeat, error handling, event versioning, hub method blueprint.

---

### ⏰ Worker Agent (`worker-agent`) — 6 children

Scheduled background jobs with Cronos. Calls API via HTTP. No business logic.

**Topics:** Job blueprint, cron scheduling, API client pattern, distributed locking, graceful shutdown, health monitoring.

---

### 📱 Flutter Agent (`flutter-agent`) — 22 children

Mobile/tablet specialist. iOS + Android from a single Dart codebase. UI bridge — no business logic.

**Topics:** Screen blueprint, state management (Riverpod), routing (go_router), API integration (Dio), responsive design, i18n, theme system, component design, form handling, offline-first, push notifications, auth flow, image handling, list patterns, navigation patterns, testing, deep linking, permissions, app lifecycle, error/loading states, WebView, Claude Design handoff (React→Flutter translation rules).

---

### 🌐 React Agent (`react-agent`) — 24 children

Web UI specialist. TypeScript + Vite (or Next.js for SSR). Admin panels, dashboards, web apps. No business logic.

**Topics:** Component blueprint, component design, state management (React Query + Zustand), routing (React Router), API integration (Axios), form handling (React Hook Form + Zod), styling (Tailwind CSS), auth flow, table patterns (TanStack Table), list patterns, modal/dialog, error/loading states, layout patterns, i18n, file upload, testing (Vitest + RTL + MSW), WebSocket (SignalR), permission guard, chart/dashboard, keyboard shortcuts, print/export, SEO/meta, white-label, Claude Design handoff (bundle adaptation rules).

---

### 🐳 Infra Agent (`infra-agent`) — 16 children

Docker, Compose, CI/CD, environment management. Everything runs in containers.

**Topics:** Compose blueprint, compose architecture, Dockerfile patterns, volume strategy, env management, health checks, port management, hot reload, logging infra, MinIO setup, CI/CD (GitHub Actions), production Dockerfile, backup/restore, multi-project coexistence, resource limits, SSL/TLS local.

---

### 🗄️ Database Agent (`database-agent`) — 9 children

PostgreSQL + EF Core specialist. Schema design, migrations, indexing, query optimization.

**Topics:** Schema design blueprint, migration management, index strategy, query optimization, naming conventions, relationships, data types, constraint patterns, monitoring.

---

### 🔴 Redis Agent (`redis-agent`) — 8 children

Cache, session, distributed lock, rate limiting specialist. Ephemeral data only.

**Topics:** Cache blueprint, key naming, data structures, TTL strategy, pub/sub, memory management, connection management, distributed patterns.

---

### 🐇 RMQ Agent (`rmq-agent`) — 8 children

RabbitMQ messaging specialist. Exchange/queue topology, producer/consumer patterns, reliability.

**Topics:** Consumer blueprint, topology design, exchange patterns, retry/DLX, idempotency, message serialization, connection management, monitoring.

---

### 🔍 Code Reviewer (`code-reviewer`) — 8 children

Reviews code changes for quality, security, performance, and convention compliance.

**Topics:** Review blueprint, SOLID check, naming review, security scan, performance check, error handling review, logging review, API review.

---

### 📋 Project Reviewer (`project-reviewer`) — 7 children

Reviews overall project health, architecture, dependencies, and technical debt.

**Topics:** Review blueprint, architecture review, dependency audit, tech debt, documentation check, configuration review, scalability assessment.

### 🎨 Design System Agent (`design-system-agent`) — 11 children

Visual foundation of every project. Colors, typography, spacing, icons, component tokens, accessibility. One design system, all platforms.

**Topics:** Design blueprint, color system, typography, spacing system, icon strategy, elevation/shadow, component tokens, accessibility (WCAG 2.1 AA), dark mode, animation/motion, tokens for Claude Design (codebase-readable token authoring).

---

### 🧑‍🎨 UX Agent (`ux-agent`) — 11 children

User experience specialist. Screen flows, navigation patterns, form design, data presentation, feedback systems.

**Topics:** Screen flow blueprint, navigation UX, form UX, data presentation, feedback patterns, onboarding UX, mobile vs tablet vs web, notification UX, error UX, accessibility UX, Claude Design prompts (high-yield prompt authoring).

## Skills (3)

### `/create-new-project [Name]`
5-phase scaffolder for a complete production-ready project from scratch. Asks key questions (project name, SaaS, modules, port offset, Claude Design opt-in), then creates the full stack: .NET API + Docker Compose + Postgres + RabbitMQ + Redis + Elasticsearch + logging pipeline + email pipeline + auth + optional Flutter/React apps. Includes `/verify-system` invocation as a mandatory final step.

### `/verify-system`
4-level end-to-end health check for this team's stack: containers running, ports accessible, applications healthy, pipelines working (logging / email / auth / socket / worker / redis / storage / audit). Returns a boxed pass/fail report.

### `/design-screen start "<description>" | done <handoff-url>`
Orchestrates the [Claude Design](https://claude.ai/design) loop. `start` derives a slug, gathers requirements via AskUserQuestion, builds a token-aware prompt, writes `intent.md` (status: pending-design), hands the prompt to the user. `done` finds the matching pending design (asks if multiple), fetches the bundle, briefs flutter-agent or react-agent for integration. State persists in `.claude/design/{slug}/intent.md` across sessions (brainstorm pattern). Requires Claude Pro/Max/Team/Enterprise; opt-in during project scaffolding.

## Settled Architectural Patterns

These are systemic capabilities the team enforces across all generated code. Each is documented in detail by the relevant agent's children files.

### UI-only i18n (1.0.3+)
Backend (API / Socket / Worker / LogIngest) is English-only. UI apps (Flutter, React admin, React public) localize via a `messageKey + placeholders + fallback` envelope inside `ProblemDetails.extensions` (errors), notification payloads (socket events), and EmailJob / PushNotificationJob (with `locale` field). MailSender + push dispatcher hold per-locale templates and render server-side. Enum values render via `{entity}_{field}_{value}` convention. Canonical reference: [`api-agent/children/user-facing-strings.md`](agents/api-agent/children/user-facing-strings.md).

### Claude Design integration (1.1.0+)
Optional visual-prototype phase between text-only flow specs (ux-agent) and code (flutter-agent / react-agent). The `/design-screen` skill orchestrates the round-trip; flutter-agent + react-agent gain handoff playbooks ([flutter](agents/flutter-agent/children/claude-design-handoff.md) / [react](agents/react-agent/children/claude-design-handoff.md)) for the React+HTML bundle; design-system-agent gains [token-authoring guidance](agents/design-system-agent/children/tokens-for-claude-design.md) for codebase readability; ux-agent gains [prompt-authoring guidance](agents/ux-agent/children/claude-design-prompts.md). Pilot evidence (login screen, 2026-04-19): 1 iteration, 4/5 fidelity, 28 min total, 0 hex literals in resulting Dart.

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
| Mobile | Flutter (Riverpod, go_router) |
| Web UI | React + TypeScript + Vite (or Next.js) |
| Infrastructure | Docker Compose, dotnet watch |

## Recommended Companion Skills

- [Brainstorm Skill](https://github.com/agentteamland/brainstorm) — structured brainstorming with persistent state
- [Rule Skill](https://github.com/agentteamland/rule) — coding rule management with guided wizard
- [Core](https://github.com/agentteamland/core) — memory system, journal, save-learnings (auto-installed as dependency)

## Key Rules

- **Everything runs in Docker** — no local SDK installations (except Flutter)
- **Minimal API only** — no Controllers
- **martinothamar/Mediator** — source generator, not MediatR
- **LogIngest logs to console only** — prevents infinite RMQ loop
- **X-Internal-Token** for system-to-system auth
- **API is the brain** — all logic lives there, everything else is a bridge

## License

MIT
