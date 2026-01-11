# CQRS & MediatR Standards

Standards for implementing Command Query Responsibility Segregation (CQRS) using MediatR.

## Overview

CQRS separates read operations (Queries) from write operations (Commands), allowing each to be optimized independently. MediatR provides in-process messaging to implement this pattern cleanly.

```
┌─────────────┐     ┌──────────────────────┐     ┌─────────────┐
│  Controller │────▶│      MediatR         │────▶│   Handler   │
└─────────────┘     │  (Mediator Pattern)  │     └─────────────┘
                    │                      │
                    │  ┌────────────────┐  │
                    │  │   Behaviors    │  │
                    │  │  (Pipeline)    │  │
                    │  └────────────────┘  │
                    └──────────────────────┘
```

---

## Package Setup

```xml
<ItemGroup>
  <!-- MediatR -->
  <PackageReference Include="MediatR" Version="12.2.0" />

  <!-- FluentValidation for pipeline validation -->
  <PackageReference Include="FluentValidation" Version="11.9.0" />
  <PackageReference Include="FluentValidation.DependencyInjectionExtensions" Version="11.9.0" />
</ItemGroup>
```

---

## Project Structure

```
Application/
├── Commands/
│   └── CreateOrder/
│       ├── CreateOrderCommand.cs
│       ├── CreateOrderCommandHandler.cs
│       └── CreateOrderCommandValidator.cs
├── Queries/
│   └── GetOrder/
│       ├── GetOrderQuery.cs
│       ├── GetOrderQueryHandler.cs
│       └── OrderDto.cs
├── Behaviors/
│   ├── ValidationBehavior.cs
│   ├── LoggingBehavior.cs
│   └── TransactionBehavior.cs
└── DependencyInjection.cs
```

---

## Commands

Commands represent intentions to change state. They should be named as imperative verbs.

### Command Definition

```csharp
// <copyright file="CreateOrderCommand.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Application.Commands.CreateOrder;

using MediatR;

/// <summary>
/// Command to create a new order.
/// </summary>
public record CreateOrderCommand : IRequest<CreateOrderResult>
{
    /// <summary>
    /// Gets the customer identifier.
    /// </summary>
    public required Guid CustomerId { get; init; }

    /// <summary>
    /// Gets the order items.
    /// </summary>
    public required IReadOnlyList<OrderItemDto> Items { get; init; }

    /// <summary>
    /// Gets the shipping address.
    /// </summary>
    public required AddressDto ShippingAddress { get; init; }
}

/// <summary>
/// Result of creating an order.
/// </summary>
public record CreateOrderResult(Guid OrderId, string OrderNumber);

/// <summary>
/// DTO for order items in the command.
/// </summary>
public record OrderItemDto(Guid ProductId, int Quantity, decimal UnitPrice);

/// <summary>
/// DTO for address in the command.
/// </summary>
public record AddressDto(string Street, string City, string PostalCode, string Country);
```

### Command Handler

```csharp
// <copyright file="CreateOrderCommandHandler.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Application.Commands.CreateOrder;

using MediatR;
using Microsoft.Extensions.Logging;

using {CompanyName}.{ProjectName}.Application.Interfaces;
using {CompanyName}.{ProjectName}.Domain.Entities;
using {CompanyName}.{ProjectName}.Domain.ValueObjects;

/// <summary>
/// Handles the create order command.
/// </summary>
public class CreateOrderCommandHandler : IRequestHandler<CreateOrderCommand, CreateOrderResult>
{
    private readonly IOrderRepository orderRepository;
    private readonly IUnitOfWork unitOfWork;
    private readonly ILogger<CreateOrderCommandHandler> logger;

    /// <summary>
    /// Initializes a new instance of the <see cref="CreateOrderCommandHandler"/> class.
    /// </summary>
    public CreateOrderCommandHandler(
        IOrderRepository orderRepository,
        IUnitOfWork unitOfWork,
        ILogger<CreateOrderCommandHandler> logger)
    {
        this.orderRepository = orderRepository ?? throw new ArgumentNullException(nameof(orderRepository));
        this.unitOfWork = unitOfWork ?? throw new ArgumentNullException(nameof(unitOfWork));
        this.logger = logger ?? throw new ArgumentNullException(nameof(logger));
    }

    /// <inheritdoc/>
    public async Task<CreateOrderResult> Handle(
        CreateOrderCommand request,
        CancellationToken cancellationToken)
    {
        this.logger.LogInformation(
            "Creating order for customer {CustomerId} with {ItemCount} items",
            request.CustomerId,
            request.Items.Count);

        // Create domain entity
        var order = Order.Create(
            new CustomerId(request.CustomerId),
            Address.Create(
                request.ShippingAddress.Street,
                request.ShippingAddress.City,
                request.ShippingAddress.PostalCode,
                request.ShippingAddress.Country));

        // Add items
        foreach (var item in request.Items)
        {
            order.AddItem(
                new ProductId(item.ProductId),
                item.Quantity,
                Money.FromDecimal(item.UnitPrice));
        }

        // Persist
        await this.orderRepository.AddAsync(order, cancellationToken).ConfigureAwait(false);
        await this.unitOfWork.SaveChangesAsync(cancellationToken).ConfigureAwait(false);

        this.logger.LogInformation(
            "Order {OrderId} created with number {OrderNumber}",
            order.Id,
            order.OrderNumber);

        return new CreateOrderResult(order.Id.Value, order.OrderNumber.Value);
    }
}
```

