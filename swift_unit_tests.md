# Swift Unit Testing Style Guide

This document adapts the **C# Unit Testing Style Guide** for Swift projects that use **XCTest**. The goal is to produce tests that are **clear, structured, and maintainable**, even when they are authored by Large Language Models (LLMs).

---

## **Target Audience**

The primary audience is developers and LLM coding assistants who need to produce Swift unit tests that reviewers can trust. These conventions are intentionally explicit so that generated tests are predictable and easy to audit.

Human authors are encouraged to apply judgment when a guideline does not suit a particular situation. Nevertheless, deviations must be deliberate and accompanied by comments that justify the departure.

---

## **1. Test Target & Module Organization**

Swift packages typically separate production code and tests into distinct targets. Mirroring that structure in files, folders, and namespaces keeps navigation simple.

- Each test file must belong to the test target that corresponds to the production module under test (e.g. `MyLibraryTests` for `MyLibrary`).
- Folder hierarchy inside the test target must mirror the module's namespaces (subdirectories) so maintainers can correlate tests with source quickly.
- When testing a type that lives in a nested module (e.g. `Networking/HTTPClient.swift`), place the test file in `Networking/HTTPClientTests.swift` inside the test target.
- Avoid mixing tests for multiple modules inside a single folder or test target.

✅ **Good Example:**
```text
Sources/
│── Networking/
│   └── HTTPClient.swift
Tests/
│── NetworkingTests/
│   └── HTTPClientTests.swift
```
🚫 **Bad Example:**
```text
Tests/
│── MiscTests.swift   // ❌ Unclear mapping to production source
```

---

## **2. Test Case Class Naming & Files**

Swift XCTest suites are organized into subclasses of `XCTestCase`. Consistent naming keeps the relationship between a test case and its subject explicit.

- Each file must contain a single `final class` that inherits from `XCTestCase`.
- Name the test case `<TypeName>Tests` when it exercises an entire type.
- When focusing on a nested scope—such as an inner enum or method group—name the class `<TypeName><Context>Tests` (e.g. `HTTPClientRetryPolicyTests`).
- Do not place tests for unrelated types in the same file. Create additional files/classes instead.
- File names must match the test class name exactly (e.g. `HTTPClientTests.swift` for `final class HTTPClientTests`).
- Test case classes must not contain underscores in their names.

---

## **3. Organizing Tests with Extensions**

Swift lacks `#region`, but extensions on `XCTestCase` can provide the same focus.

- Use `extension` blocks inside the test file to group tests by production method or behavior.
- Name extensions using `// MARK:` comments (e.g. `// MARK: - execute(request:)`) so navigation tools show the grouping.
- Keep helper functions in a private extension at the bottom of the file. Prefix the `// MARK:` comment with `Helpers`.

✅ **Good Example:**
```swift
final class HTTPClientTests: XCTestCase {
    // Shared state & tearDown/ setUp
}

// MARK: - execute(request:)
extension HTTPClientTests {
    func testExecuteReturnsResponse_whenRequestSucceeds() { /* ... */ }
}

// MARK: - Helpers
private extension HTTPClientTests {
    func makeClient() -> HTTPClient { /* ... */ }
}
```

---

## **4. Test Method Naming**

Explicit method names communicate the branch under test and the expectation.

- Test methods must begin with `test` followed by an uppercase letter (standard XCTest discovery requirement).
- Use descriptive phrases separated by underscores to highlight behavior and condition (e.g. `testExecuteReturnsResponse_whenRequestSucceeds`).
- When all methods in a file target the same production function, omit the function name from the test method to avoid duplication.
- Never use consecutive underscores or trailing underscores.
- Each test should verify exactly one observable behavior. Split multi-purpose tests into separate functions.

✅ **Good Example:**
```swift
func testExecuteReturnsResponse_whenRequestSucceeds() {
    // ...
}
```
🚫 **Bad Examples:**
```swift
func test_execute() { /* ❌ missing expectation */ }
func checkExecute() { /* ❌ missing test prefix */ }
func testExecuteHandlesSuccessAndFailure() { /* ❌ multiple behaviors */ }
```

---

## **5. Required Test Sections**

Reuse the structured sections from the C# guide to keep Swift tests readable. Use comments with full-sentence descriptions to divide each stage. Omit sections that have no content.

1. **GIVEN**
   - Define inputs and fixture state.
   - Prefix variables with `given` (e.g. `let givenRequest = URLRequest(...)`).
   - Allocate fixture factories here (types must be named `Fixture*` when shared).

2. **MOCKING** *(optional)*
   - Configure spies, mocks, or test doubles. Prefix with `mock` or `spy`.
   - When using protocols, wrap doubles in small helper types nested inside the test file.

3. **SETUP** *(optional but common)*
   - Build supporting environment values such as dependencies, storages, or dispatch queues.
   - Prefix variables with `env` to signal environment context (`let envSession = MockURLSession()`).

4. **SYSTEM UNDER TEST**
   - Instantiate the object or function under test and assign it to `sut`. If multiple SUTs are necessary, use `sutPrimary`, `sutSecondary`, etc.
   - Never pass mocks directly—wrap them in `env` variables first to keep this section clean.

5. **WHEN**
   - Perform the action being tested.
   - Assign results and side effects to `actual*` variables (`let actualResponse = try await sut.execute(...)`).
   - When capturing thrown errors, assign them to `actualError` variables via `XCTAssertThrowsError`.

6. **EXPECTATIONS**
   - Define the expected outcomes before assertions.
   - Prefix variables with `expected` and avoid referencing `actual` values in this section.

