# IDE Snippets

Ready-to-use code snippets for CS .NET development.

## Available Snippets

### Core Snippets

| Prefix | Description |
|:-------|:------------|
| `csclass` | Class with copyright header and documentation |
| `csinterface` | Interface with copyright header |
| `csservice` | Service class with DI and logging |
| `cscontroller` | API Controller with DI |
| `csrepository` | Repository with EF Core |
| `csasync` | Async method with documentation |
| `csawait` | Await with ConfigureAwait(false) |
| `csctor` | Constructor with null check |
| `cstest` | Unit test class with setup |
| `csfact` | xUnit Fact test method |
| `cstheory` | xUnit Theory test method |
| `csdto` | DTO/Request class |
| `csexception` | Custom exception class |
| `csloginfo` | Serilog LogInformation |
| `cslogerror` | Serilog LogError with exception |
| `csworker` | BackgroundService worker |
| `cscopyright` | Copyright header |

### JetBrains.Annotations Snippets

| Prefix | Description |
|:-------|:------------|
| `csnotnull` | `[NotNull]` parameter annotation |
| `cscanbenull` | `[CanBeNull]` nullable parameter annotation |
| `csctorann` | Constructor with `[NotNull]` and guard clause |
| `cspure` | `[Pure]` method annotation |
| `csmustusereturn` | `[MustUseReturnValue]` method annotation |
| `cscontracthalt` | `[ContractAnnotation("null => halt")]` |
| `csitemnotnull` | `[ItemNotNull]` collection annotation |
| `csnonnegative` | `[NonNegativeValue]` annotation |
| `csvaluerange` | `[ValueRange(min, max)]` annotation |

## Installation

### VS Code

1. Open VS Code
2. Press `Ctrl+Shift+P` (or `Cmd+Shift+P` on Mac)
3. Type "Preferences: Configure User Snippets"
4. Select "csharp.json"
5. Copy contents of `vscode/csharp.json` into the file

Or copy to:
- **Windows**: `%APPDATA%\Code\User\snippets\csharp.json`
- **macOS**: `~/Library/Application Support/Code/User/snippets/csharp.json`
- **Linux**: `~/.config/Code/User/snippets/csharp.json`

### Visual Studio 2022

1. Open Visual Studio
2. Go to Tools → Code Snippets Manager (Ctrl+K, Ctrl+B)
3. Select Language: CSharp
4. Click "Import..."
5. Navigate to `visualstudio/cs-snippets.snippet`
6. Click "Finish"

Or copy to:
- **Windows**: `%USERPROFILE%\Documents\Visual Studio 2022\Code Snippets\Visual C#\My Code Snippets\`

### JetBrains Rider

**Option 1: Import via Settings**

1. Open Rider
2. Go to **File → Manage IDE Settings → Import Settings...**
3. Select `rider/CodingStandards.DotSettings`
4. Check "Live Templates" and click OK

**Option 2: Copy to Templates Folder**

Copy `rider/CodingStandards.DotSettings` to:
- **Windows**: `%APPDATA%\JetBrains\Rider<version>\templates\`
- **macOS**: `~/Library/Application Support/JetBrains/Rider<version>/templates/`
- **Linux**: `~/.config/JetBrains/Rider<version>/templates/`

Then restart Rider.

**Option 3: Manual Entry**

1. Open Rider
2. Go to **Settings → Editor → Live Templates**
3. Create a new template group called "CodingStandards"
4. Add templates manually using the snippets as reference

## Usage

1. Type the snippet prefix (e.g., `csservice`)
2. Press `Tab` to expand
3. Fill in the placeholders
4. Press `Tab` to move between placeholders
5. Press `Enter` to complete

## Examples

### Creating a Service

Type `csservice` + Tab:

```csharp
// <copyright file="OrderService.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace CS.ProjectName.Application.Services;

using Microsoft.Extensions.Logging;

/// <summary>
/// Description of the service.
/// </summary>
public class OrderService : IOrderService
{
    private readonly ILogger<OrderService> logger;

    /// <summary>
    /// Initializes a new instance of the <see cref="OrderService"/> class.
    /// </summary>
    /// <param name="logger">The logger instance.</param>
    public OrderService(ILogger<OrderService> logger)
    {
        this.logger = logger ?? throw new ArgumentNullException(nameof(logger));
    }

    // cursor here
}
```

### Creating a Constructor with Annotations

Type `csctorann` + Tab:

```csharp
/// <summary>
/// Initializes a new instance of the <see cref="OrderService"/> class.
/// </summary>
/// <param name="repository">The repository instance.</param>
public OrderService([NotNull] IOrderRepository repository)
{
    ArgumentNullException.ThrowIfNull(repository);
    this.repository = repository;
}
```

### Creating a Pure Method

Type `cspure` + Tab:

```csharp
/// <summary>
/// Calculates the total.
/// </summary>
[Pure]
public decimal Calculate(decimal price, int quantity)
{
    return price * quantity;
}
```

### Creating a Unit Test

Type `csfact` + Tab:

```csharp
/// <summary>
/// Test description.
/// </summary>
[Fact]
public async Task MethodName_Scenario_ExpectedResult()
{
    // Arrange
    // cursor here

    // Act
    var result = await this.sut.MethodNameAsync(CancellationToken.None);

    // Assert
    result.Should().NotBeNull();
}
```

### Await with ConfigureAwait

Type `csawait` + Tab:

```csharp
await this.repository.GetByIdAsync(id, cancellationToken)
    .ConfigureAwait(false);
```

### Value Range Parameter

Type `csvaluerange` + Tab:

```csharp
[ValueRange(0, 100)] int percentage
```

## Customization

Feel free to customize these snippets for your team:

1. Copy the relevant file to your local snippets folder
2. Modify prefixes, placeholders, or template content
3. Add new snippets as needed

## Snippet Naming Convention

All CS snippets use the `cs` prefix to avoid conflicts with built-in snippets.

## Related Standards

- [Nullability & Annotations](../standards/nullability-annotations.md) - Full guide to JetBrains.Annotations
