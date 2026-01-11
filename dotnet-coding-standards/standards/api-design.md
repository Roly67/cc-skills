# API Design Standards

RESTful API design standards for web services.

## URL Design

### Use Nouns, Not Verbs

```
✅ CORRECT
GET    /api/v1/orders
GET    /api/v1/orders/{id}
POST   /api/v1/orders
PUT    /api/v1/orders/{id}
DELETE /api/v1/orders/{id}

❌ WRONG
GET    /api/v1/getOrders
POST   /api/v1/createOrder
POST   /api/v1/deleteOrder/{id}
```

### Use Plural Nouns

```
✅ CORRECT
/api/v1/orders
/api/v1/customers
/api/v1/products

❌ WRONG
/api/v1/order
/api/v1/customer
/api/v1/product
```

### Use Kebab-Case for Multi-Word Resources

```
✅ CORRECT
/api/v1/order-items
/api/v1/shipping-addresses
/api/v1/payment-methods

❌ WRONG
/api/v1/orderItems
/api/v1/order_items
/api/v1/OrderItems
```

### Nest for Relationships

```
✅ CORRECT - Clear parent-child relationship
GET /api/v1/customers/{customerId}/orders
GET /api/v1/orders/{orderId}/items
GET /api/v1/orders/{orderId}/items/{itemId}

❌ WRONG - Flat structure loses context
GET /api/v1/customer-orders?customerId=123
```

### Limit Nesting Depth

```
✅ CORRECT - Max 2 levels deep
/api/v1/customers/{id}/orders
/api/v1/orders/{id}/items

❌ WRONG - Too deep
/api/v1/customers/{id}/orders/{orderId}/items/{itemId}/discounts
```

---

## HTTP Methods

| Method | Purpose | Request Body | Response Body | Idempotent | Safe |
|--------|---------|--------------|---------------|------------|------|
| GET | Retrieve resource(s) | No | Yes | Yes | Yes |
| POST | Create resource | Yes | Yes | No | No |
| PUT | Full update/replace | Yes | Yes | Yes | No |
| PATCH | Partial update | Yes | Yes | No | No |
| DELETE | Remove resource | No | Optional | Yes | No |

### Method Usage Examples

```csharp
/// <summary>
/// Gets all orders with optional filtering.
/// </summary>
[HttpGet]
public async Task<ActionResult<PagedResult<OrderResponse>>> GetAllAsync(
    [FromQuery] OrderFilterRequest filter,
    CancellationToken cancellationToken)

/// <summary>
/// Gets a specific order by ID.
/// </summary>
[HttpGet("{id}")]
public async Task<ActionResult<OrderResponse>> GetByIdAsync(
    string id,
    CancellationToken cancellationToken)

/// <summary>
/// Creates a new order.
/// </summary>
[HttpPost]
public async Task<ActionResult<OrderResponse>> CreateAsync(
    [FromBody] CreateOrderRequest request,
    CancellationToken cancellationToken)

/// <summary>
/// Fully updates an existing order.
/// </summary>
[HttpPut("{id}")]
public async Task<ActionResult<OrderResponse>> UpdateAsync(
    string id,
    [FromBody] UpdateOrderRequest request,
    CancellationToken cancellationToken)

/// <summary>
/// Partially updates an existing order.
/// </summary>
[HttpPatch("{id}")]
public async Task<ActionResult<OrderResponse>> PatchAsync(
    string id,
    [FromBody] JsonPatchDocument<OrderResponse> patchDoc,
    CancellationToken cancellationToken)

/// <summary>
/// Deletes an order.
/// </summary>
[HttpDelete("{id}")]
public async Task<IActionResult> DeleteAsync(
    string id,
    CancellationToken cancellationToken)
```

---

## HTTP Status Codes

### Success Codes (2xx)

| Code | Name | When to Use |
|------|------|-------------|
| 200 | OK | Successful GET, PUT, PATCH with response body |
| 201 | Created | Successful POST (include Location header) |
| 202 | Accepted | Request accepted for async processing |
| 204 | No Content | Successful DELETE, or PUT/PATCH with no response body |

### Client Error Codes (4xx)

