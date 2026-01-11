# Clean Architecture Standards

Standards for implementing Clean Architecture in .NET projects.

## Overview

Clean Architecture organizes code into concentric layers where dependencies point inward. The inner layers contain business logic and are independent of external concerns like databases, UI, or frameworks.

```
┌─────────────────────────────────────────────────────────────────┐
│                        Frameworks & Drivers                      │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    Interface Adapters                    │    │
│  │  ┌─────────────────────────────────────────────────┐    │    │
│  │  │              Application Business Rules          │    │    │
│  │  │  ┌─────────────────────────────────────────┐    │    │    │
│  │  │  │        Enterprise Business Rules         │    │    │    │
│  │  │  │              (Domain Layer)              │    │    │    │
│  │  │  └─────────────────────────────────────────┘    │    │    │
│  │  │                (Application Layer)               │    │    │
│  │  └─────────────────────────────────────────────────┘    │    │
│  │                  (Infrastructure Layer)                  │    │
│  └─────────────────────────────────────────────────────────┘    │
│                        (Presentation Layer)                      │
└─────────────────────────────────────────────────────────────────┘

                    Dependencies point INWARD →
```

---

## The Dependency Rule

**The most important rule**: Source code dependencies must point inward only.

- Inner layers know nothing about outer layers
- Domain layer has zero dependencies on other project layers
- Application layer depends only on Domain
- Infrastructure depends on Application and Domain
- Presentation depends on Application (and sometimes Infrastructure for DI)

```csharp
// ✅ CORRECT - Infrastructure implements Application interface
// Application layer defines:
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(OrderId id, CancellationToken ct);
}

// Infrastructure layer implements:
public class OrderRepository : IOrderRepository
{
    private readonly AppDbContext context;

    public async Task<Order?> GetByIdAsync(OrderId id, CancellationToken ct)
    {
        return await this.context.Orders
            .FirstOrDefaultAsync(o => o.Id == id.Value, ct)
            .ConfigureAwait(false);
    }
}

// ❌ WRONG - Domain depending on Infrastructure
namespace {CompanyName}.{ProjectName}.Domain.Entities;

using Microsoft.EntityFrameworkCore; // Never in Domain!

public class Order
{
    [Key] // EF Core attribute in Domain - violation!
    public Guid Id { get; set; }
}
```

---

## Project Structure

### Standard Clean Architecture Solution

```
src/
├── {CompanyName}.{ProjectName}.Domain/           # Enterprise Business Rules
│   ├── Entities/
│   │   ├── Order.cs
│   │   └── OrderItem.cs
│   ├── ValueObjects/
│   │   ├── OrderId.cs
│   │   ├── Money.cs
│   │   └── Address.cs
│   ├── Enums/
│   │   └── OrderStatus.cs
│   ├── Events/
│   │   ├── DomainEvent.cs
│   │   └── OrderCreatedEvent.cs
│   ├── Exceptions/
│   │   ├── DomainException.cs
│   │   └── InsufficientInventoryException.cs
│   └── Interfaces/
│       └── IDomainEventDispatcher.cs
│
├── {CompanyName}.{ProjectName}.Application/      # Application Business Rules
│   ├── Interfaces/
│   │   ├── IOrderRepository.cs
│   │   ├── IUnitOfWork.cs
│   │   └── IEmailService.cs
│   ├── Commands/
│   │   └── CreateOrder/
│   │       ├── CreateOrderCommand.cs
│   │       ├── CreateOrderCommandHandler.cs
│   │       └── CreateOrderCommandValidator.cs
│   ├── Queries/
│   │   └── GetOrder/
│   │       ├── GetOrderQuery.cs
│   │       ├── GetOrderQueryHandler.cs
│   │       └── OrderDto.cs
│   ├── Behaviors/
│   │   ├── ValidationBehavior.cs
│   │   └── LoggingBehavior.cs
│   ├── Mappings/
│   │   └── OrderMappingProfile.cs
│   └── DependencyInjection.cs
│
├── {CompanyName}.{ProjectName}.Infrastructure/   # Interface Adapters (Data)
│   ├── Persistence/
│   │   ├── AppDbContext.cs
│   │   ├── Configurations/
│   │   │   └── OrderConfiguration.cs
│   │   ├── Repositories/
│   │   │   └── OrderRepository.cs
│   │   └── UnitOfWork.cs
│   ├── Services/
│   │   └── EmailService.cs
│   └── DependencyInjection.cs
│
└── {CompanyName}.{ProjectName}.Api/              # Frameworks & Drivers
    ├── Controllers/
    │   └── V1/
    │       └── OrdersController.cs
    ├── Middleware/
    │   └── ExceptionHandlingMiddleware.cs
    ├── Filters/
    │   └── ValidationFilter.cs
    └── Program.cs
```

