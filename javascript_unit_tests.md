# JavaScript Unit Testing Style Guide

This document defines **best practices** for writing **clear, structured, and maintainable** unit tests in JavaScript using Jest,
Vitest, or a comparable runner.

---

## Target Audience

The primary target audience for this guide is developers and Large Language Models (LLMs) acting as coding assistants. Following
these guidelines keeps JavaScript tests **readable, deterministic, and easy to maintain**.

LLM-generated tests should be reviewed carefully by humans before merging. Automated helpers frequently produce syntactically
valid code that violates naming or structural conventions—use this document as the checklist when reviewing output.

---

## 1. Test Module Organization
Mirroring the production module layout keeps readers oriented and lets IDE tooling jump between code and tests. Mismatched folder
structures make it hard to discover coverage gaps and encourage duplicated work.
- Every test file must sit alongside the module under test using the same directory structure (e.g. `src/services/userService.js`
  → `tests/services/userService.test.js`).
- Append `.test.js` (or `.spec.js`) to the production module name. Prefix-based patterns like `test_userService.js` are not
  permitted.
- Never mix tests for unrelated modules in the same file; create one test file per production module.
- Keep helper utilities (e.g. fixture factories) inside a dedicated `tests/helpers/` or `tests/__fixtures__/` folder so the main
  test files stay focused on behaviors.

✅ **Good Example:**
```
src/
└── services/
    └── userService.js
tests/
└── services/
    └── userService.test.js
```

🚫 **Bad Example:**
```
tests/
└── all_tests.js // ❌ mixes multiple modules
```

---

## 2. Test Suite Naming and Files
Consistent suite names make it obvious which production code a test exercises, so reviewers can spot missing scenarios quickly.
Ambiguous suite names hide responsibilities and slow down debugging.
- Each test file must export a top-level `describe()` whose name mirrors the production module (e.g. `describe('UserService', …)`).
- If a file covers a single exported function, the suite must be named `<FunctionName> function` (e.g. `describe('hashPassword function', …)`).
- Nest additional `describe()` blocks to mirror significant groupings in the production module (e.g. specific method groups or
  behaviors). Avoid anonymous or generic `describe()` names like `Utilities` or `Happy Path`.
- Keep a **one-to-one mapping between describe blocks and code responsibilities.** Do not group tests for different exports in the
  same `describe()`.

✅ **Good Example:**
```js
describe('UserService', () => {
  describe('createUser', () => {
    // …
  });
});
```

🚫 **Bad Example:**
```js
describe('Tests', () => {
  // ❌ No connection to production module
});
```

---

## 3. Focused Describe Blocks
Mirroring discrete responsibilities with nested `describe()` blocks keeps suites small and intention revealing.
- Introduce a nested `describe('<MethodName>', …)` block whenever the production module exposes multiple functions.
- If a function has several execution branches, use nested describes such as `describe('when the token is valid', …)` to isolate
  their tests.
- Never reuse a nested describe name for unrelated branches.
- Keep each `describe()` scoped to a single branch or method; do not merge multiple behaviors into one block.

---

## 4. Test Case Naming
Clear test names document the behavior under test and the expected outcome, making it easier to diagnose failures.
- Every test case must use `test()` (or `it()`) with a description string that **starts with** `Test_`.
- Use underscores to separate major phrases, following the pattern `Test_<Subject>_<Behavior>_<Outcome>`.
- When all cases inside a `describe('<MethodName>')` block target the same method, omit the method name from the description and
  focus on the branch (e.g. `Test_ReturnsToken_WhenCredentialsAreValid`).
- Avoid vague or duplicated names. Each test must describe the precise branch it covers, not merely that something "works".
- Keep test bodies focused on a single behavior; do not combine multiple WHEN/THEN flows in one test.

✅ **Good Example:**
```js
test('Test_ReturnsToken_WhenCredentialsAreValid', async () => {
  // …
});
```

🚫 **Bad Example:**
```js
test('should work', () => {
  // ❌ Vague and missing Test_ prefix
});
```

---

## 5. Test Method Sectioning
Standardized sections carve complex tests into digestible steps. Without structure, inputs, mocks, and assertions blur together,
making regressions harder to diagnose. Follow the section order below and use descriptive comment headers in **sentence case**
(e.g. `// GIVEN: a valid email and password`). If a section would be empty, omit it entirely.

### Standard Sections
1. **GIVEN**
   - Declare input data and context.
   - Variables must use the `given` prefix (e.g. `const givenEmail = …`).
   - Fixture factories should be invoked here and assigned to `given*` variables before mutation.
   - The GIVEN section must never be merged with any other section.

