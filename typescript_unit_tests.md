# TypeScript Unit Testing Style Guide

This document adapts the **C# Unit Testing Style Guide** to the TypeScript ecosystem so teams and LLMs can generate tests with the
same structure, clarity, and rigor regardless of language. It assumes **Jest** (or a close alternative such as **Vitest**) as the
primary runner, but the guidance applies to any framework that exposes `describe`, `test` / `it`, and familiar assertion helpers.

---

## 1. Target Audience

The guidelines are written for both human developers and Large Language Models that author or refactor TypeScript unit tests. The
goal is to produce suites that are **consistent, reviewable, and intentionally scoped**. When the rules feel strict, remember that
they exist to make generated tests predictable and easy to audit.

---

## 2. Test Directory Organization

Consistent structure mirrors the production code layout so that finding the system under test (SUT) never requires guesswork.

- Keep all tests inside a dedicated root such as `tests/` or `__tests__/`.
- Mirror the production directory structure beneath that root (e.g., `src/services/payment.ts` → `tests/services/payment.test.ts`).
- Do not mix unit, integration, and end-to-end suites in the same directory tree—give each its own root.
- Avoid barrel files that re-export tests; each `.test.ts` file must be discoverable by globbing under the test root.

---

## 3. Test File Naming and Scope

Test files must make their responsibility obvious at a glance.

- Name files `<ProductionName>.test.ts` (preferred) or `test_<production_name>.ts`.
- Each file should cover **one production module or class**. If the production file exports multiple classes, create separate test
  files when their behaviors are unrelated.
- Keep helper-only files (e.g., factories or custom matchers) in sibling folders such as `tests/helpers/` to avoid being collected
  as test suites.

✅ **Good Examples:**
```text
src/
└── services/
    └── auth_service.ts

__tests__/
└── services/
    └── auth_service.test.ts
```

🚫 **Bad Examples:**
```text
__tests__/
└── everything.test.ts      // ❌ Covers more than one module
```

---

## 4. `describe` Blocks and Suite Structure

`describe` blocks are the TypeScript analogue of C# test classes. They provide context, let you scope setup helpers, and keep
reports readable.

- Every test file must export a top-level `describe` block that mirrors the production symbol under test (e.g., `describe('AuthService', ...)`).
- Nest additional `describe` blocks to mirror class regions, public methods, or significant behaviors. Keep the hierarchy shallow—one or two levels is usually enough.
- Avoid anonymous `describe` blocks; always pass a descriptive string.
- Do not place assertions outside of `test` / `it` blocks.

---

## 5. Test Case Naming

Readable names are the contract reviewers rely on to understand coverage.

- Every test function name must **start with** `Test_` and describe the behavior under test using `snake_case` segments joined by
  single underscores (e.g., `Test_authenticateUser_returns_valid_token`).
- When a `describe` block targets a specific method, omit the method name from the test title to avoid duplication (e.g., inside
  `describe('AuthService.authenticateUser', ...)`, use `Test_returns_valid_token`).
- Prefer `test()` over `it()` for clarity, unless the project mandates `it()`.
- Never use vague phrases like "works" or "handles" without clarifying the scenario.
- Each behavior gets its own test. If you need to cover variations, write separate cases instead of parameterized tables unless
  they significantly improve readability.

✅ **Good Example:**
```ts
describe('AuthService.authenticateUser', () => {
  test('Test_returns_valid_token', async () => {
    // ...
  });
});
```

🚫 **Bad Examples:**
```ts
test('should work', () => { /* ❌ Vague */ });
test('Test_authenticateUser_valid_token_success_and_failure', () => { /* ❌ Multiple scenarios */ });
```

---

## 6. Test Method Sectioning

Structured sections keep even complex tests readable and make it obvious when an assertion is missing. Follow the sections below in
order, omitting any that are not needed. Section headers must be full-sentence comments describing their purpose (e.g.,
`// GIVEN: A valid email and password`).

1. **GIVEN**
    - Declare inputs, configuration primitives, and fixture instances.
    - Prefix variables with `given` (e.g., `const givenEmail = 'user@example.com';`).
    - Do not perform side effects here—limit this section to pure data.

