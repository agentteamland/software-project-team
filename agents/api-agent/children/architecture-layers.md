# Architecture Layers (Detail)

## Domain
- Pure C#, no NuGet dependencies
- Entities derive from `BaseEntity` (Id, CreatedAt, UpdatedAt)
- `IAuditableEntity` (CreatedBy, ModifiedBy) is supported
- Enums, ValueObjects, Domain Exceptions live here
- Domain Events are defined (MediatR INotification)

## Application
- Mediator (MediatR or martinothamar/Mediator — pick one; MediatR is the default in our scaffolds)
- FluentValidation (AbstractValidator<TCommand>)
- Pipeline Behaviors order: Logging → UnhandledException → Validation → Performance
- `Common/Interfaces/` — all abstractions live here
- `Common/Behaviors/` — cross-cutting pipeline behaviors
- `Common/Exceptions/` — NotFoundException, ValidationException, ForbiddenException
- `Common/Mail/` — EmailJob contract (shared with the consumer)
- `Features/{Feature}/` — Vertical Slices

### Application .csproj — required package set

Application expresses the DB-facing interface as `IApplicationDbContext { DbSet<User> Users; ... }`. That means Application DOES depend on EF Core's abstractions (it imports `Microsoft.EntityFrameworkCore` to get `DbSet<>`). This is the pragmatic choice we ship. Pure-clean variants hide `DbSet` behind `IQueryable<T>` — we do not.

Consequence: Application.csproj must reference:

```xml
<ItemGroup>
  <PackageReference Include="MediatR" Version="12.*" />
  <PackageReference Include="FluentValidation" Version="11.*" />
  <PackageReference Include="FluentValidation.DependencyInjectionExtensions" Version="11.*" />
  <PackageReference Include="Microsoft.EntityFrameworkCore" Version="9.*" />
  <PackageReference Include="Microsoft.Extensions.DependencyInjection.Abstractions" Version="9.*" />
  <PackageReference Include="Microsoft.Extensions.Logging.Abstractions" Version="9.*" />
  <PackageReference Include="Microsoft.Extensions.Localization.Abstractions" Version="9.*" />
</ItemGroup>
```

- **`FluentValidation.DependencyInjectionExtensions`** is required for the `AddValidatorsFromAssembly(...)` extension. The base `FluentValidation` package does NOT include DI wiring.
- **`Microsoft.EntityFrameworkCore`** is required because `IApplicationDbContext` uses `DbSet<>`. If you go the IQueryable-only route, you can drop this — but then you lose `.Entry()`, `SaveChangesAsync` on the interface, etc.

## Infrastructure
- EF Core + PostgreSQL (Npgsql)
- `Persistence/ApplicationDbContext.cs` — `IApplicationDbContext` implementation
- `Persistence/Configurations/` — `IEntityTypeConfiguration<T>` files
- Audit trail + soft-delete logic lives **inside the DbContext's `SaveChangesAsync` override**, NOT as separate Singleton interceptor classes. See `audit-trail.md` for the pattern — constructor-injected `ICurrentUser` (Scoped) + DbContext (Scoped) = lifetimes align cleanly. Registering interceptors as Singletons that consume Scoped services fails DI validation.
- `Auth/` — JWT, BCrypt, Redis token stores
- `Messaging/` — RabbitMQ connection, producers, `RmqLogPublisher : BackgroundService`
- `Services/` — external service implementations
- `DependencyInjection.cs` — all Infrastructure DI registration

### Infrastructure .csproj — required package set

Because Infrastructure hosts a `BackgroundService` (the log publisher) and reads options from config, it needs hosting abstractions:

```xml
<ItemGroup>
  <PackageReference Include="Microsoft.EntityFrameworkCore.Relational" Version="9.*" />
  <PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="9.*">
    <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
    <PrivateAssets>all</PrivateAssets>
  </PackageReference>
  <PackageReference Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="9.*" />
  <PackageReference Include="Microsoft.Extensions.Hosting" Version="9.*" />
  <PackageReference Include="Microsoft.Extensions.Options.ConfigurationExtensions" Version="9.*" />
  <PackageReference Include="StackExchange.Redis" Version="2.*" />
  <PackageReference Include="AWSSDK.S3" Version="3.7.*" />     <!-- preferred over the `Minio` package; works against MinIO and real S3 -->
  <PackageReference Include="RabbitMQ.Client" Version="6.*" /> <!-- 6.x sync API; 7.x is async-first and NOT compatible -->
  <PackageReference Include="BCrypt.Net-Next" Version="4.*" />
  <PackageReference Include="Microsoft.IdentityModel.Tokens" Version="8.*" />
  <PackageReference Include="System.IdentityModel.Tokens.Jwt" Version="8.*" />
</ItemGroup>
```

- **`Microsoft.Extensions.Hosting`** — required for `BackgroundService` (used by `RmqLogPublisher`). Without it, Infrastructure fails to compile.
- **`Microsoft.Extensions.Options.ConfigurationExtensions`** — required for `.Bind()` on config sections in DI registration.
- **`AWSSDK.S3` over `Minio`** — the AWS SDK is the canonical choice because it talks to both MinIO and real S3 with the same code. The `Minio` package's API has churned between 5.x and 6.x (e.g., `ListObjectsAsync` → `ListObjectsEnumAsync`), which creates scaffold drift.
- **`RabbitMQ.Client` 6.x** — stick to 6.x sync API. The 7.x line is async-first and not drop-in-compatible with our patterns.

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

