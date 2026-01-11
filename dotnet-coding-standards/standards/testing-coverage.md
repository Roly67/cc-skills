# Testing & Code Coverage Standards

All projects require comprehensive unit testing with minimum 80% code coverage.

## Requirements

| Requirement | Value |
|-------------|-------|
| Minimum Coverage | 80% line coverage |
| Test Framework | xUnit |
| Mocking Framework | Moq |
| Assertion Library | FluentAssertions |
| Coverage Tool | Coverlet |

## What Must Be Tested

### Required Tests (80%+ Coverage)

- ✅ Services (business logic)
- ✅ Handlers (command/query handlers)
- ✅ Validators
- ✅ Mappers/Converters
- ✅ Extension methods with logic
- ✅ Domain entities with behavior
- ✅ Custom middleware
- ✅ Custom exceptions

### Optional Tests (Excluded from Coverage)

- DTOs/Models (data-only classes)
- Configuration classes
- Program.cs / Startup.cs
- Auto-generated code
- Integration-only code

## Project Structure

```
tests/
├── {CompanyName}.{ProjectName}.UnitTests/
│   ├── Services/
│   │   ├── OrderServiceTests.cs
│   │   └── PaymentServiceTests.cs
│   ├── Handlers/
│   │   └── CreateOrderHandlerTests.cs
│   ├── Validators/
│   │   └── OrderValidatorTests.cs
│   ├── Extensions/
│   │   └── StringExtensionsTests.cs
│   ├── Fixtures/
│   │   └── TestDataFixture.cs
│   └── {CompanyName}.{ProjectName}.UnitTests.csproj
└── {CompanyName}.{ProjectName}.IntegrationTests/
    ├── Api/
    │   └── OrdersControllerTests.cs
    └── {CompanyName}.{ProjectName}.IntegrationTests.csproj
```

## Test Project Configuration

### .csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <IsPackable>false</IsPackable>
    <IsTestProject>true</IsTestProject>
  </PropertyGroup>

  <ItemGroup>
    <Using Include="Xunit" />
    <Using Include="FluentAssertions" />
    <Using Include="Moq" />
  </ItemGroup>

  <ItemGroup>
    <PackageReference Include="coverlet.collector" Version="6.0.0">
      <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
      <PrivateAssets>all</PrivateAssets>
    </PackageReference>
    <PackageReference Include="FluentAssertions" Version="6.12.0" />
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.8.0" />
    <PackageReference Include="Moq" Version="4.20.70" />
    <PackageReference Include="xunit" Version="2.6.4" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.5.6">
      <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
      <PrivateAssets>all</PrivateAssets>
    </PackageReference>
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\..\src\{CompanyName}.{ProjectName}.Application\{CompanyName}.{ProjectName}.Application.csproj" />
  </ItemGroup>

</Project>
```

## Test Naming Convention

```
[MethodName]_[Scenario]_[ExpectedResult]
```

Examples:
- `ProcessAsync_WithValidInput_ReturnsSuccess`
- `ProcessAsync_WithNullInput_ThrowsArgumentNullException`
- `ProcessAsync_WhenRepositoryFails_PropagatesException`
- `Validate_WithEmptyEmail_ReturnsValidationError`

## Test Template

```csharp
// <copyright file="OrderServiceTests.cs" company="{CompanyName}">
// © {CompanyName}
// </copyright>

namespace {CompanyName}.{ProjectName}.UnitTests.Services;

using {CompanyName}.{ProjectName}.Application.Interfaces;
using {CompanyName}.{ProjectName}.Application.Services;
using {CompanyName}.{ProjectName}.Domain.Exceptions;

using FluentAssertions;

using Microsoft.Extensions.Logging;

using Moq;

using Xunit;

/// <summary>
/// Unit tests for <see cref="OrderService"/>.
/// </summary>
public class OrderServiceTests
{
    private readonly Mock<ILogger<OrderService>> loggerMock;
    private readonly Mock<IOrderRepository> repositoryMock;
    private readonly Mock<IPaymentService> paymentServiceMock;
    private readonly OrderService sut;

    /// <summary>
    /// Initializes a new instance of the <see cref="OrderServiceTests"/> class.
    /// </summary>
    public OrderServiceTests()
    {
        this.loggerMock = new Mock<ILogger<OrderService>>();
        this.repositoryMock = new Mock<IOrderRepository>();
        this.paymentServiceMock = new Mock<IPaymentService>();

        this.sut = new OrderService(
            this.loggerMock.Object,
            this.repositoryMock.Object,
            this.paymentServiceMock.Object);
    }

    /// <summary>
    /// Verifies constructor throws when logger is null.
    /// </summary>
    [Fact]
    public void Constructor_WhenLoggerIsNull_ThrowsArgumentNullException()
    {
        // Arrange & Act
        var act = () => new OrderService(
            null!,
            this.repositoryMock.Object,
            this.paymentServiceMock.Object);

        // Assert
        act.Should().Throw<ArgumentNullException>()
            .WithParameterName("logger");
    }

