# Performance Guidelines

Performance best practices for .NET projects.

## Async/Await Performance

### Use Async for I/O-Bound Operations

```csharp
// ✅ CORRECT - Async for I/O
public async Task<Order> GetOrderAsync(string id, CancellationToken cancellationToken)
{
    return await this.context.Orders
        .FirstOrDefaultAsync(o => o.Id == id, cancellationToken)
        .ConfigureAwait(false);
}

// ❌ WRONG - Blocking call
public Order GetOrder(string id)
{
    return this.context.Orders.FirstOrDefault(o => o.Id == id);
}
```

### Don't Use Task.Run for I/O

```csharp
// ❌ WRONG - Wastes a thread pool thread
public async Task<Data> GetDataAsync()
{
    return await Task.Run(() => this.httpClient.GetAsync(url));
}

// ✅ CORRECT - Direct async call
public async Task<Data> GetDataAsync(CancellationToken cancellationToken)
{
    return await this.httpClient
        .GetAsync(url, cancellationToken)
        .ConfigureAwait(false);
}
```

### Use ValueTask for Hot Paths

```csharp
// ✅ CORRECT - ValueTask when method often completes synchronously
public ValueTask<CacheItem?> GetFromCacheAsync(string key)
{
    if (this.cache.TryGetValue(key, out var item))
    {
        return ValueTask.FromResult<CacheItem?>(item);
    }

    return new ValueTask<CacheItem?>(this.LoadFromSourceAsync(key));
}

// Regular Task is fine for methods that always do async work
public async Task<Data> LoadFromDatabaseAsync(string id, CancellationToken cancellationToken)
```

### Avoid Async Overhead in Tight Loops

```csharp
// ❌ WRONG - Async overhead per item
foreach (var item in items)
{
    await this.ProcessItemAsync(item, cancellationToken);
}

// ✅ BETTER - Batch when possible
var tasks = items.Select(item => this.ProcessItemAsync(item, cancellationToken));
await Task.WhenAll(tasks).ConfigureAwait(false);

// ✅ BEST - Use batched API
await this.ProcessItemsBatchAsync(items, cancellationToken).ConfigureAwait(false);
```

---

## Database Performance

### Use AsNoTracking for Read-Only Queries

```csharp
// ✅ CORRECT - No tracking for read-only
public async Task<IReadOnlyList<Order>> GetOrdersAsync(CancellationToken cancellationToken)
{
    return await this.context.Orders
        .AsNoTracking()
        .Where(o => o.Status == OrderStatus.Active)
        .ToListAsync(cancellationToken)
        .ConfigureAwait(false);
}

// ❌ WRONG - Tracking overhead when not needed
public async Task<IReadOnlyList<Order>> GetOrdersAsync(CancellationToken cancellationToken)
{
    return await this.context.Orders
        .Where(o => o.Status == OrderStatus.Active)
        .ToListAsync(cancellationToken)
        .ConfigureAwait(false);
}
```

### Use Split Queries for Multiple Includes

```csharp
// ❌ WRONG - Single query with cartesian explosion
var orders = await this.context.Orders
    .Include(o => o.Items)
    .Include(o => o.Payments)
    .Include(o => o.ShippingHistory)
    .ToListAsync(cancellationToken);

// ✅ CORRECT - Split into multiple queries
var orders = await this.context.Orders
    .Include(o => o.Items)
    .Include(o => o.Payments)
    .Include(o => o.ShippingHistory)
    .AsSplitQuery()
    .ToListAsync(cancellationToken)
    .ConfigureAwait(false);
```

### Select Only What You Need

```csharp
// ❌ WRONG - Fetches all columns
var orders = await this.context.Orders
    .Where(o => o.CustomerId == customerId)
    .ToListAsync(cancellationToken);

// ✅ CORRECT - Project to DTO
var orders = await this.context.Orders
    .AsNoTracking()
    .Where(o => o.CustomerId == customerId)
    .Select(o => new OrderSummaryDto
    {
        Id = o.Id,
        Status = o.Status,
        Total = o.Total,
        CreatedAt = o.CreatedAt,
    })
    .ToListAsync(cancellationToken)
    .ConfigureAwait(false);
```

### Avoid N+1 Queries

