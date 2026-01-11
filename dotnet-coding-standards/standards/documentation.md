# Documentation Standards

All projects require comprehensive documentation at multiple levels.

## Documentation Levels

| Level | Location | Purpose |
|-------|----------|---------|
| Solution | README.md | Project overview, setup, usage |
| Code | XML comments | API documentation, IntelliSense |
| Architecture | docs/ folder | Design decisions, diagrams |

## README Requirements

Every solution must have a README.md with ALL sections from the [template](../templates/README-template.md).

### Required Sections

1. **Overview** — What the project does
2. **Prerequisites** — Required tools and versions
3. **Getting Started** — Installation and quick start
4. **Configuration** — Environment variables, appsettings
5. **Architecture** — Project structure, design decisions
6. **Logging** — Serilog configuration, viewing logs
7. **Error Tracking** — Sentry setup, dashboard links
8. **API Reference** — Endpoints (for APIs)
9. **Testing** — Running tests, coverage requirements
10. **Deployment** — Build and deploy instructions
11. **Contributing** — Code standards, PR process
12. **Troubleshooting** — Common issues and solutions

## XML Documentation

### Classes

Every public class must have `<summary>`:

```csharp
/// <summary>
/// Provides endpoints for managing customer orders.
/// </summary>
/// <remarks>
/// This controller handles CRUD operations for orders.
/// All endpoints require authentication.
/// </remarks>
public class OrdersController : ControllerBase
```

### Methods

Every public method must have `<summary>`, `<param>`, `<returns>`, and `<exception>`:

```csharp
/// <summary>
/// Creates a new order for the specified customer.
/// </summary>
/// <param name="request">The order creation request.</param>
/// <param name="cancellationToken">The cancellation token.</param>
/// <returns>The created order details.</returns>
/// <exception cref="ArgumentNullException">
/// Thrown when <paramref name="request"/> is null.
/// </exception>
/// <exception cref="ValidationException">
/// Thrown when the request fails validation.
/// </exception>
public async Task<OrderResponse> CreateAsync(
    CreateOrderRequest request,
    CancellationToken cancellationToken)
```

### Properties

Every public property must have `<summary>`:

```csharp
/// <summary>
/// Gets or sets the order identifier.
/// </summary>
public string Id { get; set; } = string.Empty;

/// <summary>
/// Gets or sets the customer email address.
/// </summary>
/// <remarks>
/// Must be a valid email format.
/// </remarks>
[Required]
[EmailAddress]
public string CustomerEmail { get; set; } = string.Empty;
```

### Constructors

Every public constructor must have `<summary>` and `<param>`:

```csharp
/// <summary>
/// Initializes a new instance of the <see cref="OrderService"/> class.
/// </summary>
/// <param name="repository">The order repository.</param>
/// <param name="logger">The logger instance.</param>
/// <exception cref="ArgumentNullException">
/// Thrown when any parameter is null.
/// </exception>
public OrderService(IOrderRepository repository, ILogger<OrderService> logger)
{
    this.repository = repository ?? throw new ArgumentNullException(nameof(repository));
    this.logger = logger ?? throw new ArgumentNullException(nameof(logger));
}
```

### API Endpoints

Controllers must have `<response>` tags:

```csharp
/// <summary>
/// Gets an order by ID.
/// </summary>
/// <param name="id">The order ID.</param>
/// <param name="cancellationToken">The cancellation token.</param>
/// <returns>The order details.</returns>
/// <response code="200">Returns the order.</response>
/// <response code="404">Order not found.</response>
/// <response code="401">Unauthorized.</response>
[HttpGet("{id}")]
[ProducesResponseType(typeof(OrderResponse), StatusCodes.Status200OK)]
[ProducesResponseType(typeof(ErrorResponse), StatusCodes.Status404NotFound)]
[ProducesResponseType(typeof(ErrorResponse), StatusCodes.Status401Unauthorized)]
public async Task<IActionResult> GetByIdAsync(
    string id,
    CancellationToken cancellationToken)
```

### Interfaces

