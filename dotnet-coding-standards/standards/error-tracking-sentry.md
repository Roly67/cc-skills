# Sentry Error Tracking Standards

Sentry is required for exception tracking in all host processes.

## Required Packages

```xml
<ItemGroup>
  <PackageReference Include="Sentry" Version="4.3.0" />
  <PackageReference Include="Sentry.AspNetCore" Version="4.3.0" />
  <PackageReference Include="Sentry.Serilog" Version="4.3.0" />
</ItemGroup>
```

## Configuration

### appsettings.json

```json
{
  "Sentry": {
    "Dsn": "",
    "Environment": "Development",
    "MinimumBreadcrumbLevel": "Debug",
    "MinimumEventLevel": "Error",
    "AttachStacktrace": true,
    "SendDefaultPii": false,
    "TracesSampleRate": 1.0
  }
}
```

### Environment-Specific Settings

**appsettings.Development.json**:
```json
{
  "Sentry": {
    "Environment": "Development",
    "TracesSampleRate": 1.0
  }
}
```

**appsettings.Production.json**:
```json
{
  "Sentry": {
    "Environment": "Production",
    "TracesSampleRate": 0.2,
    "MinimumEventLevel": "Warning"
  }
}
```

## Setup

### Web API

```csharp
var builder = WebApplication.CreateBuilder(args);

// Configure Sentry
builder.WebHost.UseSentry(options =>
{
    options.Dsn = builder.Configuration["Sentry:Dsn"];
    options.Environment = builder.Configuration["Sentry:Environment"];
    options.Release = typeof(Program).Assembly.GetName().Version?.ToString();
    
    // Performance monitoring
    options.TracesSampleRate = 1.0;
    options.ProfilesSampleRate = 1.0;
    
    // Error details
    options.AttachStacktrace = true;
    options.MaxBreadcrumbs = 100;
    
    // Privacy
    options.SendDefaultPii = false;
    
    // Minimum levels
    options.MinimumBreadcrumbLevel = LogLevel.Debug;
    options.MinimumEventLevel = LogLevel.Error;
    
    // Filter exceptions
    options.SetBeforeSend((sentryEvent, hint) =>
    {
        // Don't send cancelled operations
        if (sentryEvent.Exception is OperationCanceledException)
        {
            return null;
        }
        
        // Don't send 404s
        if (sentryEvent.Exception is NotFoundException)
        {
            return null;
        }
        
        return sentryEvent;
    });
});

var app = builder.Build();

// Enable Sentry tracing
app.UseSentryTracing();
```

### Console App / Worker Service

```csharp
var builder = Host.CreateApplicationBuilder(args);

builder.Services.AddSentry(options =>
{
    options.Dsn = builder.Configuration["Sentry:Dsn"];
    options.Environment = builder.Configuration["Sentry:Environment"];
    options.TracesSampleRate = 1.0;
    options.AttachStacktrace = true;
});
```

## Integration with Serilog

Sentry integrates with Serilog to automatically capture errors:

```csharp
builder.Host.UseSerilog((context, services, configuration) => configuration
    .ReadFrom.Configuration(context.Configuration)
    .WriteTo.Sentry(options =>
    {
        options.Dsn = context.Configuration["Sentry:Dsn"];
        options.Environment = context.Configuration["Sentry:Environment"];
        options.MinimumBreadcrumbLevel = LogEventLevel.Debug;
        options.MinimumEventLevel = LogEventLevel.Error;
    }));
```

## What Gets Sent to Sentry

| Log Level | Sent as Event | Sent as Breadcrumb |
|-----------|---------------|-------------------|
| Verbose/Trace | No | No |
| Debug | No | Yes |
| Information | No | Yes |
| Warning | Configurable | Yes |
| Error | Yes | Yes |
| Critical/Fatal | Yes | Yes |

## Manual Sentry Usage

### Capture Exception with Context

```csharp
public class OrderService
{
    private readonly IHub sentryHub;

    public OrderService(IHub sentryHub)
    {
        this.sentryHub = sentryHub;
    }

    public async Task ProcessOrderAsync(Order order)
    {
        try
        {
            await this.ProcessInternalAsync(order);
        }
        catch (Exception ex)
        {
            // Capture with additional context
            this.sentryHub.CaptureException(ex, scope =>
            {
                scope.SetTag("order_id", order.Id);
                scope.SetTag("customer_id", order.CustomerId);
                scope.SetExtra("order_total", order.Total);
                scope.SetExtra("item_count", order.Items.Count);
            });
            
            throw;
        }
    }
}
```

### Add Breadcrumbs

