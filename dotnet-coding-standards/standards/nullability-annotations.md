# Nullability & Annotations Standards

Standards for null safety and input validation using JetBrains.Annotations alongside .NET's native nullable reference types.

---

## Overview

### Why Both NRT and JetBrains.Annotations?

| Feature | Native C# NRT | JetBrains.Annotations |
|:--------|:--------------|:----------------------|
| Compile-time null checks | Yes | Yes (enhanced) |
| IDE integration | Basic | Full (Rider/ReSharper) |
| Contract annotations | No | `[NotNull]`, `[CanBeNull]` |
| Pure method marking | No | `[Pure]` |
| String format validation | No | `[StringFormatMethod]` |
| Regex validation | No | `[RegexPattern]` |
| Value range constraints | No | `[NonNegativeValue]`, `[ValueRange]` |
| Assertion methods | No | `[AssertionMethod]`, `[ContractAnnotation]` |
| Collection item nullability | Limited | `[ItemNotNull]`, `[ItemCanBeNull]` |

### When to Use Each

```
Is the nullability expressed by the type signature alone?
├── YES → Use native NRT (string vs string?)
└── NO → Does it need IDE enforcement or documentation?
    ├── YES → Add JetBrains.Annotations
    └── NO → Native NRT is sufficient
```

**Use JetBrains.Annotations when:**
- You need collection item nullability (`[ItemNotNull]`)
- You want to mark pure methods (`[Pure]`)
- You need value constraints (`[NonNegativeValue]`, `[ValueRange]`)
- You want contract flow analysis (`[ContractAnnotation]`)
- You need format string validation (`[StringFormatMethod]`)

---

## Installation & Configuration

### NuGet Package

```xml
<!-- Directory.Build.props -->
<PropertyGroup>
  <Nullable>enable</Nullable>
  <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
</PropertyGroup>

<ItemGroup>
  <PackageReference Include="JetBrains.Annotations" Version="2024.3.0" />
</ItemGroup>
```

### Using Directive

Add to `GlobalUsings.cs` or individual files:

```csharp
global using JetBrains.Annotations;
```

### IDE Configuration

**Rider/ReSharper:**
- Annotations are automatically recognized
- Enable "Code Annotations" inspections in Settings → Inspection Severity

**Visual Studio:**
- Install ReSharper for full support
- Without ReSharper, annotations serve as documentation only

---

## Nullability Annotations

### Parameter Nullability

```csharp
// ✅ CORRECT: Non-nullable parameter with annotation
public void Process([NotNull] string input)
{
    ArgumentNullException.ThrowIfNull(input);
    // Process input
}

// ✅ CORRECT: Nullable parameter with annotation
public void Process([CanBeNull] string? input)
{
    if (input is null)
    {
        return;
    }

    // Process input
}

// ❌ WRONG: Missing annotation on non-nullable
public void Process(string input)  // Add [NotNull]
{
    // ...
}
```

### Return Value Nullability

```csharp
// ✅ CORRECT: Never returns null
[NotNull]
public string GetDisplayName()
{
    return this.name ?? "Unknown";
}

// ✅ CORRECT: May return null
[CanBeNull]
public Order? FindById(OrderId id)
{
    return this.orders.FirstOrDefault(o => o.Id == id);
}

// ✅ CORRECT: Contract annotation for Try pattern
[ContractAnnotation("=> true, result: notnull; => false, result: null")]
public bool TryGetOrder(OrderId id, [CanBeNull] out Order? result)
{
    result = this.orders.FirstOrDefault(o => o.Id == id);
    return result is not null;
}
```

### Collection Item Nullability

```csharp
// ✅ CORRECT: Collection items are never null
public void ProcessItems([ItemNotNull] IEnumerable<string> items)
{
    foreach (var item in items)
    {
        // item is guaranteed non-null
        Console.WriteLine(item.Length);
    }
}

// ✅ CORRECT: Collection items may be null
public void ProcessItems([ItemCanBeNull] IEnumerable<string?> items)
{
    foreach (var item in items)
    {
        if (item is not null)
        {
            Console.WriteLine(item.Length);
        }
    }
}

// ✅ CORRECT: Return collection with non-null items
[ItemNotNull]
public IReadOnlyList<Order> GetActiveOrders()
{
    return this.orders.Where(o => o.IsActive).ToList();
}
```

### Conditional Nullability

```csharp
// ✅ CORRECT: Output is non-null when method returns true
public bool TryGet(
    string key,
    [System.Diagnostics.CodeAnalysis.NotNullWhen(true)] out string? value)
{
    return this.dictionary.TryGetValue(key, out value);
}

// ✅ CORRECT: Input is null when method returns false
public bool IsValid(
    [System.Diagnostics.CodeAnalysis.NotNullWhen(true)] string? input)
{
    return !string.IsNullOrWhiteSpace(input);
}
```

