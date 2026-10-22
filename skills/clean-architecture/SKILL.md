---
name: clean-architecture
description: Structure a new application with Clean Architecture - Domain, Application, Infrastructure, and an entry project (API, worker) - with the dependency rule enforced by project references. Use when creating the projects of a new solution, adding a feature or a use case, modeling entities and aggregates, deciding in which layer a class, record, interface, or package belongs, writing use-case handlers, repositories, or the unit of work, registering services per layer (AddApplication, AddInfrastructure), validating Application records used by Minimal API endpoints, or reviewing project references. Doesn't cover HTTP endpoints (minimal-api skill) or repository-level setup (solution-setup skill). Existing solutions keep their structure unless a restructuring is requested.
---

# Clean Architecture

Aim for a solution where **the dependency rule is enforced by project references**: the domain depends on nothing, use cases depend only on the domain, technical details plug in from the outside, and the entry project is a thin shell. Inside each project, code is grouped by feature, so one change touches one folder per layer.

Existing solutions keep their structure. Don't move or split projects as a side effect of another change.

## 1. Projects and the dependency rule

| Project | Contains | References |
| --- | --- | --- |
| `MyApp.Domain` | Aggregates, entities, value objects, domain exceptions | Nothing |
| `MyApp.Application` | Use cases (commands, queries, handlers), their input and output records, interfaces for persistence (repositories, unit of work) and external services | `Domain` |
| `MyApp.Infrastructure` | EF Core `DbContext` and configurations, repository implementations, HTTP clients, other external services | `Application` |
| `MyApp.Api` (or another entry project) | Composition root: `Program.cs`, endpoints, authentication | `Application`, and `Infrastructure` for service registration only |

- References point inward only. A reference from `Domain` or `Application` to `Infrastructure` or the entry project breaks the architecture: invert it with an interface declared in `Application`.
- `Domain` has no package references to EF Core, ASP.NET Core, or serializers. `Application` has none to EF Core or ASP.NET Core.
- The entry project references `Infrastructure` only in the composition root (`Program.cs`, to call its registration method). Endpoints and background jobs use `Application` handlers, never `DbContext` or repositories.
- Don't add projects by technical type (`MyApp.Common`, `MyApp.Shared`, `MyApp.Helpers`): they become dependency magnets. Put the code in the layer that owns it.
- Add another project only for a real boundary: a second entry project (a worker service) that reuses `Application` and `Infrastructure`, or a library published as a package.

## 2. Folders by feature in every layer

```text
MyApp.Domain/
  DomainException.cs
  Orders/
    Order.cs                (aggregate root)
    OrderLine.cs            (child entity)
    OrderStatus.cs

MyApp.Application/
  DependencyInjection.cs
  Persistence/
    IUnitOfWork.cs
  Paging/
    PagedResponse.cs
  Orders/
    CreateOrder.cs          (CreateOrderCommand + CreateOrderHandler)
    CancelOrder.cs          (CancelOrderCommand + CancelOrderResult + CancelOrderHandler)
    GetOrder.cs             (GetOrderQuery + GetOrderHandler)
    ListOrders.cs           (ListOrdersQuery + ListOrdersHandler)
    OrderResponse.cs
    IOrderRepository.cs

MyApp.Infrastructure/
  DependencyInjection.cs
  Persistence/
    AppDbContext.cs
    Configurations/OrderConfiguration.cs
  Orders/
    OrderRepository.cs

MyApp.Api/
  Program.cs
  Features/
    Orders/OrderEndpoints.cs
```

- Group by feature (`Orders/`, `Customers/`), not by type (`Services/`, `Models/`, `Interfaces/`): the files of one change stay together.
- Use the same feature names in every layer.
- One use case per file: its input record, its result type if any, and its handler together (`CancelOrder.cs`).
- Types shared by several features of a layer go in a folder named after their purpose (`Persistence/`, `Paging/`), not `Common/`.
- Keep namespaces aligned with folders (`MyApp.Application.Orders`).

## 3. Domain

Match the modeling effort to the feature:

| Feature | Model |
| --- | --- |
| Business rules and invariants (orders, payments, scheduling) | Aggregates with methods, value objects, protected state (this section) |
| Plain data with little behavior (reference lists, settings) | Simple entities: no aggregate ceremony, but still no public setters on values with rules |

The four projects stay the same in both cases.

