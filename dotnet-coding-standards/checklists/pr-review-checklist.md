# Pull Request Review Checklist

Use this checklist when reviewing PRs or before submitting your own PR.

## Before Submitting PR

### Build & Tests

- [ ] `dotnet build` passes with **zero warnings**
- [ ] `dotnet test` passes with **100% pass rate**
- [ ] Code coverage is **≥80%** for new/modified code
- [ ] No pragma suppressions added

### Code Style

- [ ] All new files have copyright header
- [ ] All new public members have XML documentation
- [ ] File-scoped namespaces used
- [ ] Correct namespace format: `{CompanyName}.[ProjectName].[Layer].[Feature]`
- [ ] `this.` prefix used for all instance members
- [ ] Constructor parameters validated with `?? throw new ArgumentNullException()`

### Async/Await

- [ ] Async methods have `Async` suffix
- [ ] `CancellationToken` accepted and propagated
- [ ] `ConfigureAwait(false)` used in:
  - [ ] Domain layer
  - [ ] Application layer
  - [ ] Infrastructure layer
- [ ] `ConfigureAwait(false)` NOT used in:
  - [ ] Controllers
  - [ ] Blazor components
  - [ ] Worker ExecuteAsync main loop

### Logging

- [ ] Structured logging used (no string interpolation)
- [ ] Appropriate log levels used
- [ ] No sensitive data logged
- [ ] Serilog configuration used (not Microsoft.Extensions.Logging)

### Testing

- [ ] All new functional classes have unit tests
- [ ] Test names follow `[Method]_[Scenario]_[Expected]` pattern
- [ ] Tests use Arrange-Act-Assert pattern
- [ ] Edge cases and error paths tested
- [ ] Mocks verify expected interactions

---

## PR Review Checklist

### General

- [ ] PR description clearly explains the change
- [ ] PR is appropriately sized (not too large)
- [ ] Related issues are linked
- [ ] No unrelated changes included

### Code Quality

- [ ] Code is readable and maintainable
- [ ] No code duplication
- [ ] Single Responsibility Principle followed
- [ ] No magic numbers/strings (use constants)
- [ ] Error handling is appropriate

### Security

- [ ] No secrets or credentials committed
- [ ] No SQL injection vulnerabilities
- [ ] Input validation present where needed
- [ ] Sensitive data not logged

### Performance

- [ ] No obvious performance issues
- [ ] No N+1 query problems
- [ ] Async used appropriately (not blocking)
- [ ] Dispose pattern followed where needed

### Breaking Changes

- [ ] API changes are backward compatible (or documented)
- [ ] Database migrations are safe
- [ ] Configuration changes are documented

---

## Quick Verification Commands

```bash
# Verify build passes
dotnet build --warnaserrors

# Verify tests pass
dotnet test

# Check coverage
dotnet test --collect:"XPlat Code Coverage"

# Find pragma suppressions (should be empty)
grep -r "#pragma warning disable" --include="*.cs" src/

# Find string interpolation in logs (should be empty)
grep -rE 'Log(Debug|Information|Warning|Error|Critical)\(\$"' --include="*.cs" src/

# Find missing ConfigureAwait in Infrastructure
grep -r "await.*;" --include="*.cs" src/Infrastructure/ | grep -v "ConfigureAwait(false)"

# Find missing Async suffix
grep -rE "public async Task.*\(" --include="*.cs" src/ | grep -v "Async("
```

---

## Common Issues to Watch For

### ❌ StyleCop Violations

```csharp
// Missing documentation
public class MyClass { }  // SA1600

// Wrong field prefix
private readonly ILogger _logger;  // SA1309

// Missing this. prefix
logger.LogInformation("...");  // SA1101
```

### ❌ Async Issues

```csharp
// Missing ConfigureAwait in library
await service.GetAsync();  // Should be .ConfigureAwait(false)

// Missing Async suffix
public async Task<Data> GetData();  // Should be GetDataAsync

// Missing CancellationToken
public async Task ProcessAsync();  // Should accept CancellationToken
```

### ❌ Logging Issues

```csharp
// String interpolation
logger.LogInformation($"Processing {id}");  // Use structured logging

// Wrong level
logger.LogInformation(ex, "Error");  // Should be LogError
```

### ❌ Testing Issues

```csharp
// Poor test name
[Fact]
public void Test1() { }  // Use descriptive name

// No assertion
[Fact]
public async Task ProcessAsync_Works()
{
    await sut.ProcessAsync();
    // Missing assertion!
}
```

---

## Approval Criteria

A PR should be approved when:

1. ✅ All checklist items are satisfied
2. ✅ Build passes with zero warnings
3. ✅ All tests pass
4. ✅ Coverage threshold is met
5. ✅ Code review feedback is addressed
6. ✅ At least one approving review received