---

## Layer Responsibilities

### Domain Layer (Innermost)

**Purpose**: Contains enterprise-wide business rules and entities.

**Contains**:
- Entities with business logic
- Value Objects
- Domain Events
- Domain Exceptions
- Enumerations
- Interfaces for domain services

**Rules**:
- Zero external dependencies (no NuGet packages except primitives)
- No references to other project layers
- No framework-specific code (no EF attributes, no ASP.NET)
- All business rules encapsulated here

```csharp
// <copyright file="Order.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Domain.Entities;

/// <summary>
/// Represents a customer order.
/// </summary>
public class Order
{
    private readonly List<OrderItem> items = new();
    private readonly List<DomainEvent> domainEvents = new();

    private Order()
    {
        // EF Core constructor
    }

    /// <summary>
    /// Gets the order identifier.
    /// </summary>
    public OrderId Id { get; private set; } = default!;

    /// <summary>
    /// Gets the customer identifier.
    /// </summary>
    public CustomerId CustomerId { get; private set; } = default!;

    /// <summary>
    /// Gets the order status.
    /// </summary>
    public OrderStatus Status { get; private set; }

    /// <summary>
    /// Gets the order items.
    /// </summary>
    public IReadOnlyList<OrderItem> Items => this.items.AsReadOnly();

    /// <summary>
    /// Gets the domain events raised by this entity.
    /// </summary>
    public IReadOnlyList<DomainEvent> DomainEvents => this.domainEvents.AsReadOnly();

    /// <summary>
    /// Creates a new order.
    /// </summary>
    public static Order Create(CustomerId customerId)
    {
        var order = new Order
        {
            Id = OrderId.New(),
            CustomerId = customerId,
            Status = OrderStatus.Draft,
        };

        order.domainEvents.Add(new OrderCreatedEvent(order.Id));
        return order;
    }

    /// <summary>
    /// Adds an item to the order.
    /// </summary>
    public void AddItem(ProductId productId, int quantity, Money unitPrice)
    {
        if (this.Status != OrderStatus.Draft)
        {
            throw new DomainException("Cannot modify a submitted order.");
        }

        if (quantity <= 0)
        {
            throw new DomainException("Quantity must be positive.");
        }

        var existingItem = this.items.FirstOrDefault(i => i.ProductId == productId);
        if (existingItem is not null)
        {
            existingItem.IncreaseQuantity(quantity);
        }
        else
        {
            this.items.Add(new OrderItem(productId, quantity, unitPrice));
        }
    }

    /// <summary>
    /// Calculates the order total.
    /// </summary>
    public Money CalculateTotal()
    {
        return this.items
            .Select(i => i.CalculateSubtotal())
            .Aggregate(Money.Zero, (a, b) => a + b);
    }

    /// <summary>
    /// Submits the order for processing.
    /// </summary>
    public void Submit()
    {
        if (!this.items.Any())
        {
            throw new DomainException("Cannot submit an empty order.");
        }

        this.Status = OrderStatus.Submitted;
        this.domainEvents.Add(new OrderSubmittedEvent(this.Id));
    }

    /// <summary>
    /// Clears domain events after they have been dispatched.
    /// </summary>
    public void ClearDomainEvents() => this.domainEvents.Clear();
}
```

