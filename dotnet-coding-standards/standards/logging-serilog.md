# Serilog Logging Standards

Serilog is the required logging framework for all host processes.

## Core Principle

> **Serilog configuration takes precedence over Microsoft.Extensions.Logging.**
> 
> Do NOT use `Logging` section in appsettings.json for host processes.

```json
// ❌ FORBIDDEN in host appsettings.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information"
    }
  }
}

// ✅ CORRECT - Use Serilog section only
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information"
    }
  }
}
```

## Required Packages

### Web API / Blazor Server

```xml
<ItemGroup>
  <PackageReference Include="Serilog.AspNetCore" Version="8.0.0" />
  <PackageReference Include="Serilog.Sinks.Console" Version="5.0.1" />
  <PackageReference Include="Serilog.Sinks.File" Version="5.0.0" />
  <PackageReference Include="Serilog.Sinks.Seq" Version="7.0.0" />
  <PackageReference Include="Serilog.Enrichers.Environment" Version="2.3.0" />
  <PackageReference Include="Serilog.Enrichers.Thread" Version="3.1.0" />
  <PackageReference Include="Serilog.Settings.Configuration" Version="8.0.0" />
  <PackageReference Include="Sentry.Serilog" Version="4.3.0" />
</ItemGroup>
```

### Console App / Worker Service

```xml
<ItemGroup>
  <PackageReference Include="Serilog" Version="4.0.0" />
  <PackageReference Include="Serilog.Extensions.Hosting" Version="8.0.0" />
  <PackageReference Include="Serilog.Sinks.Console" Version="5.0.1" />
  <PackageReference Include="Serilog.Sinks.File" Version="5.0.0" />
  <PackageReference Include="Serilog.Settings.Configuration" Version="8.0.0" />
  <PackageReference Include="Sentry.Serilog" Version="4.3.0" />
</ItemGroup>
```

### Class Libraries

**Do NOT reference Serilog in class libraries.** Use only:

```xml
<ItemGroup>
  <PackageReference Include="Microsoft.Extensions.Logging.Abstractions" Version="10.0.0" />
</ItemGroup>
```

## Configuration

### appsettings.json (Base)

```json
{
  "Serilog": {
    "Using": [
      "Serilog.Sinks.Console",
      "Serilog.Sinks.File",
      "Serilog.Sinks.Seq",
      "Serilog.Enrichers.Environment",
      "Serilog.Enrichers.Thread",
      "Sentry.Serilog"
    ],
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "Microsoft.Hosting.Lifetime": "Information",
        "Microsoft.AspNetCore": "Warning",
        "Microsoft.EntityFrameworkCore": "Warning",
        "System": "Warning",
        "System.Net.Http.HttpClient": "Warning"
      }
    },
    "WriteTo": [
      {
        "Name": "Console",
        "Args": {
          "theme": "Serilog.Sinks.SystemConsole.Themes.AnsiConsoleTheme::Code, Serilog.Sinks.Console",
          "outputTemplate": "[{Timestamp:HH:mm:ss} {Level:u3}] {SourceContext}{NewLine}{Message:lj}{NewLine}{Exception}"
        }
      },
      {
        "Name": "File",
        "Args": {
          "path": "logs/log-.txt",
          "rollingInterval": "Day",
          "retainedFileCountLimit": 30,
          "outputTemplate": "{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} [{Level:u3}] [{SourceContext}] {Message:lj}{NewLine}{Exception}"
        }
      }
    ],
    "Enrich": [
      "FromLogContext",
      "WithMachineName",
      "WithProcessId",
      "WithThreadId"
    ],
    "Properties": {
      "Application": "{CompanyName}.{ProjectName}"
    }
  }
}
```

### appsettings.Development.json

```json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Debug",
      "Override": {
        "Microsoft": "Information",
        "Microsoft.AspNetCore": "Information"
      }
    }
  }
}
```

### appsettings.Production.json

