# Web API Projects

Standards and patterns for ASP.NET Core Web API projects.

## Project Structure

```
src/
├── {CompanyName}.ProjectName.Domain/
│   ├── Entities/
│   ├── Exceptions/
│   └── ValueObjects/
├── {CompanyName}.ProjectName.Application/
│   ├── Interfaces/
│   └── Services/
├── {CompanyName}.ProjectName.Infrastructure/
│   ├── Persistence/
│   ├── ExternalServices/
│   └── DependencyInjection/
└── {CompanyName}.ProjectName.Api/
    ├── Configuration/
    │   └── SentryConfiguration.cs
    ├── Controllers/
    │   └── V1/
    ├── DTOs/
    │   └── V1/
    ├── Middleware/
    └── Program.cs
```

## Project File (.csproj)

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
    <PackageReference Include="Serilog.Sinks.Seq" Version="7.0.0" />
    <PackageReference Include="Serilog.Enrichers.Environment" Version="2.3.0" />
    <PackageReference Include="Serilog.Enrichers.Thread" Version="3.1.0" />
    <PackageReference Include="Serilog.Settings.Configuration" Version="8.0.0" />
    
    <!-- Sentry -->
    <PackageReference Include="Sentry.AspNetCore" Version="4.3.0" />
    <PackageReference Include="Sentry.Serilog" Version="4.3.0" />
    
    <!-- API -->
    <PackageReference Include="Swashbuckle.AspNetCore" Version="6.5.0" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\{CompanyName}.ProjectName.Application\{CompanyName}.ProjectName.Application.csproj" />
    <ProjectReference Include="..\{CompanyName}.ProjectName.Infrastructure\{CompanyName}.ProjectName.Infrastructure.csproj" />
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

using {CompanyName}.ProjectName.Api.Configuration;
using {CompanyName}.ProjectName.Application;
using {CompanyName}.ProjectName.Infrastructure;

// Bootstrap logger for startup errors
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Information)
    .Enrich.FromLogContext()
    .WriteTo.Console()
    .CreateBootstrapLogger();

