# Domain Design Standards

Standards for designing domain entities, value objects, aggregates, and domain events following Domain-Driven Design (DDD) principles.

## Overview

The Domain layer contains the core business logic and rules. It should be independent of all external concerns and express the business domain in code.

---

## Entities

Entities have a unique identity that persists over time. Two entities with the same properties but different IDs are different entities.

### Entity Base Class

```csharp
// <copyright file="Entity.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Domain.Primitives;

/// <summary>
/// Base class for domain entities.
/// </summary>
/// <typeparam name="TId">The type of the entity identifier.</typeparam>
public abstract class Entity<TId> : IEquatable<Entity<TId>>
    where TId : notnull
{
    /// <summary>
    /// Gets the entity identifier.
    /// </summary>
    public TId Id { get; protected set; } = default!;

    /// <inheritdoc/>
    public bool Equals(Entity<TId>? other)
    {
        if (other is null)
        {
            return false;
        }

        if (ReferenceEquals(this, other))
        {
            return true;
        }

        return this.Id.Equals(other.Id);
    }

    /// <inheritdoc/>
    public override bool Equals(object? obj)
    {
        return obj is Entity<TId> entity && this.Equals(entity);
    }

    /// <inheritdoc/>
    public override int GetHashCode()
    {
        return this.Id.GetHashCode();
    }

    /// <summary>
    /// Equality operator.
    /// </summary>
    public static bool operator ==(Entity<TId>? left, Entity<TId>? right)
    {
        return Equals(left, right);
    }

    /// <summary>
    /// Inequality operator.
    /// </summary>
    public static bool operator !=(Entity<TId>? left, Entity<TId>? right)
    {
        return !Equals(left, right);
    }
}
```

### Entity Example

```csharp
// <copyright file="Order.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Domain.Entities;

using {CompanyName}.{ProjectName}.Domain.Enums;
using {CompanyName}.{ProjectName}.Domain.Events;
using {CompanyName}.{ProjectName}.Domain.Exceptions;
using {CompanyName}.{ProjectName}.Domain.Primitives;
using {CompanyName}.{ProjectName}.Domain.ValueObjects;

/// <summary>
/// Represents a customer order.
/// </summary>
public sealed class Order : AggregateRoot<OrderId>
{
    private readonly List<OrderItem> items = new();

    private Order()
    {
        // Required for EF Core
    }

    /// <summary>
    /// Gets the order number.
    /// </summary>
    public OrderNumber OrderNumber { get; private set; } = default!;

    /// <summary>
    /// Gets the customer identifier.
    /// </summary>
    public CustomerId CustomerId { get; private set; } = default!;

    /// <summary>
    /// Gets the order status.
    /// </summary>
    public OrderStatus Status { get; private set; }

    /// <summary>
    /// Gets the shipping address.
    /// </summary>
    public Address ShippingAddress { get; private set; } = default!;

    /// <summary>
    /// Gets the order items.
    /// </summary>
    public IReadOnlyList<OrderItem> Items => this.items.AsReadOnly();

    /// <summary>
    /// Gets the created timestamp.
    /// </summary>
    public DateTimeOffset CreatedAt { get; private set; }

    /// <summary>
    /// Gets the last modified timestamp.
    /// </summary>
    public DateTimeOffset? ModifiedAt { get; private set; }

    /// <summary>
    /// Creates a new order.
    /// </summary>
    /// <param name="customerId">The customer identifier.</param>
    /// <param name="shippingAddress">The shipping address.</param>
    /// <returns>A new order instance.</returns>
    public static Order Create(CustomerId customerId, Address shippingAddress)
    {
        var order = new Order
        {
            Id = OrderId.New(),
            OrderNumber = OrderNumber.Generate(),
            CustomerId = customerId,
            ShippingAddress = shippingAddress,
            Status = OrderStatus.Draft,
            CreatedAt = DateTimeOffset.UtcNow,
        };

        order.RaiseDomainEvent(new OrderCreatedEvent(order.Id));

        return order;
    }

    /// <summary>
    /// Adds an item to the order.
    /// </summary>
    /// <param name="productId">The product identifier.</param>
    /// <param name="productName">The product name.</param>
    /// <param name="quantity">The quantity.</param>
    /// <param name="unitPrice">The unit price.</param>
    /// <exception cref="DomainException">Thrown when the order cannot be modified.</exception>
    public void AddItem(ProductId productId, string productName, int quantity, Money unitPrice)
    {
        this.EnsureCanModify();

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
            this.items.Add(OrderItem.Create(productId, productName, quantity, unitPrice));
        }

        this.ModifiedAt = DateTimeOffset.UtcNow;
    }

    /// <summary>
    /// Removes an item from the order.
    /// </summary>
    /// <param name="productId">The product identifier.</param>
    public void RemoveItem(ProductId productId)
    {
        this.EnsureCanModify();

        var item = this.items.FirstOrDefault(i => i.ProductId == productId);

        if (item is not null)
        {
            this.items.Remove(item);
            this.ModifiedAt = DateTimeOffset.UtcNow;
        }
    }

    /// <summary>
    /// Calculates the order total.
    /// </summary>
    /// <returns>The total amount.</returns>
    public Money CalculateTotal()
    {
        if (!this.items.Any())
        {
            return Money.Zero;
        }

        return this.items
            .Select(i => i.CalculateSubtotal())
            .Aggregate((a, b) => a + b);
    }

    /// <summary>
    /// Submits the order for processing.
    /// </summary>
    /// <exception cref="DomainException">Thrown when the order cannot be submitted.</exception>
    public void Submit()
    {
        if (this.Status != OrderStatus.Draft)
        {
            throw new DomainException($"Cannot submit order in status {this.Status}.");
        }

        if (!this.items.Any())
        {
            throw new DomainException("Cannot submit an empty order.");
        }

        this.Status = OrderStatus.Submitted;
        this.ModifiedAt = DateTimeOffset.UtcNow;

        this.RaiseDomainEvent(new OrderSubmittedEvent(this.Id, this.CalculateTotal()));
    }

    /// <summary>
    /// Cancels the order.
    /// </summary>
    /// <param name="reason">The cancellation reason.</param>
    public void Cancel(string reason)
    {
        if (this.Status == OrderStatus.Shipped || this.Status == OrderStatus.Delivered)
        {
            throw new DomainException("Cannot cancel a shipped or delivered order.");
        }

        this.Status = OrderStatus.Cancelled;
        this.ModifiedAt = DateTimeOffset.UtcNow;

        this.RaiseDomainEvent(new OrderCancelledEvent(this.Id, reason));
    }

    private void EnsureCanModify()
    {
        if (this.Status != OrderStatus.Draft)
        {
            throw new DomainException("Only draft orders can be modified.");
        }
    }
}
```

