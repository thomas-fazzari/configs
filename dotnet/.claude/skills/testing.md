---
applyTo: "**/*.cs"
---

# .NET 10 Testing Rules

## Stack

| Purpose         | Library               |
| --------------- | --------------------- |
| Framework       | xUnit v3              |
| Mocking         | NSubstitute           |
| Test Data       | Bogus                 |
| Containers      | Testcontainers        |
| API Integration | WebApplicationFactory |

## Project Structure

```
tests/
├── [Project].UnitTests/
│   ├── Domain/
│   ├── Application/
│   └── _Fixtures/
└── [Project].IntegrationTests/
    ├── Features/
    ├── Infrastructure/
    └── _Fixtures/
```

## Naming Conventions

```csharp
// Class: {ClassUnderTest}Tests
public class OrderServiceTests

// Method: {Method}_{Scenario}_{ExpectedResult}
public async Task CreateOrder_WithValidItems_ReturnsOrderId()
public async Task CreateOrder_WithEmptyCart_ThrowsValidationException()
```

## Unit Tests

### Strict AAA Structure

```csharp
[Fact]
public async Task Handle_ValidCommand_CreatesOrder()
{
    // Arrange
    var command = new CreateOrderCommandFaker().Generate();
    var handler = new CreateOrderHandler(_repositoryMock, _eventBusMock);

    // Act
    var result = await handler.Handle(command, CancellationToken.None);

    // Assert
    Assert.NotEqual(Guid.Empty, result);
    await _repositoryMock.Received(1).AddAsync(Arg.Any<Order>());
}
```

### Fakers with Bogus

```csharp
public sealed class CreateOrderCommandFaker : Faker<CreateOrderCommand>
{
    public CreateOrderCommandFaker()
    {
        CustomInstantiator(f => new CreateOrderCommand(
            CustomerId: f.Random.Guid(),
            Items: new OrderItemFaker().GenerateBetween(1, 5)
        ));
    }
}
```

### Mocking with NSubstitute

```csharp
// Setup
var repository = Substitute.For<IOrderRepository>();
repository.GetByIdAsync(orderId).Returns(expectedOrder);

// Verify
await repository.Received(1).SaveAsync(Arg.Is<Order>(o => o.Status == OrderStatus.Confirmed));
```

## Integration Tests

### Base Fixture with Testcontainers

```csharp
public class DatabaseFixture : IAsyncLifetime
{
    private readonly MsSqlContainer _container = new MsSqlBuilder()
        .WithImage("mcr.microsoft.com/mssql/server:2022-latest")
        .Build();

    public string ConnectionString => _container.GetConnectionString();

    public Task InitializeAsync() => _container.StartAsync();
    public Task DisposeAsync() => _container.DisposeAsync().AsTask();
}

[CollectionDefinition(nameof(DatabaseCollection))]
public class DatabaseCollection : ICollectionFixture<DatabaseFixture>;
```

### API Tests with WebApplicationFactory

```csharp
public class OrdersApiTests(ApiFixture fixture) : IClassFixture<ApiFixture>
{
    private readonly HttpClient _client = fixture.CreateClient();

    [Fact]
    public async Task CreateOrder_Returns201_WithLocationHeader()
    {
        // Arrange
        var request = new CreateOrderRequestFaker().Generate();

        // Act
        var response = await _client.PostAsJsonAsync("/api/orders", request);

        // Assert
        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
        Assert.NotNull(response.Headers.Location);
    }
}
```

### ApiFixture

```csharp
public class ApiFixture : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly PostgreSqlContainer _db = new PostgreSqlBuilder().Build();

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            services.RemoveAll<DbContextOptions<AppDbContext>>();
            services.AddDbContext<AppDbContext>(o => o.UseNpgsql(_db.GetConnectionString()));
        });
    }

    public async Task InitializeAsync()
    {
        await _db.StartAsync();
        // Run migrations
        using var scope = Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await db.Database.MigrateAsync();
    }

    public new async Task DisposeAsync() => await _db.DisposeAsync();
}
```

## Rules

1. **Isolation**: One test = one scenario. No unrelated multiple assertions.
2. **No Sleeps**: Use `Polly` for async retries/waits.
3. **Deterministic Data**: Seed Bogus with `Faker.Seed = new Random(42)` for reproducibility if needed.
4. **No Logic in Tests**: No `if`, `for`, `switch` in tests.
5. **Test Behavior, Not Implementation**: Test the result, not internal details.

## Theory & InlineData

```csharp
[Theory]
[InlineData("", false)]
[InlineData("valid@email.com", true)]
[InlineData("invalid-email", false)]
public void IsValidEmail_ReturnsExpected(string email, bool expected)
{
    Assert.Equal(expected, Email.IsValid(email));
}
```

## CI Integration

```bash
# Run with coverage
dotnet test --collect:"XPlat Code Coverage" --results-directory ./coverage

# Watch mode (dev)
dotnet watch test --project tests/Project.UnitTests
```

## Checklist

-   [ ] Unit tests for each use case (happy path + edge cases)
-   [ ] Integration tests for API endpoints
-   [ ] No `Thread.Sleep` or hardcoded delays
-   [ ] Explicit test method naming
-   [ ] Coverage > 80% on modified code
