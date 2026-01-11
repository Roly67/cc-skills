# Security Standards

Security requirements and best practices for .NET projects.

## Core Principles

1. **Defense in Depth** — Multiple layers of security
2. **Least Privilege** — Minimal permissions required
3. **Fail Secure** — Errors should not expose data
4. **Zero Trust** — Verify everything, trust nothing

---

## Input Validation

### Validate All External Input

```csharp
// ✅ CORRECT - Validate and sanitize input
public async Task<IActionResult> CreateAsync(
    [FromBody] CreateOrderRequest request,
    CancellationToken cancellationToken)
{
    // Use FluentValidation or DataAnnotations
    if (!this.ModelState.IsValid)
    {
        return this.BadRequest(this.ModelState);
    }

    // Additional business validation
    if (request.Quantity > 1000)
    {
        return this.BadRequest("Quantity exceeds maximum allowed");
    }

    var result = await this.service.CreateAsync(request, cancellationToken);
    return this.Created($"/orders/{result.Id}", result);
}

// ❌ WRONG - No validation
public async Task<IActionResult> CreateAsync([FromBody] CreateOrderRequest request)
{
    var result = await this.service.CreateAsync(request);
    return this.Ok(result);
}
```

### Use Strong Typing

```csharp
// ✅ CORRECT - Strongly typed ID
public record OrderId(Guid Value)
{
    public static OrderId New() => new(Guid.NewGuid());
    public static OrderId Parse(string value) => new(Guid.Parse(value));
    public override string ToString() => this.Value.ToString();
}

public async Task<Order?> GetByIdAsync(OrderId id, CancellationToken cancellationToken)
{
    return await this.context.Orders
        .FirstOrDefaultAsync(o => o.Id == id.Value, cancellationToken)
        .ConfigureAwait(false);
}

// ❌ WRONG - String ID allows injection
public async Task<Order?> GetByIdAsync(string id, CancellationToken cancellationToken)
```

### Sanitize HTML Output

```csharp
// ✅ CORRECT - Encode output
public string GetSafeHtml(string userInput)
{
    return System.Web.HttpUtility.HtmlEncode(userInput);
}

// For Razor views, use @ which auto-encodes
<p>@Model.UserComment</p>

// Only use @Html.Raw() with sanitized content
<p>@Html.Raw(sanitizedHtml)</p>
```

---

## SQL Injection Prevention

### Always Use Parameterized Queries

```csharp
// ✅ CORRECT - Parameterized query with EF Core
var orders = await this.context.Orders
    .Where(o => o.CustomerId == customerId)
    .ToListAsync(cancellationToken)
    .ConfigureAwait(false);

// ✅ CORRECT - Parameterized raw SQL
var orders = await this.context.Orders
    .FromSqlInterpolated($"SELECT * FROM Orders WHERE CustomerId = {customerId}")
    .ToListAsync(cancellationToken)
    .ConfigureAwait(false);

// ❌ NEVER DO THIS - SQL injection vulnerability
var query = $"SELECT * FROM Orders WHERE CustomerId = '{customerId}'";
var orders = await this.context.Orders
    .FromSqlRaw(query)
    .ToListAsync(cancellationToken);
```

### Use Stored Procedures for Complex Queries

```csharp
// ✅ CORRECT - Stored procedure with parameters
var result = await this.context.Database
    .ExecuteSqlInterpolatedAsync(
        $"EXEC ProcessOrder @OrderId = {orderId}, @Status = {status}",
        cancellationToken)
    .ConfigureAwait(false);
```

---

## Authentication & Authorization

### JWT Token Validation

```csharp
// Program.cs
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = builder.Configuration["Auth:Authority"];
        options.Audience = builder.Configuration["Auth:Audience"];
        
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ClockSkew = TimeSpan.FromMinutes(5),
        };
    });
```

### API Key Authentication

```csharp
// <copyright file="ApiKeyAuthenticationHandler.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Api.Authentication;

/// <summary>
/// Handles API key authentication.
/// </summary>
public class ApiKeyAuthenticationHandler : AuthenticationHandler<ApiKeyAuthenticationOptions>
{
    private const string ApiKeyHeaderName = "X-API-Key";

    private readonly IApiKeyValidator validator;

    /// <inheritdoc/>
    protected override async Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        if (!this.Request.Headers.TryGetValue(ApiKeyHeaderName, out var apiKeyHeaderValues))
        {
            return AuthenticateResult.NoResult();
        }

        var apiKey = apiKeyHeaderValues.FirstOrDefault();

        if (string.IsNullOrWhiteSpace(apiKey))
        {
            return AuthenticateResult.NoResult();
        }

        // Validate API key (use constant-time comparison)
        var validationResult = await this.validator
            .ValidateAsync(apiKey, this.Context.RequestAborted)
            .ConfigureAwait(false);

        if (!validationResult.IsValid)
        {
            return AuthenticateResult.Fail("Invalid API key");
        }

        var claims = new[]
        {
            new Claim(ClaimTypes.NameIdentifier, validationResult.ClientId),
            new Claim(ClaimTypes.Name, validationResult.ClientName),
        };

        var identity = new ClaimsIdentity(claims, this.Scheme.Name);
        var principal = new ClaimsPrincipal(identity);
        var ticket = new AuthenticationTicket(principal, this.Scheme.Name);

        return AuthenticateResult.Success(ticket);
    }
}
```