---

## Value Objects

Value objects are immutable and compared by their properties, not identity. Two value objects with the same properties are equal.

### Value Object Base Class

```csharp
// <copyright file="ValueObject.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Domain.Primitives;

/// <summary>
/// Base class for value objects.
/// </summary>
public abstract class ValueObject : IEquatable<ValueObject>
{
    /// <summary>
    /// Gets the equality components for comparison.
    /// </summary>
    /// <returns>The components used for equality comparison.</returns>
    protected abstract IEnumerable<object?> GetEqualityComponents();

    /// <inheritdoc/>
    public bool Equals(ValueObject? other)
    {
        if (other is null || other.GetType() != this.GetType())
        {
            return false;
        }

        return this.GetEqualityComponents()
            .SequenceEqual(other.GetEqualityComponents());
    }

    /// <inheritdoc/>
    public override bool Equals(object? obj)
    {
        return obj is ValueObject other && this.Equals(other);
    }

    /// <inheritdoc/>
    public override int GetHashCode()
    {
        return this.GetEqualityComponents()
            .Aggregate(0, (hash, component) =>
                HashCode.Combine(hash, component?.GetHashCode() ?? 0));
    }

    /// <summary>
    /// Equality operator.
    /// </summary>
    public static bool operator ==(ValueObject? left, ValueObject? right)
    {
        return Equals(left, right);
    }

    /// <summary>
    /// Inequality operator.
    /// </summary>
    public static bool operator !=(ValueObject? left, ValueObject? right)
    {
        return !Equals(left, right);
    }
}
```

### Strongly-Typed ID