```csharp
// ❌ WRONG - N+1 problem
var customers = await this.context.Customers.ToListAsync(cancellationToken);

foreach (var customer in customers)
{
    // Each iteration hits the database!
    var orders = await this.context.Orders
        .Where(o => o.CustomerId == customer.Id)
        .ToListAsync(cancellationToken);
}

// ✅ CORRECT - Single query with Include
var customers = await this.context.Customers
    .Include(c => c.Orders)
    .ToListAsync(cancellationToken)
    .ConfigureAwait(false);

// ✅ ALSO CORRECT - Two queries with in-memory join
var customers = await this.context.Customers
    .AsNoTracking()
    .ToListAsync(cancellationToken)
    .ConfigureAwait(false);

var customerIds = customers.Select(c => c.Id).ToList();

var orders = await this.context.Orders
    .AsNoTracking()
    .Where(o => customerIds.Contains(o.CustomerId))
    .ToListAsync(cancellationToken)
    .ConfigureAwait(false);

var ordersByCustomer = orders.ToLookup(o => o.CustomerId);
```

### Use Compiled Queries for Hot Paths

```csharp
// Define once
private static readonly Func<AppDbContext, string, Task<Order?>> GetOrderByIdQuery =
    EF.CompileAsyncQuery((AppDbContext context, string id) =>
        context.Orders.FirstOrDefault(o => o.Id == id));

// Use many times
public async Task<Order?> GetOrderByIdAsync(string id)
{
    return await GetOrderByIdQuery(this.context, id).ConfigureAwait(false);
}
```

### Implement Pagination

```csharp
// ❌ WRONG - Loads everything into memory
var allOrders = await this.context.Orders.ToListAsync(cancellationToken);
return allOrders.Skip(page * pageSize).Take(pageSize);

// ✅ CORRECT - Database-level pagination
var orders = await this.context.Orders
    .AsNoTracking()
    .OrderByDescending(o => o.CreatedAt)
    .Skip((page - 1) * pageSize)
    .Take(pageSize)
    .ToListAsync(cancellationToken)
    .ConfigureAwait(false);
```

---

## HTTP Client Performance

### Use IHttpClientFactory

```csharp
// Program.cs
builder.Services.AddHttpClient<IExternalApiClient, ExternalApiClient>(client =>
{
    client.BaseAddress = new Uri("https://api.external.com");
    client.Timeout = TimeSpan.FromSeconds(30);
    client.DefaultRequestHeaders.Add("Accept", "application/json");
});

// Service
public class ExternalApiClient : IExternalApiClient
{
    private readonly HttpClient httpClient;

    public ExternalApiClient(HttpClient httpClient)
    {
        this.httpClient = httpClient;
    }

    public async Task<Data> GetDataAsync(CancellationToken cancellationToken)
    {
        var response = await this.httpClient
            .GetAsync("/data", cancellationToken)
            .ConfigureAwait(false);

        response.EnsureSuccessStatusCode();

        return await response.Content
            .ReadFromJsonAsync<Data>(cancellationToken: cancellationToken)
            .ConfigureAwait(false);
    }
}
```

### Configure Connection Pooling

```csharp
builder.Services.AddHttpClient<IExternalApiClient, ExternalApiClient>()
    .ConfigurePrimaryHttpMessageHandler(() => new SocketsHttpHandler
    {
        PooledConnectionLifetime = TimeSpan.FromMinutes(2),
        PooledConnectionIdleTimeout = TimeSpan.FromMinutes(1),
        MaxConnectionsPerServer = 100,
    });
```

### Use Streaming for Large Responses

```csharp
// ❌ WRONG - Loads entire response into memory
var json = await this.httpClient.GetStringAsync(url, cancellationToken);
var items = JsonSerializer.Deserialize<List<Item>>(json);

// ✅ CORRECT - Stream the response
using var response = await this.httpClient
    .GetAsync(url, HttpCompletionOption.ResponseHeadersRead, cancellationToken)
    .ConfigureAwait(false);

using var stream = await response.Content
    .ReadAsStreamAsync(cancellationToken)
    .ConfigureAwait(false);

var items = await JsonSerializer
    .DeserializeAsync<List<Item>>(stream, cancellationToken: cancellationToken)
    .ConfigureAwait(false);
```

---

## Memory Management

### Use Span<T> for String Operations

```csharp
// ❌ WRONG - Creates new strings
public string GetFirstWord(string input)
{
    var spaceIndex = input.IndexOf(' ');
    return spaceIndex == -1 ? input : input.Substring(0, spaceIndex);
}

// ✅ CORRECT - No allocation
public ReadOnlySpan<char> GetFirstWord(ReadOnlySpan<char> input)
{
    var spaceIndex = input.IndexOf(' ');
    return spaceIndex == -1 ? input : input[..spaceIndex];
}
```

### Use ArrayPool for Temporary Arrays

