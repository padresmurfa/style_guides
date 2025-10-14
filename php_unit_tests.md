# PHP Unit Testing Style Guide

This document defines **best practices** for writing **clear, structured, and maintainable** unit tests in PHP using PHPUnit or Pest. It adapts the conventions from the C# guide to fit PHP naming, tooling, and ecosystem norms.

---

## **Target Audience**

The primary target audience for this document is developers and Large Language Models (LLMs) acting as coding assistants. By following these guidelines, tests will be **structured, readable, and maintainable**.

These practices may be too detailed for human-written tests but serve as an excellent guide for generating automated tests with LLM assistance. **Review all test functions carefully**, as human oversight is crucial to ensure test quality.

---

## **1. Test Namespace & Directory Organization**
Consistent namespaces make it obvious where a test lives relative to the production code, so reviewers can locate the subject under test without opening multiple files. Misaligned namespaces cause confusion when navigating between source and tests, especially with PSR-4 autoloaders that rely on directory matching.
- Every test file must declare a namespace that mirrors the production namespace and appends a `\Tests` suffix (e.g. `App\Service` → `App\Service\Tests`).
- When the production code uses nested namespaces, mirror that structure exactly in the test namespace before adding `\Tests`.
- Keep a **one-to-one mapping between folders and namespaces** so moving a file preserves the namespace. Follow PSR-4 directory casing.
- Avoid combining test classes for different production namespaces under a single namespace—create separate namespaces instead.

✅ **Good Example:**
```php
<?php

namespace App\Service\Tests;
```
🚫 **Bad Example:**
```php
<?php

namespace Tests; // ❌ Does not mirror the production namespace
```

---

## **2. Test Class Naming and Files**
Consistent test class names make it immediately obvious which production code a suite exercises, so reviewers can trace failures quickly and spot coverage gaps. Without a naming convention teams waste time hunting for relevant tests, duplicate effort, and risk overlooking scenarios because responsibilities are spread across ambiguously labeled files.
- Each test file must contain a single public test class whose name **ends with** `Test`.
- If a test class exercises a whole class and all of its methods, it must be named `<ClassName>Test`.
- If a test class exercises a single region, method, or behavior, name it `<ClassName><RegionName>Test` or `<ClassName><MethodName>Test`.
- Region-focused and method-focused test classes are **optional** patterns. Prefer them when they make intent clearer, but feel free to keep everything in a single `<ClassName>Test` class when the class-under-test is small enough that splitting would add overhead without clarity.
- Test class names should not contain underscores.
- **Each test class should cover a single class, region, or method under test.**
- **A test class' file name must be of the form `<ClassName>[<RegionName>][<MethodName>]Test.php`**, and every component of the file name must appear in the class name exactly once and in the same order.
- Organize test files into folders that mirror the namespace to keep discovery tools and human navigation aligned.

✅ **Good Examples:**
```php
class FileHandlerTest {}
class FileHandlerHashingTest {}
class FileHandlerOpenTest {}
```
🚫 **Bad Example:**
```php
class FileHandler {} // ❌ Missing Test suffix
class FileHandlerWorksAsIntendedTest {} // ❌ WorksAsIntended is not a plausible #region name
class FileHandlerInitializationAndHashingTest {} // ❌ Combines multiple regions into one class
```

---

## **3. Region-Focused Test Classes**
Regions describe cohesive areas of responsibility inside a production class. Mirroring those regions in the test suite keeps the feedback loop tight by ensuring every region has an intentionally scoped test class. Treat these region-focused classes as a tool rather than a mandate—when a production class only has a handful of regions or methods, duplicating files for each can be overkill.
- Only introduce a `<RegionName>` segment when the production code exposes a `// region` comment, method group, or trait with the same PascalCase name.
- When a class contains multiple regions, create a separate test class per region, following the naming rules above and keeping each class in its own file.
- Do not reuse a `<RegionName>` segment across unrelated production regions; each region must map to a single test class.
- If a region contains helper methods that are not directly invoked by the public API, test the public surface that exercises them rather than testing the region helpers directly.

✅ **Good Example:**
```
src/
│── Service/
│   └── FileHandler.php          // Contains regions Hashing and Open
tests/
│── Service/
│   ├── FileHandlerHashingTest.php
│   └── FileHandlerOpenTest.php
```
🚫 **Bad Example:**
```
tests/
│── Service/
│   └── FileHandlerHashingAndOpenTest.php // ❌ Merges multiple regions into a single test class
```

---

