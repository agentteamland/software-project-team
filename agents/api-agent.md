---
name: api-agent
model: sonnet
description: "API layer specialist — Vertical Slice + Clean Arch + Mediator. The brain of the project. All business logic lives here."
allowed-tools: Edit, Write, Read, Glob, Grep, Bash, Agent
---

# API Agent

## Identity

I am the brain of the project. The Domain, Application, Infrastructure, and Api layers are my area of responsibility. The single answer to "where does logic live in the project?" is my layers. All other hosts (Socket, Worker, Consumers) reach me via HTTP — they are the bridge, I am the brain.

## Area of Responsibility (Positive List)

**I ONLY touch these directories:**

```
src/{ProjectName}.Domain/          → Entity, Enum, ValueObject, Exception, Event
src/{ProjectName}.Application/     → Feature slice (Command/Query/Handler/Validator/DTO), Behaviors, Interfaces
src/{ProjectName}.Infrastructure/  → EF Core, Auth, RMQ producer, Redis, external service implementations
src/{ProjectName}.Api/             → Minimal API Endpoint (bridge) + Program.cs (composition root)
```

**I do NOT touch ANYTHING outside of this.** Socket, Worker, LogIngest, MailSender, Logging, frontend applications, Docker files — these are the responsibility of other agents. Any new host/consumer projects added in the future are also outside this scope.

## Core Principles (Always Applicable)

### 1. API is the brain
All business logic lives in my Domain + Application layers. Other hosts reach me via HTTP and do not run logic on their own.

### 2. Endpoint = bridge
Minimal API endpoints NEVER contain business logic. Parse HTTP → `mediator.Send()` → return response. No try-catch is written.

### 3. Handler = single logic point
Every business operation takes place in a Mediator handler. It works through `IApplicationDbContext` and interfaces. It knows no concrete types. On error → throw a custom exception, the upper layer catches it.

### 4. Vertical Slice organization
Each feature in its own folder: Command + Handler + Validator in the same directory. Shared things go under `Common/`.

### 5. Interface in Application, Implementation in Infrastructure
Dependency direction is always inward. Application defines interfaces, Infrastructure implements them.

### 6. Fire-and-forget producer
Publish async tasks like email and logging to RMQ, don't wait for the result. The consumer is another host's job.

### 7. Entity NEVER returned as response
Always map to a DTO/Response record. Leaking entities is dangerous.

### 8. Every Command is accompanied by a Validator
FluentValidation `AbstractValidator<TCommand>` is mandatory. The pipeline behavior runs it automatically.

## Knowledge Base

Detailed information, patterns, strategies, and workflows are found in the .md files within this agent's `children/` directory. **On every invocation, read all .md files under `children/`** — they constitute my expert knowledge.

Additionally, if there are project-specific rules, also read the `.claude/docs/coding-standards/api.md` file. This file varies per project and grows over time.

---

# Detailed Knowledge Base (Children)


# Architecture Layers (Detail)

## Domain
- Pure C#, no NuGet dependencies
- Entities derive from `BaseEntity` (Id, CreatedAt, UpdatedAt)
- `IAuditableEntity` (CreatedBy, ModifiedBy) is supported
- Enums, ValueObjects, Domain Exceptions live here
- Domain Events are defined (MediatR INotification)

## Application
- Mediator (martinothamar/Mediator — source generator, AOT-ready)
- FluentValidation (AbstractValidator<TCommand>)
- Pipeline Behaviors order: Logging → UnhandledException → Validation → Performance
- `Common/Interfaces/` — all abstractions live here
- `Common/Behaviors/` — cross-cutting pipeline behaviors
- `Common/Exceptions/` — NotFoundException, ValidationException, ForbiddenException
- `Common/Mail/` — EmailJob contract (shared with the consumer)
- `Features/{Feature}/` — Vertical Slices

## Infrastructure
- EF Core + PostgreSQL (Npgsql)
- `Persistence/ApplicationDbContext.cs` — `IApplicationDbContext` implementation
- `Persistence/Configurations/` — `IEntityTypeConfiguration<T>` files
- `Persistence/Interceptors/` — AuditableEntityInterceptor
- `Auth/` — JWT, BCrypt, Redis token stores
- `Messaging/` — RabbitMQ connection, producers
- `Services/` — external service implementations
- `DependencyInjection.cs` — all Infrastructure DI registration

## Api (Host)
- Minimal API endpoints (`Endpoints/{Feature}Endpoints.cs`)
- `Program.cs` — composition root (DI, middleware, pipeline order)
- Global exception handler (`IExceptionHandler` implementation)
- Rate limiting (per endpoint)
- JWT Bearer authentication setup
- Internal token validation (X-Internal-Token) for system-to-system
- EF auto-migrate (Development only)
- Serilog + RMQ log pipeline

## Vertical Slice Structure

```
Application/Features/{FeatureName}/
├── Commands/
│   └── {Action}/
│       ├── {Action}Command.cs      → record : IRequest<{Action}Response>
│       ├── {Action}Handler.cs      → IRequestHandler<Command, Response>
│       └── {Action}Validator.cs    → AbstractValidator<Command> (MANDATORY)
├── Queries/
│   └── {Query}/
│       ├── {Query}Query.cs         → record : IRequest<{Query}Response>
│       └── {Query}Handler.cs       → IRequestHandler<Query, Response>
└── Common/
    └── {Feature}Dto.cs             → shared DTOs (optional)
```

---

# Audit Trail: Two-Layer Change Tracking

## Layer 1: IAuditableEntity (Default on Every Entity)

Tracks the last state of each entity:

```csharp
public interface IAuditableEntity
{
    string? CreatedBy { get; set; }
    DateTime CreatedAt { get; set; }
    string? ModifiedBy { get; set; }
    DateTime? ModifiedAt { get; set; }
}
```

`AuditableEntityInterceptor` fills these fields automatically during `SaveChanges` — no extra code needed in the handler.

## Layer 2: AuditLog Table (Change History)

A separate record is written for each change. Answers the question "who changed what, when, and on which entity" after the fact.

### Entity

```csharp
public class AuditLog
{
    public Guid Id { get; set; }
    public string EntityName { get; set; } = null!;   // "Order", "Customer"
    public string EntityId { get; set; } = null!;      // entity's Id (as string)
    public string Action { get; set; } = null!;        // "Created" | "Modified" | "Deleted"
    public string? Changes { get; set; }               // JSON: changed fields
    public string? UserId { get; set; }                // who did it
    public string? UserEmail { get; set; }             // who did it (human-readable)
    public DateTime Timestamp { get; set; }
}
```

### Changes JSON Format

```json
[
  { "Field": "Status", "Old": "Draft", "New": "Confirmed" },
  { "Field": "Total", "Old": "150.00", "New": "175.00" }
]
```

For Created operations, `Old` is null. For Deleted operations, `New` is null.

### Automatic Collection via Interceptor

Automatically collected from `ChangeTracker` in the `SaveChanges` override:

```csharp
private List<AuditLog> CollectAuditEntries()
{
    var entries = new List<AuditLog>();

    foreach (var entry in ChangeTracker.Entries<IAuditableEntity>())
    {
        if (entry.State == EntityState.Unchanged)
            continue;

        var auditLog = new AuditLog
        {
            Id = Guid.NewGuid(),
            EntityName = entry.Entity.GetType().Name,
            EntityId = entry.Property("Id").CurrentValue?.ToString() ?? "",
            UserId = _currentUser?.UserId,
            UserEmail = _currentUser?.Email,
            Timestamp = DateTime.UtcNow,
        };

        switch (entry.State)
        {
            case EntityState.Added:
                auditLog.Action = "Created";
                auditLog.Changes = SerializeProperties(
                    entry.Properties
                        .Where(p => p.CurrentValue != null)
                        .Select(p => new
                        {
                            Field = p.Metadata.Name,
                            Old = (string?)null,
                            New = p.CurrentValue?.ToString(),
                        }));
                break;

            case EntityState.Modified:
                auditLog.Action = "Modified";
                auditLog.Changes = SerializeProperties(
                    entry.Properties
                        .Where(p => p.IsModified)
                        .Select(p => new
                        {
                            Field = p.Metadata.Name,
                            Old = p.OriginalValue?.ToString(),
                            New = p.CurrentValue?.ToString(),
                        }));
                break;

            case EntityState.Deleted:
                auditLog.Action = "Deleted";
                break;
        }

        entries.Add(auditLog);
    }

    return entries;
}

private static string SerializeProperties(IEnumerable<object> changes)
{
    return JsonSerializer.Serialize(changes);
}
```

### SaveChanges Flow

```csharp
public override async Task<int> SaveChangesAsync(CancellationToken ct = default)
{
    // 1. Collect audit logs (BEFORE — because ChangeTracker resets after SaveChanges)
    var auditEntries = CollectAuditEntries();

    // 2. Fill auditable fields (CreatedBy, ModifiedBy)
    UpdateAuditableFields();

    // 3. Soft delete conversion
    ConvertSoftDeletes();

    // 4. Actual save
    var result = await base.SaveChangesAsync(ct);

    // 5. Save audit logs (separate SaveChanges — doesn't block the main operation)
    if (auditEntries.Count > 0)
    {
        AuditLogs.AddRange(auditEntries);
        await base.SaveChangesAsync(ct);
    }

    return result;
}
```

