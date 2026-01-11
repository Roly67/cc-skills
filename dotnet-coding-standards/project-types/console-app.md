# Console Application Projects

Standards and patterns for .NET Console applications.

## Project Structure

```
src/
├── {CompanyName}.ProjectName.Console/
│   ├── Commands/              # CLI commands (if using command pattern)
│   ├── Configuration/         # App configuration
│   ├── Services/              # Business logic
│   └── Program.cs
├── {CompanyName}.ProjectName.Core/      # Shared business logic (optional)
│   ├── Interfaces/
│   └── Models/
└── tests/
    └── {CompanyName}.ProjectName.Console.Tests/
```

## Project File (.csproj)

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
  </PropertyGroup>

  <!-- Optional: For self-contained single-file deployment -->
  <PropertyGroup Condition="'$(Configuration)' == 'Release'">
    <PublishSingleFile>true</PublishSingleFile>
    <SelfContained>true</SelfContained>
    <RuntimeIdentifier>win-x64</RuntimeIdentifier>
    <PublishTrimmed>true</PublishTrimmed>
  </PropertyGroup>

  <ItemGroup>
    <!-- Serilog -->
    <PackageReference Include="Serilog" Version="4.0.0" />
    <PackageReference Include="Serilog.Extensions.Hosting" Version="8.0.0" />
    <PackageReference Include="Serilog.Sinks.Console" Version="5.0.1" />
    <PackageReference Include="Serilog.Sinks.File" Version="5.0.0" />
    <PackageReference Include="Serilog.Settings.Configuration" Version="8.0.0" />
    
    <!-- Sentry -->
    <PackageReference Include="Sentry" Version="4.3.0" />
    <PackageReference Include="Sentry.Serilog" Version="4.3.0" />
    
    <!-- Hosting -->
    <PackageReference Include="Microsoft.Extensions.Hosting" Version="10.0.0" />
    
    <!-- Optional: For advanced CLI -->
    <PackageReference Include="System.CommandLine" Version="2.0.0-beta4.22272.1" />
  </ItemGroup>

</Project>
```

## Program.cs Template (Simple)

```csharp
// <copyright file="Program.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

using Serilog;
using Serilog.Events;

using {CompanyName}.DataProcessor.Console;
using {CompanyName}.DataProcessor.Console.Services;

// Bootstrap logger
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Debug()
    .WriteTo.Console()
    .CreateBootstrapLogger();

