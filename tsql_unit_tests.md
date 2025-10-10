# T-SQL Unit Testing Style Guide

This document defines **best practices** for writing **clear, structured, and maintainable** unit tests in Transact-SQL using [tSQLt](https://tsqlt.org/) or similar testing frameworks.

---

## **Target Audience**

The primary target audience for this document is database developers and Large Language Models (LLMs) acting as coding assistants. By following these guidelines, tests will be **structured, readable, and maintainable**, even for complex stored procedures, functions, and triggers.

These practices may be more prescriptive than human authors typically follow, but they provide an excellent baseline for automated test generation. **Always review generated tests** to ensure they express the intended behavior.

---

## **1. Test Schema Organization**
A predictable schema layout makes it easy to locate the subject under test and guarantees isolation between test fixtures. Misaligned schemas force reviewers to search for test artifacts and risk collisions with production objects.
- Create a dedicated **test schema (a “test class” in tSQLt terminology)** for each production schema and object under test.
- Name the schema by mirroring the production object and appending `Tests` (e.g., production `Sales.Order` → test schema `SalesOrderTests`).
- Nest test schemas within folders that mirror the production object's folder structure when storing scripts in source control.
- Avoid reusing a single schema to test unrelated production objects.
- Keep helper objects (e.g., factory functions) in a nested schema named `<TestSchema>_Helpers` to avoid cluttering the main test schema.

✅ **Good Example:**
```sql
-- For production object dbo.ProcessOrder
CREATE SCHEMA dboProcessOrderTests;
```
🚫 **Bad Example:**
```sql
CREATE SCHEMA UnitTests; -- ❌ Does not indicate which object is covered
```

---

## **2. Test Procedure Naming & Files**
Consistent stored procedure names make the test intent obvious and help tSQLt discovery. Ambiguous names encourage multi-purpose procedures and increase triage time when failures occur.
- Each test stored procedure must be created in the relevant test schema and **named with the prefix `test`**.
- Follow the pattern `test_<Subject>_<ExpectedOutcome>` using snake_case.
- When a test schema exercises multiple production routines, begin the procedure name with the production routine being exercised (e.g., `test_ProcessOrder_returns_success`).
- **One test per procedure.** Do not combine multiple assertions or branches within the same stored procedure.
- Store each procedure’s script in its own file named `<Schema>.<Procedure>.sql` to make code reviews easy.
- Avoid using spaces or camel case in procedure names.

✅ **Good Example:**
```sql
CREATE PROCEDURE dboProcessOrderTests.test_ProcessOrder_returns_success
AS
BEGIN
    -- test body
END;
```
🚫 **Bad Example:**
```sql
CREATE PROCEDURE dboProcessOrderTests.ProcessOrderWorks -- ❌ Missing test_ prefix and expectation
AS
BEGIN
    -- test body
END;
```

---

## **3. Schema-Level Setup Helpers**
Shared initialization belongs in dedicated helpers so each test remains focused on its unique behavior.
- Use `EXEC tSQLt.NewTestClass 'SchemaNameTests';` to register each test schema.
- If repeated setup is needed, define a `[SchemaNameTests].[SetUp]` procedure that inserts baseline data or configures fakes. Keep it short and deterministic.
- Place teardown logic in `[SchemaNameTests].[CleanUp]` when necessary. Prefer transactions that roll back automatically via tSQLt to minimize cleanup code.
- Store helper procedures, functions, or table-valued parameters in a helper schema named `<SchemaNameTests>_Helpers` and reference them from the test body.

---

## **4. Test Procedure Structure**
Segmented sections keep tests self-documenting and prevent important steps from being buried. Every test stored procedure must follow a consistent set of commented regions:

1. **GIVEN**
   - Declare input variables and arrange prerequisite data.
   - Name variables with the prefix `@given` (e.g., `DECLARE @givenOrderId INT = 42;`).
   - Insert seed data with explicit column lists and deterministic values. Avoid relying on identity auto-generation when the value matters to the assertion; assign explicit keys instead.
   - If fixtures are needed, call helper procedures named `Given_*` or use helper schemas.

2. **MOCKING** *(if applicable)*
   - Configure test doubles using tSQLt features such as `tSQLt.FakeTable`, `tSQLt.FakeFunction`, or `tSQLt.SpyProcedure`.
   - Name intermediate objects with the prefix `@mock` or `mock_` to make their purpose obvious.
   - Keep fake configuration minimal: remove columns you do not assert against, and always restore identity columns when required by the SUT.

3. **SETUP**
   - Prepare environment-level constructs not covered by GIVEN/MOCKING, such as enabling identity insert, creating temp tables, or populating configuration tables.
   - Prefix variables with `@env` to differentiate them from inputs (e.g., `DECLARE @envExpectedRowCount INT = 1;`).
   - Assign real table or procedure references to descriptive variables if the SUT expects them as parameters.

4. **SYSTEM UNDER TEST**
   - Identify the routine being exercised. When calling stored procedures, use the full schema-qualified name.
   - Assign the target name to a variable like `DECLARE @sut NVARCHAR(128) = 'dbo.ProcessOrder';` when passing into helper routines such as `tSQLt.ExpectException`.

5. **WHEN**
   - Execute the system under test, capturing outputs in `@actual*` variables or temporary tables.
   - Wrap the execution in `BEGIN TRY ... END TRY / BEGIN CATCH ... END CATCH` if you expect exceptions, but keep exception handling minimal—prefer `tSQLt.ExpectException` when possible.

6. **EXPECTATIONS**
   - Declare expected values in variables prefixed with `@expected` or populate temporary tables named `#expected_*`.
   - Build expected result sets using deterministic ordering and explicit column lists.

7. **THEN**
   - Assert outcomes using tSQLt assertions (e.g., `tSQLt.AssertEquals`, `tSQLt.AssertEqualsTable`, `tSQLt.AssertEqualsString`).
   - **Never assert directly against literals.** Always compare against `@expected` variables or `#expected_*` tables.

8. **BEHAVIOR** *(if mocks were used)*
   - Verify spy procedures or captured executions using `tSQLt.AssertEquals` against spy tables.
   - Reset fakes or spies if the test altered them.

Use uppercase comment headers beginning with `--` to delineate sections, and separate them with blank lines:

```sql
CREATE PROCEDURE dboProcessOrderTests.test_ProcessOrder_returns_success
AS
BEGIN
    -- GIVEN: a pending order
    DECLARE @givenOrderId INT = 42;
    INSERT INTO dbo.Orders (OrderId, Status)
    VALUES (@givenOrderId, 'Pending');

    -- SYSTEM UNDER TEST: target procedure
    DECLARE @sut NVARCHAR(128) = 'dbo.ProcessOrder';

    -- WHEN: execute the procedure
    EXEC dbo.ProcessOrder @OrderId = @givenOrderId;

    -- EXPECTATIONS: define expected row
    DECLARE @expectedStatus NVARCHAR(20) = 'Completed';

    -- THEN: assert order status
    DECLARE @actualStatus NVARCHAR(20);
    SELECT @actualStatus = Status FROM dbo.Orders WHERE OrderId = @givenOrderId;
    EXEC tSQLt.AssertEquals @expected = @expectedStatus, @actual = @actualStatus;
END;
GO
```

---

## **5. Fixtures & Deterministic Data**
Realistic yet deterministic fixtures prevent brittle assertions while keeping tests expressive.
- Centralize reusable data builders in helper procedures (e.g., `[dboProcessOrderTests_Helpers].[Given_Order]`).
- Always insert with explicit column lists and stable values. Avoid `SELECT *`.
- Prefer using surrogate keys that are easy to read (e.g., `1001` instead of `NEWID()`), unless the SUT explicitly requires GUIDs.
- When time-based columns are required, use deterministic values via `tSQLt.FreezeTime` or assign explicit datetime literals.

---

## **6. Mocking & Spies**
T-SQL lacks traditional mock frameworks, so rely on tSQLt’s faking and spying utilities to isolate dependencies.
- Use `tSQLt.FakeTable` to isolate the table under test and prevent triggers or constraints from interfering. Provide explicit column definitions that match the production table.
- When faking views or functions, ensure the fake returns only the columns the SUT relies on. Extra columns can mask missing assertions.
- Use `tSQLt.SpyProcedure` to capture parameters passed to downstream procedures. Always assert on the spy result set in the **BEHAVIOR** section.
- Reset spies by dropping the spy or re-executing the original procedure at the end of the test if the fake is global.
- Do not fake the SUT itself. Only fake dependencies one level away.

✅ **Good Example:**
```sql
-- MOCKING: capture calls to dbo.LogActivity
EXEC tSQLt.SpyProcedure 'dbo.LogActivity';

-- WHEN
EXEC dbo.ProcessOrder @OrderId = @givenOrderId;

-- BEHAVIOR: verify log entry
SELECT * INTO #actualLog FROM dbo.LogActivity_SpyResult;
SELECT * INTO #expectedLog FROM (VALUES (@givenOrderId, 'ProcessOrder')) AS v(OrderId, Operation);
EXEC tSQLt.AssertEqualsTable '#expectedLog', '#actualLog';
```

---

## **7. Assertions & Variable Naming**
Clear naming distinguishes between setup data, expectations, and outcomes.
- Prefix all actual values with `@actual` or `#actual_*`. Populate them immediately after executing the SUT.
- Prefix expected values with `@expected` or `#expected_*`. Declare them in the EXPECTATIONS section.
- Never compare `@given` values directly to actual results; assign them to `@expected` variables first.
- Prefer `tSQLt.AssertEqualsTable` when comparing result sets. Populate expected data in temporary tables to control ordering.
- When verifying row counts, use `tSQLt.AssertEquals` rather than `COUNT(*)` comparisons in ad-hoc IF statements.

---

## **8. Handling Exceptions & Messages**
Tests that verify error paths must capture exception details explicitly.
- Use `tSQLt.ExpectException` before executing the SUT to assert that an exception is raised. Provide the expected message text and error number when possible.
- When expecting no exception, avoid blanket TRY/CATCH blocks—let unexpected failures surface naturally.
- Store the return code of procedures in `@actualReturnCode` variables and assert against `@expectedReturnCode` in the THEN section.
- For RAISERROR or THROW assertions, capture the output of `tSQLt.GetLastError` if detailed verification is required.

✅ **Good Example:**
```sql
-- EXPECTATIONS: expect a specific error
EXEC tSQLt.ExpectException @ExpectedMessage = 'Order not found', @ExpectedSeverity = 16;

-- WHEN: execute SUT that should raise
EXEC dbo.ProcessOrder @OrderId = @givenMissingOrderId;
```

---

## **9. SetUp / CleanUp Procedures**
Shared setup must remain lightweight and deterministic.
- Keep `[SchemaNameTests].[SetUp]` focused on registering common fakes or baseline reference data. Do not insert scenario-specific data here; that belongs in the GIVEN section of each test.
- Avoid shared mutable state. Reset tables via `tSQLt.FakeTable` or `DELETE` inside SetUp to ensure each test starts clean.
- Use `[SchemaNameTests].[CleanUp]` sparingly. Prefer relying on tSQLt’s automatic transaction rollback to revert changes.

---

## **10. Breaking the Rules (Rarely)**
Occasionally a test demands deviation (e.g., complex cursor behavior or verifying global temporary objects). In those cases:
- Document the reason with an inline comment beginning `-- DEVIATION:`.
- Keep the deviation local to the specific step. Do not propagate relaxed patterns to other tests without justification.
- If multiple tests require the same deviation, update helper procedures instead of copy/pasting bespoke logic.

---

Adhering to this style guide keeps T-SQL tests deterministic, intention-revealing, and easy to maintain—mirroring the discipline expected in application-level unit tests.
