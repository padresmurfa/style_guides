# C# Unit Testing Style Guide

This document defines **best practices** for writing **clear, structured, and maintainable** unit tests in C# using xUnit, NUnit, or MSTest.

---

## **Target Audience**

The primary target audience for this document is developers and Large Language Models (LLMs) acting as coding assistants. By following these guidelines, tests will be **structured, readable, and maintainable**.

These practices may be too detailed for human-written tests but serve as an excellent guide for generating automated tests with LLM assistance. **Review all test functions carefully**, as human oversight is crucial to ensure test quality.

---

## **1. Test Namespace Organization**
Consistent namespaces make it immediately obvious where a test lives relative to the production code, so reviewers can locate the subject under test without opening multiple files. Misaligned namespaces cause confusion when navigating between source and tests, especially in IDEs that rely on namespace matching for discovery.
- Every test file must declare a namespace that mirrors the production namespace and appends a `.Tests` suffix (e.g. `MyCompany.Products.Services` → `MyCompany.Products.Services.Tests`).
- When the production code uses nested namespaces, mirror that structure exactly in the test namespace before adding `.Tests`.
- Keep a **one-to-one mapping between folders and namespaces** so moving a file preserves the namespace.
- Avoid combining test classes for different production namespaces under a single namespace—create separate namespaces instead.

✅ **Good Example:**
```csharp
namespace MyCompany.Products.Services.Tests;
```
🚫 **Bad Example:**
```csharp
namespace Tests; // ❌ Does not mirror the production namespace
```

---

## **2. Test Class Naming and Files**
Consistent test class names make it immediately obvious which production code a suite exercises, so reviewers can trace failures quickly and spot coverage gaps. Without a naming convention teams waste time hunting for relevant tests, duplicate effort, and risk overlooking scenarios because responsibilities are spread across ambiguously labeled files.
- Each test file must contain a single public test class whose name **ends with** `Tests`.
- If a test class exercises a whole class and all of its methods, it must be named `<ClassName>Tests`.
- If a test class exercises a single region within a class, it must be named `<ClassName><RegionName>Tests`.
- If a test class exercises a single method within a class, it must be named `<ClassName><MethodName>Tests` or `<ClassName><RegionName><MethodName>Tests`.
- Region-focused and method-focused test classes are **optional** patterns. Prefer them when they make intent clearer, but feel free to keep everything in a single `<ClassName>Tests` class when the class-under-test is small enough that splitting would add overhead without clarity.
- Test class names should not contain underscores.
- **Each test class should cover a single class, region, or method under test.**
- **A test class' file name must be of the form `<ClassName>[<RegionName>][<MethodName>]Tests.cs`**, and every component of the file name must appear in the class name exactly once and in the same order.
- Organize test files into folders that mirror the namespace to keep discovery tools and human navigation aligned.

✅ **Good Examples:**
```csharp
public class FileHandlerTests()
public class FileHandlerHashingTests()
public class FileHandlerOpenTests()
```
🚫 **Bad Example:**
```csharp
public class FileHandler() // ❌ Missing Tests suffix
public class FileHandlerWorksAsIntendedTests() // ❌ WorksAsIntended is not a plausible #region name
public class FileHandlerInitializationAndHashingTests() // ❌ Combines multiple #regions into one class
```

---

## **3. Region-Focused Test Classes**
Regions describe cohesive areas of responsibility inside a production class. Mirroring those regions in the test suite keeps the feedback loop tight by ensuring every region has an intentionally scoped test class. Treat these region-focused classes as a tool rather than a mandate—when a production class only has a handful of regions or methods, duplicating files for each can be overkill.
- Only introduce a `<RegionName>` segment when the production code exposes a `#region` with the same PascalCase name.
- When a class contains multiple regions, create a separate test class per region, following the naming rules above and keeping each class in its own file.
- Do not reuse a `<RegionName>` segment across unrelated production regions; each region must map to a single test class.
- If a region contains helper methods that are not directly invoked by the public API, test the public surface that exercises them rather than testing the region helpers directly.

✅ **Good Example:**
```
Services/
│── FileHandler.cs          // Contains regions Hashing and Open
Tests/
│── FileHandlerHashingTests.cs
│── FileHandlerOpenTests.cs
```
🚫 **Bad Example:**
```
Tests/
│── FileHandlerHashingAndOpenTests.cs // ❌ Merges multiple regions into a single test class
```

---

## **4. Test Method Naming**
Clear method names document the behavior under test and the expected outcome, which helps maintainers understand intent without rereading the implementation. Vague or inconsistent names hide gaps in coverage, encourage multi-purpose tests, and make regression triage harder when a failing test name does not reveal what broke.
- Each test method **must start with** `Test_`.
- Use underscores to separate major phrases in the test method name.
- Although test classes stay in PascalCase with no underscores, **test methods are encouraged to use underscores** so long names remain readable.
- Test method names should **not contain consecutive underscores**.
- If multiple production methods are being tested within the same test class, the test method **must start with** `Test_<MethodName>_`.
- If a single production method is being tested in the file, the production method name **must be omitted** from the test method.
- The method name should be **descriptive and readable**, reflecting the behavior under test.
- Favor verbosity over brevity—short names rarely communicate enough context to make the test self-documenting.
- Test method names must describe the **specific branch of execution** they cover (e.g. `Test_Foo_ReturnsBar_WhenSuccessful`), not merely state that something "works".
- **Separate tests** into distinct methods rather than combining multiple test cases.
- **Avoid parameterized tests** unless they significantly improve clarity.
- Any component found in the test class name **must NOT be duplicated in the test method name**.

