# Worker Service Projects

Standards and patterns for .NET Worker Services (background services).

## Project Structure

```
src/
├── {CompanyName}.ProjectName.Worker/
│   ├── Workers/               # BackgroundService implementations
│   ├── Handlers/              # Message/event handlers
│   ├── Configuration/         # Options classes
│   └── Program.cs
├── {CompanyName}.ProjectName.Core/      # Shared business logic
│   ├── Interfaces/
│   ├── Models/
│   └── Services/
└── tests/
    └── {CompanyName}.ProjectName.Worker.Tests/
```

## Project File (.csproj)

```xml
<Project Sdk="Microsoft.NET.Sdk.Worker">

  <PropertyGroup>
    <UserSecretsId>{project-user-secrets-id}</UserSecretsId>
  </PropertyGroup>

  <!-- For Windows Service deployment -->
  <PropertyGroup Condition="'$(RuntimeIdentifier)' == 'win-x64'">
    <PublishSingleFile>true</PublishSingleFile>
  </PropertyGroup>

  <ItemGroup>
    <!-- Serilog -->
    <PackageReference Include="Serilog" Version="4.0.0" />
    <PackageReference Include="Serilog.Extensions.Hosting" Version="8.0.0" />
    <PackageReference Include="Serilog.Sinks.Console" Version="5.0.1" />
    <PackageReference Include="Serilog.Sinks.File" Version="5.0.0" />
    <PackageReference Include="Serilog.Sinks.Seq" Version="7.0.0" />
    <PackageReference Include="Serilog.Enrichers.Environment" Version="2.3.0" />
    <PackageReference Include="Serilog.Settings.Configuration" Version="8.0.0" />
    
    <!-- Sentry -->
    <PackageReference Include="Sentry" Version="4.3.0" />
    <PackageReference Include="Sentry.Serilog" Version="4.3.0" />
    
    <!-- Hosting -->
    <PackageReference Include="Microsoft.Extensions.Hosting" Version="10.0.0" />
    <PackageReference Include="Microsoft.Extensions.Hosting.WindowsServices" Version="10.0.0" />
    <PackageReference Include="Microsoft.Extensions.Hosting.Systemd" Version="10.0.0" />
  </ItemGroup>

</Project>
```

## Program.cs Template

```csharp
// <copyright file="Program.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

using Serilog;
using Serilog.Events;

using {CompanyName}.EmailProcessor.Worker.Configuration;
using {CompanyName}.EmailProcessor.Worker.Workers;

// Bootstrap logger
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Debug()
    .WriteTo.Console()
    .CreateBootstrapLogger();

try
{
    Log.Information("Starting {CompanyName}.EmailProcessor.Worker");

    var builder = Host.CreateApplicationBuilder(args);

    // Enable Windows Service / systemd support
    builder.Services.AddWindowsService(options =>
    {
        options.ServiceName = "{ServiceDisplayName}";
    });
    builder.Services.AddSystemd();

    // Configure Serilog (REPLACES default logging)
    builder.Services.AddSerilog((services, configuration) => configuration
        .ReadFrom.Configuration(builder.Configuration)
        .ReadFrom.Services(services)
        .Enrich.FromLogContext()
        .Enrich.WithProperty("ServiceName", "EmailProcessor.Worker")
        .WriteTo.Sentry(options =>
        {
            options.Dsn = builder.Configuration["Sentry:Dsn"];
            options.Environment = builder.Configuration["Sentry:Environment"];
            options.MinimumEventLevel = LogEventLevel.Error;
        }));

    // Configure Sentry
    builder.Services.AddSentry(options =>
    {
        options.Dsn = builder.Configuration["Sentry:Dsn"];
        options.Environment = builder.Configuration["Sentry:Environment"];
        options.TracesSampleRate = 1.0;
        options.AttachStacktrace = true;
    });

    // Add configuration
    builder.Services.Configure<WorkerOptions>(
        builder.Configuration.GetSection(WorkerOptions.SectionName));

    // Add services (scoped for proper disposal)
    builder.Services.AddScoped<IEmailQueueService, EmailQueueService>();
    builder.Services.AddScoped<IEmailProcessor, EmailProcessor>();

    // Add worker
    builder.Services.AddHostedService<EmailProcessorWorker>();

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

## Worker Options Class

```csharp
// <copyright file="WorkerOptions.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.EmailProcessor.Worker.Configuration;

/// <summary>
/// Configuration options for the email processor worker.
/// </summary>
public class WorkerOptions
{
    /// <summary>
    /// The configuration section name.
    /// </summary>
    public const string SectionName = "Worker";

    /// <summary>
    /// Gets or sets the polling interval in milliseconds.
    /// </summary>
    public int PollingIntervalMs { get; set; } = 1000;

    /// <summary>
    /// Gets or sets the batch size for processing.
    /// </summary>
    public int BatchSize { get; set; } = 10;

