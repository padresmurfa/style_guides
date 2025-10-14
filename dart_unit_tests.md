# Dart Unit Testing Style Guide

This document translates the rigor of the **C# Unit Testing Style Guide** to the Dart ecosystem. It establishes
**best practices** for writing **clear, structured, and maintainable** unit tests with the [`test` package](https://pub.dev/packages/test)
and related tooling.

---

## **Target Audience**

The primary audience is developers and Large Language Models (LLMs) acting as coding assistants. Following these
rules produces tests that are **predictable, reviewable, and easy to maintain**.

Because these conventions are intentionally strict, humans should treat them as a checklist when reviewing LLM-generated tests
rather than as bureaucracy for hand-written cases. Always review generated tests carefully—the author of the production code
remains responsible for correctness.

---

## **1. Test Directory Organization**
Consistent folder structure and library directives mirror the production package, making it effortless to map a failing test to
its source file. Without that discipline, reviewers must hunt through unrelated folders to find the subject under test.
- Place unit tests inside the package's `test/` directory, mirroring the `lib/` folder structure exactly.
- Every test file must start with `library` (if used) or `part of` directives that match the mirrored package path.
- Create nested folders that mirror the production namespaces—for example `lib/src/services/payment_service.dart`
  should be tested in `test/src/services/payment_service_test.dart`.
- Avoid collapsing unrelated packages or features under shared folders. If the production code lives in separate subpackages,
  reproduce that hierarchy under `test/`.

✅ **Good Example:**
```
lib/
│── src/services/payment_service.dart
└── src/services/refund_service.dart

test/
│── src/services/payment_service_test.dart
└── src/services/refund_service_test.dart
```
🚫 **Bad Example:**
```
test/
│── services_test.dart       // ❌ Combines multiple systems under test
└── misc/payments_test.dart  // ❌ Does not mirror the production folders
```

---

## **2. Test File Naming and Scope**
When file names mirror the system under test (SUT) reviewers can open the correct test file without scanning a directory.
- Each test file must target exactly one production library, class, or public method.
- File names must end in `_test.dart` and include every segment of the production path after `lib/`.
- When focusing on a specific class inside a library, append the class name in snake case before `_test.dart`
  (e.g., `payment_service.payment_processor` → `payment_processor_test.dart`).
- Do not mix tests for unrelated types or methods. Create additional files if the SUT has multiple behaviors worth isolating.

✅ **Good Examples:**
```
payment_service_test.dart
payment_service_payment_processor_test.dart
```
🚫 **Bad Examples:**
```
payment_service_tests.dart        // ❌ Uses pluralized suffix
payment_service_and_refund_test.dart // ❌ Tests multiple systems
```

---

## **3. Group Naming Strategy**
Dart relies on `group()` calls instead of namespaces. Mirroring the production hierarchy keeps deep suites navigable.
- Wrap every test file's contents in a top-level `group()` describing the SUT (e.g., `'PaymentService'`).
- For classes with multiple regions or major responsibilities, create nested `group()` blocks named after the region or
  method being tested (e.g., `'processPayment'`).
- Do not mix multiple production classes in a single `group()`. Create siblings instead.
- Keep group descriptions in PascalCase without spaces when they map directly to class or method names; use spaces only for
  human-readable context (e.g., `'processPayment throws StateError when card is expired'`).

✅ **Good Example:**
```dart
group('PaymentService', () {
  group('processPayment', () {
    // tests...
  });
});
```
🚫 **Bad Example:**
```dart
group('Tests', () {             // ❌ Does not identify the system under test
  group('Happy paths', () {
    // ...mixed behaviors
  });
});
```

---

## **4. Test Description Strings**
Clear test descriptions act as executable documentation. Vague names obscure the behavior and hinder failure triage.
- Each `test()` call must use a description string starting with `Test` to maintain easy grepability (e.g., `'Test returns success when payment is authorized'`).
- Include three clauses—**scenario**, **action**, and **expectation**—separated by spaces or commas.
- Avoid redundant words already expressed by the surrounding `group()` hierarchy.
- Never reuse the same description across multiple `test()` calls within the same file.

✅ **Good Example:**
```dart
test('Test returns success when payment is authorized', () {
  // ...
});
```
🚫 **Bad Example:**
```dart
test('returns success', () {});  // ❌ Missing Test prefix and context
```

---

## **5. Test Method Sectioning**
Structured sections make the flow of data obvious and discourage multi-purpose tests.
- Use `// GIVEN:`, `// MOCKING:`, `// SETUP:`, `// SYSTEM UNDER TEST:`, `// WHEN:`, `// EXPECTATIONS:`, `// THEN:`, and
  `// BEHAVIOR:` comments to delineate sections. These mirror the C# guide and must appear in that order when present.
- Omit empty sections entirely rather than leaving placeholder comments.
- Split complex sections into sub-sections (e.g., `// GIVEN: A persisted customer record`).
- Variable prefixes mirror the C# style guide but follow Dart's lowerCamelCase conventions:
  - GIVEN variables start with `given` (`givenPaymentRequest`).
  - Mock variables start with `mock` (`mockClient`).
  - Environment variables created in SETUP start with `env` (`envRepository`).
  - The system under test is stored in `sut` (or `sutWidget` when multiple systems exist).
  - Outputs captured in WHEN use `actual` prefixes; expectations use `expected` prefixes.
- Multi-line doc comments at the top of each test should summarize the intent using GIVEN/WHEN/THEN language.

✅ **Good Example:**
```dart
/// GIVEN a valid payment request
/// WHEN processPayment is invoked
/// THEN it returns a successful result.
test('Test returns success when payment is authorized', () {
  // GIVEN: A valid payment request
  final givenRequest = PaymentRequest(...);

  // MOCKING: Configure payment gateway
  final mockGateway = MockGateway();
  when(() => mockGateway.charge(any())).thenReturn(Success());

  // SETUP: Wire dependencies
  final envLogger = MockLogger();

  // SYSTEM UNDER TEST: Create PaymentService
  final sut = PaymentService(gateway: mockGateway, logger: envLogger);

  // WHEN: Processing the payment
  final actualResult = sut.processPayment(givenRequest);

  // EXPECTATIONS: Expected success payload
  final expectedStatus = PaymentStatus.success;

  // THEN: Result matches expectation
  expect(actualResult.status, equals(expectedStatus));

  // BEHAVIOR: Gateway charged exactly once
  verify(() => mockGateway.charge(givenRequest)).called(1);
});
```

---

## **6. Test Fixtures and `setUp` Hooks**
Fixtures reduce duplication but must not hide critical setup steps.
- Prefer local helper functions over global state. When reuse is unavoidable, define `setUp` or `setUpAll` blocks inside the
  relevant `group()`.
- `setUp` blocks must assign reusable objects to `given*` variables declared in the outer scope.
- Avoid `late` variables unless initialization is impossible at declaration time; prefer explicit initialization in GIVEN.
- Tear downs should clean only what the test created. Avoid global singletons leaking across tests.

✅ **Good Example:**
```dart
late PaymentRequest givenValidRequest;
late MockGateway mockGateway;

setUp(() {
  // GIVEN: A reusable payment request
  givenValidRequest = PaymentRequest(...);

  // MOCKING: Gateway defaults to success
  mockGateway = MockGateway();
  when(() => mockGateway.charge(any())).thenReturn(Success());
});
```

---

## **7. Mocking Guidelines**
Mocking hides dependencies so tests focus on behavior. Without discipline mocks become brittle and unreadable.
- Use dedicated mocking libraries such as [`mocktail`](https://pub.dev/packages/mocktail) or [`mockito`](https://pub.dev/packages/mockito).
- Mock classes must be named with a `Mock` prefix (`MockGateway`) and declared in the same file when only used there.
- Configure mock expectations inside the **MOCKING** section using `when()`/`thenReturn()` (or equivalent) with `when(() => ...)` syntax.
- Assertions about interactions belong in the **BEHAVIOR** section using `verify()` or `verifyNever()`.
- Avoid partial mocks. Prefer fakes or real implementations when behavior is simple.

---

## **8. System Under Test Construction**
A well-defined SUT boundary keeps tests focused on the intended behavior.
- Instantiate the SUT in the **SYSTEM UNDER TEST** section, using constructor parameters or factory methods only.
- Do not assign mocks directly to the SUT outside of dependency injection points.
- If multiple SUTs are required, use suffixed names (`sutPrimary`, `sutSecondary`). Document why multiple SUTs are needed.

---

## **9. Assertions and Matchers**
Assertions should be explicit and intention-revealing.
- Use `expect(actual, matcher)` with matchers from `package:test/test.dart` (e.g., `equals`, `isTrue`, `throwsStateError`).
- Never compare against raw literals inside `expect`; store them in `expected*` variables first.
- For collections, use dedicated matchers like `containsAll`, `unorderedEquals`, or custom matchers.
- When asserting thrown exceptions, use `expect(() => sut.doThing(), throwsA(isA<StateError>()))` and document the message if relevant.

---

## **10. Async and Stream Tests**
Async behavior is common in Dart, so the suite must be deliberate about it.
- Mark tests `async` and `await` every asynchronous operation. Avoid ignoring returned `Future`s.
- Use `expectLater` for stream assertions, pairing with matchers like `emits`, `emitsInOrder`, and `emitsError`.
- Keep async setup inside GIVEN or SETUP sections. Do not start timers or subscriptions before the WHEN section.
- Ensure `FakeAsync` or `clock` overrides are reset in TEARDOWN or BEHAVIOR sections.

---

## **11. Parameterized and Table-Driven Tests**
While Dart lacks a built-in parameterized API, deliberate structure can still keep table-driven tests readable.
- Prefer multiple `test()` calls over loops when there are only a few permutations.
- When using loops, define a `final cases = [...]` list in GIVEN and iterate with `for (final givenCase in cases)` to create
  nested groups with dynamic descriptions (e.g., `group('Case: ${givenCase.description}', () { ... });`).
- Keep each case's expectations inside its own THEN section; never accumulate assertions across iterations.

---

## **12. Golden and Widget Tests**
Flutter-specific tests have additional constraints.
- Store golden files under `test/goldens/` mirroring the widget path.
- Use `pumpWidget` inside WHEN, and keep pump/pumpAndSettle calls inside WHEN or SETUP depending on context.
- For golden tests, create `expectedGoldenPath` variables and call `expectLater(find.byType(...), matchesGoldenFile(expectedGoldenPath));`.
- Capture screenshots or semantics logs only in THEN sections.

---

## **13. Documentation and Comments**
Readable tests double as documentation.
- Every test must include a doc comment summarizing GIVEN/WHEN/THEN.
- Complex GIVEN or SETUP blocks require inline comments explaining rationale.
- Avoid TODOs in tests; if behavior is pending, skip the test with `skip: 'reason'` and document why.

---

## **14. Linting and Imports**
Consistent imports keep files deterministic.
- Always import `package:test/test.dart` first, followed by third-party packages, and finally relative imports.
- Avoid wildcard exports or `part` files unless mirroring production structure demands it.
- Never wrap imports in try/catch blocks.
- Use `// ignore_for_file` comments sparingly and justify them directly above the ignore.

---

## **15. Breaking the Rules**
Guidelines exist to encourage clarity. Deviations require explicit justification.
- Before breaking a rule, add a comment explaining why the standard structure would harm readability or correctness.
- Prefer extracting helpers or refactoring the SUT over bending the guidelines.
- Document rule breaks in the PR description when they are significant so reviewers understand the intent.

---

Following this style guide ensures Dart unit tests deliver the same clarity and maintainability as their C# counterparts while
remaining idiomatic to the Dart ecosystem.