### Application Layer

**Purpose**: Contains application-specific business rules and orchestrates data flow.

**Contains**:
- Command and Query handlers (CQRS)
- Application services
- DTOs for external communication
- Interface definitions for infrastructure
- Validation logic
- Mapping profiles

**Rules**:
- Depends only on Domain layer
- Defines interfaces that Infrastructure implements
- Contains no infrastructure code
- Orchestrates use cases

```csharp
// <copyright file="CreateOrderCommandHandler.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Application.Commands.CreateOrder;

using MediatR;
using Microsoft.Extensions.Logging;

/// <summary>
/// Handles the create order command.
/// </summary>
public class CreateOrderCommandHandler : IRequestHandler<CreateOrderCommand, OrderDto>
{
    private readonly IOrderRepository repository;
    private readonly IUnitOfWork unitOfWork;
    private readonly ILogger<CreateOrderCommandHandler> logger;

    /// <summary>
    /// Initializes a new instance of the <see cref="CreateOrderCommandHandler"/> class.
    /// </summary>
    public CreateOrderCommandHandler(
        IOrderRepository repository,
        IUnitOfWork unitOfWork,
        ILogger<CreateOrderCommandHandler> logger)
    {
        this.repository = repository ?? throw new ArgumentNullException(nameof(repository));
        this.unitOfWork = unitOfWork ?? throw new ArgumentNullException(nameof(unitOfWork));
        this.logger = logger ?? throw new ArgumentNullException(nameof(logger));
    }

    /// <inheritdoc/>
    public async Task<OrderDto> Handle(
        CreateOrderCommand request,
        CancellationToken cancellationToken)
    {
        this.logger.LogInformation(
            "Creating order for customer {CustomerId}",
            request.CustomerId);

        var order = Order.Create(new CustomerId(request.CustomerId));

        foreach (var item in request.Items)
        {
            order.AddItem(
                new ProductId(item.ProductId),
                item.Quantity,
                new Money(item.UnitPrice, item.Currency));
        }

        await this.repository.AddAsync(order, cancellationToken).ConfigureAwait(false);
        await this.unitOfWork.SaveChangesAsync(cancellationToken).ConfigureAwait(false);

        this.logger.LogInformation("Order {OrderId} created", order.Id);

        return OrderDto.FromEntity(order);
    }
}
```

### Infrastructure Layer

**Purpose**: Contains implementations of interfaces defined in Application layer.

**Contains**:
- Database context and configurations
- Repository implementations
- External service implementations
- File system access
- Third-party service integrations

**Rules**:
- Implements interfaces from Application layer
- Can reference Domain for entity types
- Contains all framework-specific code (EF Core, HTTP clients, etc.)
- Never referenced by Domain or Application directly

```csharp
// <copyright file="OrderRepository.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Infrastructure.Persistence.Repositories;

using Microsoft.EntityFrameworkCore;

using {CompanyName}.{ProjectName}.Application.Interfaces;
using {CompanyName}.{ProjectName}.Domain.Entities;
using {CompanyName}.{ProjectName}.Domain.ValueObjects;

/// <summary>
/// Repository for order data access.
/// </summary>
public class OrderRepository : IOrderRepository
{
    private readonly AppDbContext context;

    /// <summary>
    /// Initializes a new instance of the <see cref="OrderRepository"/> class.
    /// </summary>
    public OrderRepository(AppDbContext context)
    {
        this.context = context ?? throw new ArgumentNullException(nameof(context));
    }

    /// <inheritdoc/>
    public async Task<Order?> GetByIdAsync(OrderId id, CancellationToken cancellationToken)
    {
        return await this.context.Orders
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.Id == id, cancellationToken)
            .ConfigureAwait(false);
    }

    /// <inheritdoc/>
    public async Task AddAsync(Order order, CancellationToken cancellationToken)
    {
        await this.context.Orders
            .AddAsync(order, cancellationToken)
            .ConfigureAwait(false);
    }
}
```