    /// <summary>
    /// Gets or sets the maximum retry count.
    /// </summary>
    public int MaxRetries { get; set; } = 3;

    /// <summary>
    /// Gets or sets the retry delay in milliseconds.
    /// </summary>
    public int RetryDelayMs { get; set; } = 1000;

    /// <summary>
    /// Gets or sets a value indicating whether to use exponential backoff.
    /// </summary>
    public bool UseExponentialBackoff { get; set; } = true;
}
```

## Worker Template

```csharp
// <copyright file="EmailProcessorWorker.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.EmailProcessor.Worker.Workers;

using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;

using {CompanyName}.EmailProcessor.Worker.Configuration;

/// <summary>
/// Background worker that processes the email queue.
/// </summary>
public class EmailProcessorWorker : BackgroundService
{
    private readonly IServiceScopeFactory scopeFactory;
    private readonly ILogger<EmailProcessorWorker> logger;
    private readonly WorkerOptions options;

    /// <summary>
    /// Initializes a new instance of the <see cref="EmailProcessorWorker"/> class.
    /// </summary>
    /// <param name="scopeFactory">The service scope factory.</param>
    /// <param name="logger">The logger instance.</param>
    /// <param name="options">The worker options.</param>
    public EmailProcessorWorker(
        IServiceScopeFactory scopeFactory,
        ILogger<EmailProcessorWorker> logger,
        IOptions<WorkerOptions> options)
    {
        this.scopeFactory = scopeFactory ?? throw new ArgumentNullException(nameof(scopeFactory));
        this.logger = logger ?? throw new ArgumentNullException(nameof(logger));
        this.options = options?.Value ?? throw new ArgumentNullException(nameof(options));
    }

    /// <inheritdoc/>
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        this.logger.LogInformation(
            "Worker starting with {PollingInterval}ms interval, batch size {BatchSize}",
            this.options.PollingIntervalMs,
            this.options.BatchSize);

        // Wait a bit before starting to allow other services to initialize
        await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);

        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await this.ProcessBatchAsync(stoppingToken);
            }
            catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested)
            {
                // Graceful shutdown - exit the loop
                this.logger.LogInformation("Worker received shutdown signal");
                break;
            }
            catch (Exception ex)
            {
                // Log but don't rethrow - worker should continue running
                this.logger.LogError(ex, "Error processing batch");
            }

            // Wait before next iteration
            // Note: NO ConfigureAwait(false) in ExecuteAsync main loop
            await Task.Delay(this.options.PollingIntervalMs, stoppingToken);
        }

        this.logger.LogInformation("Worker stopped");
    }

    /// <inheritdoc/>
    public override async Task StopAsync(CancellationToken cancellationToken)
    {
        this.logger.LogInformation("Worker is stopping gracefully...");

        // Allow base class to signal cancellation
        await base.StopAsync(cancellationToken);

        this.logger.LogInformation("Worker stopped gracefully");
    }

    private async Task ProcessBatchAsync(CancellationToken cancellationToken)
    {
        // Create a scope for scoped services
        await using var scope = this.scopeFactory.CreateAsyncScope();

        var queueService = scope.ServiceProvider.GetRequiredService<IEmailQueueService>();
        var processor = scope.ServiceProvider.GetRequiredService<IEmailProcessor>();

        // Use ConfigureAwait(false) in private methods
        var batch = await queueService
            .DequeueAsync(this.options.BatchSize, cancellationToken)
            .ConfigureAwait(false);

        if (batch.Count == 0)
        {
            this.logger.LogDebug("No items to process");
            return;
        }

        this.logger.LogInformation("Processing batch of {Count} items", batch.Count);

        var processed = 0;
        var failed = 0;

        foreach (var item in batch)
        {
            try
            {
                await this.ProcessItemWithRetryAsync(processor, item, cancellationToken)
                    .ConfigureAwait(false);
                processed++;
            }
            catch (Exception ex)
            {
                this.logger.LogError(ex, "Failed to process item {ItemId}", item.Id);
                failed++;

                // Mark item as failed
                await queueService
                    .MarkFailedAsync(item.Id, ex.Message, cancellationToken)
                    .ConfigureAwait(false);
            }
        }

        this.logger.LogInformation(
            "Batch complete: {Processed} processed, {Failed} failed",
            processed,
            failed);
    }

    private async Task ProcessItemWithRetryAsync(
        IEmailProcessor processor,
        QueueItem item,
        CancellationToken cancellationToken)
    {
        var attempt = 0;
        var delay = this.options.RetryDelayMs;

        while (true)
        {
            attempt++;

            try
            {
                await processor
                    .ProcessAsync(item, cancellationToken)
                    .ConfigureAwait(false);

                return; // Success
            }
            catch (Exception ex) when (attempt < this.options.MaxRetries)
            {
                this.logger.LogWarning(
                    ex,
                    "Attempt {Attempt}/{MaxRetries} failed for item {ItemId}, retrying in {Delay}ms",
                    attempt,
                    this.options.MaxRetries,
                    item.Id,
                    delay);

                await Task.Delay(delay, cancellationToken).ConfigureAwait(false);

                // Exponential backoff
                if (this.options.UseExponentialBackoff)
                {
                    delay *= 2;
                }
            }
        }
    }
}
```

## appsettings.json for Worker Services

```json
{
  "Serilog": {
    "Using": [
      "Serilog.Sinks.Console",
      "Serilog.Sinks.File",
      "Serilog.Sinks.Seq"
    ],
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "Microsoft.Hosting.Lifetime": "Information",
        "System": "Warning"
      }
    },
    "WriteTo": [
      {
        "Name": "Console",
        "Args": {
          "outputTemplate": "[{Timestamp:HH:mm:ss} {Level:u3}] [{ServiceName}] {Message:lj}{NewLine}{Exception}"
        }
      },
      {
        "Name": "File",
        "Args": {
          "path": "logs/worker-.log",
          "rollingInterval": "Day",
          "retainedFileCountLimit": 30
        }
      }
    ],
    "Enrich": ["FromLogContext", "WithMachineName", "WithProcessId"],
    "Properties": {
      "Application": "{CompanyName}.EmailProcessor.Worker"
    }
  },
  "Sentry": {
    "Dsn": "",
    "Environment": "Development"
  },
  "Worker": {
    "PollingIntervalMs": 1000,
    "BatchSize": 10,
    "MaxRetries": 3,
    "RetryDelayMs": 1000,
    "UseExponentialBackoff": true
  }
}
```

## Multiple Workers Example

```csharp
// Program.cs - Multiple workers
builder.Services.AddHostedService<EmailProcessorWorker>();
builder.Services.AddHostedService<NotificationWorker>();
builder.Services.AddHostedService<CleanupWorker>();
```

## Health Checks

```csharp
// <copyright file="Program.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

