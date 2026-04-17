---
name: create-new-project
description: "Scaffold a complete production-ready project with a single command. Asks key questions (project name, SaaS?, modules), then creates the full stack: .NET API (Vertical Slice + Clean Arch), Docker Compose, PostgreSQL, RabbitMQ, Redis, Elasticsearch, logging pipeline, email pipeline, auth, and optionally Flutter/React apps."
argument-hint: "[project-name]"
---

# /create-new-project Skill

## Purpose

Creates a complete, production-ready project skeleton from scratch. In 5 minutes, you get the same mature infrastructure that would take weeks to build manually. Every project follows the same architecture (Vertical Slice + Clean Architecture + Mediator) and includes battle-tested patterns for logging, email, auth, caching, and more.

## Flow

### Phase 1 — Gather Information

Ask the user the following questions using AskUserQuestion. Each question should have clear options. If a project name is provided as an argument, skip question 1.

#### Question 1: Project Name
```
What is the project name?
```
Free text. Must be PascalCase (e.g., WalkingForMe, ProductGlitz, MarketPlace).
Derive abbreviation for Docker container prefix (e.g., WalkingForMe → wfm, ProductGlitz → pg).

#### Question 2: Applications
```
Which applications does this project need?
```
Options (multiSelect: true):
- Flutter mobile app (iOS + Android)
- React web app (admin panel / dashboard)
- React public website (SSR with Next.js)

At least one must be selected.

#### Question 3: SaaS
```
Is this a SaaS project (multi-tenant)?
```
Options:
- No — single tenant, simple User entity only
- Yes — User + Profile + Tenant model

#### Question 3a (if SaaS = Yes): Tenant Entity Name
```
What is the tenant entity name?
```
Options:
- Dealer
- Company
- Organization
- Seller
- Other (custom name)

#### Question 3b (if SaaS = Yes): Multi-Profile
```
Can a user belong to multiple tenants?
```
Options:
- Yes — user can have profiles in different tenants (e.g., customer at multiple dealers)
- No — one user, one tenant

#### Question 3c (if SaaS = Yes): Embed Auth
```
Do tenants need to embed your app (iframe/SDK auto-login)?
```
Options:
- Yes — embed auth flow (tenant-scoped auto-login, auto-create user/profile)
- No — standard login only

#### Question 4: Additional Modules
```
Which additional modules should be included?
```
Options (multiSelect: true):
- File Storage (MinIO/S3)
- Audit Trail (change history tracking)
- Soft Delete (default: included, ask only to confirm)

Note: Logging pipeline, email pipeline, Redis, RabbitMQ, JWT auth, health checks are ALWAYS included — they're core infrastructure, not optional.

#### Question 5: Port Offset
```
Do you have other projects running on this machine?
```
Options:
- No — use default ports (3000, 5432, 6379, etc.)
- Yes — apply port offset (+10000) to avoid conflicts

#### Question 6: Team (if no team installed)
Check if `.claude/.team-installs.json` exists. If not, ask:

```
Which team should be installed for this project?
```

**Auto-detection:** Check `~/.claude/repos/agentteamland/` for directories containing `team.json` with agents. List them as options.

If no cached teams found, ask for a git repo URL:
```
Enter the team repo URL (e.g., https://github.com/agentteamland/software-project-team.git)
```

If `--team <url>` was passed as argument, use that directly without asking.

**Install the team:** Run `/team install <url>` which clones to `~/.claude/repos/agentteamland/` and symlinks agents/skills/rules into the project's `.claude/`.

### Phase 2 — Create Project Structure

Based on answers, create the complete project. Use the software-project-team agents' knowledge as reference.

#### 2.1 Root Files

Create in the specified project directory:

```
{ProjectName}/
├── {ProjectName}.sln                    ← Solution file at root
├── docker-compose.yml                   ← All services
├── .env                                 ← Environment config (gitignored)
├── .env.example                         ← Template (committed)
├── .gitignore                           ← .NET + Docker + IDE
├── .dockerignore                        ← Docker build exclusions
├── CLAUDE.md                            ← Project summary (filled with tech stack info)
└── README.md                            ← Project description
```

#### 2.2 .NET Projects (Always Created)