---

## Input Validation Annotations

### String Validation

```csharp
// ✅ CORRECT: String must not be null or empty
public void SetName([NotNull] string name)
{
    ArgumentException.ThrowIfNullOrEmpty(name);
    this.name = name;
}

// ✅ CORRECT: String must not be whitespace
public void SetDescription([NotNull] string description)
{
    ArgumentException.ThrowIfNullOrWhiteSpace(description);
    this.description = description;
}
```

### Numeric Validation

```csharp
// ✅ CORRECT: Value must be non-negative
public void SetQuantity([NonNegativeValue] int quantity)
{
    ArgumentOutOfRangeException.ThrowIfNegative(quantity);
    this.quantity = quantity;
}

// ✅ CORRECT: Value must be positive
public void SetPrice([NonNegativeValue] decimal price)
{
    ArgumentOutOfRangeException.ThrowIfNegativeOrZero(price);
    this.price = price;
}

// ✅ CORRECT: Value must be within range
public void SetPercentage([ValueRange(0, 100)] int percentage)
{
    ArgumentOutOfRangeException.ThrowIfLessThan(percentage, 0);
    ArgumentOutOfRangeException.ThrowIfGreaterThan(percentage, 100);
    this.percentage = percentage;
}
```

### Pattern Validation

```csharp
// ✅ CORRECT: Parameter is a regex pattern
public void SetPattern([RegexPattern] string pattern)
{
    ArgumentNullException.ThrowIfNull(pattern);
    this.regex = new Regex(pattern, RegexOptions.Compiled);
}

// ✅ CORRECT: Format string method
[StringFormatMethod("format")]
public void Log(string format, params object[] args)
{
    var message = string.Format(format, args);
    this.logger.LogInformation(message);
}

// ✅ CORRECT: Structured logging (Serilog-style)
[StructuredMessageTemplate]
public void LogEvent(string messageTemplate, params object[] args)
{
    this.logger.LogInformation(messageTemplate, args);
}
```

---

## Contract Annotations

### Pure Methods

Mark methods that have no side effects and return the same result for the same inputs:

```csharp
// ✅ CORRECT: Pure calculation method
[Pure]
public decimal CalculateTotal(decimal price, int quantity)
{
    return price * quantity;
}

// ✅ CORRECT: Pure factory method
[Pure]
public static Money FromDecimal(decimal amount, Currency currency)
{
    return new Money(amount, currency);
}

// ❌ WRONG: Method has side effects
[Pure]  // Don't use - this modifies state
public void UpdateBalance(decimal amount)
{
    this.balance += amount;
}
```

### Must Use Return Value

Enforce that callers use the return value:

```csharp
// ✅ CORRECT: Result must be checked
[MustUseReturnValue("Result contains success/failure status")]
public Result<Order> CreateOrder(CreateOrderCommand command)
{
    // Validation...
    if (command.Items.Count == 0)
    {
        return Result<Order>.Failure(OrderErrors.NoItems);
    }

    var order = new Order(command);
    return Result<Order>.Success(order);
}

// ✅ CORRECT: Builder pattern - must use result
[MustUseReturnValue]
public OrderBuilder WithCustomer(CustomerId customerId)
{
    return new OrderBuilder(this) { CustomerId = customerId };
}
```

### Contract Annotations for Flow Analysis

```csharp
// ✅ CORRECT: Method halts execution if value is null
[ContractAnnotation("value:null => halt")]
public static void ThrowIfNull([CanBeNull] object? value, string paramName)
{
    if (value is null)
    {
        throw new ArgumentNullException(paramName);
    }
}

// ✅ CORRECT: Return value depends on input nullability
[ContractAnnotation("value:null => false; value:notnull => true")]
public static bool HasValue([CanBeNull] string? value)
{
    return !string.IsNullOrEmpty(value);
}

// ✅ CORRECT: Method never returns normally
[ContractAnnotation("=> halt")]
public static void ThrowNotSupported()
{
    throw new NotSupportedException();
}

// ✅ CORRECT: Null input produces null output
[ContractAnnotation("input:null => null; input:notnull => notnull")]
[CanBeNull]
public static string? Normalize([CanBeNull] string? input)
{
    return input?.Trim().ToLowerInvariant();
}
```

### Assertion Methods

