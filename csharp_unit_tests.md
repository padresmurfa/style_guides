# C# Unit Testing Style Guide

This document defines **best practices** for writing **clear, structured, and maintainable** unit tests in C# using xUnit, NUnit, or MSTest.

---

## **Target Audience**

The primary target audience for this document is developers and Large Language Models (LLMs) acting as coding assistants. By following these guidelines, tests will be **structured, readable, and maintainable**.

These practices may be too detailed for human-written tests but serve as an excellent guide for generating automated tests with LLM assistance. **Review all test functions carefully**, as human oversight is crucial to ensure test quality.

---

## **How to Use This Guide**

This document intentionally leans toward exhaustive structure so that large, generated suites stay readable. When adopting the rules in day-to-day development, keep the following guardrails in mind:

- **Optimise for clarity first.** Reach for the full section taxonomy (GIVEN/CAPTURE/MOCKING/…) when it helps the reader track complex flows. For lightweight tests, a slimmed-down Arrange → Act → Assert structure with descriptive comments is acceptable as long as intent stays obvious.
- **Prefer consistency within a file over mechanical adherence.** If an existing test class uses concise comments or omits CAPTURE entirely, mirror that local style instead of introducing a competing pattern in the same file.
- **Document deviations.** When you deliberately skip a section or rely on inline literals for trivial assertions, leave a short comment that explains the choice. Future maintainers should never have to guess whether a missing section was intentional or an oversight.
- **Revisit guidance during reviews.** Treat this guide as a living document—if a rule obstructs readability for a particular scenario, capture the exception in code review and feed the learning back into the guide.

These pragmatic allowances keep the style guide approachable while preserving the deliberate naming and structure that make the tests easy to scan.

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
- When a region contains helper methods that are not directly invoked by the public API, change those helpers from `private` to `internal` and test them directly. Configure the associated project so the test assembly can access internals rather than relying on metaprogramming to reach private members.

### **3.1 Testing Helper Methods and Other Non-Public APIs**
- Prefer testing helper methods directly by making them `internal` and annotating the production project so the test assembly can access those internals. This is easier to maintain and reason about than crafting reflection or dynamic proxies that simulate access to `private` members.
- Update each relevant `.csproj` file with an `InternalsVisibleTo` entry so your tests can reference the newly `internal` helpers:
  ```xml
  <ItemGroup>
    <InternalsVisibleTo Include="MyCompany.Products.Services.Tests" />
  </ItemGroup>
  ```
  You may alternatively add `[assembly: InternalsVisibleTo("MyCompany.Products.Services.Tests")]` to the production project when the codebase centralizes assembly metadata.
- Avoid leaving helpers `private` and relying on metaprogramming wrappers just to execute them—promoting them to `internal` keeps their visibility tight while enabling straightforward tests.
- **Protected methods cannot be made accessible via `InternalsVisibleTo`.** Exercise them by either:
  - Using metaprogramming tools such as reflection to locate and invoke the protected member:
    ```csharp
    [Fact]
    public void Validate_RejectsCancelledOrders()
    {
        // GIVEN: a cancelled order candidate
        Order givenCancelledOrder = Order.Cancelled("123");

        // SETUP: capture the protected method while keeping the act phase clean
        var envValidateRepository = typeof(Repository)
            .GetMethod("Validate", BindingFlags.Instance | BindingFlags.NonPublic | BindingFlags.FlattenHierarchy)!;

        var envInvokeValidate = (Repository repository, Order candidate) =>
        {
            _ = envValidateRepository.Invoke(repository, new object[] { candidate });
        };

        var envRepository = new Repository(database: new FakeDatabase());

        // SYSTEM UNDER TEST
        Action<Order> sut = candidate => envInvokeValidate(envRepository, candidate);

        // WHEN: invoking the protected method through the captured delegate
        InvalidOperationException actualException = Assert.Throws<InvalidOperationException>(() => sut(givenCancelledOrder));

        // THEN: the protected method rejects cancelled orders
        Assert.IsType<InvalidOperationException>(actualException);
    }
    ```
  - Creating a test double that inherits the production class and exposes a forwarding method that calls the protected helper:
    ```csharp
    private sealed class RepositoryDouble : Repository
    {
        public RepositoryDouble(IDatabase database) : base(database) { }

        public void InvokeValidate(Order candidate) => Validate(candidate);
    }

    [Fact]
    public void Test_Validate_RejectsCancelledOrders()
    {
        // GIVEN: a repository double and a cancelled order
        RepositoryDouble givenRepository = new(database: new FakeDatabase());
        Order givenOrder = Order.Cancelled("123");

        // WHEN
        Assert.Throws<InvalidOperationException>(() => givenRepository.InvokeValidate(givenOrder));
    }
    ```

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
- **Use parameterized tests judiciously**. They shine when exercising the same behaviour across multiple datasets; prefer
  explicit individual tests when each case needs unique setup or assertions.
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