| Code | Name | When to Use |
|------|------|-------------|
| 400 | Bad Request | Invalid request body, validation errors |
| 401 | Unauthorized | Missing or invalid authentication |
| 403 | Forbidden | Authenticated but not authorized |
| 404 | Not Found | Resource doesn't exist |
| 405 | Method Not Allowed | HTTP method not supported |
| 409 | Conflict | Resource conflict (e.g., duplicate) |
| 422 | Unprocessable Entity | Semantic validation failure |
| 429 | Too Many Requests | Rate limit exceeded |

### Server Error Codes (5xx)

| Code | Name | When to Use |
|------|------|-------------|
| 500 | Internal Server Error | Unexpected server error |
| 502 | Bad Gateway | Upstream service error |
| 503 | Service Unavailable | Temporary unavailability |
| 504 | Gateway Timeout | Upstream service timeout |

### Implementation

```csharp
[HttpGet("{id}")]
public async Task<IActionResult> GetByIdAsync(string id, CancellationToken cancellationToken)
{
    var order = await this.service.GetByIdAsync(id, cancellationToken);

    if (order is null)
    {
        return this.NotFound(new ErrorResponse
        {
            Code = "ORDER_NOT_FOUND",
            Message = $"Order with ID '{id}' was not found",
        });
    }

    return this.Ok(order);
}

[HttpPost]
public async Task<IActionResult> CreateAsync(
    [FromBody] CreateOrderRequest request,
    CancellationToken cancellationToken)
{
    var result = await this.service.CreateAsync(request, cancellationToken);

    // 201 Created with Location header
    return this.CreatedAtAction(
        nameof(this.GetByIdAsync),
        new { id = result.Id },
        result);
}

[HttpDelete("{id}")]
public async Task<IActionResult> DeleteAsync(string id, CancellationToken cancellationToken)
{
    var deleted = await this.service.DeleteAsync(id, cancellationToken);

    if (!deleted)
    {
        return this.NotFound();
    }

    // 204 No Content
    return this.NoContent();
}
```

---

## Request/Response Models

### Request Models

```csharp
// <copyright file="CreateOrderRequest.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.OrderApi.Api.DTOs.V1.Orders;

/// <summary>
/// Request to create a new order.
/// </summary>
public class CreateOrderRequest
{
    /// <summary>
    /// Gets or sets the customer ID.
    /// </summary>
    /// <example>cust-12345</example>
    [Required]
    [StringLength(50)]
    public string CustomerId { get; set; } = string.Empty;

    /// <summary>
    /// Gets or sets the order items.
    /// </summary>
    [Required]
    [MinLength(1, ErrorMessage = "At least one item is required")]
    public List<OrderItemRequest> Items { get; set; } = new();

    /// <summary>
    /// Gets or sets the shipping address.
    /// </summary>
    [Required]
    public AddressRequest ShippingAddress { get; set; } = new();

    /// <summary>
    /// Gets or sets optional notes.
    /// </summary>
    [StringLength(500)]
    public string? Notes { get; set; }
}
```

### Response Models

```csharp
// <copyright file="OrderResponse.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.OrderApi.Api.DTOs.V1.Orders;

/// <summary>
/// Response containing order details.
/// </summary>
public class OrderResponse
{
    /// <summary>
    /// Gets or sets the order ID.
    /// </summary>
    /// <example>ord-abc123</example>
    public string Id { get; set; } = string.Empty;

    /// <summary>
    /// Gets or sets the customer ID.
    /// </summary>
    public string CustomerId { get; set; } = string.Empty;

    /// <summary>
    /// Gets or sets the order status.
    /// </summary>
    /// <example>Pending</example>
    public string Status { get; set; } = string.Empty;

    /// <summary>
    /// Gets or sets the order items.
    /// </summary>
    public List<OrderItemResponse> Items { get; set; } = new();

    /// <summary>
    /// Gets or sets the total amount.
    /// </summary>
    /// <example>99.99</example>
    public decimal Total { get; set; }

    /// <summary>
    /// Gets or sets when the order was created.
    /// </summary>
    public DateTimeOffset CreatedAt { get; set; }

    /// <summary>
    /// Gets or sets when the order was last updated.
    /// </summary>
    public DateTimeOffset? UpdatedAt { get; set; }
}
```

### Error Response