try
{
    Log.Information("Starting {CompanyName}.ProjectName.Api");

    var builder = WebApplication.CreateBuilder(args);

    // Configure Serilog (REPLACES default logging)
    builder.Host.UseSerilog((context, services, configuration) => configuration
        .ReadFrom.Configuration(context.Configuration)
        .ReadFrom.Services(services)
        .Enrich.FromLogContext()
        .WriteTo.Sentry(options =>
        {
            options.Dsn = context.Configuration["Sentry:Dsn"];
            options.Environment = context.Configuration["Sentry:Environment"];
            options.MinimumBreadcrumbLevel = LogEventLevel.Debug;
            options.MinimumEventLevel = LogEventLevel.Error;
        }));

    // Configure Sentry
    builder.WebHost.UseSentry(options =>
    {
        options.Dsn = builder.Configuration["Sentry:Dsn"];
        options.Environment = builder.Configuration["Sentry:Environment"];
        options.TracesSampleRate = 1.0;
        options.AttachStacktrace = true;
        options.SendDefaultPii = false;
        options.MinimumEventLevel = LogLevel.Error;
    });

    // Add services
    builder.Services.AddApplicationServices();
    builder.Services.AddInfrastructureServices(builder.Configuration);
    builder.Services.AddControllers();
    builder.Services.AddEndpointsApiExplorer();
    builder.Services.AddSwaggerGen();

    var app = builder.Build();

    // Use Serilog request logging
    app.UseSerilogRequestLogging(options =>
    {
        options.EnrichDiagnosticContext = (diagnosticContext, httpContext) =>
        {
            diagnosticContext.Set("RequestHost", httpContext.Request.Host.Value);
            diagnosticContext.Set("UserAgent", httpContext.Request.Headers.UserAgent.ToString());
        };
    });

    // Use Sentry
    app.UseSentryTracing();

    if (app.Environment.IsDevelopment())
    {
        app.UseSwagger();
        app.UseSwaggerUI();
    }

    app.UseHttpsRedirection();
    app.UseAuthorization();
    app.MapControllers();

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

## Controller Template

```csharp
// <copyright file="DataPointsController.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.ProjectName.Api.Controllers.V1;

using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

using {CompanyName}.ProjectName.Api.DTOs.V1.DataPoints;
using {CompanyName}.ProjectName.Application.Interfaces;

/// <summary>
/// Provides endpoints for managing data points.
/// </summary>
[Authorize(AuthenticationSchemes = "ApiKey")]
[ApiController]
[Route("api/v1/[controller]")]
[Produces("application/json")]
public class DataPointsController : ControllerBase
{
    private readonly ILogger<DataPointsController> logger;
    private readonly IDataPointService service;

    /// <summary>
    /// Initializes a new instance of the <see cref="DataPointsController"/> class.
    /// </summary>
    /// <param name="logger">The logger instance.</param>
    /// <param name="service">The data point service.</param>
    public DataPointsController(
        ILogger<DataPointsController> logger,
        IDataPointService service)
    {
        this.logger = logger ?? throw new ArgumentNullException(nameof(logger));
        this.service = service ?? throw new ArgumentNullException(nameof(service));
    }

    /// <summary>
    /// Gets a data point by ID.
    /// </summary>
    /// <param name="id">The data point ID.</param>
    /// <param name="cancellationToken">The cancellation token.</param>
    /// <returns>The data point.</returns>
    /// <response code="200">Returns the data point.</response>
    /// <response code="404">Data point not found.</response>
    /// <response code="401">Unauthorized.</response>
    [HttpGet("{id}")]
    [ProducesResponseType(typeof(DataPointResponse), StatusCodes.Status200OK)]
    [ProducesResponseType(typeof(ErrorResponse), StatusCodes.Status404NotFound)]
    [ProducesResponseType(typeof(ErrorResponse), StatusCodes.Status401Unauthorized)]
    public async Task<IActionResult> GetByIdAsync(
        string id,
        CancellationToken cancellationToken)
    {
        // NOTE: Do NOT use ConfigureAwait(false) in controllers
        this.logger.LogDebug("Getting data point {DataPointId}", id);

        var result = await this.service.GetByIdAsync(id, cancellationToken);

        return this.Ok(result);
    }

    /// <summary>
    /// Creates a new data point.
    /// </summary>
    /// <param name="request">The create request.</param>
    /// <param name="cancellationToken">The cancellation token.</param>
    /// <returns>The created data point.</returns>
    /// <response code="201">Data point created.</response>
    /// <response code="400">Invalid request.</response>
    [HttpPost]
    [ProducesResponseType(typeof(DataPointResponse), StatusCodes.Status201Created)]
    [ProducesResponseType(typeof(ValidationErrorResponse), StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> CreateAsync(
        [FromBody] CreateDataPointRequest request,
        CancellationToken cancellationToken)
    {
        this.logger.LogInformation(
            "Creating data point of type {Type}",
            request.Type);

        var result = await this.service.CreateAsync(request, cancellationToken);

        return this.CreatedAtAction(
            nameof(this.GetByIdAsync),
            new { id = result.Id },
            result);
    }

    /// <summary>
    /// Updates a data point.
    /// </summary>
    /// <param name="id">The data point ID.</param>
    /// <param name="request">The update request.</param>
    /// <param name="cancellationToken">The cancellation token.</param>
    /// <returns>The updated data point.</returns>
    [HttpPut("{id}")]
    [ProducesResponseType(typeof(DataPointResponse), StatusCodes.Status200OK)]
    [ProducesResponseType(typeof(ErrorResponse), StatusCodes.Status404NotFound)]
    [ProducesResponseType(typeof(ValidationErrorResponse), StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> UpdateAsync(
        string id,
        [FromBody] UpdateDataPointRequest request,
        CancellationToken cancellationToken)
    {
        this.logger.LogInformation("Updating data point {DataPointId}", id);

        var result = await this.service.UpdateAsync(id, request, cancellationToken);

        return this.Ok(result);
    }

    /// <summary>
    /// Deletes a data point.
    /// </summary>
    /// <param name="id">The data point ID.</param>
    /// <param name="cancellationToken">The cancellation token.</param>
    /// <returns>No content.</returns>
    [HttpDelete("{id}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(typeof(ErrorResponse), StatusCodes.Status404NotFound)]
    public async Task<IActionResult> DeleteAsync(
        string id,
        CancellationToken cancellationToken)
    {
        this.logger.LogInformation("Deleting data point {DataPointId}", id);

        await this.service.DeleteAsync(id, cancellationToken);

        return this.NoContent();
    }
}
```

## DTO Templates

### Request DTO

```csharp
// <copyright file="CreateDataPointRequest.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.ProjectName.Api.DTOs.V1.DataPoints;

using System.ComponentModel.DataAnnotations;

/// <summary>
/// Request to create a new data point.
/// </summary>
public class CreateDataPointRequest
{
    /// <summary>
    /// Gets or sets the data point type.
    /// </summary>
    [Required]
    [StringLength(100)]
    public string Type { get; set; } = string.Empty;

    /// <summary>
    /// Gets or sets the data point value.
    /// </summary>
    [Required]
    [StringLength(500)]
    public string Value { get; set; } = string.Empty;

    /// <summary>
    /// Gets or sets optional metadata.
    /// </summary>
    public Dictionary<string, string>? Metadata { get; set; }
}
```

### Response DTO

```csharp
// <copyright file="DataPointResponse.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.ProjectName.Api.DTOs.V1.DataPoints;

/// <summary>
/// Response containing data point details.
/// </summary>
public class DataPointResponse
{
    /// <summary>
    /// Gets or sets the data point ID.
    /// </summary>
    public string Id { get; set; } = string.Empty;

    /// <summary>
    /// Gets or sets the data point type.
    /// </summary>
    public string Type { get; set; } = string.Empty;

    /// <summary>
    /// Gets or sets the data point value.
    /// </summary>
    public string Value { get; set; } = string.Empty;

    /// <summary>
    /// Gets or sets the creation timestamp.
    /// </summary>
    public DateTimeOffset CreatedAt { get; set; }

    /// <summary>
    /// Gets or sets the last update timestamp.
    /// </summary>
    public DateTimeOffset? UpdatedAt { get; set; }
}
```

## API Versioning

### URL-Based Versioning Strategy

Controllers are organized by version:

```
Controllers/
├── V1/
│   ├── DataPointsController.cs
│   └── HealthController.cs
└── V2/
    └── DataPointsController.cs  # Future breaking changes
```

### When to Create V2

**Create a new version when:**
- Removing or renaming fields in request/response models
- Changing the structure of responses
- Removing endpoints
- Changing authentication mechanisms

**Do NOT create a new version for:**
- Adding optional fields
- Adding new endpoints
- Bug fixes
- Performance improvements

## Important Notes

- **ConfigureAwait**: Do NOT use `ConfigureAwait(false)` in controllers — you need HttpContext
- **Logging**: Use structured logging with Serilog
- **Error Handling**: Let exceptions propagate to global middleware; Sentry will capture them
- **Cancellation**: Always accept and pass `CancellationToken`
