# Go Unit Testing Style Guide

This document defines **best practices** for writing **clear, structured, and maintainable** unit tests in Go using the standard `testing` package alongside common helpers such as `testify/require` or generated mocks.

---

## **Target Audience**

The primary audience is Go developers and Large Language Models (LLMs) acting as coding assistants. Following these guidelines keeps Go tests **predictable, idiomatic, and easy to audit**.

The structure may feel heavyweight for human authors, but it provides an excellent checklist when collaborating with tooling. **Humans must still review every generated test** and tune the suite for performance and readability.

---

## **1. Test Package Organization**
Consistent package selection makes it obvious how a test exercises the source package and prevents accidental import cycles.
- Prefer **black-box tests** in a separate `_test` package (e.g. `package widgets_test`) when verifying the public API.
- Use the same package (e.g. `package widgets`) only when you need access to unexported helpers.
- Never mix both patterns in a single directory. Pick one per Go package and apply it consistently.
- Keep test files in the same directory tree as the code under test. Avoid sprawling `testdata` folders that do not mirror the production structure.

✅ **Good Example:**
```go
package parser_test

import (
    "testing"

    "example.com/project/parser"
)
```
🚫 **Bad Example:**
```go
package tests // ❌ no relation to the production package
```

---

## **2. Test File Naming and Scope**
Predictable file names help reviewers trace which behavior is being exercised.
- Test files **must end with `_test.go`**.
- Limit each file to a single subject under test—either a type or a cohesive function group.
- When targeting one exported symbol, name the file `<symbol>_test.go` (e.g. `handler_test.go`).
- When covering a broad type with many methods, split into focused files such as `handler_process_test.go` or `handler_validate_test.go` to map clearly to responsibilities.
- Place golden files or fixtures inside a `testdata/` directory adjacent to the test file. Never mix fixture builders and assertions in the same file without clear separation.

✅ **Good Example:**
```
handler/
│── handler.go
│── handler_process_test.go
│── handler_validate_test.go
```
🚫 **Bad Example:**
```
handler/
│── handler.go
│── handler_all_the_tests.go // ❌ vague scope
```

---

## **3. Test Function Naming**
Clear names communicate behavior, inputs, and expectations.
- Each test function **must start with** `Test` followed by the symbol under test and the scenario.
- Use mixedCase names like `TestProcess_ReturnsIDWhenValid` to remain idiomatic.
- Separate scenario details with underscores (`_`) for readability. Avoid consecutive underscores.
- Do not include the package name—Go’s tooling already shows it.
- Reserve `TestMain` for package-wide setup; keep it tiny and avoid global state.

✅ **Good Example:**
```go
func TestProcess_ReturnsIDWhenValid(t *testing.T) { ... }
```
🚫 **Bad Example:**
```go
func WorksAsIntended(t *testing.T) { /* ❌ missing Test prefix */ }
func TestProcess(t *testing.T) { /* ❌ lacks scenario context */ }
```

---

## **4. Test Structure and Sectioning**
Structured sections keep tests readable, especially in the absence of explicit blocks in Go.
Each test function uses **capitalized comment headers** to separate the phases:

1. **GIVEN**
   - Declare inputs and initial state.
   - Name variables with the `given` prefix (e.g. `givenRequest`).
   - Build fixtures using small helper constructors or data in `testdata/`.

2. **CAPTURE** *(if applicable)*
   - Introduce variables that will hold values captured from collaborators (e.g. arguments recorded by a spy implementation).
   - Prefix them with `capture` (`var capturePublishedEvent *Event`).
   - Limit this section to declarations; wire the capture mechanics in MOCKING or SETUP.
   - Skip CAPTURE entirely when no captured state is required.

3. **MOCKING** *(if applicable)*
   - Configure doubles (`mockClient`, `fakeStore`) created with interfaces or generated tools (e.g. GoMock).
   - Keep mocks minimal; prefer small real implementations when cheaper.

4. **SETUP**
   - Assemble the environment (e.g. wire dependencies, instantiate struct under test) using variables prefixed with `env`.
   - Assign mocks to production interfaces here (`envStore := mockStore`), not in GIVEN.
   - Ensure SETUP follows whichever of GIVEN, CAPTURE, and MOCKING are present; omit unused sections without reordering others.

5. **SYSTEM UNDER TEST**
   - Instantiate the SUT in a variable named `sut` (or `sutFn` for functions returned from factories).
   - The SUT should consume `env*` dependencies produced in SETUP.

6. **WHEN**
   - Execute the behavior under test and capture outputs in `actual*` variables.
   - Use `t.Helper()` inside helper functions invoked here.
   - Move any state stored in `capture*` placeholders into `actual*` variables prior to assertions.

7. **EXPECTATIONS**
   - Declare expected values in `expected*` variables, even when simple.
   - Use custom comparison helpers (`cmp.Diff`, `require.Equal`) if values are complex.

