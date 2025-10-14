# Java Unit Testing Style Guide

This document defines **best practices** for writing **clear, structured, and maintainable** unit tests in Java using JUnit 5, TestNG, or similar frameworks.

---

## **Target Audience**

The primary target audience for this document is developers and Large Language Models (LLMs) acting as coding assistants. By following these guidelines, tests will be **structured, readable, and maintainable**.

These practices may be too detailed for human-written tests but serve as an excellent guide for generating automated tests with LLM assistance. **Review all test methods carefully**, as human oversight is crucial to ensure test quality.

---

## **1. Test Package Organization**
Consistent package names make it immediately obvious where a test lives relative to the production code, so reviewers can locate the subject under test without opening multiple files. Misaligned packages cause confusion when navigating between source and tests, especially in IDEs that rely on package matching for discovery.
- Every test file must declare a package that mirrors the production package and appends a `.tests` suffix (e.g. `com.myco.product.service` → `com.myco.product.service.tests`).
- When the production code uses nested packages, mirror that structure exactly in the test package before adding `.tests`.
- Keep a **one-to-one mapping between folders and packages** so moving a file preserves the package name.
- Avoid combining test classes for different production packages under a single package—create separate packages instead.

✅ **Good Example:**
```java
package com.myco.product.service.tests;
```
🚫 **Bad Example:**
```java
package tests; // ❌ Does not mirror the production package
```

---

## **2. Test Class Naming and Files**
Consistent test class names make it immediately obvious which production code a suite exercises, so reviewers can trace failures quickly and spot coverage gaps. Without a naming convention teams waste time hunting for relevant tests, duplicate effort, and risk overlooking scenarios because responsibilities are spread across ambiguously labeled files.
- Each test file must contain a single public test class whose name **ends with** `Test` or `Tests`.
- If a test class exercises a whole class and all of its methods, it must be named `<ClassName>Test`.
- If a test class exercises a single logical feature within a class, it must be named `<ClassName><FeatureName>Test`.
- If a test class exercises a single method within a class, it must be named `<ClassName><MethodName>Test` or `<ClassName><FeatureName><MethodName>Test`.
- Feature-focused and method-focused test classes are **optional** patterns. Prefer them when they make intent clearer, but feel free to keep everything in a single `<ClassName>Test` class when the class-under-test is small enough that splitting would add overhead without clarity.
- Test class names should not contain underscores.
- **Each test class should cover a single class, feature, or method under test.**
- **A test class' file name must be of the form `<ClassName>[<FeatureName>][<MethodName>]Test.java`**, and every component of the file name must appear in the class name exactly once and in the same order.
- Organize test files into folders that mirror the package to keep discovery tools and human navigation aligned.

✅ **Good Examples:**
```java
public class FileHandlerTest { }
public class FileHandlerHashingTest { }
public class FileHandlerOpenTest { }
```
🚫 **Bad Examples:**
```java
public class FileHandler { } // ❌ Missing Test suffix
public class FileHandlerWorksAsIntendedTest { } // ❌ WorksAsIntended is not a plausible feature name
public class FileHandlerInitializationAndHashingTest { } // ❌ Combines multiple features into one class
```

---

## **3. Feature-Focused Test Classes**
Feature boundaries describe cohesive areas of responsibility inside a production class. Mirroring those boundaries in the test suite keeps the feedback loop tight by ensuring every feature has an intentionally scoped test class. Treat these feature-focused classes as a tool rather than a mandate—when a production class only has a handful of features or methods, duplicating files for each can be overkill.
- Only introduce a `<FeatureName>` segment when the production code exposes a clearly named concept (e.g. a nested static class, a public inner class, or a `// region` comment) with the same PascalCase name.
- When a class contains multiple distinct features, create a separate test class per feature, following the naming rules above and keeping each class in its own file.
- Do not reuse a `<FeatureName>` segment across unrelated production features; each feature must map to a single test class.
- If a feature contains helper methods that are not directly invoked by the public API, test the public surface that exercises them rather than testing the helpers directly.