When brevity clearly communicates intent (for example, asserting a pure function or mapping with no collaborators), a compact test that sticks to a single `// ARRANGE`, `// ACT`, `// ASSERT` progression is acceptable. Treat the detailed sections as building blocks—pull them in once the setup stops fitting on a screen or when additional naming would make the control flow easier to reason about.

Each test method follows a structured format, **separated by clear comments**:
- Follow the Arrange → Act → Assert order unless a section is unnecessary; omit empty sections entirely.
- Section comment headers should be full descriptive sentences (e.g. `// GIVEN: A valid context and wash code details`), not terse labels.
- When a single section becomes long, divide it with additional human-readable sub-headers so readers can scan for intent quickly.
- When a helper value lives entirely inside one section but helps construct a `given*`, `env*`, `expected*`, `actual*`, or other cross-section variable, name that helper with the `tmp*` prefix (e.g. `tmpSerializedOrder`). The `tmp*` prefix signals that the variable is a disposable staging value and prevents pollution of the more meaningful naming families.

### **5.1 Arrange**
The Arrange stage gathers inputs, configures doubles, and exposes collaborators to the system under test. Keeping all preparation here prevents leakage of setup noise into later sections.

#### **5.1.1 GIVEN**
- Set up initial conditions and inputs.
- Assign variables with the prefix `given`.
- When a helper exists only to build or transform data for a `given*` variable, name it with the `tmp*` prefix so readers know it never leaves the GIVEN section.
- When test fixtures are used, allocate them in this section.
- **Test Fixture classes** should be named `Fixture*` and assigned to `given*` variables prior to use.
- The GIVEN section should always be the first section in a test, unless there is nothing to define in GIVEN, in which case it should be omitted.
- **The GIVEN section must not be merged with any other section.**

✅ **Good Example:**
```csharp
[Fact]
public void Test_Parse_Succeeds_WithWellFormedPayload()
{
    // GIVEN: a JSON payload with the required fields
    string givenPayload = "{ \"id\": 1, \"name\": \"Ada\" }";

    // WHEN
    ParsedResult actualResult = Parser.Parse(givenPayload);

    // EXPECTATIONS
    string expectedName = "Ada";

    // THEN
    Assert.Equal(expectedName, actualResult.Name);
}
```

🚫 **Bad Example:**
```csharp
[Fact]
public void Test_Parse_Succeeds_WithWellFormedPayload()
{
    // WHEN: GIVEN data is inlined and unnamed
    ParsedResult actualResult = Parser.Parse("{ \"id\": 1, \"name\": \"Ada\" }");

    // EXPECTATIONS
    string expectedName = "Ada";

    // THEN
    Assert.Equal(expectedName, actualResult.Name);
}
```

#### **5.1.2 CAPTURE**
- Declare variables that will hold values captured from mocks, spies, or test fakes.
- Prefix these placeholders with `capture` (e.g. `capturePublishedMessage`).
- Keep this section focused on declarations; configure the mechanics that populate them in MOCKING or SETUP.
- Position CAPTURE immediately after GIVEN and before MOCKING whenever it is present.
- Promote capture placeholders into `actual*` variables during WHEN so assertions always reference concrete results.
- Include CAPTURE only when collaborators expose information that must be asserted later.

✅ **Good Example:**
```csharp
[Fact]
public async Task Test_SendEmail_CapturesRecipient()
{
    // GIVEN: an order that should trigger a confirmation email
    Order givenOrder = Fixtures.Order();

    // CAPTURE: observe the recipient address used by the mailer
    string? captureRecipient = null;

    // MOCKING
    Mock<IEmailGateway> mockGateway = new(MockBehavior.Strict);
    mockGateway
        .Setup(gateway => gateway.SendAsync(It.IsAny<EmailMessage>()))
        .Callback<EmailMessage>(message => captureRecipient = message.Recipient)
        .Returns(Task.CompletedTask)
        .Verifiable();

    // SETUP
    IEmailGateway envGateway = mockGateway.Object;

    // SYSTEM UNDER TEST
    ConfirmationService sut = new(envGateway);

    // WHEN
    await sut.SendOrderConfirmationAsync(givenOrder);
    string? actualRecipient = captureRecipient;

    // EXPECTATIONS
    string expectedRecipient = givenOrder.Customer.Email;

    // THEN
    Assert.Equal(expectedRecipient, actualRecipient);

    // BEHAVIOR
    mockGateway.Verify();
}
```