2. **CAPTURE**
    - Declare placeholders for values collaborators will capture (e.g., arguments forwarded to a mock).
    - Name them with the `capture` prefix (`let captureAuditEvent: AuditEvent | undefined;`).
    - Restrict this section to variable declarations; configure mocks or fakes to populate them elsewhere.
    - Omit CAPTURE entirely when no captured state is required.

3. **MOCKING**
    - Create and configure mocks or spies (e.g., `jest.fn()`, `vi.fn()`).
    - Prefix variables with `mock` and keep configuration grouped by responsibility. Large mock setups should be split with
      sub-headers such as `// MOCKING: Configure authentication client failure response`.
    - Only include this section when mocking is required.

4. **SETUP**
    - Prepare the execution environment (dependency containers, HTTP contexts, configuration objects, etc.).
    - Prefix variables with `env` (e.g., `const envLogger = createLogger(mockTransport);`).
    - Assign mocks to the concrete interfaces the SUT expects in this section (e.g., `const envAuthClient = mockAuthClient;`).
    - Ensure SETUP appears after whichever of GIVEN, CAPTURE, and MOCKING are present; skip unused sections without reordering them.

5. **SYSTEM UNDER TEST**
   - Instantiate the SUT and assign it to `sut` (or `sutFoo` if multiple instances are unavoidable).
   - The SUT should consume `env` dependencies, not raw `mock` objects.

6. **WHEN**
    - Execute the action under test and capture outputs in `actual`-prefixed variables.
    - For asynchronous operations, `await` the promise in this section before moving on.
    - When collaborators captured values into `capture*` placeholders, move them into `actual*` variables here so THEN focuses on verification rather than plumbing.

7. **EXPECTATIONS**
   - Define expected outcomes (`expected`-prefixed variables) before asserting.
   - Do not refer to `actual` variables while computing expectations.

8. **THEN**
   - Assert that actual values match expectations. Never assert directly against literals—compare `actual` and `expected` variables.
   - Prefer built-in Jest matchers with helpful diffs over custom logic.

9. **LOGGING** (optional)
   - Assert against captured log output when logging distinguishes execution paths.
   - Prefer log assertions to mock verifications when feasible—they are usually simpler to maintain.

10. **BEHAVIOR**
   - Verify interactions with mocks (`toHaveBeenCalled`, `toHaveBeenCalledWith`, etc.).
   - This section is mandatory whenever mocks are configured.

Keep sections visually separated by a blank line. Never merge two sections into a single comment (e.g., `// GIVEN & WHEN`).

---

## 7. Variable Naming Conventions

Use consistent prefixes to signal intent and stop reviewers from guessing where a value originated.

| Purpose            | Prefix     | Example                                 |
|--------------------|------------|-----------------------------------------|
| Inputs / fixtures  | `given`    | `const givenPassword = 'secret';`       |
| Environment setup  | `env`      | `const envRequest = createRequest();`   |
| Captured values    | `capture`  | `let captureAuditEvent: AuditEvent;`    |
| Mocks / spies       | `mock`     | `const mockAuthClient = jest.fn();`     |
| System under test  | `sut`      | `const sut = new AuthService(envDeps);` |
| Observed results   | `actual`   | `const actualToken = await sut.login();`|
| Expected values    | `expected` | `const expectedToken = 'abc123';`       |

Additional rules:
- Derive result booleans and numbers from `actual` variables instead of recomputing from `given` data.
- Reuse shared fixtures through factory helpers rather than inlining object literals in multiple tests.

---

## 8. Fixtures and Shared Helpers

Reusable fixtures keep tests expressive without duplicating setup logic.

- Place shared fixture builders under `tests/fixtures/` or `tests/helpers/fixtures/`.
- Name factory functions with an imperative verb (`createUser`, `buildAuthContext`).
- In tests, call fixture helpers within the **GIVEN** section and adjust the returned object locally when necessary.
- Avoid mutating shared fixture objects; instead, return new objects on each call.

---

## 9. Asynchronous and Exceptional Flows

Handle promises explicitly so failures surface in the correct section.

- Always `await` the SUT call inside the **WHEN** section unless you are testing rejection. For rejections, capture the promise in
  WHEN and assert inside THEN with `await expect(actualPromise).rejects...`.
