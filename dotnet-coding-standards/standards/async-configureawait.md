# Async & ConfigureAwait Standards

Guidelines for async/await usage and `ConfigureAwait(false)` across .NET projects.

## ConfigureAwait Rules by Layer

| Layer | Use ConfigureAwait(false)? | Reason |
|-------|---------------------------|--------|
| **Domain** | ✅ Yes | No UI/sync context needed |
| **Application** | ✅ Yes | Business logic, no UI dependency |
| **Infrastructure** | ✅ Yes | External calls, database, HTTP |
| **API Controllers** | ❌ No | Need HttpContext |
| **Blazor Components** | ❌ No | Need UI sync context |
| **Worker ExecuteAsync** | ❌ No | Main loop only |
| **Worker Helper Methods** | ✅ Yes | Private methods |

## Decision Flowchart

```
Is this code in a Controller, Razor Page, or Blazor component?
├── YES → Do NOT use ConfigureAwait(false)
└── NO → Does this code access HttpContext, User, or UI elements?
    ├── YES → Do NOT use ConfigureAwait(false)
    └── NO → USE ConfigureAwait(false) ✓
```

## Examples by Layer

### Infrastructure Layer (ALWAYS use ConfigureAwait(false))

```csharp
// <copyright file="OrderRepository.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Infrastructure.Persistence;

/// <summary>
/// Repository for order data access.
/// </summary>
public class OrderRepository : IOrderRepository
{
    private readonly DbContext context;

    /// <inheritdoc/>
    public async Task<Order?> GetByIdAsync(string id, CancellationToken cancellationToken)
    {
        // ✅ Always use ConfigureAwait(false) in infrastructure
        return await this.context.Orders
            .FirstOrDefaultAsync(o => o.Id == id, cancellationToken)
            .ConfigureAwait(false);
    }

    /// <inheritdoc/>
    public async Task<IReadOnlyList<Order>> GetAllAsync(CancellationToken cancellationToken)
    {
        // ✅ ConfigureAwait(false) on every await
        return await this.context.Orders
            .ToListAsync(cancellationToken)
            .ConfigureAwait(false);
    }

    /// <inheritdoc/>
    public async Task CreateAsync(Order order, CancellationToken cancellationToken)
    {
        await this.context.Orders
            .AddAsync(order, cancellationToken)
            .ConfigureAwait(false);

        await this.context
            .SaveChangesAsync(cancellationToken)
            .ConfigureAwait(false);
    }
}
```

### Application Layer (ALWAYS use ConfigureAwait(false))

```csharp
// <copyright file="OrderService.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Application.Services;

/// <summary>
/// Service for order operations.
/// </summary>
public class OrderService : IOrderService
{
    private readonly IOrderRepository repository;
    private readonly IPaymentService paymentService;
    private readonly ILogger<OrderService> logger;

    /// <inheritdoc/>
    public async Task<OrderResult> ProcessOrderAsync(
        OrderRequest request,
        CancellationToken cancellationToken)
    {
        this.logger.LogInformation("Processing order {OrderId}", request.Id);

        // ✅ ConfigureAwait(false) in application layer
        var order = await this.repository
            .GetByIdAsync(request.Id, cancellationToken)
            .ConfigureAwait(false);

        if (order is null)
        {
            throw new OrderNotFoundException(request.Id);
        }

        // ✅ ConfigureAwait(false) on all awaits
        var paymentResult = await this.paymentService
            .ProcessPaymentAsync(order.Total, cancellationToken)
            .ConfigureAwait(false);

        order.MarkAsPaid(paymentResult.TransactionId);

        await this.repository
            .UpdateAsync(order, cancellationToken)
            .ConfigureAwait(false);

        return new OrderResult(order);
    }
}
```

### API Controllers (DO NOT use ConfigureAwait(false))

```csharp
// <copyright file="OrdersController.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Api.Controllers.V1;

/// <summary>
/// Controller for order operations.
/// </summary>
[ApiController]
[Route("api/v1/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly IOrderService orderService;

    /// <summary>
    /// Gets an order by ID.
    /// </summary>
    [HttpGet("{id}")]
    public async Task<IActionResult> GetByIdAsync(
        string id,
        CancellationToken cancellationToken)
    {
        // ❌ NO ConfigureAwait(false) - need HttpContext
        var order = await this.orderService.GetByIdAsync(id, cancellationToken);

        if (order is null)
        {
            return this.NotFound();
        }

        // Can access HttpContext here
        var userId = this.User.FindFirst("sub")?.Value;
        
        return this.Ok(order);
    }

    /// <summary>
    /// Creates a new order.
    /// </summary>
    [HttpPost]
    public async Task<IActionResult> CreateAsync(
        [FromBody] CreateOrderRequest request,
        CancellationToken cancellationToken)
    {
        // ❌ NO ConfigureAwait(false) in controllers
        var result = await this.orderService.CreateAsync(request, cancellationToken);

        return this.CreatedAtAction(
            nameof(this.GetByIdAsync),
            new { id = result.Id },
            result);
    }
}
```

### Worker Services (Mixed Usage)