```csharp
namespace MyApp.Domain.Orders;

public sealed class Order
{
    private readonly List<OrderLine> _lines = [];

    private Order() { } // for EF Core

    public Guid Id { get; private set; }
    public string CustomerId { get; private set; } = string.Empty;
    public OrderStatus Status { get; private set; }
    public IReadOnlyCollection<OrderLine> Lines => _lines.AsReadOnly();

    public static Order Create(string customerId)
    {
        if (string.IsNullOrWhiteSpace(customerId))
            throw new DomainException("An order needs a customer.");

        return new Order { Id = Guid.NewGuid(), CustomerId = customerId, Status = OrderStatus.Open };
    }

    public void AddLine(string productId, int quantity)
    {
        if (quantity <= 0)
            throw new DomainException("Quantity must be positive.");

        _lines.Add(new OrderLine(productId, quantity));
    }

    public bool CanBeCancelled => Status == OrderStatus.Open;

    public void Cancel()
    {
        if (!CanBeCancelled)
            throw new DomainException($"Order {Id} can't be cancelled in status {Status}.");

        Status = OrderStatus.Cancelled;
    }
}
```

**Aggregates**
- An aggregate is a group of entities that must stay consistent together, changed in one transaction. Its root (`Order`) is the only entry point: child entities (`OrderLine`) are created and changed through the root's methods.
- One repository per aggregate root, never per child entity.
- Reference another aggregate by its ID (`CustomerId`), not by a navigation property: each aggregate is loaded and saved on its own.

**Protected state**
- No public setters: state changes go through methods named after the business operation (`AddLine`, `Cancel`).
- Expose collections as `IReadOnlyCollection<T>` backed by a private `List<T>`: a public `List<T>` lets any caller bypass the root's rules. EF Core maps the `_lines` backing field by convention.
- Create entities through a factory method or constructor that enforces the invariants; keep a private parameterless constructor for EF Core.
- Model concepts with rules as value objects (records): `Money`, `EmailAddress`.
- No persistence attributes and no infrastructure types in the domain: EF Core mapping lives in `Infrastructure` (`IEntityTypeConfiguration<T>`).

**Errors**
- Guards throw `DomainException` for states the caller must prevent: invalid input that validation should have stopped, or an operation called without checking its precondition. Reaching one is a bug, and the API returns `500` through its exception handler.
- Expected business outcomes are not exceptions: expose the precondition (`CanBeCancelled`) and let the handler return a result (section 4).

## 4. Application: use cases

A use case is a handler class with one public method. No mediator library: the caller injects the handler it needs.

```csharp
using System.ComponentModel.DataAnnotations;
using MyApp.Application.Persistence;
using MyApp.Domain.Orders;

namespace MyApp.Application.Orders;

// Validated by the API's AddValidation() (section 6)
public sealed record CreateOrderCommand(
    [Required, StringLength(50)] string CustomerId,
    [Required, StringLength(50)] string ProductId,
    [Range(1, 1000)] int Quantity);

public sealed class CreateOrderHandler(IOrderRepository orders, IUnitOfWork unitOfWork)
{
    public async Task<OrderResponse> HandleAsync(CreateOrderCommand command, CancellationToken cancellationToken)
    {
        var order = Order.Create(command.CustomerId);
        order.AddLine(command.ProductId, command.Quantity);

        orders.Add(order);
        await unitOfWork.SaveChangesAsync(cancellationToken);

        return OrderResponse.From(order);
    }
}
```

```csharp
namespace MyApp.Application.Orders;

public sealed record CancelOrderCommand(Guid OrderId);

public enum CancelOrderResult { Cancelled, NotFound, NotCancellable }

public sealed class CancelOrderHandler(IOrderRepository orders, IUnitOfWork unitOfWork)
{
    public async Task<CancelOrderResult> HandleAsync(CancelOrderCommand command, CancellationToken cancellationToken)
    {
        var order = await orders.FindAsync(command.OrderId, cancellationToken);
        if (order is null)
            return CancelOrderResult.NotFound;
        if (!order.CanBeCancelled)
            return CancelOrderResult.NotCancellable;

        order.Cancel();
        await unitOfWork.SaveChangesAsync(cancellationToken);
        return CancelOrderResult.Cancelled;
    }
}
```

```csharp
// MyApp.Application/Persistence/IUnitOfWork.cs
namespace MyApp.Application.Persistence;

public interface IUnitOfWork
{
    Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
}

// MyApp.Application/Orders/IOrderRepository.cs
using MyApp.Domain.Orders;

namespace MyApp.Application.Orders;

public interface IOrderRepository
{
    Task<Order?> FindAsync(Guid id, CancellationToken cancellationToken);
    void Add(Order order);
}
```

**Records and results**
- Input records (`CreateOrderCommand`, `ListOrdersQuery`) and output records (`OrderResponse`, `PagedResponse<T>`) live in `Application`. The entry project uses them directly: no duplicate request classes and no mapping layer there.
- Never return domain entities from a handler: map them to output records.
- Return expected outcomes as values: `null` for not found, a result type (`CancelOrderResult`) for business rules. The entry project translates them (HTTP status codes for an API). No HTTP types (`IResult`, status codes) in `Application`.
- Enforce limits the caller can't be trusted with in the handler, for example the maximum page size.
- Add a `CancellationToken` to every async handler method and pass it down.

