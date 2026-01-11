# Dependency Injection Standards

Dependency injection patterns and best practices for .NET projects.

## Service Lifetimes

| Lifetime | Created | Use When |
|----------|---------|----------|
| **Singleton** | Once per application | Stateless, thread-safe, expensive to create |
| **Scoped** | Once per request/scope | Database contexts, unit of work, request-specific |
| **Transient** | Every time requested | Lightweight, stateless, no shared state |

### Lifetime Decision Tree

```
Is the service stateless and thread-safe?
├── NO → Use Scoped or Transient
└── YES → Is it expensive to create?
    ├── YES → Use Singleton
    └── NO → Does it need to be shared across the request?
        ├── YES → Use Scoped
        └── NO → Use Transient
```

### Examples by Lifetime

```csharp
// SINGLETON - Created once, shared across all requests
// ✅ Good for: Configuration, caching, HTTP clients, loggers
builder.Services.AddSingleton<IConfigurationService, ConfigurationService>();
builder.Services.AddSingleton<ICacheService, MemoryCacheService>();

// SCOPED - Created once per request/scope
// ✅ Good for: DbContext, repositories, unit of work, user context
builder.Services.AddScoped<IOrderRepository, OrderRepository>();
builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();
builder.Services.AddScoped<ICurrentUserService, CurrentUserService>();

// TRANSIENT - Created every time requested
// ✅ Good for: Validators, mappers, lightweight stateless services
builder.Services.AddTransient<IOrderValidator, OrderValidator>();
builder.Services.AddTransient<IMapper, AutoMapperWrapper>();
```

---

## Registration Patterns

### Extension Methods per Layer

```csharp
// <copyright file="ApplicationServiceExtensions.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Application;

using Microsoft.Extensions.DependencyInjection;

/// <summary>
/// Extension methods for registering application layer services.
/// </summary>
public static class ApplicationServiceExtensions
{
    /// <summary>
    /// Adds application layer services to the container.
    /// </summary>
    /// <param name="services">The service collection.</param>
    /// <returns>The service collection for chaining.</returns>
    public static IServiceCollection AddApplicationServices(this IServiceCollection services)
    {
        // Services
        services.AddScoped<IOrderService, OrderService>();
        services.AddScoped<IPaymentService, PaymentService>();
        services.AddScoped<INotificationService, NotificationService>();

        // Validators
        services.AddTransient<IValidator<CreateOrderRequest>, CreateOrderValidator>();

        return services;
    }
}
```

```csharp
// <copyright file="InfrastructureServiceExtensions.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Infrastructure;

using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;

/// <summary>
/// Extension methods for registering infrastructure layer services.
/// </summary>
public static class InfrastructureServiceExtensions
{
    /// <summary>
    /// Adds infrastructure layer services to the container.
    /// </summary>
    /// <param name="services">The service collection.</param>
    /// <param name="configuration">The configuration.</param>
    /// <returns>The service collection for chaining.</returns>
    public static IServiceCollection AddInfrastructureServices(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        // Database
        services.AddDbContext<AppDbContext>(options =>
            options.UseSqlServer(configuration.GetConnectionString("Default")));

        // Repositories
        services.AddScoped<IOrderRepository, OrderRepository>();
        services.AddScoped<ICustomerRepository, CustomerRepository>();

        // External services
        services.AddHttpClient<IExternalApiClient, ExternalApiClient>(client =>
        {
            client.BaseAddress = new Uri(configuration["ExternalApi:BaseUrl"]!);
        });

        // Caching
        services.AddMemoryCache();
        services.AddSingleton<ICacheService, MemoryCacheService>();

        return services;
    }
}
```

### Program.cs Usage

```csharp
// Program.cs - Clean and organized
var builder = WebApplication.CreateBuilder(args);

// Add layers
builder.Services.AddApplicationServices();
builder.Services.AddInfrastructureServices(builder.Configuration);

// Add API services
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();
```

---

## Constructor Injection

### Standard Pattern

