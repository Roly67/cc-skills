# Error Handling Standards

Standards for handling errors across application layers, including exception hierarchies, the Result pattern, and RFC 7807 Problem Details.

## Overview

Error handling should be:
- **Consistent** - Same error types produce same response formats
- **Informative** - Enough detail for debugging, not too much for security
- **Layer-appropriate** - Domain exceptions don't leak to API responses

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Domain    │───▶│ Application │───▶│    API      │───▶│   Client    │
│  Exceptions │    │ Translation │    │  Problem    │    │  Response   │
│             │    │             │    │  Details    │    │             │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
```

---

## Exception Hierarchy

### Domain Layer Exceptions

```csharp
// <copyright file="DomainException.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Domain.Exceptions;

/// <summary>
/// Base exception for domain rule violations.
/// </summary>
public class DomainException : Exception
{
    /// <summary>
    /// Initializes a new instance of the <see cref="DomainException"/> class.
    /// </summary>
    public DomainException(string message)
        : base(message)
    {
    }

    /// <summary>
    /// Initializes a new instance of the <see cref="DomainException"/> class.
    /// </summary>
    public DomainException(string message, Exception innerException)
        : base(message, innerException)
    {
    }
}
```

### Specific Domain Exceptions

```csharp
// <copyright file="DomainExceptions.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Domain.Exceptions;

/// <summary>
/// Thrown when an entity is not found.
/// </summary>
public sealed class EntityNotFoundException : DomainException
{
    /// <summary>
    /// Initializes a new instance of the <see cref="EntityNotFoundException"/> class.
    /// </summary>
    public EntityNotFoundException(string entityName, object id)
        : base($"{entityName} with ID '{id}' was not found.")
    {
        this.EntityName = entityName;
        this.EntityId = id;
    }

    /// <summary>
    /// Gets the entity name.
    /// </summary>
    public string EntityName { get; }

    /// <summary>
    /// Gets the entity ID.
    /// </summary>
    public object EntityId { get; }
}

/// <summary>
/// Thrown when a business rule is violated.
/// </summary>
public sealed class BusinessRuleException : DomainException
{
    /// <summary>
    /// Initializes a new instance of the <see cref="BusinessRuleException"/> class.
    /// </summary>
    public BusinessRuleException(string ruleCode, string message)
        : base(message)
    {
        this.RuleCode = ruleCode;
    }

    /// <summary>
    /// Gets the rule code.
    /// </summary>
    public string RuleCode { get; }
}

/// <summary>
/// Thrown when there is a concurrency conflict.
/// </summary>
public sealed class ConcurrencyException : DomainException
{
    /// <summary>
    /// Initializes a new instance of the <see cref="ConcurrencyException"/> class.
    /// </summary>
    public ConcurrencyException(string entityName, object id)
        : base($"{entityName} with ID '{id}' was modified by another user.")
    {
        this.EntityName = entityName;
        this.EntityId = id;
    }

    /// <summary>
    /// Gets the entity name.
    /// </summary>
    public string EntityName { get; }

    /// <summary>
    /// Gets the entity ID.
    /// </summary>
    public object EntityId { get; }
}

/// <summary>
/// Thrown when the user is not authorized.
/// </summary>
public sealed class UnauthorizedException : DomainException
{
    /// <summary>
    /// Initializes a new instance of the <see cref="UnauthorizedException"/> class.
    /// </summary>
    public UnauthorizedException(string message = "You are not authorized to perform this action.")
        : base(message)
    {
    }
}

/// <summary>
/// Thrown when the user is forbidden from accessing a resource.
/// </summary>
public sealed class ForbiddenException : DomainException
{
    /// <summary>
    /// Initializes a new instance of the <see cref="ForbiddenException"/> class.
    /// </summary>
    public ForbiddenException(string resource)
        : base($"Access to '{resource}' is forbidden.")
    {
        this.Resource = resource;
    }

    /// <summary>
    /// Gets the resource that was forbidden.
    /// </summary>
    public string Resource { get; }
}
```

### Application Layer Exceptions

```csharp
// <copyright file="ApplicationException.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Application.Exceptions;

/// <summary>
/// Base exception for application layer errors.
/// </summary>
public class ApplicationException : Exception
{
    /// <summary>
    /// Initializes a new instance of the <see cref="ApplicationException"/> class.
    /// </summary>
    public ApplicationException(string message)
        : base(message)
    {
    }