```csharp
// ✅ CORRECT: Custom assertion method
[AssertionMethod]
public static void Assert(
    [AssertionCondition(AssertionConditionType.IS_TRUE)] bool condition,
    string message)
{
    if (!condition)
    {
        throw new InvalidOperationException(message);
    }
}

// ✅ CORRECT: Null assertion
[AssertionMethod]
public static void AssertNotNull<T>(
    [AssertionCondition(AssertionConditionType.IS_NOT_NULL)] [CanBeNull] T? value,
    string paramName)
    where T : class
{
    if (value is null)
    {
        throw new ArgumentNullException(paramName);
    }
}
```

---

## Validation Patterns by Layer

### Domain Layer

```csharp
// Entity with validation
public sealed class Order : AggregateRoot<OrderId>
{
    private readonly List<OrderLine> lines = new();

    private Order(
        OrderId id,
        [NotNull] CustomerId customerId,
        [ItemNotNull] IEnumerable<OrderLine> lines)
        : base(id)
    {
        ArgumentNullException.ThrowIfNull(customerId);
        ArgumentNullException.ThrowIfNull(lines);

        this.CustomerId = customerId;
        this.lines.AddRange(lines);

        if (this.lines.Count == 0)
        {
            throw new DomainException("Order must have at least one line");
        }
    }

    [NotNull]
    public CustomerId CustomerId { get; }

    [ItemNotNull]
    public IReadOnlyList<OrderLine> Lines => this.lines.AsReadOnly();

    [Pure]
    public decimal CalculateTotal()
    {
        return this.lines.Sum(l => l.Total);
    }
}

// Value Object with validation
public sealed class Money : ValueObject
{
    public Money([NonNegativeValue] decimal amount, [NotNull] Currency currency)
    {
        ArgumentOutOfRangeException.ThrowIfNegative(amount);
        ArgumentNullException.ThrowIfNull(currency);

        this.Amount = amount;
        this.Currency = currency;
    }

    [NonNegativeValue]
    public decimal Amount { get; }

    [NotNull]
    public Currency Currency { get; }

    [Pure]
    [MustUseReturnValue]
    public Money Add([NotNull] Money other)
    {
        if (this.Currency != other.Currency)
        {
            throw new DomainException("Cannot add money with different currencies");
        }

        return new Money(this.Amount + other.Amount, this.Currency);
    }

    protected override IEnumerable<object> GetEqualityComponents()
    {
        yield return this.Amount;
        yield return this.Currency;
    }
}
```

### Application Layer

```csharp
// Command with annotations
public sealed record CreateOrderCommand(
    [property: NotNull] CustomerId CustomerId,
    [property: ItemNotNull] IReadOnlyList<OrderLineDto> Lines) : IRequest<Result<OrderId>>;

// Handler with validation
public sealed class CreateOrderCommandHandler
    : IRequestHandler<CreateOrderCommand, Result<OrderId>>
{
    private readonly IOrderRepository repository;
    private readonly IUnitOfWork unitOfWork;

    public CreateOrderCommandHandler(
        [NotNull] IOrderRepository repository,
        [NotNull] IUnitOfWork unitOfWork)
    {
        ArgumentNullException.ThrowIfNull(repository);
        ArgumentNullException.ThrowIfNull(unitOfWork);

        this.repository = repository;
        this.unitOfWork = unitOfWork;
    }

    [MustUseReturnValue]
    public async Task<Result<OrderId>> Handle(
        [NotNull] CreateOrderCommand request,
        CancellationToken cancellationToken)
    {
        ArgumentNullException.ThrowIfNull(request);

        // Validation via FluentValidation in pipeline
        var order = Order.Create(request.CustomerId, request.Lines);

        await this.repository.AddAsync(order, cancellationToken).ConfigureAwait(false);
        await this.unitOfWork.SaveChangesAsync(cancellationToken).ConfigureAwait(false);

        return Result<OrderId>.Success(order.Id);
    }
}
```

### Infrastructure Layer

```csharp
// Repository with annotations
public sealed class OrderRepository : IOrderRepository
{
    private readonly AppDbContext context;

    public OrderRepository([NotNull] AppDbContext context)
    {
        ArgumentNullException.ThrowIfNull(context);
        this.context = context;
    }

    [CanBeNull]
    public async Task<Order?> GetByIdAsync(
        [NotNull] OrderId id,
        CancellationToken cancellationToken = default)
    {
        ArgumentNullException.ThrowIfNull(id);

        return await this.context.Orders
            .Include(o => o.Lines)
            .FirstOrDefaultAsync(o => o.Id == id, cancellationToken)
            .ConfigureAwait(false);
    }

    public async Task AddAsync(
        [NotNull] Order order,
        CancellationToken cancellationToken = default)
    {
        ArgumentNullException.ThrowIfNull(order);

        await this.context.Orders
            .AddAsync(order, cancellationToken)
            .ConfigureAwait(false);
    }
}
```