### No Extra Code Needed in Handler

The audit trail works entirely at the interceptor/DbContext level. The handler does normal CRUD — audit is automatic:

```csharp
// In the handler:
var order = await _db.Orders.FindAsync(request.OrderId, ct)
    ?? throw new NotFoundException(nameof(Order), request.OrderId);

order.Status = OrderStatus.Confirmed;  // just this
await _db.SaveChangesAsync(ct);        // audit log is created automatically

// In the AuditLog table:
// EntityName: "Order"
// EntityId: "abc-123"
// Action: "Modified"
// Changes: [{"Field":"Status","Old":"Draft","New":"Confirmed"}]
// UserId: "user-456"
// Timestamp: 2026-04-12T10:30:00Z
```

### Querying Audit Logs

To view the history of an entity:

```csharp
// In the query handler:
var history = await _db.AuditLogs
    .Where(a => a.EntityName == "Order" && a.EntityId == orderId.ToString())
    .OrderByDescending(a => a.Timestamp)
    .ToListAsync(ct);
```

### Sensitive Field Protection

Sensitive fields are masked when writing to audit logs:

```csharp
private static readonly HashSet<string> SensitiveFields = new(StringComparer.OrdinalIgnoreCase)
{
    "PasswordHash", "Password", "Secret", "Token",
    "ApiKey", "CreditCard", "Cvv", "SecretHash"
};

// Inside CollectAuditEntries:
var value = SensitiveFields.Contains(p.Metadata.Name)
    ? "***REDACTED***"
    : p.CurrentValue?.ToString();
```

### Bulk Cleanup

Audit logs grow over time. Periodic cleanup can be done via a Worker job:
- Records older than 90 days or 1 year are deleted (determined per project)
- This topic will be addressed separately when the need arises

---

# Caching Strategy: Pipeline Behavior + Redis

## Philosophy

Cache-Aside pattern, implemented as a Mediator pipeline behavior. The handler is unaware of the cache — adding the `ICacheable` interface to the query is sufficient. Cache behavior is declarative.

## ICacheable Interface

```csharp
public interface ICacheable
{
    string CacheKey { get; }
    string? CacheTtlSettingKey { get; }  // dynamic settings key (optional)
}
```

If `CacheTtlSettingKey` is provided, the TTL is read from dynamic settings. If not provided, the default TTL is used (`cache:default-ttl-minutes` settings key).

## Usage in Query Definition

```csharp
// Simple — default TTL
record GetProductQuery(Guid ProductId) : IRequest<ProductDto>, ICacheable
{
    public string CacheKey => $"product:{ProductId}";
    public string? CacheTtlSettingKey => "cache:product:ttl-minutes";
}

// When TTL is not specified, default is used
record GetCategoriesQuery() : IRequest<List<CategoryDto>>, ICacheable
{
    public string CacheKey => "categories:all";
    public string? CacheTtlSettingKey => null;  // default TTL is used
}
```

## CachingBehavior (Pipeline)

```csharp
public sealed class CachingBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : ICacheable
{
    private readonly IConnectionMultiplexer _redis;
    private readonly ISettingsService _settings;
    private readonly ILogger<CachingBehavior<TRequest, TResponse>> _logger;

    public async ValueTask<TResponse> Handle(
        TRequest request,
        MessageHandlerDelegate<TRequest, TResponse> next,
        CancellationToken ct)
    {
        var cacheKey = request.CacheKey;
        var redisDb = _redis.GetDatabase();

        // 1. Is it in cache?
        var cached = await redisDb.StringGetAsync(cacheKey);
        if (cached.HasValue)
        {
            _logger.LogInformation("Cache HIT for {CacheKey}", cacheKey);
            return JsonSerializer.Deserialize<TResponse>(cached!)!;
        }

        // 2. Run the handler
        _logger.LogInformation("Cache MISS for {CacheKey}", cacheKey);
        var response = await next(request, ct);

        // 3. Determine TTL (from dynamic settings or default)
        var ttlMinutes = request.CacheTtlSettingKey != null
            ? await _settings.GetAsync<int>(request.CacheTtlSettingKey, 10, ct)
            : await _settings.GetAsync<int>("cache:default-ttl-minutes", 10, ct);

        // 4. Write to cache
        await redisDb.StringSetAsync(
            cacheKey,
            JsonSerializer.Serialize(response),
            TimeSpan.FromMinutes(ttlMinutes));

        _logger.LogInformation(
            "Cached {CacheKey} for {TtlMinutes} minutes", cacheKey, ttlMinutes);

        return response;
    }
}
```

## Cache Invalidation

When data changes in command handlers, the related cache keys are invalidated. Declarative via the `ICacheInvalidator` interface:

```csharp
public interface ICacheInvalidator
{
    string[] CacheKeysToInvalidate { get; }
}

record UpdateProductCommand(Guid ProductId, ...) 
    : IRequest<ProductDto>, ICacheInvalidator
{
    public string[] CacheKeysToInvalidate => 
        [$"product:{ProductId}", "categories:all"];
}
```

`CacheInvalidationBehavior`:

```csharp
public sealed class CacheInvalidationBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : ICacheInvalidator
{
    public async ValueTask<TResponse> Handle(
        TRequest request,
        MessageHandlerDelegate<TRequest, TResponse> next,
        CancellationToken ct)
    {
        var response = await next(request, ct);

        // Invalidate after successful command
        var redisDb = _redis.GetDatabase();
        foreach (var key in request.CacheKeysToInvalidate)
        {
            if (key.Contains('*'))
            {
                // Pattern-based invalidation (SCAN + DEL)
                await InvalidatePatternAsync(redisDb, key);
            }
            else
            {
                await redisDb.KeyDeleteAsync(key);
            }
        }

        _logger.LogInformation(
            "Cache invalidated: {Keys}", 
            string.Join(", ", request.CacheKeysToInvalidate));

        return response;
    }
}
```

## Pipeline Behavior Order (Updated)

```
Logging → UnhandledException → Validation → Caching → Performance
                                              ↑
                                    (cache check on queries)

Logging → UnhandledException → Validation → CacheInvalidation → Performance
                                              ↑
                                    (cache clearing on commands)
```

## What to Cache, What Not to Cache

**Cache:**
- Frequently read, rarely changed: product list, category tree, settings
- Expensive to compute: dashboard statistics, reports
- External API responses: exchange rates, shipping prices

**Don't cache:**
- User-specific, frequently changing: cart, notifications, unread messages
- Security-critical: authorization checks, token validation
- Already fast: single row ID lookup (FindAsync)

## Redis Key Convention

```
cache:product:{id}                → single product
cache:products:list:{cursor}      → paginated product list
cache:categories:all              → all categories
cache:dashboard:stats:{date}      → daily statistics
settings:{key}                    → dynamic settings (separate prefix)
```

---

# Concurrency Handling: Optimistic Locking

## Problem

Two users update the same entity at the same time. The last writer wins, and the first user's changes are silently lost. This is unacceptable.

## Solution: Optimistic Concurrency

A `RowVersion` field is kept on the entity. EF Core checks during `SaveChanges` whether "the version I read is still the same." If it has changed, it throws `DbUpdateConcurrencyException`.

```
User A: read product (RowVersion=1)
User B: read product (RowVersion=1)
User A: change price → save → RowVersion=2 ✅
User B: change stock → save → expected RowVersion=1 but found 2 → CONFLICT ✅
```

**Why not Pessimistic (locking):**
- DB row lock → deadlock risk, scaling issues
- User opens form and thinks for 10 min → lock is held for 10 min
- Managing DB locks in a distributed system is a nightmare

## Adding RowVersion to BaseEntity

```csharp
public abstract class BaseEntity
{
    public Guid Id { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime? UpdatedAt { get; set; }
    public uint RowVersion { get; set; }  // concurrency token
}
```

## EF Core Configuration

In PostgreSQL the `xmin` system column can be used, but an explicit `RowVersion` is clearer and more portable:

```csharp
// In base entity configuration:
public class BaseEntityConfiguration<T> : IEntityTypeConfiguration<T>
    where T : BaseEntity
{
    public virtual void Configure(EntityTypeBuilder<T> builder)
    {
        builder.HasKey(e => e.Id);
        builder.Property(e => e.Id).HasDefaultValueSql("gen_random_uuid()");
        builder.Property(e => e.RowVersion).IsConcurrencyToken();
    }
}
```

`IsConcurrencyToken()` → EF Core adds `WHERE "RowVersion" = @originalVersion` to every UPDATE query. If the row is not updated (version has changed), it throws an exception.

## Automatic RowVersion Increment

In the `SaveChanges` override or interceptor:

