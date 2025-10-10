# C++ Unit Testing Style Guide

This document defines **best practices** for writing **clear, structured, and maintainable** unit tests in C++ using GoogleTest, GoogleMock, Catch2, or similar frameworks.

---

## **Target Audience**

The primary audience is developers and Large Language Models (LLMs) acting as coding assistants. Following these guidelines keeps tests **predictable, reviewable, and maintainable**.

These practices can feel verbose for human-authored tests, but they provide an excellent foundation for generated code. **Always review generated tests carefully**—human judgment is indispensable for catching subtle bugs and ensuring intent is captured accurately.

---

## **1. Test Directory and File Organization**

Consistent file structure makes it trivial to locate the subject under test (SUT) and understand how coverage is organized. Disorganized layouts force reviewers to hunt across directories and slow down feedback loops.

- Place all unit tests in a dedicated `tests/` (or similarly named) directory that mirrors the production folder layout.
- Keep a **one-to-one mapping between folders and production namespaces/modules** so moving a file preserves its relative location.
- Do not combine unrelated SUTs in the same directory—create separate folders when production code lives in distinct namespaces or components.

✅ **Good Example:**
```
src/
│── services/
│   └── file_processor.cpp
tests/
│── services/
│   └── file_processor_tests.cpp
```
🚫 **Bad Example:**
```
tests/
│── misc_tests.cpp // ❌ mixes multiple SUTs into one file
```

---

## **2. Test Fixture and File Naming**

Clear naming keeps intent obvious and prevents duplicate coverage from creeping in under vague titles.

- Each test file must focus on a **single class, namespace, or free function group**.
- Name test files using `lower_snake_case` and append `_tests.cpp` (e.g., `file_processor_tests.cpp`).
- Each file should declare one primary test fixture or test suite whose name mirrors the SUT and ends with `Tests` (e.g., `class FileProcessorTests : public ::testing::Test`).
- When a fixture targets a specific portion of a class (such as a region or subsystem), append that portion in PascalCase before the `Tests` suffix (e.g., `FileProcessorHashingTests`).
- Avoid underscores in fixture class names; reserve underscores for macro-based test names.

✅ **Good Examples:**
```cpp
class FileProcessorTests : public ::testing::Test { /* ... */ };
class FileProcessorHashingTests : public ::testing::Test { /* ... */ };
```
🚫 **Bad Examples:**
```cpp
class file_processor_test_fixture : public ::testing::Test {}; // ❌ Wrong casing and suffix
class FileProcessorWorksAsIntended : public ::testing::Test {}; // ❌ Missing Tests suffix
```

---

## **3. Region- or Method-Focused Fixtures**

If the production class is split into clearly labeled regions or exposes large, independent subsystems, mirror that structure in your tests to keep scope tight.

- Introduce a dedicated fixture only when a production class has an obvious region or method grouping that benefits from isolation.
- Keep **one fixture per file**; do not merge unrelated regions into a single fixture.
- When targeting a single method, name the fixture `<ClassName><MethodName>Tests`.
- Region/method fixtures are optional—prefer them when they clarify intent without fragmenting small, cohesive classes.

✅ **Good Layout:**
```
tests/
│── file_processor_tests.cpp              // covers general behavior
│── file_processor_hashing_tests.cpp      // covers #region Hashing
│── file_processor_open_tests.cpp         // covers #region Open
```
🚫 **Bad Layout:**
```
tests/
│── file_processor_hashing_and_open_tests.cpp // ❌ merges unrelated regions
```

---

## **4. Test Case Naming**

Expressive test names document the covered branch and expected outcome. Ambiguous names hide intent and make failures hard to triage.

- GoogleTest-style macros (`TEST`, `TEST_F`, `TEST_P`) must use a **PascalCase suite name** (mirroring the fixture) and a **`Test_`-prefixed test name** (e.g., `TEST(FileProcessorTests, Test_ProcessFile_ReturnsSuccess)`).
- Use underscores to separate major phrases in the test name. Avoid consecutive underscores.
- When the fixture covers multiple production methods, start each test name with `Test_<MethodName>_`.
- When only one production method is under test in the file, omit the method name from individual tests to avoid redundancy.
- Favor detailed names that describe the specific branch or scenario, not general statements like “works”.
- Parameterized test names must also start with `Test_` and include a readable suffix describing the case (e.g., `Test_ParseConfig_RejectsInvalidPort`).