### Presentation Layer

```csharp
// API model with annotations
public sealed class CreateOrderRequest
{
    [NotNull]
    [Required]
    public string CustomerId { get; init; } = string.Empty;

    [ItemNotNull]
    [Required]
    [MinLength(1)]
    public List<OrderLineRequest> Lines { get; init; } = new();
}

// Controller with annotations
[ApiController]
[Route("api/[controller]")]
public sealed class OrdersController : ControllerBase
{
    private readonly ISender sender;

    public OrdersController([NotNull] ISender sender)
    {
        ArgumentNullException.ThrowIfNull(sender);
        this.sender = sender;
    }

    [HttpPost]
    [ProducesResponseType(typeof(OrderResponse), StatusCodes.Status201Created)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> Create(
        [FromBody] [NotNull] CreateOrderRequest request,
        CancellationToken cancellationToken)
    {
        var command = request.ToCommand();
        var result = await this.sender.Send(command, cancellationToken);

        return result.IsSuccess
            ? CreatedAtAction(nameof(GetById), new { id = result.Value }, result.Value)
            : BadRequest(result.ToProblemDetails());
    }
}
```

---

## Options Validation

```csharp
// Options class with annotations
public sealed class ApiClientOptions
{
    public const string SectionName = "ApiClient";

    [NotNull]
    [Required]
    [Url]
    public string BaseUrl { get; set; } = string.Empty;

    [NonNegativeValue]
    [Range(1, 300)]
    public int TimeoutSeconds { get; set; } = 30;

    [NonNegativeValue]
    [Range(0, 10)]
    public int RetryCount { get; set; } = 3;

    [CanBeNull]
    public string? ApiKey { get; set; }
}

// Registration with validation
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddApiClient(
        this IServiceCollection services,
        [NotNull] IConfiguration configuration)
    {
        ArgumentNullException.ThrowIfNull(configuration);

        services.AddOptions<ApiClientOptions>()
            .Bind(configuration.GetSection(ApiClientOptions.SectionName))
            .ValidateDataAnnotations()
            .ValidateOnStart();

        return services;
    }
}
```

---

## FluentValidation Integration

```csharp
// Validator with comprehensive rules
public sealed class CreateOrderCommandValidator
    : AbstractValidator<CreateOrderCommand>
{
    public CreateOrderCommandValidator()
    {
        RuleFor(x => x.CustomerId)
            .NotNull()
            .WithMessage("Customer ID is required");

        RuleFor(x => x.Lines)
            .NotNull()
            .NotEmpty()
            .WithMessage("Order must have at least one line");

        RuleForEach(x => x.Lines)
            .NotNull()
            .SetValidator(new OrderLineDtoValidator());
    }
}

public sealed class OrderLineDtoValidator : AbstractValidator<OrderLineDto>
{
    public OrderLineDtoValidator()
    {
        RuleFor(x => x.ProductId)
            .NotNull()
            .NotEmpty();

        RuleFor(x => x.Quantity)
            .GreaterThan(0)
            .WithMessage("Quantity must be positive");

        RuleFor(x => x.UnitPrice)
            .GreaterThanOrEqualTo(0)
            .WithMessage("Unit price cannot be negative");
    }
}

// Pipeline behavior for validation
public sealed class ValidationBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> validators;

    public ValidationBehavior(
        [ItemNotNull] IEnumerable<IValidator<TRequest>> validators)
    {
        this.validators = validators;
    }

    public async Task<TResponse> Handle(
        [NotNull] TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        if (!this.validators.Any())
        {
            return await next().ConfigureAwait(false);
        }

        var context = new ValidationContext<TRequest>(request);

        var failures = this.validators
            .Select(v => v.Validate(context))
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

---

## Common Patterns

### Guard Clause Pattern

```csharp
// ✅ CORRECT: Full guard clause with annotation
public sealed class OrderService
{
    private readonly IOrderRepository repository;
    private readonly ILogger<OrderService> logger;

    public OrderService(
        [NotNull] IOrderRepository repository,
        [NotNull] ILogger<OrderService> logger)
    {
        ArgumentNullException.ThrowIfNull(repository);
        ArgumentNullException.ThrowIfNull(logger);

        this.repository = repository;
        this.logger = logger;
    }
}

// ❌ WRONG: Missing guard clause
public sealed class OrderService
{
    public OrderService([NotNull] IOrderRepository repository)
    {
        // Missing: ArgumentNullException.ThrowIfNull(repository);
        this.repository = repository;
    }
}
```

### Factory Method Pattern

```csharp
// ✅ CORRECT: Factory with validation
public sealed class Order
{
    private Order() { }