2. **CAPTURE**
   - Declare placeholders that will receive values captured by mocks, spies, or fakes.
   - Name these variables with the `capture` prefix (e.g. `let capturePublishedEvent`).
   - Keep this section limited to variable declarations; configure mocks to populate them in the MOCKING section.
   - Only include this section when the test truly needs to read captured collaborator state.

3. **MOCKING**
   - Create and configure mocks or fakes.
   - Mock variables must start with `mock` and fakes with `fake`.
   - Use Jest helpers (`jest.fn`, `jest.mock`, `vi.fn`, etc.) for dependencies.
   - Complex mocking logic may be split into sub-sections with additional descriptive comments.
   - The MOCKING section must not be merged with other sections.

4. **SETUP**
   - Prepare environment objects (e.g. request objects, dependency wiring).
   - Variables created here must use the `env` prefix (e.g. `const envRequest = …`).
   - Assign mock instances to `env*` variables before passing them to the system under test.
   - Reference `given*` data rather than literal values whenever possible.
   - Ensure SETUP follows the preceding arrangement sections (GIVEN → CAPTURE → MOCKING); omit intermediate sections when they are unnecessary.
   - The SETUP section must not be merged with other sections.

5. **SYSTEM UNDER TEST**
   - Instantiate the unit under test and assign it to `sut` (or `sutSomething` when multiple instances exist).
   - Do not pass `mock*` variables directly; always pass `env*` wrappers.
   - The SYSTEM UNDER TEST section must stand alone.

6. **WHEN**
   - Execute the action under test and assign outcomes to `actual*` variables.
   - Capture all asynchronous results with `await` and store them in `actual*` variables for later assertions.
   - When a test needs to inspect mutated `env*` values later, copy them into an `actual*` variable here.
   - Move any values stored in `capture*` placeholders into `actual*` variables before asserting on them.
   - The WHEN section must not be merged with other sections.

7. **EXPECTATIONS**
   - Define expected values before any assertions.
   - Expected values must use the `expected` prefix.
   - Do not reference `actual*` variables in this section.
   - The EXPECTATIONS section must appear between WHEN and THEN and cannot be merged with other sections.

8. **THEN**
   - Assert that `actual*` values match `expected*` values.
   - Never assert directly against literals or `given*` values.
   - Keep assertions focused on state validation; interaction verification belongs in BEHAVIOR.

9. **LOGGING** (optional)
   - Validate structured logs captured during execution (e.g. via fakes or spy transports).
   - Prefer log assertions when they clearly indicate which branch executed.

10. **BEHAVIOR**
   - Verify interactions with mocks or fakes (e.g. `expect(mockService.login).toHaveBeenCalledWith(…)`).
   - This section is mandatory whenever mocks are used.
   - All verifiable mock expectations must be checked here. Do not leave implicit expectations.

Use multiple sub-comments (e.g. `// SETUP: configure request object`) if a single section contains several logically distinct
steps.

✅ **Good Example:**
```js
test('Test_ReturnsToken_WhenCredentialsAreValid', async () => {
  // GIVEN: valid credentials
  const givenEmail = 'user@example.com';
  const givenPassword = 'secret';

  // MOCKING: configure auth service
  const mockAuthService = { login: jest.fn().mockResolvedValue('token-abc') };

  // SETUP: expose mocks as environment dependencies
  const envAuthService = mockAuthService;

  // SYSTEM UNDER TEST: create the controller
  const sut = new LoginController(envAuthService);

  // WHEN: user logs in
  const actualToken = await sut.login(givenEmail, givenPassword);

  // EXPECTATIONS: define expected outcome
  const expectedToken = 'token-abc';

  // THEN: compare outcomes
  expect(actualToken).toBe(expectedToken);

  // BEHAVIOR: verify interactions with collaborators
  expect(mockAuthService.login).toHaveBeenCalledWith(givenEmail, givenPassword);
});
```

🚫 **Bad Example:**
```js
test('Test_ReturnsToken_WhenCredentialsAreValid', async () => {
  const controller = new LoginController({ login: jest.fn() }); // ❌ No sections and mock passed inline
  const result = await controller.login('user@example.com', 'secret'); // ❌ literals used directly
  expect(result).toBe('token-abc'); // ❌ literal expectation
});
```

---

## 6. Fixtures: Best Practices
Reusable fixtures encourage realistic data while keeping tests concise. Inlining bespoke objects everywhere invites duplication
and hides relationships between tests.
- Prefer fixture factories (`createUserFixture`) over ad-hoc object literals.
- Store fixtures in dedicated helper modules under `tests/__fixtures__/` or `tests/helpers/fixtures.js`.
- Instantiate fixtures in the GIVEN or SETUP section and assign the result to a `given*` variable before mutating it.