```csharp
// <copyright file="OrderId.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Domain.ValueObjects;

/// <summary>
/// Strongly-typed identifier for orders.
/// </summary>
public readonly record struct OrderId
{
    /// <summary>
    /// Initializes a new instance of the <see cref="OrderId"/> struct.
    /// </summary>
    /// <param name="value">The underlying value.</param>
    public OrderId(Guid value)
    {
        if (value == Guid.Empty)
        {
            throw new ArgumentException("Order ID cannot be empty.", nameof(value));
        }

        this.Value = value;
    }

    /// <summary>
    /// Gets the underlying value.
    /// </summary>
    public Guid Value { get; }

    /// <summary>
    /// Creates a new order identifier.
    /// </summary>
    /// <returns>A new order ID.</returns>
    public static OrderId New() => new(Guid.NewGuid());

    /// <summary>
    /// Parses a string to an order identifier.
    /// </summary>
    /// <param name="value">The string value.</param>
    /// <returns>The parsed order ID.</returns>
    public static OrderId Parse(string value) => new(Guid.Parse(value));

    /// <inheritdoc/>
    public override string ToString() => this.Value.ToString();
}
```

### Money Value Object

```csharp
// <copyright file="Money.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Domain.ValueObjects;

using {CompanyName}.{ProjectName}.Domain.Primitives;

/// <summary>
/// Represents a monetary value with currency.
/// </summary>
public sealed class Money : ValueObject
{
    /// <summary>
    /// Gets a zero money value.
    /// </summary>
    public static readonly Money Zero = new(0, "USD");

    private Money(decimal amount, string currency)
    {
        this.Amount = amount;
        this.Currency = currency;
    }

    /// <summary>
    /// Gets the amount.
    /// </summary>
    public decimal Amount { get; }

    /// <summary>
    /// Gets the currency code.
    /// </summary>
    public string Currency { get; }

    /// <summary>
    /// Creates a money value.
    /// </summary>
    /// <param name="amount">The amount.</param>
    /// <param name="currency">The currency code.</param>
    /// <returns>A new Money instance.</returns>
    public static Money Create(decimal amount, string currency)
    {
        if (amount < 0)
        {
            throw new ArgumentException("Amount cannot be negative.", nameof(amount));
        }

        if (string.IsNullOrWhiteSpace(currency) || currency.Length != 3)
        {
            throw new ArgumentException("Currency must be a 3-letter code.", nameof(currency));
        }

        return new Money(amount, currency.ToUpperInvariant());
    }

    /// <summary>
    /// Creates a USD money value.
    /// </summary>
    /// <param name="amount">The amount.</param>
    /// <returns>A new Money instance in USD.</returns>
    public static Money FromDecimal(decimal amount) => Create(amount, "USD");

    /// <summary>
    /// Adds two money values.
    /// </summary>
    public static Money operator +(Money left, Money right)
    {
        EnsureSameCurrency(left, right);
        return new Money(left.Amount + right.Amount, left.Currency);
    }

    /// <summary>
    /// Subtracts two money values.
    /// </summary>
    public static Money operator -(Money left, Money right)
    {
        EnsureSameCurrency(left, right);
        return new Money(left.Amount - right.Amount, left.Currency);
    }

    /// <summary>
    /// Multiplies money by a quantity.
    /// </summary>
    public static Money operator *(Money money, int quantity)
    {
        return new Money(money.Amount * quantity, money.Currency);
    }

    /// <inheritdoc/>
    protected override IEnumerable<object?> GetEqualityComponents()
    {
        yield return this.Amount;
        yield return this.Currency;
    }

    /// <inheritdoc/>
    public override string ToString() => $"{this.Amount:F2} {this.Currency}";

    private static void EnsureSameCurrency(Money left, Money right)
    {
        if (left.Currency != right.Currency)
        {
            throw new InvalidOperationException(
                $"Cannot operate on different currencies: {left.Currency} and {right.Currency}");
        }
    }
}
```

### Address Value Object