    /// <summary>
    /// Verifies CreateAsync returns created order when input is valid.
    /// </summary>
    [Fact]
    public async Task CreateAsync_WithValidInput_ReturnsCreatedOrder()
    {
        // Arrange
        var request = CreateValidRequest();
        var expectedOrder = new Order { Id = "order-123" };

        this.repositoryMock
            .Setup(r => r.CreateAsync(It.IsAny<Order>(), It.IsAny<CancellationToken>()))
            .ReturnsAsync(expectedOrder);

        // Act
        var result = await this.sut.CreateAsync(request, CancellationToken.None);

        // Assert
        result.Should().NotBeNull();
        result.Id.Should().Be("order-123");
        
        this.repositoryMock.Verify(
            r => r.CreateAsync(It.IsAny<Order>(), It.IsAny<CancellationToken>()),
            Times.Once);
    }

    /// <summary>
    /// Verifies CreateAsync throws when request is null.
    /// </summary>
    [Fact]
    public async Task CreateAsync_WithNullRequest_ThrowsArgumentNullException()
    {
        // Arrange
        CreateOrderRequest? request = null;

        // Act
        var act = () => this.sut.CreateAsync(request!, CancellationToken.None);

        // Assert
        await act.Should().ThrowAsync<ArgumentNullException>()
            .WithParameterName("request");
    }

    /// <summary>
    /// Verifies CreateAsync propagates repository exceptions.
    /// </summary>
    [Fact]
    public async Task CreateAsync_WhenRepositoryThrows_PropagatesException()
    {
        // Arrange
        var request = CreateValidRequest();
        var expectedException = new DatabaseException("Connection failed");

        this.repositoryMock
            .Setup(r => r.CreateAsync(It.IsAny<Order>(), It.IsAny<CancellationToken>()))
            .ThrowsAsync(expectedException);

        // Act
        var act = () => this.sut.CreateAsync(request, CancellationToken.None);

        // Assert
        await act.Should().ThrowAsync<DatabaseException>()
            .WithMessage("Connection failed");
    }

    /// <summary>
    /// Verifies CreateAsync respects cancellation.
    /// </summary>
    [Fact]
    public async Task CreateAsync_WhenCancelled_ThrowsOperationCanceledException()
    {
        // Arrange
        var request = CreateValidRequest();
        using var cts = new CancellationTokenSource();
        cts.Cancel();

        // Act
        var act = () => this.sut.CreateAsync(request, cts.Token);

        // Assert
        await act.Should().ThrowAsync<OperationCanceledException>();
    }

    private static CreateOrderRequest CreateValidRequest()
    {
        return new CreateOrderRequest
        {
            CustomerId = "customer-123",
            Items = new List<OrderItem>
            {
                new() { ProductId = "prod-1", Quantity = 2 },
            },
        };
    }
}
```

## Testing Patterns

### Arrange-Act-Assert (AAA)

```csharp
[Fact]
public async Task MethodName_Scenario_ExpectedResult()
{
    // Arrange - Set up test data and mocks
    var input = new Input { Value = "test" };
    this.mockService.Setup(s => s.GetAsync(It.IsAny<string>()))
        .ReturnsAsync(new Result());

    // Act - Execute the method under test
    var result = await this.sut.ProcessAsync(input);

    // Assert - Verify the outcome
    result.Should().NotBeNull();
    result.IsSuccess.Should().BeTrue();
}
```

### Testing Exceptions

```csharp
[Fact]
public async Task Method_WhenCondition_ThrowsSpecificException()
{
    // Arrange
    var invalidInput = new Input { Value = null };

    // Act
    var act = () => this.sut.ProcessAsync(invalidInput);

    // Assert
    await act.Should().ThrowAsync<ValidationException>()
        .WithMessage("*Value*required*");
}
```

### Testing Collections

```csharp
[Fact]
public async Task GetAll_ReturnsAllItems()
{
    // Arrange
    var expectedItems = new List<Item>
    {
        new() { Id = "1", Name = "Item 1" },
        new() { Id = "2", Name = "Item 2" },
    };

    this.repositoryMock.Setup(r => r.GetAllAsync(It.IsAny<CancellationToken>()))
        .ReturnsAsync(expectedItems);

    // Act
    var result = await this.sut.GetAllAsync(CancellationToken.None);

    // Assert
    result.Should().HaveCount(2);
    result.Should().ContainSingle(i => i.Id == "1");
    result.Should().BeInAscendingOrder(i => i.Name);
}
```

### Parameterized Tests

```csharp
[Theory]
[InlineData("", false)]
[InlineData(" ", false)]
[InlineData("valid@email.com", true)]
[InlineData("invalid-email", false)]
public void IsValidEmail_WithVariousInputs_ReturnsExpectedResult(
    string email,
    bool expectedResult)
{
    // Act
    var result = EmailValidator.IsValid(email);

    // Assert
    result.Should().Be(expectedResult);
}

[Theory]
[MemberData(nameof(GetTestCases))]
public async Task Process_WithTestCases_ReturnsExpectedResult(
    TestCase testCase)
{
    // Act
    var result = await this.sut.ProcessAsync(testCase.Input);

    // Assert
    result.Should().Be(testCase.ExpectedOutput);
}

