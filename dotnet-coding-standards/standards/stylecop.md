# StyleCop & Code Style Standards

Centralized StyleCop configuration and code style rules for .NET projects.

## Core Principle

> **StyleCop warnings must be RESOLVED, never suppressed.**

```csharp
// ❌ FORBIDDEN - Never do this
#pragma warning disable SA1600
public class MyClass { }
#pragma warning restore SA1600

// ❌ FORBIDDEN - Never do this
[SuppressMessage("StyleCop.CSharp.DocumentationRules", "SA1600:ElementsMustBeDocumented")]
public class MyClass { }

// ✅ CORRECT - Fix the actual issue
/// <summary>
/// Provides functionality for processing data.
/// </summary>
public class MyClass { }
```

## Centralized Configuration

StyleCop is configured at the **solution root** and shared across all projects.

### Solution Structure

```
{CompanyName}.{ProjectName}/
├── stylecop.json              # Centralized StyleCop config
├── Directory.Build.props      # Shared build properties
├── .editorconfig              # Editor configuration
└── src/
    ├── Project1/              # Inherits stylecop.json
    └── Project2/              # Inherits stylecop.json
```

### stylecop.json

Place at solution root:

```json
{
  "$schema": "https://raw.githubusercontent.com/DotNetAnalyzers/StyleCopAnalyzers/master/StyleCop.Analyzers/StyleCop.Analyzers/Settings/stylecop.schema.json",
  "settings": {
    "documentationRules": {
      "companyName": "{CompanyName}",
      "copyrightText": "© {CompanyName}",
      "fileNamingConvention": "stylecop",
      "documentExposedElements": true,
      "documentInternalElements": true,
      "documentPrivateElements": false,
      "documentInterfaces": true,
      "documentPrivateFields": false
    },
    "orderingRules": {
      "usingDirectivesPlacement": "insideNamespace",
      "systemUsingDirectivesFirst": true,
      "blankLinesBetweenUsingGroups": "require"
    },
    "namingRules": {
      "allowCommonHungarianPrefixes": false,
      "allowedHungarianPrefixes": []
    },
    "layoutRules": {
      "newlineAtEndOfFile": "require"
    },
    "readabilityRules": {
      "allowBuiltInTypeAliases": false
    },
    "maintainabilityRules": {
      "topLevelTypes": ["class", "interface", "struct", "enum", "delegate"]
    }
  }
}
```

### Directory.Build.props

Place at solution root to apply StyleCop to all projects:

```xml
<Project>
  
  <!-- Shared properties for ALL projects -->
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <WarningLevel>5</WarningLevel>
    <GenerateDocumentationFile>true</GenerateDocumentationFile>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
    <AnalysisLevel>latest-recommended</AnalysisLevel>
  </PropertyGroup>

  <!-- StyleCop Analyzers - applies to all projects -->
  <ItemGroup>
    <PackageReference Include="StyleCop.Analyzers" Version="1.2.0-beta.556">
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
      <PrivateAssets>all</PrivateAssets>
    </PackageReference>
  </ItemGroup>

  <!-- Link to centralized stylecop.json -->
  <ItemGroup>
    <AdditionalFiles Include="$(MSBuildThisFileDirectory)stylecop.json" Link="stylecop.json" />
  </ItemGroup>

</Project>
```

## Key StyleCop Rules

### Documentation Rules (SA16xx)

| Rule | Description | Fix |
|------|-------------|-----|
| SA1600 | Elements must be documented | Add `<summary>` XML doc |
| SA1601 | Partial elements must be documented | Document in one location |
| SA1602 | Enumeration items must be documented | Add `<summary>` to enum values |
| SA1633 | File must have header | Add copyright header |
| SA1634 | Header must show copyright | Use correct copyright text |

### Ordering Rules (SA12xx)

| Rule | Description | Fix |
|------|-------------|-----|
| SA1200 | Using directives inside namespace | Move `using` inside namespace |
| SA1201 | Elements in correct order | Order: fields, constructors, properties, methods |
| SA1202 | Elements ordered by access | Public before private |
| SA1204 | Static before instance | Static members first |

### Naming Rules (SA13xx)

| Rule | Description | Fix |
|------|-------------|-----|
| SA1300 | Element must begin with upper case | Use `PascalCase` |
| SA1302 | Interface names begin with I | `IMyInterface` |
| SA1303 | Const field names begin with upper case | `const int MaxValue` |
| SA1309 | Field names must not begin with underscore | Use `this.fieldName` |
| SA1310 | Field names must not contain underscore | Use `camelCase` |

### Readability Rules (SA11xx)

| Rule | Description | Fix |
|------|-------------|-----|
| SA1101 | Prefix local calls with this | Use `this.field`, `this.Method()` |
| SA1116 | Split parameters on own line | Each param on new line |
| SA1117 | Parameters on same line or each on own | Be consistent |

