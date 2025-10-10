# Scala Unit Testing Style Guide

This document defines **best practices** for writing **clear, structured, and maintainable** unit tests in Scala using ScalaTest, MUnit, or similar frameworks that target the JVM.

---

## **Target Audience**

The primary target audience for this document is developers and Large Language Models (LLMs) acting as coding assistants. By following these guidelines, tests will be **structured, readable, and maintainable**.

These practices may be too detailed for human-written tests but serve as an excellent guide for generating automated tests with LLM assistance. **Review all test functions carefully**, as human oversight is crucial to ensure test quality.

---

## **1. Test Package Organization**
Consistent package names make it immediately obvious where a test lives relative to the production code, so reviewers can locate the subject under test without opening multiple files. Misaligned packages cause confusion when navigating between source and tests, especially in IDEs that rely on package matching for discovery.
- Every test file must declare a package that mirrors the production package and appends a `.tests` suffix (e.g. `com.mycompany.products.services` → `com.mycompany.products.services.tests`).
- When the production code uses nested packages, mirror that structure exactly in the test package before adding `.tests`.
- Keep a **one-to-one mapping between folders and packages** so moving a file preserves the package name.
- Avoid combining test suites for different production packages under a single package—create separate packages instead.

✅ **Good Example:**
```scala
package com.mycompany.products.services.tests
```
🚫 **Bad Example:**
```scala
package tests // ❌ Does not mirror the production package
```

---

## **2. Test Suite Naming and Files**
Consistent suite names make it immediately obvious which production code a spec exercises, so reviewers can trace failures quickly and spot coverage gaps. Without a naming convention teams waste time hunting for relevant tests, duplicate effort, and risk overlooking scenarios because responsibilities are spread across ambiguously labeled files.
- Each test file must contain a single public test suite whose name **ends with** `Spec`.
- If a test suite exercises an entire class and all of its methods, it must be named `<ClassName>Spec`.
- If a suite focuses on a single logical region within a class (e.g., a companion object or nested type), name it `<ClassName><RegionName>Spec`.
- If a suite exercises a single method within a class, name it `<ClassName><MethodName>Spec` or `<ClassName><RegionName><MethodName>Spec`.
- Region-focused and method-focused suites are **optional** patterns. Prefer them when they make intent clearer, but feel free to keep everything in a single `<ClassName>Spec` when the class under test is small enough that splitting would add overhead without clarity.
- Suite names should not contain underscores.
- **Each suite should cover a single class, region, or method under test.**
- **A suite file name must be of the form `<ClassName>[<RegionName>][<MethodName>]Spec.scala`**, and every component of the file name must appear in the suite name exactly once and in the same order.
- Organize test files into folders that mirror the package to keep discovery tools and human navigation aligned.

✅ **Good Examples:**
```scala
final class FileHandlerSpec extends AnyFunSuite
final class FileHandlerHashingSpec extends AnyFunSuite
final class FileHandlerOpenSpec extends AnyFunSuite
```
🚫 **Bad Example:**
```scala
final class FileHandler extends AnyFunSuite // ❌ Missing Spec suffix
final class FileHandlerWorksAsIntendedSpec extends AnyFunSuite // ❌ WorksAsIntended is not a plausible logical region
final class FileHandlerInitializationAndHashingSpec extends AnyFunSuite // ❌ Combines multiple regions into one suite
```

---

## **3. Focused Suites for Logical Regions**
Large classes often group behavior into companion objects, nested types, or logically separated methods. Mirroring those regions in the test suite keeps the feedback loop tight by ensuring every region has an intentionally scoped suite. Treat these focused suites as a tool rather than a mandate—when a production class only has a handful of members, duplicating files for each can be overkill.
- Only introduce a `<RegionName>` segment when the production code exposes a logical grouping with the same PascalCase name (e.g., a companion object, nested type, or clearly named helper region).
- When a class contains multiple regions, create a separate suite per region, following the naming rules above and keeping each suite in its own file.
- Do not reuse a `<RegionName>` segment across unrelated regions; each region must map to a single suite.
- If a region contains helper methods that are not directly invoked by the public API, test the public surface that exercises them rather than testing the helpers directly.