```csharp
// <copyright file="OrderProcessorWorker.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Worker.Workers;

/// <summary>
/// Background worker for processing orders.
/// </summary>
public class OrderProcessorWorker : BackgroundService
{
    private readonly IServiceScopeFactory scopeFactory;
    private readonly ILogger<OrderProcessorWorker> logger;

    /// <inheritdoc/>
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        this.logger.LogInformation("Worker starting");

        while (!stoppingToken.IsCancellationRequested)
        {
            // ❌ NO ConfigureAwait(false) in main ExecuteAsync loop
            await this.ProcessBatchAsync(stoppingToken);
            await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);
        }

        this.logger.LogInformation("Worker stopped");
    }

    private async Task ProcessBatchAsync(CancellationToken cancellationToken)
    {
        await using var scope = this.scopeFactory.CreateAsyncScope();
        var service = scope.ServiceProvider.GetRequiredService<IOrderService>();

        // ✅ ConfigureAwait(false) in private helper methods
        var pendingOrders = await this.GetPendingOrdersAsync(cancellationToken)
            .ConfigureAwait(false);

        foreach (var order in pendingOrders)
        {
            // ✅ ConfigureAwait(false) in helper methods
            await service
                .ProcessOrderAsync(order, cancellationToken)
                .ConfigureAwait(false);
        }
    }

    private async Task<IReadOnlyList<Order>> GetPendingOrdersAsync(
        CancellationToken cancellationToken)
    {
        // ✅ ConfigureAwait(false) in private methods
        await Task.Delay(1, cancellationToken).ConfigureAwait(false);
        return Array.Empty<Order>();
    }
}
```

### Blazor Components (DO NOT use ConfigureAwait(false))

```csharp
// <copyright file="OrderList.razor.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.Dashboard.Web.Pages;

/// <summary>
/// Order list page component.
/// </summary>
public partial class OrderList : ComponentBase
{
    [Inject]
    private IOrderService OrderService { get; set; } = default!;

    private List<Order>? orders;

    /// <inheritdoc/>
    protected override async Task OnInitializedAsync()
    {
        // ❌ NO ConfigureAwait(false) in Blazor components
        // Need sync context to update UI
        this.orders = await this.OrderService.GetAllAsync(CancellationToken.None);
    }

    private async Task RefreshAsync()
    {
        // ❌ NO ConfigureAwait(false) - need to update component state
        this.orders = await this.OrderService.GetAllAsync(CancellationToken.None);
        
        // StateHasChanged is called automatically after event handlers
    }
}
```

## Async Naming Convention

All async methods MUST have the `Async` suffix:

```csharp
// ✅ Correct
public async Task<Order> GetOrderAsync(string id, CancellationToken cancellationToken)
public async Task ProcessAsync(CancellationToken cancellationToken)
public async Task<bool> ValidateAsync(Order order, CancellationToken cancellationToken)

// ❌ Wrong - missing Async suffix
public async Task<Order> GetOrder(string id, CancellationToken cancellationToken)
public async Task Process(CancellationToken cancellationToken)
```

## Cancellation Token Propagation

Always accept and propagate `CancellationToken`:

```csharp
// ✅ Correct - accepts and propagates token
public async Task<Order> ProcessAsync(string id, CancellationToken cancellationToken)
{
    var order = await this.repository
        .GetByIdAsync(id, cancellationToken)
        .ConfigureAwait(false);

    await this.validator
        .ValidateAsync(order, cancellationToken)
        .ConfigureAwait(false);

    return order;
}

// ❌ Wrong - doesn't accept token
public async Task<Order> ProcessAsync(string id)
{
    var order = await this.repository.GetByIdAsync(id);
    return order;
}

// ❌ Wrong - doesn't propagate token
public async Task<Order> ProcessAsync(string id, CancellationToken cancellationToken)
{
    var order = await this.repository.GetByIdAsync(id, CancellationToken.None);
    return order;
}
```

## Common Async Anti-Patterns

### ❌ Blocking on Async Code

```csharp
// ❌ NEVER do this - causes deadlocks
var result = service.GetAsync().Result;
var result = service.GetAsync().GetAwaiter().GetResult();
service.ProcessAsync().Wait();

// ✅ Always await
var result = await service.GetAsync();
await service.ProcessAsync();
```

### ❌ Async Void

```csharp
// ❌ NEVER use async void (except event handlers)
public async void ProcessOrder(Order order)
{
    await this.service.ProcessAsync(order);
}

// ✅ Use async Task
public async Task ProcessOrderAsync(Order order, CancellationToken cancellationToken)
{
    await this.service.ProcessAsync(order, cancellationToken);
}
```

### ❌ Unnecessary Task.Run

```csharp
// ❌ Don't wrap async in Task.Run
await Task.Run(async () =>
{
    await this.service.ProcessAsync();
});

// ✅ Just await directly
await this.service.ProcessAsync();
```

### ❌ Fire and Forget

```csharp
// ❌ Don't fire and forget (exceptions lost)
_ = this.service.ProcessAsync();
this.service.ProcessAsync(); // Warning suppressed

// ✅ Await or handle properly
await this.service.ProcessAsync();

// Or if intentional, handle exceptions
_ = Task.Run(async () =>
{
    try
    {
        await this.service.ProcessAsync();
    }
    catch (Exception ex)
    {
        this.logger.LogError(ex, "Background task failed");
    }
});
```

## Verification Commands

```bash
# Find missing ConfigureAwait(false) in Infrastructure
grep -r "await.*;" --include="*.cs" src/Infrastructure/ | grep -v "ConfigureAwait(false)"

# Find missing ConfigureAwait(false) in Application
grep -r "await.*;" --include="*.cs" src/Application/ | grep -v "ConfigureAwait(false)"

# Find ConfigureAwait(false) in Controllers (should be none)
grep -r "ConfigureAwait(false)" --include="*.cs" src/*/Controllers/

# Find async methods without Async suffix
grep -rE "public async Task.*\(" --include="*.cs" src/ | grep -v "Async("

# Find .Result or .Wait() usage
grep -rE "\.(Result|Wait\(\))" --include="*.cs" src/
```