```
src/
├── {ProjectName}.Domain/                ← Pure C#, BaseEntity, IAuditableEntity, IAggregateRoot, ISoftDeletable
├── {ProjectName}.Application/           ← Mediator, FluentValidation, Behaviors, Interfaces, Exceptions
├── {ProjectName}.Infrastructure/        ← EF Core, JWT, BCrypt, Redis, RMQ, Email sender
├── {ProjectName}.Logging/               ← Shared: RmqLogSink, LogChannelQueue, LogPublisherHostedService
├── {ProjectName}.Api/                   ← Composition root, Minimal API endpoints, Dockerfile.dev
├── {ProjectName}.Socket/                ← SignalR bridge, IApiClient, Dockerfile.dev
├── {ProjectName}.Worker/                ← BackgroundService + Cronos, Dockerfile.dev
├── {ProjectName}.LogIngest/             ← Log consumer (RMQ → Elasticsearch), Dockerfile.dev
└── {ProjectName}.MailSender/            ← Email consumer (RMQ → SMTP), Dockerfile.dev
```

**Key patterns to include:**
- martinothamar/Mediator (source generator, NOT MediatR)
- Pipeline behaviors: Logging → UnhandledException → Validation → Performance → Idempotency → Caching
- IApplicationDbContext interface in Application
- ApplicationDbContext in Infrastructure with AuditableEntityInterceptor
- JWT auth (HS256, 15min access, 30day refresh in Redis)
- X-Internal-Token for system-to-system (Worker→API, API→Socket)
- RMQ connection + topology (logs.fanout, emails.fanout)
- Health endpoints (/api/health/ping)
- Global exception handler (NotFoundException→404, ValidationException→422, ForbiddenException→403)
- Swagger with Bearer token support
- Auto-migrate in Development
- seed.sql file with admin test user

**If SaaS = Yes, additionally include:**
- User entity (email, passwordHash, firstName, lastName)
- Profile entity (userId, tenantId, role, isActive)
- Tenant entity (name, code, status) with configured tenant name
- ITenantEntity interface + global query filter + TenantInterceptor
- ITenantContext (from JWT claims)
- Embed auth endpoint (if selected)
- Auth endpoints: login, register, refresh, select-profile, logout

**If SaaS = No:**
- User entity only (simple, no Profile/Tenant)
- Auth endpoints: login, register, refresh, logout

**If Soft Delete = Yes (default):**
- ISoftDeletable interface (IsDeleted, DeletedAt, DeletedBy)
- Global query filter for soft delete
- SaveChanges interceptor to convert Delete → Modified

**If Audit Trail = Yes (default):**
- AuditLog entity
- Change tracking in SaveChanges (entity, field, old value, new value, who, when)

**If File Storage = Yes:**
- IStorageService in Application
- S3StorageService in Infrastructure
- MinIO service in Docker Compose

#### 2.3 Docker Compose Services

**Always included (core infrastructure):**
- db (postgres:17-alpine) — healthcheck, persistent volume
- rabbitmq (rabbitmq:3-management-alpine) — healthcheck
- redis (redis:7-alpine) — healthcheck
- elasticsearch (elasticsearch:8.15.0) — single-node, security disabled, JVM 512m
- kibana (kibana:8.15.0) — connected to elasticsearch
- mailpit (axllent/mailpit) — dev SMTP + Web UI
- db-ui (adminer) — DB management
- redis-ui (rediscommander/redis-commander)
- api — .NET SDK, dotnet watch, depends on db+rabbitmq+redis healthy
- socket — .NET SDK, dotnet watch
- worker — .NET SDK, dotnet watch
- log-ingest — .NET SDK, dotnet watch, depends on rabbitmq+elasticsearch healthy
- mail-sender — .NET SDK, dotnet watch, depends on rabbitmq+redis healthy

**If File Storage = Yes:**
- minio (minio/minio) — S3-compatible, console UI

**If Flutter = Yes:**
- (Flutter runs natively, not in Docker — note in README)

**If React = Yes:**
- app (node, vite dev server or next dev)

**Dev Dockerfile pattern for all .NET services:**
- SDK:9.0 image
- Copy all .csproj files for restore (layer cache)
- dotnet restore
- ENTRYPOINT dotnet watch run
- Source mounted as volume (not copied)
- Shadow volumes for bin/obj
- Per-service NuGet cache volume

#### 2.4 Flutter App (if selected)

Create Flutter project with the framework's standard structure:
```
flutter/
├── lib/
│   ├── main.dart
│   ├── app/ (MaterialApp, router, theme)
│   ├── core/ (responsive, providers, extensions, constants)
│   ├── data/ (models, repositories, sources/mock + remote)
│   ├── features/ (feature-first screens)
│   └── l10n/ (ARB files)
├── pubspec.yaml (riverpod, go_router, dio, i18n packages)
└── analysis_options.yaml
```

#### 2.5 React App (if selected)