### Layout Rules (SA15xx)

| Rule | Description | Fix |
|------|-------------|-----|
| SA1500 | Braces must not be omitted | Always use `{ }` |
| SA1502 | Element must not be on single line | Expand to multiple lines |
| SA1513 | Closing brace followed by blank line | Add blank line after `}` |
| SA1516 | Elements separated by blank line | Add blank line between members |

## Copyright Header

Every `.cs` file must start with:

```csharp
// <copyright file="ClassName.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>
```

**Important**: The filename in the header must match the actual filename.

## File Structure Template

```csharp
// <copyright file="DataProcessor.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.Services;

using System.Text.Json;

using Microsoft.Extensions.Logging;

using {CompanyName}.{ProjectName}.Models;

/// <summary>
/// Processes data from various sources.
/// </summary>
public class DataProcessor : IDataProcessor
{
    private readonly ILogger<DataProcessor> logger;
    private readonly IRepository repository;

    /// <summary>
    /// Initializes a new instance of the <see cref="DataProcessor"/> class.
    /// </summary>
    /// <param name="logger">The logger instance.</param>
    /// <param name="repository">The data repository.</param>
    public DataProcessor(ILogger<DataProcessor> logger, IRepository repository)
    {
        this.logger = logger ?? throw new ArgumentNullException(nameof(logger));
        this.repository = repository ?? throw new ArgumentNullException(nameof(repository));
    }

    /// <summary>
    /// Gets the processor name.
    /// </summary>
    public string Name => "DataProcessor";

    /// <inheritdoc/>
    public async Task<ProcessResult> ProcessAsync(
        ProcessInput input,
        CancellationToken cancellationToken)
    {
        this.logger.LogInformation("Processing {InputId}", input.Id);

        var data = await this.repository
            .GetAsync(input.Id, cancellationToken)
            .ConfigureAwait(false);

        return new ProcessResult(data);
    }
}
```

## Member Ordering

Within a class, members should be ordered:

1. **Constants**
2. **Static readonly fields**
3. **Static fields**
4. **Instance readonly fields**
5. **Instance fields**
6. **Constructors**
7. **Finalizers**
8. **Delegates**
9. **Events**
10. **Properties**
11. **Indexers**
12. **Methods**
13. **Nested types**

Within each group, order by access modifier:
1. `public`
2. `internal`
3. `protected internal`
4. `protected`
5. `private protected`
6. `private`

## Using Directive Ordering

```csharp
namespace {CompanyName}.{ProjectName}.Services;

// 1. System namespaces (alphabetical)
using System;
using System.Collections.Generic;
using System.Threading.Tasks;

// 2. Microsoft namespaces (alphabetical)
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;

// 3. Third-party namespaces (alphabetical)
using Newtonsoft.Json;

// 4. Project namespaces (alphabetical)
using {CompanyName}.{ProjectName}.Models;
using {CompanyName}.{ProjectName}.Utilities;
```

## Common Violations and Fixes

### SA1101: Prefix local calls with this

```csharp
// ❌ Wrong
public void Process()
{
    logger.LogInformation("Processing");
    DoWork();
}

// ✅ Correct
public void Process()
{
    this.logger.LogInformation("Processing");
    this.DoWork();
}
```

### SA1309: Field names must not begin with underscore

```csharp
// ❌ Wrong
private readonly ILogger _logger;

// ✅ Correct
private readonly ILogger logger;
// Access with: this.logger
```

### SA1200: Using directives inside namespace

```csharp
// ❌ Wrong
using System;

namespace {CompanyName}.{ProjectName};

public class MyClass { }

// ✅ Correct
namespace {CompanyName}.{ProjectName};

using System;

public class MyClass { }
```

### SA1633: File must have header

```csharp
// ❌ Wrong - No header
namespace {CompanyName}.{ProjectName};

public class MyClass { }

// ✅ Correct
// <copyright file="MyClass.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName};

public class MyClass { }
```

## Verification Commands

```bash
# Build and see all StyleCop warnings
dotnet build 2>&1 | grep -E "SA[0-9]{4}"

# Find pragma suppressions (should return nothing)
grep -r "#pragma warning disable" --include="*.cs" src/

# Find SuppressMessage attributes (should return nothing)
grep -r "SuppressMessage" --include="*.cs" src/

# Count violations by rule
dotnet build 2>&1 | grep -oE "SA[0-9]{4}" | sort | uniq -c | sort -rn
```

## IDE Integration

### Visual Studio

StyleCop violations appear as warnings/errors in the Error List. With `TreatWarningsAsErrors=true`, they prevent build.

### JetBrains Rider

Enable StyleCop inspections in Settings → Editor → Inspection Settings.

### VS Code

Use the C# extension with OmniSharp. Violations appear in the Problems panel.