### Presentation Layer (API/UI)

**Purpose**: Entry point for the application, handles HTTP requests/responses.

**Contains**:
- Controllers or Minimal API endpoints
- View Models (if MVC)
- Middleware
- Filters
- Program.cs / Startup configuration

**Rules**:
- Thin layer - delegates to Application layer
- Handles HTTP concerns only
- Contains DI configuration
- No business logic

```csharp
// <copyright file="OrdersController.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Api.Controllers.V1;

using MediatR;
using Microsoft.AspNetCore.Mvc;

using {CompanyName}.{ProjectName}.Application.Commands.CreateOrder;
using {CompanyName}.{ProjectName}.Application.Queries.GetOrder;

/// <summary>
/// Controller for order operations.
/// </summary>
[ApiController]
[Route("api/v1/orders")]
public class OrdersController : ControllerBase
{
    private readonly IMediator mediator;

    /// <summary>
    /// Initializes a new instance of the <see cref="OrdersController"/> class.
    /// </summary>
    public OrdersController(IMediator mediator)
    {
        this.mediator = mediator ?? throw new ArgumentNullException(nameof(mediator));
    }

    /// <summary>
    /// Creates a new order.
    /// </summary>
    [HttpPost]
    [ProducesResponseType(typeof(OrderDto), StatusCodes.Status201Created)]
    public async Task<IActionResult> Create(
        [FromBody] CreateOrderCommand command,
        CancellationToken cancellationToken)
    {
        var result = await this.mediator.Send(command, cancellationToken);
        return this.CreatedAtAction(nameof(this.GetById), new { id = result.Id }, result);
    }

    /// <summary>
    /// Gets an order by ID.
    /// </summary>
    [HttpGet("{id}")]
    [ProducesResponseType(typeof(OrderDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> GetById(
        Guid id,
        CancellationToken cancellationToken)
    {
        var query = new GetOrderQuery(id);
        var result = await this.mediator.Send(query, cancellationToken);
        return result is null ? this.NotFound() : this.Ok(result);
    }
}
```

---

## Dependency Injection Setup

### Application Layer Registration

```csharp
// <copyright file="DependencyInjection.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Application;

using FluentValidation;
using MediatR;
using Microsoft.Extensions.DependencyInjection;

/// <summary>
/// Dependency injection extensions for the Application layer.
/// </summary>
public static class DependencyInjection
{
    /// <summary>
    /// Adds Application layer services.
    /// </summary>
    public static IServiceCollection AddApplicationServices(this IServiceCollection services)
    {
        var assembly = typeof(DependencyInjection).Assembly;

        services.AddMediatR(config =>
        {
            config.RegisterServicesFromAssembly(assembly);
            config.AddBehavior(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
            config.AddBehavior(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
        });

        services.AddValidatorsFromAssembly(assembly);

        return services;
    }
}
```

### Infrastructure Layer Registration

```csharp
// <copyright file="DependencyInjection.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Infrastructure;

using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;

/// <summary>
/// Dependency injection extensions for the Infrastructure layer.
/// </summary>
public static class DependencyInjection
{
    /// <summary>
    /// Adds Infrastructure layer services.
    /// </summary>
    public static IServiceCollection AddInfrastructureServices(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddDbContext<AppDbContext>(options =>
            options.UseSqlServer(configuration.GetConnectionString("Default")));

        services.AddScoped<IUnitOfWork, UnitOfWork>();
        services.AddScoped<IOrderRepository, OrderRepository>();
        services.AddScoped<IEmailService, EmailService>();

        return services;
    }
}
```

### Program.cs Composition

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Add layers in order
builder.Services.AddApplicationServices();
builder.Services.AddInfrastructureServices(builder.Configuration);

