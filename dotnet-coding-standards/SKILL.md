---
name: dotnet-coding-standards
version: 2.3.0
last-updated: 2026-01-11
description: >
  .NET coding standards and patterns. Use when: creating new .NET solutions,
  writing C# classes/services/controllers, setting up Serilog logging, configuring
  Sentry error tracking, reviewing pull requests, checking code coverage, fixing
  StyleCop warnings, adding unit tests, or scaffolding Web API/Console/Worker/Blazor
  projects. Covers Clean Architecture, async/await patterns, ConfigureAwait usage,
  and CI/CD pipeline configuration. Apply to all .cs and .csproj files.
---

# .NET Coding Standards

Comprehensive coding standards for .NET projects following Clean Architecture principles.

## Customization

This skill uses placeholders for company-specific values. Replace these throughout:

| Placeholder | Description | Example |
|-------------|-------------|---------|
| `{CompanyName}` | Your company/organization name | `Acme`, `Contoso` |
| `{ProjectName}` | Your project name | `OrderApi`, `PaymentService` |
| `{email}` | Support/contact email | `support@example.com` |
| `{domain}` | Your company domain | `example.com` |

---

## 🚀 New Developer Quick Start

1. **Clone** the repo and run `dotnet restore`
2. **Read** the project's README.md (every project has one)
3. **Check** you have .NET 10 SDK installed
4. **Run** `dotnet build` — must complete with **zero warnings**
5. **Run** `dotnet test` — must have **80%+ coverage**
6. **Before committing**: Run the [PR Review Checklist](checklists/pr-review-checklist.md)

First task? Start with the [New Solution Checklist](checklists/new-solution-checklist.md).

---

## Quick Reference

| Requirement | Rule |
|-------------|------|
| **Copyright** | Every `.cs` file starts with the copyright header |
| **Namespaces** | File-scoped: `{CompanyName}.[ProjectName].[Layer].[Feature]` |
| **Documentation** | All public members require XML docs |
| **Null safety** | `?? throw new ArgumentNullException(nameof(param))` in constructors |
| **Field access** | Always use `this.` prefix for instance members |
| **Build rule** | Zero warnings allowed (`TreatWarningsAsErrors=true`) |
| **Framework** | .NET 10, nullable reference types enabled |
| **StyleCop** | Centralized config, NO pragma suppressions allowed |
| **Testing** | All functional classes must have unit tests |
| **Coverage** | Minimum 80% code coverage required |
| **README** | Every solution must have comprehensive README.md |
| **ConfigureAwait** | Use `ConfigureAwait(false)` in library/infrastructure code |
| **Logging** | Serilog in all host processes (overrides Microsoft logging) |
| **Exceptions** | Sentry for exception tracking in all host processes |

---

## ❌ Critical Rules (Non-Negotiable)

### Never Use Pragma Suppressions

```csharp
// ❌ FORBIDDEN
#pragma warning disable SA1600
public class MyClass { }
#pragma warning restore SA1600

// ✅ CORRECT - Fix the actual issue
/// <summary>
/// Provides functionality for processing data.
/// </summary>
public class MyClass { }
```

### Never Skip Unit Tests for Functional Classes

Every class with business logic MUST have corresponding unit tests with 80%+ coverage.

### Never Use Microsoft.Extensions.Logging Config in Hosts

Serilog configuration takes precedence. See [Serilog Standards](standards/logging-serilog.md).

---

## 📁 Detailed Documentation

### By Project Type

| Project Type | Guide |
|--------------|-------|
| Web API | [project-types/web-api.md](project-types/web-api.md) |
| Console Application | [project-types/console-app.md](project-types/console-app.md) |
| Class Library | [project-types/class-library.md](project-types/class-library.md) |
| Worker Service | [project-types/worker-service.md](project-types/worker-service.md) |
| Blazor | [project-types/blazor.md](project-types/blazor.md) |

### Standards & Patterns

| Topic | Guide |
|-------|-------|
| Clean Architecture | [standards/clean-architecture.md](standards/clean-architecture.md) |
| CQRS & MediatR | [standards/cqrs-mediatr.md](standards/cqrs-mediatr.md) |
| Domain Design | [standards/domain-design.md](standards/domain-design.md) |
| Error Handling | [standards/error-handling.md](standards/error-handling.md) |
| StyleCop & Code Style | [standards/stylecop.md](standards/stylecop.md) |
| Nullability & Annotations | [standards/nullability-annotations.md](standards/nullability-annotations.md) |
| Serilog Logging | [standards/logging-serilog.md](standards/logging-serilog.md) |
| Sentry Error Tracking | [standards/error-tracking-sentry.md](standards/error-tracking-sentry.md) |
| Testing & Coverage | [standards/testing-coverage.md](standards/testing-coverage.md) |
| Async & ConfigureAwait | [standards/async-configureawait.md](standards/async-configureawait.md) |
| Documentation & README | [standards/documentation.md](standards/documentation.md) |
| Security | [standards/security.md](standards/security.md) |
| API Design | [standards/api-design.md](standards/api-design.md) |
| Performance | [standards/performance.md](standards/performance.md) |
| Dependency Injection | [standards/dependency-injection.md](standards/dependency-injection.md) |
| Git Workflow | [standards/git-workflow.md](standards/git-workflow.md) |

