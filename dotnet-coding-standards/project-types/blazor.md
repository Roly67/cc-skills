# Blazor Projects

Standards and patterns for Blazor Server and Blazor WebAssembly projects.

## Project Types

| Type | Hosting | Use Case |
|------|---------|----------|
| Blazor Server | Server-side | Real-time updates, full .NET access |
| Blazor WebAssembly | Client-side | Offline capability, reduced server load |
| Blazor United (.NET 8+) | Hybrid | Best of both worlds |

## Project Structure (Blazor Server)

```
src/
└── {CompanyName}.Dashboard.Web/
    ├── Components/
    │   ├── Layout/
    │   │   ├── MainLayout.razor
    │   │   └── NavMenu.razor
    │   └── Shared/
    │       ├── MetricCard.razor
    │       └── DataGrid.razor
    ├── Pages/
    │   ├── Dashboard.razor
    │   ├── Dashboard.razor.cs
    │   └── Settings.razor
    ├── Services/
    │   └── DashboardService.cs
    ├── Models/
    │   └── DashboardViewModel.cs
    ├── wwwroot/
    │   ├── css/
    │   └── js/
    └── Program.cs
```

## Project File (.csproj) - Blazor Server

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <UserSecretsId>{project-user-secrets-id}</UserSecretsId>
  </PropertyGroup>

  <ItemGroup>
    <!-- Serilog -->
    <PackageReference Include="Serilog.AspNetCore" Version="8.0.0" />
    <PackageReference Include="Serilog.Sinks.Console" Version="5.0.1" />
    <PackageReference Include="Serilog.Sinks.File" Version="5.0.0" />
    <PackageReference Include="Serilog.Settings.Configuration" Version="8.0.0" />
    
    <!-- Sentry -->
    <PackageReference Include="Sentry.AspNetCore" Version="4.3.0" />
    <PackageReference Include="Sentry.Serilog" Version="4.3.0" />
  </ItemGroup>

</Project>
```

## Program.cs Template (Blazor Server)

```csharp
// <copyright file="Program.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

using Serilog;
using Serilog.Events;

using {CompanyName}.Dashboard.Web.Services;

// Bootstrap logger
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Information)
    .Enrich.FromLogContext()
    .WriteTo.Console()
    .CreateBootstrapLogger();