✅ **Good Example:**
```
service/
│── FileHandler.java          // Contains features Hashing and Open
service/tests/
│── FileHandlerHashingTest.java
│── FileHandlerOpenTest.java
```
🚫 **Bad Example:**
```
service/tests/
│── FileHandlerHashingAndOpenTest.java // ❌ Merges multiple features into a single test class
```

---

## **4. Test Method Naming**
Clear method names document the behavior under test and the expected outcome, which helps maintainers understand intent without rereading the implementation. Vague or inconsistent names hide gaps in coverage, encourage multi-purpose tests, and make regression triage harder when a failing test name does not reveal what broke.
- Each test method **must start with** `test`.
- Use lower camel case with underscores to separate major phrases in the test method name (e.g. `testProcessData_returnsCorrectValue`).
- Although test classes stay in PascalCase with no underscores, **test methods are encouraged to use underscores** so long names remain readable.
- Test method names should **not contain consecutive underscores**.
- If multiple production methods are being tested within the same test class, the test method **must start with** `test<MethodName>_`.
- If a single production method is being tested in the file, the production method name **must be omitted** from the test method.
- The method name should be **descriptive and readable**, reflecting the behavior under test.
- Favor verbosity over brevity—short names rarely communicate enough context to make the test self-documenting.
- Test method names must describe the **specific branch of execution** they cover (e.g. `testProcessData_returnsCorrectValue_whenSuccessful`), not merely state that something "works".
- **Separate tests** into distinct methods rather than combining multiple test cases.
- **Avoid parameterized tests** unless they significantly improve clarity.
- Any component found in the test class name **must NOT be duplicated in the test method name**.

✅ **Good Example:**
```java
@Test
void testProcessData_returnsCorrectValue() { }
```
🚫 **Bad Examples:**
```java
@Test
void checkProcessData() { } // ❌ Missing test prefix
@Test
void testProcess() { } // ❌ Too vague
@Test
void testMultipleCases() { } // ❌ Tests multiple things at once
```

---