```json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning"
      }
    },
    "WriteTo": [
      {
        "Name": "Console"
      },
      {
        "Name": "File",
        "Args": {
          "path": "/var/log/{companyname}/{projectname}/log-.txt",
          "rollingInterval": "Day",
          "retainedFileCountLimit": 30
        }
      },
      {
        "Name": "Seq",
        "Args": {
          "serverUrl": "https://seq.example.com",
          "apiKey": ""
        }
      }
    ]
  }
}
```

## Program.cs Setup

### Web API

```csharp
using Serilog;
using Serilog.Events;

// Bootstrap logger for startup errors
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Information)
    .Enrich.FromLogContext()
    .WriteTo.Console()
    .CreateBootstrapLogger();

try
{
    Log.Information("Starting application");

    var builder = WebApplication.CreateBuilder(args);

    // Configure Serilog (REPLACES default logging)
    builder.Host.UseSerilog((context, services, configuration) => configuration
        .ReadFrom.Configuration(context.Configuration)
        .ReadFrom.Services(services)
        .Enrich.FromLogContext()
        .WriteTo.Sentry(options =>
        {
            options.Dsn = context.Configuration["Sentry:Dsn"];
            options.MinimumEventLevel = LogEventLevel.Error;
        }));

    // ... rest of setup

    var app = builder.Build();

    // Use Serilog request logging (replaces default)
    app.UseSerilogRequestLogging();

    // ... rest of pipeline

    await app.RunAsync();
}
catch (Exception ex)
{
    Log.Fatal(ex, "Application terminated unexpectedly");
    throw;
}
finally
{
    await Log.CloseAndFlushAsync();
}
```

### Console App / Worker Service

```csharp
using Serilog;
using Serilog.Events;

Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Debug()
    .WriteTo.Console()
    .CreateBootstrapLogger();

try
{
    Log.Information("Starting application");

    var builder = Host.CreateApplicationBuilder(args);

    builder.Services.AddSerilog((services, configuration) => configuration
        .ReadFrom.Configuration(builder.Configuration)
        .ReadFrom.Services(services)
        .Enrich.FromLogContext()
        .WriteTo.Sentry(options =>
        {
            options.Dsn = builder.Configuration["Sentry:Dsn"];
            options.MinimumEventLevel = LogEventLevel.Error;
        }));

    // ... rest of setup

    var host = builder.Build();
    await host.RunAsync();
}
catch (Exception ex)
{
    Log.Fatal(ex, "Application terminated unexpectedly");
    throw;
}
finally
{
    await Log.CloseAndFlushAsync();
}
```

## Log Levels

| Level | Serilog | ILogger | When to Use | Sent to Sentry |
|-------|---------|---------|-------------|----------------|
| Verbose | `Verbose` | `Trace` | Detailed tracing | No |
| Debug | `Debug` | `Debug` | Development diagnostics | No (breadcrumbs) |
| Information | `Information` | `Information` | Normal flow, significant events | No |
| Warning | `Warning` | `Warning` | Unexpected but handled | Configurable |
| Error | `Error` | `Error` | Operation failures | Yes |
| Fatal | `Fatal` | `Critical` | Application crashes | Yes |

## Structured Logging

### ✅ Correct - Structured Logging

```csharp
// Use named placeholders
this.logger.LogInformation(
    "Processing order {OrderId} for customer {CustomerId} with {ItemCount} items",
    order.Id,
    order.CustomerId,
    order.Items.Count);

// Log objects (serialized)
this.logger.LogDebug("Order details: {@Order}", order);

// Log with destructuring
this.logger.LogInformation(
    "User {UserId} performed {Action} on {Resource}",
    userId,
    action,
    resource);
```

### ❌ Wrong - String Interpolation

```csharp
// String interpolation - loses structure, poor performance
this.logger.LogInformation($"Processing order {order.Id}");

// Concatenation - same problems
this.logger.LogInformation("Processing order " + order.Id);

// ToString - loses type information
this.logger.LogInformation(order.ToString());
```

## Enrichers

### Built-in Enrichers