// Add health checks
builder.Services.AddHealthChecks()
    .AddCheck<EmailQueueHealthCheck>("email-queue")
    .AddCheck<DatabaseHealthCheck>("database");

// After building the host, map health endpoint
var host = builder.Build();

// For workers that also expose HTTP endpoints
if (builder.Configuration.GetValue<bool>("EnableHealthEndpoint"))
{
    var app = host as WebApplication;
    app?.MapHealthChecks("/health");
}

await host.RunAsync();
```

## Windows Service Installation

```powershell
# Create the service
sc.exe create "{ServiceName}" binPath="C:\Services\{CompanyName}.EmailProcessor.Worker.exe"

# Configure service
sc.exe config "{ServiceName}" start=auto
sc.exe description "{ServiceName}" "{ServiceDisplayName} Background Service"

# Start the service
sc.exe start "{ServiceName}"

# Stop and delete (for updates)
sc.exe stop "{ServiceName}"
sc.exe delete "{ServiceName}"
```

## Linux systemd Installation

```ini
# /etc/systemd/system/{service-name}.service
[Unit]
Description={ServiceDisplayName} Worker
After=network.target

[Service]
Type=notify
WorkingDirectory=/opt/{company}/{service}
ExecStart=/opt/{company}/{service}/{CompanyName}.EmailProcessor.Worker
Restart=always
RestartSec=10
User={service-user}
Environment=DOTNET_ENVIRONMENT=Production
Environment=ASPNETCORE_ENVIRONMENT=Production

[Install]
WantedBy=multi-user.target
```

```bash
# Enable and start
sudo systemctl daemon-reload
sudo systemctl enable {service-name}
sudo systemctl start {service-name}

# Check status
sudo systemctl status {service-name}

# View logs
sudo journalctl -u {service-name} -f
```

## Important Notes

### ConfigureAwait Usage

```csharp
// In ExecuteAsync main loop: NO ConfigureAwait(false)
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    while (!stoppingToken.IsCancellationRequested)
    {
        await this.ProcessBatchAsync(stoppingToken); // No ConfigureAwait
        await Task.Delay(1000, stoppingToken);       // No ConfigureAwait
    }
}

// In private helper methods: YES ConfigureAwait(false)
private async Task ProcessBatchAsync(CancellationToken ct)
{
    var data = await this.service.GetAsync(ct).ConfigureAwait(false);
    await this.processor.ProcessAsync(data, ct).ConfigureAwait(false);
}
```

### Graceful Shutdown

- Always handle `OperationCanceledException` in the main loop
- Override `StopAsync` for cleanup logic
- Use `CancellationToken` throughout

### Scoped Services

- Use `IServiceScopeFactory` to create scopes for scoped dependencies
- Dispose scopes properly with `await using`

### Retry Logic

- Implement exponential backoff for transient failures
- Set reasonable retry limits
- Log retry attempts for debugging