## **4. Test Method Naming**
Clear method names document the behavior under test and the expected outcome, which helps maintainers understand intent without rereading the implementation. Vague or inconsistent names hide gaps in coverage, encourage multi-purpose tests, and make regression triage harder when a failing test name does not reveal what broke.
- Each test method **must start with** `test_`.
- Use underscores to separate major phrases in the test method name.
- Although test classes stay in PascalCase with no underscores, **test methods are encouraged to use underscores** so long names remain readable.
- Test method names should **not contain consecutive underscores**.
- If multiple production methods are being tested within the same test class, the test method **must start with** `test_<MethodName>_`.
- If a single production method is being tested in the file, the production method name **must be omitted** from the test method.
- The method name should be **descriptive and readable**, reflecting the behavior under test.
- Favor verbosity over brevity—short names rarely communicate enough context to make the test self-documenting.
- Test method names must describe the **specific branch of execution** they cover (e.g. `test_processData_returnsBar_whenSuccessful`), not merely state that something "works".
- **Separate tests** into distinct methods rather than combining multiple test cases.
- **Avoid data providers** unless they significantly improve clarity.
- Any component found in the test class name **must NOT be duplicated in the test method name**.

✅ **Good Example:**
```php
public function test_processData_returnsCorrectValue(): void {}
```
🚫 **Bad Example:**
```php
public function testProcessData(): void {} // ❌ Missing test_ prefix
public function test_process(): void {} // ❌ Too vague
public function test_multipleCases(): void {} // ❌ Tests multiple things at once
```

---