✅ **Good Example:**
```
services/
│── FileHandler.scala          // Contains regions Hashing and Open
services/tests/
│── FileHandlerHashingSpec.scala
│── FileHandlerOpenSpec.scala
```
🚫 **Bad Example:**
```
services/tests/
│── FileHandlerHashingAndOpenSpec.scala // ❌ Merges multiple regions into a single suite
```

---

## **4. Test Name Strings**
Clear test names document the behavior under test and the expected outcome, which helps maintainers understand intent without rereading the implementation. Vague or inconsistent names hide gaps in coverage, encourage multi-purpose tests, and make regression triage harder when a failing test name does not reveal what broke.
- Each test case string **must start with** `"Test_"`.
- Use underscores to separate major phrases in the test name.
- Although suite names stay in PascalCase with no underscores, **test names are encouraged to use underscores** so long names remain readable.
- Test names should **not contain consecutive underscores**.
- If multiple production methods are being tested within the same suite, the test name **must start with** `"Test_<MethodName>_"`.
- If a single production method is being tested in the file, the production method name **must be omitted** from the test name.
- The test name should be **descriptive and readable**, reflecting the behavior under test.
- Favor verbosity over brevity—short names rarely communicate enough context to make the test self-documenting.
- Test names must describe the **specific branch of execution** they cover (e.g. `"Test_ProcessData_ReturnsProcessedValue_WhenInputIsValid"`), not merely state that something "works".
- **Separate tests** into distinct cases rather than combining multiple scenarios in one `test` block.
- **Avoid table-driven tests** unless they significantly improve clarity.
- Any component found in the suite name **must NOT be duplicated in the test name**.

✅ **Good Example:**
```scala
test("Test_ProcessData_ReturnsCorrectValue") {
  // test body
}
```
🚫 **Bad Example:**
```scala
test("processData works") // ❌ Missing Test_ prefix and too vague
test("Test_Process") // ❌ Too vague
test("Test_ProcessData_MultipleCases") // ❌ Tests multiple things at once
```

---