✅ **Good Example:**
```cpp
TEST(FileProcessorTests, Test_ProcessFile_ReturnsSuccess) { /* ... */ }
```
🚫 **Bad Examples:**
```cpp
TEST(FileProcessorTests, Works) {}                   // ❌ Vague
TEST(FileProcessorTests, process_file_success) {}    // ❌ Missing Test_ prefix and casing
TEST(FileProcessorTests, Test__DoubleUnderscore) {}  // ❌ Consecutive underscores
```

---

## **5. Test Method Sectioning**

Standardized sections turn complex tests into readable stories. Without structure, setup and verification blur together, making bugs hard to spot. Follow these rules unless a documented exception truly improves clarity.

Each test function must use **full-sentence comment headers** to delineate sections. Sections appear in the order below; omit any that would be empty. When a section has multiple logical steps, split it into clearly labeled sub-sections (e.g., `// MOCKING: Configure HTTP client`).

1. **GIVEN**
   - Initialize inputs and constants with the prefix `given_`.
   - Allocate reusable fixtures here. Fixture helper classes should be named `Fixture*` and stored in `given_*` variables.
   - Never merge GIVEN with other sections.

2. **MOCKING**
   - Create and configure mocks or fakes; prefix variables with `mock_` or `fake_`.
   - Prefer constructor injection. When using GoogleMock, configure expectations (`EXPECT_CALL`) in this section only if they represent default behavior; verification still happens later.
   - Do not merge with other sections. If mocking has multiple stages, add sub-headers such as `// MOCKING: Configure database stub`.

3. **SETUP**
   - Prepare non-mocked environment objects (e.g., dependency containers, request contexts) using the `env_` prefix.
   - Assign mocks/fakes to production-facing pointers or references here (e.g., `std::shared_ptr<ILogger> env_logger = mock_logger;`).
   - Keep literals that matter for readability in the GIVEN section rather than repeating them in SETUP.
   - Never merge SETUP with other sections.

4. **SYSTEM UNDER TEST**
   - Instantiate the SUT and assign it to a variable named `sut` (or `sut_<suffix>` when multiple SUTs exist).
   - Construct the SUT **only** with `env_` variables—never pass mocks directly.
   - Do not merge with other sections.

5. **WHEN**
   - Invoke the behavior under test and capture results in `actual_` variables.
   - If additional state will be asserted later, copy it into dedicated `actual_` variables here.
   - Never merge with other sections.

6. **EXPECTATIONS**
   - Define expected outcomes in `expected_` variables.
   - Place this section strictly between WHEN and THEN, and do not reference `actual_` values.
   - Never merge with other sections.

7. **THEN**
   - Assert that actual values match expectations. Avoid asserting against raw literals; always reference `expected_` variables.
   - Supply custom failure messages explaining the discrepancy.
   - Do not merge with other sections.

8. **LOGGING** (optional)
   - Verify logged output when relevant, using captured sinks or fakes.
   - Prefer log-based assertions to complex mock verifications when possible.

9. **BEHAVIOR**
   - Verify mock interactions (`EXPECT_CALL(...).Times(...)`, `mock.VerifyAndClearExpectations()` etc.).
   - Required whenever mocks are used.
   - Do not merge with other sections.