**Persistence**
- Repositories and `IUnitOfWork` are declared in `Application`, next to the use cases that need them, and implemented in `Infrastructure`. They exist because `Application` can't reference EF Core; they aren't a goal in themselves.
- Repositories load and track aggregates (`FindAsync`, `Add`); they never call `SaveChanges`.
- The handler commits once, at the end, with `IUnitOfWork.SaveChangesAsync`: all changes of the use case, across repositories, are saved in one transaction.
- Declare interfaces for other external needs (`IPaymentGateway`, `IClock`) the same way, next to the feature that uses them.

## 5. Service registration per layer

Each layer registers its own services through one extension method. The entry project calls them.

```csharp
// MyApp.Application/DependencyInjection.cs
using Microsoft.Extensions.DependencyInjection;
using MyApp.Application.Orders;

namespace MyApp.Application;

public static class DependencyInjection
{
    public static IServiceCollection AddApplication(this IServiceCollection services)
    {
        services.AddScoped<CreateOrderHandler>();
        services.AddScoped<CancelOrderHandler>();
        services.AddScoped<GetOrderHandler>();
        services.AddScoped<ListOrdersHandler>();
        return services;
    }
}
```

```csharp
// MyApp.Infrastructure/Persistence/AppDbContext.cs
using Microsoft.EntityFrameworkCore;
using MyApp.Application.Persistence;
using MyApp.Domain.Orders;

namespace MyApp.Infrastructure.Persistence;

internal sealed class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options), IUnitOfWork
{
    public DbSet<Order> Orders => Set<Order>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
        => modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
}

// MyApp.Infrastructure/Orders/OrderRepository.cs
using Microsoft.EntityFrameworkCore;
using MyApp.Application.Orders;
using MyApp.Domain.Orders;
using MyApp.Infrastructure.Persistence;

namespace MyApp.Infrastructure.Orders;

internal sealed class OrderRepository(AppDbContext db) : IOrderRepository
{
    public Task<Order?> FindAsync(Guid id, CancellationToken cancellationToken)
        => db.Orders.Include(o => o.Lines).FirstOrDefaultAsync(o => o.Id == id, cancellationToken);

    public void Add(Order order) => db.Orders.Add(order);
}
```

```csharp
// MyApp.Infrastructure/DependencyInjection.cs
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using MyApp.Application.Orders;
using MyApp.Application.Persistence;
using MyApp.Infrastructure.Orders;
using MyApp.Infrastructure.Persistence;

namespace MyApp.Infrastructure;

public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructure(this IServiceCollection services, IConfiguration configuration)
    {
        services.AddDbContext<AppDbContext>(options =>
            options.UseSqlServer(configuration.GetConnectionString("Default")));
        services.AddScoped<IUnitOfWork>(provider => provider.GetRequiredService<AppDbContext>());
        services.AddScoped<IOrderRepository, OrderRepository>();

        services.AddHealthChecks()
            .AddDbContextCheck<AppDbContext>("database", tags: ["ready"]);
        return services;
    }
}
```

```csharp
// MyApp.Api/Program.cs
builder.Services.AddApplication();
builder.Services.AddInfrastructure(builder.Configuration);
```

- Register `IUnitOfWork` as the scoped `AppDbContext` instance: repositories and the unit of work then share one `DbContext` per request, so one `SaveChangesAsync` commits everything.
- Keep `DbContext`, repositories, and handlers scoped. Never register a repository as a singleton: it would capture a scoped `DbContext`.
- Register handlers explicitly. Don't scan assemblies by reflection: registrations stay visible and compatible with trimming.
- Make implementation classes in `Infrastructure` `internal sealed`: only the interfaces from `Application` and `AddInfrastructure` are visible, so the entry project can't use them directly.
- Register the health checks of a dependency in the layer that configures it: the database readiness check (`AddDbContextCheck<AppDbContext>`, package `Microsoft.Extensions.Diagnostics.HealthChecks.EntityFrameworkCore`) is added in `AddInfrastructure`, because `AppDbContext` is internal. The entry project maps the health endpoints (minimal-api skill).
- `Application` needs only `Microsoft.Extensions.DependencyInjection.Abstractions` for its registration method, not ASP.NET Core.

## 6. Validation of Application records used by Minimal API endpoints (from .NET 10)

Put DataAnnotations attributes on the input records in `Application` (`System.ComponentModel.DataAnnotations`, no package needed). The API's `builder.Services.AddValidation()` validates them (minimal-api skill): it discovers the endpoint parameter types and the types they contain, including records declared in `Application`.