## **5. Test Body Sectioning**
Standardized sections carve complex tests into digestible steps, making it easier to see how inputs flow through stubs and the system under test. Without this structure tests devolve into monolithic blocks where intent, setup, and verification intermingle, obscuring bugs and encouraging brittle copy-paste patterns. The ordering rules below are intentionally strong to build reliable habits—refer to [**Section 17. Breaking the Rules**](#17-breaking-the-rules) for guidance on the rare cases where deviating is justified.
Each test body follows a structured format, **separated by clear comments**:
- Where possible, the order of sections should be as it is presented below.
- Empty sections should be omitted.
- In some cases, it may be justifiable to have multiple instances of specific sections.
- When a single **MOCKING** or **SETUP** block becomes large or involves multiple distinct steps, split it into smaller sub-sections with their own human-readable comment headers (e.g. `// SETUP: Configure HTTP client` and `// SETUP: Initialize environment variables`), so readers can quickly grasp each part.
- Section comment headers should be written as full descriptive sentences (e.g. `// GIVEN: A valid context and request payload`), not terse labels. If a section would contain only a trivial one-liner without setup complexity, you may omit its own comment.

### **Standard Sections:**

The following sections are defined for a well structured test.

1. **GIVEN**
   - Set up initial conditions and inputs.
   - Assign variables with the prefix `given`.
   - When test fixtures are used, they should be allocated in this section.
   - **Test fixture classes** should be named `Fixture*` and assigned to `given*` variables prior to use.
   - The GIVEN section should always be the first section in a test, unless there is nothing to define in GIVEN, in which case it should be omitted.
   - **The GIVEN section must not be merged with any other section.**

1. **MOCKING**
   - Define and configure **stubbed or mocked objects**.
   - Mock variables must be named `mock*`.
   - Prefer dependency injection or lightweight stubs over heavyweight mocking frameworks; when you need a mocking tool, use Mockito-Scala or similar.
   - The MOCKING section should occur before the SETUP section, or be omitted if no mocking is required. Under special circumstances described above, it may be placed after the SETUP section.
   - **The MOCKING section must not be merged with any other section.**
   - **If mocking logic itself has multiple responsibilities (e.g. creating fakes and configuring behaviors), split into sub-headers** (e.g. `// MOCKING: Create stubbed repository` and `// MOCKING: Configure responses`).

1. **SETUP**
   - Perform all test initialization that prepares the non-mocked environment (e.g., creating execution contexts, configuring dependency injection modules, and wiring request parameters).
   - Variables created in this section should be prefixed with `env*` (e.g. `envExecutionContext`, `envAction`).
   - When constants or variables are needed in this section that are important for the overall comprehensibility of the test, declare them in GIVEN instead.
   - The SETUP section should follow the GIVEN section, unless there is nothing to setup in SETUP, in which case it should be omitted.
   - The SETUP section should refer to GIVEN variables, not `mock*` variables, except in rare occasions.
   - Assign mock objects to real interface variables in the SETUP section, using the `env*` prefix (e.g., `envLogger = mockLogger`). The mock variables themselves may only be used in the SETUP and BEHAVIOR sections.
   - **The SETUP section must not be merged with any other section.**
   - **If SETUP involves multiple distinct stages (e.g. reading fixtures, wiring modules, configuring the environment), split into multiple sub-sections with descriptive comments.**

1. **SYSTEM UNDER TEST**
   - Assign the system under test to a value named `sut`.
   - If multiple systems are being tested for some reason, assign them to separate `sut*` values.
   - The SYSTEM UNDER TEST must never directly reference `mock*` variables. It must use the `env*` variables initialized in SETUP.
   - **The SYSTEM UNDER TEST section must not be merged with any other section.**

1. **WHEN**
   - Perform the action or method call being tested.
   - Assign the results to `actual*` variables. This would typically be the return value received from invoking the system under test.
     - If the test requires asserting against other results in the THEN section or BEHAVIOR section, assign those results to their own `actual*` variables after invoking the system under test.
   - If the test needs to use an `env*` in later sections, assign them to `actual*` variables in WHEN and use those instead.
   - **The WHEN section must not be merged with any other section.**

1. **EXPECTATIONS**
   - Define expected values **before assertions**.
   - Expected values must be assigned to `expected*` variables.
   - The EXPECTATIONS section should be strictly placed after WHEN and before THEN.
   - This section should not refer to any `actual*` variables.
   - **The EXPECTATIONS section must not be merged with any other section.**

1. **THEN**
   - Perform **assertions** comparing actual results to expected values.
   - **Never assert against literals**—always use `expected*` variables.
   - **The THEN section must not be merged with any other section.**

1. **LOGGING**
   - Verify any behavior that can be validated via captured logging operations.
   - Capture log output with tools such as Logback appenders or structured logging test fixtures.
   - In unit tests, assert against log entries in the LOGGING section to confirm that the intended execution branch was exercised.
   - **Prefer asserting on emitted logs over (or in addition to) mock verifications** whenever possible: log-based assertions are simpler to write and review, and often replace complex mock setups (e.g., capturing parameters).
   - Each unit test should explicitly verify that the execution branch it is meant to cover is taken, reducing false positives when setup does not exercise the desired branch.

1. **BEHAVIOR**
   - Contains assertions for **mock interactions** (e.g., `verify(mockService).doSomething()` with Mockito-Scala).
   - This section **must be present if mocks are used**.
   - **The BEHAVIOR section must not be merged with any other section.**
   - **All mocks must be created as verifiable**—either using strict mode or by declaring explicit expectations.
   - **Every verifiable setup must be explicitly verified** in this section via `verify(...)` or equivalent. Unverified mocks should cause the test to fail.

✅ **Good Example:**
```scala
final class DataServiceSpec extends AnyFunSuite {
  test("Test_ProcessData_ReturnsCorrectValue") {
    // GIVEN: A simple input payload
    val givenInput = "sample"

    // SYSTEM UNDER TEST: Construct a plain service instance
    val sut = new DataService()

    // WHEN: The input is processed
    val actualResult = sut.processData(givenInput)
    val actualSideEffect = Foo.bar()

    // EXPECTATIONS: The output and side effect are defined
    val expectedResult = "processed_sample"
    val expectedSideEffect = "fubar"

    // THEN: The results match the expectations
    assertEquals(actualResult, expectedResult)
    assertEquals(actualSideEffect, expectedSideEffect)
  }
}
```

🚫 **Bad Example:**
```scala
final class DataServiceSpec extends AnyFunSuite {
  test("Test_ProcessData_ReturnsCorrectValue") {
    // MOCKING: Create a stub
    val mockFoo = mock[Foo]

    // SYSTEM UNDER TEST: Construct a service with the mock
    val sut = new DataService(mockFoo)
  }
}
```

---

## **6. Fixtures: Best Practices**
Reusable fixtures encourage realistic, DRY data that mirrors production scenarios while keeping tests focused on behavior instead of data plumbing. Inlining bespoke objects everywhere invites duplication, increases maintenance costs when models evolve, and makes it harder to reason about how changes cascade across tests.
- **Prefer fixtures over mocks**—favor shared, reusable fixtures rather than creating one-off test data. Fixtures help keep your tests DRY and consistent, and allow them to work on more accurate test data.
- **Instantiate fixtures in the appropriate section** (e.g., GIVEN or SETUP), and modify them as needed to fit the test's needs.
- **Use a sharable fixture factory** instead of creating fixtures or complex test data inline.

✅ **Good Example:**
```scala
object Fixtures {
  def ipAddress(): String = "192.168.1.1"

  def cwConfig(): CwConfig = CwConfig(terminals = List.empty)

  def cwGetTerminalConfigCommand(): CwGetTerminalConfigCommand =
    CwGetTerminalConfigCommand(terminalFixedIpAddress = ipAddress())

  def commandContext(): CommandContext[CwGetTerminalConfigCommand, CwGetTerminalConfigResult] =
    CommandContext(command = cwGetTerminalConfigCommand())
}

final class CarWashesServiceSpec extends AnyFunSuite {
  test("Test_GetTerminalConfig_ReturnsNotFound_WhenTerminalDoesNotExist") {
    // GIVEN: A command context with no terminals configured
    val givenTerminalIp = Fixtures.ipAddress()
    val givenCommandContext = Fixtures.commandContext()
    val givenCarWashesConfig = Fixtures.cwConfig()

    // SYSTEM UNDER TEST: Instantiate the service
    val sut = new CarWashesService()

    // WHEN: Retrieving the terminal configuration
    sut.getTerminalConfig(givenCommandContext)
    val actualStatusCode = givenCommandContext.statusCode
    val actualResult = givenCommandContext.result

    // EXPECTATIONS: The service reports a missing terminal
    val expectedStatusCode = Status.NotFound
    val expectedResult: Option[CwGetTerminalConfigResult] = None

    // THEN: The status and result reflect the missing terminal
    assertEquals(actualStatusCode, expectedStatusCode)
    assertEquals(actualResult, expectedResult)
  }
}
```

🚫 **Bad Example:**
```scala
final class CarWashesServiceSpec extends AnyFunSuite {
  test("Test_GetTerminalConfig_ReturnsNotFound_WhenTerminalDoesNotExist") {
    // GIVEN: Inline construction of data
    val givenTerminalIp = "192.168.1.1"
    val givenCommandContext = CommandContext(
      command = CwGetTerminalConfigCommand(terminalFixedIpAddress = givenTerminalIp)
    )
    val givenCarWashesConfig = CwConfig(terminals = List.empty)
    StaticConfig.carWashesConfig = givenCarWashesConfig

    // SYSTEM UNDER TEST: Instantiate the service
    val sut = new CarWashesService()

    // WHEN: Retrieving the terminal configuration
    sut.getTerminalConfig(givenCommandContext)
```

---

## **7. Imports and Dependencies**
- Keep imports explicit; avoid wildcard imports except for framework-provided syntax helpers (e.g., `org.scalatest.funsuite.AnyFunSuite`).
- Prefer immutable test data—use `val` instead of `var` unless mutation is intrinsic to the behavior under test.
- When working with asynchronous code, use the test framework's async utilities (e.g., `AsyncFunSuite` with `Future`s) and keep the sectioning rules intact.
- For effect systems (ZIO, Cats Effect), rely on test runtimes or `TestRuntime` utilities rather than manual blocking.

---

## **8. Handling Asynchronous Code**
- Use the async variant of your test suite (`AsyncFunSuite`, `AsyncIOSpec`, etc.) when assertions depend on `Future` or effectful results.
- Await results in the WHEN section using the framework's helpers (e.g., `whenReady`, `IOAssertions`) and store them in `actual*` variables.
- Define timeouts explicitly in the WHEN section when using asynchronous assertions to avoid flakiness.
- Keep EXPECTATIONS and THEN sections synchronous—never perform asynchronous work during assertion phases.

---

## **9. Property-Based and Table-Driven Tests**
- Favor explicit, single-scenario tests. Use property-based or table-driven tests only when they genuinely reduce duplication while keeping intent obvious.
- When property-based testing is justified, wrap generators and checkers inside clearly labeled GIVEN and WHEN sections so the structure remains recognizable.
- Ensure each property produces deterministic expectations—random generators must be seeded when the framework allows it.

---

## **10. Test Data Builders and Factories**
- When fixtures become complex, extract builder utilities into `Fixture*` classes or objects that encapsulate default values.
- Builders should expose fluent methods that return new builder instances rather than mutating shared state.
- Prefer using builders inside the GIVEN section to keep tests declarative and avoid leaking setup details into WHEN/THEN sections.

---

## **11. Assertions**
- Use the assertion methods provided by the framework (`assertEquals`, `assert`, `assertThrows`, etc.) to keep failure messages rich and readable.
- Group related assertions together inside the THEN section, separated by blank lines when they validate different concerns.
- For comparing complex structures (e.g., case classes), rely on structural equality rather than checking individual fields unless the test intentionally focuses on one field.

---

## **12. Exception Testing**
- When verifying exceptions, capture them in the WHEN section using helpers such as `intercept`, `assertThrows`, or `try`/`recoverToSucceededIf` for async code.
- Assign the captured exception to an `actual*` variable and perform all assertions in the THEN section.
- Avoid asserting only on the exception type—also validate the message or relevant fields to ensure the correct failure path.

---

## **13. Logging and Telemetry**
- Prefer verifying observable outputs (logs, metrics) over internal state inspection.
- Use appenders or structured logging test helpers to capture logs inside the LOGGING section.
- When validating telemetry (metrics, spans), isolate collectors in the GIVEN section and assert on captured values in THEN or LOGGING.

---

## **14. Mocking and Verification**
- Keep mocking minimal; prefer real implementations or lightweight stubs when practical.
- Configure mock behavior only for the interactions required by the test. Avoid over-configuring default responses.
- Always verify interactions in the BEHAVIOR section, grouping verifications by collaborator to make failures easy to diagnose.

---

## **15. Test Reliability**
- Avoid shared mutable state between tests. Use fresh fixtures per test or leverage framework hooks such as `beforeEach`/`afterEach` to reset state.
- Do not rely on wall-clock time or sleeping to synchronize asynchronous work; use deterministic clocks or manual triggers.
- Keep tests hermetic—no file system, network, or database access unless the test explicitly verifies integration with those resources.

---

## **16. Documentation and Comments**
- Keep comments focused on clarifying intent, not repeating what the code does.
- Document non-obvious decisions (e.g., why a particular mock behavior is required) with inline comments or section headers.
- Remove commented-out code; version control already tracks history.

---

## **17. Breaking the Rules**
While this guide prefers strong conventions, pragmatic exceptions are allowed when they improve clarity or reduce duplication.
- Deviate only after confirming that following the rules would materially harm readability or maintainability.
- When breaking a rule, leave a concise comment explaining the rationale so future readers understand the trade-off.
- Reviewers should challenge deviations that lack explicit justification.