Create React + TypeScript + Vite project (or Next.js if SSR selected):
```
app/
├── src/
│   ├── main.tsx
│   ├── components/ui/ (shared components)
│   ├── features/ (feature-first pages)
│   ├── hooks/ (custom hooks)
│   ├── lib/ (api client, auth, utils)
│   ├── stores/ (Zustand stores)
│   └── locales/ (i18n JSON files)
├── package.json
├── tsconfig.json
├── vite.config.ts (or next.config.ts)
└── tailwind.config.ts
```

#### 2.6 Project-Level .claude Configuration

```
.claude/
├── settings.json                        ← PreToolUse hook for coding rules injection
├── hooks/
│   └── inject-coding-rules.sh           ← Auto-inject app-specific rules on edit
├── rules/
│   └── coding-common.md                 ← Cross-cutting project rules (starts empty)
├── docs/
│   ├── coding-standards/
│   │   ├── api.md                       ← API-specific rules (starts empty)
│   │   ├── flutter.md                   ← (if Flutter selected)
│   │   └── react.md                     ← (if React selected)
│   ├── coding-rules-system.md
│   └── documentation-workflow.md
├── brain-storms/                        ← Empty, ready for brainstorms
├── backlog.md                           ← Empty, ready for deferred items
├── wiki/                                ← Project knowledge base (living)
│   └── index.md                         ← Auto-maintained table of contents
├── agent-memory/                        ← Per-agent learning history
└── journal/                             ← Inter-agent communication
```

#### 2.7 Code Intelligence (.mcp.json)

Create `.mcp.json` at project root for codebase-memory-mcp integration:

```json
{
  "mcpServers": {
    "codebase-memory": {
      "command": "codebase-memory",
      "args": ["--project-root", "."]
    }
  }
}
```

This provides knowledge graph indexing of the entire codebase — function relationships, call chains, dependency mapping. Agents can query "who calls this?" and get precise answers instead of scanning files.

### Phase 3 — Build and Start

1. **Build .NET solution** inside Docker:
   ```bash
   docker compose run --rm api dotnet build /src/{ProjectName}.sln
   ```

2. **Start all services:**
   ```bash
   docker compose up -d
   ```

3. **Wait 30-60 seconds** for services to stabilize (boot, migrate, connect).

4. **Index codebase for code intelligence** (if codebase-memory installed):
   ```bash
   codebase-memory index --path .
   ```

### Phase 4 — System Verification (MANDATORY skill invocation)

**This is a HARD requirement.** You MUST call the `Skill` tool with `verify-system` here. Do NOT run the verification commands manually as a separate flow — the Skill tool call is what makes this step auditable for the user. The user explicitly relies on seeing the skill's invocation as proof that verification ran.

```
Skill(skill="verify-system")
```

The skill returns markdown instructions for a 4-level end-to-end check:
- Level 1: All containers running
- Level 2: All ports accessible
- Level 3: All applications healthy (meaningful responses)
- Level 4: All pipelines working (logging, email, auth, socket, worker, redis, storage)

After invoking the skill:
1. Execute every check the skill describes (the skill is a script — you are the runtime).
2. Collect results level-by-level.
3. **Show the user the final verification report block** (the boxed `╔══...══╗` summary the skill defines) before moving on.

**The project is NOT ready until the report shows ALL PASS.** If any test fails:
- Diagnose the root cause (don't paper over it)
- Fix it
- Re-invoke the skill
- Show the new report

Do not proceed to Phase 5 until verification is green and the user has seen the report.

### Phase 5 — Git Initialize

```bash
cd {project-directory}
git init   # only if not already a git repo
git add -A
git commit -m "feat: initial project setup with full infrastructure"
```

If the directory was already a git repo (e.g. user pre-cloned an empty remote), skip `git init` and just commit on top of the existing history.

## Important Rules

1. **Everything runs in Docker.** The skill NEVER installs SDKs locally (except noting Flutter needs local install).
2. **martinothamar/Mediator, NOT MediatR.** Source generator based.
3. **Minimal API, NO Controllers.** Every project uses endpoint groups.
4. **LogIngest logs to console ONLY.** Prevents infinite RMQ loop.
5. **Consumers declare their own topology.** Don't assume API created it first.
6. **seed.sql for development data.** Idempotent INSERTs, fixed UUIDs.
7. **Port offset if multi-project.** All ports configurable via .env.
8. **CLAUDE.md is filled with actual project info** — tech stack, port list, development commands.

## What Is NOT Included (by design)

- Domain entities (project-specific — added during development)
- Business logic (added via API Agent as features are built)
- Production deployment (CI/CD templates are in Infra Agent's knowledge)
- Flutter/React screens (added via Flutter/React Agent as features are built)

The skill creates the **skeleton** — the team of agents fills it with life.