try
{
    Log.Information("Starting {CompanyName}.DataProcessor.Console");

    var builder = Host.CreateApplicationBuilder(args);

    // Configure Serilog (REPLACES default logging)
    builder.Services.AddSerilog((services, configuration) => configuration
        .ReadFrom.Configuration(builder.Configuration)
        .ReadFrom.Services(services)
        .Enrich.FromLogContext()
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

    // Add services
    builder.Services.AddSingleton<IDataProcessor, DataProcessor>();
    builder.Services.AddSingleton<Application>();

    var host = builder.Build();

    var app = host.Services.GetRequiredService<Application>();
    var exitCode = await app.RunAsync(args, CancellationToken.None);

    return exitCode;
}
catch (Exception ex)
{
    Log.Fatal(ex, "Application terminated unexpectedly");
    return 1;
}
finally
{
    await Log.CloseAndFlushAsync();
}
```

## Application Entry Point Class

```csharp
// <copyright file="Application.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.DataProcessor.Console;

using Microsoft.Extensions.Logging;

/// <summary>
/// Main application entry point.
/// </summary>
public class Application
{
    private readonly ILogger<Application> logger;
    private readonly IDataProcessor processor;

    /// <summary>
    /// Initializes a new instance of the <see cref="Application"/> class.
    /// </summary>
    /// <param name="logger">The logger instance.</param>
    /// <param name="processor">The data processor.</param>
    public Application(ILogger<Application> logger, IDataProcessor processor)
    {
        this.logger = logger ?? throw new ArgumentNullException(nameof(logger));
        this.processor = processor ?? throw new ArgumentNullException(nameof(processor));
    }

    /// <summary>
    /// Runs the application.
    /// </summary>
    /// <param name="args">Command line arguments.</param>
    /// <param name="cancellationToken">The cancellation token.</param>
    /// <returns>Exit code (0 for success, non-zero for failure).</returns>
    public async Task<int> RunAsync(string[] args, CancellationToken cancellationToken)
    {
        try
        {
            this.logger.LogInformation("Application starting with {ArgCount} arguments", args.Length);

            // Use ConfigureAwait(false) in the Application class
            await this.processor.ProcessAsync(cancellationToken).ConfigureAwait(false);

            this.logger.LogInformation("Application completed successfully");
            return ExitCodes.Success;
        }
        catch (ValidationException ex)
        {
            this.logger.LogError(ex, "Validation error: {Message}", ex.Message);
            return ExitCodes.ValidationError;
        }
        catch (OperationCanceledException)
        {
            this.logger.LogWarning("Operation was cancelled");
            return ExitCodes.Cancelled;
        }
        catch (Exception ex)
        {
            this.logger.LogError(ex, "Application failed with error");
            return ExitCodes.UnexpectedError;
        }
    }
}
```

## Exit Codes

```csharp
// <copyright file="ExitCodes.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.DataProcessor.Console;

/// <summary>
/// Standard exit codes for the application.
/// </summary>
public static class ExitCodes
{
    /// <summary>
    /// Success - operation completed successfully.
    /// </summary>
    public const int Success = 0;

    /// <summary>
    /// Validation error - invalid input or arguments.
    /// </summary>
    public const int ValidationError = 1;

    /// <summary>
    /// Not found - requested resource not found.
    /// </summary>
    public const int NotFound = 2;

    /// <summary>
    /// Cancelled - operation was cancelled.
    /// </summary>
    public const int Cancelled = 3;

    /// <summary>
    /// External service error - dependency failure.
    /// </summary>
    public const int ExternalServiceError = 4;

    /// <summary>
    /// Configuration error - missing or invalid configuration.
    /// </summary>
    public const int ConfigurationError = 5;

    /// <summary>
    /// Unexpected error - unhandled exception.
    /// </summary>
    public const int UnexpectedError = 99;
}
```

## Program.cs with System.CommandLine (Advanced CLI)

```csharp
// <copyright file="Program.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

using System.CommandLine;

using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

using Serilog;
using Serilog.Events;

// Bootstrap logger
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Debug()
    .WriteTo.Console()
    .CreateBootstrapLogger();

try
{
    Log.Information("Starting {CompanyName}.DataProcessor.Console");

    var builder = Host.CreateApplicationBuilder(args);

    // Configure Serilog
    builder.Services.AddSerilog((services, configuration) => configuration
        .ReadFrom.Configuration(builder.Configuration)
        .Enrich.FromLogContext()
        .WriteTo.Sentry(options =>
        {
            options.Dsn = builder.Configuration["Sentry:Dsn"];
            options.MinimumEventLevel = LogEventLevel.Error;
        }));

    // Add services
    builder.Services.AddSingleton<IDataProcessor, DataProcessor>();

    var host = builder.Build();

    // Define CLI
    var rootCommand = new RootCommand("{ProjectDescription}");

    // Process command
    var inputOption = new Option<FileInfo>(
        aliases: ["--input", "-i"],
        description: "Input file to process")
    {
        IsRequired = true,
    };

    var outputOption = new Option<FileInfo>(
        aliases: ["--output", "-o"],
        description: "Output file path")
    {
        IsRequired = true,
    };

    var verboseOption = new Option<bool>(
        aliases: ["--verbose", "-v"],
        description: "Enable verbose output");

    var processCommand = new Command("process", "Process a data file")
    {
        inputOption,
        outputOption,
        verboseOption,
    };

    processCommand.SetHandler(
        async (input, output, verbose) =>
        {
            var logger = host.Services.GetRequiredService<ILogger<Program>>();
            var processor = host.Services.GetRequiredService<IDataProcessor>();

            logger.LogInformation(
                "Processing {Input} to {Output}",
                input.FullName,
                output.FullName);

            await processor
                .ProcessFileAsync(input.FullName, output.FullName, CancellationToken.None)
                .ConfigureAwait(false);

            logger.LogInformation("Processing complete");
        },
        inputOption,
        outputOption,
        verboseOption);

    rootCommand.AddCommand(processCommand);

    // Validate command
    var validateCommand = new Command("validate", "Validate a data file without processing")
    {
        inputOption,
    };

    validateCommand.SetHandler(
        async (input) =>
        {
            var logger = host.Services.GetRequiredService<ILogger<Program>>();
            var processor = host.Services.GetRequiredService<IDataProcessor>();

            logger.LogInformation("Validating {Input}", input.FullName);

            var isValid = await processor
                .ValidateAsync(input.FullName, CancellationToken.None)
                .ConfigureAwait(false);

            if (isValid)
            {
                logger.LogInformation("File is valid");
            }
            else
            {
                logger.LogWarning("File is invalid");
            }
        },
        inputOption);

    rootCommand.AddCommand(validateCommand);

    return await rootCommand.InvokeAsync(args);
}
catch (Exception ex)
{
    Log.Fatal(ex, "Application terminated unexpectedly");
    return 1;
}
finally
{
    await Log.CloseAndFlushAsync();
}
```

## Command Pattern (Alternative to System.CommandLine)

### ICommand Interface

```csharp
// <copyright file="ICommand.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.DataProcessor.Console.Commands;

/// <summary>
/// Represents a CLI command.
/// </summary>
public interface ICommand
{
    /// <summary>
    /// Gets the command name.
    /// </summary>
    string Name { get; }

    /// <summary>
    /// Gets the command description.
    /// </summary>
    string Description { get; }

    /// <summary>
    /// Executes the command.
    /// </summary>
    /// <param name="args">Command arguments.</param>
    /// <param name="cancellationToken">The cancellation token.</param>
    /// <returns>Exit code.</returns>
    Task<int> ExecuteAsync(string[] args, CancellationToken cancellationToken);
}
```

### Command Implementation

```csharp
// <copyright file="ProcessCommand.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.DataProcessor.Console.Commands;

using Microsoft.Extensions.Logging;

/// <summary>
/// Command to process data files.
/// </summary>
public class ProcessCommand : ICommand
{
    private readonly ILogger<ProcessCommand> logger;
    private readonly IDataProcessor processor;

    /// <summary>
    /// Initializes a new instance of the <see cref="ProcessCommand"/> class.
    /// </summary>
    /// <param name="logger">The logger instance.</param>
    /// <param name="processor">The data processor.</param>
    public ProcessCommand(ILogger<ProcessCommand> logger, IDataProcessor processor)
    {
        this.logger = logger ?? throw new ArgumentNullException(nameof(logger));
        this.processor = processor ?? throw new ArgumentNullException(nameof(processor));
    }

    /// <inheritdoc/>
    public string Name => "process";

    /// <inheritdoc/>
    public string Description => "Process data files from the specified directory";

    /// <inheritdoc/>
    public async Task<int> ExecuteAsync(string[] args, CancellationToken cancellationToken)
    {
        if (args.Length < 1)
        {
            this.logger.LogError("Usage: process <input-path> [output-path]");
            return ExitCodes.ValidationError;
        }

        var inputPath = args[0];
        var outputPath = args.Length > 1 ? args[1] : "./output";

        this.logger.LogInformation(
            "Processing files from {InputPath} to {OutputPath}",
            inputPath,
            outputPath);

        await this.processor
            .ProcessDirectoryAsync(inputPath, outputPath, cancellationToken)
            .ConfigureAwait(false);

        return ExitCodes.Success;
    }
}
```

## appsettings.json for Console Apps

```json
{
  "Serilog": {
    "Using": [
      "Serilog.Sinks.Console",
      "Serilog.Sinks.File"
    ],
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "System": "Warning"
      }
    },
    "WriteTo": [
      {
        "Name": "Console",
        "Args": {
          "outputTemplate": "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj}{NewLine}{Exception}"
        }
      },
      {
        "Name": "File",
        "Args": {
          "path": "logs/processor-.log",
          "rollingInterval": "Day",
          "retainedFileCountLimit": 7
        }
      }
    ],
    "Enrich": ["FromLogContext"],
    "Properties": {
      "Application": "{CompanyName}.DataProcessor"
    }
  },
  "Sentry": {
    "Dsn": "",
    "Environment": "Development"
  },
  "Processing": {
    "BatchSize": 100,
    "MaxRetries": 3
  }
}
```

## Important Notes

- **Exit Codes**: Always return meaningful exit codes for scripting/automation
- **ConfigureAwait**: Use `ConfigureAwait(false)` in helper methods (but not in Program.cs Main)
- **Cancellation**: Support Ctrl+C gracefully with CancellationToken
- **Logging**: Initialize Serilog before any other code runs
- **Single-file publish**: Use for deployment when appropriate
