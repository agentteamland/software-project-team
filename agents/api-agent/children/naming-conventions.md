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

