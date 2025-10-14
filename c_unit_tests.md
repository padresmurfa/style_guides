# C Unit Testing Style Guide

This document adapts the conventions from the **C# Unit Testing Style Guide** to the realities of **C-based unit testing**. It focuses on writing **clear, deterministic, and maintainable** tests when using popular C harnesses such as **Unity**, **cmocka**, or **Criterion**.

---

## **Target Audience**

The primary audience is developers and Large Language Models (LLMs) that assist with generating C tests. Because C lacks many of the guardrails found in higher-level languages, small mistakes can produce undefined behavior or silently skip assertions. These practices emphasize **readability, memory-safety, and explicit control flow** so humans can confidently review generated tests.

---

## **1. Test Directory & Module Organization**
C projects organize behavior around translation units (`.c` files) rather than namespaces. A predictable directory layout makes it obvious which production module a test exercises, keeps build scripts simple, and aligns with IDE discovery rules.
- Mirror the production directory tree under a `tests/` (or `test/`) root.
- For every production file `src/<path>/<module>.c`, create a corresponding test file `tests/<path>/<module>_tests.c`.
- Keep a **one-to-one mapping between modules and test suites**. If a module exposes multiple cohesive subsystems (e.g., a group of static helpers compiled into the same translation unit), create dedicated test files such as `tests/<path>/<module>_<feature>_tests.c`.
- Avoid combining tests for unrelated modules in the same file—even if they share headers. Doing so complicates compilation, introduces order dependencies, and obscures ownership.

✅ **Good Example:**
```
src/
│── io/
│   └── file_reader.c
 tests/
│── io/
│   └── file_reader_tests.c
```
🚫 **Bad Example:**
```
tests/
│── io_tests.c      // ❌ Aggregates multiple modules; hard to navigate
```

---

## **2. Test Suite Naming & File Conventions**
Most C harnesses group tests via suites or fixtures. Naming them consistently keeps the build system declarative and allows selective execution with predictable filters.
- Each test file must define **exactly one suite struct or registration function** (depending on the framework) that ends with `_tests`.
- When a suite exercises a single module, name it `<module>_tests` (matching the file name).
- When targeting a smaller scope (e.g., a static helper exposed for testing), append the focused area: `<module>_<feature>_tests`.
- Keep suite identifiers free of uppercase letters unless the framework requires macros. Consistent snake_case mirrors conventional C naming and avoids clashes with preprocessor directives.
- Place suite registration near the top of the file so readers can immediately see which functions belong to the suite.

✅ **Good Example (Unity):**
```c
TEST_GROUP(file_reader_tests);
```
🚫 **Bad Example:**
```c
TEST_GROUP(FileReaderWorksAsIntended); // ❌ CamelCase and vague intent
```

---

## **3. Test Function Naming**
C test functions are free functions registered with the harness. Names should read like English sentences that capture the scenario, inputs, and expected outcome.
- Every test function must start with `test_`.
- Use lowercase snake_case for the remainder of the name. Separate phrases with single underscores (no double underscores).
- When multiple production functions are exercised within the same suite, include the function name immediately after the prefix: `test_read_block_returns_eof_on_empty_file`.
- If the suite focuses on a single production function, omit the production name from individual test names to avoid redundancy: `test_returns_eof_on_empty_file`.
- Prefer explicit behavior-driven names over generic labels such as `test_success` or `test_nominal_case`.
- Keep each scenario in a dedicated function. Do not share control flow across unrelated assertions.

✅ **Good Example:**
```c
void test_read_block_returns_eof_on_empty_file(void);
```
🚫 **Bad Example:**
```c
void test_read_block(void);      // ❌ Too vague
void read_block_works(void);     // ❌ Missing test_ prefix
```

---

## **4. Test Function Sectioning**
Because C tests rely on manual setup and teardown, structuring each function into labeled sections prevents accidental reuse of state and highlights lifetime responsibilities. Use **single-line `//` comments** to identify sections.

Each test function should follow this order when the sections are present:
1. **GIVEN**
   - Declare immutable inputs and constants with the prefix `given_`.
   - Allocate fixtures and buffers here; track ownership explicitly (e.g., record whether `free` is required).
   - Do not mutate `given_` values outside of the GIVEN block.