🚫 **Bad Example:**
```csharp
[Fact]
public async Task Test_SendEmail_CapturesRecipient()
{
    // CAPTURE is skipped and the callback writes directly to an actual* variable
    string? actualRecipient = null;

    Mock<IEmailGateway> mockGateway = new();
    mockGateway
        .Setup(gateway => gateway.SendAsync(It.IsAny<EmailMessage>()))
        .Callback<EmailMessage>(message => actualRecipient = message.Recipient);

    ConfirmationService sut = new(mockGateway.Object);
    await sut.SendOrderConfirmationAsync(Fixtures.Order());

    Assert.NotNull(actualRecipient); // ❌ No explicit CAPTURE placeholder and no EXPECTATIONS section
}
```

#### **5.1.3 MOCKING**
- Define and configure **mock objects**.
- Mock variables must be named `mock*`.
- Prefer `var` when the initializer makes the mocked type obvious; explicit types are only necessary when inference would hide important details.
- Helpers that only exist to configure mocks (for example, serialized payloads or argument lists) should use the `tmp*` prefix to make it clear they are temporary scaffolding.
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

✅ **Good Example:**
```csharp
[Fact]
public void Test_Processor_InvokesRepository()
{
    // GIVEN
    Guid givenId = Guid.NewGuid();

    // MOCKING: configure a strict repository mock
    Mock<IWidgetRepository> mockRepository = new(MockBehavior.Strict);
    mockRepository.Setup(repo => repo.Load(givenId)).Returns(new Widget()).Verifiable();

    // SETUP
    IWidgetRepository envRepository = mockRepository.Object;

    // SYSTEM UNDER TEST
    WidgetProcessor sut = new(envRepository);

    // WHEN
    sut.Process(givenId);

    // BEHAVIOR
    mockRepository.Verify();
}
```

🚫 **Bad Example:**
```csharp
[Fact]
public void Test_Processor_InvokesRepository()
{
    // MOCKING: loose mock with unclear naming
    var repository = new Mock<IWidgetRepository>(); // ❌ Variable not prefixed with mock*
    repository.Setup(r => r.Load(It.IsAny<Guid>())); // ❌ Setup is never verified

    // SYSTEM UNDER TEST: mock passed directly without env* indirection
    WidgetProcessor sut = new(repository.Object); // ❌ Violates SETUP/SUT separation

    sut.Process(Guid.NewGuid());
}
```

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
- Prefer `var` in SETUP when the assigned collaborator or fixture is obvious from the right-hand side; reserve explicit types for cases where inference would obscure intent.
- When constants or variables are needed in this section that are important for comprehending the test, declare them as `given*` variables instead.
- SETUP should follow the preceding arrangement sections (GIVEN → CAPTURE → MOCKING). Omit intermediate sections when they are unnecessary.
- SETUP should refer to `given*` variables, not `mock*` variables, except in rare occasions.
- When SETUP needs staging variables (for example, constructing configuration objects before assigning them to `env*` variables), prefer the `tmp*` prefix to show the helpers do not escape the section.
- Assign mock objects to real interface variables in SETUP, using the `env*` prefix (e.g., `envLogger = mockLogger.Object`). The `mock*` variables themselves may only be used in the SETUP and BEHAVIOR sections.
- **The SETUP section must not be merged with any other section.**
- If SETUP involves multiple distinct stages (e.g. reading fixtures, wiring dependencies, configuring environment), split it into multiple sub-sections with descriptive comments.

✅ **Good Example:**
```csharp
[Fact]
public void Test_Controller_UsesConfiguredContext()
{
    // GIVEN
    DefaultHttpContext givenHttpContext = new();

    // MOCKING
    Mock<IUserRepository> mockRepository = new(MockBehavior.Strict);
    mockRepository.Setup(repo => repo.GetCurrent()).Returns(Fixtures.User()).Verifiable();

    // SETUP: promote mocks and fixtures into env* dependencies
    IUserRepository envRepository = mockRepository.Object;
    HttpContext envHttpContext = givenHttpContext;

    // SYSTEM UNDER TEST
    UserController sut = new(envRepository)
    {
        ControllerContext = new ControllerContext { HttpContext = envHttpContext }
    };

    // WHEN
    IActionResult actualResult = sut.GetCurrentUser();

    // EXPECTATIONS
    Type expectedResultType = typeof(OkObjectResult);

    // THEN
    Assert.IsType(expectedResultType, actualResult);

    // BEHAVIOR
    mockRepository.Verify();
}
```

🚫 **Bad Example:**
```csharp
[Fact]
public void Test_Controller_UsesConfiguredContext()
{
    // SETUP section is skipped entirely
    Mock<IUserRepository> mockRepository = new();

    // SYSTEM UNDER TEST: mocks and fixtures are passed inline
    UserController sut = new(mockRepository.Object)
    {
        ControllerContext = new ControllerContext { HttpContext = new DefaultHttpContext() }
    };

    sut.GetCurrentUser();
    mockRepository.Verify(repo => repo.GetCurrent()); // ❌ No env* variables and no structured sections
}
```

