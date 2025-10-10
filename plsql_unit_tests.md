# PL/SQL Unit Testing Style Guide

This document defines **best practices** for writing **clear, structured, and maintainable** unit tests in PL/SQL using frameworks such as [utPLSQL](https://utplsql.org/).

---

## **Target Audience**

The primary target audience for this document is database developers and Large Language Models (LLMs) acting as coding assistants. By following these guidelines, tests will be **structured, readable, and maintainable**.

These practices may be more prescriptive than hand-written unit tests but are ideal for automated generation and team consistency. **Review all test packages carefully**—human oversight remains essential.

---

## **1. Test Schema & Package Organization**

Consistent placement of test code keeps suites discoverable and simplifies execution through utPLSQL runners. Scattering tests across schemas or packages makes CI configuration fragile and slows reviewers.

- Place all unit test packages in a dedicated schema (e.g., `app_test`) or in a schema whose name mirrors the production schema with a `_test` suffix.
- Mirror the production package structure: for each production package `app.order_service`, create a test package `app_test.order_service_test`. 
- Keep a **one-to-one mapping between production packages and test packages**. Do not combine tests for multiple production packages inside a single package.
- Organize source files (if stored in version control) so that the directory structure mirrors the schema and package names. 

✅ **Good Example:**
```sql
CREATE OR REPLACE PACKAGE app_test.order_service_test IS
END order_service_test;
```
🚫 **Bad Example:**
```sql
CREATE OR REPLACE PACKAGE tests IS -- ❌ Does not mirror the production package
END tests;
```

---

## **2. Test Package & File Naming**

Predictable names make it obvious which package a suite validates and help utPLSQL automatically discover tests.

- Each file must contain a single test package and package body.
- **Test packages must end in `_test`** (e.g., `customer_repo_test`).
- If the tests cover a specific procedure or function, append its name: `<package>_<procedure>_test`.
- Test package names should not contain double underscores or CamelCase; prefer lower snake case following PL/SQL conventions.
- File names must match the package name (e.g., `order_service_test.pks` and `order_service_test.pkb`).
- Store test packages alongside production code in a parallel directory (e.g., `src/test/sql/app/order_service_test.pkb`).

✅ **Good Examples:**
```text
order_service_test.pkb
order_service_calculate_tax_test.pkb
```
🚫 **Bad Example:**
```text
OrderServiceTests.sql -- ❌ Wrong casing and suffix
```

---

## **3. Procedure-Focused Test Packages**

Complex production packages may contain several public entry points. Breaking tests into procedure-focused packages keeps scenarios isolated and makes failure triage straightforward.

- Introduce a `<procedure>_test` suffix when the production package exposes multiple unrelated procedures.
- Do not mix multiple procedures in the same test package if their data setup diverges substantially.
- If a procedure only has a handful of branches, keep tests in the package-wide suite to avoid needless files.
- Focus procedure-specific packages on public API behavior—test helpers through the public API that exercises them.

✅ **Good Example:**
```text
order_service_test.pkb       -- covers small package
order_service_ship_order_test.pkb
order_service_calculate_tax_test.pkb
```
🚫 **Bad Example:**
```text
order_service_misc_tests.pkb -- ❌ Combines unrelated procedures
```

---

## **4. Test Procedure Naming**

Test procedure names serve as executable documentation. Clear naming communicates the behavior and expected outcome, so test results immediately tell reviewers what failed.

- Each test procedure **must start with** `test_`.
- Use lowercase snake case and underscores to separate phrases.
- Start the name with the production function or procedure when multiple members are tested in the same package: `test_ship_order_returns_tracking_number`.
- If the entire package focuses on a single procedure, omit the procedure name in the test procedure to avoid redundancy: `test_returns_error_when_customer_missing`.
- Avoid vague names such as `test_happy_path` or `test_works`.
- Keep each test scenario in its own procedure; do not combine multiple branches in a single test.
- Avoid parameterized procedures; create explicit test procedures instead.

✅ **Good Example:**
```sql
PROCEDURE test_ship_order_returns_tracking_number;
```
🚫 **Bad Example:**
```sql
PROCEDURE check_shipping; -- ❌ Missing test_ prefix and unclear intent
```

---

## **5. Test Procedure Sectioning**

Consistent comment-based sections make PL/SQL tests easy to scan and enforce a logical flow from setup to verification.

Each test procedure must follow the sections below, separated by comment headers. Use all caps and descriptive sentences (e.g., `-- GIVEN: an existing customer`). Omit sections that are not needed.

1. **GIVEN**
   - Declare base data and input parameters.
   - Prefix variables with `given_`.
   - Insert or update fixture data in transactional tables here.
   - Never merge GIVEN with other sections.

2. **MOCKING**
   - Configure fake packages, substitution variables, or utPLSQL `anydata` stubs.
   - Prefix mock objects or tables with `mock_`.
   - This section is optional and must not be merged with others.

3. **SETUP**
   - Prepare the database environment: seed reference data, enable contexts, configure session settings.
   - Prefix environment variables with `env_`.
   - Assign mock objects to environment variables (e.g., `env_logger := mock_logger;`).
   - Keep SETUP separate from GIVEN and MOCKING.

4. **SYSTEM UNDER TEST**
   - Declare the package, procedure, or object under test using `sut_` variables where helpful.
   - This section must not reference `mock_` variables directly; use the environment variables configured in SETUP.

5. **WHEN**
   - Execute the operation under test.
   - Capture results in variables prefixed with `actual_`.
   - When exceptions are expected, capture them using utPLSQL constructs (see Section 9).

6. **EXPECTATIONS**
   - Define expected values in variables prefixed with `expected_` before asserting.
   - Do not reference `actual_` variables here.

7. **THEN**
   - Perform assertions with utPLSQL (`ut.expect(...)` or `ut_assert` packages) comparing `actual_` and `expected_` values.
   - Never compare directly against literals; use `expected_` variables.

8. **LOGGING**
   - Validate instrumented logging (e.g., rows in a logging table or output captured via DBMS_OUTPUT) if relevant.
   - Prefer verifying log rows over mock verifications when they clearly indicate branch coverage.

9. **BEHAVIOR**
   - Verify mock interactions (e.g., check that a substitution API or instrumentation table recorded a call).
   - Required whenever mocks are used.

✅ **Good Example:**
```sql
PROCEDURE test_calculate_tax_returns_rate IS
   -- GIVEN: an order in Texas
   given_order_id   orders.id%TYPE;
   given_state_code orders.state_code%TYPE := 'TX';
   expected_rate    NUMBER;
BEGIN
   -- GIVEN: create base order data
   INSERT INTO orders(state_code) VALUES(given_state_code)
      RETURNING id INTO given_order_id;

   -- SYSTEM UNDER TEST: target package
   sut_order_pkg := order_service;

   -- WHEN: calculate tax
   actual_rate := sut_order_pkg.calculate_tax(given_order_id);

   -- EXPECTATIONS: derive expected values
   expected_rate := 0.0825;

   -- THEN: assert on returned tax rate
   ut.expect(actual_rate).to_equal(expected_rate);
END;
```
🚫 **Bad Example:**
```sql
PROCEDURE test_calculate_tax IS
BEGIN
   ut.expect(order_service.calculate_tax(1)).to_equal(0.0825); -- ❌ no sections, literals in assertion
END;
```

---

## **6. Fixtures: Best Practices**

Reusable fixtures reduce duplication and make tests resilient to schema changes. Inline data creation leads to brittle tests and inconsistent assumptions.

- Store reusable data creation helpers in dedicated packages (e.g., `fixtures.customer_factory`).
- Instantiate fixtures in the GIVEN or SETUP section and mutate them per scenario as needed.
- Prefer fixture helpers over raw `INSERT` statements when creating complex entity graphs.
- Ensure fixture packages live in the test schema to avoid polluting production packages.

✅ **Good Example:**
```sql
given_customer := customer_factory.create_standard_customer();
```
🚫 **Bad Example:**
```sql
INSERT INTO customers(customer_id, name, status) VALUES(1, 'Bob', 'ACTIVE'); -- ❌ repeated hard-coded data
```

---

## **7. Mocking & Fakes**

PL/SQL lacks native mocking frameworks, so tests often rely on substitution tables, logging hooks, or fake packages. Disciplined patterns keep tests maintainable.

- Prefer real collaborators or fixture helpers when practical.
- Use utPLSQL `anydata` expectations, temporary tables, or controlled contexts to simulate collaborators.
- Name fake implementations with a `fake_` prefix, and create them in the MOCKING section.
- Assign fakes to environment variables (e.g., `env_email_sender := fake_email_sender;`) before invoking the SUT.
- Encapsulate verification logic inside the fake where possible (e.g., `fake_email_sender.assert_sent_to(expected_address);`).
- When using instrumentation tables for verification, truncate them in GIVEN and assert their contents in BEHAVIOR.

✅ **Good Example:**
```sql
fake_notifier := notifier_fake.create;
env_notifier := fake_notifier;
...
fake_notifier.assert_notified(expected_user_id);
```
🚫 **Bad Example:**
```sql
notifier_pkg.send_notification := 1; -- ❌ Mutating production package state without isolation
```

---

## **8. Assertions & Variable Naming**

Clear naming distinguishes inputs, outputs, and expectations, making failure analysis fast.

- Store expected values in `expected_` variables before asserting.
- Capture outputs in `actual_` variables immediately after executing the SUT.
- Never assert against literals, `given_`, or `env_` variables directly—always compare `actual_` to `expected_`.
- Use utPLSQL assertions (`ut.expect`, `ut_assert`) in the THEN section. Avoid `DBMS_OUTPUT` for validations.

✅ **Good Example:**
```sql
ut.expect(actual_total).to_equal(expected_total);
```
🚫 **Bad Example:**
```sql
ut.expect(actual_total).to_equal(100); -- ❌ literal instead of expected_ variable
```

---

## **9. Exception Handling**

Handle expected exceptions in the WHEN section so that control flow and validation remain explicit.

- Use `ut.expect` with the `.to_raise` or `.to_raise_exact` helpers to capture exceptions.
- Store the raised exception or error code in an `actual_` variable if additional assertions are required.
- Assert on exception type, SQLCODE, and message in the THEN section using `expected_` variables.

✅ **Good Example:**
```sql
ut.expect(
   WHEN => order_service.cancel_order(given_order_id)
).to_raise_exact(
   expected_error => expected_exception
);
```
🚫 **Bad Example:**
```sql
BEGIN
   order_service.cancel_order(given_order_id);
   ut.fail('Expected error');
EXCEPTION
   WHEN OTHERS THEN NULL; -- ❌ swallows the error without assertions
END;
```

---

## **10. Setup & Teardown Hooks**

Hooks such as `beforeall`, `beforeeach`, and `aftereach` can hide dependencies if overused. Prefer explicit setup inside each test unless shared initialization is truly common.

- Minimize use of utPLSQL setup hooks; use fixtures instead.
- When hooks are necessary, limit them to idempotent tasks (e.g., seeding reference data).
- Avoid storing per-test state in package globals initialized by hooks.

---

## **11. Transaction Management**

Reliable tests must leave the database in a clean state.

- Wrap each test in an autonomous transaction or rely on utPLSQL's automatic rollback to ensure isolation.
- Explicitly call `ROLLBACK` in `aftereach` hooks when using manual transaction control.
- Never commit inside a unit test.

---

## **12. Deterministic Data & Time**

- Replace calls to `SYSDATE`, `SYSTIMESTAMP`, or sequences with deterministic substitutes inside tests.
- Use fixtures or override session contexts to supply stable timestamps and IDs.
- Assert against deterministic values defined in `expected_` variables.

---

## **13. Randomness & External State**

- Do not depend on external systems (files, network, other schemas) unless explicitly mocked.
- Stub out random generators by injecting deterministic values through environment variables or fixture packages.

---

## **14. Refactoring Test Code**

- Keep assertion helpers inside the test schema (e.g., `assertions.order_assert`).
- Extract repeated setup into reusable procedures that still follow the GIVEN/MOCKING/SETUP pattern.
- Document complex fixtures or fakes with clear comments.

---

## **15. Working with Legacy Packages**

- If production code uses shared state (package variables), reset that state in GIVEN before invoking the SUT.
- Wrap legacy APIs with thin facades to make injection and substitution easier.
- When tests cannot follow all rules (e.g., because the SUT commits internally), document the deviation in comments above the affected section.

---

## **16. Breaking the Rules**

Occasionally, constraints such as third-party packages or autonomous transactions make strict adherence impossible.

- Document any intentional deviation using an inline comment beginning with `-- DEVIATION:` explaining the rationale.
- Keep deviations as isolated as possible and prefer local suppression over global changes.

---

Adopting these practices ensures PL/SQL unit tests remain **readable**, **predictable**, and **maintainable**, mirroring the rigor expected in modern application code.