8. **THEN**
   - Perform assertions. Prefer `t.Fatalf`/`t.Errorf` with helpful messages or `require/assert` from Testify.
   - Do not assert directly against literals; compare `actual*` to `expected*`.

9. **BEHAVIOR** *(if mocks used)*
   - Verify mock interactions (e.g. `require.Equal(t, 1, mockStore.Calls["Save"])`).
   - Ensure every expectation on a mock is asserted; otherwise default to fakes.

Comments must read as sentences (`// GIVEN: a valid request`). Omit sections that would be empty.

---

## **5. Table-Driven Tests**
Table tests are idiomatic in Go but must remain disciplined.
- Place table definitions inside the GIVEN section under a `// GIVEN: test cases` comment.
- Name the slice `givenCases` (or `given<Subject>Cases`). Each element should be a struct literal with fields `name`, `given`, `expected`, etc.
- Iterate with `t.Run(tc.name, func(t *testing.T) { ... })` using subtests. Keep the inner body structured with the same sections.
- Avoid mixing table tests with unrelated logic; prefer one table per behavior.
- Do not use anonymous functions when a single case is clearer. Small, dedicated tests often read better than bloated tables.

✅ **Good Example:**
```go
type calculateCase struct {
    name           string
    givenInput     int
    expectedOutput int
}
```
🚫 **Bad Example:**
```go
cases := []struct { // ❌ unnamed fields hide intent
    int
    bool
}{ ... }
```

---

## **6. Helpers and Fixtures**
Shared helpers reduce duplication but must stay obvious.
- Define helper constructors in files ending with `_test.go` to keep them out of production builds.
- Prefix helper functions with `build`, `new`, or `fake` to clarify intent (`buildRequest`, `fakeClock`).
- Call `t.Helper()` at the top of helper functions to keep line numbers accurate on failure reports.
- Store large fixture data under `testdata/` and load with `os.ReadFile` in GIVEN.
- Keep helper files close to the tests they support; avoid a monolithic `helpers_test.go` at the repo root.

---

## **7. Assertions and Error Handling**
- Prefer the standard library first: use `if actual != expected { t.Fatalf(...) }` with detailed context.
- When using `testify`, choose `require` for fatal checks and `assert` for non-fatal ones. Never mix both in the same section without intent.
- Always compare Go errors via `require.NoError(t, err)` or `errors.Is` rather than string comparison.
- When comparing structs, ensure zero values are intentional; initialize expected structs completely to avoid accidental omissions.
- Avoid panicking inside tests; rely on `t.Fatalf` and `t.Helper()`.

---

## **8. Mocking Guidelines**
- Favor real implementations or hand-written fakes over heavy mocking frameworks when practical.
- When using GoMock, generate mocks in a `mocks/` subpackage with `//go:generate mockgen ...` directives.
- Reset mock state at the top of each test. Do not share mocks across tests via globals.
- Keep expectations in the BEHAVIOR section and verify them with `defer mockCtrl.Finish()` or explicit assertions.
- Do not assert on private fields inside mocks—interact through the interface contract.

---

## **9. Concurrency Tests**
- Use `t.Parallel()` only after the GIVEN/MOCKING/SETUP sections are complete to avoid races.
- Guard shared resources with synchronization primitives (`sync.Mutex`) inside the test, not by mutating global state.
- For goroutines, provide explicit cancellation via contexts or channels and wait with `sync.WaitGroup` to prevent leaks.
- Employ `require.Eventually` or polling loops with deadlines when verifying asynchronous outcomes. Never rely on `time.Sleep` without a reasoned timeout.

---

## **10. Maintaining Determinism**
- Seed random number generators with constant seeds (`rand.New(rand.NewSource(123))`).
- Replace time-dependent logic with injected `time.Time` or `clock` interfaces.
- Avoid relying on OS-specific behaviors; stub environment variables and file paths explicitly.
- When tests depend on locale or number formatting, assert using exact values produced by controlled inputs.

---

## **11. Breaking the Rules**
Exceptional cases may justify deviations. When you break a rule:
- Add a comment explaining why the exception improves clarity or correctness.
- Keep the exception local to the test; do not generalize unusual patterns across the suite.
- Revisit exceptions during refactors to see if the standard structure can be restored.

---

## **12. Checklist Before Submitting**
- [ ] Test package names align with the code under test.
- [ ] Each test file focuses on a single subject and ends with `_test.go`.
- [ ] Test functions follow the `Test<Symbol>_<Scenario>` pattern.
- [ ] GIVEN/WHEN/THEN (and optional sections) are present and meaningful.
- [ ] Mocks or fakes have explicit behavior assertions.
- [ ] Table tests use named fields and subtests.
- [ ] Helpers call `t.Helper()` and live in `_test.go` files.
- [ ] Tests pass with `go test ./...` on a clean checkout without race conditions.

