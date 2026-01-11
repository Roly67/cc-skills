# Class Library Projects

Standards and patterns for .NET Class Libraries.

## Project Structure

```
src/
└── {CompanyName}.SharedKernel/
    ├── Abstractions/          # Interfaces and base classes
    ├── Extensions/            # Extension methods
    ├── Helpers/               # Utility classes
    ├── Models/                # Shared models/DTOs
    └── Exceptions/            # Custom exceptions
```

## Project File (.csproj)

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <ItemGroup>
    <!-- Logging abstraction only - NO Serilog dependency -->
    <PackageReference Include="Microsoft.Extensions.Logging.Abstractions" Version="10.0.0" />
  </ItemGroup>

  <!-- NuGet Package Configuration (if publishing) -->
  <PropertyGroup>
    <PackageId>{CompanyName}.SharedKernel</PackageId>
    <Version>1.0.0</Version>
    <Authors>{CompanyName}</Authors>
    <Company>{CompanyName}</Company>
    <Description>Shared utilities and abstractions for projects</Description>
    <PackageTags>{tags}</PackageTags>
    <RepositoryUrl>https://github.com/{organization}/{repository}</RepositoryUrl>
    <PackageLicenseExpression>MIT</PackageLicenseExpression>
    <IncludeSymbols>true</IncludeSymbols>
    <SymbolPackageFormat>snupkg</SymbolPackageFormat>
  </PropertyGroup>

</Project>
```

## Key Principle: No Host Dependencies

Class libraries should **NEVER** reference:

- ❌ `Serilog` (use `Microsoft.Extensions.Logging.Abstractions`)
- ❌ `Sentry` (host responsibility)
- ❌ `Microsoft.AspNetCore.*` (unless specifically an ASP.NET library)
- ❌ `Microsoft.Extensions.Hosting` (host responsibility)

This keeps libraries portable and testable.

## ConfigureAwait(false) is REQUIRED

All async methods in class libraries MUST use `ConfigureAwait(false)`:

```csharp
// ✅ CORRECT - Always use ConfigureAwait(false) in libraries
public async Task<Result> ProcessAsync(Input input, CancellationToken cancellationToken)
{
    var data = await this.repository
        .GetDataAsync(input.Id, cancellationToken)
        .ConfigureAwait(false);

    var transformed = await this.transformer
        .TransformAsync(data, cancellationToken)
        .ConfigureAwait(false);

    return new Result(transformed);
}

// ❌ WRONG - Missing ConfigureAwait(false)
public async Task<Result> ProcessAsync(Input input, CancellationToken cancellationToken)
{
    var data = await this.repository.GetDataAsync(input.Id, cancellationToken);
    return new Result(data);
}
```

## Extension Method Template

```csharp
// <copyright file="StringExtensions.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.SharedKernel.Extensions;

/// <summary>
/// Extension methods for <see cref="string"/>.
/// </summary>
public static class StringExtensions
{
    /// <summary>
    /// Truncates the string to the specified maximum length.
    /// </summary>
    /// <param name="value">The string to truncate.</param>
    /// <param name="maxLength">The maximum length.</param>
    /// <returns>The truncated string.</returns>
    /// <exception cref="ArgumentOutOfRangeException">
    /// Thrown when <paramref name="maxLength"/> is negative.
    /// </exception>
    public static string Truncate(this string? value, int maxLength)
    {
        if (maxLength < 0)
        {
            throw new ArgumentOutOfRangeException(
                nameof(maxLength),
                maxLength,
                "Maximum length cannot be negative.");
        }

        if (string.IsNullOrEmpty(value))
        {
            return string.Empty;
        }

        return value.Length <= maxLength ? value : value[..maxLength];
    }

    /// <summary>
    /// Converts the string to title case.
    /// </summary>
    /// <param name="value">The string to convert.</param>
    /// <returns>The title-cased string.</returns>
    public static string ToTitleCase(this string? value)
    {
        if (string.IsNullOrWhiteSpace(value))
        {
            return string.Empty;
        }

        return System.Globalization.CultureInfo.CurrentCulture.TextInfo.ToTitleCase(value.ToLower());
    }
}
```

## Interface Template

```csharp
// <copyright file="IRepository.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.SharedKernel.Abstractions;