    [MustUseReturnValue]
    public static Result<Order> Create(
        [NotNull] CustomerId customerId,
        [ItemNotNull] IReadOnlyList<OrderLineDto> lines)
    {
        ArgumentNullException.ThrowIfNull(customerId);
        ArgumentNullException.ThrowIfNull(lines);

        if (lines.Count == 0)
        {
            return Result<Order>.Failure(OrderErrors.NoItems);
        }

        var order = new Order
        {
            Id = OrderId.New(),
            CustomerId = customerId,
        };

        foreach (var line in lines)
        {
            order.AddLine(line);
        }

        return Result<Order>.Success(order);
    }
}
```

---

## Anti-Patterns

### Overusing Null Suppression

```csharp
// ❌ WRONG: Excessive use of null suppression
public class BadExample
{
    private readonly IService service = null!;  // Dangerous

    public string Process()
    {
        return this.service!.GetValue()!.ToString()!;  // Multiple suppressions
    }
}

// ✅ CORRECT: Proper null handling
public class GoodExample
{
    private readonly IService service;

    public GoodExample([NotNull] IService service)
    {
        ArgumentNullException.ThrowIfNull(service);
        this.service = service;
    }

    [CanBeNull]
    public string? Process()
    {
        var value = this.service.GetValue();
        return value?.ToString();
    }
}
```

### Ignoring Nullability Warnings

```csharp
// ❌ WRONG: Pragma to suppress nullability
#pragma warning disable CS8600
string name = GetName();  // Possible null
#pragma warning restore CS8600

// ✅ CORRECT: Handle nullability properly
string? name = GetName();
if (name is null)
{
    throw new InvalidOperationException("Name is required");
}
```

### Inconsistent Annotation Usage

```csharp
// ❌ WRONG: Mixing styles inconsistently
public class InconsistentExample
{
    public void Method1([NotNull] string input) { }
    public void Method2(string input) { }  // Missing annotation
    public void Method3([CanBeNull] string? input) { }
}

// ✅ CORRECT: Consistent annotation usage
public class ConsistentExample
{
    public void Method1([NotNull] string input) { }
    public void Method2([NotNull] string input) { }
    public void Method3([CanBeNull] string? input) { }
}
```

---

## Quick Reference

### Annotation Cheat Sheet

| Annotation | Use Case | Example |
|:-----------|:---------|:--------|
| `[NotNull]` | Parameter/return cannot be null | `void Process([NotNull] string input)` |
| `[CanBeNull]` | Parameter/return may be null | `[CanBeNull] Order? Find(string id)` |
| `[ItemNotNull]` | Collection items are never null | `void Process([ItemNotNull] List<string> items)` |
| `[ItemCanBeNull]` | Collection items may be null | `void Process([ItemCanBeNull] List<string?> items)` |
| `[Pure]` | Method has no side effects | `[Pure] int Calculate(int x, int y)` |
| `[MustUseReturnValue]` | Return value must be used | `[MustUseReturnValue] Result<T> Create()` |
| `[NonNegativeValue]` | Numeric value >= 0 | `void SetAge([NonNegativeValue] int age)` |
| `[ValueRange(min, max)]` | Numeric value in range | `void SetPercent([ValueRange(0, 100)] int p)` |
| `[RegexPattern]` | String is a regex pattern | `void SetPattern([RegexPattern] string p)` |
| `[StringFormatMethod]` | Method uses format strings | `[StringFormatMethod("format")] void Log(...)` |
| `[ContractAnnotation]` | Flow analysis contract | `[ContractAnnotation("null => halt")]` |

---

## Checklist

Before submitting code, verify:

- [ ] All non-nullable parameters have `[NotNull]` annotation
- [ ] All nullable parameters have `[CanBeNull]` and `?` suffix
- [ ] All nullable returns have `[CanBeNull]` annotation
- [ ] Collection parameters specify item nullability
- [ ] Pure methods are marked with `[Pure]`
- [ ] Result-returning methods use `[MustUseReturnValue]`
- [ ] Numeric parameters have appropriate range annotations
- [ ] Guard clauses match parameter annotations
- [ ] No `#pragma warning disable` for nullability
- [ ] No excessive use of `!` null suppression operator

---

## Related Standards

- [Error Handling](error-handling.md) - Result pattern as alternative to null returns
- [Domain Design](domain-design.md) - Entity and Value Object validation
- [Dependency Injection](dependency-injection.md) - Constructor parameter validation
- [Documentation](documentation.md) - XML docs for nullable parameters
