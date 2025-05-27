# C# Unit Testing Style Guide

This document defines **best practices** for writing **clear, structured, and maintainable** unit tests in C# using xUnit, NUnit, or MSTest.

---

## **Target Audience**

The primary target audience for this document is developers and Large Language Models (LLMs) acting as coding assistants. By following these guidelines, tests will be **structured, readable, and maintainable**.

These practices may be too detailed for human-written tests but serve as an excellent guide for generating automated tests with LLM assistance. **Review all test functions carefully**, as human oversight is crucial to ensure test quality.

---

## **1. Test Class Naming**
- Each test class **must end with** `Tests`.
- If a test class exercises a whole class and all of its methods, it must be named `<ClassName>Tests`.
- If a test class exercises a single region within a class, it must be named `<ClassName><RegionName>Tests`.
- If a test class exercises a single method within a class, it must be named `<ClassName><MethodName>Tests` or `<ClassName><RegionName><MethodName>Tests`.
- Test class names should not contain underscores.

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
public class FileHandlerInitializationAndHashingTests() // ❌ Must not combine multiple #regions into the same test class (at least not like this)
```

---

## **2. Test Method Naming**
- Each test method **must start with** `Test_`.
- Use underscores to separate major phrases in the test method name.
- Test method names should note contain consequtive underscores.
- If multiple class methods are being tested in the same file, the test method **must start with** `Test_<MethodName>_`.
- If a single method is being tested in the file, the method name **must be omitted**.
- The method name should be **descriptive and readable**, reflecting the behavior under test.
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

## **3. Test Class Organization**
- **Each test class should cover a single class, region, or method under test**.
- **When testing a complex class, each region or method should have its own test class**.
- This ensures that tests are **modular, easy to locate, and maintainable**.
- **A test class' file name must be of the form `<ClassName>[<RegionName>][<MethodName>]Tests.cs`**.
- Any component found in the file name **must be duplicated in the test class name**.

✅ **Good Example:**
```
Tests/
│── AuthenticationTests.cs    // Covers `Authenticate()`
│── RequestProcessingTests.cs // Covers `ProcessRequest()`
│── ServiceTests.cs           // Covers multiple service methods separately
```
🚫 **Bad Example:**
```
Tests/
│── AllServicesTests.cs  // ❌ Mixed tests in one file
```

---

## **4. Test Structure**
Each test method follows a structured format, **separated by clear comments**:
- Where possible, the order of sections should be as it is presented below.
- Empty sections should be omitted.
- In some cases, it may be justifiable to have multiple instances of specific sections
- When a single **MOCKING** or **SETUP** block becomes large or involves multiple distinct steps, split it into smaller sub-sections with their own human-readable comment headers (e.g. `// SETUP: Configure HTTP client factory` and `// SETUP: Initialize environment variables`), so readers can quickly grasp each part.
- Section comment headers should be written as full descriptive sentences (e.g. `// GIVEN: A valid context and wash code details`), not terse labels. If a section would contain only a trivial one-line line of code without setup complexity, you may omit its own comment.

### **Standard Sections:**

The following sections are defined for a well structured test.

1. **GIVEN**
   - Set up initial conditions and inputs.
   - Assign variables with the prefix `given`.
   - when test fixtures are used, they should be allocated in this section
   - **Test Fixture classes** should be named `Fixture*` and assigned to `given*` variables prior to use.
   - The GIVEN section should always be the first section in a test, unless there is nothing to define in GIVEN, in which case it should be omitted.
   - **The GIVEN section must not be merged with any other section**

1. **MOCKING**
   - Define and configure **mock objects**.
   - Mock variables must be named `mock*`.
   - **Use constructor injection for dependencies**, or use mocking frameworks like Moq.
   - The MOCKING section should occur before the SETUP section, or be omitted if no mocking is required. Under special circumstances described above, it may be placed after the SETUP section.
   - **The MOCKING section must not be merged with any other section**
   - **If mocking logic itself has multiple responsibilities (e.g. creating fakes and configuring behaviors), split into sub-headers** (e.g. `// MOCKING: Create mock objects` and `// MOCKING: Configure mock responses`).

1. **SETUP**
   - perform all test initialization that prepares the non-mocked environment (e.g., creating HttpContext, configuring dependency injection, and setting up request parameters)
   - variables created in this section should be prefixed with `env*` (e.g. envHttpContext, envActionContext)
   - when constants or variables are needed in this section that are important for the overall comprehensibility of the test, they should be declared in variables in the GIVEN section instead
   - the SETUP section should follow the GIVEN section, unless there is nothing to setup in SETUP, in which case it should be omitted
   - the SETUP section should refer to GIVEN variables, not `mock*` variables, except in rare occasions
   - Assign mock objects to real interface variables in the SETUP section, using the env* prefix (e.g., envLogger = mockLogger.Object). The mock variables themselves may only be used in the SETUP and BEHAVIOR sections.
   - **The SETUP section must not be merged with any other section**
   - **If SETUP involves multiple distinct stages (e.g. reading fixtures, wiring dependencies, configuring environment), split into multiple sub-sections with descriptive comments.**

1. **SYSTEM UNDER TEST**
   - Assigns the system under test to a variable named `sut`
   - if multiple systems are being tested for some reason, assign them to separate`sut*` variables.
   - The SYSTEM UNDER TEST must never directly reference mock* variables. It must use the env* variables initialized in SETUP.
   - **The SYSTEM UNDER TEST section must not be merged with any other section**