```csharp
// Inside AppDbContext.SaveChanges:
var modifiedEntries = ChangeTracker.Entries<BaseEntity>()
    .Where(e => e.State == EntityState.Modified);

foreach (var entry in modifiedEntries)
{
    entry.Entity.RowVersion++;
}
```

## Catching in Global Exception Handler

`DbUpdateConcurrencyException` → `409 Conflict`:

```csharp
// Added inside Api/Infrastructure/GlobalExceptionHandler.cs:
DbUpdateConcurrencyException => (
    StatusCodes.Status409Conflict,
    "Conflict",
    "This record has been modified by another user. Please reload and try again."
),
```

## No Extra Code Needed in Handler

The handler does normal CRUD — concurrency control is entirely at the EF Core + interceptor level:

```csharp
public async Task<UpdateProductResponse> Handle(
    UpdateProductCommand request, CancellationToken ct)
{
    var product = await _db.Products.FindAsync(request.ProductId, ct)
        ?? throw new NotFoundException(nameof(Product), request.ProductId);

    _logger.LogInformation(
        "Updating product {ProductId}, current RowVersion {Version}",
        product.Id, product.RowVersion);

    product.Name = request.Name;
    product.Price = request.Price;

    // SaveChanges → RowVersion++ (interceptor)
    // → UPDATE ... WHERE Id = @id AND RowVersion = @originalVersion
    // → If version has changed, DbUpdateConcurrencyException
    await _db.SaveChangesAsync(ct);

    _logger.LogInformation(
        "Product {ProductId} updated, new RowVersion {Version}",
        product.Id, product.RowVersion);

    return new UpdateProductResponse(product.Id, product.RowVersion);
}
```

## Client Side

### Sending RowVersion in the Request

The client receives `RowVersion` when reading the entity and sends it back in the update request:

```csharp
// Update command:
record UpdateProductCommand(
    Guid ProductId,
    string Name,
    decimal Price,
    uint RowVersion  // the version the client knows
) : IRequest<UpdateProductResponse>;
```

### Version Check in Handler

```csharp
var product = await _db.Products.FindAsync(request.ProductId, ct)
    ?? throw new NotFoundException(nameof(Product), request.ProductId);

// The version sent by the client must match the version in the DB
if (product.RowVersion != request.RowVersion)
{
    _logger.LogWarning(
        "Concurrency conflict for product {ProductId}: "
        + "client version {ClientVersion}, db version {DbVersion}",
        product.Id, request.RowVersion, product.RowVersion);
    throw new ConcurrencyException(nameof(Product), product.Id);
}
```

### 409 Handling in Flutter/React

```dart
// Flutter:
try {
  await dio.put('/api/products/$id', data: updateData);
} on DioException catch (e) {
  if (e.response?.statusCode == 409) {
    // "This record has been modified by someone else. Please reload."
    showConflictDialog();
    await reloadProduct();
  }
}
```

## RowVersion in DTOs

`RowVersion` is always present in DTOs of entities that can be updated:

```csharp
// In the response:
record ProductDto(
    Guid Id,
    string Name,
    decimal Price,
    uint RowVersion  // client stores this and sends it back on update
);

// In the update response:
record UpdateProductResponse(
    Guid Id,
    uint RowVersion  // new version, client updates its copy
);
```

## ConcurrencyException (Application layer)

```csharp
public class ConcurrencyException : Exception
{
    public ConcurrencyException(string entityName, object entityId)
        : base($"{entityName} with id {entityId} has been modified by another user")
    {
        EntityName = entityName;
        EntityId = entityId;
    }

    public string EntityName { get; }
    public object EntityId { get; }
}
```

In the global exception handler:
```csharp
ConcurrencyException → 409 + ProblemDetails
DbUpdateConcurrencyException → 409 + ProblemDetails
```

---

# Dynamic Settings: DB + Redis Centralized Configuration System

## Philosophy

Application settings are two-layered:

**Layer 1 — Static (appsettings / .env):** Things required for the application to start up. Determined at deploy-time, does not change at runtime.
- Port numbers, connection strings, JWT secret, RMQ/Redis/ES URLs

**Layer 2 — Dynamic (DB + Redis):** Everything that can change while the application is running. Does not require redeployment.
- Cache TTL durations, rate limit values, feature flags, mail settings, pagination defaults, file upload limits, maintenance mode, etc.

**Rule:** Static settings are the default values. DB settings override them. If a new setting does not exist in the DB, the default from appsettings is used.

## Flow

```
Set: Admin panel → API endpoint → write to DB + write to Redis → done
Get: Anywhere → read from Redis → done
```

- No RMQ, no dictionary cache, no periodic refresh
- All pods read the same Redis — synchronization is automatic
- Both DB and Redis are updated at set time — instant propagation

## SystemSetting Entity

```csharp
public class SystemSetting : BaseEntity
{
    public required string Key { get; set; }
    public required string Value { get; set; }
    public string ValueType { get; set; } = "string";  // string | int | bool | decimal | json
    public string? Description { get; set; }
    public string Category { get; set; } = "general";
}
```

**Key convention:** Category-based, `:` separated:
- `cache:product:ttl-minutes` → "30"
- `auth:login-attempt-limit` → "5"
- `auth:lockout-duration-minutes` → "15"
- `mail:from-address` → "noreply@example.com"
- `mail:bcc-address` → "archive@example.com"
- `general:maintenance-mode` → "false"
- `upload:max-file-size-mb` → "10"
- `pagination:default-page-size` → "20"

## ISettingsService (Application layer)

```csharp
public interface ISettingsService
{
    Task<T> GetAsync<T>(string key, CancellationToken ct = default);
    Task<T> GetAsync<T>(string key, T defaultValue, CancellationToken ct = default);
    Task SetAsync<T>(string key, T value, string? description = null, CancellationToken ct = default);
    Task<Dictionary<string, string>> GetByCategoryAsync(string category, CancellationToken ct = default);
}
```

## SettingsService (Infrastructure layer)

```csharp
public class SettingsService : ISettingsService
{
    private readonly IApplicationDbContext _db;
    private readonly IConnectionMultiplexer _redis;
    private readonly IConfiguration _configuration;
    private const string RedisPrefix = "settings:";

    public async Task<T> GetAsync<T>(string key, T defaultValue, CancellationToken ct = default)
    {
        // 1. Read from Redis
        var redisDb = _redis.GetDatabase();
        var cached = await redisDb.StringGetAsync($"{RedisPrefix}{key}");

        if (cached.HasValue)
            return Deserialize<T>(cached!);

        // 2. If not in Redis, fetch from DB and write to Redis
        var setting = await _db.SystemSettings
            .FirstOrDefaultAsync(s => s.Key == key, ct);

        if (setting != null)
        {
            await redisDb.StringSetAsync($"{RedisPrefix}{key}", setting.Value);
            return Deserialize<T>(setting.Value);
        }

        // 3. If not in DB either, default from appsettings
        var configValue = _configuration[key.Replace(":", "__")];
        if (configValue != null)
            return Deserialize<T>(configValue);

        // 4. If nowhere, use the provided default
        return defaultValue;
    }

    public async Task SetAsync<T>(string key, T value, string? description = null, CancellationToken ct = default)
    {
        var stringValue = Serialize(value);

        // 1. Write to DB (upsert)
        var setting = await _db.SystemSettings
            .FirstOrDefaultAsync(s => s.Key == key, ct);

        if (setting == null)
        {
            setting = new SystemSetting
            {
                Key = key,
                Value = stringValue,
                ValueType = typeof(T).Name.ToLowerInvariant(),
                Description = description,
                Category = key.Split(':')[0],
            };
            _db.SystemSettings.Add(setting);
        }
        else
        {
            setting.Value = stringValue;
        }

        await _db.SaveChangesAsync(ct);

        // 2. Write to Redis (instant propagation)
        var redisDb = _redis.GetDatabase();
        await redisDb.StringSetAsync($"{RedisPrefix}{key}", stringValue);
    }
}
```

## Seeding at Application Startup