```csharp
// ❌ WRONG - Allocates new array
public byte[] ProcessData(int size)
{
    var buffer = new byte[size];
    // ... use buffer
    return result;
}

// ✅ CORRECT - Rent from pool
public byte[] ProcessData(int size)
{
    var buffer = ArrayPool<byte>.Shared.Rent(size);
    try
    {
        // ... use buffer
        return result;
    }
    finally
    {
        ArrayPool<byte>.Shared.Return(buffer);
    }
}
```

### Use StringBuilder for Multiple Concatenations

```csharp
// ❌ WRONG - Creates many intermediate strings
public string BuildMessage(IEnumerable<string> parts)
{
    var result = string.Empty;
    foreach (var part in parts)
    {
        result += part + ", ";
    }
    return result;
}

// ✅ CORRECT - Single StringBuilder
public string BuildMessage(IEnumerable<string> parts)
{
    var sb = new StringBuilder();
    foreach (var part in parts)
    {
        if (sb.Length > 0)
        {
            sb.Append(", ");
        }
        sb.Append(part);
    }
    return sb.ToString();
}

// ✅ ALSO CORRECT - String.Join
public string BuildMessage(IEnumerable<string> parts)
{
    return string.Join(", ", parts);
}
```

### Implement IDisposable Properly

```csharp
// <copyright file="ResourceHandler.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Infrastructure;

/// <summary>
/// Handles external resources with proper disposal.
/// </summary>
public class ResourceHandler : IDisposable, IAsyncDisposable
{
    private readonly Stream stream;
    private bool disposed;

    /// <inheritdoc/>
    public void Dispose()
    {
        this.Dispose(true);
        GC.SuppressFinalize(this);
    }

    /// <inheritdoc/>
    public async ValueTask DisposeAsync()
    {
        await this.DisposeAsyncCore().ConfigureAwait(false);
        this.Dispose(false);
        GC.SuppressFinalize(this);
    }

    /// <summary>
    /// Disposes managed resources.
    /// </summary>
    /// <param name="disposing">True if disposing managed resources.</param>
    protected virtual void Dispose(bool disposing)
    {
        if (this.disposed)
        {
            return;
        }

        if (disposing)
        {
            this.stream.Dispose();
        }

        this.disposed = true;
    }

    /// <summary>
    /// Disposes managed resources asynchronously.
    /// </summary>
    protected virtual async ValueTask DisposeAsyncCore()
    {
        if (this.disposed)
        {
            return;
        }

        await this.stream.DisposeAsync().ConfigureAwait(false);
        this.disposed = true;
    }
}
```

---

## Caching

### Use IMemoryCache for In-Process Caching

```csharp
// Program.cs
builder.Services.AddMemoryCache();

// Service
public class CachedOrderService : IOrderService
{
    private readonly IMemoryCache cache;
    private readonly IOrderRepository repository;
    private static readonly TimeSpan CacheDuration = TimeSpan.FromMinutes(5);

    public async Task<Order?> GetByIdAsync(string id, CancellationToken cancellationToken)
    {
        var cacheKey = $"order:{id}";

        if (this.cache.TryGetValue(cacheKey, out Order? cached))
        {
            return cached;
        }

        var order = await this.repository
            .GetByIdAsync(id, cancellationToken)
            .ConfigureAwait(false);

        if (order is not null)
        {
            var options = new MemoryCacheEntryOptions()
                .SetAbsoluteExpiration(CacheDuration)
                .SetSize(1);

            this.cache.Set(cacheKey, order, options);
        }

        return order;
    }
}
```

### Use IDistributedCache for Multi-Instance

```csharp
// Program.cs
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
    options.InstanceName = "{CompanyName}:";
});

// Service
public class DistributedCacheService
{
    private readonly IDistributedCache cache;
    private static readonly DistributedCacheEntryOptions CacheOptions = new()
    {
        AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10),
    };

    public async Task<T?> GetAsync<T>(string key, CancellationToken cancellationToken)
    {
        var bytes = await this.cache
            .GetAsync(key, cancellationToken)
            .ConfigureAwait(false);

        if (bytes is null)
        {
            return default;
        }

        return JsonSerializer.Deserialize<T>(bytes);
    }

    public async Task SetAsync<T>(string key, T value, CancellationToken cancellationToken)
    {
        var bytes = JsonSerializer.SerializeToUtf8Bytes(value);

        await this.cache
            .SetAsync(key, bytes, CacheOptions, cancellationToken)
            .ConfigureAwait(false);
    }
}
```

### Cache Invalidation Patterns