```csharp
// <copyright file="Address.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Domain.ValueObjects;

using {CompanyName}.{ProjectName}.Domain.Primitives;

/// <summary>
/// Represents a physical address.
/// </summary>
public sealed class Address : ValueObject
{
    private Address(string street, string city, string postalCode, string country)
    {
        this.Street = street;
        this.City = city;
        this.PostalCode = postalCode;
        this.Country = country;
    }

    /// <summary>
    /// Gets the street address.
    /// </summary>
    public string Street { get; }

    /// <summary>
    /// Gets the city.
    /// </summary>
    public string City { get; }

    /// <summary>
    /// Gets the postal code.
    /// </summary>
    public string PostalCode { get; }

    /// <summary>
    /// Gets the country.
    /// </summary>
    public string Country { get; }

    /// <summary>
    /// Creates a new address.
    /// </summary>
    public static Address Create(string street, string city, string postalCode, string country)
    {
        if (string.IsNullOrWhiteSpace(street))
        {
            throw new ArgumentException("Street is required.", nameof(street));
        }

        if (string.IsNullOrWhiteSpace(city))
        {
            throw new ArgumentException("City is required.", nameof(city));
        }

        if (string.IsNullOrWhiteSpace(postalCode))
        {
            throw new ArgumentException("Postal code is required.", nameof(postalCode));
        }

        if (string.IsNullOrWhiteSpace(country))
        {
            throw new ArgumentException("Country is required.", nameof(country));
        }

        return new Address(street.Trim(), city.Trim(), postalCode.Trim(), country.Trim());
    }

    /// <inheritdoc/>
    protected override IEnumerable<object?> GetEqualityComponents()
    {
        yield return this.Street;
        yield return this.City;
        yield return this.PostalCode;
        yield return this.Country;
    }

    /// <inheritdoc/>
    public override string ToString() =>
        $"{this.Street}, {this.City}, {this.PostalCode}, {this.Country}";
}
```

---

## Aggregates

Aggregates are clusters of entities and value objects with a root entity (Aggregate Root). All modifications must go through the aggregate root.

### Aggregate Root Base Class

```csharp
// <copyright file="AggregateRoot.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Domain.Primitives;

/// <summary>
/// Base class for aggregate roots.
/// </summary>
/// <typeparam name="TId">The type of the entity identifier.</typeparam>
public abstract class AggregateRoot<TId> : Entity<TId>
    where TId : notnull
{
    private readonly List<DomainEvent> domainEvents = new();

    /// <summary>
    /// Gets the domain events raised by this aggregate.
    /// </summary>
    public IReadOnlyList<DomainEvent> DomainEvents => this.domainEvents.AsReadOnly();

    /// <summary>
    /// Raises a domain event.
    /// </summary>
    /// <param name="domainEvent">The domain event to raise.</param>
    protected void RaiseDomainEvent(DomainEvent domainEvent)
    {
        this.domainEvents.Add(domainEvent);
    }

    /// <summary>
    /// Clears all domain events.
    /// </summary>
    public void ClearDomainEvents()
    {
        this.domainEvents.Clear();
    }
}
```

### Aggregate Design Rules

1. **One root per aggregate** - External references only to the root
2. **Transactional boundary** - Save entire aggregate atomically
3. **Reference by ID** - Don't hold references to other aggregates, use IDs
4. **Small aggregates** - Keep aggregates small for performance
5. **Encapsulation** - All changes through aggregate root methods

```csharp
// ✅ CORRECT - Reference by ID
public class Order : AggregateRoot<OrderId>
{
    public CustomerId CustomerId { get; private set; } // ID only, not Customer entity
}

// ❌ WRONG - Direct reference to another aggregate
public class Order : AggregateRoot<OrderId>
{
    public Customer Customer { get; private set; } // Don't do this!
}
```

---

## Domain Events

Domain events represent something significant that happened in the domain.

### Domain Event Base

```csharp
// <copyright file="DomainEvent.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Domain.Primitives;

using MediatR;

/// <summary>
/// Base class for domain events.
/// </summary>
public abstract record DomainEvent : INotification
{
    /// <summary>
    /// Gets the event identifier.
    /// </summary>
    public Guid EventId { get; } = Guid.NewGuid();

    /// <summary>
    /// Gets the timestamp when the event occurred.
    /// </summary>
    public DateTimeOffset OccurredAt { get; } = DateTimeOffset.UtcNow;
}
```

### Domain Event Examples

```csharp
// <copyright file="OrderCreatedEvent.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Domain.Events;

using {CompanyName}.{ProjectName}.Domain.Primitives;
using {CompanyName}.{ProjectName}.Domain.ValueObjects;

/// <summary>
/// Event raised when an order is created.
/// </summary>
public sealed record OrderCreatedEvent(OrderId OrderId) : DomainEvent;

/// <summary>
/// Event raised when an order is submitted.
/// </summary>
public sealed record OrderSubmittedEvent(OrderId OrderId, Money Total) : DomainEvent;

/// <summary>
/// Event raised when an order is cancelled.
/// </summary>
public sealed record OrderCancelledEvent(OrderId OrderId, string Reason) : DomainEvent;
```

### Domain Event Handler