```csharp
// <copyright file="OrderService.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Application.Services;

/// <summary>
/// Service for managing orders.
/// </summary>
public class OrderService : IOrderService
{
    private readonly IOrderRepository repository;
    private readonly IPaymentService paymentService;
    private readonly ILogger<OrderService> logger;

    /// <summary>
    /// Initializes a new instance of the <see cref="OrderService"/> class.
    /// </summary>
    /// <param name="repository">The order repository.</param>
    /// <param name="paymentService">The payment service.</param>
    /// <param name="logger">The logger.</param>
    public OrderService(
        IOrderRepository repository,
        IPaymentService paymentService,
        ILogger<OrderService> logger)
    {
        this.repository = repository ?? throw new ArgumentNullException(nameof(repository));
        this.paymentService = paymentService ?? throw new ArgumentNullException(nameof(paymentService));
        this.logger = logger ?? throw new ArgumentNullException(nameof(logger));
    }

    /// <inheritdoc/>
    public async Task<Order> CreateAsync(
        CreateOrderRequest request,
        CancellationToken cancellationToken)
    {
        this.logger.LogInformation("Creating order for customer {CustomerId}", request.CustomerId);

        var order = new Order(request.CustomerId, request.Items);

        await this.repository
            .AddAsync(order, cancellationToken)
            .ConfigureAwait(false);

        return order;
    }
}
```

### Primary Constructors (C# 12+)

```csharp
// <copyright file="OrderService.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Application.Services;

/// <summary>
/// Service for managing orders.
/// </summary>
public class OrderService(
    IOrderRepository repository,
    IPaymentService paymentService,
    ILogger<OrderService> logger) : IOrderService
{
    /// <inheritdoc/>
    public async Task<Order> CreateAsync(
        CreateOrderRequest request,
        CancellationToken cancellationToken)
    {
        ArgumentNullException.ThrowIfNull(request);

        logger.LogInformation("Creating order for customer {CustomerId}", request.CustomerId);

        var order = new Order(request.CustomerId, request.Items);

        await repository
            .AddAsync(order, cancellationToken)
            .ConfigureAwait(false);

        return order;
    }
}
```

---

## Options Pattern

### Define Options Class

```csharp
// <copyright file="EmailOptions.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Infrastructure.Options;

/// <summary>
/// Configuration options for email service.
/// </summary>
public class EmailOptions
{
    /// <summary>
    /// The configuration section name.
    /// </summary>
    public const string SectionName = "Email";

    /// <summary>
    /// Gets or sets the SMTP server host.
    /// </summary>
    public string Host { get; set; } = string.Empty;

    /// <summary>
    /// Gets or sets the SMTP server port.
    /// </summary>
    public int Port { get; set; } = 587;

    /// <summary>
    /// Gets or sets the sender email address.
    /// </summary>
    public string FromAddress { get; set; } = string.Empty;

    /// <summary>
    /// Gets or sets the sender display name.
    /// </summary>
    public string FromName { get; set; } = string.Empty;

    /// <summary>
    /// Gets or sets whether to use SSL.
    /// </summary>
    public bool UseSsl { get; set; } = true;
}
```

### Register Options

```csharp
// Extension method
public static IServiceCollection AddInfrastructureServices(
    this IServiceCollection services,
    IConfiguration configuration)
{
    // Bind options from configuration
    services.Configure<EmailOptions>(
        configuration.GetSection(EmailOptions.SectionName));

    // With validation
    services.AddOptions<EmailOptions>()
        .Bind(configuration.GetSection(EmailOptions.SectionName))
        .ValidateDataAnnotations()
        .ValidateOnStart();

    return services;
}
```

### Consume Options

```csharp
// Using IOptions<T> - Singleton, read at startup
public class EmailService(IOptions<EmailOptions> options)
{
    private readonly EmailOptions options = options.Value;
}

// Using IOptionsSnapshot<T> - Scoped, refreshed per request
public class EmailService(IOptionsSnapshot<EmailOptions> options)
{
    private readonly EmailOptions options = options.Value;
}

// Using IOptionsMonitor<T> - Singleton, supports change notifications
public class EmailService(IOptionsMonitor<EmailOptions> options)
{
    public void Send()
    {
        var currentOptions = options.CurrentValue; // Always current
    }
}
```

---

## Factory Pattern

### For Runtime Resolution