Document interfaces, not implementations:

```csharp
/// <summary>
/// Defines operations for managing orders.
/// </summary>
public interface IOrderService
{
    /// <summary>
    /// Gets an order by its unique identifier.
    /// </summary>
    /// <param name="id">The order ID.</param>
    /// <param name="cancellationToken">The cancellation token.</param>
    /// <returns>
    /// The order if found; otherwise <c>null</c>.
    /// </returns>
    Task<Order?> GetByIdAsync(string id, CancellationToken cancellationToken);
}

// Implementation uses <inheritdoc/>
/// <inheritdoc/>
public class OrderService : IOrderService
{
    /// <inheritdoc/>
    public async Task<Order?> GetByIdAsync(string id, CancellationToken cancellationToken)
    {
        // ...
    }
}
```

## XML Tag Reference

| Tag | Required | Usage |
|-----|----------|-------|
| `<summary>` | Yes | Brief description |
| `<remarks>` | Sometimes | Detailed explanation |
| `<param name="x">` | Yes | Parameter description |
| `<returns>` | Yes | Return value description |
| `<exception cref="T">` | Yes | Possible exceptions |
| `<response code="xxx">` | Yes (APIs) | HTTP response codes |
| `<see cref="T"/>` | No | Cross-reference |
| `<seealso cref="T"/>` | No | Related types |
| `<inheritdoc/>` | Sometimes | Inherit from interface/base |
| `<example>` | No | Usage examples |
| `<value>` | No | Property value description |
| `<c>` | No | Inline code |
| `<code>` | No | Code block |

## Common Patterns

### Nullable Returns

```csharp
/// <summary>
/// Finds an order by ID.
/// </summary>
/// <returns>
/// The order if found; otherwise <c>null</c>.
/// </returns>
Task<Order?> FindByIdAsync(string id);
```

### Async Methods

```csharp
/// <summary>
/// Processes the order asynchronously.
/// </summary>
/// <param name="cancellationToken">
/// A token to cancel the operation.
/// </param>
/// <returns>
/// A task representing the asynchronous operation.
/// </returns>
Task ProcessAsync(CancellationToken cancellationToken);
```

### Generic Types

```csharp
/// <summary>
/// Generic repository for entity access.
/// </summary>
/// <typeparam name="T">The entity type.</typeparam>
public interface IRepository<T> where T : class
{
    /// <summary>
    /// Gets all entities of type <typeparamref name="T"/>.
    /// </summary>
    Task<IReadOnlyList<T>> GetAllAsync();
}
```

### Collections

```csharp
/// <summary>
/// Gets all active orders.
/// </summary>
/// <returns>
/// A read-only list of active orders. Returns an empty list if none found.
/// </returns>
Task<IReadOnlyList<Order>> GetActiveOrdersAsync();
```

## API Documentation (Swagger)

### Controller-Level

```csharp
/// <summary>
/// Manages customer orders.
/// </summary>
/// <remarks>
/// Provides endpoints for creating, reading, updating, and deleting orders.
/// All endpoints require API key authentication.
/// </remarks>
[ApiController]
[Route("api/v1/[controller]")]
[Produces("application/json")]
[Tags("Orders")]
public class OrdersController : ControllerBase
```

### Request Models

```csharp
/// <summary>
/// Request to create a new order.
/// </summary>
public class CreateOrderRequest
{
    /// <summary>
    /// The customer ID placing the order.
    /// </summary>
    /// <example>cust-12345</example>
    [Required]
    public string CustomerId { get; set; } = string.Empty;

    /// <summary>
    /// The items to include in the order.
    /// </summary>
    /// <example>[{"productId": "prod-1", "quantity": 2}]</example>
    [Required]
    [MinLength(1)]
    public List<OrderItemRequest> Items { get; set; } = new();
}
```

## Verification

```bash
# Build with documentation warnings
dotnet build

# Look for SA1600 (missing documentation)
dotnet build 2>&1 | grep "SA1600"

# Generate XML documentation
# (automatically enabled in Directory.Build.props)
```