```csharp
// Invalidate on write
public async Task UpdateOrderAsync(Order order, CancellationToken cancellationToken)
{
    await this.repository
        .UpdateAsync(order, cancellationToken)
        .ConfigureAwait(false);

    // Invalidate cache
    this.cache.Remove($"order:{order.Id}");
    this.cache.Remove($"customer:{order.CustomerId}:orders");
}

// Cache-aside pattern with factory
public async Task<Order?> GetOrderAsync(string id, CancellationToken cancellationToken)
{
    return await this.cache.GetOrCreateAsync(
        $"order:{id}",
        async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5);
            return await this.repository
                .GetByIdAsync(id, cancellationToken)
                .ConfigureAwait(false);
        }).ConfigureAwait(false);
}
```

---

## JSON Serialization

### Use Source Generators

```csharp
// Define context
[JsonSerializable(typeof(Order))]
[JsonSerializable(typeof(List<Order>))]
[JsonSerializable(typeof(OrderResponse))]
[JsonSourceGenerationOptions(
    PropertyNamingPolicy = JsonKnownNamingPolicy.CamelCase,
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull)]
public partial class AppJsonContext : JsonSerializerContext
{
}

// Program.cs
builder.Services.ConfigureHttpJsonOptions(options =>
{
    options.SerializerOptions.TypeInfoResolverChain.Insert(0, AppJsonContext.Default);
});

// Usage
var json = JsonSerializer.Serialize(order, AppJsonContext.Default.Order);
var order = JsonSerializer.Deserialize(json, AppJsonContext.Default.Order);
```

### Reuse JsonSerializerOptions

```csharp
// ❌ WRONG - Creates new options each time
public string Serialize(object data)
{
    return JsonSerializer.Serialize(data, new JsonSerializerOptions
    {
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
    });
}

// ✅ CORRECT - Reuse options
private static readonly JsonSerializerOptions JsonOptions = new()
{
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull,
};

public string Serialize(object data)
{
    return JsonSerializer.Serialize(data, JsonOptions);
}
```

---

## Logging Performance

### Use Structured Logging with LoggerMessage

```csharp
// Define at class level
private static readonly Action<ILogger, string, int, Exception?> LogOrderProcessed =
    LoggerMessage.Define<string, int>(
        LogLevel.Information,
        new EventId(1, nameof(LogOrderProcessed)),
        "Order {OrderId} processed with {ItemCount} items");

private static readonly Action<ILogger, string, Exception?> LogOrderFailed =
    LoggerMessage.Define<string>(
        LogLevel.Error,
        new EventId(2, nameof(LogOrderFailed)),
        "Failed to process order {OrderId}");

// Usage - More efficient than string interpolation
public async Task ProcessOrderAsync(Order order, CancellationToken cancellationToken)
{
    LogOrderProcessed(this.logger, order.Id, order.Items.Count, null);

    try
    {
        await this.ProcessAsync(order, cancellationToken).ConfigureAwait(false);
    }
    catch (Exception ex)
    {
        LogOrderFailed(this.logger, order.Id, ex);
        throw;
    }
}
```

### Check Log Level Before Expensive Operations

```csharp
// ✅ CORRECT - Check before expensive serialization
if (this.logger.IsEnabled(LogLevel.Debug))
{
    this.logger.LogDebug("Order details: {OrderJson}", JsonSerializer.Serialize(order));
}

// ❌ WRONG - Always serializes even if debug is disabled
this.logger.LogDebug("Order details: {OrderJson}", JsonSerializer.Serialize(order));
```

---

## Performance Checklist

### Async/Await
- [ ] I/O operations use async methods
- [ ] ConfigureAwait(false) in library code
- [ ] No Task.Run for I/O-bound work
- [ ] ValueTask for frequently-sync methods

### Database
- [ ] AsNoTracking for read-only queries
- [ ] AsSplitQuery for multiple includes
- [ ] Projection with Select (no SELECT *)
- [ ] No N+1 queries
- [ ] Pagination implemented

### HTTP
- [ ] IHttpClientFactory used
- [ ] Connection pooling configured
- [ ] Streaming for large responses

### Memory
- [ ] ArrayPool for temporary arrays
- [ ] StringBuilder for string concatenation
- [ ] IDisposable/IAsyncDisposable implemented
- [ ] Span<T> for parsing operations

### Caching
- [ ] Appropriate cache strategy chosen
- [ ] Cache invalidation implemented
- [ ] Cache key strategy defined

### Serialization
- [ ] Source generators for JSON
- [ ] JsonSerializerOptions reused