```csharp
// <copyright file="IPaymentProcessorFactory.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Application.Interfaces;

/// <summary>
/// Factory for creating payment processors.
/// </summary>
public interface IPaymentProcessorFactory
{
    /// <summary>
    /// Creates a payment processor for the specified payment method.
    /// </summary>
    /// <param name="paymentMethod">The payment method.</param>
    /// <returns>The appropriate payment processor.</returns>
    IPaymentProcessor Create(PaymentMethod paymentMethod);
}
```

```csharp
// <copyright file="PaymentProcessorFactory.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Infrastructure.Payments;

/// <summary>
/// Factory for creating payment processors.
/// </summary>
public class PaymentProcessorFactory : IPaymentProcessorFactory
{
    private readonly IServiceProvider serviceProvider;

    /// <summary>
    /// Initializes a new instance of the <see cref="PaymentProcessorFactory"/> class.
    /// </summary>
    /// <param name="serviceProvider">The service provider.</param>
    public PaymentProcessorFactory(IServiceProvider serviceProvider)
    {
        this.serviceProvider = serviceProvider ?? throw new ArgumentNullException(nameof(serviceProvider));
    }

    /// <inheritdoc/>
    public IPaymentProcessor Create(PaymentMethod paymentMethod)
    {
        return paymentMethod switch
        {
            PaymentMethod.CreditCard => this.serviceProvider.GetRequiredService<CreditCardProcessor>(),
            PaymentMethod.PayPal => this.serviceProvider.GetRequiredService<PayPalProcessor>(),
            PaymentMethod.BankTransfer => this.serviceProvider.GetRequiredService<BankTransferProcessor>(),
            _ => throw new ArgumentOutOfRangeException(nameof(paymentMethod)),
        };
    }
}

// Registration
services.AddScoped<CreditCardProcessor>();
services.AddScoped<PayPalProcessor>();
services.AddScoped<BankTransferProcessor>();
services.AddSingleton<IPaymentProcessorFactory, PaymentProcessorFactory>();
```

### Keyed Services (.NET 8+)

```csharp
// Registration with keys
services.AddKeyedScoped<IPaymentProcessor, CreditCardProcessor>("CreditCard");
services.AddKeyedScoped<IPaymentProcessor, PayPalProcessor>("PayPal");
services.AddKeyedScoped<IPaymentProcessor, BankTransferProcessor>("BankTransfer");

// Injection
public class PaymentService(
    [FromKeyedServices("CreditCard")] IPaymentProcessor creditCard,
    [FromKeyedServices("PayPal")] IPaymentProcessor payPal)
{
}

// Runtime resolution
public class PaymentService(IServiceProvider serviceProvider)
{
    public IPaymentProcessor GetProcessor(string key)
    {
        return serviceProvider.GetRequiredKeyedService<IPaymentProcessor>(key);
    }
}
```

---

## Decorator Pattern

```csharp
// Register the decorator
services.AddScoped<IOrderService, OrderService>();
services.Decorate<IOrderService, CachedOrderService>();
services.Decorate<IOrderService, LoggingOrderService>();

// Decorator implementation
public class CachedOrderService : IOrderService
{
    private readonly IOrderService inner;
    private readonly IMemoryCache cache;

    public CachedOrderService(IOrderService inner, IMemoryCache cache)
    {
        this.inner = inner;
        this.cache = cache;
    }

    public async Task<Order?> GetByIdAsync(string id, CancellationToken cancellationToken)
    {
        return await this.cache.GetOrCreateAsync(
            $"order:{id}",
            async entry =>
            {
                entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5);
                return await this.inner.GetByIdAsync(id, cancellationToken);
            });
    }
}
```

---

## Anti-Patterns to Avoid

### ❌ Service Locator

```csharp
// ❌ WRONG - Service Locator pattern
public class OrderService
{
    private readonly IServiceProvider serviceProvider;

    public OrderService(IServiceProvider serviceProvider)
    {
        this.serviceProvider = serviceProvider;
    }

    public async Task ProcessAsync()
    {
        // Resolving at runtime - hides dependencies
        var repository = this.serviceProvider.GetRequiredService<IOrderRepository>();
        var payment = this.serviceProvider.GetRequiredService<IPaymentService>();
    }
}

// ✅ CORRECT - Constructor injection
public class OrderService
{
    private readonly IOrderRepository repository;
    private readonly IPaymentService paymentService;

    public OrderService(IOrderRepository repository, IPaymentService paymentService)
    {
        this.repository = repository;
        this.paymentService = paymentService;
    }
}
```