- Keep the `await expect(...).rejects` expression inside the **THEN** section. Wrap the call under test in a closure only when the
  matcher requires it (e.g., `expect(() => sut.syncMethod()).toThrow()`).
- When validating thrown errors, also assert mock behavior in the **BEHAVIOR** section to prove the failure path executed.

---

## 10. Mock Verification and Spies

Mocks document collaboration; treat them with the same structure as value assertions.

- Prefer strict mocks created with `jest.fn()` or `vi.fn()` that fail when misused. Configure default `mockImplementation` values
  explicitly instead of relying on framework defaults.
- Every `mockImplementation` or `mockResolvedValue` that matters for the scenario must have a corresponding verification in the
  **BEHAVIOR** section (`toHaveBeenCalled`, argument checks, call counts, etc.).
- Reset or restore mocks at the end of the test only if leftover state would impact subsequent cases. Favor `jest.restoreAllMocks()`
  in a `afterEach` hook when necessary.

---

## 11. Comments and Docstrings

Comments provide a breadcrumb trail for reviewers, especially when tests are generated.

- Precede each test case with a divider and a one- or two-sentence docstring summarizing the behavior.
- Docstrings should reference the GIVEN, WHEN, and THEN steps in natural language.
- Inline comments are acceptable when they clarify **why** a value is chosen, not merely what it is.

```ts
// ------------------------------------------------------------------------------------------------------------
// Verifies that authenticateUser returns a valid token when credentials are correct.
// GIVEN valid credentials, WHEN authenticateUser executes, THEN the issued token matches the mocked response.
```

---

## 12. End-to-End Example

```ts
import { AuthService } from '../../src/services/auth_service';
import { createUser } from '../fixtures/users';

describe('AuthService.authenticateUser', () => {
  test('Test_returns_valid_token', async () => {
    // GIVEN: A registered user with valid credentials
    const givenUser = createUser();
    const givenEmail = givenUser.email;
    const givenPassword = 'correct-horse-battery-staple';

    // MOCKING: Configure the external authentication client to succeed
    const mockAuthClient = {
      login: jest.fn().mockResolvedValue({ token: 'abc123' })
    };

    // SETUP: Prepare environment dependencies consumed by the service
    const envLogger = { info: jest.fn(), warn: jest.fn(), error: jest.fn() };
    const envAuthClient = mockAuthClient;

    // SYSTEM UNDER TEST: Instantiate the service with environment dependencies
    const sut = new AuthService({ authClient: envAuthClient, logger: envLogger });

    // WHEN: Authenticating the user through the service
    const actualToken = await sut.authenticateUser(givenEmail, givenPassword);

    // EXPECTATIONS: Define the expected token response
    const expectedToken = 'abc123';

    // THEN: Assert that the returned token matches the expected value
    expect(actualToken).toBe(expectedToken);

    // BEHAVIOR: Ensure collaborators were invoked with the correct arguments
    expect(mockAuthClient.login).toHaveBeenCalledTimes(1);
    expect(mockAuthClient.login).toHaveBeenCalledWith(givenEmail, givenPassword);
    expect(envLogger.info).toHaveBeenCalledWith('User authenticated', { email: givenEmail });
  });
});
```

---

## 13. Quick Reference

| Guideline                  | Rule                                                                                 |
|----------------------------|--------------------------------------------------------------------------------------|
| Directory mapping          | Mirror the production folder structure inside the test root                          |
| File naming                | `<ProductionName>.test.ts` or `test_<production_name>.ts`, one SUT per file           |
| Suite naming               | Use `describe` blocks that mirror classes, regions, or methods                        |
| Test naming                | Strings begin with `Test_` and describe a single behavior                             |
| Section order              | `GIVEN → MOCKING → SETUP → SYSTEM UNDER TEST → WHEN → EXPECTATIONS → THEN → [LOGGING] → BEHAVIOR` |
| Variable prefixes          | `given`, `mock`, `env`, `sut`, `actual`, `expected`                                   |
| Assertions                 | Compare `actual` to `expected`; avoid literal comparisons                             |
| Mock verification          | Every significant mock setup is verified in **BEHAVIOR**                              |
| Documentation              | Divider + docstring before each test, descriptive section headers                     |