// Add presentation concerns
builder.Services.AddControllers();
```

---

## Common Violations

### ❌ Domain Depending on Infrastructure

```csharp
// WRONG - Domain entity with EF Core attributes
namespace {CompanyName}.{ProjectName}.Domain.Entities;

using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

public class Order
{
    [Key]
    public Guid Id { get; set; }

    [Required]
    [MaxLength(100)]
    public string CustomerName { get; set; }

    [Column(TypeName = "decimal(18,2)")]
    public decimal Total { get; set; }
}
```

**Fix**: Use fluent configuration in Infrastructure layer:

```csharp
// Infrastructure/Persistence/Configurations/OrderConfiguration.cs
public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.HasKey(o => o.Id);
        builder.Property(o => o.CustomerName).IsRequired().HasMaxLength(100);
        builder.Property(o => o.Total).HasColumnType("decimal(18,2)");
    }
}
```

### ❌ Application Layer with Database Access

```csharp
// WRONG - Handler directly using DbContext
public class GetOrderHandler : IRequestHandler<GetOrderQuery, OrderDto>
{
    private readonly AppDbContext context; // Direct dependency on Infrastructure!

    public async Task<OrderDto> Handle(GetOrderQuery request, CancellationToken ct)
    {
        return await this.context.Orders
            .Where(o => o.Id == request.Id)
            .Select(o => new OrderDto { ... })
            .FirstOrDefaultAsync(ct);
    }
}
```

**Fix**: Use repository interface:

```csharp
// CORRECT - Handler using repository
public class GetOrderHandler : IRequestHandler<GetOrderQuery, OrderDto>
{
    private readonly IOrderRepository repository;

    public async Task<OrderDto?> Handle(GetOrderQuery request, CancellationToken ct)
    {
        var order = await this.repository
            .GetByIdAsync(new OrderId(request.Id), ct)
            .ConfigureAwait(false);

        return order is null ? null : OrderDto.FromEntity(order);
    }
}
```

### ❌ Controller with Business Logic

```csharp
// WRONG - Business logic in controller
[HttpPost]
public async Task<IActionResult> Create(CreateOrderRequest request)
{
    // Validation, business rules, and data access all in controller!
    if (request.Items.Count == 0)
        return BadRequest("Order must have items");

    var total = request.Items.Sum(i => i.Quantity * i.UnitPrice);
    if (total > 10000)
        return BadRequest("Order exceeds credit limit");

    var order = new Order { ... };
    await this.context.Orders.AddAsync(order);
    await this.context.SaveChangesAsync();

    return Ok(order);
}
```

**Fix**: Move to Application layer:

```csharp
// CORRECT - Controller delegates to handler
[HttpPost]
public async Task<IActionResult> Create(
    CreateOrderCommand command,
    CancellationToken ct)
{
    var result = await this.mediator.Send(command, ct);
    return this.CreatedAtAction(nameof(this.GetById), new { id = result.Id }, result);
}
```

---

## Testing Strategy by Layer

| Layer | Test Type | Dependencies |
|-------|-----------|--------------|
| Domain | Unit Tests | None - pure logic |
| Application | Unit Tests | Mock repositories and services |
| Infrastructure | Integration Tests | Real database (TestContainers) |
| API | Integration Tests | WebApplicationFactory |

---

## Decision: When to Deviate

Clean Architecture is a guideline, not dogma. Consider simpler approaches when:

- **Small CRUD apps**: Vertical slice may be simpler
- **Prototypes**: Skip layers, refactor later
- **Read-heavy apps**: Consider CQRS with separate read models

Document deviations in `docs/ARCHITECTURE-DECISIONS.md` using ADR format.

---

## Related Standards

- [CQRS & MediatR](cqrs-mediatr.md) - Command/Query implementation
- [Domain Design](domain-design.md) - Entity and value object patterns
- [Dependency Injection](dependency-injection.md) - DI patterns and registration
- [Error Handling](error-handling.md) - Exception translation between layers