    /// <summary>
    /// Initializes a new instance of the <see cref="ApplicationException"/> class.
    /// </summary>
    public ApplicationException(string message, Exception innerException)
        : base(message, innerException)
    {
    }
}

/// <summary>
/// Thrown when validation fails.
/// </summary>
public sealed class ValidationException : ApplicationException
{
    /// <summary>
    /// Initializes a new instance of the <see cref="ValidationException"/> class.
    /// </summary>
    public ValidationException(IEnumerable<ValidationError> errors)
        : base("One or more validation errors occurred.")
    {
        this.Errors = errors.ToList();
    }

    /// <summary>
    /// Gets the validation errors.
    /// </summary>
    public IReadOnlyList<ValidationError> Errors { get; }
}

/// <summary>
/// Represents a validation error.
/// </summary>
public sealed record ValidationError(string PropertyName, string ErrorMessage);
```

---

## Result Pattern

For operations where failure is expected, use the Result pattern instead of exceptions.

### Result Types

```csharp
// <copyright file="Result.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Domain.Primitives;

/// <summary>
/// Represents the result of an operation.
/// </summary>
public class Result
{
    protected Result(bool isSuccess, Error error)
    {
        if (isSuccess && error != Error.None)
        {
            throw new InvalidOperationException("Success result cannot have an error.");
        }

        if (!isSuccess && error == Error.None)
        {
            throw new InvalidOperationException("Failure result must have an error.");
        }

        this.IsSuccess = isSuccess;
        this.Error = error;
    }

    /// <summary>
    /// Gets a value indicating whether the operation succeeded.
    /// </summary>
    public bool IsSuccess { get; }

    /// <summary>
    /// Gets a value indicating whether the operation failed.
    /// </summary>
    public bool IsFailure => !this.IsSuccess;

    /// <summary>
    /// Gets the error if the operation failed.
    /// </summary>
    public Error Error { get; }

    /// <summary>
    /// Creates a success result.
    /// </summary>
    public static Result Success() => new(true, Error.None);

    /// <summary>
    /// Creates a failure result.
    /// </summary>
    public static Result Failure(Error error) => new(false, error);

    /// <summary>
    /// Creates a success result with a value.
    /// </summary>
    public static Result<TValue> Success<TValue>(TValue value) => new(value, true, Error.None);

    /// <summary>
    /// Creates a failure result with a value type.
    /// </summary>
    public static Result<TValue> Failure<TValue>(Error error) => new(default!, false, error);
}

/// <summary>
/// Represents the result of an operation with a value.
/// </summary>
/// <typeparam name="TValue">The type of the value.</typeparam>
public class Result<TValue> : Result
{
    private readonly TValue value;

    internal Result(TValue value, bool isSuccess, Error error)
        : base(isSuccess, error)
    {
        this.value = value;
    }

    /// <summary>
    /// Gets the value if the operation succeeded.
    /// </summary>
    /// <exception cref="InvalidOperationException">Thrown when accessing value of a failed result.</exception>
    public TValue Value => this.IsSuccess
        ? this.value
        : throw new InvalidOperationException("Cannot access value of a failed result.");

    /// <summary>
    /// Implicitly converts a value to a success result.
    /// </summary>
    public static implicit operator Result<TValue>(TValue value) => Success(value);
}
```

### Error Type

```csharp
// <copyright file="Error.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Domain.Primitives;

/// <summary>
/// Represents an error with a code and message.
/// </summary>
public sealed record Error(string Code, string Message)
{
    /// <summary>
    /// Represents no error.
    /// </summary>
    public static readonly Error None = new(string.Empty, string.Empty);

    /// <summary>
    /// Represents a null value error.
    /// </summary>
    public static readonly Error NullValue = new("Error.NullValue", "A null value was provided.");

    /// <summary>
    /// Implicitly converts an error to a result.
    /// </summary>
    public static implicit operator Result(Error error) => Result.Failure(error);
}
```

### Domain Errors

```csharp
// <copyright file="DomainErrors.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Domain.Errors;

using {CompanyName}.{ProjectName}.Domain.Primitives;