/// <summary>
/// Generic repository interface for data access.
/// </summary>
/// <typeparam name="T">The entity type.</typeparam>
public interface IRepository<T>
    where T : class
{
    /// <summary>
    /// Gets an entity by ID.
    /// </summary>
    /// <param name="id">The entity ID.</param>
    /// <param name="cancellationToken">The cancellation token.</param>
    /// <returns>The entity, or null if not found.</returns>
    Task<T?> GetByIdAsync(string id, CancellationToken cancellationToken);

    /// <summary>
    /// Gets all entities.
    /// </summary>
    /// <param name="cancellationToken">The cancellation token.</param>
    /// <returns>All entities.</returns>
    Task<IReadOnlyList<T>> GetAllAsync(CancellationToken cancellationToken);

    /// <summary>
    /// Adds a new entity.
    /// </summary>
    /// <param name="entity">The entity to add.</param>
    /// <param name="cancellationToken">The cancellation token.</param>
    /// <returns>A task representing the operation.</returns>
    Task AddAsync(T entity, CancellationToken cancellationToken);

    /// <summary>
    /// Updates an existing entity.
    /// </summary>
    /// <param name="entity">The entity to update.</param>
    /// <param name="cancellationToken">The cancellation token.</param>
    /// <returns>A task representing the operation.</returns>
    Task UpdateAsync(T entity, CancellationToken cancellationToken);

    /// <summary>
    /// Deletes an entity by ID.
    /// </summary>
    /// <param name="id">The entity ID.</param>
    /// <param name="cancellationToken">The cancellation token.</param>
    /// <returns>A task representing the operation.</returns>
    Task DeleteAsync(string id, CancellationToken cancellationToken);

    /// <summary>
    /// Checks if an entity exists.
    /// </summary>
    /// <param name="id">The entity ID.</param>
    /// <param name="cancellationToken">The cancellation token.</param>
    /// <returns>True if the entity exists; otherwise false.</returns>
    Task<bool> ExistsAsync(string id, CancellationToken cancellationToken);
}
```

## Service Base Class

```csharp
// <copyright file="ServiceBase.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.SharedKernel.Abstractions;

using Microsoft.Extensions.Logging;

/// <summary>
/// Base class for services providing common functionality.
/// </summary>
/// <typeparam name="T">The service type for logging.</typeparam>
public abstract class ServiceBase<T>
    where T : class
{
    /// <summary>
    /// Initializes a new instance of the <see cref="ServiceBase{T}"/> class.
    /// </summary>
    /// <param name="logger">The logger instance.</param>
    protected ServiceBase(ILogger<T> logger)
    {
        this.Logger = logger ?? throw new ArgumentNullException(nameof(logger));
    }

    /// <summary>
    /// Gets the logger instance.
    /// </summary>
    protected ILogger<T> Logger { get; }

    /// <summary>
    /// Executes an operation with standardized error handling and logging.
    /// </summary>
    /// <typeparam name="TResult">The result type.</typeparam>
    /// <param name="operation">The operation to execute.</param>
    /// <param name="operationName">The name of the operation for logging.</param>
    /// <param name="cancellationToken">The cancellation token.</param>
    /// <returns>The operation result.</returns>
    protected async Task<TResult> ExecuteAsync<TResult>(
        Func<CancellationToken, Task<TResult>> operation,
        string operationName,
        CancellationToken cancellationToken)
    {
        this.Logger.LogDebug("Starting {OperationName}", operationName);

        try
        {
            var result = await operation(cancellationToken).ConfigureAwait(false);

            this.Logger.LogDebug("Completed {OperationName}", operationName);

            return result;
        }
        catch (Exception ex) when (ex is not OperationCanceledException)
        {
            this.Logger.LogError(ex, "Failed {OperationName}", operationName);
            throw;
        }
    }
}
```

## Custom Exception Template

```csharp
// <copyright file="NotFoundException.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.SharedKernel.Exceptions;

/// <summary>
/// Exception thrown when a requested resource is not found.
/// </summary>
public class NotFoundException : Exception
{
    /// <summary>
    /// Initializes a new instance of the <see cref="NotFoundException"/> class.
    /// </summary>
    /// <param name="resourceType">The type of resource not found.</param>
    /// <param name="resourceId">The ID of the resource not found.</param>
    public NotFoundException(string resourceType, string resourceId)
        : base($"{resourceType} with ID '{resourceId}' was not found.")
    {
        this.ResourceType = resourceType;
        this.ResourceId = resourceId;
    }

    /// <summary>
    /// Initializes a new instance of the <see cref="NotFoundException"/> class.
    /// </summary>
    /// <param name="message">The exception message.</param>
    public NotFoundException(string message)
        : base(message)
    {
        this.ResourceType = string.Empty;
        this.ResourceId = string.Empty;
    }

    /// <summary>
    /// Gets the type of resource not found.
    /// </summary>
    public string ResourceType { get; }