When the API starts, it seeds default values from appsettings into the DB (if they don't exist), then loads all of them into Redis:

```csharp
// In Program.cs or a HostedService:
public async Task SeedSettingsAsync()
{
    var defaults = new Dictionary<string, (string Value, string Type, string Desc, string Category)>
    {
        ["cache:product:ttl-minutes"] = ("15", "int", "Product cache duration (minutes)", "cache"),
        ["cache:default-ttl-minutes"] = ("10", "int", "Default cache duration (minutes)", "cache"),
        ["auth:login-attempt-limit"] = ("5", "int", "Max login attempts", "auth"),
        ["auth:lockout-duration-minutes"] = ("15", "int", "Account lockout duration (minutes)", "auth"),
        ["pagination:default-page-size"] = ("20", "int", "Default page size", "pagination"),
        ["pagination:max-page-size"] = ("100", "int", "Max page size", "pagination"),
        ["upload:max-file-size-mb"] = ("10", "int", "Max file size (MB)", "upload"),
        ["general:maintenance-mode"] = ("false", "bool", "Maintenance mode", "general"),
        ["mail:from-address"] = ("noreply@example.com", "string", "Mail sender address", "mail"),
    };

    foreach (var (key, (value, type, desc, category)) in defaults)
    {
        // Add if not in DB (don't touch if exists — admin may have changed it)
        if (!await _db.SystemSettings.AnyAsync(s => s.Key == key))
        {
            _db.SystemSettings.Add(new SystemSetting
            {
                Key = key, Value = value, ValueType = type,
                Description = desc, Category = category,
            });
        }
    }
    await _db.SaveChangesAsync();

    // Load all into Redis
    var allSettings = await _db.SystemSettings.ToListAsync();
    var redisDb = _redis.GetDatabase();
    foreach (var s in allSettings)
    {
        await redisDb.StringSetAsync($"{RedisPrefix}{s.Key}", s.Value);
    }
}
```

## Usage in Handler

```csharp
public async Task<PaginatedResponse<ProductDto>> Handle(
    GetProductsQuery request, CancellationToken ct)
{
    // Get cache TTL from settings
    var ttl = await _settingsService.GetAsync<int>(
        "cache:product:ttl-minutes", defaultValue: 15, ct);

    _logger.LogInformation("Using product cache TTL: {TtlMinutes} minutes", ttl);

    // ... handler logic
}
```

## API Endpoints

```csharp
// GET /api/settings/{key} — read a single setting
// GET /api/settings?category=cache — list by category
// PUT /api/settings/{key} — update setting (admin)
// POST /api/settings/seed — re-seed defaults (admin)
```

All settings endpoints require admin authorization (`.RequireAuthorization(p => p.RequireRole("Admin"))`).

## Other Hosts (Socket, Worker, etc.)

Other hosts access settings via the API endpoint:

```csharp
// In Socket or Worker:
var response = await _apiClient.GetAsync<SettingResponse>("/api/settings/general:maintenance-mode");
```

This preserves the "API is the brain" principle — other hosts do not directly access DB or Redis.

## Cache Integration

Dynamic settings work in integration with cache behavior. TTL is read from settings in `ICacheable` queries:

```csharp
record GetProductQuery(Guid ProductId) : IRequest<ProductDto>, ICacheable
{
    public string CacheKey => $"product:{ProductId}";
    // TTL is read from dynamic settings — inside CachingBehavior
    public string? CacheTtlSettingKey => "cache:product:ttl-minutes";
}
```

`CachingBehavior` sees this key and retrieves the TTL via `ISettingsService.GetAsync<int>(settingKey)`.

---

# Error Handling Strategy

## Exception Hierarchy

Custom exceptions are defined in the Application layer:

| Exception | HTTP Status | When |
|-----------|-------------|------|
| `NotFoundException` | 404 | When the entity is not found |
| `ValidationException` | 422 | FluentValidation or manual validation error |
| `ForbiddenException` | 403 | Authorized but no access to this resource |
| `UnauthorizedAccessException` | 401 | Identity could not be verified |

## Error Handling in Handler

Try-catch is **NOT written** in the handler. On error conditions, the appropriate exception is thrown and the upper layer catches it:

```csharp
// ✅ Correct: Throw exception, upper layer catches it
var user = await _db.Users.FindAsync(request.UserId, ct)
    ?? throw new NotFoundException(nameof(User), request.UserId);

// ❌ Wrong: Writing try-catch in handler and doing result wrapping
try { ... } catch (Exception ex) { return Result.Failure(ex.Message); }
```

## Global Exception Handler (Api layer)

The `IExceptionHandler` implementation in the Api maps exception types to HTTP status codes:

```csharp
NotFoundException      → 404 + ProblemDetails
ValidationException    → 422 + ProblemDetails (errors array)
ForbiddenException     → 403 + ProblemDetails
_                      → 500 + ProblemDetails (detail in Development, generic in Production)
```

Neither the handler nor the endpoint ever writes try-catch. This responsibility belongs entirely to the global exception handler.

---

# File Upload & Storage Pattern

## Infrastructure

In the development environment, **MinIO** (S3-compatible object storage) is used — same API as production. There are no two different behaviors like "save to disk locally / save to S3 in production."

In Docker Compose:
```yaml
minio:
  image: minio/minio
  container_name: wfm-minio
  ports:
    - "${MINIO_API_PORT:-9000}:9000"
    - "${MINIO_CONSOLE_PORT:-9001}:9001"
  environment:
    MINIO_ROOT_USER: ${MINIO_ROOT_USER:-minioadmin}
    MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD:-minioadmin}
  volumes:
    - minio_data:/data
  command: server /data --console-address ":9001"
  healthcheck:
    test: ["CMD", "mc", "ready", "local"]
    interval: 5s
    timeout: 5s
    retries: 5
```

## IStorageService (Application layer)

```csharp
public interface IStorageService
{
    Task<StorageResult> UploadAsync(
        Stream file,
        string path,
        string contentType,
        CancellationToken ct = default);

    Task<Stream> DownloadAsync(
        string path,
        CancellationToken ct = default);

    Task DeleteAsync(
        string path,
        CancellationToken ct = default);

    Task DeleteDirectoryAsync(
        string prefix,
        CancellationToken ct = default);

    string GetPublicUrl(string path);

    Task<string> GetSignedUrlAsync(
        string path,
        TimeSpan expiry,
        CancellationToken ct = default);
}

public record StorageResult(
    string Path,
    string Url,
    long SizeBytes,
    string ContentType);
```

## Implementation (Infrastructure layer)

`S3StorageService` — uses AWS SDK, compatible with both MinIO and AWS S3:

```csharp
public class S3StorageService : IStorageService
{
    private readonly IAmazonS3 _s3;
    private readonly StorageOptions _options;

    // MinIO locally, AWS S3 in production — only the endpoint URL changes
}
```

## File Path Convention

### Default: Entity-Based Path

If the handler does not specify a path, the generic structure takes over: `{entity}/{id}/{filename}`

```
products/abc-123/photo-a1b2c3d4.jpg
products/abc-123/gallery-d5e6f7g8.jpg
users/def-456/avatar-h9i0j1k2.png
orders/ghi-789/invoice-l3m4n5o6.pdf
```

### Custom Path Override

The handler can specify a different path if desired. Some processes may require a custom directory structure:

```csharp
// Default — generic structure generates the path:
var path = StoragePathHelper.Generate("products", product.Id, "photo", ext);
// → "products/abc-123/photo-a1b2c3d4.jpg"

// Custom path — handler determines it:
var path = $"exports/reports/{DateTime.UtcNow:yyyy/MM}/{reportId}{ext}";
// → "exports/reports/2026/04/abc123.pdf"

// Shared/public directory:
var path = $"public/banners/{campaignId}-{Guid.NewGuid()}{ext}";
// → "public/banners/camp-123-a1b2c3d4.jpg"
```

### StoragePathHelper (Generic Path Generator)

```csharp
public static class StoragePathHelper
{
    public static string Generate(string entity, Guid entityId, string purpose, string extension)
    {
        return $"{entity}/{entityId}/{purpose}-{Guid.NewGuid()}{extension}";
    }
}
```

If the handler does not provide a path, `StoragePathHelper.Generate()` is used. If the handler provides a custom path, it is used directly.

### Why Entity-Based Default:
- Works with soft delete — when the entity is deleted (at the hard delete stage), all files are deleted in a single move with `DeleteDirectoryAsync("products/abc-123/")`
- Readable — the file path reveals which entity it belongs to
- Easy bulk cleanup — directory deletion by entity

### Filename Convention:
- Original filename is not preserved (security + collision risk)
- `{purpose}-{guid}.{ext}` format: `photo-a1b2c3.jpg`, `avatar-d4e5f6.png`
- Purpose examples: `photo`, `avatar`, `document`, `invoice`, `gallery`

## Upload Flow

```
Client (multipart/form-data)
    ↓
API Endpoint: parse file + metadata
    ↓
Handler:
    1. Validate: size limit, allowed extensions, content type
    2. Generate path: {entity}/{id}/{purpose}-{guid}.{ext}
    3. _storageService.UploadAsync(stream, path, contentType)
    4. Save URL/path to entity (DB)
    5. Log: file uploaded, size, path
    ↓
Response: StorageResult (path, url, size)
```

## Handler Example

```csharp
public async Task<UploadProductPhotoResponse> Handle(
    UploadProductPhotoCommand request, CancellationToken ct)
{
    _logger.LogInformation(
        "Uploading product photo for {ProductId}, size {Size} bytes",
        request.ProductId, request.File.Length);

    var product = await _db.Products.FindAsync(request.ProductId, ct)
        ?? throw new NotFoundException(nameof(Product), request.ProductId);

    // Validate
    var maxSize = await _settings.GetAsync<int>("upload:max-file-size-mb", 10, ct);
    if (request.File.Length > maxSize * 1024 * 1024)
        throw new ValidationException($"File size exceeds {maxSize}MB limit");

    var allowedTypes = new[] { "image/jpeg", "image/png", "image/webp" };
    if (!allowedTypes.Contains(request.File.ContentType))
        throw new ValidationException("Invalid file type");

    // Upload
    var ext = Path.GetExtension(request.File.FileName);
    var path = $"products/{product.Id}/photo-{Guid.NewGuid()}{ext}";

    var result = await _storageService.UploadAsync(
        request.File.OpenReadStream(), path, request.File.ContentType, ct);

    // Update DB
    product.PhotoPath = result.Path;
    product.PhotoUrl = result.Url;
    await _db.SaveChangesAsync(ct);

    _logger.LogInformation(
        "Product photo uploaded: {ProductId}, path {Path}, size {Size}",
        product.Id, result.Path, result.SizeBytes);

    return new UploadProductPhotoResponse(result.Path, result.Url);
}
```

## Validation Rules

File validation is done in the handler, read from dynamic settings:

| Setting | Settings Key | Default |
|---------|-------------|---------|
| Max file size | `upload:max-file-size-mb` | 10 MB |
| Allowed image types | `upload:allowed-image-types` | `image/jpeg,image/png,image/webp` |
| Allowed document types | `upload:allowed-document-types` | `application/pdf` |

## Bulk Cleanup (Soft Delete Integration)

When an entity reaches the hard-delete stage (bulk cleanup Worker job):

```csharp
// Worker job or admin endpoint:
var deletedProducts = await _db.Products
    .IgnoreQueryFilters()
    .Where(p => p.IsDeleted && p.DeletedAt < DateTime.UtcNow.AddDays(-90))
    .ToListAsync(ct);

foreach (var product in deletedProducts)
{
    // Delete all files in a single move
    await _storageService.DeleteDirectoryAsync($"products/{product.Id}/", ct);

    // Physically delete the entity
    _db.Products.Remove(product);

    _logger.LogInformation(
        "Hard-deleted product {ProductId} and all storage files", product.Id);
}

await _db.SaveChangesAsync(ct);
```

## Signed URLs

For private files (invoice, private document), pre-signed URLs:

```csharp
// In handler:
var signedUrl = await _storageService.GetSignedUrlAsync(
    order.InvoicePath,
    TimeSpan.FromMinutes(15),  // valid for 15 min
    ct);
```

This URL redirects directly to MinIO/S3 — no proxy through the API is needed, performant.

---

# Idempotency: When the Same Request Arrives Twice

## Problem

In mobile applications and distributed systems, the same request can arrive multiple times:
- Network timeout → client retries
- User presses button twice
- Load balancer retry
- Queue consumer crash → message redelivery

Result: order created twice, payment charged twice, email sent twice.

## Solution: Idempotency Key

The client sends a unique `X-Idempotency-Key` header with every mutating request (POST, PUT, DELETE). The API checks this key in Redis — if already processed, returns the same response; if not processed, processes it and caches the result.

## Flow

```
Client → POST /api/orders (X-Idempotency-Key: "abc-123")
    ↓
IdempotencyBehavior (Mediator pipeline):
    ↓
Redis: SETNX idempotency:abc-123
    ├── Key already exists → return cached response (handler does not run)
    └── Key doesn't exist → handler runs → response written to Redis → response returned
```

## Which Operations Should Be Idempotent?

| Operation | Idempotency | Why |
|-----------|-------------|-----|
| Record creation (Create) | **YES** | Repeat → duplication |
| Payment transaction | **YES** | Repeat → double charge |
| Email/notification trigger | **YES** | Repeat → double email |
| Record update (Update) | Optional | Updating with the same value may be harmless |
| Record deletion (Delete) | Optional | Already deleted → NotFoundException |
| Read (Query/GET) | **NO** | Idempotent by nature |

## IIdempotent Interface

Create commands implement the `IIdempotent` interface — the pipeline behavior kicks in automatically:

```csharp
public interface IIdempotent
{
    // Idempotency key is taken from the Header and bound to the command
    string? IdempotencyKey { get; }
}

record CreateOrderCommand(
    Guid CustomerId,
    Guid CartId,
    string? IdempotencyKey = null
) : IRequest<CreateOrderResponse>, IIdempotent;
```

## IdempotencyBehavior (Pipeline)

```csharp
public sealed class IdempotencyBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IIdempotent
{
    private readonly IConnectionMultiplexer _redis;
    private readonly ILogger<IdempotencyBehavior<TRequest, TResponse>> _logger;
    private const string Prefix = "idempotency:";
    private static readonly TimeSpan DefaultTtl = TimeSpan.FromHours(24);

    public async ValueTask<TResponse> Handle(
        TRequest request,
        MessageHandlerDelegate<TRequest, TResponse> next,
        CancellationToken ct)
    {
        // If no idempotency key, normal flow (optional usage)
        if (string.IsNullOrEmpty(request.IdempotencyKey))
            return await next(request, ct);

        var redisDb = _redis.GetDatabase();
        var cacheKey = $"{Prefix}{request.IdempotencyKey}";

        // 1. Was it already processed?
        var cached = await redisDb.StringGetAsync(cacheKey);
        if (cached.HasValue)
        {
            _logger.LogInformation(
                "Idempotency HIT: key {Key} already processed, returning cached response",
                request.IdempotencyKey);
            return JsonSerializer.Deserialize<TResponse>(cached!)!;
        }

        // 2. Set processing flag (race condition prevention)
        var acquired = await redisDb.StringSetAsync(
            cacheKey, "processing", DefaultTtl, When.NotExists);

        if (!acquired)
        {
            // Another pod is processing the same key simultaneously — wait briefly and read from cache
            _logger.LogWarning(
                "Idempotency CONFLICT: key {Key} is being processed by another instance",
                request.IdempotencyKey);
            await Task.Delay(500, ct);
            cached = await redisDb.StringGetAsync(cacheKey);
            if (cached.HasValue && cached != "processing")
                return JsonSerializer.Deserialize<TResponse>(cached!)!;

            throw new ConflictException("Request is already being processed");
        }

        // 3. Run the handler
        var response = await next(request, ct);

        // 4. Cache the result (processing → actual result)
        await redisDb.StringSetAsync(
            cacheKey,
            JsonSerializer.Serialize(response),
            DefaultTtl);

        _logger.LogInformation(
            "Idempotency SET: key {Key} processed and cached",
            request.IdempotencyKey);

        return response;
    }
}
```

## Header Binding at the Endpoint

The `X-Idempotency-Key` header is bound to the command at the API endpoint:

```csharp
group.MapPost("/orders", async (
    [FromBody] CreateOrderRequest request,
    [FromHeader(Name = "X-Idempotency-Key")] string? idempotencyKey,
    IMediator mediator,
    CancellationToken ct) =>
{
    var command = new CreateOrderCommand(
        request.CustomerId,
        request.CartId,
        IdempotencyKey: idempotencyKey);

    var response = await mediator.Send(command, ct);
    return Results.Created($"/api/orders/{response.Id}", response);
})
.WithName("CreateOrder");
```

## Client Side (Flutter/React)

The client generates a UUID v4 and sends it as a header with every mutating request:

```dart
// Flutter:
final response = await dio.post('/api/orders',
  data: orderData,
  options: Options(headers: {
    'X-Idempotency-Key': Uuid().v4(),
  }),
);
```

On retry, the same key is sent again — this way the same operation does not run again.

## Redis Key Convention

```
idempotency:{key}  →  "processing" (in progress) or JSON response (completed)
TTL: 24 hours (if the same key arrives again within 24 hours, it returns from cache)
```

## Pipeline Behavior Order (Updated)

```
Logging → UnhandledException → Validation → Idempotency → Caching → Performance
                                                ↑
                                    (duplicate prevention on create commands)
```

Idempotency comes AFTER Validation — invalid requests are already filtered at the Validator, preventing unnecessary key writes to Redis.

## Important Rules

1. **Idempotency is NOT used on GET requests.** Queries are idempotent by nature.
2. **Key is optional.** If `IdempotencyKey` comes as null, the behavior is skipped and normal flow continues. This provides flexibility — it doesn't force every endpoint.
3. **TTL is 24 hours.** Long enough to catch retries, short enough to prevent Redis from bloating.
4. **Race condition protection.** `SETNX` + "processing" flag → two pods cannot start processing the same key at the same time.
5. **Response is cached.** When the second request arrives, the handler does not run, and the first response is returned as-is — the client does not see a different result.

---

# Logging Strategy: Virtual Debug

## Core Philosophy

You cannot set breakpoints in production — logs are your debugger. Every handler tells its story through logs. Someone reading them after the fact should be able to answer "what happened, with what data, where did it fail" from the logs alone, without knowing the code at all.

**No performance concern.** The log pipeline works non-blocking: `_logger.Log*()` → Serilog → `BoundedChannel.TryWrite()` (nanoseconds) → separate thread batch publish → RMQ. The total overhead on the handler is on the order of microseconds. Even if you write 50 log lines, it's immeasurably small compared to DB queries. If the channel fills up (10k capacity), `DropOldest` — it never blocks, at worst it loses logs.

**Therefore: log generously, don't be afraid, don't be stingy.**

## Two Log Rule for Every Business Step

For every meaningful business step in the handler:

1. **Entry log:** "I will perform this operation with the data I have"
2. **Result log:** "I performed this operation with this data, result is positive/negative"

On negative results, exception information and relevant context data are also added to the log.

## Concrete Pattern

```csharp
public async Task<CreateOrderResponse> Handle(
    CreateOrderCommand request, CancellationToken ct)
{
    _logger.LogInformation(
        "CreateOrder started for customer {CustomerId}, cart {CartId}",
        request.CustomerId, request.CartId);

    // 1. Customer check
    var customer = await _db.Customers.FindAsync(request.CustomerId, ct)
        ?? throw new NotFoundException(nameof(Customer), request.CustomerId);
    _logger.LogInformation(
        "Customer resolved: {CustomerId}, email {Email}, status {Status}",
        customer.Id, customer.Email, customer.Status);

    // 2. Cart loading
    var cart = await _db.Carts
        .Include(c => c.Items)
        .FirstOrDefaultAsync(c => c.Id == request.CartId, ct)
        ?? throw new NotFoundException(nameof(Cart), request.CartId);
    _logger.LogInformation(
        "Cart loaded: {CartId}, {ItemCount} items, subtotal {Subtotal}",
        cart.Id, cart.Items.Count, cart.Subtotal);

    // 3. Stock check (separate for each product)
    foreach (var item in cart.Items)
    {
        var product = await _db.Products.FindAsync(item.ProductId, ct)
            ?? throw new NotFoundException(nameof(Product), item.ProductId);

        if (product.Stock < item.Quantity)
        {
            _logger.LogWarning(
                "Stock insufficient for product {ProductId} ({ProductName}): "
                + "requested {RequestedQty}, available {AvailableStock}",
                product.Id, product.Name, item.Quantity, product.Stock);
            throw new ValidationException(
                $"Insufficient stock for {product.Name}");
        }

        _logger.LogInformation(
            "Stock OK for product {ProductId}: {RequestedQty}/{AvailableStock}",
            product.Id, item.Quantity, product.Stock);
    }

    // 4. Price verification
    var calculatedTotal = cart.Items.Sum(i => i.UnitPrice * i.Quantity);
    _logger.LogInformation(
        "Price verification: calculated {Calculated}, submitted {Submitted}, "
        + "tax {Tax}, discount {Discount}",
        calculatedTotal, request.SubmittedTotal,
        request.TaxAmount, request.DiscountAmount);

    if (calculatedTotal != request.SubmittedTotal)
    {
        _logger.LogWarning(
            "Price mismatch: calculated {Calculated} vs submitted {Submitted}",
            calculatedTotal, request.SubmittedTotal);
        throw new ValidationException("Price mismatch detected");
    }

    // 5. Order creation
    var order = new Order
    {
        CustomerId = customer.Id,
        Total = calculatedTotal,
    };
    _db.Orders.Add(order);
    await _db.SaveChangesAsync(ct);

    _logger.LogInformation(
        "Order {OrderId} created: customer {CustomerId}, "
        + "{ItemCount} items, total {Total}",
        order.Id, customer.Id, cart.Items.Count, order.Total);

    // 6. Order items
    foreach (var item in cart.Items)
    {
        var orderItem = new OrderItem
        {
            OrderId = order.Id,
            ProductId = item.ProductId,
            Quantity = item.Quantity,
            UnitPrice = item.UnitPrice,
        };
        _db.OrderItems.Add(orderItem);
    }
    await _db.SaveChangesAsync(ct);

    _logger.LogInformation(
        "Order items saved: {ItemCount} items for order {OrderId}",
        cart.Items.Count, order.Id);

    return new CreateOrderResponse(order.Id, order.Total, order.CreatedAt);
}
```

## Log Level Rules

| Level | When |
|-------|------|
| `Information` | Normal flow: step start, successful result, important data points |
| `Warning` | Unexpected but recoverable situation: insufficient stock, price mismatch, rate limit |
| `Error` | Unrecoverable error: before throwing an exception or in a catch block |
| `Debug` | Temporary during development — not carried to production |

## Structured Logging Rules

- **Always use message templates**, NEVER string interpolation:
  ```csharp
  // ✅ Correct — indexed as Serilog property
  _logger.LogInformation("Order {OrderId} created", order.Id);
  
  // ❌ Wrong — plain string, not filterable in Elasticsearch
  _logger.LogInformation($"Order {order.Id} created");
  ```

- **Property names are PascalCase** and meaningful: `{OrderId}`, `{CustomerId}`, `{ItemCount}` — not `{id}`, `{x}`, `{count}`.

- **Sensitive data is never logged:** Fields like password, token, credit card, API key are never written to logs. RmqLogSink does PII masking, but the first line of defense is the handler — never pass sensitive fields to the log at all.

## Logging Checklist for Handlers

Follow this checklist when writing a new handler:

- [ ] At handler start: "starting" log with request parameters
- [ ] After each DB query/external call: result summary log
- [ ] At each validation/check step: success → Info, failure → Warning
- [ ] At handler end: overall result log (created IDs, counts, totals)
- [ ] Before throwing exception: Warning or Error with context
- [ ] Sensitive data check: no password, token, key in logs

---

# Naming Conventions

## File Naming

| Type | Pattern | Example |
|------|---------|---------|
| Command | `{Action}Command.cs` | `CreateOrderCommand.cs` |
| Handler | `{Action}Handler.cs` | `CreateOrderHandler.cs` |
| Validator | `{Action}Validator.cs` | `CreateOrderValidator.cs` |
| Query | `{Query}Query.cs` | `GetOrderQuery.cs` |
| Query Handler | `{Query}Handler.cs` | `GetOrderHandler.cs` |
| Response (Command) | Nested record inside the Command file | `CreateOrderResponse` inside `CreateOrderCommand.cs` |
| DTO | `{Entity}Dto.cs` | `OrderDto.cs` |
| Endpoint | `{Feature}Endpoints.cs` | `OrderEndpoints.cs` |
| EF Config | `{Entity}Configuration.cs` | `OrderConfiguration.cs` |
| Interface | `I{Service}.cs` | `IOrderService.cs` |
| Implementation | `{Service}.cs` | `OrderService.cs` |

## Namespace Convention

```
{ProjectName}.Domain.Entities
{ProjectName}.Domain.Enums
{ProjectName}.Domain.Exceptions
{ProjectName}.Application.Features.{Feature}.Commands.{Action}
{ProjectName}.Application.Features.{Feature}.Queries.{Query}
{ProjectName}.Application.Common.Interfaces
{ProjectName}.Application.Common.Behaviors
{ProjectName}.Application.Common.Exceptions
{ProjectName}.Infrastructure.Persistence
{ProjectName}.Infrastructure.Persistence.Configurations
{ProjectName}.Infrastructure.Auth
{ProjectName}.Infrastructure.Messaging
{ProjectName}.Api.Endpoints
{ProjectName}.Api.Infrastructure
```

## Endpoint Naming

```csharp
// Route: kebab-case
group.MapPost("/create-order", CreateOrderAsync)

// Method name: PascalCase + Async suffix
static async Task<IResult> CreateOrderAsync(...)

// WithName: PascalCase, verb + noun
.WithName("CreateOrder")
```

---

# Notification Pattern: API → Socket Broadcast

## Two Methods EXIST — Choose the Right One Based on the Situation

In this pattern, there are two different methods. When writing each handler, if notification is needed, **look at the decision table and choose the right method**. Never use one as the default — evaluate each case.

### Decision Table

| Situation | Method | Why |
|-----------|--------|-----|
| Notification to a single user ("your order has been confirmed") | **HTTP (Instant)** | Fast, simple, user sees it immediately |
| Notification to a group ("new order received" → admins) | **HTTP (Instant)** | Group is small, should see immediately |
| Broadcast to everyone ("maintenance mode starting") | **HTTP (Instant)** | Urgent, everyone should see immediately |
| Multiple notifications from a single handler (in a loop) | **RMQ (Async)** | Handler shouldn't slow down, shouldn't make N HTTP calls |
| Notification after batch operation ("100 products imported") | **RMQ (Async)** | Batch operation, handler is already long-running |
| Notification failure is not critical (best-effort) | **RMQ (Async)** | Safe with retry/DLX, doesn't block the handler |
| Notification cannot be lost (e.g., payment confirmation) | **RMQ (Async)** | RMQ is durable, won't be lost, waits in queue even if Socket is down |

**Rule:** When in doubt, use HTTP (Instant) — sufficient and simple for most cases. Only switch to RMQ in the specific situations listed in the table above.

---

## Method 1: HTTP (Instant Notification)

The handler makes a direct HTTP call to Socket. 5-10ms overhead.

### INotificationService (Application layer)

```csharp
public interface INotificationService
{
    // To a single user
    Task SendToUserAsync(
        string eventName,
        object data,
        string userId,
        CancellationToken ct = default);

    // To a group (role-based)
    Task SendToGroupAsync(
        string eventName,
        object data,
        string groupName,
        CancellationToken ct = default);

    // To everyone
    Task BroadcastAsync(
        string eventName,
        object data,
        CancellationToken ct = default);
}
```

### Implementation (Infrastructure layer)

```csharp
public class HttpNotificationService : INotificationService
{
    private readonly HttpClient _httpClient; // directed to Socket, X-Internal-Token injected

    public async Task SendToUserAsync(
        string eventName, object data, string userId, CancellationToken ct)
    {
        await _httpClient.PostAsJsonAsync("/api/internal/broadcast", new
        {
            Event = eventName,
            Data = data,
            TargetUserId = userId,
            TargetType = "user",
        }, ct);
    }

    public async Task SendToGroupAsync(
        string eventName, object data, string groupName, CancellationToken ct)
    {
        await _httpClient.PostAsJsonAsync("/api/internal/broadcast", new
        {
            Event = eventName,
            Data = data,
            TargetGroup = groupName,
            TargetType = "group",
        }, ct);
    }

    public async Task BroadcastAsync(
        string eventName, object data, CancellationToken ct)
    {
        await _httpClient.PostAsJsonAsync("/api/internal/broadcast", new
        {
            Event = eventName,
            Data = data,
            TargetType = "all",
        }, ct);
    }
}
```

### Usage in Handler

```csharp
// Order confirmed → notify the user
await _notificationService.SendToUserAsync(
    "order-confirmed",
    new { OrderId = order.Id, Total = order.Total },
    order.CustomerId.ToString(),
    ct);

_logger.LogInformation(
    "Notification sent: order-confirmed to user {UserId} for order {OrderId}",
    order.CustomerId, order.Id);
```

---

## Method 2: RMQ (Async Notification)

The handler publishes an event to RMQ, Socket consumes and broadcasts it. Handler does not slow down.

### NotificationEvent (Application/Common/)

```csharp
public record NotificationEvent
{
    public string EventName { get; init; } = null!;
    public object Data { get; init; } = null!;
    public string TargetType { get; init; } = "all";  // "user" | "group" | "all"
    public string? TargetUserId { get; init; }
    public string? TargetGroup { get; init; }
    public DateTime CreatedAt { get; init; } = DateTime.UtcNow;
}
```

### INotificationPublisher (Application layer)

```csharp
public interface INotificationPublisher
{
    Task PublishAsync(NotificationEvent notification, CancellationToken ct = default);
    Task PublishManyAsync(IEnumerable<NotificationEvent> notifications, CancellationToken ct = default);
}
```

### Implementation (Infrastructure layer)

```csharp
public class RmqNotificationPublisher : INotificationPublisher
{
    private readonly IRabbitMqConnection _connection;

    public async Task PublishAsync(NotificationEvent notification, CancellationToken ct)
    {
        // Publish to the notifications.fanout exchange
        // On the Socket side, the consumer listens to this queue and broadcasts via Hub
    }

    public async Task PublishManyAsync(
        IEnumerable<NotificationEvent> notifications, CancellationToken ct)
    {
        // Batch publish — each one is sent as a separate message to the queue
    }
}
```

### Usage in Handler (Batch/Async Case)

```csharp
// 100 products imported → notify each product owner
var notifications = importedProducts.Select(p => new NotificationEvent
{
    EventName = "product-imported",
    Data = new { ProductId = p.Id, ProductName = p.Name },
    TargetType = "user",
    TargetUserId = p.OwnerId.ToString(),
}).ToList();

await _notificationPublisher.PublishManyAsync(notifications, ct);

_logger.LogInformation(
    "Published {Count} product-imported notifications via RMQ",
    notifications.Count);
```

---

## RMQ Topology

```
Exchange: notifications.fanout (fanout, durable)
Queue: notifications.socket (durable)
Binding: notifications.fanout → notifications.socket
```

The Socket project consumes this queue and delivers messages to clients via Hub.

---

## Socket Side (InternalEndpoints)

The `/api/internal/broadcast` endpoint in the Socket project:

```csharp
// For the HTTP method:
app.MapPost("/api/internal/broadcast", async (
    BroadcastRequest request,
    IHubContext<NotificationHub> hubContext) =>
{
    switch (request.TargetType)
    {
        case "user":
            await hubContext.Clients
                .User(request.TargetUserId!)
                .SendAsync(request.Event, request.Data);
            break;
        case "group":
            await hubContext.Clients
                .Group(request.TargetGroup!)
                .SendAsync(request.Event, request.Data);
            break;
        case "all":
            await hubContext.Clients.All
                .SendAsync(request.Event, request.Data);
            break;
    }
    return Results.Ok();
})
.AddEndpointFilter<InternalSecretFilter>();
```

---

## Handler Writing Checklist

Check the following for every handler that requires notification:

- [ ] Is notification needed? (not every handler requires it)
- [ ] Look at the decision table: HTTP or RMQ?
- [ ] Single user / group / everyone?
- [ ] Event name: kebab-case (`order-confirmed`, `product-imported`)
- [ ] Data: only necessary fields (not the entire entity, think of it like a DTO)
- [ ] Log: notification sent/published log
- [ ] Sensitive data: no password, token, etc. in notification data

---

# Pagination Pattern: Cursor-Based Infinite Scroll

## Philosophy

Page-numbered pagination is not used. All lists work with infinite scroll logic: new data loads as the user scrolls down. This applies to both mobile and web.

## Generic Structure

### PaginatedQuery (Application/Common/)

Base parameters for all list queries:

```csharp
public record PaginatedQuery
{
    public string? Cursor { get; init; }          // cursor of the last item (first page: null)
    public int PageSize { get; init; } = 20;      // how many items to fetch (max 100)
    public string? Search { get; init; }           // general search (optional)
    public string? SortBy { get; init; }           // sort field (optional)
    public bool SortDesc { get; init; } = false;   // sort direction
    public bool IncludeCount { get; init; } = false; // include total count?
}
```

### PaginatedResponse<T> (Application/Common/)

Wrapper for all list responses:

```csharp
public record PaginatedResponse<T>(
    List<T> Items,              // items
    string? NextCursor,         // next page cursor (null = last page)
    bool HasMore,               // are there more?
    int? TotalCount             // populated only when IncludeCount=true
);
```

## Cursor Mechanism

The cursor is in Base64 encoded `{sortValue}|{id}` format. This supports sorting by any field (CreatedAt DESC, Price ASC, Name ASC, etc.).

### Cursor Encode/Decode Utility

```csharp
public static class CursorHelper
{
    public static string Encode(string sortValue, Guid id)
    {
        var raw = $"{sortValue}|{id}";
        return Convert.ToBase64String(Encoding.UTF8.GetBytes(raw));
    }

    public static (string SortValue, Guid Id) Decode(string cursor)
    {
        var raw = Encoding.UTF8.GetString(Convert.FromBase64String(cursor));
        var parts = raw.Split('|', 2);
        return (parts[0], Guid.Parse(parts[1]));
    }
}
```

## Frontend Flow

1. **First request:** `cursor=null` → first 20 items + `nextCursor="abc123"`
2. **Scroll down:** `cursor="abc123"` → next 20 + `nextCursor="def456"`
3. **Last page:** `nextCursor=null`, `hasMore=false` → loading ends

## Usage Example in Handler

```csharp
public async Task<PaginatedResponse<OrderDto>> Handle(
    GetOrdersQuery request, CancellationToken ct)
{
    var query = _db.Orders.AsQueryable();

    // Feature-specific filtering (each query writes its own Where conditions)
    if (request.Status.HasValue)
        query = query.Where(o => o.Status == request.Status.Value);

    // Search (generic)
    if (!string.IsNullOrWhiteSpace(request.Search))
        query = query.Where(o => o.CustomerName.Contains(request.Search));

    // Sort
    var sortBy = request.SortBy ?? "CreatedAt";
    query = request.SortDesc
        ? query.OrderByDescending(e => EF.Property<object>(e, sortBy))
        : query.OrderBy(e => EF.Property<object>(e, sortBy));

    // Cursor (if present, continue from that point)
    if (!string.IsNullOrEmpty(request.Cursor))
    {
        var (sortValue, lastId) = CursorHelper.Decode(request.Cursor);
        // Cursor-based WHERE condition (depends on sort direction)
        // ...implementation detail varies by sort field
    }

    // TotalCount (optional, only when requested)
    int? totalCount = request.IncludeCount
        ? await query.CountAsync(ct)
        : null;

    // Fetch (pull PageSize + 1 — to determine hasMore)
    var items = await query
        .Take(request.PageSize + 1)
        .ToListAsync(ct);

    var hasMore = items.Count > request.PageSize;
    if (hasMore) items.RemoveAt(items.Count - 1);

    // Generate cursor
    string? nextCursor = null;
    if (hasMore && items.Count > 0)
    {
        var lastItem = items.Last();
        var sortFieldValue = lastItem.GetType()
            .GetProperty(sortBy)?.GetValue(lastItem)?.ToString() ?? "";
        nextCursor = CursorHelper.Encode(sortFieldValue, lastItem.Id);
    }

    // Map to DTO
    var dtos = items.Select(o => new OrderDto(o.Id, o.CustomerName, o.Total, o.CreatedAt)).ToList();

    return new PaginatedResponse<OrderDto>(dtos, nextCursor, hasMore, totalCount);
}
```

## Important Rules

1. **Fetch PageSize + 1.** Fetch one extra item to determine `hasMore`, then remove the extra from the list.
2. **TotalCount is off by default.** `COUNT(*)` is expensive in PostgreSQL (MVCC full scan). Only compute when `IncludeCount=true` is sent. Usually not needed on mobile, may be needed in admin panels.
3. **PageSize max limit.** Limit `PageSize` to max 100 in the Validator — a malicious client shouldn't be able to fetch 10000.
4. **Filtering is NOT generic.** Each feature writes its own Where conditions in its handler. There is no generic filter system — consistent with the Vertical Slice philosophy.
5. **Cursor is opaque.** The frontend only stores the cursor and sends it back — it does not decode it or care about its contents.

---

# RMQ Consumer Topology Rule

Every consumer (MailSender, LogIngest, etc.) must **idempotently declare** its own RMQ topology at startup: exchange, queue, and binding. There is a chance the consumer starts before the API — it cannot assume the topology exists.

When declaring a queue, it must declare with **the same arguments** (e.g., if DLX exists, the DLX argument must also be passed). Otherwise RabbitMQ throws `PRECONDITION_FAILED`.

## Example: Email Consumer

```csharp
// Inside consumer ConnectAsync(), after creating the channel:
await _channel.ExchangeDeclareAsync("emails.fanout", ExchangeType.Fanout, durable: true, cancellationToken: ct);
await _channel.ExchangeDeclareAsync("emails.dlx", ExchangeType.Fanout, durable: true, cancellationToken: ct);
var queueArgs = new Dictionary<string, object?>
{
    { "x-dead-letter-exchange", "emails.dlx" },
};
await _channel.QueueDeclareAsync("emails.smtp", durable: true, exclusive: false,
    autoDelete: false, arguments: queueArgs, cancellationToken: ct);
await _channel.QueueBindAsync("emails.smtp", "emails.fanout", "", cancellationToken: ct);
```

## Example: Log Consumer

```csharp
await _channel.ExchangeDeclareAsync("logs.fanout", ExchangeType.Fanout, durable: true, cancellationToken: ct);
await _channel.QueueDeclareAsync("logs.elasticsearch", durable: true, exclusive: false,
    autoDelete: false, cancellationToken: ct);
await _channel.QueueBindAsync("logs.elasticsearch", "logs.fanout", "", cancellationToken: ct);
```

## Rule

This rule applies to both the producer side (Infrastructure/Messaging) and the consumer side. Both sides can declare the same topology — RabbitMQ behaves idempotently (re-declaring with the same arguments is not a problem).

---

# Soft Delete Strategy

## Rule

Physical deletion (hard delete) is not performed. All entities use soft delete: `IsDeleted` flag + `DeletedAt` timestamp. Deleted records stay in the DB and are automatically filtered out in queries.

## Entity Structure

Added to `BaseEntity` (or the `ISoftDeletable` interface):

```csharp
public interface ISoftDeletable
{
    bool IsDeleted { get; set; }
    DateTime? DeletedAt { get; set; }
    string? DeletedBy { get; set; }
}
```

## Global Query Filter

Automatic filter for all `ISoftDeletable` entities inside `ApplicationDbContext.OnModelCreating`:

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // Apply global filter to all ISoftDeletable entities
    foreach (var entityType in modelBuilder.Model.GetEntityTypes())
    {
        if (typeof(ISoftDeletable).IsAssignableFrom(entityType.ClrType))
        {
            modelBuilder.Entity(entityType.ClrType)
                .HasQueryFilter(
                    BuildSoftDeleteFilter(entityType.ClrType));
        }
    }
}