/// <summary>
/// Domain errors for orders.
/// </summary>
public static class OrderErrors
{
    /// <summary>
    /// Order not found error.
    /// </summary>
    public static Error NotFound(Guid id) =>
        new("Order.NotFound", $"Order with ID '{id}' was not found.");

    /// <summary>
    /// Order is empty error.
    /// </summary>
    public static readonly Error Empty =
        new("Order.Empty", "Order must contain at least one item.");

    /// <summary>
    /// Order cannot be modified error.
    /// </summary>
    public static Error CannotModify(string status) =>
        new("Order.CannotModify", $"Order in status '{status}' cannot be modified.");

    /// <summary>
    /// Order exceeds limit error.
    /// </summary>
    public static Error ExceedsLimit(decimal limit) =>
        new("Order.ExceedsLimit", $"Order total exceeds the limit of {limit:C}.");
}
```

### Using Result Pattern

```csharp
// <copyright file="OrderService.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Application.Services;

/// <summary>
/// Service for order operations.
/// </summary>
public class OrderService : IOrderService
{
    private readonly IOrderRepository repository;

    /// <inheritdoc/>
    public async Task<Result<Order>> GetByIdAsync(
        OrderId id,
        CancellationToken cancellationToken)
    {
        var order = await this.repository
            .GetByIdAsync(id, cancellationToken)
            .ConfigureAwait(false);

        if (order is null)
        {
            return Result.Failure<Order>(OrderErrors.NotFound(id.Value));
        }

        return order;
    }

    /// <inheritdoc/>
    public Result SubmitOrder(Order order)
    {
        if (!order.Items.Any())
        {
            return OrderErrors.Empty;
        }

        var total = order.CalculateTotal();
        if (total.Amount > 10000)
        {
            return OrderErrors.ExceedsLimit(10000);
        }

        order.Submit();
        return Result.Success();
    }
}
```

---

## Problem Details (RFC 7807)

### Problem Details Response

```csharp
// <copyright file="ProblemDetailsResponse.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Api.Models;

using System.Text.Json.Serialization;

/// <summary>
/// RFC 7807 Problem Details response.
/// </summary>
public class ProblemDetailsResponse
{
    /// <summary>
    /// Gets or sets the problem type URI.
    /// </summary>
    [JsonPropertyName("type")]
    public string Type { get; set; } = "about:blank";

    /// <summary>
    /// Gets or sets the short summary of the problem.
    /// </summary>
    [JsonPropertyName("title")]
    public string Title { get; set; } = string.Empty;

    /// <summary>
    /// Gets or sets the HTTP status code.
    /// </summary>
    [JsonPropertyName("status")]
    public int Status { get; set; }

    /// <summary>
    /// Gets or sets the detailed explanation.
    /// </summary>
    [JsonPropertyName("detail")]
    public string? Detail { get; set; }

    /// <summary>
    /// Gets or sets the URI of the specific occurrence.
    /// </summary>
    [JsonPropertyName("instance")]
    public string? Instance { get; set; }

    /// <summary>
    /// Gets or sets the trace ID for correlation.
    /// </summary>
    [JsonPropertyName("traceId")]
    public string? TraceId { get; set; }

    /// <summary>
    /// Gets or sets the error code.
    /// </summary>
    [JsonPropertyName("errorCode")]
    public string? ErrorCode { get; set; }

    /// <summary>
    /// Gets or sets validation errors.
    /// </summary>
    [JsonPropertyName("errors")]
    [JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingNull)]
    public IDictionary<string, string[]>? Errors { get; set; }
}
```

### Global Exception Handler

```csharp
// <copyright file="GlobalExceptionHandler.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Api.Middleware;

using System.Diagnostics;

using FluentValidation;
using Microsoft.AspNetCore.Diagnostics;

using {CompanyName}.{ProjectName}.Api.Models;
using {CompanyName}.{ProjectName}.Domain.Exceptions;
using ApplicationException = {CompanyName}.{ProjectName}.Application.Exceptions.ApplicationException;
using ValidationException = {CompanyName}.{ProjectName}.Application.Exceptions.ValidationException;

/// <summary>
/// Global exception handler that converts exceptions to Problem Details.
/// </summary>
public class GlobalExceptionHandler : IExceptionHandler
{
    private readonly ILogger<GlobalExceptionHandler> logger;

