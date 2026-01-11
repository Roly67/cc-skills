<p align="center">
  <img src="https://img.shields.io/badge/.NET-10-512bd4?style=for-the-badge&logo=dotnet&logoColor=white" alt=".NET 10" />
  <img src="https://img.shields.io/badge/C%23-12-239120?style=for-the-badge&logo=csharp&logoColor=white" alt="C# 12" />
  <img src="https://img.shields.io/badge/Version-2.3.0-7c3aed?style=for-the-badge" alt="Version 2.3.0" />
  <img src="https://img.shields.io/badge/License-MIT-22c55e?style=for-the-badge" alt="MIT License" />
</p>

<h1 align="center">
  .NET Coding Standards
</h1>

<p align="center">
  <strong>Enterprise-grade development standards for Claude Code</strong>
</p>

<p align="center">
  <sub>16 Standards • 5 Project Types • 28 IDE Snippets • Production-Ready Templates</sub>
</p>

---

## Overview

This skill equips Claude Code with comprehensive .NET development knowledge—from architectural patterns to code style enforcement. When activated, Claude applies these standards consistently across your codebase.

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│   "Write a service class"                                        │
│                                                                  │
│   → Clean Architecture placement                                 │
│   → Serilog structured logging                                   │
│   → Proper DI registration                                       │
│   → XML documentation                                            │
│   → StyleCop compliance                                          │
│   → Unit test scaffolding                                        │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

<br>

## Installation

### Quick Install

```bash
claude /install-plugin github:roly67/cc-skills
```

### When to Use

This skill activates automatically when you're:

- Creating new .NET solutions or projects
- Writing C# classes, services, or controllers
- Setting up Serilog logging or Sentry error tracking
- Reviewing pull requests for .NET code
- Fixing StyleCop warnings or adding unit tests
- Working with any `.cs` or `.csproj` files

<br>

## What's Included

<table>
<tr>
<td align="center" width="150">
<br>
<strong>📐</strong>
<br><br>
<strong>Standards</strong>
<br>
<sub>16 documents</sub>
<br><br>
</td>
<td>

Architecture, logging, testing, error handling, security, performance, API design, and more

</td>
</tr>
<tr>
<td align="center">
<br>
<strong>🏗️</strong>
<br><br>
<strong>Project Types</strong>
<br>
<sub>5 guides</sub>
<br><br>
</td>
<td>

Web API • Console App • Worker Service • Class Library • Blazor

</td>
</tr>
<tr>
<td align="center">
<br>
<strong>✅</strong>
<br><br>
<strong>Checklists</strong>
<br>
<sub>2 guides</sub>
<br><br>
</td>
<td>

New solution setup • PR review criteria

</td>
</tr>
<tr>
<td align="center">
<br>
<strong>📄</strong>
<br><br>
<strong>Templates</strong>
<br>
<sub>7 files</sub>
<br><br>
</td>
<td>

Directory.Build.props • StyleCop • EditorConfig • GitHub Actions CI • Coverage settings

</td>
</tr>
<tr>
<td align="center">
<br>
<strong>⚡</strong>
<br><br>
<strong>Snippets</strong>
<br>
<sub>28 total</sub>
<br><br>
</td>
<td>

VS Code • Visual Studio 2022 • JetBrains Rider

</td>
</tr>
</table>

<br>

## Technical Standards

| Category | Standard | Key Topics |
|:---------|:---------|:-----------|
| **Architecture** | [Clean Architecture](./standards/clean-architecture.md) | Layers, dependency rules, boundaries |
| | [CQRS & MediatR](./standards/cqrs-mediatr.md) | Commands, queries, pipelines |
| | [Domain Design](./standards/domain-design.md) | Entities, value objects, aggregates |
| **Quality** | [Testing & Coverage](./standards/testing-coverage.md) | xUnit, 80% coverage, AAA pattern |
| | [StyleCop](./standards/stylecop.md) | Zero warnings, no suppressions |
| | [Documentation](./standards/documentation.md) | XML docs, README structure |
| **Operations** | [Logging (Serilog)](./standards/logging-serilog.md) | Structured logging, sinks |
| | [Error Tracking (Sentry)](./standards/error-tracking-sentry.md) | Exception capture, breadcrumbs |
| | [Error Handling](./standards/error-handling.md) | Exception hierarchy, Result pattern |
| **Code** | [Async & ConfigureAwait](./standards/async-configureawait.md) | Context flow, library patterns |
| | [Nullability](./standards/nullability-annotations.md) | JetBrains.Annotations, guards |
| | [Dependency Injection](./standards/dependency-injection.md) | Registration patterns, lifetimes |
| **API** | [API Design](./standards/api-design.md) | REST, versioning, DTOs |
| | [Security](./standards/security.md) | Auth, validation, secrets |
| | [Performance](./standards/performance.md) | Caching, async, memory |
| **Process** | [Git Workflow](./standards/git-workflow.md) | Branching, commits, PRs |