### Command Validator

```csharp
// <copyright file="CreateOrderCommandValidator.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Application.Commands.CreateOrder;

using FluentValidation;

/// <summary>
/// Validator for the create order command.
/// </summary>
public class CreateOrderCommandValidator : AbstractValidator<CreateOrderCommand>
{
    /// <summary>
    /// Initializes a new instance of the <see cref="CreateOrderCommandValidator"/> class.
    /// </summary>
    public CreateOrderCommandValidator()
    {
        this.RuleFor(x => x.CustomerId)
            .NotEmpty()
            .WithMessage("Customer ID is required.");

        this.RuleFor(x => x.Items)
            .NotEmpty()
            .WithMessage("Order must contain at least one item.");

        this.RuleForEach(x => x.Items)
            .ChildRules(item =>
            {
                item.RuleFor(i => i.ProductId)
                    .NotEmpty()
                    .WithMessage("Product ID is required.");

                item.RuleFor(i => i.Quantity)
                    .GreaterThan(0)
                    .WithMessage("Quantity must be positive.");

                item.RuleFor(i => i.UnitPrice)
                    .GreaterThan(0)
                    .WithMessage("Unit price must be positive.");
            });

        this.RuleFor(x => x.ShippingAddress)
            .NotNull()
            .WithMessage("Shipping address is required.");

        this.When(x => x.ShippingAddress is not null, () =>
        {
            this.RuleFor(x => x.ShippingAddress.Street)
                .NotEmpty()
                .MaximumLength(200);

            this.RuleFor(x => x.ShippingAddress.City)
                .NotEmpty()
                .MaximumLength(100);

            this.RuleFor(x => x.ShippingAddress.PostalCode)
                .NotEmpty()
                .MaximumLength(20);

            this.RuleFor(x => x.ShippingAddress.Country)
                .NotEmpty()
                .MaximumLength(100);
        });
    }
}
```

---

## Queries

Queries retrieve data without side effects. They should be named as questions or data requests.

### Query Definition

```csharp
// <copyright file="GetOrderQuery.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Application.Queries.GetOrder;

using MediatR;

/// <summary>
/// Query to retrieve an order by ID.
/// </summary>
public record GetOrderQuery(Guid OrderId) : IRequest<OrderDto?>;
```

### Query Handler

```csharp
// <copyright file="GetOrderQueryHandler.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Application.Queries.GetOrder;

using MediatR;
using Microsoft.Extensions.Logging;

using {CompanyName}.{ProjectName}.Application.Interfaces;
using {CompanyName}.{ProjectName}.Domain.ValueObjects;

/// <summary>
/// Handles the get order query.
/// </summary>
public class GetOrderQueryHandler : IRequestHandler<GetOrderQuery, OrderDto?>
{
    private readonly IOrderRepository repository;
    private readonly ILogger<GetOrderQueryHandler> logger;

    /// <summary>
    /// Initializes a new instance of the <see cref="GetOrderQueryHandler"/> class.
    /// </summary>
    public GetOrderQueryHandler(
        IOrderRepository repository,
        ILogger<GetOrderQueryHandler> logger)
    {
        this.repository = repository ?? throw new ArgumentNullException(nameof(repository));
        this.logger = logger ?? throw new ArgumentNullException(nameof(logger));
    }

    /// <inheritdoc/>
    public async Task<OrderDto?> Handle(
        GetOrderQuery request,
        CancellationToken cancellationToken)
    {
        this.logger.LogDebug("Retrieving order {OrderId}", request.OrderId);

        var order = await this.repository
            .GetByIdAsync(new OrderId(request.OrderId), cancellationToken)
            .ConfigureAwait(false);

        if (order is null)
        {
            this.logger.LogDebug("Order {OrderId} not found", request.OrderId);
            return null;
        }

        return OrderDto.FromEntity(order);
    }
}
```

