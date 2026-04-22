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
     --project ../ExampleApp.Infrastructure \
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
  --project ../ExampleApp.Infrastructure \
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