```csharp
// <copyright file="ErrorResponse.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.OrderApi.Api.DTOs.Common;

/// <summary>
/// Standard error response.
/// </summary>
public class ErrorResponse
{
    /// <summary>
    /// Gets or sets the error code.
    /// </summary>
    /// <example>ORDER_NOT_FOUND</example>
    public string Code { get; set; } = string.Empty;

    /// <summary>
    /// Gets or sets the error message.
    /// </summary>
    /// <example>Order with ID 'xyz' was not found</example>
    public string Message { get; set; } = string.Empty;

    /// <summary>
    /// Gets or sets the trace ID for debugging.
    /// </summary>
    public string? TraceId { get; set; }

    /// <summary>
    /// Gets or sets validation errors (for 400 responses).
    /// </summary>
    public Dictionary<string, string[]>? Errors { get; set; }
}
```

---

## Pagination

### Cursor-Based (Recommended for Large Datasets)

```csharp
// Request
GET /api/v1/orders?limit=20&cursor=eyJpZCI6MTAwfQ

// Response
{
    "data": [...],
    "pagination": {
        "limit": 20,
        "hasMore": true,
        "nextCursor": "eyJpZCI6MTIwfQ",
        "prevCursor": "eyJpZCI6ODB9"
    }
}
```

```csharp
/// <summary>
/// Cursor-based pagination result.
/// </summary>
public class CursorPagedResult<T>
{
    /// <summary>
    /// Gets or sets the data items.
    /// </summary>
    public IReadOnlyList<T> Data { get; set; } = Array.Empty<T>();

    /// <summary>
    /// Gets or sets the pagination metadata.
    /// </summary>
    public CursorPagination Pagination { get; set; } = new();
}

/// <summary>
/// Cursor pagination metadata.
/// </summary>
public class CursorPagination
{
    /// <summary>
    /// Gets or sets the page size limit.
    /// </summary>
    public int Limit { get; set; }

    /// <summary>
    /// Gets or sets whether more results exist.
    /// </summary>
    public bool HasMore { get; set; }

    /// <summary>
    /// Gets or sets the next page cursor.
    /// </summary>
    public string? NextCursor { get; set; }

    /// <summary>
    /// Gets or sets the previous page cursor.
    /// </summary>
    public string? PrevCursor { get; set; }
}
```

### Offset-Based (Simple Use Cases)

```csharp
// Request
GET /api/v1/orders?page=2&pageSize=20

// Response
{
    "data": [...],
    "pagination": {
        "page": 2,
        "pageSize": 20,
        "totalPages": 10,
        "totalCount": 195
    }
}
```

```csharp
/// <summary>
/// Offset-based pagination result.
/// </summary>
public class PagedResult<T>
{
    /// <summary>
    /// Gets or sets the data items.
    /// </summary>
    public IReadOnlyList<T> Data { get; set; } = Array.Empty<T>();

    /// <summary>
    /// Gets or sets the pagination metadata.
    /// </summary>
    public OffsetPagination Pagination { get; set; } = new();
}

/// <summary>
/// Offset pagination metadata.
/// </summary>
public class OffsetPagination
{
    /// <summary>
    /// Gets or sets the current page number (1-based).
    /// </summary>
    public int Page { get; set; }

    /// <summary>
    /// Gets or sets the page size.
    /// </summary>
    public int PageSize { get; set; }

    /// <summary>
    /// Gets or sets the total number of pages.
    /// </summary>
    public int TotalPages { get; set; }

    /// <summary>
    /// Gets or sets the total count of items.
    /// </summary>
    public int TotalCount { get; set; }
}
```

---

## Filtering and Sorting

### Query Parameters

```csharp
// Request
GET /api/v1/orders?status=pending&minTotal=100&sort=-createdAt,total

// Filter request model
public class OrderFilterRequest
{
    /// <summary>
    /// Filter by status.
    /// </summary>
    public string? Status { get; set; }

    /// <summary>
    /// Filter by minimum total.
    /// </summary>
    public decimal? MinTotal { get; set; }

    /// <summary>
    /// Filter by maximum total.
    /// </summary>
    public decimal? MaxTotal { get; set; }

    /// <summary>
    /// Filter by customer ID.
    /// </summary>
    public string? CustomerId { get; set; }

    /// <summary>
    /// Filter by date range start.
    /// </summary>
    public DateTimeOffset? FromDate { get; set; }

    /// <summary>
    /// Filter by date range end.
    /// </summary>
    public DateTimeOffset? ToDate { get; set; }

    /// <summary>
    /// Sort fields (prefix with - for descending).
    /// </summary>
    /// <example>-createdAt,total</example>
    public string? Sort { get; set; }

    /// <summary>
    /// Page number (1-based).
    /// </summary>
    [Range(1, int.MaxValue)]
    public int Page { get; set; } = 1;

    /// <summary>
    /// Page size.
    /// </summary>
    [Range(1, 100)]
    public int PageSize { get; set; } = 20;
}
```