### Use Constant-Time Comparison for Secrets

```csharp
// ✅ CORRECT - Prevents timing attacks
public bool ValidateApiKey(string providedKey, string storedKey)
{
    if (string.IsNullOrEmpty(providedKey) || string.IsNullOrEmpty(storedKey))
    {
        return false;
    }

    var providedBytes = Encoding.UTF8.GetBytes(providedKey);
    var storedBytes = Encoding.UTF8.GetBytes(storedKey);

    return CryptographicOperations.FixedTimeEquals(providedBytes, storedBytes);
}

// ❌ WRONG - Vulnerable to timing attacks
public bool ValidateApiKey(string providedKey, string storedKey)
{
    return providedKey == storedKey;
}
```

### Authorization Policies

```csharp
// Program.cs
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AdminOnly", policy =>
        policy.RequireRole("Admin"));

    options.AddPolicy("CanManageOrders", policy =>
        policy.RequireClaim("permission", "orders:manage"));

    options.AddPolicy("SameCustomer", policy =>
        policy.AddRequirements(new SameCustomerRequirement()));
});

// Controller
[Authorize(Policy = "CanManageOrders")]
[HttpPost]
public async Task<IActionResult> CreateOrderAsync(...)
```

---

## Secrets Management

### Never Commit Secrets

```csharp
// ❌ NEVER DO THIS
public class DatabaseSettings
{
    public string ConnectionString { get; } = "Server=prod;Password=SuperSecret123!";
}

// ❌ NEVER DO THIS
private const string ApiKey = "sk-1234567890abcdef";
```

### Use User Secrets in Development

```bash
# Initialize user secrets
dotnet user-secrets init

# Set secrets
dotnet user-secrets set "Database:Password" "DevPassword123"
dotnet user-secrets set "Sentry:Dsn" "https://key@sentry.io/123"
```

```csharp
// Program.cs - automatically loaded in Development
var builder = WebApplication.CreateBuilder(args);
// User secrets are loaded automatically when ASPNETCORE_ENVIRONMENT=Development
```

### Use Environment Variables or Key Vault in Production

```csharp
// Program.cs
builder.Configuration.AddEnvironmentVariables();

// Or Azure Key Vault
builder.Configuration.AddAzureKeyVault(
    new Uri($"https://{vaultName}.vault.azure.net/"),
    new DefaultAzureCredential());
```

### .gitignore Secrets

```gitignore
# Secrets
appsettings.*.json
!appsettings.json
!appsettings.Development.json.template
*.pfx
*.p12
secrets.json
.env
.env.*
```

---

## Logging Security

### Never Log Sensitive Data

```csharp
// ❌ NEVER LOG THESE
this.logger.LogInformation("User login: {Email} / {Password}", email, password);
this.logger.LogDebug("API Key: {ApiKey}", apiKey);
this.logger.LogInformation("Credit card: {CardNumber}", cardNumber);
this.logger.LogDebug("Token: {Token}", jwtToken);

// ✅ CORRECT - Log safely
this.logger.LogInformation("User login attempt for {Email}", email);
this.logger.LogDebug("API request authenticated for client {ClientId}", clientId);
this.logger.LogInformation("Payment processed for order {OrderId}", orderId);
```

### Mask Sensitive Data

```csharp
/// <summary>
/// Masks sensitive data for safe logging.
/// </summary>
public static class DataMasker
{
    /// <summary>
    /// Masks an email address.
    /// </summary>
    public static string MaskEmail(string email)
    {
        if (string.IsNullOrEmpty(email) || !email.Contains('@'))
        {
            return "***";
        }

        var parts = email.Split('@');
        var name = parts[0];
        var domain = parts[1];

        var maskedName = name.Length > 2
            ? $"{name[0]}***{name[^1]}"
            : "***";

        return $"{maskedName}@{domain}";
    }

    /// <summary>
    /// Masks a credit card number.
    /// </summary>
    public static string MaskCardNumber(string cardNumber)
    {
        if (string.IsNullOrEmpty(cardNumber) || cardNumber.Length < 4)
        {
            return "****";
        }

        return $"****-****-****-{cardNumber[^4..]}";
    }
}

// Usage
this.logger.LogInformation(
    "Payment for {MaskedEmail} with card {MaskedCard}",
    DataMasker.MaskEmail(email),
    DataMasker.MaskCardNumber(cardNumber));
```