✅ **Good Example:**
```js
// tests/helpers/fixtures/userFixture.js
export function createUserFixture(overrides = {}) {
  return {
    id: 'user-123',
    email: 'user@example.com',
    isActive: true,
    ...overrides,
  };
}
```

```js
// Inside a test
// GIVEN: a user needing activation
const givenUser = createUserFixture({ isActive: false });
```

🚫 **Bad Example:**
```js
// GIVEN
const givenUser = { id: 'user-123', email: 'user@example.com', isActive: true }; // ❌ duplicated literal data
```

---

## 7. Mocking: Best Practices
Disciplined mocking keeps tests reliable by ensuring only true collaborators are simulated.
- Mock only the collaborators the system under test uses directly; let their own unit tests cover internal behavior.
- Prefer fixtures or lightweight fakes when mocking becomes verbose or brittle.
- Create mocks with `jest.fn()`/`vi.fn()` and store them in `mock*` variables; assign them to `env*` variables in SETUP before
  injecting them into the SUT.
- Use Jest spies (`jest.spyOn`) only in the MOCKING section and clean them up in AFTER hooks if necessary.
- Assertions about mocks belong exclusively in the BEHAVIOR section.

### 7.1 Test Fakes
Fakes are handcrafted implementations that emulate collaborators more clearly than mocks when behavior is complex.
- Name fake instances with the `fake` prefix and create them in the MOCKING section.
- Assign each fake to an `env*` variable in SETUP before injecting it into the SUT.
- Provide helper assertion methods on fakes (e.g. `fakeLogger.assertLogged(level, message)`) and call them from LOGGING or
  BEHAVIOR sections.

---

## 8. Assertions & Variable Naming
Strict naming patterns keep inputs, expectations, and results easy to distinguish.
- Expected values must be stored in `expected*` variables before assertions.
- Actual values must be stored in `actual*` variables inside the WHEN section.
- Never assert against literals, `given*`, or `env*` values directly—always use `expected*` and `actual*` intermediaries.
- The only place a `mock*` variable can appear in an assertion is inside the BEHAVIOR section.

✅ **Good Example:**
```js
const expectedMessage = 'Welcome!';
expect(actualMessage).toBe(expectedMessage);
```

🚫 **Bad Example:**
```js
expect(actualMessage).toBe('Welcome!'); // ❌ literal expectation
```

---

## 9. Exception Handling
Treat exception capture as part of the WHEN stage so control flow remains obvious.
- For synchronous errors, wrap the SUT invocation inside `try/catch` in the WHEN section and store the error in an `actualError`
  variable.
- For asynchronous errors, use `await expect(() => …).rejects` or `await sutMethod().catch()` inside WHEN and capture the error.
- Assert on error type and message in the THEN section using `expected*` variables.

✅ **Good Example:**
```js
test('Test_Throws_WhenPasswordInvalid', async () => {
  // GIVEN
  const givenEmail = 'user@example.com';
  const givenPassword = 'wrong';

  // MOCKING
  const mockAuthService = { login: jest.fn().mockRejectedValue(new Error('Invalid password')) };

  // SETUP
  const envAuthService = mockAuthService;

  // SYSTEM UNDER TEST
  const sut = new LoginController(envAuthService);

  // WHEN
  let actualError;
  try {
    await sut.login(givenEmail, givenPassword);
  } catch (error) {
    actualError = error;
  }

  // EXPECTATIONS
  const expectedMessage = 'Invalid password';

  // THEN
  expect(actualError).toBeInstanceOf(Error);
  expect(actualError.message).toBe(expectedMessage);

  // BEHAVIOR
  expect(mockAuthService.login).toHaveBeenCalledWith(givenEmail, givenPassword);
});
```

---

## 10. Suite-Level Setup and Teardown
Hidden setup creates coupling between tests and obscures intent.
- Avoid `beforeEach`/`afterEach` unless the initialization is identical for every test in the suite.
- Prefer explicit GIVEN/SETUP sections so each test is self-contained.
- If you must use hooks, ensure they call shared fixture factories or helper functions, never inline complex logic.

---

## 11. Project Structure for Tests
```
project-root/
├── src/
│   ├── loginController.js
│   └── …
├── tests/
│   ├── loginController.test.js
│   ├── helpers/
│   │   └── fixtures/
│   └── …
└── package.json
```

---

## Summary
This guide adapts the detailed structure of the C# unit testing conventions to JavaScript. By mirroring module organization,
consistent naming, and disciplined sectioning, your Jest or Vitest suites remain easy to read, maintain, and extend.
