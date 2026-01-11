# New Solution Setup Checklist

Use this checklist when creating a new .NET solution.

## Solution Root Files

- [ ] **README.md** — Comprehensive documentation using [template](../templates/README-template.md)
- [ ] **CHANGELOG.md** — Version history (start with `## [Unreleased]`)
- [ ] **LICENSE** — License file
- [ ] **.gitignore** — Git ignore rules (use `dotnet new gitignore`)
- [ ] **.editorconfig** — Editor configuration using [template](../templates/editorconfig.txt)
- [ ] **stylecop.json** — Centralized StyleCop config using [template](../templates/stylecop.json)
- [ ] **Directory.Build.props** — Shared MSBuild properties using [template](../templates/Directory.Build.props)
- [ ] **coverlet.runsettings** — Coverage configuration using [template](../templates/coverlet.runsettings)
- [ ] **{CompanyName}.[ProjectName].sln** — Solution file

## Project Structure

- [ ] **src/** folder for source projects
- [ ] **tests/** folder for test projects
- [ ] **docs/** folder for additional documentation (optional)

### For Web API Projects

- [ ] `{CompanyName}.[ProjectName].Domain` — Domain entities, exceptions
- [ ] `{CompanyName}.[ProjectName].Application` — Business logic, interfaces
- [ ] `{CompanyName}.[ProjectName].Infrastructure` — External services, persistence
- [ ] `{CompanyName}.[ProjectName].Api` — API host

### For Console Apps

- [ ] `{CompanyName}.[ProjectName].Console` — Console host
- [ ] `{CompanyName}.[ProjectName].Core` — Shared logic (optional)

### For Worker Services

- [ ] `{CompanyName}.[ProjectName].Worker` — Worker host
- [ ] `{CompanyName}.[ProjectName].Core` — Shared logic (optional)

### Test Projects

- [ ] `{CompanyName}.[ProjectName].UnitTests` — Unit tests
- [ ] `{CompanyName}.[ProjectName].IntegrationTests` — Integration tests (optional)

## Project Configuration

- [ ] All projects inherit from `Directory.Build.props` (automatic)
- [ ] No project has local StyleCop configuration
- [ ] No project has pragma suppressions
- [ ] `TreatWarningsAsErrors` is `true` (from Directory.Build.props)
- [ ] `GenerateDocumentationFile` is `true` (from Directory.Build.props)

## Host Project Configuration

- [ ] **Serilog** packages added and configured
- [ ] **Sentry** packages added and configured
- [ ] **appsettings.json** uses Serilog section (not Logging)
- [ ] **appsettings.Development.json** created
- [ ] **appsettings.Production.json** created
- [ ] **Program.cs** uses bootstrap logger pattern
- [ ] **Program.cs** configures Serilog to replace default logging
- [ ] **Program.cs** configures Sentry

## Class Library Configuration

- [ ] Only `Microsoft.Extensions.Logging.Abstractions` for logging
- [ ] No Serilog or Sentry references
- [ ] `ConfigureAwait(false)` used on all awaits

## Testing Setup

- [ ] Unit test project created
- [ ] Test project references source project
- [ ] xUnit, Moq, FluentAssertions packages added
- [ ] Coverlet.collector package added
- [ ] Global usings for Xunit, FluentAssertions, Moq

## CI/CD

- [ ] GitHub Actions workflow created using [template](../templates/github-actions-ci.yml)
- [ ] Coverage threshold set to 80%
- [ ] StyleCop violations fail build
- [ ] Pragma suppression check included

## Documentation

- [ ] README has all required sections
- [ ] Architecture documented
- [ ] Configuration documented
- [ ] API endpoints documented (for APIs)
- [ ] Logging section in README
- [ ] Error tracking section in README

## Verification Commands

Run these commands to verify setup:

```bash
# Build with zero warnings
dotnet build --warnaserrors

# Run tests
dotnet test

# Check for pragma suppressions (should return nothing)
grep -r "#pragma warning disable" --include="*.cs" src/

# Check for Microsoft logging config (should return nothing)
grep -r '"Logging"' --include="appsettings*.json" src/

# Verify Serilog is configured
grep -r "UseSerilog" --include="Program.cs" src/

# Verify Sentry is configured
grep -r "UseSentry\|AddSentry" --include="Program.cs" src/
```

## Final Checks

- [ ] Solution builds with zero warnings
- [ ] All tests pass
- [ ] README is complete and accurate
- [ ] All team members can clone, build, and run locally