<br>

## Core Principles

<table>
<tr>
<td width="40" align="center">🚫</td>
<td><strong>Zero pragma suppressions</strong> — Fix issues at the source</td>
</tr>
<tr>
<td align="center">📊</td>
<td><strong>80%+ code coverage</strong> — Every functional class needs tests</td>
</tr>
<tr>
<td align="center">⚠️</td>
<td><strong>Warnings as errors</strong> — <code>TreatWarningsAsErrors=true</code> everywhere</td>
</tr>
<tr>
<td align="center">📝</td>
<td><strong>Structured logging only</strong> — No string interpolation in log calls</td>
</tr>
<tr>
<td align="center">⚡</td>
<td><strong>ConfigureAwait(false)</strong> — Required in library/infrastructure code</td>
</tr>
</table>

<br>

## Package Recommendations

| Package | Version | Purpose |
|:--------|:--------|:--------|
| `Serilog` | 4.0.0 | Structured logging |
| `Serilog.AspNetCore` | 8.0.0 | ASP.NET Core integration |
| `Sentry` | 4.0.0+ | Error tracking |
| `StyleCop.Analyzers` | 1.2.0-beta.556 | Code style (C#12 support) |
| `xUnit` | 2.6.x | Testing framework |
| `FluentAssertions` | 6.x | Test assertions |
| `Moq` | 4.20.x | Mocking |
| `Coverlet` | 6.0.0 | Code coverage |

<br>

## Quick Reference

```bash
# Build with zero warnings
dotnet build --warnaserrors

# Run tests with coverage
dotnet test --collect:"XPlat Code Coverage"

# Find StyleCop violations
dotnet build 2>&1 | grep -E "SA[0-9]{4}"

# Verify no pragma suppressions
grep -r "#pragma warning disable" --include="*.cs" src/
```

<br>

## File Structure

```
dotnet-coding-standards/
│
├── SKILL.md                 # Entry point & quick reference
│
├── standards/               # Technical standards
│   ├── clean-architecture.md
│   ├── logging-serilog.md
│   ├── testing-coverage.md
│   └── ... (16 total)
│
├── project-types/           # Project scaffolding guides
│   ├── web-api.md
│   ├── console-app.md
│   ├── worker-service.md
│   ├── class-library.md
│   └── blazor.md
│
├── checklists/              # Step-by-step guides
│   ├── new-solution-checklist.md
│   └── pr-review-checklist.md
│
├── templates/               # Copy-paste configs
│   ├── Directory.Build.props
│   ├── stylecop.json
│   ├── editorconfig.txt
│   └── ... (7 total)
│
└── snippets/                # IDE code snippets
    ├── vscode/
    ├── visualstudio/
    └── rider/
```

<br>

## Changelog

| Version | Date | Changes |
|:--------|:-----|:--------|
| `2.3.0` | 2026-01-11 | Added Nullability & JetBrains.Annotations |
| `2.2.0` | 2026-01-11 | Added Clean Architecture, CQRS, Domain Design, Error Handling |
| `2.1.0` | 2026-01-10 | Added Serilog, Sentry, ConfigureAwait standards |
| `2.0.0` | 2025-12-01 | Multi-project type support |
| `1.0.0` | 2025-10-15 | Initial release |

<br>

## License

This skill is licensed under the [MIT License](../LICENSE).

---

<p align="center">
  <sub>Part of <a href="../">cc-skills</a> • Built for <a href="https://claude.ai/code">Claude Code</a></sub>
</p>