## **5. Test Method Sectioning**
Standardized sections carve complex tests into digestible steps, making it easier to see how inputs flow through mocks and the system under test. Without this structure tests devolve into monolithic blocks where intent, setup, and verification intermingle, obscuring bugs and encouraging brittle copy-paste patterns. The ordering rules below are intentionally strong to build reliable habits—refer to [**Section 17. Breaking the Rules**](#-17-breaking-the-rules) for guidance on the rare cases where deviating is justified.
Each test method follows a structured format, **separated by clear comments**:
- Where possible, the order of sections should be as it is presented below.
- Empty sections should be omitted.
- In some cases, it may be justifiable to have multiple instances of specific sections.
- When a single **MOCKING** or **SETUP** block becomes large or involves multiple distinct steps, split it into smaller sub-sections with their own human-readable comment headers (e.g. `// SETUP: Configure HTTP client factory` and `// SETUP: Initialize environment variables`), so readers can quickly grasp each part.
- Section comment headers should be written as full descriptive sentences (e.g. `// GIVEN: A valid request and user account`), not terse labels. If a section would contain only a trivial one-line of code without setup complexity, you may omit its own comment.

### **Standard Sections:**

The following sections are defined for a well structured test.

1. **GIVEN**
   - Set up initial conditions and inputs.
   - Assign variables with the prefix `given`.
   - When test fixtures are used, they should be allocated in this section.
   - **Test fixture classes** should be named `Fixture*` and assigned to `given*` variables prior to use.
   - The GIVEN section should always be the first section in a test, unless there is nothing to define in GIVEN, in which case it should be omitted.
   - **The GIVEN section must not be merged with any other section.**

2. **CAPTURE**
   - Declare variables that will hold values captured from mocks or fakes.
   - Prefix captured placeholders with `capture` (e.g., `capturePublishedEvent`).
   - The CAPTURE section appears **after GIVEN and before MOCKING** when capturing is required.
   - Limit the section to declaring the capture containers. The logic that records values (e.g., configuring mocks or test doubles) belongs in MOCKING or SETUP.
   - Skip this section when no captured values are needed.

3. **MOCKING**
   - Define and configure **mock objects**.
   - Mock variables must be named `mock*`.
   - **Use constructor injection for dependencies**, or use mocking frameworks like Mockito.
   - The MOCKING section should occur before the SETUP section, or be omitted if no mocking is required. Under special circumstances described above, it may be placed after the SETUP section.
   - **The MOCKING section must not be merged with any other section.**
   - **If mocking logic itself has multiple responsibilities (e.g. creating fakes and configuring behaviors), split into sub-headers** (e.g. `// MOCKING: Create mock objects` and `// MOCKING: Configure mock responses`).

4. **SETUP**
   - Perform all test initialization that prepares the non-mocked environment (e.g., creating HttpServletRequest, configuring dependency injection, and setting up request parameters).
   - Variables created in this section should be prefixed with `env*` (e.g. `envHttpRequest`, `envServiceRegistry`).
   - When constants or variables are needed in this section that are important for the overall comprehensibility of the test, they should be declared in variables in the GIVEN section instead.
   - The SETUP section should follow the preceding arrangement sections (GIVEN → CAPTURE → MOCKING). If those sections are omitted, SETUP becomes the first applicable section.
   - The SETUP section should refer to GIVEN variables, not `mock*` variables, except in rare occasions.
   - Assign mock objects to concrete interface variables in the SETUP section, using the `env*` prefix (e.g., `envLogger = mockLogger`). The mock variables themselves may only be used in the SETUP and BEHAVIOR sections.
   - **The SETUP section must not be merged with any other section.**
   - **If SETUP involves multiple distinct stages (e.g. reading fixtures, wiring dependencies, configuring environment), split into multiple sub-sections with descriptive comments.**

5. **SYSTEM UNDER TEST**
   - Assigns the system under test to a variable named `sut`.
   - If multiple systems are being tested for some reason, assign them to separate `sut*` variables.
   - The SYSTEM UNDER TEST must never directly reference `mock*` variables. It must use the `env*` variables initialized in SETUP.
   - **The SYSTEM UNDER TEST section must not be merged with any other section.**

6. **WHEN**
   - Perform the action or method call being tested.
   - Assign the results to `actual*` variables. This would typically be the return value received from invoking the system under test.
     - If the test requires asserting against other results in the THEN section or BEHAVIOR section, then use the opportunity to assign those results to their own `actual*` variables, e.g. after invoking the system under test.
   - If the test needs to use an `env*` in later sections, then assign them to `actual*` variables in WHEN and use those instead.
   - When values have been captured into `capture*` placeholders, assign them to `actual*` variables here (e.g., `String actualPayload = capturePayload.get();`) before performing assertions.
   - **The WHEN section must not be merged with any other section.**

7. **EXPECTATIONS**
   - Define expected values **before assertions**.
   - Expected values must be assigned to `expected*` variables.
   - The EXPECTATIONS section should be strictly placed after WHEN and before THEN.
   - This section should not refer to any `actual*` variables.
   - **The EXPECTATIONS section must not be merged with any other section.**

8. **THEN**
   - Perform **assertions** comparing actual results to expected values.
   - **Never assert against literals**—always use `expected*` variables.
   - **The THEN section must not be merged with any other section.**

9. **LOGGING**
   - Verify any behavior that can be validated via captured logging operations.
   - Capture the behavior of the code under test at appropriate levels (including trace-level logging with extensive details).
   - In unit tests, assert against log entries in the LOGGING section to confirm that the intended execution branch was exercised.
   - **Prefer asserting on emitted logs over (or in addition to) mock verifications** whenever possible: log-based assertions are simpler to write and review, and often replace complex mock setups (e.g., capturing parameters).
   - Each unit test should explicitly verify that the execution branch it is meant to cover is taken, reducing false positives when setup does not exercise the desired branch.

10. **BEHAVIOR**
   - Contains assertions for **mock interactions** (e.g., `Mockito.verify(mockService).doSomething();`).
   - This section **must be present if mocks are used**.
   - **The BEHAVIOR section must not be merged with any other section.**
   - **All mocks must be created with strict behavior**—either using Mockito's `lenient()` only when needed or ensuring expectations are declared explicitly.
   - **Every verifiable setup must be explicitly verified** in this section via `Mockito.verify(...)`. Unverified verifications should cause the test to fail.

✅ **Good Example:**
```java
@Test
void testProcessData_returnsCorrectValue() {
    // GIVEN
    String givenInput = "sample";

    // SYSTEM UNDER TEST
    DataService sut = new DataService();

    // WHEN
    String actualResult = sut.processData(givenInput);
    String actualSideEffect = Foo.bar();

    // EXPECTATIONS
    String expectedResult = "processed_sample";
    String expectedSideEffect = "fubar";

    // THEN
    assertEquals(expectedResult, actualResult);
    assertEquals(expectedSideEffect, actualSideEffect);
}
```
🚫 **Bad Example:**
```java
@Test
void testProcessData_returnsCorrectValue() {
    // MOCKING
    Service mockService = Mockito.mock(Service.class); // ❌ Missing section separation and unused result

    // SYSTEM UNDER TEST
    DataService sut = new DataService(mockService);
}
```

---

## **6. Fixtures: Best Practices**
Reusable fixtures encourage realistic, DRY data that mirrors production scenarios while keeping tests focused on behavior instead of data plumbing. Inlining bespoke objects everywhere invites duplication, increases maintenance costs when models evolve, and makes it harder to reason about how changes cascade across tests.
- **Prefer fixtures over mocks**—favor shared, reusable fixtures rather than creating one-off test data. Fixtures help keep your tests DRY and consistent, and allow them to work on more accurate test data.
- **Instantiate fixtures in the appropriate section** e.g. GIVEN or SETUP, and modify them as needed to fit the test's needs.
- **Use a sharable fixture factory** instead of creating fixtures or complex test data inline.

✅ **Good Example:**
```java
public final class Fixtures {
    private Fixtures() { }

    public static String ipAddress() {
        return "192.168.1.1";
    }

    public static CwConfig cwConfig() {
        return new CwConfig(List.of());
    }

    public static CwGetTerminalConfigCommand cwGetTerminalConfigCommand() {
        String givenIpAddress = ipAddress();
        return new CwGetTerminalConfigCommand(givenIpAddress);
    }

    public static CommandContext<CwGetTerminalConfigCommand, CwGetTerminalConfigResult> commandContext() {
        CommandContext<CwGetTerminalConfigCommand, CwGetTerminalConfigResult> context = new CommandContext<>();
        context.setCommand(cwGetTerminalConfigCommand());
        return context;
    }
}

class CarWashesServiceTest {

    @Test
    void testGetTerminalConfig_returnsNotFound_whenTerminalDoesNotExist() {
        // GIVEN
        String givenTerminalIp = Fixtures.ipAddress();
        CommandContext<CwGetTerminalConfigCommand, CwGetTerminalConfigResult> givenCommandContext = Fixtures.commandContext();
        CwConfig givenCarWashesConfig = Fixtures.cwConfig();

        // SYSTEM UNDER TEST
        CarWashesService sut = new CarWashesService(givenCarWashesConfig);

        // WHEN
        sut.getTerminalConfig(givenCommandContext);
        int actualStatusCode = givenCommandContext.getStatusCode();
        CwGetTerminalConfigResult actualResult = givenCommandContext.getResult();

        // EXPECTATIONS
        int expectedStatusCode = HttpURLConnection.HTTP_NOT_FOUND;
        CwGetTerminalConfigResult expectedResult = null;

        // THEN
        assertEquals(expectedStatusCode, actualStatusCode);
        assertEquals(expectedResult, actualResult);
    }
}
```
🚫 **Bad Example:**
```java
@Test
void testGetTerminalConfig_returnsNotFound_whenTerminalDoesNotExist() {
    // GIVEN
    String givenTerminalIp = "192.168.1.1";
    CommandContext<CwGetTerminalConfigCommand, CwGetTerminalConfigResult> givenCommandContext =
        new CommandContext<>();
    givenCommandContext.setCommand(new CwGetTerminalConfigCommand(givenTerminalIp));
    CwConfig givenCarWashesConfig = new CwConfig(new ArrayList<>());

    // SYSTEM UNDER TEST
    CarWashesService sut = new CarWashesService(givenCarWashesConfig);

    // WHEN
    sut.getTerminalConfig(givenCommandContext);
    int actualStatusCode = givenCommandContext.getStatusCode();
    CwGetTerminalConfigResult actualResult = givenCommandContext.getResult();

    // EXPECTATIONS
    int expectedStatusCode = HttpURLConnection.HTTP_NOT_FOUND;
    CwGetTerminalConfigResult expectedResult = null;

    // THEN
    assertEquals(expectedStatusCode, actualStatusCode);
    assertEquals(expectedResult, actualResult);
}
```

---

## **7. Mocking: Best Practices**
Disciplined mocking keeps tests readable and trustworthy by ensuring only true collaborators are simulated and their expectations are explicit. Unrestrained mocks create brittle, implementation-driven tests that break on harmless refactors, mask missing coverage of dependencies, and clutter SUT construction with framework boilerplate.
- **Never mock what you don't have to**—prefer fixtures or real instances where practical.
- **Only mock things that the system-under-test uses directly**—this ensures that your test exercises the SUT properly, without falling into the trap of combinatorial execution path counts as call-depth increases.
- The systems that your SUT uses directly should be covered by their own direct unit tests, not by the unit tests of your SUT.
- **Use Mockito, EasyMock, or built-in mocking frameworks** for dependency injection.
- **Assertions on mock interactions go in the BEHAVIOR section, not in the THEN section.**
- **Mocks must never be passed directly to the system under test.** Instead, assign mocks to interface-conforming `env*` variables in SETUP and pass those to the SUT.
- Following this pattern keeps the SUT construction readable: the MOCKING section owns the intricate setup, the SETUP section exposes the ready-to-use interfaces via `env*` variables, and the SYSTEM UNDER TEST section reads like a clean recipe that highlights only the essential collaborators. Without the intermediate `env*` assignments the constructor or method invocation under test quickly devolves into a wall of `.get()` accessors and configuration noise, making it difficult for reviewers to decipher which dependency is which.

### **7.1 Test Fakes (a.k.a. Dummies)**
Test fakes are lightweight, purpose-built implementations of an interface that you author specifically for the test suite. They shine when mocking frameworks become gnarly—especially for complex interaction verification or structured log inspection—because you can write direct, intention-revealing code without wrestling with callback signatures or argument matchers.

- **Create fakes when setup or verification with a mock would be noisy, repetitive, or brittle.** If verifying interactions through a mocking framework requires answer callbacks, reflection, or deeply nested `ArgumentMatchers`, a fake is often clearer.
- **Name fake instances with the `fake*` prefix and construct them in the MOCKING section.** This keeps parity with mock naming so readers instantly recognize simulated collaborators.
- **Assign each `fake*` variable to an `env*` variable in the SETUP section before handing it to the SUT.** Treat a fake like any other dependency so the SYSTEM UNDER TEST section stays clean and only references `env*` collaborators.
- **Interact with the fake via its `fake*` variable in the THEN, LOGGING, and BEHAVIOR sections.** Fakes frequently expose custom assertion helpers (e.g., `fakeLogger.assertExercised(...)`) that capture behavior without `Mockito.verify(...)` boilerplate.
- **Document expectations inside the fake when possible.** Purpose-built helpers (such as storing structured log entries) make intent obvious and reduce duplicated parsing logic across tests.

#### Strengths Compared to Mocks
- **Readable behavior verification.** Fakes encapsulate the verification logic in methods that read like English, avoiding the visual clutter of `times(1)` and matcher chains.
- **Reduced setup friction.** Because you own the implementation, you can build constructors and helper methods that mirror your domain vocabulary, instead of contorting the SUT to satisfy framework APIs.
- **Deterministic assertions.** Fakes can capture state (e.g., recorded log entries) and expose first-class assertions, lowering the risk of brittle, order-dependent mock verifications.

#### Weaknesses Compared to Mocks
- **Maintenance cost.** You must maintain the fake implementation alongside production interfaces. If the interface evolves, the fake must be updated manually.
- **Limited behavioral coverage.** A fake typically encodes only the paths needed by the current test suite. Mocks can dynamically configure behaviors per test without editing shared code.
- **Risk of drifting from reality.** Because a fake is handwritten, it might omit subtle behavior the real dependency exhibits. Use fixtures and integration tests to guard against divergence.

#### Example: Logging with a Fake vs. a Mock
Consider a service that logs a structured trace message—`"Foo(branch=bar): the foo is quite barred"`—whenever it processes a branch.

**✅ Fake-based test (clean and intention revealing):**
```java
@Test
void testProcessBranch_logsTraceMessage() {
    // GIVEN: a branch identifier that should trigger logging
    String givenBranchId = "bar";

    // MOCKING: create test fakes for collaborators
    FakeLogger fakeLogger = new FakeLogger();

    // SETUP: expose the fake through env* variables
    Logger envLogger = fakeLogger;

    // SYSTEM UNDER TEST: inject the fake logger
    BranchProcessor sut = new BranchProcessor(envLogger);

    // WHEN: process the branch
    sut.processBranch(givenBranchId);

    // LOGGING: ensure the trace message was written
    fakeLogger.assertExercised("Foo", "bar");
}

private interface Logger {
    void info(String message);
}

private static final class FakeLogger implements Logger {
    private final List<LogEntry> entries = new ArrayList<>();

    @Override
    public void info(String message) {
        entries.add(new LogEntry("Foo", "bar", message));
    }

    void assertExercised(String expectedCategory, String expectedBranch) {
        LogEntry entry = assertSingle(entries);
        Assertions.assertEquals(expectedCategory, entry.category());
        Assertions.assertEquals(expectedBranch, entry.branch());
        Assertions.assertEquals("Foo(branch=" + expectedBranch + "): the foo is quite barred", entry.message());
    }

    private static LogEntry assertSingle(List<LogEntry> entries) {
        Assertions.assertEquals(1, entries.size());
        return entries.get(0);
    }
}

private record LogEntry(String category, String branch, String message) { }
```

**✅ Mock-based test (strict and explicit):**
```java
@Test
void testService_callsDependency() {
    // MOCKING: create a strict dependency mock
    Service mockDependency = Mockito.mock(Service.class, Mockito.withSettings().strictness(Strictness.STRICT_STUBS));

    // SETUP: expose the mock through the dependency interface
    Service envDependency = mockDependency;

    // SYSTEM UNDER TEST
    MyService sut = new MyService(envDependency);

    // WHEN
    sut.doWork();

    // BEHAVIOR
    Mockito.verify(mockDependency).performTask();
}
```
🚫 **Bad Example:**
```java
@Test
void testService_callsDependency() {
    Service mockDependency = Mockito.mock(Service.class);  // ❌ No section separators
    MyService sut = new MyService(mockDependency); // ❌ Passing the mock object directly to the SUT
    sut.doWork();
    Mockito.verify(mockDependency).performTask();
}
```

---

## **8. Assertions & Variable Naming**
Strict naming patterns for expected and actual values highlight the difference between inputs, outputs, and verifications, which speeds up failure analysis. Mixing literals and setup variables inside assertions hides intent, makes diffs noisy, and increases the chance of asserting against the wrong data.
- **Expected values** always assign input values (from the GIVEN section) to `expected*` variables in the EXPECTATIONS section if you intend to assert on them in the THEN or BEHAVIOR sections.
- **Actual results** all actual results must be assigned to `actual*` variables in the WHEN section.
- **Never assert against literals directly** — use `expected*` variables.
- **Never assert against `given*` variables directly** — assign them to `expected*` variables in the EXPECTATIONS section and use those.
- **Never assert against `env*` variables directly** — assign them to `actual*` variables exclusively in the WHEN section and use those.
- **Never assert against `mock*` variables directly in the THEN section** — assign them to `expected*` variables in the EXPECTATIONS section and use those.
- **It is acceptable to assert against `mock*` variables directly in the BEHAVIOR section**.
- To reiterate: **Never assert directly against `given*` or `env*` values** — always use the `expected*` or `actual*` variables in the THEN section.
- To reiterate: **This also holds true for mocks** — always use the `expected*` or `actual*` variables in the BEHAVIOR section when asserting on the behavior of mock objects, which must be named `mock*`.

✅ **Good Example:**
```java
int expectedResult = 42;
assertEquals(expectedResult, actualResult);
```
🚫 **Bad Example:**
```java
assertEquals(42, actualResult); // ❌ No expected variable
```

---

## **9. Exception Handling (`assertThrows`)**
Treating exception capture as part of the WHEN stage keeps control flow explicit and prevents assertions from being buried inside lambda expressions. When the thrown exception is not stored and verified deliberately, tests can pass for the wrong reason, masking regressions where different error types or messages are emitted.

When using a mechanism such as `assertThrows` to wrap the SUT invocation, the exception name that is being asserted against will be placed above the code block that exercises the system under test. In such cases, consider the `assertThrows` mechanism to be part of the WHEN section. Assign the actual exception to a local variable and assert that it is as expected in a subsequent THEN section.

✅ **Good Example:**
```java
@Test
void testDivideByZero_throwsException() {
    // GIVEN
    int givenNumerator = 10;
    int givenDenominator = 0;

    // WHEN
    ArithmeticException actualException = assertThrows(ArithmeticException.class, () ->
        MathUtility.divide(givenNumerator, givenDenominator)
    );

    // EXPECTATIONS
    Class<?> expectedExceptionType = ArithmeticException.class;

    // THEN
    assertEquals(expectedExceptionType, actualException.getClass());
}
```

---

## **10. Using `@BeforeEach` Methods**
Being intentional about `@BeforeEach` usage ensures shared initialization is truly common while test-specific context stays near the scenario. Overusing setup hooks leads to hidden coupling between tests, complicated fixture state, and surprises when one case mutates shared members that another silently depends on.
- **Avoid `@BeforeEach`**. Favor using sharable fixtures, declared in a static fixture factory class, instead.
- Use `@BeforeEach` only for **repeated initialization** that is truly identical across every test in the class.

✅ **Good Example:**
```java
@BeforeEach
void setUp() {
    sut = new MyService();
}
```
🚫 **Bad Example:**
```java
@BeforeEach
void setUp() {
    MyService service = new MyService(); // ❌ Redundant per-test instantiation and unused field
}
```

---

## **11. Organizing Tests in Packages**
Mirroring production packages in the test suite keeps navigation intuitive and tooling-friendly, so developers can jump between code and tests effortlessly. When package structures drift, IDE search results become noisy, automated discovery may misbehave, and contributors struggle to find the right home for new cases.
- **Group related test classes into packages**.
- Ensure **each package matches the SUT structure**.

✅ **Good Example:**
```java
package com.myco.services.tests;

class UserServiceTest { /* ... */ }
```
🚫 **Bad Example:**
```java
package services.tests; // ❌ Inconsistent naming

class UserServiceTest { /* ... */ }
```
🚫 **Bad Example:**
```java
package services.servicestests; // ❌ Excess nesting not mirrored in production

class UserServiceTest { /* ... */ }
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
- Isolate side effects behind interfaces and inject them into the system under test. Use dependency injection so the test can supply controlled fakes, especially for time-sensitive behavior.
- Avoid calling `LocalDateTime.now()`, `UUID.randomUUID()`, or static singletons from production code during tests. Instead, inject a clock, ID generator, or configuration provider that the test can stub.
- For time-sensitive logic, pass a `Clock`, `Supplier<Instant>`, or similar abstraction. Tests can freeze "now" to a predictable value while production supplies a real clock, making the deterministic path explicit.
- When code mixes deterministic calculations with nondeterministic reads, refactor into two methods: one that gathers inputs (time, randomness, HTTP responses) and another that performs pure computation. Unit test the deterministic method directly and cover the integration path separately if needed.

✅ **Example: controlling current time with DI**
```java
interface Clock {
    Instant now();
}

final class InvoiceService {
    private final Clock clock;

    InvoiceService(Clock clock) {
        this.clock = clock;
    }

    Invoice createInvoice(Order order) {
        Instant generatedAt = clock.now();
        return new Invoice(order.id(), generatedAt);
    }
}

@Test
void testCreateInvoice_usesProvidedClock() {
    // GIVEN
    Instant givenNow = Instant.parse("2024-02-01T09:30:00Z");
    Clock mockClock = Mockito.mock(Clock.class);
    Mockito.when(mockClock.now()).thenReturn(givenNow);

    // SETUP
    Clock envClock = mockClock;

    // SYSTEM UNDER TEST
    InvoiceService sut = new InvoiceService(envClock);

    // WHEN
    Invoice actualInvoice = sut.createInvoice(new Order(UUID.randomUUID()));

    // EXPECTATIONS
    Instant expectedGeneratedAt = givenNow;

    // THEN
    assertEquals(expectedGeneratedAt, actualInvoice.generatedAt());

    // BEHAVIOR
    Mockito.verify(mockClock).now();
}
```

---

## **14. Using comments in tests**
Documenting test intent with Javadoc comments provides high-level context that complements the structured sections, guiding readers through why a scenario exists. Without these explanations, future maintainers must infer purpose from mechanics alone, risking redundant cases or accidental deletion of critical coverage.

- **Each test method should be commented with a human-readable explanation of what the test is exercising.**
- **Each test class should be commented with a human-readable explanation of what the test class is exercising.**
- **Use the Javadoc `/** ... */` syntax to comment test classes and test methods.**

## **15. Increasing test clarity with section comments**
Narrated section headers transform dense setup or verification code into self-explanatory stories that highlight intent and edge cases. Skipping these comments forces reviewers to reverse-engineer the reason behind each block, slowing code reviews and making it easier for subtle regressions to slip through.

**As a best practice**, when tests have complex parts such as setting up mocks, creating complex `given` objects, asserting on mock behavior, and so forth, it is recommended to split each such part into a section of its own, and comment what that section is doing. These headers act like a decryption key for the reader: each full-sentence comment explicitly states the intent of the code beneath it, so that even when the implementation is dense or unfamiliar, the reader can immediately understand _why_ a block exists rather than mentally reverse-engineering the setup from the statements themselves.

✅ **Good Example:**
```java
package com.myco.services.tests;

/**
 * Covers the "Bar" feature of the FooUserService class.
 */
interface Logger {
    boolean isInfoEnabled();
    void info(String message);
}

class UserServiceTest {

    /**
     * Covers the case where fooBar equals "foo".
     */
    @Test
    void testFooBar_isFoo() {
        // GIVEN: LogLevel.INFO is enabled
        boolean givenInfoLevelEnabled = true;

        // MOCKING: create a mocked logger that has info level enabled
        Logger mockLogger = Mockito.mock(Logger.class);
        Mockito.when(mockLogger.isInfoEnabled()).thenReturn(givenInfoLevelEnabled);

        // SETUP: hide the fact that we're using a mocked logger
        Logger envLogger = mockLogger;

        // SYSTEM UNDER TEST: instantiate and initialize the system under test
        UserService sut = new UserService(envLogger);

        // WHEN: exercise the test case
        boolean actualResult = sut.isFoo();

        // EXPECTATION: we expect a log message to have been emitted
        String expectedLogMessageFragment = "IsFoo was called";

        // EXPECTATION: we expect the result of isFoo to be true
        boolean expectedResult = true;

        // THEN: Verify that the actual status code and result match expectations.
        assertEquals(expectedResult, actualResult);

        // BEHAVIOR: we expect isFoo to have logged a message on info level
        Mockito.verify(mockLogger).info(Mockito.argThat(argument -> argument.contains(expectedLogMessageFragment)));
    }
}
```
🚫 **Bad Example:**
```java
package services.tests; // ❌ Inconsistent naming

class UserServiceTest {
    @Test
    void testFooBar_isFoo() {
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
- When you break a rule, **document the rationale inline** (e.g. with a comment or Javadoc) so future maintainers understand why the exception exists.
- Re-evaluate exceptions periodically. If the surrounding production code changes, the original constraint may disappear and the standard structure can be restored.