#### **5.1.5 SYSTEM UNDER TEST**
- Assign the system under test to a variable named `sut`.
- If multiple systems are being tested, assign them to separate `sut*` variables.
- SYSTEM UNDER TEST must never directly reference `mock*` variables; use the `env*` variables initialised in SETUP.
- **The SYSTEM UNDER TEST section must not be merged with any other section.**

✅ **Good Example:**
```csharp
[Fact]
public void Test_Handler_ConstructsWithDependencies()
{
    // MOCKING
    Mock<IClock> mockClock = new(MockBehavior.Strict);

    // SETUP
    IClock envClock = mockClock.Object;

    // SYSTEM UNDER TEST
    ExpirationHandler sut = new(envClock);

    // WHEN
    sut.Handle();

    // BEHAVIOR
    mockClock.Verify(clock => clock.UtcNow, Times.AtLeastOnce);
}
```

🚫 **Bad Example:**
```csharp
[Fact]
public void Test_Handler_ConstructsWithDependencies()
{
    Mock<IClock> mockClock = new();

    // SYSTEM UNDER TEST: mock used directly and variable not named sut
    var handler = new ExpirationHandler(mockClock.Object); // ❌ Missing envClock indirection and sut naming

    handler.Handle();
}
```

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
- Transient calculations that help transform results before assigning to `actual*` variables may use the `tmp*` prefix to highlight that they do not leave the WHEN section.
- **The WHEN section must not be merged with any other section.**
- When using helpers such as `Assert.Throws`, treat the wrapper as part of WHEN. Store the resulting exception in an `actual*` variable and assert on it in THEN.

✅ **Good Example:**
```csharp
[Fact]
public void Test_ScoreCalculator_ReturnsExpectedScore()
{
    // GIVEN
    Player givenPlayer = Fixtures.Player();

    // SYSTEM UNDER TEST
    ScoreCalculator sut = new();

    // WHEN
    int actualScore = sut.Calculate(givenPlayer);

    // EXPECTATIONS
    int expectedScore = 9001;

    // THEN
    Assert.Equal(expectedScore, actualScore);
}
```

🚫 **Bad Example:**
```csharp
[Fact]
public void Test_ScoreCalculator_ReturnsExpectedScore()
{
    ScoreCalculator sut = new();

    // WHEN and THEN are merged; result is asserted inline
    Assert.Equal(9001, sut.Calculate(Fixtures.Player())); // ❌ Missing actual* variable and EXPECTATIONS section
}
```

### **5.3 Assert**
The Assert stage records expectations up front and verifies them explicitly. Separating the declaration of expectations from the assertions themselves keeps the intent clear and prevents verification from being scattered throughout the test.

#### **5.3.1 EXPECTATIONS**
- Define expected values **before assertions**.
- Assign expected values to `expected*` variables.
- When the expectation is a self-explanatory constant (e.g., `true`, `null`, or an enum member with an obvious name), you may
  assert directly against the literal as long as the comment above the assertion communicates intent. Introduce an
  `expected*` variable when the value needs derivation or when multiple assertions share it.
- If you need helper calculations to build an expected value (for example, formatting a timestamp), compute them in-place with `tmp*` variables so the EXPECTATIONS section remains readable and clearly scoped.
- Place EXPECTATIONS strictly after WHEN and before THEN.
- This section should not refer to any `actual*` variables.
- **The EXPECTATIONS section must not be merged with any other section.**

✅ **Good Example:**
```csharp
[Fact]
public void Test_Calculator_AddsNumbers()
{
    // GIVEN
    int givenLeft = 2;
    int givenRight = 3;

    // SYSTEM UNDER TEST
    Calculator sut = new();

    // WHEN
    int actualSum = sut.Add(givenLeft, givenRight);

    // EXPECTATIONS
    int expectedSum = givenLeft + givenRight;

    // THEN
    Assert.Equal(expectedSum, actualSum);
}
```

🚫 **Bad Example:**
```csharp
[Fact]
public void Test_Calculator_AddsNumbers()
{
    Calculator sut = new();
    int actualSum = sut.Add(2, 3);

    // EXPECTATIONS section is skipped and literals are asserted inline
    Assert.Equal(5, actualSum); // ❌ Violates EXPECTATIONS rule
}
```

#### **5.3.2 THEN**
- Perform **assertions** comparing actual results to expected values.
- **Prefer asserting against `expected*` variables.** Inline literals are acceptable for trivial expectations that are
  immediately obvious to the reader; favour named variables whenever the value is derived, reused, or benefits from explanation.
