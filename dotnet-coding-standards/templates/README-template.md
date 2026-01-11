# {CompanyName}.[ProjectName]

[One-line description of what the project does]

## Overview

[Detailed description - 2-3 paragraphs explaining:]
- What problem this solves
- Key features and capabilities
- Who uses it

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Getting Started](#getting-started)
- [Configuration](#configuration)
- [Architecture](#architecture)
- [Logging](#logging)
- [Error Tracking](#error-tracking)
- [API Reference](#api-reference)
- [Testing](#testing)
- [Deployment](#deployment)
- [Contributing](#contributing)
- [Troubleshooting](#troubleshooting)
- [License](#license)

## Prerequisites

- [.NET 10 SDK](https://dotnet.microsoft.com/download)
- [List other prerequisites - databases, services, tools]

## Getting Started

### Installation

```bash
# Clone the repository
git clone https://github.com/{organization}/{CompanyName}.[ProjectName].git
cd {CompanyName}.[ProjectName]

# Restore dependencies
dotnet restore

# Build the solution
dotnet build

# Run tests
dotnet test

# Run the application
dotnet run --project src/{CompanyName}.[ProjectName].[Host]
```

### Quick Start

[Minimal steps to get running for first-time users]

```bash
# Example quick start commands
dotnet run
```

## Configuration

### Environment Variables

| Variable | Description | Required | Default |
|----------|-------------|----------|---------|
| `SENTRY_DSN` | Sentry Data Source Name | Yes | - |
| `DATABASE_URL` | Database connection string | Yes | - |
| `SEQ_URL` | Seq server URL | No | - |

### Application Settings

**appsettings.json**:

```json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning"
      }
    }
  },
  "Sentry": {
    "Dsn": "",
    "Environment": "Development"
  }
}
```

## Architecture

### Solution Structure

```
{CompanyName}.[ProjectName]/
├── src/
│   ├── {CompanyName}.[ProjectName].Domain/        # Domain entities, exceptions
│   ├── {CompanyName}.[ProjectName].Application/   # Business logic, interfaces
│   ├── {CompanyName}.[ProjectName].Infrastructure/# External services, persistence
│   └── {CompanyName}.[ProjectName].[Host]/        # API/Console/Worker host
└── tests/
    ├── {CompanyName}.[ProjectName].UnitTests/
    └── {CompanyName}.[ProjectName].IntegrationTests/
```

### Key Components

[Describe major components and their responsibilities]

### Design Decisions

[Document important architectural choices and their rationale]

## Logging

This project uses **Serilog** for structured logging.

### Log Levels

| Level | Usage |
|-------|-------|
| Debug | Development diagnostics |
| Information | Normal application flow |
| Warning | Unexpected but handled situations |
| Error | Operation failures |
| Fatal | Application crashes |

### Viewing Logs

- **Console**: Logs output to console in development
- **Files**: `logs/` directory
- **Seq**: [URL if applicable]

## Error Tracking

This project uses **Sentry** for exception tracking.

- **Dashboard**: [Link to Sentry project]
- **Alerts**: Configured for Error and Fatal levels

## API Reference

[For APIs: Document endpoints or link to Swagger]

### Authentication

[How to authenticate]

### Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/resource` | Get all resources |
| GET | `/api/v1/resource/{id}` | Get resource by ID |
| POST | `/api/v1/resource` | Create resource |

## Testing

### Running Tests

```bash
# Run all tests
dotnet test

# Run with coverage
dotnet test --collect:"XPlat Code Coverage" --settings coverlet.runsettings

# Generate coverage report
reportgenerator -reports:./TestResults/**/coverage.cobertura.xml -targetdir:./coverage -reporttypes:Html
```

### Coverage Requirements

- **Minimum**: 80% line coverage
- All functional classes must have unit tests
- Integration tests for external dependencies

## Deployment

### Production Build

```bash
dotnet publish -c Release -o ./publish
```

### Docker

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS base
WORKDIR /app
EXPOSE 80

FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "{CompanyName}.[ProjectName].[Host].dll"]
```

```bash
docker build -t {image-name} .
docker run -p 8080:80 {image-name}
```

### Environment Configuration

| Environment | Configuration |
|-------------|---------------|
| Development | appsettings.Development.json |
| Staging | appsettings.Staging.json |
| Production | appsettings.Production.json |

## Contributing

### Code Standards

All code must follow [.NET Coding Standards](./docs/coding-standards.md):

- ✅ All StyleCop warnings resolved (no pragma suppressions)
- ✅ Minimum 80% test coverage
- ✅ All public members documented with XML
- ✅ Serilog for logging (not Microsoft.Extensions.Logging config)
- ✅ ConfigureAwait(false) in library/infrastructure code

### Pull Request Process

1. Create feature branch from `main`
2. Make changes following coding standards
3. Ensure `dotnet build` passes with zero warnings
4. Ensure `dotnet test` passes with 80%+ coverage
5. Create PR and request review
6. Squash and merge after approval

### Commit Messages

Follow conventional commits:

```
feat: add order processing endpoint
fix: resolve null reference in validator
docs: update API documentation
test: add unit tests for OrderService
refactor: extract payment logic to service
```

## Troubleshooting

### Common Issues

#### Build Fails with StyleCop Warnings

Ensure all public members have XML documentation:

```csharp
/// <summary>
/// Description of the class.
/// </summary>
public class MyClass { }
```

#### Tests Fail with Coverage Below 80%

Add unit tests for untested classes. Check coverage report for gaps.

#### Sentry Not Receiving Events

1. Verify `Sentry:Dsn` is configured
2. Check `MinimumEventLevel` isn't too high
3. Ensure exceptions aren't being filtered

### Getting Help

- **Issues**: [GitHub Issues](https://github.com/{organization}/{CompanyName}.[ProjectName]/issues)
- **Team Channel**: #{image-name} on Slack

## License

Copyright © {CompanyName}. All rights reserved.

---

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for version history.