    /// <summary>
    /// Initializes a new instance of the <see cref="GlobalExceptionHandler"/> class.
    /// </summary>
    public GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger)
    {
        this.logger = logger ?? throw new ArgumentNullException(nameof(logger));
    }

    /// <inheritdoc/>
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        var traceId = Activity.Current?.Id ?? httpContext.TraceIdentifier;

        this.logger.LogError(
            exception,
            "Exception occurred. TraceId: {TraceId}",
            traceId);

        var problemDetails = exception switch
        {
            ValidationException validationEx => CreateValidationProblemDetails(validationEx, traceId),
            FluentValidation.ValidationException fluentEx => CreateFluentValidationProblemDetails(fluentEx, traceId),
            EntityNotFoundException notFoundEx => CreateNotFoundProblemDetails(notFoundEx, traceId),
            BusinessRuleException businessEx => CreateBusinessRuleProblemDetails(businessEx, traceId),
            UnauthorizedException => CreateUnauthorizedProblemDetails(traceId),
            ForbiddenException forbiddenEx => CreateForbiddenProblemDetails(forbiddenEx, traceId),
            ConcurrencyException concurrencyEx => CreateConflictProblemDetails(concurrencyEx, traceId),
            DomainException domainEx => CreateDomainProblemDetails(domainEx, traceId),
            ApplicationException appEx => CreateApplicationProblemDetails(appEx, traceId),
            _ => CreateInternalErrorProblemDetails(traceId),
        };

        httpContext.Response.StatusCode = problemDetails.Status;
        httpContext.Response.ContentType = "application/problem+json";

        await httpContext.Response.WriteAsJsonAsync(problemDetails, cancellationToken);

        return true;
    }

    private static ProblemDetailsResponse CreateValidationProblemDetails(
        ValidationException exception,
        string traceId)
    {
        return new ProblemDetailsResponse
        {
            Type = "https://tools.ietf.org/html/rfc7231#section-6.5.1",
            Title = "Validation Failed",
            Status = StatusCodes.Status400BadRequest,
            Detail = "One or more validation errors occurred.",
            TraceId = traceId,
            ErrorCode = "VALIDATION_ERROR",
            Errors = exception.Errors
                .GroupBy(e => e.PropertyName)
                .ToDictionary(
                    g => g.Key,
                    g => g.Select(e => e.ErrorMessage).ToArray()),
        };
    }

    private static ProblemDetailsResponse CreateFluentValidationProblemDetails(
        FluentValidation.ValidationException exception,
        string traceId)
    {
        return new ProblemDetailsResponse
        {
            Type = "https://tools.ietf.org/html/rfc7231#section-6.5.1",
            Title = "Validation Failed",
            Status = StatusCodes.Status400BadRequest,
            Detail = "One or more validation errors occurred.",
            TraceId = traceId,
            ErrorCode = "VALIDATION_ERROR",
            Errors = exception.Errors
                .GroupBy(e => e.PropertyName)
                .ToDictionary(
                    g => g.Key,
                    g => g.Select(e => e.ErrorMessage).ToArray()),
        };
    }

    private static ProblemDetailsResponse CreateNotFoundProblemDetails(
        EntityNotFoundException exception,
        string traceId)
    {
        return new ProblemDetailsResponse
        {
            Type = "https://tools.ietf.org/html/rfc7231#section-6.5.4",
            Title = "Resource Not Found",
            Status = StatusCodes.Status404NotFound,
            Detail = exception.Message,
            TraceId = traceId,
            ErrorCode = "NOT_FOUND",
        };
    }

    private static ProblemDetailsResponse CreateBusinessRuleProblemDetails(
        BusinessRuleException exception,
        string traceId)
    {
        return new ProblemDetailsResponse
        {
            Type = "https://tools.ietf.org/html/rfc7231#section-6.5.1",
            Title = "Business Rule Violation",
            Status = StatusCodes.Status400BadRequest,
            Detail = exception.Message,
            TraceId = traceId,
            ErrorCode = exception.RuleCode,
        };
    }

    private static ProblemDetailsResponse CreateUnauthorizedProblemDetails(string traceId)
    {
        return new ProblemDetailsResponse
        {
            Type = "https://tools.ietf.org/html/rfc7235#section-3.1",
            Title = "Unauthorized",
            Status = StatusCodes.Status401Unauthorized,
            Detail = "Authentication is required.",
            TraceId = traceId,
            ErrorCode = "UNAUTHORIZED",
        };
    }

    private static ProblemDetailsResponse CreateForbiddenProblemDetails(
        ForbiddenException exception,
        string traceId)
    {
        return new ProblemDetailsResponse
        {
            Type = "https://tools.ietf.org/html/rfc7231#section-6.5.3",
            Title = "Forbidden",
            Status = StatusCodes.Status403Forbidden,
            Detail = exception.Message,
            TraceId = traceId,
            ErrorCode = "FORBIDDEN",
        };
    }

    private static ProblemDetailsResponse CreateConflictProblemDetails(
        ConcurrencyException exception,
        string traceId)
    {
        return new ProblemDetailsResponse
        {
            Type = "https://tools.ietf.org/html/rfc7231#section-6.5.8",
            Title = "Conflict",
            Status = StatusCodes.Status409Conflict,
            Detail = exception.Message,
            TraceId = traceId,
            ErrorCode = "CONCURRENCY_CONFLICT",
        };
    }

    private static ProblemDetailsResponse CreateDomainProblemDetails(
        DomainException exception,
        string traceId)
    {
        return new ProblemDetailsResponse
        {
            Type = "https://tools.ietf.org/html/rfc7231#section-6.5.1",
            Title = "Domain Error",
            Status = StatusCodes.Status400BadRequest,
            Detail = exception.Message,
            TraceId = traceId,
            ErrorCode = "DOMAIN_ERROR",
        };
    }

    private static ProblemDetailsResponse CreateApplicationProblemDetails(
        ApplicationException exception,
        string traceId)
    {
        return new ProblemDetailsResponse
        {
            Type = "https://tools.ietf.org/html/rfc7231#section-6.6.1",
            Title = "Application Error",
            Status = StatusCodes.Status500InternalServerError,
            Detail = exception.Message,
            TraceId = traceId,
            ErrorCode = "APPLICATION_ERROR",
        };
    }

    private static ProblemDetailsResponse CreateInternalErrorProblemDetails(string traceId)
    {
        return new ProblemDetailsResponse
        {
            Type = "https://tools.ietf.org/html/rfc7231#section-6.6.1",
            Title = "Internal Server Error",
            Status = StatusCodes.Status500InternalServerError,
            Detail = "An unexpected error occurred. Please try again later.",
            TraceId = traceId,
            ErrorCode = "INTERNAL_ERROR",
        };
    }
}
```

### Registration

```csharp
// Program.cs
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();