    /// <summary>
    /// Gets the ID of the resource not found.
    /// </summary>
    public string ResourceId { get; }
}
```

## Result Pattern

```csharp
// <copyright file="Result.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.SharedKernel.Models;

/// <summary>
/// Represents the result of an operation.
/// </summary>
/// <typeparam name="T">The type of the value.</typeparam>
public class Result<T>
{
    private Result(bool isSuccess, T? value, string? error)
    {
        this.IsSuccess = isSuccess;
        this.Value = value;
        this.Error = error;
    }

    /// <summary>
    /// Gets a value indicating whether the operation was successful.
    /// </summary>
    public bool IsSuccess { get; }

    /// <summary>
    /// Gets a value indicating whether the operation failed.
    /// </summary>
    public bool IsFailure => !this.IsSuccess;

    /// <summary>
    /// Gets the result value.
    /// </summary>
    public T? Value { get; }

    /// <summary>
    /// Gets the error message if the operation failed.
    /// </summary>
    public string? Error { get; }

    /// <summary>
    /// Creates a successful result.
    /// </summary>
    /// <param name="value">The result value.</param>
    /// <returns>A successful result.</returns>
    public static Result<T> Success(T value) => new(true, value, null);

    /// <summary>
    /// Creates a failed result.
    /// </summary>
    /// <param name="error">The error message.</param>
    /// <returns>A failed result.</returns>
    public static Result<T> Failure(string error) => new(false, default, error);
}
```

## Guard Clauses

```csharp
// <copyright file="Guard.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.SharedKernel.Helpers;

/// <summary>
/// Provides guard clause methods for parameter validation.
/// </summary>
public static class Guard
{
    /// <summary>
    /// Throws if the value is null.
    /// </summary>
    /// <typeparam name="T">The value type.</typeparam>
    /// <param name="value">The value to check.</param>
    /// <param name="parameterName">The parameter name.</param>
    /// <returns>The value if not null.</returns>
    /// <exception cref="ArgumentNullException">Thrown when value is null.</exception>
    public static T AgainstNull<T>(T? value, string parameterName)
        where T : class
    {
        return value ?? throw new ArgumentNullException(parameterName);
    }

    /// <summary>
    /// Throws if the string is null or empty.
    /// </summary>
    /// <param name="value">The value to check.</param>
    /// <param name="parameterName">The parameter name.</param>
    /// <returns>The value if not null or empty.</returns>
    /// <exception cref="ArgumentException">Thrown when value is null or empty.</exception>
    public static string AgainstNullOrEmpty(string? value, string parameterName)
    {
        if (string.IsNullOrEmpty(value))
        {
            throw new ArgumentException("Value cannot be null or empty.", parameterName);
        }

        return value;
    }

    /// <summary>
    /// Throws if the string is null, empty, or whitespace.
    /// </summary>
    /// <param name="value">The value to check.</param>
    /// <param name="parameterName">The parameter name.</param>
    /// <returns>The value if not null, empty, or whitespace.</returns>
    /// <exception cref="ArgumentException">Thrown when value is null, empty, or whitespace.</exception>
    public static string AgainstNullOrWhiteSpace(string? value, string parameterName)
    {
        if (string.IsNullOrWhiteSpace(value))
        {
            throw new ArgumentException("Value cannot be null, empty, or whitespace.", parameterName);
        }

        return value;
    }

    /// <summary>
    /// Throws if the value is out of range.
    /// </summary>
    /// <param name="value">The value to check.</param>
    /// <param name="min">The minimum value (inclusive).</param>
    /// <param name="max">The maximum value (inclusive).</param>
    /// <param name="parameterName">The parameter name.</param>
    /// <returns>The value if in range.</returns>
    /// <exception cref="ArgumentOutOfRangeException">Thrown when value is out of range.</exception>
    public static int AgainstOutOfRange(int value, int min, int max, string parameterName)
    {
        if (value < min || value > max)
        {
            throw new ArgumentOutOfRangeException(
                parameterName,
                value,
                $"Value must be between {min} and {max}.");
        }

        return value;
    }
}
```

## Important Notes

- **No Serilog**: Use `ILogger<T>` from `Microsoft.Extensions.Logging.Abstractions`
- **No Sentry**: Exception handling is the host's responsibility
- **ConfigureAwait(false)**: Required on ALL async calls
- **Minimal dependencies**: Keep the dependency graph small
- **XML Documentation**: Required for all public members (especially if publishing as NuGet)
- **Semantic Versioning**: Follow SemVer for package versions
