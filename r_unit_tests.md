# R Unit Testing Style Guide

This document defines **best practices** for writing **clear, structured, and maintainable** unit tests in R using [`testthat`](https://testthat.r-lib.org/).

---

## **Target Audience**

The primary audience for this guide is developers and Large Language Models (LLMs) acting as coding assistants. Following these guidelines keeps `testthat` suites **predictable, readable, and maintainable**, especially when many contributors generate tests programmatically.

Human reviewers should still verify every generated test. The strict conventions here are intentionally opinionated so automated tools can produce high-quality tests that align with mature R package workflows.

---

## **1. Test File Organization**
Consistent file structure makes it easy to map a test to the function it covers and keeps `devtools::test()` output predictable.
- Place all tests inside `tests/testthat/`.
- Name each file `test-<object-under-test>.R` using lowercase words separated by hyphens (e.g., `test-utils-parse-date.R`).
- Scope each file to **one exported function, S3/S4 method, or R6 class**. If a file grows beyond ~200 lines, consider splitting by behavior group (e.g., `test-foo-validate.R`, `test-foo-serialize.R`).
- Keep helper utilities in `tests/testthat/helper-*.R` and avoid placing assertions there.
- Mirror the package's subdirectory layout when helpful (e.g., tests for `R/utils/format.R` live in `tests/testthat/test-utils-format.R`).

✅ **Good Example:**
```
tests/
│── testthat/
│   │── helper-datetime.R
│   │── test-utils-parse-date.R
│   │── test-services-fetch-data.R
```
🚫 **Bad Example:**
```
tests/
│── parse_date_tests.R    # ❌ Missing test- prefix and lives outside testthat/
│── mega-tests.R          # ❌ Mixed responsibilities
```

---

## **2. Test Context Blocks**
`test_that()` descriptions serve as the primary documentation for a test's intent. Treat them as structured sentences that highlight the scenario and expected outcome.
- Each `test_that()` description must follow `"<function> <condition> <expected outcome>"` (e.g., `"parse_date returns NA on invalid input"`).
- Start descriptions with the function or method name under test; omit redundant prefixes like "function" or "method".
- Use lowercase words; reserve uppercase for constants or acronyms.
- Keep descriptions under 80 characters so they display cleanly in test reports. Split behavior into multiple `test_that()` calls instead of writing one verbose description.
- Avoid copy-pasting identical descriptions—uniqueness matters for debugging.

✅ **Good Example:**
```r
test_that("parse_date returns Date on ISO strings", {
  ...
})
```
🚫 **Bad Example:**
```r
test_that("works", {
  ...
})
```

---

## **3. Test Function Structure**
Replicate the disciplined, sectioned layout from the C# guide inside each `test_that()` block to clarify the flow.
- Divide the body into annotated sections using uppercase comments: `# GIVEN`, `# MOCKING`, `# SETUP`, `# SYSTEM UNDER TEST`, `# WHEN`, `# EXPECTATIONS`, `# THEN`, `# LOGGING`, and `# BEHAVIOR`.
- Use only the sections that apply; omit empty ones. When a section becomes lengthy, add descriptive sub-headers (e.g., `# SETUP — create temporary files`).
- Each variable name must indicate its purpose:
  - Inputs prefixed with `given_`.
  - Mock objects prefixed with `mock_`.
  - Environment/setup objects prefixed with `env_` (e.g., `env_temp_dir`).
  - System under test assigned to `sut` (or `sut_<role>` when multiple objects exist).
  - Results captured in `actual_` variables.
  - Expected values stored in `expected_` variables.
- Do not mix responsibilities across sections. For example, never create the system under test inside `# GIVEN`.
- Always include `# THEN` when assertions exist. All expectations must use `expected_` variables instead of literals.

✅ **Good Example:**
```r
test_that("parse_date returns Date on ISO strings", {
  # GIVEN — a canonical ISO-8601 string
  given_input <- "2024-01-31"

  # SYSTEM UNDER TEST — parser bound to default locale
  sut <- parse_date

  # WHEN — convert the input
  actual_result <- sut(given_input)

  # EXPECTATIONS — expected Date value
  expected_result <- as.Date("2024-01-31")

  # THEN — assert equality using testthat helpers
  expect_identical(actual_result, expected_result)
})
```
🚫 **Bad Example:**
```r
test_that("parse_date works", {
  expect_equal(parse_date("2024-01-31"), as.Date("2024-01-31"))  # ❌ No sections, literal in assertion
})
```

---

## **4. Handling Side Effects and State**
R tests frequently manipulate global options, environments, and files. Keep side effects isolated.
- Use `withr::local_*()` helpers inside the relevant section (typically `# SETUP`) and assign the handle to an `env_` variable when you need to inspect it later.
- Temporary files and directories should be created via `withr::local_tempfile()` / `withr::local_tempdir()`.
- When modifying package options, set them inside `# SETUP` and ensure cleanup occurs automatically via `withr` locals.
- Prefer working with explicit environments over relying on the global environment. Assign them to `env_` variables.

---

## **5. Mocking and Fakes**
`testthat` does not ship a dedicated mocking framework, so adopt consistent patterns when simulating collaborators.
- Favor pure functions and dependency injection; inject collaborators into the `sut` inside `# SYSTEM UNDER TEST`.
- For functions, use `withr::local_mocked_bindings()` or `mockery::stub()` inside `# MOCKING`. Store replacements in `mock_` variables and verify them later.
- When verifying mock interactions, add a `# BEHAVIOR` section with explicit expectations (e.g., counters stored in `mock_calls$count`).
- Keep mocks strict. If a mock counts invocations, assert the expected call count in `# BEHAVIOR`. Do not leave verifications implicit.

---

## **6. Expectations and Assertions**
Consistency in expectations prevents tests from becoming brittle or ambiguous.
- Use `expect_identical()` for exact matches, `expect_equal()` for numerics with tolerance, and `expect_true()`/`expect_false()` for logical outcomes.
- Wrap complex expectations (multiple checks on the same object) in `testthat::expect()` with a custom expectation function or break them into multiple assertions inside the same `# THEN` section.
- Never compare against raw literals inside assertions; always assign them to `expected_` variables first.
- Prefer `expect_snapshot()` for structured objects or printed output, but keep snapshot files under version control and review them carefully.

---

## **7. Fixtures and Sample Data**
Shared fixtures keep tests concise while documenting domain-specific data.
- Place reusable data builders in `tests/testthat/helper-*.R`. Name exported helpers `fixture_*`.
- Within tests, assign fixture outputs to `given_` variables (e.g., `given_orders <- fixture_orders()`).
- If a fixture represents an expected value, store it in an `expected_` variable after any necessary transformation.
- Keep fixtures pure—avoid writing to disk or altering global state. Use factories that accept overrides so tests can tweak values without copy-pasting the whole object.

---

## **8. Error Handling and Edge Cases**
Express failure expectations explicitly so regressions surface with actionable messages.
- Use `expect_error()`/`expect_warning()` with `class =` arguments rather than relying on generic message matching.
- When checking for messages, capture them inside the `# WHEN` section (e.g., `actual_error <- expect_error(...)`) and assert on the object in `# THEN`.
- For boundary cases (empty vectors, `NA` inputs, zero-length data frames), isolate them in individual `test_that()` blocks instead of stacking multiple scenarios in one test.

---

## **9. Skips and Conditional Logic**
Skipping should communicate intent and remain discoverable.
- Guard platform-specific tests with `skip_on_os()` or `skip_if()` inside the `# GIVEN` or `# SETUP` section, right before the condition they protect.
- Provide a short, descriptive reason (e.g., `skip_if_not_installed("curl", minimum_version = "5.0.0")`).
- Avoid dynamic skipping based on network availability; instead, design tests to stub network calls.
- Do not skip within `# THEN`; if an expectation cannot run, split the test so the skip occurs before assertions.

---

## **10. Breaking the Rules**
Rare situations may warrant deviations (e.g., property-based tests or heavily parameterized expectations). When you must break a rule:
- Add a comment explaining why the exception improves clarity or coverage.
- Keep the rest of the structure intact so readers still recognize the flow.
- Prefer writing multiple conventional tests over one unconventional test unless the latter dramatically reduces duplication.

---

Adhering to this style guide keeps R unit tests as predictable and reviewable as the C# suites that inspired it, while embracing idiomatic `testthat` tooling.