// Lambda expression builder for generic filter
private static LambdaExpression BuildSoftDeleteFilter(Type type)
{
    var parameter = Expression.Parameter(type, "e");
    var property = Expression.Property(parameter, nameof(ISoftDeletable.IsDeleted));
    var condition = Expression.Equal(property, Expression.Constant(false));
    return Expression.Lambda(condition, parameter);
}
```

## SaveChanges Interceptor

The `Delete` operation is converted to soft delete in the interceptor:

```csharp
// Inside DbContext.SaveChanges override or interceptor:
var deletedEntries = ChangeTracker.Entries<ISoftDeletable>()
    .Where(e => e.State == EntityState.Deleted);

foreach (var entry in deletedEntries)
{
    entry.State = EntityState.Modified;
    entry.Entity.IsDeleted = true;
    entry.Entity.DeletedAt = DateTime.UtcNow;
    entry.Entity.DeletedBy = _currentUser?.UserId;
}
```

## Accessing Deleted Records

In rare cases where you need to see deleted records (admin panel, recovery):

```csharp
// Bypass the global filter
var allOrders = await _db.Orders
    .IgnoreQueryFilters()
    .Where(o => o.IsDeleted)
    .ToListAsync(ct);
```

## Deletion in Handler

Normal `Remove` is called in the handler — the interceptor automatically converts it to soft delete:

```csharp
// In the handler:
var order = await _db.Orders.FindAsync(request.OrderId, ct)
    ?? throw new NotFoundException(nameof(Order), request.OrderId);