### Query DTO

```csharp
// <copyright file="OrderDto.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Application.Queries.GetOrder;

using {CompanyName}.{ProjectName}.Domain.Entities;

/// <summary>
/// Data transfer object for order information.
/// </summary>
public record OrderDto
{
    /// <summary>
    /// Gets the order identifier.
    /// </summary>
    public required Guid Id { get; init; }

    /// <summary>
    /// Gets the order number.
    /// </summary>
    public required string OrderNumber { get; init; }

    /// <summary>
    /// Gets the customer identifier.
    /// </summary>
    public required Guid CustomerId { get; init; }

    /// <summary>
    /// Gets the order status.
    /// </summary>
    public required string Status { get; init; }

    /// <summary>
    /// Gets the order total.
    /// </summary>
    public required decimal Total { get; init; }

    /// <summary>
    /// Gets the order items.
    /// </summary>
    public required IReadOnlyList<OrderItemDto> Items { get; init; }

    /// <summary>
    /// Gets the created date.
    /// </summary>
    public required DateTimeOffset CreatedAt { get; init; }

    /// <summary>
    /// Creates a DTO from a domain entity.
    /// </summary>
    public static OrderDto FromEntity(Order order)
    {
        return new OrderDto
        {
            Id = order.Id.Value,
            OrderNumber = order.OrderNumber.Value,
            CustomerId = order.CustomerId.Value,
            Status = order.Status.ToString(),
            Total = order.CalculateTotal().Amount,
            Items = order.Items.Select(OrderItemDto.FromEntity).ToList(),
            CreatedAt = order.CreatedAt,
        };
    }
}

/// <summary>
/// DTO for order item information.
/// </summary>
public record OrderItemDto
{
    /// <summary>
    /// Gets the product identifier.
    /// </summary>
    public required Guid ProductId { get; init; }

    /// <summary>
    /// Gets the product name.
    /// </summary>
    public required string ProductName { get; init; }

    /// <summary>
    /// Gets the quantity.
    /// </summary>
    public required int Quantity { get; init; }

    /// <summary>
    /// Gets the unit price.
    /// </summary>
    public required decimal UnitPrice { get; init; }

    /// <summary>
    /// Gets the subtotal.
    /// </summary>
    public required decimal Subtotal { get; init; }

    /// <summary>
    /// Creates a DTO from a domain entity.
    /// </summary>
    public static OrderItemDto FromEntity(OrderItem item)
    {
        return new OrderItemDto
        {
            ProductId = item.ProductId.Value,
            ProductName = item.ProductName,
            Quantity = item.Quantity,
            UnitPrice = item.UnitPrice.Amount,
            Subtotal = item.CalculateSubtotal().Amount,
        };
    }
}
```

---

## Pipeline Behaviors

Behaviors provide cross-cutting concerns that execute around handlers.

### Validation Behavior

```csharp
// <copyright file="ValidationBehavior.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Application.Behaviors;

using FluentValidation;
using MediatR;

/// <summary>
/// Pipeline behavior that validates requests before handling.
/// </summary>
/// <typeparam name="TRequest">The request type.</typeparam>
/// <typeparam name="TResponse">The response type.</typeparam>
public class ValidationBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    private readonly IEnumerable<IValidator<TRequest>> validators;

    /// <summary>
    /// Initializes a new instance of the <see cref="ValidationBehavior{TRequest, TResponse}"/> class.
    /// </summary>
    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators)
    {
        this.validators = validators ?? throw new ArgumentNullException(nameof(validators));
    }

    /// <inheritdoc/>
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        if (!this.validators.Any())
        {
            return await next().ConfigureAwait(false);
        }

        var context = new ValidationContext<TRequest>(request);

        var validationResults = await Task
            .WhenAll(this.validators.Select(v => v.ValidateAsync(context, cancellationToken)))
            .ConfigureAwait(false);

        var failures = validationResults
            .SelectMany(r => r.Errors)
            .Where(f => f is not null)
            .ToList();

        if (failures.Count > 0)
        {
            throw new ValidationException(failures);
        }

        return await next().ConfigureAwait(false);
    }
}
```

### Logging Behavior