try
{
    Log.Information("Starting {CompanyName}.Dashboard.Web");

    var builder = WebApplication.CreateBuilder(args);

    // Configure Serilog
    builder.Host.UseSerilog((context, services, configuration) => configuration
        .ReadFrom.Configuration(context.Configuration)
        .ReadFrom.Services(services)
        .Enrich.FromLogContext()
        .WriteTo.Sentry(options =>
        {
            options.Dsn = context.Configuration["Sentry:Dsn"];
            options.Environment = context.Configuration["Sentry:Environment"];
            options.MinimumEventLevel = LogEventLevel.Error;
        }));

    // Configure Sentry
    builder.WebHost.UseSentry(options =>
    {
        options.Dsn = builder.Configuration["Sentry:Dsn"];
        options.Environment = builder.Configuration["Sentry:Environment"];
        options.TracesSampleRate = 1.0;
    });

    // Add services
    builder.Services.AddRazorComponents()
        .AddInteractiveServerComponents();

    builder.Services.AddScoped<IDashboardService, DashboardService>();

    var app = builder.Build();

    // Configure pipeline
    if (!app.Environment.IsDevelopment())
    {
        app.UseExceptionHandler("/Error");
        app.UseHsts();
    }

    app.UseHttpsRedirection();
    app.UseStaticFiles();
    app.UseAntiforgery();

    app.UseSerilogRequestLogging();
    app.UseSentryTracing();

    app.MapRazorComponents<App>()
        .AddInteractiveServerRenderMode();

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

## Page Component with Code-Behind

### Dashboard.razor

```razor
@page "/dashboard"
@using {CompanyName}.Dashboard.Web.Models

<PageTitle>Dashboard</PageTitle>

<h1>Dashboard</h1>

@if (this.isLoading)
{
    <div class="loading-spinner">
        <span>Loading...</span>
    </div>
}
else if (this.viewModel is not null)
{
    <div class="dashboard-grid">
        <MetricCard 
            Title="Total Users" 
            Value="@this.viewModel.TotalUsers.ToString("N0")"
            TrendPercentage="@this.viewModel.UsersTrend" />
        
        <MetricCard 
            Title="Active Sessions" 
            Value="@this.viewModel.ActiveSessions.ToString("N0")" />
        
        <MetricCard 
            Title="Revenue" 
            Value="@this.viewModel.Revenue.ToString("C")"
            TrendPercentage="@this.viewModel.RevenueTrend" />
    </div>
}
else
{
    <div class="error-message">
        <p>Failed to load dashboard data.</p>
        <button @onclick="this.LoadDataAsync">Retry</button>
    </div>
}
```

### Dashboard.razor.cs (Code-Behind)

```csharp
// <copyright file="Dashboard.razor.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.Dashboard.Web.Pages;

using Microsoft.AspNetCore.Components;

using {CompanyName}.Dashboard.Web.Models;
using {CompanyName}.Dashboard.Web.Services;

/// <summary>
/// Dashboard page displaying system metrics.
/// </summary>
public partial class Dashboard : ComponentBase, IDisposable
{
    private CancellationTokenSource? cancellationTokenSource;
    private bool isLoading = true;
    private DashboardViewModel? viewModel;
    private bool isDisposed;

    /// <summary>
    /// Gets or sets the logger.
    /// </summary>
    [Inject]
    private ILogger<Dashboard> Logger { get; set; } = default!;

    /// <summary>
    /// Gets or sets the dashboard service.
    /// </summary>
    [Inject]
    private IDashboardService DashboardService { get; set; } = default!;

    /// <inheritdoc/>
    protected override async Task OnInitializedAsync()
    {
        await this.LoadDataAsync();
    }

    /// <inheritdoc/>
    public void Dispose()
    {
        this.Dispose(true);
        GC.SuppressFinalize(this);
    }

    /// <summary>
    /// Disposes managed resources.
    /// </summary>
    /// <param name="disposing">True if disposing managed resources.</param>
    protected virtual void Dispose(bool disposing)
    {
        if (this.isDisposed)
        {
            return;
        }

        if (disposing)
        {
            this.cancellationTokenSource?.Cancel();
            this.cancellationTokenSource?.Dispose();
        }

        this.isDisposed = true;
    }

    private async Task LoadDataAsync()
    {
        this.isLoading = true;
        this.cancellationTokenSource?.Cancel();
        this.cancellationTokenSource = new CancellationTokenSource();

        try
        {
            this.Logger.LogDebug("Loading dashboard data");

            // NOTE: Do NOT use ConfigureAwait(false) in Blazor components
            this.viewModel = await this.DashboardService.GetDashboardAsync(
                this.cancellationTokenSource.Token);

            this.Logger.LogDebug("Dashboard data loaded successfully");
        }
        catch (OperationCanceledException)
        {
            this.Logger.LogDebug("Dashboard load was cancelled");
        }
        catch (Exception ex)
        {
            this.Logger.LogError(ex, "Failed to load dashboard data");
            this.viewModel = null;
        }
        finally
        {
            this.isLoading = false;
        }
    }
}
```

## Reusable Component

### MetricCard.razor

```razor
<div class="metric-card @this.CssClass">
    <h3 class="metric-title">@this.Title</h3>
    <div class="metric-value">@this.Value</div>
    
    @if (this.TrendPercentage.HasValue)
    {
        <div class="metric-trend @this.TrendClass">
            @if (this.TrendPercentage > 0)
            {
                <span>▲</span>
            }
            else if (this.TrendPercentage < 0)
            {
                <span>▼</span>
            }
            <span>@Math.Abs(this.TrendPercentage.Value).ToString("F1")%</span>
        </div>
    }
</div>
```

### MetricCard.razor.cs

```csharp
// <copyright file="MetricCard.razor.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.Dashboard.Web.Components.Shared;

using Microsoft.AspNetCore.Components;

/// <summary>
/// Displays a metric card with title, value, and optional trend indicator.
/// </summary>
public partial class MetricCard : ComponentBase
{
    /// <summary>
    /// Gets or sets the card title.
    /// </summary>
    [Parameter]
    [EditorRequired]
    public string Title { get; set; } = string.Empty;

    /// <summary>
    /// Gets or sets the metric value.
    /// </summary>
    [Parameter]
    [EditorRequired]
    public string Value { get; set; } = string.Empty;

    /// <summary>
    /// Gets or sets the trend percentage (positive = up, negative = down).
    /// </summary>
    [Parameter]
    public decimal? TrendPercentage { get; set; }

    /// <summary>
    /// Gets or sets the CSS class for custom styling.
    /// </summary>
    [Parameter]
    public string? CssClass { get; set; }

    /// <summary>
    /// Gets the CSS class for the trend indicator.
    /// </summary>
    private string TrendClass => this.TrendPercentage switch
    {
        > 0 => "trend-up",
        < 0 => "trend-down",
        _ => "trend-neutral",
    };
}
```

## Service Layer

```csharp
// <copyright file="DashboardService.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.Dashboard.Web.Services;

using {CompanyName}.Dashboard.Web.Models;

/// <summary>
/// Service for retrieving dashboard data.
/// </summary>
public class DashboardService : IDashboardService
{
    private readonly HttpClient httpClient;
    private readonly ILogger<DashboardService> logger;

    /// <summary>
    /// Initializes a new instance of the <see cref="DashboardService"/> class.
    /// </summary>
    /// <param name="httpClient">The HTTP client.</param>
    /// <param name="logger">The logger.</param>
    public DashboardService(HttpClient httpClient, ILogger<DashboardService> logger)
    {
        this.httpClient = httpClient ?? throw new ArgumentNullException(nameof(httpClient));
        this.logger = logger ?? throw new ArgumentNullException(nameof(logger));
    }

    /// <inheritdoc/>
    public async Task<DashboardViewModel> GetDashboardAsync(CancellationToken cancellationToken)
    {
        this.logger.LogDebug("Fetching dashboard data from API");

        // Use ConfigureAwait(false) in service layer
        var response = await this.httpClient
            .GetFromJsonAsync<DashboardViewModel>("/api/dashboard", cancellationToken)
            .ConfigureAwait(false);

        return response ?? throw new InvalidOperationException("Failed to retrieve dashboard data");
    }
}
```

## ViewModel

```csharp
// <copyright file="DashboardViewModel.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.Dashboard.Web.Models;

/// <summary>
/// View model for the dashboard page.
/// </summary>
public class DashboardViewModel
{
    /// <summary>
    /// Gets or sets the total user count.
    /// </summary>
    public int TotalUsers { get; set; }

    /// <summary>
    /// Gets or sets the users trend percentage.
    /// </summary>
    public decimal? UsersTrend { get; set; }

    /// <summary>
    /// Gets or sets the active session count.
    /// </summary>
    public int ActiveSessions { get; set; }

    /// <summary>
    /// Gets or sets the revenue amount.
    /// </summary>
    public decimal Revenue { get; set; }

    /// <summary>
    /// Gets or sets the revenue trend percentage.
    /// </summary>
    public decimal? RevenueTrend { get; set; }
}
```

## Blazor WebAssembly Differences

### Project File (.csproj) - Blazor WASM

```xml
<Project Sdk="Microsoft.NET.Sdk.BlazorWebAssembly">

  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.Components.WebAssembly" Version="10.0.0" />
    <PackageReference Include="Microsoft.AspNetCore.Components.WebAssembly.DevServer" Version="10.0.0" PrivateAssets="all" />
  </ItemGroup>

</Project>
```

### Logging in Blazor WASM

Blazor WebAssembly has limited logging capabilities. Use browser console:

```csharp
// Program.cs for Blazor WASM
builder.Logging.SetMinimumLevel(LogLevel.Debug);
builder.Logging.AddConfiguration(builder.Configuration.GetSection("Logging"));
```

> **Note**: Serilog and Sentry are typically configured on the **server-side API** that the WASM app calls, not in the WASM app itself.

## Important Notes

### ConfigureAwait Usage

```csharp
// In Blazor components (.razor.cs): NO ConfigureAwait(false)
protected override async Task OnInitializedAsync()
{
    this.data = await this.service.GetDataAsync(cancellationToken);
    // Need UI sync context to update component state
}

// In services called by components: YES ConfigureAwait(false)
public async Task<Data> GetDataAsync(CancellationToken cancellationToken)
{
    return await this.httpClient
        .GetFromJsonAsync<Data>("/api/data", cancellationToken)
        .ConfigureAwait(false);
}
```

### Component Lifecycle

| Method | When Called | Use For |
|--------|-------------|---------|
| `OnInitialized` | Once, sync | Quick initialization |
| `OnInitializedAsync` | Once, async | Data loading |
| `OnParametersSet` | When parameters change | React to param changes |
| `OnAfterRender` | After each render | DOM manipulation |
| `Dispose` | Component removed | Cleanup, cancel operations |

### State Management

- Use `StateHasChanged()` sparingly
- Prefer `@bind` for two-way binding
- Consider using a state container for complex state

### Error Boundaries

```razor
<ErrorBoundary>
    <ChildContent>
        <RiskyComponent />
    </ChildContent>
    <ErrorContent Context="exception">
        <div class="error">
            <p>Something went wrong: @exception.Message</p>
        </div>
    </ErrorContent>
</ErrorBoundary>
```

### Performance Tips

1. Use `@key` for list rendering
2. Implement `IDisposable` to clean up resources
3. Use `virtualization` for large lists
4. Avoid unnecessary `StateHasChanged()` calls
5. Use `[EditorRequired]` for required parameters