### ❌ Captive Dependencies

```csharp
// ❌ WRONG - Singleton holding Scoped dependency
// DbContext is Scoped, but CacheService is Singleton
public class CacheService : ICacheService // Singleton
{
    private readonly AppDbContext dbContext; // Scoped - WRONG!

    public CacheService(AppDbContext dbContext)
    {
        this.dbContext = dbContext; // This DbContext will never be disposed!
    }
}

// ✅ CORRECT - Use IServiceScopeFactory
public class CacheService : ICacheService
{
    private readonly IServiceScopeFactory scopeFactory;

    public CacheService(IServiceScopeFactory scopeFactory)
    {
        this.scopeFactory = scopeFactory;
    }

    public async Task<T> GetOrLoadAsync<T>(string key, Func<AppDbContext, Task<T>> loader)
    {
        using var scope = this.scopeFactory.CreateScope();
        var dbContext = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        return await loader(dbContext);
    }
}
```

### ❌ Ambient Context

```csharp
// ❌ WRONG - Static/ambient context
public class OrderService
{
    public async Task CreateAsync()
    {
        var user = HttpContext.Current.User; // Static access - bad!
        var config = ConfigurationManager.AppSettings["Key"]; // Static - bad!
    }
}

// ✅ CORRECT - Inject dependencies
public class OrderService
{
    private readonly ICurrentUserService userService;
    private readonly IOptions<AppSettings> settings;

    public OrderService(ICurrentUserService userService, IOptions<AppSettings> settings)
    {
        this.userService = userService;
        this.settings = settings;
    }
}
```

### ❌ Constructor Over-Injection

```csharp
// ❌ WRONG - Too many dependencies (code smell)
public class OrderService
{
    public OrderService(
        IOrderRepository orderRepo,
        ICustomerRepository customerRepo,
        IProductRepository productRepo,
        IPaymentService paymentService,
        IShippingService shippingService,
        INotificationService notificationService,
        IDiscountService discountService,
        ITaxService taxService,
        IInventoryService inventoryService,
        ILogger<OrderService> logger)
    {
        // 10+ dependencies = class does too much
    }
}

// ✅ CORRECT - Split into focused services
public class OrderService
{
    public OrderService(
        IOrderRepository repository,
        IOrderProcessor processor, // Encapsulates payment, shipping, inventory
        ILogger<OrderService> logger)
    {
    }
}
```

---

## Testing with DI

### Manual Construction in Tests

```csharp
public class OrderServiceTests
{
    private readonly Mock<IOrderRepository> repositoryMock;
    private readonly Mock<IPaymentService> paymentMock;
    private readonly Mock<ILogger<OrderService>> loggerMock;
    private readonly OrderService sut;

    public OrderServiceTests()
    {
        this.repositoryMock = new Mock<IOrderRepository>();
        this.paymentMock = new Mock<IPaymentService>();
        this.loggerMock = new Mock<ILogger<OrderService>>();

        this.sut = new OrderService(
            this.repositoryMock.Object,
            this.paymentMock.Object,
            this.loggerMock.Object);
    }
}
```

### Using WebApplicationFactory for Integration Tests

```csharp
public class OrdersControllerTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> factory;

    public OrdersControllerTests(WebApplicationFactory<Program> factory)
    {
        this.factory = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                // Replace real services with fakes
                services.RemoveAll<IPaymentService>();
                services.AddScoped<IPaymentService, FakePaymentService>();
            });
        });
    }

    [Fact]
    public async Task GetOrder_ReturnsOrder()
    {
        var client = this.factory.CreateClient();
        var response = await client.GetAsync("/api/v1/orders/123");
        response.EnsureSuccessStatusCode();
    }
}
```

---

## Checklist

- [ ] Services registered with appropriate lifetime
- [ ] Extension methods organize registrations by layer
- [ ] Options pattern used for configuration
- [ ] Constructor injection used (not service locator)
- [ ] No captive dependencies
- [ ] Factory pattern for runtime resolution
- [ ] Dependencies validated in constructors
- [ ] No ambient context / static access