```csharp
// <copyright file="OrderCreatedEventHandler.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Application.EventHandlers;

using MediatR;
using Microsoft.Extensions.Logging;

using {CompanyName}.{ProjectName}.Domain.Events;

/// <summary>
/// Handles the order created event.
/// </summary>
public class OrderCreatedEventHandler : INotificationHandler<OrderCreatedEvent>
{
    private readonly ILogger<OrderCreatedEventHandler> logger;

    /// <summary>
    /// Initializes a new instance of the <see cref="OrderCreatedEventHandler"/> class.
    /// </summary>
    public OrderCreatedEventHandler(ILogger<OrderCreatedEventHandler> logger)
    {
        this.logger = logger ?? throw new ArgumentNullException(nameof(logger));
    }

    /// <inheritdoc/>
    public Task Handle(OrderCreatedEvent notification, CancellationToken cancellationToken)
    {
        this.logger.LogInformation(
            "Order {OrderId} was created at {OccurredAt}",
            notification.OrderId,
            notification.OccurredAt);

        // Send email, update analytics, etc.

        return Task.CompletedTask;
    }
}
```

---

## Domain Exceptions

```csharp
// <copyright file="DomainException.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Domain.Exceptions;

/// <summary>
/// Base exception for domain rule violations.
/// </summary>
public class DomainException : Exception
{
    /// <summary>
    /// Initializes a new instance of the <see cref="DomainException"/> class.
    /// </summary>
    /// <param name="message">The error message.</param>
    public DomainException(string message)
        : base(message)
    {
    }

    /// <summary>
    /// Initializes a new instance of the <see cref="DomainException"/> class.
    /// </summary>
    /// <param name="message">The error message.</param>
    /// <param name="innerException">The inner exception.</param>
    public DomainException(string message, Exception innerException)
        : base(message, innerException)
    {
    }
}

/// <summary>
/// Exception thrown when an entity is not found.
/// </summary>
public class EntityNotFoundException : DomainException
{
    /// <summary>
    /// Initializes a new instance of the <see cref="EntityNotFoundException"/> class.
    /// </summary>
    public EntityNotFoundException(string entityName, object id)
        : base($"{entityName} with ID '{id}' was not found.")
    {
        this.EntityName = entityName;
        this.EntityId = id;
    }

    /// <summary>
    /// Gets the entity name.
    /// </summary>
    public string EntityName { get; }

    /// <summary>
    /// Gets the entity ID.
    /// </summary>
    public object EntityId { get; }
}

/// <summary>
/// Exception thrown when a business rule is violated.
/// </summary>
public class BusinessRuleException : DomainException
{
    /// <summary>
    /// Initializes a new instance of the <see cref="BusinessRuleException"/> class.
    /// </summary>
    public BusinessRuleException(string rule, string message)
        : base(message)
    {
        this.Rule = rule;
    }

    /// <summary>
    /// Gets the rule that was violated.
    /// </summary>
    public string Rule { get; }
}
```

---

## Enumerations

Use enumeration classes for domain concepts with behavior.

```csharp
// <copyright file="OrderStatus.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Domain.Enums;

/// <summary>
/// Represents the status of an order.
/// </summary>
public enum OrderStatus
{
    /// <summary>
    /// Order is being created.
    /// </summary>
    Draft = 0,

    /// <summary>
    /// Order has been submitted.
    /// </summary>
    Submitted = 1,

    /// <summary>
    /// Order is being processed.
    /// </summary>
    Processing = 2,

    /// <summary>
    /// Order has been shipped.
    /// </summary>
    Shipped = 3,

    /// <summary>
    /// Order has been delivered.
    /// </summary>
    Delivered = 4,

    /// <summary>
    /// Order has been cancelled.
    /// </summary>
    Cancelled = 5,
}
```

---

## Summary: When to Use What

| Concept | Use When |
|---------|----------|
| **Entity** | Object has unique identity that persists |
| **Value Object** | Object defined by its properties, immutable |
| **Aggregate** | Group of entities treated as a unit |
| **Domain Event** | Something significant happened that other parts need to know |
| **Domain Exception** | Business rule violation |
| **Strongly-Typed ID** | Avoid primitive obsession, type safety |

---

## Related Standards

- [Clean Architecture](clean-architecture.md) - Layer organization
- [CQRS & MediatR](cqrs-mediatr.md) - Command/Query handlers
- [Error Handling](error-handling.md) - Exception patterns