7. **THEN**
   - Place assertion statements comparing `expected*` to `actual*` values.
   - Do not assert against literals or `given` variables directly; convert them to `expected` values first.

8. **LOGGING / BEHAVIOR** *(if verifying interactions)*
   - Validate messages captured by a logger, notifications, or delegate calls.
   - Interaction assertions for mocks and spies belong here, not in THEN.

Comment headers must be descriptive (e.g. `// GIVEN: A request that succeeds`) rather than generic labels.

---

## **6. Fixtures & Test Data**

Swift's value types make fixtures easy to construct, but shared helpers ensure consistency.

- Define reusable fixtures in nested `enum Fixture` or standalone `struct Fixtures` types within the test target.
- Fixture factory methods must start with verbs like `make` or `create` and return fully-initialized models.
- Assign fixtures to `given*` variables before use. If they represent expected outcomes, copy them into `expected*` variables.
- Prefer immutable fixtures; mutate copies when scenario-specific adjustments are necessary.

✅ **Good Example:**
```swift
enum Fixture {
    static func makeRequest() -> URLRequest { /* ... */ }
    static func makeResponse() -> HTTPURLResponse { /* ... */ }
}

func testExecuteReturnsResponse_whenRequestSucceeds() {
    // GIVEN: a request and a prepared response fixture
    let givenRequest = Fixture.makeRequest()
    let givenResponse = Fixture.makeResponse()
    // ...
}
```

---

## **7. Mocking & Test Doubles**

Swift does not include a standard mocking framework, so you often implement simple doubles manually.

- Use lightweight structs or classes that conform to the dependency protocol and record interactions.
- Name doubles with `Mock`, `Spy`, or `Fake` suffixes and instantiate them in the MOCKING section.
- Expose recorded values or helper assertion methods on the double itself (e.g. `spyLogger.assertLogged(...)`).
- When using third-party mocking frameworks, wrap them in helper types so the test file retains a consistent interface.
- Always verify interactions in the BEHAVIOR section by inspecting the mock/spy, not by relying on implicit expectations.

✅ **Good Example:**
```swift
final class SpyLogger: LoggerProtocol {
    private(set) var messages: [String] = []
    func log(_ message: String) { messages.append(message) }
}
```

---

## **8. Assertions & XCT* Helpers**

Follow strict naming rules so reviewers can distinguish setup from verification immediately.

- Assign `actual*` variables in WHEN and `expected*` variables in EXPECTATIONS. Assertions must reference these variables.
- Use the most specific XCTest assertion available (`XCTAssertEqual`, `XCTAssertTrue`, `XCTAssertNil`, `XCTAssertIdentical`).
- When comparing optionals, prefer `XCTUnwrap` to avoid force unwrapping inside the THEN section.
- For floating-point comparisons, use `XCTAssertEqual(_:accuracy:)` with an `expectedAccuracy` constant declared in EXPECTATIONS.
- Assertions must include custom failure messages when the default output would be ambiguous.

🚫 **Bad Example:**
```swift
XCTAssertEqual(42, sut.value) // ❌ literal instead of expected variable
```

---

## **9. Testing Errors & Throws**

Treat thrown errors as part of the WHEN phase and assert on the captured value.

- Use `XCTAssertThrowsError` or `XCTAssertNoThrow` within the WHEN section.
- Capture the thrown error into an `actualError` variable and assert its type or message in THEN.
- Expected error types must be stored in `expectedErrorType` variables during EXPECTATIONS.

```swift
// WHEN
var actualError: Error?
XCTAssertThrowsError(try sut.execute(givenRequest)) { error in
    actualError = error
}

// EXPECTATIONS
let expectedErrorType = HTTPClient.Error.timeout

// THEN
XCTAssertEqual(expectedErrorType as NSError, actualError as NSError)
```

---

## **10. Async Tests & Expectations**

Modern Swift code is asynchronous; tests must handle suspending functions and callback-based APIs.

- Prefer async tests (`func testFoo() async throws`) when calling async functions. Use `await` directly in WHEN and THEN.
- For callback APIs, use `XCTestExpectation`. Name expectations with `expectation*` prefix and fulfill them inside the callback.
- Expectations must be waited on in the WHEN section using `wait(for:timeout:)`. Store the timeout as an `expectedTimeout` variable in EXPECTATIONS.
- Never rely on `sleep` or polling loops; expectations make intent explicit.

---

## **11. setUp/tearDown Usage**

Shared state must be minimized to keep tests isolated.

- Prefer building fixtures inside each test rather than relying on `setUp()`.
- Use `override func setUp()` only when a dependency is required by nearly every test in the class. Name stored properties with `shared*` prefixes to distinguish them from local `given` variables.
- Reset shared properties in `tearDown()` to avoid test bleed-through.

---

## **12. Helpers & Private API**

- Helper methods must live in a `private` extension at the bottom of the file under `// MARK: - Helpers`.
- Helper names should start with `make`, `build`, or `assert` to make their purpose obvious.
- Helpers must not perform assertions silently; they should return values to be asserted in THEN unless they are dedicated assertion helpers prefixed with `assert`.

---

## **13. Breaking the Rules**

The structure above is intentionally strict. Deviate only when the alternative meaningfully improves clarity.

- Document any deviation with an inline comment explaining why the rule does not apply.
- Never omit GIVEN/WHEN/THEN comments unless the section would otherwise be empty.
- Keep deviations rare so reviewers can trust that missing sections signal unusual behavior worth inspecting.

---

By following these conventions, Swift unit tests maintain the same predictability and readability provided by the original C# style guide while respecting Swift's language idioms and XCTest's tooling.