public static IEnumerable<object[]> GetTestCases()
{
    yield return new object[] { new TestCase("input1", "output1") };
    yield return new object[] { new TestCase("input2", "output2") };
}
```

## Coverage Configuration

### coverlet.runsettings

Place at solution root:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<RunSettings>
  <DataCollectionRunSettings>
    <DataCollectors>
      <DataCollector friendlyName="XPlat Code Coverage">
        <Configuration>
          <Format>cobertura,opencover</Format>
          <Exclude>
            [*]*.Program,
            [*]*.Startup,
            [*]*Migrations*,
            [*]*.DTOs.*,
            [*]*.Models.*,
            [*]*.Configuration.*
          </Exclude>
          <ExcludeByAttribute>
            Obsolete,
            GeneratedCodeAttribute,
            CompilerGeneratedAttribute,
            ExcludeFromCodeCoverage
          </ExcludeByAttribute>
          <ExcludeByFile>**/Migrations/*.cs</ExcludeByFile>
          <IncludeTestAssembly>false</IncludeTestAssembly>
          <SkipAutoProps>true</SkipAutoProps>
        </Configuration>
      </DataCollector>
    </DataCollectors>
  </DataCollectionRunSettings>
</RunSettings>
```

## Running Tests

### Commands

```bash
# Run all tests
dotnet test

# Run with coverage
dotnet test --collect:"XPlat Code Coverage" --settings coverlet.runsettings

# Run specific test project
dotnet test tests/{CompanyName}.{ProjectName}.UnitTests

# Run tests matching filter
dotnet test --filter "FullyQualifiedName~OrderService"

# Generate HTML report
dotnet tool install -g dotnet-reportgenerator-globaltool
reportgenerator \
  -reports:./TestResults/**/coverage.cobertura.xml \
  -targetdir:./TestResults/CoverageReport \
  -reporttypes:Html
```

### CI/CD Coverage Check

```yaml
# GitHub Actions
- name: Test with Coverage
  run: dotnet test --collect:"XPlat Code Coverage" --settings coverlet.runsettings

- name: Check Coverage Threshold
  run: |
    COVERAGE=$(grep -oP 'line-rate="\K[^"]+' ./TestResults/**/coverage.cobertura.xml | head -1)
    COVERAGE_PCT=$(echo "$COVERAGE * 100" | bc)
    echo "Coverage: ${COVERAGE_PCT}%"
    if (( $(echo "$COVERAGE_PCT < 80" | bc -l) )); then
      echo "Coverage ${COVERAGE_PCT}% is below 80% threshold"
      exit 1
    fi
```

## Best Practices

### 1. Test One Thing Per Test

```csharp
// ✅ Good - Single assertion concept
[Fact]
public async Task CreateAsync_WithValidInput_ReturnsCreatedOrder()
{
    var result = await this.sut.CreateAsync(validInput);
    result.Should().NotBeNull();
    result.Id.Should().NotBeEmpty();
}

// ❌ Bad - Testing multiple behaviors
[Fact]
public async Task CreateAsync_Works()
{
    var result = await this.sut.CreateAsync(validInput);
    result.Should().NotBeNull();
    
    var retrieved = await this.sut.GetByIdAsync(result.Id);
    retrieved.Should().BeEquivalentTo(result);
    
    await this.sut.DeleteAsync(result.Id);
    var deleted = await this.sut.GetByIdAsync(result.Id);
    deleted.Should().BeNull();
}
```

### 2. Test Behavior, Not Implementation

```csharp
// ✅ Good - Tests behavior
[Fact]
public async Task ProcessAsync_WithValidOrder_SendsConfirmationEmail()
{
    await this.sut.ProcessAsync(order);
    
    this.emailServiceMock.Verify(
        e => e.SendAsync(It.Is<Email>(m => m.To == order.CustomerEmail)),
        Times.Once);
}

// ❌ Bad - Tests implementation details
[Fact]
public async Task ProcessAsync_CallsMethodsInCorrectOrder()
{
    var sequence = new MockSequence();
    this.repoMock.InSequence(sequence).Setup(...);
    this.emailMock.InSequence(sequence).Setup(...);
    // Too coupled to implementation
}
```

### 3. Use Descriptive Test Names

```csharp
// ✅ Good - Clear what's being tested
[Fact]
public async Task CreateOrder_WithInsufficientStock_ThrowsOutOfStockException()

// ❌ Bad - Unclear
[Fact]
public async Task Test1()

[Fact]
public async Task CreateOrderTest()
```

### 4. Don't Test Framework Code

```csharp
// ❌ Don't test that DI works
[Fact]
public void ServiceProvider_ResolvesOrderService()
{
    var service = serviceProvider.GetService<IOrderService>();
    service.Should().NotBeNull();
}

// ✅ Do test your code
[Fact]
public async Task OrderService_ProcessesOrderCorrectly()
{
    var result = await this.sut.ProcessAsync(order);
    result.IsSuccess.Should().BeTrue();
}
```