- **The THEN section must not be merged with any other section.**

✅ **Good Example:**
```csharp
[Fact]
public void Test_OrderProcessor_ReturnsCompletedStatus()
{
    // GIVEN
    Order givenOrder = Fixtures.Order();

    // SYSTEM UNDER TEST
    OrderProcessor sut = new();

    // WHEN
    OrderResult actualResult = sut.Process(givenOrder);

    // EXPECTATIONS
    OrderStatus expectedStatus = OrderStatus.Completed;

    // THEN
    Assert.Equal(expectedStatus, actualResult.Status);
}
```

🚫 **Bad Example:**
```csharp
[Fact]
public void Test_OrderProcessor_ReturnsCompletedStatus()
{
    OrderProcessor sut = new();
    OrderResult actualResult = sut.Process(Fixtures.Order());

    // THEN: literal assertion with no EXPECTATIONS section
    Assert.Equal(OrderStatus.Completed, actualResult.Status); // ❌ Expected value should be stored in expectedStatus
}
```

#### **5.3.3 LOGGING**
- Verify any behaviour that can be validated via captured logging operations.
- Prefer asserting on emitted logs over (or in addition to) mock verifications; log-based assertions are simpler to write and review and often replace complex mock setups.
- Each unit test should explicitly verify that the execution branch it is meant to cover is taken, reducing false positives when setup does not exercise the desired branch.

✅ **Good Example:**
```csharp
[Fact]
public void Test_Handler_EmitsInformationLog()
{
    // GIVEN
    Request givenRequest = Fixtures.Request();

    // MOCKING
    FakeLogger fakeLogger = new();

    // SETUP
    ILogger envLogger = fakeLogger;

    // SYSTEM UNDER TEST
    RequestHandler sut = new(envLogger);

    // WHEN
    sut.Handle(givenRequest);

    // LOGGING
    fakeLogger.AssertLogged(LogLevel.Information, "Handled request");
}
```

🚫 **Bad Example:**
```csharp
[Fact]
public void Test_Handler_EmitsInformationLog()
{
    FakeLogger fakeLogger = new();
    RequestHandler sut = new(fakeLogger);

    sut.Handle(Fixtures.Request());

    // LOGGING assertions are skipped entirely
    // ❌ Test passes even if no log was written
}
```

#### **5.3.4 BEHAVIOR**
- Contains assertions for **mock interactions** (e.g., `mockService.Verify(x => x.DoSomething(), Times.Once());`).
- This section **must be present if mocks are used**.
- **The BEHAVIOR section must not be merged with any other section.**
- **All mocks must be created as verifiable**—either using `new Mock<T>(MockBehavior.Strict)` or by calling `.Verifiable()` on each `.Setup(...)`.
- **Every verifiable setup must be explicitly verified** in this section via `mock.Verify(...)` or `mock.VerifyAll()`. Unverified mocks should cause the test to fail.

✅ **Good Example:**
```csharp
[Fact]
public void Test_ProcessData_PersistsRecord()
{
    // GIVEN
    Payload givenPayload = Fixtures.Payload();

    // MOCKING: capture the collaborator interaction
    Mock<IDataGateway> mockGateway = new(MockBehavior.Strict);
    mockGateway
        .Setup(gateway => gateway.Save(givenPayload))
        .Verifiable();

    // SETUP
    IDataGateway envGateway = mockGateway.Object;

    // SYSTEM UNDER TEST
    DataService sut = new(envGateway);

    // WHEN
    sut.Process(givenPayload);

    // BEHAVIOR
    mockGateway.Verify(gateway => gateway.Save(givenPayload), Times.Once);
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
- **Prefer named variables over inline literals**. Literal assertions are fine when the value is self-explanatory (`true`, `HttpStatusCode.NotFound`, etc.); otherwise, promote the value to an `expected*` variable so the assertion reads fluently.
- **Avoid asserting directly against `given*` variables.** Promote them to `expected*` variables so the THEN section reads like “expected vs. actual.”
- **Avoid asserting directly against `env*` collaborators.** Move any observed state into an `actual*` variable before verifying it.
- **Keep mock verification inside BEHAVIOR.** Use `expected*` values only when you need to compare data returned from a mock; interaction verification should rely on `mock.Verify(...)` calls instead.

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
```

🚫 **Bad Example:**
```csharp
[Test]
public void Test_DivideByZero_ThrowsException()
{
    // WHEN: exception is not captured and assertions are embedded in the delegate
    Assert.Throws<DivideByZeroException>(() => Assert.Equal(0, Math.Divide(10, 0))); // ❌ No WHEN/THEN separation and no actual* variable
}
```

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