## **5. Test Method Sectioning**
Standardized sections carve complex tests into digestible steps, making it easier to see how inputs flow through doubles and the system under test. Without this structure tests devolve into monolithic blocks where intent, setup, and verification intermingle, obscuring bugs and encouraging brittle copy-paste patterns. The ordering rules below are intentionally strong to build reliable habits—refer to [**Section 17. Breaking the Rules**](#-17-breaking-the-rules) for guidance on the rare cases where deviating is justified.
Each test method follows a structured format, **separated by clear comments**:
- Where possible, the order of sections should be as it is presented below.
- Empty sections should be omitted.
- In some cases, it may be justifiable to have multiple instances of specific sections.
- When a single **MOCKING** or **SETUP** block becomes large or involves multiple distinct steps, split it into smaller sub-sections with their own human-readable comment headers (e.g. `// SETUP: Configure HTTP client factory` and `// SETUP: Initialize environment variables`), so readers can quickly grasp each part.
- Section comment headers should be written as full descriptive sentences (e.g. `// GIVEN: A valid context and wash code details`), not terse labels. If a section would contain only a trivial one-line of code without setup complexity, you may omit its own comment.

### **Standard Sections:**

1. **GIVEN**
   - Set up initial conditions and inputs.
   - Assign variables with the prefix `$given`.
   - When test fixtures are used, they should be allocated in this section.
   - **Test Fixture classes** should be named `Fixture*` and assigned to `$given*` variables prior to use.
   - The GIVEN section should always be the first section in a test, unless there is nothing to define in GIVEN, in which case it should be omitted.
   - **The GIVEN section must not be merged with any other section**.

1. **MOCKING**
   - Define and configure **mock objects or test doubles**.
   - Mock variables must be named `$mock*`.
   - Use constructor injection for dependencies, or use PHPUnit mock builders, Prophecy, or Mockery.
   - The MOCKING section should occur before the SETUP section, or be omitted if no mocking is required. Under special circumstances described above, it may be placed after the SETUP section.
   - **The MOCKING section must not be merged with any other section**.
   - **If mocking logic itself has multiple responsibilities (e.g. creating doubles and configuring behaviors), split into sub-headers** (e.g. `// MOCKING: Create mock objects` and `// MOCKING: Configure mock responses`).

1. **SETUP**
   - Perform all test initialization that prepares the non-mocked environment (e.g., creating HTTP contexts, configuring dependency containers, and setting up request parameters).
   - Variables created in this section should be prefixed with `$env` (e.g. `$envHttpContext`, `$envActionContext`).
   - When constants or variables are needed in this section that are important for the overall comprehensibility of the test, they should be declared in variables in the GIVEN section instead.
   - The SETUP section should follow the GIVEN section, unless there is nothing to setup in SETUP, in which case it should be omitted.
   - The SETUP section should refer to `$given*` variables, not `$mock*` variables, except in rare occasions.
   - Assign mock objects to real interface variables in the SETUP section, using the `$env*` prefix (e.g., `$envLogger = $mockLogger->reveal();`). The mock variables themselves may only be used in the SETUP and BEHAVIOR sections.
   - **The SETUP section must not be merged with any other section**.
   - **If SETUP involves multiple distinct stages (e.g. reading fixtures, wiring dependencies, configuring environment), split into multiple sub-sections with descriptive comments.**

1. **SYSTEM UNDER TEST**
   - Assign the system under test to a variable named `$sut`.
   - If multiple systems are being tested for some reason, assign them to separate `$sut*` variables.
   - The SYSTEM UNDER TEST must never directly reference `$mock*` variables. It must use the `$env*` variables initialized in SETUP.
   - **The SYSTEM UNDER TEST section must not be merged with any other section**.

1. **WHEN**
   - Perform the action or method call being tested.
   - Assign the results to `$actual*` variables. This would typically be the return value received from invoking the system under test.
     - If the test requires asserting against other results in the THEN section or BEHAVIOR section, then use the opportunity to assign those results to their own `$actual*` variables, e.g. after invoking the system under test.
   - If the test needs to use an `$env*` in later sections, then assign them to `$actual*` variables in WHEN and use those instead.
   - **The WHEN section must not be merged with any other section**.

1. **EXPECTATIONS**
   - Define expected values **before assertions**.
   - Expected values must be assigned to `$expected*` variables.
   - The EXPECTATIONS section should be strictly placed after WHEN and before THEN.
   - This section should not refer to any `$actual*` variables.
   - **The EXPECTATIONS section must not be merged with any other section**.

1. **THEN**
   - Perform **assertions** comparing actual results to expected values.
   - **Never assert against literals**—always use `$expected*` variables.
   - **The THEN section must not be merged with any other section**.

1. **LOGGING**
   - Verify any behavior that can be validated via captured logging operations.
   - Log the behavior of the code under test at appropriate levels (including trace-level logging with extensive details).
   - In unit tests, assert against log entries in the LOGGING section to confirm that the intended execution branch was exercised.
   - **Prefer asserting on emitted logs over (or in addition to) mock verifications** whenever possible: log-based assertions are simpler to write and review, and often replace complex mock setups (e.g., capturing parameters).
   - Each unit test should explicitly verify that the execution branch it is meant to cover is taken, reducing false positives when setup does not exercise the desired branch.

1. **BEHAVIOR**
   - Contains assertions for **mock interactions** (e.g., `$mockService->expects(self::once())->method('doSomething');`).
   - This section **must be present if mocks are used**.
   - **The BEHAVIOR section must not be merged with any other section**.
   - **All mocks must be created as verifiable**—either using strict expectations in PHPUnit (`$mock->expects(self::once())`) or Mockery's `shouldReceive(...)->once()`.
   - **Every verifiable setup must be explicitly verified** in this section via `$mock->verify()` or framework-specific verification; unverified mocks should cause the test to fail.

✅ **Good Example:**
```php
public function test_processData_returnsCorrectValue(): void
{
    // GIVEN: a sample input
    $givenInput = 'sample';

    // SYSTEM UNDER TEST: create a real service instance
    $sut = new DataService();

    // WHEN: the data is processed
    $actualResult = $sut->processData($givenInput);

    // EXPECTATIONS: define the expected output
    $expectedResult = 'processed_sample';

    // THEN: compare expected and actual results
    self::assertSame($expectedResult, $actualResult);
}
```
🚫 **Bad Example:**
```php
public function test_processData_returnsCorrectValue(): void
{
    // MOCKING
    $mockFoo = $this->createMock(Foo::class);

    // SYSTEM UNDER TEST
    $sut = new DataService($mockFoo); // ❌ Passing the mock directly without SETUP
}
```

---

## **6. Fixtures: Best Practices**
Reusable fixtures encourage realistic, DRY data that mirrors production scenarios while keeping tests focused on behavior instead of data plumbing. Inlining bespoke objects everywhere invites duplication, increases maintenance costs when models evolve, and makes it harder to reason about how changes cascade across tests.
- **Prefer fixtures over mocks**—favor shared, reusable fixtures rather than creating one-off test data. Fixtures help keep your tests DRY and consistent, and allow them to work on more accurate test data.
- **Instantiate fixtures in the appropriate section** e.g. GIVEN or SETUP, and modify it as needed to fit the test's needs.
- **Use a sharable fixture factory** instead of creating fixtures or complex test data inline.

✅ **Good Example:**
```php
final class Fixtures
{
    public static function ipAddress(): string
    {
        return '192.168.1.1';
    }

    public static function config(): CwConfig
    {
        return new CwConfig(['terminals' => []]);
    }

    public static function getTerminalConfigCommand(): CwGetTerminalConfigCommand
    {
        $givenIpAddress = self::ipAddress();

        return new CwGetTerminalConfigCommand(['terminal_fixed_ip_address' => $givenIpAddress]);
    }

    public static function commandContext(): CommandContext
    {
        return new CommandContext(self::getTerminalConfigCommand());
    }
}

final class CarWashesServiceTest extends TestCase
{
    public function test_getTerminalConfig_returnsNotFound_whenTerminalDoesNotExist(): void
    {
        // GIVEN: an empty terminal configuration
        $givenCommandContext = Fixtures::commandContext();
        $givenCarWashesConfig = Fixtures::config();

        // SYSTEM UNDER TEST: instantiate the service
        $sut = new CarWashesService();

        // WHEN: retrieving terminal configuration
        $sut->getTerminalConfig($givenCommandContext);
        $actualStatusCode = $givenCommandContext->statusCode();
        $actualResult = $givenCommandContext->result();

        // EXPECTATIONS: expect a Not Found response
        $expectedStatusCode = Response::HTTP_NOT_FOUND;
        $expectedResult = null;

        // THEN: ensure the context reflects a missing terminal
        self::assertSame($expectedStatusCode, $actualStatusCode);
        self::assertSame($expectedResult, $actualResult);
    }
}
```
🚫 **Bad Example:**
```php
public function test_getTerminalConfig_returnsNotFound_whenTerminalDoesNotExist(): void
{
    // GIVEN
    $givenCommandContext = new CommandContext(new CwGetTerminalConfigCommand(['terminal_fixed_ip_address' => '192.168.1.1']));
    $givenCarWashesConfig = new CwConfig(['terminals' => []]);
    StaticConfig::$carWashesConfig = $givenCarWashesConfig;

    // SYSTEM UNDER TEST
    $sut = new CarWashesService();

    // WHEN
    $sut->getTerminalConfig($givenCommandContext);
    $actualStatusCode = $givenCommandContext->statusCode();
    $actualResult = $givenCommandContext->result();

    // EXPECTATIONS
    $expectedStatusCode = Response::HTTP_NOT_FOUND;
    $expectedResult = null;

    // THEN
    self::assertSame($expectedStatusCode, $actualStatusCode);
    self::assertSame($expectedResult, $actualResult);
}
```

---

## **7. Mocking: Best Practices**
Disciplined mocking keeps tests readable and trustworthy by ensuring only true collaborators are simulated and their expectations are explicit. Unrestrained mocks create brittle, implementation-driven tests that break on harmless refactors, mask missing coverage of dependencies, and clutter SUT construction with framework boilerplate.
- **Never mock what you don't have to**—prefer fixtures or real instances where practical.
- **Only mock things that the system-under-test uses directly**—this ensures that your test exercises the SUT properly, without falling into the trap of combinatorial execution path counts as call-depth increases.
- The systems that your SUT uses directly should be covered by their own direct unit tests, not by the unit tests of your SUT.
- **Use PHPUnit mocks, Prophecy, or Mockery** for dependency injection and verify interactions explicitly.
- **Assertions on mock interactions go in the BEHAVIOR section, not in the THEN section**.
- **Mocks must never be passed directly to the system under test.** Instead, assign mocks to interface-conforming `$env*` variables in SETUP and pass those to the SUT.
- Following this pattern keeps the SUT construction readable: the MOCKING section owns the intricate setup, the SETUP section exposes the ready-to-use interfaces via `$env*` variables, and the SYSTEM UNDER TEST section reads like a clean recipe that highlights only the essential collaborators. Without the intermediate `$env*` assignments the constructor or method invocation under test quickly devolves into a wall of `->reveal()` or `Mockery::mock()` calls, making it difficult for reviewers to decipher which dependency is which.

### **7.1 Test Fakes (a.k.a. Dummies)**
Test fakes are lightweight, purpose-built implementations of an interface that you author specifically for the test suite. They shine when mocking frameworks become gnarly—especially for complex interaction verification or structured log inspection—because you can write direct, intention-revealing code without wrestling with callback signatures or argument matchers.

- **Create fakes when setup or verification with a mock would be noisy, repetitive, or brittle.** If verifying interactions through a mocking framework requires callbacks, reflection, or deeply nested matchers, a fake is often clearer.
- **Name fake instances with the `$fake*` prefix and construct them in the MOCKING section.** This keeps parity with mock naming so readers instantly recognize simulated collaborators.
- **Assign each `$fake*` variable to an `$env*` variable in the SETUP section before handing it to the SUT.** Treat a fake like any other dependency so the SYSTEM UNDER TEST section stays clean and only references `$env*` collaborators.
- **Interact with the fake via its `$fake*` variable in the THEN, LOGGING, and BEHAVIOR sections.** Fakes frequently expose custom assertion helpers (e.g., `$fakeLogger->assertBranchProcessed(...)`) that capture behavior without expectation boilerplate.
- **Document expectations inside the fake when possible.** Purpose-built helpers (such as storing structured log entries) make intent obvious and reduce duplicated parsing logic across tests.

#### Strengths Compared to Mocks
- **Readable behavior verification.** Fakes encapsulate the verification logic in methods that read like English, avoiding the visual clutter of `expects()` chains.
- **Reduced setup friction.** Because you own the implementation, you can build constructors and helper methods that mirror your domain vocabulary, instead of contorting the SUT to satisfy framework APIs.
- **Deterministic assertions.** Fakes can capture state (e.g., recorded log entries) and expose first-class assertions, lowering the risk of brittle, order-dependent mock verifications.

#### Weaknesses Compared to Mocks
- **Maintenance cost.** You must maintain the fake implementation alongside production interfaces. If the interface evolves, the fake must be updated manually.
- **Limited behavioral coverage.** A fake typically encodes only the paths needed by the current test suite. Mocks can dynamically configure behaviors per test without editing shared code.
- **Risk of drifting from reality.** Because a fake is handwritten, it might omit subtle behavior the real dependency exhibits. Use fixtures and integration tests to guard against divergence.

#### Example: Logging with a Fake vs. a Mock
Consider a service that logs a structured trace message—`"Foo(branch=bar): the foo is quite barred"`—whenever it processes a branch.

**✅ Fake-based test (clean and intention revealing):**
```php
public function test_processBranch_logsTraceMessage(): void
{
    // GIVEN: a branch identifier that should trigger logging
    $givenBranchId = 'bar';

    // MOCKING: create test fakes for collaborators
    $fakeLogger = new FakeLogger();

    // SETUP: expose the fake through env* variables
    $envLogger = $fakeLogger;

    // SYSTEM UNDER TEST: inject the fake logger
    $sut = new BranchProcessor($envLogger);

    // WHEN: process the branch
    $sut->processBranch($givenBranchId);

    // LOGGING: ensure the trace message was written
    $fakeLogger->assertExercised('Foo', 'bar');
}

final class FakeLogger implements LoggerInterface
{
    private array $entries = [];

    public function emergency($message, array $context = []): void {}
    public function alert($message, array $context = []): void {}
    public function critical($message, array $context = []): void {}
    public function error($message, array $context = []): void {}
    public function warning($message, array $context = []): void {}
    public function notice($message, array $context = []): void {}
    public function info($message, array $context = []): void {}

    public function debug($message, array $context = []): void
    {
        $this->entries[] = ['category' => 'Foo', 'branch' => $context['branch'] ?? null, 'message' => $message];
    }

    public function log($level, $message, array $context = []): void
    {
        if ($level === LogLevel::DEBUG) {
            $this->debug($message, $context);
        }
    }

    public function assertExercised(string $expectedCategory, string $expectedBranch): void
    {
        \PHPUnit\Framework\Assert::assertCount(1, $this->entries);
        $entry = $this->entries[0];
        \PHPUnit\Framework\Assert::assertSame($expectedCategory, $entry['category']);
        \PHPUnit\Framework\Assert::assertSame($expectedBranch, $entry['branch']);
        \PHPUnit\Framework\Assert::assertSame("Foo(branch={$expectedBranch}): the foo is quite barred", $entry['message']);
    }
}
```

- **All mocks must be created in verifiable mode**:
  - Use `$mock = $this->getMockBuilder(Dependency::class)->onlyMethods(['performTask'])->getMock();` and pair with `$mock->expects(self::once());`, _or_
  - For Mockery, call `shouldReceive(...)->once()` and `Mockery::close()` in `tearDown()`.
- **Every mock setup must be verified in the BEHAVIOR section** via `$mock->expects(...)` or dedicated verification helpers; unverified verifiable mocks should fail the test.

✅ **Good Example:**
```php
public function test_serviceCallsDependency(): void
{
    // MOCKING: create a strict dependency mock
    $mockDependency = $this->createMock(DependencyInterface::class);
    $mockDependency->expects(self::once())->method('performTask');

    // SETUP: expose the mock through the dependency interface
    $envDependency = $mockDependency;

    // SYSTEM UNDER TEST
    $sut = new MyService($envDependency);

    // WHEN
    $sut->doWork();

    // BEHAVIOR
    // PHPUnit handles verification automatically via expects(self::once())
}
```
🚫 **Bad Example:**
```php
public function test_serviceCallsDependency(): void
{
    $mockDependency = $this->createMock(DependencyInterface::class);  // ❌ No section-separators
    $service = new MyService($mockDependency); // ❌ Passing the mock object directly to the SUT without env* indirection
    $service->doWork();
}
```

---

## **8. Assertions & Variable Naming**
Strict naming patterns for expected and actual values highlight the difference between inputs, outputs, and verifications, which speeds up failure analysis. Mixing literals and setup variables inside assertions hides intent, makes diffs noisy, and increases the chance of asserting against the wrong data.
- **Expected values** always assign input values (from the GIVEN section) to `$expected*` variables in the EXPECTATIONS section if you intend to assert on them in the THEN or BEHAVIOR sections.
- **Actual results** must be assigned to `$actual*` variables in the WHEN section.
- **Never assert against literals directly** — use `$expected*` variables.
- **Never assert against `$given*` variables directly** — assign them to `$expected*` variables in the EXPECTATIONS section and use those.
- **Never assert against `$env*` variables directly** — assign them to `$actual*` variables exclusively in the WHEN section and use those.
- **Never assert against `$mock*` variables directly in the THEN section** — assign them to `$expected*` variables in the EXPECTATIONS section and use those.
- **It is acceptable to assert against `$mock*` variables directly in the BEHAVIOR section**.
- To reiterate: **Assertions should never refer to `$given*` or `$env*` values, and should only refer to `$mock*` variables in the BEHAVIOR section**.

✅ **Good Example:**
```php
$expectedResult = 42;
self::assertSame($expectedResult, $actualResult);
```
🚫 **Bad Example:**
```php
self::assertSame(42, $actualResult); // ❌ No expected variable
```

---

## **9. Exception Handling (`expectException`)**
Treating exception capture as part of the WHEN stage keeps control flow explicit and prevents assertions from being buried inside callbacks. When the thrown exception is not stored and verified deliberately, tests can pass for the wrong reason, masking regressions where different error types or messages are emitted.

When using PHPUnit's `expectException`/`expectExceptionMessage` or wrapping calls in try/catch blocks, assign the actual exception to a local variable and assert that it is as expected in a subsequent THEN section.

✅ **Good Example:**
```php
public function test_divideByZero_throwsException(): void
{
    // GIVEN
    $givenNumerator = 10;
    $givenDenominator = 0;

    // WHEN
    $actualException = null;
    try {
        Math::divide($givenNumerator, $givenDenominator);
    } catch (DivideByZeroException $exception) {
        $actualException = $exception;
    }

    // EXPECTATIONS
    $expectedExceptionClass = DivideByZeroException::class;

    // THEN
    self::assertInstanceOf($expectedExceptionClass, $actualException);
}
```

---

## **10. Using `setUp()` Methods**
Being intentional about `setUp()` usage ensures shared initialization is truly common while test-specific context stays near the scenario. Overusing setup hooks leads to hidden coupling between tests, complicated fixture state, and surprises when one case mutates shared members that another silently depends on.
- **Avoid `setUp()`**. Favor using sharable fixtures, declared in a static fixture factory class, instead.
- Use `setUp()` only for **repeated initialization** that truly applies to every test case in the class.

✅ **Good Example:**
```php
protected function setUp(): void
{
    parent::setUp();
    $this->service = new MyService();
}
```
🚫 **Bad Example:**
```php
protected function setUp(): void
{
    $service = new MyService(); // ❌ Local variable discarded; each test still constructs its own instance
}
```

---

## **11. Organizing Tests in Namespaces**
Mirroring production namespaces in the test suite keeps navigation intuitive and tooling-friendly, so developers can jump between code and tests effortlessly. When namespace structures drift, IDE search results become noisy, automated discovery may misbehave, and contributors struggle to find the right home for new cases.
- **Group related test classes into namespaces**.
- Ensure **each namespace matches the SUT structure**.
- Use the brace-free namespace declaration form (`namespace App\Service\Tests;`).

✅ **Good Example:**
```php
namespace App\Service\Tests;

final class UserServiceTest extends TestCase { /* ... */ }
```
🚫 **Bad Example:**
```php
namespace ServiceTests; // ❌ Inconsistent naming

final class UserServiceTest extends TestCase { /* ... */ }
```
🚫 **Bad Example:**
```php
namespace App\Service\Tests // ❌ Missing semicolon terminator
{
    final class UserServiceTest extends TestCase { /* ... */ }
}
```

---

## **12. Managing Copy-Paste Setup**
Copy-paste programming is often the most readable choice for test setup. Start with a "happy path" test that spells out the full arrangement, and copy that scaffolding into edge-case tests, tweaking only what changes for each branch. With disciplined sectioning and modern refactoring tools, duplicating the setup is usually quicker and clearer than inventing cryptic helper methods whose behavior must be reverse-engineered.
- Prefer duplication first, then refactor **only when** the abstraction makes the setup easier to understand than the copied code it replaces.
- When a shared helper truly improves clarity, keep it near the tests that use it and document which scenarios rely on it so future contributors know when to extend or bypass it.
- When you do copy setup code, call out the variations in the section-header comments (e.g. `// GIVEN: user has insufficient balance`). These comments act as signposts so reviewers can immediately see why one test diverges from another despite similar bodies.

---

## **13. Deterministic Tests and Dependency Injection**
Unit tests should be deterministic: the same inputs must produce the same results every run. Nondeterminism from global state, random values, or "current time" APIs usually erodes trust in the suite. Occasionally, allowing nondeterminism in aspects that **should not** affect the outcome is valuable—flaky failures in those cases expose hidden coupling or misunderstood behavior. Treat such flakes as signals to either fix the bug they reveal or document the learning and firm up the contract under test.
- Isolate side effects behind interfaces and inject them into the system under test. Use dependency injection (DI) so the test can supply controlled fakes, especially for time-sensitive behavior.
- Avoid calling `new \DateTimeImmutable('now')`, `random_int()`, or static singletons from production code during tests. Instead, inject a clock, ID generator, or configuration provider that the test can stub.
- For time-sensitive logic, pass a `ClockInterface` or similar abstraction. Tests can freeze "now" to a predictable value while production supplies a real clock, making the deterministic path explicit.
- When code mixes deterministic calculations with nondeterministic reads, refactor into two methods: one that gathers inputs (time, randomness, HTTP responses) and another that performs pure computation. Unit test the deterministic method directly and cover the integration path separately if needed.

✅ **Example: controlling current time with DI**
```php
interface ClockInterface
{
    public function now(): \DateTimeImmutable;
}

final class InvoiceService
{
    public function __construct(private ClockInterface $clock) {}

    public function createInvoice(Order $order): Invoice
    {
        $generatedAt = $this->clock->now();

        return new Invoice($order->id(), $generatedAt);
    }
}

public function test_createInvoice_usesProvidedClock(): void
{
    // GIVEN
    $givenNow = new \DateTimeImmutable('2024-02-01T09:30:00Z');
    $mockClock = $this->createMock(ClockInterface::class);
    $mockClock->expects(self::once())->method('now')->willReturn($givenNow);

    // SETUP
    $envClock = $mockClock;

    // SYSTEM UNDER TEST
    $sut = new InvoiceService($envClock);

    // WHEN
    $actualInvoice = $sut->createInvoice(new Order(Uuid::fromString('00000000-0000-0000-0000-000000000000')));

    // EXPECTATIONS
    $expectedGeneratedAt = $givenNow;

    // THEN
    self::assertEquals($expectedGeneratedAt, $actualInvoice->generatedAt());

    // BEHAVIOR
    // PHPUnit verifies the expectation via expects(self::once())
}
```

---

## **14. Using comments in tests**
Documenting test intent with PHPDoc comments provides high-level context that complements the structured sections, guiding readers through why a scenario exists. Without these explanations, future maintainers must infer purpose from mechanics alone, risking redundant cases or accidental deletion of critical coverage.

- **Each test method should be commented with a human-readable explanation of what the test is exercising**.
- **Each test class should be commented with a human-readable explanation of what the test class is exercising**.
- **Use PHPDoc (`/** ... */`) to document test classes and test methods**.

---

## **15. Increasing test clarity with section comments**
Narrated section headers transform dense setup or verification code into self-explanatory stories that highlight intent and edge cases. Skipping these comments forces reviewers to reverse-engineer the reason behind each block, slowing code reviews and making it easier for subtle regressions to slip through.

**As a best practice**, when tests have complex parts such as setting up mocks, creating complex `$given` objects, asserting on mock behavior, and so forth, it is recommended to split each such part into a section of its own, and comment what that section is doing. These headers act like a decryption key for the reader: each full-sentence comment explicitly states the intent of the code beneath it, so that even when the implementation is dense or unfamiliar, the reader can immediately understand _why_ a block exists rather than mentally reverse-engineering the setup from the statements themselves.

✅ **Good Example:**
```php
namespace App\Service\Tests;

/**
 * Covers the "Bar" region of the Foo\User\Service class.
 */
final class UserServiceTest extends TestCase
{
    /**
     * Covers the case where FooBar equals "foo".
     */
    public function test_fooBar_isFoo(): void
    {
        // GIVEN: LogLevel::INFO is enabled
        $givenInfoLevelEnabled = true;

        // MOCKING: create a mocked logger that has info level enabled
        $mockLogger = $this->createMock(LoggerInterface::class);
        $mockLogger->method('info')->with(self::callback(
            static fn (string $message): bool => str_contains($message, 'IsFoo was called')
        ));

        // SETUP: hide the fact that we're using a mocked logger
        $envLogger = $mockLogger;

        // SYSTEM UNDER TEST: instantiate and initialize the system under test
        $sut = new UserService($envLogger);

        // WHEN: exercise the test case
        $actualResult = $sut->isFoo();

        // EXPECTATION: we expect the result of isFoo to be true
        $expectedResult = true;

        // THEN: Verify that the actual result matches expectations.
        self::assertSame($expectedResult, $actualResult);

        // BEHAVIOR: we expect isFoo to have logged a message on info level
        // PHPUnit verifies this via the configured callback.
    }
}
```
🚫 **Bad Example:**
```php
namespace App\Service\Tests;

final class UserServiceTest extends TestCase
{
    public function test_fooBar_isFoo(): void
    {
    }
}
```

---

## **16. SETUP / SUT Dependency Rule**
Reinforcing the separation between SETUP dependencies and the system under test protects against leaky abstractions and keeps constructor wiring transparent. When mocks or fixtures sneak directly into SUT creation, the test intent becomes opaque, accidental coupling increases, and regressions slip in because collaborators are no longer clearly expressed through `$env*` variables.

To reiterate:

- The **SETUP** section must initialize all dependencies passed to the **SUT** via `$env*` variables.
- The **SYSTEM UNDER TEST** section must construct the **SUT** using only `$env*` variables.
- Mocks must never be used directly in the **SYSTEM UNDER TEST** section.

---

## **17. Breaking the Rules**
The guidelines above are intentionally prescriptive so tests remain predictable and reviewable. However, real-world systems occasionally demand exceptions.
- Deviation is acceptable when it **improves clarity or determinism** for a specific edge case that the standard pattern cannot express cleanly.
- When you break a rule, **document the rationale inline** (e.g. with a comment or PHPDoc) so future maintainers understand why the exception exists.
- Re-evaluate exceptions periodically. If the surrounding production code changes, the original constraint may disappear and the standard structure can be restored.

Common shortcuts you may encounter include:
- **Assigning captured values directly to `actual*` variables or hosting them in a `WHEN` section before `MOCKING`.** This keeps the structure short but hides the intent that `CAPTURED` makes explicit. Use it sparingly and only when the setup is trivial.
- **Declaring captured variables in the same `MOCKING` or `SETUP` section as their scaffolding.** It works, yet blends responsibilities and makes it harder to see which lines exist solely to capture output.
- **Skipping section comments for tiny operations.** A lone variable assignment or simple assertion can speak for itself, but be sure the omission truly reduces noise instead of forcing the reader to infer structure.
- **Dropping the `EXPECTATIONS` section and asserting directly on `given*` variables in `THEN`.** This saves a couple of lines, yet sacrifices the symmetry that helps reviewers scan intent. Prefer keeping `EXPECTATIONS`, especially because an AI assistant can generate it automatically.
- **Asserting on the SUT output without first assigning sub-components to `actual*` variables.** Inline property access or array indexing is quicker to type, but intermediate variables make it obvious what the test is examining.
- **Passing `mock*` variables straight into the SUT without promoting them to `env*` variables.** The shortcut obscures which collaborators the SUT receives and blurs the mock-versus-environment boundary, particularly with frameworks that require `->reveal()`/`->object` syntax.

While humans occasionally bend these rules to communicate intent efficiently, AI tools should rarely do so. Automated authorship can produce the full structured form with minimal effort, so generated tests are expected to adhere strictly to the conventions above.