✅ **Good Example:**
```csharp
[Test]
public void Test_ProcessData_ReturnsCorrectValue()
```
🚫 **Bad Example:**
```csharp
[Test]
public void CheckProcessData() // ❌ Missing Test_ prefix
[Test]
public void Test_Process() // ❌ Too vague
[Test]
public void Test_MultipleCases() // ❌ Tests multiple things at once
```

---

## **5. Test Method Sectioning**
Standardized sections carve complex tests into digestible steps, making it easier to see how inputs flow through mocks and the system under test. Without this structure tests devolve into monolithic blocks where intent, setup, and verification intermingle, obscuring bugs and encouraging brittle copy-paste patterns. The ordering rules below are intentionally strong to build reliable habits—refer to [**Section 17. Breaking the Rules**](#-17-breaking-the-rules) for guidance on the rare cases where deviating is justified.

The high-level **Arrange–Act–Assert (AAA)** pattern is the conceptual root of this style guide: *Arrange* prepares the world, *Act* exercises the behaviour, and *Assert* verifies the outcome. AAA is intentionally coarse-grained—perfect for short, simple unit tests—but it starts to creak once a scenario grows beyond a handful of lines. That is where the richer sectioning described below shines. You can adopt only the headline sections for straightforward tests, add descriptive comment headers as complexity grows, and even split an individual section into multiple focused subsections (e.g., `// MOCKING: the database contains the given record` followed by `// MOCKING: the accounting API rejects the record`) when the test demands it.

Each test method follows a structured format, **separated by clear comments**:
- Follow the Arrange → Act → Assert order unless a section is unnecessary; omit empty sections entirely.
- Section comment headers should be full descriptive sentences (e.g. `// GIVEN: A valid context and wash code details`), not terse labels.
- When a single section becomes long, divide it with additional human-readable sub-headers so readers can scan for intent quickly.

### **5.1 Arrange**
The Arrange stage gathers inputs, configures doubles, and exposes collaborators to the system under test. Keeping all preparation here prevents leakage of setup noise into later sections.

#### **5.1.1 GIVEN**
- Set up initial conditions and inputs.
- Assign variables with the prefix `given`.
- When test fixtures are used, allocate them in this section.
- **Test Fixture classes** should be named `Fixture*` and assigned to `given*` variables prior to use.
- The GIVEN section should always be the first section in a test, unless there is nothing to define in GIVEN, in which case it should be omitted.
- **The GIVEN section must not be merged with any other section.**

#### **5.1.2 CAPTURE**
- Declare variables that will hold values captured from mocks, spies, or test fakes.
- Prefix these placeholders with `capture` (e.g. `capturePublishedMessage`).
- Keep this section focused on declarations; configure the mechanics that populate them in MOCKING or SETUP.
- Position CAPTURE immediately after GIVEN and before MOCKING whenever it is present.
- Promote capture placeholders into `actual*` variables during WHEN so assertions always reference concrete results.
- Include CAPTURE only when collaborators expose information that must be asserted later.

#### **5.1.3 MOCKING**
- Define and configure **mock objects**.
- Mock variables must be named `mock*`.
- **Use constructor injection for dependencies**, or use mocking frameworks like Moq.
- The MOCKING section should occur before the SETUP section, or be omitted if no mocking is required. Under special circumstances it may be placed after SETUP.
- **The MOCKING section must not be merged with any other section.**
- When mocking logic has multiple responsibilities (e.g. creating fakes and configuring behaviours), split it into sub-headers such as `// MOCKING: Create mock objects` and `// MOCKING: Configure mock responses`.

##### **Mocking Best Practices**
Disciplined mocking keeps tests readable and trustworthy by ensuring only true collaborators are simulated and their expectations are explicit. Unrestrained mocks create brittle, implementation-driven tests that break on harmless refactors, mask missing coverage of dependencies, and clutter SUT construction with framework boilerplate.
- **Never mock what you don't have to**—prefer fixtures or real instances where practical.
- **Only mock things that the system-under-test uses directly** so the test exercises the right collaborators.
- Dependencies that your SUT consumes should have their own unit tests rather than being re-verified indirectly.
- **Use Moq or built-in mocking frameworks** for dependency injection.
- **Assertions on mock interactions belong in the BEHAVIOR section, not in THEN.**
- **Mocks must never be passed directly to the system under test.** Assign mocks to interface-conforming `env*` variables in SETUP and pass those to the SUT.
- This pattern keeps SUT construction readable: MOCKING owns the intricate setup, SETUP exposes ready-to-use interfaces via `env*`, and SYSTEM UNDER TEST reads like a clean recipe. Skipping the indirection devolves constructor calls into walls of `.Object` accessors that reviewers must mentally untangle.

###### **Test Fakes (a.k.a. Dummies)**
Test fakes are lightweight, purpose-built implementations of an interface that you author specifically for the test suite. They shine when mocking frameworks become gnarly—especially for complex interaction verification or structured log inspection—because you can write intention-revealing code without wrestling with callback signatures or argument matchers.

- **Create fakes when setup or verification with a mock would be noisy, repetitive, or brittle.** If verifying interactions requires delegates, reflection, or deeply nested `It.Is` expressions, a fake is often clearer.
- **Name fake instances with the `fake*` prefix and construct them in the MOCKING section** so readers instantly recognise simulated collaborators.
- **Assign each `fake*` variable to an `env*` variable in SETUP before handing it to the SUT**, keeping SYSTEM UNDER TEST clean.
- **Interact with the fake via its `fake*` variable in the THEN, LOGGING, and BEHAVIOR sections.** Fakes often expose custom assertion helpers that capture behaviour without `mock.Verify(...)` boilerplate.
- **Document expectations inside the fake when possible.** Purpose-built helpers make intent obvious and reduce duplicated parsing logic across tests.

**Strengths Compared to Mocks**
- **Readable behaviour verification.** Fakes encapsulate verification logic in methods that read like English, avoiding the visual clutter of `Times.Once()` and `It.IsAny<T>()` chains.
- **Reduced setup friction.** You control the implementation, so constructors and helpers can mirror your domain vocabulary.
- **Deterministic assertions.** Fakes can capture state (for example, recorded log entries) and expose first-class assertions, lowering the risk of brittle, order-dependent verifications.

**Weaknesses Compared to Mocks**
- **Maintenance cost.** You must maintain the fake implementation alongside production interfaces.
- **Limited behavioural coverage.** A fake typically encodes only the paths needed by the current test suite.
- **Risk of drifting from reality.** Because a fake is handwritten, it might omit subtle behaviour the real dependency exhibits; fixtures and integration tests guard against divergence.

**Example: Logging with a Fake vs. a Mock**
Consider a service that logs a structured trace message—`"Foo(branch=bar): the foo is quite barred"`—whenever it processes a branch.

**✅ Fake-based test (clean and intention revealing):**
```csharp
[Fact]
public void Test_ProcessBranch_LogsTraceMessage()
{
    // GIVEN: a branch identifier that should trigger logging
    string givenBranchId = "bar";

    // MOCKING: create test fakes for collaborators
    FakeLogger fakeLogger = new();

    // SETUP: expose the fake through env* variables
    ILogger envLogger = fakeLogger;

    // SYSTEM UNDER TEST: inject the fake logger
    BranchProcessor sut = new(envLogger);

    // WHEN: process the branch
    sut.ProcessBranch(givenBranchId);

    // LOGGING: ensure the trace message was written
    fakeLogger.AssertExercised("Foo", "bar");
}

private sealed class FakeLogger : ILogger
{
    private readonly List<(string Category, string Branch, string Message)> _entries = new();

    public IDisposable BeginScope<TState>(TState state) => NullScope.Instance;

    public bool IsEnabled(LogLevel logLevel) => logLevel <= LogLevel.Trace;

    public void Log<TState>(LogLevel logLevel, EventId eventId, TState state, Exception? exception, Func<TState, Exception?, string> formatter)
    {
        if (logLevel == LogLevel.Trace && state is { } payload)
        {
            string message = formatter(payload, exception);
            _entries.Add(("Foo", ExtractBranch(payload), message));
        }
    }

    public void AssertExercised(string expectedCategory, string expectedBranch)
    {
        (string Category, string Branch, string Message) entry = Assert.Single(_entries);
        Assert.Equal(expectedCategory, entry.Category);
        Assert.Equal(expectedBranch, entry.Branch);
        Assert.Equal($"Foo(branch={expectedBranch}): the foo is quite barred", entry.Message);
    }

    private static string ExtractBranch<TState>(TState state) => state switch
    {
        { } when state is IReadOnlyDictionary<string, object?> dict && dict.TryGetValue("branch", out object? value) => value?.ToString() ?? string.Empty,
        _ => string.Empty
    };

    private sealed class NullScope : IDisposable
    {
        public static readonly NullScope Instance = new();
        public void Dispose() { }
    }
}
```

This fake keeps the test sections focused: the MOCKING block builds a dedicated helper (`FakeLogger`), the SETUP block wires it through `envLogger`, and the LOGGING section uses a single, expressive assertion. The helper encapsulates state capture and verification, so the test reads like a story.

**🚫 Mock-based alternative (gnarly and verbose):**
```csharp
[Fact]
public void Test_ProcessBranch_LogsTraceMessage_WithMock()
{
    // GIVEN: a branch identifier that should trigger logging
    string givenBranchId = "bar";

    // MOCKING: configure a strict logger mock to capture the structured message
    Mock<ILogger> mockLogger = new(MockBehavior.Strict);
    mockLogger
        .Setup(logger => logger.Log(
            LogLevel.Trace,
            It.IsAny<EventId>(),
            It.Is<It.IsAnyType>((state, _) =>
                state.ToString() == "Foo(branch=bar): the foo is quite barred" &&
                state.GetType().GetProperty("branch")?.GetValue(state)?.Equals("bar") == true),
            null,
            It.IsAny<Func<It.IsAnyType, Exception?, string>>()
        ))
        .Verifiable("Expected trace log was not emitted");

    // SETUP: expose the mock through env* variables
    ILogger envLogger = mockLogger.Object;

    // SYSTEM UNDER TEST: inject the mock logger
    BranchProcessor sut = new(envLogger);

    // WHEN: process the branch
    sut.ProcessBranch(givenBranchId);

    // BEHAVIOR: verify the mock interaction
    mockLogger.Verify();
}
```

Even a simple log assertion requires `It.Is<It.IsAnyType>` gymnastics, reflection to examine anonymous types, and a final `Verify()` call. The fake-based version hides that complexity and offers a single, intention-revealing `AssertExercised` method. When mocks feel this gnarly, prefer a fake.

**✅ Example with capture variables and `env*` indirection (clean SUT construction):**
```csharp
[Fact]
public async Task Test_SubmitOrder_SchedulesShipment()
{
    // GIVEN: an order that should be shipped immediately
    Guid givenOrderId = Guid.NewGuid();
    Order givenOrder = new(givenOrderId, ShipImmediately: true);

    // CAPTURE: observe the scheduled shipment sent to the message bus
    ShipmentScheduled? capturePublishedShipment = null;

    // MOCKING: configure repository, clock, and message bus behaviors
    Mock<IOrderRepository> mockRepository = new(MockBehavior.Strict);
    mockRepository.Setup(repo => repo.GetAsync(givenOrderId)).ReturnsAsync(givenOrder).Verifiable();
    Mock<ISystemClock> mockClock = new(MockBehavior.Strict);
    mockClock.Setup(clock => clock.UtcNow).Returns(DateTimeOffset.Parse("2024-01-01T12:00:00Z")).Verifiable();
    Mock<IMessageBus> mockMessageBus = new(MockBehavior.Strict);
    mockMessageBus
        .Setup(bus => bus.PublishAsync(It.IsAny<ShipmentScheduled>()))
        .Callback<ShipmentScheduled>(message => capturePublishedShipment = message)
        .Returns(Task.CompletedTask)
        .Verifiable();

    // SETUP: expose the mocks through the environment dependencies used by the SUT
    IOrderRepository envRepository = mockRepository.Object;
    ISystemClock envClock = mockClock.Object;
    IMessageBus envMessageBus = mockMessageBus.Object;
    OrderProcessingOptions envOptions = new() { BatchSize = 20 };

    // SYSTEM UNDER TEST: wire dependencies without revealing mock plumbing
    OrderProcessor sut = new(envRepository, envClock, envMessageBus, envOptions);

    // WHEN: exercise the behavior under test
    await sut.SubmitOrderAsync(givenOrderId);
    ShipmentScheduled? actualPublishedShipment = capturePublishedShipment;

    // EXPECTATIONS: the shipment should reference the order id from GIVEN
    Guid expectedOrderId = givenOrderId;

    // THEN: verify the captured message matches expectations
    Assert.NotNull(actualPublishedShipment);
    Assert.Equal(expectedOrderId, actualPublishedShipment!.OrderId);

    // BEHAVIOR: ensure the message bus publish call occurred
    mockMessageBus.Verify();
}
```

**🚫 Example without `env*` indirection (hard-to-read SUT construction):**
```csharp
[Fact]
public async Task Test_SubmitOrder_SchedulesShipment()
{
    Mock<IOrderRepository> mockRepository = new(MockBehavior.Strict);
    Mock<ISystemClock> mockClock = new(MockBehavior.Strict);
    Mock<IMessageBus> mockMessageBus = new(MockBehavior.Strict);
    mockRepository.Setup(r => r.GetAsync(It.IsAny<Guid>())).ReturnsAsync(new Order(Guid.NewGuid(), true));
    mockClock.Setup(c => c.UtcNow).Returns(DateTimeOffset.UtcNow);
    mockMessageBus.Setup(b => b.PublishAsync(It.IsAny<ShipmentScheduled>())).Returns(Task.CompletedTask);

    // SYSTEM UNDER TEST: constructor obscured by inline .Object calls and configuration noise
    OrderProcessor sut = new(
        mockRepository.Object,
        mockClock.Object,
        mockMessageBus.Object,
        new OrderProcessingOptions { BatchSize = 20 }
    );

    await sut.SubmitOrderAsync(Guid.NewGuid());
}
```
In the second snippet the constructor is dominated by mock plumbing, forcing the reader to mentally map each `.Object` back to the earlier declarations. By contrast, the first snippet lets the SYSTEM UNDER TEST section focus on the collaborators themselves (`envRepository`, `envClock`, etc.), so the purpose of each dependency remains obvious even when the setup logic is complex.
- **Prefer verifying log output in addition to mock verifications**—capturing and asserting on log entries is usually easier to implement correctly, more readable for future maintainers, and often avoids the “gnarly” setup required to capture mock parameters.
- **All mocks must be created in verifiable mode**: use `new Mock<T>(MockBehavior.Strict)` to catch unexpected calls or call `.Verifiable()` on each `mock.Setup(...)` you intend to verify.
- **Every mock setup must be verified in the BEHAVIOR section** via `mock.Verify(...)` or `mock.VerifyAll()`; unverified verifiable mocks should fail the test.

✅ **Good Example:**
```csharp
[Test]
public void Test_ServiceCallsDependency()
{
    // MOCKING: create a strict dependency mock
    Mock<IDependency> mockDependency = new(MockBehavior.Strict);
    mockDependency.Setup(x => x.PerformTask()).Verifiable();

    // SETUP: expose the mock through the dependency interface
    IDependency envDependency = mockDependency.Object;

    // SYSTEM UNDER TEST
    var sut = new MyService(envDependency);

    // WHEN
    sut.DoWork();

    // BEHAVIOR
    mockDependency.Verify(x => x.PerformTask(), Times.Once());
}
```

🚫 **Bad Example:**
```csharp
[Test]
public void Test_ServiceCallsDependency()
{
    var mockDependency = new Mock<IDependency>();  // ❌ No section-separators
    var service = new MyService(mockDependency.Object); // ❌ passing the mock object directly to the SUT
    service.DoWork();
    mockDependency.Verify(x => x.PerformTask(), Times.Once());
}
```

#### **5.1.4 SETUP**
- Perform all test initialization that prepares the non-mocked environment (e.g., creating `HttpContext`, configuring dependency injection, and setting up request parameters).
- Variables created in this section should be prefixed with `env*` (e.g. `envHttpContext`, `envActionContext`).
- When constants or variables are needed in this section that are important for comprehending the test, declare them as `given*` variables instead.
- SETUP should follow the preceding arrangement sections (GIVEN → CAPTURE → MOCKING). Omit intermediate sections when they are unnecessary.
- SETUP should refer to `given*` variables, not `mock*` variables, except in rare occasions.
- Assign mock objects to real interface variables in SETUP, using the `env*` prefix (e.g., `envLogger = mockLogger.Object`). The `mock*` variables themselves may only be used in the SETUP and BEHAVIOR sections.
- **The SETUP section must not be merged with any other section.**
- If SETUP involves multiple distinct stages (e.g. reading fixtures, wiring dependencies, configuring environment), split it into multiple sub-sections with descriptive comments.

#### **5.1.5 SYSTEM UNDER TEST**
- Assign the system under test to a variable named `sut`.
- If multiple systems are being tested, assign them to separate `sut*` variables.
- SYSTEM UNDER TEST must never directly reference `mock*` variables; use the `env*` variables initialised in SETUP.
- **The SYSTEM UNDER TEST section must not be merged with any other section.**

#### **5.1.6 Fixtures: Best Practices**
Reusable fixtures encourage realistic, DRY data that mirrors production scenarios while keeping tests focused on behaviour instead of data plumbing. Inlining bespoke objects everywhere invites duplication, increases maintenance costs when models evolve, and makes it harder to reason about how changes cascade across tests.
- **Prefer fixtures over mocks**—favour shared, reusable fixtures rather than creating one-off test data.
- **Instantiate fixtures in the appropriate Arrange subsection (usually GIVEN or SETUP)** and modify them as needed to fit the test's needs.
- **Use a sharable fixture factory** instead of creating fixtures or complex test data inline.

✅ **Good Example:**
```csharp
public static class Fixtures
{
    public static string IpAddress() => "192.168.1.1";

    public static CwConfig CwConfig() => new() { Terminals = new List<HardwareTerminalConfig>() };

    public static CwGetTerminalConfigCommand CwGetTerminalConfigCommand()
    {
        string givenIpAddress = IpAddress();
        return new CwGetTerminalConfigCommand { TerminalFixedIpAddress = givenIpAddress };
    }

    public static CommandContext<CwGetTerminalConfigCommand, CwGetTerminalConfigResult> CommandContext() => new()
    {
        Command = CwGetTerminalConfigCommand()
    };
}

public class CarWashesServiceTests
{
    [Test]
    public async Task Test_GetTerminalConfig_ReturnsNotFound_WhenTerminalDoesNotExist()
    {
        // GIVEN
        var givenTerminalIp = Fixtures.IpAddress();
        var givenCommandContext = Fixtures.CommandContext();
        var givenCarWashesConfig = Fixtures.CwConfig();

        // SYSTEM UNDER TEST
        var sut = new CarWashesService();

        // WHEN
        await sut.GetTerminalConfig(givenCommandContext);
        var actualStatusCode = givenCommandContext.StatusCode;
        var actualResult = givenCommandContext.Result;

        // EXPECTATIONS
        var expectedStatusCode = HttpStatusCode.NotFound;
        CwGetTerminalConfigResult expectedResult = null;

        // THEN
        Assert.Equal(expectedStatusCode, actualStatusCode);
        Assert.Equal(expectedResult, actualResult);
    }
}
```

🚫 **Bad Example:**
```csharp
[Test]
public async Task Test_GetTerminalConfig_ReturnsNotFound_WhenTerminalDoesNotExist()
{
    // GIVEN
    var givenTerminalIp = "192.168.1.1";
    var givenCommandContext = new CommandContext<CwGetTerminalConfigCommand, CwGetTerminalConfigResult>
    {
        Command = new CwGetTerminalConfigCommand { TerminalFixedIpAddress = givenTerminalIp }
    };
    var givenCarWashesConfig = new CwConfig { Terminals = new List<HardwareTerminalConfig>() };
    StaticConfig.CarWashesConfig = givenCarWashesConfig;

    // SYSTEM UNDER TEST
    var sut = new CarWashesService();

    // WHEN
    await sut.GetTerminalConfig(givenCommandContext);
    var actualStatusCode = givenCommandContext.StatusCode;
    var actualResult = givenCommandContext.Result;

    // EXPECTATIONS
    var expectedStatusCode = HttpStatusCode.NotFound;
    CwGetTerminalConfigResult expectedResult = null;

    // THEN
    Assert.Equal(expectedStatusCode, actualStatusCode);
    Assert.Equal(expectedResult, actualResult);
}
```

### **5.2 Act**
The Act stage performs the single behaviour under test. Keep it focused on invoking the SUT so reviewers can immediately spot what triggers the scenario.

#### **5.2.1 WHEN**
- Perform the action or method call being tested.
- Assign the results to `actual*` variables. This typically captures the return value from invoking the system under test.
  - If later sections need to inspect additional outputs (e.g. side effects or mutated collaborators), assign them to their own `actual*` variables immediately after invocation.
- If the test needs to use an `env*` value later, assign it to an `actual*` variable in WHEN and use that instead.
- Move any values stored in `capture*` placeholders into appropriately named `actual*` variables before asserting on them in THEN or BEHAVIOR.
- **The WHEN section must not be merged with any other section.**
- When using helpers such as `Assert.Throws`, treat the wrapper as part of WHEN. Store the resulting exception in an `actual*` variable and assert on it in THEN.

### **5.3 Assert**
The Assert stage records expectations up front and verifies them explicitly. Separating the declaration of expectations from the assertions themselves keeps the intent clear and prevents verification from being scattered throughout the test.

#### **5.3.1 EXPECTATIONS**
- Define expected values **before assertions**.
- Assign expected values to `expected*` variables.
- Place EXPECTATIONS strictly after WHEN and before THEN.
- This section should not refer to any `actual*` variables.
- **The EXPECTATIONS section must not be merged with any other section.**

#### **5.3.2 THEN**
- Perform **assertions** comparing actual results to expected values.
- **Never assert against literals**—always use `expected*` variables.
- **The THEN section must not be merged with any other section.**

#### **5.3.3 LOGGING**
- Verify any behaviour that can be validated via captured logging operations.
- Prefer asserting on emitted logs over (or in addition to) mock verifications; log-based assertions are simpler to write and review and often replace complex mock setups.
- Each unit test should explicitly verify that the execution branch it is meant to cover is taken, reducing false positives when setup does not exercise the desired branch.

#### **5.3.4 BEHAVIOR**
- Contains assertions for **mock interactions** (e.g., `mockService.Verify(x => x.DoSomething(), Times.Once());`).
- This section **must be present if mocks are used**.
- **The BEHAVIOR section must not be merged with any other section.**
- **All mocks must be created as verifiable**—either using `new Mock<T>(MockBehavior.Strict)` or by calling `.Verifiable()` on each `.Setup(...)`.
- **Every verifiable setup must be explicitly verified** in this section via `mock.Verify(...)` or `mock.VerifyAll()`. Unverified mocks should cause the test to fail.

✅ **Good Example:**
```csharp
[Test]
public void Test_ProcessData_ReturnsCorrectValue()
{
    // GIVEN
    var givenInput = "sample";

    // SYSTEM UNDER TEST
    var sut = new DataService();

    // WHEN
    var actualResult = sut.ProcessData(givenInput);
    var actualSideEffect = Foo.Bar();

    // EXPECTATIONS
    var expectedResult = "processed_sample";
    var expectedSideEffect = "fubar";

    // THEN
    Assert.AreEqual(expectedResult, actualResult);
    Assert.AreEqual(expectedSideEffect, actualSideEffect);
}
```

🚫 **Bad Example:**
```csharp
[Test]
public void Test_ProcessData_ReturnsCorrectValue()
{
    // MOCKING
    Mock<IFoo> mockFoo = new();

    // SYSTEM UNDER TEST
    var sut = new DataService(mockFoo.Object);
}
```

---
## **8. Assertions & Variable Naming**
Strict naming patterns for expected and actual values highlight the difference between inputs, outputs, and verifications, which speeds up failure analysis. Mixing literals and setup variables inside assertions hides intent, makes diffs noisy, and increases the chance of asserting against the wrong data.
- **Expected values** always assign input values (from the GIVEN section) to `expected*` variables in the EXPECTATIONS section if you intend to assert on them in the THEN or BEHAVIOR sections.
- **Actual results** all actual results must be assigned to `actual*` variables in the WHEN section.
- **Never assert against literals directly** — use `expected*` variables.
- **Never assert against `given*` variables directly** — assign them to`expected*` variables in the EXPECTATIONS section and use those.
- **Never assert against `env*` variables directly** — assign them to`actual*` variables exclusively in the WHEN section and use those.
- **Never assert against `mock*` variables directly in the THEN section** — assign them to`expected*` variables in the EXPECTATIONS section and use those.
- **It is acceptable to assert against `mock*` variables directly in the BEHAVIOR section** .
- To reiterate: **Never assert directly against given* or env* variable** — always use the `expected*` or `actual*` variables in the THEN section.
- To reiterate: **This also holds true for mocks** — always use the `expected*` or `actual*` variables in the BEHAVIOR section when asserting on the behavior of mock objects, which must be named `mock*`.
- To reiterate once more: assertions should never ever ever refer to `given*` or `env*` values, and should only refer to `mock*` variables in the BEHAVIOR section.

✅ **Good Example:**
```csharp
var expectedResult = 42;
Assert.AreEqual(expectedResult, actualResult);
```
🚫 **Bad Example:**
```csharp
Assert.AreEqual(42, actualResult); // ❌ No expected variable
```

---

## **9. Exception Handling (`Assert.Throws`)**
Treating exception capture as part of the WHEN stage keeps control flow explicit and prevents assertions from being buried inside delegates. When the thrown exception is not stored and verified deliberately, tests can pass for the wrong reason, masking regressions where different error types or messages are emitted.

When using a mechanism such as Assert.Throws to wrap the SUT invocation, the exception name that is being asserted against
will be placed above the code block that exercises the system under test. In such cases, consider the Assert.Throws mechanism
to be part of the WHEN section. Assign the actual exception to a local variable and assert that it is as expected in a
subsequent THEN section

✅ **Good Example:**
```csharp
[Test]
public void Test_DivideByZero_ThrowsException()
{
    // GIVEN
    var givenNumerator = 10;
    var givenDenominator = 0;

    // WHEN
    DivideByZeroException actualException = Assert.Throws<DivideByZeroException>(() =>
    {
        Math.Divide(givenNumerator, givenDenominator);
    });

    // EXPECTATIONS
    Type expectedExceptionType = typeof(DivideByZeroException);

    // THEN
    Assert.Equal(expectedExceptionType, actualException.GetType());
}

---

## **10. Using `[SetUp]` Methods**
Being intentional about `[SetUp]` usage ensures shared initialization is truly common while test-specific context stays near the scenario. Overusing setup hooks leads to hidden coupling between tests, complicated fixture state, and surprises when one case mutates shared members that another silently depends on.
- **Avoid `[SetUp]`**. Favor using sharable fixtures, declared in a static fixture factory class, instead.
- Use `[SetUp]` only for **repeated initialization**.

✅ **Good Example:**
```csharp
[SetUp]
public void Setup()
{
    _service = new MyService();
}
```
🚫 **Bad Example:**
```csharp
[SetUp]
public void Setup()
{
    var service = new MyService(); // ❌ Redundant per-test instantiation
}
```

---

## **11. Organizing Tests in Namespaces**
Mirroring production namespaces in the test suite keeps navigation intuitive and tooling-friendly, so developers can jump between code and tests effortlessly. When namespace structures drift, IDE search results become noisy, automated discovery may misbehave, and contributors struggle to find the right home for new cases.
- **Group related test classes into namespaces**.
- Ensure **each namespace matches the SUT structure**.
- use the 'namespace <name>;' form rather than the 'namespace <name> {}' form.

✅ **Good Example:**
```csharp
namespace Tests.ServicesTests;

public class UserServiceTests { ... }
```

🚫 **Bad Example:**
```csharp
namespace Services.Tests; // ❌ Inconsistent naming

public class UserServiceTests { ... }
```

🚫 **Bad Example:**
```csharp
namespace Services.ServicesTests // ❌ do not increase the indent for namespaces
{
    public class UserServiceTests { ... }
}
```

---

## **12. Managing Copy-Paste Setup**
Copy-paste programming is often the most readable choice for test setup. Start with a "happy path" test that spells out the full arrangement, and copy that scaffolding into edge-case tests, tweaking only what changes for each branch. With disciplined sectioning and modern AI-assisted refactoring tools, duplicating the setup is usually quicker and clearer than inventing cryptic helper methods whose behavior must be reverse-engineered.
- Prefer duplication first, then refactor **only when** the abstraction makes the setup easier to understand than the copied code it replaces.
- When a shared helper truly improves clarity, keep it near the tests that use it and document which scenarios rely on it so future contributors know when to extend or bypass it.
- When you do copy setup code, call out the variations in the section-header comments (e.g. `// GIVEN: user has insufficient balance`). These comments act as signposts so reviewers can immediately see why one test diverges from another despite similar bodies.

---

## **13. Deterministic Tests and Dependency Injection**
Unit tests should be deterministic: the same inputs must produce the same results every run. Nondeterminism from global state, random values, or "current time" APIs usually erodes trust in the suite. Occasionally, allowing nondeterminism in aspects that **should not** affect the outcome is valuable—flaky failures in those cases expose hidden coupling or misunderstood behavior. Treat such flakes as signals to either fix the bug they reveal or document the learning and firm up the contract under test.
- Isolate side effects behind interfaces and inject them into the system under test. Use dependency injection (DI) so the test can supply controlled fakes, especially for time-sensitive behavior.
- Avoid calling `DateTime.Now`, `Guid.NewGuid()`, or static singletons from production code during tests. Instead, inject a clock, ID generator, or configuration provider that the test can stub.
- For time-sensitive logic, pass an `ISystemClock` or similar abstraction. Tests can freeze "now" to a predictable value while production supplies a real clock, making the deterministic path explicit.
- When code mixes deterministic calculations with nondeterministic reads, refactor into two methods: one that gathers inputs (time, randomness, HTTP responses) and another that performs pure computation. Unit test the deterministic method directly and cover the integration path separately if needed.

✅ **Example: controlling current time with DI**
```csharp
public interface IClock
{
    DateTimeOffset UtcNow { get; }
}

public class InvoiceService
{
    private readonly IClock _clock;

    public InvoiceService(IClock clock) => _clock = clock;

    public Invoice CreateInvoice(Order order)
    {
        DateTimeOffset generatedAt = _clock.UtcNow;
        return new Invoice(order.Id, generatedAt);
    }
}

[Fact]
public void Test_CreateInvoice_UsesProvidedClock()
{
    // GIVEN
    DateTimeOffset givenNow = DateTimeOffset.Parse("2024-02-01T09:30:00Z");
    Mock<IClock> mockClock = new(MockBehavior.Strict);
    mockClock.Setup(clock => clock.UtcNow).Returns(givenNow).Verifiable();

    // SYSTEM UNDER TEST
    IClock envClock = mockClock.Object;
    InvoiceService sut = new(envClock);

    // WHEN
    Invoice actualInvoice = sut.CreateInvoice(new Order(Guid.Empty));

    // EXPECTATIONS
    DateTimeOffset expectedGeneratedAt = givenNow;

    // THEN
    Assert.Equal(expectedGeneratedAt, actualInvoice.GeneratedAt);

    // BEHAVIOR
    mockClock.Verify(clock => clock.UtcNow, Times.Once);
}
```

---

## **14. Using comments in tests**
Documenting test intent with XML comments provides high-level context that complements the structured sections, guiding readers through why a scenario exists. Without these explanations, future maintainers must infer purpose from mechanics alone, risking redundant cases or accidental deletion of critical coverage.

- **Each test method should be commented with a human-readable explanation of what the test is exercising**
- **Each test class should be commented with a human-readable explanation of what the test class is exercising**
- **Use the XML comment syntax to comment test classes and test methods**

## **15. Increasing test clarity with section comments**
Narrated section headers transform dense setup or verification code into self-explanatory stories that highlight intent and edge cases. Skipping these comments forces reviewers to reverse-engineer the reason behind each block, slowing code reviews and making it easier for subtle regressions to slip through.

**As a best practice**, when tests have complex parts such as setting up mocks, creating complex 'given' objects, asserting on mock behavior, and so forth, it is recommended to split each such part into a section of its own, and comment what that section is doing. These headers act like a decryption key for the reader: each full-sentence comment explicitly states the intent of the code beneath it, so that even when the implementation is dense or unfamiliar, the reader can immediately understand _why_ a block exists rather than mentally reverse-engineering the setup from the statements themselves.

✅ **Good Example:**
```csharp
namespace Tests.ServicesTests;

/// <summary>
/// Covers the 'Bar' region of the 'Foo.User.Service' class
/// </summary>
public class UserServiceTests
{

    /// <summary>
    /// Covers the case where FooBar == "foo"
    /// </summary>
    [Fact]
    public void Test_FooBar_IsFoo()
    {
        // GIVEN: LogLevel.Information is enabled
        bool givenInfoLevelEnabled = true;

        // MOCKING: create a mocked logger that has info level enabled
        Mock<ILogger<UserService>> mockLogger = new Mock<ILogger<UserService>>();
        mockLogger.Setup(l => l.IsEnabled(LogLevel.Information)).Returns(givenInfoLevelEnabled);

        // SETUP: hide the fact that we're using a mocked logger
        ILogger<UserService> envLogger = mockLogger.Object;

        // SYSTEM UNDER TEST: instantiate and initialize the system under test
        UserService sut = new UserService(envLogger);

        // WHEN: exercise the test case
        bool actualResult = sut.IsFoo();

        // EXPECTATION: we expect a log message to have been emitted
        string expectedLogMessageFragment = "IsFoo was called";

        // EXPECTATION: we expect the result of IsFoo to be true
        bool expectedResult = true;

        // THEN: Verify that the actual status code and result match expectations.
        Assert.Equal(expectedResult, actualResult);

        // BEHAVIOR: we expect IsFoo to have logged a message on info level
        mockLogger.Verify(
            log => log.Log(
                It.Is<LogLevel>(l => l == LogLevel.Information),
                It.IsAny<EventId>(),
                It.Is<It.IsAnyType>((v, t) => v.ToString()!.Contains(expectedLogMessageFragment)),
                null,
                It.IsAny<Func<It.IsAnyType, Exception?, string>>()
            ),
            Times.Once,
            $"Expected an '{expectedLogMessageFragment}' log entry."
        );
    }
}
```

🚫 **Bad Example:**
```csharp
namespace Services.Tests; // ❌ Inconsistent naming

public class UserServiceTests
{
    [Fact]
    public void Test_FooBar_IsFoo()
    {
    }
}
```

---

## **16. SETUP / SUT Dependency Rule**
Reinforcing the separation between SETUP dependencies and the system under test protects against leaky abstractions and keeps constructor wiring transparent. When mocks or fixtures sneak directly into SUT creation, the test intent becomes opaque, accidental coupling increases, and regressions slip in because collaborators are no longer clearly expressed through `env*` variables.

To re-iterate:

- The **SETUP** section must initialize all dependencies passed to the **SUT** via `env*` variables.
- The **SYSTEM UNDER TEST** section must construct the **SUT** using only `env*` variables.
- Mocks must never be used directly in the **SYSTEM UNDER TEST** section.

---

## **17. Breaking the Rules**
The guidelines above are intentionally prescriptive so tests remain predictable and reviewable. However, real-world systems occasionally demand exceptions.
- Deviation is acceptable when it **improves clarity or determinism** for a specific edge case that the standard pattern cannot express cleanly.
- When you break a rule, **document the rationale inline** (e.g. with a comment or XML doc) so future maintainers understand why the exception exists.
- Re-evaluate exceptions periodically. If the surrounding production code changes, the original constraint may disappear and the standard structure can be restored.