✅ **Good Example:**
```csharp
[Fact]
public async Task Test_ProcessPayment_ReturnsDeclined_WhenBalanceIsLow()
{
    // GIVEN: customer balance is below the threshold
    Customer givenCustomer = Fixtures.CustomerWithBalance(2.00m);
    PaymentRequest givenRequest = Fixtures.PaymentRequest(amount: 5.00m);

    // MOCKING
    Mock<IBankGateway> mockGateway = Fixtures.StrictGateway();
    mockGateway.Setup(gateway => gateway.ChargeAsync(givenRequest)).ReturnsAsync(PaymentStatus.Declined).Verifiable();

    // SETUP
    IBankGateway envGateway = mockGateway.Object;

    // SYSTEM UNDER TEST
    PaymentProcessor sut = new(envGateway);

    // WHEN
    PaymentResult actualResult = await sut.ProcessAsync(givenCustomer, givenRequest);

    // EXPECTATIONS
    PaymentStatus expectedStatus = PaymentStatus.Declined;

    // THEN
    Assert.Equal(expectedStatus, actualResult.Status);

    // BEHAVIOR
    mockGateway.Verify();
}

[Fact]
public async Task Test_ProcessPayment_ReturnsApproved_WhenBalanceIsSufficient()
{
    // GIVEN: customer balance comfortably covers the request
    Customer givenCustomer = Fixtures.CustomerWithBalance(25.00m);
    PaymentRequest givenRequest = Fixtures.PaymentRequest(amount: 5.00m);

    // MOCKING
    Mock<IBankGateway> mockGateway = Fixtures.StrictGateway();
    mockGateway.Setup(gateway => gateway.ChargeAsync(givenRequest)).ReturnsAsync(PaymentStatus.Approved).Verifiable();

    // SETUP
    IBankGateway envGateway = mockGateway.Object;

    // SYSTEM UNDER TEST
    PaymentProcessor sut = new(envGateway);

    // WHEN
    PaymentResult actualResult = await sut.ProcessAsync(givenCustomer, givenRequest);

    // EXPECTATIONS
    PaymentStatus expectedStatus = PaymentStatus.Approved;

    // THEN
    Assert.Equal(expectedStatus, actualResult.Status);

    // BEHAVIOR
    mockGateway.Verify();
}
```

🚫 **Bad Example:**
```csharp
private async Task AssertPaymentAsync(decimal balance, PaymentStatus expectedStatus)
{
    // ❌ Helper hides GIVEN/MOCKING differences and forces readers to decipher parameters
    Customer customer = Fixtures.CustomerWithBalance(balance);
    PaymentRequest request = Fixtures.PaymentRequest(amount: 5.00m);

    Mock<IBankGateway> mockGateway = new();
    mockGateway.Setup(gateway => gateway.ChargeAsync(request)).ReturnsAsync(expectedStatus);

    PaymentProcessor sut = new(mockGateway.Object);

    PaymentResult result = await sut.ProcessAsync(customer, request);
    Assert.Equal(expectedStatus, result.Status);
}

[Fact]
public Task Test_ProcessPayment_ReturnsDeclined_WhenBalanceIsLow()
    => AssertPaymentAsync(2.00m, PaymentStatus.Declined);

[Fact]
public Task Test_ProcessPayment_ReturnsApproved_WhenBalanceIsSufficient()
    => AssertPaymentAsync(25.00m, PaymentStatus.Approved);
```

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

🚫 **Bad Example:**
```csharp
[Fact]
public void Test_CreateInvoice_UsesSystemClock()
{
    // GIVEN and WHEN are intertwined by calling DateTime.UtcNow directly
    InvoiceService sut = new(new RealClock());

    Invoice actualInvoice = sut.CreateInvoice(new Order(Guid.Empty));

    // THEN: the assertion depends on DateTime.UtcNow, introducing flakiness
    Assert.True(actualInvoice.GeneratedAt <= DateTimeOffset.UtcNow); // ❌ Relies on real time and cannot be deterministic
}
```

---

## **14. Using comments in tests**
Documenting test intent with XML comments provides high-level context that complements the structured sections, guiding readers through why a scenario exists. Without these explanations, future maintainers must infer purpose from mechanics alone, risking redundant cases or accidental deletion of critical coverage.

- **Each test method should be commented with a human-readable explanation of what the test is exercising**
- **Each test class should be commented with a human-readable explanation of what the test class is exercising**
- **Use the XML comment syntax to comment test classes and test methods**
- **Include the SUT, the triggering action (WHEN), and the expected behaviour or outcome (THEN/BEHAVIOR) in the XML summary for each test method** so readers can map the prose to the structured sections instantly.