```csharp
public async Task ProcessAsync(string id)
{
    // Add breadcrumb for context
    this.sentryHub.AddBreadcrumb(
        message: $"Starting process for {id}",
        category: "process",
        level: BreadcrumbLevel.Info);

    // ... processing ...

    this.sentryHub.AddBreadcrumb(
        message: "Validation complete",
        category: "process",
        level: BreadcrumbLevel.Info,
        data: new Dictionary<string, string>
        {
            ["validation_result"] = "passed"
        });
}
```

### Performance Monitoring (Transactions)

```csharp
public async Task<Result> ExecuteAsync(Request request)
{
    // Start a transaction
    var transaction = this.sentryHub.StartTransaction(
        name: "ProcessOrder",
        operation: "order.process");

    try
    {
        // Create spans for sub-operations
        var validationSpan = transaction.StartChild("validation");
        await this.ValidateAsync(request);
        validationSpan.Finish(SpanStatus.Ok);

        var processingSpan = transaction.StartChild("processing");
        var result = await this.ProcessAsync(request);
        processingSpan.Finish(SpanStatus.Ok);

        transaction.Finish(SpanStatus.Ok);
        return result;
    }
    catch (Exception)
    {
        transaction.Finish(SpanStatus.InternalError);
        throw;
    }
}
```

### Set User Context

```csharp
// In middleware or auth handler
public class SentryUserMiddleware
{
    private readonly RequestDelegate next;

    public async Task InvokeAsync(HttpContext context, IHub sentryHub)
    {
        if (context.User.Identity?.IsAuthenticated == true)
        {
            sentryHub.ConfigureScope(scope =>
            {
                scope.User = new SentryUser
                {
                    Id = context.User.FindFirst("sub")?.Value,
                    Username = context.User.Identity.Name,
                    // Don't include email unless necessary (PII)
                };
            });
        }

        await this.next(context);
    }
}
```

## Filtering Events

### Ignore Specific Exceptions

```csharp
options.SetBeforeSend((sentryEvent, hint) =>
{
    // Ignore cancellation
    if (sentryEvent.Exception is OperationCanceledException)
    {
        return null;
    }

    // Ignore not found (expected in normal operation)
    if (sentryEvent.Exception is NotFoundException)
    {
        return null;
    }

    // Ignore validation errors (client errors)
    if (sentryEvent.Exception is ValidationException)
    {
        return null;
    }

    return sentryEvent;
});
```

### Filter Transactions (Performance)

```csharp
options.SetBeforeSendTransaction((transaction, hint) =>
{
    // Don't trace health checks
    if (transaction.Name.Contains("/health"))
    {
        return null;
    }

    // Don't trace static files
    if (transaction.Name.StartsWith("/static"))
    {
        return null;
    }

    return transaction;
});
```

## Sensitive Data

### Never Send PII Without Consent

```csharp
options.SendDefaultPii = false; // Default, keep it this way
```

### Scrub Sensitive Data

```csharp
options.SetBeforeSend((sentryEvent, hint) =>
{
    // Remove sensitive headers
    if (sentryEvent.Request?.Headers != null)
    {
        sentryEvent.Request.Headers.Remove("Authorization");
        sentryEvent.Request.Headers.Remove("Cookie");
    }

    // Scrub sensitive data from extras
    if (sentryEvent.Extra.ContainsKey("password"))
    {
        sentryEvent.Extra["password"] = "[REDACTED]";
    }

    return sentryEvent;
});
```

## Environment Configuration

| Environment | TracesSampleRate | MinimumEventLevel | Notes |
|-------------|------------------|-------------------|-------|
| Development | 1.0 (100%) | Error | Full visibility |
| Staging | 0.5 (50%) | Warning | Balance visibility/cost |
| Production | 0.1-0.2 (10-20%) | Error | Cost-effective |

## Alerts Configuration

Configure alerts in Sentry dashboard:

1. **Error spike** - Alert when error rate increases significantly
2. **New issue** - Alert on first occurrence of a new error type
3. **Regression** - Alert when resolved issue reoccurs
4. **Performance degradation** - Alert when p95 latency increases

## Verification

```bash
# Verify Sentry is configured
grep -r "UseSentry\|AddSentry" --include="Program.cs" src/

# Check Sentry DSN is set (should be in config, not hardcoded)
grep -r "Dsn.*=" --include="*.cs" src/

# Verify Sentry is in packages
grep -r "Sentry" --include="*.csproj" src/
```

## Troubleshooting

### Events Not Appearing

1. Check DSN is correct and not empty
2. Verify `MinimumEventLevel` isn't too high
3. Check `SetBeforeSend` isn't filtering too aggressively
4. Ensure exceptions aren't being swallowed

### Too Many Events

1. Add appropriate filters in `SetBeforeSend`
2. Increase `MinimumEventLevel` to `Error`
3. Filter expected exceptions (404s, validation errors)

### Missing Context

1. Add breadcrumbs before operations
2. Use `scope.SetTag()` and `scope.SetExtra()`
3. Configure user context in middleware