```csharp
// <copyright file="LoggingBehavior.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Application.Behaviors;

using System.Diagnostics;

using MediatR;
using Microsoft.Extensions.Logging;

/// <summary>
/// Pipeline behavior that logs request handling.
/// </summary>
/// <typeparam name="TRequest">The request type.</typeparam>
/// <typeparam name="TResponse">The response type.</typeparam>
public class LoggingBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    private readonly ILogger<LoggingBehavior<TRequest, TResponse>> logger;

    /// <summary>
    /// Initializes a new instance of the <see cref="LoggingBehavior{TRequest, TResponse}"/> class.
    /// </summary>
    public LoggingBehavior(ILogger<LoggingBehavior<TRequest, TResponse>> logger)
    {
        this.logger = logger ?? throw new ArgumentNullException(nameof(logger));
    }

    /// <inheritdoc/>
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        var requestName = typeof(TRequest).Name;
        var requestId = Guid.NewGuid().ToString()[..8];

        this.logger.LogInformation(
            "[{RequestId}] Handling {RequestName}",
            requestId,
            requestName);

        var stopwatch = Stopwatch.StartNew();

        try
        {
            var response = await next().ConfigureAwait(false);

            stopwatch.Stop();

            this.logger.LogInformation(
                "[{RequestId}] Handled {RequestName} in {ElapsedMs}ms",
                requestId,
                requestName,
                stopwatch.ElapsedMilliseconds);

            return response;
        }
        catch (Exception ex)
        {
            stopwatch.Stop();

            this.logger.LogError(
                ex,
                "[{RequestId}] {RequestName} failed after {ElapsedMs}ms",
                requestId,
                requestName,
                stopwatch.ElapsedMilliseconds);

            throw;
        }
    }
}
```

### Transaction Behavior

```csharp
// <copyright file="TransactionBehavior.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Application.Behaviors;

using MediatR;
using Microsoft.Extensions.Logging;

using {CompanyName}.{ProjectName}.Application.Interfaces;

/// <summary>
/// Pipeline behavior that wraps commands in transactions.
/// </summary>
/// <typeparam name="TRequest">The request type.</typeparam>
/// <typeparam name="TResponse">The response type.</typeparam>
public class TransactionBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : ICommand<TResponse>
{
    private readonly IUnitOfWork unitOfWork;
    private readonly ILogger<TransactionBehavior<TRequest, TResponse>> logger;

    /// <summary>
    /// Initializes a new instance of the <see cref="TransactionBehavior{TRequest, TResponse}"/> class.
    /// </summary>
    public TransactionBehavior(
        IUnitOfWork unitOfWork,
        ILogger<TransactionBehavior<TRequest, TResponse>> logger)
    {
        this.unitOfWork = unitOfWork ?? throw new ArgumentNullException(nameof(unitOfWork));
        this.logger = logger ?? throw new ArgumentNullException(nameof(logger));
    }

    /// <inheritdoc/>
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        var requestName = typeof(TRequest).Name;

        this.logger.LogDebug("Beginning transaction for {RequestName}", requestName);

        await using var transaction = await this.unitOfWork
            .BeginTransactionAsync(cancellationToken)
            .ConfigureAwait(false);

        try
        {
            var response = await next().ConfigureAwait(false);

            await transaction.CommitAsync(cancellationToken).ConfigureAwait(false);

            this.logger.LogDebug("Transaction committed for {RequestName}", requestName);

            return response;
        }
        catch
        {
            await transaction.RollbackAsync(cancellationToken).ConfigureAwait(false);

            this.logger.LogWarning("Transaction rolled back for {RequestName}", requestName);

            throw;
        }
    }
}
```

---

## Registration

```csharp
// <copyright file="DependencyInjection.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Application;

using FluentValidation;
using MediatR;
using Microsoft.Extensions.DependencyInjection;

using {CompanyName}.{ProjectName}.Application.Behaviors;

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

        // Register MediatR
        services.AddMediatR(config =>
        {
            config.RegisterServicesFromAssembly(assembly);

            // Pipeline behaviors execute in order registered
            config.AddBehavior(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
            config.AddBehavior(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
        });

        // Register validators
        services.AddValidatorsFromAssembly(assembly);

        return services;
    }
}
```

---

## Controller Integration

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
[Produces("application/json")]
public class OrdersController : ControllerBase
{
    private readonly ISender sender;

    /// <summary>
    /// Initializes a new instance of the <see cref="OrdersController"/> class.
    /// </summary>
    public OrdersController(ISender sender)
    {
        this.sender = sender ?? throw new ArgumentNullException(nameof(sender));
    }