✅ **Structured Example:**
```cpp
TEST(FileProcessorTests, Test_ProcessFile_ReturnsExpectedSummary) {
    // GIVEN: an input file with three records
    std::string given_input_path = "test_data/three_records.txt";
    FileSummary given_expected_summary = FixtureFileSummaries::ThreeRecords();

    // MOCKING: simulate a logger that tracks informational messages
    StrictMock<MockLogger> mock_logger;

    // SETUP: expose dependencies through environment variables
    std::shared_ptr<ILogger> env_logger = std::make_shared<MockLoggerAdapter>(&mock_logger);

    // SYSTEM UNDER TEST: construct the processor with env_ dependencies
    FileProcessor sut(env_logger);

    // WHEN: process the input file
    FileSummary actual_summary = sut.ProcessFile(given_input_path);

    // EXPECTATIONS: the summary should match the fixture data
    FileSummary expected_summary = given_expected_summary;

    // THEN: confirm the computed summary equals expectations
    EXPECT_EQ(expected_summary, actual_summary)
        << "ProcessFile should produce the fixture summary for three records.";

    // BEHAVIOR: verify the logger observed the ingestion start message
    EXPECT_TRUE(mock_logger.InfoCalledWith("Started processing three_records.txt"))
        << "Expected informational log for file ingestion.";
}
```

🚫 **Bad Example:**
```cpp
TEST(FileProcessorTests, Test_ProcessFile_ReturnsExpectedSummary) {
    FileProcessor sut;                 // ❌ No section headers
    auto result = sut.ProcessFile();   // ❌ Missing GIVEN/WHEN context
    EXPECT_TRUE(result.ok());          // ❌ Vague assertion without expectations
}
```

---

## **6. Fixtures: Best Practices**

Reusable fixtures keep tests DRY, realistic, and easy to understand. Inlining bespoke data everywhere causes duplication and makes updates error-prone.

- Prefer fixture helpers or builder objects over ad-hoc literals.
- Place reusable fixture factories in dedicated headers or namespaces (e.g., `namespace Fixtures { ... }`).
- Instantiate fixtures in the GIVEN or SETUP section and adjust them locally as needed.
- Share fixtures through helper functions rather than global variables to avoid cross-test coupling.

✅ **Good Example:**
```cpp
namespace Fixtures {
    inline FileSummary ThreeRecordSummary() {
        return FileSummary{/* ... realistic data ... */};
    }
}

TEST(FileProcessorTests, Test_ProcessFile_ReturnsExpectedSummary) {
    // GIVEN: a fixture summary describing three records
    FileSummary given_expected_summary = Fixtures::ThreeRecordSummary();
    // ... remainder omitted ...
}
```

🚫 **Bad Example:**
```cpp
TEST(FileProcessorTests, Test_ProcessFile_ReturnsExpectedSummary) {
    // GIVEN
    FileSummary given_expected_summary{/* dozens of inline fields */}; // ❌ noisy inline data
}
```

---

## **7. Mocking and Test Doubles**

Disciplined mocking keeps tests trustworthy by simulating only true collaborators with explicit expectations. Overusing mocks introduces brittleness and hides missing coverage.

- **Mock only direct dependencies** of the SUT. Collaborators should have their own unit tests.
- **Favor fakes** when mocks become verbose or require intricate matchers. Name fake instances with the `fake_` prefix and build them in the MOCKING section.
- Assign every mock or fake to an `env_` variable in SETUP before passing it to the SUT.
- Configure GoogleMock objects as `StrictMock` or mark expectations with `.Times(...)` so unverified interactions fail fast.
- Place interaction verifications in the BEHAVIOR section. Keep the THEN section focused on result assertions.

### **7.1 Fakes vs. Mocks**

**Fakes** are lightweight implementations authored specifically for the test suite. They shine when GoogleMock expectations would be noisy.

- Use fakes when verification requires complex matchers or callback gymnastics.
- Store captured state inside the fake and expose assertion helpers (e.g., `fake_logger.AssertTrace(...)`).
- Interact with the fake through its `fake_` variable in the THEN, LOGGING, or BEHAVIOR sections.
- Maintain fakes alongside the interfaces they implement to avoid drifting from real behavior.