### Audit Security Events

```csharp
// Log security-relevant events at Information or Warning level
this.logger.LogWarning(
    "Failed login attempt for {Email} from {IpAddress}",
    email,
    ipAddress);

this.logger.LogInformation(
    "User {UserId} granted {Permission} by {AdminId}",
    userId,
    permission,
    adminId);

this.logger.LogWarning(
    "API rate limit exceeded for client {ClientId}",
    clientId);
```

---

## HTTPS and Transport Security

### Enforce HTTPS

```csharp
// Program.cs
if (!app.Environment.IsDevelopment())
{
    app.UseHsts();
}

app.UseHttpsRedirection();
```

### Configure Security Headers

```csharp
// Program.cs
app.Use(async (context, next) =>
{
    context.Response.Headers.Append("X-Content-Type-Options", "nosniff");
    context.Response.Headers.Append("X-Frame-Options", "DENY");
    context.Response.Headers.Append("X-XSS-Protection", "1; mode=block");
    context.Response.Headers.Append("Referrer-Policy", "strict-origin-when-cross-origin");
    context.Response.Headers.Append(
        "Content-Security-Policy",
        "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'");

    await next();
});
```

### Configure CORS Properly

```csharp
// Program.cs
builder.Services.AddCors(options =>
{
    options.AddPolicy("Production", policy =>
    {
        policy
            .WithOrigins("https://app.{domain}")
            .WithMethods("GET", "POST", "PUT", "DELETE")
            .WithHeaders("Authorization", "Content-Type")
            .SetPreflightMaxAge(TimeSpan.FromMinutes(10));
    });
});

// ❌ NEVER in production
policy.AllowAnyOrigin().AllowAnyMethod().AllowAnyHeader();
```

---

## Rate Limiting

```csharp
// Program.cs (.NET 7+)
builder.Services.AddRateLimiter(options =>
{
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(context =>
        RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: context.User?.Identity?.Name ?? context.Request.Headers.Host.ToString(),
            factory: _ => new FixedWindowRateLimiterOptions
            {
                AutoReplenishment = true,
                PermitLimit = 100,
                Window = TimeSpan.FromMinutes(1),
            }));

    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
});

app.UseRateLimiter();
```

---

## Error Handling Security

### Don't Expose Internal Details

```csharp
// ✅ CORRECT - Generic error response
public class ErrorResponse
{
    public string Message { get; set; } = "An error occurred";
    public string? TraceId { get; set; }
}

// Global exception handler
app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        context.Response.StatusCode = 500;
        context.Response.ContentType = "application/json";

        var response = new ErrorResponse
        {
            Message = "An unexpected error occurred",
            TraceId = Activity.Current?.Id ?? context.TraceIdentifier,
        };

        // Log full details server-side
        var exception = context.Features.Get<IExceptionHandlerFeature>()?.Error;
        logger.LogError(exception, "Unhandled exception");

        // Return safe response to client
        await context.Response.WriteAsJsonAsync(response);
    });
});

// ❌ WRONG - Exposes internal details
catch (Exception ex)
{
    return StatusCode(500, new { error = ex.ToString(), stackTrace = ex.StackTrace });
}
```

---

## Dependency Security

### Keep Packages Updated

```bash
# Check for vulnerable packages
dotnet list package --vulnerable

# Check for outdated packages
dotnet list package --outdated
```

### Audit Dependencies

Add to CI/CD pipeline:

```yaml
- name: Security Audit
  run: |
    dotnet list package --vulnerable --include-transitive 2>&1 | tee audit.txt
    if grep -q "has the following vulnerable packages" audit.txt; then
      echo "❌ Vulnerable packages found"
      exit 1
    fi
```

---

## Security Checklist

### Authentication & Authorization
- [ ] JWT tokens validated (issuer, audience, signature, expiry)
- [ ] API keys use constant-time comparison
- [ ] Authorization policies enforced on all endpoints
- [ ] Failed auth attempts are logged

### Input Validation
- [ ] All user input validated
- [ ] Strong typing used for IDs
- [ ] File uploads validated (type, size)
- [ ] SQL injection prevented (parameterized queries)

### Data Protection
- [ ] Secrets not in source control
- [ ] Sensitive data encrypted at rest
- [ ] HTTPS enforced
- [ ] Security headers configured

### Logging & Monitoring
- [ ] No sensitive data in logs
- [ ] Security events audited
- [ ] Failed requests logged
- [ ] Rate limiting enabled

### Dependencies
- [ ] Packages regularly updated
- [ ] Vulnerability scanning in CI/CD
- [ ] Minimal dependency footprint
