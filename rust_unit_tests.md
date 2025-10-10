# Rust Unit Testing Style Guide

This document defines **best practices** for writing **clear, structured, and maintainable** unit tests in Rust using the built-in test harness, along with common crates such as `mockall`, `proptest`, and `assert_matches` when needed.

---

## **Target Audience**

The primary target audience for this document is developers and Large Language Models (LLMs) acting as coding assistants. By following these guidelines, tests will be **structured, readable, and maintainable**.

These practices may be too detailed for human-written tests but serve as an excellent guide for generating automated tests with LLM assistance. **Review all test functions carefully**, as human oversight is crucial to ensure test quality.

---

## **1. Test Module Organization**
Consistent module organization makes it immediately obvious where a test lives relative to the production code, so reviewers can locate the subject under test without opening multiple files. Misaligned modules cause confusion when navigating between source and tests, especially in IDEs that rely on module matching for discovery.
- Every unit test module inside a production file must live under a `#[cfg(test)] mod tests` block whose name mirrors the file name and appends `_tests` when the tests are extracted to a separate module file.
- Integration tests must mirror the crate's module path inside the `tests/` directory using nested folders and module declarations.
- Keep a **one-to-one mapping between folders and modules** so moving a file preserves the module path.
- Avoid combining tests for different production modules under a single integration test file—create separate files instead.

✅ **Good Example:**
```rust
#[cfg(test)]
mod user_service_tests {
    // ...
}
```
🚫 **Bad Example:**
```rust
#[cfg(test)]
mod tests { // ❌ Does not mirror the production module
    // ...
}
```

---

## **2. Test File Naming and Scope**
Rust uses a combination of inline unit tests and integration tests. Keeping naming consistent allows contributors to trace failures quickly and spot coverage gaps. Without a naming convention, teams waste time hunting for relevant tests, duplicate effort, and risk overlooking scenarios because responsibilities are spread across ambiguously labeled files.
- Inline unit tests must reside in the same file as the production module, within a `#[cfg(test)]` block.
- When tests become too large to live inline, move them to `tests/<module_path>/<file_name>_tests.rs` to keep parity with the production module.
- Each integration test file must focus on a single public API surface (module, struct, trait, or function). If multiple surfaces are tested, split them into distinct files.
- Avoid mixing unrelated modules in the same integration test file.

✅ **Good Examples:**
```
src/
│── services/
│   │── user_service.rs        // production code with inline tests
│
 tests/
 │── services/
 │   │── user_service_tests.rs // integration tests for services::user_service
```
🚫 **Bad Example:**
```
tests/
│── all_services.rs // ❌ Mixed tests for unrelated modules
```

---

## **3. Region-Focused Test Modules**
Rust does not have `#region` blocks, but modules often contain cohesive implementations (e.g., helper traits or submodules). Mirroring those logical regions in the test suite keeps the feedback loop tight by ensuring every area has an intentionally scoped test module.
- When a module contains multiple logical areas (public API, helper functions, conversions), create nested test modules such as `mod conversions_tests` inside the root `tests` module.
- Do not mix unrelated logical regions in a single nested test module. Create one module per area.
- If helpers are private, test the public API that exercises them rather than testing the helper functions directly.

✅ **Good Example:**
```rust
#[cfg(test)]
mod tests {
    mod parsing_tests {
        // ...
    }

    mod serialization_tests {
        // ...
    }
}
```
🚫 **Bad Example:**
```rust
#[cfg(test)]
mod tests {
    // ❌ Merges parsing and serialization cases in one module
}
```

---

## **4. Test Function Naming**
Clear function names document the behavior under test and the expected outcome, which helps maintainers understand intent without rereading the implementation. Vague or inconsistent names hide gaps in coverage, encourage multi-purpose tests, and make regression triage harder when a failing test name does not reveal what broke.
- Each test function **must start with** `fn test_` and use `#[test]`.
- Use snake_case names with underscores to separate major phrases.
- Test function names should **not contain consecutive underscores**.
- If multiple production functions are being tested within the same module, the test function **must start with** `test_<function_name>_`.
- If a single production function is being tested in the file, omit the production function name from the test function.
- Favor verbosity over brevity—short names rarely communicate enough context to make the test self-documenting.
- Test function names must describe the **specific branch of execution** they cover (e.g. `test_parse_id_returns_err_for_empty_input`), not merely state that something "works".
- **Separate tests** into distinct functions rather than combining multiple test cases.
- **Avoid property-based or table-driven tests** unless they significantly improve clarity. When using them, document each case explicitly.
- Any component found in the module name **must NOT be duplicated in the test function name**.