When in doubt, follow the structure “Verifies that `<SUT>` `<behaviour>` when `<trigger>`.” This framing keeps the prose synchronized with the SYSTEM UNDER TEST, WHEN, and BEHAVIOR/THEN sections without duplicating their exact wording.

✅ **Good Example:**
```csharp
/// <summary>
/// Verifies scenarios for <see cref="MembershipRenewalService"/>.
/// </summary>
public sealed class MembershipRenewalServiceTests
{
    /// <summary>
    /// Verifies that <see cref="MembershipRenewalService.Renew(Membership)"/> extends an active membership by one year when the renewal operation is invoked.
    /// </summary>
    [Fact]
    public void Test_Renew_ExtendsExpirationByOneYear()
    {
        // GIVEN
        Membership givenMembership = Fixtures.ActiveMembership();

        // SYSTEM UNDER TEST
        MembershipRenewalService sut = new();

        // WHEN
        Membership actualResult = sut.Renew(givenMembership);

        // EXPECTATIONS
        DateTime expectedExpiration = givenMembership.ExpirationDate.AddYears(1);

        // THEN
        Assert.Equal(expectedExpiration, actualResult.ExpirationDate);
    }
}
```

🚫 **Bad Example:**
```csharp
public sealed class MembershipRenewalServiceTests
{
    [Fact]
    public void Test_Renew_ExtendsExpirationByOneYear()
    {
        // ❌ Missing XML comments, forcing readers to infer purpose
    }
}
```

## **15. Increasing test clarity with section comments**
Narrated section headers transform dense setup or verification code into self-explanatory stories that highlight intent and edge cases. Skipping these comments forces reviewers to reverse-engineer the reason behind each block, slowing code reviews and making it easier for subtle regressions to slip through.

**As a best practice**, when tests have complex parts such as setting up mocks, creating complex 'given' objects, asserting on mock behavior, and so forth, it is recommended to split each such part into a section of its own, and comment what that section is doing. These headers act like a decryption key for the reader: each full-sentence comment explicitly states the intent of the code beneath it, so that even when the implementation is dense or unfamiliar, the reader can immediately understand _why_ a block exists rather than mentally reverse-engineering the setup from the statements themselves.

Use the following patterns when writing section headers:

- **GIVEN** headers describe the inputs or outside forces that drive the SUT. Phrase them in terms of the scenario rather than the mechanics (e.g., `// GIVEN: a library that contains at least one book`).
- **MOCKING** headers should describe the state that the mock enforces in the underlying system and reference the related `given*` values (e.g., `// MOCKING: the givenUser exists in the user repository`).
- **SETUP** headers should mirror the narrative from MOCKING, describing the environment exposure and referencing the `given*` values being wired up (e.g., `// SETUP: expose the givenUser repository to the SUT`).
- **CAPTURE** headers explain what is being captured and from where (e.g., `// CAPTURE: the HTTP POST request body sent to the FooBar API`).
- **SYSTEM UNDER TEST** should almost never need a header—your test name and XML comments should already identify the SUT. Only add a comment when the constructor or factory call needs clarification.
- **WHEN** similarly rarely needs a header because the test name and XML comment should spell out the trigger. Only document it when the invocation is non-obvious.
- **EXPECTATIONS** may be split into multiple subsections with their own headers when the assertions are complex; otherwise omit the header and let the section title speak for itself.
- **THEN** should rarely include a header. When it does, treat it the same way as EXPECTATIONS and describe what outcome the assertions verify.
- **LOGGING** sections should not normally have headers. Prefer to include the log message strings directly in this section and assert on them there rather than moving them to EXPECTATIONS.
- **BEHAVIOR** should almost never need a header; the expected interaction should already be obvious from the test name and XML documentation.

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

✅ **Good Example:**
```csharp
[Fact]
public void Test_ReportService_UsesEnvDependencies()
{
    // MOCKING
    Mock<IReportRepository> mockRepository = new(MockBehavior.Strict);
    mockRepository.Setup(repo => repo.Load()).Returns(new Report()).Verifiable();

    // SETUP
    IReportRepository envRepository = mockRepository.Object;

    // SYSTEM UNDER TEST
    ReportService sut = new(envRepository);

    // WHEN
    sut.Generate();

    // BEHAVIOR
    mockRepository.Verify();
}
```

🚫 **Bad Example:**
```csharp
[Fact]
public void Test_ReportService_UsesEnvDependencies()
{
    Mock<IReportRepository> mockRepository = new();

    // SYSTEM UNDER TEST: dependency injected via mock.Object directly
    ReportService sut = new(mockRepository.Object); // ❌ Violates env* indirection rule

    sut.Generate();
}
```

---