2. **MOCKING**
   - Configure stubs or fake dependencies. In C this usually means setting global function pointers, installing spy objects, or enabling link-time fakes.
   - Name mock handles `mock_*` and reset them at the end of the test (or in teardown) to avoid leakage across suites.
3. **SETUP**
   - Prepare runtime state that will be mutated during the test. Prefix variables with `env_` to indicate environment/configuration objects.
   - When injecting mocks through function pointers or struct fields, assign the `mock_*` objects to `env_*` handles here.
4. **SYSTEM UNDER TEST**
   - Instantiate or reference the primary object/function under test. Assign it to a variable named `sut` (or `sut_*` for multiple handles).
   - Never reference `mock_*` variables directly in this section; pass the `env_*` handles instead.
5. **WHEN**
   - Invoke the function(s) being tested. Store returned values or observable state in `actual_*` variables.
   - If the framework uses macros (e.g., `cmocka_unit_test_setup_teardown`), still capture outputs explicitly inside the test body.
6. **EXPECTATIONS**
   - Compute expected results and assign them to `expected_*` variables. Keep these immutable.
   - Avoid inline literals in assertions; storing them in variables clarifies intent and simplifies diffs.
7. **THEN**
   - Perform assertions comparing `expected_*` and `actual_*` values using the framework’s macros.
   - Keep assertions deterministic—avoid relying on unspecified order of evaluation or short-circuiting.
8. **LOGGING** (Optional)
   - If the module emits logs through callback hooks, assert on captured entries here. Favor log verification over complex mocking logic when possible.
9. **BEHAVIOR** (Required when mocks/spies are present)
   - Verify that mocks were called with the expected parameters. Include explicit counters or flag checks if the harness lacks built-in verification.
10. **TEARDOWN** (Optional)
    - Release resources created in GIVEN or SETUP (e.g., free heap allocations, close file descriptors).
    - Reset global state or function pointers so subsequent tests start from a clean baseline.

Empty sections may be omitted, but **never merge two headings into one block**. If a section grows unwieldy, split it with sub-headings such as `// GIVEN: buffer sized to trigger resize`.

✅ **Good Example:**
```c
void test_parser_returns_null_when_header_missing(void)
{
    // GIVEN: an input buffer missing the header
    const char given_input[] = "payload";

    // SYSTEM UNDER TEST
    parser_t sut = parser_create();

    // WHEN: parse the buffer
    const message_t *actual_message = parser_parse(&sut, given_input, sizeof(given_input) - 1);

    // EXPECTATIONS
    const message_t *expected_message = NULL;

    // THEN
    TEST_ASSERT_EQUAL_PTR(expected_message, actual_message);
}
```
🚫 **Bad Example:**
```c
void test_parser_returns_null_when_header_missing(void)
{
    parser_t sut = parser_create();
    TEST_ASSERT_NULL(parser_parse(&sut, "payload", 7)); // ❌ No structure, literals inline
}
```

---

## **5. Memory & Resource Hygiene**
Manual memory management is a defining risk in C tests. Leaks, double frees, or dangling pointers inside the test harness can mask bugs or crash later suites.
- Prefer stack allocation whenever sizes are fixed and small enough for the target platform.
- When heap allocation is necessary, store the pointer in a `env_*` variable and release it in a `// TEARDOWN` block at the end of the test (or via the framework’s teardown hook).
- Use helper functions to build complex fixtures, and ensure they expose symmetrical destroy functions (`*_destroy`).
- Zero-initialize structs with designated initializers to avoid undefined behavior.
- When using globals for mocks or shared fixtures, reset them in `setup()`/`teardown()` functions so each test starts from a clean slate.

---

## **6. Fixtures: Best Practices**
Fixtures provide reusable data with known lifetimes, keeping tests expressive without duplicating boilerplate.
- Centralize fixture builders in a dedicated `tests/fixtures/` directory or `fixtures.c` file.
- Name constructor helpers `fixture_<thing>()` and ensure they return fully initialized structs.
- Prefer functions that return structs by value or populate caller-provided buffers, minimizing heap churn.
- When a fixture allocates memory, provide a companion `fixture_<thing>_free()` and call it in the TEARDOWN block of every test that uses it.