---

## Versioning

### URL Path Versioning (Recommended)

```
/api/v1/orders
/api/v2/orders
```

```csharp
// Controllers organized by version
Controllers/
├── V1/
│   └── OrdersController.cs
└── V2/
    └── OrdersController.cs

// V1 Controller
[ApiController]
[Route("api/v1/[controller]")]
public class OrdersController : ControllerBase

// V2 Controller
[ApiController]
[Route("api/v2/[controller]")]
public class OrdersController : ControllerBase
```

### When to Create a New Version

**DO create a new version for:**
- Removing fields from responses
- Renaming fields
- Changing field types
- Removing endpoints
- Changing authentication

**DON'T create a new version for:**
- Adding optional fields
- Adding new endpoints
- Bug fixes
- Performance improvements

---

## API Documentation

### Swagger/OpenAPI Attributes

```csharp
/// <summary>
/// Manages customer orders.
/// </summary>
[ApiController]
[Route("api/v1/[controller]")]
[Produces("application/json")]
[Tags("Orders")]
public class OrdersController : ControllerBase
{
    /// <summary>
    /// Gets an order by ID.
    /// </summary>
    /// <param name="id">The order ID.</param>
    /// <param name="cancellationToken">Cancellation token.</param>
    /// <returns>The order details.</returns>
    /// <response code="200">Returns the order.</response>
    /// <response code="404">Order not found.</response>
    /// <response code="401">Unauthorized.</response>
    [HttpGet("{id}")]
    [ProducesResponseType(typeof(OrderResponse), StatusCodes.Status200OK)]
    [ProducesResponseType(typeof(ErrorResponse), StatusCodes.Status404NotFound)]
    [ProducesResponseType(typeof(ErrorResponse), StatusCodes.Status401Unauthorized)]
    public async Task<IActionResult> GetByIdAsync(
        string id,
        CancellationToken cancellationToken)
}
```

### Swagger Configuration

```csharp
// Program.cs
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "{CompanyName} Order API",
        Version = "v1",
        Description = "API for managing customer orders",
        Contact = new OpenApiContact
        {
            Name = "{CompanyName} Support",
            Email = "{email}",
        },
    });

    // Include XML comments
    var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
    options.IncludeXmlComments(xmlPath);

    // Add security definition
    options.AddSecurityDefinition("ApiKey", new OpenApiSecurityScheme
    {
        Type = SecuritySchemeType.ApiKey,
        In = ParameterLocation.Header,
        Name = "X-API-Key",
        Description = "API key authentication",
    });
});
```

---

## Action-Based Endpoints

For operations that don't fit CRUD:

```csharp
// ✅ CORRECT - Sub-resource for actions
POST /api/v1/orders/{id}/cancel
POST /api/v1/orders/{id}/ship
POST /api/v1/orders/{id}/refund

// Implementation
[HttpPost("{id}/cancel")]
public async Task<IActionResult> CancelAsync(
    string id,
    [FromBody] CancelOrderRequest request,
    CancellationToken cancellationToken)
{
    var result = await this.service.CancelAsync(id, request.Reason, cancellationToken);
    return this.Ok(result);
}
```

---

## Best Practices Summary

| Practice | Example |
|----------|---------|
| Use nouns | `/orders` not `/getOrders` |
| Use plural | `/orders` not `/order` |
| Use kebab-case | `/order-items` not `/orderItems` |
| Version in URL | `/api/v1/orders` |
| Consistent responses | Always use same envelope structure |
| Include pagination metadata | `{ data: [], pagination: {} }` |
| Use appropriate status codes | 201 for create, 204 for delete |
| Document with OpenAPI | XML comments + Swagger |
| Support filtering | Query parameters |
| Support sorting | `?sort=-createdAt` |