### Templates (Copy-Paste Ready)

| Template | File |
|----------|------|
| README.md | [templates/README-template.md](templates/README-template.md) |
| appsettings.json | [templates/appsettings-template.json](templates/appsettings-template.json) |
| Directory.Build.props | [templates/Directory.Build.props](templates/Directory.Build.props) |
| stylecop.json | [templates/stylecop.json](templates/stylecop.json) |
| .editorconfig | [templates/editorconfig.txt](templates/editorconfig.txt) |
| coverlet.runsettings | [templates/coverlet.runsettings](templates/coverlet.runsettings) |
| GitHub Actions CI | [templates/github-actions-ci.yml](templates/github-actions-ci.yml) |

### Checklists

| Checklist | File |
|-----------|------|
| New Solution Setup | [checklists/new-solution-checklist.md](checklists/new-solution-checklist.md) |
| PR Review | [checklists/pr-review-checklist.md](checklists/pr-review-checklist.md) |

### IDE Snippets

| IDE | Location |
|-----|----------|
| VS Code | [snippets/vscode/csharp.json](snippets/vscode/csharp.json) |
| Visual Studio | [snippets/visualstudio/cs-snippets.snippet](snippets/visualstudio/cs-snippets.snippet) |
| Installation Guide | [snippets/README.md](snippets/README.md) |

---

## Common Mistakes to Avoid

### Logging

| Mistake | Why It's Wrong | Correct Approach |
|---------|----------------|------------------|
| `logger.LogInformation($"User {userId}")` | Loses structure, poor performance | `logger.LogInformation("User {UserId}", userId)` |
| Using `ILogger` directly in class library | Couples to implementation | Use `ILogger<T>` from abstractions |
| `catch (Exception) { }` | Swallows errors silently | Log and rethrow, or handle specifically |
| Microsoft logging config in appsettings | Ignored when Serilog is used | Use `Serilog` section only |

### Async/Await

| Mistake | Why It's Wrong | Correct Approach |
|---------|----------------|------------------|
| `Task.Run(() => AsyncMethod())` | Wastes thread pool thread | `await AsyncMethod()` |
| `.Result` or `.Wait()` | Deadlock risk | Always `await` |
| Missing `ConfigureAwait(false)` in library | Captures unnecessary sync context | Add `ConfigureAwait(false)` |
| `ConfigureAwait(false)` in controller | Loses HttpContext | Don't use in controllers |
| Missing `Async` suffix | Naming inconsistency | All async methods end with `Async` |

### Testing

| Mistake | Why It's Wrong | Correct Approach |
|---------|----------------|------------------|
| No tests for service class | Untested business logic | Every functional class needs tests |
| Testing implementation details | Brittle tests | Test behavior, not internals |
| `[Fact] public void Test1()` | Unclear purpose | `[Fact] public void MethodName_Scenario_ExpectedResult()` |
| Not testing exception cases | Incomplete coverage | Test happy path AND error paths |

### StyleCop

| Mistake | Why It's Wrong | Correct Approach |
|---------|----------------|------------------|
| `#pragma warning disable` | Hides technical debt | Fix the actual issue |
| `_fieldName` | Non-standard for this codebase | `this.fieldName` |
| Missing XML docs | SA1600 violation | Document all public members |
| Block-scoped namespace | SA1513 violation | Use file-scoped `namespace X;` |

---

## Decision Flowcharts

### Should I Use ConfigureAwait(false)?

```
Is this code in a Controller, Razor Page, or Blazor component?
├── YES → Do NOT use ConfigureAwait(false)
└── NO → Does this code access HttpContext, User, or UI elements?
    ├── YES → Do NOT use ConfigureAwait(false)
    └── NO → USE ConfigureAwait(false) ✓
```

### What Log Level Should I Use?

```
Is this an unhandled exception or app crash?
├── YES → LogCritical (Fatal)
└── NO → Did an operation fail?
    ├── YES → LogError
    └── NO → Is something unexpected but handled?
        ├── YES → LogWarning
        └── NO → Is this a significant business event?
            ├── YES → LogInformation
            └── NO → Is this useful for debugging?
                ├── YES → LogDebug
                └── NO → LogTrace (or don't log)
```

### Do I Need a Unit Test for This Class?

