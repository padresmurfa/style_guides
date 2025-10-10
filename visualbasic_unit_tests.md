# Visual Basic Unit Testing Style Guide

This document defines **best practices** for writing **clear, structured, and maintainable** unit tests in Visual Basic (VB.NET) using MSTest, NUnit, or xUnit.

---

## **Target Audience**

The primary target audience for this document is developers and Large Language Models (LLMs) acting as coding assistants. By following these guidelines, tests will be **structured, readable, and maintainable**.

These practices may be too detailed for human-written tests but serve as an excellent guide for generating automated tests with LLM assistance. **Review all test procedures carefully**, as human oversight is crucial to ensure test quality.

---

## **1. Test Namespace Organization**
Consistent namespaces make it immediately obvious where a test lives relative to the production code, so reviewers can locate the subject under test without opening multiple files. Misaligned namespaces cause confusion when navigating between source and tests, especially in IDEs that rely on namespace matching for discovery.
- Every test file must declare a namespace that mirrors the production namespace and appends a `.Tests` suffix (e.g. `MyCompany.Products.Services` → `MyCompany.Products.Services.Tests`).
- When the production code uses nested namespaces, mirror that structure exactly in the test namespace before adding `.Tests`.
- Keep a **one-to-one mapping between folders and namespaces** so moving a file preserves the namespace.
- Avoid combining test classes for different production namespaces under a single namespace—create separate namespaces instead.
- Always use the block form in VB: `Namespace ... End Namespace`. The namespace line should not be indented; indent the contents inside the namespace.

✅ **Good Example:**
```vb
Namespace MyCompany.Products.Services.Tests
End Namespace
```
🚫 **Bad Example:**
```vb
Namespace Tests ' ❌ Does not mirror the production namespace
End Namespace
```

---

## **2. Test Class Naming and Files**
Consistent test class names make it immediately obvious which production code a suite exercises, so reviewers can trace failures quickly and spot coverage gaps. Without a naming convention teams waste time hunting for relevant tests, duplicate effort, and risk overlooking scenarios because responsibilities are spread across ambiguously labeled files.
- Each test file must contain a single public test class whose name **ends with** `Tests`.
- If a test class exercises a whole class and all of its members, it must be named `<ClassName>Tests`.
- If a test class exercises a single region within a class, it must be named `<ClassName><RegionName>Tests`.
- If a test class exercises a single member within a class, it must be named `<ClassName><MemberName>Tests` or `<ClassName><RegionName><MemberName>Tests`.
- Region-focused and member-focused test classes are **optional** patterns. Prefer them when they make intent clearer, but feel free to keep everything in a single `<ClassName>Tests` class when the class-under-test is small enough that splitting would add overhead without clarity.
- Test class names should not contain underscores.
- **Each test class should cover a single class, region, or member under test.**
- **A test class' file name must be of the form `<ClassName>[<RegionName>][<MemberName>]Tests.vb`**, and every component of the file name must appear in the class name exactly once and in the same order.
- Organize test files into folders that mirror the namespace to keep discovery tools and human navigation aligned.

✅ **Good Examples:**
```vb
Public Class FileHandlerTests
End Class

Public Class FileHandlerHashingTests
End Class

Public Class FileHandlerOpenTests
End Class
```
🚫 **Bad Examples:**
```vb
Public Class FileHandler ' ❌ Missing Tests suffix
End Class

Public Class FileHandlerWorksAsIntendedTests ' ❌ WorksAsIntended is not a plausible #Region name
End Class

Public Class FileHandlerInitializationAndHashingTests ' ❌ Combines multiple #regions into one class
End Class
```

---

## **3. Region-Focused Test Classes**
Regions describe cohesive areas of responsibility inside a production class. Mirroring those regions in the test suite keeps the feedback loop tight by ensuring every region has an intentionally scoped test class. Treat these region-focused classes as a tool rather than a mandate—when a production class only has a handful of regions or members, duplicating files for each can be overkill.
- Only introduce a `<RegionName>` segment when the production code exposes a `#Region` with the same PascalCase name.
- When a class contains multiple regions, create a separate test class per region, following the naming rules above and keeping each class in its own file.
- Do not reuse a `<RegionName>` segment across unrelated production regions; each region must map to a single test class.
- If a region contains helper members that are not directly invoked by the public API, test the public surface that exercises them rather than testing the region helpers directly.

✅ **Good Example:**
```
Services/
│── FileHandler.vb          ' Contains regions Hashing and Open
Tests/
│── FileHandlerHashingTests.vb
│── FileHandlerOpenTests.vb
```
🚫 **Bad Example:**
```
Tests/
│── FileHandlerHashingAndOpenTests.vb ' ❌ Merges multiple regions into a single test class
```