- Don't add `Microsoft.Extensions.Validation`, `AddValidation()`, or `[ValidatableType]` to `Application`: the API already covers its records, and the attribute is experimental in .NET 10 (`ASP0029`).
- Test one invalid request per endpoint: missing validation produces no error or log, and the test is what proves it runs.

## 7. Tests per layer

| Test project | Tests | References |
| --- | --- | --- |
| `MyApp.Domain.Tests` | Aggregates and value objects: invariants, guards, state changes. No mocks needed | `Domain` |
| `MyApp.Application.Tests` | Handlers, with fakes for the repositories and `IUnitOfWork` | `Application` |
| `MyApp.Infrastructure.Tests` | Repositories and EF Core mappings against a real database of the same engine | `Infrastructure` |
| `MyApp.Api.Tests` | The entry project end to end (`WebApplicationFactory<Program>` for an API) | Entry project |

- `Infrastructure` classes are internal: grant `InternalsVisibleTo` to the test projects only, `MyApp.Infrastructure.Tests` and `MyApp.Api.Tests`. Never to the entry project.
- In `MyApp.Api.Tests`, point the app at a test database through configuration (the connection string), not by replacing `AppDbContext` registrations. Create the schema in the test fixture with the app's own `AppDbContext`, resolved from `factory.Services` in a scope: `Database.MigrateAsync()` when the project uses migrations, otherwise `Database.EnsureCreatedAsync()`.
- Test repositories against the real database engine, not the EF Core in-memory provider: it doesn't behave like a relational database (no transactions, no constraints).
- Optional: an architecture test project that fails the build when a forbidden reference appears (for example `Domain` → `Infrastructure`, or `Api` code using `Infrastructure` types), using an architecture-testing library.

## 8. Anti-patterns

| Avoid | Do instead |
| --- | --- |
| `Domain` or `Application` referencing EF Core, ASP.NET Core, or `Infrastructure` | Interface in `Application`, implementation in `Infrastructure` |
| Entry project using `DbContext` or repositories | Call `Application` handlers |
| `Common` / `Shared` / `Helpers` projects or folders | Code in the layer and folder that owns it |
| Folders by type (`Services/`, `Models/`, `Interfaces/`) | Folders by feature, same names in every layer |
| Public setters or public `List<T>` on entities | Methods on the root, `IReadOnlyCollection<T>` over a private field |
| Repository per child entity | One repository per aggregate root |
| Navigation property to another aggregate | Reference it by ID |
| Rich aggregate ceremony for plain reference data | Simple entity, same projects |
| EF Core attributes or types in `Domain` | `IEntityTypeConfiguration<T>` in `Infrastructure` |
| Repository methods calling `SaveChanges` | `IUnitOfWork.SaveChangesAsync` once in the handler |
| Repository registered as singleton | Scoped, like the `DbContext` |
| Exceptions for expected business outcomes | Precondition on the entity, result type from the handler |
| Domain entities returned to callers | Output records mapped in the handler |
| HTTP types or status codes in `Application` | Values returned, translated by the entry project |
| Duplicate request classes in the entry project | Use the `Application` records directly |
| Mediator library for plain request/response | Handler class injected by the caller |
| Reflection-based service registration | `AddApplication()` / `AddInfrastructure()` with explicit registrations |
| Public implementation classes in `Infrastructure` | `internal sealed`, exposed by interface |
| `AddValidation()` or `[ValidatableType]` in `Application` | The API's `AddValidation()`, one invalid-request test per endpoint |
| Repository tests on the EF Core in-memory provider | A real database of the same engine |
| Restructuring an existing solution during another change | Keep the structure; restructure only on request |

## Review checklist

- [ ] References point inward: `Domain` none, `Application` → `Domain`, `Infrastructure` → `Application`, entry project → `Application` (+ `Infrastructure` in the composition root only).
- [ ] No EF Core or ASP.NET Core package in `Domain` or `Application`.
- [ ] Folders by feature, same names across layers, one use case per file.
- [ ] Aggregates where there are business rules: changes through the root, read-only collections, other aggregates referenced by ID, one repository per root.
- [ ] Guards throw `DomainException`; expected outcomes are results returned by handlers.
- [ ] Repositories never save; each handler commits once with `IUnitOfWork`, which is the scoped `AppDbContext`.
- [ ] Handlers return records or results, never entities or HTTP types.
- [ ] Each layer registers its services explicitly; `Infrastructure` implementations are internal.
- [ ] Input records validated by the API's `AddValidation()`, proven by one invalid-request test per endpoint.
- [ ] One test project per layer, Infrastructure tested against a real database; optional architecture tests.