```
Does this class contain business logic or behavior?
├── YES → Unit test REQUIRED ✓
└── NO → Is this a DTO, model, or configuration class?
    ├── YES → Unit test optional (data-only)
    └── NO → Is this auto-generated code?
        ├── YES → Unit test not needed
        └── NO → Is this startup/Program.cs?
            ├── YES → Integration test instead
            └── NO → Probably needs a unit test ✓
```

---

## Package Versions

| Package | Minimum | Recommended | Notes |
|---------|---------|-------------|-------|
| .NET SDK | 10.0 | 10.0 | Required |
| Serilog | 4.0.0 | 4.0.0 | |
| Serilog.AspNetCore | 8.0.0 | 8.0.0 | For Web API |
| Sentry | 4.0.0 | 4.3.0 | |
| Sentry.Serilog | 4.0.0 | 4.3.0 | |
| StyleCop.Analyzers | 1.2.0-beta.556 | 1.2.0-beta.556 | Beta has C#12 support |
| FluentAssertions | 6.0.0 | 6.12.0 | |
| Moq | 4.18.0 | 4.20.70 | |
| xUnit | 2.6.0 | 2.6.4 | |
| Coverlet | 6.0.0 | 6.0.0 | |

---

## Quick Commands

```bash
# Build with zero warnings
dotnet build --warnaserrors

# Run all tests with coverage
dotnet test --collect:"XPlat Code Coverage" --settings coverlet.runsettings

# Generate HTML coverage report
reportgenerator -reports:./TestResults/**/coverage.cobertura.xml -targetdir:./coverage -reporttypes:Html

# Find StyleCop violations
dotnet build 2>&1 | grep -E "SA[0-9]{4}"

# Find pragma suppressions (should return nothing)
grep -r "#pragma warning disable" --include="*.cs" src/

# Find missing ConfigureAwait in Infrastructure
grep -r "await.*;" --include="*.cs" src/Infrastructure/ | grep -v "ConfigureAwait(false)"

# Find Microsoft logging config (should not exist in hosts)
grep -r '"Logging"' --include="appsettings*.json" src/
```

---

## Exception Process

If you genuinely cannot follow a standard (e.g., third-party library conflict):

1. **Document** the exception in code: `// STANDARDS-EXCEPTION: [reason]`
2. **Create** a GitHub issue linking to the code
3. **Get** tech lead approval
4. **Add** to project's `docs/EXCEPTIONS.md` file

**Never** use pragma suppressions as an escape hatch.

---

## IDE Setup

### Visual Studio 2022
1. Tools → Options → Text Editor → C# → Code Style
2. Import `.editorconfig` from solution root
3. Enable "Run code analysis on build"

### JetBrains Rider
1. Settings → Editor → Code Style → C#
2. Import from `.editorconfig`
3. Enable StyleCop inspections in Inspections settings

### VS Code
1. Install C# Dev Kit extension
2. Install "EditorConfig for VS Code" extension
3. Add to settings.json: `"omnisharp.enableEditorConfigSupport": true`

---

## Glossary

| Term | Definition |
|------|------------|
| **Aggregate** | A cluster of domain objects treated as a single unit with a root entity |
| **Breadcrumb** | Sentry term for contextual events leading up to an error |
| **CanBeNull** | JetBrains annotation indicating a parameter or return value may be null |
| **Clean Architecture** | Layered architecture with Domain at center, dependencies pointing inward |
| **ConfigureAwait(false)** | Tells async/await not to capture the synchronization context |
| **ContractAnnotation** | JetBrains annotation for specifying method behavior based on input/output nullability |
| **CQRS** | Command Query Responsibility Segregation — separating read and write operations |
| **Domain Event** | A record of something significant that happened in the domain |
| **Host Process** | Executable project (API, Console, Worker) that runs the application |
| **MediatR** | In-process messaging library for implementing CQRS pattern |
| **NotNull** | JetBrains annotation indicating a parameter or return value cannot be null |
| **NRT** | Nullable Reference Types — C# compiler feature for null safety |
| **Pure** | JetBrains annotation marking methods with no side effects |
| **Result Pattern** | Alternative to exceptions for expected failures, returning success/failure with value |
| **Structured Logging** | Logging with named placeholders `{PropertyName}` that preserve data types |
| **SUT** | System Under Test — the class being tested in a unit test |
| **Value Object** | Immutable object defined by its attributes rather than identity |

---

## Changelog

- **2.3.0** (2026-01-11): Added Nullability & Annotations standards (JetBrains.Annotations)
- **2.2.0** (2026-01-11): Added Clean Architecture, CQRS/MediatR, Domain Design, Error Handling standards
- **2.1.0** (2026-01-10): Added Serilog, Sentry, ConfigureAwait standards; multi-file structure
- **2.0.0** (2025-12-01): Extended to multiple project types (Console, Worker, Blazor)
- **1.0.0** (2025-10-15): Initial Web API standards