---

## **4. Test Method Naming**
Clear method names document the behavior under test and the expected outcome, which helps maintainers understand intent without rereading the implementation. Vague or inconsistent names hide gaps in coverage, encourage multi-purpose tests, and make regression triage harder when a failing test name does not reveal what broke.
- Each test method **must start with** `Test_`.
- Use underscores to separate major phrases in the test method name.
- Although test classes stay in PascalCase with no underscores, **test methods are encouraged to use underscores** so long names remain readable.
- Test method names should **not contain consecutive underscores**.
- If multiple production members are being tested within the same test class, the test method **must start with** `Test_<MemberName>_`.
- If a single production member is being tested in the file, the production member name **must be omitted** from the test method.
- The method name should be **descriptive and readable**, reflecting the behavior under test.
- Favor verbosity over brevity—short names rarely communicate enough context to make the test self-documenting.
- Test method names must describe the **specific branch of execution** they cover (e.g. `Test_Foo_ReturnsBar_WhenSuccessful`), not merely state that something "works".
- **Separate tests** into distinct methods rather than combining multiple test cases.
- **Avoid parameterized tests** unless they significantly improve clarity.
- Any component found in the test class name **must NOT be duplicated in the test method name**.

✅ **Good Example:**
```vb
<Test>
Public Sub Test_ProcessData_ReturnsCorrectValue()
End Sub
```
🚫 **Bad Examples:**
```vb
<Test>
Public Sub CheckProcessData() ' ❌ Missing Test_ prefix
End Sub

<Test>
Public Sub Test_Process() ' ❌ Too vague
End Sub

<Test>
Public Sub Test_MultipleCases() ' ❌ Tests multiple things at once
End Sub
```

---