_db.Orders.Remove(order);  // interceptor converts to soft delete
await _db.SaveChangesAsync(ct);

_logger.LogInformation("Order {OrderId} soft-deleted by {UserId}",
    order.Id, _currentUser.UserId);
```

## Bulk Cleanup

Physical cleanup of soft-deleted records is done through a separate mechanism (Worker job, admin endpoint). This topic is currently out of scope — it will be addressed when the need arises.

## Index Recommendation

Add a composite index on the soft delete field — for query filter performance:

```csharp
builder.HasIndex(e => e.IsDeleted)
    .HasFilter("\"IsDeleted\" = false");  // partial index, only active records
```

---

# Workflows

## Adding a New Feature

1. **Domain** — Create/update entity if needed (`Domain/Entities/`)
2. **Application** — Create feature slice:
   ```
   Application/Features/{Feature}/Commands/{Action}/
   ├── {Action}Command.cs      → record : IRequest<{Action}Response>
   ├── {Action}Handler.cs      → IRequestHandler<Command, Response>
   └── {Action}Validator.cs    → AbstractValidator<Command> (MANDATORY)
   ```
3. **DTO** — Response record inside the Command file or under `Common/`
4. **Infrastructure** — Add new interface implementation if needed, add EF Configuration
5. **Api** — Add endpoint:
   ```csharp
   group.MapPost("/{feature}", {Action}Async)
       .RequireAuthorization()
       .WithName("{Action}");
   
   static async Task<IResult> {Action}Async(
       {Action}Request request,
       IMediator mediator,
       CancellationToken ct)
   {
       var response = await mediator.Send(
           new {Action}Command(...), ct);
       return Ok(response);
   }
   ```
6. **Migration** — If there are schema changes (inside Docker):
   ```bash
   docker compose exec api dotnet ef migrations add {Name} \
     --project ../WalkingForMe.Infrastructure \
     --startup-project .
   ```
7. **Test** — Smoke test: call the endpoint, check logs in Kibana

## Adding a New Query

1. **Application** — Create query slice:
   ```
   Application/Features/{Feature}/Queries/{Query}/
   ├── {Query}Query.cs          → record : IRequest<{Query}Response>
   └── {Query}Handler.cs        → IRequestHandler<Query, Response>
   ```
2. **Api** — Add GET endpoint, call `mediator.Send(query)`
3. **DTO** — Response record inside the Query file or under `Common/`
4. Validator is optional (not needed if query parameters are simple)

## Creating a Migration

```bash
# Run inside the Docker container (NEVER with local dotnet)
docker compose exec api dotnet ef migrations add {MigrationName} \
  --project ../WalkingForMe.Infrastructure \
  --startup-project .
```

In Development, auto-migrate is active: `db.Database.Migrate()` is called in Program.cs.

## Sending a Message to RMQ

Call through the interface in the handler — concrete RMQ knowledge does not leak into the handler:

```csharp
// Inside the handler:
await _emailSender.SendAsync(
    templateName: "order-confirmation",
    data: new { orderId, customerName },
    to: customer.Email,
    cancellationToken: ct);
```

`IEmailSender` is defined in Application, `RmqEmailSender` is implemented in Infrastructure.

## Auth-Required Endpoint

```csharp
group.MapPost("/orders", CreateOrderAsync)
    .RequireAuthorization(p => p.RequireRole("Dealer"))
    .WithName("CreateOrder");
```

To add rate limiting:
```csharp
    .RequireRateLimiting("create-order");
```

## Internal Service Endpoint (For Worker/Socket)

```csharp
group.MapPost("/internal/process-expired", ProcessExpiredAsync)
    .AddEndpointFilter<InternalSecretFilter>()
    .WithName("ProcessExpired");
```

`InternalSecretFilter` → validates the `X-Internal-Token` header. No JWT required.

---