✅ **Good Example:**
```rust
#[test]
fn test_process_data_returns_correct_value() {
    // ...
}
```
🚫 **Bad Example:**
```rust
#[test]
fn check_process_data() { /* ❌ Missing test_ prefix */ }
#[test]
fn test_process() { /* ❌ Too vague */ }
#[test]
fn test_multiple_cases() { /* ❌ Tests multiple things at once */ }
```

---

## **5. Test Function Sectioning**
Standardized sections carve complex tests into digestible steps, making it easier to see how inputs flow through mocks and the system under test. Without this structure tests devolve into monolithic blocks where intent, setup, and verification intermingle, obscuring bugs and encouraging brittle copy-paste patterns. The ordering rules below are intentionally strong to build reliable habits—refer to [**Section 17. Breaking the Rules**](#-17-breaking-the-rules) for guidance on the rare cases where deviating is justified.
Each test function follows a structured format, **separated by clear comments**:
- Where possible, the order of sections should be as it is presented below.
- Empty sections should be omitted.
- In some cases, it may be justifiable to have multiple instances of specific sections.
- When a single **MOCKING** or **SETUP** block becomes large or involves multiple distinct steps, split it into smaller sub-sections with their own human-readable comment headers (e.g. `// SETUP: Configure HTTP client factory` and `// SETUP: Initialize environment variables`), so readers can quickly grasp each part.
- Section comment headers should be written as full descriptive sentences (e.g. `// GIVEN: a valid user payload`), not terse labels. If a section would contain only a trivial one-liner without setup complexity, you may omit its own comment.

### **Standard Sections:**

1. **GIVEN**
   - Set up initial conditions and inputs.
   - Assign variables with the prefix `given_`.
   - When helper builders are used, instantiate them in this section.
   - **Helper structs** should be named `Fixture*` or `Given*` and assigned to `given_*` variables prior to use.
   - The GIVEN section should always be the first section in a test, unless there is nothing to define in GIVEN, in which case it should be omitted.
   - **The GIVEN section must not be merged with any other section.**

2. **MOCKING**
   - Define and configure **mock objects or fakes**.
   - Mock variables must be named `mock_*`; fakes must be named `fake_*`.
   - Use crates like `mockall`, `mockito`, or handcrafted fakes depending on the dependency surface.
   - The MOCKING section should occur before the SETUP section, or be omitted if no mocking is required. Under special circumstances described above, it may be placed after the SETUP section.
   - **The MOCKING section must not be merged with any other section.**
   - **If mocking logic itself has multiple responsibilities (e.g. creating fakes and configuring behaviors), split into sub-headers** (e.g. `// MOCKING: create fake repository` and `// MOCKING: record expected writes`).

3. **SETUP**
   - Perform all test initialization that prepares the non-mocked environment (e.g., constructing configuration structs, wiring dependencies, preparing temporary directories).
   - Variables created in this section should be prefixed with `env_` (e.g. `env_logger`, `env_repo`).
   - When constants or variables are needed in this section that are important for the overall comprehensibility of the test, they should be declared in variables in the GIVEN section instead.
   - The SETUP section should follow the GIVEN section, unless there is nothing to set up, in which case it should be omitted.
   - The SETUP section should refer to GIVEN variables, not `mock_*` variables, except in rare occasions.
   - Assign mock objects to real trait objects or structs in the SETUP section, using the `env_` prefix (e.g., `let env_logger = mock_logger.clone();`). The mock variables themselves may only be used in the SETUP, WHEN, and BEHAVIOR sections.
   - **The SETUP section must not be merged with any other section.**
   - **If SETUP involves multiple distinct stages (e.g. reading fixtures, wiring dependencies, configuring environment), split into multiple sub-sections with descriptive comments.**

4. **SYSTEM UNDER TEST**
   - Assign the system under test to a variable named `sut`.
   - If multiple systems are being tested, assign them to separate `sut_*` variables.
   - The SYSTEM UNDER TEST must never directly reference `mock_*` variables. It must use the `env_*` variables initialized in SETUP.
   - **The SYSTEM UNDER TEST section must not be merged with any other section.**

5. **WHEN**
   - Perform the action or method call being tested.
   - Assign the results to `actual_*` variables. This typically includes the return value from invoking the system under test and any mutated state captured after the call.
   - If the test needs to use an `env_*` or `mock_*` in later sections, assign their relevant snapshots to `actual_*` variables in WHEN and use those instead.
   - **The WHEN section must not be merged with any other section.**

6. **EXPECTATIONS**
   - Define expected values **before assertions**.
   - Expected values must be assigned to `expected_*` variables.
   - The EXPECTATIONS section should be strictly placed after WHEN and before THEN.
   - This section should not refer to any `actual_*` variables.
   - **The EXPECTATIONS section must not be merged with any other section.**

7. **THEN**
   - Perform **assertions** comparing actual results to expected values.
   - **Never assert against literals**—always use `expected_*` variables.
   - Use assertion macros from `assert_eq!`, `assert_ne!`, `assert_matches!`, or similar crates as appropriate.
   - **The THEN section must not be merged with any other section.**

8. **LOGGING**
   - Verify any behavior that can be validated via captured logging or tracing output.
   - Prefer capturing logs via crates like `test-log` or by injecting a fake logger.
   - In unit tests, assert against log entries in the LOGGING section to confirm that the intended execution branch was exercised.
   - **Prefer asserting on emitted logs over (or in addition to) mock verifications** whenever possible: log-based assertions are simpler to write and review, and often replace complex mock setups.
   - Each unit test should explicitly verify that the execution branch it is meant to cover is taken, reducing false positives when setup does not exercise the desired branch.

9. **BEHAVIOR**
   - Contains assertions for **mock interactions** (e.g., verifying call counts, arguments, or recorded invocations).
   - This section **must be present if mocks are used**.
   - **The BEHAVIOR section must not be merged with any other section.**
   - **All mocks must record expectations explicitly**—either by using `mockall::Sequence`, `expect_*` methods, or custom fake assertions.
   - **Every expectation must be explicitly verified** in this section. Unverified expectations should cause the test to fail.

✅ **Good Example:**
```rust
#[test]
fn test_process_data_returns_correct_value() {
    // GIVEN: an input payload with two records
    let given_payload = vec!["a", "b"];

    // SYSTEM UNDER TEST: a deterministic processor
    let sut = DataProcessor::new();

    // WHEN: process the payload
    let actual_result = sut.process(&given_payload);

    // EXPECTATIONS: the payload is uppercased
    let expected_result = vec!["A".to_string(), "B".to_string()];

    // THEN: compare actual and expected results
    assert_eq!(expected_result, actual_result);
}
```
🚫 **Bad Example:**
```rust
#[test]
fn test_process_data_returns_correct_value() {
    // MOCKING
    let mock_repo = MockRepository::new();

    // SYSTEM UNDER TEST
    let sut = DataProcessor::new(mock_repo); // ❌ passing mock directly without env_
}
```

---

## **6. Fixtures: Best Practices**
Reusable fixtures encourage realistic, DRY data that mirrors production scenarios while keeping tests focused on behavior instead of data plumbing. Inlining bespoke objects everywhere invites duplication, increases maintenance costs when models evolve, and makes it harder to reason about how changes cascade across tests.
- **Prefer fixtures over mocks**—favor shared, reusable builders or helper functions rather than creating one-off test data. Fixtures help keep your tests DRY and consistent, and allow them to work on more accurate test data.
- **Instantiate fixtures in the appropriate section** (e.g., GIVEN or SETUP), and modify them as needed to fit the test's needs.
- **Use dedicated fixture modules or helper functions** instead of inlining complex structures repeatedly.
- **Name fixture constructors with the `fixture_*` prefix** to make their purpose obvious.

✅ **Good Example:**
```rust
pub mod fixtures {
    pub fn fixture_ip_address() -> String {
        "192.168.1.1".into()
    }

    pub fn fixture_terminal_config() -> TerminalConfig {
        TerminalConfig { id: fixture_ip_address(), ..Default::default() }
    }
}

#[test]
fn test_get_terminal_config_returns_not_found_when_terminal_missing() {
    // GIVEN: a command context with a missing terminal
    let given_terminal_ip = fixtures::fixture_ip_address();
    let given_command = fixtures::fixture_terminal_config();

    // SYSTEM UNDER TEST: real service without mocks
    let sut = CarWashService::default();

    // WHEN: request the terminal config
    let actual_result = sut.get_terminal_config(&given_command);

    // EXPECTATIONS: the service returns None
    let expected_result: Option<TerminalConfig> = None;

    // THEN: verify the result
    assert_eq!(expected_result, actual_result);
}
```
🚫 **Bad Example:**
```rust
#[test]
fn test_get_terminal_config_returns_not_found_when_terminal_missing() {
    // GIVEN
    let given_terminal_ip = "192.168.1.1".to_string();
    let given_command = TerminalConfig { id: given_terminal_ip.clone(), ..Default::default() }; // ❌ Inline setup everywhere

    // ...
}
```

---

## **7. Mocking: Best Practices**
Disciplined mocking keeps tests readable and trustworthy by ensuring only true collaborators are simulated and their expectations are explicit. Unrestrained mocks create brittle, implementation-driven tests that break on harmless refactors, mask missing coverage of dependencies, and clutter SUT construction with framework boilerplate.
- **Never mock what you don't have to**—prefer fixtures or real instances where practical.
- **Only mock traits that the system-under-test uses directly**—this ensures that your test exercises the SUT properly, without chasing deep call stacks.
- Each dependency should have its own focused unit tests; avoid verifying downstream behavior via higher-level tests.
- **Use crates like `mockall` or `double`** for dependency injection.
- **Assertions on mock interactions go in the BEHAVIOR section, not in the THEN section.**
- **Mocks must never be passed directly to the system under test.** Instead, assign mocks to trait objects in the SETUP section and pass those to the SUT.
- Following this pattern keeps the SUT construction readable: the MOCKING section owns the intricate setup, the SETUP section exposes the ready-to-use dependencies via `env_*` variables, and the SYSTEM UNDER TEST section reads like a clean recipe that highlights only the essential collaborators.

### **7.1 Test Fakes (a.k.a. In-Memory Implementations)**
Test fakes are lightweight, purpose-built implementations of a trait that you author specifically for the test suite. They shine when mocking frameworks become gnarly—especially for complex interaction verification or structured log inspection—because you can write direct, intention-revealing code without wrestling with macro-generated expectations.

- **Create fakes when setup or verification with a mock would be noisy, repetitive, or brittle.** If verifying interactions through a mocking framework requires complicated expectation chaining, a fake is often clearer.
- **Name fake instances with the `fake_*` prefix and construct them in the MOCKING section.** This keeps parity with mock naming so readers instantly recognize simulated collaborators.
- **Assign each `fake_*` variable to an `env_*` variable in the SETUP section before handing it to the SUT.** Treat a fake like any other dependency so the SYSTEM UNDER TEST section stays clean and only references `env_*` collaborators.
- **Interact with the fake via its `fake_*` variable in the THEN, LOGGING, and BEHAVIOR sections.** Fakes frequently expose custom assertion helpers (e.g., `fake_logger.assert_logged(...)`) that capture behavior without verbose expectation code.
- **Document expectations inside the fake when possible.** Purpose-built helpers (such as storing structured log entries) make intent obvious and reduce duplicated parsing logic across tests.

#### Strengths Compared to Mocks
- **Readable behavior verification.** Fakes encapsulate the verification logic in methods that read like English, avoiding the visual clutter of chained expectations.
- **Reduced setup friction.** Because you own the implementation, you can build helpers that mirror your domain vocabulary instead of contorting the SUT to satisfy a framework.
- **Deterministic assertions.** Fakes can capture state and expose first-class assertions, lowering the risk of brittle, order-dependent verifications.

#### Weaknesses Compared to Mocks
- **Maintenance cost.** You must maintain the fake implementation alongside production traits. If the trait evolves, the fake must be updated manually.
- **Limited behavioral coverage.** A fake typically encodes only the paths needed by the current test suite. Mocks can dynamically configure behaviors per test without editing shared code.
- **Risk of drifting from reality.** Because a fake is handwritten, it might omit subtle behavior the real dependency exhibits. Use fixtures and integration tests to guard against divergence.

#### Example: Logging with a Fake vs. a Mock
Consider a service that logs a structured trace message—`"Foo(branch=bar): the foo is quite barred"`—whenever it processes a branch.

**✅ Fake-based test (clean and intention revealing):**
```rust
#[test]
fn test_process_branch_logs_trace_message() {
    // GIVEN: a branch identifier that should trigger logging
    let given_branch_id = "bar".to_string();

    // MOCKING: create test fakes for collaborators
    let fake_logger = FakeLogger::default();

    // SETUP: expose the fake through env_* variables
    let mut env_logger: Box<dyn Logger> = Box::new(fake_logger.clone());

    // SYSTEM UNDER TEST: inject the fake logger
    let sut = BranchProcessor::new(env_logger.as_mut());

    // WHEN: process the branch
    sut.process_branch(&given_branch_id);

    // LOGGING: ensure the trace message was written
    fake_logger.assert_exercised("Foo", "bar");
}

#[derive(Default, Clone)]
struct FakeLogger {
    entries: std::sync::Arc<std::sync::Mutex<Vec<(String, String, String)>>>,
}

impl FakeLogger {
    fn assert_exercised(&self, expected_category: &str, expected_branch: &str) {
        let entries = self.entries.lock().unwrap();
        let entry = entries.first().expect("expected at least one log entry");
        assert_eq!(expected_category, entry.0);
        assert_eq!(expected_branch, entry.1);
        assert_eq!(format!("Foo(branch={expected_branch}): the foo is quite barred"), entry.2);
    }
}
```

---

## **8. Assertions & Variable Naming**
Strict naming patterns for expected and actual values highlight the difference between inputs, outputs, and verifications, which speeds up failure analysis. Mixing literals and setup variables inside assertions hides intent, makes diffs noisy, and increases the chance of asserting against the wrong data.
- **Expected values** must be assigned to `expected_*` variables in the EXPECTATIONS section if you intend to assert on them in the THEN or BEHAVIOR sections.
- **Actual results** must be assigned to `actual_*` variables in the WHEN section.
- **Never assert against literals directly** — use `expected_*` variables.
- **Never assert against `given_*` variables directly** — assign them to `expected_*` variables in the EXPECTATIONS section.
- **Never assert against `env_*` variables directly** — assign them to `actual_*` variables exclusively in the WHEN section and use those.
- **Never assert against `mock_*` variables directly in the THEN section** — assign their observed state to `actual_*` variables first.
- **It is acceptable to assert against `mock_*` or `fake_*` variables directly in the BEHAVIOR section** when using their verification helpers.
- To reiterate: **assertions should never refer to `given_*` or `env_*` values directly**; rely on `expected_*` and `actual_*` variables.

✅ **Good Example:**
```rust
let expected_value = 42;
assert_eq!(expected_value, actual_value);
```
🚫 **Bad Example:**
```rust
assert_eq!(42, actual_value); // ❌ No expected variable
```

---

## **9. Error Handling (`Result`, `panic!`, and `should_panic`)**
Treating error capture as part of the WHEN stage keeps control flow explicit and prevents assertions from being buried inside closures. When the error is not stored and verified deliberately, tests can pass for the wrong reason, masking regressions where different error types or messages are emitted.

When using constructs like `anyhow::Result` or `Result<T, E>`, consider pattern matching in the WHEN section and moving assertions into THEN. For panics, prefer `std::panic::catch_unwind` over `#[should_panic]` when you need to assert on the error payload.

✅ **Good Example:**
```rust
#[test]
fn test_divide_by_zero_returns_error() {
    // GIVEN
    let given_numerator = 10;
    let given_denominator = 0;

    // WHEN
    let actual_error = divide(given_numerator, given_denominator).unwrap_err();

    // EXPECTATIONS
    let expected_message = "cannot divide by zero".to_string();

    // THEN
    assert_eq!(expected_message, actual_error.to_string());
}
```

✅ **Good Example for panics:**
```rust
#[test]
fn test_parser_panics_on_empty_input() {
    // GIVEN
    let given_input = String::new();

    // WHEN
    let actual_panic = std::panic::catch_unwind(|| Parser::parse(&given_input)).unwrap_err();

    // EXPECTATIONS
    let expected_message_fragment = "input cannot be empty".to_string();

    // THEN
    let panic_message = actual_panic.downcast_ref::<String>().unwrap();
    assert!(panic_message.contains(&expected_message_fragment));
}
```

---

## **10. Shared Setup Functions**
Being intentional about shared setup ensures common initialization is truly shared while test-specific context stays near the scenario. Overusing shared setup leads to hidden coupling between tests, complicated fixture state, and surprises when one case mutates shared state that another silently depends on.
- **Avoid `lazy_static!` or global state** for tests. Prefer per-test setup via helper functions that return fresh data structures.
- Use helper functions only for **repeated initialization** that is identical across tests.
- Ensure helpers return owned values so each test can mutate without affecting others.

✅ **Good Example:**
```rust
fn setup_service() -> UserService {
    UserService::default()
}

#[test]
fn test_user_lookup() {
    // GIVEN
    let given_service = setup_service();
    // ...
}
```
🚫 **Bad Example:**
```rust
static mut SERVICE: Option<UserService> = None; // ❌ Shared mutable state across tests
```

---

## **11. Organizing Modules and Re-exports**
Mirroring production modules in the test suite keeps navigation intuitive and tooling-friendly, so developers can jump between code and tests effortlessly. When module structures drift, search results become noisy, automated discovery may misbehave, and contributors struggle to find the right home for new cases.
- **Group related test modules into folders** that match the production structure.
- Ensure **each test module matches the SUT's module path** (e.g., `services::user` → `tests/services/user_tests.rs`).
- Prefer the `mod foo_tests;` declaration pattern in integration tests rather than `pub mod foo_tests` to keep visibility scoped.

✅ **Good Example:**
```rust
// tests/services/mod.rs
mod user_tests;
```
🚫 **Bad Example:**
```rust
// tests/services.rs
mod services_tests; // ❌ Does not mirror module path
```

---

## **12. Managing Copy-Paste Setup**
Copy-paste programming is often the most readable choice for test setup. Start with a "happy path" test that spells out the full arrangement, and copy that scaffolding into edge-case tests, tweaking only what changes for each branch. With disciplined sectioning and modern refactoring tools, duplicating the setup is usually quicker and clearer than inventing cryptic helper methods whose behavior must be reverse-engineered.
- Prefer duplication first, then refactor **only when** the abstraction makes the setup easier to understand than the copied code it replaces.
- When a shared helper truly improves clarity, keep it near the tests that use it and document which scenarios rely on it.
- When you do copy setup code, call out the variations in the section-header comments (e.g. `// GIVEN: user has insufficient balance`). These comments act as signposts so reviewers can immediately see why one test diverges from another despite similar bodies.

---

## **13. Deterministic Tests and Dependency Injection**
Unit tests should be deterministic: the same inputs must produce the same results every run. Nondeterminism from global state, random values, or "current time" APIs usually erodes trust in the suite. Occasionally, allowing nondeterminism in aspects that **should not** affect the outcome is valuable—flaky failures in those cases expose hidden coupling or misunderstood behavior. Treat such flakes as signals to either fix the bug they reveal or document the learning and firm up the contract under test.
- Isolate side effects behind traits and inject them into the system under test. Use dependency injection so the test can supply controlled fakes, especially for time-sensitive behavior.
- Avoid calling `std::time::SystemTime::now`, `rand::random`, or global singletons from production code during tests. Instead, inject a clock, ID generator, or configuration provider that the test can stub.
- For time-sensitive logic, pass a `Clock` trait or similar abstraction. Tests can freeze "now" to a predictable value while production supplies a real clock, making the deterministic path explicit.
- When code mixes deterministic calculations with nondeterministic reads, refactor into two functions: one that gathers inputs (time, randomness, IO) and another that performs pure computation. Unit test the deterministic function directly and cover the integration path separately if needed.

✅ **Example: controlling current time with DI**
```rust
trait Clock {
    fn now(&self) -> std::time::SystemTime;
}

struct InvoiceService<C: Clock> {
    clock: C,
}

impl<C: Clock> InvoiceService<C> {
    fn create_invoice(&self, order: &Order) -> Invoice {
        let generated_at = self.clock.now();
        Invoice::new(order.id, generated_at)
    }
}

#[test]
fn test_create_invoice_uses_provided_clock() {
    // GIVEN
    let given_now = std::time::UNIX_EPOCH + std::time::Duration::from_secs(42);
    let mock_clock = MockClock::new(given_now);

    // SETUP
    let env_clock = mock_clock.clone();

    // SYSTEM UNDER TEST
    let sut = InvoiceService { clock: env_clock };

    // WHEN
    let actual_invoice = sut.create_invoice(&Order::default());

    // EXPECTATIONS
    let expected_generated_at = given_now;

    // THEN
    assert_eq!(expected_generated_at, actual_invoice.generated_at());

    // BEHAVIOR
    mock_clock.assert_called_once();
}
```

---

## **14. Using Comments in Tests**
Documenting test intent with doc comments provides high-level context that complements the structured sections, guiding readers through why a scenario exists. Without these explanations, future maintainers must infer purpose from mechanics alone, risking redundant cases or accidental deletion of critical coverage.
- **Each test module should include a module-level doc comment** summarizing what it covers.
- **Each test function should include a `///` doc comment** explaining the scenario, written in complete sentences.
- Keep doc comments concise but descriptive, mentioning the SUT and the specific behavior.

---

## **15. Increasing Test Clarity with Section Comments**
Narrated section headers transform dense setup or verification code into self-explanatory stories that highlight intent and edge cases. Skipping these comments forces reviewers to reverse-engineer the reason behind each block, slowing code reviews and making it easier for subtle regressions to slip through.

**As a best practice**, when tests have complex parts such as setting up mocks, creating complex `given_` objects, asserting on mock behavior, and so forth, split each such part into a section of its own, and comment what that section is doing. These headers act like a decryption key for the reader: each full-sentence comment explicitly states the intent of the code beneath it, so that even when the implementation is dense or unfamiliar, the reader can immediately understand _why_ a block exists rather than mentally reverse-engineering the setup from the statements themselves.

✅ **Good Example:**
```rust
/// Covers the `is_foo` branch of `UserService`.
#[cfg(test)]
mod is_foo_tests {
    use super::*;

    /// Ensures `is_foo` logs and returns true when FooBar is "foo".
    #[test]
    fn test_is_foo_returns_true_when_foobar_is_foo() {
        // GIVEN: LogLevel::Info is enabled
        let given_info_enabled = true;

        // MOCKING: create a mocked logger that has info enabled
        let mock_logger = MockLogger::new(given_info_enabled);

        // SETUP: expose the mocked logger via env_
        let mut env_logger = mock_logger.clone();

        // SYSTEM UNDER TEST: instantiate the service with injected logger
        let sut = UserService::new(&mut env_logger);

        // WHEN: call `is_foo`
        let actual_result = sut.is_foo();

        // EXPECTATIONS: result is true and a log message is emitted
        let expected_result = true;
        let expected_log_fragment = "is_foo was called".to_string();

        // THEN: verify return value
        assert_eq!(expected_result, actual_result);

        // LOGGING: verify log entry
        mock_logger.assert_logged(&expected_log_fragment);
    }
}
```
🚫 **Bad Example:**
```rust
#[test]
fn test_is_foo() {
    // ... // ❌ Missing descriptive section comments
}
```

---

## **16. SETUP / SUT Dependency Rule**
Reinforcing the separation between SETUP dependencies and the system under test protects against leaky abstractions and keeps constructor wiring transparent. When mocks or fixtures sneak directly into SUT creation, the test intent becomes opaque, accidental coupling increases, and regressions slip in because collaborators are no longer clearly expressed through `env_*` variables.

To re-iterate:

- The **SETUP** section must initialize all dependencies passed to the **SUT** via `env_*` variables.
- The **SYSTEM UNDER TEST** section must construct the **SUT** using only `env_*` variables.
- Mocks or fakes must never be used directly in the **SYSTEM UNDER TEST** section.

---

## **17. Breaking the Rules**
The guidelines above are intentionally prescriptive so tests remain predictable and reviewable. However, real-world systems occasionally demand exceptions.
- Deviation is acceptable when it **improves clarity or determinism** for a specific edge case that the standard pattern cannot express cleanly.
- When you break a rule, **document the rationale inline** (e.g. with a comment or doc string) so future maintainers understand why the exception exists.
- Re-evaluate exceptions periodically. If the surrounding production code changes, the original constraint may disappear and the standard structure can be restored.
