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

