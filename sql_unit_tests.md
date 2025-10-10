# SQL Unit Testing Style Guide

This document adapts the **C# Unit Testing Style Guide** principles to the SQL domain. It defines **best practices** for writing
**clear, deterministic, and maintainable** unit tests for relational database code using frameworks such as [tSQLt](https://tsqlt.org/),
pgTAP, or any harness that executes stored procedures, functions, and triggers in isolation.

---

## **Target Audience**

The primary target audience for this document is database developers and Large Language Models (LLMs) assisting with SQL test
authoring. By following these guidelines, tests will be **structured, readable, and maintainable** while remaining faithful to
the production schema they validate.

These practices intentionally favor structure and verbosity. Human reviewers should always inspect generated tests to ensure the
chosen fixtures, fake tables, and assertions truly exercise the intended execution path.

---

## **1. Test Schema Organization**
Consistent schema organization makes it obvious which production object a test targets, allowing reviewers to locate stored
procedures or functions without cross-referencing multiple files. Mixing unrelated objects into a single schema obscures
ownership and complicates cleanup between runs.
- Create a **dedicated test schema per production schema**, appending a `_Tests` suffix (e.g. `Sales` → `Sales_Tests`).
- When production code uses nested schemas (e.g. `Sales.Reporting` in SQL Server or logical namespaces in PostgreSQL), mirror
  the full hierarchy before adding `_Tests`.
- Keep a **one-to-one mapping between directories and schemas** in source control so moving a file preserves schema alignment.
- Avoid placing tests for different production schemas in the same test schema—create additional test schemas instead.
- Test projects should include a setup script that creates all `_Tests` schemas and grants execute rights only to the harness.

✅ **Good Example:**
```sql
CREATE SCHEMA Sales_Tests AUTHORIZATION dbo;
GO
```
🚫 **Bad Example:**
```sql
CREATE SCHEMA Tests; -- ❌ Provides no context for the covered production schema
GO
```

---

## **2. Test Class (Schema) Naming and Files**
Test stored procedures live inside test schemas; treating each schema as a test class keeps intent crisp. Without a predictable
structure, teams waste time searching for coverage gaps or duplicate effort.
- Each SQL file must create or alter **only one test stored procedure**. Name the file `<ObjectName>[__<RegionName>][__<Behavior>]_Tests.sql`.
- Test procedures must be created inside their corresponding `_Tests` schema.
- When a test validates an entire stored procedure or function, use the naming pattern `<ObjectName>_Tests` for the SQL file and
  `Test_<Behavior>` for the procedure (details in Section 4).
- When a production object has multiple logical regions (e.g. `#region` blocks in C# mirrored by comment sections in SQL), place
  tests in separate files with descriptive suffixes (e.g. `ProcessOrder__Validation_Tests.sql`).
- Do **not** combine test cases for unrelated production objects into the same file.
- Store test files under a directory structure mirroring the production schema hierarchy.

✅ **Good Example:**
```
Sales/
│── ProcessOrder.sql
Sales_Tests/
│── ProcessOrder_Tests.sql
│── ProcessOrder__Validation_Tests.sql
```
🚫 **Bad Example:**
```
Sales_Tests/
│── MiscTests.sql  -- ❌ Contains mixed tests for many objects
```

---

## **3. Region-Focused Test Schemas**
Production procedures often contain logical sections for validation, branching, and persistence. Mirroring those regions in the
test suite keeps the feedback loop tight by ensuring every region has an intentionally scoped test procedure.
- Introduce a `<RegionName>` segment only when the production SQL clearly delineates the same responsibility (e.g. a validation
  block).
- When a procedure contains multiple meaningful regions, create a separate test file per region following the naming rules above.
- Avoid reusing a `<RegionName>` for unrelated blocks. Each region should map to a single test file.
- When region helpers are private (e.g. local temporary tables), test them via the public procedure that invokes them instead of
  calling helpers directly.

✅ **Good Example:**
```
Sales_Tests/
│── ProcessOrder__Validation_Tests.sql
│── ProcessOrder__Discounts_Tests.sql
```
🚫 **Bad Example:**
```
Sales_Tests/
│── ProcessOrder__ValidationAndDiscounts_Tests.sql -- ❌ Merges multiple regions
```

---

## **4. Test Procedure Naming**
Clear procedure names document the behavior under test and the expected outcome, making failures self-explanatory. Vague or
inconsistent names hide gaps in coverage and complicate triage when reviewing harness output.
- Each test stored procedure **must start with** `Test_`.
- Use underscores to separate phrases in the test name; consecutive underscores are prohibited.
- When multiple production routines are tested within the same file, the test name **must include** the target routine name as
  the first segment after `Test_` (e.g. `Test_ProcessOrder_ReturnsError_WhenCustomerMissing`).
- If the file covers a single production routine, omit the routine name from the test procedure (e.g. `Test_ReturnsError_WhenCustomerMissing`).
- Prefer descriptive names over brevity—communicate the specific branch exercised and the expected result.
- Do **not** reuse phrases already present in the file name; the combination of file name and procedure name must uniquely
  describe the scenario without redundancy.

✅ **Good Example:**
```sql
CREATE OR ALTER PROCEDURE Sales_Tests.ProcessOrder_Tests.Test_ReturnsError_WhenCustomerMissing
AS
BEGIN
    -- ...
END;
GO
```
🚫 **Bad Example:**
```sql
CREATE OR ALTER PROCEDURE Sales_Tests.ProcessOrder_Tests.CheckProcessOrder
AS
BEGIN
    -- ❌ Missing Test_ prefix and lacks intent
END;
GO
```

---

## **5. Test Procedure Structure**
Standardized sections carve complex tests into digestible steps, clarifying how data flows through staging tables, mocks, and the
system under test (SUT). Without this structure, tests devolve into monolithic scripts that obscure setup, execution, and
assertions. The ordering rules below are intentionally strong to promote reliable habits—deviate only when strictly necessary.

Within every test procedure, separate sections with full-sentence SQL comments (e.g. `-- GIVEN: A valid sales order header`). Use
blank lines between sections and remove empty sections entirely.

1. **GIVEN**
   - Declare and initialize input variables with the prefix `@given`.
   - Create fixture tables or insert seed data required by the scenario.
   - Configure `Fixture` helper procedures or functions before use.
   - The GIVEN section must be first when present and **may not be merged** with other sections.

2. **MOCKING** *(if applicable)*
   - Use framework facilities (e.g. `tSQLt.FakeTable`, `pgTAP` mocks) to isolate dependencies.
   - Name mock objects with the prefix `@mock` or create temp tables prefixed with `mock_`.
   - Configure fake tables and default rows within this section.
   - The MOCKING section occurs before SETUP and **may not be merged** with other sections.

3. **SETUP**
   - Prepare the execution environment (e.g. assign default constraints, enable identity insert, configure session settings).
   - Assign environment artifacts to variables prefixed with `@env`.
   - Apply data manipulations that transform GIVEN data for consumption by the SUT.
   - Do **not** reference `@mock` variables directly from the SUT; convert them to real tables or environment artifacts here.
   - The SETUP section follows GIVEN (or MOCKING when present) and **may not be merged** with other sections.

4. **SYSTEM UNDER TEST**
   - Assign the procedure or function being tested to an alias variable named `@sut` when possible.
   - For frameworks like tSQLt, this section typically consists of a comment noting the SUT (e.g. `-- SYSTEM UNDER TEST: Sales.ProcessOrder`).
   - Do not call the SUT in this section; invocation happens in WHEN.

5. **WHEN**
   - Execute the production routine under test.
   - Capture outputs into variables prefixed with `@actual` or persist them in temporary tables named `actual_*`.
   - If the SUT populates session state (e.g. tables), copy the relevant rows into explicit `actual_*` tables immediately for clarity.
   - The WHEN section **must not be merged** with other sections.

6. **EXPECTATIONS**
   - Declare expected values in variables prefixed with `@expected`.
   - Populate expected result tables named `expected_*` mirroring the column order of actual tables.
   - Avoid asserting directly against literals in THEN; store them here first.
   - The EXPECTATIONS section appears after WHEN and **must not be merged** with other sections.

7. **THEN**
   - Perform assertions comparing actual and expected values using the testing framework (e.g. `tSQLt.AssertEquals`, `pgTAP.ok`).
   - Always reference `@expected*` variables or `expected_*` tables—never compare actual results to raw literals.
   - Keep assertion logic readable by grouping related checks together, separated by blank lines.

8. **LOGGING** *(if applicable)*
   - Verify logging tables or audit trails that confirm the intended branch executed.
   - Prefer checking log rows over mocking procedure calls when possible; logs provide higher-fidelity branch verification.

9. **BEHAVIOR** *(if applicable)*
   - Assert interactions with mocked dependencies (e.g. verifying that a fake table received exactly one row).
   - Ensure every `FakeTable` or mock configuration has a corresponding verification query.
   - Treat unverified mocks as test failures.

✅ **Good Example:**
```sql
CREATE OR ALTER PROCEDURE Sales_Tests.ProcessOrder_Tests.Test_ReturnsError_WhenCustomerMissing
AS
BEGIN
    -- GIVEN: A sales order without a customer reference
    DECLARE @givenOrderId INT = 1;
    INSERT INTO Sales.Orders (OrderId, CustomerId, Status)
    VALUES (@givenOrderId, NULL, 'Draft');

    -- SYSTEM UNDER TEST: Sales.ProcessOrder
    DECLARE @sut NVARCHAR(128) = 'Sales.ProcessOrder';

    -- WHEN: Attempt to process the order
    EXEC Sales.ProcessOrder @OrderId = @givenOrderId;
    SELECT OrderId, Status INTO #actualOrders FROM Sales.Orders WHERE OrderId = @givenOrderId;

    -- EXPECTATIONS: The order remains in Draft status with a validation error logged
    DECLARE @expectedStatus NVARCHAR(20) = 'Draft';
    SELECT OrderId, @expectedStatus AS Status INTO #expectedOrders;

    -- THEN: The order status is unchanged
    EXEC tSQLt.AssertEqualsTable '#expectedOrders', '#actualOrders';
END;
GO
```
🚫 **Bad Example:**
```sql
CREATE OR ALTER PROCEDURE Sales_Tests.ProcessOrder_Tests.Test_ProcessOrder
AS
BEGIN
    EXEC Sales.ProcessOrder @OrderId = 1; -- ❌ No arrangement, no assertions
END;
GO
```

---

## **6. Fixtures and Shared Data**
Reusable fixtures encourage realistic, DRY data that mirrors production scenarios while keeping tests focused on behavior rather
than plumbing.
- Prefer fixture helper procedures (e.g. `Fixtures.CreateCustomer`) over inline object creation.
- Store fixtures in a dedicated schema (e.g. `Sales_Fixtures`) or within a shared module accessible to tests.
- Name fixture procedures with the prefix `Fixture_` and assign results to `@given` variables before use.
- When fixtures return tables, insert them into `given_*` temp tables to make dependencies explicit.
- Avoid mutating fixture data directly; clone rows into test-local tables before modification.

✅ **Good Example:**
```sql
CREATE OR ALTER PROCEDURE Sales_Fixtures.Fixture_Order
AS
BEGIN
    RETURN SELECT 1 AS OrderId, 42 AS CustomerId, 'Draft' AS Status;
END;
GO

CREATE OR ALTER PROCEDURE Sales_Tests.ProcessOrder_Tests.Test_ReturnsError_WhenCustomerMissing
AS
BEGIN
    -- GIVEN: A draft order without a customer
    SELECT OrderId, CustomerId, Status
    INTO #givenOrders
    FROM Sales_Fixtures.Fixture_Order();

    UPDATE #givenOrders SET CustomerId = NULL;
    INSERT INTO Sales.Orders SELECT * FROM #givenOrders;
    -- ...
END;
GO
```
🚫 **Bad Example:**
```sql
INSERT INTO Sales.Orders VALUES (1, NULL, 'Draft'); -- ❌ Inline literal data duplicated across tests
```

---

## **7. Isolation and Cleanup**
Database state leaking between tests causes nondeterministic failures and brittle suites. Enforce isolation aggressively.
- Wrap each test in a transaction and roll it back at the end (most SQL test frameworks do this automatically).
- Avoid relying on identity values or timestamps generated in previous tests; create fresh rows per test.
- Drop or truncate temp tables at the end of each test when the framework does not handle cleanup.
- Never share permanent tables between tests for storing actual results—use temporary tables or table variables.
- Ensure fake tables or mocked dependencies are reverted to their original state after each test.

---

## **8. Breaking the Rules**
Occasionally a test may need to deviate from these standards (e.g. verifying complex dynamic SQL or replication behavior).
- Document deviations with explicit comments (e.g. `-- NOTE: SETUP merged with GIVEN due to transactional constraint`).
- Keep deviations localized to a single test procedure; do not generalize them into helpers without peer review.
- Revisit exceptions periodically to ensure they remain necessary as the codebase evolves.

---

By following these conventions, SQL unit tests inherit the clarity and rigor of the C# guidelines while respecting the unique
constraints of relational databases. The result is a consistent, reviewable test suite that gives rapid feedback on stored
procedures, functions, and triggers without sacrificing maintainability.