## **17. Breaking the Rules**
The guidelines above are intentionally prescriptive so tests remain predictable and reviewable. However, real-world systems occasionally demand exceptions.
- Deviation is acceptable when it **improves clarity or determinism** for a specific edge case that the standard pattern cannot express cleanly.
- When you break a rule, **document the rationale inline** (e.g. with a comment or XML doc) so future maintainers understand why the exception exists.
- Re-evaluate exceptions periodically. If the surrounding production code changes, the original constraint may disappear and the standard structure can be restored.

That said, most shortcuts trade short-term brevity for long-term ambiguity. The patterns below are common places where authors intentionally bend the rules; use them sparingly, and only after considering whether the standard layout would actually be clearer.

- **Capturing values without `actual*` variables.** Directly asserting on `given*` variables in THEN or BEHAVIOR (e.g., `Assert.Equal("foo", givenResult.Value)` without storing `givenResult.Value` in an `actual*` variable) saves a line but conceals which data is the output under inspection. Prefer introducing an `actualResult` variable—even if it feels redundant—so the assertion reads like `Assert.Equal(expectedResult, actualResult);`.
- **Pulling capture logic forward.** Some authors slot a `WHEN` block ahead of MOCKING or SETUP just to declare `actual*` variables, or they capture directly into local variables alongside the mocks. The reordering works mechanically, yet it makes the test read out of sequence and obscures which variables are inputs versus observed outputs. Keep capture declarations in CAPTURED so the execution timeline remains obvious.
- **Combining capture and declaration.** Declaring a captured variable in the same MOCKING or SETUP block that guarantees it will be populated (for example, `ActualResponse actualResponse = default!; mockClient.Setup(...).Callback(response => actualResponse = response);`) can work, yet it blurs the boundary between scaffolding and observation. The dedicated CAPTURED section makes it obvious where side effects land, and removing it forces reviewers to scan the entire setup to find the observation point.
- **Stashing captured variables in WHEN.** Some developers capture values inside the WHEN section (e.g., `// WHEN: the handler runs` followed by assignments to `actual*` variables). While legal, it hides the distinction between invoking the SUT and storing its outputs. Prefer keeping WHEN exclusively for exercising the behaviour and move all captured assignments into CAPTURED or SYSTEM UNDER TEST.
- **Skipping fine-grained `actual*` variables.** After invoking the SUT, it can feel natural to assert directly on nested properties (e.g., `Assert.Equal(expectedFoo, actualResult.Foo.Bar);`). Extracting those sub-components into their own `actualFooBar` variables keeps THEN concise and highlights which part of the response each assertion cares about. It also reduces the risk of asserting against the wrong member after refactors.
- **Skipping the EXPECTATIONS section.** It is tempting to assert directly on `given*` values or inline expected literals when the expected data mirrors the inputs. Doing so erases the narrative that EXPECTATIONS provides. Even if EXPECTATIONS simply clones the GIVEN values into `expected*` variables, that duplication reinforces the Arrange → Assert story and pays dividends when expectations diverge in future edits.
- **Skipping comments for trivial sections.** Omitting the `// GIVEN:` style comments when a section is a single obvious line keeps the file compact, but the uniform comment headings make scanning large files easy. If you omit a comment, ensure the surrounding code is truly self-documenting and add a note explaining the exception.
- **Skipping `env*` indirection for mocks.** Passing `mockFoo.Object` straight into the SUT is faster than assigning it to `IFoo envFoo = mockFoo.Object;`, yet it couples the test to the mocking framework’s API and obscures which collaborator is being injected. Use the indirection whenever possible so the SYSTEM UNDER TEST section reads like a constructor call with domain types instead of mock plumbing.

Humans occasionally break these rules to meet deadlines or to avoid ceremony in quick experiments. **AI tools should not**. Automated authors can generate well-structured tests with negligible effort, so they are expected to follow the full format unless a human reviewer explicitly approves an exception.

✅ **Good Example:**
```csharp
[Fact]
public void Test_StreamProcessor_AllowsInlineAssertion_ForDocumentedReason()
{
    // GIVEN: processing an empty stream should exit early
    Stream givenStream = Stream.Null;

    // SYSTEM UNDER TEST
    StreamProcessor sut = new();

    // WHEN
    bool actualResult = sut.Process(givenStream);

    // THEN
    Assert.False(actualResult);
}

// NOTE: Breaking the EXPECTATIONS rule because the legacy API already returns bool and
//       the team explicitly approved inline assertions for this hotfix scenario.
```

🚫 **Bad Example:**
```csharp
[Fact]
public void Test_StreamProcessor_AllowsInlineAssertion_ForDocumentedReason()
{
    StreamProcessor sut = new();

    // WHEN/THEN: assertion inlined with no explanation
    Assert.False(sut.Process(Stream.Null)); // ❌ Rule broken with no documentation
}
```