## **5. Test Method Sectioning**
Standardized sections carve complex tests into digestible steps, making it easier to see how inputs flow through mocks and the system under test. Without this structure tests devolve into monolithic blocks where intent, setup, and verification intermingle, obscuring bugs and encouraging brittle copy-paste patterns. The ordering rules below are intentionally strong to build reliable habits—refer to [**Section 17. Breaking the Rules**](#-17-breaking-the-rules) for guidance on the rare cases where deviating is justified.
Each test method follows a structured format, **separated by clear comments**:
- Where possible, the order of sections should be as it is presented below.
- Empty sections should be omitted.
- In some cases, it may be justifiable to have multiple instances of specific sections
- When a single **MOCKING** or **SETUP** block becomes large or involves multiple distinct steps, split it into smaller sub-sections with their own human-readable comment headers (e.g. `' SETUP: Configure HTTP client factory`) so readers can quickly grasp each part.
- Section comment headers should be written as full descriptive sentences (e.g. `' GIVEN: A valid context and wash code details`), not terse labels. If a section would contain only a trivial line of code without setup complexity, you may omit its own comment.

### **Standard Sections:**

The following sections are defined for a well structured test.

1. **GIVEN**
   - Set up initial conditions and inputs.
   - Assign variables with the prefix `given`.
   - When test fixtures are used, they should be allocated in this section.
   - **Test Fixture classes** should be named `Fixture*` and assigned to `given*` variables prior to use.
   - The GIVEN section should always be the first section in a test, unless there is nothing to define in GIVEN, in which case it should be omitted.
   - **The GIVEN section must not be merged with any other section**

1. **MOCKING**
   - Define and configure **mock objects**.
   - Mock variables must be named `mock*`.
   - **Use constructor injection for dependencies**, or use mocking frameworks like Moq or NSubstitute.
   - The MOCKING section should occur before the SETUP section, or be omitted if no mocking is required. Under special circumstances described above, it may be placed after the SETUP section.
   - **The MOCKING section must not be merged with any other section**
   - **If mocking logic itself has multiple responsibilities (e.g. creating fakes and configuring behaviors), split into sub-headers** (e.g. `' MOCKING: Create mock objects`).

1. **SETUP**
   - Perform all test initialization that prepares the non-mocked environment (e.g., creating HttpContext, configuring dependency injection, and setting up request parameters).
   - Variables created in this section should be prefixed with `env` (e.g. `envHttpContext`, `envActionContext`).
   - When constants or variables are needed in this section that are important for the overall comprehensibility of the test, they should be declared in variables in the GIVEN section instead.
   - The SETUP section should follow the GIVEN section, unless there is nothing to setup in SETUP, in which case it should be omitted.
   - The SETUP section should refer to GIVEN variables, not `mock*` variables, except in rare occasions.
   - Assign mock objects to real interface variables in the SETUP section, using the `env*` prefix (e.g., `envLogger = mockLogger.Object`). The mock variables themselves may only be used in the SETUP and BEHAVIOR sections.
   - **The SETUP section must not be merged with any other section**
   - **If SETUP involves multiple distinct stages (e.g. reading fixtures, wiring dependencies, configuring environment), split into multiple sub-sections with descriptive comments.**

1. **SYSTEM UNDER TEST**
   - Assign the system under test to a variable named `sut`.
   - If multiple systems are being tested for some reason, assign them to separate `sut*` variables.
   - The SYSTEM UNDER TEST must never directly reference `mock*` variables. It must use the `env*` variables initialized in SETUP.
   - **The SYSTEM UNDER TEST section must not be merged with any other section**

1. **WHEN**
   - Perform the action or member call being tested.
   - Assign the results to `actual*` variables. This would typically be the return value received from invoking the system under test.
     - If the test requires asserting against other results in the THEN section or BEHAVIOR section, then use the opportunity to assign those results to their own `actual*` variables after invoking the system under test.
   - If the test needs to use an `env*` in later sections, then assign them to `actual*` variables in WHEN and use those instead.
   - **The WHEN section must not be merged with any other section**

1. **EXPECTATIONS**
   - Define expected values **before assertions**.
   - Expected values must be assigned to `expected*` variables.
   - The EXPECTATIONS section should be strictly placed after WHEN and before THEN.
   - This section should not refer to any `actual*` variables.
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
   - Contains assertions for **mock interactions** (e.g., `mockService.Verify(Function(x) x.DoSomething(), Times.Once())`).
   - This section **must be present if mocks are used**.
   - **The BEHAVIOR section must not be merged with any other section**
   - **All mocks must be created as verifiable**—either using `New Mock(Of T)(MockBehavior.Strict)` or by calling `.Verifiable()` on each `.Setup(...)`.
   - **Every verifiable setup must be explicitly verified** in this section via `mock.Verify(...)` or `mock.VerifyAll()`. Unverified mocks should cause the test to fail.

✅ **Good Example:**
```vb
<Test>
Public Sub Test_ProcessData_ReturnsCorrectValue()
    ' GIVEN
    Dim givenInput As String = "sample"

    ' SYSTEM UNDER TEST
    Dim sut As New DataService()

    ' WHEN
    Dim actualResult As String = sut.ProcessData(givenInput)
    Dim actualSideEffect As String = Foo.Bar()

    ' EXPECTATIONS
    Dim expectedResult As String = "processed_sample"
    Dim expectedSideEffect As String = "fubar"

    ' THEN
    Assert.AreEqual(expectedResult, actualResult)
    Assert.AreEqual(expectedSideEffect, actualSideEffect)
End Sub
```

🚫 **Bad Example:**
```vb
<Test>
Public Sub Test_ProcessData_ReturnsCorrectValue()
    ' MOCKING
    Dim mockFoo As New Mock(Of IFoo)()

    ' SYSTEM UNDER TEST
    Dim sut As New DataService(mockFoo.Object)
End Sub
```

---

## **6. Fixtures: Best Practices**
Reusable fixtures encourage realistic, DRY data that mirrors production scenarios while keeping tests focused on behavior instead of data plumbing. Inlining bespoke objects everywhere invites duplication, increases maintenance costs when models evolve, and makes it harder to reason about how changes cascade across tests.
- **Prefer fixtures over mocks**—favor shared, reusable fixtures rather than creating one-off test data. Fixtures help keep your tests DRY and consistent, and allow them to work on more accurate test data.
- **Instantiate fixtures in the appropriate section** e.g. GIVEN or SETUP, and modify it as needed to fit the test's needs.
- **Use a sharable fixture factory** instead of creating fixtures or complex test data inline.

✅ **Good Example:**
```vb
Public NotInheritable Class Fixtures
    Private Sub New()
    End Sub

    Public Shared Function IpAddress() As String
        Return "192.168.1.1"
    End Function

    Public Shared Function CwConfig() As CwConfig
        Return New CwConfig With {.Terminals = New List(Of HardwareTerminalConfig)()}
    End Function

    Public Shared Function CwGetTerminalConfigCommand() As CwGetTerminalConfigCommand
        Dim givenIpAddress As String = IpAddress()
        Return New CwGetTerminalConfigCommand With {.TerminalFixedIpAddress = givenIpAddress}
    End Function

    Public Shared Function CommandContext() As CommandContext(Of CwGetTerminalConfigCommand, CwGetTerminalConfigResult)
        Return New CommandContext(Of CwGetTerminalConfigCommand, CwGetTerminalConfigResult) With {
            .Command = CwGetTerminalConfigCommand()
        }
    End Function
End Class

<TestFixture>
Public Class CarWashesServiceTests
    <Test>
    Public Async Function Test_GetTerminalConfig_ReturnsNotFound_WhenTerminalDoesNotExist() As Task
        ' GIVEN
        Dim givenTerminalIp As String = Fixtures.IpAddress()
        Dim givenCommandContext = Fixtures.CommandContext()
        Dim givenCarWashesConfig = Fixtures.CwConfig()

        ' SYSTEM UNDER TEST
        Dim sut As New CarWashesService()

        ' WHEN
        Await sut.GetTerminalConfig(givenCommandContext)
        Dim actualStatusCode As HttpStatusCode = givenCommandContext.StatusCode
        Dim actualResult As CwGetTerminalConfigResult = givenCommandContext.Result

        ' EXPECTATIONS
        Dim expectedStatusCode As HttpStatusCode = HttpStatusCode.NotFound
        Dim expectedResult As CwGetTerminalConfigResult = Nothing

        ' THEN
        Assert.AreEqual(expectedStatusCode, actualStatusCode)
        Assert.AreEqual(expectedResult, actualResult)
    End Function
End Class
```

🚫 **Bad Example:**
```vb
<Test>
Public Async Function Test_GetTerminalConfig_ReturnsNotFound_WhenTerminalDoesNotExist() As Task
    ' GIVEN
    Dim givenTerminalIp As String = "192.168.1.1"
    Dim givenCommandContext = New CommandContext(Of CwGetTerminalConfigCommand, CwGetTerminalConfigResult) With {
        .Command = New CwGetTerminalConfigCommand With {.TerminalFixedIpAddress = givenTerminalIp}
    }
    Dim givenCarWashesConfig = New CwConfig With {.Terminals = New List(Of HardwareTerminalConfig)()}
    StaticConfig.CarWashesConfig = givenCarWashesConfig

    ' SYSTEM UNDER TEST
    Dim sut As New CarWashesService()

    ' WHEN
    Await sut.GetTerminalConfig(givenCommandContext)
    Dim actualStatusCode As HttpStatusCode = givenCommandContext.StatusCode
    Dim actualResult As CwGetTerminalConfigResult = givenCommandContext.Result

    ' EXPECTATIONS
    Dim expectedStatusCode As HttpStatusCode = HttpStatusCode.NotFound
    Dim expectedResult As CwGetTerminalConfigResult = Nothing

    ' THEN
    Assert.AreEqual(expectedStatusCode, actualStatusCode)
    Assert.AreEqual(expectedResult, actualResult)
End Function
```

---

## **7. Mocking: Best Practices**
Disciplined mocking keeps tests readable and trustworthy by ensuring only true collaborators are simulated and their expectations are explicit. Unrestrained mocks create brittle, implementation-driven tests that break on harmless refactors, mask missing coverage of dependencies, and clutter SUT construction with framework boilerplate.
- **Never mock what you don't have to**—prefer fixtures or real instances where practical.
- **Only mock things that the system-under-test uses directly**—this ensures that your test exercises the SUT properly, without falling into the trap of combinatorial execution path counts as call-depth increases.
- The systems that your SUT uses directly should be covered by their own direct unit tests, not by the unit tests of your SUT.
- **Use Moq, NSubstitute, or other .NET-friendly mocking frameworks** for dependency injection.
- **Assertions on mock interactions go in the BEHAVIOR section, not in the THEN section**.
- **Mocks must never be passed directly to the system under test.** Instead, assign mocks to interface-conforming `env*` variables in SETUP and pass those to the SUT.
- Following this pattern keeps the SUT construction readable: the MOCKING section owns the intricate setup, the SETUP section exposes the ready-to-use interfaces via `env*` variables, and the SYSTEM UNDER TEST section reads like a clean recipe that highlights only the essential collaborators. Without the intermediate `env*` assignments the constructor or member invocation under test quickly devolves into a wall of `.Object` accessors and configuration noise, making it difficult for reviewers to decipher which dependency is which.

### **7.1 Test Fakes (a.k.a. Dummies)**
Test fakes are lightweight, purpose-built implementations of an interface that you author specifically for the test suite. They shine when mocking frameworks become gnarly—especially for complex interaction verification or structured log inspection—because you can write direct, intention-revealing code without wrestling with callback signatures or argument matchers.

- **Create fakes when setup or verification with a mock would be noisy, repetitive, or brittle.** If verifying interactions through a mocking framework requires delegates, reflection, or deeply nested `It.Is` expressions, a fake is often clearer.
- **Name fake instances with the `fake*` prefix and construct them in the MOCKING section.** This keeps parity with mock naming so readers instantly recognize simulated collaborators.
- **Assign each `fake*` variable to an `env*` variable in the SETUP section before handing it to the SUT.** Treat a fake like any other dependency so the SYSTEM UNDER TEST section stays clean and only references `env*` collaborators.
- **Interact with the fake via its `fake*` variable in the THEN, LOGGING, and BEHAVIOR sections.** Fakes frequently expose custom assertion helpers (e.g., `fakeLogger.AssertExercised(...)`) that capture behavior without `mock.Verify(...)` boilerplate.
- **Document expectations inside the fake when possible.** Purpose-built helpers (such as storing structured log entries) make intent obvious and reduce duplicated parsing logic across tests.

#### Strengths Compared to Mocks
- **Readable behavior verification.** Fakes encapsulate the verification logic in methods that read like English, avoiding the visual clutter of `Times.Once()` and `It.IsAny(Of T)()` chains.
- **Reduced setup friction.** Because you own the implementation, you can build constructors and helper methods that mirror your domain vocabulary, instead of contorting the SUT to satisfy framework APIs.
- **Deterministic assertions.** Fakes can capture state (e.g., recorded log entries) and expose first-class assertions, lowering the risk of brittle, order-dependent mock verifications.

#### Weaknesses Compared to Mocks
- **Maintenance cost.** You must maintain the fake implementation alongside production interfaces. If the interface evolves, the fake must be updated manually.
- **Limited behavioral coverage.** A fake typically encodes only the paths needed by the current test suite. Mocks can dynamically configure behaviors per test without editing shared code.
- **Risk of drifting from reality.** Because a fake is handwritten, it might omit subtle behavior the real dependency exhibits. Use fixtures and integration tests to guard against divergence.

#### Example: Logging with a Fake vs. a Mock
Consider a service that logs a structured trace message—`"Foo(branch=bar): the foo is quite barred"`—whenever it processes a branch.

**✅ Fake-based test (clean and intention revealing):**
```vb
<Fact>
Public Sub Test_ProcessBranch_LogsTraceMessage()
    ' GIVEN: a branch identifier that should trigger logging
    Dim givenBranchId As String = "bar"

    ' MOCKING: create test fakes for collaborators
    Dim fakeLogger As New FakeLogger()

    ' SETUP: expose the fake through env* variables
    Dim envLogger As ILogger = fakeLogger

    ' SYSTEM UNDER TEST: inject the fake logger
    Dim sut As New BranchProcessor(envLogger)

    ' WHEN: process the branch
    sut.ProcessBranch(givenBranchId)

    ' LOGGING: ensure the trace message was written
    fakeLogger.AssertExercised("Foo", "bar")
End Sub

Private NotInheritable Class FakeLogger
    Implements ILogger

    Private ReadOnly _entries As New List(Of LogEntry)()

    Public Function BeginScope(Of TState)(state As TState) As IDisposable Implements ILogger.BeginScope
        Return NullScope.Instance
    End Function

    Public Function IsEnabled(logLevel As LogLevel) As Boolean Implements ILogger.IsEnabled
        Return logLevel <= LogLevel.Trace
    End Function

    Public Sub Log(Of TState)(logLevel As LogLevel, eventId As EventId, state As TState, exception As Exception, formatter As Func(Of TState, Exception, String)) Implements ILogger.Log
        If logLevel = LogLevel.Trace AndAlso state IsNot Nothing Then
            Dim message As String = formatter(state, exception)
            Dim branchValue As String = ExtractBranchFromMessage(message)

            _entries.Add(New LogEntry With {
                .Category = "Foo",
                .Branch = branchValue,
                .Message = message
            })
        End If
    End Sub

    Public Sub AssertExercised(expectedCategory As String, expectedBranch As String)
        Assert.AreEqual(1, _entries.Count)
        Dim entry As LogEntry = _entries(0)

        Assert.AreEqual(expectedCategory, entry.Category)
        Assert.AreEqual(expectedBranch, entry.Branch)

        Dim expectedMessage As String = $"Foo(branch={expectedBranch}): the foo is quite barred"
        Assert.AreEqual(expectedMessage, entry.Message)
    End Sub

    Private Shared Function ExtractBranchFromMessage(message As String) As String
        Dim prefix As String = "branch="
        Dim prefixIndex As Integer = message.IndexOf(prefix, StringComparison.Ordinal)

        If prefixIndex = -1 Then
            Return String.Empty
        End If

        Dim startIndex As Integer = prefixIndex + prefix.Length
        Dim endIndex As Integer = message.IndexOf(")", startIndex, StringComparison.Ordinal)

        If endIndex = -1 Then
            endIndex = message.Length
        End If

        Return message.Substring(startIndex, endIndex - startIndex)
    End Function

    Private NotInheritable Class LogEntry
        Public Property Category As String
        Public Property Branch As String
        Public Property Message As String
    End Class

    Private NotInheritable Class NullScope
        Implements IDisposable

        Public Shared ReadOnly Instance As New NullScope()

        Private Sub New()
        End Sub

        Public Sub Dispose() Implements IDisposable.Dispose
        End Sub
    End Class
End Class
```

**Mock-based tests** must still follow the same sectioning and verification rules detailed above:
- **All mocks must be created in verifiable mode**:
  - Use `New Mock(Of T)(MockBehavior.Strict)` to catch any unexpected calls, _or_
  - Call `.Verifiable()` on each `mock.Setup(...)` you intend to verify.
- **Every mock setup must be verified in the BEHAVIOR section** via `mock.Verify(...)` or `mock.VerifyAll()`; unverified verifiable mocks should fail the test.

✅ **Good Example:**
```vb
<Test>
Public Sub Test_ServiceCallsDependency()
    ' MOCKING: create a strict dependency mock
    Dim mockDependency As New Mock(Of IDependency)(MockBehavior.Strict)
    mockDependency.Setup(Function(x) x.PerformTask()).Verifiable()

    ' SETUP: expose the mock through the dependency interface
    Dim envDependency As IDependency = mockDependency.Object

    ' SYSTEM UNDER TEST
    Dim sut As New MyService(envDependency)

    ' WHEN
    sut.DoWork()

    ' BEHAVIOR
    mockDependency.Verify(Function(x) x.PerformTask(), Times.Once())
End Sub
```

🚫 **Bad Example:**
```vb
<Test>
Public Sub Test_ServiceCallsDependency()
    Dim mockDependency As New Mock(Of IDependency)() ' ❌ No section-separators
    Dim service As New MyService(mockDependency.Object) ' ❌ Passing the mock object directly to the SUT
    service.DoWork()
    mockDependency.Verify(Function(x) x.PerformTask(), Times.Once())
End Sub
```

---

## **8. Assertions & Variable Naming**
Strict naming patterns for expected and actual values highlight the difference between inputs, outputs, and verifications, which speeds up failure analysis. Mixing literals and setup variables inside assertions hides intent, makes diffs noisy, and increases the chance of asserting against the wrong data.
- **Expected values** always assign input values (from the GIVEN section) to `expected*` variables in the EXPECTATIONS section if you intend to assert on them in the THEN or BEHAVIOR sections.
- **Actual results** must be assigned to `actual*` variables in the WHEN section.
- **Never assert against literals directly** — use `expected*` variables.
- **Never assert against `given*` variables directly** — assign them to `expected*` variables in the EXPECTATIONS section and use those.
- **Never assert against `env*` variables directly** — assign them to `actual*` variables exclusively in the WHEN section and use those.
- **Never assert against `mock*` variables directly in the THEN section** — assign them to `expected*` variables in the EXPECTATIONS section and use those.
- **It is acceptable to assert against `mock*` variables directly in the BEHAVIOR section**.
- To reiterate: **Never assert directly against `given*` or `env*` variables** — always use the `expected*` or `actual*` variables in the THEN section.
- To reiterate: **This also holds true for mocks** — always use the `expected*` or `actual*` variables in the BEHAVIOR section when asserting on the behavior of mock objects, which must be named `mock*`.
- To reiterate once more: assertions should never ever ever refer to `given*` or `env*` values, and should only refer to `mock*` variables in the BEHAVIOR section.

✅ **Good Example:**
```vb
Dim expectedResult As Integer = 42
Assert.AreEqual(expectedResult, actualResult)
```
🚫 **Bad Example:**
```vb
Assert.AreEqual(42, actualResult) ' ❌ No expected variable
```

---

## **9. Exception Handling (`Assert.Throws`)**
Treating exception capture as part of the WHEN stage keeps control flow explicit and prevents assertions from being buried inside delegates. When the thrown exception is not stored and verified deliberately, tests can pass for the wrong reason, masking regressions where different error types or messages are emitted.

When using a mechanism such as `Assert.Throws` or `Assert.ThrowsAsync` to wrap the SUT invocation, the exception name that is being asserted against will be placed above the code block that exercises the system under test. In such cases, consider the assertion mechanism to be part of the WHEN section. Assign the actual exception to a local variable and assert that it is as expected in a subsequent THEN section.

✅ **Good Example:**
```vb
<Test>
Public Sub Test_DivideByZero_ThrowsException()
    ' GIVEN
    Dim givenNumerator As Integer = 10
    Dim givenDenominator As Integer = 0

    ' WHEN
    Dim actualException As DivideByZeroException = Assert.Throws(Of DivideByZeroException)(Sub()
                                                                                               Math.Divide(givenNumerator, givenDenominator)
                                                                                           End Sub)

    ' EXPECTATIONS
    Dim expectedExceptionType As Type = GetType(DivideByZeroException)

    ' THEN
    Assert.AreEqual(expectedExceptionType, actualException.GetType())
End Sub
```

---

## **10. Using `[SetUp]` Methods**
Being intentional about setup hooks ensures shared initialization is truly common while test-specific context stays near the scenario. Overusing setup hooks leads to hidden coupling between tests, complicated fixture state, and surprises when one case mutates shared members that another silently depends on.
- **Avoid setup attributes** (`<TestInitialize>`, `<SetUp>`, `<SetUpAttribute>`) unless absolutely necessary. Favor using sharable fixtures, declared in a static fixture factory class, instead.
- Use setup hooks only for **repeated initialization** that genuinely applies to every test in the class.

✅ **Good Example:**
```vb
<TestInitialize>
Public Sub Setup()
    _service = New MyService()
End Sub
```
🚫 **Bad Example:**
```vb
<TestInitialize>
Public Sub Setup()
    Dim service = New MyService() ' ❌ Redundant per-test instantiation without storing it
End Sub
```

---

## **11. Organizing Tests in Namespaces**
Mirroring production namespaces in the test suite keeps navigation intuitive and tooling-friendly, so developers can jump between code and tests effortlessly. When namespace structures drift, IDE search results become noisy, automated discovery may misbehave, and contributors struggle to find the right home for new cases.
- **Group related test classes into namespaces**.
- Ensure **each namespace matches the SUT structure**.
- Use VB's `Namespace ... End Namespace` block with consistent indentation (namespace at column 0, contents indented once).

✅ **Good Example:**
```vb
Namespace Tests.ServicesTests

Public Class UserServiceTests
End Class

End Namespace
```

🚫 **Bad Example:**
```vb
Namespace Services.Tests ' ❌ Inconsistent naming

Public Class UserServiceTests
End Class

End Namespace
```

🚫 **Bad Example:**
```vb
    Namespace Services.ServicesTests ' ❌ Namespace line must not be indented

    Public Class UserServiceTests
    End Class

    End Namespace
```

---

## **12. Managing Copy-Paste Setup**
Copy-paste programming is often the most readable choice for test setup. Start with a "happy path" test that spells out the full arrangement, and copy that scaffolding into edge-case tests, tweaking only what changes for each branch. With disciplined sectioning and modern AI-assisted refactoring tools, duplicating the setup is usually quicker and clearer than inventing cryptic helper methods whose behavior must be reverse-engineered.
- Prefer duplication first, then refactor **only when** the abstraction makes the setup easier to understand than the copied code it replaces.
- When a shared helper truly improves clarity, keep it near the tests that use it and document which scenarios rely on it so future contributors know when to extend or bypass it.
- When you do copy setup code, call out the variations in the section-header comments (e.g. `' GIVEN: user has insufficient balance`). These comments act as signposts so reviewers can immediately see why one test diverges from another despite similar bodies.

---

## **13. Deterministic Tests and Dependency Injection**
Unit tests should be deterministic: the same inputs must produce the same results every run. Nondeterminism from global state, random values, or "current time" APIs usually erodes trust in the suite. Occasionally, allowing nondeterminism in aspects that **should not** affect the outcome is valuable—flaky failures in those cases expose hidden coupling or misunderstood behavior. Treat such flakes as signals to either fix the bug they reveal or document the learning and firm up the contract under test.
- Isolate side effects behind interfaces and inject them into the system under test. Use dependency injection (DI) so the test can supply controlled fakes, especially for time-sensitive behavior.
- Avoid calling `DateTime.Now`, `Guid.NewGuid()`, or static singletons from production code during tests. Instead, inject a clock, ID generator, or configuration provider that the test can stub.
- For time-sensitive logic, pass an `ISystemClock` or similar abstraction. Tests can freeze "now" to a predictable value while production supplies a real clock, making the deterministic path explicit.
- When code mixes deterministic calculations with nondeterministic reads, refactor into two members: one that gathers inputs (time, randomness, HTTP responses) and another that performs pure computation. Unit test the deterministic member directly and cover the integration path separately if needed.

✅ **Example: controlling current time with DI**
```vb
Public Interface IClock
    ReadOnly Property UtcNow As DateTimeOffset
End Interface

Public Class InvoiceService
    Private ReadOnly _clock As IClock

    Public Sub New(clock As IClock)
        _clock = clock
    End Sub

    Public Function CreateInvoice(order As Order) As Invoice
        Dim generatedAt As DateTimeOffset = _clock.UtcNow
        Return New Invoice(order.Id, generatedAt)
    End Function
End Class

<Fact>
Public Sub Test_CreateInvoice_UsesProvidedClock()
    ' GIVEN
    Dim givenNow As DateTimeOffset = DateTimeOffset.Parse("2024-02-01T09:30:00Z")
    Dim mockClock As New Mock(Of IClock)(MockBehavior.Strict)
    mockClock.Setup(Function(clock) clock.UtcNow).Returns(givenNow).Verifiable()

    ' SYSTEM UNDER TEST
    Dim envClock As IClock = mockClock.Object
    Dim sut As New InvoiceService(envClock)

    ' WHEN
    Dim actualInvoice As Invoice = sut.CreateInvoice(New Order(Guid.Empty))

    ' EXPECTATIONS
    Dim expectedGeneratedAt As DateTimeOffset = givenNow

    ' THEN
    Assert.AreEqual(expectedGeneratedAt, actualInvoice.GeneratedAt)

    ' BEHAVIOR
    mockClock.Verify(Function(clock) clock.UtcNow, Times.Once())
End Sub
```

---

## **14. Using comments in tests**
Documenting test intent with XML comments provides high-level context that complements the structured sections, guiding readers through why a scenario exists. Without these explanations, future maintainers must infer purpose from mechanics alone, risking redundant cases or accidental deletion of critical coverage.

- **Each test method should be commented with a human-readable explanation of what the test is exercising**
- **Each test class should be commented with a human-readable explanation of what the test class is exercising**
- **Use the XML comment syntax (`'''`) to comment test classes and test methods**

## **15. Increasing test clarity with section comments**
Narrated section headers transform dense setup or verification code into self-explanatory stories that highlight intent and edge cases. Skipping these comments forces reviewers to reverse-engineer the reason behind each block, slowing code reviews and making it easier for subtle regressions to slip through.

**As a best practice**, when tests have complex parts such as setting up mocks, creating complex `given` objects, asserting on mock behavior, and so forth, it is recommended to split each such part into a section of its own, and comment what that section is doing. These headers act like a decryption key for the reader: each full-sentence comment explicitly states the intent of the code beneath it, so that even when the implementation is dense or unfamiliar, the reader can immediately understand _why_ a block exists rather than mentally reverse-engineering the setup from the statements themselves.

✅ **Good Example:**
```vb
Namespace Tests.ServicesTests

''' <summary>
''' Covers the "Bar" region of the Foo.User.Service class.
''' </summary>
Public Class UserServiceTests

    ''' <summary>
    ''' Covers the case where FooBar equals "foo".
    ''' </summary>
    <Fact>
    Public Sub Test_FooBar_IsFoo()
        ' GIVEN: LogLevel.Information is enabled
        Dim givenInfoLevelEnabled As Boolean = True

        ' MOCKING: create a mocked logger that has info level enabled
        Dim mockLogger As New Mock(Of ILogger(Of UserService))()
        mockLogger.Setup(Function(l) l.IsEnabled(LogLevel.Information)).Returns(givenInfoLevelEnabled)

        ' SETUP: hide the fact that we're using a mocked logger
        Dim envLogger As ILogger(Of UserService) = mockLogger.Object

        ' SYSTEM UNDER TEST: instantiate and initialize the system under test
        Dim sut As New UserService(envLogger)

        ' WHEN: exercise the test case
        Dim actualResult As Boolean = sut.IsFoo()

        ' EXPECTATIONS: we expect a log message to have been emitted
        Dim expectedLogMessageFragment As String = "IsFoo was called"

        ' EXPECTATIONS: we expect the result of IsFoo to be true
        Dim expectedResult As Boolean = True

        ' THEN: Verify that the actual status code and result match expectations.
        Assert.AreEqual(expectedResult, actualResult)

        ' BEHAVIOR: we expect IsFoo to have logged a message on info level
        mockLogger.Verify(Sub(log)
                              log.Log(
                                  It.Is(Of LogLevel)(Function(l) l = LogLevel.Information),
                                  It.IsAny(Of EventId)(),
                                  It.Is(Of It.IsAnyType)(Function(v, t) v.ToString().Contains(expectedLogMessageFragment)),
                                  Nothing,
                                  It.IsAny(Of Func(Of It.IsAnyType, Exception, String))()
                              )
                          End Sub,
                          Times.Once(),
                          $"Expected an '{expectedLogMessageFragment}' log entry.")
    End Sub
End Class

End Namespace
```

🚫 **Bad Example:**
```vb
Namespace Services.Tests ' ❌ Inconsistent naming

Public Class UserServiceTests
    <Fact>
    Public Sub Test_FooBar_IsFoo()
    End Sub
End Class

End Namespace
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