    /// <summary>
    /// Creates a new order.
    /// </summary>
    [HttpPost]
    [ProducesResponseType(typeof(CreateOrderResult), StatusCodes.Status201Created)]
    [ProducesResponseType(typeof(ValidationProblemDetails), StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> Create(
        [FromBody] CreateOrderCommand command,
        CancellationToken cancellationToken)
    {
        var result = await this.sender.Send(command, cancellationToken);
        return this.CreatedAtAction(
            nameof(this.GetById),
            new { id = result.OrderId },
            result);
    }

    /// <summary>
    /// Gets an order by ID.
    /// </summary>
    [HttpGet("{id:guid}")]
    [ProducesResponseType(typeof(OrderDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> GetById(
        Guid id,
        CancellationToken cancellationToken)
    {
        var result = await this.sender.Send(new GetOrderQuery(id), cancellationToken);
        return result is null ? this.NotFound() : this.Ok(result);
    }
}
```

---

## Naming Conventions

| Type | Convention | Example |
|------|------------|---------|
| Commands | `{Verb}{Noun}Command` | `CreateOrderCommand`, `CancelOrderCommand` |
| Command Handlers | `{Command}Handler` | `CreateOrderCommandHandler` |
| Command Results | `{Command}Result` or DTO | `CreateOrderResult` |
| Queries | `Get{Noun}Query` or `{Noun}Query` | `GetOrderQuery`, `OrdersByCustomerQuery` |
| Query Handlers | `{Query}Handler` | `GetOrderQueryHandler` |
| Query Results | `{Noun}Dto` | `OrderDto`, `OrderSummaryDto` |
| Validators | `{Command}Validator` | `CreateOrderCommandValidator` |
| Behaviors | `{Concern}Behavior` | `ValidationBehavior`, `LoggingBehavior` |

---

## Testing Handlers

```csharp
// <copyright file="CreateOrderCommandHandlerTests.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.UnitTests.Commands;

using FluentAssertions;
using Microsoft.Extensions.Logging;
using Moq;
using Xunit;

/// <summary>
/// Tests for <see cref="CreateOrderCommandHandler"/>.
/// </summary>
public class CreateOrderCommandHandlerTests
{
    private readonly Mock<IOrderRepository> repositoryMock;
    private readonly Mock<IUnitOfWork> unitOfWorkMock;
    private readonly Mock<ILogger<CreateOrderCommandHandler>> loggerMock;
    private readonly CreateOrderCommandHandler sut;

    /// <summary>
    /// Initializes a new instance of the <see cref="CreateOrderCommandHandlerTests"/> class.
    /// </summary>
    public CreateOrderCommandHandlerTests()
    {
        this.repositoryMock = new Mock<IOrderRepository>();
        this.unitOfWorkMock = new Mock<IUnitOfWork>();
        this.loggerMock = new Mock<ILogger<CreateOrderCommandHandler>>();

        this.sut = new CreateOrderCommandHandler(
            this.repositoryMock.Object,
            this.unitOfWorkMock.Object,
            this.loggerMock.Object);
    }

    /// <summary>
    /// Verifies that a valid command creates an order.
    /// </summary>
    [Fact]
    public async Task Handle_WithValidCommand_CreatesOrder()
    {
        // Arrange
        var command = new CreateOrderCommand
        {
            CustomerId = Guid.NewGuid(),
            Items = new[]
            {
                new OrderItemDto(Guid.NewGuid(), 2, 29.99m),
            },
            ShippingAddress = new AddressDto("123 Main St", "City", "12345", "USA"),
        };

        // Act
        var result = await this.sut.Handle(command, CancellationToken.None);

        // Assert
        result.Should().NotBeNull();
        result.OrderId.Should().NotBeEmpty();

        this.repositoryMock.Verify(
            r => r.AddAsync(It.IsAny<Order>(), It.IsAny<CancellationToken>()),
            Times.Once);

        this.unitOfWorkMock.Verify(
            u => u.SaveChangesAsync(It.IsAny<CancellationToken>()),
            Times.Once);
    }
}
```

---

## Best Practices

### Do

- ✅ One handler per command/query
- ✅ Use records for immutable commands and queries
- ✅ Validate in the pipeline, not in handlers
- ✅ Return DTOs from queries, not domain entities
- ✅ Keep handlers focused on orchestration
- ✅ Use `ISender` interface in controllers (not `IMediator`)

### Don't

- ❌ Put business logic in validators (only input validation)
- ❌ Call one handler from another
- ❌ Use MediatR for in-handler service calls
- ❌ Return domain entities from handlers
- ❌ Skip validation for "simple" commands

---

## Related Standards

- [Clean Architecture](clean-architecture.md) - Layer organization
- [Domain Design](domain-design.md) - Entity and aggregate patterns
- [Error Handling](error-handling.md) - Exception handling in pipelines
- [Testing & Coverage](testing-coverage.md) - Handler testing patterns