1. **WHEN**
   - Perform the action or method call being tested.
   - Assign the results to `actual*` variables. This would typically be the return value received from invoking the system under test.
     - If the test requires asserting against other results in the THEN section or BEHAVIOR section, then use the opportunity
       to assign those results to their own `actual*` variables, e.g. after invoking the system under test.
   - If the test needs to use an `env*` in later sections, then assign them to `actual*` variables in WHEN and use those instead.
   - **The WHEN section must not be merged with any other section**

1. **EXPECTATIONS**
   - Define expected values **before assertions**.
   - Expected values must be assigned to `expected*` variables.
   - The EXPECTATIONS section should be strictly placed after WHEN and before THEN
   - This section should not refer to any `actual*` variables
   - **The EXPECTATIONS section must not be merged with any other section**

1. **THEN**
   - Perform **assertions** comparing actual results to expected values.
   - **Never assert against literals**—always use `expected*` variables.
   - **The THEN section must not be merged with any other section**

1. **LOGGING**
   - Verify any behavior that can be validated via captured logging operations.
   - Log the behavior of the code under test at appropriate levels (including trace-level logging with extensive details).
   - In unit tests, assert against log entries in the LOGGING section to confirm that the intended execution branch was exercised.
   - **Prefer asserting on emitted logs over (or in addition to) mock verifications** whenever possible: log-based assertions are simpler to write and review, and often replace complex mock setups (e.g., capturing parameters).
   - Each unit test should explicitly verify that the execution branch it is meant to cover is taken, reducing false positives when setup does not exercise the desired branch.


1. **BEHAVIOR**
   - Contains assertions for **mock interactions** (e.g., `mockService.Verify(x => x.DoSomething(), Times.Once());`).
   - This section **must be present if mocks are used**.
   - **The BEHAVIOR section must not be merged with any other section**
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

## **5. Fixtures: Best Practices**
- **Prefer fixtures over mocks**- preferring to use shared, reusable fixtures rather than creating one-off test data. Fixtures help keep your tests DRY and consistent, and allows them to work on more accurate test data.
- **Instantiate fixtures in the appropriate section** e.g. GIVEN or SETUP, and modify it as needed to fit the test's needs.
- **Use a sharable fixture factory** instead of creating fixtures or complex test data inline

✅ **Good Example:**
```csharp

public static class Fixtures {

    public static string IpAddress()
    {
        return "192.168.1.1";
    }

    public static CwConfig CwConfig()
    {
        return new CwConfig { Terminals = new List<HardwareTerminalConfig>() };
    }

    public static CwGetTerminalConfigCommand CwGetTerminalConfigCommand()
    {
        string givenIpAddress = IpAddress();
        return new CwGetTerminalConfigCommand { TerminalFixedIpAddress = givenIpAddress };
    }

    public static CommandContext<CwGetTerminalConfigCommand, CwGetTerminalConfigResult> CommandContext()
    {
        return new CommandContext<CwGetTerminalConfigCommand, CwGetTerminalConfigResult>
        {
            Command = CwGetTerminalConfigCommand()
        };
    }
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

---

## **6. Mocking: Best Practices**
- **Never mock what you don't have to**—prefer fixtures or real instances where practical.
- **Only mock things that the system-under-test uses directly**-this ensures that your test exercises the SUT properly, without falling into the trap of combinatorial execution path counts as call-depth increases.
- The systems that your SUT uses directly should be covered by their own direct unit tests, not by the unit tests of your SUT.
- **Use Moq or built-in mocking frameworks** for dependency injection.
- **Assertions on mock interactions go in the BEHAVIOR section, not in the THEN section**.
- **Mocks must never be passed directly to the system under test.** Instead, assign mocks to interface-conforming env* variables in SETUP and pass those to the SUT.
- **Prefer verifying log output in addition to mock verifications**—capturing and asserting on log entries is usually easier to implement correctly, more readable for future maintainers, and often avoids the “gnarly” setup required to capture mock parameters.
- **All mocks must be created in verifiable mode**:
  - Use `new Mock<T>(MockBehavior.Strict)` to catch any unexpected calls, _or_
  - Call `.Verifiable()` on each `mock.Setup(...)` you intend to verify.
- **Every mock setup must be verified in the BEHAVIOR section** via `mock.Verify(...)` or `mock.VerifyAll()`; unverified verifiable mocks should fail the test.

✅ **Good Example:**
```csharp
[Test]
public void Test_ServiceCallsDependency()
{
    // GIVEN
    var mockDependency = new Mock<IDependency>();

    // MOCKING
    IDependency envDependency = mockDependency.Object;

    // SYSTEM UNDER TEST
    var givenService = new MyService(envDependency);

    // WHEN
    givenService.DoWork();

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

---

## **7. Assertions & Variable Naming**
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

## **8. Exception Handling (`Assert.Throws`)**

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
}

---

## **9. Using `[SetUp]` Methods**
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

## **10. Organizing Tests in Namespaces**
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

## **11. Using comments in tests**

- **Each test method should be commented with a human-readable explanation of what the test is exercising**
- **Each test class should be commented with a human-readable explanation of what the test class is exercising**
- **Use the XML comment syntax to comment test classes and test methods**

## **12. Increasing test clarity with section comments**

**As a best practice**, when tests have complex parts such as setting up mocks, creating complex 'given' objects, asserting on mock behavior, and so forth, it is recommended to split each such part into a section of its own, and comment what that section is doing

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

## **12. SETUP / SUT Dependency Rule **

To re-iterate:

- The **SETUP** section must initialize all dependencies passed to the **SUT** via `env*` variables.
- The **SYSTEM UNDER TEST** section must construct the **SUT** using only `env*` variables.
- Mocks must never be used directly in the **SYSTEM UNDER TEST** section.