✅ **Fake-Based Logging Test:**
```cpp
TEST(BranchProcessorTests, Test_ProcessBranch_EmitsTraceLog) {
    // GIVEN: a branch identifier that should trigger logging
    std::string given_branch_id = "bar";

    // MOCKING: provide a fake logger that captures trace entries
    FakeLogger fake_logger;

    // SETUP: expose the fake through env_ variables
    std::shared_ptr<ILogger> env_logger = fake_logger.AsLogger();

    // SYSTEM UNDER TEST
    BranchProcessor sut(env_logger);

    // WHEN: process the branch
    sut.ProcessBranch(given_branch_id);

    // LOGGING: ensure the trace message was emitted
    fake_logger.AssertTrace("Foo", "bar");
}
```

🚫 **Overly Mocked Alternative:**
```cpp
TEST(BranchProcessorTests, Test_ProcessBranch_EmitsTraceLog_WithMock) {
    StrictMock<MockLogger> mock_logger;
    EXPECT_CALL(mock_logger, Log(LogLevel::Trace, testing::_))
        .Times(1); // ❌ brittle matcher and minimal verification

    BranchProcessor sut(&mock_logger); // ❌ passes mock directly, skipping env_ indirection
    sut.ProcessBranch("bar");
}
```

---

## **8. Determinism and Time**

Unit tests must be deterministic. Nondeterministic inputs (current time, random numbers, global state) erode trust and create flaky failures.

- Inject abstractions for time, randomness, and I/O instead of calling globals inside tests.
- Supply controlled fakes or mocks for these abstractions through the SETUP and SYSTEM UNDER TEST sections.
- Refactor production code to separate pure computation from nondeterministic input gathering. Test the pure parts directly.

✅ **Good Example:**
```cpp
class Clock {
public:
    virtual ~Clock() = default;
    virtual std::chrono::system_clock::time_point Now() const = 0;
};

TEST(InvoiceServiceTests, Test_CreateInvoice_UsesProvidedClock) {
    // GIVEN
    std::chrono::system_clock::time_point given_now = /* fixed value */;
    StrictMock<MockClock> mock_clock;
    EXPECT_CALL(mock_clock, Now()).WillOnce(Return(given_now));

    // SETUP
    std::shared_ptr<Clock> env_clock = std::make_shared<MockClockAdapter>(&mock_clock);

    // SYSTEM UNDER TEST
    InvoiceService sut(env_clock);

    // WHEN
    Invoice actual_invoice = sut.CreateInvoice(Order{/*...*/});

    // EXPECTATIONS
    auto expected_generated_at = given_now;

    // THEN
    EXPECT_EQ(expected_generated_at, actual_invoice.GeneratedAt());

    // BEHAVIOR
    testing::Mock::VerifyAndClearExpectations(&mock_clock);
}
```

---

## **9. Documenting Tests with Comments**

Narrated comments explain *why* a test exists, complementing the structured sections. Without them, future maintainers must reverse-engineer intent from mechanics.

- Precede each test fixture with a brief Doxygen comment summarizing the covered behavior.
- Document each test with a Doxygen-style comment describing the scenario and expected outcome.
- Keep section headers as full sentences, not terse labels. Treat them as guidance for readers unfamiliar with the code.

✅ **Good Example:**
```cpp
/// Covers the happy-path processing of FileProcessor.
class FileProcessorTests : public ::testing::Test { /* ... */ };

/// Ensures ProcessFile returns a summary when the file is valid.
TEST(FileProcessorTests, Test_ProcessFile_ReturnsExpectedSummary) {
    // GIVEN: a valid file ...
}
```

---

## **10. SETUP / SUT Dependency Rule**

Reinforce the separation between environment preparation and SUT construction to keep dependency wiring obvious.

- Initialize every dependency in SETUP and store it in an `env_` variable.
- Construct the SUT using only `env_` variables. Passing mocks directly hides intent and complicates reviews.
- When deviating from this rule (e.g., for trivial value objects), document the rationale inline.

---

## **11. Breaking the Rules**

These guidelines are deliberately prescriptive to keep tests predictable. Occasionally, unusual scenarios merit exceptions.

- Deviate only when doing so **improves clarity or determinism** and the standard pattern cannot express the scenario cleanly.
- Whenever you break a rule, **explain the exception with an inline comment** so maintainers understand the rationale.
- Revisit exceptions during refactors; once constraints disappear, revert to the standard structure.