// ...

app.UseExceptionHandler();
```

---

## Exception to HTTP Status Mapping

| Exception Type | HTTP Status | Error Code |
|----------------|-------------|------------|
| `ValidationException` | 400 Bad Request | `VALIDATION_ERROR` |
| `BusinessRuleException` | 400 Bad Request | Custom rule code |
| `DomainException` | 400 Bad Request | `DOMAIN_ERROR` |
| `UnauthorizedException` | 401 Unauthorized | `UNAUTHORIZED` |
| `ForbiddenException` | 403 Forbidden | `FORBIDDEN` |
| `EntityNotFoundException` | 404 Not Found | `NOT_FOUND` |
| `ConcurrencyException` | 409 Conflict | `CONCURRENCY_CONFLICT` |
| `ApplicationException` | 500 Internal Server Error | `APPLICATION_ERROR` |
| Other exceptions | 500 Internal Server Error | `INTERNAL_ERROR` |

---

## Best Practices

### Do

- ✅ Use specific exception types for different error scenarios
- ✅ Include correlation IDs (TraceId) in all error responses
- ✅ Log full exception details server-side
- ✅ Return generic messages for unexpected errors
- ✅ Use Result pattern for expected failures
- ✅ Define error codes for client consumption

### Don't

- ❌ Expose stack traces to clients
- ❌ Log sensitive data in exceptions
- ❌ Use exceptions for flow control
- ❌ Catch and swallow exceptions silently
- ❌ Return inconsistent error formats

---

## Related Standards

- [Clean Architecture](clean-architecture.md) - Layer organization
- [Domain Design](domain-design.md) - Domain exception patterns
- [Security](security.md) - Secure error handling
- [Logging (Serilog)](logging-serilog.md) - Error logging