✅ **Good Example:**
```c
user_t fixture_user(void)
{
    return (user_t){
        .id = 42,
        .name = "alice",
    };
}
```

---

## **7. Mocking, Stubs, and Fakes**
C lacks a universal mocking framework, so tests often rely on link-time substitution, dependency injection through function pointers, or handcrafted spies. Apply the following discipline:
- **Mock only direct collaborators.** If the SUT calls into a dependency via a function pointer or virtual table, stub just those entry points.
- Prefer **fakes/spies** implemented in C over complex macro-based mocks when assertions require inspecting structured state.
- Prefix fake instances with `fake_` and install them through `env_*` handles in the SETUP section.
- Record interactions inside the fake (e.g., increment counters, copy arguments). Expose verification helpers such as `fake_logger_assert_called_once()` and call them in the BEHAVIOR section.
- Reset global hooks in the BEHAVIOR or TEARDOWN block to avoid cross-test contamination.
- When using compile-time conditionals to swap implementations (e.g., `#ifdef TESTING`), ensure the test still follows the sectioning rules and explicitly documents the injected behavior.

✅ **Good Example:**
```c
static int fake_write_count;

static int fake_write(void *ctx, const uint8_t *buf, size_t len)
{
    (void)ctx;
    fake_write_count++;
    return (int)len;
}

void test_driver_flush_invokes_transport_once(void)
{
    // MOCKING: install the fake transport
    fake_write_count = 0;

    // SETUP
    transport_vtable env_transport = {
        .write = fake_write,
    };

    // SYSTEM UNDER TEST
    driver_t sut = driver_create(&env_transport);

    // WHEN
    driver_flush(&sut);

    // BEHAVIOR
    TEST_ASSERT_EQUAL_INT(1, fake_write_count);
}
```

---

## **8. Assertions & Naming**
Precise naming makes it obvious what is being compared and prevents accidental swaps of expected and actual values.
- Store values destined for comparison in `expected_*` and `actual_*` variables.
- Avoid asserting directly against literals or `given_*` variables; assign them to expected variables first.
- Use assertion macros that clearly communicate intent (e.g., `TEST_ASSERT_EQUAL_INT`, `assert_true`). When the framework lacks a specialized macro, wrap comparisons in helper functions for reuse.
- When comparing floating-point values, use epsilon-aware helpers that document the tolerance.

✅ **Good Example:**
```c
const int expected_count = 2;
int actual_count = driver_pending_requests(&sut);
TEST_ASSERT_EQUAL_INT(expected_count, actual_count);
```

---

## **9. Exception-Like Scenarios**
C does not have native exceptions, but many APIs signal errors via return codes or sentinel values. Treat these pathways with the same rigor you would apply to exceptions in higher-level languages.
- Capture error indicators in `actual_*` variables immediately after the call (e.g., `int actual_status = function(...);`).
- Assert on both the return code and any side-effected state.
- When testing `setjmp`/`longjmp`-based error handling, place the `setjmp` call in the WHEN section and assert on the resulting control flow in THEN.

---

## **10. Setup & Teardown Hooks**
Many frameworks offer per-test setup/teardown callbacks.
- Favor explicit setup inside the test body (using the GIVEN/SETUP sections). Reserve global setup hooks for expensive, truly shared initialization (e.g., creating a temporary directory).
- Always reset global state in teardown to prevent cross-test pollution.
- Name setup functions `suite_setup` / `suite_teardown` or `<suite>_setup` to clarify scope.

✅ **Good Example (cmocka):**
```c
static int file_reader_setup(void **state)
{
    *state = fixture_file_reader();
    return 0;
}
```

---

## **11. Breaking the Rules**
These conventions favor explicitness and reviewability. If deviating from a rule materially improves clarity (e.g., collapsing GIVEN and SETUP because the test only declares a single scalar), document the rationale with an inline comment:
```c
// NOTE: Collapsing GIVEN and WHEN because the function is pure and returns immediately.
```
When in doubt, err on the side of following the guide—predictable structure is more valuable than individual stylistic preferences.

