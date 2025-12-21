---
applyTo: "**/*.cs"
---

# Clean Architecture (.NET 10)

## Example Solution Structure

```
src/
├── [Project].Domain/           # Entities, Value Objects, Domain Events, Domain Services
├── [Project].Application/      # Use Cases, Commands, Queries, Interfaces
├── [Project].Infrastructure/   # EF Core, External APIs, Implementations
└── [Project].Api/              # Minimal API Endpoints, Middleware
tests/
├── [Project].UnitTests/        # Domain + Application
└── [Project].IntegrationTests/ # Infrastructure + Api + Architecture Tests
```

## Layer Dependencies

```
Api → Infrastructure → Application → Domain
         ↓                  ↓
      (implements)    (defines interfaces)
```

| Layer          | Depends On     | Contains                                   |
| -------------- | -------------- | ------------------------------------------ |
| Domain         | ∅              | Entities, Value Objects, Domain Events     |
| Application    | Domain         | Use Cases, Interfaces, DTOs, Validators    |
| Infrastructure | Application    | DbContext, Repositories, External Services |
| Api            | Infrastructure | Endpoints, Request/Response, Middleware    |

**Forbidden:**

-   `Domain` / `Api` → EntityFrameworkCore
-   `Application` → Infrastructure
-   Circular dependencies

## Folder Organization (Feature-Based)

```csharp
// Correct: feature-oriented
src/[Project].Domain/
├── Orders/
│   ├── Order.cs
│   ├── OrderItem.cs
│   ├── OrderStatus.cs
│   └── Events/OrderCreatedEvent.cs
└── Customers/
    └── Customer.cs

// Wrong: technical folders
src/[Project].Domain/
├── Entities/
├── ValueObjects/
└── Events/
```

## Domain Layer

-   All business logic lives here
-   No dependencies on other layers
-   Entities are immutable where possible
-   Use Value Objects for type safety

```csharp
namespace Project.Domain.Orders;

public sealed class Order : Entity<OrderId>
{
    private readonly List<OrderItem> _items = [];

    public CustomerId CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();
    public Money Total => _items.Sum(i => i.Price);

    private Order() { }

    public static Order Create(CustomerId customerId, IEnumerable<OrderItem> items)
    {
        var order = new Order
        {
            Id = OrderId.New(),
            CustomerId = customerId,
            Status = OrderStatus.Pending
        };

        foreach (var item in items)
            order._items.Add(item);

        order.AddDomainEvent(new OrderCreatedEvent(order.Id));
        return order;
    }

    public void Confirm()
    {
        if (Status != OrderStatus.Pending)
            throw new DomainException("Only pending orders can be confirmed");

        Status = OrderStatus.Confirmed;
        AddDomainEvent(new OrderConfirmedEvent(Id));
    }
}
```

## Application Layer

-   Orchestrates domain objects
-   Defines interfaces (implemented in Infrastructure)
-   No business logic, only coordination

```csharp
namespace Project.Application.Orders;

public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(OrderId id, CancellationToken ct = default);
    Task AddAsync(Order order, CancellationToken ct = default);
}

public sealed class CreateOrderHandler(
    IOrderRepository repository,
    IUnitOfWork unitOfWork)
{
    public async Task<OrderId> HandleAsync(CreateOrderCommand command, CancellationToken ct)
    {
        var order = Order.Create(command.CustomerId, command.Items);

        await repository.AddAsync(order, ct);
        await unitOfWork.SaveChangesAsync(ct);

        return order.Id;
    }
}

public sealed record CreateOrderCommand(
    CustomerId CustomerId,
    IReadOnlyList<OrderItemDto> Items);
```

## Infrastructure Layer

-   Implements interfaces from Application
-   Database access, external APIs, file system
-   No business logic

```csharp
namespace Project.Infrastructure.Orders;

internal sealed class OrderRepository(AppDbContext db) : IOrderRepository
{
    public async Task<Order?> GetByIdAsync(OrderId id, CancellationToken ct) =>
        await db.Orders
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.Id == id, ct);

    public async Task AddAsync(Order order, CancellationToken ct) =>
        await db.Orders.AddAsync(order, ct);
}
```

## Api Layer

-   Minimal API endpoints only
-   Request/Response mapping
-   No business logic

```csharp
namespace Project.Api.Orders;

public static class OrderEndpoints
{
    public static void MapOrderEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/orders").WithTags("Orders");

        group.MapPost("/", CreateOrder);
        group.MapGet("/{id:guid}", GetOrder);
    }

    private static async Task<IResult> CreateOrder(
        CreateOrderRequest request,
        CreateOrderHandler handler,
        CancellationToken ct)
    {
        var command = request.ToCommand();
        var orderId = await handler.HandleAsync(command, ct);

        return TypedResults.Created($"/api/orders/{orderId}", new { id = orderId });
    }
}
```

## Architecture Tests

```csharp
namespace Project.IntegrationTests;

public class ArchitectureTests
{
    private static readonly Assembly DomainAssembly = typeof(DomainReference).Assembly;
    private static readonly Assembly ApplicationAssembly = typeof(ApplicationReference).Assembly;
    private static readonly Assembly InfrastructureAssembly = typeof(InfrastructureReference).Assembly;
    private static readonly Assembly ApiAssembly = typeof(Program).Assembly;

    [Fact]
    public void Domain_ShouldNotDependOnOtherLayers()
    {
        Types.InAssembly(DomainAssembly)
            .ShouldNot()
            .HaveDependencyOnAny(
                "Project.Application",
                "Project.Infrastructure",
                "Project.Api")
            .GetResult()
            .IsSuccessful
            .Should().BeTrue();
    }

    [Fact]
    public void Domain_ShouldNotUseEntityFramework()
    {
        Types.InAssembly(DomainAssembly)
            .ShouldNot()
            .HaveDependencyOn("Microsoft.EntityFrameworkCore")
            .GetResult()
            .IsSuccessful
            .Should().BeTrue();
    }

    [Fact]
    public void Application_ShouldNotDependOnInfrastructure()
    {
        Types.InAssembly(ApplicationAssembly)
            .ShouldNot()
            .HaveDependencyOnAny(
                "Project.Infrastructure",
                "Project.Api")
            .GetResult()
            .IsSuccessful
            .Should().BeTrue();
    }

    [Fact]
    public void Infrastructure_ShouldNotDependOnApi()
    {
        Types.InAssembly(InfrastructureAssembly)
            .ShouldNot()
            .HaveDependencyOn("Project.Api")
            .GetResult()
            .IsSuccessful
            .Should().BeTrue();
    }
}
```

## Coding Conventions

| Rule                        | Example                                |
| --------------------------- | -------------------------------------- |
| File-scoped namespaces      | `namespace Project.Domain.Orders;`     |
| One type per file           | `Order.cs` contains only `Order` class |
| Primary constructors for DI | `class Handler(IRepo repo)`            |
| Records for immutable data  | `record CreateOrderCommand(...)`       |
| Sealed classes by default   | `sealed class OrderRepository`         |

## Rules

1. **No Mediator** - Call handlers directly from endpoints via DI
2. **No Business Logic in Api/Infrastructure** - Only coordination and implementation
3. **Interfaces in Application** - Implementations in Infrastructure
4. **Feature Folders** - Never group by technical concern (Entities/, Services/)
5. **Dependency Injection** - All cross-layer dependencies via DI container