```json
{
  "Serilog": {
    "Enrich": [
      "FromLogContext",      // Adds properties from LogContext
      "WithMachineName",     // Server name
      "WithProcessId",       // Process ID
      "WithThreadId",        // Thread ID
      "WithEnvironmentName"  // ASPNETCORE_ENVIRONMENT
    ]
  }
}
```

### Custom Enrichment

```csharp
// In code - add to all logs in a scope
using (LogContext.PushProperty("CorrelationId", correlationId))
{
    this.logger.LogInformation("Starting operation");
    await this.ProcessAsync();
    this.logger.LogInformation("Operation complete");
}

// Per-request enrichment (in middleware)
app.Use(async (context, next) =>
{
    var correlationId = context.Request.Headers["X-Correlation-ID"].FirstOrDefault()
        ?? Guid.NewGuid().ToString();
    
    using (LogContext.PushProperty("CorrelationId", correlationId))
    {
        await next();
    }
});
```

## Output Templates

### Console (Development)

```
[14:23:45 INF] {CompanyName}.{ProjectName}.Services.OrderService
Processing order ORD-123 for customer CUST-456
```

### File (Production)

```
2026-01-10 14:23:45.123 +00:00 [INF] [{CompanyName}.{ProjectName}.Services.OrderService] Processing order ORD-123 for customer CUST-456
```

### JSON (for log aggregation)

```json
{
  "Serilog": {
    "WriteTo": [
      {
        "Name": "Console",
        "Args": {
          "formatter": "Serilog.Formatting.Json.JsonFormatter, Serilog"
        }
      }
    ]
  }
}
```

## Filtering

### Suppress Noisy Sources

```json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft.AspNetCore.Hosting": "Warning",
        "Microsoft.AspNetCore.Routing": "Warning",
        "Microsoft.AspNetCore.Mvc": "Warning",
        "Microsoft.EntityFrameworkCore.Database.Command": "Warning",
        "System.Net.Http.HttpClient": "Warning"
      }
    }
  }
}
```

### Filter by Condition

```csharp
builder.Host.UseSerilog((context, configuration) => configuration
    .Filter.ByExcluding(logEvent => 
        logEvent.Properties.ContainsKey("SourceContext") &&
        logEvent.Properties["SourceContext"].ToString().Contains("HealthChecks")));
```

## Best Practices

### 1. Log at Appropriate Levels

```csharp
// Debug - Development only
this.logger.LogDebug("Entering method {MethodName}", nameof(this.ProcessAsync));

// Information - Significant business events
this.logger.LogInformation("Order {OrderId} created successfully", order.Id);

// Warning - Unexpected but handled
this.logger.LogWarning("Retry attempt {Attempt} for {Operation}", attempt, operation);

// Error - Operation failed
this.logger.LogError(ex, "Failed to process order {OrderId}", orderId);

// Critical - Application failure
this.logger.LogCritical(ex, "Database connection lost");
```

### 2. Include Context

```csharp
// Good - includes relevant context
this.logger.LogError(
    ex,
    "Failed to send email to {Recipient} for order {OrderId}",
    email,
    orderId);

// Bad - no context
this.logger.LogError(ex, "Email failed");
```

### 3. Don't Log Sensitive Data

```csharp
// ❌ Wrong - logs password
this.logger.LogDebug("Login attempt for {User} with password {Password}", user, password);

// ✅ Correct - no sensitive data
this.logger.LogDebug("Login attempt for {User}", user);
```

### 4. Use Scopes for Related Operations

```csharp
using (this.logger.BeginScope("Processing batch {BatchId}", batchId))
{
    foreach (var item in items)
    {
        this.logger.LogDebug("Processing item {ItemId}", item.Id);
        await this.ProcessItemAsync(item);
    }
}
```

## Verification

```bash
# Check for Microsoft logging config (should not exist)
grep -r '"Logging"' --include="appsettings*.json" src/

# Check for string interpolation in logs
grep -rE 'Log(Debug|Information|Warning|Error|Critical)\(\$"' --include="*.cs" src/

# Verify Serilog is configured
grep -r "UseSerilog" --include="Program.cs" src/
```
